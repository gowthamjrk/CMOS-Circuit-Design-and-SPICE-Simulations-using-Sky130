# CMOS Circuit Design and SPICE Simulation using SKY130 Technology


---

## Table of contents

1. [About this project](#about)
2. [Why simulate before you fabricate](#why)
3. [NMOS transistor — structure and terminals](#nmos)
4. [Regions of operation](#regions)
   - [Cut-off](#cutoff)
   - [Linear / triode](#linear)
   - [Saturation](#saturation)
5. [Anatomy of a SPICE deck](#deck)
6. [NMOS I-V characterization](#iv)
   - [Long-channel device (W=5µ, L=2µ)](#long)
   - [Short-channel device and velocity saturation (W=0.39µ, L=0.15µ)](#short)
7. [The CMOS inverter](#cmos)
   - [Voltage transfer characteristic (load-line method)](#vtc)
   - [Robustness: Vm, noise margin, supply and device variation](#robustness)
8. [PVT corner study: TT vs FF vs SS](#pvt)
9. [Propagation delay tables](#delay)
10. [What this exercise reinforced](#learnings)
11. [Repository layout & reproducing the results](#repo)


---

<a name="about"></a>
## 1. About this project

The goal here is to go from an NMOS transistor's basic drain-current equations all the way up to a fully characterized CMOS inverter — switching threshold, noise margin, sensitivity to supply and device-size variation, and propagation delay across process corners — using nothing but ngspice and the SKY130 models. Everything below was generated from the `.spice` decks in [`Program/`](Program), run against the `sky130_fd_pr` model library.

<a name="why"></a>
## 2. Why simulate before you fabricate

A logic gate is, physically, just a particular sizing and interconnection of NMOS and PMOS transistors. Getting the *functionality* right is easy; getting the *timing and robustness* right is not, and that's where SPICE earns its keep. Feeding a known input waveform into a netlist and reading back the output lets you extract the numbers that downstream flows depend on — delay, noise margin, switching threshold — long before silicon exists. Skipping this step means discovering those problems after tapeout, which is a much more expensive place to find them.

<a name="nmos"></a>
## 3. NMOS transistor — structure and terminals

![nmos-structure](https://user-images.githubusercontent.com/63381455/153582815-e7c5a70f-7b06-4ab4-8dac-e6cc8159161c.png)

An NMOS is a four-terminal device — **gate, drain, source, body** — built on a p-type substrate with n⁺ diffusions for the drain and source, separated by a thin gate-oxide-and-poly (or metal) stack that drives the channel underneath it.

<a name="regions"></a>
## 4. Regions of operation

With the source and body tied together (Vsb = 0): at Vgs = 0 the source/substrate and drain/substrate junctions are just reverse-biased diodes, and the region between them is high resistance. As Vgs rises, it repels majority carriers from the surface, leaving a depletion region; push it further and the surface inverts to n-type — **strong inversion**. The gate voltage at which this happens is the **threshold voltage, Vt**.

$V_{gs} = V_{to}$, when $V_{sb} = 0$

$V_{to}$ is a function of manufacturing process

$V_{gs} = V_{to} + V_1$, when $V_{sb} = +ve$

$$V_1 = \gamma \left[\sqrt{|{-2\phi_f} + V_{sb}|} - \sqrt{|2\phi_f|}\right]$$

where, $\phi_f$ = Fermi potential $= -\phi_f \cdot \ln \dfrac{N_a}{n_i}$

$\gamma$ = body effect coefficient which expresses the changes in $V_{sb}$, whose unit is $\sqrt{V}$ or $V^{0.5}$

$$\gamma = \frac{\sqrt{2 q N_a \epsilon_{si}}}{C_{ox}}$$

where, $q$ = charge of electron $= 1.602 \times 10^{-19}$ coulombs

$N_a$ = acceptor doping concentration

$\epsilon_{si}$ = relative permittivity of Si $= 11.7$

$C_{ox}$ = oxide capacitance

With Vsb > 0 (body effect), channel formation is delayed and a larger Vgs is needed for inversion — the depletion region widens more on the source side of the junction. All of these coefficients (Vt0, body-effect coefficient γ, etc.) are process parameters set by the foundry, not by the designer.

<a name="cutoff"></a>
### Cut-off

For `Vgs < Vth`, no channel forms and `ID = 0` — the source-drain path is effectively open.

<a name="linear"></a>
### Linear / triode

Once `Vgs > Vt`, induced charge along the channel is roughly proportional to the local voltage drop; with a small Vds applied, a voltage gradient V(x) builds up along the channel length, and the *effective* channel length ends up slightly shorter than the drawn length due to fabrication effects.

Induced charge $Q_i$ at any point along the channel is given by

$$Q_i \propto -\left[(V_{gs} - V_{|x|}) - V_t\right]$$

Thus according to the law of charge conservation,

$$Q = CV$$

$$Q_i = -C_{ox} \left(\left[V_{gs} - V_{|x|}\right] - V_t\right)$$

where $C_{ox}$ = gate oxide capacitance

$$C_{ox} = \frac{\epsilon_{ox}}{T_{ox}}, \quad \text{constant value, and } T_{ox} \text{ is given by foundry}$$

$\epsilon_{ox}$ = oxide permittivity  
$= 3.97 \times \epsilon_o$  
$= 3.5 \times 10^{-11}$ F/m

$\epsilon_o$ = permittivity of free space $= 8.85 \times 10^{-12}$ F/m

$T_{ox}$ = oxide thickness

---

(Drift current = velocity of the charge $\times$ available charge, over the channel)

$$I_d = -V_n(x) \cdot Q_i \cdot w$$

$V_n(x)$ = Velocity of charge = mobility $\times$ electric field

$$\mu_n \cdot \frac{dv}{dx}$$

where $dv = 0$ to $V_{ds}$ and $dx = 0$ to $L$, and $W$ = width of the channel

Thus,

$$I_d = \mu_n \cdot \frac{dv}{dx} \, C_{ox} \left(\left[V_{gs} - V_{|x|}\right] - V_t\right) \cdot W$$

$$\frac{I_d}{dx} = \mu_n \cdot C_{ox} \left(\left[V_{gs} - V_{|x|}\right] - V_t\right) \cdot W \, dv$$

Integrating LHS w.r.t. $x$ from $0$ to $L$, and RHS w.r.t. $V$ from $0$ to $V_{ds}$:

$$I_d \cdot L = \mu_n \cdot C_{ox} \cdot W \left[(V_{gs} - V_t)V_{ds} - \left(\frac{V_{ds}^2}{2}\right)\right]$$

$$I_d = \frac{\mu_n \cdot C_{ox} \cdot W}{L} \left[(V_{gs} - V_t)V_{ds} - \left(\frac{V_{ds}^2}{2}\right)\right]$$

Now, $\mu_n \cdot C_{ox} = K_n'$ = process transconductance

$$K_n = K_n' \cdot \frac{W}{L} = \text{gain factor}$$

$$I_d = K_n\left[(V_{gs} - V_t)V_{ds} - \left(\frac{V_{ds}^2}{2}\right)\right]$$

However, from the above equation we note that it is a quadratic equation and not a linear one, which is the required region of operation.

Thus, neglecting $\frac{V_{ds}^2}{2}$ for the resistive mode of operation:

$$I_d = K_n\left[(V_{gs} - V_t)V_{ds}\right]$$

![voltage-gradient](https://user-images.githubusercontent.com/63381455/153896100-ebfd3a42-608f-415d-ab59-ac7bb4cf8a9b.JPG)

<a name="saturation"></a>
### Saturation

Once `(Vgs − Vds) ≥ Vt`, the device is in saturation — the channel near the drain "pinches off" and current becomes (to first order) independent of Vds.

For saturation, $V_{ds} = (V_{gs} - V_t)$

Thus drain current $I_d$ is given as:

$$I_d = K_n' \cdot \frac{W}{L} \left[(V_{gs} - V_t)(V_{gs} - V_t) - \frac{(V_{gs} - V_t)^2}{2}\right]$$

$$I_d = \left[K_n' \cdot \frac{W}{L} (V_{gs} - V_t)^2 - \frac{(V_{gs} - V_t)^2}{2}\right]$$

$$I_d = K_n' \cdot \frac{W}{L} \left[(V_{gs} - V_t)^2\right]$$

$$I_d = \frac{K_n}{2} \left[(V_{gs} - V_t)^2\right]$$

The above equation behaves like a perfect current source as all the values are constants. However, with increase in $V_{ds}$, the effective channel length changes, which can also be said as the channel length is modulated by $V_{ds}$. Depletion width near the drain region increases. This change in effective channel length is called **channel length modulation** $\lambda$, whose impact on the equation is:

$$I_d = K_n' \cdot \frac{W}{2L} \left[(V_{gs} - V_t)^2\right]\left[1 + \lambda V_{ds}\right]$$

$$I_d = \frac{K_n}{2} \left[(V_{gs} - V_t)^2\right]\left[1 + \lambda V_{ds}\right]$$

The plot below, extracted directly from an ngspice DC sweep of an NMOS, shows both regions on the same ID–Vds curve:

![idvds-both-regions](https://user-images.githubusercontent.com/63381455/154033661-8f20fdbc-56b8-422e-942d-5015f28c781f.JPG)

<a name="deck"></a>
## 5. Anatomy of a SPICE deck

Every `.spice` file used here follows the same four-part skeleton:

| Section | Purpose |
|---|---|
| **Netlist** | Nodes, component connectivity, device sizes — taken directly from the schematic |
| **Model include** | `.lib` pointing at the SKY130 technology file for the desired corner (tt / ff / ss) |
| **Simulation command** | `.op`, `.dc`, or `.tran` — what analysis to run |
| **Control block** | `.control … .endc` — what to plot or print once the simulation finishes |

<a name="iv"></a>
## 6. NMOS I-V characterization

<a name="long"></a>
### Long-channel device (W = 5µ, L = 2µ)

**ID–Vds sweep:**

```bash
*Model Description
.param temp=27

*Include model file
.lib "../sky130_fd_pr/models/sky130.lib.spice" tt

*Netlist Description
XM1 Vdd n1 0 0 sky130_fd_pr__nfet_01v8 w=5 l=2
R1 n1 in 55
Vdd vdd 0 1.8V
Vin in 0 1.8V

*simulation commands
.op
.dc Vdd 0 1.8 0.1 Vin 0 1.8 0.2

.control
run
display
setplot dc1
plot -Vdd#branch
.endc
.end
```

![idvds-long-channel](image/output_TT/01_IDVds_wF.png)

**ID–Vgs sweep** (same device, Vin swept instead of Vdd):

```bash
.dc Vin 0 1.8 0.1 Vdd 0 1.8 1.8
```

![idvgs-long-channel](image/output_TT/01_IDVgs_quadratic.png)

The quadratic shape here is the textbook long-channel square-law behaviour — current keeps rising smoothly with Vgs without the early roll-off you'll see below.

<a name="short"></a>
### Short-channel device and velocity saturation (W = 0.39µ, L = 0.15µ)

At long channel lengths, carrier velocity is assumed to scale linearly with the lateral electric field. At the geometries actually used in SKY130 (L in the tens to hundreds of nanometers), that assumption breaks down — at higher fields, carriers hit a scattering-limited **saturation velocity** and stop accelerating, so drain current saturates *earlier* than the square law predicts.

![velocity-saturation-1](https://user-images.githubusercontent.com/63381455/154623491-a1872005-391b-4a17-bb4d-ecf157cd041e.JPG)

![velocity-saturation-2](https://user-images.githubusercontent.com/63381455/154621528-7f71d7d3-3888-400e-8cc2-d4a6131b3aa3.JPG)

Working this into the drain-current equation:

Let $V_{gs} - V_t = V_{gt}$

$I_d = 0$, for $V_{gt} < 0$, cut-off mode

For all other modes,

$$I_d = K_n \left[(V_{gt} \cdot V_{min}) - \frac{V_{min}^2}{2}\right]\left[1 + \lambda V_{ds}\right]$$

where $V_{min} = \min(V_{gt}, V_{ds}, V_{dsat})$

**Case 1:** $V_{gt} = \min$ ($V_{ds}$ are at higher values)

$$I_d = K_n \left[(V_{gt} \cdot V_{gt}) - \frac{V_{gt}^2}{2}\right]\left[1 + \lambda V_{ds}\right]$$

$$I_d = K_n \left[\frac{V_{gt}^2}{2}\right]\left[1 + \lambda V_{ds}\right]$$

The above equation is the **saturation region equation**.

**Case 2:** $V_{ds} = \min$ (lower value of $V_{ds}$)

$$I_d = K_n \left[(V_{gt} \cdot V_{ds}) - \frac{V_{ds}^2}{2}\right]\left[1 + \lambda V_{ds}\right]$$

$$I_d = K_n \left[(V_{gt} \cdot V_{ds}) - \frac{V_{ds}^2}{2}\right]$$

$[1 + \lambda V_{ds}]$ cancels out as it is very low.

The above equation is the **resistive region equation**.

**Case 3:** $V_{dsat} = \min$, system enters the velocity saturation region, which is true only for short channel devices ($< 250\,\mu m$)

$I_d$, short channel device in saturation region:

$$I_d = K_n \left[(V_{gt} \cdot V_{dsat}) - \frac{V_{dsat}^2}{2}\right]\left[1 + \lambda V_{ds}\right]$$

$$I_d = \mu_n \cdot C_{ox} \cdot \frac{W}{L} \left[(V_{gt} \cdot V_{dsat}) - \frac{V_{dsat}^2}{2}\right]\left[1 + \lambda V_{ds}\right]$$

Scaling to smaller nodes would nominally push current *up*, but velocity saturation caps that gain — the device saturates at a lower Vds than a long-channel device would.

```bash
*Netlist description
XM1 Vdd n1 0 0 sky130_fd_pr__nfet_01v8 W=0.39 L=0.15
R1 in n1 55
Vin in 0 1.8V
Vdd Vdd 0 1.8V

.lib "../sky130_fd_pr/models/sky130.lib.spice" tt
.op
.dc Vdd 0 1.8 0.1 Vin 0 1.8 0.2
```

![idvds-short-channel](image/output_TT/02_IDVds_0.39_w015l_wf.png)

```bash
.dc Vin 0 1.8 0.1
```

![idvgs-short-channel](image/output_TT/02_IDVgs_0.39_w015l_wf.png)

Compared to the 5µ/2µ device, the short-channel curve flattens out sooner — direct evidence of velocity saturation rather than ideal square-law behaviour.

<a name="cmos"></a>
## 7. The CMOS inverter

A CMOS inverter pairs an NMOS pull-down with a PMOS pull-up. When `Vin = Vdd`, the NMOS conducts and pulls `Vout` to 0; when `Vin = 0`, the PMOS conducts and pulls `Vout` to Vdd. No static path from Vdd to ground ever exists in either steady state, which is the whole reason CMOS logic burns so little static power.

![cmos-inverter-behaviour](https://user-images.githubusercontent.com/63381455/154632526-b59ab8be-f9fb-44ac-a79d-ddc36c1f907a.JPG)

<a name="vtc"></a>
### Voltage transfer characteristic (load-line method)

Treating the MOS device as a switch: for `|Vgs| > |Vt|` there's a finite ON resistance and current flows; for `|Vgs| < |Vt|` the OFF resistance is effectively infinite.

![vgs-vds-parameters](https://user-images.githubusercontent.com/63381455/154633767-4ebe8956-dfa3-46b4-b9a7-328d25c19f83.JPG)

The VTC is built by graphically merging the individual PMOS and NMOS load curves. Taking Vdd = 2V as an example:

**Step 1 — sweep VgsP, convert to Vin** (`Vin = VgsP + Vdd`):

| VgsP | Vin |
|---|---|
| 0 V | 2 V |
| −0.5 V | 1.5 V |
| −1 V | 1 V |
| −1.5 V | 0.5 V |
| −2 V | 0 V |

**Step 2 — convert VdsP to Vout** (mirroring the PMOS curve from quadrant 3 into quadrant 1, e.g. `Vout = VdsP + Vdd`):

![pmos-load-curve](https://user-images.githubusercontent.com/63381455/154644915-304fb4d5-82b8-41c1-8087-3998b870b480.JPG)

**Step 3 — the NMOS load curve** (`Vin = VgsN` directly, no shift needed):

| VgsN | Vin |
|---|---|
| 0 V | 0 V |
| 0.5 V | 0.5 V |
| 1 V | 1 V |
| 1.5 V | 1.5 V |
| 2 V | 2 V |

![nmos-load-curve](https://user-images.githubusercontent.com/63381455/154650659-14d4ac38-1bc5-47e5-b166-e8462636aeba.png)

**Step 4 — superimpose both curves.** Since Vin and Vout are shared between the two devices, the intersection of the NMOS and PMOS load lines at each Vin gives the operating Vout, and also tells you which region (linear/saturation) each transistor is in at that point:

![vtc-final](https://user-images.githubusercontent.com/63381455/154652762-3b5e4759-5fc7-4ef5-90ce-014969880255.JPG)

For example, at `Vin = 0V`, PMOS is in the linear region and NMOS is off, giving `Vout = Vdd`.

Simulated VTC for `(W/L)p = 2.34 × (W/L)n` (Wp=0.84µ, Wn=0.36µ, L=0.15µ for both):

```bash
XM1 out in Vdd Vdd sky130_fd_pr__pfet_01v8 W=0.84 L=0.15
XM1 out in 0 0 sky130_fd_pr__nfet_01v8 W=0.36 L=0.15
cload out 0 50fF
.dc Vin 0 1.8V 0.01
```

![vtc-2.34wl](https://user-images.githubusercontent.com/63381455/152681920-f5cf0fa0-2bf9-4094-bb34-8c1cd3248615.png)

<a name="robustness"></a>
### Robustness: Vm, noise margin, supply and device variation

**Switching threshold (Vm)** — the input voltage at which `Vin = Vout` — is one of four checks used here to judge inverter robustness, alongside noise margin, sensitivity to Vdd, and sensitivity to transistor sizing.

**Noise margin**, for `(W/L)p = 1/0.15`, `(W/L)n = 0.36/0.15`:

```bash
XM1 out in Vdd Vdd sky130_fd_pr__pfet_01v8 W=1 L=0.15
XM1 out in 0 0 sky130_fd_pr__nfet_01v8 W=0.36 L=0.15
cload out 0 50fF
.dc Vin 0 1.8V 0.01
```

![noise-margin](https://user-images.githubusercontent.com/63381455/152843755-83a4eb04-74a0-4db2-91c3-3469e562ad19.png)

Read off the plot:

```
NMH = VOH − VIH = 1.63125 − 0.964516 = 0.666734 V
NML = VIL − VOL = 0.801613 − 0.121875 = 0.679738 V
```

Both margins land at roughly two-thirds of a volt on a 1.8V rail, which is a healthy amount of noise immunity for this sizing.

**Supply variation** — sweeping Vdd from 1.8V down in 0.2V steps and re-running the DC sweep each time, with device sizes held fixed:

```bash
.control
let powersupply = 1.8
alter Vdd = powersupply
let supplyvoltagevariation = 0
dowhile supplyvoltagevariation < 6
  dc Vin 0 1.8 0.01
  let powersupply = powersupply - 0.2
  alter Vdd = powersupply
  let supplyvoltagevariation = supplyvoltagevariation + 1
end
plot dc1.out vs in dc2.out vs in dc3.out vs in dc4.out vs in dc5.out vs in
.endc
```

![supply-variation](https://user-images.githubusercontent.com/63381455/152846501-56336f02-80c4-4d1e-8522-1599c3b17e11.png)



<a name="pvt"></a>
## 8. PVT corner study: TT vs FF vs SS

This is the part the original writeup under-covered — it reported corner *numbers* (the delay tables in the next section) but never plotted how VTC and propagation delay actually move across corners. The decks in [`Program/TT_corner`](Program/TT_corner), [`Program/FF_corner`](Program/FF_corner) and [`Program/SS_corner`](Program/SS_corner) sweep five PMOS/NMOS width ratios — 1×, 2×, 2.5×, 3×, 4×, 5× — at each of the three standard corners:

- **TT** — typical NMOS / typical PMOS (nominal case)
- **FF** — fast NMOS / fast PMOS (best case)
- **SS** — slow NMOS / slow PMOS (worst case)

### VTC across corners, at (W/L)p = 2.5×(W/L)n

<table>
<tr><th>TT (nominal)</th><th>FF (fast)</th><th>SS (slow)</th></tr>
<tr>
<td><img src="image/output_TT/TT_vtc_2.5wl.png" width="260"></td>
<td><img src="image/output_FF/FF_vtc_2.5wl.png" width="260"></td>
<td><img src="image/output_SS/SS_vtc_2.5wl.png" width="260"></td>
</tr>
</table>

The VTC shape holds up across all three corners at this sizing — the switching point shifts a little, but nothing breaks down, which is what you'd want from a robust design point.

### Propagation delay across corners, at (W/L)p = 2.5×(W/L)n

<table>
<tr><th>TT (nominal)</th><th>FF (fast)</th><th>SS (slow)</th></tr>
<tr>
<td><img src="image/output_TT/TT_pd_2.5wl.png" width="260"></td>
<td><img src="image/output_FF/FF_pd_2.5wl.png" width="260"></td>
<td><img src="image/output_SS/SS_pd_2.5wl.png" width="260"></td>
</tr>
</table>

FF is visibly the quickest edge of the three and SS the slowest — exactly the ordering the corner names promise, and a useful sanity check that the model library is being invoked correctly.

<details>
<summary><b>Full VTC and propagation-delay set for all six W/L ratios, all three corners (click to expand)</b></summary>

**TT corner**

| 1× | 2× | 2.5× | 3× | 4× | 5× |
|---|---|---|---|---|---|
| ![](image/output_TT/TT_vtc_1wl.png) | ![](image/output_TT/TT_vtc_2wl.png) | ![](image/output_TT/TT_vtc_2.5wl.png) | ![](image/output_TT/TT_vtc_3wl.png) | ![](image/output_TT/TT_vtc_4wl.png) | ![](image/output_TT/TT_vtc_5wl.png) |
| ![](image/output_TT/TT_pd_1wl.png) | ![](image/output_TT/TT_pd_2wl.png) | ![](image/output_TT/TT_pd_2.5wl.png) | ![](image/output_TT/TT_pd_3wl.png) | ![](image/output_TT/TT_pd_4wl.png) | ![](image/output_TT/TT_pd_5wl.png) |

**FF corner**

| 1× | 2× | 2.5× | 3× | 4× | 5× |
|---|---|---|---|---|---|
| ![](image/output_FF/FF_vtc_1wl.png) | ![](image/output_FF/FF_vtc_2wl.png) | ![](image/output_FF/FF_vtc_2.5wl.png) | ![](image/output_FF/FF_vtc_3wl.png) | ![](image/output_FF/FF_vtc_4wl.png) | ![](image/output_FF/FF_vtc_5wl.png) |
| ![](image/output_FF/FF_pd_1wl.png) | ![](image/output_FF/FF_pd_2wl.png) | ![](image/output_FF/FF_pd_2.5wl.png) | ![](image/output_FF/FF_pd_3wl.png) | ![](image/output_FF/FF_pd_4wl.png) | ![](image/output_FF/FF_pd_5wl.png) |

**SS corner**

| 1× | 2× | 2.5× | 3× | 4× | 5× |
|---|---|---|---|---|---|
| ![](image/output_SS/SS_vtc_1wl.png) | ![](image/output_SS/SS_vtc_2wl.png) | ![](image/output_SS/SS_vtc_2.5wl.png) | ![](image/output_SS/SS_vtc_3wl.png) | ![](image/output_SS/SS_vtc_4wl.png) | ![](image/output_SS/SS_vtc_5wl.png) |
| ![](image/output_SS/SS_pd_1wl.png) | ![](image/output_SS/SS_pd_2wl.png) | ![](image/output_SS/SS_pd_2.5wl.png) | ![](image/output_SS/SS_pd_3wl.png) | ![](image/output_SS/SS_pd_4wl.png) | ![](image/output_SS/SS_pd_5wl.png) |

</details>

<a name="delay"></a>
## 9. Propagation delay tables

Capacitive load = 50fF, sizing swept from `(W/L)p = 1×(W/L)n` through `5×`.

**TT corner**

| wp/lp | x·wn/ln | Rise delay (ps) | Fall delay (ps) | Vm (V) |
|---|---|---|---|---|
| wp/lp | 1(wn/ln) | 149 | 73 | 0.831765 |
| wp/lp | 2(wn/ln) | 88 | 75 | 0.889839 |
| wp/lp | 2.5(wn/ln) | 73 | 75 | 0.895161 |
| wp/lp | 3(wn/ln) | 69 | 76 | 0.895161 |
| wp/lp | 4(wn/ln) | 59 | 78 | 0.916129 |
| wp/lp | 5(wn/ln) | 52 | 80 | 0.929032 |

**FF corner**

| wp/lp | x·wn/ln | Rise delay (ps) | Fall delay (ps) | Vm (V) |
|---|---|---|---|---|
| wp/lp | 1(wn/ln) | 114 | 60 | 0.809677 |
| wp/lp | 2(wn/ln) | 69 | 62 | 0.877419 |
| wp/lp | 2.5(wn/ln) | 60 | 62 | 0.887097 |
| wp/lp | 3(wn/ln) | 54 | 62 | 0.891935 |
| wp/lp | 4(wn/ln) | 48 | 64 | 0.914516 |
| wp/lp | 5(wn/ln) | 42 | 65 | 0.935484 |

**SS corner**

| wp/lp | x·wn/ln | Rise delay (ps) | Fall delay (ps) | Vm (V) |
|---|---|---|---|---|
| wp/lp | 1(wn/ln) | 226 | 93 | 0.85 |
| wp/lp | 2(wn/ln) | 122 | 96 | 0.906452 |
| wp/lp | 2.5(wn/ln) | 103 | 97 | 0.908065 |
| wp/lp | 3(wn/ln) | 95 | 98 | 0.903226 |
| wp/lp | 4(wn/ln) | 76 | 100 | 0.922581 |
| wp/lp | 5(wn/ln) | 64 | 102 | 0.935484 |

Across all three corners, `(W/L)p ≈ 2.5×(W/L)n` is the point where rise and fall delay converge — that balance point is exactly what you want for a clock-tree buffer, where symmetric edges matter more than raw speed. Ratios above that skew toward faster rise / slower fall, which is more appropriate for data-path buffers where edge symmetry is less critical than throughput.

`Ron(PMOS) ≈ 2.5 · Ron(NMOS)` is the underlying reason: PMOS hole mobility is roughly 2–2.5× lower than NMOS electron mobility in this process, so it takes proportionally more width to match drive strength.

<a name="learnings"></a>
## 10. What this exercise reinforced

- The gap between long-channel square-law theory and what a short-channel SKY130 device actually does is not subtle — velocity saturation shows up clearly even on a simple ID–Vds sweep.
- A single VTC or delay number means very little on its own; it only becomes useful once you know *which corner* it was pulled from. Plotting all three side by side made that concrete in a way the original delay-table-only version didn't.
- Sizing a CMOS gate is a trade-off, not a single "correct" answer — the same `2.5×` ratio that balances rise/fall delay for a clock buffer is not the ratio you'd pick if raw one-directional speed were the goal.

<a name="repo"></a>
## 11. Repository layout & reproducing the results

```
.
├── image/
│   ├── output_TT/     # TT-corner IV curves, VTC, propagation delay plots
│   ├── output_FF/      # FF-corner VTC + propagation delay plots
│   └── output_SS/      # SS-corner VTC + propagation delay plots
├── Program/
│   ├── TT_corner/       # .spice decks, typical corner
│   ├── FF_corner/       # .spice decks, fast corner
│   └── SS_corner/       # .spice decks, slow corner
└── README.md
```

To reproduce any plot: point the `.lib` include in the relevant deck at your local `sky130_fd_pr` model path, then run `ngspice <deckname>.spice`.


