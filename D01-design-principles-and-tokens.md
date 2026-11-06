# D01 — Design Principles, Brand and Tokens

**Product:** War Room Social Listening (Phase 1: X + News) · **Style:** Notion-style · **Brand colour:** `#A00000`

Feed this file to the design tool together with **D02 (components)**, **D03 (screens)** and **D04 (master prompt)**. For behaviour and content, also attach the PRD, `06-information-architecture.md`, `07-screen-specifications.md` and `08-interaction-and-state-specifications.md`. Where a design choice here conflicts with those, the behaviour in 06–08 wins; the visual treatment here wins.

Markers: `[PRD]` required by the PRD · `[PROPOSED]` design recommendation · `[DECISION]` needs the owner's call.

---

## 1. Who it is for and what the design must achieve

Users: war-room media, social and content teams (daily, desktop-first, some phone use after a WhatsApp alert), the head of the war room (summary and alerts), CM Vijay (30-second read on a tablet/phone/screen), and the **big wall screen** that visitors can see. Staff-only app `[PRD]`.

The design's job, in order:
1. **Trust.** Every number shows its denominator, its freshness and its evidence. The UI never looks more certain than the data is `[PRD principle: easy to understand on top, statistically correct underneath]`.
2. **Speed to understanding.** One calm page tells the story; one tap reaches the evidence. A critical alert is unmistakable within a second.
3. **Control.** People can always correct the machine, acknowledge, assign, and see what changed and who did it.
4. **Calm by default, urgent by exception.** Most of the product feels like a quiet, well-organised workspace. Only Tier-1 alerts and source failures interrupt.

## 2. What "Notion-style" means here `[PROPOSED]`

| Notion trait | How we apply it |
|---|---|
| Document/page feel, content first | Pages are a title with an icon, a properties row, then blocks (stat blocks, tables, callouts, timelines). Minimal chrome |
| White space and hairlines instead of boxes | 1 px low-contrast borders, no heavy shadows, generous padding, text carries the hierarchy |
| Calm neutral palette, tinted tags | Warm off-black text on white; semantic colour appears mainly as soft pastel tags with dark text |
| Left sidebar workspace navigation | Collapsible sidebar with grouped sections; the client's name/logo plays the "workspace" role |
| Databases as tables with properties and filters | Mentions, Stories, alerts, outlets, tiers, audit logs are table views with property columns, filter chips, sort, and a row-click **side peek** |
| Side peek panel | Evidence, mention detail and correction open in a right-hand peek panel without leaving the page |
| Callout blocks | System explanations ("Why this alert fired"), source-health notices, "not enough data" notes |
| Hover affordances | Row actions, drag handles and "+" appear on hover; keyboard shortcuts for power users |
| Inline editing | Settings (tiers, watch terms, thresholds) edit in place with an inline "Saved" confirmation |
| Emoji/page icons | Optional icons on pages and Stories to aid scanning; Lucide line icons everywhere else |

Deliberately **different** from Notion because this is an operations tool: numbers are bigger and bolder, status is always visible (freshness stamp, source health), and Tier-1 alerts break the calm on purpose.

## 3. Brand colour `#A00000`

`#A00000` is a deep crimson. On white it has about **8.4:1** contrast (passes WCAG AA and AAA for normal text); on near-black it is too dark (≈ 2:1), so dark mode uses a lighter tint for text and icons (§4.3).

### 3.1 Brand ramp `[PROPOSED — derived tints/shades; verify with a contrast tool]`

