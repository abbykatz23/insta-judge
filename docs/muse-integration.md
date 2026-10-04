# insta-judge ← Muse: integration notes

This is insta-judge's reply to Muse's handoff, [Finite Feed → Your Service: API Endpoint Contract](../api-endpoint-contract.md). It tells whoever runs a Muse what to configure and how our endpoint behaves. **The request body schema is unchanged.** What does change: insta-judge serves many people, each running their own Muse, and our endpoint gives specific meanings to its status codes, which Muse's retry logic needs to follow.

## The short version

1. Send the same JSON body you already send, one Post per `POST`.
2. Add one header: `X-Muse-Secret: <secret>`. Each person gets their own secret from their insta-judge settings.
3. Retry on `5xx`, `429`, `401`, timeouts, and connection errors. On `400` or `422`, stop retrying and set the Post aside for a human to look at.

## What we need you to configure

| Setting | Value |
|---|---|
| Endpoint URL | `https://<TBD once deployed>/api/ingest`. The same URL for every person. |
| Auth header name | `X-Muse-Secret` |
| Auth header value | That person's secret. They generate it in insta-judge's settings, and it's shown only once. Store it in your Secure Vault. |
| Daily run time | Each person can choose their own. We recommend the morning in their timezone. See "Timing" below. |

The secret is how we know whose Post this is. If one Muse instance serves several people, it must send each person's Posts with **that person's** secret.

## Request

There are **no schema differences** from your contract. A few clarifications:

- **Fields we require** (a missing field or a wrong type returns `400`): `post_id`, `account`, `post_url`, `created_at`, `description.images` (non-empty), `description.concrete_references`, `description.uncertainties`. `caption` may be `null`.
- **Extra fields are ignored**, so adding a field never breaks delivery. If you add one you'd like us to use, let us know.
- **An unrecognized `concrete_references[].type` is accepted** and treated as `other`. Adding a new reference type isn't a breaking change.
- **We never use images, image URLs, or `uncertainties`** to judge a Post. We do use the caption, each slide's `extracted_text` and `description`, and `concrete_references`, so their quality directly affects the results.

## Response codes and what Muse should do

We judge each Post **before** responding. A `2xx` means the Post is judged and stored, and only then is it safe to drop it from staging. We never return `2xx` for a failure.

| Status | Meaning | What Muse should do |
|---|---|---|
| `200` / `201` | Judged and stored, or already judged earlier (a redelivery). | Delivered. Drop it from staging. |
| `401` | Secret missing or wrong. Usually the person rotated their secret and Muse still has the old one. | **Keep the Post staged** and alert the person. It will succeed once the secret is updated. Don't drop it. |
| `400` | Malformed payload: a missing field or wrong type. The response body says which. | Don't retry, because it will never succeed. Move it to a dead-letter list and log the body. This is a Muse-side bug. |
| `422` | Valid payload, but the Post can't be judged (for example, it's too large for our judging model). We've stored it on our side. | Don't retry. Drop it from staging, or dead-letter it if you want a record. |
| `429` | We're being rate-limited by our judging provider. | Retry with backoff. Honor `Retry-After` if we send it. |
| `5xx` | A temporary failure on our side. | Retry with backoff (your existing 3 attempts are fine), then keep it staged for the next run. |
| Timeout / connection error | — | Same as `5xx`. |

Error responses have a JSON body, `{"error": "<code>", "message": "<human-readable detail>"}`. Please log it.

This changes one thing in your contract. Today Muse treats every non-`2xx` the same way and keeps the Post staged indefinitely. The table above makes `400` and `422` permanent, so they stop retrying. If a Muse keeps retrying them anyway, nothing breaks. We'll just answer with the same status each time.

## Idempotency

`post_id` remains the idempotency key, **scoped to the person**. Redelivering a Post you've already sent returns `200` without judging it again, so redelivering is always safe. Two people's Muses can send the same `post_id` (they follow the same account), and each gets its own result.

## Timing

- **Latency:** each request typically takes well under a second and usually under 2 s. Your 30 s timeout is plenty.
- **Concurrency:** sending a run's Posts one after another is fine. If you parallelize, please limit it to about 5 concurrent requests per Muse.
- **Which Day a Post shows up on:** a Post appears in the person's Feed for the day **we received it**, in their timezone, not the day it was posted on Instagram. So:
  - Run once a day, ideally in the person's morning, and finish the run within a few minutes. A run that crosses midnight in the person's timezone gets split across two Days.
  - A Post you redeliver after an outage shows up in the Day it finally arrives. That's expected.
- **Order:** newest-first, as in your contract, is fine. We sort on our side.

## Onboarding checklist (per person)

1. The person signs in to insta-judge and creates at least one Interest.
2. They copy the endpoint URL, header name, and secret from settings.
3. They configure their Muse with those values.
4. They run Muse once, or wait for the scheduled run, then check that today's Feed in insta-judge shows Posts. A Day where nothing passed still appears in the Day picker, which confirms delivery worked.

## Things that are deliberately unchanged

- One Post per request, no batching.
- No image bytes or URLs.
- No status or polling endpoint. The response to the `POST` is final.
