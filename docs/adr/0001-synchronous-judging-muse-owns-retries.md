# Judge synchronously; Muse owns retries

The ingest endpoint judges each Post with Jev inside the request and returns `2xx` only once the Post and its Verdict are stored. Transient failures (Jev 429/529/timeout, database down) return `5xx`; permanent ones return `4xx` (`400` for a malformed payload, `422` when Jev rejects the Post as invalid or too large, with the Post stored for diagnosis). We never return `2xx` for a failure. Muse already retries 3 times and keeps undelivered Posts staged for its next daily run, so that becomes our retry queue and we run no worker, queue, or cron. Muse sends one Post per request and Jev answers in 70–500 ms against Muse's 30 s timeout, so a few hundred Posts per run is a few hundred short requests.

## Considered Options

- **Accept with `202`, judge in the background, expose a status endpoint for Muse to poll.** Rejected: Muse treats any `2xx` as delivered and drops the Post, so retrying a failed judgment would be our job anyway, and polling buys nothing because a redelivery carries no information we didn't already store. It would also require changing every User's Muse.

## Consequences

- Muse's contract must change: retry on `5xx`, `429`, timeouts and connection errors; on `401` keep the Post staged and alert the User (it's usually a rotated secret, fixable without losing Posts); on `400`/`422`, stop retrying. See `docs/muse-integration.md`. A Muse that ignores this just retries harmlessly, since we answer the same `4xx` again.

- A Jev outage delays affected Posts to the next Muse run, so they land in the next Day's Feed.
- A status endpoint, if built, is for people debugging, not for Muse.
