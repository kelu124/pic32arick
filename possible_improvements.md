# pic32arick — Cross-Project Improvement Analysis

**Signal chain baseline:** 5V PWM pulser (IRLML6244/2244 + TC4427A) → MD0100 T/R limiter → 2-stage integrated op-amp gain (MCP4531-103 I2C pot) → passive AAF → PIC32AK 12-bit 40 MSPS ADC → MCP2200 USB  
**Sources cross-referenced:** WULPUS (ETH Zurich, IUS 2022), PuLsE (Giordano 2025 IoT J.), TinyProbe (Vostrikov 2025 TUFFC), Anatomy slides (Leitner & Giordano, CEEUS 2026), Rheonics/Vostrikov (CEEUS 2026), Weik et al. survey (IEEE RBME 2026)  
**Constraint:** No major redesign; no exotic components; applies to PIC32AK + 40 MSPS ADC + single-element transducer + 5V pulser + I2C TGC as built.

---

## 1. From WULPUS

### 1.1 Replace Stage 1 integrated op-amp with OPA836

**What WULPUS does:** Uses an external OPA836 (Texas Instruments, SOT-23-5) as its fixed-gain amplifier, power-gated during idle. The OPA836 has 205 MHz GBW, 2.5 nV/√Hz input-referred noise, and a hardware enable pin.

**Why it applies:** The PIC32AK's integrated op-amps are specified at 100 MHz GBW with ~10 nV/√Hz noise. Stage 1 is the noise-critical stage — it sets the noise figure for the entire receive chain. Substituting OPA836 for OA1 would reduce input-referred noise by ~4× (2.5 vs 10 nV/√Hz), improving chain SNR by ~12 dB at low signal levels. The OPA836 is a 5-pin SOT-23-5 drop-in with rail-to-rail output and 3.3V operation; it adds one external IC to the BOM.

**Implementation:** Replace OA1 (Stage 1, fixed ~14.5 dB) with OPA836. Use the enable pin (pulled low during transmit via PWM_G2, the blanking gate already in the design) to blank the amplifier for the ring-down duration. This eliminates saturation recovery time and shortens the effective dead zone from ~5 µs to ~1–2 µs. Stages 2 and 3 (bias gen) remain as integrated PIC32AK op-amps.

**Cost:** ~€1.50. Package: SOT-23-5 (0802-comparable footprint). Supply: single 3.3V.

---

### 1.2 Add transducer-side mux for multi-element scanning

**What WULPUS does:** Uses the HV2707 8-channel high-voltage analog mux (Microchip, SPI-controlled, ±80V rated) to time-multiplex 8 transducer elements through a single receive chain. Combined with 8 elements, this gives multi-site tissue monitoring at no additional ADC cost.

**Why it applies:** pic32arick's 5V pulser does not need HV-rated switches. At 5V, a low-cost standard analog mux suffices. The CD4051B (8:1, ±8V analog range, DIP-16 or SOIC-16) or ADG1408 (8:1, ±15V, SPI-compatible) would work at 5V. This enables scanning across multiple transducer positions (or a small custom linear array) without adding ADCs or changing the signal chain.

**Implementation:** Insert mux between PWM pulser output and MD0100, and a second mux between MD0100 and Stage 1 input (or use one mux at the transducer node, as WULPUS does). Control via PIC32AK SPI or GPIO. CD4051B: 3-bit address select, 1 enable, < €0.30. For a 5V-compatible single-supply version, ADG1408 (SPI-addressable, 50Ω on-resistance) at ~€1.50 is cleaner.

**Constraint:** On-resistance of ~50–200 Ω in the receive path adds noise (Johnson noise of Ron). At 200 Ω, contribution is 1.8 nV/√Hz — dominated by OA1 noise (acceptable). Would not work at HV; for future HV pulser upgrade, swap to HV2707 on same footprint.

---

### 1.3 Power-gate Stage 1 via enable pin during transmit dead zone

