# pic32arick — Op-Amp Receive Chain Design

**Signal chain:** MD0100 output → Stage 1 (fixed gain) → Stage 2 (I2C variable gain) → Anti-aliasing filter → ADC  
**MCU:** PIC32AK1216GC41064 — three integrated 100 MHz rail-to-rail op-amps used in full  
**Supply:** Vcc = 3.3 V, Vmid = 1.65 V (mid-rail bias)  

---

## Full Chain — ASCII Stage Diagram

```
MD0100
output
  |
  C_in (100 nF)   ← AC-couple; blocks MD0100 DC offset
  |
  +--[R_b1: 100k]--→ Vmid   ← sets quiescent input to 1.65 V
  |
  +IN ─────────────────────────────────┐
       OA1 (PIC32AK integrated, 100 MHz GBW)
  -IN ─────────────────────────────────┤
  |                                    │
  [Rg1: 1 kΩ]                   [Rf1: 4.3 kΩ]
  |                              [Cf1: 2.2 pF] ← in parallel with Rf1
  GND                                  │
                                 OA1 output ──→ gain ≈ +5 (14 dB)
                                       |
                                  C_12 (100 nF)   ← AC-couple between stages
                                       |
                     +--[R_b2: 100k]──→ Vmid
                     |
                     +IN ──────────────────────────────────┐
                          OA2 (PIC32AK integrated, 100 MHz GBW)
                     -IN ──────────────────────────────────┤
                     |                                     │
                  [Rg2: MCP4531-103]              [Rf2: 10 kΩ]
                  [I2C digital pot,               [Cf2: 1.5 pF] ← in parallel with Rf2
                   0–10 kΩ, 128 steps]                    │
                     |                             OA2 output ──→ gain 6–42 dB (variable)
                    GND                                    |
                                              C_out (100 nF)   ← AC-couple to filter
                                                           |
                                            ┌──────────────┤
                                        [R_aa1: 100 Ω]     |
                                            |           [C_aa1: 56 pF]
                                        [R_aa2: 100 Ω]     |
                                            |           [C_aa2: 56 pF]
                                            |               |
                                            +───────────────+──→ ADC input
                                                               (40 MSPS, 12-bit)

─────────────────────────────────────────────────────────────
Mid-rail bias generator (separate, uses OA3):

Vcc (3.3V)
  |
[R_top: 10 kΩ, 1%]
  |
  +──[C_byp1: 10 µF]──GND
  |──[C_byp2: 100 nF]──GND
  |
  +IN ─────────────────┐
       OA3 (unity gain follower)     → Vmid = 1.65 V (stiff low-Z reference)
  -IN ─────────────────┘ (output feeds back to -IN)
  |
[R_bot: 10 kΩ, 1%]
  |
 GND
```

---

## Stage 1 — Fixed Gain, Non-Inverting

**Topology:** Non-inverting amplifier  
**Gain:** A₁ = 1 + Rf1/Rg1 = 1 + 4300/1000 = **5.3× (+14.5 dB)**

| Component | Value | Purpose |
|-----------|-------|---------|
| Rg1 | 1 kΩ (1%) | Gain-set lower resistor |
| Rf1 | 4.3 kΩ (1%) | Gain-set upper resistor |
| Cf1 | 2.2 pF (C0G) | Feedback cap — HF rolloff at 1/(2π×Rf1×Cf1) ≈ **16.8 MHz** |
| C_in | 100 nF (X7R) | AC input coupling |
| R_b1 | 100 kΩ | Bias resistor to Vmid |

**Why fixed gain here:** Stage 1 sets the noise figure for the whole chain. Fixing the gain and optimising for low noise (close-in components, good layout) gives the best SNR. Variable gain here would mean the noise figure changes with gain setting.

**Bandwidth:** GBW/A₁ = 100 MHz / 5.3 ≈ **18.9 MHz** — covers 1–10 MHz transducers cleanly.

**Stability:** Cf1 limits the noise gain peaking at high frequency. Without it, the stray input capacitance (bond wire + PCB trace) combined with Rg1 creates a zero that drives the loop toward instability. Rule of thumb for op-amp stabilisation: Cf ≈ sqrt(Cin / (2π × GBW × Rg)) — for Cin = 5 pF, Rg = 1 kΩ, GBW = 100 MHz: Cf ≈ 2.8 pF → 2.2 pF is appropriate.

---

## Stage 2 — I2C Variable Gain, Non-Inverting

**Topology:** Non-inverting amplifier with digital potentiometer as Rg  
**Gain:** A₂ = 1 + Rf2/Rg2_wiper

| Component | Value | Purpose |
|-----------|-------|---------|
| Rf2 | 10 kΩ (1%) | Fixed upper feedback resistor |
| Cf2 | 1.5 pF (C0G) | HF stability cap across Rf2 |
| Rg2 | MCP4531-103 (0–10 kΩ, 128 steps) | I2C-controlled gain resistor |
| C_12 | 100 nF (X7R) | AC coupling from Stage 1 output |
| R_b2 | 100 kΩ | Bias resistor to Vmid |

### Gain Range

| Pot Step | Rg2 (approx) | Gain | Gain (dB) |
|----------|-------------|------|-----------|
| 127 (max R) | 10,078 Ω | 1 + 10k/10.1k ≈ **2.0×** | **+6 dB** |
| 64 | 5,078 Ω | 1 + 10k/5.1k ≈ **3.0×** | **+9.5 dB** |
| 16 | 1,328 Ω | 1 + 10k/1.3k ≈ **8.5×** | **+18.5 dB** |
| 4 | 391 Ω | 1 + 10k/391 ≈ **26.6×** | **+28.5 dB** |
| 1 | 156 Ω | 1 + 10k/156 ≈ **65×** | **+36.2 dB** |
| 0 | 78 Ω (wiper only) | 1 + 10k/78 ≈ **129×** | **+42.2 dB** |

