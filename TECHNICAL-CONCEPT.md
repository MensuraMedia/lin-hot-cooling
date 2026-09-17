# Lin Hot Cooling — Technical Concept

**A premium GTK thermal-management utility for Debian-based Linux**

| | |
| --- | --- |
| Document | Technical concept, v0.2 (decisions of 2026-09-17 applied) |
| Date | 2026-09-17 |
| Status | Concept + milestone plan ([docs/milestones.md](docs/milestones.md)). No application code yet. |
| Repository | https://github.com/MensuraMedia/lin-hot-cooling (package `lin-hot-cooling`, app ID `io.mensuramedia.LinHotCooling`) |
| License | Custom: free to use, modify, distribute; **commercial use only with prior written permission** ([LICENSE](LICENSE), [NOTICE](NOTICE)) |
| Target platforms | Debian 12+, Ubuntu 24.04+, Linux Mint 22+ (X11 first; Wayland is supported where noted) |
| Stack | Python 3.12, **GTK 3, written to be portable to GTK 4** (§19), PyGObject on the `gtk-python-dashboard-starter` template, Cairo for graphics, a D-Bus + polkit privileged helper |
| Standards | MensuraMedia/universal-instruction-set v2026.04 (see §14) |
| Design reference | `universal-themes/image-reference/ui-ki-green-gray-black.jpg` (see §10) |
| Reference hardware | ASUS TUF Gaming F15 FX506LI (i5-10300H, UHD + GTX 1650 Ti), profiled in `~/projects/optimize-laptop-asus-fx50li` |

---

## 1. Purpose

Lin Hot Cooling ("Hot Cooling" in the UI) is a local desktop utility that:

1. **Discovers and assimilates** everything the machine exposes about heat and cooling:
   - hardware specs and topology;
   - temperature, fan, power and frequency sensors;
   - available controls (fan modes, curves, platform profiles, CPU/GPU power limits, charge limits);
   - which service currently manages each control (firmware, thermald, power-profiles-daemon, TLP, asusd, NVIDIA driver…).
2. **Applies cooling policy dynamically and modularly** through three **cooling categories**: *Idle Cooling*, *Optimal Cooling* and *Intense Cooling*.
3. **Drives those categories from a catalog of cooling templates** for real workloads: high-intensity gaming, long-running operations, general use, and others. Each template is built from predefined **heat vectors**: the ways a workload turns into heat (§6).
4. **Presents it all** in a stylish, sophisticated, flat-dark interface with animated fans, live gauges and restrained gradients.

The application never exceeds firmware or vendor limits. **Its job is to choose well among the controls the hardware safely offers, and to explain what it cannot do.**

### Non-goals (v1)
- Writing raw embedded-controller registers (NBFC-style). This is unsafe, and machine-specific mistakes can disable fans.
- Overclocking, undervolting or firmware flashing.
- Replacing thermald or firmware thermal protection. Hot Cooling works **on top of** them.
- Windows or macOS, or non-Debian package formats (Flatpak is a later consideration, §15).

---

## 2. Guiding principles

| Principle | Consequence |
| --- | --- |
| **Discover, don't assume** | Every feature is driven by a runtime capability probe. Controls a machine doesn't have are hidden or shown as "not available on this hardware", with the reason. |
| **Least privilege** | The GUI runs as the user. Only a small, audited helper daemon writes to sysfs, over D-Bus with polkit authorization. |
| **Safe by default** | Every change is reversible. A watchdog returns control to firmware (auto) if the helper crashes, the GUI disappears or temperatures cross a critical line. |
| **One owner per control** | The app detects conflicting managers (TLP vs power-profiles-daemon, asusd, CoreCtrl, fancontrol) and either integrates with them or asks the user to choose. It never fights them silently. |
| **Explainable** | Every automatic action is logged with its trigger, template, heat vector, before/after values and outcome ("Why did the fan spin up?"). |
| **Premium, calm UI** | Flat dark surfaces, one neon accent, gradients only where they carry meaning (heat, active state), and motion that reflects real data. |
| **Universal standards** | Project structure, memory, change tracking, hooks, security and agents follow the MensuraMedia universal instruction set. |

---

## 3. System architecture

```mermaid
flowchart LR
  subgraph User session
    UI["GTK 3 app<br/>hot-cooling"] --> VM["View models<br/>+ animation engine"]
    VM --> SS["State store<br/>(observable)"]
    SS --> PE["Policy engine<br/>categories · templates · rules"]
    SS <-- samples --- COL["Collectors<br/>unprivileged"]
    PE -- apply plan --> CL["Helper client<br/>D-Bus proxy"]
    HIST[("History<br/>SQLite")] <-- SS
  end
  subgraph System
    CL -- D-Bus system bus + polkit --> HD["hot-cooling-helperd<br/>root, systemd"]
    HD --> ACT["Actuators<br/>sysfs · PPD · NVML · powercap"]
    HD --> WD["Watchdog<br/>fail-safe to auto"]
    ACT --> HW[("Kernel drivers /<br/>firmware / EC")]
    COL --> HW
  end
```

| Layer | Runs as | Responsibility |
| --- | --- | --- |
| **UI** (`src/ui`, `src/pages`) | user | Starter-template shell: sidebar navigation, pages, theme, animation |
| **View models** (`src/viewmodels`) | user | Turn state into display values (units, colors, trends); drive animation targets |
| **State store** (`src/core/state.py`) | user | One observable model of the machine (§5). GObject signals notify pages. |
| **Collectors** (`src/collectors`) | user | Pluggable providers that read sensors and capabilities; each declares a sampling cadence and cost |
| **Policy engine** (`src/policy`) | user | Resolves category + template + rules into an **apply plan** of desired actuator states, with hysteresis |
| **Helper client** (`src/helper_client`) | user | Typed D-Bus proxy; sends plans; receives results and watchdog events |
| **Privileged helper** (`helper/`) | root (systemd, D-Bus activated) | Validates each plan against the capability manifest and allow-lists, applies it, verifies it, reverts on failure; watchdog |
| **History** (`~/.local/share/hot-cooling/history.db`) | user | Time series (downsampled), action log, benchmark results |

**Process model:**
- The GUI can close at any time.
- An optional **user-session agent** (`hot-cooling --agent`, a systemd user unit) keeps rules running without the window open: switching when AC is plugged in or removed, and when a game starts or stops. It is fully specified as the **Thermal Agent** in [docs/thermal-agent-spec.md](docs/thermal-agent-spec.md).
- The helper exits after a period of inactivity, but the watchdog state survives in `/run/hot-cooling/`.

---

## 4. Discovery and assimilation (collectors)

Each collector implements:

```python
class Collector(Protocol):
    id: str                      # "hwmon", "nvidia", ...
    cadence_ms: int              # 1000 for fast sensors, 30000 for SMART, 0 for one-shot
    def probe(self) -> CapabilitySet: ...     # what exists, what is readable/writable, who owns it
    def sample(self) -> list[Reading]: ...    # cheap, non-blocking (runs in a worker thread)
```