| Token | Hex | Use |
|---|---|---|
| `brand-50` | `#FDF2F2` | Selected row/nav background, callout background |
| `brand-100` | `#FAE1E1` | Hover on brand-tinted surfaces |
| `brand-200` | `#F3BDBD` | Borders on brand callouts, chart fills (light) |
| `brand-300` | `#E58F8F` | Disabled brand button, decorative |
| `brand-400` | `#CC5555` | Dark-mode icon accent |
| `brand-500` | `#B82A2A` | Hover state of primary button (lighter step) |
| **`brand-600`** | **`#A00000`** | **Primary brand: primary buttons, links, active nav indicator, focus ring, key accents** |
| `brand-700` | `#820000` | Pressed state, brand text on tinted surfaces |
| `brand-800` | `#640000` | Strong text on brand-50/100 |
| `brand-900` | `#470000` | Rare; dark surfaces tinted with brand |

### 3.2 Usage rules

- Brand colour is an **accent**, about 5–10 % of any screen: primary action, active navigation, links, focus ring, the Tier-1 alert treatment, the co-worker identity. Everything else is neutral.
- Never use it for large background fields on working pages. The only large brand fills are the login hero and the optional big-screen Tier-1 flash.
- Text on `#A00000` is white (`#FFFFFF`), 8.4:1.

### 3.3 The red-on-red problem `[DECISION]`

Brand is red, but red also means "negative", "error" and "danger" in this product (negative sentiment, critical alert, delete). If everything red means something, nothing does. Resolution `[PROPOSED]`:

| Meaning | Treatment | Never rely on colour alone |
|---|---|---|
| Brand / primary action | Solid `#A00000` button, white text | Verb label ("Acknowledge", "Confirm") |
| **Tier-1 critical alert** | `#A00000` left bar + solid "TIER 1" badge + pulse icon on `brand-50` surface | Text "TIER 1", icon, position at top |
| **Negative sentiment** | Soft rose tag (`#FFE2DD` bg, `#5D1715` text) with ▼ icon and the word "Negative" | Word + icon |
| Error/failure | Neutral callout with a warning icon and plain text; a small rose accent only on the icon | Icon + copy |
| Destructive action (delete, remove) | **Secondary** (outlined/text) button with trash icon, confirm dialog; never the primary brand button | Icon + confirm dialog |

If the owner prefers, change the semantic "negative" family to an orange-red (`#D9480F`) to separate it from the brand; flag this early because it affects every chart.

## 4. Colour tokens

Names are semantic. Implement as CSS variables / Tailwind theme (§12). Light first, dark second.

### 4.1 Neutrals (inspired by Notion's warm greys)

| Token | Light | Dark |
|---|---|---|
| `bg` (page) | `#FFFFFF` | `#191919` |
| `bg-subtle` (sidebar, table header) | `#F7F6F3` | `#202020` |
| `bg-hover` | `rgba(55,53,47,0.06)` | `rgba(255,255,255,0.055)` |
| `bg-selected` | `#FDF2F2` (brand-50) | `rgba(229,115,115,0.14)` |
| `border` | `rgba(55,53,47,0.09)` | `rgba(255,255,255,0.094)` |
| `border-strong` | `rgba(55,53,47,0.16)` | `rgba(255,255,255,0.18)` |
| `text` | `#37352F` | `rgba(255,255,255,0.90)` |
| `text-secondary` | `rgba(55,53,47,0.65)` | `rgba(255,255,255,0.60)` |
| `text-tertiary` | `rgba(55,53,47,0.45)` | `rgba(255,255,255,0.40)` |
| `text-on-brand` | `#FFFFFF` | `#FFFFFF` |

### 4.2 Brand and interaction

| Token | Light | Dark |
|---|---|---|
| `accent` (buttons, links, active indicator) | `#A00000` | `#E57373` (text/icons), `#A00000` fills keep white text |
| `accent-hover` | `#B82A2A` | `#EF8A8A` |
| `accent-pressed` | `#820000` | `#D65F5F` |
| `focus-ring` | `2px #A00000` + 2px offset | `2px #E57373` + 2px offset |
| `link` | `#A00000` (underline on hover) | `#E57373` |

### 4.3 Semantic tags (pastel background + dark text, Notion-style)

