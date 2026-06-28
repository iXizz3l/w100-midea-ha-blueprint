# Midea PortaSplit ↔ Aqara W100 Sync

A Home Assistant blueprint that keeps a **Midea PortaSplit air conditioner** and an **Aqara W100 thermostat controller** in sync, bidirectionally, for HVAC mode, temperature setpoint, and fan speed — plus optional push of an external temperature/humidity reading to the W100 display.

## Overview

The Aqara W100 is a wall-mounted thermostat controller with a screen and physical buttons. The Midea PortaSplit is a portable split AC exposed in Home Assistant as a `climate` entity. On their own, the two devices know nothing about each other: turning the AC up from the W100, or changing the mode on the Midea, leaves the other device showing stale values.

This blueprint bridges them. When you change anything meaningful on one device, the change is mirrored to the other, so the W100 always reflects the real state of the AC and the AC always obeys the W100. The challenge is that the two devices model the world differently — different temperature steps, different fan representations, different supported HVAC modes — and a naive mirror would create feedback loops. The blueprint handles all of that translation and loop prevention for you.

## Features

- **Bidirectional HVAC mode sync** — `off`, `cool`, `heat`, and `auto` are mirrored 1:1 in both directions. `heat` on the W100 maps to a Midea mode you choose (see configuration).
- **Bidirectional temperature setpoint sync** — with automatic rounding and clamping so each device receives a value it can accept.
- **Bidirectional fan speed sync** — W100 `low` / `medium` / `high` map to configurable Midea fan-speed percentages; W100 `auto` maps to Midea `auto`. The reverse picks the closest W100 mode for the Midea's current percentage.
- **Optional external sensor display** — push any temperature and/or humidity sensor value to the W100's onboard display.
- **Feedback-loop protection** — a debounce on every trigger plus a write-lockout timestamp prevent the two devices from ping-ponging updates at each other.
- **Preset-safe** — Midea presets that would silently override sync commands are cleared automatically.

## Prerequisites

### W100 thermostat mode switch (manual, one-time)

The W100 only acts as a controller when its **Thermostat Mode** switch is on. This switch is exposed in Home Assistant as:

```
switch.<your_w100_entity_suffix>_thermostat_mode
```

For example, if your W100 climate entity is `climate.climate_sensor_w100_1`, the switch is `switch.climate_sensor_w100_1_thermostat_mode`.

**You must turn this switch on manually, once, before using the blueprint.** Find it under the W100 device page (Settings → Devices & Services → your W100 device) or in Developer Tools → States.

The blueprint deliberately does **not** manage this switch. It is a device-level operating mode, not something that should be toggled by automation logic; leaving it under user control avoids surprising state changes and keeps the blueprint focused on syncing values. Without it on, the W100 will not behave as a controller and sync will not work.

## Required Helper

Before creating the automation you must create one **Date and/or time** helper. The blueprint uses it as a write-lockout timestamp: every time the automation sends a command, it stamps this helper with the current time, and incoming triggers are ignored for a short window afterward. This is what stops the automation from reacting to its own writes.

Create it like this:

1. Go to **Settings → Devices & Services → Helpers**.
2. Click **+ Create Helper**.
3. Choose **Date and/or time**.
4. Give it a name, e.g. `Midea W100 Write Lockout`.
5. **Enable both Date and Time** (the helper must store a full datetime, not just one or the other).
6. Save.

This produces an `input_datetime` entity such as `input_datetime.midea_w100_write_lockout`, which you will select during configuration.

## Installation

### Import the blueprint

1. In Home Assistant go to **Settings → Automations & Scenes → Blueprints**.
2. Click **Import Blueprint**.
3. Paste the raw GitHub URL of the blueprint file:

   ```
   https://raw.githubusercontent.com/iXizz3l/w100-midea-ha-blueprint/main/midea-w100-sync-blueprint.yaml
   ```

4. Click **Preview**, then **Import**.

Alternatively, copy `midea-w100-sync-blueprint.yaml` into `config/blueprints/automation/` and reload blueprints from **Developer Tools → YAML → Reload Blueprints**.

### Create an automation

