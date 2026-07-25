# LED Driver Module Design Specification — DemonEye
*Last updated: July 2026 — channel current, thermal margins, and LDO target confirmed; Rext values and BCR421/LED headroom math against final LDO setpoint still pending*
 
## 1. Overview
This module is a compact aluminum-core PCB for automotive DemonEye applications. Board target is $15\text{mm} \times 15\text{mm}$ for the LED/mounting area, open to variation depending on final fit; overall board is closer to $15.5\text{mm} \times 35\text{mm}$, with the additional ~5mm of length dedicated to the 5-pin connector.
 
The module uses a TLV767 LDO to provide a stable, known regulated rail, and a **single WS2814F** (WS2814B ruled out — package too large for this board) driving **4 independent constant-current color channels (R, G, B, W)**. Each channel is formed by **3 RGBW LED packages wired in series** (same color across all 3 packages), rather than 3 independently-addressable packages — the 3 packages are treated as a single 4-color light source, not 3 separate light sources. Only one WS2814F is needed since a series-wired string can't be independently addressed per package anyway.
 
**Design rationale (series LED packages):** the LDO provides a stable, known voltage to the LED string. Stacking 3 LED packages per channel in series drops the bulk of the excess voltage across the LEDs themselves (distributing heat across 3 packages) rather than having the LDO dissipate it all (~10V-ish if it had to drop the full differential itself). The BCR421UFDQ continues to regulate the actual channel current, same role as on the DRL board.
 
**Addition vs. DRL board:** a master P-FET, configured as a high-side switch (HSS), gates power to the entire LED circuit. Gate is pulled low by default (FET on, LEDs powered); the CLL (headlight) power signal from the OEM driver board pulls the gate high, turning the FET off and cutting power to the LEDs whenever the headlights are on.
 
