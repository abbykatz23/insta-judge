# One Jev noul per Interest, passing on any-of

Each Post is judged in a single Jev request: the Post is the `state`, and every one of the User's Interests is its own `noul` question with the Interest's "counts" and "doesn't count" text as its `true`/`false` criteria. A Post passes if any Interest's noul is at or above the Threshold (0.8 to start, global, hidden from Users); every Interest at or above it is a Matched Interest. Interests are unrelated to each other (vegan recipes, book lists, transit events), and a Post should pass by fully matching one of them, which TypeSafe's docs say needs separate conditions rather than a combined question or weighted score. See `docs/research/jev-judging-practices.md`.

## Considered Options

- **One combined "everything I value" question per User.** Rejected: the docs call hiding several judgments in one question an anti-pattern, and a Post that strongly matches one Interest and none of the others "can't be placed".
- **Weighted composite across Interests.** Rejected: a perfect vegan recipe would be dragged down by scoring zero on transit.
- **One `choice` over Interests plus "none".** Rejected: probabilities sum to 1, so a Post matching two Interests looks uncertain, "none" competes with real options, and jev-1.13 is biased toward the first option.
- **A 3-level `score` per Interest with a 1.5 cut.** Rejected: its middle level ("on topic but excluded, or loosely related") is itself a double condition, the cut is no less arbitrary than a noul threshold, and three levels are harder for Users to write than counts/doesn't-count.

## Consequences

- The Threshold must be tuned from stored Verdicts; pin the Jev model version (not `jev-latest`) so tuning stays valid, and record the returned model on each Verdict.
