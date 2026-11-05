# 02 — Functional Requirements

**Source:** PRD User Stories (35 stories, 11 groups) and Proposed Solution  
**Note:** Requirements marked [DEFAULT] use starting values subject to calibration. Requirements marked [PROPOSED] are technical recommendations not explicitly stated in the PRD.

---

## Group A — Seeing the Picture

### FR-A01: Big Screen Sentiment Display
- The big screen MUST display sentiment for TVK, CM Vijay, and AadhavArjuna simultaneously.
- Each entity MUST show: sentiment percentage, raw mention count ("of N mentions"), and last-updated timestamp (HH:MM).
- If an entity has fewer than 30 [DEFAULT] mentions in the current window, the screen MUST show "not enough data" instead of a percentage.
- Sentiment MUST update within 5 minutes of the original post being published (client target).
- The big screen MUST also display: Ally Mood panel, top 3 [DEFAULT] most severe active alerts, top rising Stories.

### FR-A02: CM Summary View
- A "CM mode" view MUST exist showing a summary readable in 30 seconds.
- The summary MUST show at most 3 changes since the last summary, each with its cause in one line.
- Each change MUST have a one-tap link to its supporting evidence (Mentions).
- Summary MUST be available in English or Tamil.
- If a change is attributed to an ally, the summary MUST show "reviewed by <name>" or be held until a lead reviews it.

### FR-A03: Drill-Down from Any Number
- Tapping/clicking any number, percentage, or Story on any screen MUST open the underlying Mentions.
- Each Mention in the drill-down MUST show: author or outlet name, engagement metrics, timestamp, label, label reason, and a link to the original source.

### FR-A04: Presentation-Safe Mode
- A toggleable presentation-safe mode MUST exist.
- When active, the big screen MUST show only: aggregates, public actors' names, Story titles.
- When active, the big screen MUST NOT show: private citizen handles, internal notes, correction details.
- The mode's active/inactive state MUST be visible on the big screen at all times.

---

## Group B — Knowing Who Matters

### FR-B05: High-Influence Single Post Alert
- A single post from a verified account in the national media tier posting negatively about a tracked entity MUST trigger an alert even with no volume spike.
- When the post's engagement reaches the influence trigger (top-tier severity threshold), within 5 minutes the big screen MUST flash and team leads MUST receive a WhatsApp notification.
- The alert MUST include the reason, e.g. "verified, 1.2M followers, national media tier, 4,000 reposts in 20 min".

### FR-B06: Standing Tier Management
- Staff MUST be able to add, remove, and edit accounts/outlets in standing tiers: national media, state media, party leaders, influencers.
- Tier changes MUST take effect immediately for new Mentions.
- Every tier change MUST be logged in an audit log with: editor name, timestamp, change made.

### FR-B07: Weighted Scores Alongside Raw Counts
- Every sentiment display MUST show the weighted score AND the raw count side by side.
- Tapping the weighted score MUST reveal: the raw count and the top 3 contributing accounts/posts.
- No single post or account MUST contribute more than 5% [DEFAULT] to any entity's score.

### FR-B08: Alert Ranking and Cap
- Multiple simultaneous alerts MUST be ranked by severity.
- The big screen MUST show at most 3 [DEFAULT] active alerts.
- Related posts MUST be grouped into one alert (not shown separately).
- Lower-severity alerts MUST remain visible in-app only.
- Any team member MUST be able to acknowledge or assign an alert.
- When acknowledged, the flash/highlight MUST clear for all users simultaneously, showing "taken by <name>".

---

## Group C — Volume Spikes

### FR-C09: Spike Detection Alert
- The system MUST detect when negative mentions for an entity rise significantly above baseline.
- Spike conditions [DEFAULT]: same hour over last 14 days, ≥3 standard deviations above mean, ≥2× normal volume, ≥25 mentions, sustained over ≥15 minutes.
- A spike alert MUST show: the baseline level, current level, and the top posts driving it.
- When a scheduled event (rally, assembly session, results day) is marked in the calendar, that time window MUST use an event-specific baseline rather than the standard historical baseline.

---

## Group D — Coordinated Campaigns

### FR-D10: Campaign Detection and Alert
- The system MUST detect potential coordinated posting: cluster of accounts posting near-identical text or a shared hashtag in a short time window.
- A "possible coordinated campaign" alert MUST show: the accounts involved, account ages, shared phrasing, timing chart, and a confidence level.

### FR-D11: Campaign Attribution Requires Human Confirmation
- A campaign alert MUST read "possible coordination" until a lead explicitly confirms attribution.
- Confirmation MUST be logged with: lead's name, timestamp, reason given.
- No campaign MUST be presented to leadership as fact without confirmed attribution.

