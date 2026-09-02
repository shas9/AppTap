# AppTap — Product Requirements Document

**Version:** 0.1 (founding draft) · **Date:** 3 September 2026 · **Status:** for founding-team review

**Companion documents (read first):**
- [`platform-api-feasibility.md`](./platform-api-feasibility.md) — what the five platform APIs actually permit. Every capability claim in this PRD derives from it.
- [`competitive-analysis.md`](./competitive-analysis.md) — why the scheduler is not the product.
- [`design-doc.md`](./design-doc.md) — the technical design: physical data model, state machines, the capability manifest as data, publish and retry mechanics.

**Interactive prototype:** <https://claude.ai/code/artifact/549422a6-0dec-4a8b-8887-cd95cb65a002>
Clickable five-screen mockup of the flows described in §6 — dashboard with app growth metrics and the "worth posting about" feed, app profile and connected accounts, the multi-platform composer with live per-platform character counters and the first-comment field (disabled on TikTok, with the reason), the content calendar, and post performance showing the 4-of-5 partial-failure case from §10. Populated with a plausible indie app (*Halftone*, a film photography logbook). The composer opens on the X tab at 548 / 280 characters to demonstrate the overflow problem in §9.

**Standing assumption, stated once:** solo/small founding team, pre-revenue, no existing code. Every scoping decision below optimises for *time-to-a-real-customer-posting*, not for feature completeness. Where a judgment call was made without evidence, it is marked **[ASSUMPTION]**.

---

## 1. Problem statement

Indie and early-stage app founders build good apps and then fail to tell anyone about them. The visible symptom is "I don't post consistently." The actual cause, in this segment, is not scheduling friction — schedulers are abundant, cheap and free — it is **content supply**: the founder sits down to post, has nothing prepared, doesn't know what would be interesting, and closes the tab.

Meanwhile the raw material for good posts is sitting in App Store Connect and Google Play Console, unread: a version that just shipped with real changes in the release notes, a five-star review that names a feature, a download milestone, a rating that just recovered. Nobody turns that into posts, because the tools that hold app data (Appfigures, AppFollow, App Radar) have no publishing surface, and the tools that publish (Buffer, Later, Publer) have no idea what the app is.

**AppTap closes that loop:** it reads what is happening to the app, drafts the post, adapts it per platform, and publishes on schedule.

**The scheduler is the delivery mechanism, not the product.** It is table stakes — required, priced at zero, and not defensible. See [`competitive-analysis.md`](./competitive-analysis.md) §1.

---

## 2. Target user

**Primary (paying):** the operator of 1–5 published mobile apps who is also the person doing the marketing. In practice: a solo founder or a 2–4 person studio. Owns App Store Connect / Play Console access. Has an audience of somewhere between zero and a few thousand. Budget: $20–80/month, sensitive.

**Design-partner segment:** indie app founders, reached through Indie Hackers, r/iOSProgramming, r/androiddev, Product Hunt, and app-dev Twitter/Bluesky. Lowest willingness to pay, fastest feedback, best launch narrative.

**Explicitly not the target:** social media agencies and multi-client managers. That market wants seats, permissions, white-labelling and approval workflows, is served by Metricool/Hootsuite/Ayrshare, and would consume the entire build.

### Personas

**Priya — solo iOS developer.** Ships a habit tracker. 3,000 downloads, 4.6★, $400 MRR. Posts to Instagram and X maybe twice a month, in bursts, usually right after a release, then goes quiet for six weeks. Owns a Mac, a Figma file of screenshots, and no marketing skill. Her blocker is the blank text box. *What she needs:* something that says "you shipped 2.4 yesterday, here's a post about it, here's the screenshot, want to publish it at 9am?"

**Marco — two-person studio.** Three apps, one growing. His co-founder designs; he does everything else. He already pays for Appfigures. He wants a calendar so posting stops being reactive, one place to reply to reviews across both stores, and to stop retyping the same caption five times with different lengths. *What he needs:* consistency infrastructure and per-app separation.

**Dana — app marketing agency, 40 client brands.** *Anti-persona.* If AppTap's roadmap starts bending toward Dana, the product has lost.

---

## 3. Goals and non-goals

### Goals
1. A founder connects their app store account and sees, within 5 minutes, at least three post ideas drawn from their real app data.
2. One composition produces correctly-formatted, correctly-sized posts on every connected platform, with per-platform text that respects each platform's limits and field structure.
3. Scheduled posts publish reliably and unattended, and when a platform fails the founder knows within minutes, knows why, and can fix it in one click.
4. A first comment (typically the link) posts automatically on every platform where the API permits it — **four of five; see §7**.
5. Store reviews land in one inbox and can be replied to without leaving AppTap.

### Non-goals (v1)
- **ASO keyword rank tracking.** Appfigures does 100 keywords for $44.99/month with real rank infrastructure. Do not compete; integrate or ignore.
- **Competitor / market intelligence.** Sensor Tower territory.
- **Paid acquisition, ad management, ad creative.**
- **Agency features:** client workspaces, approval chains, white-labelling, per-client permissions.
- **Facebook, LinkedIn, Pinterest, Bluesky, Mastodon, Reddit.** Five platforms, done properly, beats eleven done shallowly. (Bluesky is the cheapest future addition — no review, no fees — but it is not where app users are. **[ASSUMPTION]**)
- **Being a general-purpose social scheduler.** If AppTap works for someone with no app, that is a bug in positioning.
- **True install attribution in v1.** See §5 item 7.

---