| Token | Light bg / text | Dark bg / text | Used for |
|---|---|---|---|
| `tag-gray` | `#E3E2E0` / `#32302C` | `#373737` / `#E6E6E6` | Neutral stance, unknown, "model-only" |
| `tag-green` | `#DBEDDB` / `#1C3829` | `#243D30` / `#BFE3C8` | Positive stance, healthy source, ally calm |
| `tag-rose` | `#FFE2DD` / `#5D1715` | `#5A2A27` / `#FFC9C2` | Negative stance (with ▼) |
| `tag-amber` | `#FDECC8` / `#402C1B` | `#5A4A28` / `#F5DCA0` | Delayed, Tier 2, low confidence, "check scan" |
| `tag-purple` | `#E8DEEE` / `#412454` | `#483A55` / `#DCCBEA` | Sarcastic stance (with ~), co-worker content |
| `tag-blue` | `#D3E5EF` / `#183347` | `#2B4558` / `#BEDBEC` | Informational, news source badge |
| `tag-orange` | `#FADEC9` / `#49290E` | `#59402A` / `#F2CBA8` | Possible coordination, caution |
| `tag-pink` | `#F5E0E9` / `#4C2337` | — | Optional category |
| `tag-brown` | `#EEE0DA` / `#442A1E` | — | Optional category |

Stance colours (never colour-only): **Positive** green ▲ · **Negative** rose ▼ · **Neutral** gray – · **Sarcastic** purple ~ (with the stance toward the entity in the reason line).

### 4.4 Alert tiers `[PRD tiers: big screen/WhatsApp, in-app, nothing]`

| Tier | Meaning | Visual |
|---|---|---|
| Tier 1 | Flashes on big screen, WhatsApp to leads | `#A00000` left bar (4 px) + solid badge "TIER 1" + slow-pulse dot, on `brand-50` surface |
| Tier 2 | In-app only | `tag-amber` badge "TIER 2", no bar |
| (Tier 3) | No alert row | — |

### 4.5 Data-visualisation palette `[PROPOSED]`

Entities are **configured per client** and never hard-coded; assign colours from this colour-blind-safe sequence by entity order: `#A00000`, `#2F6FA3`, `#C77D0A`, `#4B8B5F`, `#7E57A1`, `#6B6B6B`. Primary entity is brand. Sentiment stacks use the stance colours of §4.3 at solid strength: positive `#4B8B5F`, neutral `#B8B6B0`, sarcastic `#7E57A1`, negative `#C0504A`.

## 5. Typography

- UI: **Inter** (Notion-like); fallbacks `ui-sans-serif, system-ui, "Segoe UI", sans-serif`.
- Tamil: **Noto Sans Tamil**. Font stack for any text that may contain Tamil: `Inter, "Noto Sans Tamil", system-ui, sans-serif`. Set `lang="ta"` on Tamil strings `[PRD: interface English and Tamil]`.
- Numbers: Inter with `font-variant-numeric: tabular-nums` for every statistic.
- Mono (IDs, hashtags in evidence, code-like values): `ui-monospace, "JetBrains Mono"`.

| Role | Size / line | Weight | Notes |
|---|---|---|---|
| Page title | 32 / 40 | 700 | With optional 40 px icon above-left, Notion style |
| H2 section | 24 / 32 | 600 | |
| H3 block title | 18 / 26 | 600 | |
| Body | 15 / 24 | 400 | Default text size (Notion ≈ 16) |
| Small | 13 / 20 | 400 | Table cells, metadata |
| Micro | 12 / 16 | 500 | Tags, timestamps, "of N mentions" |
| Stat large | 40 / 44 | 700 | Entity stat blocks |
| Stat XL (big screen) | 96 / 100 | 700 | Wall display |

