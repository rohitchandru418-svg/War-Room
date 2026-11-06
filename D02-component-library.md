# D02 — Component Library

Specifies every reusable component for the War Room UI in the Notion-style language of **D01**. Each component lists anatomy, variants, states, and the rules that come from the PRD. Component names in `code` match the front-end names in the build docs (`components/numbers/` etc.) so design and code stay aligned.

All components: light and dark, English and Tamil (allow ~30 % longer text), keyboard operable, visible focus ring (D01 §10), loading/empty/error variants where data-bound.

---

## 1. App shell

### 1.1 Sidebar
- Width 240 px, collapsible to 48 px (icons only). Background `bg-subtle`, hairline right border.
- Top: **client block** (client logo/initial + client name + small "War Room" label). This plays Notion's workspace-switcher role but is **not** a switcher in Phase 1 (single client per user).
- Sections (labels in `text-3`, 12 px): **Monitor** (Dashboard, Stories, Alerts, Campaigns), **Co-worker** (Ask, Briefs & handovers), **Input** (Upload clippings), **Learn** (Corrections & accuracy), **Admin** (Settings, Source health, Audit log). Items shown depend on role.
- Item: 32 px high, icon 20 px + label; hover `bg-hover`; **active** = `bg-selected` + 3 px `accent` bar on the left + `text` weight 600.
- Badges: unread/active alert count (Tier-1 count uses brand badge; others gray).
- Bottom: user avatar menu, language toggle, theme toggle, collapse control.

### 1.2 Top bar (48 px)
Breadcrumb (page icon + title), **freshness indicator** ("Live · updated 12:04"), **Presentation-safe** toggle (visible to leads+; shows active state), notification bell, language switch `EN | தமிழ்`.

### 1.3 Page header (document style)
Optional page icon (40 px emoji/Lucide), title (32/40), description line, **properties row** (Notion-style: label : value chips such as "Entity: CM Vijay", "Window: 24 h", "View: Effective | Model-only"), then content blocks.

### 1.4 Side peek panel
480 px right panel, `bg`, left hairline, header with title + "open as page" + close. Used for evidence, mention detail, correction, campaign account detail. Opening does not navigate; `Esc` closes; focus returns to the trigger.

### 1.5 Banners and callouts
- **Source-health banner** (top of page, amber callout): "X data delayed since 11:52" with a link to Source health. Persists until resolved. Never auto-dismisses.
- **Presentation-safe badge**: a persistent pill in the top bar and on the big screen ("Presentation-safe").
- **Callout block** (brand-50 or gray, 6 px radius, icon left): explanations, "Why this alert fired", "not enough data" notes.
- **Reconnecting** bar when the live connection drops: "Reconnecting… data may be a minute old".

## 2. Statistics components (`components/numbers/`)

These enforce the PRD rules **visually**. Designers must not draw a percentage without them.

### 2.1 `StatWithDenominator`
Large number (40/44, tabular) + unit + micro line **"of 412 mentions · last updated 12:04"**. Tap/click opens the evidence peek. Variants: default, loading skeleton, stale (amber freshness text), sampled (adds `SampledBadge`).

### 2.2 `NotEnoughData`
Replaces the number when N < minimum sample (default 30): gray text "Not enough data" + micro "N = 18, minimum 30". Same footprint as the number so layout does not jump.

### 2.3 `WeightedRaw`
Pair of values: **weighted** (primary) beside **raw** (secondary), e.g. "Negative 38 % weighted · 31 % raw". An info icon explains weighting. Click opens the breakdown peek with **top three contributing accounts**. Highlights when the two differ by more than a threshold (small amber dot: "driven by a few accounts").

### 2.4 `SampledBadge`
Small `tag-amber` pill "Sampled" with tooltip "Some data was sampled because of platform limits." Appears on any affected number or chart.

### 2.5 `StaleStamp`
Micro text "updated 12:04"; turns amber with "data delayed since 11:52 (X)" when a source is behind.

### 2.6 `SentimentBar`
Horizontal stacked bar (positive / neutral / sarcastic / negative) with hover values and `of N` caption; below min sample it renders a dashed empty bar plus `NotEnoughData`. Segment colours per D01 §4.5; icons in legend.

### 2.7 `TrendChip`
"▲ +6 pts vs previous 24 h" **only** when the change passed the significance test; otherwise nothing or "No significant change". Hover shows interval.

### 2.8 `ConfidenceMeter`
Three-step meter (low/medium/high) with the word; used for campaign confidence and label confidence. Always word + shape, not colour alone.

## 3. Status, tags and labels

### 3.1 `StanceTag`
Positive ▲ green · Negative ▼ rose · Neutral – gray · Sarcastic ~ purple, with the word. Sarcastic shows target stance in the peek ("Sarcastic · actually supportive").

