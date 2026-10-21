# VECU UDS Diagnostic Parameter Identifier (DID) Reference Manual

This technical specification details the diagnostic parameters, communication headers, data lengths, scaling formulas, and engineering units extracted from the Vehicle Electronic Control Unit (`[VECU]`) telemetry configuration.

---

## 1. Protocol Architecture & Header Specifications

* **Diagnostic Service:** Unified Diagnostic Services (ISO 14229) — Service `$22` (`ReadDataByIdentifier`).
* **CAN Request Header (CAN-ID):** `0x7E3` (Standard 11-bit Physical Addressing for Powertrain/Vehicle Control Unit).
* **Expected Response Header:** `0x7EB` (Standard 11-bit Slave Response).
* **Endianness:** Big-Endian (Most Significant Byte first: Byte $A$ followed by Byte $B$).

---

## 2. Engineering Unit Code Dictionary

Based on the `UN` values across the dataset, the engineering units correlate to the following standard physical definitions:

| `UN` Code | Physical Dimension | Standard Unit | Notes |
|:---:|:---|:---:|:---|
| `0` | State / Enum / Digital / Count | — | Discrete logic levels, enumerations, or raw bitfields |
| `1` | Ratio / Percentage | `%` | State of Charge, PWM duty cycles, throttle/pedal travel |
| `2` | Temperature | `$^\circ\text{C}$` | Inlet, battery, inverter, motor, coolant, ambient |
| `3` | Electrical Potential | `$\text{V}$` | HV bus, 12V supply, cell voltages, analog sensor taps |
| `4` | Electrical Current | `$\text{A}$` | HV battery current, OBC input/output, DC-DC current |
| `5` | Linear Velocity | `$\text{km/h}$` | Vehicle speed and cruise target speeds |
| `6` | Rotational Velocity | `$\text{RPM}$` | Motor shaft speed, compressor rotation |
| `7` | Pressure | `$\text{kPa}$` | Brake vacuum booster gauge pressure |
| `8` | Mechanical Torque | `$\text{N}\cdot\text{m}$` | Driver request, motor output, recuperation torque |
| `9` | Frequency | `$\text{Hz}$` | PWM carrier frequency (HVIL, CP signal, pumps) |
| `10`| Electrical Resistance | `$\text{k}\Omega$` | High Voltage Isolation resistance |

---

## 3. Subsystem Parameter Catalog

### 3.1 High-Voltage Battery & Contactors

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `22341D` | `HV Batt Curr` | 2 | `(A*256+B)/10 - 600` | $\text{A}$ | -130.0 | 300.0 | HV Battery Current (Signed: + Discharge, - Charge) |
| `22341E` | `HV Batt Volt` | 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | Total HV Pack Voltage |
| `22341F` | `HV -ve Cnt St` | 1 | `A` | State | 0 | 1 | Negative Main Contactor Closed/Open |
| `223420` | `HV +ve Cnt St` | 1 | `A` | State | 0 | 1 | Positive Main Contactor Closed/Open |
| `223421` | `HV Batt SOC` | 2 | `(A*256+B)/10` | `%` | 0.0 | 100.0 | High Voltage Battery State of Charge |
| `223422` | `HV Batt SOH` | 1 | `A` | `%` | 0.0 | 100.0 | High Voltage Battery State of Health |
| `223423` | `Max Cell Temp` | 1 | `A - 50` | $^\circ\text{C}$ | -10.0 | 65.0 | Peak Battery Cell Temperature |
| `223424` | `Min Cell Temp` | 1 | `A - 50` | $^\circ\text{C}$ | -10.0 | 65.0 | Minimum Battery Cell Temperature |
| `223477` | `Max Cell Volt` | 2 | `(A*256+B)/100` | $\text{V}$ | 2.40 | 3.70 | Highest Individual Cell Voltage |
| `223478` | `Min Cell Volt` | 2 | `(A*256+B)/100` | $\text{V}$ | 2.40 | 3.70 | Lowest Individual Cell Voltage |
| `22342D` | `HV Isol` | 2 | `A*256+B` | $\text{k}\Omega$ | 0.0 | 50000.0 | HV Isolation Resistance Level |
| `223458` | `Batt Op Mode` | 1 | `A` | State | 0 | 7 | HV Battery Current Operating Mode |
| `2234AC` | `Batt Req Mode` | 1 | `A` | State | 0 | 7 | VCU Requested Battery Mode |
| `2234AD` | `Busbar Volt` | 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | Pack Intermediate Busbar Voltage |
| `2234B5` | `Batt Flt Rank`| 1 | `A` | State | 0 | 5 | Battery Management Fault Severity Level |

