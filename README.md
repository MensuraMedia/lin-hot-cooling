# Lin Hot Cooling

**Premium, modular thermal management for Debian-based Linux.**
Lin Hot Cooling ("Hot Cooling" in the app) works out what your machine can do about heat, shows it in a calm dark
interface, and cools the machine the right way for what you're doing, from silent office work to hours of gaming.

> **Status: design phase.** The technical concept and milestone plan are written. Application code starts with milestone
> **M0** ([docs/milestones.md](docs/milestones.md)). The features below describe the target of v1.0.

| | |
| --- | --- |
| Platforms | Debian 12+, Ubuntu 24.04+, Linux Mint 22+ (X11 first, Wayland supported) |
| Stack | Python 3.12 · GTK 3 (PyGObject), written to be portable to GTK 4 · Cairo · D-Bus · polkit · systemd |
| Based on | [gtk-python-dashboard-starter](https://github.com/mikesdatawork/gtk-python-dashboard-starter) |
| Standards | [MensuraMedia/universal-instruction-set](https://github.com/MensuraMedia/universal-instruction-set) v2026.04 |
| License | Free to use, modify and distribute. **Commercial use requires prior written permission.** See [LICENSE](LICENSE) and [NOTICE](NOTICE). |

---

## Contents
1. [Purpose](#1-purpose)
2. [What it does](#2-what-it-does)
3. [Cooling categories, templates and heat vectors](#3-cooling-categories-templates-and-heat-vectors)
4. [Using the app](#4-using-the-app)
5. [The GTK Python stack](#5-the-gtk-python-stack)
6. [Architecture and modularity](#6-architecture-and-modularity)
7. [Design language](#7-design-language)
8. [Safety and security](#8-safety-and-security)
9. [Installation](#9-installation)
10. [Development](#10-development)
11. [Project layout](#11-project-layout)
12. [Roadmap](#12-roadmap)
13. [Documentation](#13-documentation)
14. [License and credits](#14-license-and-credits)

---

## 1. Purpose

On Linux, heat and fan management is spread across many places:
- kernel drivers (hwmon, thermal zones, cpufreq, powercap);
- vendor platform drivers (asus-wmi, thinkpad_acpi, dell-smm);
- services (thermald, power-profiles-daemon, TLP);
- GPU stacks (NVIDIA, amdgpu, i915);
- firmware.

Every machine exposes a different subset, and most tools either show numbers or write fan speeds without knowing whether that's safe.

Lin Hot Cooling brings this together:
- **Discover:** find every sensor, fan, profile and power control on *this* machine, what can be changed, and which service owns it.
- **Understand:** combine it all into one model of thermal zones (CPU, GPU, chipset, storage, battery) with live history.
- **Act:** apply cooling that fits the workload through three simple categories and a catalog of templates, only using controls the hardware safely supports.
- **Explain:** every automatic change says what happened and why ("Game started → High-Intensity Gaming → fan boost at 86 °C").

It never exceeds firmware limits, never writes raw embedded-controller registers, and hands control back to firmware when anything goes wrong.

## 2. What it does

| Area | Functionality |
| --- | --- |
| **Hardware discovery** | Model, BIOS, CPU, GPUs and drivers, OS/kernel; a capability matrix showing what's readable and writable, with reasons; conflict detection between power managers |
| **Thermal monitoring** | All temperature sensors grouped into zones, trip and critical points, sparklines, throttle-event counters, NVMe temperatures |
| **Fans** | Live RPM with animated rotors; available modes (auto, full-speed boost, firmware or software curves where supported); fan-response charts |
| **Power** | AC/battery state, drain in watts, charge limit, CPU package watts (through the helper), GPU watts and state, EPP/turbo, PRIME (hybrid graphics) advice |
| **Cooling control** | One-tap **Idle / Optimal / Intense** categories; workload **templates**; timed **Max fan** boost; emergency protection |
| **Automation** | Rules triggered by AC/battery changes, game start/stop (GameMode, Steam), process types, temperature thresholds, sustained load, lid state and schedules, with anti-flapping delays |
| **Templates** | A shipped catalog plus your own clones, edited in a form or as YAML and validated against a schema |
| **Insight** | History (1 h / 24 h / 7 d), an action log ("Why"), guided benchmarks comparing categories (FPS, temperatures, RPM, throttling, watts) |
| **Background agent** | Optional user service that keeps rules running when the window is closed |
| **Adaptivity** | Features appear or hide according to the hardware. A laptop with only auto/full fan modes gets profile control and boosts; a desktop with PWM headers gets full curves. |

### Hardware support model

| Your machine offers | Lin Hot Cooling provides |
| --- | --- |
| Fan curves in firmware (`pwm*_auto_point*`) | Curve editor per template |
| Manual PWM (desktop boards, some laptops) | Software curves with a watchdog |
| Only auto/full fan modes (many laptops) | Profile-based control + timed full-speed boost |
| `platform_profile` / power-profiles-daemon | Category → profile mapping |
| Writable RAPL / GPU power limits | Per-template power caps (M5) |
| No fan sensor | Temperature-first view; fans marked as firmware-managed |

## 3. Cooling categories, templates and heat vectors

**Categories** are the three modes you choose between:

| Category | Feels like | Typical effect |
| --- | --- | --- |
| **Idle Cooling** | Silent, efficient | Quiet / power-saver profile, efficient CPU preference, no boosts |
| **Optimal Cooling** | Everyday balance | Balanced profile, automatic fans |
| **Intense Cooling** | Maximum heat removal | Performance profile, early fan boost, protective limits for long runs |

**Templates** refine a category for a workload:

| Template | Category |
| --- | --- |
| High-Intensity Gaming | Intense |
| Esports / Competitive | Intense |
| Long-Running Operations (builds, renders, training) | Intense → Optimal |
| Development / Compile bursts | Optimal |
| General Operations | Optimal |
| Media & Streaming | Optimal |
| Quiet Office / Night | Idle |
| On Battery | Idle |
| Cool-down (after heavy load) | Intense → Idle |

**Heat vectors** are the reusable building blocks behind templates. Each describes how a workload turns into heat:
- short CPU bursts;
- sustained CPU load;
- sustained GPU load;
- an idle-but-awake discrete GPU;
- mixed gaming load;
- storage I/O;
- charging under load;
- chassis and room heat.

Each vector defines how to detect it, the safe temperature envelope, and which controls help most. Templates list their vectors, so new templates reuse known thermal behavior.

Default automation (editable):
- **AC** → performance; **battery** → quiet.
- **A game starts on AC** → High-Intensity Gaming.
- **Heavy load ends** → Cool-down → back to the power-source default.

## 4. Using the app

**Pages:**

| Page | What you do there |
| --- | --- |
| **Overview** | See the thermal state (cool/warm/hot hero card), active category and template, power source, animated fans, CPU/GPU gauges, recent "Why" events |
| **Fans** | Watch fan speeds, trigger a timed Max-fan boost, edit curves (where supported) |
| **Thermals** | Inspect every zone and sensor, trip points and throttling |
| **Power** | Source, drain, charge limit, CPU/GPU watts, hybrid-graphics advice |
| **Profiles** | Switch between Idle / Optimal / Intense and see exactly what each changes on your machine |
| **Templates** | Browse the catalog, see resolved settings and "not available here" notes, clone and edit |
| **Automation** | Create and prioritize rules, test triggers, review the event history |
| **Hardware** | Specs, zone topology, capability matrix, detected conflicts |
| **Benchmarks** | Run guided idle/load/cool-down comparisons between categories or templates |
| **History** | Charts and the action log |
| **Settings** | Units, sampling rate, animations (full / reduced / off), accent, background agent, helper status, diagnostics export |

**Typical workflows:**
- **"Keep it quiet while I work"** → Profiles → Idle, or let the battery rule do it.
- **"I'm about to game"** → nothing to do: GameMode/Steam starts the gaming template on AC.
- **"Render all night"** → Templates → Long-Running Operations → Apply (stable throughput, protected temperatures).
- **"Is my cooling any good?"** → Benchmarks → compare Optimal vs Intense.

Command line (planned): `lin-hot-cooling` (window), `lin-hot-cooling --agent` (background rules), `--debug`, `--version`.

## 5. The GTK Python stack

| Layer | Technology | Why |
| --- | --- | --- |
| Language | **Python 3.12** (3.11 minimum) | Fast to develop, native on Debian, rich system libraries |
| UI toolkit | **GTK 3** via **PyGObject** (`gi.repository.Gtk`, `Gdk`, `GLib`, `Gio`) | Native Linux look and performance; the base of the starter template. The code is written to port to GTK 4. |
| Graphics | **Cairo** (`pycairo`) in custom drawing widgets | Animated fan rotors, arc gauges, sparklines and gradients without web views |
| Animation | GTK **frame-clock tick callbacks** + an easing library (cubic-bezier, spring) | vsync-aligned, efficient, pauses when hidden |
| Styling | **GTK CSS** with design tokens (`@define-color`), bundled in a **GResource** | One source of truth for colors, radii and gradients |
| App model | `Gtk.Application` (single instance, `Gio` actions), `GObject` signals | Desktop integration, clean state propagation |
| Threads | Worker threads for sampling; `GLib.idle_add` to update the UI | The UI never blocks on I/O |
| System access | sysfs/procfs readers, **D-Bus** (`Gio.DBusProxy`) for power-profiles-daemon, UPower, systemd and GameMode | Standard, distro-agnostic interfaces |
| GPU data | **NVML** (`pynvml`), with an `nvidia-smi` fallback; amdgpu hwmon; i915/xe DRM fdinfo | Accurate GPU telemetry without root |
| Privileged actions | A separate **helper daemon** on the system D-Bus, authorized by **polkit**, run by **systemd** with sandboxing | The GUI never runs as root |
| Data | **YAML** templates and heat vectors, **JSON Schema** validation, **SQLite** history, **TOML** themes | Data-driven, versioned, user-extensible |
| Quality | `ruff`, `mypy`, `pytest` (+ Hypothesis), `xvfb-run` UI smoke tests, import-linter, a GTK 4 portability lint | Keeps the modular structure and portability honest |
| Packaging | `debhelper` + `dh-python` `.deb`, `.desktop`, AppStream metainfo | Native install on Debian/Ubuntu/Mint |

System packages (runtime): `python3-gi gir1.2-gtk-3.0 python3-gi-cairo python3-yaml python3-jsonschema polkitd dbus`.
Recommended: `lm-sensors power-profiles-daemon python3-pynvml smartmontools nvme-cli gamemode`.

**Portability to GTK 4:**
- The GTK version is chosen in one place.
- Pages use a small compatibility layer (`ui/compat.py`) instead of GTK 3-only calls (`pack_start`, `show_all`, event signals, `Gtk.Menu`).
- Custom widgets draw through a shared canvas wrapper.
- A CI lint blocks new GTK 3-only code.
- Milestone M7 flips the switch.

## 6. Architecture and modularity

```
┌─────────────── user session ───────────────┐          ┌──────────── system ────────────┐
│ UI (pages, components)                     │          │ lin-hot-cooling-helperd (root)  │
│   ↓                                        │  D-Bus   │  • validates plans (allow-list) │
│ View models → Core (state, events, sched.) │ ───────▶ │  • actuators: profiles, EPP,    │
│                 ↑              ↓           │ + polkit │    fan modes/curves, power caps │
│ Providers (collectors)   Policy (resolver, │          │  • leases + watchdog → firmware │
│   sysfs · D-Bus · NVML    rules, templates)│          │    auto on any failure          │
└────────────────────────────────────────────┘          └─────────────────────────────────┘
```

Built for **robust modularity and universality**:
- **Layers with one-way dependencies.** UI → view models → core ← policy; providers feed core. The rules are enforced in CI.
- **Everything is a plugin.** Collectors, actuators, pages, triggers and themes implement small interfaces and register themselves. The sidebar is generated from the page registry. New hardware or features are added as a new module or data file, never as special cases in core code.
- **Capabilities, not models.** The code asks "can this fan boost?", never "is this an ASUS?". Vendor specifics live only in providers, actuators and machine-profile data files.
- **Data-driven.** Categories, templates, heat vectors, rules, thresholds and themes are versioned data with schemas.
- **Dependency injection** (`AppContext`) and an **event bus** keep modules independent and testable with fake sysfs trees, clocks and D-Bus.
- **Toolkit-agnostic core.** Only `ui/` knows GTK, which keeps the GTK 4 port (and any future CLI/TUI) cheap.

Details: [TECHNICAL-CONCEPT.md §3–§5, §20](TECHNICAL-CONCEPT.md).

## 7. Design language

A flat, dark, premium look taken from the MensuraMedia `ui-ki-green-gray-black` reference:

| Token | Color | Role |
| --- | --- | --- |
| Base | `#080C17` | Window background (deep navy-black) |
| Surfaces | `#1E222E` / `#262A36` / `#353743` | Cards, raised elements, controls |
| Accent | `#C1FF14` | Neon lime: active state, primary actions, live indicator |
| Hero gradient | `#ECFEC2 → #D6F2A5 → #B8E86A` | "Cool" state card |
| Warm / Hot | `#F5C542` / `#F14D8A → #A01743` | Heat states and alerts |
| Cool | `#5DDEA5` | OK / cooling |
| Text | `#F5F7FA` / `#8B8F99` / `#585C65` | Primary / secondary / muted |

- **Gradients** only where they mean something: state hero cards, gauge arcs, chart fills, the sidebar depth.
- **Typography:** Inter (fallback Ubuntu/Cantarell), tabular figures for live values.
- **Shapes:** 18 px card radii, pill buttons.
- **Iconography:** a custom 24 px symbolic set (fan, thermometer, flame, snowflake-gauge, bolt, chip, GPU, gamepad, hourglass, …).
- **Motion:** fan rotors spin in proportion to real RPM; gauges ease with a spring; there's a reduced-motion mode; animation stays under 2 % CPU.

## 8. Safety and security

- **Firmware stays in charge of protection.** thermald and firmware critical trips are never disabled. Emergencies only add cooling.
- **Least privilege.** Only the helper writes to hardware, only to allow-listed attributes it discovered itself, and only after polkit authorization.
- **Leases and a watchdog.** Boosts and manual control expire unless renewed. A crash, exit or critical temperature returns control to firmware automatically.
- **No conflicts.** If TLP, asusd or another manager changes a control, automation pauses for it instead of fighting.
- **Private.** No network access. Diagnostics exports are local and redacted.
- **Sandboxed helper.** systemd hardening (`ProtectSystem=strict`, `NoNewPrivileges`, restricted syscalls, no network).

## 9. Installation

*Available from milestone M6.*

```bash
# From a GitHub release
sudo apt install ./lin-hot-cooling_<version>_all.deb
```

The custom non-commercial license means the package won't be in the official Debian/Ubuntu archives. Releases come from GitHub, and optionally from a PPA or apt repository. Uninstalling returns all fans and profiles to firmware defaults.

## 10. Development

*Available from milestone M0.*

```bash
sudo apt install python3-gi gir1.2-gtk-3.0 python3-gi-cairo python3-yaml python3-jsonschema xvfb
git clone git@github.com:MensuraMedia/lin-hot-cooling.git
cd lin-hot-cooling
./run.sh                 # creates a venv with --system-site-packages, compiles resources, launches
make check               # ruff, mypy, pytest, UI smoke test, GTK 4 portability lint, layer contracts
```

- **UI work without root:** the helper runs in dry-run mode on the session bus.
- **Workflow:** the project follows the MensuraMedia universal instruction set (`CLAUDE.md`, `.claude/` rules, hooks, memory and changelog).
  - Plan each milestone with `/plan-first`.
  - Verify with `/build-test`.
  - Log with `/session-end`.

## 11. Project layout

Planned layout (TECHNICAL-CONCEPT §11):

```
src/        main.py · app.py · gtk_version.py
            core/ (interfaces, registry, context, events, state, scheduler, units, ids, paths, easing, history)
            collectors/ · policy/ · helper_client/ · viewmodels/
            ui/ (window, sidebar, content area, compat, components/) · pages/ · config/ · modules/ · utils/
helper/     privileged daemon: service, validation, actuators/, watchdog
data/       templates/ · heat-vectors/ · machines/ · schema/
resources/  css/ · icons/ · images/ · fonts/ · GResource manifest
packaging/  debian/ · systemd/ · dbus/ · polkit/ · desktop/
tests/      fixtures/sysfs/<machine>/ · unit/ · integration/ · ui/
docs/       milestones.md · mockups/ · development.md · security.md · template-authoring.md · machine-profiles.md
```

## 12. Roadmap

| Milestone | Scope |
| --- | --- |
| **M0 Foundations** | Project setup, starter import, modular core, design system, icons, GTK 4 compat layer, CI |
| **M1 Read-only monitor** | Collectors, state model, live pages, animated components |
| **M2 Helper + categories** | Privileged helper, Idle/Optimal/Intense, Fans and Profiles pages |
| **M3 Templates + rules** | Heat vectors, catalog, resolver, automation, background agent |
| **M4 Insight** | History, benchmarks, power page |
| **M5 Advanced control** | Software curves, power caps, charge pause, manager integrations |
| **M6 Release (v1.0)** | Debian packaging, AppStream, documentation |
| **M7 GTK 4 port (v2.0)** | Version flip, compat shims removed, optional libadwaita |

Work packages and acceptance criteria: [docs/milestones.md](docs/milestones.md).

## 13. Documentation

| Document | Content |
| --- | --- |
| [TECHNICAL-CONCEPT.md](TECHNICAL-CONCEPT.md) | Full technical concept: architecture, discovery sources, state model, heat vectors, templates, rule engine, helper API, UI, design system, packaging, safety, testing, GTK 4 portability, modularity principles |
| [docs/milestones.md](docs/milestones.md) | Milestone plan M0–M7 |

## 14. License and credits

- **License:** free to use, copy, modify and distribute. **Commercial use requires prior explicit written permission** from MensuraMedia ([LICENSE](LICENSE)).
- **Starter template:** the application shell derives from [gtk-python-dashboard-starter](https://github.com/mikesdatawork/gtk-python-dashboard-starter) by mikesdatawork ("free for personal and educational use"). Third-party terms are in [NOTICE](NOTICE).
- **Design reference:** MensuraMedia universal-themes `ui-ki-green-gray-black`.
- **Reference hardware and field data:** the ASUS TUF F15 FX506LI tuning project.
