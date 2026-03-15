# UI Design Specification — Dark Tech Theme

> **Purpose**: Pixel-level UI spec for Claude Code to generate WPF XAML + Styles. Every color, size, spacing, animation, and layout is defined here. Follow exactly.

---

## 1. Design System

### 1.1 Color Palette — Primary

| Token | Hex | RGB | Usage |
|-------|-----|-----|-------|
| `BgPrimary` | `#0D1B2A` | 13,27,42 | Page background, main window |
| `BgCard` | `#1B2838` | 27,40,56 | Card/panel backgrounds |
| `BgCardHover` | `#233044` | 35,48,68 | Card hover, selected item bg |
| `BgNav` | `#0A1628` | 10,22,40 | Top navigation bar |
| `BgStatus` | `#060F1A` | 6,15,26 | Bottom status bar |
| `Accent` | `#00D4FF` | 0,212,255 | Icons, highlights, button borders, progress bars, active nav indicator |
| `AccentDark` | `#0099CC` | 0,153,204 | Button hover, secondary accent |
| `Border` | `#2A3A4A` | 42,58,74 | Card borders, dividers |

### 1.2 Color Palette — Semantic

| Token | Hex | Usage |
|-------|-----|-------|
| `OK` | `#00E676` | Pass/OK results, green light, connected status |
| `NG` | `#FF1744` | Fail/NG results, red light, alarms, disconnected |
| `Warn` | `#FFD600` | Yellow light, warnings, waiting |
| `Info` | `#40C4FF` | Informational text, guidance |
| `Disabled` | `#455A64` | Disabled buttons, unavailable |
| `TextPrimary` | `#E8EDF2` | Main text, titles |
| `TextSecondary` | `#8899AA` | Labels, secondary info |
| `TextDim` | `#556677` | Placeholders, hints |

### 1.3 Gradients

| Name | Definition | Usage |
|------|-----------|-------|
| AccentGradient | `LinearGradient(#00D4FF → #0099CC, 45°)` | Active nav button |
| OKGradient | `LinearGradient(#00E676 → #00C853, 90°)` | OK result banner |
| NGGradient | `LinearGradient(#FF1744 → #D50000, 90°)` | NG result banner |
| ProgressGradient | `LinearGradient(#00D4FF → #00E676, 0°)` | Progress bar fill |

### 1.4 Typography

| Level | Font | Size | Weight | Color | Usage |
|-------|------|------|--------|-------|-------|
| H1 | Microsoft YaHei | 28px | Bold | TextPrimary | Page titles |
| H2 | Microsoft YaHei | 20px | Bold | TextPrimary | Card titles |
| H3 | Microsoft YaHei | 16px | SemiBold | TextSecondary | Section headers |
| Body | Microsoft YaHei | 14px | Regular | TextPrimary | Normal text |
| DataValue | Consolas | 18px | SemiBold | Accent | Coordinates, measurements |
| LargeNumber | Segoe UI | 60px | Bold | White | Counters |
| SuperResult | Segoe UI | 120px | ExtraBold | White | OK/NG banner text |
| Button | Microsoft YaHei | 14px | SemiBold | White | Button labels |
| Label | Microsoft YaHei | 12px | Regular | TextSecondary | Field labels |
| StatusBar | Segoe UI | 13px | Regular | TextSecondary | Status bar items |

### 1.5 Spacing & Touch

| Rule | Value |
|------|-------|
| Minimum button size | 100×60 px |
| Jog button size | 80×70 px |
| Nav button size | 110×50 px |
| Dropdown row height | 50 px |
| Input field height | 50 px |
| Minimum gap between interactive elements | 12 px |
| Card padding | 16 px |
| Card border radius | 12 px |
| Card border | 1px solid `#2A3A4A` |
| Card shadow | `0 4px 12px rgba(0,0,0,0.3)` |
| Page padding (inside ContentControl) | 12 px |
| Virtual keyboard | Windows `TabTip.exe` — auto-invoked on TextBox focus via attached behavior |

---

## 2. Component Specifications

### 2.1 Button Styles

| Style Key | Size | Background | Border | Radius | Hover | Pressed | Example |
|-----------|------|------------|--------|--------|-------|---------|---------|
| `PrimaryButton` | 120×60 | `#00D4FF` | none | 8px | `#0099CC` | `#007799` | [Auto Start] |
| `DangerButton` | 120×60 | `#FF1744` | none | 8px | `#D50000` | `#B71C1C` | [Auto Stop] |
| `SecondaryButton` | 100×50 | transparent | 1px `#00D4FF` | 6px | `#1B2838` | `#233044` | [Record Point] |
| `NavButton` | 110×50 | transparent | none | 6px | `#233044` | `#00D4FF20` | [Operation] |
| `NavButtonActive` | 110×50 | `#00D4FF10` | bottom 3px `#00D4FF` | 6px | — | — | Current page |
| `IconButton` | 50×50 | transparent | 1px `#2A3A4A` | 25px (circle) | `#233044` | — | [×] [↑] [↓] |
| `JogButton` | 80×70 | `#1B2838` | 1px `#00D4FF` | 8px | `#233044` | `#00D4FF30` | [J1+] [X-] |
| `DisabledButton` | any | `#1B2838` | 1px `#2A3A4A` | — | — | — | Gray text |