## 4. Prioritised feature set (MoSCoW)

Priorities below fold in the Part C analysis; §5 gives the reasoning per feature.

### Must have (MVP)
| # | Feature | Notes |
|---|---|---|
| M1 | App Store Connect + Google Play connection, per app | The wedge substrate. No platform approval needed. |
| M2 | App profile + growth dashboard: downloads, ratings, rating trend, version history | Deliberately shallow. Not competing with Appfigures. |
| M3 | Store review inbox (both stores) + reply | The daily-open hook. |
| M4 | Draft generation from app moments: new release, notable review, milestone | **This is the product.** |
| M5 | Social account OAuth connection + health status | |
| M6 | Multi-platform composer: canonical post + per-platform variants, live character counters, per-platform field mapping, compose-time validation | Directly answers the founder's stated character-limit requirement. |
| M7 | AI caption adaptation: "shorten to fit 280", "rewrite for platform" | Makes M6 usable rather than merely correct. |
| M8 | Media asset library + per-platform transcoding/derivation | Not optional — Instagram is JPEG-only, TikTok needs pull-from-verified-domain. |
| M9 | Scheduling + unattended publish, per-platform adapters | |
| M10 | Auto first comment where API permits (YouTube, Instagram, Threads, X) | **Not TikTok.** |
| M11 | Partial-failure handling, pre-flight checks, retry, notifications | See §9. |
| M12 | Content calendar (month/week read view) | |
| M13 | Tracked short links with per-platform campaign tags + click counts | Cheap; preserves the attribution moat. |
| M14 | "Manual assist" fallback for any platform not yet approved or not API-supported | See §8.6. Ships permanently for TikTok comments. |

### Should have (v1.1)
| # | Feature |
|---|---|
| S1 | Cross-platform published-post analytics (views, likes, comments, shares) |
| S2 | Link-in-bio / app landing page — the compliant answer to TikTok's missing comment API and Instagram's non-clickable captions |
| S3 | Drag-and-drop calendar rescheduling |
| S4 | Auto-import App Store screenshots and preview videos into the asset library |
| S5 | Team seats and multi-app workspaces |
| S6 | Recurring / evergreen re-post queue |
| S7 | Store campaign-parameter join (App Store campaign links, Play install referrer) |

### Could have (later)
C1 install-level attribution · C2 A/B caption testing · C3 RSS/changelog auto-post · C4 ASO keyword tracking *or* an Appfigures/AppFollow import · C5 competitor rank tracking · C6 additional platforms · C7 press-kit generator

### Won't have
W1 agency/client management · W2 ad management · W3 DM/inbox management · W4 influencer discovery · W5 white-label

---

## 5. Feature analysis — build cost vs. value

Each verdict is MVP / v1.1 / Later, with the reasoning that produced it.

**1. App Store Connect + Google Play ingestion — MVP.**
Cost: medium (~2 weeks). ASC uses JWT auth with a `.p8` key and *asynchronous* analytics reports (request → poll → download gzip/CSV), so ingestion is a pipeline, not a REST call. Play needs a service account linked in Play Console. Onboarding friction is real and is the biggest activation risk in the product.
Value: highest in the product — it is the only thing no competitor in either band has. Scope MVP to: app metadata, version history + release notes, ratings, reviews, daily downloads. Not full analytics dimensionality.

