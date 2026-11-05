# 06 — Information Architecture

---

## 1. Application Modes

The application operates in three distinct modes based on audience:

| Mode | Audience | Access Device | Key Characteristics |
|------|----------|--------------|-------------------|
| **Big Screen Mode** | Whole war room, visitors, CM Vijay | Wall display / kiosk | Auto-refresh, no interaction needed, can be presentation-safe |
| **Team Mode** | Media, social, content teams | Desktop / tablet | Full interactive access, drill-down, correction, Q&A |
| **Admin Mode** | War Room Admin, Minsky team | Desktop | Configuration, audit, accuracy monitoring |

---

## 2. Top-Level Navigation (Team Mode)

```
War Room App
├── Dashboard (Home)
│   ├── Entity Sentiment Summary (TVK / CM Vijay / AadhavArjuna)
│   ├── Ally Mood Panel
│   ├── Active Alerts (top 3 shown; full list via "All Alerts")
│   └── Rising Stories
│
├── Stories
│   ├── All Stories (filterable by entity, source type, date range, severity)
│   ├── Story Detail
│   │   ├── Mentions (X posts + news articles + e-paper + clippings)
│   │   ├── Reach breakdown (N outlets, N posts)
│   │   ├── Timeline
│   │   ├── Labels + corrections
│   │   └── Campaign link (if campaign-flagged)
│   └── Response Queue (Stories needing TVK reaction)
│
├── Alerts
│   ├── Active Alerts
│   │   ├── Volume Spikes
│   │   ├── Influence Triggers
│   │   ├── Campaign Signals
│   │   └── Response Opportunities
│   ├── Alert History
│   └── Alert Detail
│       ├── Explanation (2-line co-worker summary)
│       ├── Evidence (Mentions)
│       ├── Acknowledge / Assign
│       └── Campaign detail (if applicable)
│
├── Campaigns
│   ├── Active Campaigns
│   ├── Campaign Detail
│   │   ├── Accounts + ages
│   │   ├── Shared phrasing
│   │   ├── Timing chart
│   │   ├── Confidence level
│   │   ├── Confirm Attribution (lead only)
│   │   └── Spread Timeline
│   └── Campaign History
│
├── Entities & Parties
│   ├── Tracked Entities (TVK, CM Vijay, AadhavArjuna)
│   ├── All Parties (relationship tags, mood indicators)
│   └── Ally Detail View (mentions from ally public figures)
│
├── Co-worker (AI Chat)
│   ├── Chat Interface (Tamil / English Q&A)
│   ├── Suggested Questions
│   └── Answer History
│
├── Uploads
│   ├── Upload Clippings
│   └── Upload History / Status
│
└── Admin (War Room Admin / System Admin only)
    ├── Configuration
    │   ├── Watch Terms
    │   ├── Standing Tiers
    │   ├── Outlet Weights
    │   ├── Party Relationship Tags
    │   ├── Alert Thresholds
    │   ├── Alert Recipients
    │   └── Event Calendar
    ├── Accuracy & Learning
    │   ├── Weekly Accuracy Chart (model-only vs. corrected)
    │   ├── Co-worker Stage Dashboard
    │   ├── Correction Rate per Language/Entity
    │   └── Gold Set Management
    ├── Audit Logs
    │   ├── Configuration Changes
    │   ├── Corrections
    │   ├── Campaign Confirmations
    │   └── Alert Actions
    ├── Platform Health
    │   ├── Source Status (X, news outlets, e-paper)
    │   ├── API Usage vs. Limits
    │   └── WhatsApp Delivery Status
    └── Filter Audit
        ├── Weekly Sample Review
        └── Missed-Item Rate Chart
```

---