**What WULPUS does:** The OPA836 enable pin is pulled low during the transmit pulse and for a configurable hold-off period, preventing saturation of the first gain stage by the transmit artifact. Recovery to full gain takes ~100 ns — much faster than waiting for the saturated op-amp to recover naturally (~5–10 µs).

**Why it applies:** The current design uses PWM_G2 as a blanking gate — but only if the external op-amp has an enable pin. The integrated PIC32AK op-amps do not have enable pins. If OPA836 is adopted per improvement 1.1, this blanking is immediately available: route PWM_G2 to OPA836 EN (active-high). The hold-off period (blanking gate width) is set by PDC2 in the PIC32AK PWM module, adjustable per-application with 2.5 ns resolution.

**Implementation:** PWM_G2 → OPA836 EN. Set PHASE2 = 0 (starts at t=0, same as transmit) and PDC2 = dead_zone_duration / 2.5 ns. This is a 0-component change given OPA836 is already used — just a GPIO connection and firmware register.

---

### 1.4 WULPUS-style DMA chaining for zero-overhead acquisition

**What WULPUS does:** Uses the MSP430FR5043 USS_A sequencer to trigger ADC acquisitions and DMA transfers autonomously, with zero CPU involvement during the receive window. The CPU wakes only when a full burst is in SRAM.

**Why it applies:** The design_ideas.md already flags this as the right approach; the PIC32AK supports ADC-DMA chaining. The specific WULPUS pattern worth borrowing: use a **ping-pong DMA buffer** (two alternating SRAM regions, each holding one burst). While buffer A is being transmitted over USB, buffer B is capturing the next acquisition. This eliminates dead time between pulses caused by USB latency and doubles effective throughput at the same PRF.

**Implementation:** Configure DMA with two descriptor chains, alternating between Buffer_A and Buffer_B (each 8 KB for 4000 samples × 2 bytes). DMA interrupt fires at end of each buffer-fill; firmware posts a "TX ready" flag and swaps to the other buffer. ADC acquisition continues uninterrupted. Firmware change only; no hardware cost.

---

## 2. From PuLsE

### 2.1 Analog envelope detection path for MCP2200-constrained applications

**What PuLsE does:** Performs envelope detection in the analog domain before the ADC, reducing required ADC bandwidth by >5×. A Schottky diode half-wave rectifier followed by a low-pass filter produces the signal envelope directly. The ADC then samples the slowly-varying envelope rather than the raw RF carrier.

**Why it applies:** The MCP2200 USB bridge limits the effective PRF to ~125 Hz for full 8 KB bursts. Many applications (depth measurement, presence detection, heart-rate monitoring, thickness gauging) only need the signal envelope, not raw RF. Adding an analog envelope path allows the same MCP2200 to stream data at >1 kHz effective PRF by reducing burst size from 8 KB (4000 RF samples) to ~400 bytes (200 envelope samples at ~1 MSPS equivalent).

**Implementation:** After Stage 2 output (before AAF), add a fork:
- **Path A (raw RF):** existing AAF → ADC input (unchanged)
- **Path B (envelope):** Schottky diode (BAT54, anode to Stage 2 out, cathode to RC filter) + 1 kΩ + 100 pF RC low-pass (f₀ ≈ 1.6 MHz, adequate for depth envelope at 5 MHz carrier) → mux input B

The analog mux (TS5A3159 or SN74CBTLV3257, already mentioned in design_ideas.md for the self-test path) selects between raw RF (path A) and envelope (path B) under firmware control. Adding one BAT54 diode, one 1 kΩ, and one 100 pF cap to the existing mux footprint costs < €0.20.

**Important caveat (from PuLsE analysis):** Analog enveloping is irreversible — phase information and coherent processing (matched filtering, Doppler) are lost on the envelope path. Retain path A (raw RF) as the default for development; path B is for deployment where only amplitude is needed.

---

### 2.2 Low-PRF duty cycling to extend USB headroom

**What PuLsE does:** At 5.8 mW total system power, PuLsE operates at low PRF (heart rate monitoring requires ~50–200 Hz temporal resolution, not 6.67 kHz). The system sleeps between acquisitions.