### 2.2 TrafficLight Control

Three circles (horizontal layout), diameter 30px each, gap 8px.

| Light | Active Color | Inactive Color | Active Border | Glow Effect |
|-------|-------------|----------------|---------------|-------------|
| Red | `#FF1744` | `#3A1A1A` | 1px `#552222` | `0 0 8px #FF1744` |
| Yellow | `#FFD600` | `#3A3A1A` | 1px `#555522` | `0 0 8px #FFD600` |
| Green | `#00E676` | `#1A3A1A` | 1px `#225522` | `0 0 8px #00E676` |

Active light: breathing glow animation (DoubleAnimation on Opacity, 0.5→1.0, 1s, AutoReverse, Forever).

### 2.3 ConnectionIndicator Control

Circle 12px diameter + text label.

| State | Circle Color | Text | Animation |
|-------|-------------|------|-----------|
| Connected | `#00E676` solid | "Connected" (green) | None |
| Disconnected | `#FF1744` solid | "Disconnected" (red) | Blink (0.5s) |
| Connecting | `#FFD600` solid | "Connecting..." (yellow) | Pulse (1s) |

### 2.4 BigResultBanner Control

- Width: stretch to parent container
- Height: 180px
- Border radius: 16px
- Centered content

| State | Background | Text | Font | Animation |
|-------|-----------|------|------|-----------|
| OK | OKGradient | "OK" | 120px ExtraBold White | Glow shadow |
| NG | NGGradient | "NG" | 120px ExtraBold White | Blink (0.5s) |
| Idle | `#1B2838` | "—" | 60px `#556677` | None |

### 2.5 ComboBox (Dropdown) Style

- Height: 50px
- Background: `#1B2838`
- Border: 1px `#2A3A4A`, radius 6px
- Text: `#E8EDF2` 14px
- Arrow icon: ▼ `#00D4FF`
- Popup background: `#1B2838`
- Item height: 50px
- Item hover: background `#233044`
- Selected item: left border 3px `#00D4FF`

### 2.6 DataGrid Style

- Header: bg `#0D1B2A`, text `#8899AA` 12px Bold, height 40px
- Row: bg `#1B2838`, alt row `#1F2D3D`, height 40px, text `#E8EDF2` 13px
- Selected row: bg `#233044`, left border 3px `#00D4FF`
- NG row: bg `#3A1A1A`, text `#FF1744` Bold
- Horizontal lines: 1px `#2A3A4A`
- No vertical lines

### 2.7 ProgressBar Style

- Height: 8px, radius 4px
- Track: `#2A3A4A`
- Fill: ProgressGradient
- Percentage text above right: `#8899AA` 12px

### 2.8 Card Container Template

```xml
<Border Style="{StaticResource CardStyle}">
  <!-- Card header -->
  <StackPanel Orientation="Horizontal" Margin="0,0,0,12">
    <TextBlock Text="&#xE7F4;" FontFamily="Segoe Fluent Icons" 
               FontSize="20" Foreground="{StaticResource AccentBrush}"/>
    <TextBlock Text="Card Title" Style="{StaticResource H2Style}" Margin="8,0,0,0"/>
  </StackPanel>
  <!-- Card content -->
</Border>
```

---

## 3. Icons

Use **Segoe Fluent Icons** font (built into Windows 11). All icons `#00D4FF`, 20-24px.

| Location | Icon | Unicode | Description |
|----------|------|---------|-------------|
| Nav: Operation | Monitor | `\uE7F4` | Dashboard |
| Nav: Result | CheckMark | `\uE73E` | Result/check |
| Nav: Recipe | Document | `\uE8A5` | Document |
| Nav: Teaching | Joystick | `\uE7FC` | Robot/motion |
| Nav: Calibration | Ruler | `\uECC6` | Calibration |
| Nav: Query | Search | `\uE721` | Search |
| Nav: Settings | Settings | `\uE713` | Gear |
| Nav: Alarm | Warning | `\uE7BA` | Alert triangle |
| Status: Clock | Clock | `\uE823` | Time |
| Status: User | People | `\uE716` | User |
| Status: Heartbeat | Heart | `\uEB51` | Heartbeat |
| Card: Robot | Joystick | `\uE7FC` | Robot status |
| Card: Safety | Shield | `\uE83D` | Safety status |
| Card: Vision | Camera | `\uE722` | Camera/vision |
| Btn: Save | Save | `\uE74E` | Save |
| Btn: Delete | Delete | `\uE74D` | Delete |
| Btn: Import | Import | `\uE8B5` | Import |
| Btn: Export | Download | `\uE896` | Export |
| Btn: Up | ChevronUp | `\uE70E` | Move up |
| Btn: Down | ChevronDown | `\uE70D` | Move down |
| Btn: Play | Play | `\uE768` | Test/run |
| Btn: Stop | Stop | `\uE71A` | Stop |

