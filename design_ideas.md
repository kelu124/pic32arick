# pic32arick — Design Feasibility Assessment & Ideas

**Repo:** kelu124/pic32arick  
**Signal chain:** PWM pulser → MD0100 → 2-stage I2C-controlled op-amp gain → mid-rail bias → 40 MSPS ADC → MCP2200 USB  
**Assessment date:** 2026-07-04  

---

## Executive Summary

The architecture is **technically sound and feasible** for a low-cost, single-element pulse-echo ultrasound module. The PIC32AK1216GC41064 is an unusually well-matched MCU for this application — its integrated 100 MHz op-amps, 12-bit 40 MSPS ADCs, and 2.5 ns-resolution PWM eliminate several external ICs that would otherwise add cost and board area. The main risks are not architectural but implementation-level: analog front-end noise floor, USB bandwidth ceiling, and the absence of firmware.

**Verdict:** Green-light the hardware bring-up, but resolve the five flagged gaps before committing a PCB run.

---

## Stage-by-Stage Analysis

### 1. PWM Pulser (5 V, 3.3 V → 5 V level-shift)

| Item | Assessment |
|------|-----------|
| PWM resolution | 2.5 ns is more than adequate for sub-millimetre depth resolution |
| Voltage level | 5 V is marginal for driving a piezo transducer with good amplitude; 50–100 V pulsers are common in clinical gear, but 5 V is acceptable for short-range/hobby use |
| Booster path | Good idea — allows upgrading to a higher-voltage stage without rerouting |
| Level shifter | Standard practice; use a fast MOSFET driver (e.g. TC4427) rather than a resistor divider to keep edge rates sharp |

**Risk:** Slow edges on the pulser degrade range resolution. Target < 20 ns rise/fall.

**Idea:** Add a small ferrite bead + TVS on the transducer output line to absorb the kick-back from the piezo capacitance and protect the MD0100 limiter.

---

### 2. MD0100 Limiter / Protection

The MD0100 (Microchip/formerly Supertex) is a passive T/R switch and limiter designed exactly for this role — it presents low impedance to the transmit pulse and high impedance / clamped voltage during receive. **This is the right part.**

| Item | Assessment |
|------|-----------|
| Insertion loss | ~1.5 dB in receive — acceptable |
| Clamping voltage | ±2.5 V typical — within the 3.3 V op-amp rails |
| Bandwidth | Rated to 100 MHz — comfortably covers 1–10 MHz transducer range |

**Risk:** The MD0100 requires a small hold-off resistor (typically 200–500 Ω) in series on the transducer side to limit transmit current through the limiter diodes. Missing this resistor can destroy the part.

**Idea:** Add a 1:1 RF transformer (e.g. Mini-Circuits ADT1-1WT) between the transducer and the MD0100 for galvanic isolation and common-mode rejection. Not mandatory, but improves SNR in noisy environments.

---

### 3. Input MUX (MD0100 output vs. DAC self-test)

Switching between the live signal path and an internal DAC-driven test tone is excellent design practice. Allows:
- Calibration of the gain chain without a transducer attached
- Characterisation of ADC non-linearity
- Factory/field verification

**Idea:** Use an analog switch (e.g. TS5A3159 or SN74CBTLV3257) rather than a mechanical jumper so the MCU can toggle paths under firmware control. This enables automated self-test sequences.

---

### 4. 2-Stage I2C-Controlled Op-Amp Gain

The PIC32AK has three integrated 100 MHz op-amps — using two for a 2-stage VGA is clever (keeps external part count low). The I2C-controlled gain most likely uses a digital potentiometer (e.g. MCP4531 or AD5245) in the feedback network.

| Item | Assessment |
|------|-----------|
| Bandwidth | 100 MHz GBW — sufficient for 10 MHz transducers; marginal for 20 MHz |
| Noise | Integrated op-amps typically NF ~10 nV/√Hz — competitive but not audiophile-grade |
| Gain range | Depends on pot value ratio; design for at least 20–60 dB total across two stages |

**Risks:**
1. **Stability:** Two cascaded inverting stages at high gain can oscillate. Each stage needs a feedback capacitor (10–100 pF across the feedback resistor) to limit high-frequency gain.
2. **I2C speed vs. gain-switching latency:** Standard 100 kHz I2C is too slow to implement real-time time-gain compensation (TGC). Use 400 kHz Fast Mode and pre-program a ramp sequence at pulse time.

**Ideas:**
- **Stage 1:** Fixed gain (~20 dB), optimised for noise figure — use a low-noise bipolar input stage configuration.
- **Stage 2:** Variable gain (0–40 dB) via digital pot — handles TGC.
- Consider adding a simple RC low-pass (anti-aliasing) filter between stage 2 and the ADC, cutoff at ~18 MHz for a 40 MSPS system (Nyquist = 20 MHz).

---

### 5. Mid-Rail Bias

