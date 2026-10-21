# MCU UDS Diagnostic Parameter Identifier (DID) Reference Manual

This technical specification details the diagnostic parameters, communication headers, data lengths, scaling formulas, two's-complement arithmetic, and engineering units extracted from the Motor Control Unit (`[MCU]`) telemetry configuration.

---

## 1. Protocol Architecture & Header Specifications

* **Diagnostic Service:** Unified Diagnostic Services (ISO 14229) — Service `$22` (`ReadDataByIdentifier`).
* **CAN Request Header (CAN-ID):** `0x783` (Standard 11-bit Physical Request Addressing for Traction Inverter / Motor Control Unit).
* **Expected Response Header:** `0x78B` (Standard 11-bit Slave Response via `Request ID + 0x08`).
* **Endianness:** Big-Endian (Most Significant Byte first: Byte $A$ followed by Byte $B$).
* **Signed Arithmetic Format:** Big-Endian 16-bit Two's Complement (`int16_t`), indicated in formulas as $\text{Signed}(A) \times 256 + B$.

---

## 2. Engineering Unit Code Dictionary

The MCU configuration introduces unit identifier `11` alongside the standard dimension definitions:

| `UN` Code | Physical Dimension | Standard Unit | Notes / Typical Subsystem Context |
|:---:|:---|:---:|:---|
| `0` | State / Enum / Discrete | — | Inverter operating state machine mode (`0-7`) |
| `2` | Temperature | `$^\circ\text{C}$` | Motor stator windings, power electronics (IGBT/SiC) |
| `3` | Electrical Potential | `$\text{V}$` | High-Voltage DC Link capacitor rail potential |
| `4` | Electrical Current | `$\text{A}$` | DC Link input current & 3-phase AC stator currents |
| `6` | Rotational Velocity | `$\text{RPM}$` | Rotor mechanical shaft speed & speed ceilings |
| `8` | Mechanical Torque | `$\text{N}\cdot\text{m}$` | Shaft torque generation and regenerative braking caps |
| `11`| Active Electric Power | `$\text{kW}$` | Delivered/regenerated mechanical-equivalent electric power |

---

## 3. MCU Parameter Catalog

All parameters are addressed via Service `$22` targeting DID range `0x0506` through `0x0520`:

