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

## Files

| File | Description |
|------|-------------|
| `smart_chamber.cfg` | Main script — include this in your `printer.cfg` |
| `stealthburner_led_effects.cfg` | Optional LED effects companion file |

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

Edit the `[_BEDFANVARS]` block at the top of `smart_chamber.cfg`:

```ini
variable_fan:                    "Bed_Fans"     # must match your [fan_generic] name
variable_chamber_sensor:         "Chamber_Temp" # must match your [temperature_sensor] name
variable_heating_threshold:      90             # min bed temp to enable fans (skips PLA)
variable_chamber_target_default: 63             # fallback if slicer sends CHAMBER=0
```

### Advanced Tuning

```ini
variable_slow:                   0.25   # min fan speed (above off_below threshold)
variable_fast:                   1.0    # max fan speed
variable_cooling_threshold:      40     # bed temp at which cooldown stops
variable_cooling_delta:          10     # bed/chamber gap to switch from fast to slow during cooldown
variable_ramp_increment:         0.05   # fan speed step per call (5%)
variable_heatsoak_start_delay:   60     # seconds to wait after bed reaches target before starting fans

variable_interval_ramp:          5      # seconds between checks when chamber is climbing
variable_interval_hold:          15     # seconds between checks in sweet spot
variable_interval_bleed:         30     # seconds between checks when bleeding (damping)
variable_interval_bed_warn:      5      # seconds between checks when bed is recovering

variable_chamber_deadband:       0.5    # degrees C either side of target = sweet spot
variable_bed_throttle_slow:      1.0    # degrees C below bed target to slow fans
variable_bed_throttle_off:       3.0    # degrees C below bed target to kill fans
```

### LED Effect Hooks (optional)

```ini
variable_chamber_heating_effect:    "status_chamber_heating"  # or "off"
variable_chamber_cooldown_effect:   "status_cooling"          # or "off"
variable_cooldown_complete_effect:  "status_part_ready"       # or "off"
```

---

## PRINT_START Setup

### a) At the very top of your PRINT_START, add the reset block:

```jinja
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_target VALUE=0
SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_confirm_count VALUE=0
```

### b) Replace your existing M190 and bed soak logic with this block:

```jinja
{% if target_bed >= 90 %}
    # High-temp path: ABS/ASA - heatsoak with autonomous fan management
    SET_GCODE_VARIABLE MACRO=_BEDFANVARS VARIABLE=chamber_target VALUE={target_chamber}
    M190 S{target_bed}                    # fans stay OFF during heat-up
    {% if target_chamber > 0 %}
        CHAMBER_HEATSOAK                  # ramp fans, wait 30s stable, restores chamber_target
    {% endif %}
    UPDATE_DELAYED_GCODE ID=bedfanheatloop DURATION=15
{% else %}
    # Low-temp path: PLA/PETG - no fans, fixed soak
    M190 S{target_bed}
    G4 P300000                            # 5 min soak
{% endif %}
```

