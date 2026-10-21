# Battery Management System (BMS) UDS Diagnostic Parameter Identifier (DID) Reference Manual

This technical specification details the diagnostic parameters, communication headers, data lengths, scaling formulas, cell topology metrics, contactor diagnostic registers, and engineering units extracted from the High-Voltage Battery Management System (`[BMS]`) telemetry configuration.

---

## 1. Protocol Architecture & Header Specifications

* **Diagnostic Service:** Unified Diagnostic Services (ISO 14229) — Service `$22$ (`ReadDataByIdentifier`).
* **CAN Request Header (CAN-ID):** `0x785` (Standard 11-bit Physical Addressing for the High-Voltage BMS Node).
* **Expected Response Header:** `0x78D` (Standard 11-bit Slave Response via $\text{Request ID} + \text{0x08}$).
* **Endianness:** Big-Endian (Most Significant Byte first: Byte $A$ followed by Byte $B$).
* **Bus / Cluster Domain:** Powertrain / EV Subsystem CAN (`BCM: 1003`).
* **Pack Topology Insight:** Maximum cell positions reach index $112$ (DIDs `0x3419` / `0x341A`), confirming a **$112\text{S}$ series battery topology**. Operating across $270.0\,\text{V}$ to $410.0\,\text{V}$ ($2.41\,\text{V}$ to $3.66\,\text{V}$ per cell), this corresponds to an industrial Lithium Iron Phosphate ($\text{LFP}$) or Lithium Nickel Manganese Cobalt ($\text{NMC}$) high-voltage traction pack.

---

## 2. Engineering Unit & Dimension Dictionary

While raw configuration files frequently label dimensions as discrete unit code `0`, physical interpretation of parameter identifiers reveals the following standard physical dimensions:

| Dimension Code / Context | Physical Dimension | Standard Unit | Notes / Typical BMS Subsystem Context |
|:---:|:---|:---:|:---|
| `State / Code` | Discrete Enumeration | — | Operating states, init phases, mode requests, runaway codes |
| `Bitfield` | Bitmask Status | `bits` | Relay states, fault masks, derate triggers, cell balancing flags |
| `Count / Pos` | Discrete Integer | `cnt` / `#` | Rolling alive counters, fault counters, series cell indices ($1–112$) |
| `Voltage (HV)` | High-Voltage Potential | $\text{V}$ | Pack total voltage, busbar, contactor pre/post relay taps |
| `Voltage (Cell)`| Electrochemical Cell Potential | $\text{mV}$ | Individual cell peak, trough, and delta voltages ($1\,\text{mV}$ resolution) |
| `Voltage (LV)` | Auxiliary Supply Potential | $\text{V}$ | Low-voltage Terminal 30 logic board power supply ($0.1\,\text{V}$ step) |
| `Current` | Electrical Current | $\text{A}$ | Primary pack draw/charge, charge limits (with broad bidirectional offset) |
| `Power` | Active Power | $\text{kW}$ | Maximum dynamic discharge power and recuperation ceilings |
| `Temperature` | Thermal Monitoring | $^\circ\text{C}$ | Cell peak/trough, coolant inlet/outlet, pack weighted average |
| `Ratio` | State of Charge / Health | $\%$ | High-precision $0.1\%$ SOC/SOH and accumulation registers |
| `Resistance` | Isolation Integrity | $\text{k}\Omega$ | High-voltage positive/negative rail isolation to chassis ground |
| `Aerosol` | Particulate Concentration | $\mu\text{g}/\text{m}^3$ | Optical laser smoke / cell venting detection before thermal runaway |

---

## 3. Comprehensive BMS Parameter Catalog

All parameters are retrieved via Service `$22` across DID groups `0x3400–0x341A`, `0x3479–0x3493`, `0x34D5`, and `0x3528–0x3568`.

