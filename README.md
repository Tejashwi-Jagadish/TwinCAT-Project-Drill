# Drilling Machine PLC — TwinCAT 3
> Exercise 3 — Drilling Machine Manual/Auto Mode  
> University West | Course: Industrial Automation  
> Programmed in TwinCAT 3 (IEC 61131-3)

---

## 📋 Project Overview

This project implements a PLC control program for a simulated drilling machine using TwinCAT 3. The program follows **EU Machinery Regulation (EU) 2023/1230** safety requirements and demonstrates good programming structure using FBD, SFC, and Structured Text.

The machine has three moving parts:
- **Slider A** — pushes the workpiece into drilling position
- **Drill** — moves down to drill, then retracts back up
- **Slider B** — moves the finished workpiece out of the machine

---

## 🏗️ Project Structure

```
TwinCAT Project Drill/
│
├── PLCStudent/
│   └── PLCStudent Project/
│       ├── DUTs/
│       │   └── STATES (ENUM)        ← State names for sequence
│       ├── GVLs/                    ← Global Variable Lists
│       └── POUs/
│           ├── BLINK (FB)           ← Reusable blink function block
│           ├── ControlFBD (PRG)     ← Safety + Mode + Output logic
│           ├── ControlFBD_Sequence (FB) ← Auto sequence in SFC
│           └── MAIN (PRG)           ← Connects everything together
│
└── PLCSim/
    └── PLCSim Project/
        └── MAIN (PRG)               ← Simulation of physical machine
```

---

## 🔧 POU Descriptions

### BLINK (Function Block — ST)
A reusable blink generator used for all indicator lamps.
- **Input:** `OnTime`, `OffTime` (TIME)
- **Output:** `OUT` (BOOL)
- Uses two TON timers internally to toggle output

---

### ControlFBD (Program — FBD)
The main control logic. Divided into clear numbered networks:

| Network | What it does |
|---------|-------------|
| 1 | BLINK instance — generates mxBlink signal |
| 2 | E-Stop summation — both buttons ANDed |
| 3 | E-Stop reset with self-holding latch |
| 4 | E-Stop lamp — blinks when ready, steady when reset |
| 5 | Safety stop summation |
| 6 | Safety stop reset with self-holding latch |
| 7 | Safety stop lamp |
| 8 | Manual mode permission (mxManRunOk) |
| 9 | Auto mode ready (mxAutoModeReady) |
| 10 | Home position lamp |
| 11 | Cycle stop latch |
| 12 | Auto run OK (mxAutoRunOk) |
| 13 | Start lamp indication |
| 14–19 | All physical outputs — manual OR auto |

---

### ControlFBD_Sequence (Function Block — SFC)
The automatic drill cycle sequence.

**Cycle order:**
```
Init → Slider A Out → Drill Down → Drill Up → Slider B In → Slider A In → Slider B Out → Init
```

Each step:
- Waits for a **sensor confirmation** before moving to the next step
- Has an **escape transition** (`NOT mxAutoModeReady → Init_Sequence`) at every step

---

### MAIN (Program — ST)
Connects ControlFBD and the SFC together every scan cycle.

```
1. Call ControlFBD()
2. Pass sensor + permission signals INTO fbSequence
3. Read auto commands BACK from fbSequence
4. Write auto commands INTO ControlFBD inputs
5. Call fbBlink()
```

---

### STATES (ENUM — DUT)
Named states used by the sequence state machine:
```
init, Aout, drillDown, drillUp, Ain, Bout, Bin
```

---

## 🔒 Safety Features

### Emergency Stop
- Dual channel — **two buttons** must both be released
- Self-holding reset — operator must **manually press Reset**
- Machine does **not restart automatically**
- Lamp **blinks** when ready to reset → **steady** when safe

### Safety Stop
- Covers safety gates and light curtains
- Same reset logic as Emergency Stop
- Required for **Auto mode only**
- Manual mode allowed without it (per risk assessment)

### Home Position Check
- All three sensors must confirm safe start position before auto can begin:
  - Slider A is IN
  - Slider B is OUT
  - Drill is UP

### Cycle Stop
- Finishes the **current cycle completely**
- Returns machine to **home position**
- Then stops and waits
- Machine never stops mid-movement

---

## ⚙️ Operating Modes

### Manual Mode (Selector = MAN)
- Individual button control of each actuator
- Requires: Emergency Stop OK
- Safety stop not required
- Each button only active while held

### Auto Mode (Selector = AUTO)
- Full automatic drill cycle
- Requires: Emergency Stop OK + Safety Stop OK + Home Position
- Operator must press Start to begin
- Sequence repeats until Cycle Stop or E-Stop

---

## 🔄 How Manual/Auto Switching Works

| Event | Result |
|-------|--------|
| Selector → MAN | mxManRunOk = TRUE, SFC escapes to Init immediately |
| Selector → AUTO | mxManRunOk = FALSE, manual buttons have no effect |
| E-Stop pressed | All outputs FALSE, SFC resets to Init |
| Cycle Stop pressed | Finishes cycle, stops at home |

---

## 📐 Variable Naming Convention

| Prefix | Type | Example |
|--------|------|---------|
| `ix` | Digital Input | `ixSliderAout` |
| `qx` | Digital Output | `qxSliderAout` |
| `mx` | Internal memory BOOL | `mxAutoRunOk` |
| `fb` | Function Block instance | `fbBlink` |
| `mi` | Internal memory INT | `miSeq` |

---

## 🚀 How to Run

1. Open `TwinCAT Project Drill.sln` in TwinCAT 3 XAE
2. Select runtime: `<UmRT_Default>`
3. Click **Activate Configuration** → OK
4. Start the PLC
5. Open the **Visualization** in PLCSim to interact with the machine

---

## 📋 Prerequisites

- TwinCAT 3 XAE (version 3.1 or later)
- Windows PC with TwinCAT runtime
- No external hardware required — fully simulated

---

## 📚 References

- EU Machinery Regulation (EU) 2023/1230
- IEC 61131-3 Programming Languages Standard
- University West — Lecture 5: Programming Structure

---

## 👤 Author

**Ash**  
University West — Industrial Automation Program  
2025
