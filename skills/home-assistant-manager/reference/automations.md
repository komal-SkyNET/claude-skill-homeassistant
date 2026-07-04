# Automations reference

Read this when writing or debugging automations. Core deploy/verify rules are in `../SKILL.md`.

## Modern syntax (2024.10+) — write this, not the legacy form
Most published examples predate the rename; the legacy keys still work but don't emit them
in new code:

| Legacy | Modern |
|--------|--------|
| top-level `trigger:` / `condition:` / `action:` | `triggers:` / `conditions:` / `actions:` |
| `platform: state` (inside a trigger) | `trigger: state` |
| `service: light.turn_on` | `action: light.turn_on` |

```yaml
- id: front_door_left_open        # stable + unique; traces and the UI editor need it
  alias: "Front door left open"
  triggers:
    - trigger: state
      entity_id: binary_sensor.front_door
      to: "on"
      for: "00:05:00"
  conditions:
    - condition: state
      entity_id: person.komal
      state: home
  actions:
    - action: notify.mobile_app_phone
      data:
        message: "Front door has been open for 5 minutes"
  mode: single
```

Every automation gets a stable `id:` (never change it once set) and a human `alias`.

## Mode — what happens on re-trigger
| Mode | Behavior | Typical use |
|------|----------|-------------|
| `single` (default) | Ignore new trigger while running | Most automations |
| `restart` | Kill the run, start over | Motion light with `delay`/`wait` timeout — re-trigger resets the timer |
| `queued` | Run again after current finishes | Sequential actions that must not interleave |
| `parallel` | Run concurrently | Independent per-entity runs |

A motion light that turns off after a `delay` **must** use `mode: restart`, or fresh motion
won't extend the timer.

## Blueprints
- Live in `<config>/blueprints/automation/<namespace>/<file>.yaml` (subdirs allowed).
- Instantiate with `use_blueprint`:
```yaml
- id: kitchen_motion_light
  alias: "Kitchen motion light"
  use_blueprint:
    path: homeassistant/motion_light.yaml
    input:
      motion_entity: binary_sensor.kitchen_motion
      light_target:
        entity_id: light.kitchen
```
- After editing a blueprint file or its instances, **reload automations** to re-render them.

## Debugging
- **Traces** are the primary tool: every run records an interactive step-by-step graph —
  Settings → Automations → open the automation → Traces. YAML automations **must have an
  `id:`** or traces/UI editing won't work.
- `automation.trigger` **bypasses `conditions` by default** (`skip_condition` defaults to
  true). A manual trigger proving the actions run does *not* prove the conditions gate
  correctly — pass `skip_condition: false`, or exercise the real trigger, then read the trace.
- Trigger variables: `{{ trigger.entity_id }}`, `{{ trigger.to_state.state }}` are only
  defined for the trigger type that fired — a manual `automation.trigger` leaves them unset,
  so templates using them can error only in manual tests. Check the trace, not just logs.
- Logbook (UI) shows which trigger fired each run; `ha core logs` shows template/action errors
  (see SKILL.md verify loop).

## Pitfalls
| Symptom | Cause → Fix |
|---------|-------------|
| Automation missing from UI / no traces | No `id:` → add a stable one, reload |
| Motion light turns off mid-occupancy | `mode: single` swallows re-triggers → `mode: restart` |
| `for:` never fires | State flaps (e.g. sensor blips) resetting the timer → template trigger or debounced helper |
| Notification fires when away-check should block it | Manual trigger skipped conditions → re-test with `skip_condition: false` |
| `TypeError` comparing states | Bare states are strings → `| int(0)` / `| float(0)` (see SKILL.md Templates) |
