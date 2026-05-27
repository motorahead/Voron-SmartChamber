# Voron-SmartChamber

Autonomous chamber temperature management for Klipper-based 3D printers.

Inspired by [Ellis bedfans.cfg](https://github.com/VoronDesign/VoronUsers/tree/8e5067f4f6457da8a552983dd210ec48c40be2ca/printer_mods/Ellis/Bed_Fans/Klipper_Macros) and [3DPrintDemon's bed fan monitor](https://github.com/3DPrintDemon/Demon_Klipper_Essentials_Unified).  
Development supported by Kiro/Claude in addition to functional testing and code verification.

---

## Features

### Heat-Up Efficiency — "Not Heating at Expected Rate" Protection
- Fans stay **OFF** during M190 bed heat-up to prioritize bed heating
- Configurable stabilization delay before fans start after bed reaches target
- Fans throttle to slow or off if bed drops below target during heatsoak

### Proportional Chamber Temperature Management
- Fans ramp up in configurable steps (default 5%) until chamber target is reached
- **30 second stable confirmation** required before proceeding to print — avoids false triggers
- Adaptive loop timing: faster checks when recovering, slower when stable (damping)
- Deadband threshold prevents unnecessary fan adjustments in the sweet spot

### Dynamic Print-Time Maintenance
- Bed fan loop continues autonomously during the entire print
- Bed protection logic remains active — throttles fans if bed loses temp mid-print
- Self-rescheduling loop with adaptive intervals based on current state

### Preventative Print Warping via Cooldown Management
- Full speed fans until bed/chamber delta closes to defined threshold
- Slows fans as temperatures converge — safer for ABS warping prevention
- Auto-shutdown once bed reaches defined cooling threshold

### PLA / Low-Temp Safety
- Fans stay off entirely when bed target is below heating threshold
- No interference with PLA/PETG prints

### LED Integration (optional)
- Hooks for LED status macros at key moments: chamber heatsoak, cooldown start, cooldown complete
- Compatible with `stealthburner_led_effects.cfg`

### Universal Compatibility
- Works with any number of fans (1 to N) on a single `[fan_generic]` output
- Compatible with any enclosed printer: Voron 2.4, Trident, V0, etc.
- All parameters tunable from a single `[_BEDFANVARS]` configuration block

---

## Architecture

All fan logic lives in a single macro (`_BED_FAN_MANAGE`) rather than spread across multiple `[delayed_gcode]` blocks. This is intentional — Klipper can silently fail complex Jinja logic inside delayed_gcode bodies, so those blocks are kept to a single line each. Everything meaningful happens in the regular gcode macros below.

### How a print flows

**1. Before the print — Heatsoak**

`PRINT_START` calls `CHAMBER_HEATSOAK` after the bed reaches temperature. This runs a blocking loop (up to 30 min) that repeatedly calls three helpers:

```
PRINT_START
  └── CHAMBER_HEATSOAK
        ├── _BED_FAN_MANAGE     # Adjusts fan speed: ramps up toward target, throttles back
        │                       # if bed loses temp, or bleeds speed if chamber overshoots
        ├── _CHAMBER_READY      # Checks if chamber is at target. Requires 2 consecutive
        │                       # passes (~30s) before declaring stable — avoids false triggers
        │                       # from brief temperature spikes. Resets counter if temp drops.
        ├── _CHAMBER_HEATSOAK_RESULT  # Reports "complete" or "timed out" to console
        └── _DRAIN_LOOP_WAIT    # Waits 15s per tick during warmup; switches to 100ms
                                # once chamber is confirmed stable, so the loop exits fast
```

**2. During the print — Maintenance loop**

Once heatsoak completes, a background timer fires every 15–30s to keep the chamber at target throughout the print. The same fan logic runs, including bed protection.

```
bedfanheatloop  ──►  _BED_FAN_MANAGE (HEATSOAK mode)
  (fires every 15–30s)   Maintains chamber temp, protects bed from heat loss
```

**3. After the print — Cooldown**

`TURN_OFF_HEATERS` (called by `PRINT_END`) cancels the heat loop and starts the cooldown loop. Fans run at full speed until the bed/chamber temperature gap closes, then slow down, then shut off completely once the bed is cool enough.

```
bedfancoolloop  ──►  _BED_FAN_MANAGE (COOLDOWN mode)
  (fires every 5s)       Full speed → slow → off, based on bed/chamber delta
                         Stops when bed drops below cooling_threshold (default 40°C)
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
| `variable_heating_threshold` | Min bed temp to enable fans — set to your ABS/ASA temp (default `90`) |

All other variables are pre-tuned for a Voron 2.4 350mm with 4 bed fans. Adjust as needed — each has an inline comment explaining its effect.

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
{% if target_bed >= 90 %}
    # High-temp path: ABS/ASA - heatsoak with autonomous fan management
    SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_target VALUE={target_chamber}
    M190 S{target_bed}                    # fans stay OFF during heat-up
    {% if target_chamber > 0 %}
        CHAMBER_HEATSOAK                  # ramps fans, waits for 30s stable, then proceeds
    {% endif %}
    UPDATE_DELAYED_GCODE ID=bedfanheatloop DURATION=15
{% else %}
    # Low-temp path: PLA/PETG - no fans, fixed soak
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

### Debug Verbosity Levels

Set `variable_debug` in `[_BEDFANVARS]` to control console output:

```ini
variable_debug: 0   # Chamber/Fan milestones only
variable_debug: 1   # Chamber/Fan state changes
variable_debug: 2   # Chamber/Fan full per-tick output
```

### What Each Level Shows

**Level 0 — Chamber/Fan milestones only**
These messages always print regardless of debug level:
```
[PRINT_START] Bed ready. Starting chamber heatsoak to 63C
Bed stabilizing before fans start (60s)
Chamber heatsoak complete
Chamber heatsoak timed out after 30 min - proceeding anyway
Bed fans: starting cooldown loop
Bed fans off: cooldown complete
```

**Level 1 — Chamber/Fan state changes**
Adds a message when the fan logic reacts to a change:
```
Bed fans off: bed temp low (108.2C / 115.0C)
Bed fans slow: bed recovering (113.1C / 115.0C)
Bed fans off: bed target below threshold
Chamber at target: 63.2C / 63.0C fans=40% (confirm 1/2)
Chamber at target: 63.2C / 63.0C fans=40% (confirm 2/2)
Chamber stable for 30s - proceeding
```

**Level 2 — Chamber/Fan full per-tick output**
Adds a message every loop tick — useful for tuning intervals and fan ramp behavior:
```
Chamber Heatsoak: 58.4C / 63.0C fans=75%
Cooldown: bed=114.8C chamber=63.5C fans=100%
```

### Reading the Console

Messages appear in:
- **Mainsail** → Console tab
- **KlipperScreen** → popup notifications (echo messages)
- **klippy.log** → searchable with `grep "M118" ~/printer_data/logs/klippy.log`
- **Voron Log Analyzer** → Milestone Timeline table and chart annotations

### Troubleshooting

**Fans never start during heatsoak**
- Check `variable_fan` matches your `[fan_generic]` name exactly (case-sensitive)
- Check `variable_chamber_sensor` matches your `[temperature_sensor]` name exactly
- Verify bed target is ≥ `variable_heating_threshold` (default 90°C)
- Set `variable_debug: 2` and watch the console for output

**Heatsoak times out (30 min)**
- Chamber target may be too high for your enclosure
- Lower `variable_chamber_target_default` and try again
- Verify the chamber sensor is reading correctly in Mainsail
- Set `variable_debug: 2` to watch the temp climb tick by tick

**Bed still triggers "not heating at expected rate"**
- The script throttles fans when bed loses temp, but your thresholds may need tightening
- Lower `variable_bed_throttle_slow` (e.g. `0.5°C`) to throttle fans sooner
- Lower `variable_bed_throttle_off` (e.g. `2.0°C`) to kill fans sooner
- Check for enclosure drafts or leaks

**Cooldown takes a very long time**
- Normal for ABS — 60–90 min is expected depending on enclosure insulation
- Use the Voron Log Analyzer to visualize the cooldown curve and verify fans are running
- Check `variable_cooling_threshold` — raise it slightly if you want cooldown to stop earlier

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
