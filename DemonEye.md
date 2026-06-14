# LED Driver Module Design Specification

## 1. Overview
This module is an ultra-compact ($16mm \times 14mm$) aluminum-core PCB designed for automotive DemonEye applications. It utilizes an LDO to provide a stable rail and four independent constant-current channels driven by WS2814 logic.

## 2. Power Architecture
* **Input:** Valid Automotive $\text{V}_{bat} (11.8\text{V} - 15.6\text{V})$. $~~\text{Voltages} >15.6\text{V}$ are clamped by an onboard TVS diode.
* **Regulation:** TLV767 adjustable LDO $(\text{V}_{FB} = 0.8\text{V})$.
    * **Target Output ($\text{V}_{out}$):** $11.7 \text{V}$
    * **Feedback Divider:** Initial divider ratio $K = \frac{\text{V}_{FB}}{\text{V}_{out}}$. To hit a specific target, choose $\text{R}_{top}$ and $\text{R}_{bottom}$ such that:
      *  $\text{R}_{top} = \text{R}_{bottom} \cdot (\frac{\text{V}{out}}{V_{FB}} - 1)$
      * $R1 + R2 \leq \frac{\text{V}_{out}}{(I_{FB} \times 100)}$, where $I_{FB} = 50n\text{A}$
      * $R1 + R2 \leq 2.34 M\Omega$
    * targeting $\text{R}_{top} + \text{R}_{bottom} \approx 10\% ~ of ~ \text{R}_{max}$:
      * $\text{R}_{top}=221 k\Omega$
      * $\text{R}_{bottom}=16.2 k\Omega$
* **Inhibit:** A series PFET (on the rail) provides global disable functionality $(\text{R}_{ds(on)} = 64 m\Omega)$.

## 3. Constant Current Channel Design
```
                    ┌───────┐                                   
       ┌────────────┤ 5R6 Ω ├───────────┐                         
       │            └───────┘           │                         
       │                                │                         
 Vsw   │   ┌────────┐       ┌───────┐   │   ┌───────────┐     Iload 
●──────┼───┤4k7Ω NTC├───┬───┤ 511 Ω ├───┴───┤ S       D ├──────────●
       │   └────────┘   │   └───────┘       │           │           
       │                │                   │     G     │           
       │          ┌─────┴─────┐             └─────┬─────┘             
       │          │     B     │                   │                 
       │          │           │                   │                 
       └──────────┤ E       C ├───────────────────┤                 
                  └───────────┘                   │                 
                                                  │                 
 OUTx               ┌───────┐                     │                 
●───────────────────┤  1kΩ  ├─────────────────────┘                 
                    └───────┘                                   
```
The load consists of **3 series-connected RGBW LED packages**, creating 1 channel of 3 LEDs for each color. Each of the 4 channels utilizes a temperature-compensated BJT/NTC-based feedback loop to regulate current through a PFET. 

* $\text{V}_{sw} = \text{V}_{LDO} - \text{V}_{inh}$, where
    * $\text{V}_{LDO}$: voltage output from the LDO
    * $\text{V}_{inh}$: voltage drop across the master inhibitor PFET
* $\text{OUT}_{X} = \text{WS2814 sink (connected to PFET}_{GATE} \text{), }  \text{X} \in \left\{ R, B, G, W \right\}$

* **Topology:** An NTC thermistor and base resistor create a temperature variable voltage divider across $\text{R}_{sense}$. The BJT (BC807-25QA) senses the voltage drop across the NTC to control the gate of the PFET (CSD25501F3).
* **Regulation Mechanism:** The PNP BJT pulls the PFET gate high (toward $\text{V}_{in}$) to throttle channel conduction. In doing so, it fights the external low-side drive coming from the WS2814 OUTX line through the $1k\Omega$ isolation resistor. This active feedback maintains $\text{V}_{be(on)}$ across the $\frac{\text{R}_{ntc}}{\text{R}_{base}}$ network, ensuring a target constant current within a $\pm 2\%$ margin across the $75\degree\text{C}$ operating range.  