### 3.1 Pack Energy, State of Charge (SOC) & State of Health (SOH)

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|
| `223400` | `[BMS] Batt Volt (V)` | 2 | `((A*256)+B)/10` | $\text{V}$ | 270.0 | 410.0 | 0 | Total pack instantaneous high-voltage potential |
| `223401` | `[BMS] Batt Curr (A)` | 2 | `(((A*256)+B)/10)-3200` | $\text{A}$ | -200.0 | 300.0 | 0 | Pack net current ($- = \text{Charge}, + = \text{Discharge}$) |
| `223402` | `[BMS] SOC (%)` | 2 | `((A*256)+B)/10` | $\%$ | 0.0 | 100.0 | 2 | Primary active pack State of Charge |
| `223403` | `[BMS] SOH (%)` | 2 | `((A*256)+B)/10` | $\%$ | 60.0 | 100.0 | 10 | Estimated battery health capacity retention factor |
| `223568` | `[BMS] Calc SOC (%)` | 2 | `((A*256)+B)/10` | $\%$ | 0.0 | 100.0 | 3 | Coulomb-counting secondary reference SOC |
| `223566` | `[BMS] SOC Accum (%)` | 2 | `((A*256)+B)/10` | $\%$ | 0.0 | 500.0 | 10 | Throughput accumulation since last $100\%$ saturation |
| `22353D` | `[BMS] SOCCal [code]` | 1 | `A` | Code | 0 | 15 | 10 | OCV resting curve calibration status flag |

### 3.2 Cell Voltages, Balancing & Topology

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|
| `223415` | `[BMS] Max Cell (mV)` | 2 | `(A*256)+B` | $\text{mV}$ | 2500.0 | 3700.0 | 2 | Highest single cell voltage recorded in pack |
| `223417` | `[BMS] Min Cell (mV)` | 2 | `(A*256)+B` | $\text{mV}$ | 2500.0 | 3700.0 | 2 | Lowest single cell voltage recorded in pack |
| `2234D5` | `[BMS] Cell Diff (mV)` | 2 | `(A*256)+B` | $\text{mV}$ | 0.0 | 150.0 | 2 | Cell imbalance spread ($V_{\max} - V_{\min}$) |
| `223419` | `[BMS] Max Cell (#)` | 1 | `A` | Pos | 1 | 112 | 5 | Physical series index ($1–112$) of highest cell |
| `22341A` | `[BMS] Min Cell (#)` | 1 | `A` | Pos | 1 | 112 | 5 | Physical series index ($1–112$) of lowest cell |
| `223479` | `[BMS] CellBal [bits]` | 1 | `A` | Bitfield | 0 | 255 | 5 | Passive bleed balancing active channel bitfield |

### 3.3 Thermal Monitoring & Liquid Cooling

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|
| `223409` | `[BMS] Max Temp (°C)` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 70.0 | 5 | Highest cell thermistor temperature |
| `22340B` | `[BMS] Min Temp (°C)` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 70.0 | 5 | Lowest cell thermistor temperature |
| `223412` | `[BMS] Avg Temp (°C)` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 70.0 | 5 | Pack volume-weighted mean temperature |
| `223410` | `[BMS] Cool In (°C)` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 70.0 | 5 | Liquid cold plate inlet coolant temperature |
| `223411` | `[BMS] Cool Out (°C)` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 70.0 | 5 | Liquid cold plate outlet coolant temperature |

### 3.4 Power & Current Limits (Dynamic Derating)

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|
| `22347B` | `[BMS] Max Chg (A)` | 2 | `((A*256)+B)/10` | $\text{A}$ | 0.0 | 160.0 | 1 | Maximum continuous charging current permitted |
| `22347E` | `[BMS] Max Dsch (kW)` | 2 | `((A*256)+B)/10` | $\text{kW}$ | 0.0 | 100.0 | 1 | Dynamic peak discharge power envelope |
| `22347F` | `[BMS] Max Regn (kW)` | 2 | `((A*256)+B)/10` | $\text{kW}$ | 0.0 | 50.0 | 1 | Instantaneous recuperative regenerative brake cap |
| `22340F` | `[BMS] Derate [bits]` | 1 | `A` | Bitfield | 0 | 255 | 3 | Derating root causes (Over-temp, Low SOC, etc.) |

### 3.5 High-Voltage Relays, Busbar & Diagnostic Potentials