---

### 3.2 Traction Motor & Inverter (MCU / E-Drive)

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `2234A4` | `E-Drv Out Rpm` | 2 | `A*256+B - 17000` | $\text{RPM}$ | -2500.0 | 12000.0 | Traction Motor Actual Shaft Velocity |
| `2234AA` | `E-Drv Req Rpm` | 2 | `A*256+B - 17000` | $\text{RPM}$ | -2500.0 | 12000.0 | VCU Target Inverter Velocity Request |
| `2234A5` | `E-Drv Out Trq` | 2 | `A*256+B - 510` | $\text{N}\cdot\text{m}$ | -160.0 | 190.0 | Delivered Motor Shaft Torque |
| `2234A9` | `E-Drv Req Trq` | 2 | `A*256+B - 510` | $\text{N}\cdot\text{m}$ | -160.0 | 190.0 | Target Demanded Drive Torque |
| `2234A6` | `E-Drv HV Curr` | 2 | `(A*256+B)/2 - 510` | $\text{A}$ | -130.0 | 300.0 | DC Inverter Input Current |
| `2234A7` | `E-Drv HV Volt` | 2 | `(A*256+B)/2` | $\text{V}$ | 260.0 | 400.0 | DC Inverter DC-Link Voltage |
| `223472` | `Motor Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 150.0 | Stator Winding Internal Temperature |
| `223473` | `Inverter Temp`| 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 120.0 | IGBT/SiC Power Stage Temperature |
| `223459` | `E-Drv Op Mode` | 1 | `A` | State | 0 | 7 | Active Inverter Operating Mode |
| `2234A8` | `E-Drv Req St` | 1 | `A` | State | 0 | 7 | Commanded Inverter Target State |
| `2234AB` | `E-Drv Trq Dir` | 1 | `A` | State | 0 | 3 | Torque Direction Vector (Fwd/Rev) |
| `2234B6` | `MCU Flt Grade` | 1 | `A` | State | 0 | 5 | Inverter Diagnostic Trouble Rank |
| `22344E` | `MCU Cont Cmd` | 1 | `A` | State | 0 | 1 | MCU Precharge/Main Relay Low Side Cmd |
| `2234D8` | `MCU Cnt HS Cmd`| 1 | `A` | State | 0 | 1 | MCU Contactor High Side Driver Output |

---

### 3.3 AC & DC Fast Charging (CCS / IEC 61851 / ISO 15118)

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---:|:---|
| `223469` | `Inlet PP Volt` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Proximity Pilot (PP) Voltage Level |
| `22346A` | `Inlet CP Amp` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Control Pilot (CP) Analog Peak Voltage |
| `22346B` | `Inlet CP Duty` | 2 | `(A*256+B)/100` | `%` | 0.0 | 100.0 | Control Pilot PWM Duty Cycle |
| `22346C` | `Inlet CP Freq` | 2 | `A*256+B` | $\text{Hz}$ | 800.0 | 1200.0 | Control Pilot PWM Frequency |
| `22346D` | `CP S2 En Cmd` | 1 | `A` | State | 0 | 1 | S2 Vehicle State C/D Load Switch Cmd |
| `22346E` | `CP S2 Status` | 1 | `A` | State | 0 | 1 | S2 Switch Actual Closed Status |
| `223436` | `Gun Lock Cmd` | 1 | `A` | State | 0 | 3 | Inlet Motor Lock/Unlock Driver State |
| `223441` | `Gun Lock Fbk` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Microswitch Position Voltage Feedback |
| `2234C8` | `Gun Lock BSW` | 1 | `A` | State | 0 | 3 | Basic Software State of Lock Pin |
| `2234CB` | `CCS Gun Stat` | 1 | `A` | State | 0 | 7 | DC Plug Connection & Latch State |
| `223470` | `Inlet +ve T` | 1 | `A - 40` | $^\circ\text{C}$ | -10.0 | 90.0 | DC Positive Terminal Thermistor |
| `223471` | `Inlet -ve T` | 1 | `A - 40` | $^\circ\text{C}$ | -10.0 | 90.0 | DC Negative Terminal Thermistor |
| `2234F5` | `AC Neut Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -10.0 | 90.0 | AC Neutral Pin Temperature |
| `2234F6` | `AC Pos Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -10.0 | 90.0 | AC Line (Phase) Pin Temperature |
| `223450` | `DC +ve Cnt Cmd`| 1 | `A` | State | 0 | 1 | DC Fast Charge Positive Contactor LS Cmd |
| `223451` | `DC -ve Cnt Cmd`| 1 | `A` | State | 0 | 1 | DC Fast Charge Negative Contactor LS Cmd |
| `2234D9` | `DC +ve HS Cmd` | 1 | `A` | State | 0 | 1 | DC Positive Contactor High Side Cmd |
| `2234D7` | `DC -ve HS Cmd` | 1 | `A` | State | 0 | 1 | DC Negative Contactor High Side Cmd |
| `223487` | `DC FC Cnt St` | 1 | `A` | State | 0 | 3 | Readback State of DC Charge Contactors |
| `22351E` | `CCS EV Ready` | 1 | `A` | State | 0 | 1 | Vehicle Charge Readiness Flag |
| `22351F` | `CCS Inlet V` | 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | Measured DC Bus Voltage at Vehicle Inlet |
| `223520` | `CCS Curr Req` | 2 | `(A*256+B)/32` | $\text{A}$ | 0.0 | 140.0 | Target DC Current Demanded by VCU |
| `223521` | `CCS Volt Req` | 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | Target DC Charge Voltage Demanded |
| `223522` | `CCS Stat Curr`| 2 | `(A*256+B)/32` | $\text{A}$ | 0.0 | 140.0 | Reported Delivered EVSE Current |
| `223523` | `CCS Stat Volt`| 2 | `(A*256+B)/10` | $\text{V}$ | 260.0 | 400.0 | Reported Delivered EVSE Voltage |
| `223524` | `CCS Isol Stat`| 1 | `A` | State | 0 | 3 | Station EVSE Pre-Charge Isolation Check |
| `2234CE` | `FC Sequence` | 1 | `A` | State | 0 | 15 | Active Fast Charging State Machine Step |
| `2234CF` | `Slow Chg Seq` | 1 | `A` | State | 0 | 15 | AC Slow Charging Sequence State |
| `2234CA` | `Chg Shut Stat`| 1 | `A` | State | 0 | 7 | Charge Interruption / Abort Sequence |
| `2234C9` | `Chg Shut Rsn` | 1 | `A` | Count | 0 | 255 | Charge Abort Classification Reason Code |
| `22352F` | `Chg Curr Lim` | 2 | `(A*256+B)/10` | $\text{A}$ | 0.0 | 140.0 | Calculated Charge Current Throttle Cap |
| `2234CD` | `Chg Lim Src 1`| 1 | `A` | State | 0 | 15 | Root Component Restricting Rate (BMS/OBC) |
| `223530` | `Chg Lim Src 2`| 1 | `A` | State | 0 | 15 | Secondary Component Restricting Rate |

---

### 3.4 3-in-1 Power Conversion (Bi-Directional OBC & DC-DC Converter)

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---|:---|
| `2234E1` | `3in1 DCDC Out` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | Low Voltage (12V) Bus Output Voltage |
| `2234E2` | `3in1 DCDC InA` | 1 | `A / 10` | $\text{A}$ | 0.0 | 12.0 | High Voltage Side Draw of DC-DC |
| `2234E3` | `3in1 DCDC InV` | 2 | `(A*256+B)/50` | $\text{V}$ | 260.0 | 400.0 | High Voltage DC Bus Input to DC-DC |
| `2234E4` | `3in1 DCDC OutA`| 2 | `(A*256+B)/100` | $\text{A}$ | 0.0 | 140.0 | 12V Auxiliary Supply Output Current |
| `2234E5` | `3in1 DCDC St` | 1 | `A` | State | 0 | 3 | DC-DC Operating State (Standby/Active/Fault) |
| `2234E7` | `3in1 DCDC Tmp` | 1 | `A - 48` | $^\circ\text{C}$ | -20.0 | 110.0 | Internal DC-DC Transformer/MOSFET Temp |
| `2234EE` | `DCDC V Cmd` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | Commanded 12V Target Float Voltage |
| `2234EF` | `DCDC En Cmd` | 1 | `A` | State | 0 | 1 | Enable Pin Command to DC-DC Converter |
| `2234E8` | `3in1 OBC DC V` | 2 | `(A*256+B)/50` | $\text{V}$ | 260.0 | 400.0 | On-Board Charger DC Rectified Output |
| `2234E9` | `3in1 OBC DC A` | 2 | `(A*256+B)/50` | $\text{A}$ | 0.0 | 25.0 | OBC Delivered HV Charging Current |
| `2234EA` | `3in1 OBC Stat` | 1 | `A` | State | 0 | 3 | On-Board Charger Operating State |
| `2234EB` | `3in1 OBC AC A` | 1 | `A` | $\text{A}$ | 0.0 | 36.0 | AC Mains RMS Input Phase Current |
| `2234EC` | `3in1 OBC AC V` | 2 | `(A*256+B)/50` | $\text{V}$ | 160.0 | 275.0 | Grid Single-Phase AC RMS Voltage |
| `2234ED` | `3in1 OBC CRC` | 1 | `A` | State | 0 | 1 | OBC Internal CAN Message CRC Fault |
| `2234E6` | `3in1 CRC Flt` | 1 | `A` | State | 0 | 1 | Overall 3-in-1 Comms Integrity Alert |
| `2234F0` | `3in1 CP Volt` | 1 | `A / 10` | $\text{V}$ | 0.0 | 14.0 | CP Pilot Rail Voltage measured by OBC |
| `2234F1` | `3in1 CP Duty` | 1 | `A` | `%` | 0.0 | 100.0 | CP Duty Cycle extracted by OBC |
| `2234F2` | `3in1 CP Freq` | 2 | `A*256+B` | $\text{Hz}$ | 800.0 | 1200.0 | CP Carrier Frequency observed by OBC |
| `2234F3` | `3in1 Chg Cmd` | 1 | `A` | State | 0 | 3 | VCU Enable Signal to OBC Subsystem |
| `2234F4` | `OBC DC A Cmd` | 2 | `(A*256+B)/50` | $\text{A}$ | 0.0 | 25.0 | Target DC Output Current from VCU |
| `22354A` | `3in1 OBC Wake` | 1 | `A` | State | 0 | 1 | Bidirectional / V2X Power Wakeup Trigger |

---

### 3.5 Thermal Management, HVAC & Cabling Safety

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---|:---|
| `223425` | `Comp Speed` | 2 | `A*256+B` | $\text{RPM}$ | 0.0 | 8500.0 | Electric AC Compressor Motor Speed |
| `22342B` | `Comp HV Volt` | 1 | `A * 2` | $\text{V}$ | 260.0 | 400.0 | High Voltage Supplied to Compressor |
| `22342C` | `Comp HV Curr` | 1 | `A / 10` | $\text{A}$ | 0.0 | 20.0 | Current Draw of AC Compressor Inverter |
| `223475` | `Comp Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 110.0 | Compressor Scroll/Inverter Temperature |
| `223526` | `Comp State` | 1 | `A` | State | 0 | 3 | Operational Mode of Electric Compressor |
| `22345C` | `Comp Diag 1` | 1 | `A` | Count | 0 | 255 | Compressor Primary Diagnostic Register |
| `22345D` | `Comp Diag 2` | 1 | `A` | Count | 0 | 255 | Compressor Secondary Fault Register |
| `223445` | `AC Press Sens`| 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Refrigerant Circuit High Pressure Sensor |
| `2234B9` | `FATC AC Req` | 1 | `A` | State | 0 | 1 | Climate Control (FATC) Compressor On Req |
| `2234BA` | `FATC Heat %` | 1 | `A` | `%` | 0.0 | 100.0 | Cabin Electric Heating Target Duty |
| `2234BB` | `FATC Cool %` | 1 | `A` | `%` | 0.0 | 100.0 | Cabin Air Conditioning Target Duty |
| `22350A` | `PTC Temp FATC`| 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 100.0 | PTC Coolant/Core Temperature Sensor |
| `22344F` | `PTC Cont Cmd` | 1 | `A` | State | 0 | 1 | Low-Side Command to PTC Power Relay |
| `2234DA` | `PTC Cnt HS Cmd`| 1 | `A` | State | 0 | 1 | High-Side Driver for PTC Heater |
| `223437` | `Batt Pmp Duty` | 1 | `A` | `%` | 0.0 | 100.0 | Battery Coolant Circulation Pump PWM |
| `223438` | `Batt Pmp Freq` | 1 | `A` | $\text{Hz}$ | 0.0 | 200.0 | Battery Pump Feedback Frequency |
| `223439` | `Trac Pmp Duty` | 1 | `A` | `%` | 0.0 | 100.0 | Inverter/Motor Cooling Loop Pump PWM |
| `22343A` | `Trac Pmp Freq` | 1 | `A` | $\text{Hz}$ | 0.0 | 200.0 | Motor Cooling Pump Feedback Frequency |
| `22344C` | `Cab Cool Sol` | 1 | `A` | State | 0 | 1 | Cabin Refrigerant Loop Diverter Solenoid |
| `22344D` | `Batt Cool Sol` | 1 | `A` | State | 0 | 1 | Battery Chiller Loop Diverter Solenoid |
| `223452` | `Fan Fast Cmd` | 1 | `A` | State | 0 | 1 | Radiator Fan High-Speed Relay Driver |
| `223453` | `Fan Slow Cmd` | 1 | `A` | State | 0 | 1 | Radiator Fan Low-Speed Relay Driver |
| `22345E` | `Veh HVIL Duty` | 1 | `A` | `%` | 0.0 | 100.0 | Vehicle-Level HV Interlock Loop Sense |
| `22345F` | `Veh HVIL Freq` | 1 | `A` | $\text{Hz}$ | 0.0 | 250.0 | Vehicle HVIL Continuity Frequency |
| `223460` | `PDU HVIL Duty` | 1 | `A` | `%` | 0.0 | 100.0 | Power Distribution Unit HVIL Duty |
| `223461` | `PDU HVIL Freq` | 1 | `A` | $\text{Hz}$ | 0.0 | 250.0 | PDU HVIL Interlock Carrier Frequency |
| `223426` | `PDU +ve Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 120.0 | PDU Positive Internal Terminal Temp |
| `223427` | `PDU -ve Temp` | 1 | `A - 40` | $^\circ\text{C}$ | -20.0 | 120.0 | PDU Negative Internal Terminal Temp |
| `223442` | `PDU Cover Sw` | 1 | `A` | State | 0 | 1 | PDU Top Shell Service Cover Interlock |

---

### 3.6 Chassis Dynamics, Braking & ESP Interface

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---|:---|
| `223462` | `Veh Speed` | 2 | `((A*256+B)/32)*3.6` | $\text{km/h}$ | 0.0 | 150.0 | Filtered CAN Wheel Vehicle Speed |
| `223468` | `Brake Vac` | 1 | `A - 120` | $\text{kPa}$ | -100.0 | 10.0 | Brake Vacuum Booster Pressure Gauge |
| `223449` | `Vac Pump Rly` | 1 | `A` | State | 0 | 1 | Auxiliary Vacuum Pump Control Relay |
| `223431` | `Brake Sens 1` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Main Brake Pedal Potentiometer 1 |
| `223432` | `Brake Sens 2` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Redundant Brake Pedal Potentiometer 2 |
| `223443` | `Brake Sw Stat`| 1 | `A` | State | 0 | 1 | Primary Hydraulic Brake Lamp Switch |
| `223542` | `Brk Comp Sw` | 1 | `A` | State | 0 | 1 | Brake Pedal Inverse Complementary Switch |
| `2234F7` | `ESP Brk Pos %`| 1 | `A / 2` | `%` | 0.0 | 100.0 | Decoded Brake Travel percentage via ESP |
| `2234F8` | `ESP Recup Trq`| 2 | `(A*256+B)/10` | $\text{N}\cdot\text{m}$ | 0.0 | 160.0 | ESP Permitted Regenerative Braking Cap |
| `2234F9` | `EPB Error ESP`| 1 | `A` | State | 0 | 3 | Electronic Parking Brake Internal Error |
| `2234FA` | `AVH State ESP`| 1 | `A` | State | 0 | 3 | Auto Vehicle Hold Standstill Hold State |
| `2234FB` | `Brk S2 St ESP`| 1 | `A` | State | 0 | 3 | ESP Plausibility Check on Sensor 2 |
| `2234FC` | `EPB State ESP`| 1 | `A` | State | 0 | 3 | Electro-Mechanical Caliper Clamp State |
| `2234FD` | `Brk S2 V ESP` | 2 | `(A*256+B)/100` | $\text{V}$ | 0.0 | 5.0 | Sensor 2 Voltage Digested by ESP |
| `2234FE` | `Brk S1 St ESP`| 1 | `A` | State | 0 | 3 | ESP Plausibility Check on Sensor 1 |
| `2234FF` | `Brk S1 V ESP` | 2 | `(A*256+B)/100` | $\text{V}$ | 0.0 | 5.0 | Sensor 1 Voltage Digested by ESP |
| `223500` | `ESP Trq Inc` | 2 | `(A*256+B)/10 - 500`| $\text{N}\cdot\text{m}$ | -160.0 | 190.0 | Stability System Positive Torque Adjust |
| `223501` | `ESP Trq Lim` | 2 | `(A*256+B)/10 - 500`| $\text{N}\cdot\text{m}$ | -160.0 | 190.0 | Dynamic Stability Fast Torque Cut Limit |
| `223502` | `ABS Active` | 1 | `A` | State | 0 | 1 | Anti-Lock Braking System Intervention |
| `223503` | `TCS Active` | 1 | `A` | State | 0 | 1 | Traction Control Slip Intervention |
| `223504` | `ESP Active` | 1 | `A` | State | 0 | 1 | Electronic Stability Dynamic Intervention |
| `223506` | `HHC Active` | 1 | `A` | State | 0 | 1 | Hill Hold Control Active Incline Hold |
| `223507` | `HDC Active` | 1 | `A` | State | 0 | 1 | Hill Descent Autonomous Brake Speed Hold |
| `223508` | `ESP Regen En` | 1 | `A` | State | 0 | 1 | Dynamic Clearance to Regenerate Power |
| `223509` | `ESP Sport En` | 1 | `A` | State | 0 | 1 | Stability Clearance to Engage Sport Map |

---

### 3.7 Driver Controls, Regeneration & Vehicle Power States

| Command (DID) | Short Name | DL | Formula | Unit | Min | Max | Description |
|:---|:---|:---:|:---|:---:|:---:|:---|:---|
| `22342E` | `Accel Sens 1` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Accelerator Pedal Sensor Track 1 |
| `22342F` | `Accel Sens 2` | 1 | `A / 51` | $\text{V}$ | 0.0 | 5.0 | Accelerator Pedal Sensor Track 2 |
| `223517` | `Drv Req Torq` | 2 | `(A*256+B)/4 - 2048`| $\text{N}\cdot\text{m}$ | -160.0 | 190.0 | Normalized Driver Intended Torque |
| `223516` | `Max Regen Trq`| 2 | `(A*256+B)/4 - 2048`| $\text{N}\cdot\text{m}$ | -160.0 | 0.0 | Powertrain Instantaneous Regen Ceiling |
| `223518` | `Recup Brk St` | 1 | `A` | State | 0 | 3 | Regenerative Braking Interaction Phase |
| `223519` | `Recup Act Trq`| 2 | `(A*256+B)/4 - 2048`| $\text{N}\cdot\text{m}$ | -160.0 | 0.0 | Actual Drag / Recuperation Shaft Torque |
| `223510` | `Regen Display`| 1 | `A` | State | 0 | 3 | Active Regen Setting Shown on Cluster |
| `223549` | `Paddle Regen` | 1 | `A` | State | 0 | 3 | Steering Paddle Selected Regen Step |
| `223514` | `Regen Sel Pos`| 1 | `A` | State | 0 | 1 | Regen Toggle Allowed (Inhibited if Full) |
| `22350F` | `Drv Mode Sw` | 1 | `A` | State | 0 | 3 | Drive Mode Console Switch Position |
| `223511` | `Drv Mode Disp`| 1 | `A` | State | 0 | 3 | Verified Active Mode on Cockpit Screen |
| `223513` | `Drv Mode Poss`| 1 | `A` | State | 0 | 1 | Mode Switch Clearance (Eco/Norm/Sport) |
| `223541` | `Sport Sw Stat`| 1 | `A` | State | 0 | 1 | Sport Button Physical Contact State |
| `223543` | `Eco Sw Status`| 1 | `A` | State | 0 | 1 | Eco Button Physical Contact State |
| `22350B` | `Gear Sw State`| 1 | `A` | State | 0 | 7 | Shifter Dial / Switch Position (P/R/N/D) |
| `223546` | `MSS Pos Req` | 1 | `A` | State | 0 | 5 | Mode Selector Switch Request to VCU |
| `223547` | `MSS Pos Disp` | 2 | `A*256+B` | Count | 0 | 255 | MSS Backlit Indicator Status |
| `223548` | `MSS Wake Req` | 1 | `A` | State | 0 | 1 | MSS High-Speed Bus Wakeup Request |
| `22350D` | `Cruise Sw St` | 1 | `A` | State | 0 | 3 | Steering Cruise Rocker (+ / - / Cancel) |
| `22350E` | `Cruise State` | 1 | `A` | State | 0 | 5 | Cruise Control State Machine Status |
| `223512` | `Cruise Fn St` | 1 | `A` | State | 0 | 5 | Cruise Functional Availability Layer |
| `223515` | `Set Cruise Sp`| 1 | `A` | $\text{km/h}$ | 0.0 | 140.0 | Target Stored Speed Setpoint |
| `223505` | `ESP Off Cruis`| 1 | `A` | State | 0 | 1 | ESP Overriding & Disabling Cruise |
| `223544` | `Cruise Sw Cnt`| 2 | `A*256+B` | Count | 0 | 1024 | Raw Steering Resistor Ladder ADC Count |
| `22343D` | `Ign Status` | 1 | `A` | State | 0 | 3 | Ignition Switch Key State (OFF/ACC/ON) |
| `22343E` | `Crank Sw Stat`| 1 | `A` | State | 0 | 1 | Push-to-Start / Crank Start Pulse |
| `223457` | `Power Mode` | 1 | `A` | State | 0 | 5 | Overall Vehicle Power Manager State |
| `22345B` | `Veh Status` | 1 | `A` | State | 0 | 5 | Powertrain System Readiness Level |
| `2234BE` | `Wakeup Mode` | 1 | `A` | State | 0 | 7 | VCU Hardware Wakeup Event Trigger |
| `2234BF` | `Supply Volt` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | Terminal 30 Unswitched Battery Voltage |
| `227002` | `ECU Power B` | 1 | `A / 10` | $\text{V}$ | 9.0 | 16.0 | VCU Dual Redundant Power Rail B |
| `223444` | `Keep Alive Rly`| 1 | `A` | State | 0 | 1 | VCU Latched Power Hold Relay State |
| `223446` | `E-Drv Sup Rly`| 1 | `A` | State | 0 | 1 | High-Voltage Inverter 12V Supply Relay |
| `223447` | `Batt Key Rly` | 1 | `A` | State | 0 | 1 | Battery Pack Electronics Switched Relay |
| `223448` | `E-Drv Ign En` | 1 | `A` | State | 0 | 1 | Motor Controller Hardwired Ignition En |
| `223489` | `Crash Status` | 1 | `A` | State | 0 | 1 | Pyrotechnic Airbag Deployment Crash Flag|
| `2234C1` | `Immo Status` | 1 | `A` | State | 0 | 3 | Transponder Key Security Challenge State|
| `22350C` | `TCU Immo Req` | 1 | `A` | State | 0 | 1 | Telematics Remote Immobilizer Inhibit |
| `2234C5` | `HV Crit Alert`| 1 | `A` | State | 0 | 1 | Critical High Voltage Hazard Light |
| `2234C6` | `Limphome Alrt`| 1 | `A` | State | 0 | 1 | Restricted Performance Warning on Dash |
| `2234BD` | `Diag Lamp St` | 1 | `A` | State | 0 | 3 | Check Engine / MIL Indication State |

---

## 4. Decoding Mathematical Formulations

To unpack raw UDS responses from hex frames, parse according to parameter data length ($DL$):

### 4.1 Single-Byte Formulas ($DL = 1$)
For an incoming single-byte payload $A \in [0, 255]$:
* **Linear Voltage Sensor ($0-5\,\text{V}$):**
  $$V = \frac{A}{51} \quad \left(A \times \frac{5.0}{255}\right)$$
* **Tenths Scaling ($0.1$ step):**
  $$X = \frac{A}{10}$$
* **Percentage / Half Step:**
  $$\% = \frac{A}{2}$$
* **Standard Industrial Temperature ($-40^\circ\text{C}$ Offset):**
  $$T = A - 40$$
* **High-Temperature Offset ($-50^\circ\text{C}$ for Pack Cells):**
  $$T_{\text{cell}} = A - 50$$
* **Vacuum Pressure ($-120\,\text{kPa}$ Offset):**
  $$P = A - 120$$

### 4.2 Double-Byte Formulas ($DL = 2$)
For big-endian words where $W = (A \times 256) + B$:
* **Vehicle Velocity:**
  $$\text{Speed} = \left(\frac{W}{32}\right) \times 3.6 = \frac{9 \times W}{80} \quad [\text{km/h}]$$
* **Battery / Inverter Current (Signed with Offset):**
  $$I_{\text{pack}} = \frac{W}{10} - 600 \quad [\text{A}]$$
  $$I_{\text{inv}} = \frac{W}{2} - 510 \quad [\text{A}]$$
* **Motor Torque & Recuperation (Signed with Offset):**
  $$\tau_{\text{motor}} = W - 510 \quad [\text{N}\cdot\text{m}]$$
  $$\tau_{\text{recup}} = \frac{W}{4} - 2048 \quad [\text{N}\cdot\text{m}]$$
* **Motor Rotational Speed (Signed with Offset):**
  $$\text{RPM} = W - 17000 \quad [\text{RPM}]$$
* **Cell & Precise Voltages:**
  $$V_{\text{cell}} = \frac{W}{100} \quad [\text{V}]$$
  $$V_{\text{OBC/DCDC}} = \frac{W}{50} \quad [\text{V}]$$

---

## 5. Python UDS Payload Decoder

Below is an automated parser to ingest raw UDS responses matching these definitions:

```python
from typing import Dict, Any, List

