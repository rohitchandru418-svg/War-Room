# PRD Gap Analysis — War Room Social Listening Phase 1

**Purpose:** Document missing information, ambiguities, conflicting requirements, and open decisions that must be resolved before or during implementation.  
**Classification:** 🔴 Blocker | 🟡 Important | 🟢 Nice-to-know

---

## 1. Blockers — Must Resolve Before Build

### 1.1 X API Access Method (🔴 Blocker)
- **Gap:** The PRD identifies that 3M reads/month will not cover all parties + campaign detection + history backfill from 2026-05-10. The CTO must decide between pay-as-you-go, X Enterprise, or a licensed firehose vendor.
- **Impact:** Polling frequency, backfill completeness, streaming vs. polling architecture, and campaign detection all depend on this.
- **PRD says:** "Start X enterprise or firehose vendor talks now? Recommended: yes."
- **Action required:** Client/CTO decision. Contract lead time is a factor.

### 1.2 E-Paper Subscription Terms (🔴 Blocker)
- **Gap:** Whether existing e-paper subscriptions permit automated ingestion has not been confirmed per publisher.
- **Impact:** E-paper ingestion module cannot be built to full spec until terms are confirmed. Risk of legal exposure.
- **PRD says:** "Whether the subscription terms allow automated ingestion is a fact owed per publisher."
- **Action required:** Legal/client to confirm per publisher before ingestion code is written.

### 1.3 WhatsApp Alert Recipients (🔴 Blocker)
- **Gap:** Names and phone numbers of team leads to receive WhatsApp alerts are not provided.
- **Impact:** WhatsApp delivery module cannot be tested end-to-end.
- **Action required:** War room to supply recipient list and routing rules (which alerts go to which leads by entity).

### 1.4 Campaign Attribution Leads (🔴 Blocker)
- **Gap:** Who are the two shift leads authorised to confirm campaign attribution and approve ally-blame in CM summaries?
- **PRD says:** "Recommended: AadhavArjuna names two leads, one per shift."
- **Action required:** Client to name the specific individuals and confirm shift times.

### 1.5 Gold Set — Labelled Starter Data (🔴 Blocker)
- **Gap:** The gold set (real war-room material with human labels) is required before calibration, accuracy bars, and co-worker staging can begin. No data has been provided.
- **Impact:** F1 targets, alert thresholds, and co-worker staging all depend on a gold set built from real data.
- **PRD says:** "Ideally past clippings and posts the war room acted on."
- **Action required:** War room to supply labelled examples before calibration week. Build system can start but cannot be validated.

### 1.6 Tamil Text Recognition — Build or Buy (🔴 Blocker)
- **Gap:** The CTO must decide whether to build or buy Tamil OCR for e-papers and clippings.
- **Impact:** Affects vendor selection, cost model, and accuracy expectations.
- **Action required:** CTO decision with vendor evaluation.

---

## 2. Important — Resolve in First Sprint

### 2.1 Outlet Register (🟡)
- **Gap:** A draft outlet register is provided, but the war room must confirm it. Missing: specific RSS/sitemap URLs per outlet, YouTube channel IDs, which outlets have no machine-readable feed (requiring full crawl), which are party-affiliated.
- **PRD says:** "A draft list follows for the war room to confirm."
- **Action required:** War room to annotate the draft register.

### 2.2 Party and Leader List (🟡)
- **Gap:** "All Indian parties and key leaders" is not enumerated. The list of parties with their relationship tags (ally, supporting, opposition, neutral) is not provided.
- **Impact:** Entity recognition, entity tagging, Ally Mood panel, and alert routing depend on this list.
- **Action required:** War room to provide the full list with initial relationship tags.

### 2.3 Standing Tiers — Specific Accounts (🟡)
- **Gap:** The tier categories are defined (national media, state media, party leaders, influencers) but the initial population of handles/accounts per tier is not provided.
- **Impact:** Influence scoring and high-influence-post alerts cannot be calibrated without this.
- **Action required:** War room / social media lead to provide initial tier lists.

### 2.4 Co-worker Name and Identity (🟡)
- **Gap:** The co-worker's name in the product has not been confirmed by the client. The elephant avatar must use official TVK artwork "supplied by the party, not redrawn."
- **PRD says:** "Name and voice to be confirmed with the client."
- **Action required:** Client to supply TVK elephant artwork files and confirm the co-worker's name.

### 2.5 Watch Terms List (🟡)
- **Gap:** The specific watch terms (names, nicknames, Tamil and Tanglish spellings, hashtags) for each tracked entity are not provided.
- **Impact:** The filter step (Step 2) cannot be built or tuned without an initial watch-terms list.
- **Action required:** War room to supply initial watch-term lists per entity.

### 2.6 X Display Rules for Wall Screen (🟡)
- **Gap:** "Showing post text on the wall screen may fall under X's display requirements for offline or broadcast use. To be confirmed before launch."
- **Impact:** The big-screen view may need to be modified depending on X's ruling.
- **Action required:** Legal/CTO to confirm with X before launch.

### 2.7 Historical Backfill Scope (🟡)
- **Gap:** The volume of data from 2026-05-10 is unknown. The PRD says "daily volume: unknown; to be measured in the first two weeks."
- **Impact:** Database sizing, backfill strategy, and cost estimation cannot be finalised.
- **Action required:** Measure in first two weeks of operation.

