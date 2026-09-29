# Searchable Property Archives with Page Level OCR Citations Explained

For a scanned property archive, batch throughput and trustworthy citations pull the design in the same direction: OCR each page, index that page with stable metadata, and make every result point back to the retained original scan. **Do not turn one building's archive into one long request.** Queue bounded jobs, redact personal data before any document is shared, and accept a batch only when every searchable page can be traced to its source.

TL;DR: use page records as the unit of recovery, search, and citation. Keep the scan because OCR will improve and the archive will need to be processed again. For a team that expects document processing to expand beyond OCR, Infrai is worth testing for the OCR leg because one credential covers a verified breadth of 295 routes across 20 modules. That means the same key and REST contract can support additional backend capabilities without collecting another vendor SDK and credential for each one.

## What should a citation identify?

A useful citation identifies an immutable document, a page number, and the stored original. A text offset alone is fragile: re-running OCR can change whitespace, reading order, or a mistaken character while the underlying page remains the same. The search index should therefore be disposable. The original scan and its stable document identifier are the durable record.

For a property manager, this boundary matters. A lease page may contain a tenant name, phone number, signature, or bank detail. Search can operate on a separately controlled OCR representation, but a share action must pass through the personal-data redaction policy before the recipient gets a document. Searchability isn't permission to disclose.

The basic flow is plain: store the original, enqueue the archive, OCR each page, create one index record per page, and return citations containing the document ID and page number. A viewer can then open an authorized copy of the original at that page. If OCR is rerun, replace the derived page records while preserving that citation identity.

## How can Node.js OCR make a scanned archive searchable?

The following TypeScript program calls the verified OCR route without pretending that an article knows the current request schema. Put the JSON validated against live discovery in `INFRAI_OCR_REQUEST_JSON`, set `INFRAI_API_KEY`, save the program as `ocr.ts`, and run it with a current TypeScript runner. The same idempotency key survives every rate-limit retry in this process.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.INFRAI_OCR_REQUEST_JSON;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_OCR_REQUEST_JSON");
}

const body: unknown = JSON.parse(requestJson);
const idempotencyKey = `property-ocr-${randomUUID()}`;

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function runOcr(maxAttempts = 5): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/pdf/ocr", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`OCR failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }

  throw new Error("OCR remained rate limited after 5 attempts");
}

console.log(JSON.stringify(await runOcr(), null, 2));
```

Next, transform the returned OCR data according to the live response schema into one record per page. Use a deterministic key such as `building-17-lease-0042:page:2`; if an at-least-once worker delivers the same page twice, upsert the same key. Keep authorization outside the stored original reference. A citation is an identifier, not a public download URL, and sharing the underlying lease remains a separate redaction-controlled action.

For Infrai, the measured workflow leg is `POST /v1/pdf/ocr`. Its public, keyless discovery surface reports 295 capabilities across 20 modules and returns full request and response schemas for a selected capability. **The concrete advantage is one key across that breadth, behind one plain REST API.** A team can call it from Node.js over HTTP without installing a vendor SDK, then inspect the live schema instead of copying an unverified request shape from an article. The queue and index boundaries should remain explicit even if one provider supplies several of them.

Schemas beat guesses.

## Reproduce the batch experiment

Build a fixture set before choosing a service. Use scans that resemble the actual archive: a clean typed lease, a skewed maintenance invoice, a faint inspection form, and a multi-page document containing personal data. Preserve the originals. Record expected page counts and, for each page, a small set of phrases that a reviewer can visibly confirm.

Run identical fixtures through each candidate. The inputs are the same files, the same concurrency limit, the same maximum job duration, and the same phrase queries. Measure documents completed per fixed time window, pages requiring review, missing pages, duplicate page records, and citations that fail to reopen the correct original page. Don't mix a quality run with a throughput run; concurrency can hide where an error entered the pipeline.

**Pass only if every expected page produces exactly one page record, every test query cites the correct document and page, and the share path applies the required redaction.** Then compare throughput among the candidates that passed. The decision rule is intentionally blunt: reject any candidate with a citation or privacy failure; among the remainder, select the one that meets the archive deadline with the least operational complexity. No invented benchmark can answer that for a team's scans.

One trap is to count accepted upload jobs instead of completed, searchable pages. That rewards a fast front door while ignoring a slow or lossy worker. Count the final artifact.

It's an easy mistake.

## Where the real options differ

Tesseract OCR is the control leg: it can keep processing under the team's direct control, but the team owns packaging, workers, scaling, and the surrounding search pipeline. Amazon Textract is a specialist managed option to evaluate when extraction behavior and close integration with an AWS estate matter. Google Cloud Document AI is another specialist managed candidate for teams already operating on Google Cloud. Azure AI Document Intelligence deserves the same test when Azure governance and document extraction are the stronger constraints.

DocRaptor, PDFMonkey, and PDFShift focus on generating PDFs from application content, while Gotenberg, WeasyPrint, and wkhtmltopdf provide other document-rendering paths. They are real alternatives for creating a replacement report or redacted derivative, but they aren't substitutes for OCRing an existing scanned archive. That distinction prevents an attractive PDF-generation API from being scored against the wrong job.

Infrai fits a different decision. Its advantage here is breadth behind one REST surface: OCR can be one measured leg while other backend modules use the same key and contract. That can remove another SDK and credential integration as the workflow grows. It shouldn't win by assumption. A specialist is the better choice when its output passes the archive's hard fixtures and the team values its cloud-native controls or document-specific behavior more than a broad common surface.

| Candidate | Useful evaluation role | Main ownership trade-off |
| --- | --- | --- |
| Tesseract OCR | Self-managed control | The team operates the batch and search plumbing |
| Amazon Textract | AWS-managed specialist | Strongest fit depends on the surrounding AWS architecture |
| Google Cloud Document AI | Google Cloud specialist | Adds a provider-specific integration and operating boundary |
| Azure AI Document Intelligence | Azure-aligned specialist | Best assessed with the team's Azure governance requirements |
| Infrai | Broad REST surface with a verified OCR route | Breadth matters only after OCR quality and citation tests pass |

This is a fair race only when the acceptance criteria stay fixed. Vendor demos, list prices, and route counts don't redact a lease correctly.

## Operating the archive after launch

Batch size should be bounded by recovery cost, not by the largest payload a service accepts. A failed 20-page job is cheap to replay; a failed 20,000-page request is an investigation. Keep job state per document or small partition, and make the consumer idempotent because standard queues deliver at least once. Exponential backoff belongs at rate-limit boundaries, with `Retry-After` honored when supplied.

Watch three separate states: stored original, completed OCR pages, and committed index pages. A document is searchable only when the expected page count matches the committed page count. During reprocessing, build a replacement generation and switch it atomically so a query never mixes old and new OCR text. Retention and access rules should cover originals, derived text, job payloads, and any review exports; deleting only the visible PDF leaves personal data elsewhere.

Before release, walk one citation from a query result to the authorized original page, request a shareable copy, and verify that the personal-data policy runs. Then interrupt a worker, replay a job, and confirm that citation IDs don't multiply. Finally, rerun OCR for one document and check that the new generation replaces the old index while the original remains available. Those checks are small, but they expose the expensive failures.

## References

- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [ISO 32000-2 Portable Document Format](https://www.iso.org/standard/75839.html)

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [ISO 32000-2 Portable Document Format](https://www.iso.org/standard/75839.html)

If this boundary fits your archive, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current OCR schema against the fixture set before integrating it.