### 4.1 Sources (Debian/Ubuntu/Mint)

| Domain | Source (unprivileged unless noted) | Provides |
| --- | --- | --- |
| Identity and specs | `/sys/class/dmi/id/*`, `/proc/cpuinfo`, `lscpu -J`, `lspci -nnk`, `/sys/class/drm`, `/etc/os-release`, `uname` | Vendor/model/BIOS, CPU model/cores/TDP class, GPUs + drivers, OS/kernel |
| Temperatures | `/sys/class/hwmon/*/temp*_input` (coretemp, k10temp, pch_*, nvme, iwlwifi, acpitz, amdgpu), `/sys/class/thermal/thermal_zone*` (type, trip points) | Per-sensor °C, trip/critical thresholds |
| Fans | `hwmon/*/fan*_input`, `pwm*`, `pwm*_enable`, `pwm*_auto_point*` (asus-nb-wmi, thinkpad_acpi, dell-smm-hwmon, nct6775/it87, amdgpu) | RPM, available modes, whether curves are supported |
| Firmware profiles | `/sys/firmware/acpi/platform_profile{,_choices}`, vendor attributes (`throttle_thermal_policy`, `fan_boost_mode`) | quiet / balanced / performance (etc.) |
| Power-profile manager | power-profiles-daemon over D-Bus (`org.freedesktop.UPower.PowerProfiles`, legacy `net.hadess.PowerProfiles`); TLP (`tlp-stat` availability) | Active profile, holds, conflicts |
| CPU | `cpufreq` (governor, EPP, `scaling_cur_freq`), `intel_pstate` (`no_turbo`, `max_perf_pct`), `/proc/stat`, `thermal_throttle/*_count` | Utilization, frequency, EPP, throttle events |
| CPU power | `/sys/class/powercap/intel-rapl*/energy_uj`, `constraint_*_power_limit_uw` — **root-only** (read through the helper) | Package watts, PL1/PL2 |
| iGPU | i915/xe **DRM fdinfo** (`/proc/<pid>/fdinfo`, `drm-engine-*`), `gt_*_freq_mhz` | Per-process busy %, frequency |
| NVIDIA | NVML (`python3-pynvml`), `nvidia-smi` fallback, `/proc/driver/nvidia/gpus/*/power` | Temp, W, utilization, VRAM, P-state, throttle reasons, power-limit range, RTD3 support |
| AMD GPU | `amdgpu` hwmon + `pp_od_clk_voltage`, `power1_cap` | Temp, W, fan and power caps |
| PRIME | `prime-select query`, `/sys/bus/pci/devices/*/power/runtime_status` | Hybrid mode, dGPU sleep state |
| Battery / AC | UPower D-Bus; `power_supply/*` (`online`, `current_now`, `voltage_now`, `charge_control_end_threshold`) | Source, drain W, charge limit |
| Storage | nvme hwmon; `smartctl -j` / `nvme smart-log -o json` (helper) | Drive temperatures, throttle counters, health |
| Services | systemd D-Bus (`thermald`, `power-profiles-daemon`, `tlp`, `asusd`, `nvidia-persistenced`, `fancontrol`, `coolercontrold`) | Who owns what → conflict detection |
| Workload | Process table, GameMode D-Bus (`com.feralinteractive.GameMode`), Steam `reaper`/`SteamLaunch` parents, `systemd-run` scopes | Triggers for templates (§7) |
| Environment | Lid state (`/proc/acpi/button/lid`), time of day, optional ambient (none by default) | Rule conditions |

### 4.2 Assimilation
- **Normalization.** Raw readings become typed `Reading(sensor_id, kind, value, unit, ts)`. Sensors get stable IDs (`cpu.package`, `gpu.nvidia0.core`, `fan.cpu0`, `storage.nvme0.composite`) based on driver + label, not hwmon numbers, which change between boots.
- **Topology.** Sensors, fans and actuators are grouped into **thermal zones** (CPU, dGPU, iGPU, chipset, storage, battery, chassis). A zone links its sensors to the fans and actuators that affect it. The link can be declared by the machine profile (§4.3) or learned from benchmark correlation.
- **Capability manifest.** A JSON document of everything probed: readable/writable, ranges, owner, confidence. The helper only accepts plans that fit the manifest it generated itself.
- **Machine profiles.** Optional community JSON files (`data/machines/<vendor>/<model>.json`) add labels, zone maps and known quirks. Example for the reference laptop:

```jsonc
{
  "match": { "sys_vendor": "ASUSTeK COMPUTER INC.", "product_name": "ASUS TUF Gaming F15 FX506LI*" },
  "fans": [{ "id": "fan.cpu0", "hwmon": "asus", "input": "fan1_input", "label": "System fan",
             "modes": { "auto": 2, "full": 0 }, "curves": false, "zones": ["cpu", "gpu"] }],
  "platform_profile": { "choices": ["quiet", "balanced", "performance"], "hotkey": "Fn+F5" },
  "gpu": { "nvidia0": { "rtd3": false, "power_limit_writable": false } },
  "battery": { "charge_limit": "BAT1/charge_control_end_threshold" },
  "quirks": ["asus_wmi fan_curve_get_factory_default fails (-19/-61): no custom curves",
             "RAPL energy_uj root-only; read via helper"]
}
```

### 4.3 Degradation matrix (how features adapt)

| Machine offers | Hot Cooling provides |
| --- | --- |
| Fan curves (`pwm*_auto_point*`) | Full curve editor per template |
| Manual PWM (`pwm*_enable=1`) | Software curves (helper loop at 1 Hz, with hysteresis and a watchdog) |
| Only auto/full (e.g. FX506LI) | Profile-based control + a timed "Max fan" boost; the curve editor is hidden, with an explanation |
| `platform_profile` only | Category → profile mapping |
| No fan sensor | Temperature-only UI; fans shown as "firmware-managed" |
| RAPL writable (helper) | PL1/PL2 caps per template (within firmware max) |
| NVML power limit writable (desktop GPUs) | GPU power caps per template |

---

## 5. State model

```
MachineState
├── identity: vendor, model, bios, cpu, gpus[], os, kernel
├── capabilities: CapabilityManifest
├── zones[]: id, sensors[], fans[], actuators[], limits{warn, crit}
├── readings: ring buffers per sensor (last 10 min @1 Hz) + SQLite downsampled history (1 min, 30 days)
├── power: source (ac|battery), drain_w, charge_limit, ppd_profile, platform_profile
├── workload: active triggers (game, build, render, idle), GameMode clients, top processes
├── policy: active category, active template, pending plan, last applied plan, overrides
└── health: throttle events (delta), NVMe throttle, conflicts[], helper status, watchdog status
```

The state is exposed as a GObject with signals (`reading-updated`, `capability-changed`, `policy-changed`, `alert`), so pages stay decoupled, as the starter's `NavigationManager` pattern expects.

---

## 6. Heat vectors

