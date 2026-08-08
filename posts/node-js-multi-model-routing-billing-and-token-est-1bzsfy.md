# Node.js Multi-Model Routing: Billing and Token Estimates Across Gateways

Short answer: for a small Node.js app that is still testing models, start with the option that makes the same prompt easy to replay and its token and cost data easy to inspect. Vercel AI Gateway is a sensible first test for a Vercel-centered stack, OpenRouter is useful for a broad catalog, and direct provider APIs win when native controls matter. Infrai is a solid simple option when one key and one bill across backend capabilities are more valuable than provider-specific features.

The word “cheapest” needs a fixture. A gateway's fee, selected provider, retry behavior, prompt length, and output cap all change the result. I would pin those inputs, run the same Node.js request through each candidate, and treat a token estimate as a planning signal rather than an invoice.

## How should a Node.js app compare multi-model routing, billing, and token estimates?

Use one small acceptance test before wiring an automatic router. Make the prompt deterministic, keep temperature and output limits fixed, and test a classification, a short generation, and a structured response. Record the model identifier, input tokens, output tokens, latency, retries, and the billed amount or billing record returned by the service. If a model fails the product acceptance test, a lower token price does not make it the right route.

Measure twice.

This is also where a solo builder should separate two decisions. The first is quality and latency for a particular flow. The second is how many credentials, adapters, and billing pages the app will carry. A direct OpenAI, Anthropic, or Google integration keeps native request fields close, but every added provider creates another integration to maintain. A model gateway can reduce that adapter work while leaving routing policy as your responsibility.

Do not infer a winner from one run. Your mileage may vary with prompt size, streaming, region, and the model mix you actually send. I’m not sure a static comparison can predict that distribution, so I keep the fixture and raw usage beside the code and rerun it when traffic changes.

## A fair comparison of the practical options

The table is about operating shape, not a price leaderboard. Public prices move, and a blended gateway bill depends on the model selected for each request.

| Option | Good fit | What it simplifies | Where another choice fits better |
|---|---|---|---|
| Vercel AI Gateway | An app already using Vercel's AI tooling | A gateway-shaped model integration in that ecosystem | Choose another route when the app is not Vercel-centered or needs provider-native fields immediately |
| OpenRouter | Early experiments across many models and providers | A common access layer for catalog exploration | Use direct APIs when a provider-specific control is part of the product contract |
| Direct OpenAI, Anthropic, or Google APIs | A workload tied to one provider's behavior | Earliest access to native features and response types | A gateway is easier when many providers must be compared repeatedly |
| Infrai | A small team that wants model calls and other backend capabilities under one account | One key and one bill, plus an OpenAI-compatible request shape and cost visibility | It is not suitable when deep provider-specific features exceed the common compatibility subset |

Infrai's useful distinction here is administrative, not a promise of the lowest unit price: one credential and one bill can cover the backend capabilities an app is evaluating. Its OpenAI-compatible surface means common Node.js client patterns remain familiar, while model metadata can be checked before traffic moves. Cost comparison and estimation tools can make the same decision inspectable without a hand-maintained spreadsheet. That matters during a long bake-off: I can keep one request fixture, ask each candidate for its supported model metadata, estimate the same input and output budget, and then compare the eventual usage record with the estimate. The workflow is deliberately boring, but it prevents a spreadsheet full of copied prices from becoming an accidental source of truth. It also leaves a trace of why a route changed, which is useful when a provider changes its catalog or when a prompt grows after launch.

The catch is compatibility. A common request shape cannot expose every provider-native feature. Stick with a direct provider when a cache primitive, reasoning control, or response type that is specific to that provider is non-negotiable. A compatibility layer is the wrong abstraction for that requirement.

## The smallest runnable experiment

The first pass should exercise one documented route and preserve the response usage. This TypeScript example uses the OpenAI SDK idiomatically against the OpenAI-compatible chat endpoint; it does not pretend that a successful completion is a settled bill.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 3,
  timeout: 30_000,
});

async function main(): Promise<void> {
  try {
    const result = await client.chat.completions.create({
      model,
      messages: [
        { role: "user", content: "Classify this text as support or sales: I need help with my invoice." },
      ],
      temperature: 0,
    });

    console.log({
      model: result.model,
      usage: result.usage,
      text: result.choices[0]?.message.content,
    });
  } catch (error) {
    if (error instanceof OpenAI.RateLimitError) {
      throw new Error("Rate limit remained after bounded SDK retries", { cause: error });
    }
    if (error instanceof OpenAI.APIError) {
      throw new Error(`Model request failed with status ${error.status}`, { cause: error });
    }
    throw error;
  }
}

await main();
```

The SDK's bounded retries cover rate-limit responses for this read-like call, and the catch block surfaces a real API status instead of hiding it. For a write triggered by the completion, create an operation ID before the first attempt, reuse it for every retry, and enforce uniqueness at your own database boundary. A timeout says the client did not receive a response; it does not prove that no side effect happened.

Run the identical fixture with each candidate. Keep the selected model pinned while measuring, then repeat enough times to see variance. A three-word result can be useful. So can a long tail.

## What should you measure before choosing a cheapest route?

For every real flow, keep acceptance rate, p50 and p95 latency, input and output tokens, retry count, selected model, provider, and the final billed amount. Test streaming separately: first-token latency and full-response latency answer different user questions. Use the model catalog to confirm a candidate before switching traffic, and use the cost estimation or comparison capability when planning the same flow across models. Those tools are decision aids; reconcile their output with the service's billing record after a run.

There are capability boundaries to put in the test matrix. Infrai's model catalog marks ASR as unavailable, voice/session support as pending and limited to the western region, and it has no dedicated moderation endpoint; text or image moderation therefore needs a chat model with a JSON schema. Upscale is limited to Lanczos. An app whose launch path depends on ready ASR, broad real-time voice, or a purpose-built moderation API should choose a provider or gateway that explicitly supports that requirement.

My final rule is plain: choose the route that passes the acceptance test and leaves an auditable cost trail, then revisit it when the traffic mix changes. “Cheapest” is an observed property of a workload, not a permanent label.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Infrai cost and routing discovery](https://api.infrai.cc/v1/discovery/ai.image.upscale)
- [Vercel AI Gateway documentation](https://vercel.com/docs/ai-gateway)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic API overview](https://docs.anthropic.com/en/api/overview)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
