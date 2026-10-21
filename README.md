# High-Voltage EV Powertrain & Diagnostics Network Architecture
## Unified ISO 14229 (UDS) Parameter Identification & Telemetry Reference

---

## 1. System Overview

This repository provides an end-to-end technical specification, diagnostic parameter catalog, mathematical conversion handbook, and unified decoder suite for the High-Voltage ($HV$) Electric Vehicle Powertrain network. 

The onboard communication infrastructure implements **Unified Diagnostic Services (ISO 14229-1 / UDS)** over Controller Area Network (**ISO 11898-1 / ISO 15765-2 DoCAN**). Diagnostic telemetry can be acquired via two complementary mechanisms:
1. **Direct Physical Node Addressing:** Querying individual subsystem microcontrollers (`BMS`, `MCU`, `DCDC`, `OBC`) directly on their private powertrain CAN cluster to access raw high-precision engineering registers, internal status bitmasks, and sub-module hardware flags.
2. **Gateway / Domain Relayed Addressing:** Querying the central **Vehicle Electronic Control Unit (`VECU`)**, which continuously ingests, sanitizes, and aggregates powertrain metrics, re-encoding them into normalized diagnostic identifiers (DIDs) for external vehicle scan tools and high-level telematics units (TCU).

```
                            +-------------------------------------------------+
                            |               Vehicle OBD-II Port               |
                            |         (Standard Diagnostic Tester / VCI)      |
                            +------------------------+------------------------+
                                                     |
                                                     | Physical CAN (11-bit)
                                                     v
                                  +--------------------------------------+
                                  |     Vehicle Electronic Control Unit  |
                                  |               (VECU)                 |
                                  |     CAN-ID: 0x7E3 / Resp: 0x7EB       |
                                  +------------------+-------------------+
                                                     |
                 +-----------------------------------+-----------------------------------+
                 |           Powertrain / EV Subsystem CAN Bus (500 kbps)            |
                 |                                                                   |
                 v                                   v                               v
    +-------------------------+         +-------------------------+         +-------------------------+
    | Battery Management (BMS)|         |   Motor Controller (MCU)|         |  3-in-1 Power Converter |
    | CAN-ID: 0x785           |         |   CAN-ID: 0x783         |         |  * DCDC (0x784 / 0x78C) |
    | Resp:   0x78D           |         |   Resp:   0x78B         |         |  * OBC  (0x786 / 0x78E) |
    +-------------------------+         +-------------------------+         +-------------------------+
                 |                                   |                                   |
    - 112S Cell Voltage Taps            - Inverter DC Link & Phases         - High-Voltage Buck 12V
    - Laser Aerosol Smoke Sense         - Two's Complement Speeds           - Grid AC RMS Rectification
    - Dual Contactor Potentials         - Dynamic Motoring Limits           - J1772 / IEC 61851-1 CP
```

---

## 2. CAN Diagnostic Addressing Matrix

All physical communication utilizes standard 11-bit identifiers conforming to the standard $+8$ offset slave response architecture:

| Subsystem Node | Acronym | Physical Request ID | Expected Response ID | Functional Domain | Sub-Bus ID | DID Ranges | Parameter Count | Reference Specification |
|:---|:---:|:---:|:---:|:---|:---:|:---:|:---:|:---|
| **Vehicle Control Unit** | `[VECU]` | `0x7E3` | `0x7EB` | Gateway / Powertrain | `BCM: 1001` | `0x341D–0x354A`, `0x7002` | 68 | `vecu_uds_parameters_reference.md` |
| **Battery Management** | `[BMS]` | `0x785` | `0x78D` | Energy Storage / Safety | `BCM: 1003` | `0x3400–0x3568` | 42 | `bms_uds_parameters_reference.md` |
| **DC-DC Converter** | `[DCDC]` | `0x784` | `0x78C` | Auxiliary Low-Voltage | `BCM: 1003` | `0x130E–0x1329`, `0xF0F1–0xF0F4` | 20 | `dcdc_uds_parameters_reference.md` |
| **Motor Control Unit** | `[MCU]` | `0x783` | `0x78B` | Traction / Inverter | `BCM: 1003` | `0x0506–0x0520` | 15 | `mcu_uds_parameters_reference.md` |
| **On-Board Charger** | `[OBC]` | `0x786` | `0x78E` | AC Mains Rectification | `BCM: 1001` | `0x0110–0x2100` | 29 | `obc_uds_parameters_reference.md` |
| **Total Ecosystem** | — | — | — | **Unified EV Network** | — | — | **174** | — |