The ADC input is single-ended and the op-amps are rail-to-rail. Biasing to Vcc/2 (1.65 V) is correct for maximising dynamic range with a unipolar supply.

**Idea:** Use the third integrated op-amp as a unity-gain mid-rail buffer (Vcc/2 from a resistor divider into op-amp follower). This provides a stiff, low-impedance reference that decouples the bias from signal-path currents. Bypass the divider output with 10 µF + 100 nF to ground.

---

### 6. 40 MSPS ADC (12-bit, integrated)

The PIC32AK's onboard ADC is a key differentiator.

| Item | Assessment |
|------|-----------|
| Sample rate | 40 MSPS → Nyquist at 20 MHz, good for 1–15 MHz transducers |
| Resolution | 12-bit → 72 dB theoretical SNR; likely 65–68 dB ENOB in practice |
| Trigger source | Internal PWM → ADC trigger is supported; verify trigger latency jitter (should be < 1 sample period = 25 ns) |

**Risk (critical): Data rate.** At 40 MSPS × 12 bits = 480 Mbit/s. Even if only capturing a 100 µs window (4000 samples × 2 bytes = 8 kB per pulse), at 1 kHz PRF that's 8 MB/s — right at the USB 2.0 Full Speed ceiling.

**Mitigation:** Buffer a full acquisition burst in the 16 kB RAM, then stream at lower rate between pulses. At ≤ 500 Hz PRF with 100 µs windows this is comfortable. Document the PRF ceiling explicitly.

---

### 7. MCP2200 USB-UART Bridge

The MCP2200 is USB Full Speed (12 Mbit/s) with a practical UART throughput of ~1 MB/s.

| Item | Assessment |
|------|-----------|
| Throughput | Sufficient for burst-mode streaming at ≤ 500 Hz PRF × 4000 samples |
| Latency | Fine for offline post-processing; not suitable for real-time display at high PRF |
| Driver support | HID + CDC dual-mode; CDC is easier for Python/MATLAB host scripts |

**Risk:** At high PRF or long acquisition windows, the UART FIFO can overflow. Implement a hardware flow-control handshake (CTS/RTS) between PIC32A and MCP2200.

**Idea:** For a future revision, consider replacing MCP2200 with a USB 2.0 High Speed bridge (e.g. FTDI FT2232H at 480 Mbit/s) to remove the bandwidth ceiling entirely. For the current scope, MCP2200 is fine if PRF and window length are kept within limits.

---

## Key Design Gaps to Resolve Before PCB Tape-Out

1. **Anti-aliasing filter spec** — Define cutoff frequency and topology (active Sallen-Key or passive RC) between the last op-amp stage and the ADC input. Cutoff target: 18 MHz.

2. **Digital potentiometer selection** — Confirm part number, I2C address, wiper resistance vs. frequency, and end-to-end gain calculations. MCP4531-103 (10 kΩ, 128-step) is a reasonable starting point.

3. **Pulser edge rate** — Specify maximum allowable rise time and choose the level-shift driver accordingly. Add this to the schematic acceptance criteria.

4. **ADC input protection** — Even with MD0100 limiting to ±2.5 V, add Schottky clamp diodes (e.g. BAT54S) from ADC input to 3.3 V and GND as a second line of defence.

5. **Firmware skeleton** — The repo has zero firmware. At minimum, stub out: (a) PWM config, (b) ADC trigger config, (c) DMA transfer from ADC to RAM, (d) UART TX of burst data. The PIC32A peripheral library examples for ADC + DMA are the fastest starting point.

---

## Suggested Next Steps

| Priority | Action |
|----------|--------|
| High | Add KiCad schematic — even a first-pass block-level schematic exposes connection errors early |
| High | Write ADC + DMA firmware stub and test with DAC self-test path |
| High | Define gain stage feedback networks and simulate in LTspice |
| Medium | Specify anti-aliasing filter and include it in the schematic |
| Medium | Add CTS/RTS lines to MCP2200 UART interface |
| Low | Evaluate FTDI FT2232H as a future USB upgrade path |
| Low | Add RF transformer option footprint on transducer input (optional DNP) |

---

## Overall Feasibility Verdict

| Aspect | Rating | Notes |
|--------|--------|-------|
| Architecture | ✅ Solid | Signal chain topology is standard and well-proven |
| Component selection | ✅ Good | PIC32AK + MD0100 combination is well-matched |
| USB bandwidth | ⚠️ Constrained | Feasible within PRF/window limits; document ceiling |
| Analog front-end | ⚠️ Incomplete | Needs filter spec and gain calculations |
| Firmware | ❌ Missing | Blocking for validation; begin stub immediately |
| PCB readiness | ❌ Not yet | Schematic must be completed first |

The project is on the right track. The hardest part — choosing an MCU with the right integrated peripherals — is done well. The remaining work is mostly detailed analog design and firmware, both of which are tractable for a single developer.
