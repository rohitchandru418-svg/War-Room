# D03 — Screen Designs

Screen-by-screen design briefs for the designer or design tool. Visual language: **D01**. Components: **D02**. Behaviour, fields and states: `06-information-architecture.md`, `07-screen-specifications.md`, `08-interaction-and-state-specifications.md` (screen numbers below follow doc 07; if numbering differs, doc 07 wins). Each brief gives purpose, who sees it, layout, key content, states and responsive notes.

Sample content is **illustrative placeholder content** drawn from the PRD's scenarios. Use generic names ("National journalist, verified"), not real people's quotes. Tamil strings are placeholders for native review.

Frame sizes to produce for every screen: **Desktop 1440×900** (light), **Desktop dark**, **Tablet 834×1112**, **Mobile 390×844**; Big screen **1920×1080** (dark). Provide English and Tamil variants of the key screens (1, 3, 5, 10, 11).

Legend in wireframes: `[ ]` button · `( )` tag · `▲▼–~` stance icons · `█` bar · `>` peek opens.

---

## Screen 0 — Sign-in

**Who:** all staff. Separate **PIN** variant for the CM device (CM View).
**Layout:** split screen. Left: calm white panel with the client logo, "Sign in", email, password, `[Continue]` (primary), "Forgot password". Right (desktop only): brand `#A00000` panel with the co-worker avatar and one line ("Know what matters, first."). No sign-up, no social login.
**PIN variant:** large numeric keypad, 44 px keys, 30-minute inactivity timeout notice.
**States:** wrong password (plain message, no hints), locked after repeated failures ("Try again in 15 minutes or contact an admin"), password-reset sent.

## App shell (all screens except 0, 10)

Sidebar + top bar + page content (D02 §1). Sidebar items depend on role; CM role sees only the CM summary.

---

## Screen 1 — Dashboard

**Purpose:** one calm picture of where we stand and what needs attention.
**Who:** all staff roles; default landing page.

```
[sidebar] | 🏠 Dashboard                       Live · updated 12:04   [Presentation-safe ◯]  🔔  EN|தமிழ்
          | Window: (24 h ▾)  View: (Effective ▾)  Entity group: (All ▾)
          | ┌ Source banner (only if a source is behind): X data delayed since 11:52 ─────────────┐
          |
          | Sentiment                                    "of N mentions · last updated 12:04"
          | ┌────────────┐ ┌────────────┐ ┌────────────────┐
          | │ TVK        │ │ CM Vijay   │ │ AadhavArjuna   │   (StatWithDenominator + SentimentBar
          | │ 34% ▼ neg  │ │ 29% ▼ neg  │ │ Not enough data│    + WeightedRaw + TrendChip)
          | │ of 412     │ │ of 380     │ │ N=18, min 30   │
          | └────────────┘ └────────────┘ └────────────────┘
          |
          | Needs attention (max 3)                              All alerts ›
          | ▌TIER 1  Influence · National journalist (verified) criticises CM Vijay …  12 min ago  [Acknowledge][Assign]
          | ▌TIER 2  Spike · Negative mentions about TVK up 2.4× …  35 min ago
          |
          | Ally Mood                                  | Rising stories
          | Party A  ▬▬▬▬ calm  (of 64)                 | 1 Cabinet decision criticism   9 outlets, 1,200 posts
          | Party B  ▬▬▬▬ unhappy ▼ (of 41)  ›          | 2 Hashtag push (possible coordination)
          |
          | Response queue (3)                                                    Open queue ›
```
**Key content:** entity stat blocks (every number opens the evidence peek), ranked alerts (at most three; "+2 more in Alerts"), Ally Mood panel, rising Stories, response queue summary, source-health banner and freshness stamp.
**Interactions:** hover a segment for value and N; click any number → peek with mentions, label reasons, and the top three contributors; acknowledge/assign inline; window and view selectors (Effective vs Model-only).
**States:** loading skeletons; below min sample → `NotEnoughData`; stale/delayed (amber stamps + banner); empty alerts ("No active alerts. Last alert 2 h ago."); live-connection lost ("Reconnecting…").
**Responsive:** tablet two columns; mobile single column, alerts first, then entities.

## Screen 2 — Stories

### 2A Stories list (database view)
Table with properties: Story (icon + title), Reach ("9 outlets, 1,200 posts"), Sentiment (mini bar + N), Entities, Started, Status (Rising / Needs review / Unattributed), Response status. Filters: entity, source type, language, status, window. Row click → peek preview; "Open" → 2B.
**Empty:** "No stories match these filters."

