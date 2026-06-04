# LED Driver Module Design Specification

## 1. Overview
This module is an ultra-compact ($16\text{mm} \times 14\text{mm}$) aluminum-core PCB designed for automotive DemonEye applications. It utilizes an LDO to provide a stable rail and four independent constant-current channels driven by WS2814 logic.

## 2. Power Architecture
* **Input:** Valid Automotive $V_{bat}$ ($11.8\text{V} - 15.6\text{V}$). Voltages $>15.6\text{V}$ are clamped by an onboard TVS diode.
* **Regulation:** TLV767 adjustable LDO ($V_{FB} = 0.8\text{V}$).
    * **Target Output ($V_{out}$):** 11.7V
    * **Feedback Divider:** Initial divider ratio $K = \frac{V_{FB}}{V_{out}}$. To hit a specific target, choose $R_{top}$ and $R_{bottom}$ such that:
      *  $R_{top} = R_{bottom} \cdot (\frac{V_{out}}{0.8\text{V}} - 1)$
      * $R1 + R2 \leq \frac{V_{out}}{(I_{FB} \times 100)}$, where $I_{FB} = 50nA$
      * $R1 + R2 \leq 2.34 M\Omega$
    * $R_{top}$ = 1.43 MΩ
    * $R_{bottom}$ = 105 kΩ
* **Inhibit:** A series PFET (on the rail) provides global disable functionality ($R_{DS(on)}$ = 64 mΩ).

## 3. Constant Current Channel Design
```
                    ┌───────┐                                   
       ┌────────────┤ 5R6 Ω ├──────────┐                         
       │            └───────┘          │                         
       │                               │                         
 Vin   │    ┌───────┐       ┌───────┐  │  ┌───────────┐     Iload 
●──────┼────┤ 4k7 Ω ├───┬───┤ 511 Ω ├──┴──┤ S       D ├──────────●
       │    └───────┘   │   └───────┘     │           │           
       │                │                 │     G     │           
       │          ┌─────┴─────┐           └─────┬─────┘             
       │          │     B     │                 │                 
       │          │           │                 │                 
       └──────────┤ E       C ├─────────────────┤                 
                  └───────────┘                 │                 
                                                │                 
 OUTX               ┌───────┐                   │                 
●───────────────────┤  1kΩ  ├───────────────────┘                 
                    └───────┘                                   
```
The load consists of **3 series-connected RGBW LED packages**, creating 1 channel of 3 LEDs for each color. Each of the 4 channels utilizes a temperature-compensated BJT/NTC-based feedback loop to regulate current through a PFET. 

* **Topology:** An NTC thermistor and base resistor create a temperature variable voltage divider across $R_{sense}$. The BJT (BC807-25QA) senses the voltage drop across the NTC to control the gate of the PFET (CSD25501F3).
* **Regulation Mechanism:** The BJT shunts the PFET gate, fighting the internal 10 kΩ pull-up to maintain $V_{BE(on)}$ across $R_{ntc}$, thereby maintaining a constant pre-defined current ($\pm$ 2%)

### Headroom Analysis
The total voltage drop across the CC block must be sufficient to keep the BJT in its active region and the PFET in saturation.
* **Required Drop:** $\text{Headroom} = V_{BE} + V_{DS(sat)}$. 
* **Worst-Case LED $V_f$ (Max):**
    * **Red String:** $3 \times 2.4\text{V} = 7.2\text{V}$
    * **Other Channels:** $3 \times 3.6\text{V} = 10.8\text{V}$
* **Worst-Case LED $V_f$ (Min):**
    * **Red String:** $3 \times 2.0\text{V} = 6.0\text{V}$
    * **Other Channels:** $3 \times 2.8\text{V} = 8.4\text{V}$
* **$R_{sense}$ Selection:** Currently under evaluation to balance power dissipation against the $V_{BE}$ regulation threshold.



## 4. Thermal Management Strategy
The compact PCB area requires strategic heat distribution to avoid localized hotspots on the FemtoPFETs.

### Red Channel Ballast Array
Because the red LED channels have the lowest combined forward voltage, the CC channel must drop the largest amount of excess voltage. With $V_{out} = 11.7\text{V}$ and worst-case Red $V_{f} = 6.0\text{V}$, there is approx. $5.95\text{V}$ to dissipate between the PFET and ballast resistor bank.
* **Strategy:** Utilize a bank of 5 parallel 100mW 0402 resistors to dissipate heat.
* **Benefit:** Shifts power dissipation from the ultra-small PFET package to the board's surface-mount resistor array.

## 5. Component Summary Table

| Component | Role | Key Spec | Datasheet |
| :--- | :--- | :--- | :--- |
| **TLV767** | LDO Regulator | $V_{FB} = 0.8\text{V}$; Dropout: $0.3\text{V}-0.5\text{V}$ | https://www.ti.com/lit/ds/symlink/tlv767.pdf |
| **CSD25501F3** | CC PFET | $R_{DS(on)} = 64m\Omega$; $V_{GS(th)} = 0.75\text{V}$ | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf |
| **BC807** | BJT Controller | $V_{BE(on)} \approx 0.5\text{V} for I_{C} = 0.1mA @ 25°\text{C}$, $\approx 0.25\text{V} for I_{C} = 10mA @ 100°\text{C}$ | https://assets.nexperia.com/documents/data-sheet/BC807-25QA_40QA.pdf |
| **NCP15XM472J03RC** | NTC Thermistor | 4.7 kΩ, β(25/100)=3560K | https://mm.digikey.com/Volume0/opasdata/d220001/medias/docus/8920/NTC%20Thermistors_83.pdf | 
| **$R_{BASE}$** | Offset for BJT's base votage divider | 243 Ω ||
| **$R_{sense}$** | Current Sense | Pending final calculation |