The BMS performs dual-sided voltage sense measurements across positive and negative contactor poles to detect welded contacts or failed pre-charge sequences:

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|
| `223484` | `[BMS] Busbar (V)` | 2 | `((A*256)+B)/10` | $\text{V}$ | 0.0 | 410.0 | 0 | Internal intermediate link busbar potential |
| `223404` | `[BMS] PosRel [bits]` | 1 | `A` | Bitfield | 0 | 255 | 3 | Primary positive main contactor feedback register |
| `223528` | `[BMS] FC Pos Rel (V)` | 2 | `((A*256)+B)/10` | $\text{V}$ | 0.0 | 410.0 | 3 | DC Fast Charging Positive Contactor load terminal |
| `223529` | `[BMS] Mn Pos Rel (V)` | 2 | `((A*256)+B)/10` | $\text{V}$ | 0.0 | 410.0 | 3 | Main Traction Positive Contactor load terminal |
| `22352A` | `[BMS] FC Neg Rel (V)` | 2 | `((A*256)+B)/10` | $\text{V}$ | 0.0 | 410.0 | 3 | DC Fast Charging Negative Contactor load terminal |
| `22352B` | `[BMS] Mn Neg Rel (V)` | 2 | `((A*256)+B)/10` | $\text{V}$ | 0.0 | 410.0 | 3 | Main Traction Negative Contactor load terminal |

### 3.6 Isolation Monitoring, Safety Interlocks & Thermal Runaway Protection

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---|
| `223413` | `[BMS] Insul Res (kΩ)`| 2 | `(A*256)+B` | $\text{k}\Omega$ | 0.0 | 10000.0 | 5 | Pack isolation resistance to chassis ground |
| `223414` | `[BMS] InsulEn [code]` | 1 | `A` | Code | 0 | 3 | 10 | On-board insulation measurement injection enable |
| `223565` | `[BMS] HVIL [code]` | 1 | `A` | Code | 0 | 3 | 3 | High Voltage Interlock Loop hardwired continuity |
| `223540` | `[BMS] Aerosol (µg/m³)`| 2 | `(A*256)+B` | $\mu\text{g}/\text{m}^3$ | 0.0 | 2000.0 | 5 | Optical laser aerosol chamber density |
| `22353A` | `[BMS] Smoke [code]` | 1 | `A` | Code | 0 | 15 | 5 | Solid-state ionization smoke detection logic |
| `22353B` | `[BMS] ThRun [code]` | 1 | `A` | Code | 0 | 15 | 5 | Thermal runaway classification alert level |

### 3.7 BMS Operating States, Diagnostics & Health Monitoring

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Skip Cycles | Detailed Description |
|:---|:---|:---:|:---|:---:|:---|:---:|:---:|:---|
| `223405` | `[BMS] InitSt [code]` | 1 | `A` | Code | 0 | 15 | 10 | Bootloader & AFE initialization state machine |
| `223406` | `[BMS] Alive (cnt)` | 1 | `A` | Count | 0 | 255 | 0 | Rolling cyclic sanity alive counter |
| `22340D` | `[BMS] FLTRank [code]` | 1 | `A` | Code | 0 | 5 | 3 | Battery system fault severity grade ($0 = \text{Normal}$) |
| `22340E` | `[BMS] OpMode [code]` | 1 | `A` | Code | 0 | 15 | 3 | Active operational profile (Standby, Drive, Charge) |
| `223483` | `[BMS] ModeReq [code]` | 1 | `A` | Code | 0 | 15 | 3 | Requested target operational mode |
| `223492` | `[BMS] LV Supply (V)` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | 3 | Low-voltage Terminal 30 logic board potential |
| `223493` | `[BMS] Flag [bits]` | 1 | `A` | Bitfield | 0 | 255 | 3 | General internal hardware status flags |
| `22353C` | `[BMS] Flt Cnt (cnt)` | 1 | `A` | Count | 0 | 255 | 5 | Diagnostic trouble code rolling event counter |

---

## 4. Architectural Comparison: BMS Direct (`0x785`) vs. VECU Relayed (`0x7E3`)

The following matrix compares internal engineering telemetry acquired directly from the BMS node against synthesized vehicle parameters exposed by the central Vehicle Electronic Control Unit:

| Metric Parameter | Direct BMS DID (`0x785`) | Direct Scaling Formula | VECU Relayed DID (`0x7E3`) | VECU Scaling Formula | Architectural & Precision Differences |
|:---|:---:|:---|:---:|:---|:---|
| **Pack Current** | `0x3401` | $\frac{W}{10} - 3200\;[\text{A}]$ | `0x341D` | $\frac{W}{10} - 600\;[\text{A}]$ | Direct BMS uses a wide $-3200\,\text{A}$ zero-point offset ($W=32000 \implies 0\,\text{A}$); VECU remaps to a narrower $-600\,\text{A}$ offset. |
| **Pack Voltage** | `0x3400` | $\frac{W}{10}\;[\text{V}]$ | `0x341E` | $\frac{W}{10}\;[\text{V}]$ | Identical $100\,\text{mV}$ resolution across both diagnostic nodes. |
| **State of Health (SOH)** | `0x3403` | $\frac{W}{10}\;[\%]\;(DL=2)$ | `0x3422` | $A\;[\%]\;(DL=1)$ | Direct BMS provides 16-bit word with $0.1\%$ resolution; VECU drops LSB to an 8-bit integer ($1\%$). |
| **Max Cell Voltage** | `0x3415` | $W\;[\text{mV}]$ | `0x3477` | $\frac{W}{100}\;[\text{V}]$ | Direct BMS exposes millivolts directly ($1\,\text{mV}$ step); VECU scales in Volts ($10\,\text{mV}$ step). |
| **Min Cell Voltage** | `0x3417` | $W\;[\text{mV}]$ | `0x3478` | $\frac{W}{100}\;[\text{V}]$ | Direct BMS exposes millivolts directly ($1\,\text{mV}$ step); VECU scales in Volts ($10\,\text{mV}$ step). |
| **Cell Temperature** | `0x3409` | $A - 40\;[^\circ\text{C}]$ | `0x3423` | $A - 50\;[^\circ\text{C}]$ | Direct BMS uses standard automotive $-40^\circ\text{C}$ offset; VECU applies custom $-50^\circ\text{C}$ offset. |
| **Cell Position Taps** | `0x3419–1A`| $A\;(1–112)$ | *Not relayed* | — | Individual physical cell coordinates are exposed exclusively on `0x785`. |
| **Cell Imbalance Spread**| `0x34D5` | $W\;[\text{mV}]$ | *Not relayed* | — | Granular cell voltage spread ($\Delta V$) calculated directly on `0x785`. |
| **Aerosol & Thermal Runaway**| `0x3540, 353A-B` | Aerosol $\mu\text{g}/\text{m}^3$ & codes | `0x34C5` | Binary flag | Direct BMS provides laser particulate chamber concentration and detailed fault codes; VECU exposes a generic warning light. |
| **Relay Contact Potentials**| `0x3528–2B`| 4 individual relay taps | `0x3487` | Readback status | Direct BMS tracks pre- and post-contactor voltage drops to detect contact welding; VECU abstracts to a 2-bit state. |

---

## 5. Mathematical Formulations & Scaling Proofs

### 5.1 Single-Byte Conversions ($DL = 1$)

For an incoming byte payload $A \in [0, 255]$:

* **Low-Voltage Board Power Supply ($0.1\,\text{V}$ step):**
  $$V_{\text{LV}} = \frac{A}{10} \quad [\text{V}]$$
  $$\text{Example: } A = \text{0x87}\,(135) \implies V_{\text{LV}} = \frac{135}{10} = 13.5\,\text{V}$$

* **Industrial Cell & Coolant Temperature ($-40^\circ\text{C}$ offset):**
  $$T = A - 40 \quad [^\circ\text{C}]$$
  $$\text{Example: } A = \text{0x41}\,(65) \implies T = 65 - 40 = 25^\circ\text{C}$$

* **Cell Series Coordinates & Discrete Codes:**
  $$\text{Index} = A \quad (\text{Valid Cell Positions: } 1 \le A \le 112)$$

### 5.2 Double-Byte Conversions ($DL = 2$)

Given raw payload bytes $A$ (MSB) and $B$ (LSB), the raw unsigned 16-bit word is:
$$W = (A \times 256) + B$$

