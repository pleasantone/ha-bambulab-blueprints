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
- Auto power-down: optional restart-safe timing (`idle_since` input_datetime and
  `end_time_sensor`). Restarts and automation reloads no longer reset the idle clock, a deadline
  that passed while HA was down is acted on at startup (with the warning first), and a stale or
  brand new timestamp restarts the clock instead of cutting power. Without the new inputs it
  behaves exactly as before.
- Chamber heater control: optional device auto-off keepalive. For a relay with its own auto-off
  timer, re-sends "on" every 5/10/15 minutes while the heater is on, so the device turns the
  heater off by itself if HA stops, this automation is turned off, or any automation listed
  under "Only while these are on" is turned off. Off by default; existing automations behave
  as before.
- Minimum Home Assistant version is 2025.4.
- README and PATTERNS.md.
