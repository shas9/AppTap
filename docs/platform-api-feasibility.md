# AppTap — Platform API Feasibility Report

**Scope:** Can AppTap legitimately schedule and auto-publish a founder's content to YouTube, Instagram, TikTok, Threads and X from one composer, and post a pre-written first comment on each published post?

**Companion documents:** [`competitive-analysis.md`](./competitive-analysis.md) · [`PRD.md`](./PRD.md) · [`design-doc.md`](./design-doc.md) — §6 of the design doc encodes every constant below as a machine-readable capability manifest.

**Interactive prototype:** <https://claude.ai/code/artifact/549422a6-0dec-4a8b-8887-cd95cb65a002> — the composer screen shows these limits enforced live (t.co-aware counting on X, UTF-8 byte counting on Threads), and the connected-accounts screen surfaces the audit and billing caveats found below.

**All research retrieved 3 September 2026.** Every claim below is sourced. Where official developer documentation does not state something, it is marked **UNVERIFIED** rather than filled in with a plausible guess. Platform terms in this space change frequently; re-verify before writing the first line of adapter code.

---

## 0. Executive verdict

| Platform | Publish verdict | First-comment verdict | Headline caveat |
|---|---|---|---|
| **Threads** | 🟢 Green | 🟢 Green | Cleanest API of the five. 500-char limit is the binding constraint. |
| **Instagram** | 🟡 Yellow | 🟢 Green | JPEG-only images, media must be fetched from a public URL, App Review + Business Verification (2–6 weeks, multi-round). Creator-vs-Business account support for publishing is genuinely ambiguous. |
| **YouTube** | 🟡 Yellow | 🟢 Green | Every video AppTap uploads is **forced private** until Google audits the API project. Post-audit it is Green. |
| **X** | 🟡 Yellow | 🟡 Yellow | Technically clean, economically hostile. No free tier for new developers. A post containing a URL costs **$0.20** — 13× a normal post. The first-comment-with-link feature is the single most expensive call in the product. |
| **TikTok** | 🟡 Yellow overall | 🔴 **Red** | No public comment-posting API exists at all. Additionally, all posts are forced `SELF_ONLY` and capped at 5 users/24h until TikTok audits the client, and TikTok mandates a specific composer UX that constrains AppTap's own compose screen. |

### The three findings that break the founder's stated flow