Fallback: If Segoe Fluent Icons unavailable, use Lucide Icons SVGs bundled in Resources.

---

## 4. Animations

| Animation | Usage | WPF Implementation | Parameters |
|-----------|-------|---------------------|------------|
| Breathing glow | Active traffic light, OK result | DoubleAnimation on Opacity | 0.5→1.0, 1s, AutoReverse, Forever |
| Blink | NG result, alarm, disconnect | DoubleAnimation on Opacity | 0→1, 0.5s, AutoReverse, Forever |
| Pulse | "In progress" guidance text | DoubleAnimation on Opacity | 0.3→1.0, 1.5s, AutoReverse, Forever |
| Page transition | ContentControl content change | Storyboard FadeOut→FadeIn | 0.15s out + 0.15s in |
| Progress bar | Value update | DoubleAnimation on Width | 0.3s, EaseOut |

---

## 5. Page Layouts

### 5.1 MainWindow (1920×1080, fullscreen, no title bar)

```
┌═══════════════════════════ 1920px ══════════════════════════════┐
║ NavBar: #0A1628, H=70px                                        ║
║ [◆Logo][Title]  [⊞Op][✓Res][📋Rec][🤖Tch][🔧Cal][🔍Qry]     ║
║ [⚙Set][⚠Alm]                          [Mode] [●PLC●CR5●VM]   ║
╠═════════════════════════════════════════════════════════════════╣
║ ContentControl: H=950px, Padding=12px, bg=#0D1B2A              ║
║ (Dynamic page content)                                          ║
╠═════════════════════════════════════════════════════════════════╣
║ StatusBar: #060F1A, H=60px                                      ║
║ 🕐Time | 👤Worker | 📦Part | Mode | State | ✓OK:N ✗NG:N | ♥  ║
╚═════════════════════════════════════════════════════════════════╝
```

#### NavBar Elements (left to right):

| Element | X | Size | Content | Style |
|---------|---|------|---------|-------|
| Logo icon | 20 | 32×32 | ◆ diamond SVG | `#00D4FF` |
| System title | 60 | — | "方向盘3D检测系统" | `#E8EDF2` 20px Bold |
| btn_Operation | 320 | 110×50 | "⊞ 操作" | NavButton (all users) |
| btn_Result | 440 | 110×50 | "✓ 结果" | NavButton (all users) |
| btn_Recipe | 560 | 110×50 | "📋 配方" | NavButton (≥Engineer) |
| btn_Teaching | 680 | 110×50 | "🤖 示教" | NavButton (≥Engineer) |
| btn_Calibration | 800 | 110×50 | "🔧 校准" | NavButton (≥Engineer) |
| btn_Query | 920 | 110×50 | "🔍 查询" | NavButton (≥Leader) |
| btn_Settings | 1040 | 110×50 | "⚙ 设置" | NavButton (Admin only) |
| btn_Alarm | 1160 | 110×50 | "⚠ 报警" | NavButton (all users) |
| Mode label | 1500 | — | "手动"/"自动" | `#FFD600`(manual) / `#00E676`(auto) |
| PLC status | 1700 | 12px dot + text | "●PLC" | ConnectionIndicator |
| CR5 status | 1780 | 12px dot + text | "●CR5" | ConnectionIndicator |
| VM status | 1860 | 12px dot + text | "●VM" | ConnectionIndicator |

**Active page**: bottom border 3px `#00D4FF`, bg `#00D4FF10`.  
**Insufficient permission**: `Visibility=Collapsed` (completely hidden, not grayed out).

#### StatusBar Elements:

| Element | X | Content | Style | Binding |
|---------|---|---------|-------|---------|
| Clock | 20 | "🕐 2026-03-14 10:23:45" | `#8899AA` 13px | DispatcherTimer 1s |
| Worker | 340 | "👤 W001 张三" | `#8899AA` 13px | WorkerId + WorkerName |
| Part | 580 | "📦 FXP-001" | `#00D4FF` 13px | PartNumber |
| Mode | 780 | "单面有码" | `#FFD600` 13px | DetectMode→string |
| State | 920 | "S0:空闲" | `#8899AA` 13px | CurrentState→string |
| OK count | 1200 | "✓ OK:123" | `#00E676` 14px Bold | OKCount |
| NG count | 1380 | "✗ NG:2" | `#FF1744` 14px Bold | NGCount |
| Yield | 1530 | "98.4%" | `#E8EDF2` 14px Bold | Calculated |
| Heartbeat | 1870 | "●" | `#00E676` 12px blinking | Heartbeat toggle |

