---
layout: post
title: "How To: Creality Print 7.2 — Ender 3 S1 Plus Configuration Guide (PLA & PETG)"
author: "Bill Tetrault"
date: 2026-05-29
description: ""
tags: [Creality, Ender3, PLA, PETG, Tutorial, Guide, 3D, printing]
categories: [guides, tutorials, printing]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Creality Print 7.2 — Ender 3 S1 Plus Configuration Guide (PLA & PETG)

*Focused on first-layer adhesion and conservative, stable printing. Assumes stock textured PEI magnetic spring-steel plate.*

***

## Section 1 — Printer & Bed Setup

### 1.1 Select or Create a Printer Profile

Creality Print organizes everything into three preset categories: **Printer**, **Filament**, and **Process**. Work left to right across the top preset bar.[^1]

1. Launch Creality Print 7.2. In the top-left **Printer Presets** panel, click the dropdown and choose **Add / Manage Printers**.
2. In the search box type `Ender-3 S1 Plus`. Select it from the Creality list and click **Confirm**.
3. The profile pre-fills the build volume at **X 315 mm × Y 310 mm × Z 300 mm** and sets bed shape to **Rectangular**, origin **front-left**. Verify these under **Printer Presets → Edit (pencil icon) → Machine Settings**.[^1]
4. If the values need correction: set X=315, Y=310, Z=300. Confirm bed shape = Rectangular, origin = (0,0). Click **Save**.

### 1.2 Auto-Level / Mesh Features

The Ender 3 S1 Plus uses a **CR Touch** probe. Creality Print does not drive the leveling mesh directly, but your start G-code must invoke it.[^2]

5. In **Printer Presets → Edit → Machine G-code → Start G-code**, confirm these lines are present (add them if missing):
   ```
   G28          ; Auto-home all axes
   G29          ; Run CR Touch mesh leveling
   M500         ; Save mesh to EEPROM
   M420 S1 Z3   ; Load saved mesh, fade over 3 mm
   ```
6. Keep the **slicer-side Z-offset at 0**. All Z-offset tuning is done on the printer touchscreen (see Section 4).[^2]

***

## Section 2 — PLA Profile

### 2.1 Create the Profile

1. In the **Filament Presets** dropdown (top center), click **Add** → select **Generic PLA** as the base.
2. Click the **Edit (pencil)** icon on the new filament entry. Name it `PLA – S1 Plus Conservative`.
3. In the **Process Presets** dropdown (top right), click **Add** → select the closest Creality preset as a base (e.g., `0.20mm Standard`). Name it `PLA – 0.20 Conservative`.
4. To expose all settings, click **Advanced: All Object's Settings** (the expand button on the right panel).[^3]

### 2.2 Temperatures — Filament Presets → Edit → Temperatures

| Setting | Value |
|---|---|
| Nozzle temp – first layer | **210 °C** |
| Nozzle temp – other layers | **205 °C** |
| Bed temp – first layer | **60 °C** |
| Bed temp – other layers | **60 °C** |

> **Path:** Filament Presets → Edit → **Temperature** tab

These are well-established conservative starting points for PLA on a PEI surface.[^4]

### 2.3 Speeds — Process Presets → Edit → Speed Tab

| Setting | Value |
|---|---|
| First layer speed | **15 mm/s** |
| First layer infill speed | **20 mm/s** |
| Outer wall speed | **40 mm/s** |
| Inner wall speed | **40 mm/s** |
| Infill speed | **60 mm/s** |
| Top surface speed | **35 mm/s** |
| Travel speed | **120 mm/s** |

> **Path:** Process Presets → Edit → **Speed** tab[^5]

### 2.4 Cooling — Process Presets → Edit → Cooling Tab

| Setting | Value |
|---|---|
| Fan speed – first layer | **0%** |
| Fan speed – layer 2 | **30%** |
| Fan speed – layers 3+ | **100%** |
| Min layer time | **8 s** |

> **Path:** Process Presets → Edit → **Cooling** tab

Keeping the fan off for the first layer maximizes bed adhesion.[^6]

### 2.5 Quality / Layer Settings — Process Presets → Edit → Quality Tab

