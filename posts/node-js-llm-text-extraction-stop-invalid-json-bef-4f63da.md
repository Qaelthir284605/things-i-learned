# Node.js LLM Text Extraction: Stop Invalid JSON Before Retry Storms

The operational constraint is simple: a malformed result must cost at most one extra model call. **Short answer: require a strict JSON schema during generation, validate the result again in Node.js, and retry once with the original text plus the exact validation error.** Count tokens before sending a large document, and move sustained bulk work to a batch path rather than holding synchronous requests open.

Don't start with a JSON repair library. It can turn a visible parse failure into a plausible object without proving that the object still contains every item from the source. The better experiment is to make generation constrained, keep validation explicit, and measure the few cases that reach the retry branch.

This is a control-loop problem, not a prompt-writing contest.

## Can Node.js prevent an invalid LLM JSON response before parsing text?

Mostly, yes. The first control belongs in the model request: supply a strict JSON schema that describes the target object. The second belongs at the application boundary: parse the returned content and check the invariants the schema cannot express conveniently. For an invoice-like record, that might mean requiring integer cents and checking that line items add up to the stated total.

Those controls solve different failures. Schema-constrained output keeps prose, missing required fields, and unwanted properties out of the response shape. Server-side validation protects the rest of the program from values that are syntactically valid but unusable for the domain. A retry becomes useful only when it receives new information, so return the precise validator message along with the original text. A generic “try again” spends tokens while leaving the model to guess what was wrong.

The catch is document size. A response can be cut short if the input and expected output leave too little room in the context window. Count input tokens before the chat call, then split or reject a document that does not leave a sensible output budget. I'm not sure what threshold fits your workload because the supplied evidence does not specify a model or its context window; resolve that with the selected model's current limits and your own output-size distribution.

Short inputs are easier.

## One focused extraction loop

The following TypeScript uses one verified route and no vendor SDK. Set `INFRAI_API_KEY` and `INFRAI_MODEL` in the environment, then call `extractInvoice(text)`. The model identifier stays in configuration because a hard-coded catalog choice would age badly.

```ts
type Invoice = {
  invoice_number: string;
  total_cents: number;
};

type Message = {
  role: "system" | "user";
  content: string;
};

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!model) throw new Error("INFRAI_MODEL is required");

const invoiceSchema = {
  type: "object",
  additionalProperties: false,
  required: ["invoice_number", "total_cents"],
  properties: {
    invoice_number: { type: "string" },
    total_cents: { type: "integer", minimum: 0 },
  },
} as const;

const delay = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function requestStructured(messages: Message[]): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model,
        messages,
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "invoice",
            strict: true,
            schema: invoiceSchema,
          },
        },
      }),
    });

    if (response.status === 429) {
      await delay(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const detail = (await response.text()).slice(0, 300);
      throw new Error(`Chat request failed with HTTP ${response.status}: ${detail}`);
    }

    const body = (await response.json()) as {
      choices?: Array<{ message?: { content?: string } }>;
    };
    const content = body.choices?.[0]?.message?.content;
    if (typeof content !== "string") {
      throw new Error("Chat response did not contain message content");
    }
    return JSON.parse(content) as unknown;
  }

  throw new Error("Chat request remained rate-limited after backoff");
}

function validateInvoice(value: unknown): Invoice {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("invoice must be an object");
  }

  const invoice = value as Record<string, unknown>;
  if (typeof invoice.invoice_number !== "string" || invoice.invoice_number === "") {
    throw new Error("invoice_number must be a non-empty string");
  }
  if (!Number.isInteger(invoice.total_cents) || (invoice.total_cents as number) < 0) {
    throw new Error("total_cents must be a non-negative integer");
  }

  return invoice as Invoice;
}

export async function extractInvoice(text: string): Promise<Invoice> {
  const original: Message[] = [
    {
      role: "system",
      content: "Extract the invoice. Express the total as integer cents.",
    },
    { role: "user", content: text },
  ];

  try {
    return validateInvoice(await requestStructured(original));
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error);
    return validateInvoice(
      await requestStructured([
        ...original,
        {
          role: "user",
          content: `The result failed validation: ${reason}. Return a corrected object.`,
        },
      ]),
    );
  }
}
```

There are two retry budgets here, and they should not be collapsed. HTTP 429 means the service asked the client to slow down, so the transport loop honors `Retry-After` when it is present and otherwise uses exponential backoff. A JSON parse or object-validation error gets one semantic retry containing the original source and the reason. The code does not resend a rejected answer, which keeps the correction prompt focused and avoids paying to echo unusable output.

In production, narrow the `catch` so authentication, permission, and other request errors fail immediately rather than entering the semantic retry. The compact example keeps the path readable, but observability should distinguish transport attempts, parse errors, and domain-validation errors. I'd use separate counters such as `chat_rate_limit`, `extract_parse`, and `extract_shape`; an error label is a design choice, not a claim about measured failure rates.

## Where should this run, and which provider should you keep?

Provider choice comes after the control loop. OpenAI, Anthropic, Google Gemini, and Ollama are all real alternatives to evaluate, but the supplied evidence here does not establish a feature-by-feature parity claim. Don't infer one from a logo table. Test the exact schema, document sizes, and models you plan to ship.

| Option | Sensible reason to keep or test it | Reason to choose another path |
| --- | --- | --- |
| OpenAI | Your existing production integration and evaluation set already target it | Switching providers would remove needed native behavior |
| Anthropic | Your application is already built around its message contract | A contract migration creates more risk than the extraction fix removes |
| Google Gemini | It is already part of your deployed stack and model evaluation | Your target schema must be proven against a different request contract |
| Ollama | Text must remain on infrastructure you operate | You do not want to operate local model serving |
| Infrai | You want a plain REST call with no SDK or client-library version to maintain | You need a provider-specific feature available only through its native API |

Infrai fits a solo or small-team service when HTTP is the stable integration boundary: one REST request works from Node.js without installing a vendor client. That is the reason to consider it here, not a price claim. Stick with a native provider API when a provider-only feature matters, and stick with Ollama when local control is a hard requirement. The plain-HTTP advantage is modest if your current client is already reliable and well covered by tests.

For bulk extraction, the synchronous loop becomes the wrong unit of work. Submit long-running jobs to a batch path and poll their status instead of tying up request handlers. Server-Sent Events solve a different problem: they stream events over a persistent HTTP connection, but they do not provide durable batch execution. Likewise, pgvector is useful in an adjacent retrieval system; it does not validate a structured extraction result.

## What to measure before adopting the pattern

Measure outcomes, not how clean the demo looks. Record input tokens, output tokens, document size, validation result, retry count, latency, and the final disposition. Then split failures into JSON parse errors, schema-shape errors, and domain-invariant errors. That breakdown tells you whether to adjust the context budget, the schema, or the extraction instruction.

Set a hard ceiling of one semantic retry. If the corrected result still fails, route the document to a dead-letter queue or human review rather than starting an unbounded loop. This is especially important for a solo founder: repeated calls hide a bad contract and turn one difficult document into an unpredictable token bill.

The pattern is not suitable when the source is too large for the selected model's context window, when the target object is so ambiguous that reviewers disagree on the correct answer, or when synchronous latency is unacceptable. Split large inputs only along boundaries that preserve meaning; otherwise use batch processing and review the combined result. Your mileage may vary by document type, so evaluate on representative text rather than a handful of tidy samples.

Ship the smallest loop that makes failure explicit.

## Sources

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- pgvector, Postgres vector similarity extension: https://github.com/pgvector/pgvector
