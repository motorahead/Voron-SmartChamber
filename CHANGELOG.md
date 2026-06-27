# Changelog

All notable changes to this project will be documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.2] - 2026-06-27
### Fixed
- Cooldown loop no longer starts when bed is already below cooling_threshold (prevents idle timeout spam every 30 min on a cold printer)

### Changed
- Add inline comments to LED effect conditionals explaining the "off" disable pattern

## [1.0.1] - 2026-06-24
### Changed
- Document CHAMBER=0 behavior: heatsoak skipped, fans still manage toward chamber_target_default
- Add "When Do Fans Run?" section with truth table clarifying all three fan management paths
- Update chamber_target_default inline comment to describe actual behavior
- Clarify PLA Safety feature references heating_threshold gate
- Update PRINT_START example comments to distinguish heatsoak vs fan management

## [1.0.0] - 2026-06-16
### Added
- Initial public release
- Proportional fan ramping with deadband control
- Bed protection (throttle/cut fans if bed loses temp)
- Rate-of-change stability confirmation before print starts
- Heatsoak timeout with configurable minimum chamber temp
- Print-time fan maintenance loop
- Automatic cooldown on heaters-off event
- Delta-based cooldown fan speed management
- PLA safety (fans disabled for low-temp materials)
- Slicer fallback (chamber_target_default when slicer sends 0)
- LED status hooks (optional)
- Debug levels 0/1/2 for console output
- TEST_BED_FANS and TEST_FAN_COOLDOWN macros
- Universal compatibility (any enclosed printer with fan_generic output)