---

## 3. Ambiguities — Clarify During Design

### 3.1 "News from the web" — Scope Boundaries
- **Ambiguity:** How far does crawling extend? Is there a cap on number of outlets? What is the process for adding a new outlet vs. the initial register?
- **Recommendation:** Define a maximum outlet count for Phase 1 and a staff workflow for adding new outlets.

### 3.2 Significance Test for "Change"
- **Ambiguity:** The PRD says "a change is called a change only when it clears a significance test or a confidence interval." The specific statistical test is not specified.
- **Recommendation:** CTO/data team to specify (e.g., z-test, chi-squared) during architecture phase.

### 3.3 Story Merging Logic
- **Ambiguity:** How are items grouped into Stories automatically? What NLP/similarity threshold triggers a merge? The PRD says "staff can merge or split" but does not specify the automatic merge algorithm.
- **Recommendation:** CTO to specify merge rules as part of ADR process.

### 3.4 "Response Opportunity" Trigger Definition
- **Ambiguity:** Story 31 says alert fires "when it crosses the response trigger" but the trigger is not numerically defined. It references "condemnation from other parties, media or public figures, and no TVK response."
- **Recommendation:** Define the trigger: e.g., ≥N mentions from non-TVK accounts expressing condemnation + 0 TVK leader responses within window W.

### 3.5 Shift Times
- **Ambiguity:** Shift handovers are mentioned (22:00 in the typical day example) but shift times are not formally defined.
- **Recommendation:** War room to define shift schedule (start/end times and shift names).

### 3.6 "Real Past Misses" for Problem Statement
- **Ambiguity:** The PRD says "real past misses to replace the illustrative ones" are needed from the war room. This affects the must-pass test cases.
- **Recommendation:** War room to supply 3–5 real historical incidents for must-pass testing.

### 3.7 Minimum Sample Size for "Not Enough Data"
- **Ambiguity:** Default minimum sample is 30, but it is applied globally. Should it vary by entity or time window?
- **Recommendation:** Use 30 globally for Phase 1; revisit in calibration.

### 3.8 "Sajan" — Role Formalisation
- **Ambiguity:** Sajan is named as "the named lead for response opportunities" in Story 31 but is not formalised in the user roles section.
- **Recommendation:** Treat as a named user in the War Room Lead role with a specific alert routing rule.

### 3.9 Presentation-Safe Mode Trigger
- **Ambiguity:** How is presentation-safe mode activated? Manual toggle? Scheduled? Who can toggle it?
- **Recommendation:** Manual toggle by any War Room Lead or Admin; visible state indicator on the big screen.

### 3.10 AI Model Selection
- **Ambiguity:** "One model or per-language models for labels" is listed as an open CTO decision. The co-worker model is also unspecified.
- **Recommendation:** CTO to decide based on gold-set results. For Phase 1 build, design for configurable model selection per capability.

---

## 4. Potential Conflicts

### 4.1 "No Budget Cap" vs. "Cost Efficiency Designed In"
- The PRD simultaneously says "no monthly cap" and "cost efficiency still designed in at every step."
- **Resolution:** Cost efficiency is an engineering discipline (cheap classification first, strong model only for reasoning), not a hard spend cap. Both statements are compatible.

### 4.2 "Wide Filter" vs. Weekly Audit
- The filter is intentionally wide ("letting in too much is fine"), but a weekly audit checks what was filtered out.
- **Resolution:** These are complementary. The filter errs toward inclusion; the audit catches systematic misses. No conflict.

### 4.3 Co-worker "Learns from Corrections" vs. "Original Always Kept"
- Corrections feed the co-worker's learning, but originals are never overwritten.
- **Resolution:** The correction feeds the training/example pipeline; the original label record is immutable. The UI shows both. No conflict — clarify in implementation.

---

## 5. Missing Technical Specifications

| Item | Status | Notes |
|------|--------|-------|
| Specific OCR vendor / library for Tamil | Missing | CTO decision |
| Specific NLP/embedding model for Story merging | Missing | CTO decision |
| WhatsApp Business API account / credentials | Missing | Client to supply |
| X API credentials and account tier | Missing | Depends on enterprise decision |
| E-paper access credentials per publisher | Missing | Client to supply |
| Tamil language detection library | Missing | Custom or vendor — CTO decision |
| Sentiment model — Tamil/Tanglish specific | Missing | Build or buy |
| Data retention period | Missing | Required for DPDP compliance |
| Backup and disaster recovery targets (RPO/RTO) | Missing | CTO to define |
| Infrastructure region (data residency for DPDP) | Missing | India region required |

---

## 6. Items the PRD Explicitly Defers to CTO

These are not gaps — they are intentionally deferred:

1. Architecture, ADRs, and build slices
2. X data acquisition strategy (enterprise vs. vendor)
3. X polling frequency vs. streaming
4. News crawling strategy (generic extraction vs. site-specific vs. headless browser)
5. One model vs. per-language models for labelling
6. Tamil OCR — build or buy
7. Story merging algorithm
8. Co-worker memory storage and correction mechanism
9. Final accuracy bars (set during calibration from real data)
10. All default thresholds (starting values given; finals set after two weeks of live data)
