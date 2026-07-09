---
layout: post
title: "How To: OrcaSlicer Configuration Guide — Creality Ender 3 S1 Plus (PLA & PETG)"
author: "Bill Tetrault"
date: 2026-05-29
description: ""
tags: [OrcaSlicer, Creality, Ender3, PLA, PETG, Tutorial, Guide, 3D, printing]
categories: [guides, tutorials, printing]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# OrcaSlicer Configuration Guide — Creality Ender 3 S1 Plus (PLA & PETG)

This guide provides a conservative, stable OrcaSlicer setup for a Creality Ender 3 S1 Plus with stock Marlin firmware, CR Touch automatic bed leveling, direct-drive extruder, and textured PEI build plate. It adds a practical calibration playlist for PLA and PETG so the slicer profiles can be tuned in a repeatable order for reliable first layers and steady print quality.[cite:3][cite:4][cite:6][cite:10][cite:14]

## Printer and bed setup

### Add the printer profile

1. Launch OrcaSlicer.
2. In the top-left printer selector, click the edit icon next to the active printer to open **Printer Settings**.[cite:11]
3. Click **Add Printer** (or the **+** icon), search for `Ender-3 S1`, and choose the closest stock profile available, typically **Ender-3 S1 Pro** if an S1 Plus profile is not present.[cite:3][cite:5]
4. Open **Printer Settings → Basic Information** and set:
   - Printable area X: **315 mm**
   - Printable area Y: **310 mm**
   - Printable height Z: **300 mm**
   - Bed shape: **Rectangular**
   - G-code flavor: **Marlin**.[cite:4]
5. Save the printer as **Ender-3 S1 Plus**.

### Configure machine start and end G-code

1. Open **Printer Settings → Machine G-code**.
2. In **Machine start G-code**, use:

```gcode
M413 S1                          ; Enable power-loss recovery
G90                              ; Absolute positioning
M83                              ; Extruder relative mode
M140 S[bed_temperature_initial_layer_single]  ; Start heating bed
M104 S150                        ; Pre-heat nozzle to 150°C
G4 S10                           ; Brief pause
G28                              ; Home all axes
M420 L1 S1 T0 V Z75.0            ; Load saved bed mesh, fade over 75 mm
G1 Z50 F240
G1 X2 Y10 F3000
M104 S[nozzle_temperature_initial_layer]
M190 S[bed_temperature_initial_layer_single]
M109 S[nozzle_temperature_initial_layer]
G1 Z0.28 F240
G92 E0
G1 Y140 E10 F1500
G1 X2.3 F5000
G92 E0
G1 Y10 E10 F1200
G92 E0
```

This start sequence heats safely, homes the machine, loads the saved bed mesh, and lays down a consistent purge line before the print begins.[cite:10][cite:14]

3. In **Machine end G-code**, use:

```gcode
G91
G1 Z0.2 E-2 F2400
G1 X5 Y5 F3000
G1 Z10
G90
G1 X0 Y200
M140 S0
M104 S0
M106 S0
M84
```

4. Keep **Printer Settings → Basic Information → Z-offset** at **0.00 mm** as a starting point and do final tuning with calibration prints and live adjustments.[cite:10][cite:14]

### Prepare the textured PEI plate

1. Wash the removable PEI plate with warm water and a small amount of dish soap when adhesion declines.
2. Dry it fully, then wipe with 99% IPA on a lint-free cloth before printing.[cite:6][cite:11]
3. For PETG, apply a thin layer of PVA glue stick before printing so the part releases cleanly and does not bond too aggressively to PEI.[cite:6]
4. Let PETG prints cool fully before removal because release improves substantially as the bed temperature drops.[cite:6]

## PLA profile

### Create the PLA filament profile

1. In the top bar, open the **Filament** selector.
2. Click the edit icon, choose **Add**, and select **Generic PLA** as the base preset.[cite:14]
3. Name it **PLA – S1 Plus Conservative** and save.

### PLA temperatures

Open **Filament Settings → Basic Information** and set:

| Setting | Value |
|---|---:|
| Nozzle temperature – first layer | 210 °C |
| Nozzle temperature – other layers | 205 °C |
| Bed temperature – first layer | 60 °C |
| Bed temperature – other layers | 60 °C |

These values are conservative and favor first-layer consistency on textured PEI over outright speed.[cite:14]

