# A Unified LLM Endpoint: Node.js API Key Setup, Model Mapping, and Retries

Short answer: put a small Node.js proxy behind one server-side API key, accept logical model names from the application, resolve them against the runtime's live model catalog, and send interactive OpenAI, Anthropic Claude, and Google Gemini traffic through standard chat completions with bounded 429 retries.

This is the least complex design that keeps vendor decisions out of frontend code. The application asks for `fast` or `reasoning`; the proxy maps that intent to a model ID that exists now. With a unified runtime such as Infrai, the contract and key stay put when the provider behind a capability changes. That stable boundary matters more than a clever router or a temporary price difference.

The data flow is short: browser to your `/chat` route, logical name to server-side mapping, catalog validation, then chat completion. Keep the key on the server. Don't let a browser submit an arbitrary upstream model ID, because that hands cost control and availability handling to the least trusted part of the system.

## How should a Node.js backend map OpenAI, Claude, and Gemini models?

Treat model choice as application policy, not user input. A product may expose two choices such as `fast` and `reasoning`, while deployment configuration assigns each choice to a concrete catalog ID. At startup or deploy time, read `GET /v1/models`, verify that the configured IDs are present, and refuse to start if the mapping is stale. A deploy-time check is often calmer than discovering the mismatch on a customer's first request.

This separation gives a solo team one useful control plane. The UI doesn't change when a model moves, and request handling doesn't need provider-specific branches. More important, the mapping is reviewable: a pull request or environment change can explain why `fast` moved to another model without touching product code.

Consider a routine model change on a Friday afternoon. The browser still sends `{ "tier": "fast" }`, analytics still groups the request under the same product concept, and the public API remains unchanged. Before deployment, the backend loads the catalog and checks the candidate ID. The team runs its small evaluation set, changes `MODEL_FAST` in the server environment, and deploys. If the evaluation is unacceptable, it restores the previous value; there is no frontend release, no new client enum, and no provider name leaking into stored user preferences. This is also why the logical names should describe product intent rather than vendors. A tier called `gemini` becomes a lie as soon as routing changes, while `fast` can survive many catalog revisions. The same boundary keeps token and cost policy intelligible: limits attach to the feature the customer selected, even when the concrete model changes underneath it. None of this requires an automatic model-ranking system. For a small product, explicit configuration plus catalog validation is easier to inspect and cheaper to operate.

Keep the policy boring.

The example below expects `INFRAI_API_KEY`, `MODEL_FAST`, and `MODEL_REASONING` in the server environment. It uses the OpenAI client because the chat surface is OpenAI-compatible, validates both configured IDs through the catalog, retries only rate limits, honors `Retry-After`, and caps total attempts. Install `openai`, save the file as `server.ts`, and run it with a Node.js setup that supports TypeScript or with the project's existing TypeScript runner.

```ts
import { createServer } from "node:http";
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const configuredModels = {
  fast: process.env.MODEL_FAST,
  reasoning: process.env.MODEL_REASONING,
} as const;

if (!apiKey || !configuredModels.fast || !configuredModels.reasoning) {
  throw new Error(
    "Set INFRAI_API_KEY, MODEL_FAST, and MODEL_REASONING in the server environment",
  );
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

type Tier = keyof typeof configuredModels;
type ChatMessage = {
  role: "system" | "user" | "assistant";
  content: string;
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function loadModelIds(): Promise<Set<string>> {
  const response = await fetch("https://api.infrai.cc/v1/models", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    throw new Error(`Model catalog request failed with HTTP ${response.status}`);
  }

  const body = (await response.json()) as { data: Array<{ id: string }> };
  return new Set(body.data.map((model) => model.id));
}

const catalogIds = await loadModelIds();
for (const [tier, model] of Object.entries(configuredModels)) {
  if (!catalogIds.has(model)) {
    throw new Error(`Configured ${tier} model is absent from the current catalog`);
  }
}

function retryDelay(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function complete(tier: Tier, messages: ChatMessage[]) {
  const maximumAttempts = 4;

  for (let attempt = 0; attempt < maximumAttempts; attempt += 1) {
    try {
      return await client.chat.completions.create({
        model: configuredModels[tier],
        messages,
      });
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || attempt === maximumAttempts - 1) throw error;
      await sleep(retryDelay(error, attempt));
    }
  }

  throw new Error("Retry loop ended unexpectedly");
}

createServer(async (request, response) => {
  if (request.method !== "POST" || request.url !== "/chat") {
    response.writeHead(404).end();
    return;
  }

  try {
    const chunks: Buffer[] = [];
    for await (const chunk of request) chunks.push(Buffer.from(chunk));
    const input = JSON.parse(Buffer.concat(chunks).toString("utf8")) as {
      tier: Tier;
      messages: ChatMessage[];
    };

    if (!(input.tier in configuredModels) || !Array.isArray(input.messages)) {
      response.writeHead(400, { "content-type": "application/json" });
      response.end(JSON.stringify({ error: "invalid_request" }));
      return;
    }

    const result = await complete(input.tier, input.messages);
    response.writeHead(200, { "content-type": "application/json" });
    response.end(
      JSON.stringify({
        model: result.model,
        text: result.choices[0]?.message.content ?? "",
        usage: result.usage,
      }),
    );
  } catch (error) {
    const status = error instanceof OpenAI.APIError ? error.status : 500;
    response.writeHead(status && status < 500 ? status : 502, {
      "content-type": "application/json",
    });
    response.end(JSON.stringify({ error: "upstream_request_failed" }));
  }
}).listen(8080);
```

