# 05 — Business Rules

**Note:** Values marked [DEFAULT] are starting values for calibration, not final decisions. Final values are set after two weeks of live data against the gold set.

---

## 1. Data Collection Rules

### BR-001: Filter Inclusivity
- The filter MUST err toward inclusion. Missing an item is worse than including too much.
- Any post, article, or clipping matching a watch term for any tracked entity MUST be kept.
- Items not matching any watch term MUST be stored raw (not shown in the UI) and included in weekly audit sampling.

### BR-002: Duplicate Prevention
- No item MUST be paid for or processed twice.
- Items already ingested (by X post ID, article URL, e-paper edition + page, clipping hash) MUST be deduplicated before processing.

### BR-003: Source Identification
- Every Mention MUST carry its source type tag: `x_post`, `news_article`, `epaper`, `clipping`, `youtube_transcript`.
- Every news Mention MUST carry its outlet's tier label and reason.

### BR-004: X Display Rules
- Showing full post text on the big screen wall may require compliance with X's display requirements for offline/broadcast use. Confirm before launch. [GAP — see PRD-GAP-ANALYSIS.md 2.6]

### BR-005: Crawling Ethics
- Web crawling MUST respect each site's `robots.txt`.
- Crawlers MUST identify themselves (User-Agent string with product name and contact).
- Crawlers MUST pace requests to avoid overloading outlet servers.
- A crawl error MUST be reported to admins and MUST NOT look like "no new articles".

### BR-006: Content Storage Limits
- For news articles: store metadata, headline, snippet, and link. Do NOT store full article text unless licensed.
- For e-papers: store page images and extracted text from war room's subscription editions only.
- For X posts: store post text, engagement, author profile data as permitted by X API terms.

---

## 2. Entity and Relationship Rules

### BR-007: Tracked Entity Set
- Phase 1 tracks: TVK, CM Vijay, AadhavArjuna, all Indian political parties and key leaders.
- Alliance partners MUST be watched as closely as opposition.

### BR-008: Relationship Tags
- Each party carries exactly one relationship tag: `ally`, `supporting`, `opposition`, or `neutral`.
- Tags are staff-editable at any time.
- Tag changes take effect immediately for new Mentions.
- Historical Mentions retain the relationship tag that was active at the time they were logged.

### BR-009: Entity Recognition Coverage
- Watch terms MUST include names, nicknames, Tamil spellings, Tanglish spellings, and known hashtags for each entity.
- Watch-term lists are staff-editable without a code deploy.

---

## 3. Labelling Rules

### BR-010: Label Completeness
- Every Mention MUST have labels for: entity (which tracked entity), stance (positive/negative/neutral/sarcastic), claim type (on-record/named-source/anonymous-insinuation/satire), language (Tamil/English/Tanglish/mixed).

### BR-011: Reason Required on Every Label
- Every AI-generated label MUST include a human-readable reason explaining why the label was assigned.
- Reason MUST be shown in the UI alongside the label.

### BR-012: Uncertainty Escalation
- Items the system is unsure about (below a confidence threshold) MUST be escalated: first to a stronger model; if still uncertain, to a human reviewer queue.

### BR-013: Language Detection
- Language detection MUST be the system's own model — the X language tag is NOT reliable for Tanglish.
- Tamil script, English, Tanglish (Tamil in Latin letters), and mixed posts are all supported.

### BR-014: Tamil Numerals
- Tamil numerals and amounts MUST be correctly parsed (a known gap in prior Minsky news work).

### BR-015: Anonymous Insinuations
- A single item labelled `anonymous-insinuation` about a tracked entity MUST surface as a Story, even if it comes from one source.
- It MUST be marked "unattributed" in the UI.
- It MUST NOT be suppressed due to low volume.

---

## 4. Influence and Weighting Rules

### BR-016: Influence Score Composition
- An account's influence score is composed of: verified status, follower count (log-scaled), engagement velocity (how fast engagement is growing), and tier assignment.
- Outlet influence is composed of: outlet tier, reach (circulation/traffic), independence (party-affiliated vs. independent).
- All weights MUST be documented, versioned, and staff-editable.

