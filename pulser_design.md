# pic32arick — Pulser Design Notes

**Covers:** 5 V transistor drive feasibility, buffered supply, PWM timing, FT2232H vs MCP2200  
**Date:** 2026-07-04  

---

## Q1 — 5 V Transistor Pulser: Feasibility vs. High-Voltage Approach

### Physics of piezo excitation

A piezo transducer is electrically a capacitor (C_piezo, typically 100–2000 pF depending on element size and frequency) in parallel with a motional branch (series RLC representing the mechanical resonance). The key relationships:

```
Acoustic pressure  ∝  drive voltage  (linear)
SNR penalty (5V vs 100V)  =  20 × log10(100/5)  =  +26 dB in favour of 100 V
Peak drive current  =  C_piezo × ΔV / t_rise
```

For a typical 5 MHz element (C_piezo ≈ 500 pF) with a 20 ns rise time at 5 V:

```
I_peak = 500 pF × 5 V / 20 ns = 125 mA
```

This is comfortably within the ratings of common small-signal NPN transistors (e.g., MMBT3904: 200 mA) or logic-level N-MOSFETs (e.g., 2N7002: 300 mA). So the drive current is achievable.

### Is 5 V enough signal?

| Parameter | 5 V pulser | 100 V pulser (MD1213 approach) |
|-----------|-----------|-------------------------------|
| Drive voltage | 5 V | 50–100 V |
| Transmitted acoustic pressure | Baseline | +20 to +26 dB |
| Receive echo amplitude | Baseline | +20 to +26 dB |
| Achievable penetration depth | < 3–5 cm (tissue) | 10–20 cm |
| Pulse energy (500 pF load) | 6.25 nJ | 1.25–5 µJ |
| Suitable applications | Thin samples, contact sensing, < 5 cm water path | Standard medical/industrial ranging |

**Verdict for the pic32arick application:** 5 V is feasible for short-range work (thin phantom, liquid bath, < 5 cm). For anything deeper, the 26 dB signal penalty will push you below the noise floor even with 57 dB of receive gain.

### Transistor selection for a 5 V pulser

For a single-ended positive pulse (unipolar):

```
                3.3V PWM
                   │
              [Level shift]
                   │
                  Gate
            ┌───────────────┐
3V3 ──[R]──┤  N-MOSFET      │── Drain ──→ Transducer (one end)
            │  (e.g. BSS138) │
            └───────┬───────┘
                  Source
                   │
                  GND
```

Good transistor choices:

| Part | Package | V_DS | I_D | R_DS(on) | t_r |
|------|---------|------|-----|----------|-----|
| BSS138 | SOT-23 | 50 V | 200 mA | 3.5 Ω | ~4 ns |
| 2N7002 | SOT-23 | 60 V | 300 mA | 4 Ω | ~5 ns |
| MMBT3904 | SOT-23 NPN | 40 V | 200 mA | — | ~10 ns |
| DMG2302UK | SOT-23 | 20 V | 4 A | 55 mΩ | ~2 ns |

**Recommendation:** BSS138 or DMG2302UK. Drive the gate from a dedicated MOSFET gate driver (e.g. TC4427A, 1.5 A peak) rather than directly from the 3.3 V PWM pin — this halves the rise time and prevents gate charge from loading the PWM output.

For a bipolar pulse (push-pull, better ring-down control):

```
5V ──┤P-MOSFET├──┬── Transducer
                  │
GND──┤N-MOSFET├──┘
```

Use a half-bridge gate driver (e.g. IR2104 or TC4452) — this gives a cleaner pulse shape and the falling edge actively pulls the transducer back toward 0 V, reducing ring-down.

### Comparison with the MD1213 + TO-package approach

The Supertex/Microchip MD1213 is a high-voltage MOSFET gate driver rated for ±100 V operation. Paired with a discrete HV MOSFET in a TO-92 or TO-220 package (e.g. IRF510, ZVN4424A), it delivers:

- Gate drive current: up to 1 A peak → fast edge rates even with high-capacitance HV MOSFETs
- Operating voltage: 20–100 V pulser supply
- Pulse rise time: 10–20 ns into typical piezo load