## 3. Big Screen Mode Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  [WAR ROOM LOGO]  Last updated: 09:42   [DATA DELAYED: X 2 min]    │
├─────────────────┬─────────────────┬────────────────┬───────────────┤
│    TVK          │   CM VIJAY      │  AADHAV ARJUNA │  ALLY MOOD    │
│  64% positive   │  51% positive   │  58% positive  │  Congress 🔴  │
│  of 312 mentions│  of 187 mentions│  of 94 mentions│  AIADMK ✅   │
├─────────────────┴─────────────────┴────────────────┴───────────────┤
│  ACTIVE ALERTS (3)                                                   │
│  🔴 [Alert 1 - HIGH] Influence trigger: Journalist name, 1.2M fol  │
│  🟡 [Alert 2 - MED]  Volume spike: Negative mentions TVK +180%     │
│  🟡 [Alert 3 - MED]  Possible campaign: AadhavArjuna hashtag       │
├─────────────────────────────────────────────────────────────────────┤
│  RISING STORIES                                                       │
│  1. [Story title] — 9 outlets, 1,200 posts — ↑ rising fast         │
│  2. [Story title] — 3 outlets, 450 posts — ↑ rising                │
│  3. [Story title] — 1 outlet, 89 posts — new                       │
└─────────────────────────────────────────────────────────────────────┘
```

**Presentation-Safe Mode (CM visit):** Hides all private citizen handles and internal notes. Shows only aggregates, public actor names, Story titles.

---

## 4. CM Summary Mode Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [TVK Elephant Avatar] SITUATION SUMMARY — Today, 18:00         │
│  Reviewed by: Meena                                             │
├─────────────────────────────────────────────────────────────────┤
│  3 changes since last summary:                                   │
│                                                                  │
│  1. ⚠️ AadhavArjuna mentions turned more negative              │
│     Cause: 300-account hashtag campaign, medium confidence      │
│     [See evidence →]                                            │
│                                                                  │
│  2. 📰 New article: Journalist criticised CM Vijay              │
│     Cause: Verified, 1.2M followers, now on national TV         │
│     [See evidence →]                                            │
│                                                                  │
│  3. 🤝 Congress ally showing discontent                         │
│     Cause: 3 Congress MLAs posting about cabinet decision        │
│     [See evidence →]  reviewed by Meena                        │
├─────────────────────────────────────────────────────────────────┤
│  Decision needed: Response to journalist criticism               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Key Entity Relationships in the UI

```
Mention
  ├── belongs to 1 or more Entities
  ├── belongs to 0 or 1 Story
  ├── has Labels (entity, stance, claim type, language)
  ├── has Influence Weight
  └── has 0 or 1 Correction

Story
  ├── contains 1 or more Mentions (across source types)
  ├── has reach breakdown by source type
  ├── may be linked to a Campaign
  └── may be in the Response Queue

Alert
  ├── linked to 1 or more Mentions or Stories
  ├── has type (spike / influence / campaign / response-opportunity / ally)
  ├── has severity tier (1 / 2 / 3)
  └── has acknowledgement and assignment records

Campaign
  ├── linked to a set of Mentions
  ├── has accounts list with ages
  ├── has confidence level
  ├── has attribution status (possible / confirmed + who / dismissed)
  └── has spread timeline

Entity
  ├── TVK, CM Vijay, AadhavArjuna → primary tracked entities
  ├── Parties → with relationship tag
  └── Watch terms (names, nicknames, hashtags) → per entity

Account / Outlet
  ├── Influence score (composition visible)
  ├── Tier assignment
  └── Bot suspicion flag
```

---

## 6. Navigation Design Principles

1. **Numbers always open their evidence** — tapping any metric reveals the Mentions behind it.
2. **Source always visible** — every Mention shows outlet/account and type tag.
3. **Timestamps on everything** — every number shows when it was last updated.
4. **Corrections never hide originals** — history always accessible.
5. **Co-worker always reachable** — persistent chat button available on all screens in Team Mode.
6. **Source health always visible** — delayed sources shown at the top of Dashboard, never silently dropped.
7. **Weighted and raw always together** — no weighted score appears without raw count beside it.