---

### 5.2 LoginView (content area: 1896×950)

**Layout**: Center-aligned, 3 cards.

```
┌─────────────────────── 1896×950 ──────────────────────────┐
│          (vertically centered)                             │
│  ┌── Card 1: Login (W:500) ──┐  ┌── Card 2: Product (W:500) ──┐
│  │ 👤 Worker Login            │  │ 📦 Product Selection         │
│  │ Worker ID: [____dropdown▼] │  │ Part Number: [____dropdown▼] │
│  │ Name: Zhang San (readonly) │  │ Mode: Single+QR (auto,readonly)│
│  │ [  Login  ] [  Logout  ]   │  │ Recipe: ● Ready (green/gray)  │
│  └────────────────────────────┘  │ [  Load Recipe  ]             │
│                                  └────────────────────────────────┘
│  ┌── Card 3: Device Status (W:1040) ──────────────────────┐
│  │ ●PLC: 192.168.2.1 Connected   ●CR5: Connected+Enabled │
│  │ ●VM: 127.0.0.1:502 Connected  [Reconnect All]         │
│  └────────────────────────────────────────────────────────┘
│  ┌── Step Guide (W:1040) ────────────────────────────────┐
│  │ ①Login → ②Select Product → ③Calibrate → ④[Auto Start]│
│  │ (completed=green✓, current=cyan pulse, future=gray)   │
│  └───────────────────────────────────────────────────────┘
└────────────────────────────────────────────────────────────┘
```

**Step guide bar**: 4 circles horizontally with connecting lines. Completed step: green circle with ✓. Current step: cyan circle with pulse animation. Future step: gray circle. Connecting line: green if completed, gray if not.

**Dropdown contents**:
- Worker ID: from `T_User WHERE IsActive=true`, maintained by Admin/Engineer in Settings
- Part Number: from `T_Recipe WHERE PointsReady=true AND CalibReady=true`, auto-populated
- Mode: auto-displayed from selected recipe's DetectMode, read-only

---

### 5.3 OperationView (content area: 1896×950)

**Layout**: 3 columns.

```
┌────────────────────────── 1896×950 ──────────────────────────────┐
│┌─ Station 1 Card (W:570) ─┐┌─ System Card (W:700) ─┐┌─ Station 2 (W:570) ─┐│
││ 🔵 Station 1 (A-side)    ││ 🤖 Robot Status        ││ 🔵 Station 2 (B-side) ││
││                           ││ Mode: Enabled (#00E676)││                       ││
││ ◉Red  ◉Yellow  ◉Green   ││ X: +123.456 mm         ││ ◉Red  ◉Yellow  ◉Green ││
││ (TrafficLight horizontal) ││ Y:  -78.901 mm         ││ (TrafficLight)        ││
││                           ││ Z: +456.789 mm         ││                       ││
││ State: S0 Idle            ││ Rx: +180.00°           ││ State: Inactive       ││
││ (Accent 18px)             ││ Ry:    +0.00°          ││ (TextDim)             ││
││                           ││ Rz:  -90.00°           ││                       ││
││ Workpiece: ● None         ││                        ││ Workpiece: ● None     ││
││ QR Code: —                ││ Speed: ████░░ 50%      ││ QR Code: —            ││
││                           ││                        ││                       ││
││ ┌─ Progress ────────────┐││ ┌─ Safety Status ─────┐││ ┌─ Progress ────────┐ ││
││ │ ████████░░ 80%        │││ │ 🛡 E-Stop: ●Normal  │││ │ ░░░░░░░░░░ 0%    │ ││
││ └───────────────────────┘││ │ 🛡 Curtain: ●Normal │││ └──────────────────┘ ││
││                           ││ │ 🛡 C# HB: ●Normal  │││                       ││
││ Last Result:              ││ │ 🛡 VM HB: ●Normal  │││ Last Result:          ││
││ ┌───────────────────┐   ││ └─────────────────────┘││ ┌──────────────────┐  ││
││ │    ██ OK ██        │   ││                        ││ │      — —         │  ││
││ │ (mini banner H:80) │   ││                        ││ │ (gray, H:80)     │  ││
││ └───────────────────┘   ││                        ││ └──────────────────┘  ││
│└───────────────────────────┘└────────────────────────┘└───────────────────────┘│
│┌── Guidance Banner (full width, H:100) ──────────────────────────────────────┐│
││ >>> Please place the workpiece on Station 1 and press [Station 1 Start] <<< ││
││ (Info color 22px, centered, card bg, left accent bar 4px #00D4FF)           ││
│└─────────────────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────┘
```

