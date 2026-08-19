---
name: Generate a studio avatar video and collect the result
description: Start an asynchronous Moonscale studio-avatar video render from a script and a voice, then collect the finished video URL by polling on your own correlation id or by receiving the completion callback.
api: openapi/moonscale-openapi-original.json
base_url: https://api-prd.moonscale.com
operations:
  - 'POST /api/studio-avatar/generate-video'
  - 'GET /api/studio-avatar/generated-video/{correlationId}'
generated: '2026-08-14'
method: generated
source: openapi/moonscale-openapi-original.json
---

# Generate a studio avatar video

Both operations below are exactly as published in the Moonscale OpenAPI. The spec declares
no operationIds, so refer to them by method and path.

## Before you start

- **Base URL is `https://api-prd.moonscale.com`.** The spec's `servers[]` says
  `http://sandbox.mintlify.com` — a documentation-template default. Ignore it.
  `https://api-prd.vidlab7.com` is the pre-rename host and still serves the same operations,
  but new work should target the moonscale.com host.
- **Auth:** `x-api-key: <key>` on every request. This surface declares the API key at both
  the root and the operation level.
- **This surface is unversioned.** These paths have no `/v1/` segment, unlike the
  live-avatar operations. There is no published versioning or deprecation policy for them.

## Step 1 — start the render

`POST /api/studio-avatar/generate-video`

Four fields are required: `avatarId`, `script`, `voiceId`, **and `correlationId`**.

```bash
curl --location --request POST 'https://api-prd.moonscale.com/api/studio-avatar/generate-video' \
  --header 'Content-Type: application/json' \
  --header 'x-api-key: <YOUR_API_KEY>' \
  --data-raw '{
    "avatarId": "<studio-avatar-id>",
    "script": "Hi Dana — here is the two minute walkthrough you asked for.",
    "voiceId": "<voice-id>",
    "correlationId": "<your-own-unique-reference>",
    "webhookUrl": "https://your-app.example.com/hooks/moonscale",
    "metadata": { "campaign": "q3-outbound" }
  }'
```

**`correlationId` is yours to mint and it is how you get the video back.** Generate one
unique value per render — a UUID or your own job id — and persist it before you send the
request. It is the only key the retrieval operation accepts.

`metadata` is a free-form object for your own attributes and is echoed nowhere in the
documented response, so do not depend on reading it back.

On `201` you get `{ id, correlationId, status }` with `status: "PENDING"`. `400` means a
required field is missing or invalid.

## Step 2 — collect the result

Two paths, and you can use both.

**Poll:**

`GET /api/studio-avatar/generated-video/{correlationId}`

```bash
curl --location --request GET 'https://api-prd.moonscale.com/api/studio-avatar/generated-video/<your-correlation-id>' \
  --header 'x-api-key: <YOUR_API_KEY>'
```

Returns `id`, `correlationId`, `status`, `videoUrl`, `avatarId`, `voiceId`, `script`.
`status` moves through `PENDING` → `PROCESSING` → `COMPLETED` or `FAILED`. Read `videoUrl`
only once `status` is `COMPLETED`. `404` means the correlation id does not match any
request.

No polling interval or expected duration is published. The one published reference point is
third-party: the Activepieces connector polls every 5 seconds and gives up after 300
seconds. Use that as a starting shape, not as a guarantee.

**Or receive the callback:**

Set `webhookUrl` on the request and Moonscale POSTs you the **same payload** the polling
operation returns — the OpenAPI says so explicitly ("Get generated video (same as webhook
payload) by correlation ID"). Match it to your job on `correlationId`.

## Rules

- **Verify nothing about the callback — because you cannot.** No signing secret, HMAC
  header, timestamp, retry policy, delivery guarantee, expected acknowledgement, or source
  IP range is published for this webhook. Treat an inbound callback as an untrusted hint:
  when one arrives, call `GET /api/studio-avatar/generated-video/{correlationId}` yourself
  with your API key and act on THAT response. Never act on the callback body alone.
- **Not idempotent.** `correlationId` is a retrieval handle, not a de-duplication key.
  Nothing states that replaying a POST with the same `correlationId` returns the original
  render instead of starting a second one. A render consumes generation capacity, so send
  it once, record it, and check status before ever re-sending.
- **`FAILED` is not an HTTP error.** A failed render returns `200` with `status: "FAILED"`
  and no reason field. Check the status value on every poll — do not treat `2xx` as success.
  The cause of a failure is not retrievable through the API; escalate to
  `support@moonscale.com` with the `x-request-id` you captured.
- **No rate limits are published** and no `RateLimit-*` or `Retry-After` header is returned.
  Throttle yourself.
- **Errors are a flat `{"message": "..."}` object**, not RFC 9457. Branch on HTTP status.
- **`videoUrl` is a delivery URL.** No expiry or access-control behaviour is documented for
  it, so do not assume it is either permanent or private — fetch and store the asset
  yourself if you need durability, and do not publish the URL where you would not publish
  the video.