UDS_DEFINITIONS: List[Dict[str, Any]] = [
    # Subset example matching the config layout
    {"CMD": "22341D", "NM": "HV Battery Current", "DL": 2, "calc": lambda A, B: round((A * 256 + B) / 10.0 - 600.0, 2), "UN": "A"},
    {"CMD": "22341E", "NM": "HV Battery Voltage", "DL": 2, "calc": lambda A, B: round((A * 256 + B) / 10.0, 2), "UN": "V"},
    {"CMD": "223421", "NM": "HV Battery SOC", "DL": 2, "calc": lambda A, B: round((A * 256 + B) / 10.0, 1), "UN": "%"},
    {"CMD": "223462", "NM": "Vehicle Speed", "DL": 2, "calc": lambda A, B: round(((A * 256 + B) / 32.0) * 3.6, 2), "UN": "km/h"},
    {"CMD": "223472", "NM": "Motor Winding Temp", "DL": 1, "calc": lambda A, _: A - 40, "UN": "°C"},
    {"CMD": "2234A4", "NM": "Traction Motor RPM", "DL": 2, "calc": lambda A, B: (A * 256 + B) - 17000, "UN": "RPM"},
    {"CMD": "2234A5", "NM": "Traction Motor Torque", "DL": 2, "calc": lambda A, B: (A * 256 + B) - 510, "UN": "Nm"},
]

def decode_uds_response(did_hex: str, raw_data_bytes: bytes) -> Dict[str, Any]:
    """
    Decodes raw positive UDS response payload: 62 [DID High] [DID Low] [Data...]
    """
    for entry in UDS_DEFINITIONS:
        if entry["CMD"] == did_hex:
            if entry["DL"] == 1:
                val = entry["calc"](raw_data_bytes[0], 0)
            elif entry["DL"] == 2:
                val = entry["calc"](raw_data_bytes[0], raw_data_bytes[1])
            else:
                val = list(raw_data_bytes)
            return {
                "parameter": entry["NM"],
                "value": val,
                "unit": entry["UN"]
            }
    return {"error": f"Unknown DID: {did_hex}"}

# Example usage:
# DID 0x341D response with bytes [0x25, 0x80] -> 9600 / 10 - 600 = +360.0 A
result = decode_uds_response("22341D", bytes([0x25, 0x80]))
print(result) # {'parameter': 'HV Battery Current', 'value': 360.0, 'unit': 'A'}
```