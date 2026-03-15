# Steering Wheel Die-Cast 3D Vision Inspection System — Development Specification

> **Purpose**: This document is the single source of truth for Claude Code / Codex to implement the entire C# application. Every interface, protocol detail, register address, database field, and state transition is defined here. Do NOT guess — if something is not specified, ask.

---

## 1. System Overview

### 1.1 Hardware Inventory

| Device | Model | Qty | Communication | C# Role |
|--------|-------|-----|---------------|---------|
| PLC | Siemens S7-200 SMART | 1 | S7.Net (Ethernet) | IO read/write, heartbeat, safety interlock relay |
| Robot | Dobot CR5 (6-axis collaborative) | 1 | TCP Socket ×3 ports | Motion execution, scanning path |
| 3D Scanner | Hikvision MV-DP3580 V2.0 | 1 | Driven by VisionMaster | C# does NOT communicate directly |
| 2D Camera | Hikvision CE200 | 1 | Driven by VisionMaster | C# does NOT communicate directly |
| VisionMaster | Hikvision MVS software | 1 | Modbus TCP (VM=Slave, C#=Master) | Receive measurement data, defect results, QR code |
| Control Panel | 4-button box (center) | 1 | PLC DI | Enable/Reset/AutoStart/AutoStop |
| Station Buttons | 1-button box ×2 | 2 | PLC DI | Station 1 / Station 2 start |
| Indicator Lights | Red/Yellow/Green + Buzzer ×2 | 2 | PLC DO | Status indication per station |
| Safety Light Curtain | — | 1 | PLC DI + CR5 Safety IO (SI4/5) | Protection during auto mode |
| Touch Displays | 1920×1080 ×2 | 2 | — | Display 1: C# app (fullscreen), Display 2: VisionMaster |

### 1.2 Software Architecture

```
[Touch Screen] ← WPF Views ← ViewModels ← Business Services
                                                ↕
                    ┌──────── Communication Services ────────┐
                    │  PlcService    RobotService    VMService │
                    │  (S7.Net)      (TCP Socket)    (Modbus)  │
                    └─────┬─────────────┬──────────────┬──────┘
                          ↓             ↓              ↓
                    S7-200 SMART    Dobot CR5     VisionMaster
                    (IO/Safety)    (Motion)      (3D/2D/QR)
```

### 1.3 Four Inspection Modes

| Mode | Value | Has QR | Inspect A-side | Inspect B-side | QR Action |
|------|-------|--------|----------------|----------------|-----------|
| Single-side with QR | 1 | Yes | Yes | No | First action: robot moves to QR photo point → VM reads QR → validate |
| Single-side no QR | 2 | No | Yes | No | Skip QR, auto-generate ID |
| Dual-side with QR | 3 | Yes | Yes | Yes (if A=OK) | Same as mode 1; if A-side NG → skip B-side, fill zeros |
| Dual-side no QR | 4 | No | Yes | Yes (if A=OK) | Skip QR; if A-side NG → skip B-side, fill zeros |

**Auto-numbering format (no QR)**: `{PartNumber}_{YYYYMMDD}_{0001}` e.g. `FXP001_20260314_0001`

---

## 2. PLC Design (S7-200 SMART)

### 2.1 Digital Inputs (DI)

| PLC IO | Signal | V-area Address | Description |
|--------|--------|----------------|-------------|
| I0.0 | Enable Button (latching, illuminated) | VB100 bit0 | ON=enable request, OFF=disable request |
| I0.1 | Reset Button (momentary, yellow) | VB100 bit1 | Rising edge = clear alarm + restore light curtain pause |
| I0.2 | Auto Start Button (momentary, green) | VB100 bit2 | Rising edge = enter auto mode |
| I0.3 | Auto Stop Button (momentary, red) | VB100 bit3 | Rising edge = exit auto mode |
| I0.4 | Station 1 Start (momentary, green) | VB100 bit4 | Rising edge = trigger station 1 inspection |
| I0.5 | Station 2 Start (momentary, green) | VB100 bit5 | Rising edge = trigger station 2 inspection |
| I0.6 | Safety Light Curtain | VB100 bit6 | 1=normal, 0=blocked |
| I0.7 | CR5 E-Stop Button Status | VB100 bit7 | 1=normal, 0=e-stop active (NC contact) |
| I1.0 | CR5 Running Status (SO3) | VB101 bit0 | 1=stationary, 0=in motion |
| I1.1 | Station 1 Workpiece Sensor | VB101 bit1 | 1=workpiece present |
| I1.2 | Station 2 Workpiece Sensor | VB101 bit2 | 1=workpiece present |

### 2.2 Digital Outputs (DO)

| PLC IO | Signal | V-area Address | Description |
|--------|--------|----------------|-------------|
| Q0.0 | Station 1 Red Light | VB200 bit0 | NG / error / QR mismatch |
| Q0.1 | Station 1 Yellow Light | VB200 bit1 | Waiting for placement |
| Q0.2 | Station 1 Green Light | VB200 bit2 | OK / pass |
| Q0.3 | Station 2 Red Light | VB200 bit3 | |
| Q0.4 | Station 2 Yellow Light | VB200 bit4 | |
| Q0.5 | Station 2 Green Light | VB200 bit5 | |
| Q0.6 | Buzzer | VB200 bit6 | Alarm sound |
| Q0.7 | Enable Button Lamp | VB200 bit7 | Lit when robot enabled |
| Q1.0 | CR5 Protective Stop Control | VB201 bit0 | → SI4/SI5; LOW=pause robot |
| Q1.1 | CR5 Reset Signal | VB201 bit1 | → SI10; 500ms pulse to resume |

### 2.3 C# → PLC Control Variables

| V-area | Name | Direction | Description |
|--------|------|-----------|-------------|
| VB200 | Light command byte | C# writes | bit0-7 → Q0.0-Q0.7 |
| VB201 bit0 | Mode flag | C# writes | 0=manual (light curtain disabled), 1=auto (light curtain enabled) |
| VB250 bit0 | Heartbeat | C# writes | C# toggles every 500ms; PLC watchdog: 5s no change → cut enable relay Q0.7 + buzzer |

### 2.4 PLC Ladder Logic (Critical)

**Light curtain logic (PLC local)**:
```
VB201.0=1 (auto mode) AND I1.0=0 (CR5 in motion) AND I0.6=0 (curtain blocked)
  → Q1.0 = LOW (trigger CR5 protective stop via SI4/SI5)
```
**Manual mode (VB201.0=0)**: Light curtain completely disabled. Engineer can freely approach robot.

**Reset button (I0.1 rising edge)**: Q1.1 outputs 500ms HIGH pulse → CR5 SI10 rising edge → resume.

**Watchdog**: VB250.0 unchanged for 5 seconds → Q0.7=0 (cut enable relay) + Q0.6=1 (buzzer).

### 2.5 S7.Net Connection Parameters

```csharp
CpuType = CpuType.S7200Smart  // NOT S7200
IP = from appsettings.json     // e.g. "192.168.2.1"
Rack = 0
Slot = 1
```

**Batch read optimization**: Read VB100-VB101 (2 bytes) in one call every 100ms. Write VB200-VB201 (2 bytes) as needed. Toggle VB250.0 every 500ms.

---

## 3. Dobot CR5 Robot TCP/IP Protocol

### 3.1 Three TCP Connections

| Port | Purpose | Protocol | C# Implementation |
|------|---------|----------|-------------------|
| 29999 | Dashboard: enable/status/IO/speed | ASCII request-response | TcpClient + StreamReader/Writer |
| 30003 | Motion: MovJ/MovL/Sync/MoveJog | ASCII request-response | TcpClient + StreamReader/Writer |
| 30004 | Real-time feedback: 1440 bytes | Binary continuous push | TcpClient + background Task loop |

### 3.2 Command Format

**Send**: `CommandName(Param1,Param2,...ParamN)` — ASCII string, case-insensitive  
**Receive**: `ErrorID,{value1,...,valueN},CommandName(Params);`  
- `ErrorID=0` → success
- `ErrorID!=0` → error (see error code table)
- `{...}` → return values; `{}` if none

### 3.3 Complete Command List Used in This System

| Command | Port | Parameters | Return | Purpose |
|---------|------|------------|--------|---------|
| `PowerOn()` | 29999 | none | ErrorID | Power on robot (wait ~10s) |
| `EnableRobot()` | 29999 | none | ErrorID | Enable robot |
| `DisableRobot()` | 29999 | none | ErrorID | Disable robot |
| `ClearError()` | 29999 | none | ErrorID | Clear alarms |
| `ResetRobot()` | 29999 | none | ErrorID | Stop + clear motion queue |
| `RobotMode()` | 29999 | none | `{mode}` | Returns 1-11 (see 3.4) |
| `SpeedFactor(ratio)` | 29999 | ratio: 1-100 | ErrorID | Set global speed % |
| `GetPose()` | 29999 | none | `{X,Y,Z,Rx,Ry,Rz}` | Get current TCP pose (Cartesian) |
| `GetAngle()` | 29999 | none | `{J1,J2,J3,J4,J5,J6}` | Get current joint angles |
| `DO(index,status)` | 29999 | index, 0/1 | ErrorID | Set DO (queued) |
| `DOExecute(index,status)` | 29999 | index, 0/1 | ErrorID | Set DO (immediate) |
| `DI(index)` | 29999 | index | `{value}` | Read DI |
| `MovJ(X,Y,Z,Rx,Ry,Rz,SpeedJ=R)` | 30003 | coords + speed | ErrorID | Joint motion to Cartesian point |
| `MovL(X,Y,Z,Rx,Ry,Rz,SpeedL=R)` | 30003 | coords + speed | ErrorID | Linear motion to Cartesian point |
| `JointMovJ(J1,J2,J3,J4,J5,J6,SpeedJ=R)` | 30003 | joints + speed | ErrorID | Joint motion to joint target |
| `Sync()` | 30003 | none | ErrorID | Block until motion complete |
| `MoveJog(axisID)` | 30003 | "J1+"/"X-" etc. | ErrorID | Continuous jog |
| `MoveJog()` | 30003 | no params | ErrorID | Stop jog |
| `RelMovJUser(oX,oY,oZ,oRx,oRy,oRz,User)` | 30003 | offsets + coord sys | ErrorID | Relative move (step mode) |

### 3.4 RobotMode Values

| Value | Name | Description |
|-------|------|-------------|
| 1 | Init | Initializing |
| 2 | BrakeOpen | Brake released |
| 3 | PowerOff | Not powered |
| 4 | Disabled | Disabled (not enabled) |
| **5** | **Enabled** | **Enabled and idle ★ Ready state** |
| 6 | Backdrive | Drag mode |
| **7** | **Running** | **Executing motion** |
| 8 | Recording | Trajectory recording |
| **9** | **Error** | **Has alarm ★ Needs ClearError()** |
| 10 | Pause | Paused |
| 11 | Jog | Jogging |

### 3.5 Real-time Feedback (30004 Port) — 1440 Bytes

All values are **little-endian**.

| Byte Offset | Data Type | Field | Description |
|-------------|-----------|-------|-------------|
| 24-31 | uint64 | RobotMode | Robot mode (1-11) |
| 64-71 | double | SpeedScaling | Speed scaling factor |
| 432-479 | double×6 | QActual[6] | Actual joint angles J1-J6 (degrees) |
| 624-671 | double×6 | ToolVectorActual[6] | Actual TCP pose X,Y,Z,Rx,Ry,Rz (mm/degrees) |
| 864-911 | double×6 | MotorTemperatures[6] | Joint temperatures (°C) |
| 1025 | byte | BrakeStatus | Brake status bitmask |
| 1026 | byte | EnableStatus | Enable status |
| 1028 | byte | RunningStatus | Running status |
| 1029 | byte | ErrorStatus | Error status (!=0 means alarm) |

**C# parsing example**:
```csharp
double x = BitConverter.ToDouble(buffer, 624);  // TCP X
double j1 = BitConverter.ToDouble(buffer, 432); // Joint 1
ulong mode = BitConverter.ToUInt64(buffer, 24);  // RobotMode
byte error = buffer[1029];                       // Error flag
```

### 3.6 Response Parsing

```csharp
// Input: "0,{1,2,3},GetPose(...);"
// Parse: ErrorID = 0, Values = ["1","2","3"]
public static RobotResponse Parse(string raw)
{
    int firstComma = raw.IndexOf(',');
    int errorId = int.Parse(raw[..firstComma]);
    int braceStart = raw.IndexOf('{');
    int braceEnd = raw.IndexOf('}');
    string valueStr = raw[(braceStart+1)..braceEnd];
    string[] values = string.IsNullOrEmpty(valueStr) 
        ? Array.Empty<string>() 
        : valueStr.Split(',');
    return new RobotResponse(errorId, values, raw);
}
```

---

## 4. VisionMaster Modbus TCP Register Map

**Connection**: C#=Master, VisionMaster=Slave  
**IP**: 127.0.0.1 (same machine) or actual IP  
**Port**: 502  
**Unit ID**: 1  
**Float32**: 2 registers, Big-Endian byte order

> ⚠️ **VisionMaster must be configured exactly per this table.**

### 4.1 Control Registers (C# writes → VM reads) — Holding Registers

| Address | Name | Type | Values | Description |
|---------|------|------|--------|-------------|
| 40001 | CMD_Trigger | UINT16 | 0=idle, 1=3D scan station 1, 2=3D scan station 2, 3=2D photo station 1, 4=2D photo station 2, 5=QR code read | C# writes non-zero to trigger; writes 0 to reset |
| 40002 | CMD_PhotoIndex | UINT16 | 1-10 | Current photo point index (used with CMD=3 or 4) |
| 40003 | CMD_RecipeID | UINT16 | Recipe ID | Reserved: switch VM recipe |
| 40004 | CMD_Heartbeat | UINT16 | 0/1 alternating | C# heartbeat, toggle every 500ms |

### 4.2 Status Registers (VM writes → C# reads) — Holding Registers

| Address | Name | Type | Values | Description |
|---------|------|------|--------|-------------|
| 40101 | STA_Status | UINT16 | 0=idle, 1=executing, 2=complete OK, 3=complete NG, 9=error | VM execution status |
| 40102 | STA_Heartbeat | UINT16 | 0/1 alternating | VM heartbeat |
| 40103 | STA_ErrorCode | UINT16 | error code | VM error code |

### 4.3 3D Measurement Results (VM writes → C# reads)

| Address | Name | Type | Description |
|---------|------|------|-------------|
| 40201-40202 | MEAS_3D_01 | Float32 | 1st measurement value (e.g. Z1 height) |
| 40203-40204 | MEAS_3D_02 | Float32 | 2nd measurement value |
| 40205-40206 | MEAS_3D_03 | Float32 | 3rd |
| ... | ... | Float32 | Sequential |
| 40239-40240 | MEAS_3D_20 | Float32 | 20th (max 20 reserved) |
| 40241 | MEAS_3D_Count | UINT16 | Actual number of measurements this scan |

### 4.4 2D Inspection Results (VM writes → C# reads)

| Address | Name | Type | Description |
|---------|------|------|-------------|
| 40301 | MEAS_2D_Result | UINT16 | Current photo point: 1=OK, 0=NG |
| 40302 | MEAS_2D_DefectCode | UINT16 | Defect type code (0=no defect) |
| 40303-40312 | MEAS_2D_Batch[10] | UINT16×10 | Summary results for photo points 1-10 |

### 4.5 QR Code Results (VM writes → C# reads)

| Address | Name | Type | Description |
|---------|------|------|-------------|
| 40401-40420 | QR_Content | UINT16×20 | ASCII encoded, 2 chars per register, max 40 chars |
| 40421 | QR_Length | UINT16 | Valid character count |
| 40422 | QR_ReadOK | UINT16 | 1=read success, 0=read failed |

### 4.6 Interaction Sequence (MUST follow strictly)

```
Step 1: C# writes CMD_Trigger = N (1-5) + CMD_PhotoIndex (if applicable)
Step 2: C# polls STA_Status every 100ms, waiting for value to change from 1 to 2 or 3
Step 3: If STA_Status=2 (OK): C# reads corresponding result registers
        If STA_Status=3 (NG): C# records NG
        If timeout 15s: C# declares VM communication error → current inspection NG → alarm
Step 4: C# writes CMD_Trigger = 0 (reset)
Step 5: VM detects CMD_Trigger=0 → resets STA_Status to 0 (idle)
Step 6: C# confirms STA_Status=0 → interaction complete, ready for next trigger
```

---

## 5. Database Schema (SQL Server + EF Core 8)

**Database name**: `SWI_InspectionDB`  
**Approach**: EF Core Code-First with migrations

### 5.1 T_User

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| Id | int | INT PK IDENTITY | |
| WorkerId | string | VARCHAR(20) UNIQUE | Employee ID |
| WorkerName | string | NVARCHAR(50) | Name |
| Password | string | VARCHAR(100) | Hashed password |
| Role | string | VARCHAR(20) | "Admin" / "Engineer" / "Leader" / "Operator" |
| IsActive | bool | BIT | Account active flag |

### 5.2 T_Recipe

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| RecipeId | int | INT PK IDENTITY | |
| PartNumber | string | VARCHAR(50) UNIQUE | Part number (unique identifier) |
| PartName | string | NVARCHAR(100) | Part name |
| DetectMode | int | INT | 1=single+QR, 2=single-noQR, 3=dual+QR, 4=dual-noQR |
| HasQRCode | bool | BIT | Whether QR code exists |
| QRCodePrefix | string | VARCHAR(50) | QR matching prefix (required when HasQRCode=true) |
| MeasureCount_A | int | INT | A-side 3D measurement count |
| MeasureCount_B | int | INT | B-side 3D measurement count |
| PhotoCount_A | int | INT | A-side 2D photo point count (0-10) |
| PhotoCount_B | int | INT | B-side 2D photo point count (0-10) |
| PointsReady | bool | BIT | Teaching complete flag |
| CalibReady | bool | BIT | Calibration complete flag |
| CreateTime | DateTime | DATETIME | |
| UpdateTime | DateTime | DATETIME | |

**Production-ready condition**: `PointsReady=true AND CalibReady=true`

### 5.3 T_Standard (imported from Excel)

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| Id | int | INT PK IDENTITY | |
| RecipeId | int | INT FK→T_Recipe | |
| PointIndex | int | INT | Measurement index (1-N) |
| PointName | string | NVARCHAR(50) | e.g. "Flange Z Height" |
| Side | string | CHAR(1) | "A" or "B" |
| MeasureAxis | string | CHAR(1) | "X", "Y", or "Z" |
| NominalValue | double | FLOAT | Nominal value (mm) |
| TolUpper | double | FLOAT | Upper tolerance (mm) |
| TolLower | double | FLOAT | Lower tolerance (mm) |
| VM_RegisterIndex | int | INT | Maps to MEAS_3D_XX (1-20) |

### 5.4 T_CalibOffset

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| Id | int | INT PK IDENTITY | |
| RecipeId | int | INT FK | |
| PointIndex | int | INT | Maps to T_Standard.PointIndex |
| Side | string | CHAR(1) | |
| OffsetValue | double | FLOAT | Compensation = scan_avg - nominal |
| CalibCount | int | INT | Number of calibration runs averaged |

### 5.5 T_RobotPoint (saved via Teaching UI)

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| Id | int | INT PK IDENTITY | |
| RecipeId | int | INT FK | |
| Side | string | CHAR(1) | "A" or "B" |
| PointType | string | VARCHAR(20) | See below |
| SeqOrder | int | INT | Execution order (1=first) |
| X | double | FLOAT | mm |
| Y | double | FLOAT | mm |
| Z | double | FLOAT | mm |
| Rx | double | FLOAT | degrees |
| Ry | double | FLOAT | degrees |
| Rz | double | FLOAT | degrees |

**PointType values**:
- `SCAN_A` — 3D scan start point
- `SCAN_B` — 3D scan end point
- `QR_PHOTO` — QR code photo point (only for has-QR modes, SeqOrder=1)
- `PHOTO_01` through `PHOTO_10` — 2D inspection photo points

### 5.6 T_DetectResult

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| RecordId | long | BIGINT PK IDENTITY | |
| QRCode | string | VARCHAR(100) | QR content or auto-generated ID |
| PartNumber | string | VARCHAR(50) | |
| WorkerId | string | VARCHAR(20) | |
| DetectTime | DateTime | DATETIME | |
| DetectMode | int | INT | |
| ResultA | int | INT | 1=OK, 0=NG, -1=not inspected |
| ResultB | int | INT | 1=OK, 0=NG, -1=not inspected, 0=skipped(A-side NG) |
| Result2D_A | int | INT | A-side 2D aggregate |
| Result2D_B | int | INT | B-side 2D aggregate |
| OverallResult | int | INT | 1=OK, 0=NG |
| CycleTime | double | FLOAT | Inspection duration (seconds) |

### 5.7 T_DetectDetail

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| DetailId | long | BIGINT PK IDENTITY | |
| RecordId | long | BIGINT FK | |
| PointIndex | int | INT | |
| Side | string | CHAR(1) | |
| PointName | string | NVARCHAR(50) | |
| MeasureAxis | string | CHAR(1) | |
| NominalValue | double | FLOAT | |
| ActualValue | double | FLOAT | Corrected value (after calibration offset) |
| Deviation | double | FLOAT | ActualValue - NominalValue |
| TolUpper | double | FLOAT | |
| TolLower | double | FLOAT | |
| IsOK | bool | BIT | TolLower ≤ Deviation ≤ TolUpper |

### 5.8 T_AlarmLog

| Column | C# Type | DB Type | Description |
|--------|---------|---------|-------------|
| Id | long | BIGINT PK IDENTITY | |
| Time | DateTime | DATETIME | |
| Level | string | VARCHAR(20) | "Emergency" / "Fault" / "Warning" / "Info" |
| Source | string | VARCHAR(50) | "PLC" / "Robot" / "VM" / "System" |
| Message | string | NVARCHAR(500) | |
| IsAcknowledged | bool | BIT | |
| AckBy | string | VARCHAR(20) | Worker ID who acknowledged |
| AckTime | DateTime? | DATETIME NULL | |

---

## 6. Global Configuration (appsettings.json)

```json
{
  "Plc": {
    "IP": "192.168.2.1",
    "Rack": 0,
    "Slot": 1,
    "PollingMs": 100
  },
  "Robot": {
    "IP": "192.168.5.1",
    "DashboardPort": 29999,
    "MotionPort": 30003,
    "FeedbackPort": 30004,
    "ScanSpeed": 20,
    "PhotoMoveSpeed": 40,
    "TeachMaxSpeed": 30,
    "Home": { "X": 0, "Y": 0, "Z": 0, "Rx": 0, "Ry": 0, "Rz": 0 }
  },
  "VisionMaster": {
    "IP": "127.0.0.1",
    "Port": 502,
    "UnitId": 1,
    "TimeoutMs": 15000
  },
  "Database": {
    "ConnectionString": "Server=.;Database=SWI_InspectionDB;Trusted_Connection=true;TrustServerCertificate=true;"
  }
}
```

- `ScanSpeed` = MovL speed % for 3D scanning (fixed for all products)
- `PhotoMoveSpeed` = MovJ speed % for 2D photo point moves (fixed)
- `TeachMaxSpeed` = max speed allowed in teaching mode (safety)
- `Home` = global HOME position shared by all recipes

---

## 7. State Machine (Complete Definition)

### 7.1 States

```csharp
public enum InspectionState
{
    Idle = 0,            // Waiting for station start button
    QRCapture = 1,       // Robot to QR point + VM reads QR
    QRValidation = 2,    // Validate QR prefix match
    MoveToScanA = 3,     // Robot MovJ to 3D scan start point
    Scanning3D = 4,      // MovL A→B (constant speed) + VM scanning simultaneously
    Read3DData = 5,      // Read 3D measurement data from Modbus
    Photo2DLoop = 6,     // Loop: MovJ to each photo point + VM capture
    ReturnHome = 7,      // Robot MovJ to HOME
    CalcResult = 8,      // Tolerance calculation + overall judgment
    ShowResult = 9,      // Display result + save to DB + wait for part removal
    QRMismatch = 10,     // QR code does not match recipe prefix
    Error = 99           // System error
}
```

### 7.2 Complete Action Sequence (Dual-side with QR = most complex mode)

| Step | State | C# Logic | Hardware Action | Next |
|------|-------|----------|-----------------|------|
| 1 | S0: Idle | Read PLC VB100.4 (station 1 button rising edge) | Wait for button | HasQR→S1, NoQR→S3 |
| 2 | S1: QRCapture | Read QR_PHOTO point from T_RobotPoint. TCP: `MovJ(qr_point, speed)` → `Sync()`. Modbus: write CMD_Trigger=5. Poll STA_Status until =2. Read QR_Content registers. Decode ASCII from UINT16 array. Modbus: write CMD_Trigger=0. | Robot moves to QR point. 2D camera captures image and reads QR. | Success→S2. Read fail→S10 |
| 3 | S2: QRValidation | Compare: `qrString.StartsWith(recipe.QRCodePrefix)` | — | Match→S3. No match→S10 |
| 4 | S3: MoveToScanA | Read SCAN_A point from T_RobotPoint. TCP: `MovJ(scanA, PhotoMoveSpeed)` → `Sync()` | Robot moves to 3D scan start | Arrived→S4 |
| 5 | S4: Scanning3D | Read SCAN_B point from T_RobotPoint. Modbus: write CMD_Trigger=1 (station 1) or 2 (station 2). TCP: `MovL(scanB, ScanSpeed)`. Simultaneously poll STA_Status. Wait for BOTH: MovL complete (Sync returns) AND STA_Status=2. | Robot linear motion A→B at constant speed. 3D scanner acquires. | Both complete→S5. VM error/timeout→S99 |
| 6 | S5: Read3DData | Modbus: read 40241 (count). Read 40201-40240 (Float32 values). Store in memory array. Modbus: write CMD_Trigger=0 and confirm STA_Status returns to 0. | — | →S6 |
| 7 | S6: Photo2DLoop | For i=1 to PhotoCount: Read PHOTO_0i from T_RobotPoint. TCP: `MovJ(point, PhotoMoveSpeed)` → `Sync()`. Modbus: write CMD_Trigger=3(st1) or 4(st2), CMD_PhotoIndex=i. Poll STA_Status until =2 or =3. Read 40301 (OK/NG). Modbus: CMD_Trigger=0. Store result in array. | Robot moves to each photo point sequentially. 2D camera captures at each point. | All done→S7. Any timeout→S99 |
| 8 | S7: ReturnHome | TCP: `MovJ(Home, PhotoMoveSpeed)` → `Sync()` | Robot returns HOME | →S8 |
| 9 | S8: CalcResult | For each measurement point: `corrected = vmValue - calibOffset`. `deviation = corrected - nominal`. `ok = tolLower <= deviation <= tolUpper`. Aggregate: all3DOK && all2DOK → OverallOK | — | →S9 |
| 10 | S9: ShowResult | Write PLC lights (green if OK, red if NG). Write to DB (T_DetectResult + T_DetectDetail). Update UI (ResultView). Increment counters. Wait for workpiece sensor VB101.1=0 (part removed). Then clear lights, clear QR, reset state. | Lights on. Buzzer if NG. | Part removed→S0. Dual mode & A-OK→activate B-side S0 |
| 11 | S10: QRMismatch | Write PLC: red light. Do NOT save to database. Wait for Reset button (VB100.1 rising edge). | Red light | Reset→S0 |
| 12 | S99: Error | Write PLC: red light + buzzer. Write alarm log to DB. Wait for Reset button. | Red light + buzzer | Reset→S0 |

### 7.3 Mode Differences

| Aspect | Mode 1 (single+QR) | Mode 2 (single-noQR) | Mode 3 (dual+QR) | Mode 4 (dual-noQR) |
|--------|--------------------|-----------------------|-------------------|---------------------|
| S0 next | →S1 (QR) | →S3 (skip QR) | →S1 (QR) | →S3 (skip QR) |
| QR capture | Execute | Skip | Execute | Skip |
| A-side NG | Final NG | Final NG | Skip B-side, B=all zeros, Final NG | Skip B-side, B=all zeros, Final NG |
| B-side | Not executed | Not executed | Execute if A=OK | Execute if A=OK |
| DB identifier | QR code string | Auto-number | QR code string | Auto-number |

### 7.4 Dual-side B-side Skip Logic (A-side NG)

When A-side result is NG in dual mode:
```csharp
// Do NOT activate B-side state machine
// Fill B-side detail records with zeros:
foreach (var std in bSideStandards)
{
    details.Add(new DetectDetail {
        PointIndex = std.PointIndex,
        Side = "B",
        ActualValue = 0,
        Deviation = 0,
        IsOK = false
    });
}
result.ResultB = 0; // NG
result.OverallResult = 0; // NG
```

---

## 8. Teaching System

### 8.1 Prerequisites

- User role ≥ Engineer
- System in manual mode (AutoMode=false)
- Robot enabled (RobotMode=5)
- On entering Teaching page: `SpeedFactor(TeachMaxSpeed)` + write PLC VB201.0=0 (manual mode, disables light curtain)
- On leaving Teaching page: restore speed, keep manual mode unless entering auto

### 8.2 Jog Modes

| Mode | Trigger | C# Command | Use Case |
|------|---------|------------|----------|
| Continuous | PointerPressed → send, PointerReleased → stop | `MoveJog("J1+")` / `MoveJog("")` | Coarse positioning |
| Step | Single click | `RelMovJUser(stepSize,0,0,0,0,0,0)` + `Sync()` | Fine adjustment |

**Step size dropdown options**: 0.1 / 0.5 / 1.0 / 5.0 / 10.0 (mm for Cartesian, degrees for joints)

### 8.3 Record Point Flow

1. Engineer jogs robot to desired position
2. Clicks [Record Point] button
3. Inline panel expands (not modal dialog — touch friendly):
   - PointType dropdown: SCAN_A / SCAN_B / QR_PHOTO / PHOTO_01-10
   - Current coordinates displayed (read-only)
   - [Confirm Save] [Cancel] buttons
4. On confirm: C# sends `GetPose()` for precise coordinates → saves to T_RobotPoint → refreshes list
5. SeqOrder auto-incremented

### 8.4 Point List Operations

| Button | Action |
|--------|--------|
| [Move To Selected] | `MovJ(selectedPoint)` → `Sync()` — verify point is correct |
| [Overwrite Selected] | Replace selected point's coordinates with current robot position |
| [Delete Selected] | Delete with confirmation dialog ("Delete PHOTO_03?") |
| [▲ Move Up] | Decrease SeqOrder (swap with previous) |
| [▼ Move Down] | Increase SeqOrder (swap with next) |
| [Test Full Path] | Sequential MovJ to all points (by SeqOrder), 2s pause each, no VM trigger. Shows progress "Testing point 3/8...". Can stop with [Stop] button (sends `ResetRobot()`). |

### 8.5 Calibration (Independent from Teaching)

**Calibration = establishing mapping between VM scan values and CMM true values.**

Steps:
1. Select recipe + side
2. Place calibration standard part on station
3. Click [Execute Calibration Scan] — runs full 3D scan sequence (same as production)
4. C# reads VM measurement values, compares each to T_Standard nominal values
5. Offset = VM_value - NominalValue for each point
6. Repeat 3-5 times, calculate average offset
7. Engineer clicks [Save Calibration] → writes to T_CalibOffset
8. Click [Mark as Calibrated] → sets T_Recipe.CalibReady=true

**Teaching and calibration are completely independent. No dependency between them.**  
**Recipe is production-ready only when: PointsReady=true AND CalibReady=true**

---

## 9. Connection Management & Watchdog

### 9.1 Startup Sequence

```
App starts → ① Connect PLC (S7.Net) → start heartbeat
           → ② Connect VisionMaster (Modbus TCP) → check VM heartbeat
           → ③ Connect CR5 (TCP ×3 ports) → read RobotMode
           → All OK → Login page ready
           → Any fail → show which device failed → retry every 3s → block production until all connected
```

### 9.2 Disconnection Handling

| Device | Detection | Behavior | Reconnect |
|--------|-----------|----------|-----------|
| PLC | S7.Net read/write exception | Red popup "PLC Disconnected!". All operations locked. State machine paused. | Auto retry every 3s. Auto resume on success. |
| CR5 | Socket exception / Read returns 0 | Red popup "Robot Disconnected!". PLC watchdog cuts enable. State machine paused. | Auto retry every 3s (all 3 ports). Requires Reset button press after reconnect. |
| VisionMaster | Modbus connection error / heartbeat 40102 unchanged 5s | Orange popup "VM Disconnected!". Current inspection → NG. Manual operations still work. | Auto retry every 3s. Continue on success. |

**Universal rule**: Any disconnection → state machine pauses → after reconnect + Reset button → current workpiece re-inspected from beginning.

### 9.3 Watchdog

- C# toggles PLC VB250.0 every 500ms via S7.Net
- PLC monitors: 5s no change → cut Q0.7 (enable relay) + activate buzzer
- C# monitors VM heartbeat register 40102: 5s no change → alarm

---

## 10. Solution Structure

```
SteeringWheelInspection.sln
├── src/
│   ├── SWI.App/                           # WPF Application (.NET 8)
│   │   ├── App.xaml / App.xaml.cs          # DI container + startup
│   │   ├── MainWindow.xaml/.cs             # Main window (nav + status bar)
│   │   ├── Views/
│   │   │   ├── LoginView.xaml
│   │   │   ├── OperationView.xaml
│   │   │   ├── ResultView.xaml
│   │   │   ├── RecipeView.xaml
│   │   │   ├── TeachingView.xaml
│   │   │   ├── CalibrationView.xaml
│   │   │   ├── DataQueryView.xaml
│   │   │   ├── SettingsView.xaml
│   │   │   └── AlarmView.xaml
│   │   ├── ViewModels/
│   │   │   ├── MainViewModel.cs
│   │   │   ├── LoginViewModel.cs
│   │   │   ├── OperationViewModel.cs
│   │   │   ├── ResultViewModel.cs
│   │   │   ├── RecipeViewModel.cs
│   │   │   ├── TeachingViewModel.cs
│   │   │   ├── CalibrationViewModel.cs
│   │   │   ├── DataQueryViewModel.cs
│   │   │   ├── SettingsViewModel.cs
│   │   │   └── AlarmViewModel.cs
│   │   ├── Controls/
│   │   │   ├── TrafficLight.xaml           # Red/Yellow/Green indicator
│   │   │   ├── ConnectionIndicator.xaml    # Device connection status dot
│   │   │   └── BigResultBanner.xaml        # OK/NG large display
│   │   ├── Behaviors/
│   │   │   └── VirtualKeyboardBehavior.cs  # TabTip.exe auto-invoke
│   │   ├── Converters/
│   │   │   ├── BoolToColorConverter.cs
│   │   │   ├── StateToTextConverter.cs
│   │   │   └── ModeToStringConverter.cs
│   │   └── Themes/
│   │       └── Styles.xaml                 # Global styles + colors
│   │
│   ├── SWI.Communication/                  # Communication Layer (.NET 8 Class Library)
│   │   ├── PLC/
│   │   │   ├── IPlcService.cs
│   │   │   ├── SmartPlcService.cs
│   │   │   └── PlcDataModel.cs
│   │   ├── Robot/
│   │   │   ├── IRobotService.cs
│   │   │   ├── DobotCR5Service.cs
│   │   │   ├── RobotCommandBuilder.cs
│   │   │   ├── RobotResponseParser.cs
│   │   │   ├── RobotFeedbackParser.cs
│   │   │   └── RobotModels.cs
│   │   ├── Vision/
│   │   │   ├── IVisionMasterService.cs
│   │   │   └── VisionMasterModbusService.cs
│   │   └── Common/
│   │       ├── ModbusTcpClient.cs
│   │       └── ConnectionWatchdog.cs
│   │
│   ├── SWI.Business/                       # Business Logic Layer
│   │   ├── StateMachine/
│   │   │   ├── InspectionStateMachine.cs
│   │   │   └── InspectionState.cs
│   │   ├── Services/
│   │   │   ├── InspectionService.cs
│   │   │   ├── CalibrationService.cs
│   │   │   ├── ToleranceCalculator.cs
│   │   │   ├── RecipeService.cs
│   │   │   ├── AlarmService.cs
│   │   │   ├── UserService.cs
│   │   │   └── ExcelImporter.cs
│   │   └── Models/
│   │
│   ├── SWI.Data/                           # Data Access Layer
│   │   ├── AppDbContext.cs
│   │   ├── Entities/
│   │   ├── Repositories/
│   │   └── Migrations/
│   │
│   └── SWI.Common/                         # Shared Utilities
│       ├── Configuration/AppSettings.cs
│       ├── Logging/
│       └── Events/EventAggregator.cs
│
├── appsettings.json
├── appsettings.Development.json
└── docs/
```

### 10.1 NuGet Packages

| Package | Project | Purpose |
|---------|---------|---------|
| S7netplus | SWI.Communication | S7-200 SMART PLC communication |
| FluentModbus (or NModbus4) | SWI.Communication | Modbus TCP Master |
| Microsoft.EntityFrameworkCore | SWI.Data | ORM |
| Microsoft.EntityFrameworkCore.SqlServer | SWI.Data | SQL Server provider |
| Microsoft.EntityFrameworkCore.Tools | SWI.Data | Migration tooling |
| CommunityToolkit.Mvvm | SWI.App | MVVM (ObservableObject, RelayCommand) |
| Microsoft.Extensions.Hosting | SWI.App | DI + config + logging host |
| Serilog.Extensions.Hosting | SWI.Common | Structured logging |
| Serilog.Sinks.File | SWI.Common | Log to file |
| ClosedXML | SWI.Business | Excel import/export |
| LiveChartsCore.SkiaSharpView.WPF | SWI.App | Charts (calibration trends) |

### 10.2 Project References

```
SWI.App → SWI.Communication, SWI.Business, SWI.Data, SWI.Common
SWI.Business → SWI.Communication (interfaces only), SWI.Data, SWI.Common
SWI.Communication → SWI.Common
SWI.Data → SWI.Common
```

---

## 11. Development Phases

| Phase | Task | Key Files | Acceptance Criteria |
|-------|------|-----------|---------------------|
| P0 | Create Solution + 5 Projects + NuGet + references | *.csproj, .sln | Compiles successfully |
| P1 | EF Core entities + AppDbContext + Migration | Entities/, AppDbContext | Database created with all 8 tables |
| P2 | SmartPlcService (S7.Net) | IPlcService, SmartPlcService | Read VB100, write VB200, heartbeat works |
| P3 | DobotCR5Service (TCP ×3) | IRobotService, CR5Service, Parsers | Connect + enable + read pose + MovJ + Sync |
| P4 | VisionMasterModbusService | IVMService, ModbusTcpClient | Trigger + read results + heartbeat |
| P5 | MainWindow + navigation + StatusBar | MainWindow.xaml, MainVM | Page switching + status bar live |
| P6 | LoginView + dropdowns + recipe loading | LoginView, LoginVM, UserService | Login + select product + load recipe |
| P7 | OperationView + traffic lights + guidance | OperationView, OpVM, TrafficLight | Dual-station live display |
| P8 | InspectionStateMachine | StateMachine, InspectionService | All 4 modes state transitions correct |
| P9 | TeachingView + jog + record points | TeachingView, TeachVM | Continuous + step jog, save points to DB |
| P10 | RecipeView + Excel import | RecipeView, RecipeVM, ExcelImporter | Full recipe CRUD |
| P11 | ToleranceCalculator + result saving | Calculator, ResultRepository | Calculation correct + saved to DB |
| P12 | ResultView + DataGrid + OK/NG banner | ResultView, ResultVM, BigResultBanner | Results display with NG red highlight |
| P13 | CalibrationView + calibration flow | CalibView, CalibService | Calibrate + save offsets |
| P14 | Connection management + watchdog | ConnectionWatchdog, error handling | Disconnect alarm + auto reconnect |
| P15 | AlarmView + AlarmService | AlarmView, AlarmService | Alarm logging + acknowledge |
| P16 | DataQueryView + Excel export | QueryView, QueryVM | Query + export |
| P17 | SettingsView + user management + virtual keyboard | SettingsView, UserService | All settings functional |
| P18 | Full integration testing | — | All 4 modes end-to-end pass |