### PLA cooling

Open **Filament Settings → Cooling** and set:

| Setting | Value |
|---|---:|
| Enable cooling | Yes |
| Fan speed – first layer | 0% |
| Fan speed – other layers | 100% |
| No cooling for first N layers | 1 |
| Slow down for cooling | Yes |
| Minimum layer time | 8 s |
| Fan speed for overhangs | 100% |

Turning off part cooling for the first layer is one of the most reliable ways to improve PLA adhesion on PEI.[cite:6][cite:14]

### PLA retraction

Open **Filament Settings → Advanced** and enable the retraction override.

| Setting | Value |
|---|---:|
| Retraction distance | 3.5 mm |
| Retraction speed | 45 mm/s |
| Wipe before retraction | Enabled |

This is a safe starting point for a direct-drive setup and should be refined with OrcaSlicer’s Retraction Test.[cite:10][cite:14]

### PLA process settings

Create a process preset from **0.20mm Standard** and name it **PLA – 0.20 Conservative**.[cite:14]

Open **Process Settings → Quality** and set:

| Setting | Value |
|---|---:|
| Layer height | 0.20 mm |
| Initial layer height | 0.24 mm |
| Initial layer line width | 110% |
| Seam position | Aligned or Back |
| Walls | 3 |
| Top shell layers | 5 |
| Bottom shell layers | 4 |

The thicker first layer and 110% line width improve bed contact and help small variations in leveling print more consistently.[cite:6][cite:14]

Open **Process Settings → Speed** and set:

| Setting | Value |
|---|---:|
| First layer speed | 15 mm/s |
| First layer infill speed | 20 mm/s |
| Outer wall speed | 40 mm/s |
| Inner wall speed | 40 mm/s |
| Infill speed | 60 mm/s |
| Top surface speed | 35 mm/s |
| Travel speed | 120 mm/s |
| Initial layer travel speed | 60 mm/s |

### PLA flow and pressure settings

Open **Filament Settings → Basic Information** and set:

| Setting | Value |
|---|---:|
| Flow ratio | 1.00 to start, then calibrate |
| Max volumetric speed | 12–14 mm³/s |

Then run **Calibration → Flow Ratio** and **Calibration → Pressure Advance** to finalize the actual values used by that PLA spool.[cite:10][cite:14]

## PETG profile

### Create the PETG filament profile

1. In the **Filament** selector, click edit, choose **Add**, and start from **Generic PETG**.[cite:14]
2. Name it **PETG – S1 Plus Conservative** and save.
3. Duplicate the PLA process preset and save it as **PETG – 0.20 Conservative**.

### PETG temperatures

Open **Filament Settings → Basic Information** and set:

| Setting | Value |
|---|---:|
| Nozzle temperature – first layer | 240 °C |
| Nozzle temperature – other layers | 235 °C |
| Bed temperature – first layer | 80 °C |
| Bed temperature – other layers | 80 °C |

PETG usually needs a hotter bed than PLA to stay flat and avoid corner lift on larger parts.[cite:6][cite:14]

### PETG cooling

Open **Filament Settings → Cooling** and set:

| Setting | Value |
|---|---:|
| Fan speed – first layer | 0% |
| No cooling for first N layers | 3 |
| Fan speed – layers 4+ | 35% |
| Bridge fan speed | 80% |
| Minimum layer time | 10 s |

PETG generally prints best with restrained cooling because high fan speeds can weaken layer bonding.[cite:6][cite:14]

### PETG retraction

Open **Filament Settings → Advanced** and set:

| Setting | Value |
|---|---:|
| Retraction distance | 1.5 mm |
| Retraction speed | 40 mm/s |
| Wipe before retraction | Enabled |
| Z-hop height | 0.20 mm |

For direct-drive PETG, short retractions are safer and usually reduce the risk of heat-creep clogs.[cite:10][cite:14]

### PETG process settings

Open **Process Settings → Quality** and set:

| Setting | Value |
|---|---:|
| Layer height | 0.20 mm |
| Initial layer height | 0.24 mm |
| Initial layer line width | 110% |
| Walls | 3 |
| Top shell layers | 5 |
| Bottom shell layers | 4 |

Open **Process Settings → Speed** and set:

