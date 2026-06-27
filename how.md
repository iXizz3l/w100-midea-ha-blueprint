# Midea PortaSplit ↔ Aqara W100 Sync — Blueprint Reference

This document explains the blueprint section by section: what each block does, why it's structured the way it is, and the non-obvious decisions baked into it.

---

## `blueprint:` metadata

```yaml
blueprint:
  name: Midea PortaSplit ↔ Aqara W100 Sync
  description: >
    ...
    Prerequisites — The W100 thermostat mode switch must be ON ...
  domain: automation
```

The description front-loads the one manual prerequisite that cannot be automated by this blueprint: enabling `switch.<w100_suffix>_thermostat_mode` once. The W100 only acts as a climate controller when that switch is ON. Because it's a one-time setup step — not something that needs to be toggled per automation run — the blueprint documents it and moves on. Managing it inside the automation would add complexity for no ongoing benefit.

---

## `input:` — User-configurable parameters

### Entity selectors

```yaml
    midea_ac:
      selector:
        entity:
          domain: climate

    aqara_w100:
      selector:
        entity:
          domain: climate
```

Both are constrained to `domain: climate` to prevent accidental misconfiguration. The W100 exposes a climate entity (not just a switch or sensor), so this filter is correct for both devices.

### `w100_heat_target`

```yaml
    w100_heat_target:
      default: heat
      selector:
        select:
          options:
            - label: Heat
              value: heat
            - label: Dry
              value: dry
            - label: Fan Only
              value: fan_only
```

The W100 only has four HVAC modes: `off`, `cool`, `heat`, `auto`. Midea supports additional modes (`dry`, `fan_only`) that have no W100 equivalent. This input lets you nominate one Midea-only mode to be represented as `heat` on the W100 display — useful if you primarily use `dry` or `fan_only` for heating-season comfort without actual heat. The default is straightforward `heat` (Midea → W100 heat).

The reverse is equally important: when syncing Midea → W100, a Midea mode of `dry` or `fan_only` normally maps to W100 `off` (no equivalent), but if that mode equals `w100_heat_target`, the W100 stays on `heat` instead of being turned off. This prevents a feedback loop where the W100 shows `heat`, you switch Midea to `dry` (the designated heat-target), and the sync immediately turns the W100 off.

### Fan speed inputs (`fan_low_pct`, `fan_medium_pct`, `fan_high_pct`)

```yaml
    fan_low_pct:
      default: 1
      ...
    fan_medium_pct:
      default: 50
      ...
    fan_high_pct:
      default: 100
```

The W100 has named fan modes (`low`, `medium`, `high`, `auto`). The Midea exposes fan control as a `number` entity (1–100%) because once you set the Midea fan via percentage, it enters `fan_mode: custom` and named modes become unreliable. These three inputs are the bridge: they define what percentage corresponds to each W100 named level, in both directions. They're configurable because the meaningful breakpoints may differ per installation (e.g., some units are noisy at 50%, others aren't).

The minimum is 1 (not 0) because 0% would attempt to turn the fan off via the number entity, which is not how the Midea is turned off.

### `debounce_delay`

```yaml
    debounce_delay:
      default: 3
      selector:
        number:
          min: 3
          max: 15
```

Added to every trigger's `for:` field. A state change is only acted on if the device has held that state for this many seconds. Without debounce, the two devices would ping-pong: W100 changes → sync Midea → Midea state change fires → sync W100 → W100 state change fires → ... The 3-second minimum is intentional; below that, IR/Zigbee round-trip times make false feedback loops likely.

Critically, `debounce_delay` is **not** in the `variables:` block (see that section below). It's consumed only via `!input` directly in trigger `for:` fields, which is evaluated before `variables:` is in scope. This is not an oversight — it's by design.

### `sync_delay`

```yaml
    sync_delay:
      default: 2
      selector:
        number:
          min: 1
          max: 5
```