| Setting | Value |
|---|---|
| Layer height | **0.20 mm** |
| Initial layer height | **0.24 mm** (slightly thicker for better squish) |
| Initial layer line width | **110%** |
| Initial layer flow | **105–110%** |

> **Path:** Process Presets → Edit → **Quality** tab

A thicker initial layer and slightly increased flow improve first-layer contact area.[^7]

***

## Section 3 — PETG Profile

### 3.1 Create the Profile

1. In **Filament Presets**, click **Add** → select **Generic PETG** as the base. Name it `PETG – S1 Plus Conservative`.
2. In **Process Presets**, click **Add** → duplicate your PLA process preset. Name it `PETG – 0.20 Conservative`. All modifications below go into this PETG process preset.

### 3.2 Temperatures — Filament Presets → Edit → Temperatures

| Setting | Value |
|---|---|
| Nozzle temp – first layer | **240 °C** |
| Nozzle temp – other layers | **235 °C** |
| Bed temp – first layer | **80 °C** |
| Bed temp – other layers | **80 °C** |

> **Path:** Filament Presets → Edit → **Temperature** tab

PETG requires a bed of at least 75–80 °C for reliable adhesion; below 70 °C warping is common on prints wider than ~100 mm. The textured PEI plate provides excellent self-release when the bed cools after printing — do not attempt to remove PETG prints from a hot bed.[^6][^7]

### 3.3 Speeds — Process Presets → Edit → Speed Tab

| Setting | Value |
|---|---|
| First layer speed | **12 mm/s** |
| First layer infill speed | **15 mm/s** |
| Outer wall speed | **40 mm/s** |
| Inner wall speed | **40 mm/s** |
| Infill speed | **50 mm/s** |
| Top surface speed | **35 mm/s** |
| Travel speed | **150 mm/s** |

> **Path:** Process Presets → Edit → **Speed** tab

High travel speed (150–200 mm/s) is one of the most effective anti-stringing measures for PETG because it minimizes ooze time during moves.[^6]

### 3.4 Cooling — Process Presets → Edit → Cooling Tab

| Setting | Value |
|---|---|
| Fan speed – layers 1–3 | **0%** |
| Fan speed – layers 4+ | **30–40%** |
| Bridge fan speed | **100%** (let slicer auto-apply to bridges only) |
| Min layer time | **10 s** |

> **Path:** Process Presets → Edit → **Cooling** tab

PETG does not need aggressive cooling and reacts poorly to it on the first few layers — excessive cooling kills interlayer adhesion and causes delamination.[^7][^6]

### 3.5 Retraction (Direct Drive) — Filament Presets → Edit → Retraction

| Setting | Value |
|---|---|
| Retraction distance | **1.5 mm** |
| Retraction speed | **40 mm/s** |
| Minimum travel for retraction | **2 mm** |
| Wipe before retraction | **Enabled** |

> **Path:** Filament Presets → Edit → **Retraction** tab

The Ender 3 S1 Plus uses a direct-drive extruder. For direct drive, 1–2 mm retraction is the recommended range for PETG. Exceeding 2 mm risks pulling molten filament into the cold zone, causing heat creep clogs. Set retraction speed no higher than 45 mm/s — above 50 mm/s the extruder gear can strip PETG.[^7][^6]

### 3.6 PETG Quality / Layer Settings

| Setting | Value |
|---|---|
| Layer height | **0.20 mm** |
| Initial layer height | **0.24 mm** |
| Initial layer line width | **110%** |
| Initial layer flow | **105%** |

> **Path:** Process Presets → Edit → **Quality** tab

***

## Section 4 — Z-Offset and First-Layer Tuning (Printer Side)

All Z-offset adjustment is done on the **printer touchscreen**, not in the slicer. Leave the slicer Z-offset at 0.[^2]

### 4.1 Initial Z-Offset Setup

1. **Preheat** the printer to printing temperatures before leveling (bed expands when hot — level cold and the offset will be wrong when hot).[^8]
   - On the touchscreen: **Prepare → Preheat PLA** (or PETG).