## PRINT_END Setup

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
variable_debug: 0   # Silent — milestones only
variable_debug: 1   # Normal — key state changes (default)
variable_debug: 2   # Full — every loop tick with temps and fan %
```

### What Each Level Shows

**Level 0 — Milestones only**
These messages always print regardless of debug level:
```
[PRINT_START] Bed ready. Starting chamber heatsoak to 63C
Bed stabilizing before fans start (60s)
Chamber heatsoak complete
Chamber heatsoak timed out after 30 min - proceeding anyway
Bed fans: starting cooldown loop
Bed fans off: cooldown complete
```

**Level 1 — Normal status** (default)
Adds state-change messages when the fan logic reacts:
```
Bed fans off: bed temp low (108.2C / 115.0C)
Bed fans slow: bed recovering (113.1C / 115.0C)
Bed fans off: bed target below threshold
Chamber at target: 63.2C / 63.0C fans=40% (confirm 1/2)
Chamber at target: 63.2C / 63.0C fans=40% (confirm 2/2)
Chamber stable for 30s - proceeding
```

**Level 2 — Full debug**
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

### Common Issues

**Fans never start during heatsoak**
- Check `variable_fan` matches your `[fan_generic]` name exactly (case-sensitive)
- Check `variable_chamber_sensor` matches your `[temperature_sensor]` name
- Verify bed target is ≥ `variable_heating_threshold` (default 90°C)
- Enable `variable_debug: 2` and watch the console

**"Not heating at expected rate" error**
- Fans are throttling correctly but bed is still losing temp
- Reduce `variable_bed_throttle_slow` (e.g. `0.5`) to throttle sooner
- Reduce `variable_bed_throttle_off` (e.g. `2.0`) to kill fans sooner
- Check for drafts or enclosure leaks

**Heatsoak times out (30 min)**
- Chamber target may be too high for your enclosure
- Check `variable_chamber_target_default` — lower it if needed
- Verify chamber sensor is reading correctly
- Enable `variable_debug: 2` to watch chamber temp climb

**Cooldown never completes**
- Check `variable_cooling_threshold` — bed must drop below this (default 40°C)
- Long cooldowns are normal for ABS — can take 60–90 min
- Use the Voron Log Analyzer to visualize the full cooldown curve

**Fans stuck on after print**
- `TURN_OFF_HEATERS` starts the cooldown loop automatically
- Fans will shut off once bed reaches `variable_cooling_threshold`
- Run `TEST_FAN_COOLDOWN` to manually trigger and observe

### Testing Macros

```gcode
TEST_BED_FANS       # Starts the heatsoak fan loop without a print
                    # Uses chamber_target_default as the target
                    # Safe to run with bed heated

TEST_FAN_COOLDOWN   # Triggers TURN_OFF_HEATERS → starts cooldown loop
                    # Observe fan behavior and console output
```

### Log Analysis

Use the **Voron Log Analyzer** (`Voron Log Analyzer/voron_log_analyzer.html`) to visualize a full session:

1. Load `klippy.log` + `KlipperScreen.log`
2. The **Milestone Timeline** shows all echo messages with timestamps
3. The **Temperature chart** shows heatsoak (blue) and cooldown (orange) phases
4. Filter milestones by **CHAMBER** or **COOLDOWN** category to focus on fan events
5. Use **All Sessions** view to compare behavior across multiple prints

---

## Architecture

All complex logic lives in `_BED_FAN_MANAGE` — a regular gcode macro. The `[delayed_gcode]` blocks are intentionally minimal (one line each) to avoid Klipper silent-failure issues with complex Jinja in delayed_gcode bodies. Heatsoak fan control is driven from `CHAMBER_HEATSOAK`, called by `PRINT_START`.

```
PRINT_START
  └── CHAMBER_HEATSOAK          # heatsoak loop with fan management
        ├── _BED_FAN_MANAGE     # core fan logic (HEATSOAK mode)
        ├── _CHAMBER_READY      # checks temp, manages confirm count
        ├── _HEATSOAK_RESULT    # reports outcome
        └── _DRAIN_LOOP_WAIT    # adaptive timing

bedfanheatloop (delayed_gcode)
  └── _BED_FAN_MANAGE           # print-time maintenance (HEATSOAK mode)

bedfancoolloop (delayed_gcode)
  └── _BED_FAN_MANAGE           # cooldown management (COOLDOWN mode)
```

---

## Comparison to Ellis bedfans.cfg

| Feature | Ellis | SmartChamber |
|---------|-------|--------------|
| Proportional ramping | ❌ on/off only | ✅ 5% steps |
| Bed protection | ❌ | ✅ throttle/kill on heat loss |
| Chamber confirmation | ❌ | ✅ 30s stable required |
| Adaptive timing | ❌ | ✅ faster/slower based on state |
| Cooldown management | ❌ | ✅ delta-based |
| LED hooks | ❌ | ✅ |
| Tunable parameters | minimal | ✅ full control |

---

## Tested On

- Voron 2.4 350mm — Leviathan V1.1, SB2209 CANBUS, 4x bed fans (Nevermore)

---

## License

MIT
