# Finite Feed → Your Service: API Endpoint Contract

What Muse's daily collection job sends to your service, and how. Written for whoever builds the receiving endpoint.

## The short version

One HTTPS `POST` per Instagram post, JSON body, `Content-Type: application/json`.
Text and JSON only — **no photos are ever sent**. Each post's images are converted
to a structured text description before delivery.

## Delivery mechanics

- **Method:** `POST` with a single JSON object as the body (UTF-8). One request
  per post, not batched. A typical run delivers ~5 posts (today's run: 5).
- **Cadence:** once daily at a confirmed time (TBD). Each run sends only posts
  newer than the previous run's watermark — you never get the same post twice
  unless a delivery retry fires.
- **Order:** newest-first (feed order).
- **Success:** any `2xx` status counts as delivered.
- **Retries:** 3 attempts with exponential backoff, 30s timeout per attempt. If
  all attempts fail, the payload stays staged on our side and is retried on the
  next run.
- **Idempotency key: `post_id`** (Instagram media ID, string). Safe to upsert on
  it — a redelivery after a retry is the same object.
- **Auth:** your choice. Either no auth, or a custom header — tell us the header
  name and secret and we'll send it. The secret lives in our Secure Vault; it is
  never stored in code or config files.
- **Volume/cost context:** ~5 small JSON payloads per day (each a few KB).
  Trivial bandwidth.

## Request body schema

| Field | Type | Notes |
|---|---|---|
| `post_id` | string | Instagram media ID. **Idempotency key.** |
| `account` | string | Instagram handle, no `@`. |
| `caption` | string \| null | The post's caption, verbatim. Never rewritten or summarized by us. `null` if the post has none. |
| `post_url` | string | Canonical post URL, e.g. `https://www.instagram.com/p/DeEtJ9AE6QA/`. Use this for official Instagram embeds in your frontend. |
| `created_at` | string | `"YYYY-MM-DD HH:MM:SS"`, UTC, as returned by the Instagram API. |
| `description.images` | array | One entry per **image** slide, in slide order. |
| `description.images[].index` | integer | 1-based slide number. |
| `description.images[].extracted_text` | string | Text visible in the image, transcribed verbatim; `"none"` if the image has no text. |
| `description.images[].description` | string | Plain visual description of what the slide shows. Observable details only — no interpretation, no value judgment. |
| `description.concrete_references` | array | Named things worth acting on or looking up. |
| `description.concrete_references[].type` | enum | `book` \| `place` \| `product` \| `person` \| `recipe` \| `event` \| `tool` \| `other` |
| `description.concrete_references[].name` | string | The specific name. |
| `description.concrete_references[].details` | string | Identifying details (where it appeared, what it is). |
| `description.uncertainties` | string[] | Things the vision pass couldn't confirm, plus a record of any skipped slides and why (e.g. video slides). |

### Rules your endpoint can rely on

- `images` contains **image slides only**. Video slides are excluded by design
  (MVP scope is photos); each skipped slide is noted in `uncertainties` with its
  reason and slide number.
- A post where every slide was skipped (e.g. all-video carousel) is **never
  sent** — you won't receive empty fact sheets.
- There are **no image bytes and no image URLs** in the payload. Instagram's CDN
  URLs are signed and expire within ~15–20 minutes, so they'd be dead before you
  could use them. Render visuals with official embeds via `post_url`.
- `caption` and `extracted_text` preserve calls to action verbatim — there's no
  separate calls-to-action field because it would duplicate them.
- Deliberately absent: engagement metrics (likes/comments), platform labels,
  post-type flags, post-level synthesis, brand reference type. If you need any
  of these added, that's a schema change we can discuss.

## What we need from you to go live

1. **Endpoint URL** (where to POST).
2. **Auth scheme** — none, or a header name + secret.
3. **Schema differences** — field renames, extra required fields, anything your
   framework needs that isn't above.
4. **Confirmed daily run time** (proposed default: 7:00 AM ET, not confirmed).

## Full real example

An actual payload from the 2026-10-04 verification run (`@rosies_place`, 3 slides):

```json
{
  "post_id": "18250562938308833",
  "account": "rosies_place",
  "caption": "1 in 8 women in the U.S. will be diagnosed with breast cancer in their lifetime.* Early detection can make a world of difference in treating cancer. That’s why we are so grateful for our partner @bhchp1985, whose Cancer Screening Patient Navigator regularly visits Rosie’s Place to provide vital screenings to our guests. \n\n*@americancancersociety",
  "post_url": "https://www.instagram.com/p/DeEldkijBiQ/",
  "created_at": "2026-10-04 05:01:19",
  "description": {
    "concrete_references": [
      {
        "details": "cited as the source of the 1-in-8 statistic; website cancer.org shown on slide 2",
        "name": "American Cancer Society",
        "type": "other"
      },
      {
        "details": "partner of Rosie's Place; its Cancer Screening Patient Navigator regularly visits Rosie's Place to provide cancer screenings to guests",
        "name": "BHCHP (@bhchp1985)",
        "type": "other"
      },
      {
        "details": "poster of the content; guests receive cancer screenings from BHCHP's Cancer Screening Patient Navigator",
        "name": "Rosie's Place (@rosies_place)",
        "type": "other"
      },
      {
        "details": "subject of the post; slide 1 title with pink ribbon motif",
        "name": "Breast Cancer Awareness Month",
        "type": "event"
      }
    ],
    "images": [
      {
        "description": "A solid bright-pink graphic with large white serif text reading \"Breast Cancer Awareness Month\"; the letter \"o\" in \"Month\" is replaced by a light-pink awareness ribbon. A small white line drawing of a steaming teacup on a saucer sits at the bottom center.",
        "extracted_text": "Breast Cancer Awareness Month",
        "index": 1
      },
      {
        "description": "A solid bright-pink graphic with white text. Smaller text at top reads \"The American Cancer Society estimates\"; very large bold text reads \"1 in 8\"; below, \"women in the US will be diagnosed with breast cancer in their lifetime.\" Small text at the bottom reads \"cancer.org\".",
        "extracted_text": "The American Cancer Society estimates\n1 in 8\nwomen in the US will be diagnosed with breast cancer in their lifetime.\ncancer.org",
        "index": 2
      },
      {
        "description": "A solid bright-pink graphic with white text. A bold heading reads \"What you can do:\" followed by \"Know your risk factors, schedule your routine exams and talk to your doctor if you have a concern.\"",
        "extracted_text": "What you can do:\nKnow your risk factors, schedule your routine exams and talk to your doctor if you have a concern.",
        "index": 3
      }
    ],
    "uncertainties": [
      "The small steaming-teacup line icon at the bottom of slide 1 appears to be the account's logo mark, but this is not confirmed by any text in the images."
    ]
  }
}
```

## Downstream context (for the service builder)

Your service owns everything after delivery: send each post's `description` text
to Jev (TypeSafe AI System One — text/JSON in, no image support), apply your
preconfigured value criteria and threshold, save passing posts (embeddable
`post_url` + date) to your pass table, and render official Instagram embeds in
a finite scroll ending in a "you're done" state.