### BR-017: Per-Post Cap
- No single post or account MUST contribute more than 5% [DEFAULT] to any entity's weighted score.

### BR-018: Suspected Bot Down-weighting
- Accounts identified as likely bots MUST be down-weighted.
- The down-weighting MUST be indicated in the UI wherever the account appears.

### BR-019: Weighted and Raw Always Together
- Wherever a weighted score is shown, the raw count MUST be shown beside it.
- The top 3 contributing accounts/posts MUST be accessible with one tap.

---

## 5. Story Rules

### BR-020: Story Formation
- Items about the same event or happening, across X and news, are grouped into one Story.
- Story grouping is automatic (algorithm TBD by CTO) with manual override.
- Staff can merge Stories or split a Story into two.

### BR-021: Story Reach Display
- Each Story MUST show combined reach with source breakdown: e.g. "9 outlets, 1,200 posts".
- Reach is the sum across all Mentions in the Story, split by source type.

### BR-022: Rumour Surfacing
- Rumours about tracked entities MUST surface as a Story even from a single source (see BR-015 for anonymous insinuations).

---

## 6. Alert Rules

### BR-023: Alert Tiers
- Three alert tiers exist:
  - **Tier 1 (Highest):** Big screen flash + WhatsApp to team leads + in-app
  - **Tier 2:** In-app alert only
  - **Tier 3:** Story flagging only (no active alert)

### BR-024: Big Screen Alert Cap
- The big screen MUST show at most 3 [DEFAULT] active alerts simultaneously.
- Related posts about the same event MUST be grouped into one alert, not shown separately.

### BR-025: Alert Acknowledgement
- Any team member can acknowledge or assign an alert.
- When acknowledged, the flash/highlight clears for ALL users simultaneously.
- Acknowledgement is recorded: "taken by <name>".
- Assignment is recorded with assignee name.

### BR-026: Spike Detection Thresholds [DEFAULT]
- Spike = negative mentions for entity exceed: same hour over last 14 days, ≥3 standard deviations, ≥2× normal volume, ≥25 mentions, over ≥15 minutes.
- During a scheduled event: use event-specific baseline, not standard historical baseline.

### BR-027: Influence Trigger [DEFAULT]
- A single post from a verified account in the national media tier posting negatively triggers a Tier 1 alert when engagement reaches the top-tier severity threshold.
- Exact threshold set during calibration.

### BR-028: Ally Threshold
- Ally Mood panel fires an alert when ally public figures' negative mentions toward TVK cross the ally threshold.
- Exact threshold set during calibration.

### BR-029: Response Opportunity Trigger
- Response opportunity alert fires when: an attack on a tracked entity is drawing condemnation from other parties/media/public figures AND no TVK leader has responded AND the trigger threshold is crossed.
- Alert routed to Sajan (or designated response lead) on WhatsApp within 5 minutes.
- Alert ceases when a TVK leader posts a response (detected automatically).

---

## 7. Campaign Detection Rules

### BR-030: Campaign Alert Initial State
- A campaign alert MUST always initially read "possible coordination" — NEVER naming a party.
- Party attribution requires explicit confirmation by a War Room Lead.

### BR-031: Campaign Confirmation Requirements
- Confirmation MUST be logged: lead name, timestamp, reason.
- If a lead cannot confirm (insufficient evidence), the alert stays as "possible coordination" indefinitely.

### BR-032: Campaign Detection Signals
- Signals that contribute to a campaign detection: new or low-history accounts clustering, near-identical text, posts timed together, accounts boosting each other, a hashtag pushed by a small network.
- Each detection shows its confidence level and the specific evidence.

---

## 8. Correction Rules

### BR-033: Correction Immutability
- Original AI-generated labels MUST NEVER be overwritten or deleted.
- A correction creates a new record alongside the original, with: who corrected, when, why (required), old value, new value.
- The UI MUST show the corrected label as active but make the original visible in history.

### BR-034: Bias Guard Trigger
- A second reviewer is REQUIRED (before aggregate changes take effect) when:
  - A correction makes a negative label about TVK entities softer (less negative), OR
  - A correction touches a high-severity Story.

### BR-035: Model-Only View Always Available
- Regardless of corrections, a view showing the AI model's original labels (before any corrections) MUST always be available.

