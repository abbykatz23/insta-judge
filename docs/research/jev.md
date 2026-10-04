# Jev (TypeSafe AI): research notes

_Researched 2026-10-04. Everything here comes from primary sources: TypeSafe's own site, docs, and GitHub, plus Vercel's changelog for Vercel's own gateway integration. Lines marked **Inference** are my reading of those sources, not something a source says._

## Summary

"Jev" is positively identified. It is the flagship model from **TypeSafe AI**, a San Francisco company, and is currently in **early access** ([typesafe.ai](https://typesafe.ai), [docs intro](https://docs.typesafe.ai/introduction)). Jev is not a chat LLM. TypeSafe calls it a "System One" decision model: you send it a `state` (a string, JSON object, or array) and a set of named, typed questions (`score`, `choice`, `noul` = yes/no). It sends back typed answers with probabilities and a confidence value. It does **not** produce reasoning text ([API reference](https://docs.typesafe.ai/api), [System One](https://docs.typesafe.ai/concepts/system-one)). Scoring against user-defined criteria fits the `score` question type: you give 2–10 ordered level descriptions, and you get back a fractional score from 0 to the top level index, plus the probability of each level. The API is a single synchronous endpoint, `POST https://api.typesafe.ai/v1/systemone`, authenticated with a Bearer key. There are official Python and JS/TS SDKs. The price is $0.042 per million input tokens, and output is free. No free tier is documented.

## 1. What Jev is and who makes it

- Jev is "TypeSafe's flagship model and the first System One model" ([llms.txt index](https://docs.typesafe.ai/llms.txt), [introduction](https://docs.typesafe.ai/introduction)).
- TypeSafe describes itself as "an AI lab building machine-native intelligence infrastructure for automation, designed to make decisions within software." Jev is offered in early access, and you sign up through the console ([typesafe.ai](https://typesafe.ai)).
- The launch post is written by Diogo Almeida, founder. It says Jev is "available today in early access" and that developers are being brought "off the waitlist as quickly as we can" ([launch blog](https://typesafe.ai/blog/introducing-system-one-models-and-jev)).
- The founders are Diogo Almeida (CEO), Sasha Sheng (COO), and Erik Gafni (CTO), based in San Francisco ([team](https://typesafe.ai/team)).
- System One models "do not write replies, produce code, or generate explanations of their reasoning." Input is text only (strings, JSON objects, arrays of text). Images, audio, and video are not supported "(yet)" ([System One](https://docs.typesafe.ai/concepts/system-one)).
- Jev "does not generate text, write code, or hold a conversation." TypeSafe positions it for "routing, classification, scoring, guardrails, or any structured decision" ([coding agents](https://docs.typesafe.ai/introduction/coding-agents)).
- The official GitHub org is `typesafe-ai`, created 2024-05-28. It hosts the official SDKs `typesafe-sdk-python` and `typesafe-sdk-js`, agent `skills`, `system-one-adapter-python` ("Drop-in TypeSafeClient replacement backed by LLM APIs"), and an n8n node ([github.com/typesafe-ai](https://github.com/typesafe-ai), checked via GitHub API).
- Jev is also available through Vercel AI Gateway as model `typesafe-ai/jev`, called via the AI SDK's `experimental_evaluate`. Vercel's changelog calls the yes/no type `boolean` (TypeSafe calls it `noul`) ([Vercel changelog](https://vercel.com/changelog/typesafe-ai-jev-now-available-on-ai-gateway)).

**Ambiguity check.** Searches for "Jev", "jev.ai", "Jev AI", and "Jev LLM judge" all pointed to this TypeSafe product, and I found no competing AI scoring product called Jev. Several lookalike domains turned up in search: `jevai.org`, `jevapi.io`, `jev-ai.pro`, and `jevtypesafeai.com`. None of them is linked from typesafe.ai or docs.typesafe.ai, so I treated them as unofficial and did not use them. The official domains are `typesafe.ai`, `docs.typesafe.ai`, `console.typesafe.ai`, and `api.typesafe.ai`.

## 2. Defining criteria and the score format

**Question types** ([API reference](https://docs.typesafe.ai/api), [primitives](https://docs.typesafe.ai/primitives)):

| Type | `criteria` | Answer |
|---|---|---|
| `score` | **Required.** An array of 2–10 level descriptions (strings or objects), ordered lowest to highest | `score` (number), `legend`, `probabilities`, `confidence` |
| `choice` | **Required.** A map of option name to description (max 255 options) | `choice` (string), `probabilities`, `confidence` |
| `noul` | Optional. An object with `"true"` and `"false"` descriptions | `noul`: a probability from 0 to 1. No separate confidence |

Each question also takes `instructions`, which can be a string, an object, or an array ([API reference](https://docs.typesafe.ai/api)).

**Score semantics** ([Score](https://docs.typesafe.ai/primitives/score)):
- The score is the "probability-weighted mean of the level numbers", so it runs from 0 to (number of levels − 1) and can land between levels. Example from the docs: `0×0.0 + 1×0.57 + 2×0.43 = 1.43`.
- `legend` maps each level index (as a string key) back to its description. `probabilities` maps each level index to a probability, and they sum to 1.
- `confidence` is "a number from 0 to 1 computed from how probabilities is spread". For score questions it is `1 − (probability-weighted distance from peak) / (even-spread average distance)`, floored at 0. For choice questions it is `(p_max − 1/n)/(1 − 1/n)` ([Confidence](https://docs.typesafe.ai/confidence)).
- Criteria can be plain strings ("Start with strings") or objects with consistent fields such as `{"what": ..., "examples": [...]}` when a level needs examples. "Every level is evaluated separately. The model doesn't see a level's number or its neighbours." The docs advise you to "Describe situations, not degrees" and to avoid levels written only as numbers ([Score](https://docs.typesafe.ai/primitives/score)).
- **Multiple criteria:** the documented pattern is one `score` question per dimension, with the scores normalized and weighted in your own code ([Composite scoring](https://docs.typesafe.ai/patterns/composite-scoring)). All questions in a request are evaluated in parallel: "Adding questions barely changes the response time", and extra questions cost only their tokens ([primitives](https://docs.typesafe.ai/primitives)).
- **No reasoning text** comes back ([System One](https://docs.typesafe.ai/concepts/system-one)).

**Inference:** a "per-criterion breakdown" maps naturally onto one `score` question per user-defined criterion. The answers come back keyed by question id. Any overall score would have to be computed by the app, because Jev does not combine scores itself.

## 3. API

**Endpoint and auth** ([API reference](https://docs.typesafe.ai/api), [Models](https://docs.typesafe.ai/models)):
- `POST https://api.typesafe.ai/v1/systemone`. This is the only endpoint.
- Headers: `Authorization: Bearer <API_KEY>`, `Content-Type: application/json`.
- API keys are issued in the console at https://console.typesafe.ai/keys. The conventional environment variable is `TYPESAFE_API_KEY` ([quickstart](https://docs.typesafe.ai/introduction/quickstart)). Access requires early-access approval ([launch blog](https://typesafe.ai/blog/introducing-system-one-models-and-jev)).

**Request body:** `state` (string | object | array, required), `model` (string, required, e.g. `"jev-latest"`), and `questions` (a map from question id to Question, required) ([API reference](https://docs.typesafe.ai/api)).

**Response body:** `model` (the resolved version, e.g. `"jev-1.13.0"`), `answers` (keyed by question id), and `usage` (`input_tokens`, `output_tokens`) ([API reference](https://docs.typesafe.ai/api)).

**Example from the docs** ([API reference](https://docs.typesafe.ai/api)):

```json
{
  "state": "Help! My payouts have been failing for 3 days.",
  "model": "jev-latest",
  "questions": {
    "frustration": {
      "type": "score",
      "instructions": "How frustrated is the customer?",
      "criteria": ["Calm", "Frustrated", "Very angry"]
    }
  }
}
```

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "frustration": {
      "type": "score",
      "score": 1.05,
      "legend": { "0": "Calm", "1": "Frustrated", "2": "Very angry" },
      "probabilities": { "0": 0.0, "1": 0.95, "2": 0.05 },
      "confidence": 0.92
    }
  },
  "usage": { "input_tokens": 304, "output_tokens": 18 }
}
```

**Illustrative request with JSON state (my construction, not from the docs).** It follows the documented schema: `state` may be a JSON object ([State](https://docs.typesafe.ai/concepts/state)), and there is one score question per criterion.

```json
{
  "model": "jev-latest",
  "state": { "post": { "caption": "…", "author": "…", "likes": 1200 } },
  "questions": {
    "caption_quality": {
      "type": "score",
      "instructions": "How well-written is the caption?",
      "criteria": ["Unreadable or empty", "Readable but generic", "Clear and engaging"]
    }
  }
}
```

**Errors:** 401 means an invalid or missing key, 422 means validation failed, 429 means the rate limit was hit, and 529 means the service is overloaded. For 429 and 529, the docs say to "retry with exponential backoff" ([API reference](https://docs.typesafe.ai/api)).

**SDKs** ([Client SDKs](https://docs.typesafe.ai/sdk)):
- **Python:** `typesafe-sdk` (`pip install typesafe-sdk`, with an optional `[http2]` extra). It has both a sync `TypeSafeClient` and an async `AsyncTypeSafeClient` with `system_one(state=..., questions=...)` ([Python SDK](https://docs.typesafe.ai/sdk/python)). The quickstart lists Python >= 3.10 ([quickstart](https://docs.typesafe.ai/introduction/quickstart)). Repo: https://github.com/typesafe-ai/typesafe-sdk-python (latest release v0.7.2, 2026-09-26, per GitHub API).
- **JavaScript/TypeScript:** `@typesafe-ai/sdk` (`npm install @typesafe-ai/sdk`). It requires Node.js 20+, ships ESM, CJS, and type declarations, and offers `client.systemOne({state, questions})` with helper functions `score()`, `choice()`, and `noul()`. "Answer types are inferred from your questions" ([JS SDK](https://docs.typesafe.ai/sdk/javascript)). Repo: https://github.com/typesafe-ai/typesafe-sdk-js (latest release v0.6.0, 2026-09-15).
- No other languages are listed. Anything else calls the HTTP API directly.
- Third-party routes exist, such as Vercel AI Gateway (`typesafe-ai/jev`) ([Vercel changelog](https://vercel.com/changelog/typesafe-ai-jev-now-available-on-ai-gateway)). Search results also mention LiteLLM, OpenRouter, Cloudflare, and Pydantic AI integrations, which I did not verify.

## 4. Limits

From [Models](https://docs.typesafe.ai/models) (current model Jev 1.13, `jev-1.13.0`; the aliases `jev-latest` and `jev-preview` both resolve to it):
- **Rate limits:** 100K tokens/second and 80 requests/second. The page notes these are "adjusting dynamically" during high demand.
- **Max input:** a 64k-token context per request in total, of which 32k is for "state plus longest question".
- **Input type:** text only, meaning a string, a JSON object, or an array of text values. English gives the best accuracy. Other languages are supported but less reliable.
- **Fine-tuning:** no per-account fine-tuning. Customization happens through `state`, `instructions`, and `criteria`.
- **Training on your data:** "Jev is not trained on customer requests or responses."
- **Latency:** the docs pages give no figures. The launch blog states an "end-to-end response time is 70ms-500ms" ([launch blog](https://typesafe.ai/blog/introducing-system-one-models-and-jev)).
- **Determinism and consistency:** no docs page says whether identical inputs give identical outputs, and no temperature or sampling parameter is documented ([API reference](https://docs.typesafe.ai/api), [Confidence](https://docs.typesafe.ai/confidence)). The launch blog claims Jev is "calibrated: higher confidence means higher accuracy" ([launch blog](https://typesafe.ai/blog/introducing-system-one-models-and-jev)). The Score docs warn that "Higher confidence does not establish which answer is correct" ([Score](https://docs.typesafe.ai/primitives/score)). Third-party write-ups (MLflow, LangChain, Langfuse) report low run-to-run score variance, but they are not primary sources and are not relied on here.
- **Known weaknesses of jev-1.13** ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)):
  - It reads questions literally.
  - It is poor at math, counting, and numbers ("not a calculator").
  - It handles dates as text.
  - It struggles with double negatives and indirection.
  - Accuracy drops when the state contains a lot of irrelevant content.
  - It does not treat data as hostile, so it is susceptible to injected instructions.
  - Contradictory instructions and criteria confuse it.
  - In choice questions it is biased toward the first option.
  - It is not built for text generation.
- **Max questions per request:** not documented ([primitives](https://docs.typesafe.ai/primitives)).

## 5. Pricing

- Input costs $0.042 per million tokens ($42 per billion). Output is free ("too cheap to meter") ([Models](https://docs.typesafe.ai/models), [typesafe.ai](https://typesafe.ai), [launch blog](https://typesafe.ai/blog/introducing-system-one-models-and-jev)).
- **Free tier:** none documented. No free credits are mentioned on the homepage, in the docs, or in the launch blog. `typesafe.ai/pricing` returns 404.
- **Inference:** at that price, a 1,000-input-token request costs about $0.000042. Billing mechanics (prepaid or invoiced, minimums) are not documented publicly.

## 6. Calling it from a backend POST endpoint

- **Synchronous request/response only.** The API reference documents a single endpoint, with no async jobs, batch endpoint, streaming, or webhooks ([API reference](https://docs.typesafe.ai/api)).
- **SDK timeouts:** the default is 10 s in both SDKs (JS `timeout: 10000` ms in [TypeSafeClientConfig](https://docs.typesafe.ai/sdk/javascript/api/interfaces/TypeSafeClientConfig); Python `10.0` s in [constants](https://docs.typesafe.ai/sdk/python/api/constants)).
- **SDK retries (JS):** by default the SDK makes up to 2 retries after the first attempt. It retries HTTP 408, 429, and 500–599, plus connection and timeout errors. Backoff starts at 500 ms, doubles up to 5000 ms, and has 25% jitter. The SDK honors `Retry-After` up to 60 s ([RetryPolicy](https://docs.typesafe.ai/sdk/javascript/api/interfaces/RetryPolicy)).
- **Keep the key on the server:** the JS SDK has a `dangerouslyAllowBrowser` option that defaults to `false` ([TypeSafeClientConfig](https://docs.typesafe.ai/sdk/javascript/api/interfaces/TypeSafeClientConfig)). **Inference:** the SDK expects server-side use, so the API key should stay in the backend.
- **Configuration:** the SDK reads `TYPESAFE_API_KEY`, `TYPESAFE_BASE_URL`, `TYPESAFE_DEFAULT_MODEL`, and `TYPESAFE_LOG_LEVEL` from the environment ([constants](https://docs.typesafe.ai/sdk/python/api/constants)).
- **Data handling:** the DPA gives no concrete retention period for API data and mentions no zero-retention option ([DPA](https://typesafe.ai/legal/data-processing)). Vercel AI Gateway exposes a `zeroDataRetention` provider option when routing through Vercel ([Vercel changelog](https://vercel.com/changelog/typesafe-ai-jev-now-available-on-ai-gateway)).
- **Inference:** given the stated 70–500 ms latency and the 10 s default timeout, a synchronous call inside the POST handler looks workable. Larger JSON payloads could run into the 32k state limit. The jaggedness notes suggest trimming fields that don't matter for the criteria, and treating user-supplied text (such as Instagram captions) as possible prompt injection.

## Open questions

1. How long does early-access approval take, and can the project get a key at all right now? Is there a free or trial allowance? (Nothing is documented.)
2. Are scores deterministic for identical input, and is there any seed or temperature control? (Not documented.)
3. What is the maximum number of questions per request? Is there a per-request size cap in bytes, separate from the token limits?
4. Do the published rate limits (80 req/s, 100K tok/s) apply to early-access accounts, or are they lower per account?
5. How long are API inputs retained? Is zero-data-retention available when calling TypeSafe directly rather than through Vercel?
6. Jev returns no reasoning text. Does the frontend need an explanation for each score? (This is a product question for the design interview.)
7. Jev only accepts text. Instagram embeds would have to be judged from text and metadata (caption, counts, and so on), not from the images. Is that acceptable?
8. Two docs pages disagree slightly: the quickstart states Python >= 3.10, but the Python SDK page gives no version. The Vercel changelog is dated 2026-09-16, while the TypeSafe blog's fetched date reads 2026-09-28. Neither affects the design.

## Sources

- https://typesafe.ai
- https://typesafe.ai/blog/introducing-system-one-models-and-jev
- https://typesafe.ai/team
- https://typesafe.ai/legal/data-processing
- https://docs.typesafe.ai/llms.txt
- https://docs.typesafe.ai/introduction
- https://docs.typesafe.ai/introduction/quickstart
- https://docs.typesafe.ai/introduction/coding-agents
- https://docs.typesafe.ai/concepts/system-one
- https://docs.typesafe.ai/concepts/state
- https://docs.typesafe.ai/primitives
- https://docs.typesafe.ai/primitives/score
- https://docs.typesafe.ai/confidence
- https://docs.typesafe.ai/patterns/composite-scoring
- https://docs.typesafe.ai/models
- https://docs.typesafe.ai/model-jaggedness/jev-1.13
- https://docs.typesafe.ai/api
- https://docs.typesafe.ai/sdk
- https://docs.typesafe.ai/sdk/python
- https://docs.typesafe.ai/sdk/python/api/constants
- https://docs.typesafe.ai/sdk/javascript
- https://docs.typesafe.ai/sdk/javascript/api/interfaces/TypeSafeClientConfig
- https://docs.typesafe.ai/sdk/javascript/api/interfaces/RetryPolicy
- https://github.com/typesafe-ai (org and repos, via GitHub API)
- https://github.com/typesafe-ai/typesafe-sdk-python
- https://github.com/typesafe-ai/typesafe-sdk-js
- https://vercel.com/changelog/typesafe-ai-jev-now-available-on-ai-gateway (Vercel's own integration)
