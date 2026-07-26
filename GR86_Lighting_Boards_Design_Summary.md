# GR86 Lighting Boards — Combined Design Summary
*Last updated: July 2026 — merged from separate DRL and DemonEye design documents. Vehicle context, OEM reverse-engineering, and shared circuit topology consolidated into one location; board-specific sections reference back to it rather than duplicating. Two open architectural questions surfaced during the merge are flagged inline — see [Merge Notes / Open Questions](#merge-notes--open-questions) at the end.*

---

## Table of Contents

- [Project Overview](#project-overview)
- [Vehicle Context](#vehicle-context)
  - [OEM Electrical Measurements](#oem-electrical-measurements-direct-on-vehicle)
- [OEM Driver Board Reverse Engineering](#oem-driver-board-reverse-engineering)
  - [8-Pin Input Connector (Driver Board ↔ Vehicle Harness)](#8-pin-input-connector-driver-board--vehicle-harness)
  - [4-Pin Output Connector (Driver Board ↔ DRL LED Board)](#4-pin-output-connector-driver-board--drl-led-board)
  - [OEM Reverse Polarity Protection](#oem-reverse-polarity-protection)
  - [MAX16833C Thermal Parameters (OEM Thermal Anchor)](#max16833c-thermal-parameters-oem-thermal-anchor)
- [Shared Circuit Building Blocks](#shared-circuit-building-blocks)
  - [BCR421UFDQ / CSD25501F3 Constant-Current Channel Topology](#bcr421ufdq--csd25501f3-constant-current-channel-topology)
  - [QLSP06RGBXWAJ LED Package](#qlsp06rgbxwaj-led-package)
- [Part 1 — DRL Replacement Board](#part-1--drl-replacement-board)
  - [Power Architecture](#power-architecture-drl)
  - [Buck Converter Design](#buck-converter-design)
  - [LED Circuit Architecture (DRL)](#led-circuit-architecture-drl)
  - [Protection Components (DRL)](#protection-components-drl)
  - [Interface Connector (DRL)](#interface-connector-drl)
  - [PCB Specification (DRL)](#pcb-specification-drl)
  - [Key Design Decisions Log (DRL)](#key-design-decisions-log-drl)
  - [Complete BOM (DRL)](#complete-bom-drl)
- [Part 2 — DemonEye Board](#part-2--demoneye-board)
  - [Overview](#overview-demoneye)
  - [Power Architecture (DemonEye)](#power-architecture-demoneye)
  - [Constant Current Channel Design (DemonEye)](#constant-current-channel-design-demoneye)
  - [Thermal Management (DemonEye)](#thermal-management-demoneye)
  - [Component Summary / BOM (DemonEye)](#component-summary--bom-demoneye)
  - [Interface Connector (DemonEye)](#interface-connector-demoneye)
- [WLED / Control Architecture (Shared)](#wled--control-architecture-shared)
- [WLED Color Order & Cross-Board OUTx Wiring — RESOLVED (final)](#wled-color-order--cross-board-outx-wiring--resolved-final)
- [Open Items / Pending (Both Boards)](#open-items--pending-both-boards)
- [Merge Notes / Open Questions](#merge-notes--open-questions)
- [DemonEye Pre-Order Verification Checklist](#demoneye-pre-order-verification-checklist)
- [Reference Documents](#reference-documents)

---

## Project Overview

Two custom lighting boards for a 2024 Toyota GR86 MT Premium with Performance Package, both tapping power and (for DemonEye) control signals from the same OEM headlamp driver board connector:

1. **DRL Replacement Board** — replaces the OEM DRL (Daytime Running Light) board. 280mm × 34.5mm, 11 × QLSP06RGBXWAJ RGBW LEDs at 1040 mil (26.416mm) pitch (fixed by lens refractor geometry), driven by a WS2814B/BCR421UFDQ/CSD25501F3 architecture from an existing ESP32-S3/WLED system.
2. **DemonEye Board** — small supplementary decorative light. Aluminum-core PCB, target 15mm×15mm LED/mounting area (overall board closer to 15.5mm×35mm including connector), 3 × QLSP06RGBXWAJ RGBW LEDs wired in series per channel, driven by a single WS2814F/BCR421UFDQ/CSD25501F3 architecture, same WLED control system as DRL.

Both boards share the same LED part, the same constant-current regulation topology (BCR421UFDQ + CSD25501F3), and — per current understanding — the same vehicle power tap point. See [Shared Circuit Building Blocks](#shared-circuit-building-blocks) for the common topology, and [Merge Notes / Open Questions](#merge-notes--open-questions) for an open question about DemonEye's power tap that surfaced during this merge.

---

## Vehicle Context

- **Vehicle:** 2024 Toyota GR86 MT Premium with Performance Package
- **OEM DRL:** 11 white LEDs in series, driven by a boost converter on the headlamp driver board
- **Headlamp Driver IC:** MAX16833CAUE/V+ (Maxim/Analog Devices, 16-pin TSSOP-EP, frequency dithering C variant, AEC-Q100)
  - Datasheet: https://www.analog.com/media/en/technical-documentation/data-sheets/MAX16833-MAX16833G.pdf
- **OEM LED current:** ~307mA (calculated from 2× 1.3Ω sense resistors in parallel = 0.65Ω, VSENSE = 200mV, VICTRL > 1.4V confirmed) — see [OEM Electrical Measurements](#oem-electrical-measurements-direct-on-vehicle) for directly-measured values, which run lower than this calculated figure under default conditions
- **OEM boost converter inductor:** 2.2µH (marked "2R2" on driver board)
- **Power supply line:** Relay-switched 12-14.4V DC — confirmed not PWM at the supply level, not electronically dimmable via the supply itself (GR86 wiring data + Google search)
- **Wiring diagram confirms CLL runs straight from the vehicle fuse box (F/B)** — no upstream voltage clamping or regulation between the alternator/battery and this connector. A fuse protects against overcurrent only, not overvoltage. **Design implication: this line is exposed to genuine ISO 16750-2 load-dump transients; both boards' protection must be sized for that, not for a pre-clamped/reduced-voltage line.**
- **Headlamp enclosure:** Sealed plastic housing, ~70°C worst-case internal ambient, no conductive thermal path to external metal
- **Vertical clearance above PCB:** 8.64mm (measured) — applies to the DRL board's headlamp-enclosure location; DemonEye's mounting location/clearance not yet documented

### OEM Electrical Measurements (direct, on-vehicle)

| Measurement | Value | Condition |
|---|---|---|
| CLL pin voltage | 12.08V | Engine off |
| CLL pin voltage | 14.1V | Engine running |
| DRL pin voltage | 0.18V | LEDs ON (DRL mode active) |
| OEM LED string current | ~11.6mA (11.6mV across 1Ω shunt) | Default/low state (DRL pin not driven) |
| OEM LED string current | ~250mA | DRL pin pulled up to CLL potential |
| Voltage across all 11 LEDs (series) | 27.2V | Default/low state |
| Voltage across all 11 LEDs (series) | 32V | DRL pin pulled to CLL (full-brightness state) |
| Single LED Vf | 2.477V | Default/low state |
| Single LED Vf (calculated, 32V/11) | ~2.909V | Full-brightness state |
| LED+ wire gauge | 24AWG | — |

**Key finding — CLL is the actual power supply, DRL is a control/enable signal, not a power conductor:** CLL reads clean battery-range voltage in both engine states, consistent with an always-hot supply. DRL reads near-zero (0.18V) when DRL mode is active, and pulling DRL up to CLL potential increased LED current from ~11.6mA to ~250mA (close to the originally-calculated ~307mA sense-resistor figure) and brightened the LEDs visibly. This is consistent with DRL driving a transistor gate/base that selects between two current-reference states on the MAX16833C (a low/dim default vs. full DRL brightness) — the same pattern already documented for R+/R- → PWMDIM, just a second, separate control input. **The exact downstream circuit DRL feeds into is not confirmed** (traced to run under a metal shield toward the MAX16833C region; a wide, shielded trace from the input connector was also found in this area, more likely to be the CLL power path than DRL's signal path, but this hasn't been fully traced).

**Confirmed:** CLL remains active with headlights off (board and LEDs stayed powered) — consistent with, and necessary for, correct DRL function, since daytime running lights are meant to be on precisely when headlights are off.

**Practical implication:** the current/Vf figures above were read on a DC multimeter, which reports average current — if the MAX16833C internally PWM-dims the LED string (independent of the DC-clean CLL supply), these readings would not distinguish that from genuine continuous current. Not yet confirmed with a scope. Treat the ~250mA figure as consistent with, but not a precise replacement for, the original ~307mA sense-resistor calculation.

---

## OEM Driver Board Reverse Engineering

### 8-Pin Input Connector (Driver Board ↔ Vehicle Harness)

| Pin | Label | Function |
|-----|-------|----------|
| 1 | CLL | Main power relay — headlights. **Confirmed by direct measurement to be the actual battery-range power supply (12.08V rest / 14.1V running, present regardless of DRL state) ← 12V power tap for the DRL replacement board (corrected from DRL pin — see Vehicle Context / OEM Electrical Measurements)** |
| 2 | DRL | Main power relay — DRL. **Confirmed by direct measurement to be a low-current control/enable signal, not a power conductor** (reads 0.18V when DRL mode is active; not connected to the DRL replacement board) |
| 3 | GND | Ground |
| 4 | TRN | Turn signal input |
| 5 | GND | Ground |
| 6 | PO | Disconnection detect |
| 7 | LA | Side marker anode |
| 8 | LK | Side marker cathode |

### 4-Pin Output Connector (Driver Board ↔ DRL LED Board)

| Pin | Label | Function | Replacement Board |
|-----|-------|----------|-------------------|
| 1 | LED+ | Boost converter output (~35V CC) | **Not connected** |
| 2 | LED- | LED string return (common with R-) | **Not connected** |
| 3 | R+ | PWMDIM control circuit | **Not connected** — see note below |
| 4 | R- | Common ground with LED- | **Not connected** |

**R+/R- Circuit Finding:** Tracing confirmed R+ connects through a 2kΩ → 20kΩ (R10) network to a transistor, then via 15kΩ (R17) to the PWMDIM pin of the MAX16833C. PWMDIM has a 75kΩ pull-down to GND. The OEM 1.3kΩ resistor across R+/R- raises PWMDIM to enable the boost converter.

**Key Decision:** The 1.3kΩ resistor is **intentionally omitted** from the DRL replacement board. This holds PWMDIM low via the 75kΩ pull-down, permanently disabling the MAX16833C boost converter and preventing any ~35V output from the LED+ pin.

**LED- and R-** are internally connected on the driver board — confirmed by continuity check.

### OEM Reverse Polarity Protection

The OEM DRL board uses a **S4PJ-style** (Vishay, 4A rectifier, TO-277A/SMPC flat leadless package, AEC-Q101) part in series with the LED+ line for reverse polarity protection, read from the physical part marking. This is replicated on the DRL replacement board on the Vbat input line using **AS4PJHM3_A/H** — confirmed as the correct orderable Vishay part number from the KiCad schematic (D2), superseding the earlier informal "S4PJHM3_B" reference derived from the marking.

### MAX16833C Thermal Parameters (OEM Thermal Anchor)

| Parameter | Value |
|-----------|-------|
| Tj max | 150°C |
| Thermal shutdown | 160°C (resumes at 150°C) |
| RθJA (16-pin TSSOP-EP) | 38.3°C/W |
| Continuous Pd at 70°C ambient | 2089mW |
| Architecture | Controller + external FET + external inductor (distributed dissipation) |

**Significance:** This is an existence proof that a 150°C Tj part with distributed dissipation architecture can operate reliably in this exact sealed headlamp enclosure — validating both the LMR51460F-Q1 thermal target (DRL board) and the external inductor architectural decision.

---

## Shared Circuit Building Blocks

Both boards use the same LED part and the same constant-current channel topology. This section documents that shared design once; both board-specific sections below reference it rather than repeating it.

### BCR421UFDQ / CSD25501F3 Constant-Current Channel Topology

```
Rail ─────────────────────────────────────────┐
                                                │
                                          LED Anode (×4 channels)
                                                │
                                          LED Cathode
                                                │
                                     BCR421UFDQ OUT pin
                                                │
                                          Rext to GND
                                     BCR421UFDQ GND ─── GND

WS2814x OUTx ──── CSD25501F3 Gate
Rail ──────────── CSD25501F3 Source (internal 10kΩ RC clamp pulls Gate to Source when OUTx = Hi-Z)
                  CSD25501F3 Drain ──── BCR421UFDQ EN pin
```

**Signal inversion logic:**
- WS2814x OUTx **sinks** ~1.2mA → Gate pulled to 0V → Vgs = -(rail voltage) → PFET **ON** → EN pulled high → LED current flows
- WS2814x OUTx **Hi-Z** → 10kΩ RC pulls Gate to Source (rail voltage) → Vgs = 0V → PFET **OFF** → EN = 0V → LED off
- WS2814x needs to sink ~1.2mA (rail/10kΩ RC) — well within 16.5mA sink capability ✅ (confirmed for both WS2814B on DRL and WS2814F on DemonEye — see [Part 2](#constant-current-channel-design-demoneye) for WS2814F-specific VDD/sink-current confirmation)
- CSD25501F3 Vgs(th) = -0.75V typ, -1.05V max — rail-level Vgs on both boards provides large margin ✅
- CSD25501F3 Vds max = -20V — both boards' rail voltages (5.4V on DRL, 12V on DemonEye) provide adequate margin ✅

**BCR421UFDQ REXT formula:**
$$R_{EXT} = \frac{R_{INT} \times R_{parallel}}{R_{INT} - R_{parallel}}, \quad R_{parallel} = \frac{V_{DROP}}{I_{OUT}}$$

Where $R_{INT}$ = 95Ω typ, $V_{DROP}$ = 0.95V typ (datasheet spec, 10mA test condition — used here specifically for the REXT-setting formula, which is a different application than headroom/margin checking below).

**BCR421UFDQ headroom/margin checking — separate spec, do not confuse with the above:** the applicable spec for checking whether a channel has enough voltage headroom to stay in regulation at real operating currents ($I_{OUT}$ > 18mA, true for every channel on both boards) is $V_{OUT(MIN)}$ = 1.4V typ — **not** the 0.95V $V_{DROP}$ figure used in the REXT formula above. These are two distinct datasheet specs used for two distinct purposes; see [Merge Notes / Open Questions](#merge-notes--open-questions) for a flag on making sure this distinction is applied consistently everywhere it's used.

### QLSP06RGBXWAJ LED Package

| Parameter | Value |
|-----------|-------|
| Manufacturer | QueLighting |
| Package | 3030 ceramic substrate, 3.0×3.0×1.8mm |
| Max current | 300mA DC per channel |
| Max junction temp | 120°C |
| View angle | >120° |
| Total Power Dissipation | 3600mW @ Ta=25°C (used for $R_{th(JA)}$ derivation — see [DemonEye Thermal Management](#thermal-management-demoneye)) |

**Electrical Specifications (at 250mA test condition, Ta=25°C):**

| Channel | Vf Typ (V) | Vf Min (V) | Vf Max (V) | Flux Typ (lm) | Wavelength |
|---------|-----------|-----------|-----------|--------------|-----------|
| Red | 2.2 | 2.0 | 2.6 | 37 | 620-625nm |
| Green | 3.2 | 2.8 | 3.6 | 52 | 520-530nm |
| Blue | 3.2 | 2.8 | 3.6 | 10.4 | 450-460nm |
| White | 3.2 | 2.8 | 3.6 | 50 | 2700-7000K |

---

## Part 1 — DRL Replacement Board

### Power Architecture (DRL)

**Power Tap Source:** CLL pin (pin 1) on 8-pin input connector → replacement board Vbat input.

**Corrected from an earlier plan to tap the DRL pin (pin 2).** Direct electrical measurement showed DRL reads near-zero voltage (0.18V) when DRL mode is active and is not a power conductor — see [Vehicle Context / OEM Electrical Measurements](#oem-electrical-measurements-direct-on-vehicle). CLL is confirmed as clean, battery-range voltage present regardless of DRL state, and per the vehicle wiring diagram runs straight from the fuse box with no upstream clamping — so it carries full ISO 16750-2 transient/load-dump exposure, same as any raw battery-derived automotive line.

**Do NOT tap LED+** — the MAX16833C is a constant-current boost converter source. A CC source driving a CV buck converter input causes fundamental control loop incompatibility and potential device damage.

**CLL line protection chain (in order from connector):**
1. **Reverse polarity:** AS4PJHM3_A/H series rectifier (4A, AEC-Q101, TO-277A)
2. **Transient suppression:** Vishay T15B22AHM3/I TVS (1500W, 18.8V standoff, 30.6V clamp, AEC-Q101, DO-221AA, 185°C Tj)
3. **High-frequency bypass:** 100nF/50V X7R 0402
4. **Bulk input capacitance:** 2× Samsung CL32B226KAJNNNE (22µF/25V/X7R/1210)
5. **Buck converter:** LMR51460F-Q1

**Current Budget (All Channels, 100% PWM):**

| Channel | Current per LED | × 11 | Total |
|---------|----------------|------|-------|
| White | 247.5mA | × 11 | 2.723A |
| Red | 83.0mA | × 11 | 0.913A |
| Green | 83.0mA | × 11 | 0.913A |
| Blue | 83.0mA | × 11 | 0.913A |
| WS2814B quiescent | ~5mA | × 11 | 0.055A |
| **Total** | | | **5.517A** |

**12V input current at 95% efficiency:**
$$I_{12V} = \frac{5.4V \times 5.517A}{0.95 \times 12V} = 2.62A$$

**Wire sizing:** 18AWG minimum. **Fuse:** 5A.

### Buck Converter Design

**Component Selection:**

| Component | MPN | Manufacturer | Key Specs |
|-----------|------------|--------------|-----------|
| Buck IC | LMR51460F-Q1 | Texas Instruments | AEC-Q100, 4-36V in, 6A, WSON-12, FPWM, 150°C Tj, 160°C TSD |
| Inductor | Wurth 744393465120 | Wurth Elektronik | 12µH ±20%, AEC-Q200, 6.6×6.4×6.1mm, 8.9A Irms, 9.8A Isat(30%), 22.8mΩ DCR typ, -55°C to +150°C |

**Why LMR51460F-Q1:**
- WEBENCH thermal at 70°C ambient: Tj ≈ 106°C — adequate margin to 150°C limit
- TPSM843620E (prior candidate) exceeded its 125°C Tj limit at same conditions (Tj ≈ 135°C)
- External inductor separates major heat source from IC, distributing dissipation across more aluminum PCB area
- FPWM mode: fixed 200kHz spectrum, simpler EMI filtering vs PFM variable frequency
- **Note:** No frequency dithering/spread spectrum (unlike OEM MAX16833CAUE/V+). Accepted for one-off build with no EMC certification target. If switching noise causes practical issues, evaluate LM61460-Q1 (DRSS-capable) as replacement

**Why Wurth 744393465120 (12µH):**
- Fits within 8.64mm headlamp clearance (6.1mm tall vs 8.0mm for Coilcraft XAL8080) ✅
- 6.6×6.4mm footprint vs 8.0×8.0mm for Coilcraft ✅
- -55°C to +150°C operating range vs -40°C to +125°C for Coilcraft ✅
- Coilcraft XAL8080-123MED has lower DCR (16.4mΩ vs 22.8mΩ) and runs ~4.3°C cooler at operating current, but height margin of only 0.44mm after solder is unacceptable
- 12µH gives 27.3% ripple at 200kHz — within 30% target ✅

**Inductor ripple current verification:**
$$\Delta I_L = \frac{V_{OUT}(V_{IN} - V_{OUT})}{V_{IN} \times f_{SW} \times L} = \frac{5.4 \times (16-5.4)}{16 \times 200kHz \times 12\mu H} = \frac{57.24}{38.4} = 1.490A = 27.3\%$$

**Buck Converter Configuration:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Vin | 12-16V (TVS clamped) | From CLL pin |
| Vout | 5.4V | Set by FB divider |
| Iout max | 5.517A | Within 6A rating |
| Switching frequency | 200kHz | Minimum frequency = minimum switching losses = maximum thermal margin |
| Estimated efficiency | ~95% | At 200kHz vs 93.79% at 241kHz from WEBENCH |
| Rfbt | 1kΩ (E96, 1%) — YAGEO AC0402FR-071KL | Low impedance for single-layer PCB noise immunity |
| Rfbb | 174Ω (E96, 1%) — YAGEO AC0402FR-07174RL | Vout = 0.8V × (1 + 1k/174) = 5.40V |
| Rt | 71.5kΩ (E96, 1%) — Ever Ohms Tech CR0402F71K5Q10Z | Sets 200kHz per LMR51460F-Q1 datasheet Table 7-1 |
| Cboot | 100nF/50V X7R 0402 — Samsung CL05B104KB54PNC | Required external bootstrap cap between BOOT and SW pins; same part as Cinx (schematic confirms 50V-rated part used, not the originally-specified 16V Murata part) |
| Cff | Not fitted | Not discussed in LMR51460F-Q1 datasheet; absent from WEBENCH BOM; at Rfbt=1kΩ the zero frequency would be >4MHz — no benefit |
| PGOOD | GND or float | Datasheet: "If not used, PG can be left open or connected to GND." No pullup fitted. Tie to GND if reachable by direct trace; float otherwise |

**FB divider rationale:** VREF = 0.8V. Scaled down 10× from WEBENCH's 100kΩ/16.9kΩ to 10kΩ/1.69kΩ, then further to 1kΩ/174Ω for single-layer PCB noise immunity. Maintains exact WEBENCH ratio. Vout = 5.40V.

**Buck Converter Capacitors — Input:**

| Designator | Value | Voltage | Dielectric | Package | Qty | MPN | Purpose |
|------------|-------|---------|------------|---------|-----|-------------|---------|
| Cin | 22µF | 25V | X7R | 1210 | 2 | Samsung CL32B226KAJNNNE | Bulk input capacitance |
| Cinx | 100nF | 50V | X7R | 0402 | 1 | Samsung CL05B104KB54PNC | HF bypass — suppress switching transients from escaping onto 12V rail; place as close as possible to VIN pin |

**Input cap note:** 25V is the maximum available voltage rating for 22µF MLCC in 1210 package (physical dielectric limitation). At 16V on 25V rated cap = 64% voltage stress with ~50% derating → ~11µF effective per cap → ~22µF total. Adequate — input ripple irrelevant to LED output due to buck regulation + BCR421 CCR double isolation.

**Buck Converter Capacitors — Output:**

| Designator | Value | Voltage | Dielectric | Package | Qty | MPN | Location |
|------------|-------|---------|------------|---------|-----|-------------|----------|
| Cout_local | 22µF | 25V | X7R | 1210 | 1 | Samsung CL32B226KAJNNNE | Adjacent to buck IC |
| Cout_dist | 10µF | 25V | X7R | 0805 | 11 | Samsung CL21B106KAYQNNE* | One per LED circuit |
| Cboot | 100nF | 50V | X7R | 0402 | 1 | Samsung CL05B104KB54PNC | Between BOOT and SW pins — same part as Cinx, confirmed via schematic net trace (SW–BOOT node), not the 16V Murata part previously specified |

*Samsung CL21B106KAYQNNE: Not in Samsung's active catalog but confirmed via Samsung's dynamic datasheet system. JLCPCB has 280,000 units in stock. Confirmed specs: 10µF, 25V, X7R, 0805.

**Output cap rationale:** At 5.4V on 25V rated cap = 22% voltage stress → ~5-10% derating → ~20µF effective locally + ~99µF effective distributed = ~119µF total. Well above minimum required ~14µF at 200kHz. Distributed caps placed at each LED circuit reduce IR drop sensitivity across 280mm board length.

**Samsung CL32B226KAJNNNE note:** NNN variant confirmed identical specs to F (power application) variant — only difference is position 9 product classification code. NNN is the only variant in stock at JLCPCB.

**Buck Circuit Layout Guidelines:**

- **Available zone:** ≤15mm (width) × 30mm (height)
- **Inductor width:** 6.6mm — 0.7mm closer to adjacent LED circuits vs original 8mm constraint; acceptable given shielded inductor + linear BCR421 regulators are insensitive to small induced voltages
- **Vertical stack order (top to bottom, connector at bottom):**
  1. FB resistors (Rfbt, Rfbb)
  2. Cout_local (22µF 1210)
  3. Cboot (0402) | Inductor (6.6×6.4mm) — side by side
  4. Buck IC (WSON-12) | PGOOD connection — side by side
  5. Cinx (0402) | Rt (0402) — side by side
  6. Cin (2× 22µF 1210)
- **FB trace routing:** Route down the LED-circuit side of the stack (maximum distance from inductor switching node). Flank with dedicated Vcc/GND guard traces (~0.15mm). Maximum air-gap to inductor on the other side.

**Buck Thermal Analysis:**

LMR51460F-Q1 at 70°C ambient (WEBENCH, conservative):
- IC Pd = 1.648W (at 241kHz — actual at 200kHz will be slightly less)
- RθJA on aluminum PCB estimated 18-22°C/W
- Tj ≈ 70 + (1.648 × 22) = **106°C** — 44°C margin to 150°C limit ✅
- Target copper pour: ~0.5 in² around buck IC

### LED Circuit Architecture (DRL)

Per-channel topology, signal inversion logic, and the BCR421UFDQ REXT formula are documented once in [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology) — this section covers only DRL-specific values.

**Current Setting (BCR421UFDQ Rext), using $R_{INT}$=95Ω, $V_{DROP}$=0.95V typ:**

| Channel | Target | Rext | Actual Current | BCR421 Pd worst case |
|---------|--------|------|---------------|---------------------|
| White | 247.5mA | 4.02Ω | 247.5mA | (5.4-2.8)×247.5mA = 644mW |
| Red | 83.0mA | 13.0Ω | 83.0mA | (5.4-2.0)×83mA = 282mW |
| Green | 83.0mA | 13.0Ω | 83.0mA | (5.4-2.8)×83mA = 216mW |
| Blue | 83.0mA | 13.0Ω | 83.0mA | (5.4-2.8)×83mA = 216mW |

All within BCR421UFDQ 1.7W package rating ✅. BCR421 NTC characteristic naturally reduces current as temperature rises — accepted graceful derating behavior. White channel at Vf=2.8V worst case is the thermal concern.

**Use 1% tolerance resistors.**

**RGB channel rationale:** 11×3×83mA = 2.739A ≈ white total 2.723A — equal loading on both halves of the 5.4V power budget, balanced thermal distribution across board.

**Per-Package BOM (×11 total packages):**

| Component | MPN | Qty |
|-----------|------------|-----|
| RGBW LED | QLSP06RGBXWAJ | 1 |
| LED Controller | WS2814B | 1 |
| CCR | BCR421UFDQ-7 | 4 |
| PFET | CSD25501F3 | 4 |
| Rext White (schematic labeled 4Ω; calculated target 4.02Ω) 1% | **TBD** — no MPN assigned in schematic | 1 |
| Rext RGB 13.0Ω 1% | **TBD** — no MPN assigned in schematic | 3 |
| Internal regulator series resistor (R5, WS2814B VDD pin) | **TBD** — no MPN assigned in schematic; value 100Ω | 1 |
| Decoupling cap (C1, WS2814B internal regulator node to GND) | **TBD** — no value or MPN assigned in schematic (0402 footprint only) | 1 |
| Cout distributed 10µF/25V X7R 0805 | Samsung CL21B106KAYQNNE | 1 |

**Newly identified from schematic (not previously in this document):** each LED circuit includes an R5 (100Ω)/C1 network on the WS2814B VDD pin — R5 is a series resistor limiting voltage/current into the WS2814B's internal voltage regulator, with C1 as a decoupling cap on the regulator node. R5's value (100Ω) is set in the schematic but has no manufacturer/part number assigned; C1 has neither a value nor a part number assigned (still shows the KiCad library placeholder). Both need to be resolved before layout/ordering.

**Board Totals (11 packages):**

| Component | Total Qty |
|-----------|----------|
| QLSP06RGBXWAJ | 11 |
| WS2814B | 11 |
| BCR421UFDQ | 44 |
| CSD25501F3 | 44 |
| Rext 4.02Ω (schematic labeled 4Ω) | 11 |
| Rext 13.0Ω | 33 |
| WS2814B internal regulator series resistor (R5, 100Ω) | 11 |
| WS2814B internal regulator decoupling cap (C1, value TBD) | 11 |
| Cout distributed 10µF/25V 0805 | 11 |

### Protection Components (DRL)

| Component | MPN | Manufacturer | Function | Key Specs |
|-----------|------------|--------------|----------|-----------|
| Reverse polarity | AS4PJHM3_A/H | Vishay | Series rectifier on Vbat | 4A, AEC-Q101, TO-277A, same as OEM |
| TVS | T15B22AHM3/I | Vishay | Input transient suppression | 1500W, VRWM=18.8V, VC=30.6V, AEC-Q101, DO-221AA, 185°C Tj |
| ESD (data line) | SZESD7241N2T5G | onsemi | WS2814B data input ESD protection | AEC-Q101, VRWM=24V, CJ=1.0pF max |

**TVS Selection Notes:**
- Clamp voltage 30.6V provides 5.4V margin below LMR51460F-Q1 absolute max VIN = 36V ✅
- 185°C Tj max — highest available in this class ✅
- Available via DigiKey marketplace order through JLCPCB ✅
- AEC-Q101 /I suffix confirmed ✅

**Topology Verification — D1 (rectifier) / D2 (TVS) Series Order:**

D1 is placed upstream of D2 (connector → D1 → node → D2 shunt to GND). This order is deliberate and required for reverse-polarity protection to function:

- **Reverse-battery fault path:** with the connector reversed, board GND carries the reversed battery potential. Current flows GND → D2 anode → D2 (forward-biased, conducts like a standard diode) → node → D1 cathode, where it is **blocked** by D1 (reverse-biased looking back toward the connector). D1 is what breaks the fault loop; if the TVS were placed upstream of D1 instead, there would be no diode in the path to block reverse conduction at all.
- **Consequence — normal-polarity load dump:** because D2 sits downstream of D1, a load-dump transient on Vbat passes *through* D1 (forward-biased, conducting) on its way to D2's clamp action. D1 must survive the full TVS clamp current, not just steady-state forward current.
- **Verified, not just assumed:**
  - T15B22A rated peak pulse surge current (IPPM, 10/1000µs waveform): **50.7A** (cross-checked: PPPM/VC = 1500W/30.6V ≈ 49A, consistent)
  - AS4PJHM3 rated surge current (IFSM, S4P-series family datasheet): **100A @ 10ms single half-sine**
  - D1 comfortably exceeds the ~51A it needs to pass, with the actual pulse duration (1ms, per TVS's 10/1000µs waveform) 10x shorter than the diode's rated 10ms test condition — less thermal energy than the diode's rated point, not more.
  - Not yet done: rigorous I²t energy comparison (rather than peak-current-vs-peak-current) — datasheet doesn't give AS4PJHM3 a surge curve at exactly 1ms, so this is a reasonable-margin check, not a fully rigorous one.

### Interface Connector (DRL)

DRL board uses a **DuraClik 2.00mm 1×04** header. The data pin (pin 5) has no mating socket in the OEM harness connector.

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | DATA | WS2814B data signal from WLED/ESP32-S3 |
| 2 | GND | Data signal GND |
| 3 | GND | Power Return Path |
| 4 | $\text{+V}_{BAT}$ | 12-16V, 3A, tapped from CLL pin on driver board |

### PCB Specification (DRL)

| Parameter | Value |
|-----------|-------|
| Layers | 1 (single-layer aluminum PCB) |
| Substrate | Aluminum 1060, 1.6mm |
| Copper weight | 1oz |
| Dielectric thermal conductivity | 1 W/mK |
| Dielectric thickness | 0.11-0.15mm |
| Manufacturer | JLCPCB |
| Max component height | 8.64mm (measured headlamp clearance) |
| Component strategy | MLCCs only — no electrolytic caps |

### Key Design Decisions Log (DRL)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| LED current — White | 247.5mA (4.02Ω Rext) | Below 250mA rated; thermal headroom; matches RGB total for balanced buck loading |
| LED current — RGB | 83.0mA (13.0Ω Rext) | Equal loading to white — 2.739A vs 2.723A |
| Supply voltage | 5.4V (Rfbt=1kΩ, Rfbb=174Ω) | BCR421 worst case: 5.4-3.6=1.8V headroom >> 1.4V min; lower than 5.5V reduces BCR421 dissipation |
| FB divider impedance | 1kΩ/174Ω | Low impedance for single-layer PCB noise immunity per LMR51460 datasheet guidance |
| Buck IC | LMR51460F-Q1 | 150°C Tj — adequate at 70°C ambient (Tj≈106°C); TPSM843620E failed thermal at same conditions |
| Buck frequency | 200kHz (Rt=71.5kΩ) | Minimum frequency = minimum switching losses = maximum thermal margin; ~95% efficiency |
| Inductor | Wurth 744393465120 (12µH) | Fits 8.64mm clearance (6.1mm tall); AEC-Q200; -55°C to +150°C; 27.3% ripple within 30% target |
| Cff | Omitted | Not discussed in LMR51460 datasheet; absent from WEBENCH; zero benefit at Rfbt=1kΩ |
| PGOOD | GND or float | Explicitly permitted by datasheet; no pullup; prevents floating pin antenna effect if GND reachable |
| Output cap — local | 1× 22µF/25V X7R 1210 | Single cap sufficient with distributed caps at point of load |
| Output cap — distributed | 11× 10µF/25V X7R 0805 | Point-of-load bulk; reduces IR drop across 280mm board |
| Input caps | 2× 22µF/25V X7R 1210 | 25V max available for 22µF 1210 MLCC; input ripple irrelevant to LED output |
| Bulk cap part | Samsung CL32B226KAJNNNE | Single part for Cin and Cout_local; NNN variant = identical specs to F (power) variant |
| 1.3kΩ R+/R- resistor | Omitted | Deliberately disables MAX16833C via PWMDIM pull-down |
| Power tap | CLL relay pin (8-pin connector pin 1) — **corrected from DRL pin (pin 2)** | DRL confirmed by direct measurement to be a low-current control/enable signal (0.18V when active), not a power conductor; CLL confirmed clean battery-range voltage (12.08V/14.1V) regardless of DRL state. CC→CV incompatibility still prevents LED+ tap |
| PCB type | Single-layer aluminum | Thermal management; no height for tall components |
| Per-LED bypass caps | Omitted | BCR421 is linear — no switching noise; OEM caps were for boost converter switching |
| TVS | Vishay T15B22AHM3/I | AEC-Q101, 185°C Tj, 30.6V clamp (5.4V below LMR51460 36V max), marketplace via JLCPCB |
| Reverse polarity | AS4PJHM3_A/H | AEC-Q101, same package/specs as OEM — proven in this application; corrected from initial "S4PJHM3_B" read off OEM part marking |
| ESD data line | SZESD7241N2T5G | AEC-Q101, 1.0pF max — minimal data signal impact |

### Complete BOM (DRL)

*Regenerated from `GR86_DRL.kicad_sch`, `5V5_Buck_Supply.kicad_sch`, and `WS2814_RGBW_LED.kicad_sch` (net-level trace, not just symbol Value fields) — July 2026. LCSC codes included where present in the schematic component fields.*

**Protection & Power Input:**

| Component | MPN | Manufacturer | LCSC | Qty |
|-----------|------------|--------------|------|-----|
| Reverse polarity rectifier | AS4PJHM3_A/H | Vishay | C6125239 | 1 |
| TVS diode | T15B22AHM3/I | Vishay | C20038291 | 1 |
| ESD protection (data) | SZESD7241N2T5G | onsemi | C604770 | 1 |

**Buck Converter:**

| Component | MPN | Manufacturer | LCSC | Qty |
|-----------|------------|--------------|------|-----|
| Buck IC | LMR51460FSQDRRRQ1 | Texas Instruments | C48573057 | 1 |
| Inductor | 744393465120 | Wurth Elektronik | C49303300 | 1 |
| Rfbt 1kΩ 1% 0402 | AC0402FR-071KL | YAGEO | C144789 | 1 |
| Rfbb 174Ω 1% 0402 | AC0402FR-07174RL | YAGEO | C226815 | 1 |
| Rt 71.5kΩ 1% 0402 | CR0402F71K5Q10Z | Ever Ohms Tech | C2835750 | 1 |
| Cboot 100nF/50V X7R 0402 | CL05B104KB54PNC | Samsung | C307331 | 1 |
| Cin 22µF/25V X7R 1210 | CL32B226KAJNNNE | Samsung | C309062 | 2 |
| Cinx 100nF/50V X7R 0402 | CL05B104KB54PNC | Samsung | C307331 | 1 |
| Cout_local 22µF/25V X7R 1210 | CL32B226KAJNNNE | Samsung | C309062 | 1 |

**Corrections vs. prior BOM:** Rfbt, Rfbb, Rt, and Cinx were previously listed as TBD — all four are populated components in the schematic. Cboot was previously specified as a Murata GRM155R71C104KA88D (16V); the schematic shows the same Samsung CL05B104KB54PNC (50V) part used for both Cboot and Cinx. Cin/Cout_local part number typo corrected (CL32B226KAJNNN**N**E → CL32B226KAJNNNE, one fewer "N").

**LED Circuits (×11 packages):**

| Component | MPN | Manufacturer | LCSC | Qty (total) |
|-----------|------------|--------------|------|-------------|
| RGBW LED | QLSP06RGBXWAJ | Quelighting | C17645191 | 11 |
| LED controller | WS2814B | Worldsemi | C22371542 | 11 |
| CCR | BCR421UFDQ-7 | Diodes Incorporated | C2678624 | 44 |
| PFET | CSD25501F3 | Texas Instruments | C2872196 | 44 |
| Rext White (target 4.02Ω, 1%, 0402) | **TBD** — no MPN in schematic | — | — | 11 |
| Rext RGB (13.0Ω, 1%, 0402) | **TBD** — no MPN in schematic | — | — | 33 |
| WS2814B internal regulator series resistor (100Ω, 0402) | **TBD** — no MPN in schematic | — | — | 11 |
| WS2814B internal regulator decoupling cap (0402) | **TBD** — no value or MPN in schematic | — | — | 11 |
| Cout_dist 10µF/25V X7R 0805 | CL21B106KAYQNNE | Samsung | C3039694 | 11 |

**Corrections vs. prior BOM:** Cout_dist part number typo corrected (CL21B106KAYQNN → CL21B106KAYQNNE, trailing "E" restored). Two previously-undocumented components identified per LED circuit: a 100Ω series resistor limiting voltage/current into the WS2814B's internal voltage regulator, and a decoupling capacitor on that regulator node — both still need parts selected.

**Interface:**

| Component | MPN | Manufacturer | LCSC | Qty |
|-----------|------------|--------------|------|-----|
| Board connector header, 1×04 | 560020-0420 | Molex (DuraClik 2.00mm) | C277592 | 1 |
| Mating receptacle housing, 1×04 | 502351-0400 | Molex (DuraClik 2.00mm) | — | 1 |
| Crimp contact (tin, 22–26AWG) | 560085-0101 | Molex (DuraClik 2.00mm) | — | 4 |

**Corrections vs. prior BOM:** "1×05" typo corrected to 1×04 (4-circuit header, matching the DRL/GND/GND/+VBAT pinout in [Interface Connector (DRL)](#interface-connector-drl)). Actual header, receptacle, and contact part numbers now included — the header itself is CN1 in the schematic; receptacle and contact are the harness-side mating parts (not schematic symbols, carried over from prior sourcing notes).

---

## Part 2 — DemonEye Board

### Overview (DemonEye)

This module is a compact aluminum-core PCB for automotive DemonEye applications. Board target is 15mm×15mm for the LED/mounting area, open to variation depending on final fit; overall board is closer to 15.5mm×35mm, with the additional ~5mm of length dedicated to the connector. Physical constraint: 16mm×20mm must hold the WS2814F, 4× BCR421, 5× CSD25501F3, 3× LED packages, and a 5- or 6-pin picoBlade header.

**DemonEye is an optional data pass-through board.** It has both a data-in and a data-out, allowing it to be inserted into the WS281x data chain ahead of the DRL board: WLED controller → DemonEye DIN → (WS2814F processes/forwards) → DemonEye DOUT → DRL board DIN. This is why DemonEye's connector needs more pins than DRL's (DRL is the last device in the chain and only needs DATA in, not a DOUT) — see [Interface Connector (DemonEye)](#interface-connector-demoneye).

The module uses a **TPS7A19** (SON-8, adjustable, wide-VIN) LDO to provide a stable, known regulated rail, and a **single WS2814F** (WS2814B ruled out — package too large for this board) driving **4 independent constant-current color channels (R, G, B, W)**. Each channel is formed by **3 RGBW LED packages wired in series** (same color across all 3 packages), rather than 3 independently-addressable packages — the 3 packages are treated as a single 4-color light source, not 3 separate light sources. Only one WS2814F is needed since a series-wired string can't be independently addressed per package anyway.

**Design rationale (series LED packages):** the LDO provides a stable, known voltage to the LED string. Stacking 3 LED packages per channel in series drops the bulk of the excess voltage across the LEDs themselves (distributing heat across 3 packages) rather than having the LDO dissipate it all. The BCR421UFDQ continues to regulate the actual channel current — same shared topology as the DRL board; see [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology).

**Addition vs. DRL board:** a master P-FET, configured as a high-side switch (HSS), gates power to the entire LED circuit. Gate is pulled low by default (FET on, LEDs powered); a CLL (headlight) power signal from the OEM driver board pulls the gate high, turning the FET off and cutting power to the LEDs whenever the headlights are on. **See [Merge Notes / Open Questions](#merge-notes--open-questions) — this mechanism needs to be re-examined now that CLL is confirmed to be the shared power-tap net, not a separate always-distinguishable "headlights on" signal.**

### Power Architecture (DemonEye)

- **Input:** Originally documented as "Same OEM DRL feed as the DRL board," valid automotive $\text{V}_{bat}$ (11.8V–15.6V), clamped by an onboard TVS diode. **This description predates the CLL/DRL pin correction documented in [Vehicle Context](#vehicle-context) and the extended TVS/LDO voltage-margin investigation — DemonEye's actual power tap point and protection scheme have not been finalized to match. See [Merge Notes / Open Questions](#merge-notes--open-questions).**
- **Regulation:** TPS7A19 (SON-8/DRB package, 3.00×3.00mm, adjustable output, 450mA rated) LDO.
    - **Target Output ($\text{V}_{out}$):** 12V, set via feedback divider ($V_{FB}$ = 1.233V typ, R1+R2 constrained to 10–100kΩ per datasheet): $R_2$ = 10kΩ (nominal, per datasheet's own worked example), $R_1$ = $R_2 \times (\frac{V_{out}}{V_{FB}} - 1)$ = 87.3kΩ. $R_1+R_2$ = 97.3kΩ, just under the 100kΩ ceiling.
    - **Input range:** 4–40V continuous (recommended operating), abs max 45V (IN/EN pins). At 400mA total load, dropout is 240mV typ / 450mV max — so minimum practical $V_{in}$ for full regulation is ~12.24–12.45V. At engine-off (CLL = 12.08V), the part is in dropout — same accepted mild-dimming behavior as previously planned for other LDO candidates.
    - **Load-dump exposure — resolved by continuous input range, not by a shutdown mechanism:** unlike the earlier LM2940-N candidate (which survives load dump by disabling its output via OVSD around ~30V), TPS7A19's 40V continuous rating already covers the ~35V suppressed load-dump ceiling directly — the datasheet states the part *"withstands and maintains regulation during voltage transients by acting as a simple surge protection circuit."* LEDs should stay lit through a transient rather than going dark, which is a better outcome for this application than the OVSD approach.
    - **No built-in reverse-polarity protection.** Unlike LM2940-N (which has genuine reverse-polarity tolerance to -30V DC / -75V transient, would have allowed omitting the series rectifier), TPS7A19's datasheet makes no reverse-polarity claim. **A series rectifier is required** — see [Component Summary / BOM (DemonEye)](#component-summary--bom-demoneye) for the selected part (NSVR05F40NXT5G — this doc previously and incorrectly listed B5819WS-SL here, a merge-artifact discrepancy between two conflicting BOM rows, corrected against the actual schematic).
    - **Output capacitor — resolves the capacitor architecture conflict that ruled out LM2940-N:** per the Pin Functions table, OUT requires *"a 10-μF to 500-μF ceramic capacitor with an equivalent series resistance (ESR) from 0.001 to 20 Ω to assure stability"* — no meaningful ESR floor, so a plain MLCC works, consistent with this project's "MLCCs only" convention (unlike LM2940-N's 100mΩ–1Ω ESR window, which would have forced tantalum — a ~31mm² Case D part on a 16×20mm board).
    - **Not AEC-Q100 qualified.** Confirmed via TI's own LDO selection guide, which places TPS7A19 in the general-purpose column, separate from the dedicated "Automotive (AEC-Q100)" list (which requires a "-Q1" suffix TPS7A19 doesn't have). Same gap as LM2940-N had (neither candidate ended up formally automotive-qualified) — not a differentiator between the two, just a known limitation either way.
    - **Thermal margin — comparable to LM2940-N once calculated properly, not worse.** An initial comparison using only each part's exposed thermal pad suggested TPS7A19 had meaningfully less margin (smaller pad: 3.96mm² vs. LM2940's 6.6mm²). Redoing the calculation with all leads' parallel heat paths included (TPS7A19: 8 independent leads at 0.3×0.6mm each; LM2940: 6 independent leads at 0.5×0.3mm each, 2 of 8 fused to the pad) brought both parts to a comparable total conduction-path resistance (~30.4°C/W TPS7A19 vs. ~22.0°C/W LM2940, using this board's actual dielectric spec — 0.13mm, 1 W/m·K — plus the 1.6mm aluminum spreading layer and 0.254mm/1 W/m·K thermal tape to a 14×20×6mm 7-fin heatsink). Both land within a similar range (~17.7°C/W vs. ~18.5°C/W) of convection performance needed to match their respective datasheet RθJA figures. **The convection stage itself (heatsink fins to ambient air) remains unverified for either part** — an early attempt at that calculation used an invalid fin-spacing assumption and was retracted; a trustworthy number requires either a properly-scaled natural-convection correlation or prototype measurement, neither done yet.
- **Master Inhibit / HSS (new vs. DRL) — gate-bias network RESOLVED (final, v2):** CSD25501F3 P-FET (Q5), $\text{R}_{ds(on)} = 64\text{m}\Omega$, in series on the 12V rail (Source = TPS7A19 LDO's 12V output, downstream of regulation — not the raw input). **Final gate-bias network: SRH → Ra (200Ω, `AC0402FR-0723RL` or equiv. 0402 200Ω) → Gate node → [R10 (10kΩ, `AC0402FR-0710KL`) ∥ Zener D_GS (`BZX884-C16,315`, cathode-at-Gate/anode-at-GND)] → GND.** This supersedes an earlier 33Ω Ra value and adds the Zener, which the original network (Ra+R10 alone) didn't have. SRH = the headlight-on signal (was informally called "CLL" earlier in this doc before being renamed to avoid confusion with the DRL board's CLL power net — see [Merge Notes / Open Questions](#merge-notes--open-questions)), sourced from the low/high beam relay signals (B121/B122), $\text{V}_{BAT}$ class (12–14V, floor 12.08V per vehicle measurement, ceiling 14.1V engine-running).
    - **Why the Zener was added:** with only Ra/R10, a load-dump event on SRH (clamped by D2 to up to 27V) would pull Gate to within a volt or two of SRH's clamped value, driving Vgs to an uncharacterized positive region — CSD25501F3's datasheet only rates VGS at **-20V** (P-FET's intended polarity); the positive direction has no manufacturer rating at all, not an assumed-symmetric one. A Gate-to-GND Zener bounds this directly, independent of SRH's fault voltage.
    - **Why not a plain diode instead of a Zener:** initially considered a forward-conducting G-S diode (Gate→diode→Source), but it would falsely engage during **ordinary** headlights-on operation (SRH=12-14V normally already gives Vgs=0 to +2.35V, well above any diode's Vf) — clamping continuously, not just during faults, and dissipating real continuous power in Ra. The Zener's higher trigger threshold (16V nominal, referenced to GND rather than Source) avoids this.
    - **Vz selection (16V, BZX884-C16,315):** checked against BZT52C15 (Vz_min=13.8V — **fails**, already breaks down at SRH's normal 14.4V ceiling), BZT52C18 (no AEC qualification found, long lead time), before landing on BZX884-C16,315 (Nexperia, AEC-Q101, DFN1006-2 — same footprint family as D1-D5, in stock).
    - **Normal operation (SRH=14.4V ceiling, worst case for false-triggering):** Gate≈14.35V (Ra/R10 divider barely attenuates) vs. Vz_min=15.3V → **0.95V margin**, zener stays off, Vgs=+2.35V as intended (FET OFF, inhibit active).
    - **Fault (SRH=27V via D2's own worst-case clamp), at Ra=200Ω:** current ≈ (27-17)/200 ≈ **50mA** (using Vz_max=17.1V + dynamic impedance rise as the clamp estimate) → Vclamp ≈ 17.1V + (0.050-0.005)×40Ω ≈ **18.9V** → Vgs ≈ **+6.9V**, comfortably inside the FET's actual ±20V rating (this 6.9V number stacks worst-case tolerance on every variable simultaneously — real-world exposure is likely lower).
    - **Ra sizing (200Ω, not the originally-discussed 33Ω):** at 33Ω, fault current would have been ≈303mA — too high for a standard small-signal Zener's pulse rating, would have needed a surge-rated TVS instead. Increasing Ra to 200Ω cuts this to ≈50mA (trivial pulse energy, ~17µJ at 8/20µs — no dedicated surge rating needed) while only costing ~0.19V of extra margin at SRH's floor (12V): $V_{gs} = -12 \times 200/10200 ≈ -0.24V$, still comfortably clear of CSD25501F3's loosest $V_{GS(th)}$ = -0.45V min.
    - **Default (SRH = 0V/Hi-Z):** unaffected by any of the above — Gate pulled fully to GND by R10 → $V_{GS}$ ≈ −12V → deep enhancement → **ON (LEDs powered)**.
    - **Open dependency (unchanged):** Ra/the Zener still sit downstream of raw SRH, which is presumably unclamped and ISO 16750-2 load-dump exposed upstream of D2 — this is the still-open **"Headlight-on input voltage limiting <20V"** checklist item.
    - This is a single, board-level power gate — separate from the per-channel switch FETs (Q1–Q4) described below.

### Constant Current Channel Design (DemonEye)

Same core topology as the DRL board (see [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology)), with one structural change: 3 LED packages in series per channel instead of 1 package per set of channels.

- **One WS2814F** drives all 4 channels (R, G, B, W) for the module (swapped from WS2814B — package too large for this board's footprint; WS2814F confirmed 8-pin package). Pin names are the same as WS2814B (OUTR/OUTG/OUTB/OUTW/GND/DOUT/DIN/VDD), and VDD range (3.7–5.3V) plus OUTx sink current (16.5mA typ, 15.5–17.5mA range) match WS2814B, so the R5/C1 series-resistor-into-VDD network carries over from DRL unchanged.
- **Data order / OUTx→LED wiring — RESOLVED (final):** WS2814B and WS2814F have *different, fixed* internal latch orders (WS2814B: position1→OUTR, 2→OUTG, 3→OUTB, 4→OUTW; WS2814F: position1→OUTW, 2→OUTR, 3→OUTG, 4→OUTB), so a single WLED color order cannot map "straight" (own-name-to-own-name) onto both boards' existing physical layouts simultaneously. Resolved by choosing a WLED color order that keeps DemonEye's **already-fixed layout** wired as-is, and pushing all required rewiring onto the DRL board, which still has layout room. See **[WLED / Control Architecture (Shared)](#wled--control-architecture-shared)** for the final agreed byte order and both boards' required OUTx→LED mapping — this supersedes the earlier "straight RGBW, no firmware change" assumption below.
- **DemonEye OUTx→LED wiring (as currently laid out, confirmed compatible with the final WLED order — no schematic/layout change needed):**
  - OUTW → LED **Red**
  - OUTR → LED **White**
  - OUTG → LED **Blue**
  - OUTB → LED **Green**
- **Per channel current path:** LDO rail (12V) → LED1 → LED2 → LED3 (same color, 3 packages in series) → BCR421UFDQ OUT pin → Rext → GND. Identical regulation topology to DRL; the difference is 3 series LED voltage drops in the current path instead of 1.
- **Channel current (confirmed):** target was 100mA, equal across all 4 channels (R, G, B, W) — reduced from an earlier 125mA design point to improve thermal margin across LED packages, BCR421UFDQ, and the LDO. **REXT resolved (final):** AF0201FR-0710RL (YAGEO, 10Ω ±1%, 0201, Anti-Sulfur/AEC-Q200 automotive grade) populated on R4–R7. The simplified $R_{INT}$∥$R_{EXT}$ formula (below) suggested ~100.5mA at 10.5Ω, but the datasheet's own REXT-vs-IOUT graph (Fig. 8, BCR421UFDQ) — which is what Diodes actually recommends using, since the formula demonstrably breaks down at higher currents — reads **~85mA at REXT=10Ω**. **10Ω was deliberately chosen over a closer-to-100mA value (~8.5–9Ω) to err toward the lower/safer side of the current-thermal tradeoff.** Actual per-channel current is therefore **~85mA**, not 100mA — thermal/LDO-current figures below were computed at the 100mA design point and are now conservative (i.e. real margins are better than shown) rather than wrong; not yet re-derived at 85mA. Total LDO output current at the design-point assumption = 4 × 100mA = 400mA (89% of TPS7A19's 450mA rating); actual ~4×85mA=340mA (76%) is comfortably better.
- **Per-channel switch FET:** CSD25501F3, used identically to DRL — gates the BCR421UFDQ EN pin, driven by WS2814F OUTx through the same RC network / signal-inversion logic as DRL. One CSD25501F3 per channel (4 total), separate from the master Inhibit/HSS FET.
- **LED package:** QLSP06RGBXWAJ (same part as DRL — see [Shared Circuit Building Blocks](#qlsp06rgbxwaj-led-package)), ×3 per board (not ×1 per channel-set — 3 physical packages shared across all 4 channels).

**Open / pending for this section:**
- ~~Per-channel target current is now set at 100mA. **Rext value itself still needs calculation** from the BCR421UFDQ REXT vs. IOUT graph for 100mA — not yet pulled.~~ **Resolved:** REXT = 10Ω (AF0201FR-0710RL, YAGEO), populated on R4–R7 in the schematic. Graph-read actual current ≈85mA/channel (not exactly 100mA — see above), accepted deliberately as the more conservative option. TCR at exactly 10Ω falls on the AF0201 series' -100/+350ppm/°C bucket boundary (vs. ±200ppm/°C for R>10Ω) — flagged inline in the schematic Description field as still worth a direct datasheet confirmation.
- The previously-listed $R_{sense} = 5.6\Omega$ value is **legacy from a different design and should be disregarded.**
- At 100mA and a 12V rail, headroom checks out for typical Vf across the board (BGW: 3×3.2V + 1.4V = 10.6V ≤ 12V; Red: 3×2.2V + 1.4V = 8V ≤ 12V), using the correct $V_{OUT(MIN)}$=1.4V spec per [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology) — full worst-case-bin verification against $V_{OUT(MIN)}$ still pending as a formal check, though the worst-case power numbers are captured in thermal management below.

### Thermal Management (DemonEye)

The compact PCB area requires strategic heat distribution to avoid localized hotspots. Distributing voltage drop across 3 series LED packages per channel is the primary thermal strategy vs. concentrating dissipation in the LDO. Board is aluminum-core with a 15×15mm heatsink on the back.

**LED Package Thermal Budget (QLSP06RGBXWAJ):**

Design point: 100mA per channel, equal across R/G/B/W, evaluated at $T_{amb} = 70°C$. (Reduced from an earlier 125mA design point for improved margin — see below.) **Note:** REXT is now finalized at 10Ω (AF0201FR-0710RL), which per the BCR421UFDQ's own REXT-vs-IOUT graph reads ≈85mA, not 100mA — all figures below use the 100mA design point and are therefore conservative (real margins will be somewhat better); not yet re-derived at the actual ~85mA.

- **Derived thermal resistance** (from datasheet Pd = 3600mW @ Ta=25°C, $T_{J,max}$ = 120°C — see [Shared Circuit Building Blocks](#qlsp06rgbxwaj-led-package)):
$$R_{th(JA)} = \frac{T_{J,max} - T_a}{P_d} = \frac{120-25}{3.6W} = 26.4°C/W$$
- **Available power budget at 70°C ambient:**
$$P_{d(70°C)} = \frac{120-70}{26.4} = 1895mW \text{ per package}$$
- **Actual dissipation at 100mA, typical Vf (Red=2.2V, Blue/Green/White=3.2V, ΣVf=11.8V)** — valid to sum this way *because* all 4 channels share the same current, so $P = I_R V_{f,R} + I_G V_{f,G} + I_B V_{f,B} + I_W V_{f,W}$ factors to $I \times \Sigma V_f$:
$$P = 0.100A \times 11.8V = 1180mW$$
- **Resulting junction temp:** $T_J = 70 + (1.180 \times 26.4) = 101.2°C$ → **18.8°C margin below $T_{J,max}$ (≈37.7% power margin below the 1895mW budget)** — up from 11.1°C/22% at the earlier 125mA design point.

This applies per physical package — each of the 3 series-connected packages dissipates independently from its own 4 dies; the series connection shares current, not heat, across packages.

**BCR421UFDQ Dissipation (per channel, $P = V_{OUT} \times I_{OUT}$, at 12V rail):**

- **Green / Blue / White:** $V_{OUT}$ = 1.4V (rail 12V − typ. string 3×3.2V=9.6V, minus the 1.4V regulation node itself lands consistently with datasheet $V_{OUT(MIN)}$ typ) → $P = 1.4V \times 0.100A = 140mW$ each (420mW combined for 3 channels).
- **Red — worst case (16V battery, LDO in full regulation at 12V, Red string at min-Vf bin 3×2.0V=6.0V):**
$$V_{OUT} = 12V - 6.0V = 6.0V \rightarrow P = 6.0V \times 0.100A = 600mW$$
This is the worst-case condition for the Red BCR421UFDQ, since max battery voltage keeps the rail pinned at 12V (LDO in regulation) while minimum Vf leaves the most voltage for the BCR421 to absorb.
- **BCR421UFDQ package rating context:** datasheet gives PD = 1.3W/RθJA=100°C/W (25×25mm 1oz Cu) or PD=1.7W/RθJA=75°C/W (50×50mm 1oz Cu), both on standard FR4 — worse thermal performance than this board's aluminum-core + dedicated 15×15mm heatsink construction. No MCPCB-specific RθJA figure available from the datasheet; real performance on this board is expected to be meaningfully better than the FR4 curves suggest. **Confirmed acceptable per design review; not independently quantified via dielectric/heatsink Rth calc or prototype measurement.**

**LDO (TPS7A19) Dissipation:**

- **Total LDO output current:** 4 channels × 100mA = 400mA (89% of TPS7A19's 450mA rated max — tight margin, worth monitoring; also close to the current-limit spec's guaranteed minimum of 470mA at 90% of nominal output voltage, leaving ~15% headroom below that threshold).
- **Worst case (16V battery, 12V rail):** $V_{drop}$ = 4V → $P = 4V \times 0.4A = 1.6W$ (same physics as any LDO at this Vin/Vout/current — this figure doesn't change between LDO candidates at a fixed operating point).
- **Thermal budget:** using TPS7A19's own RθJA (48°C/W, JEDEC-reference) and Recommended Operating Conditions TJ max (125°C, the datasheet's explicit rated limit — cleaner than LM2940-N's ambiguous 125°C/150°C conflict, since TPS7A19 states 125°C plainly with no contradicting figure): $P_{d,max}(70°C) = (125-70)/48 = 1.146W$. The 1.6W worst case exceeds this by ~40% on the JEDEC reference board — same category of gap LM2940-N had at its original 125mA design point, now shifted to TPS7A19 instead. Closing this relies on the same aluminum-board/heatsink margin argument used throughout this document — see [Power Architecture (DemonEye)](#power-architecture-demoneye) for the detailed conduction-path calculation (~30.4°C/W just for conduction, meaning the heatsink's convection performance needs to be quite good, and that piece is still unverified).
- **Best case (engine running, ~13.6–14.4V battery):** $V_{drop}$ ≈ 1.6–2.4V → $P ≈ 0.64$–$0.96W$ — comfortably within budget.
- Engine-off dropout condition (below ~12.24-12.45V, per TPS7A19's dropout spec at 400mA) reduces LDO dissipation further at the cost of some LED dimming — self-limiting in the direction that helps thermal margin, same pattern as other LDO candidates evaluated.
- **Load-dump event:** rather than going dark (LM2940-N's OVSD behavior), TPS7A19 is rated to continue regulating through the transient, per its "simple surge protection circuit" description — LEDs should stay lit.

**Summary — Worst-Case Total Heat (16V battery, 12V rail, min-Vf Red bin):**

| Source | Power |
|---|---|
| LED packages (3× combined, typical Vf) | ~3.5W (3 × 1180mW) |
| BCR421UFDQ (G+B+W, typ) | 420mW |
| BCR421UFDQ (Red, worst case) | 600mW |
| LDO (worst case) | 1.6W |
| **Total** | **~6.1W** |

Note: LED package figures use typical Vf (per-package self-heating, independent of rail/battery voltage) while BCR421/LDO figures use worst-case battery/Vf conditions — these are somewhat mismatched conditions layered for a conservative combined estimate, not a single fully self-consistent worst-case corner. Board-level thermal (shared heatsink loading from all sources simultaneously) not yet analyzed.

### Component Summary / BOM (DemonEye)

| Component | Role | Key Spec | Datasheet | Qty (per board) |
| :--- | :--- | :--- | :--- | :--- |
| **TPS7A1901DRBR** | LDO Regulator (adjustable) | SON-8 (DRB), 3.00×3.00mm; $V_{FB}$=1.233V typ; 4–40V in (45V abs max); 450mA rated; target $V_{out}$=12V via R1=87.3kΩ/R2=10kΩ; not AEC-Q100 qualified; requires ceramic-compatible output cap (0.001-20Ω ESR window — MLCC-friendly) | https://www.ti.com/lit/ds/symlink/tps7a19.pdf | 1 |
| **NSVR05F40NXT5G** | Series rectifier (reverse-polarity protection, power input line) | DSN2 (0402), CSP; VR=40V; IF(DC)=500mA continuous (system load ≈350–400mA at actual ~85mA/channel × 4 — 70–80% utilization, tighter margin than a larger part but adequate); VF=0.42–0.46V @500mA; AEC-Q101 qualified; **required** since TPS7A19 has no built-in reverse-polarity protection (unlike LM2940-N, which was the point of comparison that led here) | https://www.onsemi.com (NSR05F40D.PDF, project file) | 1 |
| Output capacitor (TPS7A19 OUT pin) | Bulk/stability cap | 22µF (within datasheet's 10-500µF range), any small X7R/X5R MLCC — no meaningful ESR floor per datasheet (0.001-20Ω), consistent with "MLCCs only" board convention; **exact part/package TBD** | | 1 |
| **CSD25501F3** | Master Inhibit / HSS FET (rail disable via SRH, the headlight-beam signal) | $\text{R}_{DS(on)} = 64\text{m}\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$ typ (-0.45V min to -1.05V max); Source confirmed downstream of the LDO (only sees regulated 12V rail, not raw line); $\text{V}_{DS}$/$\text{V}_{GS}$ abs max = ±20V (built-in 10kΩ die-internal gate clamp resistor); gate-bias network resolved, see [Power Architecture (DemonEye)](#power-architecture-demoneye) | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf | 1 |
| **CSD25501F3** | Per-channel switch FET (gates BCR421UFDQ EN, same use as DRL) | $\text{R}_{DS(on)} = 64\text{m}\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$ | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf | 4 |
| **BCR421UFDQ** | Constant-current regulator, per channel | $R_{INT}$ = 95Ω typ; $V_{OUT(MIN)}$ = 1.4V typ ($I_{OUT}$>18mA) | (project file) BCR420UFDQBCR421UFDQ.pdf | 4 |
| **QLSP06RGBXWAJ** | RGBW LED package, ×3 in series per channel | same part as DRL board — see [Shared Circuit Building Blocks](#qlsp06rgbxwaj-led-package) | (project file) qlsp06rgbxwaj274_v1_0.pdf | 3 |
| **WS2814F** | LED controller, drives all 4 channels | 8-pin; VDD 3.7–5.3V; OUTx sink 16.5mA typ; data frame order **WRGB** (vs. WS2814B's RGBW) — requires OUTx→LED pin reassignment, see above | (project file) WS2814F.pdf | 1 |
| **AF0201FR-0710RL** | $\text{R}_{ext}$, Current Sense (per channel) | YAGEO, 10Ω ±1%, 0201, Anti-Sulfur/AEC-Q200 automotive; graph-read actual current ≈85mA/channel (target was 100mA — 10Ω chosen deliberately over a closer ~8.5-9Ω fit, erring conservative); LCSC C143979 | https://yageogroup.com/content/datasheet/asset/file/PYU-AF_51_ROHS_L | 4 |
| **D15V0H1U2LP-7B** | TVS, power input line (D1) | X1-DFN1006-2 (~1.07×0.67mm); VRWM=15V; VBR=16-19V typ @1mA; VCL=22V@5A/27V@12A (8/20µs); sized for routine ISO 7637-2 transients only — load-dump survival handled by TPS7A19's wide continuous input range, not this part; not AEC-Q qualified (consumer ESD-protection line) | (project file) D15V0H1U2LP.pdf | 1 |
| **D15V0H1U2LP-7B** | TVS, SRH line (D2, before Ra) | Same part as D1, reused per BOM-consolidation convention. **Margin against SRH's <20V target not yet fully confirmed** — VBR/22V-@5A clamp point is comfortably under 20V, but the 27V-@12A clamp point exceeds it; whether 12A is a realistic worst case for this relay-switched (not direct-battery) signal hasn't been characterized — see [Pre-Order Checklist](#demoneye-pre-order-verification-checklist) | (project file) D15V0H1U2LP.pdf | 1 |
| **AC0402FR-0710KL** | R10, Gate-to-GND pull-down (Q5 master inhibit) | YAGEO, 10kΩ ±1%, 0402, AEC-Q200; updated from an earlier 1.2kΩ value — see [Power Architecture (DemonEye)](#power-architecture-demoneye) for the power-dissipation reasoning | https://www.digikey.com/en/products/detail/yageo/AC0402FR-0710KL/5895030 (LCSC C144807) | 1 |
| **AC0402FR-0723RL** (or nearest 200Ω 0402) | Ra, SRH series gate-bias resistor | YAGEO, 200Ω ±1%, 0402 — updated on the board from an earlier 33Ω design point to limit fault current into D6 to ≈50mA (was ≈303mA at 33Ω); exact YAGEO part number TBD-confirm (200Ω point on the AC0402FR-07xxxL family, not yet cross-checked against LCSC); **placed on PCB** | (project file) YAGEO AC-series | 1 |
| **BZX884-C16,315** | D6, Gate-Source Zener clamp (Q5 master inhibit) | Nexperia, 16V ±5% (Vz 15.3-17.1V), Zzt=40Ω, DFN1006-2 (1.0×0.6mm) — same footprint family as D1-D5; AEC-Q101 qualified; cathode-at-Gate/anode-at-GND, bounds Vgs during SRH load-dump events independent of D2's clamp voltage; **placed on PCB** | https://mm.digikey.com/Volume0/opasdata/d220001/medias/docus/8911/bzx884c16v.pdf | 1 |
| **SZESD7241N2T5G** (×2) | ESD protection, DIN and DOUT | X2DFN-2, single-channel/2-pin per part (not a shared array — needs one per line); AEC-Q101, VRWM=24V, VC=48V max; matches DRL's DATA-line protection approach applied to both connector-exposed data pins | https://www.digikey.com/en/products/detail/onsemi/SZESD7241N2T5G/7221019 | 2 |

### Interface Connector (DemonEye)

5- or 6-pin picoBlade header (exact pin count pending — see note below). Unlike the DRL board's connector (DATA/GND/GND/VBAT, 4 pins), DemonEye needs both a data input and a data output, since it is designed to sit in the WS281x chain ahead of the DRL board rather than being the last device on the chain.

| Pin | Signal | Notes |
|-----|--------|-------|
| — | DIN | WS2814F data input, from WLED/ESP32-S3 (or from an upstream board, if DemonEye is not first in the chain) |
| — | DOUT | WS2814F data output, feeds DRL board's DIN |
| — | GND | Data/logic ground |
| — | VBAT | Power input — **tap point not yet finalized, see [Merge Notes / Open Questions](#merge-notes--open-questions)** |
| — | GND | Power return (if a separate pin from data GND, mirroring the DRL board's split data-GND/power-GND scheme) |
| — | *(possible 6th pin)* | If SRH (the headlight-on master-inhibit trigger — see [Merge Notes / Open Questions](#merge-notes--open-questions)) turns out to need a dedicated sense line (rather than being derivable from DemonEye's own VBAT, per Merge Note 1), it would need its own pin here |

**Pin assignments, exact count, and connector part number not yet finalized** — this table lists the known required signals, not a confirmed schematic-matched pinout.

---

## WLED / Control Architecture (Shared)

- **Controller:** ESP32-S3 running WLED — separate board, independently powered, sends WS281x data to both the DRL board and DemonEye board
- **Protocol:** WS281x single-wire to WS2814B (DRL) / WS2814F (DemonEye) chain
- **DRL mode:** White channel full brightness + optional RGB trim
- **Turn signal:** RGB amber (Red+Green) sequential animation — supplementary to OEM turn signal and fender side markers; no regulatory brightness requirement
- **DRL board on/off:** Board powered from CLL relay. Confirmed by observation that CLL remains active with headlights off, consistent with correct DRL behavior (board should be powered whenever DRL is meant to operate, independent of headlight state). WLED sends data regardless; board simply doesn't respond when unpowered.
- **DemonEye board on/off:** Board itself intended to be powered by default; a separate signal, **SRH** (headlight-on, sourced from beam relay signals B121/B122 — not the same net as CLL), gates a master-inhibit FET to cut LED power when headlights are on. Gate-bias network for this is now resolved — see [Power Architecture (DemonEye) — Master Inhibit / HSS](#power-architecture-demoneye). **Still open:** SRH's physical wire tap doesn't exist yet (not present on the connector DemonEye's power comes from) and its exact voltage/polarity hasn't been measured — see [Merge Notes / Open Questions](#merge-notes--open-questions).
- **PWM frequency:** WS2814x OUTx at 2kHz — above flicker threshold, within BCR421 25kHz EN pin limit
- **Flicker:** Not a concern on DRL — 200kHz buck + 2kHz WS2814B PWM + BCR421 linear regulation = triple isolation from input ripple to LED current. DemonEye's equivalent isolation chain (LDO or buck front end, still TBD) not yet analyzed.

### WLED Color Order & Cross-Board OUTx Wiring — RESOLVED (final)

WS2814B and WS2814F have different, fixed internal latch orders and cannot both be wired "straight" (own-color pin to own-color LED) under a single WLED color order. **DemonEye's physical layout is already fixed and was treated as the constraint; DRL still has layout room, so all required rewiring was pushed onto DRL.**

**Final WLED output color order: R, W, B, G** (not straight RGBW).

| Board | Chip | Pin | LED |
|---|---|---|---|
| DemonEye | WS2814F | OUTW | Red |
| DemonEye | WS2814F | OUTR | White |
| DemonEye | WS2814F | OUTG | Blue |
| DemonEye | WS2814F | OUTB | Green |
| DRL | WS2814B | OUTR | Red |
| DRL | WS2814B | OUTG | White |
| DRL | WS2814B | OUTB | Blue |
| DRL | WS2814B | OUTW | Green |

**DemonEye:** matches its current/already-committed physical layout — no schematic or layout change required.
**DRL:** OUTR→Red and OUTB→Blue are "straight" (already correct if wired conventionally), but **OUTG↔OUTW must be swapped** (OUTG→White instead of Green, OUTW→Green instead of White) — this is DRL's required layout rework, not yet implemented. See [Open Items](#open-items--pending-both-boards).

---

## Open Items / Pending (Both Boards)

| Item | Board | Status | Notes |
|------|-------|--------|-------|
| PCB layout | DRL | **Pending** | Full layout within constraints; confirm all components fit |
| Prototype thermal validation | DRL | **Post-build** | Measure actual Tj on LMR51460F-Q1 and BCR421 under load |
| Rext White part selection | DRL | **Pending** | Schematic (R4, ×11) has no manufacturer/MPN assigned; target 4.02Ω 1% 0402, schematic symbol currently labeled "4Ω" |
| Rext RGB part selection | DRL | **Pending** | Schematic (R1–R3, ×33) has no manufacturer/MPN assigned; target 13.0Ω 1% 0402 |
| WS2814B internal regulator series resistor (R5, ×11) | DRL | **Pending** | Value set at 100Ω in schematic; no manufacturer/MPN assigned |
| WS2814B internal regulator decoupling cap (C1, ×11) | DRL | **Pending** | Neither value nor manufacturer/MPN assigned in schematic — still shows library default; needs a value chosen |
| Confirm CLL vs. DRL activation behavior | DRL | **Resolved** | Confirmed by observation: board and LEDs remained powered with headlights off, meaning CLL is active independent of headlight beam state — consistent with correct DRL function. |
| Confirm DRL pin's actual downstream function | DRL | **Pending** | Working theory: DRL pin drives a transistor gate/base selecting between a low-current default state (~11.6mA measured) and full brightness (~250mA measured) on the MAX16833C's current reference — not confirmed by tracing the actual circuit (runs under a metal shield). Not required for the corrected power-tap plan, but worth resolving for full documentation accuracy. |
| Confirm OEM current measurements against possible internal PWM dimming | DRL | **Pending** | ~11.6mA and ~250mA readings were taken on a DC multimeter (average-reading). If the MAX16833C PWM-dims the LED string internally, these won't distinguish that from genuine continuous current — recommend scope verification across the shunt if precision matters for future comparisons. |
| DRL OUTx rewiring (OUTG↔OUTW swap) | DRL | **Pending** | Required by final WLED color order decision (R,W,B,G) — see [WLED Color Order & Cross-Board OUTx Wiring](#wled-color-order--cross-board-outx-wiring--resolved-final). OUTR→Red and OUTB→Blue stay as conventionally wired; OUTG and OUTW must swap (OUTG→White, OUTW→Green). Chosen to keep DemonEye's already-fixed layout unchanged, since DRL still has layout room. Not yet implemented in DRL schematic/layout. |
| DemonEye power tap / SRH conflict | DemonEye | **Partially resolved — gate-bias math done, physical tap still TBD** | Confirmed CLL cannot double as both DemonEye's power source and a "headlights on" trigger — resolved by naming the trigger signal **SRH** (distinct net from CLL) sourced from beam relay signals B121/B122. Master inhibit FET's Source confirmed downstream of the LDO (only sees regulated 12V rail, not raw line — no protection concern there). **Gate-bias network now fully resolved:** SRH→Ra(33Ω)→Gate→R10(10kΩ)→GND — see [Power Architecture (DemonEye)](#power-architecture-demoneye). **Still TBD:** SRH is not present on the shared 8-pin DRL driver board connector DemonEye's power comes from — requires a new, separate wire tap into B121/B122; voltage/polarity of the beam relay signals not yet measured (assumed $V_{BAT}$ class, 12-14V, per design decision — not yet confirmed on-vehicle); OR-logic to combine low+high beam into one SRH signal; the physical wire run itself; and the upstream <20V clamp on raw SRH (see pre-order checklist). |
| DemonEye LDO selection | DemonEye | **Decided (final)** | **TPS7A19 (TPS7A1901DRBR), SON-8 3×3mm, adjustable, selected over LM2940-N.** Reasons: (1) TPS7A19's 0.001-20Ω output ESR window is MLCC-compatible, avoiding LM2940-N's forced tantalum (Case D, ~31mm² — a major board-space cost); (2) TPS7A19's 40V continuous input range covers the ~35V suppressed load-dump ceiling directly, maintaining regulation through the event rather than LM2940-N's OVSD going dark; (3) TPS7A19's package is ~44% smaller (9mm² vs. 16mm²). **Trade accepted:** TPS7A19 has no built-in reverse-polarity protection (LM2940-N does, -30V DC/-75V transient), so a series rectifier is now required — but the rectifier (B5819WS-SL, SOD-323, ~1.9mm²) is far smaller than the tantalum cap it replaces, making this a clear net space win even after accounting for it. TPS7A19 is also not AEC-Q100 qualified (confirmed via TI's own selection guide) — same gap LM2940-N had, not a differentiator. Thermal margin is comparable between the two parts once calculated with full lead-path credit (~17.7°C/W vs. ~18.5°C/W convection targets) — initial analysis suggesting TPS7A19 was thermally worse was based on an incomplete calculation (main pad only, no leads) and has been corrected. Current headroom is tighter on TPS7A19 (400mA against 450mA rated, 89% utilization) than LM2940-N would have been (40%) — worth monitoring. |
| DemonEye TVS/rectifier selection | DemonEye | **Decided (both), rectifier part number corrected** | **TVS: D15V0H1U2LP-7B** (X1-DFN1006-2, VRWM=15V, VBR=16-19V, VCL=22V@5A/27V@12A) — sized for routine ISO 7637-2 transients only; load-dump survival now handled by TPS7A19's wide continuous input range (40V) rather than an LDO shutdown mechanism, but the TVS's own scope is unchanged. Confirmed no DFN1006-class part (including automotive-grade options) carries a genuine ISO 7637-2 pulse rating (checked Taiwan Semiconductor's automotive DFN1006-2LW line specifically) — accepted as adequate for the "routine transient" scope. **Rectifier: NSVR05F40NXT5G** (DSN2/0402, 500mA IF, 40V VR, AEC-Q101) — this doc previously listed B5819WS-SL here, a stale/conflicting entry from a merge artifact (two rows existed for the same component in the BOM table); corrected against the actual populated schematic part. |
| DemonEye SRH TVS (D2) margin vs. <20V target | DemonEye | **Resolved** | Originally framed as "does D2 clamp SRH below 20V" — turned out no single TVS could hit that at rated current while also staying off during normal 12-14V operation (checked D10V0/D12V0/D14V0H1U2LP, all failed one requirement or the other). **Resolved by reframing the fix**: added a Gate-Source Zener (`BZX884-C16,315`) directly on Q5's gate-bias network, which bounds Vgs to ≈6.9V worst-case regardless of what D2/SRH actually clamp to — see [Power Architecture (DemonEye) — Master Inhibit / HSS](#power-architecture-demoneye). D2 remains in place for its original ESD/routine-transient role but is no longer the sole thing protecting Q5. |
| DemonEye heatsink convection performance | DemonEye | **Pending — load-bearing** | Heatsink: 14×20×6mm aluminum, 7 vertical fins along the 20mm direction, attached via 0.254mm/1 W/m·K thermal tape. Conduction path (dielectric + aluminum spreading + tape, all the way to the heatsink base) is now well-quantified (~22-30°C/W depending on LDO package, see [Power Architecture (DemonEye)](#power-architecture-demoneye)) — the remaining unknown is convection from the fins to ambient air inside the sealed headlamp housing (natural convection only, no forced air). An early attempt at this calculation used an invalid fin-spacing rule of thumb (borrowed from a different size regime) and produced a result implying the heatsink underperforms bare FR4, which is physically implausible and was retracted. **No trustworthy convection number exists yet for this heatsink.** Needs either a correctly-scaled natural-convection correlation (e.g., Churchill-Chu, properly applied to this fin height/spacing) or prototype thermal measurement — the latter requires boards to exist, which has cost implications. This is the single largest source of uncertainty remaining in the DemonEye thermal case, affecting both the LDO decision and (to a lesser extent) the BCR421/LED margins that also lean on this same heatsink. |
| DemonEye Rext value | DemonEye | **Resolved** | REXT = 10Ω, **AF0201FR-0710RL** (YAGEO, 0201, ±1%, Anti-Sulfur/AEC-Q200), populated on R4–R7. Datasheet's own REXT-vs-IOUT graph (not the simplified formula, which was shown to deviate ~31% at higher currents) reads ≈85mA/channel at 10Ω — target was 100mA; 10Ω accepted deliberately as the more conservative of the two closest standard values. TCR at this exact boundary value (-100/+350ppm/°C vs. ±200ppm/°C for R>10Ω) flagged in schematic for direct datasheet confirmation. |
| DemonEye PCB layout | DemonEye | **Pending** | Fit WS2814F, 4× BCR421, 5× CSD25501F3, 3× LED packages, TVS/rectifier (TBD part), LDO/buck (TBD part), and 5/6-pin picoBlade header within 16mm×20mm |
| DemonEye connector pinout | DemonEye | **Pending** | Needs DIN, DOUT, VBAT, and at least one GND at minimum (data pass-through to DRL board requires DOUT, unlike DRL's connector); exact pin count (5 vs. 6) depends on whether SRH needs a dedicated pin — see Merge Notes |
| DemonEye DIN/DOUT ESD protection | DemonEye | **Decided, default** | Both DIN and DOUT to get their own SZESD7241N2T5G (or equivalent), matching DRL's single-line protection philosophy applied to both connector-exposed data pins (2 parts total, since the part is single-channel/2-pin, not a shared array). **Fallback if board space runs out:** drop DOUT's dedicated TVS and rely on WS2814F's native pin ESD rating (~2kV HBM class) for that line only — DIN keeps its TVS regardless. |

---

## Merge Notes / Open Questions

Two things surfaced while combining these documents that weren't fully resolved in either source document individually — flagging both explicitly rather than silently picking an answer:

1. **DemonEye's power tap vs. its headlight-on trigger — naming/identity conflict RESOLVED, physical implementation still pending.** DemonEye's design (written before the CLL/DRL pin correction) originally described two things as both being "CLL" — its own Vbat input and a headlight-on trigger for the master inhibit FET — which, once CLL was confirmed as DemonEye's actual shared power supply, could no longer also serve as a distinguishable "headlights on" trigger (a signal can't indicate "off" by being present when it's also required to always be present for power). **Resolved by treating them as two genuinely separate nets:** DemonEye's power tap is CLL; the trigger signal is now called **SRH**, sourced from the beam relay signals (B121/B122) rather than from CLL. The gate-bias network consuming SRH is fully designed (see [Power Architecture (DemonEye)](#power-architecture-demoneye)). What's still open is purely physical: SRH's wire tap doesn't exist on the current connector and needs to be added, and its actual voltage/polarity on-vehicle hasn't been measured (assumed $V_{BAT}$ class per design decision, not yet confirmed).

2. **BCR421UFDQ's two voltage specs ($V_{DROP}$ vs. $V_{OUT(MIN)}$) are correctly used for two different purposes in the current documents, contrary to an earlier working assumption in this project that they were being conflated.** The REXT-setting formula correctly uses $V_{DROP}$ = 0.95V typ; headroom/margin checks correctly use $V_{OUT(MIN)}$ = 1.4V typ. Flagging this so it doesn't get "fixed" incorrectly in a future pass — worth a final check that every instance in this document uses the right one for its purpose, but no error was found in this merge.

---

## DemonEye Pre-Order Verification Checklist

*Added [date TBD] following schematic MPN/footprint verification pass (all passive and active component MPNs and footprints confirmed correct — see prior verification notes). This checklist covers what remains before placing a 40-unit production order.*

### Electrical / Schematic

- [x] **REXT value verification** — ~~confirm R4–R7 (feeding each BCR421UFDQ) actually compute to the target 125mA/channel via the BCR421UFDQ datasheet's Rext-setting formula.~~ **Resolved:** target moved to 100mA, then finalized at 10Ω (AF0201FR-0710RL, YAGEO 0201, LCSC C143979). Graph-verified (not formula — formula showed ~31% error vs. datasheet's own calibration point) actual current ≈85mA/channel; 10Ω accepted deliberately over a closer ~8.5–9Ω fit, erring conservative. Schematic (R4/R5/R6/R7) updated with new MPN, manufacturer, description, datasheet link, LCSC number, and distributor links.
- [x] **R8/R9 divider function** — confirm what this divider (88k7/10k) actually sets (LDO FB divider vs. EN threshold) and that it yields the correct output voltage / UVLO point for U2 (TPL8051Q).
- [x] **WS2814F pin remap wiring** — ~~confirm the documented remap (OUTW→Red, OUTR→Green, OUTG→Blue, OUTB→White) is actually wired that way in this schematic's nets, not just documented as a requirement.~~ **Resolved:** final WLED color order settled as R,W,B,G (not straight RGBW) specifically so DemonEye's already-committed layout needs **no changes**. Confirmed mapping: OUTW→Red, OUTR→White, OUTG→Blue, OUTB→Green. All required rewiring pushed to the DRL board instead (still has layout room) — see [WLED Color Order & Cross-Board OUTx Wiring](#wled-color-order--cross-board-outx-wiring--resolved-final) and the new DRL Open Item.
- [x] **FET gate-bias network** — ~~confirm R1, R10, and the "10k" pull embedded in the P-Channel FemtoFET symbol give correct VGS at 12V bus for Q1–Q5.~~ **Fully resolved (v2, final):** Topology is SRH→Ra(200Ω)→Gate node→[R10(10kΩ) ∥ D6(`BZX884-C16,315` Zener, cathode-at-Gate)]→GND. Root issues caught along the way: (1) R1 is actually WS2814F's internal-regulator series resistor, unrelated to FET bias — original item was mis-scoped; (2) R10's original 1.2kΩ value dissipated 2-2.7× its 0402 rating while SRH is asserted; (3) a plain Gate-Source diode (considered before the Zener) would have falsely conducted during **ordinary** headlights-on operation, not just faults; (4) the original 33Ω Ra let fault current reach ≈303mA, too high for a standard small-signal Zener — increasing to 200Ω cut this to ≈50mA. **Schematic status: complete.** R10's value/MPN updated directly; Ra (now 200Ω, was originally placed at 33Ω) and the new Zener (**D6**) are both placed on the board. See [Power Architecture (DemonEye) — Master Inhibit / HSS](#power-architecture-demoneye) for full derivation.
- [x] **Full ERC pass in KiCad** — check for unrouted nets, conflicting power pins, missing net ties.
- [x] **Protection stage math** — D1 (TVS), D5 (Schottky rectifier — corrected from a stale "B5819WS-SL" doc entry to the actual populated part, `NSVR05F40NXT5G`), D3/D4 (ESD): confirm clamp/survival current against ISO 16750-2 load-dump, same rigor already applied to the DRL board's D1/TVS pairing. **CLL-line math (D1, D3/D4) unchanged from earlier "Decided" status** — see Open Items. **D5 surge survival — resolved as accepted risk, not rigorously quantified:** worst-case D1 clamp current calc gives ≈16A (8/20µs, Ri=0.5Ω, Vs=35V per ISO 16750-2), vs. D5's IFSM=10A rating — but that 10A figure is specified at 60Hz/1-cycle (≈16.7ms), ~3 orders of magnitude longer than the 8/20µs clamp pulse, and onsemi doesn't publish a pulse-width derating curve for this part (checked Octopart, Mouser, Farnell, direct datasheet — none exist). A rough I²t scaling estimate suggests the true short-pulse capability is far above 10A (~289A, not a trustworthy number but directionally confirms no real constraint here). **Accepted without further quantification:** D5 isn't a safety-critical node — failure modes are open (board simply doesn't power on/protect, easily noticed) or closed (fixable by cutting the wire) — so the unquantified-but-directionally-fine result is an acceptable risk rather than a blocking item.
- [x] **Headlight-on (SRH) input voltage limiting <20V** — **resolved, via a different mechanism than originally framed.** The original approach tried to bound *SRH's own voltage* to <20V so it couldn't push Q5's Vgs out of spec. That turned into a dead end — D2's clamp curve couldn't hit <20V at its own rated current, and no single TVS in the searched families satisfied both "stays off during normal 12-14V operation" and "clamps under 20V during a fault" simultaneously. **Resolved instead by bounding Vgs directly at the FET**, independent of whatever SRH/D2 actually do: the Gate-Source Zener (`BZX884-C16,315`) added under the FET gate-bias network item caps Vgs to ≈6.9V worst-case regardless of SRH's clamp voltage — see [Power Architecture (DemonEye) — Master Inhibit / HSS](#power-architecture-demoneye). This makes D2's own precise clamp behavior no longer safety-relevant to Q5 (D2 still serves its original ESD/routine-transient role on the SRH line, just isn't the sole thing standing between a fault and the FET anymore).

### Layout (not visible from schematic alone)

- [x] **LED orientation consistency** — all QLSP06RGBXWAJ placements share the same orientation (asymmetric die placement requires this).
- [x] **Thermal vias / copper pour** — under BCR421UFDQ and CSD25501F3 (EP-based packages).
- [x] **Courtyard/clearance DRC pass** — for sub-mm parts (PicoStar 0.7×0.6mm, DFN1006-2, 0201s) against JLCPCB's assembly tolerance limits.
- [x] **Aluminum PCB stackup spec** — submitted correctly (thermal path, dielectric).
- [x] **Mechanical fit check** — board outline vs. headlight housing / LGB cone geometry.

### Fab / Assembly Package

- [ ] **Gerbers, drill file, paste layer, pick-and-place, BOM** — exported and cross-checked against schematic (every line has an LCSC part number, including the custom BCR421UFDQ footprint part).
- [ ] **JLCPCB assembly capability confirmation** — for the smallest packages at target panel/qty.
- [ ] **Stock / lead-time check at 40-unit quantities** — reel MOQs on DFN1006-2, PicoStar, WS2814F.

---

## Reference Documents

| Document | Location |
|----------|----------|
| BCR421UFDQ datasheet (DS39535 Rev 4-2) | Project |
| QLSP06RGBXWAJ-274 datasheet (V1.0) | Project |
| CSD25501F3 datasheet (SLPS692C) | Project |
| LMR51460F-Q1 datasheet | Project |
| WS2814B datasheet (V1.4) | Project |
| WS2814F datasheet | Project |
| TPS7A19 datasheet (SBVS256A) | https://www.ti.com/lit/ds/symlink/tps7a19.pdf |
| CSD25501F3 datasheet | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf |
| Samsung MLCC catalog (MLCC_2512.pdf) | Project |
| Vishay T15B22A datasheet (98360) | Project |
| S4PM/S4PJ datasheet | Project |
| SZESD7241 datasheet | Project |
| D15V0H1U2LP datasheet (DemonEye TVS candidate, evaluated and not yet finalized) | Project |
| Wurth 744393465120 datasheet | https://www.we-online.com/components/products/datasheet/744393465120.pdf |
| MAX16833C datasheet | https://www.analog.com/media/en/technical-documentation/data-sheets/MAX16833-MAX16833G.pdf |
| WEBENCH Design Report — LMR51460 12V-16V to 5.50V @ 6A | TI WEBENCH, July 2026 |