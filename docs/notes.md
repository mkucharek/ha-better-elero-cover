# Design notes & Home Assistant gotchas

Why this blueprint exists and the constraints that shaped it. Background for
anyone iterating on the blueprint or the [template-cover wrapper](../examples/template-covers.yaml).

## The hub has no usable tilt

Elero motors driven through a Fibaro / Yubii hub present as a plain `cover`:

- No native tilt **position** and no position **feedback** at all.
- The hub's tilt commands are broken — `cover.open_cover_tilt` /
  `cover.close_cover_tilt` (and the dashboard tilt buttons) just do a full
  open / close, identical to the non-tilt commands.

So tilt has to be faked: drive the slats to a known end stop, then run the motor
for a measured time. That's what the blueprint does (close → wait → timed open →
stop). It's open-loop — accuracy depends on consistent motor speed and a
`close_duration` that reliably reaches the end stop.

## Why wrap the cover in a template cover

You **cannot intercept a service call on the real entity** — there's no way to
say "when someone sets tilt on `cover.foo`, run my script instead." The fix is
to wrap the real cover in a **Template Cover** that delegates open/close/stop to
the real entity and routes `set_cover_tilt_position` to the blueprint script.
Put the wrapper on dashboards; leave the raw cover alone.

## `set_cover_tilt_position` behavior (the key finding)

Defining `set_cover_tilt_position` on the template cover advertises the
`SET_TILT_POSITION` feature. As a result, Home Assistant renders **both**:

- a tilt **slider** (0–100), and
- the up/down tilt **buttons**.

All of them route to the same script. The slider passes its 0–100 value through
as `tilt_pct: "{{ tilt }}"`; the tilt buttons send `100` (open-tilt) and `0`
(close-tilt).

**There is no YAML-only way to get the buttons without the slider** (or vice
versa). Suppressing one would require a custom integration — out of scope here.
We accept the slider; with the percentage blueprint it's actually the useful
control.

## Other template-cover decisions

- **`tilt_optimistic: true`** — there's no real tilt feedback, so the slider
  optimistically reflects the last requested angle instead of snapping back.
- **`state:` maps `unknown` → `open`.** A partially-tilted blind reports
  `unknown` (the hub gives no position); showing it as `open` reads better than
  "Unknown". `unavailable` passes through untouched.
- **"Partially open" can't be shown as a cover state** — cover states are a
  fixed enum (`open`/`closed`/`opening`/`closing`/`unavailable`). Representing a
  partial-tilt state would need a separate template sensor.
- **Use the modern `template:` integration**, not the legacy
  `cover: platform: template`. Reload via Dev Tools → YAML → **Template
  entities** — no restart needed.

## Source

These conclusions come from a working session (2026-05-29/30) debugging the live
setup; the wrapper in `examples/template-covers.yaml` is the genericized result.