### 3.2 `LabelWithReason`
`StanceTag` + entity + a "why" link/icon. Hover or click opens a popover with the **reason** (≤ 240 characters) and "Corrected by <name>" history link when corrected. Corrected labels show a small pencil mark and keep the original visible in history.

### 3.3 `TierBadge`
Tier 1: solid `#A00000` with white text "TIER 1"; Tier 2: `tag-amber` "TIER 2".

### 3.4 `SourceTypeBadge`
X, News, E-paper, Clipping, YouTube/TV with small icons; e-paper and clipping show "page N". Outlet name + **outlet tier with reason** ("State daily · Tamil · high reach", "Party-affiliated channel" with a partisan marker).

### 3.5 `HealthDot`
8 px dot + word: Healthy (green), Delayed (amber), Down (rose + icon). Plus last-success time.

### 3.6 `RelationshipTag`
Ally / Supporting / Opposition / Neutral as tags (neutral colours; not brand); labels come from client vocabulary.

### 3.7 `StageBadge`
Assist · Trusted · Co-worker with a three-step ladder icon.

### 3.8 `UnattributedMarker`
`tag-orange` "Unattributed" for anonymous-insinuation Stories of one.

## 4. Data display blocks

### 4.1 Table (Notion "database" style)
Header row `bg-subtle`, 36 px rows, hairline row borders, **property columns** with icons in headers, sticky first column, resizable columns, hover row highlight + row actions on the right, click row → side peek. Toolbar: **filter chips** ("Entity is CM Vijay ×"), sort, search within view, "Add filter". Selected rows show a left brand bar. Empty state per §8. Pagination: "Load more" or infinite scroll with count "Showing 50 of 1,204".

### 4.2 Mention row (`MentionRow`)
Avatar/outlet icon · author or outlet (with verified tick, followers or outlet tier) · snippet (2 lines, no full article text) · engagement (reposts, likes) · time · `LabelWithReason` · source badge · link-out icon. Private-citizen handles never appear in presentation-safe mode (replaced by "Public account — hidden").

### 4.3 `AlertCard`
Left bar (Tier 1 brand 4 px), `TierBadge`, alert type tag (Influence, Spike, Campaign, Ally, Response opportunity), title, **reason line** ("verified, 1.2M followers, national media, 4,000 reposts in 20 min"), 2-line co-worker explanation, evidence chips (3 posts), age ("12 min ago"), owner ("taken by Meena" or "Unassigned"), actions: **Acknowledge**, **Assign**, **View evidence**. States: new (pulse), acknowledged, assigned, resolved, "WhatsApp not delivered" warning chip.

### 4.4 `StoryCard`
Title, "9 outlets, 1,200 posts" reach split by source type with small icons, started-ago, sentiment bar (with N), top entities, status tags (Rising, Needs review, Unattributed), response status when in the queue ("No TVK response yet").

### 4.5 `ResponseQueueItem`
Story title, suggested response type (Condemn, Rebut, Congratulate, Condole) tag, who else has reacted (avatars with names), time since start, "TVK response: none yet / responded by <name> at HH:MM", actions "Mark acted" / "Not needed" (both recorded).

### 4.6 `CampaignPanel`
Header: status tag **"Possible coordination"** (or "Confirmed by <name>"), `ConfidenceMeter`. Evidence blocks: accounts table (handle, account age, followers, bot-suspicion flag), shared phrasing, **timing chart**, account-age histogram. Lead-only attribution form (reason required).

### 4.7 `AllyMoodPanel`
Per ally/supporting party: mood bar (with N), top people and posts, alert indicator. Header uses client vocabulary ("Ally Mood").

### 4.8 `Timeline`
Vertical/horizontal event timeline for Stories and campaigns: dots for first mention, news pickup, alert fired, acknowledgement, TVK response; times in client timezone.

### 4.9 `EvidenceList`
Ordered list of Mentions (author/outlet, engagement, time, label + reason, link to original) used in every peek; counts shown at top ("14 mentions").

### 4.10 Charts
Line, stacked bar, area (sparingly), sparkline, histogram, heatmap (hour × day for baselines). Follow D01 §8. Caption always includes denominator + window + updated time.

## 5. Inputs and actions

- **Buttons:** Primary (solid `#A00000`, white text) · Secondary (outlined, hairline) · Tertiary/ghost (text) · Destructive (secondary style + trash icon + confirm). Heights 32 (default), 40 (touch/kiosk). Loading state: spinner replaces icon, label stays.
- **Inline editable cell/field:** click to edit, `Enter` saves, `Esc` cancels, then a quiet "Saved · 12:04" confirmation; validation message under the field in plain text.
- **Select / multi-select tags**, **combobox with search**, **date-time picker** (timezone shown), **toggle**, **checkbox**, **slider** only for thresholds with visible numeric input and default shown ("Default: 30").
- **Slash/“+” menu** in the brief editor for blocks (heading, list, callout, evidence link).
- **Confirm dialog:** states the consequence plainly ("This will change how alerts are routed from now on"), single primary action, cancel.
- **Reason field** (corrections, campaign confirmation, second-review decisions): required, 10-character minimum with counter, placeholder shows an example.

