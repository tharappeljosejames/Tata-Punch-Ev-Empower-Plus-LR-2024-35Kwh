# DCDC Converter UDS Diagnostic Parameter Identifier (DID) Reference Manual

This technical specification details the diagnostic parameters, communication headers, data lengths, scaling formulas, operational state machines, and engineering units extracted from the Auxiliary High-Voltage to Low-Voltage DC-DC Converter (`[DCDC]`) telemetry configuration.

---

## 1. Protocol Architecture & Header Specifications

* **Diagnostic Service:** Unified Diagnostic Services (ISO 14229) — Service `$22` (`ReadDataByIdentifier`).
* **CAN Request Header (CAN-ID):** `0x784` (Standard 11-bit Physical Request Addressing for Auxiliary Power Module / DC-DC Converter).
* **Expected Response Header:** `0x78C` (Standard 11-bit Slave Response via $\text{Request ID} + \text{0x08}$).
* **Endianness:** Big-Endian (Most Significant Byte first: Byte $A$ followed by Byte $B$).
* **Bus / Cluster Domain:** Powertrain / Subsystem CAN (`BCM: 1003`).

---

## 2. Engineering Unit Code Dictionary

The DCDC parameter configuration utilizes five primary unit dimension identifiers:

| `UN` Code | Physical Dimension | Standard Unit | Notes / Typical Subsystem Context |
|:---:|:---|:---:|:---|
| `0` | State / Discrete / Bitmask / Count | — | Converter operating modes, enable flags, trouble codes, raw status words |
| `2` | Temperature | `$^\circ\text{C}$` | Internal switching bridge MOSFETs (Synchronous Rectifier, Main, Primary) |
| `3` | Electrical Potential | `$\text{V}$` | High-Voltage traction rail input, 12V auxiliary output, KL15 ignition line |
| `4` | Electrical Current | `$\text{A}$` | High-Voltage DC draw (primary side) & Low-Voltage 12V current delivery |
| `11`| Active Electric Power | `$\text{kW}$` | Output power demand / dynamic transfer target ($\text{rated up to } 2.2\,\text{kW}$) |

---

## 3. DCDC Parameter Catalog

All parameters are retrieved via Service `$22` targeting DID range `0x130E` through `0x1329` and system identification registers `0xF0F1` through `0xF0F4`:

### 3.1 Operating State, Control & Diagnostics

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `22130E` | `Work State` | 1 | `A` | State | 0 | 5 | Operational state machine (e.g., Init, Standby, Buck Active, Derated, Fault) |
| `22130F` | `Enable Cmd` | 1 | `A` | State | 0 | 1 | Hardware / VCU converter operational enable flag (`0 = Inhibit, 1 = Enable`) |
| `221325` | `Derate Mode` | 1 | `A` | State | 0 | 7 | Active thermal/current derating profile step (`0 = No Derate`) |
| `221326` | `Error Flag` | 1 | `A` | Bitfield | 0 | 255 | Primary fault classification bitmask register |
| `221329` | `VCU CRC Fail` | 1 | `A` | State | 0 | 1 | VCU high-speed control frame checksum verification status (`1 = Fail`) |

### 3.2 High-Voltage (HV) & Low-Voltage (LV) Power Conversion

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `221310` | `HV In Volt` | 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | High-Voltage DC bus input potential from traction pack |
| `221311` | `HV In Curr` | 2 | `(A*256+B)/10` | $\text{A}$ | 0.0 | 10.0 | High-Voltage primary side current consumption |
| `221312` | `LV Out Volt` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | Regulated low-voltage auxiliary bus (12V battery rail) output voltage |
| `221313` | `LV Out Curr` | 2 | `(A*256+B)/10` | $\text{A}$ | 0.0 | 140.0 | Delivered low-voltage output charging/load current |
| `221327` | `Out Curr Req` | 2 | `(A*256+B)/100`| $\text{A}$ | 0.0 | 140.0 | Demanded LV target current threshold |
| `221328` | `Out Pwr Req` | 2 | `(A*256+B)/1000`| $\text{kW}$ | 0.0 | 2.2 | Demanded dynamic electrical load power capacity |

### 3.3 Power Stage Thermal Monitoring

The DCDC converter incorporates multi-zone thermistors tracking individual switching stage semiconductors:

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `221314` | `SR MOS Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 115.0 | Synchronous Rectification stage MOSFET junction temperature |
| `221315` | `MT MOS Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 115.0 | Main Switching Transformer driver transistor temperature |
| `221316` | `Pri MOS Temp`| 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 115.0 | Primary bridge high-side/low-side switching MOSFET temperature |

### 3.4 Auxiliary Low-Voltage Sense & Wake-Up Signals

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---|:---|
| `221317` | `12V Supply V` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | Terminal 30 logic board unswitched power supply voltage |
| `221318` | `KL15 Wake V` | 1 | `A / 10` | $\text{V}$ | 0.0 | 16.0 | Hardwired Terminal 15 switched ignition sense line voltage |

