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
- [Open Items / Pending (Both Boards)](#open-items--pending-both-boards)
- [Merge Notes / Open Questions](#merge-notes--open-questions)
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

The module uses a **TPL8051Q** (DFN3X3-8, adjustable, wide-VIN, AEC-Q100 Grade 1) LDO to provide a stable, known regulated rail, and a **single WS2814F** (WS2814B ruled out — package too large for this board) driving **4 independent constant-current color channels (R, G, B, W)**. Each channel is formed by **3 RGBW LED packages wired in series** (same color across all 3 packages), rather than 3 independently-addressable packages — the 3 packages are treated as a single 4-color light source, not 3 separate light sources. Only one WS2814F is needed since a series-wired string can't be independently addressed per package anyway.

**Design rationale (series LED packages):** the LDO provides a stable, known voltage to the LED string. Stacking 3 LED packages per channel in series drops the bulk of the excess voltage across the LEDs themselves (distributing heat across 3 packages) rather than having the LDO dissipate it all. The BCR421UFDQ continues to regulate the actual channel current — same shared topology as the DRL board; see [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology).

**Addition vs. DRL board:** a master P-FET, configured as a high-side switch (HSS), gates power to the entire LED circuit. Gate is pulled low by default (FET on, LEDs powered); a CLL (headlight) power signal from the OEM driver board pulls the gate high, turning the FET off and cutting power to the LEDs whenever the headlights are on. **See [Merge Notes / Open Questions](#merge-notes--open-questions) — this mechanism needs to be re-examined now that CLL is confirmed to be the shared power-tap net, not a separate always-distinguishable "headlights on" signal.**

### Power Architecture (DemonEye)

- **Input:** Originally documented as "Same OEM DRL feed as the DRL board," valid automotive $\text{V}_{bat}$ (11.8V–15.6V), clamped by an onboard TVS diode. **This description predates the CLL/DRL pin correction documented in [Vehicle Context](#vehicle-context) and the extended TVS/LDO voltage-margin investigation — DemonEye's actual power tap point and protection scheme have not been finalized to match. See [Merge Notes / Open Questions](#merge-notes--open-questions).**
- **Regulation:** TPL8051Q (DFN3X3-8 package, 3.00×3.00mm nominal, adjustable output, 450mA rated, **AEC-Q100 Grade 1 qualified**) LDO.
    - **Target Output ($\text{V}_{out}$):** 12V, set via feedback divider ($V_{FB}$ = 1.215V typ): $R_2$ = 10kΩ (standard reference value), $R_1$ = $R_2 \times (\frac{V_{out}}{V_{FB}} - 1)$ = 88.8kΩ.
    - **Input range:** 3–42V continuous (recommended operating), 45V max transient (abs max EN/IN). At 400mA total load, dropout is ~700mV typ / 1000mV max (datasheet's 3.3V/5V test points — not explicitly characterized at 12V, but dropout is primarily a function of the pass-device/current, not output voltage, so this is a reasonable estimate). Minimum practical $V_{in}$ for full regulation at 400mA ≈ 12.7–13V. At engine-off (CLL = 12.08V), the part is in dropout — same accepted mild-dimming behavior as every other LDO candidate evaluated.
    - **AEC-Q100 Grade 1 qualified — the first candidate in this search to actually clear this bar.** Both LM2940-N and TPS7A19 lacked formal automotive qualification; TPL8051Q states it directly on the datasheet cover.
    - **Load-dump exposure — resolved by continuous input range**, same mechanism as TPS7A19 (not LM2940-N's OVSD-based shutdown): 42V continuous / 45V transient comfortably covers the ~35V suppressed load-dump ceiling, maintaining regulation through the event.
    - **No built-in reverse-polarity protection** — same gap as TPS7A19. Series rectifier (B5819WS-SL) still required.
    - **Output capacitor:** 2.2–200µF with ESR 0.001–5Ω — MLCC-compatible, same win over LM2940-N's tantalum-forcing window. Datasheet explicitly recommends a 10µF X7R ceramic.
    - **Thermal margin — clears the datasheet-reference budget outright, no aluminum-board credit required.** Unlike TPS7A19 (whose Recommended Operating Conditions caps **TJ** itself at 125°C), TPL8051Q's Recommended Operating Conditions caps **TA** (ambient) at 125°C, with TJ abs max separately stated at 150°C and no conflicting figure elsewhere in the datasheet — the same clean structure LM2940-N had. Using RθJA = 45.0°C/W (JEDEC-reference) and TJ=150°C: $P_{d,max}(70°C) = (150-70)/45.0 = 1.778W$. Worst-case dissipation (1.6W, same physics as any LDO at this operating point) clears this with **~10% margin, without needing the still-unverified heatsink convection number to close the gap.** The heatsink/aluminum-board question (open item, see below) becomes pure upside here rather than a load-bearing assumption.
    - **Current margin — tight on paper, but 400mA is a hardware ceiling, not a worst-realistic-case estimate.** ICL (guaranteed minimum current-limit trip point) = 451mA against the 400mA design load — ~12.7% margin. This margin is real and permanent, not usage-dependent: each channel's current is set by its own BCR421UFDQ + Rext (100mA), which regulates independently of whatever WLED commands — PWM duty cycle, brightness level, or which combination of channels are active. There is no WLED command, effect, preset, or firmware state that can push any channel's current above its Rext-set value, so 400mA is the actual, hardware-enforced total ceiling under all possible operating conditions, not an estimate that depends on usage patterns being well-behaved.
- **Master Inhibit / HSS (new vs. DRL):** CSD25501F3 P-FET in series on the rail, $\text{R}_{ds(on)} = 64\text{m}\Omega$. Gate pulled low by default (FET on, LEDs powered); CLL (headlight) power pulls gate high → FET off → LED power cut. This is a single, board-level power gate — separate from the per-channel switch FETs described below. **See [Merge Notes / Open Questions](#merge-notes--open-questions) regarding the CLL-as-both-supply-and-trigger conflict.**

### Constant Current Channel Design (DemonEye)

Same core topology as the DRL board (see [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology)), with one structural change: 3 LED packages in series per channel instead of 1 package per set of channels.

- **One WS2814F** drives all 4 channels (R, G, B, W) for the module (swapped from WS2814B — package too large for this board's footprint; WS2814F confirmed 8-pin package). Pin names are the same as WS2814B (OUTR/OUTG/OUTB/OUTW/GND/DOUT/DIN/VDD), and VDD range (3.7–5.3V) plus OUTx sink current (16.5mA typ, 15.5–17.5mA range) match WS2814B, so the R5/C1 series-resistor-into-VDD network carries over from DRL unchanged.
- **Data order difference (WS2814F vs. WS2814B) — requires OUTx→LED pin reassignment:** WS2814B latches its 32-bit frame in R,G,B,W byte order; WS2814F latches in **W,R,G,B** order. To keep both boards receiving identical RGBW-ordered packets from the WLED controller (no firmware change), DemonEye's OUTx pins must be cross-wired to the LED color inputs as follows:
  - OUTW → LED **Red** input
  - OUTR → LED **Green** input
  - OUTG → LED **Blue** input
  - OUTB → LED **White** input

  (Each WS2814F output pin ends up driving the color whose data byte actually lands in that pin's latch position, given the WRGB frame order.)
- **Per channel current path:** LDO rail (12V) → LED1 → LED2 → LED3 (same color, 3 packages in series) → BCR421UFDQ OUT pin → Rext → GND. Identical regulation topology to DRL; the difference is 3 series LED voltage drops in the current path instead of 1.
- **Channel current (confirmed):** 100mA, equal across all 4 channels (R, G, B, W) — reduced from an earlier 125mA design point to improve thermal margin across LED packages, BCR421UFDQ, and the LDO. See [Thermal Management (DemonEye)](#thermal-management-demoneye) for the derivation. Total LDO output current = 4 × 100mA = 400mA (89% of TPL8051Q's 450mA rating — tight, but accepted; see Power Architecture for the design-usage justification).
- **Per-channel switch FET:** CSD25501F3, used identically to DRL — gates the BCR421UFDQ EN pin, driven by WS2814F OUTx through the same RC network / signal-inversion logic as DRL. One CSD25501F3 per channel (4 total), separate from the master Inhibit/HSS FET.
- **LED package:** QLSP06RGBXWAJ (same part as DRL — see [Shared Circuit Building Blocks](#qlsp06rgbxwaj-led-package)), ×3 per board (not ×1 per channel-set — 3 physical packages shared across all 4 channels).

**Open / pending for this section:**
- Per-channel target current is now set at 100mA. **Rext value itself still needs calculation** from the BCR421UFDQ REXT vs. IOUT graph for 100mA — not yet pulled.
- The previously-listed $R_{sense} = 5.6\Omega$ value is **legacy from a different design and should be disregarded.**
- At 100mA and a 12V rail, headroom checks out for typical Vf across the board (BGW: 3×3.2V + 1.4V = 10.6V ≤ 12V; Red: 3×2.2V + 1.4V = 8V ≤ 12V), using the correct $V_{OUT(MIN)}$=1.4V spec per [Shared Circuit Building Blocks](#bcr421ufdq--csd25501f3-constant-current-channel-topology) — full worst-case-bin verification against $V_{OUT(MIN)}$ still pending as a formal check, though the worst-case power numbers are captured in thermal management below.

### Thermal Management (DemonEye)

The compact PCB area requires strategic heat distribution to avoid localized hotspots. Distributing voltage drop across 3 series LED packages per channel is the primary thermal strategy vs. concentrating dissipation in the LDO. Board is aluminum-core with a 15×15mm heatsink on the back.

**LED Package Thermal Budget (QLSP06RGBXWAJ):**

Design point: 100mA per channel, equal across R/G/B/W, evaluated at $T_{amb} = 70°C$. (Reduced from an earlier 125mA design point for improved margin — see below.)

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

**LDO (TPL8051Q) Dissipation:**

- **Total LDO output current:** 4 channels × 100mA = 400mA (89% of TPL8051Q's 450mA rated max; ~12.7% margin below the 451mA guaranteed current-limit trip point). This is a hardware ceiling, not a usage estimate — each channel's current is set independently by its own BCR421UFDQ + Rext regardless of WLED's commands, so 400mA is the true maximum under any possible operating condition, not just a typical or expected one.
- **Worst case (16V battery, 12V rail):** $V_{drop}$ = 4V → $P = 4V \times 0.4A = 1.6W$ (same physics as any LDO at this Vin/Vout/current point).
- **Thermal budget — clears outright, no heatsink credit needed:** using TPL8051Q's RθJA (45.0°C/W, JEDEC-reference) and TJ abs max (150°C, cleanly stated with no conflicting recommended-operating figure — see Power Architecture for why this reading is well-supported for this part specifically): $P_{d,max}(70°C) = (150-70)/45.0 = 1.778W$. The 1.6W worst case clears this with **~10% margin on the bare JEDEC reference board.** This is the first LDO candidate in this search where the thermal case doesn't depend on the still-unverified aluminum-board/heatsink assumption to close.
- **Best case (engine running, ~13.6–14.4V battery):** $V_{drop}$ ≈ 1.6–2.4V → $P ≈ 0.64$–$0.96W$ — comfortably within budget.
- Engine-off dropout condition (below ~12.7-13V, estimated from the 3.3V/5V dropout spec at 400mA) reduces LDO dissipation further at the cost of some LED dimming — self-limiting in the direction that helps thermal margin, same pattern as every LDO candidate evaluated.
- **Load-dump event:** TPL8051Q's 42V continuous/45V transient rating covers the ~35V suppressed load-dump ceiling directly — LEDs should stay lit through the transient, same mechanism as TPS7A19, not LM2940-N's go-dark OVSD behavior.

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
| **TPL8051ADQ-DFCR-S** | LDO Regulator (adjustable, AEC-Q100 Grade 1) | DFN3X3-8, 3.00×3.00mm nominal; $V_{FB}$=1.215V typ; 3–42V in (45V transient max); 450mA rated; target $V_{out}$=12V via R1=88.8kΩ/R2=10kΩ; ±2% output accuracy; requires ceramic-compatible output cap (0.001-5Ω ESR window — MLCC-friendly); JLCPCB catalog part | https://static.3peak.com/res/doc/ds/Datasheet_TPL8051Q.pdf | 1 |
| **B5819WS-SL** | Series rectifier (reverse-polarity protection, power input line) | SOD-323; 1A IF(AV) continuous (2× margin over 500mA std. operating current); VRRM=30V; Schottky, low Vf (~0.4-0.5V @400-500mA); **required** since TPL8051Q has no built-in reverse-polarity protection | (JLCPCB catalog) | 1 |
| Output capacitor (TPL8051Q OUT pin) | Bulk/stability cap | 10µF (datasheet-recommended, within 2.2-200µF range), X7R MLCC — no meaningful ESR floor per datasheet (0.001-5Ω), consistent with "MLCCs only" board convention; **exact part/package TBD** | | 1 |
| **CSD25501F3** | Master Inhibit / HSS FET (rail disable via headlight-beam signal — gate source TBD, see [Merge Notes](#merge-notes--open-questions) and Open Items) | $\text{R}_{DS(on)} = 64\text{m}\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$; Source confirmed downstream of the LDO (only sees regulated ~12V rail, not raw line) | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf | 1 |
| **CSD25501F3** | Per-channel switch FET (gates BCR421UFDQ EN, same use as DRL) | $\text{R}_{DS(on)} = 64\text{m}\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$ | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf | 4 |
| **BCR421UFDQ** | Constant-current regulator, per channel | $R_{INT}$ = 95Ω typ; $V_{OUT(MIN)}$ = 1.4V typ ($I_{OUT}$>18mA) | (project file) BCR420UFDQBCR421UFDQ.pdf | 4 |
| **QLSP06RGBXWAJ** | RGBW LED package, ×3 in series per channel | same part as DRL board — see [Shared Circuit Building Blocks](#qlsp06rgbxwaj-led-package) | (project file) qlsp06rgbxwaj274_v1_0.pdf | 3 |
| **WS2814F** | LED controller, drives all 4 channels | 8-pin; VDD 3.7–5.3V; OUTx sink 16.5mA typ; data frame order **WRGB** (vs. WS2814B's RGBW) — requires OUTx→LED pin reassignment, see above | (project file) WS2814F.pdf | 1 |
| $\text{R}_{ext}$ | Current Sense (per channel) | Target 100mA/channel — **value TBD**, needs REXT vs. IOUT graph lookup (was 5.6Ω, flagged legacy/disregard) | | 4 |
| **D15V0H1U2LP-7B** | TVS (power input line) | X1-DFN1006-2 (~1.07×0.67mm); VRWM=15V; VBR=16-19V typ @1mA; VCL=22V@5A/27V@12A (8/20µs); sized for routine ISO 7637-2 transients only — load-dump survival handled by TPL8051Q's wide continuous input range, not this part; not AEC-Q qualified (consumer ESD-protection line) | (project file) D15V0H1U2LP.pdf | 1 |
| **SZESD7241N2T5G** (×2) | ESD protection, DIN and DOUT | X2DFN-2, single-channel/2-pin per part (not a shared array — needs one per line); AEC-Q101, VRWM=24V, VC=48V max; matches DRL's DATA-line protection approach applied to both connector-exposed data pins | https://www.digikey.com/en/products/detail/onsemi/SZESD7241N2T5G/7221019 | 2 |
| Rectifier (power input line) | **Not yet selected** | — | — | — |

### Interface Connector (DemonEye)

5- or 6-pin picoBlade header (exact pin count pending — see note below). Unlike the DRL board's connector (DATA/GND/GND/VBAT, 4 pins), DemonEye needs both a data input and a data output, since it is designed to sit in the WS281x chain ahead of the DRL board rather than being the last device on the chain.

| Pin | Signal | Notes |
|-----|--------|-------|
| — | DIN | WS2814F data input, from WLED/ESP32-S3 (or from an upstream board, if DemonEye is not first in the chain) |
| — | DOUT | WS2814F data output, feeds DRL board's DIN |
| — | GND | Data/logic ground |
| — | VBAT | Power input — **tap point not yet finalized, see [Merge Notes / Open Questions](#merge-notes--open-questions)** |
| — | GND | Power return (if a separate pin from data GND, mirroring the DRL board's split data-GND/power-GND scheme) |
| — | *(possible 6th pin)* | If the CLL master-inhibit trigger turns out to need a dedicated sense line (rather than being derivable from DemonEye's own VBAT, per Merge Note 1), it would need its own pin here |

**Pin assignments, exact count, and connector part number not yet finalized** — this table lists the known required signals, not a confirmed schematic-matched pinout.

---

## WLED / Control Architecture (Shared)

- **Controller:** ESP32-S3 running WLED — separate board, independently powered, sends WS281x data to both the DRL board and DemonEye board
- **Protocol:** WS281x single-wire to WS2814B (DRL) / WS2814F (DemonEye) chain
- **DRL mode:** White channel full brightness + optional RGB trim
- **Turn signal:** RGB amber (Red+Green) sequential animation — supplementary to OEM turn signal and fender side markers; no regulatory brightness requirement
- **DRL board on/off:** Board powered from CLL relay. Confirmed by observation that CLL remains active with headlights off, consistent with correct DRL behavior (board should be powered whenever DRL is meant to operate, independent of headlight state). WLED sends data regardless; board simply doesn't respond when unpowered.
- **DemonEye board on/off:** Intended to be powered by default and cut by a CLL-derived "headlights on" signal — **see [Merge Notes / Open Questions](#merge-notes--open-questions)**, since CLL is now understood to be the same net used as the DRL board's primary power supply, not a separately-distinguishable headlight-specific signal.
- **PWM frequency:** WS2814x OUTx at 2kHz — above flicker threshold, within BCR421 25kHz EN pin limit
- **Flicker:** Not a concern on DRL — 200kHz buck + 2kHz WS2814B PWM + BCR421 linear regulation = triple isolation from input ripple to LED current. DemonEye's equivalent isolation chain (LDO or buck front end, still TBD) not yet analyzed.

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
| DemonEye power tap / CLL conflict | DemonEye | **Partially resolved — new TBD** | Confirmed CLL cannot double as both DemonEye's power source and a "headlights on" trigger (same conflict as originally flagged). Master inhibit FET's Source confirmed downstream of the LDO (only sees regulated ~12V rail, not raw line — no protection concern there). **Gate signal source identified but not yet implemented:** low beam and high beam relay signals are separate wires in the main headlight connector (B121/B122) — not present on the shared 8-pin DRL driver board connector DemonEye's power comes from. Requires a new, separate wire tap into B121/B122, not available "for free" from the existing power connector. **Still TBD:** voltage/polarity of the beam relay signals (switched-hot vs. switched-ground, same ambiguity as DRL/CLL before measurement), OR-logic to combine low+high beam into one gate signal, and the physical wire run itself. |
| DemonEye LDO selection | DemonEye | **Decided (final)** | **TPL8051Q (TPL8051ADQ-DFCR-S), DFN3X3-8, adjustable, AEC-Q100 Grade 1 — selected over both TPS7A19 and LM2940-N.** Reasons: (1) genuinely AEC-Q100 qualified, the only candidate in this search to clear that bar (both TPS7A19 and LM2940-N lacked formal automotive qualification); (2) 42V continuous/45V transient input covers the ~35V suppressed load-dump ceiling directly, maintaining regulation through the event (same mechanism as TPS7A19, better than LM2940-N's go-dark OVSD); (3) MLCC-compatible output cap (0.001-5Ω ESR window), avoiding LM2940-N's forced tantalum; (4) 3×3mm package, matching the original size target; (5) **thermal budget clears outright using the bare JEDEC-reference RθJA (45.0°C/W) and the part's cleanly-stated 150°C TJ abs max — ~10% margin (1.6W worst case vs. 1.778W budget) without needing the still-unverified aluminum-board/heatsink assumption to close, unlike every prior candidate.** **Trade accepted:** no built-in reverse-polarity protection (series rectifier B5819WS-SL required, same as TPS7A19); current margin is tight (400mA against 450mA rated, ~12.7% margin below the 451mA guaranteed current-limit trip point) — but this is a hardware ceiling, not a usage estimate: each channel's current is set independently by its own BCR421UFDQ + Rext regardless of what WLED commands, so 400mA is the true maximum under any possible operating condition, not just an expected typical case. Sourcing confirmed available via JLCPCB catalog (exact stock quantity not verified). |
| DemonEye TVS/rectifier selection | DemonEye | **Decided (both)** | **TVS: D15V0H1U2LP-7B** (X1-DFN1006-2, VRWM=15V, VBR=16-19V, VCL=22V@5A/27V@12A) — sized for routine ISO 7637-2 transients only; load-dump survival now handled by TPL8051Q's wide continuous input range (42V) rather than an LDO shutdown mechanism, but the TVS's own scope is unchanged. Confirmed no DFN1006-class part (including automotive-grade options) carries a genuine ISO 7637-2 pulse rating (checked Taiwan Semiconductor's automotive DFN1006-2LW line specifically) — accepted as adequate for the "routine transient" scope. **Rectifier: B5819WS-SL** (SOD-323 Schottky, 1A, 30V VRRM) — required because TPL8051Q has no built-in reverse-polarity protection; same part identified earlier in the original small-rectifier search. |
| DemonEye heatsink convection performance | DemonEye | **Pending — no longer load-bearing for the LDO, still relevant elsewhere** | Heatsink: 14×20×6mm aluminum, 7 vertical fins along the 20mm direction, attached via 0.254mm/1 W/m·K thermal tape. With TPL8051Q selected, the LDO's own thermal budget clears using the bare JEDEC-reference RθJA alone — this heatsink question is no longer required to close that particular margin. It remains relevant for the BCR421 (Red channel worst-case 600mW) and LED package (108.9°C/18.8°C-margin) thermal calls made earlier in this document, which still lean on the general aluminum-board-helps assumption. An early attempt at calculating fin convection performance used an invalid fin-spacing rule of thumb and was retracted — no trustworthy number exists yet. Needs either a correctly-scaled natural-convection correlation or prototype measurement if those other margins are ever worth tightening up, but is no longer urgent the way it was when the LDO thermal case depended on it. |
| DemonEye Rext value | DemonEye | **Pending** | Target 100mA/channel; needs REXT vs. IOUT graph lookup from BCR421UFDQ datasheet |
| DemonEye PCB layout | DemonEye | **Pending** | Fit WS2814F, 4× BCR421, 5× CSD25501F3, 3× LED packages, TVS/rectifier (TBD part), LDO/buck (TBD part), and 5/6-pin picoBlade header within 16mm×20mm |
| DemonEye connector pinout | DemonEye | **Pending** | Needs DIN, DOUT, VBAT, and at least one GND at minimum (data pass-through to DRL board requires DOUT, unlike DRL's connector); exact pin count (5 vs. 6) depends on whether the CLL master-inhibit trigger needs a dedicated pin — see Merge Notes |
| DemonEye DIN/DOUT ESD protection | DemonEye | **Decided, default** | Both DIN and DOUT to get their own SZESD7241N2T5G (or equivalent), matching DRL's single-line protection philosophy applied to both connector-exposed data pins (2 parts total, since the part is single-channel/2-pin, not a shared array). **Fallback if board space runs out:** drop DOUT's dedicated TVS and rely on WS2814F's native pin ESD rating (~2kV HBM class) for that line only — DIN keeps its TVS regardless. |

---

## Merge Notes / Open Questions

Two things surfaced while combining these documents that weren't fully resolved in either source document individually — flagging both explicitly rather than silently picking an answer:

1. **DemonEye's power tap vs. its "CLL trigger" mechanism may now be in direct conflict.** DemonEye's design (written before the CLL/DRL pin correction) describes two things that were previously assumed independent: (a) DemonEye's own Vbat input, originally described as "same OEM DRL feed as the DRL board," and (b) a separate "CLL (headlight) power signal" used purely as a logic-level trigger to gate the master inhibit FET off when headlights are on. Given this session's finding that **DRL is not a power-carrying net at all**, and **CLL is the actual shared power supply**, DemonEye's own power tap almost certainly also needs to move to CLL — the same correction just made for the DRL board. But if DemonEye's own supply *is* CLL, it can no longer also treat CLL as a distinguishable "headlights are on, turn yourself off" signal, since by definition CLL would always be present whenever the board itself has power. This needs a real design decision: either DemonEye needs a genuinely different signal to detect "headlights on" (if such a signal exists on this connector or elsewhere), or the master-inhibit-on-headlights concept needs to be rethought entirely for DemonEye specifically.

2. **BCR421UFDQ's two voltage specs ($V_{DROP}$ vs. $V_{OUT(MIN)}$) are correctly used for two different purposes in the current documents, contrary to an earlier working assumption in this project that they were being conflated.** The REXT-setting formula correctly uses $V_{DROP}$ = 0.95V typ; headroom/margin checks correctly use $V_{OUT(MIN)}$ = 1.4V typ. Flagging this so it doesn't get "fixed" incorrectly in a future pass — worth a final check that every instance in this document uses the right one for its purpose, but no error was found in this merge.

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
| TPL8051Q datasheet (DA20250302A0) | https://static.3peak.com/res/doc/ds/Datasheet_TPL8051Q.pdf |
| CSD25501F3 datasheet | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf |
| Samsung MLCC catalog (MLCC_2512.pdf) | Project |
| Vishay T15B22A datasheet (98360) | Project |
| S4PM/S4PJ datasheet | Project |
| SZESD7241 datasheet | Project |
| D15V0H1U2LP datasheet (DemonEye TVS candidate, evaluated and not yet finalized) | Project |
| Wurth 744393465120 datasheet | https://www.we-online.com/components/products/datasheet/744393465120.pdf |
| MAX16833C datasheet | https://www.analog.com/media/en/technical-documentation/data-sheets/MAX16833-MAX16833G.pdf |
| WEBENCH Design Report — LMR51460 12V-16V to 5.50V @ 6A | TI WEBENCH, July 2026 |