### FR-D12: Campaign Spread Tracking
- Once a campaign is confirmed, its detail page MUST show: mentions per hour, new accounts joining over time, TVK response Stories, all on one timeline.

---

## Group E — Allies and All Parties

### FR-E13: Ally Mood Panel
- A dedicated Ally Mood panel MUST be displayed on the big screen.
- The panel MUST monitor public figures from ally parties: official handles, MPs, MLAs, spokespeople.
- When ally public figures' mentions turn negative toward TVK, the panel MUST show: the people involved, the posts, and fire an alert at the ally threshold.

### FR-E14: Party Relationship Tag Editing
- Staff MUST be able to change any party's relationship tag: ally, supporting, opposition, neutral.
- Tag changes MUST apply immediately to all alerts and panels.
- Historical Mentions MUST retain the relationship tag that was active at the time of the mention (past records not retroactively changed).

---

## Group F — News, E-Paper, and Clippings

### FR-F15: Cross-Source Story Fusion
- The system MUST group items about the same event across X and news into one Story.
- Each Story MUST display combined reach with source breakdown, e.g. "9 outlets, 1,200 posts".
- Staff MUST be able to manually merge or split Stories.

### FR-F16: Clipping Upload to Mention
- Staff MUST be able to upload scanned newspaper clippings (image files).
- Each uploaded scan MUST be processed to extract text and become a Mention.
- Each Mention derived from a clipping MUST include: outlet name, publication date, page number.
- Low-confidence text recognition results MUST be flagged with "check scan".

### FR-F17: E-Paper Daily Ingestion
- E-paper editions from war room subscriptions MUST be ingested daily.
- Each ingested story MUST appear as a Mention tagged "e-paper".
- Page images MUST be accessible one tap away from each e-paper Mention.

### FR-F18: Outlet Weight Transparency
- Every news Mention MUST display its outlet's tier and the reason for that tier.
- Example reasons: "state daily, Tamil, high reach" or "party-affiliated channel".

### FR-F19: Anonymous Insinuation Surfacing
- A single item labelled as an anonymous insinuation about a tracked entity MUST appear as a Story of one, marked "unattributed".
- These MUST NOT be suppressed even with low volume.

---

## Group G — The Co-worker

### FR-G20: Natural Language Q&A
- Any team member MUST be able to ask the co-worker questions in Tamil or English.
- Answers MUST be in the same language as the question.
- Every claim in an answer MUST cite the specific Mentions behind it.
- Every count in an answer MUST come from a live data query, not the model's memory.
- When data is insufficient, the answer MUST say "not enough data".
- First answer MUST arrive within 10 seconds for a typical question [DEFAULT].

### FR-G21: Per-Alert Explanation
- Every alert MUST carry a two-line explanation and links to its evidence.
- The explanation MUST be generated by the co-worker automatically when the alert fires.

### FR-G22: Shift Handover and CM Summary Drafting
- When a shift ends, the co-worker MUST draft a shift handover listing: open alerts, their owners, what to watch overnight.
- The handover draft MUST be editable by a person before being sent.
- The co-worker MUST be able to draft the CM summary on demand or on schedule.

### FR-G23: Action Recommendations Without Acting
- The co-worker MAY suggest recommended actions in response to events.
- Recommendations MUST be presented as suggestions with a stated reason.
- The co-worker MUST have no mechanism to post, reply, publish, or message anyone outside the war room.

---

## Group H — Correcting the Machine

### FR-H24: Label Correction
- Any team member MUST be able to correct any AI-generated label (sentiment, entity, stance, attribution, severity, campaign flag).
- After correction: the new label MUST show "corrected by <name>"; the original label MUST remain visible in history.
- Aggregates MUST update to reflect corrected labels, with corrected counts identifiable separately.

### FR-H25: Corrections Feed Learning
- Corrections MUST join the gold set and the co-worker's training examples.
- A weekly accuracy chart MUST show model-only accuracy vs. corrected accuracy per capability and language.

### FR-H26: Bias Guard on Corrections
- If a correction softens a negative label about TVK entities (makes it less negative), it MUST require a second reviewer before it changes aggregates.
- If a correction touches a high-severity Story, it MUST require a second reviewer.
- A model-only view (pre-correction) MUST always be available for reference.

---

## Group I — Operating Honestly and Failure Paths

### FR-I27: Weekly Filter Audit
- Weekly, a sample of items rejected by the filter MUST be reviewed by staff.
- The sample MUST be stratified by source, language, and entity.
- The missed-item rate MUST be displayed on the admin page.