### 2B Story detail (page)
```
📰 Cabinet decision criticised by ally MLAs                     (Rising) (Ally concern)
Started 09:12 · 9 outlets, 1,200 posts · Sentiment: ▬▬▬▬ of 212        [Merge…] [Split…]
─────────────────────────────────────────────────────────────────────────
Timeline:  09:12 first post ─ 09:40 news pickup ─ 10:05 alert fired ─ 10:20 acknowledged
Reach by source:  X 1,200 posts | News 9 outlets | E-paper 1 | Clipping 1
Why it matters (callout): two-line co-worker explanation + reason string
Mentions  [filters ▾]    ← MentionRow list: author/outlet · snippet · engagement · time · (stance) why? · source
Related alerts · Response status · Staff notes (internal; hidden in presentation-safe)
```
**Notes:** `MentionRow` label reasons open a popover; "Correct" opens `CorrectionPanel` in the peek. **Unattributed** single-source rumours appear as a Story of one with the orange marker.

### 2C Response queue
List of `ResponseQueueItem`s ordered by urgency: suggested response type, who has reacted, time since start, TVK response status. Actions: **Mark acted**, **Not needed**. Header copy: "Stories likely to need a reaction."

## Screen 3 — Alerts

### 3A Alert list
Filters: tier, type (Influence, Spike, Campaign, Ally, Response opportunity), status (new, acknowledged, assigned, resolved), entity, window. Each row is an `AlertCard` in compact form. Tier-1 unacknowledged pinned at top with the slow-pulse dot.

### 3B Alert detail (page; also the mobile landing from WhatsApp)
```
▌TIER 1   Influence alert · CM Vijay                                    12 min ago
▌"National journalist (verified) posts criticism of CM Vijay"
▌Reason: verified · 1.2M followers · national media tier · 4,000 reposts in 20 min
[Acknowledge]  [Assign ▾]  [Open Story]                       WhatsApp: delivered to 3 leads
Why this fired (callout, two lines from the co-worker)
Evidence  ← EvidenceList (original post link, engagement over time)
Suggested response (RecommendationCard — suggestion only)
History: fired 11:40 · WhatsApp accepted 11:41 · acknowledged by Meena 11:43 · assigned to Content team
```
**States:** WhatsApp not delivered (amber chip + "A lead has been notified"); taken by <name> (flash cleared for everyone); resolved; explanation pending ("Explanation will appear shortly"), explanation unavailable (co-worker down).
**Mobile:** full-width, actions pinned at bottom, 44 px targets.

## Screen 4 — Campaigns

### 4A List
Table: Campaign (hashtag/narrative), status (Possible coordination / Confirmed / Dismissed), confidence meter, accounts count, % accounts under 3 months, first seen, spread trend.

### 4B Campaign detail
```
(Possible coordination)  ConfidenceMeter ███░ Medium        Target: AadhavArjuna
Callout: "About 300 accounts, 70% under 3 months old, near-identical Tanglish lines in a short window."
Evidence tabs: Accounts | Shared phrasing | Timing | Spread over time
Timing chart (posts per 5 min) · Account-age histogram · Phrase clusters with counts
Attribution (leads only): "Naming a party as behind this campaign is permanent once shared."
  [Reason ______ (min 10 chars)]  [Confirm attribution…]  [Dismiss…]
```
**Rules:** wording stays "possible coordination" until a lead confirms; the attribution form is hidden or disabled for team members with an explanation line; confirmation is logged with name and reason (visible in History).

## Screen 5 — Co-worker

### 5A Ask (chat)
Centered document-like column (max 760 px). Header: co-worker avatar + name + stage badge. Message list per D02 §6 with citations `[1]`, "Based on 64 posts", 👍/👎 icons. Input at bottom with attach-none, send, language auto-detected. 3 suggested questions when empty. Persistent "Ask" button available from every page opens the chat as a right panel.
**States:** thinking, taking a moment (>10 s), unavailable (dashboards unaffected), not enough data answer, error with retry.

### 5B Briefs & handovers
List of drafts (CM summary, shift handover) as database rows (title, type, status Draft / Needs review / Sent, author "Co-worker draft", reviewer). Detail = `BriefEditor` (Notion-like doc). Required "Reviewed by <name>" before send when the draft attributes a change to an ally.

## Screen 6 — Upload clippings

Drag-and-drop zone (large, dashed hairline), camera button on mobile, per-file form (outlet, date, page), per-file status row: Processing → OCR complete → Mention created / **Check scan** (amber) / Error. After OCR: the extracted text with low-confidence words underlined for quick correction, then "Confirm". Duplicate files rejected with a clear message.
**Empty:** "Upload this morning's clippings to add print-only stories."

## Screen 7 — Settings (Admin)

Tabbed page with inline-editable tables (Notion database feel), audit trail icon on every row:
7A Entities & relationship tags (tag change shows "applies from now; history keeps the old tag") · 7B Watch terms (type: name, nickname, hashtag, Tamil, Tanglish) · 7C Standing tiers & members (national media, state media, party leaders, influencers) · 7D Outlets (tier, partisan flag with note, feed/crawl, health) · 7E Thresholds (value + default + plain-language explanation + last changed by) · 7F Alert recipients & routing (lead for response opportunities; **numbers masked**) · 7G Event calendar (rally, assembly session, results day with baseline note).
Confirm dialogs for changes that affect alert routing. Everything saves inline with "Saved · time". Roles: war-room admin and system admin only.

