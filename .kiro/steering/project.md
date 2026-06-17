# Voron-SmartChamber Project

## Overview
Autonomous chamber temperature management system for Klipper-based 3D printers. Single drop-in cfg file providing closed-loop chamber control with proportional fan ramping, bed protection, heatsoak confirmation, and automatic cooldown.

## Technology
- **Language**: Klipper G-code macros with Jinja2 templating
- **Config format**: INI-style `.cfg` files
- **Runtime**: Klipper firmware on Raspberry Pi 4
- **Hardware**: Voron 2.4 350mm, Leviathan V1.1, SB2209 CANBUS, 4x bed fans

## File Structure
```
Voron-SmartChamber/
├── .kiro/steering/                        # AI guidance documents
├── README.md                              # Documentation and setup guide
├── smart_chamber.cfg                      # Main chamber control (blocking heatsoak)
├── WIP_DONOTUSE_smart_chamber_interruptible.cfg  # Interruptible variant (WIP)
├── WIP_DONOTUSE_README_interruptible.md   # Interruptible variant docs (WIP)
├── Temp Profile.png                       # Example temperature chart
├── LICENSE                                # GPL v3
└── .gitignore
```

## Architecture
All fan logic lives in `_BED_FAN_MANAGE` (single source of truth), called by minimal `[delayed_gcode]` blocks. Modes:
- `FAN_HEATSOAK` — proportional ramp toward chamber target, bed protection active
- `FAN_COOLDOWN` — delta-based fan speed, full --> slow --> off

Key macros:
- `CHAMBER_HEATSOAK` — public entry point, blocking loop
- `_BED_FAN_MANAGE` — core logic (fan speed, bed protection, state management)
- `_CHAMBER_STABILITY_CHECK` — rate-of-change detection for heatsoak completion
- `_HEATSOAK_TIMEOUT_HANDLER` — timeout logic (cancel vs proceed)
- `_DRAIN_LOOP_WAIT` — adaptive loop timing
- `TEST_BED_FANS` / `TEST_FAN_COOLDOWN` — functional testing macros

## chamber_state Values
- `0` — not set (PLA/reset state)
- `> 0` — active heatsoak target being pursued
- `-1` — heatsoak confirmed stable (internal signal)
- Use `> 0` to check "actively heating", not `!= -1`
- `chamber_target_default` is the fallback temp when slicer sends 0

## Code Quality Rules
- Never write empty `{% if %}` blocks with all logic in `{% else %}` — invert the condition
- All `{% if %}` / `{% elif %}` blocks must be explicit — no silent fallthrough via `{% else %}`
- `_BED_FAN_MANAGE` modes must be explicitly named and checked — no default fallback
- Remove dead code (no-op `G4 P1` commands not needed in modern Klipper)
- Internal macros use `_` prefix; public macros have no prefix
- Code should be self-documenting — rewrite confusing logic before adding comments

## Naming Conventions
- Macro names describe purpose: `_CHAMBER_STABILITY_CHECK` not `_CHAMBER_READY`
- Internal macros: `_VERB_NOUN` or `_NOUN_VERB` pattern, underscore prefix
- Public macros: `VERB_NOUN` or `NOUN_VERB`, no underscore
- Mode values scoped to their macro: `FAN_HEATSOAK` not `HEATSOAK`
- State variables named for role: `chamber_state` not `chamber_target`
- Config variables use descriptive names: `chamber_target_default`, `rate_threshold`

## File Parity
- Both `smart_chamber.cfg` and the interruptible variant must stay in sync for shared logic
- Changes to `_BEDFANVARS`, `_BED_FAN_MANAGE`, cooldown, heater overrides, or config variables apply to BOTH
- Changes specific to heatsoak architecture (blocking vs PAUSE/monitor) apply to relevant file only
- Each cfg has a corresponding README — update both when shared behavior changes
- `TEST_HEATSOAK` is interruptible-only

## File Encoding
- Must be plain ASCII except intentional emoji
- After any edit, scan for encoding issues before pushing
- Em dashes, multiplication signs, broken bars are common culprits from copy-paste

## After Every Code Change
- Update inline comments in the changed file that reference modified logic
- Update README.md if change affects documented behavior, config variables, architecture, or debug output
- Scan for encoding issues before pushing
- Check for discrepancies between README and actual cfg behavior
- When renaming, update: cfg, README, steering docs, and any call sites

## Testing
```gcode
TEST_BED_FANS       # Starts heatsoak fan loop without a print
TEST_FAN_COOLDOWN   # Triggers TURN_OFF_HEATERS --> starts cooldown loop
```

## Git
- Remote repo: Voron-SmartChamber
- Commit messages describe what changed and why
