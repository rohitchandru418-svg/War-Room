# 04 — User Flows and Workflows

---

## 1. The Typical Day (Primary Reference Flow)

This is the canonical user journey provided by the PRD (Meena, social media team lead).

```
06:30  WhatsApp notification (overnight summary + high alert)
  └── Tap → alert detail (accounts, phrasing, timeline)

07:15  Clipping upload
  └── Intern scans morning newspaper cuttings
  └── Upload via app → OCR processing → Mention creation
  └── Within the hour: Mention joins relevant Story

09:00  Big screen morning review
  └── Sentiment for TVK / CM Vijay / AadhavArjuna
  └── Ally Mood panel
  └── Top 3 active alerts
  └── Top rising Stories

11:40  Real-time alert fires (high-influence post)
  └── Screen flashes red
  └── Alert shown: "verified, 1.2M followers, national media tier, 4,000 reposts in 20 min"
  └── Lead acknowledges → assigns to content team → flash clears for everyone

12:10  Co-worker Q&A
  └── Ask in Tamil: "Why did Congress mentions turn negative?"
  └── Co-worker answers: 3 posts from Congress MLAs, negative share before/after, sample size

15:30  Label correction
  └── Tanglish post marked Negative identified as sarcastic support
  └── Lead changes to Positive (sarcastic) + one-line reason
  └── Original label retained in history

18:00  CM visit → presentation-safe mode activated
  └── CM reads 30-second summary: what changed, why, what needs a decision

22:00  Shift handover
  └── Co-worker drafts handover: open alerts, owners, what to watch overnight
  └── Outgoing lead reviews, edits if needed, confirms send
```

---

## 2. Data Ingestion Flows

### 2A. X (Twitter/X) Ingestion

```
X API (streaming or polling)
  └── Post received with: text, author, engagement metrics, timestamp, language tag
  └── Language detection (own model — X language tag not reliable for Tanglish)
  └── Watch-term filter: does this mention a tracked entity?
      ├── No match → store raw (not shown), included in weekly audit sample
      └── Match → queue for Label step
          └── Label: entity, stance, claim type, language (reason attached)
          └── Weigh: author influence score (tier × verified × followers × engagement velocity)
          └── Story matching: does this belong to an existing Story?
              ├── Yes → add to Story, update Story reach
              └── No → create new Story seed
          └── Campaign detection: add to pattern analysis
          └── Severity scoring: influence × negativity × velocity
          └── Alert routing: top-tier → big screen + WhatsApp; mid-tier → in-app; low → Story only
```

### 2B. Online News Ingestion

```
Per outlet (checked periodically):
  ├── RSS/Sitemap feed available? → Parse feed
  ├── No feed → crawl outlet website directly (robots.txt compliant, identified, paced)
  └── Article not yet tracked → search API discovery

For each article:
  └── Extract: headline, snippet, publication time, author (if available), outlet
  └── Watch-term filter
  └── OCR not needed (HTML text)
  └── Label + Weigh (outlet tier) + Story matching + Alert routing (same as X flow)

Outlet health monitoring:
  └── If crawl/feed fails → "data delayed since HH:MM, source: <outlet>" shown to admins
  └── Error never shown as "no new articles"
```

### 2C. E-Paper Ingestion

```
Daily (morning):
  └── E-paper editions fetched from subscription access
  └── Tamil OCR applied to pages
  └── Low-confidence text → flagged "check scan"
  └── Each story → Mention tagged "e-paper" with page image attached
  └── Watch-term filter → Label + Weigh + Story matching
```

### 2D. Clipping Upload Flow

```
Team member (usually intern):
  └── Scan physical newspaper pages (image files)
  └── Upload via app (drag-drop or file picker, mobile camera)
  └── System: OCR (Tamil + English) → extract text
  └── Auto-fill: outlet name (from scan or manual entry), date, page number
  └── Low-confidence OCR → flagged "check scan"
  └── Confirm and submit → becomes Mention
  └── Mention joins Story matching flow
```

---

## 3. Alert Handling Flows

### 3A. High-Influence Single Post Alert

```
Post ingested
└── Influence score calculated
└── Score >= top-tier threshold AND sentiment negative?
    └── YES → alert created with reason
        └── Big screen: flash red, alert card shown
        └── WhatsApp: sent to team leads within 5 min of post
        └── In-app: alert notification
        └── Lead sees: account, followers, tier, engagement numbers, reason
        └── Lead: acknowledge → flash clears for all with "taken by <name>"
        └── Lead: assign to team member → alert shows assignee
```

### 3B. Volume Spike Alert

```
Every 15 minutes [DEFAULT]:
  └── Calculate negative mentions for entity in current hour
  └── Compare to baseline (same hour, 14 days) → Z-score
  └── Conditions met (≥3σ, ≥2× normal, ≥25 mentions)?
      └── YES + event window? → use event baseline
      └── Create spike alert: show baseline, current level, top driving posts
      └── Route by severity (big screen / in-app)
```

### 3C. Coordinated Campaign Detection

```
Continuous batch analysis:
  └── Clustering: accounts posting same/similar text in short window
  └── Signals: account age distribution, hashtag spread, timing patterns, boosting patterns
  └── Confidence calculated
  └── If signals cross threshold:
      └── Alert: "possible coordinated campaign" (never names party yet)
      └── Shows: account list, ages, shared phrasing, timing chart, confidence
      └── Lead reviews → Confirm attribution?
          ├── YES → logs: lead name, reason, timestamp → campaign confirmed
          └── NO → remains "possible coordination" or dismissed
      └── If confirmed: campaign spread page activated (mentions/hour, new accounts, TVK responses)
```

