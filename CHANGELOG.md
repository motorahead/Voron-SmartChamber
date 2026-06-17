# Changelog

All notable changes to this project will be documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
