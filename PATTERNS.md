# How I structure my printer automations

The blueprints cover the parts that are easy to share. This is the stuff around them that isn't
blueprint-shaped. It's mostly lessons from running two printers and having things quietly break.

## One map of entity IDs

With two printers I kept copy-pasting entity IDs between automations, and they drifted. Now
there's one script that holds a map per printer and returns it:

```yaml
alias: Bambu - Printer Entities
fields:
  printer:
    required: true
    selector:
      select:
        options: [x1c, h2c]
sequence:
  - variables:
      printers:
        x1c:
          slug: x1c
          status: sensor.x1c_..._print_status
          plug: switch.x1c_plug
          power: sensor.x1c_plug_power
          # ...
        h2c:
          # ...
  - variables:
      entities: "{{ printers.get(printer | default('', true), {}) }}"
  - stop: Resolved
    response_variable: entities
```

Anything that needs an entity in its **actions** calls it with `response_variable` and reads
`p.plug`, `p.status` and so on. Triggers still use literal entity IDs, because that's what they
need. When I moved both printers to a new power strip, the plug swap was a one-line change in the
map instead of four edits scattered around.

The catch: the map is plain strings, so an entity rename breaks it silently. Search for the
script before you rename anything.

## One door for notifications

Everything goes through one notify script. It stamps the printer's name onto the title, picks
the tag, and fans out to phones, a browser push, the TV while it's on, and so on. Callers never
build titles or targets themselves. Adding a device or changing how notifications look is one
edit.

## Tags decide what replaces what

The phone replaces a notification when a new one arrives with the same tag. Getting the tags
wrong means you lose things. Mine used to share one tag per printer, so "powering down in 2
minutes" silently replaced the "print finished" summary 4 hours earlier. Now:

| Tag | Replaces |
|---|---|
| `bambu_<printer>_print` | finished ↔ failed ↔ canceled (a newer outcome replaces an older one) |
| `bambu_<printer>_hms_<code>` | only the same HMS code |
| `bambu_<printer>_power` | power-down warning ↔ cancelled ↔ skipped |
| `filter_<filter>` | filter alert ↔ daily reminder |

The rule of thumb: things that supersede each other share a tag. A cause never shares a tag with
its effect. An HMS error must not get wiped by the print failure it caused.

## Gate on developer mode to avoid duplicates

When a printer is cloud-connected, the Bambu app already notifies you about prints. HA
notifications on top of that are duplicates. So every notifier that has a cloud equivalent
(finished, failed, error, HMS) only runs while `developer_lan_mode` is **on**. Things the cloud
never sends (power-down, filter wear, over-temperature) aren't gated.

It's easy to get this backwards: gating on dev mode *off* notifies exactly when the app is
already doing it.

## Fail loud, never silently skip

A `numeric_state` condition on an unavailable sensor is just false, so an automation can sit
dead for weeks and look fine. My power-down did exactly that after a plug swap. Anything that
depends on a reading now checks it first and **notifies** if it's unreadable, rather than
quietly doing nothing, or worse, treating "unavailable" as "0 W, must be idle".

## Re-check after every delay

If an automation warns, waits and then acts, the world may have changed during the wait. The
power-down re-checks idle, power and AMS drying after its warning. Otherwise a print started
inside that 2-minute window gets its power cut in prepare. Same idea for the over-temperature safety re-applying
itself every minute: don't assume the state you set is the state you still have.

## One owner per entity

Two automations turning the same thing on and off will eventually fight. The Bento Box fan
belongs to its blueprint; the heater automation deliberately doesn't touch it, even though
"turn everything off when the print ends" would be the obvious thing to write.

## Snapshots that can't fill the disk

Failure snapshots are named `failure_<printer>_d<day>h<hour>_<token>.jpg`. A failure overwrites
the file from the same day-and-hour a month ago, so the folder can't grow without bound and
needs no cleanup job. The token matters because `/config/www` is served at `/local/` with **no
login**.

## Say why, in the description

Every automation's description says *why* it's built the way it is: why the threshold is 45 W
and not 20, why a sensor is deliberately left out, what not to "optimise". Future me reads those
before "fixing" something that isn't broken.