### FR-I28: Source Health Indicators
- When any source (X, a news site, e-paper, AI model) fails or is delayed, the affected panels MUST show "data delayed since HH:MM" with the source named.
- No number MUST silently stop updating — a stale number MUST always indicate when it was last updated.
- A source error MUST NEVER look like "no new items".

### FR-I29: WhatsApp Alert Delivery Tracking
- Failed WhatsApp alerts MUST be retried automatically.
- If delivery fails, the in-app alert MUST show "WhatsApp not delivered".
- A lead MUST be notified in-app of any undelivered WhatsApp alert.

### FR-I30: X API Usage Monitoring
- API usage MUST be tracked continuously against monthly caps.
- Admins MUST receive alerts when usage reaches 50%, 80%, and 95% of the monthly cap.
- Any number derived from sampled data MUST be labelled as "sampled" on every display.

---

## Group J — Response Opportunities

### FR-J31: Response Opportunity Alert
- When an attack on a tracked entity is drawing condemnation from other parties, media, or public figures, AND no TVK leader has responded, the system MUST alert Sajan (or the designated response lead).
- The alert MUST arrive on WhatsApp within 5 minutes of the trigger crossing its threshold.
- The alert MUST include: the triggering clip or article, who is already reacting, "no TVK response yet".
- When a TVK leader posts a response, the opportunity MUST update to show "responded by <name> at HH:MM" and stop alerting.

### FR-J32: Response Queue
- A response queue MUST exist listing Stories likely to need a TVK reaction: condemnation, rebuttal, congratulation, condolence.
- Each item in the queue MUST show: suggested response type, who else has reacted, time since the story started, TVK's current response status (read from TVK leaders' own accounts).
- Marking an item as "acted on" or "not needed" MUST be logged and used to train the queue's prioritisation.

### FR-J33: Event and TV Quote Capture
- Remarks made at events and on TV, quoted in news reports or TV channel YouTube videos, MUST become Mentions.
- Each such Mention MUST be linked to the source article or video, with the quoted line and the speaker's name where known.

---

## Group K — The Co-worker Earning Trust

### FR-K34: Capability Stage Progression
- Each co-worker capability MUST have a measured accuracy metric tracked against a defined bar (set during calibration).
- When a capability's accuracy stays above its bar for the agreed calibration period, it MUST automatically progress: Assist → Trusted → Co-worker.
- The admin page MUST display each capability's current stage and its accuracy trend.

### FR-K35: Automatic Stage Demotion
- If a capability's accuracy falls below its bar, it MUST automatically drop back one stage.
- When a stage drops, human review MUST resume for that capability.
- Leads MUST see a notification explaining why the stage dropped.

---

## Non-Functional Requirements

### NFR-01: Freshness
- X post to screen: ≤5 minutes (p95), client requirement
- Online news: as fast as the outlet's feed allows; floor per source type to be defined by CTO
- E-paper and clippings: same morning they are published/uploaded

### NFR-02: Accuracy
- F1 targets [DEFAULT]: English 0.80, Tamil 0.75, Tanglish 0.72
- Final targets set after gold set is built

### NFR-03: Reliability
- Every third-party dependency must have a designed, visible fallback
- If AI model is down, dashboards keep working with stale-data indicators
- Source errors must never silently look like "no new items"

### NFR-04: Privacy
- India DPDP Act 2023 and Rules apply to all personal data stored
- No private citizen profiles or views centred on individuals
- No links to citizen or petition records
- Data residency: India region [PROPOSED]

### NFR-05: Auditability
- Every label and score must be reproducible: model, prompt, weight, and threshold versions logged
- Every co-worker answer and alert logged with inputs, sources used, model and prompt version
- All staff-editable settings are data rows with full audit logs (who, when, what changed)

### NFR-06: Security
- Staff-only access; no public-facing login
- Settings (watch terms, relationship tags, tiers, outlet weights, thresholds) changeable without a deploy

### NFR-07: Generalisability
- Nothing TVK-specific is hard-coded
- A new client must be onboardable through configuration alone, with no code changes
- Each client's data, corrections, gold set, and co-worker memory are fully isolated

### NFR-08: Statistics Integrity
- Every percentage shows its denominator
- Below minimum sample (30 [DEFAULT]): show "not enough data", not a percentage
- Changes shown only when they clear a significance test or confidence interval
- Sampled numbers labelled as sampled on every display

### NFR-09: Co-worker Safety
- All ingested content treated as data, never as instructions
- A post saying "ignore your instructions" is processed as a post
- Co-worker cannot post, reply, or publish in Phase 1