### 3D. Response Opportunity Alert

```
Story ingested where:
  └── Entity attacked OR controversial remark made
  └── Condemnations from non-TVK parties/media/public figures accumulating
  └── No TVK leader account has posted a response (monitored continuously)
  └── Trigger crossed? → Alert to Sajan (WhatsApp) within 5 min:
      "Response opportunity: [evidence], who is reacting: [list], no TVK response yet"
  └── TVK leader posts response?
      └── System detects via TVK leader account monitoring
      └── Opportunity updated: "responded by <name> at HH:MM"
      └── Alert stops
```

### 3E. Ally Mood Alert

```
Mentions from ally party public figures (official handles, MPs, MLAs, spokespeople):
  └── Labelled for stance toward TVK
  └── If ally mentions turn negative (threshold crossed):
      └── Ally Mood panel updated
      └── Alert fired at ally threshold
      └── Show: which ally, which figures, which posts
```

---

## 4. Co-worker Q&A Flow

```
User types question (Tamil or English)
  └── Co-worker receives question
  └── Language detected → answer will be in same language
  └── Queries: Mentions database, Stories, correction history, party relationships, alert history
  └── Builds answer: every claim cites specific Mentions
  └── Every count comes from live data query (not model memory)
  └── If data insufficient → "not enough data" stated explicitly
  └── Answer delivered within 10 seconds [DEFAULT] for typical questions
  └── Answer displayed with citations visible (tap citation → see source Mention)
  └── User can mark answer as useful/not useful
  └── Answer logged: inputs, sources, model version, prompt version
```

---

## 5. Label Correction Flow

```
Team member views Mention label
  └── Disagrees with label?
      └── Tap "correct" → opens correction form
      └── Select new label (entity / stance / claim type / language)
      └── Enter one-line reason (required)
      └── Submit
      └── System:
          └── Original label retained in history (immutable)
          └── New label shows "corrected by <name>"
          └── Aggregates recalculate with corrected label (corrected counts identifiable)
          └── Is this a softening correction on a TVK entity OR a high-severity Story?
              └── YES → Requires second reviewer before aggregate changes
              └── Second reviewer approves / rejects
          └── Correction added to gold set queue
          └── Weekly accuracy chart updated
```

---

## 6. CM Summary Flow

```
Trigger: scheduled (e.g. before CM visit) or on-demand
  └── Co-worker queries: changes since last summary (new high-severity Stories, resolved alerts, ally mood shifts)
  └── Selects up to 3 most significant changes
  └── For each change: one-line cause + link to evidence Mentions
  └── Checks for ally attribution: does any change blame an ally?
      └── YES → flag for lead review: "reviewed by <name>" or held until reviewed
  └── Summary delivered in English or Tamil (CM preference)
  └── CM opens CM mode → 30-second summary shown
  └── Tap any change → evidence Mentions
  └── Big screen switches to presentation-safe mode if CM is visiting in person
```

---

## 7. Shift Handover Flow

```
Shift end time reached (time configured per client):
  └── Co-worker queries: open alerts + owners, top Stories still active, overnight watch items
  └── Drafts handover document:
      "Open alerts: [list with owners]
       What to watch overnight: [reasoning from recent patterns]
       High-risk Stories: [any with still-rising velocity]"
  └── Outgoing lead receives draft in-app
  └── Lead reviews, edits if needed
  └── Confirms → handover distributed (in-app notification to incoming team)
```

---

## 8. Configuration Change Flows

### 8A. Standing Tier Change
```
War Room Admin selects tier management
  └── Add/remove/edit account handle in a tier
  └── Confirm change
  └── System: immediate effect on new Mention weighting
  └── Audit log entry: admin name, timestamp, tier, account, change type
```

### 8B. Party Relationship Tag Change
```
War Room Lead or Admin selects party
  └── Changes tag: ally / supporting / opposition / neutral
  └── Confirm
  └── System: immediate effect on alerts and panels
  └── Historical Mentions retain their original tag for the period they were logged
  └── Audit log entry: user name, party, old tag, new tag, timestamp
```

---

## 9. Weekly Filter Audit Flow

```
Weekly (automated trigger):
  └── System samples rejected items (stratified by source, language, entity)
  └── Sample assigned to Minsky reviewer(s) in admin interface
  └── Reviewers re-label items: should this have been kept?
  └── Missed-item rate calculated: (items that should have been kept / total sample)
  └── Rate shown on admin page
  └── If rate above threshold → watch-term list or filter settings reviewed
```

---

## 10. Must-Pass Test Flows (Pre-Launch Validation)

### Test 1: Councillor Incident
```
Input: news reports of rival-party speaker attacking CM Vijay at a public event
       condemnation posts from other parties/media accumulating
       TVK leader accounts: no response
Timer starts at: earliest qualifying condemnation post/article crossing trigger threshold
Expected: Sajan receives WhatsApp "Response opportunity" within 5 minutes
Alert must include: triggering evidence, list of who is reacting, "no TVK response yet"
Pass criteria: delivery timestamp − trigger timestamp ≤ 5 minutes
```

### Test 2: One Post That Mattered
```
Input: single post from verified national journalist (top tier) criticising CM Vijay
       no volume spike (this is a single post)
Timer starts at: post published timestamp
Expected: big screen flashes + team leads notified within 5 minutes
Alert must include: reason ("verified, Xm followers, national media tier, N reposts in Y min")
Pass criteria: delivery timestamp − post timestamp ≤ 5 minutes
```