---

## 3. Engineering Unit Code (`UN`) Dictionary

The parameter definitions across all five subsystems standardize on discrete physical dimension codes:

| `UN` Identifier | Physical Dimension | Standard SI / Display Unit | Application Scope Across EV Subsystems |
|:---:|:---|:---:|:---|
| `0` | State / Enum / Bitmask / Count | — | Contactors, state machines, hardware status words, rolling heartbeat counters |
| `1` | Ratio / Percentage | $\%$ | State of Charge (SOC), State of Health (SOH), PWM duty cycles (CP, HVIL, Fans, Pumps) |
| `2` | Temperature | $^\circ\text{C}$ | Battery cells, inverter IGBTs, stator windings, coolant loops, transformer core |
| `3` | Electrical Potential | $\text{V}$ | Pack voltage, intermediate busbar, DC link rail, grid AC RMS, low-voltage 12V rails |
| `4` | Electrical Current | $\text{A}$ | Primary pack bidirectional current, inverter DC link draw, 3-phase AC stator currents |
| `5` | Linear Velocity | $\text{km/h}$ | CAN wheel vehicle speed setpoints and governor limits |
| `6` | Rotational Velocity | $\text{RPM}$ | Traction motor mechanical rotor velocity and electric compressor scroll speed |
| `7` | Pressure | $\text{kPa}$ | Hydraulic braking vacuum booster gauge pressure relative to atmosphere |
| `8` | Mechanical Torque | $\text{N}\cdot\text{m}$ | Motor shaft output torque, driver requested torque, recuperation drag ceilings |
| `9` | Frequency | $\text{Hz}$ | HVIL continuity carrier, CP pilot carrier ($1000\,\text{Hz}$), cooling pump feedback |
| `10` | Electrical Resistance | $\text{k}\Omega$ | High-voltage positive/negative rail isolation resistance to chassis ground |
| `11` | Active Power | $\text{kW}$ | Net mechanical traction power, auxiliary DC-DC load transfer targets |

---

## 4. Architectural Cross-Analysis: Direct vs. Relayed Telemetry

Querying internal microcontrollers directly versus querying the central VECU exposes fundamental differences in resolution, numerical sign representation, and hardware safety visibility:

```
               +----------------------------------------------------------------+
               |  DIRECT NODE (BMS: 0x785, MCU: 0x783, DCDC: 0x784, OBC: 0x786) |
               +-------------------------------+--------------------------------+
                                               |
                     Unsigned offset / Signed 2's Complement / Granular Steps
                     [Current: -3200A zero-point; SOH: 0.1%; Cell: 1mV; RPM: int16]
                                               |
                                               v
               +----------------------------------------------------------------+
               |           VECU RELAY LAYER (0x7E3 / 0x7EB Domain Aggregator)   |
               +-------------------------------+--------------------------------+
                                               |
                     Remapped positive offsets / Truncated byte resolutions
                     [Current: -600A zero-point; SOH: 1%; Cell: 10mV; RPM: W - 17000]
                                               v
                                   Standard Diagnostic Tools
```

### 4.1 Numerical Encodings & Offset Discrepancies

