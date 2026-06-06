# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Home Assistant **script** blueprint (`elero_cover_tilt.yaml`) that tilts a
venetian / horizontal blind by briefly reversing the motor. Built for Elero
covers that expose only open/close/stop (no native tilt position). Distributed
as a public GitHub repository.

## Repository Structure

- `elero_cover_tilt.yaml` — the blueprint (the deliverable users import)
- `README.md` — user-facing docs and the Home Assistant import button
- `CLAUDE.md` — this file

## How the Blueprint Works

One script instance per cover. The user invokes it manually (button card,
dashboard, automation, etc.).

**Flow:** close_cover -> delay(close_duration) -> if tilt_pct>0 { open_cover ->
delay(open_duration * tilt_pct/100) } -> stop_cover.

Open-loop percentage tilt: drive closed for a fixed time to reach the 0%
baseline, then open for a time proportional to the requested tilt. No position
feedback (the Yubii/Elero hub reports none), so accuracy relies on consistent
motor speed and a `close_duration` that reliably hits the end stop.

`tilt_pct` is a runtime **field** (not a blueprint input) — passed per call,
e.g. from a template cover's `set_cover_tilt_position` via `data: {tilt_pct:
"{{ tilt }}"}`. Defaults to 100 when called bare.

**Keep `mode: single`.** Dragging a tilt slider fires multiple calls;
`single` + `max_exceeded: silent` lets the first run finish (including its
`stop_cover`) and drops the rest. `restart` would be dangerous — cancelling
between `open_cover` and `stop_cover` leaves the motor running.

**Mechanism:** close drives slats to the fully-closed end stop (known position),
then a timed open sets the tilt angle. The two delays are the only tuning knobs:
- `close_duration` must be long enough to reach the end stop from any position.
- `open_duration` sets the final tilt angle (longer = more open).

**Domain is `script`, not `automation`** — unlike automation blueprints there is
no trigger/condition; the top-level key is `sequence` and inputs become script
fields.

**Mode:** `single` with `max_exceeded: silent` — re-invokes while running are
dropped, not queued, so a mid-sequence press can't fight the motor.

**Delays** are templated from float seconds to integer milliseconds inside the
`delay` dict, so fractional-second tuning works.

## Versioning

Version is in the blueprint `name` field. Update it on every change. Use semver.

## Publishing

Repo: `mkucharek/ha-better-elero-cover` (public). `origin` points there; the
original gist is kept as the `gist` remote.

Push to `main` publishes. Users import via the raw blueprint URL
(`https://github.com/mkucharek/ha-better-elero-cover/blob/main/elero_cover_tilt.yaml`),
wired into the My Home Assistant import button in `README.md`. Keep that button
and any version references in `README.md` in sync with the blueprint.
