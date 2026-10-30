# USB-C-CHARGER
A portable USB-C charger(power bank) designed in Altium Designer. It has two USB-C ports: one to charge the internal 3.7 V Li-Po cell (MCP73831) and one to charge your phone or other devices from the cell (PAM2401 boost converter, 5 V / 1 A).

## Table of Contents
1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [Specifications](#specifications)
4. [Schematics](#schematics)
5. [Design Calculations](#design-calculations)
6. [Bill of Materials](#bill-of-materials)
7. [Custom Footprints and Symbols](#custom-footprints-and-symbols)
8. [Known Issues and Design Notes](#known-issues-and-design-notes)


## Overview

The design has two sub-circuits:

| Block | IC | Function |
|---|---|---|
| Charge controller | Microchip **MCP73831T-2ACI/OT** | CC/CV Li-Po charger, 4.2 V cutoff, 500 mA charge current |
| Step-up converter | Diodes/PAM **PAM2401** | Boosts 3.0-4.2 V battery to a regulated ~5 V, up to 1 A |

Each block has its own USB-C connector: **USB-C input** (charge) and **USB-C output** (power-bank port, with CC pull-ups and iOS-style D+/D- dividers).

## How It Works

Every power bank is the same idea: store energy in a lithium cell, then convert it to 5 V for USB. This design does exactly that with two small circuits.

1. **Charging (USB-C input):** a 5 V charger is plugged into the input port. The MCP73831 charges the cell in two phases: constant current (0.5 A) until the cell reaches 4.2 V, then constant voltage while the current tapers off, and it stops when the current falls to about 7.5 % of 0.5 A. The CHG LED is on while charging.
2. **Storage:** a single 3.7 V (nominal) Li-Po cell, 2000 mAh, stores 7.4 Wh. The cell voltage moves between about 3.0 V (empty) and 4.2 V (full).
3. **Discharging (USB-C output):** USB needs 5 V, which is higher than the cell can ever give, so the PAM2401 step-up (boost) converter raises the 3.0-4.2 V cell voltage to a regulated ~5 V. The PWR LED shows the output is live.
4. **Device detection:** the output port has CC pull-up resistors (so USB-C devices see a power source) and D+/D- voltage dividers (so Apple devices accept 500 mA charging).

### About the capacity rating (mAh)

The 2000 mAh figure is the cell's rating **at 3.7 V**. This is the same convention real power banks use. A "10000 mAh" bank is several 3.7 V cells in parallel, and the label counts their capacity at 3.7 V, not at the 5 V output. After boosting to 5 V and converter losses, usable capacity at 5 V is roughly 60-70 % of the label. For this build: 7.4 Wh x ~90 % efficiency = ~6.7 Wh, which is about **1300 mAh at 5 V**.

This design uses **one cell** (PKCELL 803860, an 8 x 38 x 60 mm pouch cell), so it is a small power bank, roughly one fifth of a typical 10000 mAh unit. It can add about 40 % charge to a 4000 mAh phone.

### Limitations

- 5 V only, no USB-PD or Quick Charge, 1 A maximum output.
- Cannot charge the cell and power a device at the same time (no load sharing).
- Charge current is 0.5 A, so a full charge takes about 4.5-5 hours.

## Specifications

| Parameter | Value |
|---|---|
| Battery | PKCELL 803860, Li-Po, 3.7 V nominal, 2000 mAh (7.4 Wh) |
| Charge input | 5 V via USB-C |
| Charge current | 0.5 A (set by R2 = 2 kΩ) |
| Charge cutoff voltage | 4.2 V |
| Output | ~5 V, 1 A max |
| Boost input range | 0.9-4.75 V per converter note (3.0-4.2 V expected for Li-Po) |
| Peak efficiency | up to 95 % (datasheet claim) |
| Indicators | CHG LED (charging), PWR LED (output present) |
| Battery switch | SW1 (PWR) in series with the battery |

## Schematics


### Charge controller (MCP73831)
- U1 takes 5VIN on VIN (pin 4) and delivers VBATT (pin 3).
- **R2 (2 kΩ)** on PROG sets the fast-charge current.
- **STAT** drives LED1 through R1 (1 kΩ), so the LED is on while charging.
- C2 and C1 (4.7 µF each) decouple input and battery pins, as the datasheet requires.
- SW1 lets you disconnect the battery; J1 is the battery connector.

### Step-up converter (PAM2401)
- L1 (2.2 µH) from VIN to LX; C3 (22 µF) on the input, C4 (22 µF) on the output.
- **R5 / R3** (750 kΩ / 100 kΩ) form the feedback divider that sets the output voltage.
- R4 (750 kΩ) pulls EN to VIN so the converter is enabled whenever the battery is connected.
- LED2 + R7 (4.7 kΩ) indicate 5 V present.
- R7-R10 form the D+/D- dividers advertising 500 mA to Apple devices.
- R12/R13 (56 kΩ) pull CC1/CC2 up to +5 V so the port acts as a USB-C source.
- D1/D2 (40 V / 1.5 A) are the protection diodes noted on the sheet.

## Design Calculations

### 1. Charge current (MCP73831)

I_REG = 1000 V / R_PROG(kΩ) = 1000 / 2 = **500 mA** (0.25 C for a 2000 mAh cell)

| Stage | Value | Notes |
|---|---|---|
| Pre-conditioning current | 10 % of I_REG = **50 mA** | below the pre-charge threshold (~3.0 V), per datasheet |
| Fast charge (CC) | **500 mA** | |
| Regulation voltage | **4.2 V** | "-2" voltage option |
| Termination current | ~7.5 % of I_REG ≈ **37.5 mA** | check the exact % for the "A" option in the datasheet |

Estimated charge time: 2000 mAh / 0.5 A ≈ 4 h for CC, plus ~0.5-1 h for CV → **about 4.5-5 h** from empty.

LED indicator current: (5 V − V_F ≈ 2 V) / 1 kΩ ≈ **3 mA**.

### 2. Charger power dissipation

The MCP73831 is a linear charger, so it dissipates (V_IN − V_BATT) × I_CHG.

| V_BATT | P_diss at 0.5 A |
|---|---|
| 3.0 V | 1.00 W |
| 3.7 V | 0.65 W |
| 4.1 V | 0.45 W |

The SOT-23-5 package has a junction-to-ambient resistance of roughly 200+ °C/W, so around 0.65 W the die reaches its thermal regulation limit and the charge current is reduced automatically. Give U1 as much copper as possible (see Known Issues).

### 3. Boost output voltage (PAM2401)

V_OUT = V_FB × (1 + R5 / R3) = 0.6 V × (1 + 750 k / 100 k) = 0.6 × 8.5 = **5.1 V**

(Assumes V_FB = 0.6 V. Confirm against the datasheet.)

### 4. iOS D+/D- dividers (R7-R10)

V_D = 5.1 V × 49.9 k / (75 k + 49.9 k) = **≈ 2.04 V** on both D+ and D−

Both lines at ~2.0 V is the Apple signature for a **500 mA** charger. Divider current is only ≈ 41 µA.

### 5. Runtime and battery-side current

Battery energy: 3.7 V × 2.0 Ah = **7.4 Wh**

| Efficiency | Usable energy | Equivalent capacity at 5 V | Runtime at 5 V / 1 A (5 W) |
|---|---|---|---|
| 95 % | 7.03 Wh | ~1400 mAh | ~1.4 h |
| 90 % | 6.66 Wh | ~1330 mAh | ~1.3 h |
| 85 % | 6.29 Wh | ~1260 mAh | ~1.25 h |

Battery-side input current at full load (5.1 V × 1 A = 5.1 W) and 90 % efficiency:

| V_BATT | I_IN |
|---|---|
| 4.2 V | 1.35 A |
| 3.7 V | 1.53 A |
| 3.0 V | 1.89 A |

Choose an inductor with saturation current above the PAM2401 peak switch current limit, and make sure the battery, switch SW1 and J1 are rated for ≥ 2 A.

### 6. USB-C CC resistors

| Port | Resistor | Role |
|---|---|---|
| Output (R12/R13) | 56 kΩ pull-up to +5 V | Advertises **default USB power** as a source |
| Input | **5.1 kΩ pull-down to GND on CC1 and CC2** | Required for a sink (see Known Issues) |

## Bill of Materials


## Custom Footprints and Symbols



## Known Issues and Design Notes

These came up while reviewing the schematics. Fix them before sending the board for fabrication.

1. **Input CC network is wrong for a charge input.** R11/R14 pull CC1_IN/CC2_IN **up** to 5VIN with 56 kΩ. An input port that receives power is a *sink* and needs **5.1 kΩ pull-downs to GND** on both CC pins, otherwise a C-to-C source will not turn on VBUS. Pull-ups belong on the output port only.
2. **Duplicate designator R7** on the step-up sheet (4.7 kΩ LED resistor and 49.9 kΩ D- divider). Re-annotate.
3. **Output current advertisement.** 56 kΩ advertises default USB power (500/900 mA), but the output is rated 1 A. For a 1.5 A advertisement use **22 kΩ**.
4. **Connector part number.** The note mentions 10118193-0001LF as the cheap alternative, which appears to be a Micro-USB B receptacle rather than USB-C. Verify before using it as the USB-C alternative.
5. **Charger thermals.** With 0.5 A from a 5 V input, U1 will thermally throttle on warm days. Add a copper pour on the VSS pin and consider lowering charge current (R2 = 3.3 kΩ → 300 mA).
6. **D1/D2 role.** As drawn, they sit between the +5 V rail and GND. Confirm orientation and purpose against the PAM2401 application circuit.
7. **No load sharing.** Charging and discharging at the same time is not supported by this topology. Use the PWR switch.
8. **Battery protection.** Make sure the cell has a built-in protection circuit (PCM); there is no protection IC on this board.
