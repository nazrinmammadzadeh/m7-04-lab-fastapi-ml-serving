# DECISIONS.md

## Lint command

```bash
npx @redocly/cli lint openapi.yaml
```

Or using the web validator: paste `openapi.yaml` into https://editor.swagger.io — the status bar must read "No Errors."

---

## 1. Versioning — path-based (`/v1/`) over header-based

Path-based versioning is visible in server access logs, reverse-proxy routing rules, and API gateway configuration without any header-parsing logic. Clients also don't need to set a special header on every request; the version is part of the resource URL and follows REST conventions, making it obvious in documentation, curl examples, and browser-based testing.

---

## 2. Batch ordering and partial failures

**Behavior:** `POST /v1/predict-batch` always returns HTTP `200` when the request itself is structurally valid (correct JSON, 1–32 items, unique IDs, no total body size violation). Each item in the `results` map carries an explicit `status` field — either `"success"` (with labels) or `"error"` (with an `Error` object). Callers must check per-item `status`, not the HTTP status code, to determine whether each image was processed.

**Rationale:** Returning HTTP `207 Multi-Status` would be correct RFC-wise but is rarely supported by HTTP client libraries and causes confusion in monitoring dashboards that aggregate by status code. HTTP `200` with per-item errors is simpler to consume and is consistent with how major APIs (Stripe, Twilio) handle batch partial failures. The `status` discriminator makes it impossible to silently ignore an error — callers must read the field.

**What triggers HTTP 413 instead of a per-item error:** only when the entire request body exceeds 160 MB (32 × 5 MB). An individual image over 5 MB inside an otherwise valid batch is returned as a per-item error with `code: "PAYLOAD_TOO_LARGE"`, so the other 31 images are not penalised.

---

## 3. Async lifecycle and retention

**States:** `queued → processing → completed | failed`. A job enters `queued` immediately on HTTP 202. The worker picks it up and transitions to `processing`; once inference finishes it becomes `completed` (with labels) or `failed` (with an `Error` object). There are no intermediate states and no cancellation in v1.

**Retention:** results are retained for **24 hours** after the job reaches a terminal state (`completed` or `failed`). The `expires_at` field in the `AsyncResult` response tells the caller exactly when the result will be deleted. After expiry, `GET /v1/predictions/{job_id}` returns HTTP `404`. Callers should store results they need beyond 24 hours in their own storage.

**Polling:** there is no webhook or push mechanism in v1; callers must poll. A recommended polling interval is 1 s for the first 5 s, then exponential back-off up to 10 s, for a maximum of 10 minutes before treating the job as timed out on the client side.

---

## Note on `image_data` placeholders in examples

The `image_data` values in `examples/` are truncated base64 strings, not full images. Each placeholder is documented with a `_note` field explaining the assumed original file size and format. Real API requests must supply a complete, valid base64-encoded image. The truncation is intentional for readability — the spec's `format: byte` constraint and the 5 MB size limit still apply to any real implementation.
