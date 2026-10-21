# On-Board Charger (OBC) UDS Diagnostic Parameter Identifier (DID) Reference Manual

This technical specification details the diagnostic parameters, communication headers, data lengths, scaling formulas, operational state machines, memory/flashing registers, and engineering units extracted from the high-voltage On-Board Charger (`[OBC]`) telemetry configuration.

---

## 1. Protocol Architecture & Header Specifications

* **Diagnostic Service:** Unified Diagnostic Services (ISO 14229) — Service `$22` (`ReadDataByIdentifier`).
* **CAN Request Header (CAN-ID):** `0x786` (Standard 11-bit Physical Request Addressing for the On-Board Charger Subsystem).
* **Expected Response Header:** `0x78E` (Standard 11-bit Slave Response via $\text{Request ID} + \text{0x08}$).
* **Endianness:** Big-Endian (Most Significant Byte first: Byte $A$ followed by Byte $B$).
* **Bus / Cluster Domain:** Powertrain / EV Subsystem CAN (`BCM: 1001`).

---

## 2. Engineering Unit Code Dictionary

The OBC telemetry configuration employs six core physical unit dimension identifiers:

| `UN` Code | Physical Dimension | Standard Unit | Notes / Typical Subsystem Context |
|:---:|:---|:---:|:---|
| `0` | State / Discrete / Bitmask / Count | — | Charger modes, flashing attempt counters, bus statuses, fault flags |
| `1` | Ratio / Percentage | `%` | Control Pilot (CP) PWM duty cycle |
| `2` | Temperature | `$^\circ\text{C}$` | Multi-zone thermal sensors (Primary Board, Secondary Board, Transformer) |
| `3` | Electrical Potential | `$\text{V}$` | Grid AC RMS voltage, HV DC output voltage, 12V supply, CP rail voltage |
| `4` | Electrical Current | `$\text{A}$` | Grid AC RMS phase draw & rectified HV DC battery charging current |
| `9` | Frequency | `$\text{Hz}$` | Grid utility frequency ($50/60\,\text{Hz}$) & Control Pilot carrier ($1\,\text{kHz}$) |

---

## 3. OBC Parameter Catalog

All parameters are retrieved via Service `$22` across DID groups `0x0110–0x0219`, `0x1000–0x1106`, `0x1330–0x1332`, and `0x2000–0x2100`.

### 3.1 Operating State, Flashing & Hardware Diagnostics

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `220110` | `Mfg Mode` | 1 | `A` | State | 0 | 3 | End-of-line (EOL) production / manufacturing test mode |
| `220200` | `Reprog Count` | 2 | `A*256+B` | Count | 0 | 50 | Total count of successfully completed firmware flash events |
| `220201` | `Reprog Attempt` | 2 | `A*256+B` | Count | 0 | 50 | Total flash reprogramming attempts initiated |
| `220211` | `Charger State` | 1 | `A` | State | 0 | 15 | Primary charging operational state machine step |
| `220212` | `Int CAN Stat` | 1 | `A` | State | 0 | 3 | Internal sub-bus communication health status |
| `220213` | `Ext CAN Stat` | 1 | `A` | State | 0 | 3 | External powertrain CAN bus communication health status |
| `220214` | `Ext CAN Trx` | 1 | `A` | State | 0 | 3 | Physical layer CAN transceiver circuit status |
| `220215` | `FEE Status` | 1 | `A` | State | 0 | 3 | Flash Emulated EEPROM virtual driver status |
| `220216` | `EEPROM Error` | 1 | `A` | State | 0 | 1 | Non-volatile memory read/write hardware failure flag |
| `220217` | `SCI Comm Fail` | 1 | `A` | State | 0 | 1 | Internal Serial Comm Interface (DSP $\leftrightarrow$ MCU link) error |
| `220218` | `SCI CRC Fail` | 1 | `A` | State | 0 | 1 | Internal SCI packet cyclic redundancy check fault |
| `220219` | `OBD Sensors` | 1 | `A` | Bitfield | 0 | 255 | On-board sensor readiness and hardware sanity bitfield |
| `221000` | `12V Supply V` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | Low-voltage logic board Terminal 30 supply potential |
| `222100` | `HW Wake State` | 1 | `A` | State | 0 | 1 | Hardwired high-side/low-side wake-up line output state |

