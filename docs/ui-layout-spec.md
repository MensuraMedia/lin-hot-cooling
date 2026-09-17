# Lin Hot Cooling — UI Layout Specification

| | |
| --- | --- |
| Version | 1.2 (2026-09-17): green/yellow/red heat levels; red and crimson only for red states (no pink) |
| Status | **Approved:** Overview, Fans, Templates. **Provisional:** Hardware (not reviewed yet). |
| Source of truth | `docs/mockups/*.dc.html` (1280 × 800 artboards) + this document. When they disagree, this document wins and the mockup is updated. |
| Related | [TECHNICAL-CONCEPT.md](../TECHNICAL-CONCEPT.md) §9 (UI), §10 (design system), §19 (GTK 4 portability), §20 (modularity); [milestones.md](milestones.md) M0.4–M0.7, M1.6–M1.7, M2.8, M3.7 |
| Units | Logical pixels (GTK scale 1). All sizes scale with the display scale factor. |

This document fixes the layout, dimensions, tokens, component anatomy, states, motion and accessibility of the approved mockups, so the GTK implementation reproduces them exactly and consistently.

---

## 1. Conventions

- **Grid:** 4 px base unit; most spacing is a multiple of 4. The documented exceptions (14 px, 18 px, 22 px) come from the approved mockups and are kept.
- **Naming:** CSS classes use the prefix `hc-`, following `hc-<component>[-<part>]` and state classes `.cool/.warm/.hot`, `.active`, `.selected`, `.unavailable`.
- **Custom widgets:** subclasses of `ui.compat.CanvasArea` (TECHNICAL-CONCEPT §19), named in PascalCase: `FanRotor`, `ArcGauge`, …
- **Colors:** never hard-coded in Python. Every color comes from `config/config_theme.py` tokens (§3), which are exported to CSS as `@define-color`.
- **Numbers:** live values use tabular figures (Pango `font_features="tnum"`). Temperatures render as `54 °C` (narrow no-break space before °C is acceptable), and RPM uses thousands separators from the locale.

---

## 2. Application shell

```
┌──────────────────────── window 1280 × 800 (default) ─────────────────────────┐
│ ┌── sidebar 216 ──┐┌──────────────── main (flex) ───────────────────────────┐ │
│ │ pad 24/14       ││ padding 28 top/bottom · 32 left/right                  │ │
│ │ brand 40 + text ││ header  (≈ 52 tall)                                    │ │
│ │ ── 22 ──        ││ ── gap 20 (Overview, Fans) / 18 (Templates, Hardware)  │ │
│ │ nav item 40 ×10 ││ page body                                              │ │
│ │ (gap 4)         ││                                                        │ │
│ │ flex spacer     ││                                                        │ │
│ │ Settings 40     ││                                                        │ │
│ └─────────────────┘└────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘
```

| Element | Spec |
| --- | --- |
| Window | Default 1280 × 800; minimum 1040 × 680 (§9); background `bg.base` |
| Sidebar | Fixed width 216; padding 24 vertical, 14 horizontal; vertical gradient `bg.sidebar.top → bg.base`; right hairline 1 px `border.sidebar` |
| Content width | 1280 − 216 − 64 = **1000 px** at default size |
| Header | Left: title + subtitle stacked (gap 4). Right: pills/actions (gap 10), vertically centered. `justify: space-between`. |
| Page body | Fills the remaining height; the page scrolls vertically (`Gtk.ScrolledWindow`, horizontal policy NEVER) only when the window is shorter than the layout |

**GTK 3 structure:** `Gtk.ApplicationWindow` › `Gtk.Box(H)` › [`Sidebar` (`Gtk.Box(V)`, `.hc-sidebar`), `Gtk.Stack` (crossfade 200 ms) › per page `Gtk.ScrolledWindow` › `BasePage` (`Gtk.Box(V)`, margins 28/32, spacing 20 or 18)].

---

## 3. Design tokens

### 3.1 Colors

| Token | Value | Use |
| --- | --- | --- |
| `bg.base` | `#080C17` | Window, page background, rotor hub fill |
| `bg.sidebar.top` | `#0D1322` | Sidebar gradient start |
| `surface.1` | `#1E222E` | Cards, list rows, pills |
| `surface.2` | `#2A2E3A` | Gauge/ring tracks, neutral chips, table stripes |
| `surface.3` | `#353743` | Segmented tracks on surfaces, round icon buttons |
| `border.hairline` | `rgba(255,255,255,0.06)` | Card borders |
| `border.row` | `rgba(255,255,255,0.04)` | Table row dividers |
| `border.sidebar` | `rgba(255,255,255,0.05)` | Sidebar edge |
| `border.outline` | `rgba(255,255,255,0.14)` | Secondary buttons |
| `text.primary` | `#F5F7FA` | Titles, values |
| `text.secondary` | `#8B8F99` | Labels, captions, inactive nav |
| `text.chip` | `#B9BDC6` | Text on `surface.2` chips, code-like interface names |
| `text.onAccent` | `#0A0F05` | Text on lime and on the cool gradient |
| `text.onWarm` | `#1A1204` | Text on the warm gradient |
| `text.onIdle` | `#06140E` | Text on the idle (mint) gradient |
| `accent.lime` | `#C1FF14` | Active nav, primary buttons, live dot, duty arcs, links |
| `accent.lime.hover` | `#D9FF6B` | Link hover |
| `accent.lime.tint` | `rgba(193,255,20,0.10)` / `0.12` | Active nav background / optimal badge |
| `accent.lime.deep` | `#293C1C` | Chart fill end |
| `heat.nominal` / `.text` | `#22C55E` / `#4ADE80` | **Heat level Nominal (green)**: marks / text on dark |
| `heat.medium` / `.text` | `#FACC15` / `#FDE047` | **Heat level Medium (yellow)** |
| `heat.intense` / `.text` | `#EF4444` / `#F87171` | **Heat level Intense (red)**; also Critical |
| `heat.intense.fill` / `.deep` | `#DC2626` / `#B91C1C` | Red fills behind white text (emergency banner, hot hero) |
| `state.cool` | `#5DDEA5` | OK/applied status, Idle template category (not a heat level) |
| `state.warm` | `#F5C542` | Advisory status, boost timer (not a heat level) |
| `category.intense` | `#DC143C` (crimson) | Intense template category marks: dots, bars, icons (not text on dark, not a text background) |
| `category.intense.strong` | `#C8102E` | Crimson fills behind white text (performance pill, intense template header) |
| `category.intense.deep` | `#A50E2A` / `#6B0A1A` | Crimson gradient middle/end |
| `category.intense.text` | `#FF5468` | Intense category text and icons on dark surfaces |
| Tints | `rgba(<semantic>, 0.12–0.16)` | Icon tiles, badges, feed avatars, chips |