* **Traction Battery Current:**
  * **Direct BMS (`0x785`, DID `0x3401`):** Employs an ultra-wide industrial zero-point offset:
    $$I = \frac{W}{10} - 3200 \quad [\text{A}]$$
    A raw reading of $W = 32000$ corresponds to $0.0\,\text{A}$. This allows continuous sensing from $-200.0\,\text{A}$ to $+300.0\,\text{A}$ with substantial headroom.
  * **Relayed VECU (`0x7E3`, DID `0x341D`):** Re-encodes the measurement into a tighter vehicle offset:
    $$I = \frac{W}{10} - 600 \quad [\text{A}]$$
    Here, $W = 6000$ represents $0.0\,\text{A}$.
* **Motor Rotational Speed (RPM):**
  * **Direct MCU (`0x783`, DID `0x0510`):** Operates on standard **two's-complement big-endian signed 16-bit integers** (`int16_t`):
    $$\text{RPM} = \text{Signed}_{16}(W) \quad [\text{RPM}]$$
  * **Relayed VECU (`0x7E3`, DID `0x34A4`):** Translates signed rotation into strictly positive integers using a $+17000$ baseline offset:
    $$\text{RPM} = W - 17000 \quad [\text{RPM}]$$
* **Battery State of Health (SOH):**
  * **Direct BMS (`0x785`, DID `0x3403`):** Uses a **2-byte word** ($DL=2$) scaled by $0.1$ to provide high-precision fractional wear analysis ($60.0\%–100.0\%$).
  * **Relayed VECU (`0x7E3`, DID `0x3422`):** Truncates to an **8-bit integer** ($DL=1$), dropping fractional capacity tracking ($0\%–100\%$).
* **Cell Voltage Granularity:**
  * **Direct BMS (`0x785`, DIDs `0x3415`, `0x3417`):** Native millivolts directly from the Analog Front End (AFE) with $1\,\text{mV}$ resolution ($2500\,\text{mV}–3700\,\text{mV}$).
  * **Relayed VECU (`0x7E3`, DIDs `0x3477`, `0x3478`):** Scaled in Volts with $10\,\text{mV}$ resolution:
    $$V_{\text{cell}} = \frac{W}{100} \quad [\text{V}]$$

### 4.2 Traction Pack Electrochemical Topology

Cross-referencing telemetry parameters across the BMS catalog reveals the fundamental traction battery architecture:
* **Series Cell Count:** Maximum cell coordinate indices reach index $112$ (DID `0x3419` / `0x341A`), confirming a **$112\text{S}$ series cell topology**.
* **Operating Voltage Range:** Nominal pack operating voltage spans $270.0\,\text{V}$ to $410.0\,\text{V}$.
* **Per-Cell Chemistry Formulation:**
  $$V_{\text{cell, min}} = \frac{270.0\,\text{V}}{112} \approx 2.41\,\text{V}, \qquad V_{\text{cell, max}} = \frac{410.0\,\text{V}}{112} \approx 3.66\,\text{V}$$
  This operational window specifically corresponds to high-power **Lithium Iron Phosphate ($\text{LiFePO}_4$ / LFP)** or conservative **Lithium Nickel Manganese Cobalt ($\text{NMC}$)** industrial traction modules.

### 4.3 Advanced Safety & Hardware Registers (Exclusive Direct Access)

The direct diagnostic IDs provide lower-level failure classification not forwarded to the VECU:
1. **Early Thermal Runaway Detection:** BMS DID `0x3540` provides real-time particulate density from an internal optical laser aerosol sensor ($\mu\text{g}/\text{m}^3$), accompanied by discrete smoke ionization codes (`0x353A`) and runaway stages (`0x353B`).
2. **Dual-Sided Contactor Health & Welded Contact Diagnostics:** BMS DIDs `0x3528–0x352B` monitor voltage potential on both the battery-side and load-side terminals of the DC Fast Charge and Main Traction contactors to ensure complete galvano-isolation before closing.
3. **Firmware Integrity & Flashing Counters:** OBC DIDs `0x0200` and `0x0201` track successful reprogrammings versus initiation attempts, while DCDC DIDs `0xF0F1–0xF0F3` expose lower-level gate-driver and semiconductor lock flags.