| Setting | Value |
|---|---:|
| First layer speed | 12 mm/s |
| First layer infill speed | 15 mm/s |
| Outer wall speed | 40 mm/s |
| Inner wall speed | 40 mm/s |
| Infill speed | 50 mm/s |
| Top surface speed | 35 mm/s |
| Travel speed | 150 mm/s |
| Initial layer travel speed | 80 mm/s |

The higher travel speed helps reduce PETG stringing by reducing ooze time during moves.[cite:12]

### PETG Z-offset note

PETG usually prefers slightly less squish than PLA. A practical starting point is to run the PETG first-layer calibration and, if needed, raise the effective Z-offset by about **+0.05 mm** compared with a well-tuned PLA profile.[cite:6][cite:14]

## Z-offset and first-layer tuning

### Establish the hardware baseline

1. On the printer touchscreen, preheat the machine to the actual material temperatures before leveling.
2. Run **Prepare → Home**.
3. Open **Settings → Leveling → Aux Level** and use the paper test at the four corners and center until resistance feels even everywhere.[cite:4]
4. Run **AUTO LVL → Start** so the CR Touch mesh is measured and stored.[cite:4][cite:11]

### Fine-tune with OrcaSlicer calibration

1. In OrcaSlicer, open **Calibration → Z Offset** or **Calibration → First Layer** depending on the version.[cite:14]
2. Generate the test patch using the exact filament and process profile you plan to print with.
3. Inspect the result:
   - Too high: lines are round and separate.
   - Too low: lines smear and the nozzle drags.
   - Correct: lines lightly squash together with a smooth top surface.[cite:10][cite:14]
4. Adjust in **Printer Settings → Basic Information → Z-offset** or live on the printer in steps of **0.02 mm** and rerun as needed.[cite:14]

## Raft, brim, and adhesion aids

### Find the adhesion controls

1. Open the relevant process preset.
2. Switch OrcaSlicer to **Expert** mode if required.
3. Open **Process Settings → Others → Bed Adhesion** to find skirt, brim, and raft controls.[cite:6][cite:9][cite:13]

### Recommended brim settings

Use a brim before a raft in most cases because it uses less material and preserves the model bottom surface more cleanly.[cite:9]

| Setting | PLA | PETG |
|---|---:|---:|
| Brim type | Outer brim only | Outer brim only |
| Brim width | 8 mm | 10 mm |
| Brim-object gap | 0.10 mm | 0.10 mm |

These values are a stable starting point for narrow parts, small footprints, and parts that tend to lift at the corners.[cite:13]

### Recommended raft settings

Use a raft only when the model has very little bed contact or you are troubleshooting severe adhesion problems.[cite:9]

| Setting | Value |
|---|---:|
| Raft layers | 3–4 |
| Raft expansion | 5–8 mm |
| Raft contact Z distance | 0.20–0.25 mm |
| Raft first layer speed | 12–15 mm/s |

A smaller raft contact gap improves bottom support, while a larger one removes more easily but sacrifices surface finish.[cite:9]

## Calibration playlist

The most repeatable workflow is to tune each filament in a fixed order so later tests are not skewed by earlier errors. OrcaSlicer’s own calibration set supports a sequence of temperature, flow, pressure advance, retraction, and Z-offset tuning.[cite:10][cite:14]

### PLA playlist

1. **Hot level + saved mesh**: manual corner level hot, then run CR Touch auto-level and save the mesh.[cite:4][cite:18]
2. **Temperature Tower**: choose the cleanest section with strong bridges, low stringing, and good layer bonding, then set first-layer and normal-layer nozzle temperatures from that result.[cite:20][cite:14]
3. **Flow Ratio**: use YOLO or Pass 1+2, then enter the final flow ratio in **Filament Settings → Basic Information**.[cite:14][cite:18]
4. **Pressure Advance**: choose the value with the cleanest corners and the most even line width through speed changes, then save it under **Filament Settings → Advanced**.[cite:10][cite:12]
5. **Retraction Test**: reduce or increase retraction until stringing drops without restart gaps, then save the distance and speed in **Filament Settings → Advanced**.[cite:10][cite:19]
6. **Z Offset / First Layer**: run the first-layer patch last, because it is easiest to judge once temperature, flow, and extrusion dynamics are already tuned.[cite:14]

**Expected outcomes for PLA**