### Headroom Analysis
The total voltage drop across the CC block must be sufficient to keep the BJT in its active region and the PFET in saturation.
* **Required Drop:** $\text{Headroom} = \text{V}_{be} + \text{V}_{ds(sat)}$. 
* **Worst-Case LED $\text{V}_{f(max)}$:**
    * **Red String:** $3 \times 2.4\text{V} = 7.2\text{V}$
    * **Other Channels:** $3 \times 3.6\text{V} = 10.8\text{V}$
* **Worst-Case LED $\text{V}_{f(min)}$:**
    * **Red String:** $3 \times 2.0\text{V} = 6.0\text{V}$
    * **Other Channels:** $3 \times 2.8\text{V} = 8.4\text{V}$
* **$\textbf{R}_{sense}$ Selection:** See Section 6



## 4. Thermal Management Strategy
The compact PCB area requires strategic heat distribution to avoid localized hotspots on the FemtoPFETs.

### Red Channel Ballast Array
Because the red LED channel has the lowest possible combined forward voltage, the CC channel must drop the largest amount of excess voltage. With a nominal $\text{V}_{out} = 11.7\text{V}$ and Red $\text{V}_{f} = 6.6\text{V}$, there is approx. $5.1\text{V}$ that must be dissipated across the PFET and ballast resistor bank. However, when factoring in worst-case tolerances of design components, $\text{V}_{out}$ could vary as high as $11.95\text{V}$ while $\text{V}_{f(red)}$ could be as low as $6.0\text{V}_{total}$, resulting in a need to dissipate up to $5.95\text{V}$.
* **Strategy:** Implement a bank of 8 parallel $50m\text{W}$ 0201 resistors to dissipate heat which, with a $50\%$ engineered derating, can safely dissipate $2\text{V}$, lowering the power dissipation requirements of the PFET to well below its $500m\text{W}$ limit
* **Benefit:** Distributes power dissipation across all components more evenly, keeping all components below $70\%$ of their respective power ratings, especially the ultra-small PFET package, even with worst-case tolerance components.

## 5. Component Summary Table

| Component | Role | Key Spec | Datasheet |
| :--- | :--- | :--- | :--- |
| **TLV767** | LDO Regulator | $\text{V}_{FB} = 0.8\text{V}$; $\text{V}_{DO} = 0.3\text{V}-0.5\text{V}$ | https://www.ti.com/lit/ds/symlink/tlv767.pdf |
| **CSD25501F3** | CC PFET | $\text{R}_{DS(on)} = 64m\Omega$; $\text{V}_{GS(th)} = 0.75\text{V}$ | https://www.ti.com/lit/ds/symlink/csd25501f3.pdf |
| **BC807** | BJT Controller | $\text{V}_{be(on)}$<BR> $\approx 0.5\text{V}  @ 25\degree\text{C}$, <BR>$\approx 0.25\text{V}@ 100\degree\text{C}$ | https://assets.nexperia.com/documents/data-sheet/BC807-25QA_40QA.pdf |
| **NCP15XM472J03RC** | NTC Thermistor | $4.7 k\Omega, \beta(25/100)=3560\text{K}$ | https://mm.digikey.com/Volume0/opasdata/d220001/medias/docus/8920/NTC%20Thermistors_83.pdf | 
| **$\text{R}_{base}$** | Offset for BJT's base votage divider | $511\Omega$ ||
| **$\text{R}_{sense}$** | Current Sense | $5.6\Omega$ | |

## 6. Solving for $\text{R}_{sns}$ and $\text{R}_{base}$