The main differences versus a 5 V transistor pulser:

| Aspect | 5 V transistor | MD1213 + TO-chip |
|--------|---------------|-----------------|
| Drive voltage | 5 V | 20–100 V |
| Circuit complexity | Very simple | Needs HV supply, bootstrap caps |
| BOM cost | < €0.50 | ~€5–10 |
| Board area | Tiny | Larger (HV decoupling) |
| SNR benefit | Baseline | +20 to +26 dB |
| Safety | Low risk | HV hazard; requires isolation |

**For a development board / proximity sensing:** start with 5 V. **For diagnostic-grade imaging or > 5 cm depth:** revisit the HV approach.

### Ring-down analysis

After the transmit pulse, the piezo rings at its resonant frequency. Ring-down duration determines the **dead zone** — the minimum distance at which you can detect a reflection.

```
Dead zone depth  =  (ring-down duration × speed of sound) / 2
Speed of sound in water ≈ 1500 m/s
```

Ring-down is driven by the stored energy in the piezo's mechanical resonance, not by the drive voltage. However:
- Higher drive voltage → more initial excitation energy → longer observable ring-down amplitude above noise
- Lower drive voltage → ring-down decays below the noise floor sooner → **shorter apparent dead zone** (minor advantage of 5 V)

**Mitigation strategies regardless of drive voltage:**
1. **Damping resistor:** 50–200 Ω in parallel with the transducer. Dissipates ring-down energy. Increases insertion loss on transmit.
2. **Active damping:** Issue a second pulse of opposite polarity ~T/2 after the first (T = resonant period). Most effective with a push-pull driver.
3. **Mechanical backing layer:** An absorptive backing on the piezo element (manufacturer option). Widens bandwidth, reduces ring-down at the cost of sensitivity.
4. **MD0100 impedance:** The MD0100 presents a defined impedance to the transducer in receive mode, which acts as a partial damping load.

Expected dead zone at 5 V, 5 MHz, typical element:
- Ring-down amplitude falls to noise floor in ~3–5 µs → dead zone ≈ 2–4 mm in water.

---

## Q2 — Dedicated Buffered 5 V Supply for Pulser

**Yes — this is essential, not optional.**

### Why a shared rail is problematic

A 150 ns, 125 mA current transient (the piezo charge pulse) has a frequency content up to ~1/(2×150ns) ≈ 3 MHz. Even a few nH of PCB trace inductance on the shared 5 V rail causes a voltage spike of:

```
ΔV = L × dI/dt = 5 nH × 125 mA / 20 ns = 31 mV
```

31 mV of supply noise at 3 MHz will couple into the receive path and add directly to ADC input noise. Worse, the ground return current for this spike flows through the shared ground plane, causing ground bounce at the same frequencies.

### Recommended supply architecture

```
USB 5V input
    │
    ├──[Ferrite bead, ~600Ω @ 100 MHz]──[C: 10µF + 100nF]── Logic 5V
    │                                                            │
    │                                                       [LDO: 3.3V]── Digital/Analog 3.3V
    │
    └──[Ferrite bead, ~600Ω @ 100 MHz]──[C: 100µF + 10µF + 100nF]── Pulser 5V (dedicated)
```

**Pulser rail decoupling stack (place as close to transistor drain/source as possible):**

| Cap | Value | Type | Purpose |
|-----|-------|------|---------|
| C_bulk | 100 µF | Electrolytic or polymer | Bulk charge reservoir for pulse current |
| C_mid | 10 µF | MLCC X5R/X7R | Mid-frequency decoupling |
| C_hf | 100 nF | MLCC X7R (0402) | High-frequency spike suppression |
| C_hf2 | 10 nF | MLCC C0G (0402) | Very high-frequency, low ESL |

**Ground topology:** The pulser's high-current return ground should connect to the main ground star point (the USB connector ground) directly — not through the analog circuit ground pour. Use a PCB slot or gap to force the pulser return current to take the intended path.

**Ferrite bead selection:** Use a part rated for the DC current (> 500 mA), e.g. Murata BLM21PG601SN1 (600 Ω @ 100 MHz, 3 A rated).

