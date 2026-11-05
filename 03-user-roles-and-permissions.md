# 03 — User Roles and Permissions

---

## 1. Role Definitions

### Role 1: System Admin (Minsky Team)
**Who:** Minsky engineering and operations staff  
**Context:** Builds, maintains, and operates the platform. Has full system access.

### Role 2: War Room Admin
**Who:** Designated senior war room staff  
**Context:** Manages configuration for the TVK client: entities, tiers, watch terms, outlet weights, thresholds, recipients.

### Role 3: War Room Lead
**Who:** AadhavArjuna, Meena (social media team lead), Sajan (response opportunity lead), other designated shift leads  
**Context:** Senior staff who receive high-severity alerts, approve campaign attribution, review ally-blame before CM delivery, manage the response queue.

### Role 4: Team Member
**Who:** War room media team, social media team, content team staff  
**Context:** Day-to-day monitoring, labelling, correction, clipping upload. No configuration access.

### Role 5: CM View (Read-Only)
**Who:** CM Vijay  
**Context:** Read-only access to the CM summary and big screen. No editing or correction access. Presentation-safe mode may be active.

### Role 6: Big Screen (Passive Display)
**Who:** The war room wall display  
**Context:** Not an interactive user role; a display mode. Subject to presentation-safe mode restrictions.

---

## 2. Permission Matrix

| Capability | Sys Admin | War Room Admin | War Room Lead | Team Member | CM View |
|-----------|:---------:|:-------------:|:-------------:|:-----------:|:-------:|
| **Data Access** | | | | | |
| View big screen | ✅ | ✅ | ✅ | ✅ | ✅ |
| View CM summary | ✅ | ✅ | ✅ | ✅ | ✅ |
| View team drill-down | ✅ | ✅ | ✅ | ✅ | ❌ |
| View raw Mention detail | ✅ | ✅ | ✅ | ✅ | ❌ |
| View private citizen handles | ✅ | ✅ | ✅ | ✅ | ❌ |
| View internal notes | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Alerts** | | | | | |
| Acknowledge alerts | ✅ | ✅ | ✅ | ✅ | ❌ |
| Assign alerts | ✅ | ✅ | ✅ | ✅ | ❌ |
| Receive WhatsApp alerts | ❌ | ✅ | ✅ | ❌ | ❌ |
| Configure alert recipients | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Labels and Corrections** | | | | | |
| Correct any AI label | ✅ | ✅ | ✅ | ✅ | ❌ |
| Second-review biased corrections | ✅ | ✅ | ✅ | ❌ | ❌ |
| View correction history | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Campaign Attribution** | | | | | |
| View campaign alerts | ✅ | ✅ | ✅ | ✅ | ❌ |
| Confirm campaign attribution | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Ally/CM Review** | | | | | |
| Review ally-blame before CM | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Response Queue** | | | | | |
| View response queue | ✅ | ✅ | ✅ | ✅ | ❌ |
| Mark response queue item acted/not needed | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Content Upload** | | | | | |
| Upload clippings | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Configuration** | | | | | |
| Manage standing tiers | ✅ | ✅ | ❌ | ❌ | ❌ |
| Manage party relationship tags | ✅ | ✅ | ✅ | ❌ | ❌ |
| Manage watch terms | ✅ | ✅ | ❌ | ❌ | ❌ |
| Manage outlet weights/tiers | ✅ | ✅ | ❌ | ❌ | ❌ |
| Manage alert thresholds | ✅ | ✅ | ❌ | ❌ | ❌ |
| Manage event calendar | ✅ | ✅ | ✅ | ❌ | ❌ |
| Toggle presentation-safe mode | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Admin / Audit** | | | | | |
| View audit logs | ✅ | ✅ | ❌ | ❌ | ❌ |
| View API usage / platform limits | ✅ | ✅ | ❌ | ❌ | ❌ |
| View weekly filter audit | ✅ | ✅ | ❌ | ❌ | ❌ |
| View accuracy trend charts | ✅ | ✅ | ✅ | ❌ | ❌ |
| View co-worker stage and error trend | ✅ | ✅ | ✅ | ❌ | ❌ |
| Conduct weekly filter audit review | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Co-worker** | | | | | |
| Ask co-worker questions | ✅ | ✅ | ✅ | ✅ | ❌ |
| Receive co-worker shift handover | ✅ | ✅ | ✅ | ❌ | ❌ |
| Correct co-worker answers | ✅ | ✅ | ✅ | ✅ | ❌ |

---

## 3. Special Named Users

### Sajan — Response Opportunity Lead
- Role: War Room Lead
- Special routing: All "Response opportunity" WhatsApp alerts must be routed to Sajan (or the designated shift equivalent)
- Responsibility: Briefs AadhavArjuna on response opportunities

### Shift Leads for Campaign Attribution / Ally-Blame Review
- Role: War Room Lead
- Special authority: One per shift; named by AadhavArjuna
- Responsibility: Confirm campaign attribution; approve ally-blame before it reaches the CM summary
- **[GAP: specific names to be supplied by client — see PRD-GAP-ANALYSIS.md]**

---

## 4. Interface Language Preference
- Each user can set their preferred interface language: English or Tamil
- The co-worker answers in the language the question is asked in (not the UI language setting)

---

## 5. Authentication Notes
- Staff-only access; no public-facing authentication
- Full auth specification in `13-authentication-and-security.md`
- No account links to private citizens or petition records
- CM View access may be a separate, simplified login or a device-specific kiosk mode

---

## 6. Audit Trail Requirements for Role Actions

The following role actions MUST be logged with actor, timestamp, and action detail:

| Action | Logged Fields |
|--------|-------------|
| Label correction | User, original label, new label, reason, timestamp |
| Tier change (add/remove account) | User, tier, account, change type, timestamp |
| Campaign attribution confirmation | User, campaign ID, reason, timestamp |
| Party relationship tag change | User, party, old tag, new tag, timestamp |
| Alert acknowledgement/assignment | User, alert ID, action, assignee (if assigned), timestamp |
| Ally-blame review (CM summary) | User, summary ID, approval/hold, timestamp |
| Presentation-safe mode toggle | User, action (on/off), timestamp |
| Response queue mark (acted/not needed) | User, story ID, decision, timestamp |
| Any config change (thresholds, watch terms, outlet weights) | User, field, old value, new value, timestamp |
