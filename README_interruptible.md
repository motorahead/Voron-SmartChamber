# Voron-SmartChamber (Interruptible Version)

Autonomous chamber temperature management for Klipper-based 3D printers.

Several approaches to bed fan control inspired this effort:
[EricZimmerman's fake heater method](https://github.com/EricZimmerman/Voron24/blob/master/config/fans.cfg) uses a `[heater_generic]` block with the chamber thermistor to drive fans via Klipper's PID controller - minimal macro complexity, but no bed protection or heatsoak gating.
[Ellis bedfans.cfg](https://github.com/VoronDesign/VoronUsers/tree/8e5067f4f6457da8a552983dd210ec48c40be2ca/printer_mods/Ellis/Bed_Fans/Klipper_Macros) introduced the macro-based approach with bed protection.
[3DPrintDemon's implementation](https://github.com/3DPrintDemon/Demon_Klipper_Essentials_Unified) expanded on that as part of a broader print management system.
[garethky's interruptible heatsoak](https://github.com/garethky/klipper-voron2.4-config/blob/mainline/printer_data/config/heatsoak.cfg). Rate-of-change heatsoak completion detection and non-blocking architecture.

SmartChamber builds on these ideas with a focus on closed-loop chamber temperature control: fans ramp proportionally toward a configurable chamber target, a stable confirmation period is required before the print starts, bed protection remains active throughout the print - not just during heatsoak - and cooldown is managed automatically based on the bed/chamber temperature delta. All logic lives in a single drop-in cfg file.

This version uses a non-blocking PAUSE/RESUME approach for heatsoak - the cancel button works natively during heatsoak.

Development supported by Kiro/Claude in addition to functional testing and code verification.

---

## Temperature Profile

A typical ABS print session - bed heat-up, chamber heatsoak, print, and cooldown:

![Temperature Profile](Temp%20Profile.png)

---

## Features

| Feature | Description |
|---------|-------------|
| **Interruptible Heatsoak** | Cancel button works natively during heatsoak - no queue blocking |
| **Slicer Fallback** | If slicer sends `CHAMBER=0` or omits it, `chamber_target_default` is used automatically |
| **Bed Protection** | Fans throttle or cut off if bed loses temp - prevents "not heating at expected rate" during heatsoak and mid-print |
| **Proportional Ramping** | Fans ramp in 5% steps with deadband control - no on/off switching |
| **Stable Confirmation** | Chamber must reach target and hold within `rate_threshold` C/min change for `chamber_reached_delay` consecutive passes - avoids false triggers from brief spikes or overshoots |
| **Heatsoak Timeout** | 30 min max wait - proceeds or cancels based on `chamber_min_start`, never hangs indefinitely |
| **Print-Time Maintenance** | Fan loop and bed protection continue autonomously throughout the entire print |
| **Adaptive Intervals** | Loop timing adjusts by state - faster when recovering, slower when stable or bleeding |
| **Auto Cooldown** | Cooldown starts on any heaters-off event - print end, cancel, or error - no manual trigger needed |
| **Cooldown Management** | Delta-based fan speed during cooldown - full speed --> slow --> off |
| **Clean State on Start** | Reset block clears stale state from previous cancelled or failed prints |
| **Single Config Block** | All tunable parameters in one `[_BEDFANVARS]` section |
| **PLA Safety** | Fans stay off entirely for low-temp materials - no interference with PLA/PETG |
| **LED Hooks** | Optional status macro triggers at heatsoak, cooldown, and complete |
| **Universal Compatibility** | Any enclosed printer (Voron 2.4, Trident, V0, etc.) - works with Nevermore, bed fans, or any `[fan_generic]` output. Adapts to hardware changes with two config lines - see [Configuration](#configuration) |

---

## Architecture

All fan logic lives in a single macro (`_BED_FAN_MANAGE`). The heatsoak uses a non-blocking `[delayed_gcode]` monitor with PAUSE/RESUME instead of a blocking loop.

**Design note:** `heating_threshold` is checked twice - once in `PRINT_START` to decide whether to call `CHAMBER_HEATSOAK`, and again inside `_BED_FAN_MANAGE` on every tick. The first gates the heatsoak process entirely. The second is a live guard that protects against the bed target dropping mid-print. Intentionally redundant.

**1. Heatsoak** - `PRINT_START` calls `CHAMBER_HEATSOAK` after bed reaches temperature:

```
PRINT_START
  +-- CHAMBER_HEATSOAK
        +-- PAUSE                     # Holds the print queue (cancel works while paused)
        +-- heatsoak_monitor          # Non-blocking delayed_gcode, fires every 15s:
              +-- _BED_FAN_MANAGE     # Ramps fans toward target, throttles if bed loses temp
              +-- Stability check     # Rate-of-change: requires chamber_reached_delay consecutive
              |                       # passes where rate < rate_threshold AND temp >= target
              +-- RESUME              # Called when stable or on timeout - releases print queue
```

**2. Print-time maintenance** - background loop keeps chamber at target throughout the print:

```
bedfanheatloop  -->  _BED_FAN_MANAGE (MODE=FAN_HEATSOAK)
  (adaptive intervals)   Maintains chamber temp, protects bed from heat loss
```

**3. Cooldown** - triggered automatically by `TURN_OFF_HEATERS`:

```
bedfancoolloop  -->  _BED_FAN_MANAGE (MODE=FAN_COOLDOWN)
  (fires every 5s)       Full speed --> slow --> off, based on bed/chamber delta
```

---

## PAUSE/RESUME Integration

This version overrides PAUSE and RESUME using `rename_existing` to make them heatsoak-aware. During heatsoak, PAUSE skips retract/park actions and RESUME skips unretract/move-back actions.

**Include order matters.** `smart_chamber_interruptible.cfg` must be included AFTER `mainsail.cfg` (or any other file that defines PAUSE/RESUME) so the overrides take precedence:

```ini
[include mainsail.cfg]              # Mainsail defines PAUSE/RESUME
[include smart_chamber_interruptible.cfg]  # Overrides PAUSE/RESUME with heatsoak awareness
```

If you have custom PAUSE/RESUME logic (filament runout retract, toolhead park, etc), add it to the `{% else %}` branches inside the overrides in `smart_chamber_interruptible.cfg`.

---

## Configuration

Edit the `[_BEDFANVARS]` block at the top of `smart_chamber_interruptible.cfg`. The key variables to set for your printer:

| Variable | Description |
|----------|-------------|
| `variable_fan` | Must match your `[fan_generic]` name exactly (case-sensitive) |
| `variable_chamber_sensor` | Must match your `[temperature_sensor]` name exactly |
| `variable_heating_threshold` | Min bed temp to enable fans - set to your ABS/ASA bed temp |
| `variable_chamber_target_default` | Fallback chamber target if slicer sends `CHAMBER=0` |
| `variable_chamber_min_start` | Min chamber temp to allow print after heatsoak timeout. `0` = disabled |
| `variable_rate_threshold` | Degrees C/min below which chamber is considered stable. Default `0.5` |
| `variable_debug` | `0` = milestones only, `1` = state changes, `2` = full per-tick output |

All other variables are pre-tuned for a Voron 2.4 350mm with 4 bed fans. Each has an inline comment in the cfg explaining its effect.

Swapping hardware only requires updating the two name variables:
```ini
variable_fan:            "Nevermore"       # was "Bed_Fans"
variable_chamber_sensor: "Chamber_Temp"    # unchanged if sensor name didn't change
```

---

## Installation

> 💡 See a complete working example at [motorahead/Voron24-350 printer.cfg](https://github.com/motorahead/Voron24-350/blob/f3062e6b5fda8e9c04e60c185e820d402831bb71/printer_data/config/printer.cfg)

### 1. File setup

1. Copy `smart_chamber_interruptible.cfg` to your Klipper config directory
2. Add to your `printer.cfg` AFTER `mainsail.cfg` (include order matters):
   ```ini
   [include mainsail.cfg]
   [include smart_chamber_interruptible.cfg]
   ```
3. Edit `[_BEDFANVARS]` to match your printer (see Configuration above)
4. If you have custom PAUSE/RESUME logic (retract, park, etc), move it into the `{% else %}` branches inside the PAUSE/RESUME overrides in `smart_chamber_interruptible.cfg`

### 2. PRINT_START

**a) At the very top of PRINT_START, add the reset block:**

```jinja
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_state VALUE=0
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_confirm_count VALUE=0
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=last_chamber_temp VALUE=0
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=heatsoak_ticks VALUE=0
```

**b) Replace your existing M190 and bed soak logic:**

> M190 is called internally - do not add it separately.

```jinja
{% if target_bed >= printer["gcode_macro _BEDFANVARS"].heating_threshold %}  # threshold set in smart_chamber.cfg
    # High-temp path: ABS/ASA - heatsoak with autonomous fan management
    SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_state VALUE={target_chamber}
    M190 S{target_bed}                    # fans stay OFF during heat-up
    {% if target_chamber > 0 %}
        CHAMBER_HEATSOAK                  # pauses print, starts non-blocking heatsoak monitor
    {% endif %}
    UPDATE_DELAYED_GCODE ID=bedfanheatloop DURATION=15
{% else %}
    # Low-temp path: PLA/PETG - no fans, fixed soak
    M190 S{target_bed}
    G4 P300000                            # 5 min soak
{% endif %}
```

### 3. PRINT_END

`TURN_OFF_HEATERS` automatically stops the heatsoak monitor (if running) and starts the cooldown loop - no extra steps needed:

```jinja
TURN_OFF_HEATERS
{% if printer["gcode_macro _BEDFANVARS"].chamber_state|float != 0 %}
    SET_DISPLAY_TEXT MSG="Print Complete - Cooling"
{% else %}
    SET_DISPLAY_TEXT MSG="Print Complete"
{% endif %}
```

---

## Debugging

Set `variable_debug` in `[_BEDFANVARS]` to control console output:

**Level 0** - milestones only (always shown):
```
[PRINT_START] Bed ready. Starting chamber heatsoak to 63C
Bed stabilizing before fans start (60s)
Chamber heatsoak complete - stable at 63.2C
Chamber heatsoak timed out - 60.1C - proceeding anyway
Bed fans: starting cooldown loop
Bed fans off: cooldown complete
```

**Level 1** - adds state changes:
```
Bed fans off: bed temp low (108.2C / 115.0C)
Bed fans slow: bed recovering (113.1C / 115.0C)
Chamber stable: 63.2C / 63.0C rate=0.18C/min (confirm 1/4)
Chamber heatsoak complete - stable at 63.2C
```

**Level 2** - adds per-tick output (useful for tuning):
```
Chamber Heatsoak: 58.4C / 63.0C rate=1.20C/min fans=75%
Cooldown: bed=114.8C chamber=63.5C fans=100%
```

### Troubleshooting

**Fans never start during heatsoak**
- Verify `variable_fan` and `variable_chamber_sensor` match your cfg names exactly (case-sensitive)
- Verify bed target is >= `variable_heating_threshold`
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
- Normal for ABS - 60-90 min is expected depending on enclosure insulation
- Raise `variable_cooling_threshold` slightly to stop the loop earlier

**Mainsail shows "Paused" during heatsoak**
- This is expected behavior - the print queue is paused while the non-blocking heatsoak monitor runs
- Display text shows "Chamber Heatsoak: XXc" to indicate what's happening
- RESUME is called automatically when heatsoak completes

**Klipper errors "option already exists" or "already renamed" on PAUSE/RESUME**
- Include order is wrong - `smart_chamber_interruptible.cfg` must be included AFTER `mainsail.cfg`
- If another cfg file also overrides PAUSE/RESUME with `rename_existing`, only one can do it
- Solution: merge the heatsoak check into that file's PAUSE/RESUME instead

**Print starts immediately without waiting for heatsoak**
- PAUSE may not be firing correctly - check include order
- Verify `CHAMBER_HEATSOAK` is being called in PRINT_START
- Set `variable_debug: 2` and watch for "Bed stabilizing" message

**RESUME triggers retract/park unexpectedly after heatsoak**
- The RESUME override's `chamber_state == -1` check may not be matching
- Verify `chamber_state` is being set to `-1` by the monitor before RESUME is called
- Set `variable_debug: 1` to confirm "Chamber heatsoak complete" appears before resume

### Testing Macros

```gcode
TEST_BED_FANS       # Starts the heatsoak fan loop without a print
                    # Uses chamber_target_default as the target
                    # Safe to run with bed heated

TEST_HEATSOAK       # Full heatsoak test - PAUSE/monitor/RESUME flow
                    # Requires bed heated above heating_threshold
                    # Verifies cancel works, stability confirms, RESUME fires

TEST_FAN_COOLDOWN   # Triggers TURN_OFF_HEATERS --> starts cooldown loop
                    # Observe fan behavior and console output
```

---

## Tested On

- Voron 2.4 350mm - Leviathan V1.1, SB2209 CANBUS, 4x bed fans (THE FILTER)

---

## License

[GPL v3](LICENSE)