**Tamil rules:** line-height ≥ 1.6 for Tamil body text (conjuncts and vowel signs need space); no letter-spacing or uppercase transforms; allow ~20–30 % longer strings than English (buttons and tabs must not truncate Tamil labels; allow wrapping or widen); minimum 14 px for Tamil body; never mix Tamil into small-caps or ALL-CAPS labels (use sentence case in both languages).

## 6. Spacing, shape, elevation, icons, motion

- **Spacing scale (px):** 2, 4, 8, 12, 16, 24, 32, 48, 64. Page content padding 24 (desktop) / 16 (mobile).
- **Radius:** 3 (inputs, tags, table cells), 6 (buttons, cards, callouts), 8 (modals, peek panel). Notion-small radii; no pills except tags.
- **Elevation:** flat. Hairline borders by default. Shadows only on floating layers: popover/menu `0 2px 8px rgba(15,15,15,.12), 0 0 0 1px rgba(15,15,15,.05)`; modal `0 8px 24px rgba(15,15,15,.18)`.
- **Icons:** Lucide, 1.5 px stroke, 16 px in rows and 20 px in nav and headers. Optional emoji page icons for Stories and briefs.
- **Motion:** 120–180 ms ease-out for hover, expand and peek; 200 ms for panels. Respect `prefers-reduced-motion` (then no animation except state change). **No flashing faster than 1 Hz**; the PRD's "flash" means a slow pulse of the Tier-1 treatment, never a strobe (WCAG 2.3.1).

## 7. Layout

| Element | Spec |
|---|---|
| Breakpoints | Mobile < 640, Tablet 640–1023, Desktop 1024–1439, Wide ≥ 1440, Big screen = fixed 16:9 canvas 1920×1080 (also 3840×2160) |
| App shell (desktop) | Sidebar 240 px (collapses to 48 px icons) · top bar 48 px · page content centred, max-width 1120 px, "full width" toggle for tables |
| Side peek | 480 px right panel (tablet: full-height sheet; mobile: full screen) |
| Grid | 12 columns, 24 px gutters for dashboards; single column for document-like pages |
| Mobile | Bottom-safe actions; alert detail and acknowledge are fully usable on a phone (WhatsApp tap-through) |

## 8. Data-visualisation style

- Minimal: no gradients, no 3-D, light gridlines, direct labels over legends, axis units spelled out.
- Every chart caption states the **denominator and window** ("of 412 mentions · last 24 h · updated 12:04").
- Show confidence intervals as light bands or whiskers where a "change" is claimed; no trend arrows unless the change passed the significance test `[PRD]`.
- Sampled data carries a visible `Sampled` badge on the chart header.
- Small multiples and sparklines preferred over dual-axis charts.
- Tables for exact values beside every chart (toggle "Show data").

## 9. Voice and microcopy

Calm, factual, short. Say what happened, why, and what is possible next. No exclamation marks, no emoji in system messages, no alarmist words outside Tier-1 copy.

| Situation | Copy pattern |
|---|---|
| Not enough data | "Not enough data (N = 18, minimum 30)" |
| Freshness | "of 412 mentions · last updated 12:04" |
| Source problem | "X data delayed since 11:52" (names the source) |
| Reason on a label | One sentence, ≤ 240 chars: "Sarcastic; praises while mocking critics." |
| Alert explanation | Two lines: what happened · why it matters (with reason string) |
| Campaign wording | "Possible coordination" until a lead confirms; never names a party before confirmation |

Vocabulary placeholders come from client settings `[PRD "built to generalise"]`: design with tokens `{party}`, `{ally}`, `{leadership_summary}` and show example resolutions (TVK: "Ally", "CM summary"; business: "Partner", "Leadership summary"). **Never hard-code client names in components.**

Tamil strings in mockups are placeholders for native review.

## 10. Accessibility `[PROPOSED, mandatory]`

