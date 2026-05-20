# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Klipper firmware configuration for a **QIDI Plus4 CoreXY 3D printer** with a 4-gate **Happy Hare MMU** (multi-material unit). The stack is: Klipper (motion firmware) + Moonraker (API/backend) + Mainsail (web UI) + FreeDi (QIDI LCD UI).

## Applying Configuration Changes

Changes to `.cfg` files take effect after restarting Klipper. From Mainsail UI or via SSH:

```bash
# Reload config without full restart (works for most changes)
FIRMWARE_RESTART   # G-code command in Mainsail console

# Full Klipper service restart
sudo systemctl restart klipper

# Moonraker restart (only needed for moonraker.conf changes)
sudo systemctl restart moonraker
```

Klipper validates config on startup — syntax errors will prevent startup and appear in the Mainsail log.

## Architecture & Include Hierarchy

`printer.cfg` is the root config and includes everything else:

```
printer.cfg
├── bunnybox_macros.cfg       # Waste bin, filament cutting, nozzle cleaning hardware
├── mmu/base/*.cfg            # Happy Hare MMU core (hardware, params, sequences, state)
├── mmu/optional/client_macros.cfg
├── mmu/addons/mmu_erec_cutter.cfg   # Automatic filament cutter addon
├── freedi.cfg                # QIDI LCD display UI and material preset buttons
├── gcode_macro.cfg           # 43+ custom G-code macros (print lifecycle, fans, movement)
├── timelapse.cfg             # Timelapse frame capture
├── preheat.cfg               # Material temperature presets
├── moonraker_obico_macros.cfg
└── shell_command.cfg
```

Hardware pin definitions and calibration values live directly in `printer.cfg`. The MMU system uses its own MCU (`mmu` serial device, STM32F401XC).

## Key Files

| File | Purpose |
|------|---------|
| `printer.cfg` | All hardware: steppers (TMC2240/2209), heaters, probing (Cartographer V3), kinematics |
| `gcode_macro.cfg` | Print lifecycle (`PRINT_START`), fan overrides (`M106`), nozzle wiping, filament ops |
| `bunnybox_macros.cfg` | BunnyBox-specific: trash bin macros, filament cutting with tension relief, nozzle cleaning |
| `freedi.cfg` | QIDI LCD presets (PLA/ABS/PETG/ASA/TPU/PA-CF), display preferences |
| `preheat.cfg` | Material preheat macros called by `freedi.cfg` presets |
| `mmu/base/mmu_parameters.cfg` | Happy Hare tuning parameters |
| `mmu/base/mmu_hardware.cfg` | MMU stepper/sensor pin assignments |
| `mmu/base/mmu_macro_vars.cfg` | MMU macro behavior variables (speeds, distances, etc.) |
| `mmu/mmu_vars.cfg` | **Auto-generated** — persistent state (gate assignments, calibration). Do not hand-edit. |
| `moonraker.conf` | Moonraker API config, update managers, Spoolman, timelapse |

## Physical Safety Zones

**Critical**: Y > 305 mm is the "danger zone" — this is where the trash/waste bin lives. Macros that move to `GO_TO_POOP_SHOOT` use Y values beyond 305. Never add arbitrary moves to Y > 305 without coordinating with waste bin state.

The sensorless homing (TMC2240 stallguard) on X/Y means aggressive motion during homing can trigger false stalls — avoid changing `[stepper_x]`/`[stepper_y]` hold/run current without re-tuning.

## MMU System

The 4-gate MMU uses gate indices 0–3. State is persisted in `mmu/mmu_vars.cfg` (auto-written by Happy Hare). The active gate assignment, filament type, and encoder calibration are all stored there. Gate changes made via Klipper macros/console are authoritative.

Key MMU entry points:
- `mmu/base/mmu_sequence.cfg` — print start/end sequences, tool-change logic
- `mmu/base/mmu_form_tip.cfg` / `mmu_cut_tip.cfg` — tip shaping and cutting behavior
- `mmu/addons/blobifier.cfg` — purge waste management
- `mmu/addons/mmu_erec_cutter.cfg` — automatic filament cutting on tool change

## Calibration Values (Do Not Change Without Re-Calibrating)

These are tuned hardware values — modifying them without running the corresponding calibration procedure will break print quality:

- `pressure_advance: 0.032` in `[extruder]`
- `[input_shaper]` X: 3HumpEI @ 76.6 Hz, Y: MZV @ 40.6 Hz
- `z_offset` in `[cartographer model default]`
- Rotation distances in `mmu_vars.cfg` (encoder and gear calibration)
- PID values for extruder, bed, and chamber heater

## Backup Convention

Timestamped backup directories (`backup_hh_YYYYMMDD_HHMMSS/`) are auto-created by Happy Hare before MMU updates. These are snapshots — do not edit files inside backup directories.

## Material Temperature Reference

| Material | Extruder | Bed | Chamber |
|----------|----------|-----|---------|
| PLA | 210°C | 60°C | — |
| PETG | 240°C | 70°C | — |
| ABS | 250°C | 95°C | 45°C |
| ASA | 250°C | 100°C | 50°C |
| TPU | 250°C | 40°C | — |
| PA-CF | 250°C | 100°C | 60°C |