1. **"AppTap posts a first comment on all five platforms" is not achievable.** TikTok has no public API to create a comment on a video. The only comment-related API control is the `disable_comment` boolean set at post creation time. ([TikTok Direct Post reference](https://developers.tiktok.com/docs/en/content-posting-api-reference-direct-post), retrieved 2026-09-03.) The feature must ship as **4 of 5 platforms**, with TikTok either omitted or downgraded to a "copy this to your clipboard, we'll remind you" manual step. Do not let marketing copy say "all platforms."

2. **Two of the five platforms ship your content into a black hole until you pass an audit.** YouTube forces videos uploaded by unverified API projects to private ([videos.insert](https://developers.google.com/youtube/v3/docs/videos/insert)); TikTok forces `SELF_ONLY` visibility and requires the connected accounts themselves be private ([Content Sharing Guidelines](https://developers.tiktok.com/doc/content-sharing-guidelines)). Combined with Meta's App Review + Business Verification, **AppTap cannot demo a real end-to-end public post on 4 of 5 platforms until platform approvals land.** Budget 6–10 weeks of calendar time before the core promise works, and start the approval clock on day one — before the product is finished.

3. **X now bills per call and there is no free tier.** Standard post $0.015; **post containing a URL $0.200**; post read $0.005 ([X API pricing](https://docs.x.com/x-api/getting-started/pricing)). AppTap's "first comment = the link" design means every X post pair costs ~$0.215 in raw platform fees before any analytics reads. This is a real, recurring COGS line that must appear in the pricing model.

---

## 1. YouTube

### 1.1 API and account preconditions

| Item | Finding | Source |
|---|---|---|
| API | YouTube Data API v3, `videos.insert` (resumable upload protocol) | [videos.insert](https://developers.google.com/youtube/v3/docs/videos/insert) |
| Scopes | `youtube.upload`, `youtube` , `youtube.force-ssl`, `youtubepartner` | same |
| Account precondition | A YouTube channel on the authorising Google Account. No Business/Creator tier required. For comment insertion, "the YouTube account used to authorize the API request must be merged with the user's Google Account" | [commentThreads.insert](https://developers.google.com/youtube/v3/docs/commentThreads/insert) |

### 1.2 Unattended publishing

**Permitted — with one enormous asterisk.** There is no user-confirmation step in the API. But: *"All videos uploaded via the `videos.insert` endpoint from unverified API projects created after 28 July 2020 will be restricted to private viewing mode."* Lifting this requires a compliance audit against the YouTube API Services Terms of Service. ([videos.insert](https://developers.google.com/youtube/v3/docs/videos/insert), retrieved 2026-09-03.)

Practical consequence: until AppTap's Google Cloud project is audited, **every YouTube publish silently produces a private video.** This will look to the user like the product is broken. Handle it in the UI explicitly during the pre-audit period.

### 1.3 Media

| Item | Finding |
|---|---|
| Media types | Video only. No image posts, no carousels. |
| Max file size | 256 GB |
| Accepted MIME | `video/*`, `application/octet-stream` |
| Shorts | No API flag and no Shorts-specific endpoint. YouTube classifies automatically: videos uploaded on/after 15 October 2024 with square or vertical aspect ratio and length ≤ 3 minutes are categorised as Shorts ([YouTube Help: three-minute Shorts](https://support.google.com/youtube/answer/15424877)). |
| Shorts detection reliability | **UNVERIFIED.** Community reports describe borderline videos failing to be classified as Shorts. Google's developer docs do not document the classifier. Treat Shorts targeting as best-effort. |

### 1.4 Text limits and links

| Field | Limit | Notes |
|---|---|---|
| `snippet.title` | 100 characters | `<` and `>` invalid |
| `snippet.description` | 5,000 **bytes** (not characters — emoji cost more) | `<` and `>` invalid; timestamps create chapters |
| `snippet.tags[]` | 500 characters total across all tags, commas count, tags with spaces are quoted and the quotes count | |
| Links in description | Supported, **but** *"For URLs in the description to be rendered as clickable links in the YouTube UI, the channel must meet platform-level requirements, such as channel verification or having Advanced Features enabled."* | [videos resource](https://developers.google.com/youtube/v3/docs/videos) |

The link caveat is material for AppTap's target user: a brand-new indie app channel will have non-clickable description links. This *strengthens* the case for the first-comment feature on YouTube, but comment links have the same rendering constraint — **UNVERIFIED** whether comment URLs are clickable on unverified channels; test empirically.

### 1.5 Comment posting — 🟢 supported

`commentThreads.insert` creates a top-level comment on a video. Requires `part=snippet`, `snippet.channelId`, `snippet.videoId`, `snippet.topLevelComment.snippet.textOriginal`. Scope: `youtube.force-ssl`. Quota cost 50 units. Fails if comments are disabled on the video or the account is not merged/eligible. ([commentThreads.insert](https://developers.google.com/youtube/v3/docs/commentThreads/insert))

Exact comment character limit is **UNVERIFIED** — docs say only that "excessively long comments will be rejected."

### 1.6 Quota — note the 2026 model change

The old flat 1,600-units-per-upload model is gone. Current documented model ([Determine quota cost](https://developers.google.com/youtube/v3/determine_quota_cost)):

- **Separate `videos.insert` bucket: 100 uploads/day per project.**
- Separate `search.list` bucket: 100 calls/day.
- 10,000 units/day shared across all other endpoints.
- `commentThreads.insert` = 50 units → ~200 first-comments/day at the shared cap, before any analytics reads.

**The 100 uploads/day project-wide cap is the real scaling ceiling.** It is per *API project*, not per user. At 1 video/user/day AppTap hits it at ~100 active YouTube-connected customers. Requesting more requires the audit. Plan for this well before it bites. (Community reports of a newly-enforced hidden "Video Uploads per day" 429 in mid-2026 corroborate that this bucket is actively enforced — [google-api-python-client#2753](https://github.com/googleapis/google-api-python-client/issues/2753), community source, **treat as indicative not authoritative**.)

### 1.7 Review, cost, tokens, ToS

- **Audit:** required both to raise quota and (per §1.2) to lift the private-video restriction. Google states a team member "will contact you as soon as possible" and gives **no turnaround SLA** ([Quota and Compliance Audits](https://developers.google.com/youtube/v3/guides/quota_and_compliance_audits)). Turnaround is **UNVERIFIED**; community reports range from weeks to months.
- **Cost:** free.
- **Tokens:** Google refresh tokens die when: unused for 6 months; the user revokes; the OAuth app is in *Testing* publishing status (**7-day token expiry** — this will bite during development); the account exceeds 100 refresh tokens per client ID (oldest silently invalidated); or a Gmail-scoped password change. ([Using OAuth 2.0](https://developers.google.com/identity/protocols/oauth2))
- **ToS:** YouTube API Services Terms of Service, enforced via the audit.

**Verdict: 🟡 Yellow → 🟢 Green after audit.** No architectural blocker. The blocker is calendar time and an undisclosed approval SLA.

---

## 2. Instagram

### 2.1 API and account preconditions

Two distinct setups ([Instagram Platform](https://developers.facebook.com/docs/instagram-platform/)):

| Setup | Facebook Page required? | Scopes |
|---|---|---|
| **Instagram API with Instagram Login** (recommended for AppTap) | **No** — *"This API setup does not require a Facebook Page to be linked to the Instagram professional account."* | `instagram_business_basic`, `instagram_business_content_publish`, `instagram_business_manage_comments` |
| Instagram API with Facebook Login | Yes | `instagram_basic`, `instagram_content_publish`, `pages_read_engagement`, plus `ads_management`/`ads_read` in some Business Manager configurations |

Choose **Instagram Login**. Removing the Facebook Page requirement removes the single worst onboarding step for indie founders. Cost: no ads or product-tagging access — irrelevant for AppTap's MVP.

**Account type — genuinely ambiguous, must test.** The Instagram Login overview says the API serves *"Instagram professionals — businesses and creators."* The Content Publishing limitations page describes professional accounts without confirming Creator support. Multiple third-party integration guides state flatly that **Reels publishing works only on Business accounts and Creator accounts are not supported for content publishing** (e.g. [Phyllo](https://www.getphyllo.com/post/a-complete-guide-to-the-instagram-reels-api)). This is **UNVERIFIED against official Meta docs and is a real product risk** — many indie founders run Creator accounts. Resolve empirically with a test Creator account before onboarding copy is written.

### 2.2 Unattended publishing — 🟢 permitted

Two-step, fully server-side, no user confirmation:
1. `POST /{ig-user-id}/media` → creates a container (expires after 24 h if unpublished)
2. `POST /{ig-user-id}/media_publish`

([Content Publishing](https://developers.facebook.com/docs/instagram-platform/content-publishing/))

### 2.3 Media

| Item | Finding |
|---|---|
| Types | Single image, single video, **REELS**, **STORIES**, **CAROUSEL** (2–10 mixed items) |
| **Image format** | **JPEG only.** *"Extended JPEG formats such as MPO and JPS are not supported."* No PNG, no HEIC, no WebP. |
| Image specs | ≤ 8 MB, aspect ratio 4:5 → 1.91:1, width 320–1440 px (auto-scaled outside range), sRGB (auto-converted) |
| Carousel | ≤ 10 items; **all items cropped to the first item's aspect ratio** (default 1:1); child items cannot carry captions; Reels cannot be carousel items |
| Reels video | MOV/MP4, no edit lists, moov atom at front; H.264 or HEVC; AAC ≤ 48 kHz mono/stereo; 23–60 FPS; aspect 0.01:1–10:1 (9:16 recommended); ≤ 1920 px horizontal; VBR ≤ 25 Mbps; **3 s – 15 min**; **≤ 300 MB** |
| Delivery | **Media must be hosted on a publicly accessible URL at publish time** — Meta fetches it. Resumable upload sessions exist for large video. |

Sources: [IG User Media reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media/), [Content Publishing](https://developers.facebook.com/docs/instagram-platform/content-publishing/).

**Two hard engineering consequences.** (a) AppTap must transcode/convert *everything* to JPEG + spec-compliant H.264 MP4 — a founder dragging in a PNG screenshot is the common case, not the edge case. (b) AppTap needs public, unguessable, time-limited media URLs. Signed CDN URLs with a TTL comfortably longer than the container's 24-hour life.

### 2.4 Text limits and links

- Caption: **2,200 characters, 30 hashtags, 20 @mentions** ([IG User Media reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media/)).
- Over-limit captions are reported to fail with error `2207010` (third-party source, **treat as indicative**) — AppTap should validate client-side and never rely on this.
- **Links in captions are not clickable.** This is long-standing Instagram product behaviour and is the entire reason the founder wants a first-comment feature — but note it is *not* stated in the developer docs. Marked **UNVERIFIED at the documentation level; universally observed in the product.**
- The claim that Instagram *algorithmically penalises* link-bearing captions is folklore. **Unverified, no primary source. Do not put it in marketing copy.**

### 2.5 Comment posting — 🟢 supported

`POST /{ig-media-id}/comments` with required `message` parameter *"Creates an IG Comment on an IG Media object."* Requires `instagram_business_manage_comments` (Instagram Login) or `instagram_manage_comments` (Facebook Login). Not supported on live video media. ([IG Media comments reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-media/comments/))

Docs do **not** state a comment character limit or an explicit "own media only" restriction — both **UNVERIFIED**.

### 2.6 Rate limits

**100 API-published posts per rolling 24 h per Instagram account.** Carousels count as one. ([Content Publishing](https://developers.facebook.com/docs/instagram-platform/content-publishing/)) This is per-account and generous — not a scaling constraint for AppTap. Standard Graph API app-level rate limiting also applies.

### 2.7 Review, cost, tokens

- **App Review + Business Verification** required for Advanced Access (serving accounts you don't own). Meta's stated timeline is 2–4 weeks per submission; third-party reports put a realistic end-to-end at **2–6 weeks with multiple rejection rounds**, with the screencast and Business Verification as the common failure points ([bundle.social](https://bundle.social/blog/meta-app-review-20-days), [singhamandeep.com](https://singhamandeep.com/instagram-api-advanced-access-approval/) — community sources, **indicative**).
- **Cost:** free.
- **Tokens:** short-lived 1 h → long-lived **60 days**. Refresh via `GET /refresh_access_token?grant_type=ig_refresh_token`; the token must be **≥ 24 h old and unexpired**. A token not refreshed within 60 days is permanently dead and the user must re-authorise. ([Refresh Access Token](https://developers.facebook.com/docs/instagram-platform/reference/refresh_access_token/))

**Verdict: 🟡 Yellow.** Nothing in the founder's flow is prohibited. The yellow is: JPEG-only conversion burden, public-URL media hosting, a multi-week review with real rejection risk, and unresolved Creator-account support.

---

## 3. TikTok

### 3.1 API and account preconditions

| Item | Finding | Source |
|---|---|---|
| Video endpoint | `POST /v2/post/publish/video/init/` then chunked `PUT` to returned `upload_url`, or `PULL_FROM_URL` | [Direct Post](https://developers.tiktok.com/docs/en/content-posting-api-reference-direct-post) |
| Photo endpoint | `POST /v2/post/publish/content/init/` with `media_type=PHOTO` | [Photo Post](https://developers.tiktok.com/doc/content-posting-api-reference-photo-post) |
| Scope | `video.publish` (Direct Post) or `video.upload` (draft into TikTok inbox) | same |
| Mandatory pre-flight | Must query `creator_info` before every post to obtain the creator's allowed privacy options, interaction settings, and `max_video_post_duration_sec` | [Content Sharing Guidelines](https://developers.tiktok.com/doc/content-sharing-guidelines) |
| Account type | Any TikTok account. **But unaudited clients require the account be set to private at the time of posting.** | same |

### 3.2 Unattended publishing — 🟡 the hard case

Three separate constraints stack:

**(a) The unaudited-client wall.** Until TikTok audits the API client: all content is forced `SELF_ONLY`; connected accounts must be private at post time; **at most 5 users may post per 24-hour window**. To make anything public afterwards the creator must manually flip account visibility to public *and then* change each post's privacy to "Everyone." Enforcement is server-side. ([Content Sharing Guidelines](https://developers.tiktok.com/doc/content-sharing-guidelines))

**(b) The mandatory composer UX.** TikTok's guidelines require the integration to: display the creator nickname, call `creator_info` to check posting availability, force **manual selection of privacy status with no default**, disable Comment/Duet/Stitch toggles where the creator's own settings restrict them, disclose commercial content with the correct label, show a content preview, and show publish status. ([Content Sharing Guidelines](https://developers.tiktok.com/doc/content-sharing-guidelines))

This is not a suggestion — auditors check it, and community reports name "skipping the UX composer" as the most reliable audit failure ([bundle.social](https://bundle.social/blog/tiktok-api-approval), community source). **AppTap's unified composer must contain a TikTok-specific sub-panel that reproduces this UX.** That is real design and engineering work that the "one composer for everything" concept does not anticipate.

**(c) The consent wording.** The guidelines state *"API Clients must only start sending content materials to TikTok after the user has expressly consented to the upload."*

Read literally, this could forbid unattended scheduled publishing. **It does not, in practice.** Buffer and Later are official TikTok Marketing Partners publishing on a schedule through this same API ([Buffer TikTok scheduling](https://buffer.com/resources/how-to-schedule-tiktok-posts/)); the consent is understood to be given at compose/schedule time, when the user picks privacy and confirms the preview. Note also that TikTok has **no server-side scheduling** — the tool holds the media and calls the API at the target time.

**Confidence: moderate-high, but this is an interpretation, not a documented ruling.** It is the single most important thing to raise explicitly in the audit application. Flag as an open risk, not a settled question.

### 3.3 Media

| Item | Finding |
|---|---|
| Video formats | MP4 (recommended), WebM, MOV; codecs H.264 (recommended), H.265, VP8, VP9 |
| Video size | ≤ 4 GB; 360–4,096 px each dimension; 23–60 FPS |
| Video duration | **≤ 10 minutes via API**; per-creator cap returned by `creator_info` (commonly `max_video_post_duration_sec: 300`) |
| Photos | WebP or JPEG, ≤ 1080p, **≤ 20 MB each, up to 35 per post** |
| Photo transfer | **`PULL_FROM_URL` only** — no direct file upload for photos |
| URL ownership | `PULL_FROM_URL` requires domain or URL-prefix ownership verification in the TikTok developer portal |
| Chunking | 5 MB min / 64 MB max per chunk (final chunk ≤ 128 MB), 1–1,000 chunks, sequential |

([Media Transfer Guide](https://developers.tiktok.com/doc/content-posting-api-media-transfer-guide))

### 3.4 Text limits

| Post type | Field | Limit |
|---|---|---|
| Video | `title` (the caption, includes hashtags and mentions) | **2,200 UTF-16 runes** |
| Photo | `title` | **90 UTF-16 runes** |
| Photo | `description` | **4,000 UTF-16 runes** |

The 90-character photo title is a trap: it is by far the tightest limit of any field on any of the five platforms, and it is not the field most people assume is the caption. AppTap's validator must model video and photo TikTok posts as *different* variant types.

Link handling in TikTok captions: **UNVERIFIED.** Not documented in the Content Posting API reference.

### 3.5 Comment posting — 🔴 NOT POSSIBLE

**There is no public TikTok API endpoint for a third-party app to create a comment.** The only comment control exposed by the Content Posting API is the `disable_comment` boolean at post creation. TikTok's Research API has a read-only comment-listing endpoint (`POST /v2/research/video/comment/list/`) restricted to credentialed academic researchers in approved jurisdictions — not usable here.

This kills the founder's stated feature on this platform. Independent corroboration: Publer's own documentation lists follow-up comments as **not supported on TikTok** ([Publer post callbacks](https://publer.com/docs/posting/create-posts/content-types/post-callbacks)) — i.e. a mature commercial competitor with full TikTok partner status also cannot do it.

**Compliant alternatives, in order of quality:**
1. Put the link in the TikTok bio and let the caption say "link in bio." AppTap could own the link-in-bio page (see PRD Part C).
2. Ship a "pending manual comment" task: at publish time AppTap pushes a notification with the comment text pre-copied and a deep link to the post. Honest, low-effort, still saves the founder the retyping.
3. Do not ship it on TikTok and say so in the UI.

### 3.6 Rate limits

- **6 requests per minute per user access token.**
- Daily per-creator posting caps are set during the audit based on the usage estimate you supply in the application. Pick that number carefully — it becomes your ceiling.
- Unaudited: 5 users per 24 h across the whole client.

### 3.7 Review, cost, tokens, ToS

- **Audit:** sandbox validation first, then production review requiring a demo video of the upload flow, privacy policy URL, and a data-handling description. Community reports: **5–10 business days**, rejections common ([bundle.social](https://bundle.social/blog/tiktok-api-approval), [PostPeer](https://www.postpeer.dev/blog/best-tiktok-posting-api) — community sources, **indicative**).
- **Cost:** free.
- **Tokens:** access token **24 h** (`expires_in: 86400`); refresh token **365 days** (`refresh_expires_in: 31536000`), refreshable without user interaction. Refresh 10–30 minutes before expiry. Expiry or user revocation ⇒ full re-auth. ([User Access Token Management](https://developers.tiktok.com/docs/en/oauth-user-access-token-management))

**Verdict: 🟡 Yellow for publishing (Green only after audit), 🔴 Red for first-comment.** TikTok is the most expensive platform to integrate per unit of value delivered, and it is also the one indie app founders most want. Build it, but budget for it honestly.

---

## 4. Threads

### 4.1 API and account preconditions

| Item | Finding | Source |
|---|---|---|
| Endpoints | `POST /{threads-user-id}/threads` (container) → `POST /{threads-user-id}/threads_publish` | [Threads Posts](https://developers.facebook.com/docs/threads/posts) |
| Scopes | `threads_basic` (all calls) + `threads_content_publish` (publishing); `threads_manage_replies` for reply management | [Create Posts](https://developers.facebook.com/docs/threads/create-posts/) |
| Account precondition | A Threads profile. **Without advanced approval, posting is limited to your own account and app-tester accounts.** | same |
| Note | API-published posts are shared to the fediverse for users who have enabled that setting | same |

### 4.2 Unattended publishing — 🟢 permitted

Same container-then-publish pattern as Instagram. No confirmation step.

### 4.3 Media

| Type | Specs |
|---|---|
| TEXT | **500 characters** (emoji counted as UTF-8 bytes) |
| IMAGE | JPEG or PNG, ≤ 8 MB, aspect ≤ 10:1, width 320–1440 px, sRGB |
| VIDEO | MOV or MP4, ≤ 1 GB, **≤ 300 s**, VBR ≤ 100 Mbps, 23–60 FPS, aspect 0.01:1–10:1 (9:16 recommended), ≤ 1920 px horizontal |
| CAROUSEL | **2–20 items**, images and videos may be mixed; counts as one post for rate limits |

Note Threads accepts **PNG** where Instagram does not — the two Meta platforms do *not* share a media pipeline.

### 4.4 Text, links, tags

- 500 characters, hard.
- **`link_attachment`** parameter attaches a URL to the post. Text-only posts support up to **5 unique links**.
- `topic_tag` parameter (1–50 chars) or in-text `#hashtag`.
- `alt_text` ≤ 1,000 characters.
- `reply_control`: `everyone`, `accounts_you_follow`, `mentioned_only`, `parent_post_author_only`, `followers_only`.

([Publishing reference](https://developers.facebook.com/docs/threads/reference/publishing/))

Because Threads has native link attachment, the first-comment-for-links pattern is *less* necessary here — but still useful for a secondary CTA.

### 4.5 Comment posting — 🟢 supported

`POST /{threads-user-id}/threads` accepts **`reply_to_id`** — *"Required if replying to a post."* ([Publishing reference](https://developers.facebook.com/docs/threads/reference/publishing/)) So AppTap publishes the post, captures the returned media ID, then creates a second container with `reply_to_id` set and publishes it. Replies must be sequential; there is no batch endpoint for a whole thread.

This means Threads also supports the "auto-thread" feature for free — worth noting as a differentiator opportunity.

### 4.6 Rate limits

Per Threads profile, 24-hour rolling window ([Threads troubleshooting](https://developers.facebook.com/docs/threads/troubleshooting/)):

| Action | Limit |
|---|---|
| Posts | 250 |
| Replies | 1,000 |
| Deletions | 100 |
| Location searches | 500 |

Check with `GET /{threads-user-id}/threads_publishing_limit` — **AppTap should call this pre-flight and surface remaining quota in the UI.**

### 4.7 Review, cost, tokens

- **Review:** Meta App Review for advanced access, same machinery and roughly the same timeline as Instagram.
- **Cost:** free.
- **Tokens:** short-lived 1 h → long-lived **60 days** via `GET /access_token` (server-side only). Refresh via `GET /refresh_access_token` when the token is ≥ 24 h old, unexpired, and `threads_basic` is granted. **Not refreshed for 60 days ⇒ permanently expired, no recovery except re-auth.** ([Long-Lived Tokens](https://developers.facebook.com/docs/threads/get-started/long-lived-tokens/))

**Verdict: 🟢 Green.** Full media coverage, native carousel up to 20, native reply, generous limits, free. The only real constraint is 500 characters.

---

## 5. X

### 5.1 API and account preconditions

| Item | Finding | Source |
|---|---|---|
| Endpoint | `POST /2/tweets` | [Creation of a Post](https://docs.x.com/x-api/posts/creation-of-a-post) |
| Scopes | `tweet.write`, `tweet.read`, `users.read` (+ `offline.access` for refresh tokens) | same; [OAuth 2.0 PKCE](https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code) |
| Media upload | v2 chunked: `POST /2/media/upload/initialize` → `POST /2/media/upload/{id}/append` → `POST /2/media/upload/{id}/finalize` → `GET /2/media/upload` (STATUS) | [Chunked media upload](https://docs.x.com/x-api/media/quickstart/media-upload-chunked) |
| Account precondition | Any X account. Post length and video ceiling depend on X Premium status. |

### 5.2 Unattended publishing — 🟢 permitted

No confirmation step. X's automation rules apply (the public rules page was not fetchable during research — **UNVERIFIED**; re-check [help.x.com automation rules](https://help.x.com/en/rules-and-policies/x-automation) before launch, particularly around duplicate/substantially-similar content posted across accounts).

### 5.3 Media

| Item | Finding |
|---|---|
| Per post | 1–4 `media_ids`; max **4 photos**, or **1 GIF**, or **1 video** |
| Images | ≤ 5 MB (`tweet_image`) |
| GIF | ≤ 15 MB |
| Video | ≤ 20 min / 8 GB standard; ≤ 125 min / 16 GB for X Premium |
| Chunk size | ≤ 5 MB per append |
| Quote posts | **Enterprise plan only** — not available on self-serve tiers |

X's 4-photo cap is the binding constraint on AppTap's carousel feature: an Instagram 10-image carousel cannot be mirrored to X. The composer must warn and offer a truncation-to-4 (or thread-the-rest) strategy.

### 5.4 Text limits and links

- **280 characters** standard.
- X Premium allows longer posts; reported figures conflict (**4,000** for Premium vs **25,000** for verified organisations). **UNVERIFIED — no authoritative primary source located.** Design AppTap's counter against **280** and treat anything longer as an opt-in advanced setting the user asserts.
- Links are fully supported and clickable, and consume characters via t.co shortening. **The often-repeated claim that X down-ranks link posts is unverified folklore — no primary source. Do not build product copy on it.**

### 5.5 Comment posting — 🟡 supported but billed

`POST /2/tweets` with `reply.in_reply_to_tweet_id`. Same endpoint, same billing. Works fine — but see §5.7 for what it costs.

### 5.6 Rate limits

- **100 posts / 15 minutes per user**
- **10,000 posts / 24 h per app**
- Rate limits and billing are explicitly separate systems — staying inside rate limits does not reduce spend.

([Rate limits](https://docs.x.com/x-api/fundamentals/rate-limits))

### 5.7 Cost — the decisive finding

X moved to **pay-per-usage credit billing**, no subscription ([X API pricing](https://docs.x.com/x-api/getting-started/pricing), retrieved 2026-09-03):

| Operation | Price |
|---|---|
| Create post | **$0.015** |
| **Create post containing a URL** | **$0.200** |
| Summoned post | $0.010 |
| Post read | $0.005 per resource |
| User read | $0.010 per resource |
| **Owned reads** (your own data) | **$0.001 per resource** |

Monthly cap of 3M post reads on pay-per-usage; above that requires Enterprise. Resources are de-duplicated within a 24-hour UTC window.

Third-party sources report the **free tier was discontinued for new developers on 6 February 2026**, legacy Basic ($200/mo) retired with subscribers force-migrated, Pro ($5,000/mo) closed to new signups, Enterprise from ~$42,000/mo ([Postproxy](https://postproxy.dev/blog/x-api-pricing-2026/), [SocialCrawl](https://www.socialcrawl.dev/blog/x-twitter-api-2026) — community sources). Official docs describe only pay-per-usage and mention no free tier, which is consistent. **Treat as high-confidence but confirm in the developer portal before pricing AppTap.**

**Unit economics, worked:** an AppTap user posting once a day to X with a link in the first comment costs `$0.015 (post) + $0.200 (reply with URL) = $0.215/day ≈ $6.45/month` in raw X fees alone. Add per-post analytics polling (owned reads at $0.001) and X plausibly consumes **20–30% of a $29/month subscription for a single connected platform.** Mitigations: batch/limit analytics polling; consider putting the link in the post body rather than the comment (same $0.20 either way, but one call not two); or make X a paid add-on.

### 5.8 Tokens

OAuth 2.0 Authorization Code with PKCE. **Access token 2 hours.** A refresh token is issued **only if `offline.access` scope is requested** — omit it and there is no refresh token at all, which is a silent, catastrophic bug for a scheduling product. Refresh-token lifetime is **UNVERIFIED**. ([OAuth 2.0 PKCE](https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code))

**Verdict: 🟡 Yellow.** Technically the easiest integration of the five; commercially the one that can quietly destroy gross margin.

---

## 6. Cross-platform summary matrix

| | YouTube | Instagram | TikTok | Threads | X |
|---|---|---|---|---|---|
| **Unattended publish** | ✅ (private until audit) | ✅ | ⚠️ SELF_ONLY until audit; mandated composer UX | ✅ | ✅ |
| **Single image** | ❌ | ✅ (JPEG only) | ✅ (as 1-photo post) | ✅ (JPEG/PNG) | ✅ |
| **Carousel** | ❌ | ✅ 2–10 | ✅ up to 35 photos | ✅ 2–20 mixed | ⚠️ max 4 images |
| **Video** | ✅ ≤ 256 GB | ✅ Reels 3 s–15 min, ≤ 300 MB | ✅ ≤ 10 min, ≤ 4 GB | ✅ ≤ 300 s, ≤ 1 GB | ✅ ≤ 20 min (125 min Premium) |
| **Short-form format** | Shorts (auto-classified, ≤ 3 min vertical) | Reels | native | native | native |
| **Caption limit** | title 100 / desc 5,000 bytes | 2,200 (30 #, 20 @) | video 2,200 · photo title 90 + desc 4,000 | **500** | **280** (Premium longer, unverified) |
| **Links in caption** | ✅ but non-clickable without channel verification | ⚠️ not clickable (product behaviour) | unverified | ✅ `link_attachment`, up to 5 | ✅ clickable, **$0.20/post** |
| **First comment via API** | ✅ `commentThreads.insert` | ✅ `POST /{media}/comments` | ❌ **none** | ✅ `reply_to_id` | ✅ `in_reply_to_tweet_id` (billed) |
| **Publish cap** | 100 uploads/day **per API project** | 100/day per account | 6 req/min per token; audit-set daily cap | 250 posts + 1,000 replies/day per profile | 100/15 min per user; 10,000/day per app |
| **Access cost** | free | free | free | free | **~$0.015–0.215 per post** |
| **Approval** | Compliance audit (no SLA) | App Review + Business Verification, 2–6 wks | Sandbox + audit, ~5–10 business days reported | Meta App Review | Credit purchase; no review gate found |
| **Access token life** | ~1 h | 1 h | 24 h | 1 h | **2 h** |
| **Refresh token life** | until revoked / 6 mo idle / 7 days in Testing | long-lived 60 d, must refresh ≥24 h old | 365 d | long-lived 60 d | only with `offline.access`; lifetime unverified |
| **Verdict** | 🟡→🟢 | 🟡 | 🟡 / 🔴 comment | 🟢 | 🟡 |

---

## 7. Where the founder's assumptions do not survive

1. **"AppTap posts a first comment on all five."** False. TikTok: no API. Ship 4/5 and say so.
2. **"Post it once, it goes everywhere automatically."** True only after 3 separate approvals land (Google audit, Meta App Review + Business Verification, TikTok audit). Before that, YouTube produces private videos and TikTok produces SELF_ONLY posts on forcibly-private accounts. Approvals are the critical path, not the code.
3. **"One composer for everything."** TikTok's guidelines require a platform-specific consent/preview/privacy UX with no defaulted privacy value. The composer must have a TikTok panel that is *not* a generic caption box.
4. **"The same media goes to all five."** No. Instagram takes JPEG only and caps Reels at 300 MB; X takes at most 4 images; YouTube takes only video; TikTok photos must be pulled from a domain you have verified with TikTok. Transcoding and per-platform media derivatives are MVP work, not v2 polish.
5. **"Character limits differ" is the smallest of the problems.** True and worth solving — but the *field structure* differs more than the counts: YouTube has title + description + tags; TikTok photo posts have title + description; everyone else has one caption field. A single canonical caption with per-platform overrides needs a per-platform *field map*, not just a per-platform length.
6. **"APIs are free."** No longer true for X.

---

## 8. Open items to verify before writing adapter code

| # | Item | Why it matters |
|---|---|---|
| 1 | Does Instagram content publishing (esp. Reels) work on **Creator** accounts, or Business only? | Determines whether a large share of indie founders can use AppTap at all |
| 2 | X refresh-token lifetime and rotation behaviour | Determines re-auth frequency; a 2-hour access token with no working refresh is fatal |
| 3 | X Premium character ceiling (4,000 vs 25,000) | Composer counter correctness |
| 4 | Whether X's free tier truly ended for new developers | Whether AppTap has any zero-cost dev path on X |
| 5 | YouTube audit turnaround, and whether the audit lifts both the private-video restriction and the 100/day upload cap | The 100 uploads/day project cap is a hard ceiling at ~100 customers |
| 6 | Whether YouTube comment URLs are clickable on unverified channels | Determines whether the first-comment feature actually delivers a working link on YouTube |
| 7 | Instagram and YouTube comment character limits | Composer validation |
| 8 | TikTok's position on scheduled (non-user-present) publishing, in writing, via the audit application | The one interpretation in this report that is not documented |
| 9 | X automation rules on cross-posted / substantially similar content | Cross-posting is literally the product |
| 10 | Whether Meta treats a multi-tenant scheduler differently from a single-brand tool in App Review | Affects the screencast and use-case narrative |

---

## Sources

- [YouTube Data API — Videos: insert](https://developers.google.com/youtube/v3/docs/videos/insert)
- [YouTube Data API — Videos resource](https://developers.google.com/youtube/v3/docs/videos)
- [YouTube Data API — CommentThreads: insert](https://developers.google.com/youtube/v3/docs/commentThreads/insert)
- [YouTube Data API — Determine quota cost](https://developers.google.com/youtube/v3/determine_quota_cost)
- [YouTube Data API — Quota and Compliance Audits](https://developers.google.com/youtube/v3/guides/quota_and_compliance_audits)
- [YouTube Help — Understand three-minute Shorts](https://support.google.com/youtube/answer/15424877)
- [Google Identity — Using OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)
- [Meta — Instagram Platform: Content Publishing](https://developers.facebook.com/docs/instagram-platform/content-publishing/)
- [Meta — Instagram API with Instagram Login](https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login)
- [Meta — IG User Media reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media/)
- [Meta — IG Media comments reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-media/comments/)
- [Meta — Instagram comment moderation](https://developers.facebook.com/docs/instagram-platform/comment-moderation/)
- [Meta — Refresh Access Token](https://developers.facebook.com/docs/instagram-platform/reference/refresh_access_token/)
- [Meta — Threads Posts](https://developers.facebook.com/docs/threads/posts)
- [Meta — Threads Create Posts](https://developers.facebook.com/docs/threads/create-posts/)
- [Meta — Threads Publishing reference](https://developers.facebook.com/docs/threads/reference/publishing/)
- [Meta — Threads Reply Management](https://developers.facebook.com/docs/threads/reply-management/)
- [Meta — Threads Troubleshooting / rate limits](https://developers.facebook.com/docs/threads/troubleshooting/)
- [Meta — Threads Long-Lived Tokens](https://developers.facebook.com/docs/threads/get-started/long-lived-tokens/)
- [TikTok — Content Sharing Guidelines](https://developers.tiktok.com/doc/content-sharing-guidelines)
- [TikTok — Direct Post reference](https://developers.tiktok.com/docs/en/content-posting-api-reference-direct-post)
- [TikTok — Photo Post reference](https://developers.tiktok.com/doc/content-posting-api-reference-photo-post)
- [TikTok — Media Transfer Guide](https://developers.tiktok.com/doc/content-posting-api-media-transfer-guide)
- [TikTok — User Access Token Management](https://developers.tiktok.com/docs/en/oauth-user-access-token-management)
- [X — API pricing](https://docs.x.com/x-api/getting-started/pricing)
- [X — Creation of a Post](https://docs.x.com/x-api/posts/creation-of-a-post)
- [X — Chunked media upload](https://docs.x.com/x-api/media/quickstart/media-upload-chunked)
- [X — Rate limits](https://docs.x.com/x-api/fundamentals/rate-limits)
- [X — OAuth 2.0 Authorization Code Flow with PKCE](https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code)
- [Apple — App Store Connect API: Customer Reviews](https://developer.apple.com/documentation/appstoreconnectapi/customer-reviews) / [Customer Review Responses](https://developer.apple.com/documentation/appstoreconnectapi/customer-review-responses) / [Downloading Analytics Reports](https://developer.apple.com/documentation/appstoreconnectapi/downloading-analytics-reports)
- [Google Play Developer API — reviews.reply](https://developers.google.com/android-publisher/api-ref/rest/v3/reviews/reply) / [Reply to Reviews](https://developers.google.com/android-publisher/reply-to-reviews)

Community / third-party sources, used only where official docs are silent and marked as indicative in-text: [bundle.social TikTok API approval](https://bundle.social/blog/tiktok-api-approval), [bundle.social Meta App Review timing](https://bundle.social/blog/meta-app-review-20-days), [PostPeer TikTok Content Posting API](https://www.postpeer.dev/blog/best-tiktok-posting-api), [Phyllo Instagram Reels API guide](https://www.getphyllo.com/post/a-complete-guide-to-the-instagram-reels-api), [Postproxy X API pricing 2026](https://postproxy.dev/blog/x-api-pricing-2026/), [SocialCrawl X API 2026](https://www.socialcrawl.dev/blog/x-twitter-api-2026), [Publer post callbacks](https://publer.com/docs/posting/create-posts/content-types/post-callbacks), [Buffer TikTok scheduling](https://buffer.com/resources/how-to-schedule-tiktok-posts/), [google-api-python-client issue #2753](https://github.com/googleapis/google-api-python-client/issues/2753).