A **heat vector** is a normalized description of how a workload turns into heat over time. Templates are composed from heat vectors, so new templates reuse proven thermal behavior.

| Vector | Source | Shape | Typical signals | Primary levers |
| --- | --- | --- | --- | --- |
| `cpu.burst` | Short CPU spikes (UI, compiles of small units, web) | Seconds, high peak | Package temperature jumps, PL2 | EPP, turbo, fan ramp speed |
| `cpu.sustained` | Long all-core load (builds, encodes, simulations) | Minutes to hours, plateau | Package temperature at PL1, throttle events | PL1, max_perf_pct, profile, fan mode |
| `gpu.sustained` | 3D rendering, games, ML inference | Minutes to hours | dGPU temp/W, P0 | Profile, GPU power cap (if any), fan boost, frame cap advice |
| `gpu.idle-leak` | dGPU awake but idle (no RTD3) | Constant, low | dGPU P8 at a few W | PRIME mode advice, integrated-only suggestion |
| `mixed.gaming` | CPU + GPU together, bursty frames | Sustained with spikes | Both packages hot, fan at max | Performance profile, fan boost, GameMode |
| `io.storage` | Large copies, indexing, torrenting | Sustained | NVMe composite temp, throttle state | Warnings; I/O scheduling advice (`ionice`) |
| `power.charging` | Charging while under load | Sustained | Battery temp, input W | Charge limit / charge pause during intense runs |
| `ambient.chassis` | Lid closed, soft surface, hot room | Slow drift | Rising idle baseline, PCH temp | Alerts, quieter targets, reminder to improve airflow |

Every vector specifies:
- **Detection:** sensor slopes, utilization windows, process classes.
- **Expected thermal envelope:** warning and critical per zone.
- **Recommended actuator bias:** e.g. `fan: early`, `cpu_pl1: -15 %`, `profile: performance`.

These are stored in `data/heat-vectors/*.yaml`.

---

## 7. Cooling categories and the template catalog

### 7.1 Categories (the three user-facing modes)

| Category | Intent | Default mapping (resolved against capabilities) |
| --- | --- | --- |
| **Idle Cooling** | Silence and efficiency when little is happening | Profile `quiet`/`power-saver`; EPP `power`; fans auto; dGPU sleep advice; charge limit on |
| **Optimal Cooling** | Balanced acoustics and performance for everyday work | Profile `balanced`; EPP `balance_performance`; fans auto; moderate PL1 |
| **Intense Cooling** | Maximum heat removal for heavy sustained load | Profile `performance`; EPP `performance`; fan boost/full when temperatures cross thresholds; GPU/CPU caps per template; pause charging if the battery is hot |

A category is the **baseline**. A **template** refines it for a workload, and **rules** decide when each applies.

### 7.2 Template catalog (shipped, read-only; users can clone)

| Template | Category | Heat vectors | Highlights |
| --- | --- | --- | --- |
| **High-Intensity Gaming** | Intense | `mixed.gaming`, `gpu.sustained`, `power.charging` | Performance profile on AC; fan boost above CPU 85 °C / GPU 80 °C; charge pause above 40 °C battery; applies GameMode hooks; suggests an FPS cap when throttling is detected |
| **Esports / Competitive (low latency)** | Intense | `mixed.gaming`, `cpu.burst` | Performance EPP, no fan boost delay, turbo on |
| **Long-Running Operations** (builds, renders, training) | Intense → Optimal | `cpu.sustained`, `gpu.sustained`, `io.storage` | Caps PL1 slightly below the throttle point for *stable* throughput, fan boost with a 60 s hold, NVMe temperature watch, "runs for hours" safety margins |
| **Development / Compile bursts** | Optimal | `cpu.burst`, `cpu.sustained` | Fast ramp-up, fast ramp-down, inotify-heavy I/O advice |
| **General Operations** | Optimal | `cpu.burst` | Balanced; no manual fan actions |
| **Media & Streaming** | Optimal | `gpu.sustained` (light), `cpu.sustained` (encode) | Keeps fans steady (avoids pulsing), prefers the iGPU for decode |
| **Quiet Office / Night** | Idle | `cpu.burst`, `ambient.chassis` | Quiet profile, EPP power, fan boost disabled unless a critical temperature is reached |
| **On Battery** | Idle | `gpu.idle-leak` | Power-saver; PRIME advice; no boosts |
| **Cool-down** (transient) | Intense → Idle | all | After heavy load: holds fans up until zones fall below baseline + 5 °C, then steps down |

### 7.3 Template schema (YAML, validated with JSON Schema)

```yaml
id: gaming-high-intensity
name: High-Intensity Gaming
version: 1
category: intense
heat_vectors: [mixed.gaming, gpu.sustained, power.charging]
applies_when:                      # evaluated by the rule engine
  any:
    - gamemode.active: true
    - process.class: game           # Steam reaper child, Proton, known executables
  all:
    - power.source: ac
targets:                           # desired end-state; the resolver drops what isn't supported
  platform_profile: performance
  cpu.epp: performance
  cpu.turbo: on
  gpu.prime_offload_hint: true
  battery.charge_pause_above_c: 40
fan:
  mode: auto                       # auto | boost | curve
  boost:
    when: { any: [{zone.cpu.temp_c: {gt: 85}}, {zone.gpu.temp_c: {gt: 80}}] }
    hold_s: 45
    release_below: { zone.cpu.temp_c: 75, zone.gpu.temp_c: 70 }
  curve:                           # used only where curves are supported
    - { temp_c: 50, duty: 30 }
    - { temp_c: 70, duty: 60 }
    - { temp_c: 85, duty: 100 }
limits:
  critical: { zone.cpu.temp_c: 97, zone.gpu.temp_c: 90 }   # → emergency: full fan + profile quiet + notify
hysteresis: { temp_c: 3, min_dwell_s: 20 }
exit:
  after_trigger_gone_s: 30
  then: template:cool-down
```

**Resolution pipeline:**
1. **Category baseline**, then **template overrides**, then **user overrides**.
2. **Capability filter:** unsupported keys are dropped and recorded as "not applied: reason".
3. **Conflict filter:** if another service owns a control, integrate with it (e.g. set the power-profiles-daemon profile instead of writing `platform_profile`) or skip.
4. **Diff** against the current state.
5. **Apply plan**, sent to the helper.

### 7.4 Rule engine
- **Triggers:**
  - power source change;
  - GameMode registration;
  - process class start/stop;
  - zone temperature crossing a threshold;
  - sustained utilization window;
  - lid state;
  - schedule;
  - manual selection.
