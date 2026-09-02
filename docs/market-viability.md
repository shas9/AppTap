# AppTap — Market Viability Assessment

**Date:** 3 September 2026 · **Status:** decision input for the founding team
**Reads with:** [`PRD.md`](./PRD.md) · [`competitive-analysis.md`](./competitive-analysis.md) · [`platform-api-feasibility.md`](./platform-api-feasibility.md) · [`design-doc.md`](./design-doc.md)

**Conventions used below.** *[DOC]* = claim carried from the existing documents. *[VERIFIED]* = independently confirmed this session, with a link. *[ASSUMPTION]* = a rate or ratio I introduced, with the reasoning stated. Every number in the funnel is one of these three.

---

## 1. Verdict

**Do not build AppTap as specified.** The demand arithmetic says the realistic base case is roughly 60 paying customers and ~$2,300 MRR at steady state — enough to cover infrastructure and nothing else, against 4–6 months of unpaid build time and three platform-approval gauntlets on the critical path. Only the optimistic case, which requires six independent assumptions to land at their ceilings simultaneously, produces a salary.

Confidence: **high** on the unit economics and break-even arithmetic (the inputs are verified prices); **medium** on the demand sizing (two of the six funnel ratios have thin evidence).

**The two facts that would flip this:** (1) if a paid concierge probe converts ≥ 5 of 30 cold-contacted founders at $19–29/month within 30 days, the willingness-to-pay objection dissolves and the base case moves toward the optimistic column; (2) if the paying segment is genuinely 3–5-app studios at $79–149/month rather than solo founders at $29, break-even for a founder salary drops from ~171 customers to ~79, which is inside reach. Both are testable in weeks, without writing an adapter.

---

## 2. Demand arithmetic

### 2.1 Bottom-up market size

Not "2.58 million apps." Almost all of those are dead, corporate, or published by someone who will never pay for a marketing tool.

