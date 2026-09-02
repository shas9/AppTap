# AppTap — Technical Design Document

**Version:** 0.1 · **Date:** 3 September 2026 · **Status:** draft for founding-team review

**Reads with:**
- [`PRD.md`](./PRD.md) — what we are building and why. This document does not restate product rationale.
- [`platform-api-feasibility.md`](./platform-api-feasibility.md) — the source of every constant in §6. All figures retrieved 3 September 2026.
- [`competitive-analysis.md`](./competitive-analysis.md) — why the publishing layer is built rather than bought.

**Interactive prototype:** <https://claude.ai/code/artifact/549422a6-0dec-4a8b-8887-cd95cb65a002> — clickable five-screen mockup (dashboard, composer with live per-platform character counters, connected accounts, calendar, post performance with a partial-failure case). It is the visual reference for every screen named below.

**Scope of this document.** Everything an engineer needs to start: component boundaries, physical data model, state machines, the platform capability manifest as data, publish and retry mechanics, media pipeline targets, token lifecycle, security, and observability. **Out of scope:** language/framework choice, hosting provider, CI, and UI component library. Those are deliberately unfixed — see §14.

Anything marked **[UNVERIFIED]** is carried forward from the feasibility report and must be resolved before the relevant adapter is written.

---

## 1. Design principles

These decide arguments later, so they are stated first.

1. **Platform differences live in data, not code.** The capability manifest (§6) is a versioned row set. Composer validation, pre-flight and adapters all read the same copy. A limit change is a migration, not a deploy of five files.
2. **Nothing is computed for the first time at publish o'clock.** Variants, media renditions and the publish plan are materialised at schedule time. T-0 does I/O only.
3. **Publishing is per-platform independent and never rolled back.** No distributed transaction, no compensating delete of a live post.
4. **Double-posting is worse than not posting.** Every publish call carries an idempotency key; ambiguous outcomes stop and ask rather than retry.
5. **Postgres is the system of record, including for the queue.** A founder asking "why didn't my post go out" must be answerable with one SQL query.
6. **Degrade, don't fail.** A platform without an approval, or without a capability, falls back to manual assist rather than disappearing.

---

## 2. System context

```
                        ┌──────────────────────────────────────────┐
   Founder ── HTTPS ───▶ │  AppTap web + API  (single deployable)   │
                        └───────────────┬──────────────────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
  ┌───────────┐                 ┌──────────────┐               ┌────────────────┐
  │ Postgres  │                 │    Redis     │               │ Object store   │
  │ records + │                 │ cache, rate  │               │  + CDN on a    │
  │ job queue │                 │ buckets,     │               │ TikTok-verified│
  └───────────┘                 │ advisory     │               │ media domain   │
                                │ locks        │               └────────────────┘
                                └──────────────┘
        ▲
        │  workers (same image, different entrypoint)
   ┌────┴──────┬────────────┬────────────┬──────────────┬───────────────┐
   │ scheduler │ publisher  │ transcoder │  ingestor    │ token-refresher│
   └───────────┴─────┬──────┴─────┬──────┴──────┬───────┴───────┬───────┘
                     │            │             │               │
             ┌───────▼────────────▼─────────────▼───────────────▼────────┐
             │  YouTube Data v3 · Instagram Graph · TikTok Content       │
             │  Posting · Threads · X v2 · App Store Connect · Google    │
             │  Play Developer · LLM provider                            │
             └──────────────────────────────────────────────────────────┘
```

**Why one deployable.** A team of one to three cannot operate five services. Workers are the same image with a different entrypoint so there is one build, one migration path, one log format. Split only when a worker class needs independent scaling — `transcoder` is the first candidate, because FFmpeg is CPU-bound and everything else is I/O-bound.

---

## 3. Components