**Why it applies:** pic32arick's 6.67 kHz PRF (150 µs period from pulser_design.md) produces 8 KB bursts that the MCP2200 cannot drain in real-time. Setting PRF to 200 Hz (5 ms period) gives 4.8 ms between bursts — comfortably enough for MCP2200 to drain 8 KB at 1 MB/s before the next burst arrives. This requires no hardware change, only a firmware parameter adjustment (PTPER register).

**Implementation:** Change `PTPER = 59999` (150 µs, 6.67 kHz) to `PTPER = 1999999` (5 ms, 200 Hz) for contact imaging applications. The ADC trigger and dual-pulse timing scales automatically. This immediately unlocks real-time A-scan streaming without any hardware change.

---

## 3. From TinyProbe

### 3.1 Replace I2C digital pot (MCP4531) with SPI version (MCP4161) for fine-grained TGC

**What TinyProbe does:** Uses 30 MSPS ADC with programmable delay beamforming — precise, hardware-timed control. The key principle: TGC (time-gain compensation) must track the echo's depth in real time, with step resolution finer than the ADC sample period.

**Why it applies:** The current design uses I2C at 400 kHz for TGC via MCP4531. Each gain step takes ~45 µs (2 bytes × 9 bits / 400 kHz), giving a 16-step ramp over ~720 µs. At 40 MSPS and 1500 m/s sound speed, 45 µs = 33.75 mm depth step — very coarse. SPI at 10 MHz reduces per-step time to <1 µs, enabling a 64-step ramp over 64 µs (48 mm total depth at <1 mm per step).

**Implementation:** Replace MCP4531-103 (I2C, address 0x2C) with **MCP4161-103** (SPI, otherwise identical: 10 kΩ end-to-end, 128 steps, MSOP-8 or DIP-8). The footprint is the same package family; the wire connections change from SDA/SCL to SDI/SCK/CS. One additional CS line from PIC32AK is needed. MCP4161 costs ~€1.20, comparable to MCP4531. Firmware change: I2C transactions → SPI transactions (PIC32AK SPI peripheral, already present).

**TGC ramp implementation:** Pre-compute the gain ramp table as an array of 64 SPI bytes. Use DMA to send the table to SPI at a fixed rate (one byte per µs, clocked by a timer). This runs entirely in background with zero CPU overhead during acquisition.

---

### 3.2 On-chip decimation before USB to multiply effective PRF

**What TinyProbe does:** Runs neural-network channel reconstruction on-FPGA to reduce data volume before Wi-Fi transmission. The principle: compress on the acquisition device, not on the host.

**Why it applies:** For a single-channel 40 MSPS system with a 5 MHz transducer, the ADC oversamples by 8× relative to Nyquist. A 4-tap moving average (implemented in firmware) decimates 40 MSPS → 10 MSPS for 5 MHz transducers with no loss of information, reducing USB data volume by 4× and raising the MCP2200 effective PRF ceiling from ~125 Hz to ~500 Hz.

**Implementation:** After DMA captures the burst into SRAM, apply a 4-sample sum-and-shift (bit-shift right by 2) before copying to the USB TX buffer. Total compute: 4000 additions + 4000 shifts = ~8000 instructions at 200 MHz PIC32AK clock ≈ 40 µs. Well within the inter-pulse gap at any practical PRF. No hardware change; firmware only.

For a 10 MHz transducer, decimate by 2× instead (20 MSPS effective); for 15 MHz, no decimation needed. Make the decimation factor a firmware parameter.

---

## 4. From Anatomy Slides (Leitner & Giordano, CEEUS 2026)

### 4.1 Blanking gate as first-class timing signal

**What the Anatomy taxonomy identifies:** The five-block model (TX → T/R → RX → Compute → Host) treats the blanking gate as a separate control output, not a side effect of the TX pulse. The blanking gate duration is a design parameter equal to the dead zone depth × 2 / c.