---

## 5. Subsystem Reference Manuals

Detailed signal maps, bit definitions, scaling proofs, and unit test suites are available in the dedicated reference documents:

* [`vecu_uds_parameters_reference.md`](./vecu_uds_parameters_reference.md): Vehicle Electronic Control Unit parameter catalog (68 parameters).
* [`bms_uds_parameters_reference.md`](./bms_uds_parameters_reference.md): High-Voltage Battery Management System parameter catalog (42 parameters).
* [`mcu_uds_parameters_reference.md`](./mcu_uds_parameters_reference.md): Traction Inverter & Motor Control Unit parameter catalog (15 parameters).
* [`dcdc_uds_parameters_reference.md`](./dcdc_uds_parameters_reference.md): High-to-Low Voltage DC-DC Auxiliary Converter catalog (20 parameters).
* [`obc_uds_parameters_reference.md`](./obc_uds_parameters_reference.md): On-Board AC Grid Charger & Bidirectional V2L converter catalog (29 parameters).

---

## 6. Unified Python Master Diagnostic Decoder

The following production-ready Python engine (`unified_uds_decoder.py`) unifies decoding across all five ECUs. It features automatic routing based on CAN Arbitration ID or Subsystem Identifier, big-endian signed two's-complement arithmetic, and payload verification.