| Component | Responsibility | Does **not** |
|---|---|---|
| **web/api** | HTTP, auth, composer validation, calendar, review inbox, OAuth callbacks | call platform publish endpoints |
| **scheduler** | Claims due `scheduled_jobs`, enqueues to the right worker class, enforces visibility timeouts, reaps orphans | know anything about platforms |
| **publisher** | Runs exactly one `PublishAttempt` against exactly one adapter | fan out; each variant is its own job |
| **transcoder** | FFmpeg/ImageMagick; derives `media_renditions` per platform target (§8) | decide which platforms a post targets |
| **ingestor** | App Store Connect and Google Play polling, review sync, metric snapshots, `content_moments` generation | write posts |
| **token-refresher** | Sweeps `social_accounts`, refreshes per §9, marks `reconnect_required` | publish |
| **metrics-poller** *(v1.1)* | On-view, cached post metrics | poll on a timer — X charges per read |
| **adapters/** | The only code that knows platform HTTP. One module per platform behind §7's interface | contain product logic |

---

## 4. Physical data model

Postgres 16. All ids are UUIDv7 (time-sortable, index-friendly). All timestamps `timestamptz`, stored UTC. `created_at`/`updated_at` on every table, omitted below for brevity.

### 4.1 Identity and app

```sql
organizations (
  id uuid pk, name text not null, plan text not null default 'free',
  x_post_allowance int not null default 0,       -- monthly, plan-derived
  x_posts_used_this_period int not null default 0,
  period_started_at timestamptz not null
)

users (id uuid pk, email citext unique not null, name text, last_seen_at timestamptz)

memberships (
  id uuid pk, org_id uuid fk→organizations, user_id uuid fk→users,
  role text not null check (role in ('owner','member')),
  unique (org_id, user_id)
)

apps (
  id uuid pk, org_id uuid fk→organizations,
  name text not null, icon_url text,
  bundle_id text, apple_app_id text, play_package text,
  category text, positioning text, voice_note text,
  default_hashtags text[], default_link text, default_platforms text[],
  timezone text not null default 'UTC',
  unique (org_id, coalesce(apple_app_id,''), coalesce(play_package,''))
)

store_connections (
  id uuid pk, app_id uuid fk→apps,
  store text not null check (store in ('app_store','google_play')),
  credential_ref text not null,          -- KMS envelope handle, never the secret
  key_id text, issuer_id text,           -- App Store Connect
  service_account_email text,            -- Google Play
  status text not null default 'active'  -- active | invalid | revoked
    check (status in ('active','invalid','revoked')),
  last_sync_at timestamptz, last_error text,
  unique (app_id, store)
)
```

### 4.2 App data ingested from the stores

```sql
app_versions (
  id uuid pk, app_id uuid fk→apps, store text not null,
  version text not null, release_notes text, released_at timestamptz,
  unique (app_id, store, version)
)

app_metric_snapshots (
  id uuid pk, app_id uuid fk→apps, store text not null,
  metric_date date not null,
  downloads int, impressions int, product_page_views int,
  conversion_rate numeric(6,4),
  rating_avg numeric(3,2), rating_count int,
  is_partial bool not null default false,   -- ASC reports lag 1–2 days
  unique (app_id, store, metric_date)
)
-- index: (app_id, store, metric_date desc)

store_reviews (
  id uuid pk, app_id uuid fk→apps, store text not null,
  external_id text not null,
  rating smallint not null check (rating between 1 and 5),
  title text, body text, author text, locale text, app_version text,
  reviewed_at timestamptz not null,
  reply_state text not null default 'none'
    check (reply_state in ('none','drafted','sent','expired')),
  unique (app_id, store, external_id)
)
-- index: (app_id, reviewed_at desc) where reply_state = 'none'

store_review_replies (
  id uuid pk, review_id uuid fk→store_reviews unique,
  body text not null, sent_at timestamptz, external_id text,
  status text not null check (status in ('queued','sent','failed')),
  error text
)

content_moments (
  id uuid pk, app_id uuid fk→apps,
  kind text not null check (kind in
    ('new_release','notable_review','download_milestone','rating_recovery','keyword_rank')),
  payload jsonb not null,          -- the facts the draft generator reads
  score numeric(5,2) not null,     -- ranking for the dashboard feed
  occurred_at timestamptz not null,
  dismissed_at timestamptz, consumed_by_post_id uuid,
  unique (app_id, kind, (payload->>'dedupe_key'))
)
```

`content_moments.payload` shapes, so the draft generator has a contract rather than a guess:

```jsonc
// new_release
{ "dedupe_key":"app_store:2.4.0", "version":"2.4.0", "store":"app_store",
  "release_notes":"Push/pull development calculator · 12 new film stocks · EXIF export fixed on iOS 26",
  "released_at":"2026-09-01T00:00:00Z" }

// notable_review
{ "dedupe_key":"app_store:rev_88213", "review_id":"…", "rating":5, "locale":"en-US",
  "quote":"The push/pull calculator saved a roll of Tri-X I'd have binned.",
  "mentions_feature":"push/pull calculator" }

// download_milestone
{ "dedupe_key":"lifetime:8000", "threshold":8000, "actual":8412,
  "split":{"ios":0.62,"android":0.38}, "months_to_reach":14 }
```

### 4.3 Social accounts

```sql
social_accounts (
  id uuid pk, org_id uuid fk→organizations,
  platform text not null check (platform in ('youtube','instagram','tiktok','threads','x')),
  external_account_id text not null,
  handle text, display_name text, avatar_url text,
  account_type text,                    -- 'business' | 'creator' | 'channel' | null
  scopes text[] not null,
  access_token_ref text not null,       -- KMS envelope handle
  refresh_token_ref text,
  access_expires_at timestamptz, refresh_expires_at timestamptz,
  last_refreshed_at timestamptz,
  health text not null default 'healthy'
    check (health in ('healthy','expiring','reconnect_required','degraded')),
  degraded_reason text,                 -- 'awaiting_youtube_audit', 'tiktok_unaudited', …
  capability_snapshot jsonb not null default '{}',
  unique (org_id, platform, external_account_id)
)
-- index: (health, access_expires_at) for the refresher sweep
```

`capability_snapshot` holds facts the *account* told us, as opposed to facts about the *platform* (§6):

```jsonc
// tiktok — from creator_info, refreshed before every publish
{ "privacy_level_options":["PUBLIC_TO_EVERYONE","MUTUAL_FOLLOW_FRIENDS","SELF_ONLY"],
  "comment_disabled":false, "duet_disabled":false, "stitch_disabled":true,
  "max_video_post_duration_sec":300, "creator_nickname":"halftoneapp" }

// instagram
{ "account_type":"BUSINESS", "linked_page":null, "publishing_limit_remaining":97 }

// x
{ "premium":false, "text_limit":280 }
```

### 4.4 Media

```sql
media_assets (
  id uuid pk, org_id uuid fk→organizations, app_id uuid fk→apps null,
  kind text not null check (kind in ('image','video')),
  filename text not null, content_hash text not null,
  storage_key text not null, bytes bigint not null,
  width int, height int, duration_ms int,
  mime text not null, source text not null default 'upload'
    check (source in ('upload','store_screenshot','generated')),
  unique (org_id, content_hash)
)

media_renditions (
  id uuid pk, asset_id uuid fk→media_assets,
  platform text not null, profile text not null,   -- e.g. 'ig_feed_4x5', 'tt_video_1080x1920'
  spec_version int not null,                       -- matches platform_capabilities.version
  storage_key text not null, mime text not null, bytes bigint not null,
  width int, height int, duration_ms int,
  status text not null check (status in ('pending','ready','failed')),
  error text,
  public_url text, url_expires_at timestamptz,     -- signed, ≥48h; IG containers live 24h
  unique (asset_id, platform, profile, spec_version)
)
```

### 4.5 Posts, variants, attempts

```sql
posts (
  id uuid pk, app_id uuid fk→apps, author_id uuid fk→users,
  source_moment_id uuid fk→content_moments null,
  origin text not null check (origin in ('generated','blank','calendar_slot')),
  canonical_title text, canonical_body text not null default '',
  canonical_link text, canonical_first_comment text,
  media_asset_ids uuid[] not null default '{}',
  scheduled_at timestamptz, timezone text not null,
  status text not null default 'draft',
  unique_key text                                   -- optional client idempotency
)

post_variants (
  id uuid pk, post_id uuid fk→posts, social_account_id uuid fk→social_accounts,
  platform text not null,
  fields jsonb not null default '{}',       -- platform field map, see §6.2
  overridden_fields text[] not null default '{}',
  first_comment_text text,
  media_selection uuid[] not null default '{}',     -- ordered subset of the post's assets
  validation jsonb not null default '{}',           -- last ValidationResult
  status text not null default 'pending',
  platform_post_id text, permalink text, published_at timestamptz,
  skipped_reason text,
  unique (post_id, social_account_id)
)
-- index: (status, post_id)

publish_attempts (
  id uuid pk, variant_id uuid fk→post_variants,
  attempt_no int not null,
  idempotency_key text not null,
  started_at timestamptz not null, finished_at timestamptz,
  outcome text not null check (outcome in ('success','transient','terminal_fixable',
                                           'terminal_permanent','ambiguous')),
  error_code text, provider_message text, provider_request_id text,
  http_status int, latency_ms int,
  unique (variant_id, attempt_no)
)

comment_jobs (
  id uuid pk, variant_id uuid fk→post_variants unique,
  text text not null,
  status text not null default 'blocked'
    check (status in ('blocked','pending','published','failed','unsupported','manual')),
  platform_comment_id text, attempts int not null default 0, last_error text
)

tracked_links (
  id uuid pk, post_id uuid fk→posts, platform text not null,
  slug text unique not null,                 -- apptap.link/<slug>
  destination text not null, campaign text not null,
  unique (post_id, platform)
)

link_clicks (
  id bigserial pk, tracked_link_id uuid fk→tracked_links,
  clicked_at timestamptz not null, ua_family text, country char(2),
  referrer text, is_bot bool not null default false
)
-- partition by month; index (tracked_link_id, clicked_at)
```

### 4.6 Jobs

```sql
scheduled_jobs (
  id uuid pk,
  kind text not null check (kind in
    ('preflight','publish','comment','transcode','ingest','refresh_token','poll_metrics')),
  subject_type text not null, subject_id uuid not null,
  run_at timestamptz not null,
  priority smallint not null default 100,
  claimed_at timestamptz, claimed_by text, claim_expires_at timestamptz,
  attempts int not null default 0, max_attempts int not null default 6,
  last_error text,
  status text not null default 'queued'
    check (status in ('queued','claimed','done','failed','cancelled','needs_review')),
  dedupe_key text
)
-- unique (kind, dedupe_key) where status in ('queued','claimed')
-- index: (status, run_at) where status = 'queued'
```

```sql
platform_capabilities (
  platform text not null, version int not null,
  manifest jsonb not null, effective_from timestamptz not null,
  primary key (platform, version)
)

audit_log (
  id bigserial pk, org_id uuid, actor_id uuid, action text not null,
  subject_type text, subject_id uuid, before jsonb, after jsonb, at timestamptz not null
)
```

---

## 5. State machines

### 5.1 `post_variants.status`

```
pending ──▶ validated ──▶ preflight_ok ──▶ publishing ──▶ published
   │            │              │                │
   │            │              │                └──▶ failed ──(user retry)──▶ preflight_ok
   │            │              └──▶ blocked  (token/quota/media; user must act)
   │            └──▶ invalid   (validation failed; blocks scheduling for this platform only)
   └──▶ skipped  (user chose to drop this platform)
                                                 └──▶ needs_review (ambiguous outcome — §10.3)
                                                 └──▶ expired_needs_decision (>60 min late — §10.4)
```

### 5.2 `posts.status` — derived, never set directly

| Condition across variants | Post status |
|---|---|
| all `pending`/`validated`, `scheduled_at` in future | `scheduled` |
| any `publishing` | `publishing` |
| all non-skipped are `published` | `published` |
| ≥1 `published` and ≥1 in {`failed`,`blocked`,`needs_review`} | `partially_published` |
| 0 `published` and all terminal | `failed` |
| no `scheduled_at` | `draft` |

A materialised trigger keeps `posts.status` current so the calendar can filter without a join fan-out.

### 5.3 `comment_jobs.status`

`blocked` until the parent variant reaches `published` and returns a `platform_post_id`; then `pending` → `published` | `failed`. Platforms where `capabilities.comment.supported = false` are created directly as `unsupported`, and — if the org has manual assist on — a `manual` task is emitted instead. **TikTok is always `unsupported`.**

---

## 6. Platform capability manifest

The single source of platform truth. Stored in `platform_capabilities.manifest`, read by the composer, the validator, pre-flight and the adapters. Every value below traces to a cited source in [`platform-api-feasibility.md`](./platform-api-feasibility.md).

### 6.1 Field map per platform

| Platform | Fields (in composer order) |
|---|---|
| youtube | `title`, `description`, `tags[]` |
| instagram | `caption` |
| tiktok (video) | `title` |
| tiktok (photo) | `title`, `description` |
| threads | `text` |
| x | `text` |

`post_variants.fields` is keyed by these names. A single `caption` column would be wrong the moment YouTube is added properly.

### 6.2 Manifest — populated

```jsonc
{
  "youtube": {
    "version": 1,
    "publish": { "endpoint": "POST youtube/v3/videos (resumable)", "container_flow": false },
    "scopes": ["youtube.upload", "youtube.force-ssl"],
    "fields": {
      "title":       { "limit": 100,   "unit": "chars",  "invalid_chars": ["<", ">"] },
      "description": { "limit": 5000,  "unit": "utf8_bytes" },
      "tags":        { "limit": 500,   "unit": "chars_total_incl_commas_and_quotes" }
    },
    "media": {
      "image": { "supported": false },
      "carousel": { "supported": false },
      "video": { "max_bytes": 274877906944, "mime": ["video/*"],
                 "shorts": { "auto_classified": true, "max_seconds": 180,
                             "requires": "square_or_vertical", "api_flag": null } }
    },
    "links": { "in_body": true, "clickable_requires": "channel_verification_or_advanced_features" },
    "comment": { "supported": true, "endpoint": "commentThreads.insert",
                 "quota_units": 50, "char_limit": null, "char_limit_status": "UNVERIFIED" },
    "rate": { "uploads_per_day_per_project": 100, "other_units_per_day": 10000,
              "search_calls_per_day": 100 },
    "tokens": { "access_ttl_sec": 3600, "refresh": "google_oauth",
                "refresh_dies_after_idle_days": 180, "testing_mode_refresh_ttl_days": 7,
                "max_refresh_tokens_per_client_per_account": 100 },
    "approval": { "required": true, "kind": "compliance_audit", "sla": null,
                  "pre_approval_behaviour": "uploads forced private" },
    "cost_per_post_usd": 0.0
  },

  "instagram": {
    "version": 1,
    "publish": { "endpoint": "POST /{ig-user-id}/media → POST /{ig-user-id}/media_publish",
                 "container_flow": true, "container_ttl_hours": 24 },
    "scopes": ["instagram_business_basic", "instagram_business_content_publish",
               "instagram_business_manage_comments"],
    "fields": { "caption": { "limit": 2200, "unit": "chars",
                             "max_hashtags": 30, "max_mentions": 20 } },
    "media": {
      "delivery": "public_url_fetch",
      "image": { "formats": ["image/jpeg"], "max_bytes": 8388608,
                 "aspect_min": 0.8, "aspect_max": 1.91,
                 "width_min": 320, "width_max": 1440, "color_space": "sRGB" },
      "carousel": { "supported": true, "min_items": 2, "max_items": 10,
                    "crops_to_first_item": true, "child_captions": false,
                    "reels_as_child": false },
      "video": { "profile": "REELS", "container": ["mp4", "mov"],
                 "video_codec": ["h264", "hevc"], "audio_codec": "aac",
                 "audio_hz_max": 48000, "fps_min": 23, "fps_max": 60,
                 "aspect_min": 0.01, "aspect_max": 10, "recommended_aspect": "9:16",
                 "max_horizontal_px": 1920, "max_video_bitrate_mbps": 25,
                 "min_seconds": 3, "max_seconds": 900, "max_bytes": 314572800 }
    },
    "links": { "in_body": true, "clickable": false,
               "clickable_status": "product behaviour, not documented in dev docs" },
    "comment": { "supported": true, "endpoint": "POST /{ig-media-id}/comments",
                 "field": "message", "char_limit": null, "char_limit_status": "UNVERIFIED",
                 "not_supported_on": ["live_video"] },
    "rate": { "posts_per_24h_per_account": 100, "carousel_counts_as": 1 },
    "tokens": { "short_ttl_sec": 3600, "long_ttl_days": 60,
                "refresh_min_age_hours": 24, "dies_permanently_after_days": 60 },
    "approval": { "required": true, "kind": "app_review + business_verification",
                  "observed_weeks": [2, 6] },
    "account_types": { "business": true, "creator": "UNVERIFIED" },
    "cost_per_post_usd": 0.0
  },

  "tiktok": {
    "version": 1,
    "publish": { "video": "POST /v2/post/publish/video/init/",
                 "photo": "POST /v2/post/publish/content/init/",
                 "preflight_required": "creator_info" },
    "scopes": ["video.publish"],
    "fields": {
      "video": { "title": { "limit": 2200, "unit": "utf16" } },
      "photo": { "title": { "limit": 90, "unit": "utf16" },
                 "description": { "limit": 4000, "unit": "utf16" } }
    },
    "media": {
      "video": { "delivery": ["file_upload_chunked", "pull_from_url"],
                 "container": ["mp4", "webm", "mov"],
                 "codecs": ["h264", "h265", "vp8", "vp9"],
                 "px_min": 360, "px_max": 4096, "fps_min": 23, "fps_max": 60,
                 "max_seconds": 600, "max_bytes": 4294967296,
                 "chunk_min_bytes": 5242880, "chunk_max_bytes": 67108864,
                 "final_chunk_max_bytes": 134217728, "max_chunks": 1000 },
      "photo": { "delivery": ["pull_from_url"], "requires_domain_verification": true,
                 "formats": ["image/jpeg", "image/webp"],
                 "max_px": 1080, "max_bytes": 20971520, "max_count": 35 }
    },
    "links": { "in_body": "UNVERIFIED" },
    "comment": { "supported": false,
                 "reason": "no public endpoint exists; only disable_comment at creation" },
    "rate": { "requests_per_minute_per_user_token": 6,
              "daily_posts_per_creator": "set at audit",
              "unaudited": { "users_per_24h": 5, "forced_privacy": "SELF_ONLY",
                             "account_must_be_private": true } },
    "tokens": { "access_ttl_sec": 86400, "refresh_ttl_sec": 31536000,
                "refresh_without_user": true },
    "approval": { "required": true, "kind": "sandbox → audit",
                  "observed_business_days": [5, 10] },
    "ux_obligations": ["show_creator_nickname", "query_creator_info_before_post",
                       "privacy_selection_no_default", "respect_interaction_toggles",
                       "commercial_content_disclosure", "content_preview", "publish_status"],
    "cost_per_post_usd": 0.0
  },

  "threads": {
    "version": 1,
    "publish": { "endpoint": "POST /{threads-user-id}/threads → /threads_publish",
                 "container_flow": true },
    "scopes": ["threads_basic", "threads_content_publish"],
    "fields": { "text": { "limit": 500, "unit": "chars_emoji_as_utf8_bytes" } },
    "media": {
      "delivery": "public_url_fetch",
      "image": { "formats": ["image/jpeg", "image/png"], "max_bytes": 8388608,
                 "aspect_max": 10, "width_min": 320, "width_max": 1440,
                 "color_space": "sRGB" },
      "carousel": { "supported": true, "min_items": 2, "max_items": 20, "mixed_media": true },
      "video": { "container": ["mov", "mp4"], "max_bytes": 1073741824,
                 "max_seconds": 300, "max_bitrate_mbps": 100,
                 "fps_min": 23, "fps_max": 60, "max_horizontal_px": 1920 }
    },
    "links": { "in_body": true, "attachment_param": "link_attachment",
               "max_links_text_only": 5 },
    "comment": { "supported": true, "mechanism": "reply_to_id on the same publish flow",
                 "sequential_only": true },
    "rate": { "posts_per_24h": 250, "replies_per_24h": 1000, "deletes_per_24h": 100,
              "check_endpoint": "GET /{threads-user-id}/threads_publishing_limit" },
    "tokens": { "short_ttl_sec": 3600, "long_ttl_days": 60,
                "refresh_min_age_hours": 24, "dies_permanently_after_days": 60 },
    "approval": { "required": true, "kind": "meta_app_review" },
    "cost_per_post_usd": 0.0
  },

  "x": {
    "version": 1,
    "publish": { "endpoint": "POST /2/tweets",
                 "media": "POST /2/media/upload/{initialize,append,finalize} + GET status" },
    "scopes": ["tweet.write", "tweet.read", "users.read", "offline.access"],
    "fields": { "text": { "limit": 280, "unit": "chars_urls_as_23",
                          "premium_limit": null, "premium_limit_status": "UNVERIFIED" } },
    "media": {
      "delivery": "chunked_upload", "append_chunk_max_bytes": 5242880,
      "per_post": { "max_media_ids": 4, "max_photos": 4, "max_gifs": 1, "max_videos": 1 },
      "image": { "max_bytes": 5242880 },
      "gif": { "max_bytes": 15728640 },
      "video": { "max_seconds": 1200, "max_bytes": 8589934592,
                 "premium_max_seconds": 7500, "premium_max_bytes": 17179869184 },
      "carousel": { "supported": true, "max_items": 4 }
    },
    "links": { "in_body": true, "clickable": true, "shortener": "t.co", "counted_as": 23 },
    "comment": { "supported": true, "mechanism": "reply.in_reply_to_tweet_id",
                 "billed_as_a_post": true },
    "rate": { "posts_per_15min_per_user": 100, "posts_per_24h_per_app": 10000 },
    "tokens": { "access_ttl_sec": 7200, "refresh_requires_scope": "offline.access",
                "refresh_ttl": "UNVERIFIED" },
    "approval": { "required": false, "note": "no review gate located — verify" },
    "cost_usd": { "post": 0.015, "post_with_url": 0.200, "post_read": 0.005,
                  "owned_read": 0.001, "free_tier": "discontinued for new developers (reported)" }
  }
}
```

### 6.3 Character counting — one function per platform

A naive `string.length` is wrong for exactly the posts users care about (emoji, links). Implement and unit-test these:

| Platform / field | Algorithm |
|---|---|
| youtube.title | code points, limit 100 |
| youtube.description | `TextEncoder().encode(s).length`, limit 5000 **bytes** |
| youtube.tags | join with `,`, quote any tag containing a space, count characters, limit 500 |
| instagram.caption | code points, limit 2200; separately count `#\w+` ≤ 30 and `@\w+` ≤ 20 |
| tiktok.* | `s.length` (UTF-16 code units) |
| threads.text | ASCII char = 1, any non-ASCII = its UTF-8 byte length; limit 500 |
| x.text | replace every `https?://\S+` with 23 characters, then count code points; limit 280 |

Golden test cases to lock in: `"🎞️"` (7 UTF-8 bytes, 2 UTF-16 units, 2 code points), `"https://halftone.app/2-4"` (24 chars raw, 23 on X), an em dash `"—"` (3 UTF-8 bytes).

---

## 7. Adapter contract

```ts
type ValidationResult = {
  ok: boolean;
  errors: Array<{ field: string; code: string; message: string;
                  fix?: 'shorten' | 'reencode' | 'trim_media' | 'reconnect' | 'skip' }>;
  warnings: Array<{ field: string; message: string }>;
};

type PreflightResult = {
  ok: boolean;
  reason?: 'token_expired' | 'quota_exhausted' | 'account_disconnected'
         | 'media_unreachable' | 'validation_stale' | 'account_state';
  remainingQuota?: number;
  accountSnapshot?: Record<string, unknown>;   // written back to capability_snapshot
};

interface PlatformAdapter {
  readonly platform: Platform;
  capabilities(): Manifest;                                  // from platform_capabilities
  validate(v: Variant, r: Rendition[]): ValidationResult;    // composer AND preflight call this
  preflight(a: Account, v: Variant): Promise<PreflightResult>;
  publish(a: Account, v: Variant, r: Rendition[], idem: string):
    Promise<{ platformPostId: string; permalink: string }>;
  comment(a: Account, platformPostId: string, text: string):
    Promise<{ platformCommentId: string } | { unsupported: true; reason: string }>;
  fetchMetrics?(a: Account, platformPostId: string): Promise<Metrics>;   // v1.1
}
```

`comment()` returning `{ unsupported }` is a first-class, tested path — not an exception. TikTok's adapter returns it unconditionally, and the composer reads `capabilities().comment.supported` to grey the field out before the user ever schedules.

### 7.1 Publish sequences

**Instagram / Threads** (container flow)
```
1. ensure renditions ready + signed public_url valid ≥ 2h
2. POST /media (or /threads) with the field map + media URLs → creation_id
3. poll status if video (IG: FINISHED before publish)
4. POST /media_publish (or /threads_publish) with creation_id → platform_post_id
5. store permalink; enqueue comment job
```

**TikTok video**
```
1. GET creator_info → refresh capability_snapshot; abort if privacy option no longer allowed
2. POST /v2/post/publish/video/init/ with post_info (privacy_level chosen by the user, no default)
3. FILE_UPLOAD: PUT chunks 5–64 MB sequentially to upload_url   (or PULL_FROM_URL for photos)
4. poll publish status until PUBLISH_COMPLETE
5. comment job → created directly as 'unsupported' / 'manual'
```

**YouTube**
```
1. resumable session: POST videos.insert?uploadType=resumable with snippet + status
2. PUT the bytes; resume with exponential backoff on 5xx
3. response → videoId
4. if project unaudited: video is private — set variant.degraded_reason = 'awaiting_youtube_audit'
5. commentThreads.insert (50 quota units) for the first comment
```

**X**
```
1. media: initialize → append (≤5 MB chunks) → finalize → poll STATUS
2. POST /2/tweets { text, media.media_ids }                       → cost $0.015 / $0.200 w/ URL
3. reply: POST /2/tweets { text, reply.in_reply_to_tweet_id }     → billed again
4. increment organizations.x_posts_used_this_period
```

---

## 8. Media pipeline

Derive renditions **at schedule time**, not publish time, so spec violations surface while the user is still in the composer.

| Profile | Target |
|---|---|
| `ig_feed_4x5` | JPEG q85, 1080×1350, sRGB, ≤ 8 MB, EXIF stripped |
| `ig_reel_9x16` | MP4, H.264 high, yuv420p, 1080×1920, ≤ 30 fps, ≤ 25 Mbps VBR, AAC 128k/48k, `+faststart`, ≤ 300 MB, 3–900 s |
| `th_image` | JPEG or PNG passthrough if ≤ 8 MB and width 320–1440, else JPEG q85 1440 wide |
| `th_video_9x16` | MP4 H.264, ≤ 1920 horizontal, ≤ 300 s, ≤ 1 GB |
| `tt_video_9x16` | MP4 H.264, 1080×1920, 23–60 fps, ≤ 600 s, ≤ 4 GB |
| `tt_photo` | JPEG ≤ 1080p, ≤ 20 MB, served from the TikTok-verified domain |
| `x_image` | JPEG q82, ≤ 5 MB, longest edge 2048 |
| `x_video` | MP4 H.264, ≤ 1200 s |
| `yt_video` | passthrough; only remux if the container is unsupported |

`+faststart` (moov atom at the front) is mandatory for Meta and cheap everywhere — apply it to every MP4.

**Serving.** One dedicated domain, e.g. `media.apptap.io`, registered as a verified property in the TikTok developer portal (domain-level verification covers all paths and subdomains). Signed URLs with a **48-hour TTL** — comfortably longer than Instagram's 24-hour container lifetime, so a container created at the edge of the window still resolves. Renditions are garbage-collected 7 days after the post reaches a terminal state.

---

## 9. Token lifecycle

The refresher sweeps `social_accounts` every 5 minutes ordered by `access_expires_at`. Strategies differ enough that a single "refresh when expired" rule would lose accounts permanently.

| Platform | Access TTL | Refresh strategy | Failure mode if missed |
|---|---|---|---|
| **X** | 2 h | On demand **and** preemptively 10 min before any scheduled publish. Assert `offline.access` was granted at connect time — without it no refresh token is issued at all. | Publish fails; user reconnects |
| **TikTok** | 24 h | Refresh at ~20 h. Refresh token valid 365 days, no user interaction. | Publish fails; user reconnects |
| **Instagram / Threads** | 1 h short, **60 d long-lived** | Refresh **weekly**, not at day 59. Token must be ≥ 24 h old and unexpired. | **Permanent death at 60 days unrefreshed** — full re-auth, no recovery |
| **YouTube** | ~1 h | Refresh on demand. Watch: 6-month idle expiry, **7-day refresh TTL while the OAuth app is in Testing status**, 100 refresh tokens per client per account (oldest silently invalidated). | `invalid_grant`; user reconnects |

A weekly Meta cadence gives roughly eight chances to recover from an outage before the 60-day cliff. Alert if the refresher's oldest un-refreshed Meta account exceeds 14 days.

On refresh failure: set `health = 'reconnect_required'`, notify the user immediately, and mark every future `post_variants` row on that account as `blocked` **in advance** — not silently at publish time.

---

## 10. Publishing, errors and partial failure

### 10.1 Job flow

```
schedule time set
      │
      ├─ T-15m  preflight job per variant  → ok | blocked (+notify BEFORE the slot)
      │
      └─ T-0    one publish job per variant, fanned out, independent
                      │
                      ├─ success → store platform_post_id → unblock comment job
                      └─ failure → classify (§10.2) → retry or stop
```

### 10.2 Error taxonomy

| Class | Signals | Behaviour |
|---|---|---|
| **transient** | HTTP 5xx, timeouts, 429, TikTok rate-limit, Meta transient graph errors | Exponential backoff with full jitter, honour `Retry-After`; base 2 s, cap 5 min, **max 6 attempts over ~30 min**, then `failed` |
| **terminal_fixable** | `invalid_grant`, expired/revoked token, caption over limit (IG reports `2207010` — **[UNVERIFIED]**, do not depend on it), media spec violation, unsupported account type, quota exhausted for the day | **No retry.** Store a `fix` hint; surface a one-click action |
| **terminal_permanent** | content policy rejection, duplicate content, account suspended | No retry. Show the platform's own message **verbatim** — never paraphrase a moderation decision |
| **ambiguous** | connection dropped after the request was sent, unknown 5xx after upload finalize | See §10.3 |

Adapters map provider errors into this taxonomy; the classification never leaks platform-specific codes upward.

### 10.3 Ambiguous outcomes — the anti-double-post rule

Never blind-retry. On an ambiguous result:

1. Query the platform for a post created by this account within `scheduled_at ± 10 min` matching the idempotency marker (X: `POST /2/tweets` idempotency; others: list recent media and match on `platform_post_id` absence + creation timestamp + first 40 characters of the body).
2. If found → record it, mark `published`, no second call.
3. If provably absent → one retry with the **same** idempotency key.
4. If undeterminable → `status = needs_review`, job `needs_review`, notify: *"We couldn't confirm whether this posted to X. Check your profile and tell us."* **Never auto-retry from this state.**

Double-post incidents are P0. There is no acceptable rate.

### 10.4 Late publishing

A variant still unpublished more than **60 minutes** past `scheduled_at` moves to `expired_needs_decision` and asks: publish now, reschedule, or cancel. Silently auto-posting launch content six hours late is a real harm.

### 10.5 The 4-of-5 scenario, as implemented

Post targets all five at 09:00. Instagram's token expires at 08:57 and cannot be refreshed.

| Time | Event |
|---|---|
| 08:45 | Pre-flight: 4 healthy, Instagram refresh returns `invalid_grant` → variant `blocked`, **user notified 15 min early** |
| 09:00 | YouTube, TikTok, Threads, X publish; median end-to-end 6.2 s |
| 09:00 | Instagram variant → `failed`, class `terminal_fixable`, fix `reconnect` |
| 09:01 | Comment jobs fire on YouTube, Threads, X; TikTok comment created as `unsupported` → manual task |
| 09:01 | **One** notification: "Published to 4 of 5. Instagram needs reconnecting." |
| — | `posts.status = partially_published`. Four green rows, one amber with the reason and a Reconnect button |
| later | **Retry Instagram** re-runs pre-flight and publishes that variant only, same media, same caption. The four live posts are untouched. |

**No rollback, ever.** Deleting a published post to make a set consistent is a visible action taken on the user's behalf and is worse than the inconsistency.

### 10.6 Notification policy

One notification per post per outcome, aggregated. Successes are digest-only. Pre-flight warnings arrive *before* the slot — that is the whole point of having a pre-flight.

---

## 11. Job queue mechanics

```sql
-- claim (scheduler, every 2s)
UPDATE scheduled_jobs SET
  status = 'claimed', claimed_at = now(), claimed_by = $worker,
  claim_expires_at = now() + interval '5 minutes', attempts = attempts + 1
WHERE id IN (
  SELECT id FROM scheduled_jobs
  WHERE status = 'queued' AND run_at <= now()
  ORDER BY priority, run_at
  FOR UPDATE SKIP LOCKED
  LIMIT 20
)
RETURNING *;

-- reap orphans (every 60s)
UPDATE scheduled_jobs SET status = 'queued', claimed_by = NULL
WHERE status = 'claimed' AND claim_expires_at < now() AND attempts < max_attempts;
```

**Why Postgres and not a broker.** It survives deploys and restarts, it is inspectable in SQL when a founder asks why a post did not go out, it needs no extra infrastructure, and `FOR UPDATE SKIP LOCKED` handles the concurrency correctly. Redis holds only ephemeral state: rate-limit token buckets and advisory locks.

**Rate limiting.** A Redis token bucket per `(account_id, platform)` sits in front of every adapter call. TikTok's 6 requests/minute is the binding one; a publish that would exceed it is deferred, not failed.

**Idempotency.** `idempotency_key = sha256(variant_id || attempt_epoch)` where `attempt_epoch` increments only on a *user-initiated* retry — never on an automatic one. Automatic retries reuse the key so the platform can de-duplicate.

---

## 12. Internal API surface

REST, JSON, cookie session for the web app. Sketch, not a contract.

```
POST   /api/apps/:id/store-connections          connect ASC or Play
GET    /api/apps/:id/overview                   dashboard payload (metrics + moments)
GET    /api/apps/:id/moments                    ranked content moments
POST   /api/moments/:id/draft                   → a draft post, returns post + variants
GET    /api/apps/:id/reviews?state=none
POST   /api/reviews/:id/reply                   350-char cap enforced for Play

GET    /api/accounts                            with health + capability_snapshot
GET    /api/accounts/connect/:platform          → OAuth redirect
GET    /api/accounts/callback/:platform
DELETE /api/accounts/:id

POST   /api/posts                               create draft
PATCH  /api/posts/:id                           canonical fields; recomputes non-overridden variants
PATCH  /api/posts/:id/variants/:vid             per-platform override; marks overridden_fields
POST   /api/posts/:id/validate                  → ValidationResult per platform
POST   /api/posts/:id/schedule                  materialises plan + jobs
POST   /api/posts/:id/variants/:vid/retry       user-initiated; bumps attempt_epoch
POST   /api/posts/:id/variants/:vid/skip

POST   /api/media                               upload → asset + rendition jobs
GET    /api/media/:id/renditions

GET    /api/calendar?from=&to=
GET    /api/posts/:id/performance
GET    /r/:slug                                 public tracked-link redirect (no auth)
```

---

## 13. Cross-cutting

### 13.1 Security
- Platform tokens, App Store Connect `.p8` keys and Play service-account JSON are **envelope-encrypted**; the data-encryption key never leaves the KMS. The database stores only a `*_ref` handle.
- The `.p8` is write-only through the API — never returned, never logged, never rendered.
- OAuth state parameter is a signed, single-use, 10-minute nonce bound to the session.
- Signed media URLs are unguessable (128-bit slug) and expire in 48 h.
- `audit_log` records every credential change, account connect/disconnect, and publish.
- Rendered post text is never interpolated into HTML without escaping — user content flows to five external APIs and back into the UI.

### 13.2 Observability

SLOs:

| Metric | Target |
|---|---|
| Scheduled variant published within 5 min of target | ≥ 99% |
| Partial-failure rate per post | < 2% |
| **Double-post incidents** | **0** — any occurrence is P0 |
| Pre-flight catches a preventable failure before T-0 | ≥ 80% of eventual failures |

Alerts: refresher lag > 14 days on any Meta account · `needs_review` count > 0 · YouTube `videos.insert` daily bucket > 80 of 100 · X monthly spend > 120% of forecast · transcode queue depth > 50 · any `terminal_permanent` classified as a policy rejection (early signal of a ToS problem).

Every `publish_attempts` row stores `provider_request_id` — without it, platform support tickets are unanswerable.

### 13.3 Cost accounting
X is metered per call at publish time against `organizations.x_post_allowance`. A post with a URL in either the body or the first comment costs $0.200; the meter charges both calls independently. The composer shows remaining allowance; exceeding it blocks X scheduling rather than silently overspending.

---

## 14. Open technical decisions

Deliberately unfixed. Each needs a decision before the relevant milestone, not before the first line of code.

| # | Decision | Needed by |
|---|---|---|
| 1 | Language and framework for web/api and workers | Week 1 |
| 2 | Hosting and object-storage provider (must support signed URLs on a custom domain — TikTok verification depends on it) | Week 2 |
| 3 | Where transcoding runs: in-process FFmpeg vs a managed transcoding service (CPU cost vs ops cost) | Week 3 |
| 4 | LLM provider and prompt strategy for draft generation and shorten-to-fit | Week 2 |
| 5 | Whether `metrics-poller` is on-view only, or on-view plus a nightly batch for non-X platforms | v1.1 |
| 6 | Link-click storage: partitioned Postgres table vs a dedicated analytics store | when clicks/day > 10⁵ |

## 15. Blocking unknowns carried from research

These change the design if the answer is unfavourable and must be resolved during Phase 0.

1. **Instagram Creator-account publishing.** If unsupported, `social_accounts.account_type` becomes a hard gate at connect time and onboarding needs a "switch to Business" flow. *(Manifest: `account_types.creator = "UNVERIFIED"`)*
2. **X refresh-token lifetime and rotation.** A 2-hour access token with a broken refresh is fatal to a scheduler. *(Manifest: `tokens.refresh_ttl = "UNVERIFIED"`)*
3. **X Premium character ceiling** — 4,000 or 25,000. Affects §6.3's counter.
4. **YouTube audit turnaround**, and whether passing it lifts *both* the private-video restriction and the 100/day upload bucket. The 100/day bucket is per API project and binds at roughly 100 daily-posting customers.
5. **Are URLs in YouTube comments clickable on an unverified channel?** If not, the first-comment feature delivers nothing on YouTube for exactly our users.
6. **Instagram and YouTube comment character limits** — undocumented; the composer currently cannot validate them.
7. **TikTok's written position on scheduled, non-user-present publishing.** Buffer and Later do it as Marketing Partners; the guideline text says "expressly consented to the upload." Ask explicitly in the audit application.
8. **X automation rules** on cross-posted or substantially similar content. The rules page was not fetchable during research, and cross-posting is literally the product.
9. **App Store Connect analytics latency and granularity** — sets what the dashboard can honestly claim is "today".

---

*Constants in §6 were retrieved 3 September 2026 and are cited in [`platform-api-feasibility.md`](./platform-api-feasibility.md). Platform terms in this space change without notice; treat the manifest as a living document and version it rather than editing in place.*
