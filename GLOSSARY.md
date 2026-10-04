# insta-judge

Receives Instagram posts from each user's Muse, judges them against that user's Interests, and serves each user a finite feed of only the posts worth their time.

## Language

### Intake

**User**:
A person with their own Muse, Interests, and Feeds.
_Avoid_: Account (collides with the Instagram `account` handle), customer

**Muse**:
A user's own collection job that delivers their Instagram posts to us. Anything that happens before delivery is Muse's concern.
_Avoid_: Collector, scraper

**Post**:
One Instagram post as delivered by a Muse, belonging to exactly one User.
_Avoid_: Item, payload, media

**Fact Sheet**:
The structured text stand-in for a Post's visuals: per-slide transcriptions and descriptions, concrete references, and uncertainties.
_Avoid_: Description (reserved for one slide's visual description)

### Judging

**Interest**:
One kind of Post a User finds valuable, including its own exclusions (e.g. "vegan recipes", "upcoming Boston public-transit events"). A User has several, and they are independent of each other.
_Avoid_: Criterion (clashes with Jev's level `criteria`), category, value prompt, topic

**Verdict**:
The outcome of judging one Post: how well it matches each of the User's Interests, and whether it passed.
_Avoid_: Judgment, result, rating

**Threshold**:
How strongly a Post must match an Interest to pass.
_Avoid_: Cutoff

**Pass**:
A Post that fully matches at least one of its User's Interests. Partial matches across several Interests never add up to a Pass.
_Avoid_: Pass table, approved post, hit

**Matched Interest**:
The Interest a Pass is shown under in the Feed.
_Avoid_: Tag, label, reason

### Reading

**Day**:
The calendar date, in the User's timezone, on which a Post was delivered to us. A Day holds roughly the last 24 hours of Posts as of that day's Muse run. Every Post belongs to exactly one Day.
_Avoid_: Run, batch, publish date

**Feed**:
A User's finite, ordered list of the Passes from one Day, ending in a "done" state. Days never blend into a single Feed.
_Avoid_: Timeline, scroll
