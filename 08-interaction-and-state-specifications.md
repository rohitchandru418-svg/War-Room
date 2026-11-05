# 08 — Interaction and State Specifications

---

## 1. Loading States

### 1.1 Initial App Load
- Show skeleton loaders for entity sentiment cards, alert list, and stories list
- Source health banner appears immediately (no skeleton — it's always visible)
- Load order priority: alerts first, then entity scores, then stories
- If auth check fails: redirect to login with return URL preserved

### 1.2 Dashboard Refresh (Auto-refresh)
- Sentiment cards: update in-place with fade-in of new values (no full page reload)
- Alert list: new alerts animate in from top; resolved alerts fade out
- Rising stories: reorder smoothly (CSS transition)
- Stale indicator: if WebSocket connection drops, show "Reconnecting..." in source health banner; show last-updated time on all numbers

### 1.3 Co-worker Answer Loading
- Show "Thinking..." indicator with TVK elephant avatar animation after message send
- Streaming response: show text as it arrives (character-by-character or token-by-token)
- Citation links appear after full text is delivered
- If response exceeds 10 seconds: show "Taking a moment..." sub-label (not an error)
- If response fails: show error state (see Section 3.5)

### 1.4 Clipping Upload Processing
- Upload: immediate progress bar for file transfer
- OCR processing: show "Processing..." status with estimated time
- Completion: status updates to "Mention created" or "Check scan required"
- Each file tracked independently (multi-file upload)

---

## 2. Empty States

### 2.1 No Mentions in Window
- Entity sentiment card: "not enough data (fewer than 30 mentions in this window)"
- Stories list: "No stories in this period. The filter may be catching new content — check back shortly."
- Ally Mood panel: "No mentions from ally public figures in the last 24 hours."

### 2.2 No Active Alerts
- Alert section: "No active alerts right now. All is quiet."
- Do NOT show this message as a source health indicator — always check source health separately.

### 2.3 No Response Opportunities
- Response queue: "No pending response opportunities."

### 2.4 New Client / First Day
- Dashboard: "Setting up your data. The first mentions will appear within a few minutes of connection."
- History backfill: "Loading historical data from [date]. This may take several hours for the full history."

### 2.5 Filter Audit — No Items to Review
- "This week's audit sample has been reviewed. Missed-item rate: X%."

---

## 3. Error States

### 3.1 Source Delay / Outage (non-blocking)
**When:** A data source (X API, news outlet feed, e-paper, AI model) fails or is delayed.  
**Display:**
- Source health banner at top of all screens: "[Source name] data delayed since HH:MM"
- Affected panels show stale timestamp: "Last updated HH:MM" in amber/orange
- Sentiment percentages: retain last value but show "⚠ as of HH:MM"
- Never show stale data without the timestamp

**What must NOT happen:**
- A delayed source must NEVER look like "no new items"
- Sentiment must NEVER silently stop updating without an indicator

### 3.2 AI Model Unavailable
**Display:**
- Co-worker chat: "The co-worker is temporarily unavailable. Dashboards and alerts continue to work."
- Alert explanations: "Explanation unavailable — AI model offline."
- Scheduled briefs and handovers: "Could not generate — AI model offline. Please write manually."
- Do NOT show stale co-worker answers as if they are current

### 3.3 WhatsApp Delivery Failure
**Display:**
- In-app alert shows "WhatsApp not delivered" badge
- War Room Lead receives in-app notification
- Admin platform health screen shows failed delivery in log

**Recovery:**
- System retries automatically (retry count and interval defined in deployment config)
- After max retries: escalate to in-app only, mark lead notified in-app

### 3.4 X API Limit Approaching
**Display:**
- Admin health screen: usage bar turns amber at 80%, red at 95%
- Admin receives in-app notification at 50%, 80%, 95%
- All numbers derived from sampled data: labelled "sampled" in all display locations

### 3.5 Co-worker Answer Failure
**Display:**
- "I couldn't get an answer right now. Please try again or ask a different question."
- Error is logged with question, timestamp, error type (not shown to user)
- Do NOT show a previous answer as if it is a new one

### 3.6 OCR Low Confidence (Clipping / E-paper)
**Display:**
- Mention shows "check scan" badge
- Specific words/lines with low confidence are highlighted
- Team member can manually correct the extracted text before confirming

### 3.7 Label Correction — Second Reviewer Required
**Display:**
- Correction form shows: "This correction softens a negative label / touches a high-severity Story. It will be held for second review."
- Correction is submitted but marked "pending review"
- Second reviewer receives in-app notification
- Aggregates do NOT update until second review is complete
- If second reviewer rejects: original correction is logged as rejected with reason; original label remains active

### 3.8 Network / General Error
**Display:**
- Persistent "Connection lost — trying to reconnect" banner
- All numbers show last-updated timestamps
- Actions (acknowledge, correct, upload) queue locally and retry on reconnect [PROPOSED — confirm with CTO]

---

## 4. Success States

### 4.1 Alert Acknowledged
- Flash animation stops immediately for all users
- Alert card updates: "Taken by <name> at HH:MM"
- In-app confirmation: subtle toast "Alert acknowledged"

### 4.2 Alert Assigned
- Alert card updates: "Assigned to <name> by <assigner> at HH:MM"
- Assigned user receives in-app notification

### 4.3 Label Corrected
- Label shows immediately as corrected: "corrected by <name>"
- Original label visible in expandable history
- Toast: "Label corrected"
- If second reviewer required: "Correction submitted — awaiting second review"

### 4.4 Campaign Attribution Confirmed
- Campaign status badge changes from "Possible Coordination" to "Confirmed: [Party]"
- Alert headline updates
- Logged: lead name, timestamp, reason

### 4.5 Clipping Uploaded and Processed
- Status: "Mention created"
- Quick link to new Mention in context of its Story (if matched)

### 4.6 Configuration Saved
- Inline confirmation on changed row: "Saved" with timestamp
- Audit log entry visible immediately

### 4.7 Response Marked (acted on / not needed)
- Story removed from response queue
- Status logged with actor name and decision

---

## 5. Real-Time Update Behaviour

### 5.1 Alert Flash
- Tier 1 alert: red flash on affected entity card AND alert banner, pulsing until acknowledged
- Flash does not block reading or interaction with other screen elements
- Flash stops for ALL users simultaneously on first acknowledgement

### 5.2 Sentiment Updates
- In-place update: number transitions smoothly (animated counter or fade)
- Change direction indicated: ↑ positive shift, ↓ negative shift (shown for 30 seconds then removed)

### 5.3 Story Arrival
- New Story or Story reaching top-3 rising: subtle slide-in animation from right
- Badge: "New" or "Rising fast" for first 5 minutes

### 5.4 Source Health Banner
- Appears / disappears without page refresh
- Updates the HH:MM timestamp in real time as delay grows

---

## 6. Presentation-Safe Mode Transitions

- **Activation:** Any lead taps "Presentation Mode" button → takes effect within 2 seconds across all displays
- **Visual state:** Subtle overlay or badge on big screen: "Presentation Mode"
- **Deactivation:** Lead taps again → normal mode resumes
- **Private data masking:** Handles that are private citizens shown as "Account [N]" or hidden; internal notes hidden

---

## 7. Language Switching

### 7.1 UI Language
- Toggle in user settings: English / Tamil
- Applies to navigation labels, UI chrome, system messages
- Does NOT affect content language (content is in whatever language it was written)

### 7.2 Co-worker Interaction
- Co-worker always responds in the language of the question
- If question is mixed (Tanglish), co-worker responds in Tamil or English (whichever is clearer — PROPOSED: default to Tamil for Tanglish input)

---

## 8. Correction Flow — Step-by-Step Interaction

```
1. Team member sees a Mention label they disagree with
2. Tap "Correct label" button on the Mention
3. Correction panel slides open:
   - Current label shown (read-only)
   - New label fields: entity / stance / claim type / language
   - Reason text field (required, min 10 characters)
4. Submit
5. System checks:
   a. Is this a softening correction on TVK entity? → flag for second review
   b. Is this a high-severity Story? → flag for second review
6. If second review required:
   - "Correction submitted — awaiting second review"
   - Aggregates NOT updated yet
   - Second reviewer notified in-app
7. If no second review required (or second review approved):
   - Mention label updated: "corrected by <name>"
   - Aggregates recalculate
   - Original label visible in history accordion
   - Toast: "Label corrected"
8. Correction added to gold set queue (async)
```

---

## 9. Co-worker Capability Stage Indicators

The admin screen shows each capability's stage with:
- Green check: at this stage and accuracy above bar
- Yellow warning: at this stage but accuracy near lower bound (within 5% of bar)
- Red down-arrow: just demoted — reason shown
- Stage history chart: last 30 days of stage and accuracy

Stage demotion notification (in-app to leads):
- "[Capability] has been demoted from [stage] to [previous stage]. Accuracy dropped to [X]% (bar: [Y]%). Human review has resumed for this capability."