2. **Auto-home**: On the touchscreen tap **Prepare → Home** (or the Home icon). Wait for the printer to complete homing.
3. Navigate to **Settings → Leveling**. The printer will home automatically.
4. Tap **Aux Level** (manual corner leveling). Place a standard sheet of printer paper on the bed.
5. Tap position **①** (bed center or front-left). Use the **↓ Z-offset** button to lower the nozzle until you feel slight resistance when sliding the paper — it should drag but not crinkle or tear.[^9][^2]
6. Tap positions **②–⑤** in sequence. At each corner, use the **leveling knob** (turn counterclockwise to raise the bed, clockwise to lower) until the paper resistance matches position ①.[^8]
7. Once all four corners match, tap **AUTO LVL → Start** to run the CR Touch automatic mesh. Wait for it to complete.[^2]
8. The mesh is saved in EEPROM automatically. Return to the main menu.

### 4.2 Live Z-Adjustment During the First Print

1. Slice a **100 × 100 mm square** at 0.20 mm layer height, 1 perimeter, 0% infill using your PLA or PETG profile. Export as G-code and print.
2. While the first layer is printing, observe the lines:
   - **Too far**: Filament looks round, not squished. Lines may not adhere. Lines are separated.
   - **Too close**: Filament is smeared flat, nozzle drags in previous lines, or you hear grinding.
   - **Correct**: Lines are slightly squished, touching adjacent lines, with a slight ridge visible but not raised.
3. During the first layer, tap **Settings (gear icon top bar) → Z-axis Comp.** (or "Z Adjust" depending on firmware version). Adjust in **±0.02 mm steps**: tap **Z↓** to move the nozzle closer, **Z↑** to move it away.[^10][^2]
4. When the first layer looks correct, note the final Z Comp. value. This value is automatically saved for future prints.
5. **PETG-specific note**: PETG needs slightly *more* nozzle-to-bed gap than PLA (~0.05–0.10 mm higher). If the first PETG layer looks smeared and translucent, back the Z-offset up until lines show a slightly rounded top profile.[^6]

***

## Section 5 — Raft, Brim, and Adhesion Aids

### 5.1 Finding Adhesion Settings in Creality Print 7

In Creality Print 7, brim and raft settings are **hidden by default** — you must enable Advanced mode to access them.[^3]

**To access:**
1. In **Process Presets → Edit**, click **Advanced: All Object's Settings** (the expand/unlock icon on the right-side panel).
2. In the left-side tab list, scroll down and click **Brim** (it appears as a separate tab in advanced mode).[^3]
3. Alternatively, look for **Build Plate Adhesion** within the advanced tabs.

### 5.2 Adhesion Types and When to Use Each

| Type | When to Use |
|---|---|
| **Skirt** | Default for most prints. Primes the nozzle. No adhesion benefit. |
| **Brim** | Tall narrow parts, parts with small footprints, any print that lifts at corners. |
| **Raft** | Very small contact patches, severe adhesion problems, or first-time PETG when troubleshooting. |
| **None** | Large flat prints with full-bed contact that already stick reliably. |

> **Path:** Process Presets → Edit → Advanced → **Brim** tab → **Brim Type** dropdown[^11][^3]

### 5.3 Brim Settings

| Setting | Value |
|---|---|
| Brim type | **Outer brim only** |
| Brim width | **8 mm** (5 mm minimum; 10 mm for tall/narrow parts) |
| Brim-object gap | **0.10 mm** (leaves a hairline gap for easier removal) |

> **Path:** Advanced → **Brim** tab

A brim of 8 mm adds roughly 19 lines of additional contact perimeter at the 0.4 mm standard line width, dramatically increasing hold on the first layer. For PETG, consider 10 mm brim width as a starting point.[^11]

### 5.4 Raft Settings

| Setting | Value |
|---|---|
| Raft layers (interface + base) | **3–4 total** |
| Raft margin | **5–8 mm** around the part |
| Raft first-layer speed | **12–15 mm/s** (match your first-layer speed setting) |
| Raft air gap | **0.20–0.25 mm** (enough to hold the part, allows peel-off) |

> **Path:** Advanced → **Raft** tab (if separate) or **Build Plate Adhesion → Raft** section

The air gap setting controls how easily the part separates from the raft after printing. A smaller gap (0.15 mm) gives stronger hold; a larger gap (0.30 mm) makes removal easier.[^12]

### 5.5 Physical Surface Prep Routine