- WCAG 2.2 AA minimum. Body text ≥ 4.5:1, large text/UI ≥ 3:1. Verify every tag pair and the dark-mode accent.
- Colour is never the only signal (stance has icon + word; tiers have badge + text; health has icon + text).
- Focus ring visible on every interactive element; full keyboard operation; logical order; `Esc` closes peek/modals.
- Targets ≥ 44×44 px on touch (mobile, CM tablet, big-screen interactive areas).
- Screen-reader text for charts (summary sentence) and for icon-only buttons; live region for new alerts.
- No flash > 1 Hz; respect reduced motion; big screen has a non-animated state.
- Tamil: correct shaping and line-height; test at 200 % zoom.

## 11. Dark mode

Full dark theme required. Default **dark** for the big screen and **light** for working pages (user-switchable, remembered per user). Dark surfaces use `#191919` / `#202020`, not pure black; use the lighter accent `#E57373` for text/icons and keep `#A00000` fills with white text.

## 12. Token implementation (hand-off block)

```css
:root {
  --bg:#FFFFFF; --bg-subtle:#F7F6F3; --bg-hover:rgba(55,53,47,.06); --bg-selected:#FDF2F2;
  --border:rgba(55,53,47,.09); --border-strong:rgba(55,53,47,.16);
  --text:#37352F; --text-2:rgba(55,53,47,.65); --text-3:rgba(55,53,47,.45);
  --accent:#A00000; --accent-hover:#B82A2A; --accent-pressed:#820000; --on-accent:#FFFFFF;
  --focus:#A00000;
  --tag-green-bg:#DBEDDB; --tag-green-fg:#1C3829;
  --tag-rose-bg:#FFE2DD;  --tag-rose-fg:#5D1715;
  --tag-amber-bg:#FDECC8; --tag-amber-fg:#402C1B;
  --tag-purple-bg:#E8DEEE;--tag-purple-fg:#412454;
  --tag-blue-bg:#D3E5EF;  --tag-blue-fg:#183347;
  --tag-gray-bg:#E3E2E0;  --tag-gray-fg:#32302C;
  --radius-sm:3px; --radius-md:6px; --radius-lg:8px;
  --font-ui:Inter,"Noto Sans Tamil",system-ui,sans-serif;
}
/* Dark theme: explicit toggle wins; otherwise follow the OS unless the user chose light */
:root[data-theme="dark"] {
  --bg:#191919; --bg-subtle:#202020; --bg-hover:rgba(255,255,255,.055);
  --bg-selected:rgba(229,115,115,.14); --border:rgba(255,255,255,.094);
  --text:rgba(255,255,255,.90); --text-2:rgba(255,255,255,.60); --text-3:rgba(255,255,255,.40);
  --accent:#E57373; --accent-hover:#EF8A8A; --accent-pressed:#D65F5F; --focus:#E57373;
}
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --bg:#191919; --bg-subtle:#202020; --bg-hover:rgba(255,255,255,.055);
    --bg-selected:rgba(229,115,115,.14); --border:rgba(255,255,255,.094);
    --text:rgba(255,255,255,.90); --text-2:rgba(255,255,255,.60); --text-3:rgba(255,255,255,.40);
    --accent:#E57373; --accent-hover:#EF8A8A; --accent-pressed:#D65F5F; --focus:#E57373;
  }
}
```

Tailwind mapping: `colors.brand` = the §3.1 ramp; semantic names above as `theme.extend.colors`; `fontFamily.sans` = `--font-ui`; radii as above. Component library: Radix UI primitives + Tailwind (the build stack in doc 10).

## 13. Don'ts

- No heavy drop shadows, gradients, glossy buttons, or rainbow dashboards.
- No brand-red large fills on working pages; no red for things that are not critical or negative.
- No bare percentages. No numbers without "of N" and a freshness stamp.
- No post text, private-citizen handles or internal notes on the big screen or in presentation-safe mode `[PRD]`.
- No client-specific words, colours or logos baked into components.
- No autoplay motion on the CM view; no strobing anywhere.
- No truncating Tamil strings to fit an English-sized button.