### 3.2 AC Mains Input & Grid Monitoring

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `221100` | `AC Frequency` | 2 | `A*256+B` | $\text{Hz}$ | 0.0 | 65.0 | Grid utility frequency ($50\,\text{Hz} / 60\,\text{Hz}$ nominal) |
| `221105` | `AC In Current` | 2 | `(A*256+B)/10` | $\text{A}$ | 0.0 | 36.0 | Single-phase AC mains RMS input line current |
| `221106` | `AC In Voltage` | 2 | `A*256+B` | $\text{V}$ | 0.0 | 275.0 | Single-phase AC mains RMS grid input voltage |

### 3.3 High-Voltage DC Output (Rectification Stage)

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `222003` | `DC Out Curr` | 2 | `(A*256+B)/100` | $\text{A}$ | 0.0 | 25.0 | Delivered high-voltage DC battery charging current |
| `222004` | `DC Out Volt` | 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | High-voltage DC output potential applied to traction rail |

### 3.4 Control Pilot (CP) Interface (IEC 61851-1 / SAE J1772)

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `220210` | `CP Line Stat` | 1 | `A` | State | 0 | 7 | Analog CP state classification ($+12\text{V}, +9\text{V}, +6\text{V}$, Error) |
| `222000` | `CP Voltage` | 1 | `A` | $\text{V}$ | 0.0 | 14.0 | Measured peak analog Control Pilot rail voltage |
| `222001` | `CP Duty Cycle` | 1 | `A` | `%` | 0.0 | 100.0 | Decoded EVSE maximum allowable current PWM duty ratio |
| `222002` | `CP Frequency` | 2 | `A*256+B` | $\text{Hz}$ | 800.0 | 1200.0 | Observed CP oscillator carrier frequency ($1000\,\text{Hz}$ nominal) |

### 3.5 Power Stage Thermal Monitoring

The OBC incorporates internal negative temperature coefficient (NTC) thermistors across critical switching and magnetic elements:

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `221102` | `Pri Brd Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 115.0 | Primary PFC boost stage PCB / heatsink temperature |
| `221103` | `Sec Brd Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 115.0 | Secondary DC-DC synchronous rectification PCB temp |
| `221104` | `Transfmr Temp`| 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 120.0 | High-frequency isolation power transformer core temp |

### 3.6 Bidirectional, Discharge & V2X / V2L Operations

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `221330` | `Chg/Disch Cmd` | 1 | `A` | State | 0 | 3 | VCU operational command (`0 = Standby, 1 = Charge, 2 = Discharge / V2L`) |
| `221331` | `Disch Gun Stat`| 1 | `A` | State | 0 | 3 | Vehicle-to-Load discharge adapter plug latch/connection status |
| `221332` | `Operation Mode`| 1 | `A` | State | 0 | 7 | Converter operational mode (e.g., PFC active, Inverter active, Derated) |

---

## 4. Architectural Comparison: OBC Direct (`0x786`) vs. VECU Relayed (`0x7E3`)

The OBC telemetry can be queried directly at its physical node (`0x786`) or via relayed aggregates synthesized by the Vehicle Electronic Control Unit (`0x7E3`):

