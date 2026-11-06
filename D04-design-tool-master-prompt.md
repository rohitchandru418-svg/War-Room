# D04 — Master Prompt for the Design Tool

Paste the **Prompt** below into your design tool (Figma Make, v0, Lovable, Galileo, or similar) and attach the files in §1. Section 3 has short per-tool notes; section 4 has a sample-data pack so every screen is filled with realistic, consistent content.

---

## 1. Files to attach

| Priority | File | Why |
|---|---|---|
| 1 | `D01-design-principles-and-tokens.md` | Notion-style direction, brand `#A00000`, colour/type/spacing tokens |
| 2 | `D02-component-library.md` | Every component, variants and states |
| 3 | `D03-screen-designs.md` | Screen briefs, wireframes, flows |
| 4 | `PRD_TVK_War_Room_Social_Listening__Phase_1__X___News_.pdf` | Product intent and scenarios |
| 5 | `07-screen-specifications.md`, `08-interaction-and-state-specifications.md`, `06-information-architecture.md` | Fields, behaviours, states, navigation (if the tool accepts more files) |

If the tool accepts only a few files, attach D01–D03 and paste the Prompt; the sample-data pack below can be pasted after the Prompt.

## 2. Prompt

```
You are a senior product designer. Design the complete UI/UX for "War Room", a staff-only
social-listening web application that collects X posts and news, labels and weighs them, groups
them into Stories, detects coordinated campaigns, and alerts a political war room, with an AI
co-worker that explains everything with cited evidence. Use the attached files as the source of
truth: D01 (style and tokens), D02 (components), D03 (screens and flows), and the PRD.

STYLE
- Notion-style: calm, document-like pages, white space, hairline borders, tinted pastel tags,
  left sidebar, tables that behave like Notion databases, a right-hand "side peek" panel for
  evidence, callout blocks, inline editing, line icons (Lucide) and optional emoji page icons.
- Brand colour is #A00000 (deep crimson). Use it sparingly (5-10% of any screen): primary
  buttons, active navigation, links, focus ring, Tier-1 alert treatment, co-worker identity.
  Never use large red fills on working pages. White text on #A00000.
- Because red also means negative/critical, follow D01 section 3.3: negative sentiment is a soft
  rose tag with a down-arrow icon and the word "Negative"; destructive actions are secondary
  buttons with a trash icon and a confirm step. Never rely on colour alone.
- Typography: Inter for UI and Noto Sans Tamil for Tamil; tabular numbers for statistics; Tamil
  text needs line-height 1.6+, no letter-spacing, and room for ~30% longer strings.
- Light theme for working pages, dark theme for the big screen. Provide both themes for all screens.

NON-NEGOTIABLE PRODUCT RULES (design them visibly)
1. Every percentage shows its denominator and freshness: "of 412 mentions - last updated 12:04".
2. Below the minimum sample (30) show "Not enough data (N = 18, minimum 30)" instead of a percentage.
3. Weighted scores always appear beside raw counts; one tap shows the top 3 contributing accounts.
4. Every number, story, alert and campaign opens an evidence peek listing the underlying posts or
   articles with author/outlet, engagement, time, label, the label's reason, and a link to the original.
5. Every AI label shows a reason; any label can be corrected with a required reason; the original
   stays visible in history; corrections that soften criticism of primary entities need a second reviewer.
6. A source failure is always visible ("X data delayed since 11:52"); an error never looks like
   "no new items".
7. Campaign wording is "Possible coordination" until a lead confirms; never name a party as behind
   it before confirmation. Ally-blame in a CM summary shows "Reviewed by <name>" before it is sent.
8. The co-worker never posts, replies or publishes; it only reads, explains, drafts and recommends.
9. Presentation-safe mode: only aggregates, public actors' names and Story titles are shown; no
   private-citizen handles or internal notes. A visible badge shows when it is on.
10. Nothing client-specific is baked in: entity names, relationship labels ("ally") and summary
    names ("CM summary") are client-configured vocabulary tokens.
11. No flashing faster than 1 Hz anywhere; the "flash" for Tier-1 alerts is a slow pulse.

SCREENS TO DESIGN (see D03 for layout and content)
0 Sign-in (+ CM PIN variant) | 1 Dashboard | 2A Stories list, 2B Story detail, 2C Response queue |
3A Alerts list, 3B Alert detail (mobile-ready) | 4A Campaigns list, 4B Campaign detail with
attribution form | 5A Co-worker chat, 5B Briefs and handovers editor | 6 Upload clippings |
7 Settings (7A Entities and tags, 7B Watch terms, 7C Standing tiers, 7D Outlets, 7E Thresholds,
7F Alert recipients, 7G Event calendar) | 8 Source health and usage | 9 Corrections and accuracy
(9A stage ladder, 9B weekly accuracy, 9C review queue) | 10 Big screen (1920x1080 dark) |
11 CM summary (tablet-first) | 12 Filter audit, Audit log, Notifications.

DELIVERABLES
1. A design system page: colour tokens (light and dark), type scale, spacing, radii, icons, and
   all components from D02 with variants and states (default, hover, focus, loading, empty,
   error, stale/delayed).
2. All screens above in Desktop 1440x900 (light and dark), Tablet 834x1112, Mobile 390x844, and
   Big screen 1920x1080. Provide English and Tamil versions of screens 1, 3, 5, 10 and 11.
3. State frames for the Dashboard and Alert detail: loading, empty, error, stale/delayed,
   not-enough-data, presentation-safe, WhatsApp-not-delivered.
4. Clickable prototype flows (D03 "Prototype flows"): typical day, must-pass 1, must-pass 2,
   campaign confirmation, bias guard, clipping upload, CM mode.
5. A short handoff note listing any design decision that needs the owner's call (for example
   the brand-red versus negative-red choice).

CONSTRAINTS
- Use the sample data pack for all content; do not invent real people's statements.
- Keep accessibility at WCAG 2.2 AA: contrast, visible focus ring, 44px touch targets on touch
  devices, keyboard operation, screen-reader text for charts.
- Use consistent components; do not create one-off styles.
- Keep the interface calm; reserve strong visual urgency for Tier-1 alerts and source failures.
- Ask no clarifying questions; where information is missing, choose the simplest option that
  follows the attached documents and list it in the handoff note.
```