* **High-Voltage Pack Current ($-3200\,\text{A}$ zero-offset):**
  $$I_{\text{pack}} = \frac{W}{10} - 3200 \quad [\text{A}]$$
  $$\text{Discharging Proof (e.g., }+150.0\,\text{A}\text{): } W = (150.0 + 3200) \times 10 = 33500\,(\text{0x82DC})$$
  $$I = \frac{33500}{10} - 3200 = 3350.0 - 3200 = +150.0\,\text{A}$$
  $$\text{Charging Proof (e.g., }-80.0\,\text{A}\text{): } W = (-80.0 + 3200) \times 10 = 31200\,(\text{0x79E0})$$
  $$I = \frac{31200}{10} - 3200 = 3120.0 - 3200 = -80.0\,\text{A}$$

* **Pack, Busbar & Contactor Voltage Potentials ($0.1\,\text{V}$ step):**
  $$V_{\text{HV}} = \frac{W}{10} \quad [\text{V}]$$
  $$\text{Example: } W = \text{0x0DF4}\,(3572) \implies V_{\text{HV}} = \frac{3572}{10} = 357.2\,\text{V}$$

* **Dynamic Power Thresholds ($0.1\,\text{kW}$ step):**
  $$P_{\text{lim}} = \frac{W}{10} \quad [\text{kW}]$$
  $$\text{Example: } W = \text{0x0384}\,(900) \implies P_{\text{disch}} = \frac{900}{10} = 90.0\,\text{kW}$$

* **Direct Millivolt Cell Measurements ($1\,\text{mV}$ step):**
  $$V_{\text{cell}} = W \quad [\text{mV}]$$
  $$\text{Example: } W = \text{0x0CDA}\,(3290) \implies V_{\text{cell}} = 3290\,\text{mV} = 3.290\,\text{V}$$

* **Aerosol Particulate Chamber Concentration ($1\,\mu\text{g}/\text{m}^3$ step):**
  $$C_{\text{aerosol}} = W \quad [\mu\text{g}/\text{m}^3]$$
  $$\text{Example: } W = \text{0x01B8}\,(440) \implies C_{\text{aerosol}} = 440\,\mu\text{g}/\text{m}^3$$

---

## 6. Python BMS Payload Decoder

The following standalone script handles all 42 telemetry parameters extracted from the `0x785` BMS dataset:

```python
import struct
from typing import Dict, Any, List, Optional

BMS_DEFINITIONS: List[Dict[str, Any]] = [
    {"CMD": "223400", "NM": "Battery Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "223401", "NM": "Battery Current", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": -3200.0, "UN": "A"},
    {"CMD": "223402", "NM": "State of Charge (SOC)", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "%"},
    {"CMD": "223403", "NM": "State of Health (SOH)", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "%"},
    {"CMD": "223404", "NM": "Main Positive Relay Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "223405", "NM": "Initialization State", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "223406", "NM": "Heartbeat / Alive Counter", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "cnt"},
    {"CMD": "223409", "NM": "Max Present Cell Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "22340B", "NM": "Min Present Cell Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "22340D", "NM": "BMS Fault Rank", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "22340E", "NM": "Operation Mode", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "22340F", "NM": "Derate Flag", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "223410", "NM": "Coolant Inlet Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "223411", "NM": "Coolant Outlet Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "223412", "NM": "Battery Average Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "223413", "NM": "Insulation Resistance", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "kΩ"},
    {"CMD": "223414", "NM": "Insulation Measurement Command", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "223415", "NM": "Max Cell Voltage", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "mV"},
    {"CMD": "223417", "NM": "Min Cell Voltage", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "mV"},
    {"CMD": "223419", "NM": "Max Cell Voltage Position", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "#"},
    {"CMD": "22341A", "NM": "Min Cell Voltage Position", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "#"},
    {"CMD": "223479", "NM": "Cell Balance Status Flag", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "22347B", "NM": "Max Allowed Charging Current", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "A"},
    {"CMD": "22347E", "NM": "Max Allowed Discharge Power", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "kW"},
    {"CMD": "22347F", "NM": "Max Allowed Regen Power", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "kW"},
    {"CMD": "223483", "NM": "BMS Mode Request", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "223484", "NM": "Busbar Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "223492", "NM": "LV Power Supply Voltage", "DL": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "223493", "NM": "BMS Status Flag", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "2234D5", "NM": "Cell Voltage Difference", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "mV"},
    {"CMD": "223528", "NM": "Fast Charging Positive Relay Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "223529", "NM": "Main Positive Relay Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "22352A", "NM": "Fast Charging Negative Relay Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "22352B", "NM": "Main Negative Relay Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "22353A", "NM": "Smoke Sensor Logic", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "22353B", "NM": "Thermal Runaway Reason", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "22353C", "NM": "Fault Rolling Counter", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "cnt"},
    {"CMD": "22353D", "NM": "SOC Calibration Flag", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "223540", "NM": "Aerosol Concentration", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "µg/m³"},
    {"CMD": "223565", "NM": "HVIL Hardwire Signal", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Code"},
    {"CMD": "223566", "NM": "SOC Accumulation Since 100% Charge", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "%"},
    {"CMD": "223568", "NM": "Calculated SOC Reference", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "%"},
]

def decode_bms_response(did_hex: str, raw_payload: bytes) -> Dict[str, Any]:
    """
    Decodes a BMS UDS Service 0x22 response payload.
    raw_payload: Byte sequence directly following the 3-byte positive response header (0x62 DID_HI DID_LO).
    """
    entry: Optional[Dict[str, Any]] = next((item for item in BMS_DEFINITIONS if item["CMD"] == did_hex), None)
    if not entry:
        return {"error": f"DID {did_hex} not found in BMS catalog"}

    fmt = entry["fmt"]
    scale = entry["scale"]
    offset = entry["offset"]
    dl = entry["DL"]

    if len(raw_payload) < dl:
        return {"error": f"Payload length {len(raw_payload)} is less than expected {dl} bytes"}

    raw_val = struct.unpack(fmt, raw_payload[:dl])[0]
    physical_val = round((raw_val * scale) + offset, 3)

    return {
        "did": entry["CMD"],
        "name": entry["NM"],
        "raw_value": raw_val,
        "physical_value": physical_val,
        "unit": entry["UN"]
    }

# --- Verification & Unit Test Vectors ---
if __name__ == "__main__":
    # Test 1: Total Pack Voltage (DID 0x3400) -> 356.4 V (Raw = 3564 -> 0x0DEC)
    res_v = decode_bms_response("223400", bytes([0x0D, 0xEC]))
    print(res_v)  # Expected: 356.4 V

    # Test 2: Battery Current Discharging +120.0 A (DID 0x3401)
    # Raw = (120.0 + 3200) * 10 = 33200 -> 0x81B0
    res_i_disch = decode_bms_response("223401", bytes([0x81, 0xB0]))
    print(res_i_disch)  # Expected: +120.0 A

    # Test 3: Battery Current Charging -65.0 A (DID 0x3401)
    # Raw = (-65.0 + 3200) * 10 = 31350 -> 0x7A76
    res_i_chg = decode_bms_response("223401", bytes([0x7A, 0x76]))
    print(res_i_chg)  # Expected: -65.0 A

    # Test 4: Max Cell Voltage (DID 0x3415) -> 3350 mV (Raw = 3350 -> 0x0D16)
    res_cell = decode_bms_response("223415", bytes([0x0D, 0x16]))
    print(res_cell)  # Expected: 3350 mV

    # Test 5: Max Cell Position (DID 0x3419) -> Position 84 (Raw = 84 -> 0x54)
    res_pos = decode_bms_response("223419", bytes([0x54]))
    print(res_pos)  # Expected: 84

    # Test 6: Aerosol Concentration (DID 0x3540) -> 450 µg/m³ (Raw = 450 -> 0x01C2)
    res_aerosol = decode_bms_response("223540", bytes([0x01, 0xC2]))
    print(res_aerosol)  # Expected: 450 µg/m³

    # Test 7: Max Cell Temp (DID 0x3409) -> 32 °C (Raw = 72 -> 0x48)
    res_temp = decode_bms_response("223409", bytes([0x48]))
    print(res_temp)  # Expected: 32 °C
```