#### Station Card Elements:

| Element | Y offset | Size | Description |
|---------|----------|------|-------------|
| Title | 10 | icon 20px + H2 | "🔵 Station 1 (A-side)" |
| Traffic lights | 60 | 3×30px circles, horizontal, gap 12 | TrafficLight control |
| State text | 110 | — | "S0 Idle" in Accent 18px |
| Workpiece sensor | 160 | dot 12px + text | "Workpiece: ● Present/None" |
| QR code | 200 | — | "QR: ABC123..." (truncated) |
| Progress bar | 260 | full width, H:8px | ProgressBar |
| Percentage | 252 | — | "80%" right-aligned 12px |
| Last result mini | 310 | full width, H:80px | Miniature BigResultBanner |

#### Guidance Banner Text Mapping:

| State | Text | Color | Animation |
|-------|------|-------|-----------|
| S0 (has QR) | "请将方向盘放到工位,按[启动]按钮" | `#40C4FF` | none |
| S0 (no QR) | "请将方向盘放到工位,按[启动]按钮" | `#40C4FF` | none |
| S1 | "📷 正在读取二维码..." | `#FFD600` | pulse |
| S2 | "🔍 二维码验证中..." | `#FFD600` | pulse |
| S3 | "🤖 机器人就绪检查..." | `#FFD600` | pulse |
| S4 | "📐 3D扫描进行中..." | `#FFD600` | pulse+blink |
| S5 | "📐 读取3D测量数据..." | `#FFD600` | pulse |
| S6 | "📷 2D拍照中(第{n}/{total}个)..." | `#FFD600` | pulse+blink |
| S7 | "🤖 机器人回位中..." | `#FFD600` | pulse |
| S8 | "🧮 正在计算检测结果..." | `#FFD600` | pulse |
| S9-OK | "✅ 检测合格! 请放到合格挂车" | `#00E676` | breathing |
| S9-NG | "❌ 不合格! 请放到不合格挂车" | `#FF1744` | blink |
| S9-DualOK | "✅ A面合格! 请翻面放到2工位" | `#00E676` | breathing |
| S10 | "❌ 二维码不匹配! 请剔除,按[复位]" | `#FF1744` | blink |
| S99 | "⚠️ 系统异常! 按[复位]恢复" | `#FF1744` | fast blink |

---

### 5.4 ResultView (content area: 1896×950)

```
┌──────────────────────── 1896×950 ─────────────────────────────────┐
│ ┌─ Info Bar Card (H:60) ──────────────────────────────────────┐   │
│ │ QR: ABC12345678 | FXP-001 | W001 Zhang | 10:23:45 | A-side │   │
│ └─────────────────────────────────────────────────────────────┘   │
│ ┌─ 3D Data Card (H:480, W:1500) ────────┐ ┌─ 2D Card (W:360) ─┐│
│ │ DataGrid:                               │ │ 📷 2D Defect       ││
│ │ #│Name     │Axis│Nominal│Actual│Dev │OK │ │ ●OK Point 1        ││
│ │ 1│Flange H │ Z  │12.500 │12.512│+.01│ ✓│ │ ●OK Point 2        ││
│ │ 2│Width    │ X  │85.000 │85.234│+.23│ ✗│ │ ●NG Point 3 ←red   ││
│ │ 3│Depth    │ Z  │ 8.200 │ 8.195│-.00│ ✓│ │ ●OK Point 4        ││
│ │                                         │ │ ●OK Point 5        ││
│ │ 3D Summary: Pass 8/10  Fail 2/10       │ │ 2D: OK 4/5         ││
│ └─────────────────────────────────────────┘ └────────────────────┘│
│ ┌─ BigResultBanner (H:180) ──────────────────────────────────┐   │
│ │                    ██████  NG  ██████                        │   │
│ │               (NGGradient bg, White 120px, glow)            │   │
│ └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

**DataGrid Columns**:

| Column | Width | Align | Content | NG Style |
|--------|-------|-------|---------|----------|
| # | 60 | Center | PointIndex | — |
| Name | 180 | Left | PointName | — |
| Axis | 60 | Center | MeasureAxis | — |
| Nominal | 120 | Right | NominalValue (F3) | — |
| Actual | 120 | Right | ActualValue (F3) | Red text if !IsOK |
| Deviation | 120 | Right | Deviation with ± sign | Red text if !IsOK |
| Upper Tol | 100 | Right | TolUpper | — |
| Lower Tol | 100 | Right | TolLower | — |
| Judgment | 60 | Center | ✓(`#00E676`) / ✗(`#FF1744`) | — |