| Stage | Pess. | Base | Opt. | Basis |
|---|---|---|---|---|
| Mobile app publishers worldwide (deduped union) | 1.3M | 1.3M | 1.3M | 1,093,902 iOS publishers ([42matters, 2 Sep 2026](https://42matters.com/ios-apple-app-store-statistics-and-trends)) + ~739K–785K Google Play publishers ([42matters](https://42matters.com/google-play-statistics-and-trends)), assuming heavy overlap *[VERIFIED]* |
| × actively maintaining an app (shipped in 12 mo) | 30% | 40% | 50% | 51% of App Store apps had no update in ≥ 1 year ([Sensor Tower](https://sensortower.com/blog/abandoned-apps)) *[VERIFIED, dated study]* |
| = | 390K | 520K | 650K | |
| × solo / micro publisher who does their own marketing | 50% | 60% | 65% | *[ASSUMPTION]* — publisher counts are dominated by the long tail; excludes games studios, enterprise, agency-published |
| = | 195K | 312K | 423K | |
| × app earns enough to fund $29–79/mo of tooling | 15% | 22% | 30% | Median subscription app earns **$492/month**; **57.7% of new apps never reach $1,000 in total revenue**; top 10% capture 94.5% of revenue ([RevenueCat State of Subscription Apps 2026](https://www.revenuecat.com/state-of-subscription-apps)) *[VERIFIED]* |
| = | 29K | 69K | 127K | |
| × actively wants to market on the 5 target platforms | 30% | 40% | 50% | *[ASSUMPTION — thin evidence]* many indie devs market via ASO, Reddit, or not at all |
| × reachable in English via IH / Reddit / PH / X | 40% | 50% | 60% | *[ASSUMPTION]* r/iOSProgramming ≈ 204K members, Indie Hackers ≈ 140K registered ([sources](https://gummysearch.com/r/iOSProgramming/), [Vibe](https://www.vibecontentcreation.com/blog/indie-hacker-communities-where-solo-founders-hang-out)) *[VERIFIED]* |
| **Qualified prospects (SOM)** | **3.5K** | **14K** | **38K** | |

A 14,000-business serviceable market is not fatal by itself — plenty of good micro-SaaS lives there. What makes it hard is the *value per prospect*, which the next table settles.

### 2.2 Year-one funnel to steady state

| Stage | Pess. | Base | Opt. | Basis |
|---|---|---|---|---|
| Signups, year 1 | 500 | 1,200 | 3,000 | *[ASSUMPTION]* — solo founder, zero ad budget, community distribution only |
| × activation (completes App Store Connect `.p8` connect) | 25% | 35% | 45% | *[ASSUMPTION]* — the PRD itself calls this "the activation cliff" ([PRD §6.1](./PRD.md)); creating an ASC API key and uploading a `.p8` is materially harder than an OAuth click |
| = activated | 125 | 420 | 1,350 | |
| × activated → paying | 8% | 12% | 15% | Developer tools convert at **2–4% freemium / 10–18% trial** ([benchmarks](https://userpilot.com/blog/saas-average-conversion-rate/)); activation already filters hard, so the trial band applies *[VERIFIED benchmark, applied by assumption]* |
| = new paying customers, year 1 | 10 | 50 | 200 | |
| Net adds / month | 0.8 | 4.2 | 16.7 | |
| Monthly logo churn | 8% | 7% | 5% | SMB/self-serve **3–7%/month**, highest under $500/year ([churn benchmarks](https://www.koji.so/blog/saas-churn-rate-benchmarks-2026)); AppTap sits at the bad end because the customer churns when the *app* dies *[VERIFIED band, top-of-band by assumption]* |
| **Steady-state customers** (adds ÷ churn) | **10** | **60** | **333** | |
| Blended ARPU | $29 | $39 | $55 | Plan mix from [PRD §12](./PRD.md) |
| **Steady-state MRR** | **$290** | **$2,340** | **$18,300** | |

The steady-state formula (monthly adds ÷ monthly churn) is the honest one for a self-serve product with no expansion revenue: at 7% churn, 4.2 adds/month can never sustain more than 60 logos no matter how long you run.

**On the optimistic column:** it requires 3,000 signups, 45% activation, 15% conversion *and* 5% churn *and* a studio-heavy mix, all at once. Treating each as roughly independent and each as a ~40–50% chance of hitting its ceiling, the joint probability is well under 10%. Do not plan against it.

---

## 3. Unit economics and break-even

### 3.1 Per-customer variable cost, monthly

| Item | Moderate user | Heavy user | Basis |
|---|---|---|---|
| X API | $2.58 | $6.45 | **$0.015/post, $0.20 if the post contains a link**, no free tier for new developers, legacy Basic retired and force-migrated to pay-per-use after 1 June 2026 ([X API pricing 2026](https://postproxy.dev/blog/x-api-pricing-2026/), [OpenTweet](https://opentweet.io/blog/x-api-cost-2026)) *[VERIFIED — the doc's figures are current]* |
| LLM (drafts + per-platform adaptation) | $1.50 | $3.00 | *[ASSUMPTION]* ~30 drafts + 60 adaptations at frontier-class per-call cost |
| Media storage, transcode, egress | $2.00 | $4.00 | *[ASSUMPTION]* — Instagram fetches by public URL and TikTok photos are `PULL_FROM_URL`, so hosting is mandatory, not optional ([feasibility doc](./platform-api-feasibility.md)) |
| Stripe | $1.43 | $1.43 | 2.9% + $0.30 on $39 |
| **Total COGS** | **$7.51** | **$14.88** | |
| **Contribution on $39 blended ARPU** | **$31.49 (81%)** | **$24.12 (62%)** | |
| Contribution on $29 Indie, heavy X | — | **$17.80 (61%)** | |

Margin is acceptable **only with a hard X post cap**. An uncapped user posting 3× daily with links costs $19.35/month in X fees alone and turns a $29 subscription into $8 of contribution. The PRD's [§12 margin warning](./PRD.md) is correct and must become a metered limit, not a note.

Fixed monthly cost, solo, at this scale: infra/workers/DB/object store ~$250–400, monitoring and tooling ~$60, Apple $99/yr, domains and misc. Call it **$400/month** *[ASSUMPTION]*.

### 3.2 The two break-evens

| Scenario | Contribution needed | Customers required | Base case reaches it? |
|---|---|---|---|
| (a) COGS + fixed, founder takes $0 | $400/mo | **13 customers** | Yes — comfortably |
| (b) + modest founder draw of $5,000/mo | $5,400/mo | **171 customers** at $39 ARPU | **No** — base case is 60 |
| (b′) same, studio-heavy mix at $79 ARPU (~$68 contribution) | $5,400/mo | **79 customers** | Borderline |

Base case at 60 customers yields ~$1,890/month contribution, ~$1,490/month after fixed costs. **That is a break-even side project, not an income.** $5,000/month is already a below-market draw for a developer who could contract instead; the true opportunity cost of the build is higher than the model shows.

**Time to first dollar.** The app-data loop needs no platform approval and is ~8–12 weeks of solo build. Publishing needs Meta App Review (now averaging **20 days per round**, plus Business Verification, rejections common on first submission — [bundle.social](https://bundle.social/blog/meta-app-review-20-days), [Phyllo](https://www.getphyllo.com/post/instagram-api-integration-101-for-developers-of-the-creator-economy)) *[VERIFIED — worse than the doc's 2–6 week estimate]*, plus the Google audit and the TikTok audit. Realistic first paying customer: **month 4–6**. Full five-platform publishing: **month 7–10**. Runway required at zero income: **6–9 months**.

---

## 4. What the documents got right, and what changed

**Confirmed, unchanged:**
- X pay-per-use at $0.015/post and $0.20/post-with-link, no free tier for new developers. Additionally: legacy Basic was *retired* and its subscribers force-migrated to pay-per-use after 1 June 2026, and Pro is closed to new signups. There is no cheaper route. ([source](https://postproxy.dev/blog/x-api-pricing-2026/))
- Buffer: free = 3 channels / 10 posts per channel; Essentials **$5/channel/month** annual, $6 monthly. ([source](https://www.blotato.com/blog/buffer-pricing)) The $25/month-for-5-channels comparison in the competitive analysis holds.
- Appfigures Connect **$9.99/month** ($7.99 annual). The $9.99 indie price anchor is real. ([source](https://www.g2.com/products/appfigures/pricing))
- No Band 1 scheduler has shipped App Store Connect ingestion; no Band 2 tool has shipped social publishing. AppFollow's Q2 2026 release notes are all review-side AI — translation, auto-reply tracking, concern filing — with nothing publishing-shaped. ([AppFollow Q2 2026](https://appfollow.io/blog/appfollow-q2-2026-steam-beta-ai-reply-style-and-report-a-concern-for-everyone))

**Changed or newly material:**
1. **Meta App Review is slower than the docs assume** — ~20 days per round in 2026, before Business Verification and before the near-certain first rejection. Budget 6–10 weeks for Instagram alone, not 2–6.
2. **The intersection is no longer empty — it is being approached from the adjacent side.** [Shipstar](https://shipstar.ai/solutions/turn-changelog-into-social-posts), [SocialClaw](https://getsocialclaw.com/blog/turn-your-saas-changelog-into-social-posts-automatically) and [PersonaBox](https://personabox.app/blog/best-changelog-tools) all ship "turn your shipping activity into social posts, with a review queue" *today*, aimed at exactly the build-in-public founder segment. They read **GitHub**, not App Store Connect — so the competitive analysis's literal claim survives, but the *job to be done* it rests on is already contested, and contested by products with a far easier onboarding step (one OAuth click versus generating and uploading a `.p8`). This is the single most important change to the strategic picture.

**Not confirmed:** I could not find a study measuring how many indie app founders actually want automated social posting, at any price. That number does not appear to exist publicly, which is why §7 recommends generating it directly.

---

## 5. The wedge under scrutiny

The whole plan rests on one sentence in [PRD §1](./PRD.md): the binding constraint is *content supply*, not scheduling friction.

**Evidence for.** Community sources consistently describe marketing, not tooling, as the solo-dev bottleneck — "the building part being easy and the marketing part being hard is the most common solo dev problem in 2026" ([ApsteQ](https://apsteq.com/blog/indie-app-marketing/)), with consistency named as the specific failure. And three funded-enough-to-exist products now sell exactly this loop for GitHub repos, which is mild market validation that someone will pay to have their shipping activity turned into posts.

**Evidence against, in order of severity.**

1. **App-store data does not supply enough posts.** A release, a notable review and a milestone might yield 4–8 postable events per month for an active indie app — and far fewer for one that ships quarterly. A consistent cadence needs 12–20. AppTap therefore fills perhaps a third of the calendar and hands the blank text box back for the rest. The product solves the *first* post, not the *habit*. This is a structural flaw in the value proposition, not an execution detail.
2. **The customers who most need it can least pay.** The founder whose blank text box is the problem is, statistically, the founder whose app earns $492/month or less — 57.7% of new apps never clear $1,000 in *lifetime* revenue. $29/month is 6% of median app MRR spent on marketing tooling by someone whose app is not yet working.
3. **Generic AI has partly absorbed the problem.** Buffer's AI assistant and SocialBee's AI writer already generate captions; ChatGPT does it free. AppTap's differentiator is not "AI writes it" but "AI knows what happened to your app" — a narrower claim than the positioning line implies.
4. **The moat is a data connection, not a technology.** Any of Buffer, Appfigures, AppFollow or Shipstar can add an App Store Connect key upload in a sprint. Nothing about it is hard; it is simply unbuilt.

**Assessment: the wedge is real but thin.** It is a defensible *feature* and a weak *company*. The genuinely defensible thing in the documents is the post→install attribution loop in [competitive-analysis §5](./competitive-analysis.md) — which the same document correctly flags as too hard for v1.

---

## 6. Alternative paths, ranked

| # | Path | Viability | Rationale |
|---|---|---|---|
| 1 | **Draft engine only — no publishing layer.** Ship app-data → drafts, with copy-to-clipboard and "manual assist." | **Highest** | Zero platform approvals, zero X fees, ~$3 COGS, buildable in 3–4 weeks, and it tests the *only* load-bearing hypothesis. If founders won't use free drafts, nothing downstream matters. |
| 2 | **Studio-first at $79–149.** Make 3–5-app studios the primary paying segment, not the design-partner segment. | **High** | Break-even for a salary drops from 171 customers to ~79. Studios churn less (they outlive any one app) and have budget. The competitive analysis already recommends this ([§5 segment recommendation](./competitive-analysis.md)) — the PRD's pricing does not fully commit to it. |
| 3 | **Drop X entirely at launch.** | **Do this regardless** | Removes the only unbounded variable cost. It is a cost decision, not a strategy — it improves any path but rescues none. |
| 4 | **Post→install attribution as the product.** | **Medium, later** | The real moat and the only thing justifying $149+. Genuinely hard, partly outside your control, and not credible from a pre-revenue solo team. Preserve the option (tag every post with a campaign ID); do not build it now. |
| 5 | **AppTap as specified — full five-platform publishing in v1.** | **Lowest** | Ten weeks of approvals, an unbounded X cost, a commoditised layer priced at zero by eight incumbents, delivered to the lowest-willingness-to-pay segment in software. Worst risk-adjusted return of the five. |

---

## 7. Kill criteria and the cheapest next test

### The test — run it before writing any adapter code

**A two-week manual concierge, with a payment ask.** No product, no OAuth, no `.p8` upload, no approvals.

1. Pick **30 real indie apps** whose founders are publicly reachable (Indie Hackers, r/iOSProgramming, X). Everything needed is *public*: current version, release notes, recent reviews, rating, category. No API access required from the founder.
2. For each, generate **three post drafts** with an LLM — one from the latest release notes, one from a recent five-star review, one from a milestone.
3. Send them cold, personally. No pitch deck, no landing page. Just: "I made these for your app — want three more every week?"
4. On a positive reply, ask for **$19/month, paid now**, for a weekly hand-delivered batch. Take the money manually.

Total cost: a script and two weeks. It measures the exact thing the entire business rests on — will a founder in this segment part with money for drafted posts — with no dependency on scheduling, approvals, or the activation cliff.

### Kill criteria — decide by 17 September 2026

Stop the project if any of these holds:

- **< 12 of 30** founders reply positively to the free drafts. The drafts aren't compelling; nothing built on top of them will be.
- **< 5 of 30** pay $19 within 30 days of the ask. Willingness to pay is absent, and every downstream number in §2 is a fantasy.
- The positive replies say *"nice, but I already use [X]"* more often than *"how do I get more of these"* — the job is already served.

Stop later, after any build begins, if:

- Activation (store connected) is **< 25%** of signups — the `.p8` onboarding cliff is fatal and no amount of downstream quality recovers it.
- **Drafted-post share < 40%** at day 30 — the PRD's own experiment ([§5 corollary](./PRD.md)) failing means AppTap is a worse Buffer.
- Month-3 logo retention **< 70%** ([PRD §13 target](./PRD.md)) — at 7%+ monthly churn the steady state is a hobby regardless of acquisition.

### What to do in the next two weeks

1. Write the draft-generation script against public store data. (2 days)
2. Build the list of 30 founders. (1 day)
3. Send, personally, over 5 days. Track reply rate and the exact wording of objections.
4. Make the $19 ask to everyone who replies positively.
5. On 17 September, count. **Five payers or stop.**

Do not open App Store Connect API documentation, do not start a Meta App Review, and do not write a platform adapter until that count is in.

---

## Sources

- [42matters — iOS App Store statistics](https://42matters.com/ios-apple-app-store-statistics-and-trends) · [Google Play statistics](https://42matters.com/google-play-statistics-and-trends)
- [Sensor Tower — abandoned apps](https://sensortower.com/blog/abandoned-apps)
- [RevenueCat — State of Subscription Apps 2026](https://www.revenuecat.com/state-of-subscription-apps) · [Subscription Insider summary](https://www.subscriptioninsider.com/article-type/news/revenuecat-data-shows-subscription-app-growth-concentrating-at-the-top)
- [X API pricing 2026 — all tiers](https://postproxy.dev/blog/x-api-pricing-2026/) · [X API cost per post](https://opentweet.io/blog/x-api-cost-2026)
- [Buffer pricing 2026](https://www.blotato.com/blog/buffer-pricing) · [Appfigures pricing 2026](https://www.g2.com/products/appfigures/pricing)
- [Meta App Review now takes 20 days](https://bundle.social/blog/meta-app-review-20-days) · [Instagram API integration guide 2026](https://www.getphyllo.com/post/instagram-api-integration-101-for-developers-of-the-creator-economy)
- [Shipstar — changelog to social posts](https://shipstar.ai/solutions/turn-changelog-into-social-posts) · [SocialClaw](https://getsocialclaw.com/blog/turn-your-saas-changelog-into-social-posts-automatically) · [PersonaBox](https://personabox.app/blog/best-changelog-tools)
- [AppFollow Q2 2026 updates](https://appfollow.io/blog/appfollow-q2-2026-steam-beta-ai-reply-style-and-report-a-concern-for-everyone)
- [SaaS free-to-paid conversion benchmarks](https://userpilot.com/blog/saas-average-conversion-rate/) · [SaaS churn benchmarks 2026](https://www.koji.so/blog/saas-churn-rate-benchmarks-2026)
- [r/iOSProgramming stats](https://gummysearch.com/r/iOSProgramming/) · [Indie hacker communities 2026](https://www.vibecontentcreation.com/blog/indie-hacker-communities-where-solo-founders-hang-out)
- [Indie app marketing 2026](https://apsteq.com/blog/indie-app-marketing/)