```python
"""
Unified UDS Service 0x22 Diagnostic Decoder Suite
Supports: VECU (0x7E3), BMS (0x785), MCU (0x783), DCDC (0x784), OBC (0x786)
"""

import struct
from typing import Dict, Any, List, Optional, Union


# --- SUBSYSTEM DEFINITIONS DATABASE ---

SYSTEM_REGISTRY: Dict[str, Dict[str, Any]] = {
    # -------------------------------------------------------------
    # 1. BMS (0x785 / 0x78D) - Battery Management System
    # -------------------------------------------------------------
    "BMS": {
        "REQ_ID": 0x785,
        "RESP_ID": 0x78D,
        "PARAMS": {
            "3400": {"name": "Pack Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "3401": {"name": "Pack Current", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": -3200.0, "unit": "A"},
            "3402": {"name": "State of Charge", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "%"},
            "3403": {"name": "State of Health", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "%"},
            "3404": {"name": "Main Pos Relay Status", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "bits"},
            "3405": {"name": "Initialization State", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "code"},
            "3406": {"name": "Heartbeat Counter", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "cnt"},
            "3409": {"name": "Max Cell Temperature", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "340B": {"name": "Min Cell Temperature", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "340D": {"name": "BMS Fault Rank", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "code"},
            "340E": {"name": "Operation Mode", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "code"},
            "340F": {"name": "Derate Flags", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "bits"},
            "3410": {"name": "Coolant Inlet Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "3411": {"name": "Coolant Outlet Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "3412": {"name": "Pack Average Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "3413": {"name": "Insulation Resistance", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "kΩ"},
            "3415": {"name": "Max Cell Voltage", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "mV"},
            "3417": {"name": "Min Cell Voltage", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "mV"},
            "3419": {"name": "Max Cell Voltage Index", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "#"},
            "341A": {"name": "Min Cell Voltage Index", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "#"},
            "3479": {"name": "Cell Balancing Status", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "bits"},
            "347B": {"name": "Max Charge Current Limit", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "A"},
            "347E": {"name": "Max Discharge Power Cap", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "kW"},
            "347F": {"name": "Max Regen Power Cap", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "kW"},
            "3484": {"name": "Internal Busbar Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "3492": {"name": "LV Terminal 30 Supply", "dl": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "34D5": {"name": "Cell Imbalance Delta", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "mV"},
            "3528": {"name": "FC Pos Relay Side Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "3529": {"name": "Main Pos Relay Side Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "352A": {"name": "FC Neg Relay Side Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "352B": {"name": "Main Neg Relay Side Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "353A": {"name": "Smoke Sensor State", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "code"},
            "353B": {"name": "Thermal Runaway Reason", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "code"},
            "3540": {"name": "Aerosol Concentration", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "µg/m³"},
            "3565": {"name": "HVIL Hardwire Status", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "code"},
            "3566": {"name": "SOC Accumulated Throughput", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "%"},
            "3568": {"name": "Calculated SOC Reference", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "%"}
        }
    },

    # -------------------------------------------------------------
    # 2. MCU (0x783 / 0x78B) - Traction Motor Control Unit
    # -------------------------------------------------------------
    "MCU": {
        "REQ_ID": 0x783,
        "RESP_ID": 0x78B,
        "PARAMS": {
            "0506": {"name": "Max Motoring Torque Limit", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "N·m"},
            "0507": {"name": "Max Regen Torque Limit", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "N·m"},
            "0508": {"name": "Max Forward Speed Limit", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "RPM"},
            "0509": {"name": "Reverse Speed Limit", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "RPM"},
            "0510": {"name": "Rotor Mechanical Speed", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "RPM"},
            "0511": {"name": "Delivered Shaft Torque", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "N·m"},
            "0512": {"name": "DC Link Rail Current", "dl": 2, "fmt": ">h", "scale": 0.5, "offset": 0.0, "unit": "A"},
            "0513": {"name": "DC Link Rail Voltage", "dl": 2, "fmt": ">H", "scale": 0.5, "offset": 0.0, "unit": "V"},
            "0514": {"name": "Inverter Operation Mode", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "0515": {"name": "Motor Stator Winding Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "0516": {"name": "Power Module (IGBT) Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "0517": {"name": "Delivered Drive Power", "dl": 2, "fmt": ">h", "scale": 1.0, "offset": 0.0, "unit": "kW"},
            "0518": {"name": "Phase A AC Current", "dl": 2, "fmt": ">h", "scale": 0.5, "offset": 0.0, "unit": "A"},
            "0519": {"name": "Phase B AC Current", "dl": 2, "fmt": ">h", "scale": 0.5, "offset": 0.0, "unit": "A"},
            "0520": {"name": "Phase C AC Current", "dl": 2, "fmt": ">h", "scale": 0.5, "offset": 0.0, "unit": "A"}
        }
    },

    # -------------------------------------------------------------
    # 3. DCDC (0x784 / 0x78C) - DC-DC Auxiliary Converter
    # -------------------------------------------------------------
    "DCDC": {
        "REQ_ID": 0x784,
        "RESP_ID": 0x78C,
        "PARAMS": {
            "130E": {"name": "Converter Working State", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "130F": {"name": "Enable Command Status", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "1310": {"name": "HV Input Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "1311": {"name": "HV Primary Draw Current", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "A"},
            "1312": {"name": "LV Regulated Output Voltage", "dl": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "1313": {"name": "LV Delivered Output Current", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "A"},
            "1314": {"name": "SR MOSFET Temperature", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "1315": {"name": "Transformer MT MOS Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "1316": {"name": "Primary Bridge MOS Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "1317": {"name": "Terminal 30 Logic Voltage", "dl": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "1318": {"name": "KL15 Ignition Sense Voltage", "dl": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "1325": {"name": "Thermal Derating Mode", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "1326": {"name": "Primary Error Flag", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "bits"},
            "1327": {"name": "Demanded Output Current", "dl": 2, "fmt": ">H", "scale": 0.01, "offset": 0.0, "unit": "A"},
            "1328": {"name": "Demanded Output Power", "dl": 2, "fmt": ">H", "scale": 0.001, "offset": 0.0, "unit": "kW"},
            "1329": {"name": "VCU Control CRC Status", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "F0F4": {"name": "Gateway Software Version", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "cnt"}
        }
    },

    # -------------------------------------------------------------
    # 4. OBC (0x786 / 0x78E) - On-Board Charger
    # -------------------------------------------------------------
    "OBC": {
        "REQ_ID": 0x786,
        "RESP_ID": 0x78E,
        "PARAMS": {
            "0110": {"name": "Manufacturing Mode", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "0200": {"name": "Reprogramming Success Count", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "cnt"},
            "0201": {"name": "Reprogramming Attempt Count", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "cnt"},
            "0210": {"name": "CP Line State", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "0211": {"name": "Charger State Machine", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "1100": {"name": "Mains Grid AC Frequency", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "Hz"},
            "1102": {"name": "Primary PFC Board Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "1103": {"name": "Secondary Rectifier Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "1104": {"name": "Isolation Transformer Temp", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "unit": "°C"},
            "1105": {"name": "Grid AC Phase RMS Current", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "A"},
            "1106": {"name": "Grid AC Phase RMS Voltage", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "V"},
            "1330": {"name": "Charge/Discharge Command", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "1331": {"name": "Discharge Gun Connect State", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"},
            "2000": {"name": "Control Pilot Peak Voltage", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "V"},
            "2001": {"name": "Control Pilot Duty Cycle", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "%"},
            "2002": {"name": "Control Pilot Oscillator Freq", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "unit": "Hz"},
            "2003": {"name": "HV DC Output Current", "dl": 2, "fmt": ">H", "scale": 0.01, "offset": 0.0, "unit": "A"},
            "2004": {"name": "HV DC Output Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "2100": {"name": "HW Wake-Up Output Pin", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "state"}
        }
    },

    # -------------------------------------------------------------
    # 5. VECU (0x7E3 / 0x7EB) - Vehicle Coordinator / Gateway
    # -------------------------------------------------------------
    "VECU": {
        "REQ_ID": 0x7E3,
        "RESP_ID": 0x7EB,
        "PARAMS": {
            "341D": {"name": "Relayed Pack Current", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": -600.0, "unit": "A"},
            "341E": {"name": "Relayed Pack Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "3421": {"name": "Relayed Battery SOC", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "%"},
            "3422": {"name": "Relayed Battery SOH", "dl": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "unit": "%"},
            "3462": {"name": "Vehicle Wheel Speed", "dl": 2, "fmt": ">H", "custom": lambda w: round(((w / 32.0) * 3.6), 2), "unit": "km/h"},
            "3468": {"name": "Brake Booster Vacuum", "dl": 1, "fmt": "B", "scale": 1.0, "offset": -120.0, "unit": "kPa"},
            "3477": {"name": "Relayed Max Cell Voltage", "dl": 2, "fmt": ">H", "scale": 0.01, "offset": 0.0, "unit": "V"},
            "3478": {"name": "Relayed Min Cell Voltage", "dl": 2, "fmt": ">H", "scale": 0.01, "offset": 0.0, "unit": "V"},
            "34A4": {"name": "Relayed Motor RPM", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": -17000.0, "unit": "RPM"},
            "34A5": {"name": "Relayed Motor Torque", "dl": 2, "fmt": ">H", "scale": 1.0, "offset": -510.0, "unit": "N·m"},
            "34A6": {"name": "Relayed Inverter Current", "dl": 2, "fmt": ">H", "custom": lambda w: round((w / 2.0) - 510.0, 2), "unit": "A"},
            "34A7": {"name": "Relayed Inverter Voltage", "dl": 2, "fmt": ">H", "scale": 0.5, "offset": 0.0, "unit": "V"},
            "3517": {"name": "Driver Requested Torque", "dl": 2, "fmt": ">H", "custom": lambda w: round((w / 4.0) - 2048.0, 2), "unit": "N·m"},
            "351F": {"name": "CCS Inlet Rail Voltage", "dl": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "unit": "V"},
            "3520": {"name": "CCS Current Demand", "dl": 2, "fmt": ">H", "custom": lambda w: round(w / 32.0, 2), "unit": "A"}
        }
    }
}


# --- DECODER CORE CLASS ---

class UnifiedEVDiagnosticDecoder:
    """
    Decodes ISO 14229 Service 0x22 UDS frames for EV powertrain components.
    """

    @staticmethod
    def _find_subsystem_by_can_id(can_id: int) -> Optional[str]:
        """Maps CAN response ID back to the owning subsystem."""
        for system_name, config in SYSTEM_REGISTRY.items():
            if config["RESP_ID"] == can_id or config["REQ_ID"] == can_id:
                return system_name
        return None

    @classmethod
    def decode_frame(
        cls,
        payload_bytes: bytes,
        can_id: Optional[int] = None,
        subsystem: Optional[str] = None
    ) -> Dict[str, Any]:
        """
        Parses a standard UDS Service 0x22 Positive Response payload:
        [0x62] [DID_HI] [DID_LO] [DATA_BYTES...]
        """
        # 1. Resolve Target Subsystem
        sys_key = subsystem.upper() if subsystem else (cls._find_subsystem_by_can_id(can_id) if can_id else None)
        if not sys_key or sys_key not in SYSTEM_REGISTRY:
            return {"error": f"Cannot identify subsystem from CAN ID {hex(can_id) if can_id else 'None'} or system key '{subsystem}'"}

        sys_data = SYSTEM_REGISTRY[sys_key]

        # 2. Validate Minimum Positive Response Length: 0x62 + 2-byte DID + Data
        if len(payload_bytes) < 3:
            return {"error": f"Payload length ({len(payload_bytes)}) is too short for valid UDS Service 0x22 response"}

        service_byte = payload_bytes[0]
        if service_byte != 0x62:
            if service_byte == 0x7F:
                nrc = payload_bytes[2] if len(payload_bytes) >= 3 else 0x00
                return {"error": f"Negative Response Received (Service 0x{payload_bytes[1]:02X}, NRC: 0x{nrc:02X})"}
            return {"error": f"Expected Service 0x62 positive response, received 0x{service_byte:02X}"}

        # 3. Extract DID and Parameter Metadata
        did_hex = f"{payload_bytes[1]:02X}{payload_bytes[2]:02X}"
        if did_hex not in sys_data["PARAMS"]:
            return {"error": f"DID 0x{did_hex} is not mapped in {sys_key} parameter registry"}

        param = sys_data["PARAMS"][did_hex]
        raw_data = payload_bytes[3:]
        dl = param["dl"]

        if len(raw_data) < dl:
            return {"error": f"Expected at least {dl} data bytes for DID 0x{did_hex}, received {len(raw_data)}"}

        # 4. Binary Unpacking and Mathematical Transformation
        raw_value = struct.unpack(param["fmt"], raw_data[:dl])[0]

        if "custom" in param:
            physical_value = param["custom"](raw_value)
        else:
            scale = param.get("scale", 1.0)
            offset = param.get("offset", 0.0)
            physical_value = round((raw_value * scale) + offset, 3)

        return {
            "subsystem": sys_key,
            "did": f"0x{did_hex}",
            "parameter": param["name"],
            "raw_payload_hex": raw_data[:dl].hex().upper(),
            "raw_int": raw_value,
            "physical_value": physical_value,
            "unit": param["unit"]
        }


# --- VERIFICATION TEST SUITE ---

if __name__ == "__main__":
    decoder = UnifiedEVDiagnosticDecoder()

    print("==========================================================================")
    print("           HIGH-VOLTAGE POWERTRAIN UNIFIED UDS TELEMETRY TEST            ")
    print("==========================================================================")

    # Test 1: Direct BMS (0x78D) - Pack Current Discharging +145.0 A
    # Formula: W/10 - 3200 = 145.0 => W = 33450 = 0x82AA
    frame_bms_i = bytes([0x62, 0x34, 0x01, 0x82, 0xAA])
    print(decoder.decode_frame(frame_bms_i, can_id=0x78D))

    # Test 2: Direct BMS (0x78D) - Optical Aerosol Laser Sensor (380 ug/m3)
    # Raw = 380 = 0x017C
    frame_bms_aerosol = bytes([0x62, 0x35, 0x40, 0x01, 0x7C])
    print(decoder.decode_frame(frame_bms_aerosol, subsystem="BMS"))

    # Test 3: Direct MCU (0x78B) - Motor Speed Reverse -1250 RPM (Signed 16-bit 2's Complement)
    # -1250 in int16_t = 0xFB1E
    frame_mcu_speed = bytes([0x62, 0x05, 0x10, 0xFB, 0x1E])
    print(decoder.decode_frame(frame_mcu_speed, can_id=0x78B))

    # Test 4: Direct DCDC (0x78C) - LV Auxiliary Output Voltage (14.2 V)
    # Raw = 142 = 0x8E
    frame_dcdc_v = bytes([0x62, 0x13, 0x12, 0x8E])
    print(decoder.decode_frame(frame_dcdc_v, can_id=0x78C))

    # Test 5: Direct OBC (0x78E) - Grid Input Voltage (230 V) & Frequency (50 Hz)
    frame_obc_vac = bytes([0x62, 0x11, 0x06, 0x00, 0xE6])
    print(decoder.decode_frame(frame_obc_vac, can_id=0x78E))

    # Test 6: Relayed VECU (0x7EB) - Vehicle Speed (72.0 km/h)
    # Speed = (W / 32) * 3.6 => 72.0 = W * 0.1125 => W = 640 = 0x0280
    frame_vecu_spd = bytes([0x62, 0x34, 0x62, 0x02, 0x80])
    print(decoder.decode_frame(frame_vecu_spd, can_id=0x7EB))

    # Test 7: Relayed VECU (0x7EB) - Relayed Pack Current (+145.0 A with -600A Offset)
    # Formula: W/10 - 600 = 145.0 => W = 7450 = 0x1D1A
    frame_vecu_i = bytes([0x62, 0x34, 0x1D, 0x1D, 0x1A])
    print(decoder.decode_frame(frame_vecu_i, can_id=0x7EB))
```

