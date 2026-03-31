# Drilling Machine PLC — TwinCAT 3

> Exercise 3 — Drilling Machine Manual/Auto Mode  
> University West | Course: Industrial Automation  
> Programmed in TwinCAT 3 (IEC 61131-3)

---

## 📋 Project Overview

A PLC control program for a simulated drilling machine built in TwinCAT 3, following  
**EU Machinery Regulation (EU) 2023/1230** safety requirements.  
Demonstrates structured programming using FBD, SFC, and Structured Text.

The machine has three actuators:

| Actuator | Function |
|----------|----------|
| **Slider A** | Pushes the workpiece into drilling position |
| **Drill** | Moves down to drill, then retracts back up |
| **Slider B** | Moves the finished workpiece out of the machine |

---

## 🏗️ Project Structure

```
TwinCAT Project Drill Man Auto/
│
├── TwinCAT Project Drill.sln              ← Open this in TwinCAT 3 XAE
│
└── TwinCAT Project Drill/
    ├── TwinCAT Project Drill.tsproj       ← TwinCAT solution/system config
    │
    ├── PLCStudent/                        ← Student control program
    │   ├── POUs/
    │   │   ├── BLINK.TcPOU               ← Function Block (ST) — reusable blink timer
    │   │   ├── ControlFBD.TcPOU          ← Program (FBD) — safety, mode, output logic
    │   │   ├── ControlFBD_Sequence.TcPOU ← Function Block (SFC) — automatic drill cycle
    │   │   ├── MAIN.TcPOU                ← Program (ST) — connects everything together
    │   │   ├── POUfbd.TcPOU              ← Additional FBD program
    │   │   ├── Sequence.TcPOU            ← Additional sequence program
    │   │   └── STATES.TcDUT              ← ENUM: init, Aout, drillDown, drillUp, Ain, Bin, Bout
    │   └── GVLs/                         ← Global Variable Lists
    │
    └── PLCSim/                            ← Machine simulation
        ├── POUs/
        │   └── MAIN.TcPOU                ← Program (ST) — simulates physical machine I/O
        └── VISUs/
            └── Visualization.TcVIS       ← HMI visualization panel
```

---

## 💻 Getting Started

### Step 1 — Install TwinCAT 3

TwinCAT 3 XAE is free to download from Beckhoff. It runs on top of Visual Studio.

1. Go to the Beckhoff download page:  
   👉 https://www.beckhoff.com/en-en/support/download-finder/software-and-tools/

2. Search for **TwinCAT 3 XAE** and download the installer (you may need to create a free Beckhoff account)

3. Run the installer and follow the setup wizard  
   > ⚠️ TwinCAT requires **Windows 10 or 11** (64-bit). It does not run on macOS or Linux.

4. During installation, allow it to install the **TwinCAT RT (real-time) driver** when prompted — this is required to run the PLC simulation

5. Restart your PC after installation

> **Note:** A free 7-day trial license is included with TwinCAT. The project already contains a `TrialLicense.tclrs` file which activates automatically when you open it.

---

### Step 2 — Clone or Download the Project

**Option A — Clone with Git:**
```bash
git clone https://github.com/Tejashwi-Jagadish/TwinCAT-Project-Drill.git
```