**Why it applies:** The dual PWM architecture in pulser_design.md already allocates PWM_G2 as a blanking gate. The Anatomy paper formalizes that this signal should also: (1) gate the Stage 1 enable (per improvement 1.3 above), (2) disable the I2C/SPI TGC controller during transmit, and (3) optionally gate the ADC trigger to prevent false triggers on the transmit artifact.

**Implementation:** Wire PWM_G2 to: OPA836 EN (if OPA836 is used), a GPIO-controlled EN line on the MCP4161/MCP4531, and the ADC trigger INHIBIT input (check PIC32AK ADC control register for trigger masking). This is 3 GPIO connections and firmware configuration — zero BOM cost.

---

### 4.2 Transducer frequency selection for PIC32AK's Nyquist ceiling

**What the Anatomy paper shows:** System efficiency scales as mW/MHz of transducer center frequency. At 40 MSPS, the PIC32AK's hard Nyquist ceiling is 20 MHz. The sweet spots for maximising resolution within this constraint:

| Transducer | Nyquist ratio | Oversampling | Resolution |
|------------|--------------|-------------|-----------|
| 2.25 MHz (WULPUS) | 8.9× | Excellent | ~0.33 mm in water |
| 5 MHz | 4× | Good | ~0.15 mm in water |
| 10 MHz | 2× | Marginal | ~0.075 mm in water |
| 15 MHz | 1.33× | Below 2× — aliasing risk | ~0.05 mm |

**Recommendation:** Target 5 MHz as the primary transducer (e.g. Olympus V110-RM, ~$200, BNC connector). At 5 MHz, 40 MSPS gives 8 samples/wavelength — adequate for interpolated peak-finding. At 10 MHz, apply the 2× decimation trick from improvement 3.2 to restore 4 samples/wavelength before peak finding.

Avoid ≥15 MHz transducers with this ADC without adding a 2× anti-aliasing pre-filter.

---

## 5. From Rheonics / Vostrikov CEEUS 2026

### 5.1 SPI-controlled analog mux instead of I2C for all gain/path control

**What the Rheonics paper documents:** WULPUS's reproducibility was hindered by the 3-toolchain complexity (TI CCS for MSP430 + Segger for nRF + Nordic SDK). A key lesson: the fewer independent control buses, the lower the firmware integration cost.

**Why it applies:** pic32arick currently mixes I2C (MCP4531 gain control) with PWM (pulse timing) and UART (USB bridge). Consolidating gain control to SPI (MCP4161, per improvement 3.1) means a single SPI bus can also control the transducer mux (ADG1408, per improvement 1.2) and the path selector (ADG1208 or similar). One SPI bus, multiple CS lines — straightforward DMA operation, no I2C clock-stretching edge cases.

**Implementation:** Route SPI1 of PIC32AK to: MCP4161 CS (gain), ADG1408 CS (mux), optional analog switch CS. Reserve I2C for slow control (e.g., a future accelerometer or EEPROM for calibration data). This is a layout decision, not a circuit change.

---

### 5.2 Open-source from inception: Tag-Connect + KiCad

**What the Rheonics paper warns:** WULPUS's Altium dependency created a 12-month lag between publishing and community replication. pic32arick already specifies KiCad (correct) and Tag-Connect (correct). The additional recommendation from Vostrikov's lessons learned:

- **Provide a JLCPCB-compatible BOM from day 1.** Map every component to a JLCPCB LCSC part number in the KiCad BOM fields. This eliminates sourcing friction for the most common low-cost PCB+assembly path.
- **Add calibration footprints on the PCB.** Include a 0402 resistor footprint (DNP) in parallel with Rf1 (Stage 1) and Rf2 (Stage 2) to allow trimming the fixed gain in case the integrated op-amp gain deviates from the calculated value. WULPUS builders reported per-unit signal variability — calibration pads prevent this issue.

**Cost:** Zero BOM cost. PCB area: ~4 × 0402 pads (< 1 mm² each).

---

## 6. From Weik 2026 Survey (IEEE RBME)