**Contrast (WCAG 2.2, computed):**

| Foreground on background | Ratio | Allowed for |
| --- | --- | --- |
| `text.primary` on `bg.base` / `surface.1` / `surface.3` | 18.2 / 14.8 / 11.0 | All text |
| `text.secondary` on `bg.base` / `surface.1` | 6.0 / 4.9 | All text |
| `text.secondary` on `surface.3` | 3.6 | **Icons or ≥ 18 px bold text only** |
| `text.chip` on `surface.2` | 7.2 | All text |
| `accent.lime` on `bg.base` / `surface.1` | 16.4 / 13.3 | All text |
| `text.onAccent` on lime / cool gradient stops | 16.2 / 13.7–18.0 | All text |
| `text.onWarm` on warm gradient stops | 7.8–11.4 | All text |
| White on `category.intense.strong` / `#A50E2A` / `#6B0A1A` | 5.9 / 7.8 / 12.5 | All text |
| White on `#DC143C` | 5.0 | Allowed, but prefer `category.intense.strong` for fills |
| `category.intense` `#DC143C` on `surface.1` | 3.2 | **Marks only** (≥ 3:1 for graphics), never text |
| `state.cool` / `state.warm` / `category.intense.text` on `surface.1` | 9.4 / 9.8 / 5.1 | All text |
| `#4ADE80` / `#FDE047` / `#F87171` on `surface.1` | 9.1 / 12.0 / 5.7 | All text (heat levels) |
| `#22C55E` / `#FACC15` / `#EF4444` on `surface.1` | 7.0 / 10.4 / 4.2 | Marks, icons, large text; `#EF4444` is not for small text |
| `#052E16` on `#DCFCE7` / `#86EFAC` / `#22C55E` | 13.6 / 10.6 / 6.5 | All text (nominal fill) |
| `#1A1204` on `#FEF9C3` / `#FDE047` / `#EAB308` | 17.3 / 14.1 / 9.7 | All text (medium fill) |
| White on `#DC2626` / `#B91C1C` / `#7F1D1D` | 4.8 / 6.5 / 10.0 | All text (intense fill) |
| `#585C65` on `bg.base` | 2.9 | **Decorative only**, never text |

**Red-state rule (owner decision, 2026-09-17):** every red state in the app, whether heat (Intense/Critical), errors, emergencies or the Intense category, uses **red (hue ≈ 0°) or crimson (hue ≈ 350°)**. Pink and magenta hues (≈ 320–345°) are not used anywhere. Heat levels use pure red; the Intense template category uses crimson, so the two stay distinguishable.

### 3.2 Gradients

| Token | Definition | Use |
| --- | --- | --- |
| `grad.heat.nominal` | 135°: `#DCFCE7` 0 % → `#86EFAC` 45 % → `#22C55E` 100 %; text `#052E16` | Hero (Nominal), mini window edge, indicator |
| `grad.heat.medium` | 135°: `#FEF9C3` 0 % → `#FDE047` 50 % → `#EAB308` 100 %; text `#1A1204` | Hero (Medium) |
| `grad.heat.intense` | 135°: `#DC2626` 0 % → `#B91C1C` 55 % → `#7F1D1D` 100 %; text white | Hero (Intense / Critical) |
| `grad.hero.cool` | 135°: `#ECFEC2` 0 % → `#D6F2A5` 45 % → `#B8E86A` 100 % | Optimal template header (brand lime; no longer a heat state) |
| `grad.hero.warm` | *(retired in v1.1; replaced by `grad.heat.medium`)* | — |
| `grad.category.intense` | 135°: `#C8102E` 0 % → `#A50E2A` 55 % → `#6B0A1A` 100 % (crimson) | Intense template header (category, not heat) |
| `grad.hero.idle` | 135°: `#D8FBEA` 0 % → `#8FE8C1` 55 % → `#3FBF88` 100 % | Idle template header |
| `grad.sidebar` | 180°: `#0D1322` → `#080C17` | Sidebar |
| `grad.brand` | 135°: `#1E2A12` → `#0B101D`, border lime 35 % | Brand tile |
| `grad.blade` | Diagonal: `#ECFEC2` → `#7FB51A` (tile) / `#6E9E12` (large) | Rotor blades |
| `grad.ring.boost` | Diagonal: `#C1FF14` → `#F5C542` | Fan hero duty ring |
| `grad.gauge` | Bottom-left → top-right: `#22C55E` 0 → `#FACC15` 0.5 → `#EF4444` 1 | Arc gauges, heat meters |
| `grad.area.<color>` | Vertical: color 35 % → 0 % (sparklines); lime 28 % → `#293C1C` 0 % (response chart) | Chart fills |
| `glow.fan` | Radial at 50 % / 42 %: lime 10 % → transparent at 60 % | Fan hero card background |

Gradients are allowed only on these surfaces. Everything else is flat.

### 3.3 Typography

Font family: **Manrope** (as in the mockups), with the fallback chain Ubuntu → Cantarell → sans-serif. Bundle Manrope (OFL) in the GResource. Mono text uses JetBrains Mono → `ui-monospace` → monospace.