A pause inserted between sequential service calls within a single sync run (mode change → temperature change → fan change). Both devices need time to process a command and report their new state before the next command is sent. Without it, rapid successive commands can be dropped or applied out of order. `sync_delay` does appear in `variables:` because it's used in Jinja2 templates inside `actions:`.

### External sensor inputs

```yaml
    enable_external_data:
      default: false
      selector:
        boolean:

    external_temperature_sensor:
      default: []
      selector:
        entity:
          domain: sensor
          device_class: temperature

    external_humidity_sensor:
      default: []
      selector:
        entity:
          domain: sensor
          device_class: humidity
```

The W100 can display an external temperature and humidity reading (from a separate sensor) via its `number.<suffix>_external_temperature` and `number.<suffix>_external_humidity` entities. This is purely cosmetic on the W100 display — it doesn't affect thermostat operation.

`enable_external_data` acts as a master gate. The sensor inputs default to `[]` (empty list) rather than a string because an empty entity selector returns an empty list, not an empty string. Using `[]` as the default lets you safely pass the input to trigger `entity_id` without errors when no sensor is selected — HA tolerates an empty list as "no entity" for triggers.

The triggers for these sensors use `enabled: !input enable_external_data`, so when the feature is off, the triggers don't even register, saving unnecessary evaluations.

---

## `trigger_variables:`

```yaml
trigger_variables:
  _midea_ac: !input midea_ac
```

`trigger_variables` is a special HA block that is evaluated *before* triggers are set up, at blueprint instantiation time. It has limited template support compared to `variables:`. It exists here for one sole purpose: to construct the fan speed number entity ID for the third trigger.

The `variables:` block (which has full Jinja2 support) is not available at trigger evaluation time, so you cannot use `{{ midea_fan_entity }}` inside a `trigger:` entity_id. The workaround is to pull the raw entity ID into `trigger_variables` first, then use it in the trigger's template:

```yaml
  - trigger: state
    entity_id: "{{ 'number.' ~ _midea_ac.split('.')[1] ~ '_fan_speed' }}"
```

This is the only reason `trigger_variables` exists in this blueprint.

---

## `variables:`

```yaml
variables:
  midea_ac: !input midea_ac
  aqara_w100: !input aqara_w100
  w100_heat_target: !input w100_heat_target
  fan_low_pct: !input fan_low_pct
  fan_medium_pct: !input fan_medium_pct
  fan_high_pct: !input fan_high_pct
  sync_delay: !input sync_delay
  enable_external_data: !input enable_external_data
  external_temperature_sensor: !input external_temperature_sensor
  external_humidity_sensor: !input external_humidity_sensor
  midea_fan_entity: "{{ 'number.' ~ midea_ac.split('.')[1] ~ '_fan_speed' }}"
  w100_ext_temp_entity: "{{ 'number.' ~ aqara_w100.split('.')[1] ~ '_external_temperature' }}"
  w100_ext_hum_entity: "{{ 'number.' ~ aqara_w100.split('.')[1] ~ '_external_humidity' }}"
```

Every `!input` that appears inside a Jinja2 template in `actions:` must be declared here. `!input` cannot be used inside `{{ }}` template expressions — it's a YAML-level directive, not a template function. The pattern is: pull `!input` into a named variable, then use the variable name in templates.

The three derived entity IDs (`midea_fan_entity`, `w100_ext_temp_entity`, `w100_ext_hum_entity`) are built by taking the suffix of the climate entity ID (everything after the `domain.`) and prepending the relevant entity type. For example, `climate.living_room_ac` → `number.living_room_ac_fan_speed`. This is a naming convention assumption based on how the Midea and Aqara integrations generate their entity IDs.

`debounce_delay` is the one input that is **absent** from `variables:` — deliberately. It's consumed only in trigger `for:` fields via `!input debounce_delay` directly. Adding it to `variables:` would be harmless but misleading; it's documented here as intentional.

---

## `triggers:`

