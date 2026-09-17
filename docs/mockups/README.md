# UI mockups (concept stage)

Four 1280×800 desktop screens in the Lin Hot Cooling design language (TECHNICAL-CONCEPT §9–§10). The values are illustrative and
based on real readings and capabilities of the ASUS TUF F15 FX506LI reference laptop.

| File | Screen | Interactive |
| --- | --- | --- |
| `Main.dc.html` | **Overview**: thermal-state hero card (cool / warm / hot tweak), Idle / Optimal / Intense switch, animated fan rotor, CPU/dGPU/iGPU gauges, "Why did that happen?" feed, zone sparklines | Category switch; state tweak |
| `Fans.dc.html` | **Fans**: large rotor that spins in proportion to RPM, boost countdown, fan-response chart, "custom curves unavailable" explanation, firmware profile control, capability chips, watchdog status | RPM tweak |
| `Templates.dc.html` | **Templates**: catalog of 9 templates with category badges and heat-vector chips; detail panel showing what each template resolves to on this machine | Category filter; card selection |
| `Hardware.dc.html` | **Hardware**: spec cards, capability matrix (read / change / owner), zone topology, conflict status | Sidebar links |

- **Format:** Design Component HTML (`.dc.html`) with an index in `canvas.json`. They're viewable on the project's private design
  canvas, and are a visual reference for M0–M3, not production code (the app itself is GTK + Cairo).
- **Fonts:** Manrope, with inline stroke icons that preview the planned symbolic set.
- **Motion:** fan rotors spin with CSS animation and honor `prefers-reduced-motion`.
