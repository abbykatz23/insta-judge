# Instagram embed availability: research notes

_Researched 2026-10-04. Sources are Meta's developer docs (developers.facebook.com), the Instagram Help Center and Instagram's own blog, and the official `embed.js` served from instagram.com. I also made a few live calls to the oEmbed endpoint and labelled them **Observed** (run 2026-10-04 with no access token). Lines marked **Inference** are my reading of the sources, not something a source says._

## Summary

Reliable server-side detection is possible through Meta's official **Instagram oEmbed endpoint**, `GET https://graph.facebook.com/v26.0/instagram_oembed?url=<post URL>` ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed), [reference](https://developers.facebook.com/docs/graph-api/reference/instagram-oembed/)). Since **May 15, 2026** it can be called **without an access token** ([Instagram Platform changelog](https://developers.facebook.com/docs/instagram-platform/changelog)). That removes the old requirements: a Meta app, App Review for "Meta oEmbed Read", and business verification. Meta documents that posts from private, inactive, and age-restricted accounts, and from accounts with embeds turned off, are "not supported". **Observed:** an unembeddable post returns HTTP 400 with `code 24 / error_subcode 2207045` ("Media Not Found"), while an embeddable post returns 200 with `html`. That one error covers every failure reason: you get a yes/no answer, not a cause. The documented limit is **1,000 requests per hour**. Meta's terms say oEmbed output may be used **only** to provide a front-end view, and they forbid "persisting" the metadata and content. Using a 200 response as the gate for whether to render the embed fits that use. Storing the responses or using them for analytics does not. The official embed iframe does send `postMessage` events to the host page, but they are **undocumented**, so they are a weak fallback only.

## 1. The Instagram oEmbed endpoint