```yaml
triggers:
  - trigger: state
    entity_id: !input aqara_w100
    for:
      seconds: !input debounce_delay
    id: w100_state

  - trigger: state
    entity_id: !input midea_ac
    for:
      seconds: !input debounce_delay
    id: midea_state

  - trigger: state
    entity_id: "{{ 'number.' ~ _midea_ac.split('.')[1] ~ '_fan_speed' }}"
    for:
      seconds: !input debounce_delay
    id: midea_fan_speed

  - trigger: state
    entity_id: !input external_temperature_sensor
    id: ext_temp
    enabled: !input enable_external_data

  - trigger: state
    entity_id: !input external_humidity_sensor
    id: ext_hum
    enabled: !input enable_external_data
```

Five triggers, each with a named `id` so the `actions:` block can route cleanly with `condition: trigger id:`.

**Why separate `midea_state` and `midea_fan_speed`?**

The Midea climate entity does not change state when only the fan speed percentage changes. A fan % change only updates `number.<suffix>_fan_speed` — the climate entity itself stays in the same HVAC mode with the same temperature. Without a separate trigger on the number entity, a fan speed knob-turn from the Midea side would never propagate to the W100. Conversely, an HVAC mode or temperature change on the Midea does trigger `midea_state`, which also reads the current fan speed and syncs it — so both paths are covered.

**Why no `for:` debounce on external sensor triggers?**

Sensor readings don't create feedback loops (the W100 display is read-only from HA's perspective), so there's no risk of ping-pong. The sensors also update infrequently enough that immediate forwarding is fine. Adding debounce would delay display updates unnecessarily.

**The `enabled:` field on ext_temp and ext_hum**

When `enable_external_data` is `false`, these triggers are disabled at the trigger level — they don't even register. This is more efficient than letting them fire and failing a condition check inside the action. It also means the `entity_id` of an unselected optional sensor (which defaults to `[]`) never gets evaluated as a trigger.

---

## `conditions: []`

No global conditions. All branching is handled inside `actions:` via `choose:`. This is the right call when you have multiple mutually-exclusive trigger paths — a global condition would block all of them if it failed, which is too coarse.

---

## `actions:` — Top-level `choose:`

