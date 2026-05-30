# Voron-SmartChamber

Autonomous chamber temperature management for Klipper-based 3D printers.

Several approaches to bed fan control exist in the Voron community.
[EricZimmerman's fake heater method](https://github.com/EricZimmerman/Voron24/blob/master/config/fans.cfg) uses a `[heater_generic]` block with the chamber thermistor to drive fans via Klipper's PID controller — minimal macro complexity, but no bed protection or heatsoak gating.
[Ellis bedfans.cfg](https://github.com/VoronDesign/VoronUsers/tree/8e5067f4f6457da8a552983dd210ec48c40be2ca/printer_mods/Ellis/Bed_Fans/Klipper_Macros) introduced the macro-based approach with bed protection
[3DPrintDemon's implementation](https://github.com/3DPrintDemon/Demon_Klipper_Essentials_Unified) expanded on that as part of a broader print management system.

SmartChamber builds on these ideas with a focus on closed-loop chamber temperature control:
Fans ramp proportionally toward a configurable chamber target rather than switching between fixed speeds
A stable confirmation period is required before the print starts
Bed protection remains active throughout the print — not just during heatsoak — and cooldown is managed automatically based on the bed/chamber temperature delta.
All logic lives in a single drop-in cfg file.

Development supported by Kiro/Claude in addition to functional testing and code verification.

---

## Temperature Profile

A typical ABS print session — bed heat-up, chamber heatsoak, print, and cooldown:

![Temperature Profile](Temp%20Profile.png)

---

## Features

| Feature | Description |
|---------|-------------|
| **Bed Protection** | Fans throttle or cut off if bed loses temp during heatsoak — prevents "not heating at expected rate" |
| **Proportional Ramping** | Fans ramp in 5% steps with deadband control — no on/off switching |
| **Stable Confirmation** | Chamber must hold target for 60s before print starts — avoids false triggers |
| **Print-Time Maintenance** | Fan loop continues autonomously throughout the print, bed protection included |
| **Cooldown Management** | Delta-based fan speed during cooldown — full speed → slow → off |
| **Min Start Temp** | Optional minimum chamber temp — cancels print if not reached after timeout |
| **PLA Safety** | Fans stay off entirely for low-temp materials |
| **LED Hooks** | Optional status macro triggers at heatsoak, cooldown, and complete |

---

## Architecture

All fan logic lives in a single macro (`_BED_FAN_MANAGE`) called by minimal `[delayed_gcode]` blocks — one line each. This avoids Klipper's silent failure issue with complex Jinja inside delayed_gcode bodies.

**1. Heatsoak** — `PRINT_START` calls `CHAMBER_HEATSOAK` after bed reaches temperature:

```
PRINT_START
  └── CHAMBER_HEATSOAK
        ├── _BED_FAN_MANAGE           # Ramps fans toward target, throttles if bed loses temp
        ├── _CHAMBER_READY            # Requires chamber_reached_delay consecutive passes at target
        │                             # (default 4 × 15s = 60s) before declaring stable
        ├── _CHAMBER_HEATSOAK_RESULT  # Reports complete/timed out. Cancels if below chamber_min_start
        └── _DRAIN_LOOP_WAIT          # 15s per tick during warmup, 100ms once stable to exit fast
```

**2. Print-time maintenance** — background loop keeps chamber at target throughout the print:

```
bedfanheatloop  ──►  _BED_FAN_MANAGE (HEATSOAK mode)
  (fires every 15–30s)   Maintains chamber temp, protects bed from heat loss
```

**3. Cooldown** — triggered automatically by `TURN_OFF_HEATERS`:

```
bedfancoolloop  ──►  _BED_FAN_MANAGE (COOLDOWN mode)
  (fires every 5s)       Full speed → slow → off, based on bed/chamber delta
```

---

## Installation

1. Copy `smart_chamber.cfg` to your Klipper config directory
2. Add to your `printer.cfg`:
   ```
   [include smart_chamber.cfg]
   ```
3. Edit `[_BEDFANVARS]` to match your printer (see Configuration below)
4. Update your `PRINT_START` and `PRINT_END` macros (see Setup below)

---

## Configuration

Edit the `[_BEDFANVARS]` block at the top of `smart_chamber.cfg`. The key variables to set for your printer:

| Variable | Description |
|----------|-------------|
| `variable_fan` | Must match your `[fan_generic]` name exactly (case-sensitive) |
| `variable_chamber_sensor` | Must match your `[temperature_sensor]` name exactly |
| `variable_heating_threshold` | Min bed temp to enable fans — set to your ABS/ASA bed temp |
| `variable_chamber_target_default` | Fallback chamber target if slicer sends `CHAMBER=0` |
| `variable_chamber_min_start` | Min chamber temp to allow print after heatsoak timeout. `0` = disabled |
| `variable_debug` | `0` = milestones only, `1` = state changes, `2` = full per-tick output |

All other variables are pre-tuned for a Voron 2.4 350mm with 4 bed fans. Each has an inline comment in the cfg explaining its effect.

---

## PRINT_START Setup

> 💡 See a complete working `PRINT_START` implementation at [motorahead/Voron24-350 printer.cfg](https://github.com/motorahead/Voron24-350/blob/f3062e6b5fda8e9c04e60c185e820d402831bb71/printer_data/config/printer.cfg)

### a) At the very top of your PRINT_START, add the reset block:

```jinja
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_target VALUE=0
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_confirm_count VALUE=0
```

### b) Replace your existing M190 and bed soak logic with this block:

> M190 is called internally — do not add it separately.

```jinja
{% if target_bed >= printer["gcode_macro _BEDFANVARS"].heating_threshold %}
    # High-temp path: ABS/ASA — heatsoak with autonomous fan management
    SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_target VALUE={target_chamber}
    M190 S{target_bed}                    # fans stay OFF during heat-up
    {% if target_chamber > 0 %}
        CHAMBER_HEATSOAK                  # ramps fans, waits for stable confirmation, then proceeds
    {% endif %}
    UPDATE_DELAYED_GCODE ID=bedfanheatloop DURATION=15
{% else %}
    # Low-temp path: PLA/PETG — no fans, fixed soak
    M190 S{target_bed}
    G4 P300000                            # 5 min soak
{% endif %}
```

## PRINT_END Setup

`TURN_OFF_HEATERS` automatically starts the cooldown loop — no extra steps needed. Optionally show a status message:

```jinja
TURN_OFF_HEATERS
{% if printer["gcode_macro _BEDFANVARS"].chamber_target|float != 0 %}
    SET_DISPLAY_TEXT MSG="Print Complete - Cooling"
{% else %}
    SET_DISPLAY_TEXT MSG="Print Complete"
{% endif %}
```

---

## Debugging

Set `variable_debug` in `[_BEDFANVARS]` to control console output:

**Level 0** — milestones only (always shown):
```
[PRINT_START] Bed ready. Starting chamber heatsoak to 63C
Bed stabilizing before fans start (60s)
Chamber heatsoak complete
Chamber heatsoak timed out — 60.1C — proceeding anyway
Bed fans: starting cooldown loop
Bed fans off: cooldown complete
```

**Level 1** — adds state changes:
```
Bed fans off: bed temp low (108.2C / 115.0C)
Bed fans slow: bed recovering (113.1C / 115.0C)
Chamber at target: 63.2C / 63.0C fans=40% (confirm 1/4)
Chamber stable for 60s - proceeding
```

**Level 2** — adds per-tick output (useful for tuning):
```
Chamber Heatsoak: 58.4C / 63.0C fans=75%
Cooldown: bed=114.8C chamber=63.5C fans=100%
```

### Troubleshooting

**Fans never start during heatsoak**
- Verify `variable_fan` and `variable_chamber_sensor` match your cfg names exactly (case-sensitive)
- Verify bed target is ≥ `variable_heating_threshold`
- Set `variable_debug: 2` and watch the console

**Heatsoak times out**
- Lower `variable_chamber_target_default` if target is too high for your enclosure
- Verify the chamber sensor reads correctly in Mainsail
- Set `variable_chamber_min_start` to cancel the print if chamber is too cold to proceed safely

**Bed triggers "not heating at expected rate"**
- Lower `variable_bed_throttle_slow` to throttle fans sooner
- Lower `variable_bed_throttle_off` to cut fans sooner
- Check for enclosure drafts or leaks

**Cooldown takes a long time**
- Normal for ABS — 60–90 min is expected depending on enclosure insulation
- Raise `variable_cooling_threshold` slightly to stop the loop earlier

### Testing Macros

```gcode
TEST_BED_FANS       # Starts the heatsoak fan loop without a print
                    # Uses chamber_target_default as the target
                    # Safe to run with bed heated

TEST_FAN_COOLDOWN   # Triggers TURN_OFF_HEATERS → starts cooldown loop
                    # Observe fan behavior and console output
```

---

## Tested On

- Voron 2.4 350mm — Leviathan V1.1, SB2209 CANBUS, 4x bed fans (Nevermore)

---

## License

MIT
