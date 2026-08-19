---
name: Run a live AI sales avatar meeting and collect the transcript
description: Create a Moonscale meeting room hosted by a live AI sales avatar, hand the join URL to a buyer, then retrieve the conversation transcript and AI-generated summary once the call has ended.
api: openapi/moonscale-openapi-original.json
base_url: https://api-prd.moonscale.com
operations:
  - 'POST /api/v1/rooms'
  - 'GET /api/v1/conversations/{conversationShortCode}'
  - 'GET /api/v1/conversations'
generated: '2026-08-14'
method: generated
source: openapi/moonscale-openapi-original.json + https://vidlab7-d7584a5d.mintlify.app/api-reference/introduction
---

# Run a live AI sales avatar meeting

The Moonscale API has no operationIds — every operation below is referenced by method and
path exactly as they appear in the published OpenAPI and reference. Do not invent names.

## Before you start

- **Base URL is `https://api-prd.moonscale.com`.** Do NOT use the `servers[]` value in the
  published spec — it is `http://sandbox.mintlify.com`, a documentation-template default on
  a host Moonscale does not own.
- **Auth:** send `x-api-key: <key>` on every request. Keys are generated in Moonscale Studio
  (app.moonscale.com) under **API**. If the account has no API section, API access is not
  feature-flagged on and you must request it from `support@moonscale.com` — there is no
  self-service path.
- **Get the avatar identifier first.** Open the avatar in Studio; the identifier is the last
  path segment of `https://app.moonscale.com/real-time-avatars/<liveAvatarId>` and is a UUID.
  Confirm the avatar's **Active** toggle is on, or the room will have no host.
- **The avatar's behaviour is not API-configurable.** System prompt, voice, guardrails,
  knowledge base, language, greeting, max call length and contact collection are all set in
  Studio. Configure the avatar fully there before creating rooms.

## Step 1 — create the meeting room

`POST /api/v1/rooms`

The two published sources disagree on the field names, so send what the reference documents
and be prepared for the spec's names:

- The **reference** documents `name`, `liveAvatarId`, `conversationShortCode`,
  `contactEmail`, `privacy`, `enableChat`, `roomSettings`, `context`.
- The **OpenAPI** requires `name` and `liveAvatarShortCode`, and calls the settings object
  `properties`.

Use the reference field names, since that is the surface Moonscale documents for callers.

```bash
curl --location --request POST 'https://api-prd.moonscale.com/api/v1/rooms' \
  --header 'Content-Type: application/json' \
  --header 'x-api-key: <YOUR_API_KEY>' \
  --data-raw '{
    "name": "acme-discovery-call",
    "liveAvatarId": "<live-avatar-uuid>",
    "contactEmail": "buyer@example.com",
    "privacy": "private",
    "enableChat": true,
    "roomSettings": { "shouldBotJoinImmediately": false }
  }'
```

Leave `conversationShortCode` out (or blank) for a first contact. Pass it to continue an
existing thread with the same buyer, so the avatar has the prior conversation as context.

On `201` you get back:

- `url` — the meeting-room URL **with an entry token already attached**. Treat it as a
  credential: anyone holding it can join the room. Deliver it over a channel you trust and
  do not log it.
- `conversationId` (the reference) / `conversationShortCode` (the OpenAPI) — the handle you
  need in step 2. Store whichever key the response actually returns.

`privacy` accepts only `public` or `private`. A room is single-participant.

`404` on this call means the room could not be set up — usually a bad avatar identifier.
Note that Moonscale returns `404`, not `400` or `422`, for a malformed create-room request.

## Step 2 — wait for the call to end

There is no callback for a finished conversation. `webhookUrl` exists only on the
video-generation surface, not here. Poll, and give the call time to complete: the avatar's
maximum call length is a per-avatar Studio setting, and the session ends automatically when
it elapses.

## Step 3 — retrieve the transcript and summary

`GET /api/v1/conversations/{conversationShortCode}`

```bash
curl --location --request GET 'https://api-prd.moonscale.com/api/v1/conversations/<conversation-id>' \
  --header 'x-api-key: <YOUR_API_KEY>'
```

Returns `transcript[]` and `summary`. Handle both published shapes defensively:

- The OpenAPI models each transcript entry with a `content` field and `summary` as an object
  with a `text` property.
- The reference models each entry with a `text` field and `summary` as a plain string.

Read `entry.text ?? entry.content` and `summary.text ?? summary`. A `404` means the
conversation handle is wrong.

## Step 4 — reconcile a batch instead

`GET /api/v1/conversations?dateStart=<unix>&dateEnd=<unix>`

Documented in the reference only — it is **absent from the OpenAPI**, so no generated client
will have it.

- `dateStart` and `dateEnd` are Unix timestamps in seconds and are both required.
- **The window must not exceed 24 hours** and `dateEnd` must be strictly after `dateStart`.
  Violating either returns `400`.
- Optional `liveAvatarId` filters to one avatar.
- There is **no pagination** — no cursor, page or limit. To cover a longer period, walk it
  in windows of 24 hours or less.

Each conversation carries `utm_id` — the Contact ID passed through the conversation URL.
That is your join key back to your own CRM record; there is no other correlation field.

## Rules

- **Not idempotent.** No idempotency key is supported anywhere in this API. Retrying
  `POST /api/v1/rooms` creates another room. Make the call once, persist the returned
  handle, and retry only after confirming the first attempt did not land.
- **No rate limits are published** and no `RateLimit-*` or `Retry-After` header is returned.
  Self-throttle conservatively and back off on any non-2xx.
- **Errors are a flat `{"message": "..."}` object**, not RFC 9457. There is no error code to
  branch on — branch on HTTP status. `401` is a missing or invalid key; a bare
  `403 {"message":"Forbidden"}` comes from the gateway and does not distinguish "no key"
  from "key not allowed here".
- **Capture `x-request-id`** from every response. It is undocumented but returned on all of
  them, and it is the only handle support can correlate against.
- **Transcripts are personal data.** They contain participant speech and, where contact
  collection is on, name and email. Studio has a transcript anonymization toggle — check
  whether it is on before moving transcripts into another system, and honour whatever your
  own retention policy requires.