**Endpoint and parameters**
- `GET /instagram_oembed` on the `graph.facebook.com` host. The sample requests use `v26.0` ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)).
- `url` is required. The accepted formats are `https://www.instagram.com/p/{shortcode}/`, `/reel/{shortcode}/`, and profile URLs. The optional parameters are `maxwidth` (320–658), `hidecaption`, `omitscript`, and `fields` ([reference](https://developers.facebook.com/docs/graph-api/reference/instagram-oembed/)).
- The default response fields are `html`, `provider_name`, `provider_url`, `type`, `version`, and `width` ([reference](https://developers.facebook.com/docs/graph-api/reference/instagram-oembed/)). `author_name`, `author_url`, and `thumbnail_*` stopped being returned on Nov 3, 2025 ([changelog](https://developers.facebook.com/docs/instagram-platform/changelog), Apr 8, 2025 entry).

**Auth, app review, and tokens: how the requirements changed**
- 2020: the legacy oEmbed endpoints were replaced by Graph API endpoints that "require client or app access tokens" ([Meta blog, 2020-10-14](https://developers.facebook.com/blog/post/2020/10/14/required-migration-token-based-access-user-picture-oEmbed-endpoints/)).
- Meta oEmbed Read "Requires App Review" and "is only available with business verification". The allowed use is "To provide front-end views of public Facebook and Instagram pages, posts, and videos" ([Meta oEmbed Read feature](https://developers.facebook.com/docs/features-reference/meta-oembed-read)). It replaced the older oEmbed Read feature on Nov 3, 2025 ([changelog](https://developers.facebook.com/docs/instagram-platform/changelog)).
- Token formats, if you still use one: an app access token (`{app-id}|{app-secret}` or a generated token, server-side only) or a client token (`{app-id}|{client-token}`, not secret) ([access tokens guide](https://developers.facebook.com/docs/facebook-login/guides/access-tokens)).
- **May 15, 2026:** "You can now call Instagram oEmbed API without an access token." The note applies to all versions ([changelog](https://developers.facebook.com/docs/instagram-platform/changelog)). The current guide no longer mentions tokens at all ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)).
- **Observed:** with no token, `GET .../v26.0/instagram_oembed?url=https://www.instagram.com/p/DeEldkijBiQ/&omitscript=true` returned HTTP 200 with `html`. The response headers included `access-control-allow-origin: *` and `cache-control: private, no-cache, no-store, must-revalidate`. No `X-App-Usage` header was returned.
- **Inference:** the remaining question is whether the token-free path is still tied to "Meta oEmbed Read" policy and App Review. The feature page still says review is required, but the changelog says no token is needed. I found no document that reconciles the two (see Open questions).

**Rate limits**
- "You can make up to 1,000 requests every hour" ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)). The guide does not say whether that limit is per app, per IP, or global when no token is sent.
- Generic Graph API throttling errors are code 4 (app), 17 (user), 32 (Pages), and 613 (custom) ([rate limiting](https://developers.facebook.com/docs/graph-api/overview/rate-limiting)). The oEmbed reference also lists 368 ("deemed abusive or is otherwise disallowed") ([reference](https://developers.facebook.com/docs/graph-api/reference/instagram-oembed/)).

**Errors for posts that will not render**
- Documented error codes: 200 permissions, 100 invalid parameter, 190 invalid OAuth token, 368 abusive/disallowed ([reference](https://developers.facebook.com/docs/graph-api/reference/instagram-oembed/)). Meta does not document a code for each cause (deleted, private, embeds disabled).
- Documented as unsupported: "Posts on private, inactive, and age-restricted Instagram accounts", "Accounts that have disabled Embeds", and Stories ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)).
- **Observed:** for a nonexistent shortcode (`/p/AAAAAAAAAAA/`), the endpoint returned HTTP 400 with `{"code":24,"error_subcode":2207045,"is_transient":false,"error_user_title":"Media Not Found","error_user_msg":"The requested media could not be embedded either because it does not exist or you don't have permission to embed it."}`. For a non-Instagram URL it returned HTTP 400 with `code 100 / subcode 2207047` ("Invalid URL").
- **Inference:** the wording of `error_user_msg` suggests that deleted, private, embeds-disabled, and probably age-restricted posts all return the same `24/2207045`. I could only test a nonexistent post, though. Treat any non-200 with `is_transient:false` as "drop". Treat 4/17/32/613, 5xx, and `is_transient:true` as "unknown, retry later". Do not drop on those.

**Calling it per post at render time, and caching**
- **Inference:** per-post calls at render time are technically feasible but budget-limited. 1,000 per hour is about 16 per minute, so a feed that renders N posts per page view uses up the budget quickly. A better design is to check each post once when it is ingested, then re-check on a schedule (for example daily, or when a post is about to be shown and its last check is older than X). Keep only a boolean or a timestamp, not the returned `html`.
- Terms: "Using metadata and page, post, or video content (or their derivations) from the endpoint for any purpose other than providing a front-end view of the page, post, or video is strictly prohibited. This prohibition encompasses consuming, manipulating, extracting, or persisting the metadata and content…" ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)). Meta's Platform Terms also require deleting Platform Data once it is "no longer necessary" ([Platform Terms §3.d](https://developers.facebook.com/terms/)).
- **Inference:** checking embeddability in order to decide whether to show the front-end embed serves the front-end view, so it looks allowed. Persisting the `html` or metadata looks prohibited. The response's own `no-store` cache header points the same way. Meta never explicitly addresses an availability flag derived from the response (see Open questions).

## 2. Client-side signals from the official embed

- Meta documents only this: `embed.js` "scans the page for the post HTML and generates the fully rendered post", and you can call `instgrm.Embeds.process()` yourself after loading it with `omitscript=true` ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)). Meta documents no events, callbacks, or success/failure signals.
- **Observed (undocumented, from reading `https://www.instagram.com/embed.js` on 2026-10-04):** the script listens for `message` events whose origin matches `instagram.com`/`instagr.am`. It parses JSON `{type, details}` with types `LOADING`, `MEASURE`, and `MOUNTED`. On `MEASURE` it sets the iframe height. On `MOUNTED` it removes the placeholder blockquote and calls `window.__igEmbedLoaded({frameId, stats})` if the host page defines that function.
- **Inference:** a host page could listen for these messages itself, or define `__igEmbedLoaded`. A timeout without `MOUNTED` could then be treated as a failure. But this is private, undocumented behaviour that could change without notice. I also did not verify whether an "unavailable" embed still sends `MOUNTED`, which seems likely because the iframe does mount its error card. Cross-origin rules stop the host page from reading the iframe's DOM, so these `postMessage` events are the only channel.

## 3. Other documented ways to check

- **Business Discovery** (`GET /{ig-user-id}?fields=business_discovery.username(...){media{...}}`) returns public media for **Business or Creator** accounts only, and it omits age-gated accounts. It requires a Facebook User access token with `instagram_basic`, `instagram_manage_insights`, and `pages_read_engagement`, plus a linked Page/IG professional account ([Business Discovery reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/business_discovery)). **Inference:** it does not cover personal accounts, it says nothing about embed settings, and it costs much more to set up. Not a good fit.
- I found no other official way to check embeddability. Fetching `instagram.com/p/<code>/` or `/embed/` HTML from a server, or using internal JSON endpoints, is **unofficial scraping**. Do not use it. (**Observed:** the `/embed/` HTML is client-rendered and looks the same for valid and missing posts when fetched raw.)

## 4. Owners disabling embeds

- Instagram Help Center has an article titled "Turn off embed settings on Instagram" ([help.instagram.com/252460186989212](https://help.instagram.com/252460186989212)). Its search-index snippet reads: "If your account is public, you can turn off the setting that allows blogs and websites to embed your posts and profile." I could not load the article body because the page is client-rendered and blocked automated fetches. Its sections on "what happens to embedded posts" are **unverified** from the primary source.
- Private accounts: "Embed code is only available to those whose photos and videos are public" ([Instagram blog, 2013-07-10](https://about.instagram.com/blog/announcements/introducing-web-embedding-instagram-content-on-websites)). The oEmbed endpoint does not support accounts with embeds disabled or private accounts ([oEmbed guide](https://developers.facebook.com/docs/instagram-platform/oembed)).
- **Inference:** an owner's change shows up in two places. The oEmbed endpoint returns an error (probably `24/2207045`), and already-placed embeds degrade to an "unavailable" card in the iframe. That card is the broken state we want to avoid.

## Open questions

- Is the 1,000/hour limit per IP, per app, or global for token-free calls? Is it still 1,000 if you send an app token?
- Do private, embeds-disabled, and age-restricted posts all return `24/2207045`, or do some return different subcodes? I tested only a nonexistent post.
- Does token-free use still fall under the "Meta oEmbed Read" policy (front-end-view-only use)? **Inference:** yes, because the Limitations section still applies to the endpoint as a whole.
- Does Meta explicitly allow keeping a derived "embeddable: true/false + checked_at" flag? The docs neither permit nor forbid it.
- How long after an owner disables embeds or goes private does oEmbed start failing? The help article reportedly says it "may take time". Unverified.
- Does an unavailable embed still send `MOUNTED` through `postMessage`?

## Sources

- Meta: Embed an Instagram Post (oEmbed guide). https://developers.facebook.com/docs/instagram-platform/oembed
- Meta: Graph API Reference v26.0, Instagram oEmbed. https://developers.facebook.com/docs/graph-api/reference/instagram-oembed/
- Meta: Instagram Platform changelog (May 15, 2026 and Apr 8, 2025 entries). https://developers.facebook.com/docs/instagram-platform/changelog
- Meta: Meta oEmbed Read feature reference. https://developers.facebook.com/docs/features-reference/meta-oembed-read
- Meta: Access tokens guide. https://developers.facebook.com/docs/facebook-login/guides/access-tokens
- Meta: Required migration to token-based access for oEmbed (2020-10-14). https://developers.facebook.com/blog/post/2020/10/14/required-migration-token-based-access-user-picture-oEmbed-endpoints/
- Meta: Graph API rate limiting. https://developers.facebook.com/docs/graph-api/overview/rate-limiting
- Meta: Platform Terms. https://developers.facebook.com/terms/
- Meta: IG User Business Discovery reference. https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/business_discovery
- Meta: Embed Button. https://developers.facebook.com/docs/instagram-platform/embed-button
- Instagram Help Center: Turn off embed settings on Instagram. https://help.instagram.com/252460186989212
- Instagram Help Center: Embed an Instagram post or profile. https://help.instagram.com/620154495870484
- Instagram blog: Introducing Web Embedding (2013-07-10). https://about.instagram.com/blog/announcements/introducing-web-embedding-instagram-content-on-websites
- Instagram `embed.js` (official script, read directly). https://www.instagram.com/embed.js
