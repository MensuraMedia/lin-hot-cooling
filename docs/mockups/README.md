# UI mockups (concept stage)

> **Colors (2026-09-17):** heat levels are green / yellow / red, and the Intense template category is crimson (no pink anywhere), as specified in ui-layout-spec v1.2.

Four 1280×800 desktop screens in the Lin Hot Cooling design language (TECHNICAL-CONCEPT §9–§10). The values are illustrative and
based on real readings and capabilities of the ASUS TUF F15 FX506LI reference laptop.

| File | Screen | Interactive |
| --- | --- | --- |
| `Main.dc.html` | **Overview**: heat-level hero card (nominal / medium / intense tweak), Idle / Optimal / Intense switch, animated fan rotor, CPU/dGPU/iGPU gauges, "Why did that happen?" feed, zone sparklines | Category switch; state tweak |
| `Fans.dc.html` | **Fans**: large rotor that spins in proportion to RPM, boost countdown, fan-response chart, "custom curves unavailable" explanation, firmware profile control, capability chips, watchdog status | RPM tweak |
| `Templates.dc.html` | **Templates**: catalog of 9 templates with category badges and heat-vector chips; detail panel showing what each template resolves to on this machine | Category filter; card selection |
| `AgentOverview.dc.html` | **Overview · Thermal Agent**: intense (red) state with the Cooling Template Card (collapsed by default; expand for template and mode) in place of the zones card; "Why" feed shows the agent's reasoning | Card: Apply, mode pill, Not now |
| `AgentProfiles.dc.html` | **Profiles · Agent ranking**: category cards, full template ranking (score bars, relief, Apply), selected-template plan, cooling history | Agent mode, row details, Apply |
| `AgentStates.dc.html` | **Thermal Agent states**: card in Hot/Suggest, Elevated/Auto, Nominal/Observe, Critical and Off; compact variant; desktop notification; mode menu; template popover | Each card instance |
| `AgentViews.dc.html` | **Collapsed and expanded views**: the Overview card collapsed (default) in green / yellow / red, expanded (Intense·Suggest, Medium·Auto), critical collapsed with its banner, and the compact bar collapsed and expanded | Expanders, Apply, mode switches |
| `CompactBar.dc.html` | Reusable compact bar component (`preset`, `view` tweaks) | Yes |
| `Tray.dc.html` | **Panel indicator**: temperature-gauge tray icon (anatomy, dark/light panels, tooltip), mini window collapsed and expanded, right-click menu | Level tweak; mode and Apply in the expanded window |
| `AgentCard.dc.html` | Reusable Cooling Template Card component (imported by the agent artboards; `preset` and `view` tweaks, collapsed by default) | Yes |
| `Hardware.dc.html` | **Hardware**: spec cards, capability matrix (read / change / owner), zone topology, conflict status | Sidebar links |

- **Format:** Design Component HTML (`.dc.html`) with an index in `canvas.json`. They're viewable on the project's private design
  canvas, and are a visual reference for M0–M3, not production code (the app itself is GTK + Cairo).
- **Fonts:** Manrope, with inline stroke icons that preview the planned symbolic set.
- **Motion:** fan rotors spin with CSS animation and honor `prefers-reduced-motion`.
