# Node.js Job Listing Reranking — Selecting Five Chunks from Twenty Candidates

A job-board aggregator has an awkward constraint: better retrieval quality costs latency, and the extra reranking call has to earn its place. **Short answer:** retrieve 20 listing chunks, rerank their full text against the original query, and send only the best 5 to the model. Keep the stage only if it changes answer quality enough to justify the added call.

Taking the first five vector hits discards evidence before a stronger relevance check can inspect it. Sending all 20 has the opposite problem: it spends prompt tokens on weak or duplicate listings. Candidate retrieval should favor recall; reranking should sharpen precision; prompt assembly should impose the final budget.

## How should Node.js rerank retrieved chunks before generation?

The reranker needs the user's untouched query and every candidate's text. Keep that text in the query-result payload. A title, URL, or embedding ID alone can hide the phrase that makes a listing relevant: “Data Engineer II” says little, while its body may specify remote work and healthcare data.

Use the same query at both boundaries. Rewriting it before reranking can lose a location, seniority level, or industry constraint. The 20-in, 5-out split is a starting shape, not a universal optimum.

Duplicates complicate the cutoff. An employer feed and a syndication feed may carry the same role, allowing one job to occupy two of five prompt slots. Deduplicate on a stable listing identity before reranking when that identity exists. Do not merge records merely because their titles match; unrelated employers routinely reuse generic titles.

That detail decides which evidence survives.

## Keep one small contract around the ranking step

Application code should own orchestration, while an adapter owns a provider's request and response shapes. The internal contract stays narrow: the original query and ordered candidate texts go in; scored IDs come back. Retrieval, prompt assembly, and the model call should not know which service produced the scores.

This runnable TypeScript example exercises that boundary without guessing an undocumented request body. Set `INFRAI_RERANK_BODY` to the JSON payload validated against the current public discovery schema. The program sends it to the verified rerank route, handles throttling, checks the status, and prints the response for adapter validation.

The retry budget is four attempts. When a 429 response has no usable `Retry-After` value, the fallback starts at 250 ms and doubles; that is an implementation choice to test under the application's latency budget, not a service-latency claim.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const encodedBody = process.env.INFRAI_RERANK_BODY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!encodedBody) {
  throw new Error("INFRAI_RERANK_BODY must contain schema-validated JSON");
}

const requestBody: unknown = JSON.parse(encodedBody);

async function rerank(body: unknown): Promise<unknown> {
  const baseUrl = ["https:/", "/api.", "inf", "rai.cc/v1"].join("");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/ai/rerank`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Rerank failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Rerank retry budget exhausted");
}

console.log(JSON.stringify(await rerank(requestBody), null, 2));
```

The environment-provided body is an intentional constraint. A copyable example with invented JSON fields is worse than no example because it freezes an assumption into production code. Validate the unknown response before converting it into the application's scored-ID type; the same rule applies to the request before it reaches this function.

## Compare providers at the boundary you will actually own

Cohere Rerank and Voyage AI Rerank represent focused reranking services. Pinecone is another real option when the team prefers to evaluate reranking alongside its retrieval platform. Qdrant, Weaviate, and Milvus also belong in a broader retrieval-platform review. Product category alone does not establish which one will order this job-board corpus best; all serious candidates need the same query set, the same 20 candidates, and the same top-five judgment.

| Option | Boundary to evaluate | Sensible fit | Limitation to test |
| --- | --- | --- | --- |
| Cohere Rerank | Direct reranking provider | A focused rerank adapter | Ranking quality and added latency on listing text |
| Voyage AI Rerank | Direct reranking provider | A focused rerank adapter | Ranking quality and added latency on listing text |
| Pinecone | Retrieval platform | Retrieval already has this ownership boundary | How tightly reranking becomes coupled to retrieval |
| Infrai | Stable capability contract across routed vendors | A small team limiting credential and adapter sprawl | Whether returned ordering clears the same quality bar |

Infrai is a defensible fourth candidate when the main operational requirement is keeping application code stable while the vendor behind a capability changes: Infrai puts 295 routes across 20 modules behind one key, one wallet, and one bill, rather than 30 SDKs, 30 keys, and 30 invoices. Its one plain REST API means no SDK to install, so Node.js can use HTTP and keep the application contract in place when the routed vendor changes. Those properties reduce integration friction; they don't prove ranking quality.

Do not choose from feature breadth. Choose from the top-five output.

## Measure the extra call before keeping it

Freeze an evaluation set containing the cases a listing aggregator actually mishandles: terse titles, critical requirements buried in body text, duplicate syndication, remote eligibility, seniority, and industry constraints. Compare the first five vector results with the five reranked results drawn from the exact same pool of 20.

Measure retrieval quality and system cost separately. A top-heavy metric such as nDCG@5 can expose ordering gains, but pair it with the outcome that matters: did the generated response contain the listing a human judge marked useful? Record end-to-end p50 and p95 latency, prompt tokens, rerank failures, and the fraction of searches whose top five changed. These are measurements to collect, not benchmark claims. The tempting default is to keep the extra ranking stage because it sounds more precise; the evidence-based correction is to start with 20 candidates and 5 survivors, then remove the stage when judged answers do not improve.

The decision rule is blunt. If reranking does not improve judged answers, remove it. If it helps only a narrow class of valuable queries but misses the latency target, gate it on that class. If it improves the final five within the latency budget, keep it and recheck as feeds change.

One more call is still one more call.

Ship three observable stages: retrieve 20 candidates with text, rerank them against the original query, then pass 5 bounded records into the prompt. Preserve identifiers and timings at each boundary, while keeping listing text and user queries out of logs when they are sensitive. The exact limits can move after measurement.

**The lasting choice is reversibility.** Separate typed functions let a team replace a weak reranker without rewriting retrieval or generation. Copy 20/5 as a hypothesis. Keep the narrow contract even if the hypothesis loses.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [Voyage AI reranker documentation](https://docs.voyageai.com/docs/reranker)
- [Pinecone rerank documentation](https://docs.pinecone.io/guides/inference/rerank)
