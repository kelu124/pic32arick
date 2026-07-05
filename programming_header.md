# pic32arick — On-Board Programming Interface

**MCU:** PIC32AK1216GC41064 (64-pin LQFP)  
**Programmer:** MPLAB Snap (low-cost, confirmed in README), MPLAB ICD 4, or MPLAB PICkit 5  
**Interface:** 2-wire ICSP (In-Circuit Serial Programming)  
**Date:** 2026-07-05  

---

## TL;DR

| Question | Answer |
|----------|--------|
| Boot button required? | **No** — MCLR is controlled by the programmer; no button needed |
| Programming interface | 2-wire ICSP: MCLR + PGEC1 + PGED1 + VDD sense + GND |
| Minimum header | 5-pin, 0.1" single-row |
| Compact option | Tag-Connect TC2030-IDC-NL (no connector, pads only) |
| High-voltage on MCLR? | No — LVP mode uses normal VDD; set CFGLVP config bit to enable |
| Dedicated BOOT pin? | No — PIC32AK has no separate boot pin; boot mode via config bits |

---

## Boot Button — Not Required

Unlike STM32 (BOOT0 pin) or ESP32 (IO0), PIC32 devices do not use a dedicated boot pin to enter programming mode. The MPLAB programmer takes control of the MCLR line directly over the ICSP header. There is no need for a button.

What **is** required on the board:

- **10 kΩ pull-up from MCLR to VDD** — holds the MCU in run mode when the programmer is disconnected. Do not tie MCLR directly to VDD; that disables external reset and prevents ICSP.
- **100 nF bypass capacitor from MCLR to GND** — filters glitches on the reset line. Place close to the MCLR pin.

```
VDD (3.3V)
    │
  [10 kΩ]
    │
    ├──[100 nF]── GND
    │
   MCLR (MCU pin)  ──→ header pin 1
```

An optional **reset pushbutton** (normally open, from MCLR to GND) is useful for manual resets during development. It is not required for programming.

---

## ICSP Programming Pins

The PIC32AK ICSP interface uses three signal pins plus power/ground:

| Signal | Function | Direction |
|--------|----------|-----------|
| MCLR | Reset / programming enable | Input to MCU |
| PGEC1 | Programming clock | Input to MCU |
| PGED1 | Programming data | Bidirectional |

**Pin assignment in the 64-pin LQFP package:**  
⚠️ Verify exact pin numbers against Table 4-x (Pin Diagrams) in the PIC32AK1216GC41064 datasheet (DS70005592). The ICSP pins are multiplexed with GPIO and their physical package pin numbers must be confirmed before PCB layout.

From the datasheet, search for `PGEC1`, `PGED1`, and `MCLR` in the pinout table. These are the only three pins needed on the signal side.

**PGEC2 / PGED2:** The PIC32AK has alternate ICSP pin pairs. Unless PGEC1/PGED1 conflict with critical signals in your design, use pair 1 — it is the default for MPLAB Snap.

---

## Programming Header — Standard 6-Pin ICSP (Microchip Standard)

The Microchip standard ICSP header is a **6-pin, 0.1" single-row** connector. This is what MPLAB Snap, ICD 4, and PICkit 5 cable out-of-the-box:

```
        ┌───────────────────────────────────────┐
Pin 1 → │ MCLR/Vpp  │ VDD  │ GND  │ PGED1 │ PGEC1 │ NC │
        └───────────────────────────────────────┘
         1           2      3      4        5       6

1: MCLR   — reset / program-enable
2: VDD    — sense line (programmer reads supply voltage; do NOT power MCU from this)
3: GND
4: PGED1  — data (bidirectional)
5: PGEC1  — clock
6: NC     — no connect (leave floating; sometimes used for LVP on older devices)
```

**Important:** Pin 2 (VDD) is a **sense line only** on the MPLAB Snap. The MCU must be powered by its own supply (USB 5 V or onboard regulator). Never remove C_byp caps from VDD pins when using ICSP.

### Schematic symbol

```
J_ICSP (2×3 or 1×6, 0.1" pitch)
┌──────────────┐
│ 1  MCLR      │
│ 2  VDD_SENSE │
│ 3  GND       │
│ 4  PGED1     │
│ 5  PGEC1     │
│ 6  NC        │
└──────────────┘
```

Add a 47 Ω series resistor on PGEC1 and PGED1 between the MCU pin and the header. This limits current if the programmer connects while the lines are being driven by other circuitry.

---

## Low Voltage Programming (LVP)

LVP allows ICSP at normal VDD voltage — no 12 V on MCLR required. Enable it by setting the `CFGLVP` configuration bit in the device configuration registers (set during programming; not a physical pin).

MPLAB Snap supports LVP natively. **Recommended:** enable LVP in all configurations. Without it, ICSP requires Vpp ≈ 8.5–9 V on MCLR, which the MPLAB Snap cannot supply.

---

## Compact Option: Tag-Connect TC2030-IDC-NL

For a minimal-BOM / minimal-footprint board, replace the 6-pin through-hole or SMD header with a **Tag-Connect TC2030-IDC-NL** footprint. This is a 6-pin spring-loaded pogo-pin connector — no board-mounted connector at all, just a pad pattern.

Benefits:
- Zero BOM cost for the connector
- ~6 × 10 mm PCB footprint
- Mates with the standard Microchip ICSP cable via the TC2030-CLIP-3LEGGED adapter

Pin mapping is identical to the 6-pin ICSP header above.

```
Tag-Connect TC2030 pad layout (top view):

  ○  ○  ○     (round pads: 1=MCLR, 2=VDD, 3=GND)
  ○  ○  ○     (round pads: 4=PGED1, 5=PGEC1, 6=NC)
    ▲  ▲       (triangular locating pins)
```

Download the KiCad footprint from tag-connect.com.

---

## Summary: What to Put on the PCB

| Item | Value/Note |
|------|-----------|
| MCLR pull-up | 10 kΩ to VDD, 1% |
| MCLR bypass cap | 100 nF, 0402, close to pin |
| Reset button (optional) | SPST, normally open, MCLR to GND |
| Series resistors on PGEC/PGED | 47 Ω, 0402 |
| Programming header | 6-pin 0.1" single-row **or** Tag-Connect TC2030 pads |
| Programmer | MPLAB Snap (≈ €10 via Mouser) |
| LVP config bit | Enable (CFGLVP = 1) |

No boot button. No level shifters. No separate programming voltage. The MCLR + 2 signal pins + power/ground is everything the programmer needs.