**NG row**: entire row bg `#3A1A1A`, text Bold.

---

### 5.5 RecipeView (≥Engineer)

```
┌── 1896×950 ──────────────────────────────────────────────────────┐
│┌─ Product List (W:350) ──┐┌─ Recipe Detail (W:1530) ────────────┐│
││ [+ New] [× Delete]      ││ 📋 FXP-001 Steering Wheel A          ││
││ ┌─────────────────┐     ││ Part#: [FXP-001]  Name: [Wheel A]   ││
││ │▸ FXP-001 ✓      │←sel ││ Mode: [Single+QR ▼]  Prefix: [FXP] ││
││ │  FXP-002 ✓      │     ││ A-side: Measure[5▼] Photo[3▼]       ││
││ │  FXP-003 ○      │     ││ B-side: Measure[4▼] Photo[2▼]       ││
││ └─────────────────┘     ││                                      ││
││ ✓=ready ○=not ready     ││ ┌─Tab──────────────────────────────┐ ││
││                          ││ │[Standards]│[Points]│[Calibration]│ ││
││                          ││ ├──────────────────────────────────┤ ││
││                          ││ │ [📥Import Excel] [+ Add Row]     │ ││
││                          ││ │ DataGrid: std data editable      │ ││
││                          ││ └──────────────────────────────────┘ ││
││                          ││ [💾 Save Recipe]                     ││
│└──────────────────────────┘└──────────────────────────────────────┘│
└───────────────────────────────────────────────────────────────────┘
```

**Tab: Standards** = editable DataGrid of T_Standard rows.  
**Tab: Points** = read-only view of T_RobotPoint (edit in TeachingView).  
**Tab: Calibration** = read-only view of T_CalibOffset (edit in CalibrationView).

---

### 5.6 TeachingView (≥Engineer)

```
┌── 1896×950 ──────────────────────────────────────────────────────┐
│ Recipe:[FXP-001▼] Side:[A▼] Mode:[Continuous▼] Step:[1.0mm▼]   │
│┌─ Jog Control (W:400) ─┐┌─ Coordinates (W:400) ─┐┌─ Points (W:700) ─┐│
││ ═══ Joints ═══         ││ Cartesian:             ││ #│Type    │X,Y,Z  ││
││ [J1-] J1 [J1+]        ││ X: +123.456 mm         ││ 1│QR_PHOTO│100,200││
││ [J2-] J2 [J2+]        ││ Y:  -78.901 mm         ││ 2│PHOTO_01│150,250││
││ [J3-] J3 [J3+]        ││ Z: +456.789 mm         ││ 3│PHOTO_02│200,300││
││ [J4-] J4 [J4+]        ││ Rx: +180.00 °          ││ 4│SCAN_A  │-300.. ││
││ [J5-] J5 [J5+]        ││ Ry:    +0.00 °         ││ 5│SCAN_B  │-300.. ││
││ [J6-] J6 [J6+]        ││ Rz:  -90.00 °          ││                   ││
││                         ││                        ││ [▲Up] [▼Down]     ││
││ ═══ Cartesian ═══      ││ Joints:                ││                   ││
││ [X-] X [X+]           ││ J1:  +0.000 °          ││ [Record Point]    ││
││ [Y-] Y [Y+]           ││ J2: +45.230 °          ││ [Move To Point]   ││
││ [Z-] Z [Z+]           ││ J3: -89.100 °          ││ [Overwrite Point] ││
││ [Rx-] Rx [Rx+]        ││ J4:  +0.000 °          ││ [Delete Point]    ││
││ [Ry-] Ry [Ry+]        ││ J5: +88.770 °          ││                   ││
││ [Rz-] Rz [Rz+]        ││ J6: -12.340 °          ││ [▶ Test Full Path]││
││                         ││                        ││                   ││
││ Speed: [████░░ 30%]    ││ Temp: J1:32° J2:28°    ││                   ││
│└─────────────────────────┘└────────────────────────┘└───────────────────┘│
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 5.7 CalibrationView (≥Engineer)

```
┌── 1896×950 ──────────────────────────────────────────────────────┐
│ Recipe:[FXP-001▼]  Side:[A▼]  Status: ○ Not Calibrated          │
│┌─ Calibration Guide (W:400) ─┐┌─ Calibration Data (W:1470) ───┐│
││ 🔧 Calibration Wizard        ││ DataGrid:                      ││
││                               ││ #│Name   │Nominal│Run1│Run2│..│Avg│Offset│
││ ① Place standard part        ││ 1│Flange │12.500│12.51│12.50││12.505│0.005│
││                               ││ 2│Width  │85.000│85.03│85.02││85.025│0.025│
││ ② [Execute Calibration Scan] ││                                ││
││    (primary button)           ││                                ││
││                               ││ Trend Chart (LiveCharts):     ││
││ Runs: 3/5  ████████░░ 60%   ││ X=run#, Y=offset per point    ││
││                               ││ Multi-line (one per point)    ││
││ ③ [Save Calibration Results] ││                                ││
││    [Mark as Calibrated]       ││                                ││
│└───────────────────────────────┘└────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

