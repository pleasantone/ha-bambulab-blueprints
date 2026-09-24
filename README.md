# Bambu Lab blueprints for Home Assistant

A few blueprints I use to run my X1C and H2C from Home Assistant. They sit on top of the
[ha-bambulab](https://github.com/greghesp/ha-bambulab) integration.

I run every one of these myself. They're maintained on a "fix it when it breaks for me"
basis. Issues and PRs are welcome, but no promises on turnaround.

Requires Home Assistant 2025.1 or newer.

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

## License

MIT