| Role | Size / weight | Tracking | Notes |
| --- | --- | --- | --- |
| `display.state` | 52 / 800 | −0.03 em | Hero state word, line-height 1.0 |
| `display.rpm` | 44 / 800 | −0.03 em | Fan hero value, tnum |
| `value.lg` | 30 / 800 | −0.02 em | Fan tile RPM, tnum |
| `title.page` | 26 / 700 | −0.02 em | Page title |
| `value.md` | 24 / 800 | 0 | Gauge center value, tnum |
| `title.hero` | 22 / 800 | −0.01 em | Hero "Active …", template detail name |
| `title.card` | 16 / 700 | 0 | Card titles; brand name 16/800 |
| `body.strong` | 15 / 600–700 | 0 | Hero metrics (85 % opacity), template name |
| `body` | 14 / 500–600 | 0 | Nav, subtitles, feed titles, buttons (700–800) |
| `body.sm` | 13 / 500–700 | 0 | Pills, secondary lines, table control names |
| `caption` | 12 / 500–700 | 0 | Meta, times, chips, legends |
| `overline` | 12 / 600 or 13 / 700 | +0.04 / +0.05 em | Uppercase labels (`hc-cap`) |
| `badge` | 11 / 800 | +0.04 em | Uppercase category badge; chips 11/600 |

GTK 3 CSS has no `text-transform`. Uppercase text is produced in code from translated strings (`str.upper()` on the translated string). If CSS letter-spacing isn't honored, tracking is applied with Pango markup (`letter_spacing`, in 1/1024 pt).

### 3.4 Spacing, radii, borders, icons

| Scale | Values |
| --- | --- |
| Spacing | 1 · 2 · 4 · 6 · 8 · 10 · 12 · 14 · 16 · 18 · 20 · 22 · 24 · 28 · 32 |
| Card padding | 18 (tiles), 20×22 (content cards), 16×20 (strips), 22 (fan hero), 16 (template cards) |
| Grid gaps | 16 (cards), 14 (template grid, spec cards), 4 (segments, nav), 10 (header cluster) |
| Radii | 999 (pills, segments, chips, round buttons) · 24 (hero) · 22 (template detail panel) · 18 (cards) · 14 (spec icon tile) · 12 (nav item, brand and icon tiles) |
| Borders | Cards 1 px `border.hairline`; selected template 1.5 px lime + 4 px ring `rgba(193,255,20,0.08)` |
| Elevation | None (flat). Only the live dot has a glow (`0 0 10px rgba(193,255,20,0.8)`). |
| Icons | 24-grid stroke icons, stroke 1.7 (1.8–2.4 at ≤ 18 px); sizes 14 (status dots) · 16 (pills) · 18 (buttons, bell) · 20 (nav, feed, info) · 22 (tiles) · 24 (brand) |

---

## 4. Components

Each entry gives the anatomy, dimensions, states, tokens and implementation.

### 4.1 Navigation item — `.hc-nav`
- 40 tall, full width, padding 0 × 14, radius 12, gap 12; 20 px icon + label `body` 14/500 `text.secondary`.
- **States:**
  - active: background `accent.lime.tint` 10 %, icon and text lime, `aria-current=page`;
  - hover: background `rgba(255,255,255,0.04)`;
  - focus: focus ring (§8).
- **Items are generated from the page registry, in `order`:** Overview, Fans, Thermals, Power, Profiles, Templates, Automation, Hardware, Benchmarks, History; flexible spacer; Settings.
- **GTK:** `Gtk.Button` (flat, relief NONE), or a `Gtk.ListBox` with single selection. Icon `Gtk.Image` from the GResource symbolic icon.
- **Brand block:** 40 × 40 tile (`grad.brand`, radius 12) with a 24 px lime rotor glyph; name 16/800, tagline "Thermal control" 12 `text.secondary`; bottom padding 22.

### 4.2 Page header — `.hc-header`
- Title `title.page`; subtitle `body` `text.secondary`, stating the device or context (for example "ASUS TUF Gaming F15 · FX506LI · Linux Mint 22.2").
- **Right cluster** (gap 10), in this order: context pill (optional), live pill, power pill, icon button(s) or primary action.

### 4.3 Pills — `.hc-pill`
- 36 tall, padding 0 × 14, radius 999, gap 8, `body.sm` 13/600.
- **Variants:**
  - `.live`: pulse dot 8 px lime with glow, text "Live · 1 s", where the text reflects the sampling interval;
  - `.power`: 16 px plug (AC) or battery icon in lime + text "AC power · charge limit 80 %";
  - `.template.<category>`: tinted background (hot 14 %), text/icon `category.intense.text` (intense), lime (optimal) or `state.cool` (idle).

### 4.4 Icon button — `.hc-icon-button`
- **Visual:** 36 × 36 circle `surface.3`, 18 px icon `text.primary`.
- **Hit area:** at least 44 × 44 (4 px transparent margin), with an accessible name (e.g. "Notifications").

### 4.5 Card — `.hc-card`
- `surface.1`, 1 px `border.hairline`, radius 18; padding per §3.4.
- **Title row:** title (`title.card`) left, meta or link (`caption`/`body.sm`, lime link) right, `space-between`.

### 4.6 Hero state card — `.hc-hero.heat-nominal|.heat-medium|.heat-intense`
- Radius 24, padding 26 × 30. Horizontal: three blocks, `space-between`, gap ≥ 24, vertically centered.
- **Block 1, Thermal state:**
  - `overline` 13/700 at 75 % opacity;
  - `display.state` word Nominal / Medium / Intense (Critical during an emergency);
  - metrics line in `body.strong` at 85 % opacity: "CPU {t} °C · dGPU {t} °C · Fan {rpm} RPM".
- **Block 2, Active:** overline; "{Category} Cooling" in `title.hero`; "Template · {name}" in `body` 14/600 at 85 % opacity.
- **Block 3:** category segmented control (§4.7, hero variant).
- **State colors:**

  | State | Background | Text | Segment track | Active segment | Inactive segment text |
  | --- | --- | --- | --- | --- | --- |
  | Nominal (green) | `grad.heat.nominal` | `#052E16` | `rgba(5,46,22,0.12)` | bg `#052E16`, text `#4ADE80` | `#052E16` |
  | Medium (yellow) | `grad.heat.medium` | `#1A1204` | `rgba(26,18,4,0.12)` | bg `#1A1204`, text `#FDE047` | `#1A1204` |
  | Intense (red) | `grad.heat.intense` | `#FFFFFF` | `rgba(0,0,0,0.22)` | bg `#FFFFFF`, text `#B91C1C` | `#FFFFFF` |