1. find $\text{R}_{ntc} ~ @ ~ 100\degree\text{C}$
    1. $\text{R}_{ntc\text{@}100} = \text{R}_{25} \cdot {e}^{\beta\cdot(\frac{1}{T_{100}}-\frac{1}{T_{25}})}$, where $T$ is in Kelvin
    1. $\text{R}_{ntc\text{@}100} = \text{R}_{25} \cdot {e}^{\beta\cdot(\frac{1}{373.15\text{K}}-\frac{1}{298.15\text{K}})}$
    1. $\text{R}_{ntc\text{@}100} = \text{R}_{25} \cdot {e}^{3560\text{K}\cdot(\frac{1}{373.15\text{K}}-\frac{1}{298.15\text{K}})}$
    1. $\text{R}_{ntc\text{@}100} = \text{R}_{25} \cdot {e}^{3560\text{K}\cdot(\frac{0.002679887-0.003354016}{\text{K}})}$
    1. $\text{R}_{ntc\text{@}100} = \text{R}_{25} \cdot {e}^{3560\text{K}\cdot(\frac{-0.000674129}{\text{K}})}$
    1. $\text{R}_{ntc\text{@}100} = 4.7 k\Omega \cdot {e}^{-2.39989924}$
    1. $\text{R}_{ntc\text{@}100} = 4.7 k\Omega \cdot 0.09072709$
    1. $\text{R}_{ntc\text{@}100} = 426.42\Omega$
1. Determine formula and solve for nominal $\text{R}_{base}$
    1. $\text{V}_{sns} = V_{ntc} \cdot (1 + \frac{\text{R}_{base}}{\text{R}_{ntc}})$
        1. $\text{V}_{ntc(25)} = 0.5\text{V}$
        1. $\text{R}_{ntc(25)} = 4700\Omega$
        1. $\text{V}_{ntc(100)} = 0.25\text{V}$
        1. $\text{R}_{ntc(100)} = 426.42\Omega$
    1. $\text{V}_{sns(25)} = \text{V}_{sns(100)}$
        1. $\text{V}_{ntc(25)} \cdot (1+\frac{\text{R}_{base}}{\text{R}_{ntc(25)}}) = \text{V}_{ntc(100)} \cdot (1+\frac{\text{R}_{base}}{\text{R}_{ntc(100)}})$
        1. $0.5\text{V} \cdot (1+\frac{\text{R}_{base}}{4700\Omega}) = 0.25\text{V} \cdot (1+\frac{\text{R}_{base}}{426.42\Omega})$
        1. $0.5\text{V} + \frac{0.5\text{V}\cdot\text{R}_{base}}{4700\Omega} = 0.25\text{V} + \frac{0.25\text{V}\cdot\text{R}_{base}}{426.42\Omega}$
        1. $0.5\text{V} + 0.000106383\text{A}\cdot\text{R}_{base} = 0.25\text{V} + 0.000586276\text{A}\cdot\text{R}_{base}$
        1. $0.000106383\text{A}\cdot\text{R}_{base} = 0.000586276\text{A}\cdot\text{R}_{base} - 0.25\text{V}$
        1. $0.000479893\text{A}\cdot\text{R}_{base} = 0.25\text{V}$
        1. $\text{R}_{base} = \frac{0.25\text{V}}{0.000479893\text{A}}$
        1. $\text{R}_{base} = 520.\Omega$
1. Now that $\text{R}_{base}$ and the target constant current of $100m\text{A}$ are known, we can solve for $\text{R}_{sns}$
    1. $\text{V}_{eb} = \text{V}_{sns}(\frac{\text{R}_{ntc}}{\text{R}_{ntc} + \text{R}_{base}})$
    1. Using Ohm's Law, we can substitute $\text{V}_{sns} = \text{I}_{load} \times \text{R}_{sns}$, 
        - $\text{V}_{eb} = \text{I}_{load} \times \text{R}_{sns}\cdot(\frac{\text{R}_{ntc}}{\text{R}_{ntc} + \text{R}_{base}})$
    1. Rearrange to solve for $\text{R}_{sns}$
        - $\text{R}_{sns} = \frac{\text{V}_{eb}}{\text{I}_{load}}\cdot(\frac{\text{R}_{ntc}+\text{R}_{base}}{\text{R}_{ntc}})$
    1. Since we know $\text{V}_{eb}$ at $25\degree\text{C}$ and $100\degree\text{C}$ and our desired $\text{I}_{load} = 100m\text{A}$...
        - $\text{R}_{sns(25)} = 5.55\Omega$
        - $\text{R}_{sns(100)} = 5.55\Omega$
- Ideal values for $\text{R}_{sns} = 5.6\Omega$
- Ideal values for $\text{R}_{base} = 511\Omega$
- $\text{I}_{load} = 98.57m\text{A}$

              