### 3.5 System Identification & Diagnostic Bitmasks

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---|:---|
| `22F0F4` | `SW Version` | 2 | `A*256+B` | Count | 0 | 65535 | Gateway embedded bootloader/application firmware revision |
| `22F0F1` | `Int Status 1` | 2 | `A*256+B` | Bitfield | 0 | 65535 | Internal diagnostic flag word 1 (over-voltage, under-voltage, hardware lock) |
| `22F0F2` | `Int Status 2` | 2 | `A*256+B` | Bitfield | 0 | 65535 | Internal diagnostic flag word 2 (thermal protection trips, gate driver fault) |
| `22F0F3` | `Int Status 3` | 2 | `A*256+B` | Bitfield | 0 | 65535 | Internal diagnostic flag word 3 (CAN communication, watchdog, timing alerts) |

---

## 4. Architectural Comparison: DCDC (`0x784`) vs. VECU (`0x7E3`)

The DCDC converter can be queried directly via its physical node (`0x784`) or through relayed vehicle-level parameters synthesized by the Vehicle Electronic Control Unit (`0x7E3`):

| Telemetry Metric | DCDC Direct DID (`0x784`) | Direct Scaling Formula | VECU Relayed DID (`0x7E3`) | VECU Scaling Formula | Architectural & Precision Differences |
|:---|:---:|:---|:---:|:---|:---|
| **HV Input Voltage** | `0x1310` | $\frac{W}{10}\;[\text{V}]$ | `0x34E3` | $\frac{W}{50}\;[\text{V}]$ | VECU uses $20\,\text{mV}$ step; direct DCDC uses standard $100\,\text{mV}$ step. |
| **HV Input Current** | `0x1311` | $\frac{W}{10}\;[\text{A}]\;(DL=2)$ | `0x34E2` | $\frac{A}{10}\;[\text{A}]\;(DL=1)$ | Direct node provides 16-bit word; VECU compresses payload into a single byte ($0-12\,\text{A}$). |
| **LV Output Voltage**| `0x1312` | $\frac{A}{10}\;[\text{V}]\;(DL=1)$ | `0x34E1` | $\frac{A}{10}\;[\text{V}]\;(DL=1)$ | Identical $100\,\text{mV}$ resolution across both nodes. |
| **LV Output Current**| `0x1313` | $\frac{W}{10}\;[\text{A}]$ | `0x34E4` | $\frac{W}{100}\;[\text{A}]$ | VECU provides $10\,\text{mA}$ resolution; direct DCDC telemetry provides $100\,\text{mA}$ step. |
| **Operating State** | `0x130E` | `A` (Modes $0-5$) | `0x34E5` | `A` (States $0-3$) | Direct DCDC has a granular 6-state machine; VECU abstracts to a 4-state summary. |
| **MOSFET Temperatures** | `0x1314-16` | $A - 40\;[^\circ\text{C}]$ (3 zones) | `0x34E7` | $A - 48\;[^\circ\text{C}]$ (Single) | Direct node exposes 3 discrete thermal zones; VECU summarizes highest point with a $-48$ offset. |
| **Enable Command** | `0x130F` | `A` (`0-1`) | `0x34EF` | `A` (`0-1`) | Identical binary logic level. |
| **Internal Bitfields** | `0xF0F1-F3` | 16-bit registers | *Not relayed* | — | Lower-level engineering fault registers available only on `0x784`. |

---

## 5. Mathematical Formulations & Scaling Proofs

### 5.1 Single-Byte Conversions ($DL = 1$)

For an incoming byte $A \in [0, 255]$:

* **Tenths Scaling ($0.1$ step):**
  $$V_{\text{LV}} = \frac{A}{10} \quad [\text{V}]$$
  $$\text{Example: } A = \text{0x87}\,(135) \implies V = \frac{135}{10} = 13.5\,\text{V}$$

* **Standard Industrial Temperature ($-40^\circ\text{C}$ offset):**
  $$T_{\text{MOS}} = A - 40 \quad [^\circ\text{C}]$$
  $$\text{Example: } A = \text{0x55}\,(85) \implies T = 85 - 40 = 45^\circ\text{C}$$

### 5.2 Double-Byte Conversions ($DL = 2$)

Given raw payload bytes $A$ (MSB) and $B$ (LSB), the raw unsigned 16-bit word is:
$$W = (A \times 256) + B$$

* **High-Voltage Input & Delivered Low-Voltage Current ($0.1$ step):**
  $$V_{\text{HV}} = \frac{W}{10} \quad [\text{V}], \qquad I_{\text{LV}} = \frac{W}{10} \quad [\text{A}]$$
  $$\text{Example: } W = \text{0x0DD4}\,(3540) \implies V_{\text{HV}} = \frac{3540}{10} = 354.0\,\text{V}$$

