# AppTap

**Status: build paused. Validating demand first.** · Decision date: **17 September 2026**

AppTap reads what is happening to a mobile app — a new release, a five-star review, a download milestone — turns it into social posts, adapts them per platform, and publishes them on schedule. The founding documents for it are complete and in [`docs/`](./docs/).

No code has been written, and none will be until a two-week demand test comes back positive.

---

## Why the build is stalled

The product documents were finished before anyone did the arithmetic on customers and money. When that arithmetic was done ([`docs/market-viability.md`](./docs/market-viability.md)), it did not support starting the build. Four things decided it.

**1. The realistic outcome is a break-even side project, not an income.**
Working bottom-up from publisher counts, activation rates, developer-tool conversion benchmarks and SMB churn, the base case lands at roughly **60 paying customers and ~$2,340 MRR** at steady state. That covers infrastructure and leaves about $1,500 a month. Paying the founder a modest $5,000 a month needs **171 customers** at the planned $39 blended price. Only the optimistic scenario clears that bar, and it requires six independent assumptions to hit their ceilings simultaneously.

**2. The target segment mostly cannot afford the product.**
The median subscription app earns **$492 per month**, and **57.7% of new apps never reach $1,000 in total revenue** ([RevenueCat, 2026](https://www.revenuecat.com/state-of-subscription-apps)). The founder whose blank text box AppTap solves is, statistically, the founder whose app is not yet earning. Meanwhile the price anchors in the category are Appfigures at $9.99 and Buffer at $5 per channel.

**3. Four to six months of unpaid work sits in front of the first dollar.**
Publishing to Instagram, YouTube and TikTok requires three separate platform approvals. Meta App Review alone now averages ~20 days per round, before business verification and before the first rejection. Full five-platform publishing realistically lands in month 7–10.

**4. The wedge is thinner than the strategy documents claim.**
The claim that no product joins app-store data to social publishing is still literally true. But the underlying job — turn your shipping activity into posts automatically — is already sold by Shipstar, SocialClaw and PersonaBox, which read GitHub instead of App Store Connect and onboard with one OAuth click instead of a `.p8` key upload. Separately, app-store events yield maybe 4–8 postable moments a month against the 12–20 a real posting cadence needs. AppTap solves the first post, not the habit.

None of this says the idea is bad. It says the idea has one load-bearing assumption — that indie app founders will pay for drafted posts — and that assumption has never been tested, while the plan on the table spends six months before testing it.

---

## What happens instead

A two-week manual concierge test. No product, no OAuth, no API approvals, no code beyond a script.

1. Pick 30 indie apps whose founders are publicly reachable. Everything needed is public: version history, release notes, recent reviews, ratings.
2. Generate three post drafts for each — one from the latest release, one from a recent five-star review, one from a milestone.
3. Send them cold and personally: *"I made these for your app — want three more every week?"*
4. On a positive reply, ask for **$19/month, paid now**, for a weekly hand-delivered batch.

This measures the one thing the whole business rests on, with no dependency on scheduling, platform approvals, or the App Store Connect onboarding cliff.

### Kill criteria

Stop the project if any of these holds on 17 September:

- Fewer than **12 of 30** founders reply positively to the free drafts.
- Fewer than **5 of 30** pay $19 within 30 days of the ask.
- Positive replies skew toward *"nice, but I already use something for this"* rather than *"how do I get more of these."*

### If the test passes

Build the **draft engine only** first — app data in, drafts out, copy to clipboard. It needs zero platform approvals, costs roughly $3 per customer per month to run, and is the only part of AppTap that Buffer and AppFollow cannot copy this quarter. Price it for 3–5-app studios at $79–149 rather than solo founders at $29; that alone drops salary break-even from 171 customers to about 79. Drop X from v1 regardless — it is the only unbounded variable cost in the product, at $0.20 per post containing a link.

The scheduling and publishing layer is built later, if at all. It is commoditised, priced at zero by eight incumbents, and gated behind ten weeks of approvals.

---

## Documents

| Document | What it covers |
|---|---|
| [`docs/market-viability.md`](./docs/market-viability.md) | Demand arithmetic, unit economics, break-even, ranked alternatives, kill criteria. **Read this first.** |
| [`docs/PRD.md`](./docs/PRD.md) | Product requirements: users, features, flows, pricing hypothesis, risks. |
| [`docs/competitive-analysis.md`](./docs/competitive-analysis.md) | The competitive landscape and the argument for the app-data wedge. |
| [`docs/platform-api-feasibility.md`](./docs/platform-api-feasibility.md) | What YouTube, Instagram, TikTok, Threads and X actually permit. |
| [`docs/design-doc.md`](./docs/design-doc.md) | Technical design: data model, state machines, publish and retry mechanics. |

The PRD, competitive analysis and design doc were written before the viability assessment. Where they conflict with it — particularly on pricing, segment and build sequence — the viability assessment is the current position.

Pricing and platform API terms in these documents were retrieved on 3 September 2026 and change frequently. Re-verify before quoting any figure externally.
