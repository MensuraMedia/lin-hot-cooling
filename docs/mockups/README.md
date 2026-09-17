# UI mockups (concept stage)

> **Pending update (A0.3–A0.5, see docs/milestones-thermal-agent.md):** the heat colors in these mockups still use the earlier lime/amber/magenta states. The decision of 2026-09-17 changes heat levels to **green / yellow / red** and makes agent surfaces heat-first and collapsed by default. The specs (ui-layout-spec v1.1, thermal-agent-spec v1.2) take precedence.

Four 1280×800 desktop screens in the Lin Hot Cooling design language (TECHNICAL-CONCEPT §9–§10). The values are illustrative and
based on real readings and capabilities of the ASUS TUF F15 FX506LI reference laptop.

| File | Screen | Interactive |
| --- | --- | --- |
| `Main.dc.html` | **Overview**: thermal-state hero card (cool / warm / hot tweak), Idle / Optimal / Intense switch, animated fan rotor, CPU/dGPU/iGPU gauges, "Why did that happen?" feed, zone sparklines | Category switch; state tweak |
| `Fans.dc.html` | **Fans**: large rotor that spins in proportion to RPM, boost countdown, fan-response chart, "custom curves unavailable" explanation, firmware profile control, capability chips, watchdog status | RPM tweak |
| `Templates.dc.html` | **Templates**: catalog of 9 templates with category badges and heat-vector chips; detail panel showing what each template resolves to on this machine | Category filter; card selection |
| `AgentOverview.dc.html` | **Overview · Thermal Agent**: hot state with the Cooling Template Card in place of the zones card; "Why" feed shows the agent's reasoning | Card: Apply, mode pill, Not now |
| `AgentProfiles.dc.html` | **Profiles · Agent ranking**: category cards, full template ranking (score bars, relief, Apply), selected-template plan, cooling history | Agent mode, row details, Apply |
| `AgentStates.dc.html` | **Thermal Agent states**: card in Hot/Suggest, Elevated/Auto, Nominal/Observe, Critical and Off; compact variant; desktop notification; mode menu; template popover | Each card instance |
| `AgentCard.dc.html` | Reusable Cooling Template Card component (imported by the agent artboards; `preset` tweak) | Yes |
| `Hardware.dc.html` | **Hardware**: spec cards, capability matrix (read / change / owner), zone topology, conflict status | Sidebar links |

- **Format:** Design Component HTML (`.dc.html`) with an index in `canvas.json`. They're viewable on the project's private design
  canvas, and are a visual reference for M0–M3, not production code (the app itself is GTK + Cairo).
- **Fonts:** Manrope, with inline stroke icons that preview the planned symbolic set.
- **Motion:** fan rotors spin with CSS animation and honor `prefers-reduced-motion`.
