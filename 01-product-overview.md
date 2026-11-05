# 01 — Product Overview

**Document:** War Room Social Listening, Phase 1 (X + News)  
**Client:** TVK (Tamilaga Vettri Kazhagam)  
**Status:** Draft for CTO review  
**PRD Owner:** Kishore (Minsky product), 2026-11-03

---

## 1. Product Vision

Give the TVK war room one live, trustworthy picture of how X (Twitter) and the news are talking about TVK, CM Vijay, and AadhavArjuna — with an AI co-worker that explains it and alerts the team in time to act.

## 2. Business Context

TVK formed the Tamil Nadu government on 2026-05-10. The war room operates across media, social media, and content teams, with AadhavArjuna as head of the war room. The political communication environment is fast-moving: coalition dynamics, coordinated opposition campaigns, and the speed of social media mean that missing a key moment by even hours has measurable political cost (see the councillor incident in the PRD).

## 3. The Core Problem

The TVK war room learns about important moments too late. Specifically:

- **Response opportunities are missed** — attack or condemnation happens, allies and media react, TVK stays silent because nobody flagged it in time (confirmed real incident: councillor's remarks about CM Vijay).
- **High-influence single posts are invisible** — volume-based monitoring misses the one journalist with 1M followers whose post drives tomorrow's TV coverage.
- **Coordinated surrogate campaigns go undetected** — 300 new accounts posting near-identical Tanglish overnight is invisible to any person scrolling a feed.
- **Ally discontent is not watched** — the team tracks rivals by habit, missing coalition partners' unhappiness until it escalates.
- **Print and digital are siloed** — a column in a Tamil weekly and the X chatter it triggers are never connected into one Story.

## 4. The Solution in One Sentence

One system that collects everything said about TVK on X and in the news, labels and weighs it, groups it into Stories, spots coordinated campaigns, and alerts the war room, with an AI co-worker that explains all of it.

## 5. The Seven Processing Steps

```
1 Collect → 2 Filter → 3 Label → 4 Weigh → 5 Group into Stories → 6 Detect Campaigns → 7 Rank and Alert
                                                                                              ↑
                                                               AI Co-worker across all steps  │
                                                               Corrections feed back ─────────┘
```

| Step | What Happens |
|------|-------------|
| 1. Collect | X posts (engagement + profiles), online news (feeds + crawling), e-paper editions, scanned newspaper clippings |
| 2. Filter | Wide net — keep anything that could concern TVK. Audited weekly. Missing an item is worse than including too much. |
| 3. Label | Entity, stance (positive/negative/neutral/sarcastic), claim type (on record / anonymous / satire), language. Reason shown for every label. |
| 4. Weigh | Influence score per account/outlet (verified status, followers, engagement velocity, staff-curated tier). All weights visible and editable. |
| 5. Group into Stories | Items about the same event across X and news → one Story with reach split by source type. Staff can merge/split. |
| 6. Detect Campaigns | Coordination signals: new accounts clustering, near-identical text, timing patterns. Confidence shown. Human confirms party attribution. |
| 7. Rank and Alert | Big screen (top tier), WhatsApp to leads (top tier), in-app (next tier). Number on big screen capped to prevent noise. |

## 6. The AI Co-worker

The co-worker (displayed with the TVK elephant avatar) works across all seven steps:

- Answers questions in Tamil or English
- Explains every spike and alert with the posts behind it
- Writes the CM summary and shift handovers
- Recommends actions (never takes them)
- Learns from every correction the team makes

The co-worker grows through three stages based on measured accuracy:

| Stage | What It Does | What People Do |
|-------|-------------|----------------|
| 1. Assist | Suggests labels, alerts, responses | Review most output; correct freely |
| 2. Trusted | Acts alone on capabilities that cleared accuracy bars | Spot-check; review low-confidence and high-stakes items |
| 3. Co-worker | Handles routine monitoring, labelling, alerting, briefing end-to-end | Make decisions; handle exceptions |

Stage movement is automatic when measured accuracy on the team's corrections clears defined bars. Stages drop back automatically if accuracy falls.

**The co-worker never publishes.** In Phase 1, it cannot post, reply, or message anyone outside the war room.

## 7. Three Views

| View | Audience | Purpose |
|------|----------|---------|
| Big Screen | Whole war room; visitors; CM Vijay | Live situation picture at a glance |
| CM Summary | CM Vijay | 30-second read: what changed, why, what needs a decision |
| Team Drill-Down | Media/social/content teams | Full access to Mentions, labels, evidence, correction |

## 8. What Is in Scope (Phase 1)

| Item | Detail |
|------|--------|
| X ingestion | Posts, replies, quotes, engagement, author profiles; history from 2026-05-10 |
| News ingestion | Tamil and English outlets from the web: feeds + crawling + TV news + YouTube channel feeds |
| E-paper ingestion | From existing war room subscriptions, with Tamil text recognition |
| Clipping upload | Daily scanned newspaper cuttings → Mentions |
| Entities and relationships | TVK, CM Vijay, AadhavArjuna, all Indian parties and key leaders; each party tagged ally/supporting/opposition/neutral |
| Sentiment and stance | Per entity, per language (Tamil/English/Tanglish), with reason per label |
| Influence scoring | Account and outlet influence scores; weighted and raw counts always shown together |
| Stories | One Story per happening, reach split by source type |
| Alerts | Volume spikes, influence triggers, campaign signals → big screen / WhatsApp / in-app |
| Campaign detection | Coordination signals with confidence and evidence; human confirmation before naming a party |
| Three views | Big screen, CM summary, team drill-down |
| AI co-worker | Q&A, explanations, briefs, handovers, recommendations, memory; elephant avatar |
| Human correction and learning | Any AI output correctable; corrections improve system; bias guard |
| Interface languages | English and Tamil |

## 9. What Is Out of Scope (Phase 1)

| Item | Reason |
|------|--------|
| Publishing, replying, or campaigning | Founder decision: bring in data first; automated counter-campaigning is a later phase |
| WhatsApp group monitoring | Later phase, requires legal review |
| Instagram and Facebook | Later phase; access method under research |
| Other clients (parties, organisations) | Later phase; but the build must be generalisable without code changes |
| Multiple X accounts to bypass limits | Violates X Developer Policy |
| Private citizen profiles | Privacy: analysis covers conversations and public actors only |
| Paywalled content beyond subscriptions | Only content the war room is entitled to read |
| Draft responses in TVK's voice | Its own separate project |

## 10. Generalisability Requirement

**This is a Phase 1 requirement, not a Phase 1 feature.** The build must be architected so that a new client (another political party, a brand, any organisation) can be onboarded through configuration alone — no code changes.

What must be per-client configuration:
- Tracked entities and relationship tags
- Watch terms per entity
- Standing tiers and their members
- Outlet lists and weights
- Alert recipients and routing
- Thresholds
- Co-worker name and avatar
- UI vocabulary ("party" / "competitor", "ally" / "partner", "CM summary" / "leadership summary")
- Languages
- Each client's data, corrections, gold set, and co-worker memory are fully isolated

## 11. Must-Pass Tests (Pre-Launch)

Two scenarios must be reproduced from real or realistic data before launch:

**Test 1 — The Councillor Incident:**  
A rival party's speaker attacks CM Vijay at a public event; news reports and others' condemnations spread; no TVK leader has responded. Sajan (named lead) gets a "Response opportunity" on WhatsApp within 5 minutes of the trigger, with evidence and who is already reacting.

**Test 2 — The One Post That Mattered:**  
A verified national journalist's sharply critical post about CM Vijay, with no volume spike, reaches the big screen and team leads within 5 minutes of posting, with its reason shown.

## 12. Success Criteria Summary

| Category | Key Metric | Target |
|----------|-----------|--------|
| Freshness | Post-to-screen time, p95 | ≤5 minutes |
| Usefulness | Share of alerts acted on (not dismissed) | ≥60% (calibration default) |
| Accuracy | F1 per language | English 0.80, Tamil 0.75, Tanglish 0.72 (calibration defaults) |
| Coverage | Missed-item rate (weekly audit) | Trending down |
| Campaign detection | Precision (share confirmed by leads) | Calibration |
| Co-worker quality | Eval-set accuracy, citation correctness | Calibration |
| Learning | Correction rate per 1,000 labels | Falling week on week |
| Daily use | Active users per day | Every lead daily |

**Stop or roll back if:** Most big-screen alerts are dismissed after calibration; Tamil/Tanglish label accuracy stays below agreed bars; a privacy incident occurs.