| Command (DID) | Short Name | DL | Raw Formula | Physical Unit | Min Range | Max Range | Detailed Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `220506` | `Max Trq Lim` | 2 | `Signed(A)*256+B` | $\text{N}\cdot\text{m}$ | 0.0 | 190.0 | Maximum allowed motoring shaft torque limit |
| `220507` | `Max Regen Trq` | 2 | `Signed(A)*256+B` | $\text{N}\cdot\text{m}$ | -160.0 | 160.0 | Maximum permitted electrical recuperation torque |
| `220508` | `Max Speed Lim` | 2 | `Signed(A)*256+B` | $\text{RPM}$ | 0.0 | 12000.0 | Forward rotor overspeed protection threshold |
| `220509` | `Rev Speed Lim` | 2 | `Signed(A)*256+B` | $\text{RPM}$ | -2500.0 | 2500.0 | Reverse rotor velocity governor limit |
| `220510` | `Motor Speed` | 2 | `Signed(A)*256+B` | $\text{RPM}$ | -2500.0 | 12000.0 | Measured rotor mechanical shaft velocity |
| `220511` | `Motor Torque` | 2 | `Signed(A)*256+B` | $\text{N}\cdot\text{m}$ | -160.0 | 190.0 | Instantaneous delivered/absorbed shaft torque |
| `220512` | `DC Link Curr` | 2 | `(Signed(A)*256+B)/2` | $\text{A}$ | -130.0 | 300.0 | Inverter DC Link bus current (+ Motoring, - Regen) |
| `220513` | `DC Link Volt` | 2 | `(A*256+B)/2` | $\text{V}$ | 260.0 | 400.0 | Measured High-Voltage DC Link rail voltage |
| `220514` | `Operation Mode`| 1 | `A` | State | 0 | 7 | Active inverter internal state machine mode |
| `220515` | `Motor Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 150.0 | Stator winding internal thermistor temperature |
| `220516` | `Inverter Temp`| 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 120.0 | Power module (IGBT / MOSFET junction) temp |
| `220517` | `E-Drive Power` | 2 | `Signed(A)*256+B` | $\text{kW}$ | -45.0 | 95.0 | Net electrical power transferred to/from stator |
| `220518` | `Phase A Curr` | 2 | `(Signed(A)*256+B)/2` | $\text{A}$ | -380.0 | 380.0 | Instantaneous stator Phase A AC current |
| `220519` | `Phase B Curr` | 2 | `(Signed(A)*256+B)/2` | $\text{A}$ | -380.0 | 380.0 | Instantaneous stator Phase B AC current |
| `220520` | `Phase C Curr` | 2 | `(Signed(A)*256+B)/2` | $\text{A}$ | -380.0 | 380.0 | Instantaneous stator Phase C AC current |

---

## 4. Architectural Comparison: MCU (`0x783`) vs. VECU (`0x7E3`)

A key difference exists between parameter encoding on the internal MCU diagnostic interface versus the vehicle-level coordinator (VECU):

| Telemetry Parameter | MCU Direct DID (`0x783`) | MCU Encoding Method | VECU Relayed DID (`0x7E3`) | VECU Encoding Method |
|:---|:---:|:---|:---:|:---|
| **Motor Speed** | `0x0510` | Big-Endian Signed 16-bit (`int16`) | `0x34A4` | Unsigned 16-bit with offset `(W - 17000)` |
| **Motor Torque** | `0x0511` | Big-Endian Signed 16-bit (`int16`) | `0x34A5` | Unsigned 16-bit with offset `(W - 510)` |
| **DC Bus Current** | `0x0512` | Signed 16-bit halved `(int16 / 2)` | `0x34A6` | Unsigned 16-bit with offset `(W / 2 - 510)`|
| **DC Bus Voltage** | `0x0513` | Unsigned 16-bit halved `(W / 2)` | `0x34A7` | Unsigned 16-bit halved `(W / 2)` |
| **Operating Mode** | `0x0514` | Single byte enumeration (`A`) | `0x3459` | Single byte enumeration (`A`) |
| **Winding Temp** | `0x0515` | Single byte offset `(A - 40)` | `0x3472` | Single byte offset `(A - 40)` |
| **Inverter Temp** | `0x0516` | Single byte offset `(A - 40)` | `0x3473` | Single byte offset `(A - 40)` |
| **Phase A/B/C Currents** | `0x0518-20`| Signed 16-bit halved `(int16 / 2)`| *Not mapped directly to VECU UDS tables* | — |

* **Firmware Implementation Insight:** The MCU stores signed values internally as standard two's-complement words. When the VECU packages these metrics for higher-level vehicle telemetry or standard diagnostic testers, it frequently adds an artificial positive offset (e.g., $+17000$ for speed, $+510$ for torque) to avoid negative integer handling across CAN matrices.

---

## 5. Mathematical Unpacking & Two's Complement Conversion

### 5.1 Signed 16-bit Conversion ($DL = 2$)
Given raw payload bytes $A$ (MSB) and $B$ (LSB):
1. Compute the raw unsigned 16-bit integer:
   $$W_{\text{raw}} = (A \times 256) + B$$
2. Apply standard two's-complement signed conversion:
   $$\text{Signed}(W) = \begin{cases} W_{\text{raw}}, & \text{if } W_{\text{raw}} < 32768 \\ W_{\text{raw}} - 65536, & \text{if } W_{\text{raw}} \ge 32768 \end{cases}$$
3. Scale according to physical dimension:
   * **Rotational Speed & Direct Limits ($1\,\text{RPM}$ / $1\,\text{N}\cdot\text{m}$ step):**
     $$\text{Value} = \text{Signed}(W)$$
   * **DC Link & Phase Currents ($0.5\,\text{A}$ resolution):**
     $$I = \frac{\text{Signed}(W)}{2}$$
   * **DC Link Voltage ($0.5\,\text{V}$ resolution, Unsigned):**
     $$V_{\text{dc}} = \frac{W_{\text{raw}}}{2}$$

---

## 6. Python MCU Payload Decoder

The following script handles big-endian signed (`>h`) and unsigned (`>H`) byte conversions directly using Python's `struct` module:

```python
import struct
from typing import Dict, Any, List