- **State rule:** the heat level comes from the Thermal Agent's assessment (thermal-agent-spec §3.5, §4): Nominal → green; Elevated/Recovering → yellow; Hot/Critical → red. Without the agent, the same rule is computed in-process. The rule below is the fallback:
  - Cool: every zone below warning − 10 °C.
  - Warm: any zone at or above warning − 10 °C.
  - Hot: any zone at or above warning.
  - Hysteresis 3 °C and a minimum dwell of 10 s, so the card doesn't flicker.
- **Transition:** crossfade between gradients over 400 ms (two stacked layers, opacity animated).

### 4.7 Segmented control — `.hc-segmented`
- **Track:** padding 4, gap 4, radius 999.
- **Segment:** height 40, min-width 104, padding 0 × 18, radius 999, `body` 14/700.
- **Variants:**

  | Variant | Track | Active segment | Inactive text | Used for |
  | --- | --- | --- | --- | --- |
  | `.on-hero` | per hero state (§4.6) | per hero state | per hero state | Category switch |
  | `.on-surface` | `surface.3` | lime / `text.onAccent` | `text.primary` | Fan mode (Automatic / Max boost) |
  | `.on-surface.semantic` | `surface.3` | `category.intense.strong` + white (performance), lime (balanced), `state.cool` + `text.onIdle` (quiet) | `text.primary` | Firmware profile (min-width 84) |
  | `.filter` | `surface.1` | lime / `text.onAccent` | `text.chip` | Template filter (height 36, min-width 72) |
- **Behavior:**
  - It's a radio group (`role=group` + `aria-pressed`, GTK: `Gtk.RadioButton` with draw-indicator off, or toggle buttons in a group).
  - Arrow keys move between segments; Enter/Space selects.
  - The active pill slides between positions over 220 ms, `cubic-bezier(0.2, 0.8, 0.2, 1)` (Cairo-drawn indicator, or a CSS transition on background).