**Option B — Download ZIP:**
1. Click the green **Code** button at the top of this page
2. Select **Download ZIP**
3. Extract the ZIP to a folder on your PC (e.g. `C:\TwinCAT Projects\`)

---

### Step 3 — Open the Project

1. Open **TwinCAT XAE Shell** (installed with TwinCAT 3)
2. Go to **File → Open → Project/Solution**
3. Navigate to the extracted folder and open:
   ```
   TwinCAT Project Drill.sln
   ```
4. The solution will load with two PLC instances visible in the Solution Explorer:
   - **PLCStudent** — the control logic
   - **PLCSim** — the machine simulation

---

### Step 4 — Activate the Configuration

1. In the top toolbar, click **Activate Configuration** (or press `Ctrl+Shift+F4`)
2. A dialog will appear asking to switch to **Run Mode** — click **OK**
3. If prompted about the license, click **7 Days Trial License** → confirm

> This step links TwinCAT to the local real-time runtime and prepares both PLC instances for execution.

---

### Step 5 — Start the PLC

1. In the Solution Explorer, right-click **PLCSim** → **Login**  
   Then click **Start** (▶) or press `F5`

2. Do the same for **PLCStudent** → **Login** → **Start**

Both PLCs are now running and communicating with each other via linked I/O variables.

---

### Step 6 — Open the Visualization

1. In the Solution Explorer, expand:  
   `PLCSim → VISUs → Visualization`
2. Double-click **Visualization** to open the HMI panel
3. The panel shows the machine state, buttons, lamps, and selector switch

You can now interact with the machine:
- Toggle the **MAN/AUTO selector**
- Press **E-Stop** and **Reset**
- In Manual mode: use individual actuator buttons
- In Auto mode: press **Start** to run the drill cycle

---

## 🔧 POU Descriptions

### BLINK (Function Block — ST)
Reusable blink generator used for all indicator lamps.
- **Inputs:** `OnTime`, `OffTime` (TIME)
- **Output:** `OUT` (BOOL)
- Uses two TON timers internally to toggle the output

---

### ControlFBD (Program — FBD)
The main control logic, divided into numbered networks:

| Network | Function |
|---------|----------|
| 1 | BLINK instance — generates `mxBlink` signal |
| 2 | E-Stop summation — both buttons ANDed |
| 3 | E-Stop reset with self-holding latch |
| 4 | E-Stop lamp — blinks when ready, steady when reset |
| 5 | Safety stop summation |
| 6 | Safety stop reset with self-holding latch |
| 7 | Safety stop lamp |
| 8 | Manual mode permission (`mxManRunOk`) |
| 9 | Auto mode ready (`mxAutoModeReady`) |
| 10 | Home position lamp |
| 11 | Cycle stop latch |
| 12 | Auto run OK (`mxAutoRunOk`) |
| 13 | Start lamp indication |
| 14–19 | All physical outputs — manual OR auto |

---

### ControlFBD_Sequence (Function Block — SFC)
The automatic drill cycle sequence:

```
Init → Slider A Out → Drill Down → Drill Up → Slider B In → Slider A In → Slider B Out → Init
```

- Each step waits for **sensor confirmation** before advancing
- Every step has an **escape transition** back to Init if Auto mode is lost

---

### MAIN (Program — ST)
Wires ControlFBD and the SFC together each scan cycle:
1. Call `ControlFBD()`
2. Pass sensor + permission signals **into** `fbSequence`
3. Read auto commands **back** from `fbSequence`
4. Write auto commands **into** `ControlFBD` inputs
5. Call `fbBlink()`

---

### STATES (DUT — ENUM)
Named states used by the sequence:
```
init, Aout, drillDown, drillUp, Ain, Bout, Bin
```

---

## 🔒 Safety Features

### Emergency Stop
- Dual channel — **both buttons** must be released to reset
- Self-holding latch — operator must **manually press Reset**
- Machine does **not restart automatically**
- Lamp blinks when ready to reset → steady when safe

### Safety Stop
- Covers safety gates and light curtains
- Same reset logic as Emergency Stop
- Required for **Auto mode only** (not manual, per risk assessment)

### Home Position Check
All three sensors must confirm safe position before Auto can start:
- Slider A is IN
- Slider B is OUT
- Drill is UP

### Cycle Stop
- Finishes the **current cycle completely** before stopping
- Machine never stops mid-movement
- Returns to home position and waits

---

## ⚙️ Operating Modes

### Manual Mode (Selector = MAN)
- Individual button control of each actuator
- Requires: E-Stop OK
- Safety stop not required
- Buttons are momentary — active only while held

### Auto Mode (Selector = AUTO)
- Full automatic drill cycle
- Requires: E-Stop OK + Safety Stop OK + Home Position confirmed
- Operator presses Start to begin; sequence repeats until Cycle Stop or E-Stop

---

## 🔄 Mode Switching Behaviour

| Event | Result |
|-------|--------|
| Selector → MAN | `mxManRunOk = TRUE`, SFC escapes to Init immediately |
| Selector → AUTO | `mxManRunOk = FALSE`, manual buttons disabled |
| E-Stop pressed | All outputs FALSE, SFC resets to Init |
| Cycle Stop pressed | Finishes current cycle, stops at home |

---

## 📐 Variable Naming Convention

| Prefix | Type | Example |
|--------|------|---------|
| `ix` | Digital Input | `ixSliderAout` |
| `qx` | Digital Output | `qxSliderAout` |
| `mx` | Internal BOOL | `mxAutoRunOk` |
| `fb` | Function Block instance | `fbBlink` |
| `mi` | Internal INT | `miSeq` |

---

## 📋 Prerequisites

- TwinCAT 3 XAE (version 3.1 or later)
- Windows 10 or 11 (64-bit)
- No external hardware required — fully simulated

---

## 📚 References

- EU Machinery Regulation (EU) 2023/1230
- IEC 61131-3 Programming Languages Standard
- University West — Industrial Automation, Lecture 5

---

## 👤 Author

**Tejashwi K J**  
MSc Robotics — University West, Trollhättan  
2025