1. Still under **Settings → Automations & Scenes → Blueprints**, find **Midea PortaSplit ↔ Aqara W100 Sync**.
2. Click **Create Automation**.
3. Fill in the inputs (see [Configuration](#configuration) below). At minimum: the Midea climate entity, the W100 climate entity, and the write-lockout helper you created.
4. Save the automation.

## Configuration

| Input | What it does | Default | Notes |
|---|---|---|---|
| **Midea AC** | The Midea PortaSplit `climate` entity. | _required_ | The blueprint derives the fan-speed entity from this as `number.<suffix>_fan_speed`. |
| **Aqara W100** | The Aqara W100 `climate` entity. | _required_ | The blueprint derives the external temp/humidity entities from this as `number.<suffix>_external_temperature` / `_external_humidity`. |
| **W100 Heat Mode Target** | Which Midea mode to activate when the W100 is set to `heat`. Options: `heat`, `dry`, `fan_only`. | `heat` | When the Midea is in this mode, the reverse sync shows `heat` on the W100 instead of `off`. Use this if your unit has no real heat mode and you want the W100 heat button to trigger, e.g., dry mode. |
| **Fan Speed — Low** | Midea fan-speed % to set when the W100 is `low`. | `1` | Range 1–100, step 1. |
| **Fan Speed — Medium** | Midea fan-speed % to set when the W100 is `medium`. | `50` | Range 1–100, step 1. |
| **Fan Speed — High** | Midea fan-speed % to set when the W100 is `high`. | `100` | Range 1–100, step 1. |
| **Debounce Delay** | Seconds a device state must stay stable before the automation syncs it. | `3` | Range 3–15. Higher values reduce chatter from rapid changes but make sync feel slower. |
| **Sync Delay** | Pause inserted between sequential commands within a single sync run. | `2` | Range 1–5. Gives each device time to settle before the next command. |
| **Write Lockout Helper** | The `input_datetime` helper that records when the automation last sent a command. | _required_ | Must be the Date **and** time helper created in [Required Helper](#required-helper). |
| **Write Lockout Duration** | Seconds to ignore incoming triggers after the automation last wrote to a device. | `10` | Range 5–60. The core defense against reverse feedback loops; raise it if you still see echoed updates. |
| **Enable External Sensor Data** | Master switch for pushing external temp/humidity to the W100 display. | `false` | When off, the external-sensor triggers are disabled entirely. |
| **External Temperature Sensor** | A temperature `sensor` to display on the W100. | _empty_ | Optional. Only used when external data is enabled. |
| **External Humidity Sensor** | A humidity `sensor` to display on the W100. | _empty_ | Optional. Only used when external data is enabled. |
| **External Sensor Debounce** | Seconds an external sensor value must stay stable before being pushed to the W100. | `30` | Range 10–300, step 10. Keeps minor sensor jitter from flooding the display. |

## How it works

### Fan control goes through a percentage, not named fan modes

The blueprint never sets named fan modes (silent, max, etc.) on the Midea. Instead it writes a percentage to the derived `number.<midea_suffix>_fan_speed` entity. The reason: when a fan speed % is set, the Midea climate entity reports its `fan_mode` as `custom`, and the named fan modes cannot be reliably restored or matched afterward. Driving everything through the percentage gives a single, stable representation.

So the three W100 modes — `low`, `medium`, `high` — map to the three configurable percentages, and the reverse direction reads the Midea's current percentage and picks the **closest** configured W100 mode (ties go to the higher mode).

### `auto` fan is a special case

`auto` is the one fan mode that is passed through directly. W100 `auto` sets Midea `fan_mode: auto` (not a percentage), and Midea `auto` sets W100 `auto`. This keeps the AC's own automatic fan behavior intact rather than pinning it to a fixed speed.

### Temperature rounding and clamping

The W100 uses 0.5 °C steps while the Midea uses 1 °C steps with a minimum of 16 °C.

- **W100 → Midea:** the setpoint is always **rounded up** to the nearest whole degree (e.g. 20.5 °C → 21 °C) and then **clamped to a minimum of 16 °C**. If the W100 is set below 16 °C, it keeps showing its own lower value but the Midea receives 16 °C.
- **Midea → W100:** the Midea's whole-degree value is always valid on the W100's finer grid, so it is passed through unchanged.

### `fan_only` and `dry` HVAC modes

The W100 does not support `fan_only` or `dry`. When the Midea reports one of these modes, the W100 is set to `off` — **unless** that mode is the one you configured as the **W100 Heat Mode Target**, in which case the W100 stays on `heat`. This keeps the heat-button mapping consistent in both directions.

### Write lockout prevents reverse feedback loops

Whenever the automation sends a command, it stamps the write-lockout helper with the current time. At the start of each sync branch it checks: if less than **Write Lockout Duration** seconds have passed since that stamp, the run stops immediately. This is what stops the automation from treating its own writes as a fresh user change and bouncing the update back.

### Presets are cleared before syncing

The Midea supports presets (`eco`, `ieco`, `sleep`, `boost`). An active preset will **silently override** fan speed, temperature, and other commands — they appear to be sent but do nothing. So before applying a W100 → Midea sync, the blueprint resets the Midea preset to `none` (only when one is actually active, to avoid unnecessary beeps).

### Debounce on triggers

Every trigger has a `for:` debounce — the state must stay stable for the configured number of seconds before the automation fires. This absorbs rapid, transient updates (and chatty sensor reporting) so the automation doesn't fire constantly, and it gives the write lockout room to take effect.

### Attribute-specific triggers

Rather than firing on every state change, the climate triggers are split so they only act on **meaningful** attributes: `hvac_mode` (the base state), the `temperature` setpoint, and `fan_mode`. A guard additionally drops any plain state-change trigger where the base state didn't actually change (i.e. only the `current_temperature` reading updated). This keeps the automation from running on every ambient-temperature reading from the device's onboard sensor.

The external temperature and humidity sensors get their own triggers, each gated behind the **Enable External Sensor Data** switch and the longer external-sensor debounce.

## Known behaviors

These are intentional and expected — not bugs:

- **The Midea will report `fan_mode: custom`.** Because fan speed is set by percentage, the climate entity shows `custom` rather than a named mode. This is by design.
- **The W100 thermostat mode switch is a one-time manual prerequisite.** The blueprint never touches `switch.<w100_suffix>_thermostat_mode`; you turn it on yourself once. See [Prerequisites](#prerequisites).
- **Midea presets are cleared on sync.** Any active preset (`eco`, `boost`, `sleep`, `ieco`) is reset to `none` when a W100 → Midea command is sent, because presets would otherwise override the synced values. If you rely on a preset, expect it to be cleared the next time you change something on the W100.
- **Swing modes are not synced.** They are ignored entirely.
- **Sub-16 °C setpoints don't reach the Midea.** The W100 can show below 16 °C but the Midea receives the clamped 16 °C minimum.

## Troubleshooting

- **Nothing syncs at all.** Confirm `switch.<w100_suffix>_thermostat_mode` is on. Without it the W100 is not a controller and never produces the changes the automation listens for.
- **Changes bounce back / the devices fight each other.** Increase **Write Lockout Duration** and/or **Debounce Delay**. The lockout is the primary loop guard; a longer window gives each write time to settle before the opposite trigger is evaluated.
- **Fan speed doesn't change on the Midea.** Check that the derived `number.<midea_suffix>_fan_speed` entity exists and matches your unit's naming. The blueprint builds it from the Midea climate entity's suffix; if your integration names it differently, the writes will silently miss.
- **External temperature/humidity not showing on the W100.** Ensure **Enable External Sensor Data** is on, the sensors are selected, and the derived `number.<w100_suffix>_external_temperature` / `_external_humidity` entities exist. Out-of-range or `unknown`/`unavailable` readings are skipped (temperature accepted between −100 and 100 °C, humidity between 0 and 100 %).
- **Temperature is off by half a degree.** Expected — W100 → Midea always rounds up to the nearest whole degree.
- **Sync feels sluggish.** Lower **Debounce Delay** and **Sync Delay**, but be aware that doing so increases the risk of feedback loops; compensate with a slightly higher write lockout if needed.

## Files

| File | Description |
|---|---|
| `midea-w100-sync-blueprint.yaml` | The blueprint. |
| `midea-w100-sync-notes.md` | Planning notes and design decisions. |