## 6. Co-worker components

- **Co-worker identity:** circular avatar (the official elephant artwork supplied by the client, shown as an image; never redrawn) + the co-worker's configured name. Brand-tinted ring.
- **`ChatMessage`:** user bubbles right-aligned light gray; co-worker messages left with avatar, no bubble (document-like), citations as numbered superscripts `[1]` that open the evidence peek, a footer line "Based on 64 posts" (sample size) and 👍/👎 controls (icons, not emoji). Language matches the question.
- **`ThinkingState`:** "Looking at the data…" with subtle animation; after 10 s "Taking a moment…".
- **`UnavailableState`:** "Co-worker unavailable — data may be stale. Dashboards still work." (never answers from old data).
- **Suggested questions:** 3 chips above the input ("Why did Congress mentions turn negative?").
- **`BriefEditor`:** Notion-like document editor for handovers/summaries: title, blocks, "Draft by co-worker" tag, "Reviewed by <name>" tag required before sending where the ally-blame rule applies, actions Edit / Send.
- **`RecommendationCard`:** suggestion with reason and the explicit line "The co-worker cannot post or reply" (shown once in the footer, quiet).

## 7. Correction and trust components

- **`CorrectionPanel`** (in peek): current label, dropdowns for stance/entity/claim type/attribution, reason field, notice when second review applies ("This change softens a criticism of a primary entity. A second reviewer must approve before it changes totals."), history of earlier values.
- **`ReviewQueueItem`:** original vs proposed label, who proposed, reason, Approve/Reject with reason.
- **`StageLadder`:** three-stage horizontal ladder per capability with current stage highlighted, error-rate sparkline, "bar" threshold line, reason for last change.
- **Accuracy charts:** model-only vs corrected agreement per week; correction rate per 1,000 labels per language.

## 8. States (all data-bound components)

| State | Treatment |
|---|---|
| Loading | Skeleton blocks matching final layout; stats show grey bars; never spinners over whole pages |
| Empty | Short sentence + the reason ("No alerts in the last 24 h") + one action; calm illustration optional (line art, brand accent) |
| Error | Plain-language message + retry + link to Source health if relevant; keep last known data visible with `StaleStamp` |
| Stale/delayed | Amber stamp and banner; numbers remain visible but marked |
| Success | Quiet inline "Saved" or toast (bottom-left, auto-dismiss 4 s); alerts never auto-dismiss |
| No permission | Hidden by default; when navigated to, a calm "You don't have access" page |
| Presentation-safe | Aggregates, public actors' names and Story titles only |

## 9. Feedback and overlays

Toasts (non-blocking), tooltips (explain jargon: "Influence score", "Weighted"), popovers (reason, definitions), modals (rare), context menus (row actions), command-style **quick filter** inside tables (not global search; global search is out of scope for Phase 1).

## 10. Big-screen kit (`/bigscreen`)

Larger sizes (stat XL 96 px; body ≥ 28 px; Tamil ≥ 30 px), high contrast dark theme, no pointer UI, no scrolling. Components: `WallEntityStat` (name, big number, "of N", freshness), `WallAlertStrip` (max three, severity-ranked, Tier-1 slow-pulse), `WallStoryList`, `WallAllyMood`, `WallClock/Freshness`, `WallDelayedBanner` ("Data delayed since 11:52 (X)"). Shows aggregates, public actors' names and Story titles; post text is hidden until the display-rights question is resolved `[PRD: X display rules to be confirmed]`.

## 11. Component inventory checklist (for the design file)

App shell, sidebar, top bar, page header, peek panel, banners/callouts · StatWithDenominator, NotEnoughData, WeightedRaw, SampledBadge, StaleStamp, SentimentBar, TrendChip, ConfidenceMeter · StanceTag, LabelWithReason, TierBadge, SourceTypeBadge, HealthDot, RelationshipTag, StageBadge, UnattributedMarker · Table, MentionRow, AlertCard, StoryCard, ResponseQueueItem, CampaignPanel, AllyMoodPanel, Timeline, EvidenceList, charts · buttons, inline fields, selects, pickers, dialogs, reason field · co-worker identity, ChatMessage, thinking/unavailable states, suggestions, BriefEditor, RecommendationCard · CorrectionPanel, ReviewQueueItem, StageLadder, accuracy charts · skeleton/empty/error/stale/success states · big-screen kit. Each in light, dark, English, Tamil.
