# Bambu Lab blueprints for Home Assistant

A few blueprints I use to run my X1C and H2C from Home Assistant. They sit on top of the
[ha-bambulab](https://github.com/greghesp/ha-bambulab) integration.

I run every one of these myself. They're maintained on a "fix it when it breaks for me"
basis. Issues and PRs are welcome, but no promises on turnaround.

Requires Home Assistant 2025.4 or newer.

## Blueprints

### Filter wear tracker

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpleasantone%2Fha-bambulab-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fpleasantone%2Fbambu_filter_wear.yaml)

I never knew when to change my carbon filters until the room started to stink, which isn't
healthy. This keeps a running count of **weighted hours** for each filter, and nags you when
the filter is past the life you set.

- While air is moving through the filter, every 5 minutes adds 5 minutes times a material
  weight. By default ABS/ASA/PC/PA/PPA/PPS/HIPS count 3×, PETG/PCTG/PET/TPU/PVA 1×, PLA 0.5×,
  and anything else 1×.
- "Air moving" is either the printer being in prepare/running/pause, or, if you give it one,
  the filter's own fan. That's the right choice for a Bento Box that only runs for some
  materials.
- While idle, the carbon still soaks up moisture. With a humidity sensor, it adds
  `0.05 × RH` weighted hours per hour, which is about 0.7 h a day at 60% RH.
- It notifies once when the counter crosses the rated life, then once a day until you press
  the Replaced button.
- The counters are plain `input_number`s, so they don't create any long-term statistics.

**The weights are educated guesses.** I haven't calibrated them against real filter
replacements yet. If you do, open an issue with your numbers.

#### Setup

One instance per filter. A printer with two filters gets two instances, pointed at the same
printer sensors.

Create these helpers for each filter first (Settings → Devices & services → Helpers), because
blueprints can't create helpers:

| Helper | Type | Settings |
|---|---|---|
| Wear counter | Number | min 0, max 100000, step 0.001, box, unit `h`. **No initial value**, or it resets on every restart |
| Rated life | Number | min 10, max 5000, step 10, unit `h`. I started at 300 |
| Replaced | Button | press it after you swap the filter; its state is the replacement date |

The notification is an action you supply, so it works with whatever you already use. The
action gets `notify_title`, `notify_message`, `notify_tag`, `notify_event` (`worn` /
`remind`), `filter_name`, `wear` and `rated`. `notify_tag` is unique per filter, so the daily
reminder replaces yesterday's instead of piling up. For a phone:

```yaml
action: notify.mobile_app_your_phone
data:
  title: "{{ notify_title }}"
  message: "{{ notify_message }}"
  data:
    tag: "{{ notify_tag }}"
```

A dashboard card, if you have [entity-progress-card](https://github.com/francois-le-ko4la/lovelace-entity-progress-card):

```yaml
type: custom:entity-progress-card
entity: input_number.x1c_bento_filter_wear
max_value: input_number.x1c_bento_filter_rated_life
name: Bento Box carbon
icon: mdi:air-filter
```

### Auto power-down

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpleasantone%2Fha-bambulab-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fpleasantone%2Fbambu_auto_power_down.yaml)

Turns the printer's smart plug off after it's been idle for a while (4 hours by default). Lots
of these exist. This one is paranoid about killing the printer at the wrong time:

- It only counts as idle if the stage is idle **and** the plug reads idle-level power.
- If the power sensor is unavailable, it tells you and leaves the printer on, instead of
  quietly assuming "0 W = idle".
- It warns you, waits (2 min default), then **checks everything again** before cutting power.
  If you started a print during the warning, it cancels instead of killing it in prepare.
- Give it your AMS drying sensors and it won't cut power while any of them is drying, or
  reporting unknown. Include the AMS units that have their own power supply. They still need
  the printer on to dry.
- Optional `input_boolean` kill switch.

Measure your printer's idle draw on the plug and set the threshold comfortably above it. My
X1C idles around 11 W (I use 20 W); my H2C idles 20–36 W depending on the airduct (I use 45 W).

### Chamber heater control (third-party heaters)

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpleasantone%2Fha-bambulab-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fpleasantone%2Fbambu_chamber_heater.yaml)

For bolt-on heaters like a BambuSauna on a relay or smart plug. It turns the heater on when the
bed target goes above ~90 °C, and makes sure it's off whenever the printer isn't actually
printing. That includes the easy-to-miss cases: a cancel during prepare, a cancel while paused,
a manual bed preheat you forgot about.

A printer that drops off the network mid-print keeps printing, so offline/unavailable only turns
the heater off after an hour. It never counts as "not printing" for the other exits.

**Your heater still needs its own thermostat or thermal fuse.** This is control, not safety.

### Chamber over-temperature safety

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpleasantone%2Fha-bambulab-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fpleasantone%2Fbambu_chamber_overtemp.yaml)

A second layer for the heater above. If the chamber stays over the limit (65 °C default) it
turns the heater off, runs your fans (100% on the ones with speed control, plain on for
switches like a window exhaust) and sends one high-priority notification. It re-applies all of
that every minute while the chamber is still over, because the printer firmware and other
automations will happily turn your fans back down.

It can't see the chamber while the printer is offline, and it can't do anything while HA is
down. That's why it's a second layer and not the only one. Stock Bambu chamber heaters (H2D/H2C/H2S)
handle over-temperature themselves; you don't need this there.

### Print notifications

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fpleasantone%2Fha-bambulab-blueprints%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fpleasantone%2Fbambu_print_notifications.yaml)

Finished, failed, canceled, print error and HMS notifications for one printer. You pick the
device and it finds the task/time/weight sensors itself. What's different from the usual ones:

- **No duplicates with the Bambu app.** Point it at the `developer_lan_mode` sensor and it only
  notifies in LAN developer mode, which is exactly when the app can't.
- **One incident, one notification.** A failed print usually raises a print error too; the
  "failed" path waits 10 s and stands down if the error already covered it.
- **Sensible tags.** Outcomes share one tag per printer, so "finished" replaces yesterday's
  "failed". HMS errors get one per code, so a repeat replaces itself, but a print outcome never
  wipes the HMS error that caused it.
- **Optional failure snapshot.** Read the note in the blueprint first: snapshots under
  `/config/www` are served **without a login**, so it puts a random token in the file name. It
  also reuses day+hour slots, so the folder can't grow forever.

## Notifications, in general

Every blueprint here takes a **notification action** instead of a notify service, and hands it
`notify_title`, `notify_message` and `notify_tag` (plus a few extras, listed in each blueprint).
That way it works with the mobile app, a notify group, Pushover, a script of your own, whatever.
I route all of mine through one script, which is one of the patterns in
[PATTERNS.md](PATTERNS.md).

## License

MIT