The entire action is a single `choose:` block with five branches, one per trigger ID (plus one sub-branch for fan speed). Each branch uses `condition: trigger id:` to match the trigger that fired, then runs its own sequence. There is no `default:` — if no branch matches (shouldn't happen given the trigger design), nothing runs.

---

## Branch 1: W100 → Midea

```yaml
      - conditions:
          - condition: trigger
            id: w100_state
        sequence:
```

Fires when the W100 climate entity changes and holds for `debounce_delay` seconds. Syncs three attributes to the Midea in order: HVAC mode, temperature, fan speed.

### Reset Midea preset

```yaml
          - action: climate.set_preset_mode
            target:
              entity_id: !input midea_ac
            data:
              preset_mode: none
          - delay:
              seconds: "{{ sync_delay }}"
```

This runs unconditionally, before any other sync step. Midea preset modes (`eco`, `ieco`, `sleep`, `boost`) silently override any other command — if a preset is active, setting temperature or fan speed has no effect. Clearing the preset first ensures subsequent commands actually take effect. The `sync_delay` after it gives the Midea time to apply the change and report its new state before the next command.

No "is a preset active?" guard is added — the overhead of always sending `preset_mode: none` is negligible, and a conditional check adds complexity without meaningful gain.

### Sync HVAC mode

```yaml
          - variables:
              _w100_mode: "{{ states(aqara_w100) }}"
              _target_mode: "{{ w100_heat_target if _w100_mode == 'heat' else _w100_mode }}"
          - if:
              - "{{ states(midea_ac) != _target_mode }}"
            then:
              - action: climate.set_hvac_mode
                ...
              - delay:
                  seconds: "{{ sync_delay }}"
```

The W100's `heat` mode maps to `w100_heat_target` on the Midea (which may be `heat`, `dry`, or `fan_only` per user config). All other W100 modes (`off`, `cool`, `auto`) pass through unchanged — those modes exist identically on the Midea.

The `if` guard (`states(midea_ac) != _target_mode`) prevents writing the same mode back to the Midea if it's already correct. This matters because a write would cause the Midea entity state to change, firing `midea_state`, which would then attempt a Midea → W100 sync. The guard breaks the potential feedback loop at the source. The `delay` after a successful write is only inserted when a write actually happened — no unnecessary waiting when skipped.

### Sync temperature

```yaml
          - variables:
              _target_temp: >-
                {{ [state_attr(aqara_w100, 'temperature') | float | round(0, 'ceil') | int, 16] | max }}
          - if:
              - "{{ state_attr(midea_ac, 'temperature') | int != _target_temp }}"
            then:
              - action: climate.set_temperature
                ...
```

Two conversions happen here:

1. **Ceiling rounding**: The W100 supports 0.5°C steps; the Midea only supports 1°C steps. A W100 setpoint of 20.5°C must become 21°C on the Midea (round up, not nearest-even or truncate). `round(0, 'ceil')` does this. The `| int` after it strips any decimal artifact from the float.

2. **Minimum clamp**: Midea won't accept temperatures below 16°C. `[..., 16] | max` clamps anything below 16 up to 16. This handles the case where the W100 is set below Midea's minimum range.

The guard compares `state_attr(midea_ac, 'temperature') | int` (coerced to int) against the already-integer `_target_temp`. Using `| int` on the Midea attribute avoids float comparison artifacts.

### Sync fan speed

```yaml
          - variables:
              _w100_fan: "{{ state_attr(aqara_w100, 'fan_mode') }}"
          - if:
              - "{{ _w100_fan == 'auto' }}"
            then:
              - if:
                  - "{{ state_attr(midea_ac, 'fan_mode') != 'auto' }}"
                then:
                  - action: climate.set_fan_mode
                    ...
                    data:
                      fan_mode: auto
            else:
              - variables:
                  _target_pct: >-
                    {{ fan_low_pct if _w100_fan == 'low'
                       else (fan_medium_pct if _w100_fan == 'medium'
                             else fan_high_pct) }}
              - if:
                  - "{{ states(midea_fan_entity) | int != _target_pct | int }}"
                then:
                  - action: number.set_value
                    target:
                      entity_id: "{{ midea_fan_entity }}"
                    data:
                      value: "{{ _target_pct | int }}"
```

Fan sync is bifurcated on `auto`:

- **`auto`**: Mapped directly to Midea's named `auto` fan mode via `climate.set_fan_mode`. The Midea's `auto` mode has the AC internally manage fan speed — this is semantically correct and doesn't use the percentage entity.

- **`low` / `medium` / `high`**: Translated to the user-configured percentage and written to `number.<suffix>_fan_speed`. Using the number entity (rather than named fan modes) is necessary because named Midea fan modes (`silent`, `low`, etc.) are unreliable once the number entity has been used — the Midea enters `fan_mode: custom` and named modes may not restore cleanly.

The outer guard (`_w100_fan != 'auto'` path) checks the current number entity value to avoid unnecessary writes. The inner guard on the `auto` path checks `fan_mode != 'auto'` for the same reason.

The fallback when `_w100_fan` is neither `low` nor `medium` is `fan_high_pct`. This means any unexpected W100 fan mode value defaults to high rather than silently doing nothing or erroring.

---

## Branch 2: Midea → W100 (climate state change)

```yaml
      - conditions:
          - condition: trigger
            id: midea_state
```

Fires when the Midea climate entity changes and holds for `debounce_delay` seconds. Syncs HVAC mode, temperature, and fan to the W100.

### Sync HVAC mode

```yaml
          - variables:
              _midea_mode: "{{ states(midea_ac) }}"
              _target_w100_mode: >-
                {{ _midea_mode if _midea_mode in ['off', 'cool', 'auto', 'heat']
                   else ('heat' if _midea_mode == w100_heat_target else 'off') }}
```

The W100 only supports `off`, `cool`, `heat`, `auto`. Midea-only modes (`dry`, `fan_only`) need mapping:

- If the Midea mode is one of the four W100-native modes → pass through directly.
- Otherwise (`dry` or `fan_only`): check if this mode equals `w100_heat_target`. If yes → W100 shows `heat` (because this is the mode the user designated as the heat analog). If no → W100 shows `off`.

This "heat if matches target, else off" logic is the reverse of the W100→Midea mapping and prevents a feedback loop: if W100 heat → Midea dry (via `w100_heat_target`), then Midea dry → W100 heat (not off), so the W100 doesn't get turned off by its own action.

### Sync temperature

```yaml
          - if:
              - "{{ state_attr(midea_ac, 'temperature') != state_attr(aqara_w100, 'temperature') }}"
            then:
              - action: climate.set_temperature
                target:
                  entity_id: !input aqara_w100
                data:
                  temperature: "{{ state_attr(midea_ac, 'temperature') }}"
```

No conversion needed in this direction. Midea uses 1°C steps; the W100 accepts 0.5°C steps. Any valid 1°C integer is trivially valid on a 0.5°C grid. The attribute value is passed through verbatim.

The comparison uses attribute values directly (not coerced to int) because both entities might report 20.0 vs 20.0 as floats, and coercion isn't needed when the types match.

### Sync fan speed

```yaml
          - variables:
              _midea_fan: "{{ state_attr(midea_ac, 'fan_mode') }}"
              _midea_pct: "{{ states(midea_fan_entity) | float(0) }}"
              _low_d: "{{ (_midea_pct - fan_low_pct) | abs }}"
              _med_d: "{{ (_midea_pct - fan_medium_pct) | abs }}"
              _high_d: "{{ (_midea_pct - fan_high_pct) | abs }}"
              _target_w100_fan: >-
                {{ 'auto' if _midea_fan == 'auto'
                   else ('high' if _high_d <= _low_d and _high_d <= _med_d
                         else ('medium' if _med_d <= _low_d else 'low')) }}
```

This is the inverse fan mapping: Midea percentage → W100 named mode.

The approach is "nearest neighbor by absolute distance":
- Compute the absolute distance from the current percentage to each of the three configured breakpoints.
- Pick the W100 mode whose breakpoint is closest.

**Tie-breaking**: `high` is checked with `<=` (not `<`). If the current percentage is exactly equidistant between two modes, the higher mode wins. This is intentional — erring toward more airflow is a conservative default for HVAC comfort.

`float(0)` is the default fallback in case the number entity is temporarily unavailable. This would map to "closest to 0%", which would select `low` — a safe conservative fallback (doesn't blast air unexpectedly).

---

## Branch 3: Midea → W100 (fan speed % change)

```yaml
      - conditions:
          - condition: trigger
            id: midea_fan_speed
          - "{{ state_attr(midea_ac, 'fan_mode') != 'auto' }}"
```

This branch handles the case where fan speed changes on the Midea without the climate entity's main state changing. The `fan_mode != 'auto'` condition is critical:

When Midea is in `auto` fan mode, the AC continuously adjusts the fan speed internally, causing the `number.<suffix>_fan_speed` entity to update frequently. Without this guard, the trigger would fire on every such micro-adjustment, flooding the action queue with W100 fan-mode commands (which would all resolve to `auto` anyway, since `_midea_fan == 'auto'` maps to W100 `auto`). The guard prevents this noise entirely — when Midea is in auto fan mode, fan % changes are the AC doing its job, not the user making a selection.

The sequence is identical to the fan-sync portion of Branch 2 but without a trailing `delay` (since this is the only action in the sequence, there's nothing to delay before).

---

## Branch 4: External temperature → W100

```yaml
      - conditions:
          - condition: trigger
            id: ext_temp
          - "{{ enable_external_data }}"
          - "{{ has_value(external_temperature_sensor) }}"
        sequence:
          - if:
              - condition: numeric_state
                entity_id: !input external_temperature_sensor
                above: -100
                below: 100
            then:
              - action: number.set_value
                target:
                  entity_id: "{{ w100_ext_temp_entity }}"
                data:
                  value: "{{ states(external_temperature_sensor) | float }}"
```

Three gates before writing to the W100:

1. `enable_external_data` — the master feature flag. The trigger is also disabled at the trigger level when this is false, but the condition is a belt-and-suspenders check in case of edge cases.

2. `has_value(external_temperature_sensor)` — guards against an undefined or empty sensor entity ID (the default is `[]`). Without this, a misconfigured or empty sensor would cause a template error.

3. `numeric_state above: -100 below: 100` — sanity-checks the sensor value is within a realistic temperature range before writing it to the W100 display. This prevents a faulty sensor reading (e.g., -999 or NaN-adjacent values) from being forwarded. The W100's external temperature entity likely has its own range limits, but validating before writing avoids unnecessary errors in the HA logs.

The W100's external temperature entity ID is derived by the same suffix convention as `midea_fan_entity`: `number.<aqara_w100_suffix>_external_temperature`.

---

## Branch 5: External humidity → W100

```yaml
      - conditions:
          - condition: trigger
            id: ext_hum
          - "{{ enable_external_data }}"
          - "{{ has_value(external_humidity_sensor) }}"
        sequence:
          - if:
              - condition: numeric_state
                entity_id: !input external_humidity_sensor
                above: 0
                below: 100
```

Identical in structure to Branch 4, but for humidity. The range check is `above: 0 below: 100` — physically valid humidity range. Note the lower bound is `above: 0` (exclusive), not `above: -100`; a reading of exactly 0% humidity would be filtered out as unrealistic (and could indicate a sensor error). A reading of exactly 100% would also be filtered — typical humidity sensors cap at 99.x%.

No trailing `delay` in this branch (it's the last action in the sequence).

---

## `mode: queued` / `max: 10`

```yaml
mode: queued
max: 10
```

`queued` means if the automation is triggered while already running, the new run waits in a queue rather than being dropped or interrupting the current run. This is the correct mode for a bidirectional sync where:

- Both devices trigger the same automation
- Each run contains multiple sequential service calls with delays between them
- A run triggered by the Midea should not interrupt a still-executing run triggered by the W100

`max: 10` caps the queue at 10 pending runs. In practice, the debounce delays on triggers mean rapid-fire triggering is already filtered, so the queue depth rarely matters — but it prevents unbounded memory growth in pathological cases (e.g., a sensor spamming state changes).

`single` mode would be wrong here because it would drop the second trigger if the first is still executing. `parallel` would be wrong because concurrent runs could send conflicting commands. `queued` threads the needle: ordered, non-dropping, sequential.

---

## Overall design philosophy

**Feedback loop prevention** operates at two levels:
- **Debounce** (`for: seconds: debounce_delay`) filters transient state changes at the trigger level.
- **Difference guards** (`if states(a) != states(b)`) before every service call prevent writing a value that's already correct, which would cause the target device to report a state change, which would trigger the automation in reverse.

**Write ordering** within each sync branch (mode → temperature → fan, with delays between) respects the dependency chain: mode changes on some ACs affect what temperature range or fan modes are valid, so mode is always set first.

**Asymmetric design** for fan speed: the W100→Midea direction uses `number.set_value` (percentage); the Midea→W100 direction reads the percentage and maps it back to a named mode via nearest-neighbor. This is a consequence of the Midea entering `fan_mode: custom` when controlled via percentage — named modes become a one-way door, so the percentage entity is the canonical source of truth for fan speed.