### PCB layout summary

```
USB GND ──────────────────────────── Star ground point
              │                 │
         Analog GND        Pulser GND return
         (quiet)           (high current, direct star return)
```

---

## Q3 — PWM Configuration: 150 ns Pulse / 150 µs Period

### Resolution check

The PIC32AK high-resolution PWM has **2.5 ns resolution** (from the datasheet). This is derived from the system clock or a dedicated PLL — verify clock frequency in your configuration (2.5 ns implies 400 MHz base clock or edge-delay unit).

```
Pulse width:   150 ns / 2.5 ns  =  60 counts   ✓  (exact integer)
Period:        150 µs / 2.5 ns  =  60,000 counts ✓  (exact integer)
PRF:           1 / 150 µs       =  6,666.67 Hz  ≈  6.67 kHz
```

Both values are exact multiples of the 2.5 ns resolution. **Zero quantization error.**

### Register configuration (pseudocode)

```c
// Assuming PTPER (period register) and PDCx (duty cycle register)
// with 2.5ns per count

#define COUNTS_PER_NS   (1.0 / 2.5)   // = 0.4 counts per ns
#define PERIOD_COUNTS   60000          // 150 µs
#define PULSE_COUNTS    60             // 150 ns

PWM1_PERIOD  = PERIOD_COUNTS - 1;     // PTPER = 59999
PWM1_DUTYCYC = PULSE_COUNTS;          // PDC1  = 60

PWM1_enable();
```

Verify with the PIC32AK PWM peripheral registers (PTPER, PDCx, DTRx) in the family datasheet, Section 17.

### Timing accuracy in practice

The 2.5 ns step is accurate to the precision of the source clock. With a crystal oscillator (typical ±20 ppm), the absolute period accuracy is:

```
Period error = 150 µs × 20 ppm = 3 ns   (negligible)
```

Cycle-to-cycle jitter is dominated by:
- PLL phase noise (if PWM clock derives from PLL): typically < 1 ns RMS
- MCU interrupt latency (does not affect hardware PWM — the timer runs independently)

The **ADC trigger** should be sourced directly from the PWM peripheral (not software), using the internal PWM-to-ADC trigger path. This gives deterministic, jitter-free synchronisation between the transmit pulse and the ADC capture window. Use the PWM trigger postscaler to fire the ADC once per N pulses if needed.

### Depth coverage at 6.67 kHz PRF

```
Max unambiguous range:
  = speed of sound × (period / 2)
  = 1500 m/s × 75 µs
  = 112.5 mm  (in water)
  = ~11 cm    — adequate for close-range applications
```

If deeper range is needed, lower the PRF (increase period) — the PWM resolution accommodates any integer multiple of 2.5 ns.

---

## Q4 — FT2232H vs MCP2200 for this Application

### Bandwidth reality check first

At 6.67 kHz PRF (150 µs period) with a 100 µs acquisition window:
- Samples per burst: 40 MSPS × 100 µs = **4,000 samples × 2 bytes = 8 KB**
- Inter-pulse gap: 150 µs − 100 µs = 50 µs
- Required real-time throughput: 8 KB / 50 µs = **160 MB/s** — impossible over any USB 2.0 connection

This means real-time streaming at full PRF is off the table for both chips. The practical mode is **burst-then-drain**: capture one burst into SRAM, pause acquisition, transmit, resume. Both chips operate within this constraint — the question is how that constraint differs between them.

### Side-by-side comparison

| Feature | MCP2200 | FT2232H |
|---------|---------|---------|
| USB mode | Full Speed (12 Mbit/s) | High Speed (480 Mbit/s) |
| Practical bulk throughput | ~1 MB/s | **25–40 MB/s** (synchronous 245 FIFO mode) |
| Latency (USB frame) | 1 ms (FS SOF interval) | **125 µs** (HS microframe interval) |
| Hardware FIFO | ~64 bytes internal | **4096 bytes TX + 4096 bytes RX** |
| Channels | 1 (UART only) | 2 (each: UART / SPI / I2C / JTAG / FIFO) |
| Max baud rate | 1 Mbit/s UART | 12 Mbit/s UART; unlimited in FIFO mode |
| Driver (host) | HID + CDC, no drivers needed | FTDI VCP or D2XX (must install) |
| Package | 28-SOIC / QFN | 64-LQFP |
| External crystal | Not required | **12 MHz required** |
| External EEPROM | Not required | Optional (for PID/VID customisation) |
| BOM cost | ~€1–2 | ~€5–8 |
| Supply pins | Single 3.3 V or 5 V | VCC (3.3V), VCCIO (1.8–3.3V), VPLL (3.3V) |