**Total combined gain range (Stage 1 + Stage 2):**  
- Minimum: +14.5 + 6 = **+20.5 dB**  
- Maximum: +14.5 + 42 = **+56.5 dB**

### I2C Gain Control — MCP4531-103

```
PIC32AK          MCP4531-103
  SDA  ─────────────  SDA
  SCL  ─────────────  SCL
  GND  ─────────────  VSS, A0, A1 (address = 0x2C)
  3V3  ─────────────  VDD

              PA (pin 6) ──→ Vmid (or floating — only PB & PW used)
              PW (pin 5) ──→ -IN of OA2  (wiper = Rg2 bottom)
              PB (pin 4) ──→ GND         (Rg2 bottom reference)
```

**Write gain over I2C (one transaction, 2 bytes):**
```
START | 0x2C W | 0b00000000 (command: write volatile wiper 0) | step_value (0–127) | STOP
```
Step 0 = minimum resistance (maximum gain); step 127 = maximum resistance (minimum gain).

**Time-Gain Compensation (TGC):** Pre-compute a ramp table of wiper values (e.g., 16 steps, spaced 6 µs apart after the transmit pulse) and write them sequentially over 400 kHz Fast Mode I2C during the receive window. Each write takes ~45 µs at 400 kHz, so a 16-step TGC ramp spans ~720 µs of receive time.

---

## Anti-Aliasing Filter

**Topology:** Passive 2-pole RC low-pass  
**Target:** ≤ −3 dB at 18–20 MHz (Nyquist for 40 MSPS)

```
OA2 out
  |
[R_aa1: 100 Ω]
  |
  +──[C_aa1: 56 pF]──GND
  |
[R_aa2: 100 Ω]
  |
  +──[C_aa2: 56 pF]──GND
  |
ADC input
```

**Per-pole −3 dB frequency:** f₀ = 1/(2π × 100 × 56 pF) = **28.4 MHz**  
**2-pole combined −3 dB:** f₀ × √(√2 − 1) = 28.4 × 0.644 ≈ **18.3 MHz** ✓

**Attenuation at Nyquist (20 MHz):** ≈ −3.7 dB (mild — acceptable for a 12-bit ADC where the fundamental limit is at the noise floor, not alias rejection).

For better alias rejection (e.g., if using oversampling to push ENOB), replace with a 3-pole Butterworth implemented as a 2nd-order Sallen-Key + one RC section — requires an additional op-amp stage.

**ADC input protection:** Add BAT54S dual Schottky clamp (anode to GND, cathode to 3.3 V) at the ADC pin. Second line of defence behind MD0100 clamping.

---

## Mid-Rail Bias Generator

**Purpose:** Provides a stiff 1.65 V reference for AC-coupled signal biasing at OA1 and OA2 non-inverting inputs, and sets the quiescent output of both gain stages.

**Topology:** Resistor divider + OA3 unity-gain buffer

| Component | Value | Purpose |
|-----------|-------|---------|
| R_top | 10 kΩ (1%) | Divider upper arm |
| R_bot | 10 kΩ (1%) | Divider lower arm |
| C_byp1 | 10 µF (tantalum/MLCC) | Low-frequency supply noise rejection |
| C_byp2 | 100 nF (X7R) | High-frequency noise rejection |
| OA3 | PIC32AK integrated, unity-gain | Buffers the divider node to < 1 Ω output impedance |

**Output voltage:** Vmid = 3.3 V × (10k / 20k) = **1.65 V ±0.5%** (dominated by resistor tolerance)  
**Output impedance:** < 1 Ω (op-amp follower) — critical for biasing multiple stages without loading the divider.

Vmid is distributed to:
- R_b1 (Stage 1 bias) via short low-impedance trace
- R_b2 (Stage 2 bias) via short low-impedance trace
- (Optional) as ADC reference midpoint for future differential operation

---

## PCB Layout Notes

1. **Stage 1 input trace:** Keep < 5 mm from MD0100/MUX output to OA1 +IN. Stray capacitance here forms a zero with R_b1 that can cause peaking; minimize by keeping the trace short.
2. **Feedback caps Cf1, Cf2:** Place physically as close to op-amp pins as possible. Use 0402 C0G parts — they are stable with voltage and temperature and have minimal parasitic inductance.
3. **Vmid distribution:** Route the OA3 output as a low-impedance bus (wider trace, 0.3 mm) with decoupling caps (10 nF) at each R_bX tap point.
4. **Digital pot placement:** MCP4531 should be close to OA2 to minimise the Rg2 trace length (stray capacitance on this node directly affects gain at high frequency).
5. **Ground plane:** Solid uninterrupted analog ground plane under the entire receive chain. Keep digital PWM/I2C return currents away from the analog section with a split or a guard trace.

---

## Summary Table

| Stage | Topology | Gain | Key Parts | -3dB BW |
|-------|----------|------|-----------|---------|
| Bias gen | Divider + OA3 follower | 1× (unity) | 2× 10 kΩ, OA3, 10 µF + 100 nF | DC–100 MHz |
| Stage 1 | Non-inverting, fixed | **+14.5 dB** | Rg=1k, Rf=4.3k, Cf=2.2pF | **~19 MHz** |
| Stage 2 | Non-inverting, I2C variable | **+6 to +42 dB** | Rf=10k, MCP4531-103, Cf=1.5pF | ~10–19 MHz (gain-dep.) |
| AAF | Passive 2-pole RC | −3.7 dB @ 20 MHz | 2× (100Ω + 56pF) | **~18 MHz** |
| **Total** | | **+20.5 to +56.5 dB** | | |