- **Evaluation:** every 1 s (fast path for temperatures), with **hysteresis** and a **minimum dwell time**, so the system doesn't flap.
- **Priority when triggers conflict:** Emergency > Manual override (time-boxed) > Workload template > Power-source default > Category baseline.
- **Default rules** (these match the reference laptop's documented policy):
  - AC → `performance`, battery → `quiet`.
  - A game starts → *High-Intensity Gaming* (AC only).
  - Load ends → *Cool-down* → back to the power-source default.

---

## 8. Privileged helper (`hot-cooling-helperd`)

| Aspect | Design |
| --- | --- |
| Transport | D-Bus **system bus**, name `io.mensuramedia.LinHotCooling1`, D-Bus-activated systemd service |
| Authorization | polkit actions: `io.mensuramedia.linhotcooling.apply-profile` (allowed for active local sessions without a password), `…apply-fan` and `…apply-power-limits` (auth_admin_keep), `…configure-rules` |
| API (sketch) | `GetCapabilities() → a{sv}`, `ReadPrivileged(keys) → a{sv}` (RAPL energy, SMART), `ApplyPlan(plan: a{sv}, lease_s: u) → (applied, rejected, token)`, `RenewLease(token)`, `RevertAll()`, signal `WatchdogFired(reason)` |
| Validation | Every key must be in the helper's own capability manifest. Values are checked against probed ranges and a static allow-list of sysfs paths (no path strings accepted from clients). |
| Lease + watchdog | Aggressive changes (full fan, manual PWM, reduced limits) carry a **lease**. If the lease isn't renewed, the helper returns everything to firmware defaults. Critical temperature crossings trigger an emergency revert regardless of plan. |
| Verification | After writing, the helper reads back the value and confirms the effect (for example, fan RPM changes within 5 s). Mismatches are reported, not retried endlessly. |
| Persistence | Nothing is persistent by default. An optional "restore last category at boot" uses a system unit that runs a stored, validated plan. |
| Hardening | systemd sandboxing: `ProtectSystem=strict`, a `ReadWritePaths=` allow-list under `/sys`, `NoNewPrivileges`, `CapabilityBoundingSet` limited to what sysfs writes need, `PrivateNetwork=yes`, `SystemCallFilter=@system-service` |
| Audit | Structured journal entries (`HOT_COOLING_ACTION=`, `PLAN_ID=`, `KEY=`, `OLD=`, `NEW=`, `RESULT=`) |

---

## 9. User interface

Built on the **gtk-python-dashboard-starter** shell:
- fixed sidebar with a logo area;
- page-based routing through `NavigationManager`;
- `BasePage` subclasses;
- centralized `config_layout` / `config_theme(s)`;
- a CSS provider applied by `ThemeApplicator`.

The template is extended rather than replaced.

### 9.1 Navigation (sidebar, 150 → 200 px for icon + label)

| Page | Content |
| --- | --- |
| **Overview** | Hero gradient card ("Thermal state: Cool / Warm / Hot") showing the active category, template and power source; animated fan tiles; mini gauges for CPU, dGPU and iGPU; a "Why" feed of the last three automatic actions |
| **Fans** | One large animated rotor per fan (RPM, duty, mode); a curve editor where supported; *Max fan* boost button with countdown; fan-response chart |
| **Thermals** | Zone cards with sparklines (10 min) and trip points; a heat map strip per zone; throttle event counters |
| **Power** | Source, drain W, charge limit, CPU package W (via helper), GPU W, EPP/turbo, PRIME status |
| **Profiles** | The three category cards (Idle / Optimal / Intense) with a single-tap switch, and what each one changes on *this* machine |
| **Templates** | Catalog grid (cards with icon, heat-vector chips, category badge); detail view with resolved targets and "not applicable here" notes; clone/edit (YAML with form view) |
| **Automation** | Rule list (triggers → template), priorities, test-trigger button, event history |
| **Hardware** | Specs, zone topology diagram, capability matrix (✔/✖ + reason), detected conflicts (TLP vs PPD, etc.), machine profile source |
| **Benchmarks** | Guided idle/load runs per category (a successor to `bench-gpu-thermal.sh`); comparison charts (FPS, temperatures, RPM, throttle events) |
| **History** | Timeline charts (1 h / 24 h / 7 d) and the action log with filters |
| **Settings** | Units, sampling rate, animations (full / reduced / off), theme accents, agent autostart, helper status, export diagnostics |
| **About** | Version, licenses, credits (starter template, icon sources) |

### 9.2 Signature components (custom `Gtk.DrawingArea` + Cairo)

| Component | Behavior |
| --- | --- |
| **FanRotor** | A 7-blade vector rotor that spins at an angular speed mapped from RPM: `ω = clamp(rpm / rpm_max, 0, 1) · ω_max`, with `ω_max` ≈ 2.5 rev/s (the visual cap, so it doesn't strobe). Blade motion blur (alpha-layered ghost blades) above 60 % speed. A lime glow ring shows duty. It "spins down" with easing when RPM falls to 0, and a dashed outline means firmware-managed with no reading. |
| **ThermalGauge** | A 270° arc gauge whose arc paint is a green → yellow → red gradient. Needle and value animate with critically damped spring easing. Trip points are drawn as ticks. |
| **HeatSparkline** | An area chart with a vertical gradient fill (`#293C1C` → transparent). The line turns magenta above the warning line. |
| **CategorySwitch** | A segmented control (Idle / Optimal / Intense) in the style of the reference "Monthly / Weekly / Daily" tabs. The active pill slides. |
| **HeroCard** | A gradient card for the current heat level: green (nominal), yellow (medium) or red (intense), as decided on 2026-09-17 |
| **HeatVectorChip** | Small rounded chip with an icon and label; its color comes from the vector class. |
| **PulseDot** | A live indicator (lime) that pulses at the sampling rate; amber while waiting for the helper. |

### 9.3 Motion system
- **Timing:** a single `Gtk.Widget.add_tick_callback` per animated widget, driven by the frame clock (vsync-aligned, efficient). No `GLib.timeout_add` loops for animation.
- **Easing:** a shared library of cubic-bezier and spring easing curves; data transitions last 250–400 ms, page transitions 200 ms with a crossfade (`Gtk.Stack`).
- **Accessibility:** respect `gtk-enable-animations` and the app's *Reduced motion* setting (rotors show a static speed ring plus a numeric value).
- **Budget:** animations must stay **under 2 % CPU** on the reference laptop. Rotors pause when the window is hidden or minimized (`map`/`unmap`, `window-state-event`), and sampling drops to 5 s while hidden.

### 9.4 Iconography
- **Custom symbolic SVG set** in `resources/icons/scalable/actions/hc-*-symbolic.svg`: 24 px grid, 1.5 px strokes, rounded caps. Loaded through `Gtk.IconTheme.add_resource_path`, recolored by CSS (`-gtk-icon-style: symbolic`).
- **Core glyphs:**
  - fan rotor;
  - thermometer with a wave;
  - flame (heat vector);
  - snowflake with a gauge (cooling);
  - bolt (power);
  - plug and battery (source);
  - chip (CPU);
  - GPU card;
  - drive;
  - leaf (Idle), scale (Optimal), rocket (Intense);
  - gamepad (gaming), hourglass (long-running), briefcase (general), moon (quiet);
  - wand (automation), shield (watchdog), info.
- **App icon:** a lime rotor over a deep-navy rounded square with a subtle inner gradient. Shipped as SVG plus PNG sizes 16–512 for `hicolor`.
- **Fallback:** freedesktop symbolic names (`sensors-fan-symbolic`, `temperature-symbolic`, …) if a custom glyph is missing.

### 9.5 Wireframe (Overview, 1200 × 800)

High-fidelity mockups of Overview, Fans, Templates and Hardware are in [docs/mockups/](docs/mockups/README.md).

```
┌──────────┬───────────────────────────────────────────────────────────────────┐
│  (logo)  │  Overview                                   ● live   [AC ⚡]       │
│          │ ┌──────────────── lime gradient hero ───────────────────────────┐ │
│ ◉ Overview│ │ Thermal state   COOL            Category  Optimal ▸ General  │ │
│ ⟳ Fans    │ │ 52 °C CPU · 49 °C GPU · 0 RPM          [ Idle | Optimal | Intense ] │
│ ≋ Thermals│ └────────────────────────────────────────────────────────────────┘ │
│ ⚡ Power   │ ┌── Fan ──────────┐ ┌── CPU ──────┐ ┌── dGPU ─────┐ ┌─ iGPU ──┐  │
│ ▣ Profiles│ │   (spinning     │ │  (arc gauge)│ │ (arc gauge) │ │ (gauge) │  │
│ ☰ Templates│ │    rotor)      │ │   54 °C     │ │   49 °C 3 W │ │  36 %   │  │
│ ✦ Automation│ │ 2 900 RPM auto │ │ 4.0 GHz 9 % │ │ P8          │ │         │  │
│ ▦ Hardware│ └─────────────────┘ └─────────────┘ └─────────────┘ └─────────┘  │
│ ◔ Benchmarks│ ┌── Why did that happen? ──────────────────────────────────────┐ │
│ ◷ History │ │ 21:22  Game started → High-Intensity Gaming (AC)              │ │
│──────────│ │ 21:40  Game ended → Cool-down (fans held 45 s) → Optimal      │ │
│ ⚙ Settings│ └──────────────────────────────────────────────────────────────┘ │
└──────────┴───────────────────────────────────────────────────────────────────┘
```

---

## 10. Visual design system

The palette is sampled from `ui-ki-green-gray-black.jpg`: a deep navy-black base, charcoal cards, one neon-lime accent, lime gradient hero cards, and magenta for negatives.

### 10.1 Color tokens

| Token | Hex | Use |
| --- | --- | --- |
| `bg.base` | `#080C17` | Window background |
| `bg.sidebar` | `#0B101D` | Sidebar (slightly lifted) |
| `surface.1` | `#1E222E` | Cards, list rows |
| `surface.2` | `#262A36` | Raised card, hover |
| `surface.3` | `#353743` | Round icon buttons, inputs, segmented track |
| `border.subtle` | `#FFFFFF0F` | 1 px hairlines (6 % white) |
| `text.primary` | `#F5F7FA` | Titles, values |
| `text.secondary` | `#8B8F99` | Labels |
| `text.muted` | `#585C65` | Captions, inactive nav |
| `accent.lime` | `#C1FF14` | Primary actions, active nav, FAB, live dot |
| `accent.lime.soft` | `#ECFEC2` → `#D6F2A5` → `#B8E86A` | Hero gradient |
| `accent.lime.deep` | `#293C1C` | Chart fills, gauge background |
| `heat.nominal` / `heat.medium` / `heat.intense` | `#22C55E` / `#FACC15` / `#EF4444` | **Heat levels: green / yellow / red** (decision 2026-09-17; see ui-layout-spec v1.1) |
| `state.cool` | `#5DDEA5` | OK/applied status, Idle category (mint) |
| `state.warm` | `#F5C542` | Advisory status, boost timer |
| `category.intense` | `#DC143C` / `#C8102E` | Intense template category (crimson; red states are never pink) |
| `category.intense.deep` | `#A50E2A` / `#6B0A1A` | Intense category gradient |
| `on.accent` | `#0A0F05` | Text on lime surfaces |

### 10.2 Typography and layout
- **Font:** Inter (or Manrope). The fallback chain is Ubuntu → Cantarell → sans-serif.
- **Sizes:** page title 22 px/600; hero numerals 40 px/600 with tabular figures (`font-feature-settings: "tnum"`); labels 11 px/500 at +2 % letter-spacing.
- **Radii:** cards 18 px, pills 999 px, buttons 12 px.
- **Spacing:** 8 px grid; card padding 20 px; page margin 32 px.
- **Elevation:** no drop shadows (flat). Depth comes from surface steps and hairlines only.

### 10.3 Gradients (only where they carry meaning)

```css
/* resources/css/hot-cooling.css — GTK 3 CSS */
@define-color hc_bg #080C17;
@define-color hc_surface1 #1E222E;
@define-color hc_accent #C1FF14;

window, .content-area { background-color: @hc_bg; }

.card { background-color: @hc_surface1; border-radius: 18px; border: 1px solid alpha(white, 0.06); padding: 20px; }

/* heat levels: green / yellow / red (never pink); see docs/ui-layout-spec.md v1.2 */
.hero-card { border-radius: 22px; }
.hero-card.nominal { background-image: linear-gradient(135deg, #DCFCE7 0%, #86EFAC 45%, #22C55E 100%); color: #052E16; }
.hero-card.medium  { background-image: linear-gradient(135deg, #FEF9C3 0%, #FDE047 50%, #EAB308 100%); color: #1A1204; }
.hero-card.intense { background-image: linear-gradient(135deg, #DC2626 0%, #B91C1C 55%, #7F1D1D 100%); color: #FFFFFF; }

.sidebar { background-image: linear-gradient(180deg, #0B101D 0%, #080C17 100%); }
.nav-button:checked { background-color: alpha(@hc_accent, 0.10); color: @hc_accent; border-left: 3px solid @hc_accent; }

.pill-primary { background-color: @hc_accent; color: #0A0F05; border-radius: 999px; font-weight: 600; }
.segmented { background-color: #353743; border-radius: 999px; padding: 4px; }
.segmented .active { background-color: #8B8F99; color: #0A0F05; border-radius: 999px; }
```

The Cairo components use the same tokens from `config/config_theme.py`, with no hard-coded colors in drawing code. The starter's seven themes are kept as optional accents, but **"Hot Cooling Lime"** is the default and only fully designed theme.

---

## 11. Repository layout (starter template + additions)

```
lin-hot-cooling/                  # local checkout: ~/projects/hot-cooling
├── CLAUDE.md  changelog.md  .claudeignore  .gitignore  README.md  TECHNICAL-CONCEPT.md  LICENSE  NOTICE
├── .claude/                      # universal-instruction-set deployment (§14)
├── run.sh                        # starter launcher (venv with --system-site-packages)
├── requirements.txt              # PyGObject, pycairo, PyYAML, jsonschema, pynvml (optional)
├── src/
│   ├── main.py                   # starter entry → Gtk.Application (single instance, --agent)
│   ├── config/                   # config_layout.py, config_theme.py (tokens), config_themes.py
│   ├── ui/                       # dashboard_window, sidebar, content_area (starter) + components/
│   │   └── components/           # fan_rotor.py, thermal_gauge.py, heat_sparkline.py,
│   │                             # category_switch.py, hero_card.py, heat_vector_chip.py
│   ├── pages/                    # page_overview, page_fans, page_thermals, page_power, page_profiles,
│   │                             # page_templates, page_automation, page_hardware, page_benchmarks,
│   │                             # page_history, page_settings, page_about (all BasePage)
│   ├── viewmodels/
│   ├── core/                     # state.py, units.py, easing.py, scheduler.py, history.py
│   ├── collectors/               # hwmon.py, thermal_zone.py, cpufreq.py, drm_fdinfo.py, nvidia.py,
│   │                             # amdgpu.py, power_supply.py, ppd.py, services.py, workload.py, dmi.py
│   ├── policy/                   # resolver.py, rules.py, templates.py, heat_vectors.py, conflicts.py
│   ├── helper_client/            # dbus_proxy.py, plans.py
│   ├── modules/                  # manager_navigation.py, manager_theme_applicator.py (starter)
│   └── utils/                    # manager_theme.py (starter), sysfs.py, logging.py
├── helper/                       # hot-cooling-helperd (root): service.py, actuators/, validate.py, watchdog.py
├── data/
│   ├── templates/*.yaml          # the catalog (§7.2)
│   ├── heat-vectors/*.yaml
│   ├── machines/<vendor>/*.json  # machine profiles (§4.2)
│   └── schema/*.json             # JSON Schemas for templates, vectors, plans, profiles
├── resources/
│   ├── css/hot-cooling.css  icons/  images/  fonts/
│   └── hot-cooling.gresource.xml
├── packaging/
│   ├── debian/                   # debhelper + dh-python (§12)
│   ├── systemd/                  # hot-cooling-helperd.service, hot-cooling-agent.service (user)
│   ├── dbus/                     # io.mensuramedia.LinHotCooling1.conf + .service
│   ├── polkit/                   # io.mensuramedia.linhotcooling.policy
│   └── desktop/                  # io.mensuramedia.LinHotCooling.desktop, metainfo.xml, icons
├── tests/
│   ├── fixtures/sysfs/<machine>/ # recorded fake sysfs trees (the FX506LI first)
│   ├── unit/  integration/  ui/
└── docs/                         # architecture.md, template-authoring.md, machine-profiles.md, security.md
```

Starter conventions are kept:
- `page_*.py` modules inherit `BasePage` and implement `build_content()`.
- Pages are registered in `content_area.py` and added to `nav_items` in `sidebar.py`.
- Layout constants go only in `config_layout.py`.
- Every module starts with a header docstring.

---

## 12. Debian integration and packaging

| Item | Plan |
| --- | --- |
| Runtime dependencies | `python3 (>= 3.11)`, `python3-gi`, `gir1.2-gtk-3.0`, `python3-gi-cairo`, `python3-yaml`, `python3-jsonschema`, `policykit-1`/`polkitd`, `dbus` |
| Recommends | `lm-sensors`, `power-profiles-daemon`, `python3-pynvml` or the `nvidia-utils-*` `nvidia-smi` fallback, `smartmontools`, `nvme-cli`, `gamemode` |
| Conflicts / notes | TLP, fancontrol, asusd and coolercontrold are **detected, not declared as conflicts**. The app offers an integration mode per service. |
| License impact | The non-commercial license is not DFSG-free, so no upload to Debian/Ubuntu archives. Distribution is via GitHub releases (`.deb`) and optionally a Launchpad PPA / own apt repo; `debian/copyright` states the custom terms. |
| Build | `debhelper-compat (= 13)`, `dh-python`, `pybuild`; GResource compiled at build time |
| Installed files | `/usr/lib/python3/dist-packages/hot_cooling/`, `/usr/libexec/hot-cooling/hot-cooling-helperd`, `/usr/share/hot-cooling/{templates,heat-vectors,machines,schema}`, `/usr/share/dbus-1/system.d/`, `/usr/share/dbus-1/system-services/`, `/usr/share/polkit-1/actions/`, `/usr/lib/systemd/system/`, `/usr/lib/systemd/user/`, `/usr/share/applications/`, `/usr/share/metainfo/`, `/usr/share/icons/hicolor/` |
| Maintainer scripts | Reload D-Bus and systemd; no config written to `/etc` (user templates live in `~/.config/hot-cooling/`). Uninstall reverts to firmware auto. |
| Development | `./run.sh` in a venv created **with `--system-site-packages`** (system GTK bindings), per the universal-memory decision; the helper can run in `--session-bus --dry-run` mode for UI work without root |
| Quality gates | `ruff` (lint + format), `mypy --strict` on `core/`, `policy/`, `helper/`; `pytest`; `desktop-file-validate`; `appstreamcli validate`; `lintian` |

---

## 13. Safety, security and reliability

1. **Firmware first.** Never disable thermald or the firmware's critical trips. Emergency handling only ever **adds** cooling.
2. **Allow-listed writes only.** The helper maps plan keys to known sysfs attributes it discovered itself. Client-supplied paths are rejected.
3. **Leases and a watchdog.** Boosts, manual PWM and reduced limits expire unless renewed. When the helper stops, `ExecStopPost` restores auto mode.
4. **Emergency path.** Any zone at or above critical − 3 °C, **or** a throttle-event storm → full fan (if available) + the firmware's safest profile + a desktop notification. It holds until the zone is below warning for 60 s.
5. **Conflict safety.** If another manager changes a control the app owns, the app pauses automation for that control and notifies the user. It doesn't fight back.
6. **Privacy.** No network access. Diagnostics export is local and redacts serial numbers and MAC addresses.
7. **Side channels.** RAPL energy counters stay root-only, and the helper publishes only coarse (≥ 1 s averaged) package watts (the PLATYPUS mitigation).
8. **Suspend/resume.** Plans are re-applied after resume, because firmware may reset controls (observed on ASUS: keyboard backlight, charge threshold).
9. **Package safety.** Debian maintainer scripts never remove other packages. Apt transactions in development tooling are simulated first (lesson from the reference laptop).

---

## 14. Build process under the universal instruction set

The project is initialized and maintained exactly as `MensuraMedia/universal-instruction-set/universal-instruction-set/CLAUDE.md` prescribes (copy, don't reference; universal files are immutable; additions go alongside them):

| Step | Action for `hot-cooling` |
| --- | --- |
| 1 | Repo `MensuraMedia/lin-hot-cooling` (exists; local checkout `~/projects/hot-cooling`, SSH remote); `.gitignore` for Python, venv, build and `debian/` artifacts |
| 2 | `universal-agents/setup.sh ~/projects/hot-cooling "Lin Hot Cooling" "GTK thermal management utility for Debian-based Linux"`, then `universal-permissions/setup.sh` |
| 3 | `CLAUDE.md` from the v2026.04 template: build/test/lint/run commands (`./run.sh`, `pytest`, `ruff`, `mypy`, `dpkg-buildpackage -us -uc`) |
| 4 | Rules: universal `memory-rules.md`, `security.md`, `token-hygiene.md` + project rules `python-gtk.md` (GTK main-thread-only UI, no blocking I/O in callbacks, `GLib.idle_add` from workers), `privileged-helper.md` (allow-lists, polkit, no shell-outs with user input), `thermal-safety.md` (§13) |
| 5 | Enable the Python section in `post-edit-lint.sh` (ruff); keep the security gate and session hooks |
| 6 | Memory: `decisions.md` seeded with this document's key decisions (GTK 3 via the starter, helper architecture, template model); `pending.md` with the milestones below; `changelog.md` first entry |
| 7 | **universal-themes:** `ui-ki-green-gray-black.jpg` is the binding visual reference (§10) |
| 8 | Agents: *architect* for the helper API and policy resolver; *implementer* for collectors and pages; *code-reviewer* on every helper change; *scout* / *data-checker* for machine-profile JSON and template validation |
| 9 | Workflow: `/plan-first` for each milestone, `/build-test` before each commit, `/session-end` for logs |

---

## 15. Roadmap

The detailed plan (work packages, deliverables, acceptance criteria) is in [docs/milestones.md](docs/milestones.md).

| Milestone | Scope | Exit criteria |
| --- | --- | --- |
| **M0 Foundations** | Project init (§14), starter imported, theme tokens + CSS, icon set v1, `Gtk.Application` single instance | Starter runs with the Hot Cooling theme; lint/test pipeline green |
| **M1 Read-only monitor** | Collectors (hwmon, thermal zones, cpufreq, DRM fdinfo, NVML, power_supply, PPD, services), state store, Overview/Thermals/Hardware pages, FanRotor/ThermalGauge/Sparkline | Accurate readings on the FX506LI and one desktop (nct6775); < 2 % CPU; capability matrix correct |
| **M2 Helper + categories** | helperd (D-Bus, polkit, allow-lists, leases, watchdog), Idle/Optimal/Intense mapped to profile/EPP/fan-boost, Profiles and Fans pages | Category switching works and reverts on helper kill; fuzzed plan validation passes |
| **M3 Templates + rules** | YAML catalog + schemas, heat vectors, resolver, rule engine (AC/battery, GameMode, process classes, temperature thresholds), Automation page, "Why" feed | Reference policy (performance on AC, quiet on battery, gaming template) behaves as documented in `optimize-laptop-asus-fx50li/docs/fan-control.md` |
| **M4 Insight** | History DB, Benchmarks page (port of `bench-gpu-thermal.sh`), Power page with helper-provided RAPL | Per-template comparison report reproduces the manual benchmark |
| **M5 Advanced control** | Software fan curves (manual PWM machines), curve editor, RAPL PL1/PL2 caps, charge-pause, conflict integrations (TLP, asusd) | Tested on at least 3 machine classes |
| **M6 Release** | Debian packaging, AppStream, docs, translations scaffold, machine-profile contribution guide | `lintian` clean; installs/uninstalls cleanly on Debian 12, Ubuntu 24.04, Mint 22 |
| **M7 GTK 4 port** | Flip the version gate, replace `ui/compat.py` shims, optional libadwaita styling | Same feature set and tests pass on GTK 4; portability lint no longer needed |

---

## 16. Testing strategy

- **Unit:** collectors run against **recorded sysfs fixtures** (a copy of `/sys` subsets from real machines, starting with the FX506LI), plus the resolver, rules, hysteresis and schema validation.
- **Helper:** the D-Bus API runs on a private bus in CI with the dry-run actuator. Plan validation is property-tested/fuzzed (Hypothesis). A watchdog test kills the client and the helper.
- **UI:** smoke tests under `xvfb-run` (every page builds; no GTK warnings); frame-time sampling for animations; screenshot diffs for the theme.
- **Hardware in the loop:** scripted idle → load → cool-down runs per category (glmark2 / stress-ng), asserting temperature envelopes, fan response and zero unexpected throttle growth.
- **Distro matrix:** Debian 12, Ubuntu 24.04, Mint 22 (X11 + Wayland sessions) in VMs for packaging and D-Bus/polkit behavior.

---

## 17. Reference hardware: FX506LI capability snapshot (2026-09-16/17)

| Area | Observed | Impact on design |
| --- | --- | --- |
| Fans | One sensor (`asus` hwmon `fan1_input`); `pwm1_enable` accepts 2 (auto) and 0 (full) only; no curves | Proves the "profile + boost" degradation path (§4.3) |
| Profiles | `platform_profile`: quiet / balanced / performance (Fn+F5); power-profiles-daemon 0.21 maps power-saver → quiet | Category mapping through PPD, not direct writes |
| CPU | intel_pstate, EPP set; `package_throttle_count` in the tens of thousands per boot | Throttle-event deltas are a first-class metric |
| dGPU | NVIDIA 595 open, PRIME on-demand, **RTD3 not supported** (idles in P8) | `gpu.idle-leak` vector + PRIME advice |
| iGPU | Busy % available through DRM fdinfo without root | No `intel_gpu_top`/root dependency |
| Power | RAPL root-only; battery gives `current_now × voltage_now`; 80 % charge limit service | Helper-read watts; `power.charging` vector |
| Resume | Firmware resets some ASUS controls after sleep | Re-apply after resume (§13.8) |
| Tools | `bench-gpu-thermal.sh`, `test-steam.sh`, `fx506li-handoff.sh` | Seeds for the Benchmarks and Hardware pages |

---

## 18. Risks and open questions

| # | Risk / question | Mitigation / owner decision |
| --- | --- | --- |
| 1 | GTK 3 is in maintenance mode; GTK 4 + libadwaita is the modern path | **Decided 2026-09-17:** GTK 3 now, written for portability (§19); GTK 4 port is milestone M7 |
| 2 | Wide hardware variance and few writable controls on laptops | Capability-driven UI, machine profiles, honest "not available" states |
| 3 | Conflicts with PPD/TLP/asusd/thermald | Integration modes, conflict detection, never fight other managers |
| 4 | Privileged helper security | Minimal API, allow-lists, polkit, systemd sandboxing, reviews on every change |
| 5 | Animation cost on battery | Frame-clock animations, pause when hidden, reduced-motion mode |
| 6 | NVIDIA mobile GPUs rarely allow power-limit changes | Advisory actions (FPS cap, PRIME) where controls are missing |
| 7 | Naming/branding and license | **Decided 2026-09-17:** repo/package `lin-hot-cooling`, app ID `io.mensuramedia.LinHotCooling`; custom license (free use/modify/distribute, commercial use by written permission). The starter's "personal and educational use" terms are recorded in NOTICE; commercial permission may also be needed from its author |
| 8 | Repository | **Resolved:** https://github.com/MensuraMedia/lin-hot-cooling |
| 9 | Thermal Agent defaults and heat colors | **Decided 2026-09-17:** Suggest mode by default (Auto is opt-in); heat state is the default (collapsed) view, expandable to template and mode; heat levels are green / yellow / red; a panel indicator is included. Plan: [docs/milestones-thermal-agent.md](docs/milestones-thermal-agent.md) |

---

## 19. GTK 3 → GTK 4 portability rules

The app ships on GTK 3 (the starter template) but is written so that a GTK 4 port (milestone M7) is mostly mechanical.

| Area | Rule in GTK 3 code | GTK 4 equivalent it maps to |
| --- | --- | --- |
| App lifecycle | Use `Gtk.Application` + `Gtk.ApplicationWindow` (not `Gtk.main()`); actions through `Gio.SimpleAction` | Same API |
| Packing | Never call `pack_start`/`add` directly in pages; use `ui/compat.py` helpers (`append(box, child, expand=False)`, `set_child(container, child)`) | `Gtk.Box.append`, `set_child` |
| Visibility | No `show_all()`; widgets are visible by default via `compat.show()` | Widgets visible by default |
| Drawing | Custom widgets subclass `compat.CanvasArea`, which wraps the `draw` signal and exposes `on_draw(cr, w, h)` | `Gtk.DrawingArea.set_draw_func` |
| Input | Use event controllers available in GTK 3.24 (`Gtk.GestureMultiPress`, `Gtk.EventControllerMotion`, `Gtk.EventControllerKey`); no `button-press-event` handlers | `Gtk.GestureClick`, `Gtk.EventControllerMotion/Key` |
| Animation | Frame-clock tick callbacks (`add_tick_callback`) only | Same |
| Menus | `Gio.Menu` + `Gtk.Popover` (no `Gtk.Menu`, no `Gtk.StatusIcon`) | `Gtk.PopoverMenu` |
| Styling | CSS classes only; no `override_*` or `modify_*`; no `-gtk-gradient`; tokens via `@define-color` in one file | CSS custom properties / `@define-color`; libadwaita optional |
| Display access | `Gdk.Display` APIs; no `Gdk.Screen` or `Gtk.Window.set_position` in page code | `Gdk.Display`; window positioning is the compositor's job |
| Resources | Icons, CSS and UI files in a GResource bundle; symbolic icons from resource paths | Same |
| Threads | Workers never touch widgets; results go through `GLib.idle_add` | Same |
| Version gate | `gi.require_version('Gtk', os.environ.get('HC_GTK', '3.0'))` in one place (`src/gtk_version.py`) | Flip to `4.0` |

CI runs a **portability lint** (`tools/gtk4_lint.py`) that fails on banned GTK 3-only calls in `src/pages`, `src/ui` and `src/viewmodels`.

---

## 20. Structural principles: modularity and universality

The owner requires robust modularity and universality in structure, features and functions. These rules are binding for every milestone.

### 20.1 Layers and dependency direction

```
ui (pages, components)  →  viewmodels  →  core (state, scheduler, events)  ←  policy
                                              ↑                                  ↓
                                        providers (collectors)          helper_client  →  [D-Bus]  →  helper (actuators)
```

- Dependencies only point **inward**, toward `core`. `core` imports nothing from `ui`, `policy`, providers or `helper`.
- `ui` never touches sysfs, D-Bus or subprocesses. Providers never import GTK.
- The rules are enforced with **import-linter** contracts (`[tool.importlinter]` in `pyproject.toml`) in CI.

### 20.2 Everything pluggable through registries

| Extension point | Interface (`typing.Protocol`, in `core/interfaces.py`) | Registered via |
| --- | --- | --- |
| Sensor/capability providers | `Collector` (`probe()`, `sample()`, `cadence_ms`) | `hot_cooling.collectors` entry-point group + built-in registry |
| Actuators (helper side) | `Actuator` (`capabilities()`, `read()`, `apply(value, lease)`, `revert()`) | `hot_cooling.actuators` entry-point group |
| Pages | `Page` (`id`, `title`, `icon`, `order`, `build()`, `on_shown()`/`on_hidden()`) | `hot_cooling.pages`; the sidebar is generated from the registry, not hard-coded |
| Visual components | `Component` (+ `CanvasArea` base) | Normal imports; no global state |
| Rule triggers | `Trigger` (`subscribe(bus)`, `evaluate(state)`) | `hot_cooling.triggers` |
| Heat vectors, templates, machine profiles | Data files validated by JSON Schema | `data/` directories + `~/.config/lin-hot-cooling/` overlays |
| Themes | Token sets (`ThemeTokens` dataclass) | `config/themes/*.toml` |

Adding support for a new vendor, sensor, GPU, trigger or page means **adding a module and registering it**. Existing modules don't change.

### 20.3 Universality rules
- **No vendor or model names in core, policy or UI code.** Hardware specifics live only in providers and actuators, and in machine-profile data (`asus`, `thinkpad`, `nct6775`, …).
- **Capabilities, not models:** code asks "is `fan.boost` writable?", never "is this an ASUS?".
- **Units and IDs are normalized once** (in `core/units.py`, `core/ids.py`); the rest of the app works only with normalized values.
- **Data-driven behavior:** categories, templates, heat vectors, rules, thresholds and themes are data (YAML/TOML/JSON) with versioned schemas and migration functions.
- **Distro-agnostic within Debian family:** service detection goes through systemd/D-Bus APIs, not package names; paths come from `core/paths.py` (XDG, FHS).
- **Toolkit-agnostic core:** only `ui/` and `viewmodels/` know about GTK (and only `ui/compat.py` knows about the GTK version), which keeps the GTK 4 port and a possible CLI/TUI front end cheap.

### 20.4 Engineering practices
- **Dependency injection:** services (state, scheduler, helper client, clock, filesystem root) are passed in through an `AppContext`. Tests swap in fakes (fake sysfs root, fake clock, fake D-Bus).
- **Events over calls:** components communicate through an in-process event bus (`core/events.py`, typed events) and GObject signals at the UI edge.
- **Pure core logic:** the resolver, rule evaluation, hysteresis and easing are pure functions with property-based tests.
- **Small modules:** about 300 lines maximum per module; one responsibility each; module header docstring (starter convention).
- **Versioned contracts:** the D-Bus API (`…LinHotCooling1`), the plan schema and the data schemas carry versions, with compatibility tests.
- **Feature flags:** risky features (software curves, RAPL caps) sit behind `settings.experimental.*` until their milestone's acceptance criteria pass.

---

*Prepared from:*
- the gtk-python-dashboard-starter source (structure, BasePage, themes, layout);
- universal-instruction-set v2026.04 (process, immutability, registry);
- the `ui-ki-green-gray-black.jpg` palette (sampled);
- live measurements and findings from `~/projects/optimize-laptop-asus-fx50li`.