### 6.1 Active termination to minimise dead zone

**What the review confirms as a standard technique:** Clinical and wearable systems across TRL 6–9 all use some form of active ring-down termination to minimise the dead zone. The two approaches: (1) passive damping resistor (50–200 Ω across transducer, always present), or (2) active termination (switch a low impedance across the transducer ~T/2 after the pulse, cancelling the residual oscillation).

**Why it applies:** pulser_design.md already includes R_damp = 100 Ω. The additional technique: use the push-pull driver's **off state** (both FETs open) to allow the damping resistor to dissipate ring-down energy, rather than leaving the piezo floating. The key timing: after PWM_G1 goes low (end of pulse), immediately set both IRLML6244 and IRLML2244 to off state. The piezo then rings into R_damp only — no additional energy from the driver. Combined with OPA836 blanking (improvement 1.3), this reduces the dead zone from ~5 µs to ~1–2 µs in water.

**Implementation:** Firmware change only — ensure the PWM dead-band registers (DTRx) force both FET gates off simultaneously after the pulse. The TC4427A's two channels, when both inputs are low, both output low — correctly turning off both MOSFETs. Verify this behaviour in the TC4427A datasheet (both outputs pull their respective gate to the supply/ground to turn FETs off).

---

### 6.2 SPI EEPROM for calibration data storage

**What the review identifies as a gap across all systems:** Per-unit calibration (gain trim, timing offset, transducer characterisation) is absent in almost all prototype systems, limiting replication fidelity. This is directly mentioned in WULPUS builder feedback (todo.md Gap #8).

**Why it applies:** The MCP2200 USB bridge has no NVM for calibration storage. Adding a small SPI EEPROM (e.g. 25AA040A, 4Kbit, SOT-23-5, ~€0.50) on the existing SPI bus (with a dedicated CS line) allows storing: gain calibration coefficients for both stages, ADC offset, transducer serial number, and a calibration date. The host Python library reads the EEPROM on connect, applies corrections, and outputs calibrated amplitude data. Users replicating the board can run a one-time calibration routine and store results permanently on the device.

**Cost:** 25AA040A: ~€0.50, SOT-23-5. No firmware complexity beyond standard SPI read/write.

---

## Summary — Priority Order

| Priority | Improvement | Source | Type | BOM cost |
|----------|-------------|--------|------|----------|
| **H** | OPA836 for Stage 1 (12 dB SNR improvement) | WULPUS | 1 IC swap | ~€1.50 |
| **H** | MCP4531 → MCP4161 (SPI TGC, 64× finer steps) | TinyProbe | 1 IC swap | ~€0 delta |
| **H** | Ping-pong DMA buffer for continuous acquisition | WULPUS | Firmware only | €0 |
| **H** | Analog envelope path (BAT54 + RC + mux fork) | PuLsE | 3 passives | <€0.20 |
| **M** | On-chip decimation before USB (4× PRF increase) | TinyProbe | Firmware only | €0 |
| **M** | OPA836 enable → PWM_G2 blanking gate wiring | WULPUS + Anatomy | 1 GPIO wire | €0 |
| **M** | CD4051B/ADG1408 transducer mux (8 elements) | WULPUS | 1 IC + layout | ~€0.30–1.50 |
| **M** | SPI bus consolidation (mux + pot on one bus) | Rheonics | Layout only | €0 |
| **L** | SPI EEPROM for calibration storage (25AA040A) | Weik survey | 1 IC | ~€0.50 |
| **L** | JLCPCB LCSC BOM mapping + trim resistor pads | Rheonics | Layout/BOM | €0 |
| **L** | Set PRF = 200 Hz in firmware for real-time USB streaming | PuLsE | Firmware only | €0 |

The three highest-impact changes for a v1 prototype: OPA836 Stage 1, MCP4161 SPI pot, and ping-pong DMA. These together lift SNR by ~12 dB, enable fine TGC, and remove the USB latency bottleneck — all without changing the fundamental signal chain architecture.