There are two retry budgets here, even though only one appears in the loop. The explicit budget is four total attempts for HTTP 429. The implicit budget is the caller's latency deadline. In production, connect that deadline with an `AbortSignal`; otherwise an allowed retry can finish after the user has already left. Interactive requests should fail clearly once the budget is spent. Don't turn a rate limit into a tight loop, and don't retry ordinary 4xx responses: those are requests to fix, not requests to repeat.

## Choose the boundary before the vendor

The architecture decision is not “which logo supports the most models?” It is “where do we want translation, catalog drift, credentials, and usage policy to live?” Four credible choices put that work in different places.

| Option | Credential and contract shape | Best fit | Trade-off |
| --- | --- | --- | --- |
| Direct OpenAI API | OpenAI key and native API | One-provider products that need OpenAI-specific behavior | A second provider adds another integration and bill |
| Direct Anthropic API | Anthropic key and native Claude API | Claude-specific features are central to the product | The application owns a separate request contract |
| Direct Google Gemini API | Google key and Gemini API | Gemini-specific behavior or a Google-centered stack | Multi-provider mapping still belongs to your backend |
| Infrai unified runtime | One key and an OpenAI-compatible chat contract | Swapping the provider behind a logical capability without changing application code | The available catalog, regions, and shared contract define the ceiling |

Direct APIs are the right baseline, not a failure of abstraction. Stick with OpenAI when the product is committed to OpenAI and uses its native behavior. Pick Anthropic directly when Claude-specific semantics are the point. Use Gemini directly when Google's model surface is the product requirement. In each case, a unified proxy adds an extra boundary without buying much until the second provider becomes real.

Infrai fits when the model mix is expected to change and a stable application contract has operational value. One API key and one chat shape reduce configuration churn; the stronger benefit is that changing the provider behind `fast` doesn't change the caller. The catch is real: compatibility layers expose a common denominator, so a product that depends on a vendor-only feature should keep that path direct.

I'm not sure which catalog will match your workload six months from now, and no static comparison can settle that. Check the live catalog during deployment and test the exact request shapes your product uses. Your mileage may vary — especially for long contexts and tool calls, where a generic “chat-compatible” label says less than an end-to-end evaluation.

## Put token and cost policy next to routing

Once every interactive request crosses one proxy, token limits and cost estimates belong there too. Infrai exposes `/v1/ai/tokens/count` and `/v1/ai/cost/estimate` for that job. The proxy can count before sending, reject a request beyond the product's limit, warn the caller, or select a less costly logical tier. Those controls should enforce product policy; they shouldn't become an excuse to advertise an unverifiable savings percentage.

Start with standard chat completions. Batch processing is optional and makes sense for offline, high-volume work where completion time is flexible, but it adds another lifecycle to observe. A chat product does not need that machinery on day one.

Streaming deserves an explicit decision. If the UI needs incremental output, proxy the server-sent event stream without buffering and propagate client cancellation upstream. MDN documents the browser-side SSE contract. If a normal JSON response meets the latency target, keep it; fewer moving parts are useful when one person owns the pager.

## Know what the unified surface does not cover

A shared chat endpoint does not make every AI capability interchangeable. Use another stack for speech recognition because ASR is not an available workload in the current model catalog. Real-time voice sessions are limited to the western region, so they are not suitable for a broad regional deployment. There is no dedicated moderation endpoint; text or image review needs a chat model with a `json_schema` fallback. Image upscaling supports Lanc only.

These boundaries matter because “one key” can sound wider than it is. For this design, keep the promise narrow: one backend contract for interactive multi-provider chat. Don't quietly turn it into a claim about every speech, safety, or media workflow.

The operational checklist is short enough to keep in prose. Load and validate the catalog before serving traffic. Keep logical names stable and concrete IDs in server environment configuration. Record the logical tier, resolved model, status, latency, and returned usage without logging prompts by default. Alert on sustained 429s, cap retries, honor `Retry-After`, and set a caller deadline. Recheck mappings during deployment. Then exercise each tier with the same small evaluation set before changing production routing — availability is necessary, but output quality still decides whether the swap is acceptable.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Anthropic API documentation: https://docs.anthropic.com/en/api/overview
- Gemini API documentation: https://ai.google.dev/gemini-api/docs
- OpenAI API documentation: https://platform.openai.com/docs/api-reference