| Telemetry Parameter | OBC Direct DID (`0x786`) | Direct Scaling Formula | VECU Relayed DID (`0x7E3`) | VECU Scaling Formula | Architectural & Precision Differences |
|:---|:---:|:---|:---:|:---|:---|
| **AC Input Voltage** | `0x1106` | $W\;[\text{V}]$ | `0x34EC` | $\frac{W}{50}\;[\text{V}]$ | Direct OBC uses integer $1\,\text{V}$ steps; VECU relay re-encodes to $20\,\text{mV}$ steps. |
| **AC Input Current** | `0x1105` | $\frac{W}{10}\;[\text{A}]\;(DL=2)$ | `0x34EB` | $A\;[\text{A}]\;(DL=1)$ | Direct OBC provides 16-bit word with $100\,\text{mA}$ resolution; VECU drops LSB to an 8-bit integer. |
| **HV DC Output Voltage** | `0x2004` | $\frac{W}{10}\;[\text{V}]$ | `0x34E8` | $\frac{W}{50}\;[\text{V}]$ | Direct OBC uses $100\,\text{mV}$ step; VECU scales with $20\,\text{mV}$ step. |
| **HV DC Output Current** | `0x2003` | $\frac{W}{100}\;[\text{A}]$ | `0x34E9` | $\frac{W}{50}\;[\text{A}]$ | Direct OBC gives $10\,\text{mA}$ resolution; VECU provides $20\,\text{mA}$ step. |
| **CP Rail Voltage** | `0x2000` | $A\;[\text{V}]$ | `0x34F0` | $\frac{A}{10}\;[\text{V}]$ | VECU interpolates analog pilot rail with $100\,\text{mV}$ step; direct node truncates to integer $\text{V}$. |
| **Thermal Monitoring** | `0x1102-04` | $A - 40\;[^\circ\text{C}]$ (3 zones) | *Not relayed* | — | 3-zone thermistors (PFC, Rectifier, Core) are exposed exclusively on `0x786`. |
| **Flashing & Hardware Health** | `0x0200-18` | Diagnostic counts & flags | *Not relayed* | — | Bootloader counters, FEE, EEPROM, and internal DSP SCI integrity flags exist only on `0x786`. |
| **Bidirectional State** | `0x1330-31` | Direct V2L registers | `0x354A` | `A` (`0-1`) | Direct node exposes charge/discharge modes and gun state; VECU exposes a binary wake signal. |

---

## 5. Mathematical Formulations & Scaling Proofs

### 5.1 Single-Byte Formulas ($DL = 1$)

For an incoming payload byte $A \in [0, 255]$:

* **Tenths Scaling ($0.1$ step):**
  $$V_{\text{12V}} = \frac{A}{10} \quad [\text{V}]$$
  $$\text{Example: } A = \text{0x82}\,(130) \implies V = \frac{130}{10} = 13.0\,\text{V}$$

* **Standard Industrial Temperature ($-40^\circ\text{C}$ offset):**
  $$T = A - 40 \quad [^\circ\text{C}]$$
  $$\text{Example: } A = \text{0x5A}\,(90) \implies T = 90 - 40 = 50^\circ\text{C}$$

* **Direct Integers (States, Frequency, Percentages):**
  $$\text{Value} = A$$

### 5.2 Double-Byte Formulas ($DL = 2$)

Given raw payload bytes $A$ (MSB) and $B$ (LSB), the unsigned 16-bit word is:
$$W = (A \times 256) + B$$

* **High-Voltage DC Output & AC Input Current ($0.1$ step):**
  $$V_{\text{DC}} = \frac{W}{10} \quad [\text{V}], \qquad I_{\text{AC}} = \frac{W}{10} \quad [\text{A}]$$
  $$\text{Example: } W = \text{0x0DF0}\,(3568) \implies V_{\text{DC}} = \frac{3568}{10} = 356.8\,\text{V}$$
  $$\text{Example: } W = \text{0x00A0}\,(160) \implies I_{\text{AC}} = \frac{160}{10} = 16.0\,\text{A}$$

* **High-Precision Charging Current ($0.01\,\text{A} = 10\,\text{mA}$ step):**
  $$I_{\text{DC}} = \frac{W}{100} \quad [\text{A}]$$
  $$\text{Example: } W = \text{0x03E8}\,(1000) \implies I_{\text{DC}} = \frac{1000}{100} = 10.00\,\text{A}$$

* **Direct Word Values (AC Voltage, Reprogramming Counters, Frequencies):**
  $$\text{Value} = W$$
  $$\text{Example: } W = \text{0x00E6}\,(230) \implies V_{\text{AC}} = 230\,\text{V}$$
  $$\text{Example: } W = \text{0x03E8}\,(1000) \implies f_{\text{CP}} = 1000\,\text{Hz}$$