### 4.8 Fan tile (Overview) — `FanTile`
- **Card** (§4.5), padding 18, column gap 10.
- **Header row:** overline "System fan" (the fan's label) + mode text 12/600 (`state.cool` "Auto"; `state.warm` "Boost"; `text.secondary` "Firmware").
- **Body row** (gap 16):
  - left: 104 × 104 `RingMeter` + `FanRotor`;
  - right: `value.lg` RPM, then caption "RPM · {duty} % of max".
- **RingMeter:** radius 54 on a 120 viewbox scaled to 104; track `surface.2` stroke 6; duty arc lime stroke 6, round cap, starting at 12 o'clock and running clockwise. Duty = RPM ÷ max RPM, where max RPM comes from the machine profile or the observed peak.
- **FanRotor:** 76 × 76, inset 14 from the ring box.

### 4.9 Fan hero (Fans page) — `FanHero`
- **Card:** 420 wide column, padding 22, column centered, gap 14, background `surface.1` + `glow.fan`.
- **Header row:** overline "System fan · fan1" left; status right (`state.warm` 12/700 "Boost · {n} s left" while leased; otherwise `state.cool` "Automatic").
- **Ring box 250 × 250:**
  - outer ring r 116, stroke 8: track `surface.2`, duty arc `grad.ring.boost`, starting at 12 o'clock;
  - inner decorative ring r 100, 1 px `rgba(255,255,255,0.05)`, dash 2/6;
  - rotor 180 × 180 at inset 35.
- **Value:** `display.rpm` + "RPM" in `body.strong` `text.secondary`, baseline aligned, gap 8.
- **Mode control:** segmented `.on-surface` Automatic | Max boost.
  - Choosing Max boost requests a lease through the helper (polkit if required) and starts the countdown.
  - The segment is **disabled with a tooltip** when the capability is missing.

### 4.10 Fan rotor — `FanRotor` (Cairo)
- **Geometry** (200-unit design box, scaled uniformly):
  - 7 blades at 360/7 = 51.43° steps;
  - blade path `M100 100 C96 62 112 30 138 22 c14 16 2 52 −38 78 Z`;
  - fill `grad.blade`;
  - hub circle r 22 (tile) or r 24 (hero): fill `bg.base`, stroke lime 4;
  - hero adds a lime cap r 8.
- **Angular speed:** `ω = clamp(rpm / rpm_max, 0, 1) × 2.5 rev/s`, which caps the visual speed (no strobing).
  - At 0 RPM with a known reading: blades static at 60 % opacity.
  - At unknown RPM: a dashed hub outline and no rotation.
- **Speed changes:** ease ω with a critically damped spring (ζ = 1, ωₙ = 6 rad/s), so spin-up and spin-down look physical.
- **Motion blur above 60 % speed:** draw 2 ghost copies at −4° and −8° with 25 % and 12 % alpha.
- **Reduced motion:** no rotation. The static rotor plus the RingMeter duty and numeric RPM carry the information.
- **Timing:** driven by `add_tick_callback`; the angle advances by ω·Δt from the frame clock (never a fixed-step timer). Paused when unmapped.

### 4.11 Arc gauge — `ArcGauge`
- **Box:** 104 × 104 (120-unit viewbox).
- **Arc:** radius 48, stroke 10, round caps; 270° sweep starting at 135° (bottom-left), running clockwise.
  - Track: `surface.2`, dash length 226 (= 0.75 × 2π × 48).
  - Value arc: `grad.gauge`, dash = `clamp((v − min) / (max − min), 0, 1) × 226`.
- **Center:** value `value.md` centered at y 36; unit `caption` `text.secondary` at y 66.
- **Default ranges:** CPU 30–100 °C; dGPU 30–90 °C; iGPU 0–100 % busy. The ranges come from zone limits when known.
- **Card layout** (§4.5, padding 18):
  - header: overline name + device meta 12/600 `text.secondary`;
  - body row gap 14: gauge + two `body.sm` lines, for example "Package 9 W" / "EPP balanced", "3 W · P8" / "PRIME on-demand", "Desktop renderer" / "DRM fdinfo".
- **Motion:** the value animates with a spring (ζ = 0.9, ωₙ = 8 rad/s), and the number tweens over 300 ms.
- **Trip points:** when known, 2 px ticks at warning (amber) and critical (magenta) on the outer edge.

### 4.12 "Why" feed row — `.hc-feed-row`
- **Row:** horizontal, gap 14, centered.
  - Avatar: 40 px circle, tinted 12–16 % with the event's semantic color, holding a 20 px icon in that color.
  - Text: title `body` 14/600 over meta `caption` `text.secondary` (gap 2).
  - Time: `caption` tnum, right-aligned.
- **Colors by event type:** game / intense template → `category.intense`; fan boost → lime; cool-down / idle → `state.cool`; heat level changes → the heat-level color (`heat.nominal/medium/intense`); advisory → `state.warm`.
- **Overview:** shows the latest 3 events, with a "View all" link (lime, 13/600) to History.

### 4.13 Zone sparkline row — `.hc-zone-row`
- **Grid:** 110 | 1fr | 64, gap 14, rows centered.
- **Content:** zone name 14/600; `Sparkline` 32 tall; value 16/700 tnum right-aligned "{t} °C".
- **Sparkline:** 10-minute window, at least 12 points (1 Hz data downsampled to the width); 2 px line; `grad.area` fill.
- **Color by zone:** CPU lime, dGPU `state.cool`, chipset `#8B8F99`-tinted amber, storage `text.secondary`. Segments turn `heat.medium` within 10 °C of warning and `heat.intense` above warning.
- **Card:** title "Thermal zones", meta "Last 10 minutes"; shows the 4 hottest zones.

### 4.14 Fan response chart — `ResponseChart`
- **Height:** 180, full card width.
- **Grid:** 3 horizontal lines at 25/50/75 %, 1 px `rgba(255,255,255,0.05)`.
- **Series:**
  - RPM: 2.5 px lime line with lime area fill;
  - CPU °C: 2 px line colored by heat level per segment (`heat.nominal` / `.medium` / `.intense`), on its own axis mapping.
- **Event markers:** 1 px `state.warm` vertical line, dashed 4/4, with a label below in `caption` 12/600 amber ("Boost triggered · CPU 86 °C").
- **Legend:** 12 × 3 swatches (radius 2) + `caption` labels, and a "Last 10 minutes" caption. X-axis labels at the start and end times.

### 4.15 Explanation and profile cards (Fans)
- **"Custom curves unavailable":**
  - title row: 20 px info icon in `text.secondary` + `body.strong` 15/700;
  - body 13/1.5 `text.secondary` explaining why, in plain language;
  - lime link "See capability details" → Hardware.
  - The card is shown only when the capability probe reports no curves.
  - When curves are supported, this slot shows the curve editor entry point.
- **"Firmware fan table":** `body.strong` title; segmented `.on-surface.semantic` Quiet | Balanced | Performance; caption stating who set it ("Set by the gaming template · Fn+F5 still works").

### 4.16 Capability strip and chips — `.hc-chip`
- **Strip:** card padding 16 × 20, horizontal wrap, gap 12. Content in order:
  1. overline "Capabilities";
  2. chips;
  3. flexible spacer;
  4. watchdog status: 16 px lime shield + `caption` "Watchdog armed · returns to automatic if the app stops".
- **Chip:** 30 tall, padding 0 × 12, radius 999, `caption` 12/600.
  - `.available`: `state.cool` text on 12 % `state.cool` tint.
  - `.unavailable`: `text.secondary` on `surface.2`.
  - Chips state facts ("Reads fan1_input", "Writes pwm1_enable · auto / full", "No curve points"), never vendor marketing.

### 4.17 Template card — `.hc-template-card`
- **Box:** a `Gtk.FlowBoxChild` that acts as a button, 200 tall in a 3-column grid; padding 16; column gap 10; radius 18; `surface.1`.
- **Rows, in order:**
  1. Header (`space-between`): 40 × 40 icon tile (radius 12, category tint, 22 px category-colored icon) + category badge (24 tall, padding 0 × 10, `badge`, category color on tint).
  2. Name: `body.strong` 15/700.
  3. Description: `caption` 12, line-height 1.45, `text.secondary`, at most 3 lines, ellipsized.
  4. Heat-vector chips: 22 tall, padding 0 × 8, 11/600 `text.chip` on `surface.2`, wrapping, gap 6.
- **Category colors:**

  | Category | Text / icon | Tint |
  | --- | --- | --- |
  | Intense | `#FF5468` (crimson) | `rgba(220,20,60,0.16)` |
  | Optimal | lime | lime 12 % |
  | Idle | `state.cool` | cool 14 % |
- **States:**
  - default: border hairline;
  - hover: border 12 % white;
  - **selected:** 1.5 px lime border + 4 px `rgba(193,255,20,0.08)` outer ring;
  - custom (user) templates add a small "Custom" chip.
- **Activation:** click or Enter selects and updates the detail panel. Double-click or Enter a second time applies (with confirmation for intense templates on battery).

### 4.18 Buttons — `.hc-button`
- **Primary:** 44 tall, padding 0 × 20, radius 999, lime background, `text.onAccent` 14/800; optional 18 px icon (stroke 2.2) with gap 8.
- **Secondary:** 44 tall, padding 0 × 18, transparent background, 1 px `border.outline`, `text.primary` 14/700.
- **Destructive or emergency actions:** `heat.intense.fill` (`#DC2626`) background with white text.
- **Disabled:** 40 % opacity, not focusable, with a tooltip saying why.

### 4.19 Template detail panel — `.hc-template-detail`
- **Box:** fixed width 340, radius 22, `surface.1` with hairline, column layout, overflow clipped.
- **Header** (padding 20 × 22, gap 6): category gradient (`grad.hero.hot` / `.cool` / `.idle`) with its text color.
  - overline "{Category} cooling" at 80 % opacity;
  - name `title.hero`;
  - trigger summary `body.sm` 13/600 at 85 % opacity ("When a game starts · on AC power").
- **Body** (padding 18 × 22, gap 10, grows): overline "Resolved on this machine", then resolution rows.
  - Each row: 22 px status dot + label 13/700 + note 12 `text.secondary`.

    | Status | Dot | Glyph |
    | --- | --- | --- |
    | Applied | `state.cool` on 16 % tint | check |
    | Not supported / not used | `text.secondary` on `surface.2` | dash |
    | Advisory / partial | `state.warm` on 16 % tint | "!" |

  - The rows come straight from the resolver's output (TECHNICAL-CONCEPT §7.3), including the "not applied: reason" notes.
- **Footer** (padding 16 × 22 × 20, gap 10): primary "Apply now" (fills the width) + secondary "Clone".

### 4.20 Hardware components (provisional)
- **Spec card:** padding 16 × 18, gap 14; 44 px icon tile (radius 14, lime 10 %, 22 px lime icon); overline label, value 14/700, sub-line 12 `text.secondary`. Four across (gap 14): Model, Processor, Graphics, System.
- **Capability table:**
  - columns `1.3fr | 1.6fr | 70 | 90 | 1.2fr` (Control, Interface, Read, Change, Owner), gap 12;
  - header row with overline labels and a bottom hairline;
  - rows padding 10 × 22 with a `border.row` divider;
  - interface in mono 12 `text.chip`;
  - Read/Change values 12/700 colored `state.cool` (yes), `state.warm` (partial or helper), `heat.intense.text` `#F87171` (no), `text.secondary` (not applicable). Values are always words, never color alone.
- **Zone topology:** diagram 260 × 210. Zone pills on the left (70 × 24, radius 12; active zones `surface.2`, passive zones outlined 12 % white with secondary text); fans as circles r 30 with a lime stroke; links as 1.5 px lime 35 % curves (active) or dashed 12 % white (monitored only).
- **Conflict card:** shield icon (`state.cool`, or `state.warm` when there are conflicts) + title + one-sentence explanation.

---

## 5. Page layouts

### 5.1 Overview (approved)

```
main 1000 wide
├─ header ─────────────────────────────────────────── Live · AC · 🔔
├─ gap 20
├─ Hero (§4.6) ─ full width, ≈ 170 tall
├─ gap 20
├─ Grid 4 × 1 (equal columns, gap 16) ─ tiles ≈ 176 tall
│   FanTile │ ArcGauge CPU │ ArcGauge dGPU │ ArcGauge iGPU
├─ gap 20
└─ Grid 2 × 1 (equal, gap 16) ─ fills remaining height
    Why feed card (§4.12) │ Thermal zones card (§4.13)
```

| Area | Data binding | Empty / unavailable |
| --- | --- | --- |
| Hero | `state.thermal`, zone temps, fan RPM, `policy.category`, `policy.template` | No fan sensor → metrics omit "Fan"; helper offline → segmented disabled + amber banner (§6) |
| Tiles | `fans[0]`, `zones.cpu`, `zones.gpu.discrete`, `zones.gpu.integrated` | Missing dGPU → tile hidden and the grid becomes 3 columns; more fans → a "+N fans" link in the fan tile |
| Why feed | `history.actions[-3:]` | "No automatic changes yet" (`caption`) |
| Zones | Top 4 zones by temperature relative to their limit | Fewer zones → fewer rows |

**Interactions:** the category switch changes the category at once (applied through the helper; the hero shows a spinner overlay on the segment until the change is confirmed). The tiles link to Fans and Thermals.

### 5.2 Fans (approved)

```
main 1000 wide
├─ header ────────────────────────── [template pill] Live
├─ gap 20
├─ Grid 2 columns: 420 │ 1fr (gap 16)
│   FanHero (§4.9)     │ Column (gap 16):
│                      │   ResponseChart card (§4.14)
│                      │   Grid 2 × 1 (gap 16): Curves-unavailable card │ Firmware-table card (§4.15)
├─ gap 20
└─ Capability strip (§4.16), full width
```

- **Several fans:** the left column becomes a vertical list of `FanHero`s (the ring shrinks to 180 px when there are 2 or more), or a fan selector (segmented control of fan names) above a single hero. The chart shows the selected fan.
- **Machines with curves:** the curves-unavailable card becomes "Fan curve", with a mini curve preview and an "Edit curve" secondary button that opens the editor (M5).

### 5.3 Templates (approved)

```
main 1000 wide
├─ header ───────────── [All | Intense | Optimal | Idle]  [+ New template]
├─ gap 18
└─ Row (gap 16), fills height
    Grid 3 columns, rows 200, gap 14 (scrolls) │ Detail panel 340 (§4.19)
```

- **Order:** shipped templates in catalog order (Intense → Optimal → Idle, then Cool-down), followed by user templates.
- **Filtering:** the filter hides non-matching cards; the current selection stays in the detail panel even when filtered out.
- **"New template"** opens the editor (form + YAML tabs) prefilled from the selected template.
- **Grid** uses `Gtk.FlowBox` (`max-children-per-line` 3, `min-children-per-line` 2, homogeneous, selection mode SINGLE).

### 5.4 Hardware (provisional; needs review)

```
header ─────────────────────────────── [Re-scan hardware]
gap 18
Grid 4 × 1 spec cards (gap 14)
gap 18
Grid: capability table (1fr) │ column 300: Zone topology card, Conflict card
```

The capability table rows are generated from the capability manifest (TECHNICAL-CONCEPT §4.2). They're never hard-coded per vendor.

### 5.5 Thermal Agent card

The Cooling Template Card (standard, compact and full variants) is specified in [thermal-agent-spec.md §7](thermal-agent-spec.md#7-the-cooling-template-card--coolingtemplatecard), using this document's tokens and components.

### 5.6 Pages not yet mocked

Thermals, Power, Profiles, Automation, Benchmarks, History, Settings and About use the same shell, header, cards and tokens:
- Thermals reuses the zone rows at full width, with trip-point ticks.
- Profiles reuses three category cards plus template-detail-style resolution lists.
- Automation uses `Gtk.ListBox` rows styled like feed rows.
- Mockups for these pages come before their milestone (M1–M4).

---

## 6. Global states and feedback

| State | Presentation |
| --- | --- |
| Loading (first probe) | Cards show `surface.2` placeholder blocks of the final size. No shimmer when reduced motion is on; otherwise a 1.2 s opacity pulse from 60 to 100 %. |
| Capability missing | Control disabled (§4.18) + tooltip + an entry in the Hardware capability table |
| Helper offline or not authorized | Amber banner under the header: `state.warm` 12 % tint, 44 tall, radius 12, text "Cooling control unavailable — read-only mode" + "Retry" secondary button |
| Emergency (critical temperature) | Hero forced to Intense (red); banner `heat.intense.deep` (`#B91C1C`) background, white text, "Emergency cooling active" + reason; category switch locked until zones fall below warning |
| Conflict detected | Amber banner naming the other manager, with the actions "Let Hot Cooling manage" and "Pause automation" |
| Unsaved template edits | Secondary footer bar with "Discard" / "Save" |
| Notifications | Desktop notification only for emergency, helper failure and conflicts; everything else goes to the "Why" feed |

---

## 7. Motion

| Element | Motion | Duration / curve | Reduced motion |
| --- | --- | --- | --- |
| Fan rotor | Continuous rotation, speed from RPM (§4.10) | Frame clock; spring on speed changes | Static |
| Live dot | Opacity 1 → 0.35 → 1 | 1 s ease-in-out, repeating (period = sampling interval) | Static at full opacity |
| Arc gauge / ring meter | Arc length follows value | Spring (ζ 0.9, ωₙ 8) | Jump |
| Numeric values | Count between values | 300 ms ease-out | Jump |
| Hero state change | Gradient crossfade | 400 ms ease-in-out | Instant |
| Segmented selection | Pill slides | 220 ms cubic-bezier(0.2, 0.8, 0.2, 1) | Instant |
| Page switch | `Gtk.Stack` crossfade | 200 ms | None |
| Card hover | Border color | 120 ms | Instant |
| Boost countdown | Ring arc drains | Linear over the lease time | Text countdown only |

**Rules:**
- Reduced motion is on when `gtk-enable-animations` is false **or** Settings → Animations is set to Reduced/Off.
- All animations stop when their widget is unmapped or the window is minimized.
- Total animation cost must stay under 2 % CPU on the reference laptop.

---

## 8. Accessibility

- **Contrast:** only the pairs allowed in §3.1. Status is never shown by color alone; every colored status has a word or glyph (Yes/No/Helper, check/dash/!).
- **Focus:**
  - Every interactive element is keyboard focusable, in visual order: sidebar → header actions → page content, left to right, top to bottom.
  - Focus ring: 2 px `accent.lime` outline with 2 px offset (`#0A0F05` on lime or cool-gradient surfaces).
- **Targets:** at least 44 × 44 hit areas (segment 40 tall + track padding; icon buttons get extra margin).
- **Names and roles:** icon-only buttons have accessible names. The segmented controls expose radio semantics.
  - Gauges expose "CPU temperature 54 degrees Celsius, range 30 to 100".
  - Rotors expose "System fan 4850 RPM, boost, 87 percent".
  - GTK 3 uses `Atk` (`get_accessible().set_name()` / `set_role`); GTK 4 uses the `Gtk.Accessible` properties.
- **Text scaling:** the layout tolerates a 125 % font scale and 30 % longer translations. Labels ellipsize; cards grow vertically; nothing clips.
- **Charts:** each chart has a text summary (last value, min/max, events) available to screen readers, and on hover as a tooltip.

---

## 9. Responsive rules

| Width | Changes |
| --- | --- |
| ≥ 1280 | As specified |
| 1180–1279 | Content width shrinks; Templates grid stays at 3 columns (cards narrower) |
| 1100–1179 | Overview tiles → 2 × 2 grid; Templates grid → 2 columns; Fans right column stacks its two small cards vertically |
| 1040–1099 | Sidebar collapses to a 72 px icon rail (labels become tooltips; brand text hidden) |
| < 1040 | Not supported (the minimum window size) |

- **Height:** below 800, pages scroll. The hero, tiles and header never shrink below their natural height.
- **HiDPI:** all Cairo drawing uses logical units and the widget scale factor, and SVG icons render at the device scale.

---

## 10. GTK 3 implementation mapping

| Spec element | GTK 3 implementation | Notes |
| --- | --- | --- |
| Shell | `Gtk.ApplicationWindow`, `Gtk.Box`, `Gtk.Stack` | Stack transition `CROSSFADE`, 200 ms |
| Sidebar items | `Gtk.ListBox` rows or flat `Gtk.Button`s generated from the page registry | Active state via the `.active` class |
| Cards, pills, chips, buttons | `Gtk.Box` / `Gtk.Button` + CSS classes (§3) | GTK 3 CSS supports `border-radius`, `background-image` gradients, `box-shadow` and `transition`; spacing uses `Gtk.Box` spacing (no CSS `gap`) |
| Grids | `Gtk.Grid` (column-homogeneous) or `Gtk.FlowBox` (template cards) | No CSS grid in GTK |
| Hero gradient crossfade | `Gtk.Overlay` with two `Gtk.Box` layers + opacity animation (tick callback) | CSS classes `.heat-nominal/.heat-medium/.heat-intense` per layer |
| Segmented control | `Gtk.Box` of `Gtk.RadioButton`s (`draw_indicator=False`) + `SegmentIndicator` (`CanvasArea`) behind them | Slide animation drawn in Cairo |
| FanRotor, RingMeter, ArcGauge, Sparkline, ResponseChart, TopologyDiagram | `ui.compat.CanvasArea` subclasses drawing with pycairo (`cairo.LinearGradient`, `RadialGradient`, `arc`, `curve_to`) | Colors from theme tokens; `set_antialias(ANTIALIAS_BEST)` |
| Uppercase overlines | Code-side `upper()` of translated strings | GTK CSS has no `text-transform` |
| Tabular numbers | Pango markup `<span font_features="tnum">` | Also for tables |
| Icons | Symbolic SVGs in the GResource, `Gtk.Image.new_from_icon_name` with the `hc-*-symbolic` names | Recolored via CSS `color` |
| Tooltips / disabled reasons | `set_tooltip_text` | Required on every disabled control |

**CSS skeleton** (`resources/css/lin-hot-cooling.css`; generated values from `config_theme.py`):

```css
@define-color hc_bg #080C17;
@define-color hc_surface1 #1E222E;
@define-color hc_surface2 #2A2E3A;
@define-color hc_surface3 #353743;
@define-color hc_text #F5F7FA;
@define-color hc_text2 #8B8F99;
@define-color hc_lime #C1FF14;
@define-color hc_cool #5DDEA5;
@define-color hc_heat_nominal #22C55E;
@define-color hc_heat_medium #FACC15;
@define-color hc_heat_intense #EF4444;
@define-color hc_intense_category #C8102E;

.hc-sidebar { background-image: linear-gradient(180deg, #0D1322, @hc_bg); border-right: 1px solid alpha(white, 0.05); padding: 24px 14px; }
.hc-nav { min-height: 40px; padding: 0 14px; border-radius: 12px; color: @hc_text2; font-weight: 500; }
.hc-nav.active { background-color: alpha(@hc_lime, 0.10); color: @hc_lime; }
.hc-card { background-color: @hc_surface1; border: 1px solid alpha(white, 0.06); border-radius: 18px; }
.hc-hero { border-radius: 24px; padding: 26px 30px; }
.hc-hero.heat-nominal { background-image: linear-gradient(135deg, #DCFCE7 0%, #86EFAC 45%, #22C55E 100%); color: #052E16; }
.hc-hero.heat-medium  { background-image: linear-gradient(135deg, #FEF9C3 0%, #FDE047 50%, #EAB308 100%); color: #1A1204; }
.hc-hero.heat-intense { background-image: linear-gradient(135deg, #DC2626 0%, #B91C1C 55%, #7F1D1D 100%); color: #FFFFFF; }
.heat-nominal-text { color: #4ADE80; }  .heat-medium-text { color: #FDE047; }  .heat-intense-text { color: #F87171; }
.hc-segmented { border-radius: 999px; padding: 4px; }
.hc-segmented button { min-height: 40px; min-width: 104px; padding: 0 18px; border-radius: 999px; font-weight: 700; }
.hc-button.primary { min-height: 44px; padding: 0 20px; border-radius: 999px; background-color: @hc_lime; color: #0A0F05; font-weight: 800; }
.hc-chip { min-height: 30px; padding: 0 12px; border-radius: 999px; font-size: 12px; font-weight: 600; }
.hc-chip.available { color: @hc_cool; background-color: alpha(@hc_cool, 0.12); }
.hc-chip.unavailable { color: @hc_text2; background-color: @hc_surface2; }
.hc-template-card.selected { border: 1.5px solid @hc_lime; box-shadow: 0 0 0 4px alpha(@hc_lime, 0.08); }
*:focus { outline: 2px solid @hc_lime; outline-offset: 2px; }
```

**GTK 4 notes (M7):**
- `Gtk.RadioButton` becomes `Gtk.ToggleButton` groups, and `Gtk.FlowBox` / `Gtk.Grid` stay.
- CSS mostly carries over, and `outline` is supported natively.
- Drawing moves to `Gtk.DrawingArea.set_draw_func` through the existing `CanvasArea` shim.
- Libadwaita, if adopted, must not override these tokens (use `Adw.StyleManager` with the dark scheme and the custom CSS loaded at application priority).

---

## 11. Traceability (mockup → spec → milestone)

| Mockup element | Spec | Implemented in |
| --- | --- | --- |
| Sidebar, header, pills | §2, §4.1–4.4 | M0.3–M0.4 |
| Hero + category switch | §4.6–4.7 | M1.6 (display), M2.8 (switching) |
| Fan tile / fan hero / rotor | §4.8–4.10 | M0.7 (demo rotor), M1.6, M2.8 (boost) |
| Arc gauges | §4.11 | M1.6 |
| Why feed | §4.12 | M2.8 (v1), M3.7 (full) |
| Zone sparklines / response chart | §4.13–4.14 | M1.6–M1.7, M4.2 |
| Capability strip / chips / table | §4.16, §4.20 | M1.5, M1.7 |
| Template grid / card / detail | §4.17–4.19, §5.3 | M3.7 |
| Global states | §6 | M1 (loading/unavailable), M2 (helper, emergency), M5 (conflicts) |

---

## 12. Change log

| Date | Change |
| --- | --- |
| 2026-09-17 | v1.0: spec written from the approved Overview, Fans and Templates mockups; Hardware recorded as provisional. |
| 2026-09-17 | v1.2 (owner decision): red states are red or crimson, never pink. The Intense category moved from magenta (`#F14D8A`, `#C22061`, `#FF6FA3`, `#FF8DB6`) to crimson (`#DC143C`, `#C8102E → #A50E2A → #6B0A1A`, `#FF5468`). All mockups were recolored, including the green/yellow/red heat levels. |
| 2026-09-17 | v1.1 (owner decision D3): heat detection uses **green / yellow / red** (`heat.nominal/medium/intense`) for hero cards, gauges, meters, sparklines and the indicator. The former cool/warm/hot hero gradients are retired; magenta is now only the Intense *template category* color. Agent surfaces are heat-first and collapsed by default (thermal-agent-spec §7.0). Mockups are to be updated in A0.3–A0.5. |
| 2026-09-17 | Accessibility fix (applied to the mockups too): the hot gradient changed from `#FFC4DA → #F14D8A → #A01743` to `#C22061 → #A01743 → #7A1235` (white text ≥ 5.7:1), and the "Performance" segment fill changed from `#F14D8A` (3.4:1) to `#C22061`. `#F14D8A` is kept for non-text marks only. |