### Throughput impact on effective PRF

Burst size = 8 KB (4000 samples, 16-bit packed):

| Chip | Transmit time for 8 KB | Max burst-then-drain PRF |
|------|------------------------|--------------------------|
| MCP2200 @ 1 MB/s | **8 ms** | ~125 Hz |
| FT2232H FIFO @ 25 MB/s | **0.32 ms** | ~3,000 Hz |

**Practical conclusion:** For streaming at ≥ 500 Hz PRF the MCP2200 cannot keep up; the FT2232H can. For ≤ 100 Hz PRF (A-scan display, slow imaging), MCP2200 is fine.

### FT2232H synchronous 245 FIFO mode

This mode is the key differentiator. The host-side D2XX library gives near-native USB HS throughput without custom firmware — the FT2232H handles all USB protocol framing. Interface to PIC32AK is an 8-bit parallel bus:

```
PIC32AK                FT2232H (channel A, FIFO mode)
  D[7:0] ──────────────  ADBUS[7:0]
  WR_n   ──────────────  WR#
  RXF_n  ──────────────  RXF#   (low when FIFO has data)
  TXE_n  ──────────────  TXE#   (low when FIFO can accept data)
  CLK    ──────────────  CLKOUT (60 MHz from FT2232H)
```

The PIC32AK writes a byte per clock cycle when TXE# is low. At 60 MHz that's 60 MB/s — the USB HS link is the bottleneck at ~40 MB/s. This bypasses the UART baud rate entirely.

### Is the FT2232H worth it for this design?

**Arguments for sticking with MCP2200:**
- The minimal-BOM principle is explicitly stated as a design constraint
- At ≤ 100 Hz PRF (reasonable for a dev/prototype board) MCP2200 is sufficient
- No driver installation required on the host — lowers the barrier to use
- 36 fewer pins to route; smaller PCB footprint
- The 8-bit parallel FIFO interface requires 10+ additional GPIO lines from the PIC32AK

**Arguments for upgrading to FT2232H:**
- Without it, the maximum useful PRF is ~125 Hz, which limits the device to slow A-scan use. Fast imaging sequences (Doppler, compound imaging) are impossible.
- Channel B can serve as a JTAG/SWD debug interface or I2C control channel — eliminating the MPLAB Snap debugger for production use
- The 4096-byte FIFO absorbs burst jitter without needing tight firmware timing
- Once you're beyond the prototype stage, the €5–6 delta is trivial in a PCB BOM

**Recommendation:** Keep MCP2200 for the v1 prototype (validates the signal chain with minimal complexity). **Add FT2232H as an optional footprint** in parallel — same USB connector, DNP (do not populate) selection via solder bridge. This costs nothing in BOM for v1 but gives a clear upgrade path when the throughput ceiling becomes limiting.

---

## Summary Recommendations

| Topic | Recommendation |
|-------|---------------|
| 5V pulser transistor | BSS138 or DMG2302UK in SOT-23; drive gate via TC4427A |
| Push-pull for ring-down | Add P-channel MOSFET + half-bridge driver; reduces dead zone |
| vs. MD1213 + HV MOSFET | 5V is 20–26 dB weaker; acceptable for < 5 cm range only |
| Dedicated pulser 5V | Essential — ferrite bead isolation + 100µF + 10µF + 100nF stack |
| PWM timing | 60 counts pulse / 60,000 counts period — exact, zero quantization error |
| ADC trigger | Use PWM hardware trigger, not software interrupt |
| MCP2200 vs FT2232H | Keep MCP2200 for v1; add FT2232H as DNP option for > 125 Hz PRF streaming |