Slicer adhesion settings only work when the bed surface is properly prepared. The following should be done physically before printing — these are not slicer settings.[^13][^6]

**Every 5–10 prints (or whenever adhesion weakens):**
1. Remove the magnetic PEI spring-steel plate from the printer.
2. Wash with **warm water + a drop of dish soap**. Rub with fingertips, rinse thoroughly, air dry.
3. Wipe down with **99% isopropyl alcohol (IPA)** on a lint-free cloth. Let fully evaporate before printing. This removes skin oils and residue that are the #1 cause of adhesion failure.

**For PETG specifically:**
- Apply a **thin film of PVA glue stick** (e.g., Elmer's) to the textured PEI before printing. Spread it thin and even — it acts as a **release agent**, not an adhesive, preventing PETG from chemically bonding to the PEI surface and potentially tearing it.[^13][^6]
- Do NOT attempt to remove PETG prints from a hot bed. Let the bed cool fully to room temperature (or 30 °C or below) — PETG self-releases from textured PEI as it cools.[^7]

**For stubborn PLA:**
- A thin PVA glue stick layer can also help PLA on first prints with a new or washed PEI plate, though it is usually not needed.[^14]

***

## Quick-Reference Settings Tables

### PLA — Ender 3 S1 Plus (Conservative Profile)

| Parameter | Value | Location in Creality Print 7 |
|---|---|---|
| Nozzle – first layer | 210 °C | Filament Presets → Edit → Temperature |
| Nozzle – other layers | 205 °C | Filament Presets → Edit → Temperature |
| Bed – all layers | 60 °C | Filament Presets → Edit → Temperature |
| First layer speed | 15 mm/s | Process → Edit → Speed |
| Outer/inner wall speed | 40 mm/s | Process → Edit → Speed |
| Infill speed | 60 mm/s | Process → Edit → Speed |
| Top surface speed | 35 mm/s | Process → Edit → Speed |
| Fan – first layer | 0% | Process → Edit → Cooling |
| Fan – layer 2 | 30% | Process → Edit → Cooling |
| Fan – layer 3+ | 100% | Process → Edit → Cooling |
| Initial layer height | 0.24 mm | Process → Edit → Quality |
| Initial layer flow | 105–110% | Process → Edit → Quality |
| Brim width | 8 mm | Advanced → Brim |

### PETG — Ender 3 S1 Plus (Conservative Profile)

| Parameter | Value | Location in Creality Print 7 |
|---|---|---|
| Nozzle – first layer | 240 °C | Filament Presets → Edit → Temperature |
| Nozzle – other layers | 235 °C | Filament Presets → Edit → Temperature |
| Bed – all layers | 80 °C | Filament Presets → Edit → Temperature |
| First layer speed | 12 mm/s | Process → Edit → Speed |
| Outer/inner wall speed | 40 mm/s | Process → Edit → Speed |
| Infill speed | 50 mm/s | Process → Edit → Speed |
| Travel speed | 150 mm/s | Process → Edit → Speed |
| Fan – layers 1–3 | 0% | Process → Edit → Cooling |
| Fan – layers 4+ | 30–40% | Process → Edit → Cooling |
| Bridge fan speed | 100% | Process → Edit → Cooling |
| Retraction distance | 1.5 mm | Filament Presets → Edit → Retraction |
| Retraction speed | 40 mm/s | Filament Presets → Edit → Retraction |
| Initial layer height | 0.24 mm | Process → Edit → Quality |
| Initial layer flow | 105% | Process → Edit → Quality |
| Brim width | 10 mm | Advanced → Brim |

***

## Troubleshooting First-Layer Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| First layer not sticking at all | Z too far, cold bed, dirty plate | Clean plate with IPA, increase bed temp, adjust Z-offset down 0.02–0.04 mm |
| First layer smearing / nozzle dragging | Z too close | Raise Z-offset 0.02–0.04 mm steps during live print |
| PETG won't release from bed | No glue stick used, Z too close, or bed too hot | Apply thin PVA glue stick; let bed cool fully before removing |
| PETG stringing between features | Temp too high, retraction too low, travel too slow | Lower nozzle temp 5 °C, increase travel speed to 150+ mm/s, enable wipe on retract |
| Corners lifting on PLA | Fan on too early, first layer too fast, Z offset too high | Fan off layer 1; slow to 15 mm/s; check Z squish |
| Elephant's foot (base too wide) | Z too close or bed too hot for PETG | Raise Z-offset slightly; for PETG reduce bed to 75 °C after first layer |

---

## References

1. [Creality Print Interface Layout Introduction](https://wiki.creality.com/en/software/update-released/Basic-introduction/Interface-introduction) - The main interface of the software can be divided into 8 functional areas: Menu Bar, Navigation Bar,...

2. [Ender 3 S1 Leveling Failed Fix | S1 Pro & S1 Plus - Smith3D Malaysia](https://www.smith3d.com/ender-3-s1-leveling-failed-fix-s1-pro-s1-plus/) - Prepare > Auto Home · Click “Move” – “Move Z” and set Z to 0 · Return to previous menu and click “Z-...

3. [Creality Print v7 – Brim/Adhesion settings are basically hidden ...](https://www.reddit.com/r/3Dprinting/comments/1s0az12/creality_print_v7_brimadhesion_settings_are/) - Naturally, I go looking for brim/skirt settings to improve adhesion. Except…they don't exist. Not in...

4. [Best Cura Settings & Profile for Ender 3S1,S1 Pro, PLUS - Creality](https://www.creality3dofficial.com/blogs/unboxing-product-comparison/best-cura-settings-&-profile-for-ender-3-s1) - In this article, we have compiled the basic printing setting of different pla of Ender-3S1 Series. T...

5. [How To Use A 3D Printer：Complete Creality 3D Printing Guide for ...](https://www.creality.com/blog/how-to-use-a-3d-printer) - Check slicer settings and filament quality. Print Speed: Reduce both "Default Printing Speed" and "X...

6. [PETG Print Settings Deep Dive: First Layer Adhesion, Fan Speed ...](https://blog.uavmodel.com/petg-print-settings-deep-dive-first-layer-adhesion-fan-speed-and-stringing-control-2026-guide/) - Bed temperature: 80°C for the first layer, 75-80°C for subsequent layers. Below 70°C and PETG warps ...

7. [PETG Print Settings — Temp, Bed & Troubleshooting - Overture 3D](https://overture3d.com/blogs/overture-blogs/petg-print-settings-guide) - PETG requires a bed temperature of 80–90°C for reliable first-layer adhesion. Unlike PLA, PETG can b...

8. [How To Level Your Bed on the Ender 3 S1 Pro - YouTube](https://www.youtube.com/watch?v=U7cTvLpXL6U) - In this video, I show you a step-by-step guide on how to level your bed both manually and automatica...

9. [How to Level, Set Z Offset, and Make an ABL Mesh, on an Creality ...](https://www.youtube.com/watch?v=dpF_h8aOTAI) - GreggAdventure will show you my Leveling, Z-Offset, and ABL setup, on an Ender 3 S1 PLUS & S! Pro He...

10. [Ender 3 S1 pro bed adhesion issues](https://forum.creality.com/t/ender-3-s1-pro-bed-adhesion-issues/18817) - Try hair spray (must contain pva) a light spray to the build plate lasts months and improves adhesio...

11. [Creality Print Basics - Build Plate Adhesion Types - YouTube](https://www.youtube.com/watch?v=JjlzkroHnUY) - In this video, I'll be showing 4 different types of build plate adhesions that you can use in Creali...

12. [Rafts, Skirts, and Brims Tutorial | Simplify3D](https://www.simplify3d.com/resources/articles/rafts-skirts-and-brims/) - The software also includes many settings that allow you to customize the raft for faster print times...

13. [Too strong bed adhesion on PETG : r/Ender3S1](https://www.reddit.com/r/Ender3S1/comments/1hzhgne/too_strong_bed_adhesion_on_petg/) - I cleared the glue off. Put it in my craft room at 72f degree's and not the garage. Rough leveled th...

14. [Print Parameter Settings - Creality Wiki](https://wiki.creality.com/en/cr-series/cr-10-se/quick-start-guide/print-parameter-settings) - ¶ 2. Slicing Settings · For printing with regular PLA, select Generic-PLA on the right · For printin...

