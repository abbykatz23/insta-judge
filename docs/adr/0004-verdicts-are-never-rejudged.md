# Verdicts are never re-judged

Once a Post has a Verdict it is final. Editing, adding, or deleting an Interest applies only to Posts delivered afterward; a redelivered Post that already has a Verdict gets `200` without calling Jev; and each Verdict keeps a snapshot of the Interests as they were when it was judged, so past Days (and their Matched Interest labels) show exactly what the User saw at the time. Re-judging would let Passes appear or vanish in Feeds the User has already read, and Jev is not guaranteed to be deterministic, so even an identical re-judge can flip a result. Previewing a draft Interest against recent Posts runs in memory and is never stored.

## Consequences

- One exception to "past Days show what the User saw": when a Feed is opened, Passes whose embeddability hasn't been checked in 24 hours are checked against Meta's oEmbed endpoint (in parallel, capped at ~1.5 s, failing open on errors or rate limits), and any that Meta says can't be embedded are hidden. The Verdict itself is unchanged. See `docs/research/instagram-embed-availability.md`.