## Screen 8 — Source health and usage

Table of sources (X, each outlet, YouTube, e-paper, AI model, WhatsApp): `HealthDot`, last success, reason (e.g., "blocked by site", "rate limited"), delay duration. **X usage** bar showing reads vs monthly cap with markers at 50 %, 80 %, 95 % and a "Sampling active" notice. **WhatsApp delivery log:** time, recipient (masked), alert, status (accepted/sent/delivered/read/failed), retries.

## Screen 9 — Corrections & accuracy

9A **Accuracy overview:** per-capability `StageLadder` (labels per language, spike and influence alerts, campaign flags, response suggestions, briefs) with error trend. 9B **Weekly accuracy:** model-only vs corrected agreement and correction-rate per 1,000 labels by language and entity. 9C **Review queue:** second-review items (original vs proposed, who proposed, reason, Approve/Reject with reason). Bias-guard summary ("Corrections that soften criticism: 4 this week").

## Screen 10 — Big screen (`/bigscreen`)

Dark, 1920×1080, no scrolling, no pointer UI, large type (D02 §10).
```
┌──────────────────────────────────────────────────────────────────────────────┐
│ ● Live · updated 12:04                                  Presentation-safe    │
│                                                                              │
│  TVK             CM Vijay           AadhavArjuna                             │
│  34%  ▼          29%  ▼             Not enough data                          │
│  of 412          of 380             N = 18 (min 30)                          │
│                                                                              │
│  ALERTS (max 3)                                    ALLY MOOD                 │
│  ▌1 TIER 1  Influence — national media …           Party A  calm             │
│  ▌2 TIER 2  Spike — negative ×2.4 …                Party B  unhappy ▼        │
│  ▌3 …                                                                         │
│  RISING STORIES                                                              │
│  1 Cabinet decision criticism   9 outlets, 1,200 posts                       │
└──────────────────────────────────────────────────────────────────────────────┘
```
**Behaviour:** Tier-1 unacknowledged alert shows the brand treatment with a **slow pulse (≤ 1 Hz)**; clears for everyone when acknowledged ("taken by …"). Shows only aggregates, public actors' names and Story titles. **No post text** until the display-rights check passes. Data-delayed state shows an amber strip across the top with the source name and time. A "last updated" stamp is always visible.

## Screen 11 — CM summary

For CM Vijay; tablet-first, also phone and big display. Calm, light or dark, very large type, zero navigation chrome except language toggle.
```
CM summary · today 18:00                              EN | தமிழ்
What changed (max 3)
1  Negative mentions about the Cabinet decision rose after 11:00.        Evidence ›
   Why: three ally MLAs posted criticism; 64 posts.   Reviewed by <name>
2  …
3  …
What needs a decision
•  Respond to … (response opportunity open since 17:20)                 Evidence ›
```
**Rules:** at most three changes, each with a one-line cause and one tap to evidence; any change attributed to an ally shows "Reviewed by <name>" or the item is held back (internal view shows "Awaiting review"); presentation-safe redaction applies.

## Screen 12 — Operations pages (staff, lower fidelity)

- **12A Filter audit reviewer:** weekly sample of rejected items; reviewer marks "should have been kept"; missed-item rate chart on the admin page.
- **12B Audit log:** read-only table (who, what, old → new, when) with filters; visible to admins.
- **12C Notifications centre:** second-review requests, WhatsApp undelivered, stage demotion, X usage thresholds.

---

## Cross-screen elements

- **Evidence peek:** every number, story, alert and campaign opens it (D02 §1.4).
- **Presentation-safe mode:** toggle in the top bar (leads+); instantly reflected on all connected displays; badge visible; private-citizen handles and internal notes are replaced everywhere.
- **Language switch:** EN | தமிழ், remembered per user; all UI strings, dates and numbers formatted for the client timezone.
- **Theme:** light default for working pages, dark default for big screen; user switchable.
- **Mobile adaptation:** sidebar becomes a bottom sheet/tab bar (Dashboard, Alerts, Ask, More); tables collapse to cards.

## Prototype flows to link (design tool)

1. **Typical day (Meena):** 06:30 WhatsApp alert → alert detail on phone → evidence → campaign page; 11:40 flash on desktop → acknowledge → assign; 12:10 ask co-worker (Tamil) → citations; 15:30 correct a label with a reason; 22:00 review the handover draft.
2. **Must-pass 1:** response opportunity WhatsApp → Story with "who is reacting" → response queue → TVK response appears → opportunity closes.
3. **Must-pass 2:** one verified post → Tier 1 on big screen and in-app → acknowledge clears for everyone.
4. **Campaign confirmation:** possible coordination → lead enters reason → confirmed attribution logged.
5. **Bias guard:** correction that softens criticism → second reviewer approves.
6. **Clipping upload:** drag scans → OCR → check scan → Mention joins an existing Story.
7. **CM mode:** open CM summary → tap evidence → back; toggle presentation-safe.