### BR-036: Correction Contributes to Learning
- Every correction joins the gold set and co-worker's training examples.
- A weekly accuracy chart tracks model-only vs. corrected agreement per capability and language.

---

## 9. Statistical Integrity Rules

### BR-037: Denominator Always Shown
- Every percentage or rate MUST show its denominator (e.g. "64% of 312 mentions").

### BR-038: Minimum Sample Gate
- When mention count is below 30 [DEFAULT] in the display window, show "not enough data" instead of a percentage.
- This applies per entity, per time window.

### BR-039: Significance Threshold
- A change in sentiment or volume MUST only be called a "change" when it clears a significance test or confidence interval.
- Noise MUST NOT be displayed as a trend.

### BR-040: Sampled Data Label
- Any number derived from sampled data (e.g. if X reads are capped) MUST be labelled "sampled" on every display where it appears.

---

## 10. Co-worker Rules

### BR-041: Grounding Requirement
- Every co-worker claim MUST cite its source Mentions.
- Every count MUST come from a live database query — NEVER from the model's own arithmetic or memory.
- If data does not support an answer, the co-worker MUST say so explicitly.

### BR-042: Content-as-Data Safety Rule
- All ingested content (posts, articles, etc.) is treated as DATA, never as instructions.
- A post or article containing "ignore your instructions" or similar prompt-injection attempts MUST be processed as a post, not acted upon.

### BR-043: Never Publishes
- The co-worker MUST have no mechanism to post, reply, publish, or message anyone outside the war room in Phase 1.

### BR-044: Ally Blame Review Gate
- Any co-worker output (summary, brief, recommendation) that attributes fault or negative action to an ally MUST be held for lead review before it reaches the CM or is distributed.

### BR-045: Campaign Party Naming Gate
- Naming a specific party as behind a campaign ALWAYS requires human confirmation from a War Room Lead. The co-worker CANNOT name a party on its own.

### BR-046: Co-worker Stage Rules
- Each capability independently progresses through stages: Assist → Trusted → Co-worker.
- Progression requires: measured accuracy above its bar for the agreed calibration period.
- Stage bars are set from real calibration data, not guessed.
- If accuracy falls below the bar: automatic demotion to previous stage, human review resumes.

---

## 11. Privacy Rules

### BR-047: No Private Citizen Profiles
- The system MUST NOT create or display profiles centred on private individuals.
- Analysis covers conversations and public actors only.

### BR-048: DPDP Compliance
- India's DPDP Act 2023 and Rules apply to any personal data stored.
- Data retention and deletion rules MUST be defined before ingestion starts. [GAP — not specified in PRD]

### BR-049: Presentation-Safe Mode
- In presentation-safe mode, the display MUST show ONLY: aggregates, public actors' names, Story titles.
- Private citizen handles and internal notes MUST be hidden.

---

## 12. Generalisability Rules

### BR-050: No Hard-coded Client Data
- Nothing TVK-specific may be hard-coded in application logic.
- All client-specific configuration (entities, watch terms, tiers, thresholds, recipients, co-worker name/avatar, UI vocabulary) MUST be data rows in a client configuration table.

### BR-051: Client Isolation
- Each client's data, corrections, gold set, and co-worker memory MUST be fully isolated.
- One client's corrections MUST NEVER change another client's labels.

### BR-052: Configurable UI Vocabulary
- Words like "party", "ally", "CM summary" come from the client's configuration.
- A brand client sees "competitor" instead of "party", "partner" instead of "ally", "leadership summary" instead of "CM summary".

---

## 13. Platform Limit Rules

### BR-053: X API Cap Monitoring
- Admins MUST receive alerts at 50%, 80%, and 95% of monthly X API read cap.
- Any number affected by X read sampling MUST be labelled "sampled".

### BR-054: Multiple Account Prohibition
- Multiple X accounts or apps to bypass API limits are STRICTLY PROHIBITED (X Developer Policy violation).
- Scale comes only from enterprise access or a licensed firehose vendor.

### BR-055: Content Rights
- Only content the war room is licensed or subscribed to ingest MUST be ingested.
- Full text of paywalled articles MUST NOT be stored without a license.