## 3. Tool-specific notes

| Tool | Tip |
|---|---|
| Figma Make | Attach D01–D03 as context, paste the Prompt, then run one screen group at a time (Dashboard + Alerts first) so styles stay consistent; ask it to create shared variables for the tokens in D01 §12 |
| v0 / Lovable (code-first) | Ask for Next.js + Tailwind + Radix with `components/numbers/*` named as in D02; map tokens to Tailwind theme per D01 §12; generate static screens with mock data before wiring anything |
| Galileo / Uizard-style | Generate the Dashboard, Alert detail and Big screen first; then replicate the style across the rest using the exported style guide |
| Any tool | If output drifts from the brand, re-state: "brand `#A00000` is an accent only; neutrals everywhere else; tags are pastel with dark text" |

Recommended order: design system page → Dashboard → Alert detail → Big screen → Story detail → Campaign detail → Co-worker → CM summary → Admin and operations screens → mobile/tablet variants → prototype links.

## 4. Sample-data pack (illustrative placeholders)

Use these consistently. Names of private individuals are fictional; public offices are as in the PRD.

**Client vocabulary (TVK example):** party = "Party", ally = "Ally", leadership summary = "CM summary", co-worker name = "Co-worker" (final name TBD).

**Entities:** TVK; CM Vijay; AadhavArjuna. Parties for Ally Mood: "Ally Party A" (calm), "Ally Party B" (unhappy), "Supporting Party C" (calm).

**Dashboard numbers:** TVK negative 34 % weighted (31 % raw) of 412; CM Vijay negative 29 % (27 % raw) of 380; AadhavArjuna "Not enough data" N = 18. Last updated 12:04. Trend chip on TVK: "▲ +6 pts vs previous 24 h" (passed significance).

**Alerts:**
1. Tier 1 · Influence · "National journalist (verified) criticises CM Vijay" · Reason: "verified, 1.2M followers, national media tier, 4,000 reposts in 20 minutes" · 12 min ago · Unassigned · WhatsApp delivered to 3 leads.
2. Tier 1 · Response opportunity · "Remark at a rival party event is drawing condemnation" · Reacting: Ally Party A spokesperson, Public figure X, News channel Y · "No TVK response yet" · 6 min ago · routed to the named response lead.
3. Tier 2 · Spike · "Negative mentions about TVK up 2.4× normal in 15 minutes" · baseline and current level shown.
4. Tier 2 · Campaign · "Possible coordination: about 300 accounts, 70 % under 3 months old, near-identical Tanglish lines" · Confidence: Medium.
5. Tier 2 · Ally · "Ally Party B MLAs criticise a Cabinet decision" · of 41 posts.

**Stories:** "Cabinet decision criticised by ally MLAs" (9 outlets, 1,200 posts, Rising); "New hashtag against AadhavArjuna (possible coordination)" (also in print: a Tamil weekly); "Anonymous insinuation in a Tamil weekly column" (Story of one, Unattributed).

**Mentions (for evidence lists):** vary X posts, a news article, an e-paper story (page 4), a clipping (page 7, "check scan"); stances Positive, Negative, Neutral, Sarcastic (target stance: supportive); reasons like "Sarcastic; praises while mocking critics."

**Campaign detail:** account-age histogram (many accounts < 90 days), shared phrase clusters with counts, posts per 5 minutes with a sharp rise at 23:05, "Confirmed by" empty.

**Co-worker conversation (Tamil question, Tamil answer):** Q: "Congress mentions ஏன் negative ஆனது?" A: summary of three posts from ally MLAs about the Cabinet decision, negative ally mentions 22 % → 41 %, footer "Based on 64 posts", citations [1][2][3]. (Placeholder Tamil for native review.)

**Corrections:** Tanglish post changed from Negative to "Positive (sarcastic)" with reason "Mocks critics; supports the minister." Second-review example: "Negative → Neutral on a primary entity" pending approval.

**Source health:** X delayed since 11:52 (rate limited); an outlet "blocked by site"; e-paper healthy; AI model healthy; WhatsApp: one failed delivery retrying. X usage: 82 % of the monthly cap, "Sampling active" off until 95 %.

**Big screen:** same entity numbers, three alerts (Tier 1 pulsing), Ally Mood panel, two rising Stories, "Presentation-safe" badge on, no post text.

**CM summary:** three changes (Cabinet decision criticism rising; hashtag push possible coordination; print-only insinuation) with one-line causes, "Reviewed by <name>" on the ally-attributed item, one decision needed (response opportunity open since 17:20).