**2. Store review monitoring + reply — MVP.**
Cost: low once (1) exists. Both stores expose it: Apple has full CRUD on [customer review responses](https://developer.apple.com/documentation/appstoreconnectapi/customer-review-responses); Google has [`reviews.reply`](https://developers.google.com/android-publisher/api-ref/rest/v3/reviews/reply) — **≤ 350 characters, production versions only, comment-bearing reviews only, ~7-day retrieval window, 200 GET/hour and 2,000 POST/day**.
Value: high, and structurally important — it is the reason a founder opens AppTap on a day they are not posting. Reviews are also the highest-quality raw material for post drafts. Build the Play 7-day window into the ingestion cadence (poll at least daily or reviews are lost permanently).

**3. ASO keyword tracking — Later, probably never.**
Cost: high and *ongoing* — rank tracking needs continuous store scraping or purchased data.
Value: real, but fully commoditised at $44.99/month. Building it converts AppTap from a differentiated product into a worse Appfigures. Offer an import instead.

**4. Release notes / changelog → post drafts — MVP.**
Cost: low. The data arrives free with (1); the transform is one LLM call with a good prompt and the app's screenshots attached.
Value: this is the single most demonstrable expression of the wedge. It is the demo. Ship it first.

**5. AI per-platform caption rewriting — MVP, tightly scoped.**
Cost: low-medium. LLM API + per-account usage caps.
Value: high, and it is what makes the character-limit problem *solved* rather than merely *surfaced*. A counter that turns red is a nag; a counter with a "shorten to 280" button is a feature. Scope to two operations — shorten-to-fit and adapt-tone-for-platform — plus draft generation. **Not** an AI content studio; that is a different product and everyone has one.

**6. Asset library — MVP (thin).**
Cost: near-zero *incremental*. AppTap must host media on public, signed, verified-domain URLs anyway because Instagram fetches media by URL and TikTok photo posts are `PULL_FROM_URL`-only from a domain verified in TikTok's portal. The library is a UI over storage that has to exist.
Value: medium alone, high in combination — reusing the same three demo videos across a month of posts is exactly what this user does.

**7. UTM / deep-link tracking and attribution — split.**
- **MVP:** AppTap-owned short links (`apptap.link/xyz`) with a per-post, per-platform campaign tag, a redirect that counts clicks, and correct routing to App Store / Play. Cost: low. Value: it is the only outcome signal in the MVP, and it lays the groundwork for the real moat.
- **v1.1:** join click data to App Store campaign links and Play install-referrer UTMs.
- **Later:** install-level attribution. Genuinely hard, partly outside AppTap's control, and not credible from a v1 team.

**8. Content calendar — MVP (read-only month/week), drag-drop v1.1.**
Cost: low. Value: medium, but it is table stakes — its absence reads as "unfinished." Do not gold-plate it.

**9. Unified post-performance analytics — v1.1.**
Cost: medium-high, and *unevenly* so. Instagram insights need extra scopes; TikTok read data needs its own API approval track; **X charges $0.005 per post read and $0.001 per owned read**, so polling five platforms hourly is a live financial decision, not a technical one.
Value: high for retention, near-zero for activation. MVP ships publish status + link clicks and says so plainly. Poll on view, cache aggressively, never on a timer.

**10. Competitor / rank tracking — Later.** Same argument as (3), weaker value.

**11. Link-in-bio / landing page — v1.1, with MVP groundwork.**
Cost: low.
Value: strategically higher than it looks, because it is the **compliant workaround for two verified platform limitations**: TikTok has no comment API at all, and Instagram captions are not clickable. "Link in bio" is the only legitimate answer on TikTok, and AppTap should own that page rather than sending users to Linktree. Build the redirect/short-link service in MVP (item 7) so this is a UI layer later.

### The ruthless MVP cut

**Ship the loop, not the pipes.**

MVP is: **connect your app → AppTap shows you what happened → AppTap drafts the post → you adjust per platform with live counters and one-click shortening → it schedules and publishes to every platform that is approved, and hands you a one-tap manual assist for the rest.**

**Defence (three sentences).** The platform approvals — Google audit, Meta App Review + Business Verification, TikTok audit — take 6–10 weeks with rejection rounds, so a scheduler-first MVP is a product that cannot demonstrate its core promise for two months and then demonstrates a promise eight competitors already keep more cheaply. The app-data-to-draft loop needs **zero** platform approval, is useful on day one, and is the only part of AppTap that Buffer and AppFollow cannot copy this quarter. Ship the loop while the approvals are in flight, and let publishing light up platform-by-platform as they land — which also produces four launch moments instead of one.

**Corollary that must not be softened:** if founders use the drafts but never connect a social account, the wedge is real and the pipes are optional. If they connect accounts but ignore the drafts, AppTap is a worse Buffer and the strategy needs rewriting, not more features. **§12's "drafted-post share" metric is the experiment.**

---

## 6. Core user flows

### 6.1 Onboarding
1. Sign up (email + magic link, or Google/Apple SSO). **[ASSUMPTION]** Organization is created implicitly; no team setup in MVP.
2. **"Connect your app"** — first and only required step. Two paths:
   - *App Store Connect:* guided flow — create an API key in ASC (Users & Access → Integrations), choose role, download the `.p8`, paste Key ID + Issuer ID, upload `.p8`. AppTap validates immediately and names the apps it found.
   - *Google Play:* guided flow — create a Google Cloud service account, grant it access in Play Console, upload the JSON key.
   Either alone is sufficient. **This step is the activation cliff; instrument every sub-step.**
3. AppTap ingests app metadata, versions, ratings, recent reviews, and 30 days of downloads (backfill runs in the background; the UI does not block on it).
4. **Immediate payoff, before any social connection:** dashboard renders with real numbers and **three generated post drafts** from the last release, the best recent review, and the nearest milestone.
5. Prompt (skippable) to connect social accounts.

**Design rule:** a user who connects a store account and nothing else must still find AppTap useful. That is what makes onboarding survive a 10-week approval backlog.

### 6.2 App profile
Per-app record: display name, icon, store URLs, category, one-paragraph positioning blurb, default hashtag set, default CTA link, brand voice note (fed to the draft generator), and default platform targets. Editable. One organization, many apps; every post belongs to exactly one app.

### 6.3 Connecting a social account
1. User picks a platform.
2. OAuth (PKCE where supported). Scopes requested per [`platform-api-feasibility.md`](./platform-api-feasibility.md):
   - YouTube: `youtube.upload` + `youtube.force-ssl` (the latter is required for comments)
   - Instagram (Instagram Login): `instagram_business_basic`, `instagram_business_content_publish`, `instagram_business_manage_comments`
   - TikTok: `video.publish`
   - Threads: `threads_basic`, `threads_content_publish`
   - X: `tweet.write`, `tweet.read`, `users.read`, **`offline.access`** — omit `offline.access` and no refresh token is issued at all, which turns a 2-hour access token into a permanent outage. Assert this in a test.
3. Post-connect capability probe: fetch account type and constraints and store them on the `SocialAccount`. For Instagram, record whether the account is Business or Creator (**Creator support for publishing is unverified — see §13 R3**). For TikTok, call `creator_info` and cache allowed privacy options, interaction settings and `max_video_post_duration_sec`.
4. Show a health badge: Connected / Expiring soon / Reconnect needed / Degraded (e.g. "YouTube uploads will be private until our audit completes").

### 6.4 Compose and schedule
1. User starts from a generated draft, from a calendar slot, or from a blank composer. App context is always attached.
2. Chooses media: video, single image, or image carousel — pulled from the asset library or uploaded. On upload, AppTap immediately derives per-platform renditions and reports any platform that cannot accept the asset **(e.g. "X accepts 4 of your 8 carousel images")**.
3. Writes the **canonical post**: body text, optional link, optional first comment, optional title.
4. Selects target platforms. Each gets a **variant tab** showing:
   - the mapped fields for that platform (YouTube: title / description / tags; TikTok photo: title / description; everyone else: one caption)
   - a live counter against the real limit (Threads 500, X 280, IG 2,200, TikTok video 2,200 / photo title 90, YouTube title 100 + description 5,000 bytes)
   - live media validation against that platform's specs
   - the first-comment field — **greyed out with an explanation on TikTok**
   - an "Adapt with AI" button
5. Picks a time (or "next available slot" from a simple per-platform cadence heuristic). **[ASSUMPTION]** No AI best-time optimisation in MVP; there is no engagement data to train on.
6. **Compose-time validation gate.** Any variant that fails hard validation blocks scheduling *for that platform only*, with an inline fix. The user may always schedule the passing platforms.
7. On save: variants, media renditions and the full publish plan are materialised. Nothing is computed for the first time at publish o'clock.

### 6.5 Publish
1. **T-15 min — pre-flight** per variant: token valid (refresh if not), media renditions present and reachable, caption still within limits, account rate-limit headroom (Threads `threads_publishing_limit`, TikTok `creator_info`), account still connected. A pre-flight failure notifies the user **before** the scheduled time — this is where most failures should be caught.
2. **T-0 — fan-out.** Each variant is an independent job. No variant waits on another.
3. Per-platform publish per the adapter contract (§8.4).
4. On success, store the platform post ID and permalink; enqueue the first-comment job as a child of that variant.
5. **First comment** fires only after the parent variant is `published` and has returned a post ID. Independent retry. If it fails, the post stays up.
6. Post-level status is derived: `scheduled` → `publishing` → `published` | `partially_published` | `failed`.

### 6.6 Failure and retry
Covered in full in §9.

### 6.7 Review reply
Review lands in the inbox with app, store, rating, version and text. Founder replies inline; AppTap enforces Google's 350-character cap on Play replies and warns that Play replies are public and editable-once. **[ASSUMPTION]** AI-suggested replies are v1.1, not MVP — a bad auto-reply on a public store listing is a worse failure than a slow one.

---

## 7. Per-platform capability matrix (build against this)

Derived from [`platform-api-feasibility.md`](./platform-api-feasibility.md); every cell is sourced there. **Encode this as data (`PlatformCapability`), not as `if` statements** — it changes without notice, and the composer, the validator and the adapters must all read the same copy.

| Capability | YouTube | Instagram | TikTok | Threads | X |
|---|---|---|---|---|---|
| Publish endpoint | `videos.insert` | `POST /{ig-user}/media` → `/media_publish` | `POST /v2/post/publish/video\|content/init/` | `POST /{user}/threads` → `/threads_publish` | `POST /2/tweets` |
| Unattended publish | ✅ | ✅ | ✅ *post-audit* | ✅ | ✅ |
| Text fields | title, description, tags | caption | video: title · photo: title + description | text | text |
| Text limits | 100 / 5,000 **bytes** / 500 tag chars | 2,200 · 30 # · 20 @ | 2,200 · photo 90 + 4,000 | **500** | **280** |
| Single image | ❌ | ✅ JPEG only | ✅ | ✅ JPEG/PNG | ✅ |
| Carousel | ❌ | ✅ 2–10, cropped to first item | ✅ ≤ 35 photos | ✅ 2–20 mixed | ⚠️ **max 4** |
| Video | ✅ ≤ 256 GB | ✅ 3 s–15 min, ≤ 300 MB | ✅ ≤ 10 min, ≤ 4 GB | ✅ ≤ 300 s, ≤ 1 GB | ✅ ≤ 20 min (125 min Premium) |
| Image constraints | n/a | **JPEG only**, ≤ 8 MB, 4:5–1.91:1, 320–1440 px, sRGB | JPEG/WebP, ≤ 1080p, ≤ 20 MB | ≤ 8 MB, ≤ 10:1, 320–1440 px | ≤ 5 MB |
| Media delivery | direct resumable upload | **public URL fetch** | chunked upload (video) · **`PULL_FROM_URL` from a TikTok-verified domain (photos)** | public URL fetch | chunked upload |
| **First comment** | ✅ `commentThreads.insert` | ✅ `POST /{media}/comments` | ❌ **no API exists** | ✅ `reply_to_id` | ✅ `in_reply_to_tweet_id` |
| Links in caption | ✅ not clickable without channel verification | ⚠️ not clickable | unverified | ✅ `link_attachment`, ≤ 5 links | ✅ clickable, **$0.20/post** |
| Publish cap | **100 uploads/day per API project** | 100/day per account | 6 req/min per token; audit-set daily cap | 250 posts + 1,000 replies/day | 100/15 min per user; 10,000/day per app |
| Access token TTL | ~1 h | 1 h | 24 h | 1 h | **2 h** |
| Refresh model | refresh token; dies at 6 mo idle, 7 d while app in Testing | long-lived 60 d, refresh when ≥ 24 h old | refresh token 365 d | long-lived 60 d, refresh when ≥ 24 h old | refresh only with `offline.access` |
| Approval gate | compliance audit, **no SLA**; **videos are private until it passes** | App Review + Business Verification, 2–6 wks | sandbox → audit; **`SELF_ONLY` + 5 users/24 h until passed** | Meta App Review | credits; no review gate located |
| Per-post platform cost | free | free | free | free | **$0.015, or $0.200 with a URL** |
| Special UX obligation | — | — | **mandated composer: creator nickname, `creator_info` pre-check, no default privacy value, interaction toggles reflecting creator settings, commercial-content disclosure, preview, status** | — | — |

---

## 8. Architecture

### 8.1 Shape
A single deployable web application (API + server-rendered or SPA front end) plus a worker pool, on PostgreSQL, Redis, and object storage behind a CDN. **[ASSUMPTION]** No language/framework decision is made here; a boring, single-process monolith with a separate worker binary is right for a team of one to three.

```
Browser ── API ──┬── Postgres  (system of record; also the job queue)
                 ├── Redis     (cache, per-account rate-limit token buckets, locks)
                 └── Object store + CDN  (media, signed URLs on a TikTok-verified domain)

Workers (same codebase, different entrypoint):
  scheduler      → claims due jobs
  publisher      → runs one PublishAttempt against one adapter
  transcoder     → FFmpeg; derives per-platform renditions
  ingestor       → App Store Connect / Google Play polling
  token-refresher→ keeps every SocialAccount live
  metrics-poller → v1.1, on-view and cached
```

### 8.2 Scheduled jobs
Do **not** create an OS-level cron entry or an in-memory timer per post. Use a `scheduled_jobs` table with a `run_at` timestamp, claimed by workers with `SELECT ... FOR UPDATE SKIP LOCKED` inside a transaction, with a visibility timeout and an attempt counter. Reasons: it survives deploys and restarts, it is inspectable in SQL when a founder asks "why didn't my post go out," and it needs no extra infrastructure. Redis-backed queues are acceptable but must not be the system of record for scheduling.

**Every job carries an idempotency key** (`variant_id + attempt_epoch`). Publishing is the one operation where an at-least-once queue is unacceptable without one.

### 8.3 Media pipeline
1. Upload to object storage; compute a content hash; store as a `MediaAsset`.
2. **At compose time — not publish time —** derive a `MediaRendition` per targeted platform: Instagram → JPEG within 4:5–1.91:1, ≤ 8 MB, sRGB, and H.264 MP4 ≤ 300 MB with moov atom at the front; TikTok → MP4/H.264 within its bounds; X → ≤ 5 MB images; Threads → JPEG/PNG ≤ 8 MB; YouTube → pass-through. Deriving early means spec violations surface while the user is looking at the composer.
3. Serve renditions from a **dedicated media domain verified with TikTok** (domain or URL-prefix ownership), via signed URLs with a TTL comfortably exceeding Instagram's 24-hour container lifetime — 48 hours is the floor.
4. Retain renditions until the post is published plus a grace window, then garbage-collect.

**Do not skip transcoding to save time.** "Instagram accepts JPEG only" plus "founders upload PNG screenshots" makes conversion a day-one requirement, not an optimisation.

### 8.4 Publish adapters
One interface, five implementations, no `if platform == ...` outside the adapter layer:

```
interface PlatformAdapter {
  capabilities()                    -> PlatformCapability   // data, versioned
  validate(variant, renditions)     -> ValidationResult     // used by the composer AND pre-flight
  preflight(account, variant)       -> PreflightResult      // token, quota, account state
  publish(account, variant, media)  -> { platformPostId, permalink }
  comment(account, platformPostId, text) -> { platformCommentId } | Unsupported
  fetchMetrics(account, platformPostId)  -> Metrics         // v1.1
}
```

`comment()` returning `Unsupported` — TikTok — is a first-class, tested case, surfaced in the composer, not an exception thrown at publish time.

Adapters are the only code that knows about platform HTTP. Rate limiting is enforced by a per-account token bucket in Redis in front of every adapter call (TikTok's 6 req/min is the binding one).

### 8.5 Token lifecycle
A `token-refresher` worker sweeps every `SocialAccount` and refreshes according to each platform's own rules, because they differ more than they look:

| Platform | Strategy |
|---|---|
| X | Access token 2 h → refresh on demand **and** preemptively 10 min before any scheduled publish. Verify `offline.access` was granted at connect time. |
| TikTok | Access token 24 h → refresh at ~20 h. Refresh token 365 d. |
| Instagram / Threads | Long-lived 60 d → refresh **weekly**, not at day 59. Refresh requires the token be ≥ 24 h old and not expired; miss the 60-day window and it is permanently dead. A weekly cadence gives ~8 chances to recover from an outage. |
| YouTube | Refresh on demand. Watch for the 6-month idle rule, the **7-day expiry while the OAuth app is in Testing status** (this *will* bite in development), and the 100-refresh-tokens-per-client-per-account cap. |

Tokens are encrypted at rest with envelope encryption; the data-encryption key never leaves the KMS. Refresh failure sets the account to `reconnect_required`, notifies the user immediately, and marks every future scheduled variant on that account as `blocked` **in advance** — not silently at publish time.

### 8.6 Manual-assist fallback
For any platform that is not yet approved, or that lacks the API capability (TikTok comments, permanently), AppTap generates a **publish task**: at the scheduled moment it sends a push/email with the caption pre-copied to clipboard, the media downloadable, and a deep link into the native app. The user pastes and posts. AppTap records it as `manually_published` when the user confirms.

This is not a consolation prize. It is what makes AppTap useful during the 10-week approval window, it is the permanent honest answer for TikTok first comments, and it means a platform's API breaking degrades the product instead of breaking it.

---

## 9. Character limits, variants, and validation

### Model
One **canonical post** per `Post`: body, optional link, optional first comment, optional title, media set. Each targeted platform gets a **`PostVariant`** that *inherits* every canonical field until the user edits it, at which point that field alone is marked overridden and stops tracking the canonical. Editing the canonical body updates every non-overridden variant live. This is the behaviour users expect and the one that avoids the two classic failure modes: silently discarding a user's per-platform edit, and silently failing to propagate a canonical fix.

### Field mapping
Platforms do not merely differ in *length*, they differ in *shape*. YouTube has title + description + tags; a TikTok photo post has title + description; everyone else has one caption. The `PlatformCapability` record therefore defines a field map, and the composer renders whatever fields that platform declares. A single "caption with a per-platform max length" model is wrong and will need rewriting the first time YouTube is added properly.

### Counting
Count what the platform counts: Threads counts characters but treats emoji as UTF-8 bytes; YouTube's description limit is **5,000 bytes**, not characters; TikTok counts **UTF-16 runes**; X applies t.co shortening to URLs. A naive `string.length` counter will be wrong for exactly the posts users care about — the ones with emoji and links. Encode the counting function per platform and unit-test it against known edge cases.

### What happens when a caption is too long
1. Counter turns amber at 90%, red past 100%, and always shows the overflow amount ("X: 312 / 280 — 32 over").
2. Scheduling is **blocked for that platform only**. Every other platform proceeds.
3. Three offered actions: **Shorten with AI** (returns 2–3 options at length, preserving the link and the CTA), **Edit manually**, **Skip X for this post**.
4. **Never auto-truncate.** A caption cut mid-word on a public post is a worse outcome than an unposted variant, and it destroys trust in every future automated publish.
5. **Re-validate at pre-flight.** Limits change, and a variant may have been drafted weeks earlier.

Media validation follows exactly the same pattern: an 8-image carousel targeting X shows "X supports 4 — use the first 4, or skip X," decided by the user at compose time, never silently at publish time.

---

## 10. Error handling and partial-failure semantics

**The governing decision: publishing is per-platform independent, gated by a pre-flight, and never rolled back.**

All-or-nothing publishing was considered and rejected. There is no distributed transaction across five APIs, deleting a successfully published post to "roll back" is a visible, embarrassing action taken on the user's behalf, and the most common failures are knowable *before* T-0. So: catch what you can early, publish independently, and be precise about what happened.

### Error classification
| Class | Examples | Behaviour |
|---|---|---|
| **Transient** | 5xx, timeout, 429 | Exponential backoff with jitter, honour `Retry-After`, max ~6 attempts over ~30 min, then `failed`. |
| **Terminal-fixable** | expired/revoked token, caption over limit, media spec violation, unsupported account type | **No retry.** Surface the specific fix and a one-click path (Reconnect / Shorten / Re-encode / Skip). |
| **Terminal-permanent** | content policy rejection, duplicate content, account suspended | No retry. Show the platform's own message verbatim — never paraphrase a moderation decision. |
| **Ambiguous** | connection dropped after the request was sent | **Never blind-retry.** Query the platform for a post matching the idempotency marker within a time window. If it exists, mark published. If it cannot be determined, set `needs_review` and ask the user. **Double-posting is a worse failure than not posting.** |

### The 4-of-5 scenario, concretely
Post targets all five at 09:00. Instagram fails at 09:00:12 with an expired token that pre-flight could not refresh.

- 09:00:00 — YouTube, TikTok, Threads and X publish. First comments fire on YouTube, Threads and X. TikTok's comment is created as a manual-assist task.
- 09:00:12 — Instagram's variant enters `failed`, classified terminal-fixable.
- 09:00:15 — **one** notification (not five): *"Published to 4 of 5. Instagram needs reconnecting."* Deep link to a screen with a Reconnect button.
- Post status = `partially_published`. The post card shows four green rows and one amber row with the reason and the fix.
- After reconnecting, **Retry Instagram** re-runs pre-flight and publishes only that variant, reusing the same media and caption. Successful platforms are untouched.
- The other four posts stay up. AppTap never deletes a published post to enforce consistency.

### Late publishing
If a variant is still unpublished more than **60 minutes** past its scheduled time, do not publish it silently. Move it to `expired_needs_decision` and ask: publish now, reschedule, or cancel. Auto-posting launch content six hours late is a real harm, and it is the kind of harm that ends a trial.

### Notification policy
One notification per post per outcome, aggregated. Successes are silent by default (digest only). Pre-flight warnings arrive *before* the scheduled time, which is the whole point of having a pre-flight.

---

## 11. Data model (entity level)

```
Organization ──< Membership >── User
Organization ──< App
Organization ──< SocialAccount
Organization ──< MediaAsset
Organization ──< Subscription

App ──< StoreConnection      (APP_STORE | GOOGLE_PLAY; encrypted credentials; sync state)
App ──< AppVersion           (version, release notes, released_at, store)
App ──< AppMetricSnapshot    (date, store, downloads, rating_avg, rating_count, [conversion])
App ──< StoreReview ──o StoreReviewReply
App ──< ContentMoment        (NEW_RELEASE | NOTABLE_REVIEW | MILESTONE | RANK_CHANGE; payload; consumed_at)
App ──< Post

SocialAccount   (platform, external_account_id, handle, account_type, capability_snapshot,
                 encrypted access/refresh tokens, expires_at, health, last_refreshed_at)

MediaAsset ──< MediaRendition   (platform, spec_version, storage_key, width/height/duration/bytes,
                                 signed_url, url_expires_at, status)

Post            (app_id, author_id, canonical_body, canonical_link, canonical_first_comment,
                 canonical_title, source_moment_id, scheduled_at, timezone, status)
Post ──< PostVariant
PostVariant     (post_id, social_account_id, platform, fields JSONB, overridden_fields[],
                 first_comment_text, media_selection[], validation_state, status,
                 platform_post_id, permalink, published_at)
PostVariant ──< PublishAttempt   (attempt_no, idempotency_key, started_at, finished_at,
                                  outcome, error_class, error_code, provider_message, request_id)
PostVariant ──o CommentJob       (text, status, platform_comment_id, attempts, depends on parent published)

ScheduledJob    (kind: PREFLIGHT|PUBLISH|COMMENT|INGEST|REFRESH|TRANSCODE,
                 subject_type, subject_id, run_at, claimed_at, claimed_by, attempts, last_error)

TrackedLink ──< LinkClick        (per post+platform campaign tag, destination, click ts, ua, geo)
PlatformCapability               (platform, version, limits/fields/media specs as data, effective_from)
AuditLog                         (actor, action, subject, before/after, ts)
```

Notes that matter:
- `PublishAttempt` is separate from `PostVariant` so a variant that succeeded on attempt 3 still shows attempts 1 and 2 with their errors. Support requests are unanswerable without this.
- `capability_snapshot` on `SocialAccount` caches per-account facts the platform told us (TikTok's allowed privacy levels and max duration; Instagram's account type) — these are *account* properties, not *platform* properties, and the composer needs both.
- `ContentMoment` is what turns store data into drafts. It is the wedge, in one table.
- `PlatformCapability` is versioned data so a limit change is a row, not a deploy.

---

## 12. Pricing hypothesis

**[ASSUMPTION — untested.]** Anchors: Buffer $5/channel, Appfigures $9.99, SocialBee $29. AppTap must sit above pure schedulers (it does more) and below AppFollow (it does less analytics).

| Plan | Price | Contents |
|---|---|---|
| **Free** | $0 | 1 app, store connect, dashboard, review inbox (read-only), 5 generated drafts/month, 2 social channels, 10 scheduled posts/month. **X excluded** — it has real per-post cost. |
| **Indie** | **$29/mo** | 1 app, 5 channels, unlimited scheduling, unlimited drafts (fair use), review replies, first comment, tracked links, AI adaptation. X included under a fair-use cap (~300 posts/month ≈ $6.45 at $0.215/post-with-link). |
| **Studio** | **$79/mo** | 3 apps, 15 channels, 2 seats, post analytics, link-in-bio, priority publishing. |
| Add-ons | +$19/app · +$9/seat | |

**Margin warning, carried forward from research:** X bills $0.015 per post and **$0.200 for a post containing a URL**. A daily X poster with a link in the first comment costs ~$6.45/month — roughly 22% of the Indie plan, for one of five platforms. Track per-account platform COGS from day one, cap X volume explicitly, and consider surfacing "X: 47 of 300 posts used this month" rather than absorbing an unbounded cost. If usage runs hot, X becomes a paid add-on rather than a plan inclusion.

---

## 13. Success metrics

**Activation**
- % of signups that complete a store connection — **target 60%**. This is the biggest funnel risk (`.p8` files and service accounts are hostile).
- % of signups that publish or manually-assist a post within 7 days — **target 40%**.

**The wedge metric (the one that decides the strategy)**
- **% of published posts that originated from an AppTap-generated draft — target ≥ 40%.** Below ~25% sustained, the content loop is not working and AppTap is competing as a scheduler, which [`competitive-analysis.md`](./competitive-analysis.md) says it loses. Review this number before adding any feature.

**Engagement**
- Median posts published per active app per week — target ≥ 3.
- Weekly active apps / total connected apps.
- Store-review replies sent per active app per week (the non-posting reason to open the product).

**Reliability**
- % of scheduled variants published within 5 minutes of target — **target ≥ 99%**.
- Partial-failure rate per post — target < 2%.
- Mean time from token failure to user reconnect.
- **Double-post incidents — target 0.** Any occurrence is a P0.

**Business**
- Free → paid conversion; 3-month logo retention (target ≥ 70%).
- **Platform COGS per paying account** — target < 15% of ARPU, X-dominated.

---

## 14. Roadmap

**Phase 0 — Week 1 (do this before writing product code).**
Register the developer accounts and **submit every approval on day one**: Google Cloud project + YouTube audit request, Meta app + Business Verification + App Review for Instagram and Threads, TikTok sandbox app then audit, X developer account + credits. Verify the media domain with TikTok. Spike each adapter against sandbox. **Approvals are the critical path; nothing else is.**

**Phase 1 — MVP, weeks 1–10 (parallel to approvals).** M1–M14. Publishing goes live per platform as each approval lands; every unapproved platform ships as manual assist. Target: 10 design partners with a connected store account and at least one published post.

**Phase 2 — v1.1, weeks 11–18.** Remaining platforms live. S1 cross-platform analytics, S2 link-in-bio, S3 drag-drop calendar, S4 screenshot import, S5 seats, S7 store campaign join. Launch publicly when at least four platforms publish natively.

**Phase 3 — quarter 2+.** C1 install attribution (the durable moat), C2 A/B captions, C3 RSS/changelog auto-post, C6 additional platforms if and only if customers ask by name.

**Kill criteria — decide these now, not later.** If by end of Phase 1: store-connection completion is below 40%, or the drafted-post share is below 25%, or fewer than 3 of 10 design partners publish in week 4 — stop and reconsider the concept before building Phase 2. Write these down where they cannot be quietly renegotiated.

---

## 15. Risks and open questions

Every Red/Yellow finding and every unverified item from [`platform-api-feasibility.md`](./platform-api-feasibility.md), carried forward.

### Red
| # | Risk | Impact | Response |
|---|---|---|---|
| R1 | **TikTok has no comment-posting API.** The first-comment feature cannot work there. | Core promise is 4/5, not 5/5. | Ship manual assist; own the link-in-bio page; never claim "all platforms" in any copy. Publer has the same gap — it is a platform limit, not a build gap. |

### Yellow — product-shaping
| # | Risk | Impact | Response |
|---|---|---|---|
| R2 | **YouTube forces videos private until the project passes Google's audit; no published SLA.** | The core promise silently produces invisible videos. | Submit week 1. Show a persistent "uploads are private until our audit clears" banner on the YouTube account. Do not launch YouTube publicly before it clears. |
| R3 | **Instagram Creator-account publishing support is unverified** — third-party sources say Reels publishing is Business-only. | Could exclude a large share of indie founders. | **Test with a real Creator account in week 1.** If confirmed, detect account type at connect time and tell the user to switch, with instructions. |
| R4 | **TikTok forces `SELF_ONLY` + private accounts + 5 users/24 h until audited**, and mandates a specific composer UX. | TikTok is unusable pre-audit and needs bespoke composer work post-audit. | Budget the TikTok panel as its own work item. Sequence TikTok last. |
| R5 | **X has no free tier and bills per post; a post with a URL costs $0.200.** | Direct, unbounded COGS on the plan's cheapest tier. | Meter per account, cap by plan, display usage, revisit as an add-on. Consider putting the link in the post body (one call, same $0.20) rather than a second call. |
| R6 | **YouTube allows 100 `videos.insert`/day per API *project*** — not per user. | Hard ceiling around ~100 daily-posting YouTube customers. | Monitor from day one; request quota increase via audit well before it binds. |
| R7 | **Meta App Review + Business Verification: 2–6 weeks with rejection rounds.** | Delays Instagram and Threads together. | Submit week 1; prepare the screencast and use-case narrative carefully; expect one rejection. |
| R8 | **Instagram is JPEG-only and fetches media from a public URL; TikTok photos are `PULL_FROM_URL` from a TikTok-verified domain.** | Transcoding + signed public CDN are day-one requirements. | Built into §8.3. Do not defer. |
| R9 | **Token expiry is unforgiving on Meta** — 60 days unrefreshed is permanent death. | Silent mass disconnection if the refresher stops. | Weekly refresh cadence, alerting on refresher lag, user-visible health badges. |
| R10 | **Onboarding requires `.p8` keys and GCP service accounts.** | The largest activation risk in the product, ahead of anything platform-related. | Guided flows with screenshots; instrument every sub-step; consider an OAuth path if Apple/Google offer one. |

### Strategic
| # | Risk | Response |
|---|---|---|
| R11 | **The scheduling layer is commoditised** (Buffer $5/channel, Postiz open-source). | Do not compete there. The wedge is the app-data loop; the drafted-post metric (§13) is the test. |
| R12 | **Incumbents are one integration away** — AppFollow already ships an MCP/AI toolkit over app data; Buffer already has 11 approved platforms and an AI assistant. | Move fast on the loop; build toward attribution, which is far harder to bolt on. |
| R13 | **Buying publishing infrastructure is unaffordable** — Ayrshare ≈ $20/customer/month against a $29 price. | Build direct adapters. Confirmed by arithmetic, not preference. |
| R14 | **The target segment may not pay.** The empty intersection may be an unserved need or a dead market. | Kill criteria in §14. Design the pricing ladder so a 3-app studio is the paying segment. |

### Open questions (must be answered before or during Phase 0)
1. Does Instagram content publishing — especially Reels — work on **Creator** accounts? *(blocks onboarding copy)*
2. What is X's **refresh-token lifetime and rotation** behaviour? *(a 2-hour access token with a broken refresh is fatal to a scheduler)*
3. Is X's free tier genuinely gone for new developers, and what is the true X Premium character ceiling — 4,000 or 25,000? *(pricing + composer counter)*
4. What is YouTube's audit **turnaround**, and does passing it lift both the private-video restriction and the 100/day cap? *(launch sequencing)*
5. Are URLs in **YouTube comments** clickable on an unverified channel? *(if not, the first-comment feature delivers nothing on YouTube for exactly our users)*
6. What are Instagram's and YouTube's **comment character limits**? *(undocumented)*
7. Will TikTok accept **scheduled, non-user-present publishing** in writing? Buffer and Later do it as Marketing Partners, but the guideline text says "expressly consented to the upload." Ask explicitly in the audit application.
8. What do **X's automation rules** say about cross-posted or substantially similar content? The page was not fetchable during research, and cross-posting is literally the product.
9. Does Meta review a **multi-tenant scheduler** differently from a single-brand tool? *(affects the App Review narrative)*
10. Do App Store Connect **analytics report latency and granularity** support a dashboard that feels live, or is a 1–2 day lag inherent? *(sets dashboard expectations)*

---

*Every capability claim in this document traces to a cited source in [`platform-api-feasibility.md`](./platform-api-feasibility.md), retrieved 3 September 2026. Platform terms in this space change without notice — re-verify before implementation, and treat anything marked unverified as unverified.*