---

## 6. Python OBC Payload Decoder

The following script decodes raw incoming UDS diagnostic responses matching the `0x786` OBC parameter specifications:

```python
import struct
from typing import Dict, Any, List, Optional

OBC_DEFINITIONS: List[Dict[str, Any]] = [
    {"CMD": "220110", "NM": "Manufacturing Mode", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220200", "NM": "Reprogramming Success Counter", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Count"},
    {"CMD": "220201", "NM": "Reprogramming Attempt Counter", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Count"},
    {"CMD": "220210", "NM": "CP Line Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220211", "NM": "Charger State", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220212", "NM": "Internal CAN Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220213", "NM": "External CAN Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220214", "NM": "External CAN Transceiver Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220215", "NM": "FEE Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220216", "NM": "EEPROM Write Error", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220217", "NM": "Internal SCI Comm Fail", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220218", "NM": "Internal SCI CRC Fail", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "220219", "NM": "OBD Sensors", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "221000", "NM": "12V Power Supply Voltage", "DL": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "221100", "NM": "Input AC Frequency", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Hz"},
    {"CMD": "221102", "NM": "Primary Board Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "221103", "NM": "Secondary Board Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "221104", "NM": "Transformer Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "221105", "NM": "Input AC Current", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "A"},
    {"CMD": "221106", "NM": "Input AC Voltage", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "V"},
    {"CMD": "221330", "NM": "Charge/Discharge Mode Command", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "221331", "NM": "Discharge Gun Connect Status", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "221332", "NM": "Operation Mode", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "222000", "NM": "CP Voltage", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "V"},
    {"CMD": "222001", "NM": "CP Duty Cycle", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "%"},
    {"CMD": "222002", "NM": "CP Frequency", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Hz"},
    {"CMD": "222003", "NM": "Output DC Current", "DL": 2, "fmt": ">H", "scale": 0.01, "offset": 0.0, "UN": "A"},
    {"CMD": "222004", "NM": "Output DC Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "222100", "NM": "HW Wake-Up Output State", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
]

def decode_obc_response(did_hex: str, raw_payload: bytes) -> Dict[str, Any]:
    """
    Decodes an OBC UDS Service 0x22 response payload.
    raw_payload: Byte string representing data bytes immediately following DID.
    """
    entry: Optional[Dict[str, Any]] = next((item for item in OBC_DEFINITIONS if item["CMD"] == did_hex), None)
    if not entry:
        return {"error": f"DID {did_hex} not found in OBC catalog"}

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

# --- Verification Test Vectors ---
if __name__ == "__main__":
    # Test 1: Input AC Voltage (DID 0x1106) -> 230 V (Raw = 230 -> 0x00E6)
    res_ac_v = decode_obc_response("221106", bytes([0x00, 0xE6]))
    print(res_ac_v)  # Expected: 230 V

    # Test 2: Input AC Current (DID 0x1105) -> 16.0 A (Raw = 160 -> 0x00A0)
    res_ac_i = decode_obc_response("221105", bytes([0x00, 0xA0]))
    print(res_ac_i)  # Expected: 16.0 A

    # Test 3: Output DC Current (DID 0x2003) -> 10.25 A (Raw = 1025 -> 0x0401)
    res_dc_i = decode_obc_response("222003", bytes([0x04, 0x01]))
    print(res_dc_i)  # Expected: 10.25 A

    # Test 4: Output DC Voltage (DID 0x2004) -> 362.4 V (Raw = 3624 -> 0x0E28)
    res_dc_v = decode_obc_response("222004", bytes([0x0E, 0x28]))
    print(res_dc_v)  # Expected: 362.4 V

    # Test 5: Transformer Temperature (DID 0x1104) -> 65 °C (Raw = 105 -> 0x69)
    res_tx_t = decode_obc_response("221104", bytes([0x69]))
    print(res_tx_t)  # Expected: 65 °C
```