MCU_DEFINITIONS: List[Dict[str, Any]] = [
    {"CMD": "220506", "NM": "E-Drive Max Torque Limit", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "N·m"},
    {"CMD": "220507", "NM": "E-Drive Max Regen Torque Limit", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "N·m"},
    {"CMD": "220508", "NM": "E-Drive Max Speed Limit", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "RPM"},
    {"CMD": "220509", "NM": "E-Drive Reverse Speed Limit", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "RPM"},
    {"CMD": "220510", "NM": "Motor Rotational Speed", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "RPM"},
    {"CMD": "220511", "NM": "Motor Output Torque", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "N·m"},
    {"CMD": "220512", "NM": "HV DC Link Current", "DL": 2, "fmt": ">h", "scale": 0.5, "UN": "A"},
    {"CMD": "220513", "NM": "HV DC Link Voltage", "DL": 2, "fmt": ">H", "scale": 0.5, "UN": "V"},
    {"CMD": "220514", "NM": "E-Drive Operation Mode", "DL": 1, "fmt": "B", "scale": 1.0, "UN": "State"},
    {"CMD": "220515", "NM": "Motor Winding Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40, "UN": "°C"},
    {"CMD": "220516", "NM": "Inverter Temperature", "DL": 1, "fmt": "B", "scale": 1.0, "offset": -40, "UN": "°C"},
    {"CMD": "220517", "NM": "E-Drive Power", "DL": 2, "fmt": ">h", "scale": 1.0, "UN": "kW"},
    {"CMD": "220518", "NM": "E-Drive Phase A Current", "DL": 2, "fmt": ">h", "scale": 0.5, "UN": "A"},
    {"CMD": "220519", "NM": "E-Drive Phase B Current", "DL": 2, "fmt": ">h", "scale": 0.5, "UN": "A"},
    {"CMD": "220520", "NM": "E-Drive Phase C Current", "DL": 2, "fmt": ">h", "scale": 0.5, "UN": "A"},
]

def decode_mcu_response(did_hex: str, raw_payload: bytes) -> Dict[str, Any]:
    """
    Decodes an MCU UDS response payload for diagnostic service 0x22.
    """
    entry = next((item for item in MCU_DEFINITIONS if item["CMD"] == did_hex), None)
    if not entry:
        return {"error": f"DID {did_hex} not defined in MCU catalog"}

    fmt = entry["fmt"]
    scale = entry.get("scale", 1.0)
    offset = entry.get("offset", 0.0)

    # Unpack binary payload
    raw_val = struct.unpack(fmt, raw_payload[:entry["DL"]])[0]
    physical_val = round((raw_val * scale) + offset, 2)

    return {
        "did": entry["CMD"],
        "name": entry["NM"],
        "value": physical_val,
        "unit": entry["UN"]
    }

# --- Verification & Test Vectors ---
if __name__ == "__main__":
    # Test 1: Motor Rotational Speed (DID 0x0510) with negative speed (-1500 RPM -> 0xFA24)
    res_speed = decode_mcu_response("220510", bytes([0xFA, 0x24]))
    print(res_speed)  # Expected: -1500 RPM

    # Test 2: Phase A Current (DID 0x0518) with -120.0 A (Raw = -240 -> 0xFF10)
    res_phase_a = decode_mcu_response("220518", bytes([0xFF, 0x10]))
    print(res_phase_a)  # Expected: -120.0 A

    # Test 3: DC Link Voltage (DID 0x0513) with 360.0 V (Raw = 720 -> 0x02D0)
    res_dc_volt = decode_mcu_response("220513", bytes([0x02, 0xD0]))
    print(res_dc_volt)  # Expected: 360.0 V
```