---

### 5.8 DataQueryView (≥Leader)

- **Top filter bar (H:80, card)**: Date from [picker] Date to [picker] Part# [dropdown▼] Result [All/OK/NG▼] QR [textbox] [🔍Search] [📥Export]
- **Main DataGrid (H:750, card)**: Time | QR | Part# | Worker | Mode | A-result | B-result | Overall | CycleTime — click row to expand detail
- **Bottom stats (H:60)**: Total: 1234 | OK: 1210 | NG: 24 | Yield: 98.1% (large numbers in Accent)

### 5.9 SettingsView (Admin only)

Left vertical tab nav: [Devices] [Fixed Points] [Speeds] [Users] [System Info]

- **Devices**: IP inputs for PLC/CR5/VM + [Test Connection] buttons + status indicators
- **Fixed Points**: HOME X/Y/Z/Rx/Ry/Rz inputs + [Modify] with confirmation dialog
- **Speeds**: Sliders (1-100) for ScanSpeed / PhotoMoveSpeed / TeachMaxSpeed
- **Users**: DataGrid (WorkerId | Name | Role | Active) + [Add] [Delete] [Change Password] + Role dropdown
- **System Info**: Software version | .NET version | Uptime | DB size | Disk free | [Clean Logs]

### 5.10 AlarmView (all users)

Top tabs: [🔴 Active Alarms] [📜 Alarm History]

- **Active**: DataGrid (Time | Level icon+color | Source | Message | Status) + [✓ Acknowledge] [✓✓ Ack All]
- **History**: Same + date filter + export
- Level colors: Emergency=`#FF1744`, Fault=`#FF6D00`, Warning=`#FFD600`, Info=`#40C4FF`

---

## 6. Dropdown Menu Master List

| Page | Dropdown | Content Source | Maintained By |
|------|----------|----------------|---------------|
| LoginView | Worker ID | T_User (IsActive=true) | Admin/Engineer (Settings page) |
| LoginView | Part Number | T_Recipe (PointsReady=true AND CalibReady=true) | Engineer (Recipe page) |
| LoginView | Detect Mode | Auto from selected recipe | — (read-only) |
| RecipeView | Detect Mode | Fixed 4 items: 单面有码/单面无码/双面有码/双面无码 | — (fixed) |
| RecipeView | Measure Axis | Fixed 3 items: X / Y / Z | — (fixed) |
| RecipeView | Measure Count (A/B) | Fixed range: 0-20 | — (fixed) |
| RecipeView | Photo Count (A/B) | Fixed range: 0-10 | — (fixed) |
| TeachingView | Recipe | All T_Recipe records | — |
| TeachingView | Side | Fixed: A面 / B面 | — (fixed) |
| TeachingView | Jog Mode | Fixed: 连续 / 步进 | — (fixed) |
| TeachingView | Step Size | Fixed: 0.1 / 0.5 / 1.0 / 5.0 / 10.0 | — (fixed) |
| TeachingView | Point Type | Fixed: SCAN_A/SCAN_B/QR_PHOTO/PHOTO_01-10 | — (fixed) |
| CalibrationView | Recipe | All T_Recipe records | — |
| CalibrationView | Side | Fixed: A面 / B面 | — (fixed) |
| DataQueryView | Part Number | All T_Recipe + "全部" | — |
| DataQueryView | Result Filter | Fixed: 全部 / OK / NG | — (fixed) |
| SettingsView | User Role | Fixed: Admin / Engineer / Leader / Operator | — (fixed) |

---

