# Changelog

## Unreleased

- Filter wear tracker: first version.
- Auto power-down: first version.
- Chamber heater control and chamber over-temperature safety: first version.
- Print notifications: first version.
- Filter fan dropout recovery: first version.
- Dropout recovery: renamed from "Filter fan dropout recovery" and reworded for any device
  (fan, heater, anything on a plug). Same file and input names, so existing automations keep
  working.
- Chamber heater control: the idle cutoff now defaults to 75 minutes, to leave room for a
  pre-print heat soak.
- Low filament warning: first version.
- Auto power-down: optional mode `input_select` (when print ends / after idle time / stay on)
  and print status input. "When print ends" powers off 10 minutes (configurable) after a print
  finishes or fails. Switching the mode to "stay on", or the kill switch off, during the warning
  now cancels it. Existing automations behave as before.
- Minimum Home Assistant version is 2025.4.
- README and PATTERNS.md.