## 2. Power Architecture
* **Input:** Same OEM DRL feed as the DRL board. Valid automotive $\text{V}_{bat}$ (11.8V–15.6V). Voltages >15.6V clamped by an onboard TVS diode.
* **Regulation:** TLV767 adjustable LDO ($\text{V}_{FB} = 0.8\text{V}$).
    * **Target Output ($\text{V}_{out}$):** 12V. At this setpoint, the LDO needs ~0.8–0.9V of headroom above $V_{out}$ to stay in regulation. With $V_{bat}$ 12–16V (engine running), this is comfortably met. With engine off (battery sag toward ~12V or below) and lights on, the LDO will drop into dropout — accepted behavior, results in mild dimming rather than malfunction, and also reduces heat from downstream components under that condition.
    * **Worst-case thermal condition:** max battery voltage (16V) with the LDO in full regulation at 12V — this simultaneously maximizes LDO dissipation (4V drop) and Red-channel BCR421UFDQ dissipation (fixed 12V rail against Red's minimum-Vf-bin string voltage). See Section 4.
    * **Feedback Divider:** $\text{R}_{top} = \text{R}_{bottom} \cdot (\frac{\text{V}_{out}}{V_{FB}} - 1)$; $R_{top} + R_{bottom} \leq 2.34\text{M}\Omega$ (per $I_{FB} = 50\text{nA}$ constraint), targeting $R_{top}+R_{bottom} \approx 10\%$ of $R_{max}$:
      * $\text{R}_{top} = 221\text{k}\Omega$
      * $\text{R}_{bottom} = 16.2\text{k}\Omega$
* **Master Inhibit / HSS (new vs. DRL):** CSD25501F3 P-FET in series on the rail, $\text{R}_{ds(on)} = 64\text{m}\Omega$. Gate pulled low by default (FET on, LEDs powered); CLL (headlight) power pulls gate high → FET off → LED power cut. This is a single, board-level power gate — separate from the per-channel switch FETs described in Section 3.
## 3. Constant Current Channel Design
Same core topology as the DRL board, with one structural change: 3 LED packages in series per channel instead of 1 package per set of channels.
 
* **One WS2814F** drives all 4 channels (R, G, B, W) for the module (swapped from WS2814B — package too large for this board's footprint; WS2814F confirmed 8-pin package). Pin names are the same as WS2814B (OUTR/OUTG/OUTB/OUTW/GND/DOUT/DIN/VDD), and VDD range (3.7–5.3V) plus OUTx sink current (16.5mA typ, 15.5–17.5mA range) match WS2814B, so the R5/C1 series-resistor-into-VDD network carries over from DRL unchanged.
* **Data order difference (WS2814F vs. WS2814B) — requires OUTx→LED pin reassignment:** WS2814B latches its 32-bit frame in R,G,B,W byte order; WS2814F latches in **W,R,G,B** order. To keep both boards receiving identical RGBW-ordered packets from the WLED controller (no firmware change), DemonEye's OUTx pins must be cross-wired to the LED color inputs as follows:
  * OUTW → LED **Red** input
  * OUTR → LED **Green** input
  * OUTG → LED **Blue** input
  * OUTB → LED **White** input
  (Each WS2814F output pin ends up driving the color whose data byte actually lands in that pin's latch position, given the WRGB frame order.)
* **Per channel current path:** LDO rail (12V) → LED1 → LED2 → LED3 (same color, 3 packages in series) → BCR421UFDQ OUT pin → Rext → GND. Identical regulation topology to DRL (BCR421UFDQ sets constant current via Rext, independent of rail voltage given adequate headroom); the difference is 3 series LED voltage drops in the current path instead of 1.
* **Channel current (confirmed):** 125mA, equal across all 4 channels (R, G, B, W) — see Section 4 for thermal derivation. Total LDO output current = 4 × 125mA = 500mA, exactly half of the TLV767's 1A rating.
* **Per-channel switch FET:** CSD25501F3, used identically to DRL — gates the BCR421UFDQ EN pin, driven by WS2814F OUTx through the same RC network / signal-inversion logic as DRL (OUTx sinks → gate pulled low → Vgs negative → PFET on → EN high → channel current flows; OUTx Hi-Z → RC pulls gate to source → PFET off → channel off). One CSD25501F3 per channel (4 total), separate from the master Inhibit/HSS FET in Section 2.
* **LED package:** QLSP06RGBXWAJ (same part as DRL), ×3 per board (not ×1 per channel-set — 3 physical packages shared across all 4 channels).
**Open / pending for this section:**
- Per-channel target current is now set at 125mA (see above / Section 4). **Rext value itself still needs calculation** from the BCR421UFDQ REXT vs. IOUT graph for 125mA — not yet pulled.
- The previously-listed $R_{sense} = 5.6\Omega$ value is **legacy from a different design and should be disregarded.**
- **BCR421UFDQ headroom spec correction:** the DRL design summary's per-channel margin notes cite $V_{DROP} = 0.95\text{V typ}$, which is the datasheet's 10mA test-condition spec. The applicable spec at real operating currents ($I_{OUT} > 18\text{mA}$, which covers all DRL and DemonEye channel currents) is **$V_{OUT(MIN)} = 1.4\text{V typ}$** (minimum voltage the BCR421UFDQ needs across itself to stay in regulation). This affects both DemonEye's 3-series-LED headroom math and DRL's existing single-LED margin figures — full analysis pending.
- At 125mA and a 12V rail, headroom checks out for typical Vf across the board (BGW: 3×3.2V + 1.4V = 10.6V ≤ 12V; Red: 3×2.2V + 1.4V = 8V ≤ 12V) — full worst-case-bin verification against $V_{OUT(MIN)}$ still pending as a formal check, though the worst-case power numbers are captured in Section 4.
## 4. Thermal Management Strategy
The compact PCB area requires strategic heat distribution to avoid localized hotspots. Distributing voltage drop across 3 series LED packages per channel (Section 1 rationale) is the primary thermal strategy vs. concentrating dissipation in the LDO. Board is aluminum-core with a 15×15mm heatsink on the back.
 
### 4.1 LED Package Thermal Budget (QLSP06RGBXWAJ-274)
Design point: **125mA per channel, equal across R/G/B/W**, evaluated at $T_{amb} = 70°C$.
 
* **Derived thermal resistance** (from datasheet Pd = 3600mW @ Ta=25°C, $T_{J,max}$ = 120°C):
$$R_{th(JA)} = \frac{T_{J,max} - T_a}{P_d} = \frac{120-25}{3.6W} = 26.4°C/W$$
* **Available power budget at 70°C ambient:**
$$P_{d(70°C)} = \frac{120-70}{26.4} = 1895mW \text{ per package}$$
* **Actual dissipation at 125mA, typical Vf (Red=2.2V, Blue/Green/White=3.2V, ΣVf=11.8V)** — valid to sum this way *because* all 4 channels share the same current, so $P = I_R V_{f,R} + I_G V_{f,G} + I_B V_{f,B} + I_W V_{f,W}$ factors to $I \times \Sigma V_f$:
$$P = 0.125A \times 11.8V = 1475mW$$
* **Resulting junction temp:** $T_J = 70 + (1.475 \times 26.4) = 108.9°C$ → **11.1°C margin below $T_{J,max}$ (≈22% power margin below the 1895mW budget).**
This applies per physical package — each of the 3 series-connected packages dissipates independently from its own 4 dies; the series connection shares current, not heat, across packages.
 
### 4.2 BCR421UFDQ Dissipation (per channel, $P = V_{OUT} \times I_{OUT}$, at 12V rail)
* **Green / Blue / White:** $V_{OUT}$ = 1.4V (rail 12V − typ. string 3×3.2V=9.6V, minus the 1.4V regulation node itself lands consistently with datasheet $V_{OUT(MIN)}$ typ) → $P = 1.4V \times 0.125A = 175mW$ each (525mW combined for 3 channels).
* **Red — worst case (16V battery, LDO in full regulation at 12V, Red string at min-Vf bin 3×2.0V=6.0V):**
$$V_{OUT} = 12V - 6.0V = 6.0V \rightarrow P = 6.0V \times 0.125A = 750mW$$
This is the worst-case condition for the Red BCR421UFDQ, since max battery voltage keeps the rail pinned at 12V (LDO in regulation) while minimum Vf leaves the most voltage for the BCR421 to absorb.
* **BCR421UFDQ package rating context:** datasheet gives PD = 1.3W/RθJA=100°C/W (25×25mm 1oz Cu) or PD=1.7W/RθJA=75°C/W (50×50mm 1oz Cu), both on standard FR4 — worse thermal performance than this board's aluminum-core + dedicated 15×15mm heatsink construction. No MCPCB-specific RθJA figure available from the datasheet; real performance on this board is expected to be meaningfully better than the FR4 curves suggest. **Confirmed acceptable per design review; not independently quantified via dielectric/heatsink Rth calc or prototype measurement.**
### 4.3 LDO (TLV767) Dissipation
* **Total LDO output current:** 4 channels × 125mA = 500mA (exactly half of TLV767's 1A rating).
* **Worst case (16V battery, 12V rail):** $V_{drop}$ = 4V → $P = 4V \times 0.5A = 2.0W$
* **Best case (engine running, ~12–13V battery):** $V_{drop}$ ≈ 0.8–1V → $P ≈ 0.4$–$0.5W$
* Engine-off dropout condition (Section 2) reduces LDO dissipation further (output sags with input, drop voltage shrinks) at the cost of some LED dimming — self-limiting in the direction that helps thermal margin.
### 4.4 Summary — Worst-Case Total Heat (16V battery, 12V rail, min-Vf Red bin)
| Source | Power |
|---|---|
| LED packages (3× combined, typical Vf) | ~4.4W (3 × 1475mW) |
| BCR421UFDQ (G+B+W, typ) | 525mW |
| BCR421UFDQ (Red, worst case) | 750mW |
| LDO (worst case) | 2.0W |
| **Total** | **~7.7W** |
 
Note: LED package figures use typical Vf (per-package self-heating, independent of rail/battery voltage) while BCR421/LDO figures use worst-case battery/Vf conditions — these are somewhat mismatched conditions layered for a conservative combined estimate, not a single fully self-consistent worst-case corner. Board-level thermal (shared heatsink loading from all sources simultaneously) not yet analyzed.
 
## 5. Component Summary Table
 
| Component | Role | Key Spec | Datasheet | Qty (per board) |
| :--- | :--- | :--- | :--- | :--- |
| **TLV767** | LDO Regulator | $\text{V}_{FB} = 0.8\text{V}$; $\text{V}_{DO} = 0.3\text{V}-0.5\text{V}$; target $V_{out}$=12V, $I_{out}$≈500mA (4×125mA) | https://www.ti.com/lit/ds/symlink/tlv767.pdf | 1 |
| **CSD25501F3** | Master Inhibit / HSS FET (rail disable via CLL signal) | $\text{R}_{DS(on)} = 64\text{m}\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$ | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf | 1 |
| **CSD25501F3** | Per-channel switch FET (gates BCR421UFDQ EN, same use as DRL) | $\text{R}_{DS(on)} = 64\text{m}\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$ | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf | 4 |
| **BCR421UFDQ** | Constant-current regulator, per channel | $R_{INT}$ = 95Ω typ; $V_{OUT(MIN)}$ = 1.4V typ ($I_{OUT}$>18mA) | (project file) BCR420UFDQBCR421UFDQ.pdf | 4 |
| **QLSP06RGBXWAJ** | RGBW LED package, ×3 in series per channel | (same part as DRL board) | (project file) qlsp06rgbxwaj274_v1_0.pdf | 3 |
| **WS2814F** | LED controller, drives all 4 channels | 8-pin; VDD 3.7–5.3V; OUTx sink 16.5mA typ; data frame order **WRGB** (vs. WS2814B's RGBW) — requires OUTx→LED pin reassignment, see Section 3 | (project file) WS2814F.pdf | 1 |
| $\text{R}_{ext}$ | Current Sense (per channel) | Target 125mA/channel — **value TBD**, needs REXT vs. IOUT graph lookup (was 5.6Ω, flagged legacy/disregard) | | 4 |