## 7. Styles.xaml Template

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- ═══ COLORS ═══ -->
    <Color x:Key="BgPrimary">#0D1B2A</Color>
    <Color x:Key="BgCard">#1B2838</Color>
    <Color x:Key="BgCardHover">#233044</Color>
    <Color x:Key="BgNav">#0A1628</Color>
    <Color x:Key="BgStatus">#060F1A</Color>
    <Color x:Key="Accent">#00D4FF</Color>
    <Color x:Key="AccentDark">#0099CC</Color>
    <Color x:Key="OK">#00E676</Color>
    <Color x:Key="NG">#FF1744</Color>
    <Color x:Key="Warn">#FFD600</Color>
    <Color x:Key="Info">#40C4FF</Color>
    <Color x:Key="Disabled">#455A64</Color>
    <Color x:Key="TextPrimary">#E8EDF2</Color>
    <Color x:Key="TextSecondary">#8899AA</Color>
    <Color x:Key="TextDim">#556677</Color>
    <Color x:Key="Border">#2A3A4A</Color>
    <Color x:Key="NGRowBg">#3A1A1A</Color>

    <!-- ═══ BRUSHES ═══ -->
    <SolidColorBrush x:Key="BgPrimaryBrush" Color="{StaticResource BgPrimary}"/>
    <SolidColorBrush x:Key="BgCardBrush" Color="{StaticResource BgCard}"/>
    <SolidColorBrush x:Key="BgCardHoverBrush" Color="{StaticResource BgCardHover}"/>
    <SolidColorBrush x:Key="BgNavBrush" Color="{StaticResource BgNav}"/>
    <SolidColorBrush x:Key="BgStatusBrush" Color="{StaticResource BgStatus}"/>
    <SolidColorBrush x:Key="AccentBrush" Color="{StaticResource Accent}"/>
    <SolidColorBrush x:Key="AccentDarkBrush" Color="{StaticResource AccentDark}"/>
    <SolidColorBrush x:Key="OKBrush" Color="{StaticResource OK}"/>
    <SolidColorBrush x:Key="NGBrush" Color="{StaticResource NG}"/>
    <SolidColorBrush x:Key="WarnBrush" Color="{StaticResource Warn}"/>
    <SolidColorBrush x:Key="InfoBrush" Color="{StaticResource Info}"/>
    <SolidColorBrush x:Key="TextPrimaryBrush" Color="{StaticResource TextPrimary}"/>
    <SolidColorBrush x:Key="TextSecondaryBrush" Color="{StaticResource TextSecondary}"/>
    <SolidColorBrush x:Key="TextDimBrush" Color="{StaticResource TextDim}"/>
    <SolidColorBrush x:Key="BorderBrush" Color="{StaticResource Border}"/>

    <!-- ═══ CARD STYLE ═══ -->
    <Style x:Key="CardStyle" TargetType="Border">
        <Setter Property="Background" Value="{StaticResource BgCardBrush}"/>
        <Setter Property="BorderBrush" Value="{StaticResource BorderBrush}"/>
        <Setter Property="BorderThickness" Value="1"/>
        <Setter Property="CornerRadius" Value="12"/>
        <Setter Property="Padding" Value="16"/>
        <Setter Property="Effect">
            <Setter.Value>
                <DropShadowEffect BlurRadius="12" ShadowDepth="4" Opacity="0.3" Color="Black"/>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- ═══ PRIMARY BUTTON ═══ -->
    <Style x:Key="PrimaryButton" TargetType="Button">
        <Setter Property="Background" Value="{StaticResource AccentBrush}"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="FontFamily" Value="Microsoft YaHei"/>
        <Setter Property="FontSize" Value="14"/>
        <Setter Property="FontWeight" Value="SemiBold"/>
        <Setter Property="MinWidth" Value="120"/>
        <Setter Property="MinHeight" Value="50"/>
        <Setter Property="Cursor" Value="Hand"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border" Background="{TemplateBinding Background}"
                            CornerRadius="8" Padding="16,8">
                        <ContentPresenter HorizontalAlignment="Center" 
                                          VerticalAlignment="Center"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Background" 
                                    Value="{StaticResource AccentDarkBrush}"/>
                        </Trigger>
                        <Trigger Property="IsEnabled" Value="False">
                            <Setter TargetName="border" Property="Background"
                                    Value="{StaticResource BgCardBrush}"/>
                            <Setter Property="Foreground" Value="{StaticResource TextDimBrush}"/>
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- ═══ DANGER BUTTON ═══ -->
    <Style x:Key="DangerButton" TargetType="Button" BasedOn="{StaticResource PrimaryButton}">
        <Setter Property="Background" Value="{StaticResource NGBrush}"/>
    </Style>

    <!-- ═══ JOG BUTTON ═══ -->
    <Style x:Key="JogButton" TargetType="Button">
        <Setter Property="Background" Value="{StaticResource BgCardBrush}"/>
        <Setter Property="Foreground" Value="{StaticResource AccentBrush}"/>
        <Setter Property="BorderBrush" Value="{StaticResource AccentBrush}"/>
        <Setter Property="BorderThickness" Value="1"/>
        <Setter Property="FontFamily" Value="Consolas"/>
        <Setter Property="FontSize" Value="16"/>
        <Setter Property="FontWeight" Value="Bold"/>
        <Setter Property="Width" Value="80"/>
        <Setter Property="Height" Value="70"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border" Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}"
                            CornerRadius="8">
                        <ContentPresenter HorizontalAlignment="Center" 
                                          VerticalAlignment="Center"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsPressed" Value="True">
                            <Setter TargetName="border" Property="Background" Value="#00D4FF30"/>
                            <Setter Property="Foreground" Value="White"/>
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

</ResourceDictionary>
```