* **Requested Current Resolution ($0.01\,\text{A}$ / $10\,\text{mA}$ step):**
  $$I_{\text{req}} = \frac{W}{100} \quad [\text{A}]$$
  $$\text{Example: } W = \text{0x2328}\,(9000) \implies I_{\text{req}} = \frac{9000}{100} = 90.00\,\text{A}$$

* **Requested Power Resolution ($0.001\,\text{kW} = 1\,\text{W}$ step):**
  $$P_{\text{req}} = \frac{W}{1000} \quad [\text{kW}]$$
  $$\text{Example: } W = \text{0x07D0}\,(2000) \implies P_{\text{req}} = \frac{2000}{1000} = 2.000\,\text{kW}\,(2000\,\text{W})$$

---

## 6. Python DCDC Payload Decoder

The following script decodes incoming UDS diagnostic responses matching the `0x784` parameter definitions:

```python
import struct
from typing import Dict, Any, List, Optional

DCDC_DEFINITIONS: List[Dict[str, Any]] = [
    {"CMD": "22F0F4", "NM": "Gateway Software Version", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Count"},
    {"CMD": "22130E", "NM": "Working State", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "22130F", "NM": "Enable Command", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "221325", "NM": "Derating Mode", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "221326", "NM": "Error Flag", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "221329", "NM": "VCU CRC Fail", "DL": 1, "fmt": "B", "scale": 1.0, "offset": 0.0, "UN": "State"},
    {"CMD": "221310", "NM": "HV Input Voltage", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "221311", "NM": "HV Input Current", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "A"},
    {"CMD": "221312", "NM": "LV Output Voltage", "DL": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "221313", "NM": "LV Output Current", "DL": 2, "fmt": ">H", "scale": 0.1, "offset": 0.0, "UN": "A"},
    {"CMD": "221314", "NM": "Internal SR MOS Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "221315", "NM": "Internal MT MOS Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "221316", "NM": "Internal Pri MOS Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40.0, "UN": "°C"},
    {"CMD": "221317", "NM": "12V Power Supply Voltage", "DL": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "221318", "NM": "KL15 Wake-Up Voltage", "DL": 1, "fmt": "B", "scale": 0.1, "offset": 0.0, "UN": "V"},
    {"CMD": "221327", "NM": "Output Current Request", "DL": 2, "fmt": ">H", "scale": 0.01, "offset": 0.0, "UN": "A"},
    {"CMD": "221328", "NM": "Output Power Request", "DL": 2, "fmt": ">H", "scale": 0.001, "offset": 0.0, "UN": "kW"},
    {"CMD": "22F0F1", "NM": "Internal Status 1", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "22F0F2", "NM": "Internal Status 2", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
    {"CMD": "22F0F3", "NM": "Internal Status 3", "DL": 2, "fmt": ">H", "scale": 1.0, "offset": 0.0, "UN": "Bitfield"},
]

def decode_dcdc_response(did_hex: str, raw_payload: bytes) -> Dict[str, Any]:
    """
    Decodes a DCDC UDS Service 0x22 payload.
    Expected raw_payload format: raw data bytes directly after the DID bytes.
    """
    entry: Optional[Dict[str, Any]] = next((item for item in DCDC_DEFINITIONS if item["CMD"] == did_hex), None)
    if not entry:
        return {"error": f"DID {did_hex} not defined in DCDC catalog"}

    fmt = entry["fmt"]
    scale = entry["scale"]
    offset = entry["offset"]
    dl = entry["DL"]

    if len(raw_payload) < dl:
        return {"error": f"Payload length {len(raw_payload)} is shorter than required {dl} bytes"}

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
    # Test 1: HV Input Voltage (DID 0x1310) -> 355.2 V (Raw = 3552 -> 0x0DE0)
    res_hv_v = decode_dcdc_response("221310", bytes([0x0D, 0xE0]))
    print(res_hv_v)  # Expected: 355.2 V

    # Test 2: LV Output Voltage (DID 0x1312) -> 13.8 V (Raw = 138 -> 0x8A)
    res_lv_v = decode_dcdc_response("221312", bytes([0x8A]))
    print(res_lv_v)  # Expected: 13.8 V

    # Test 3: Output Power Request (DID 0x1328) -> 1.75 kW (Raw = 1750 -> 0x06D6)
    res_pwr = decode_dcdc_response("221328", bytes([0x06, 0xD6]))
    print(res_pwr)  # Expected: 1.75 kW

    # Test 4: SR MOS Temperature (DID 0x1314) -> 58 °C (Raw = 98 -> 0x62)
    res_temp = decode_dcdc_response("221314", bytes([0x62]))
    print(res_temp)  # Expected: 58 °C
```