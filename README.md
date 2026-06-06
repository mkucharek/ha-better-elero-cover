# Elero Cover Tilt

A Home Assistant **script blueprint** that tilts a venetian / horizontal blind
to a percentage by timing the motor — for Elero covers that expose only
open / close / stop and have no native tilt support.

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fmkucharek%2Fha-better-elero-cover%2Fblob%2Fmain%2Felero_cover_tilt.yaml)

## Why

Elero motors driven through hubs like Fibaro / Yubii present as a plain `cover`
with open / close / stop and **no tilt position**. This blueprint fakes tilt by
driving the slats to a known end stop and then opening for a measured amount of
time.

## How it works

```
close_cover → wait close_duration → (if tilt > 0: open_cover → wait open_duration × tilt%) → stop_cover
```

1. **Close** drives the slats to the fully-closed end stop — a known **0%
   baseline**.
2. **Open** for a time proportional to the requested tilt, then **stop**.

It is **open-loop**: there is no position feedback, so accuracy depends on a
consistent motor speed and a `close_duration` long enough to reliably reach the
end stop from any starting position.

## Install

1. Click the **import button** above (or copy the URL below into
   *Settings → Automations & Scenes → Blueprints → Import Blueprint*):

   ```
   https://github.com/mkucharek/ha-better-elero-cover/blob/main/elero_cover_tilt.yaml
   ```
2. Create a **script** from the blueprint, one per cover.

## Inputs

| Input | Default | What it does |
| --- | --- | --- |
| **Cover** | — | The venetian blind to tilt. |
| **Close Duration (s)** | `2` | Time to drive fully closed before opening — the 0% baseline. Must reach the closed end stop from any position. |
| **Open Duration at 100% (s)** | `5` | Open time that corresponds to 100% tilt. Actual open time = `open_duration × tilt% / 100`. |

### Runtime field

`tilt_pct` (0–100) is passed **per call**, not configured on the blueprint.
Defaults to `100` when the script is called bare. At `0` the blind just closes.

## Usage

### Call directly

```yaml
action: script.elero_blind_tilt   # your script's entity_id
data:
  tilt_pct: 40
```

### As a tilt slider (template cover)

Wrap the script in a [template cover](https://www.home-assistant.io/integrations/cover.template/)
so the blind exposes a real tilt control in the UI and to automations:

```yaml
cover:
  - platform: template
    covers:
      living_room_blind:
        friendly_name: "Living Room Blind"
        value_template: "{{ states('cover.living_room') }}"
        open_cover:
          - action: cover.open_cover
            target: { entity_id: cover.living_room }
        close_cover:
          - action: cover.close_cover
            target: { entity_id: cover.living_room }
        stop_cover:
          - action: cover.stop_cover
            target: { entity_id: cover.living_room }
        set_cover_tilt_position:
          - action: script.elero_blind_tilt   # your script's entity_id
            data:
              tilt_pct: "{{ tilt }}"
```

## Tuning

The two durations are the only knobs:

- **`close_duration`** — increase until the blind reliably reaches the closed
  end stop from any position. Too short and the 0% baseline drifts.
- **`open_duration`** — sets the angle at 100%. Longer = more open.

## Notes & caveats

- **Open-loop, no feedback.** Tilt accuracy relies on consistent motor speed and
  a `close_duration` that always hits the end stop.
- **`mode: single`** (with `max_exceeded: silent`). Dragging a slider fires many
  calls; the first run finishes — including its `stop_cover` — and the rest are
  dropped, so presses can't fight the motor mid-sequence.
- Every close-then-open cycle moves the slats fully closed first, so tilt changes
  are not instantaneous.

## License

MIT