---

## 7. Implementation Guidelines & Automotive Standards

When developing diagnostic loggers, automated Hardware-in-the-Loop (HIL) benches, or end-of-line test routines against this architecture, observe the following automotive integration rules:

1. **Multi-Frame Reassembly (ISO 15765-2 / DoCAN):**
   * Service `$22` responses exceeding 7 data bytes (e.g., cell arrays or concatenated DIDs) require an ISO-TP transport layer implementation with First Frame (`0x10`), Flow Control (`0x30`), and Consecutive Frames (`0x21...`).
2. **Diagnostic Session Control (Service `$10`):**
   * Routine readouts for basic telemetry are permitted in the Default Session (`$01`).
   * Actuator overrides, Contactor Force-Opening/Closing commands, and flash attempt registers frequently require transitioning to Extended Diagnostic Session (`$03`) or Programming Session (`$02`).
3. **Cyclic Polling & Bus Load Budgeting:**
   * High-priority dynamic parameters (such as `Batt Volt`, `Batt Curr`, `Motor Speed`, `Alive cnt`) have `SkipCycles = 0` and can be queried cyclically at $50\,\text{ms}–100\,\text{ms}$ intervals.
   * Thermal, SOH, and calibration flags (`SkipCycles = 5–10`) must be queried at lowered frequencies ($500\,\text{ms}–2000\,\text{ms}$) to prevent CAN bus saturation on the $500\,\text{kbps}$ powertrain link.
4. **Negative Response Code (NRC) Handling:**
   * `0x11` (*ServiceNotSupported*): Subsystem does not support Service `$22` in current power mode.
   * `0x22` (*ConditionsNotCorrect*): Contactor taps or isolation tests cannot execute while driving.
   * `0x31` (*RequestOutOfRange*): DID not supported by that specific firmware build.
   * `0x33` (*SecurityAccessDenied*): Calibration/flashing parameters require Service `$27` cryptographic unlock.