- Smooth walls with crisp corners and limited ghosting from extrusion transitions.[cite:10][cite:12]
- First layer that sticks without elephant’s foot or nozzle scraping.[cite:10][cite:14]
- Minimal wispy strings on travel moves after retraction is tuned.[cite:10][cite:19]

**When to rerun PLA calibration**

- Rerun **Temperature Tower** and **Flow Ratio** for each new PLA brand or formulation.[cite:18][cite:19]
- Rerun **Pressure Advance** and **Retraction** after nozzle, hotend, or extruder changes.[cite:14][cite:18]
- Rerun **Z Offset** after re-leveling the bed, swapping the plate, or changing nozzle length.[cite:14]

### PETG playlist

1. **Hot level + saved mesh**: repeat the same leveling routine at PETG temperatures because bed expansion changes the geometry slightly.[cite:4][cite:18]
2. **Temperature Tower**: look for the best compromise between layer bonding and manageable stringing, then save PETG nozzle temperatures.[cite:20][cite:22]
3. **Flow Ratio**: run the PETG flow test and save the result in the PETG filament profile.[cite:14][cite:22]
4. **Pressure Advance**: select the PA result that reduces bulged corners and keeps wall thickness consistent on speed transitions.[cite:10][cite:12]
5. **Retraction Test**: tune retraction within the short direct-drive PETG range so stringing improves without triggering restart under-extrusion or heat creep.[cite:19][cite:22]
6. **Z Offset / First Layer**: finish by tuning first-layer squish, usually with slightly less squish than PLA.[cite:6][cite:14]

**Expected outcomes for PETG**

- Better corner quality and fewer blobs once pressure advance is tuned.[cite:10][cite:12]
- Noticeably reduced strings after retraction and travel settings are dialed in.[cite:12][cite:19]
- First layer that adheres reliably but still releases cleanly after cooling.[cite:6]

**When to rerun PETG calibration**

- Rerun **Temperature Tower** and **Flow Ratio** for new PETG brands or formulations.[cite:19][cite:22]
- Rerun **Retraction** when stringing increases or after changing nozzle size.[cite:19]
- Rerun **Z Offset** when PETG starts sticking too aggressively or corners begin lifting.[cite:6][cite:14]

## Quick-reference settings

### PLA quick reference

| Parameter | Value | OrcaSlicer path |
|---|---:|---|
| Nozzle – first layer | 210 °C | Filament Settings → Basic Information |
| Nozzle – other layers | 205 °C | Filament Settings → Basic Information |
| Bed – all layers | 60 °C | Filament Settings → Basic Information |
| First layer speed | 15 mm/s | Process Settings → Speed |
| Outer/inner wall speed | 40 mm/s | Process Settings → Speed |
| Infill speed | 60 mm/s | Process Settings → Speed |
| Travel speed | 120 mm/s | Process Settings → Speed |
| Fan – first layer | 0% | Filament Settings → Cooling |
| Fan – other layers | 100% | Filament Settings → Cooling |
| Initial layer height | 0.24 mm | Process Settings → Quality |
| Initial layer line width | 110% | Process Settings → Quality |
| Retraction distance | 3.5 mm start point | Filament Settings → Advanced |
| Retraction speed | 45 mm/s | Filament Settings → Advanced |
| Brim width | 8 mm | Process Settings → Others → Bed Adhesion |

### PETG quick reference

| Parameter | Value | OrcaSlicer path |
|---|---:|---|
| Nozzle – first layer | 240 °C | Filament Settings → Basic Information |
| Nozzle – other layers | 235 °C | Filament Settings → Basic Information |
| Bed – all layers | 80 °C | Filament Settings → Basic Information |
| First layer speed | 12 mm/s | Process Settings → Speed |
| Outer/inner wall speed | 40 mm/s | Process Settings → Speed |
| Infill speed | 50 mm/s | Process Settings → Speed |
| Travel speed | 150 mm/s | Process Settings → Speed |
| Fan – layers 1–3 | 0% | Filament Settings → Cooling |
| Fan – layers 4+ | 35% | Filament Settings → Cooling |
| Bridge fan speed | 80% | Filament Settings → Cooling |
| Retraction distance | 1.5 mm start point | Filament Settings → Advanced |
| Retraction speed | 40 mm/s | Filament Settings → Advanced |
| Z-hop height | 0.20 mm | Filament Settings → Advanced |
| Brim width | 10 mm | Process Settings → Others → Bed Adhesion |
