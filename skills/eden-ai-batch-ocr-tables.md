---
name: eden-ai-batch-ocr-tables
description: >-
  Extract tables from a batch of documents with Eden AI's asynchronous Universal AI endpoint —
  upload each file once, submit one async job per document, then either poll for the result or
  receive a signed webhook. Use when the user has PDFs, scans or images with tabular data and wants
  structured rows out, or names Eden AI for OCR, table extraction, invoice or document parsing at
  volume.
api: Eden AI API V3
base_url: https://api.edenai.run
eu_base_url: https://api.eu.edenai.run
operations:
  - upload_file_v3_upload_post
  - create_async_job_v3_universal_ai_async_post
  - get_async_job_v3_universal_ai_async__job_id__get
  - list_async_jobs_v3_universal_ai_async_get
  - delete_async_job_v3_universal_ai_async__job_id__delete
x-provenance:
  generated: '2026-09-06'
  method: generated
  source: >-
    openapi/_original/eden-ai-v3-openapi.json (every operationId above was grepped out of the
    harvested first-party spec) and https://www.edenai.co/docs/v3/quickstart/batch-ocr-tables
  note: >-
    Eden AI publishes its own general-purpose skill at skills/eden-ai-edenai.md. This one is
    narrower: it packages the single async batch flow together with the runtime rules recorded in
    conventions/, errors/ and rate-limits/ in this repository.
---

# Batch table extraction on Eden AI

## Auth

One bearer key on every call. Use `https://api.eu.edenai.run` instead of `https://api.edenai.run`
when the workload needs EU-only processing — same key, same shapes, only the host changes.

```
Authorization: Bearer $EDENAI_API_KEY
Content-Type: application/json
```

Use a `sandbox_api_token` while wiring this up: identical endpoints, mock responses that match the
real structure, no provider call and no charge. Swap to an `api_token` only when the shape is right.

## 1. Upload each document once — `upload_file_v3_upload_post`

`POST /v3/upload` (multipart). Returns a `file_id`. Upload once and reuse the id across jobs; do not
re-upload the same document for a retry.

Already have a public URL? Skip this step — file parameters accept a URL or a `file_id`.

## 2. Submit one async job per document — `create_async_job_v3_universal_ai_async_post`

`POST /v3/universal-ai/async`, which answers **202**, not 200.

```json
{
  "model": "ocr/ocr_tables_async/amazon",
  "input": { "file": "<file_id or public URL>" },
  "fallbacks": ["ocr/ocr_tables_async/google"],
  "webhook_receiver": "https://your-service.example/webhooks/edenai",
  "user_webhook_parameters": { "document": "invoice-4471" }
}
```

- `model` is `feature/subfeature/provider`. Call `GET /v3/info/ocr` before you hardcode a provider —
  the catalog is the only source of valid names, and there is no structural validation of the string.
- `fallbacks` gives you a second provider automatically when the first fails. Set 1-3 in production.
- `user_webhook_parameters` is echoed back verbatim as `user_parameters`; put your own record id
  there so you can correlate the callback.

**There is no idempotency key.** Nothing in this request lets Eden AI recognise a retry, and a
resubmitted job is a second billed inference. Record the returned job id before you retry anything,
and treat a timed-out submission as *possibly succeeded*.

## 3a. Poll — `get_async_job_v3_universal_ai_async__job_id__get`

`GET /v3/universal-ai/async/{job_id}` until `status` is `success` or `fail`. Back off; the account
rate limit is 10 requests/second shared across every key, and **no rate-limit headers are returned**,
so you cannot read a remaining budget or a reset time. `list_async_jobs_v3_universal_ai_async_get`
(`GET /v3/universal-ai/async`) lists your jobs if you lose an id.

## 3b. Or take the webhook

On completion Eden AI POSTs a signed `async_job_completed` payload to `webhook_receiver`:

```
User-Agent: EdenAI/Ai-Features
X-Edenai-Webhook: true
X-Edenai-Signature: <hex>
X-Edenai-Hash-Algorithm: SHA256
```

Verify it: canonical-JSON the body (sorted keys, 2-space indent, UTF-8), SHA-256 it, take the **hex
digest**, and check the RSA PKCS1 v1.5 signature over that digest. Do not re-serialize with default
formatting. The public key is not published — request `webhook_rsa.pub.pem` from Eden AI support
before you go live, because there is no endpoint to fetch it from.

## 4. Check for failure *inside* success

Two rules, both of which will bite an agent that only looks at status codes:

1. **An error can arrive in a 2xx.** When the primary model and every fallback fail, the failure is
   reported in the response body. Inspect `error` on every response, whatever the status.
2. **The success value is misspelled in the contract.** The v2 response schemas declare
   `StatusEnum: ["sucess", "fail"]`. Match defensively — accept both spellings — rather than testing
   for `"success"`.

Otherwise: `422` is a validation error listing the offending field in `detail[].loc`; `402` means the
credit balance is empty; `429` and `5xx` are retriable with backoff; `4xx` auth errors are not.

## 5. Clean up

Results are retained for **7 days** and then deleted permanently — pull what you need inside that
window. `delete_async_job_v3_universal_ai_async__job_id__delete`
(`DELETE /v3/universal-ai/async/{job_id}`) removes a job record; Eden AI does not state whether that
stops in-flight provider work or refunds its cost, so do not rely on it as a cancel. Files are
removed with `POST /v3/upload/delete` (by ids) or `DELETE /v3/upload` (everything).

## Cost

Every response carries a `cost` field in USD. Surface it — cost comparison across providers is half
the reason to be on this gateway.
