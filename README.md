# w100-midea-ha-blueprint

A Home Assistant blueprint for bidirectional sync between a **Midea PortaSplit AC** and an **Aqara W100** thermostat controller.

## What it does

Keeps both devices in sync — changes made on either the Midea or the W100 are reflected on the other:

- **HVAC mode** (off / cool / heat / auto) — bidirectional
- **Temperature setpoint** — bidirectional
- **Fan speed** — bidirectional (via fan speed % on the Midea side)
- **External temperature / humidity** — push a sensor reading to the W100 display (optional)

## Prerequisites

The W100 **Thermostat Mode** switch must be `on` before the blueprint will work. Enable it once manually in HA:

```
switch.<your_w100_entity_suffix>_thermostat_mode
```

The blueprint does not manage this switch.

## Installation

1. Copy `midea-w100-sync-blueprint.yaml` into your HA blueprints folder:
   ```
   config/blueprints/automation/
   ```
2. Reload blueprints in HA (Developer Tools → YAML → Reload Blueprints)
3. Go to **Settings → Automations → Blueprints** and create a new automation from the blueprint

## Configuration

| Input | Description | Default |
|---|---|---|
| Midea AC | The Midea PortaSplit climate entity | required |
| Aqara W100 | The W100 climate entity | required |
| W100 Heat Mode Target | Which Midea mode activates when W100 is set to heat (`heat`, `dry`, `fan_only`) | `heat` |
| Fan Speed — Low | Midea fan speed % for W100 Low mode | 1% |
| Fan Speed — Medium | Midea fan speed % for W100 Medium mode | 50% |
| Fan Speed — High | Midea fan speed % for W100 High mode | 100% |
| Debounce Delay | Seconds a state must be stable before syncing (prevents feedback loops) | 3s |
| Sync Delay | Pause between sequential commands in one sync run | 2s |
| Enable External Sensor Data | Push temperature/humidity to the W100 display | off |
| External Temperature Sensor | Optional sensor to display on W100 | — |
| External Humidity Sensor | Optional sensor to display on W100 | — |

## Notes

- **Fan speed**: The Midea reports `fan_mode: custom` when controlled by percentage. Named fan modes (silent, max, etc.) are never set — all fan control goes through the `number.<suffix>_fan_speed` entity (1–100%).
- **W100 `auto` fan mode** maps directly to Midea `auto` (not a percentage).
- **Midea presets** (`eco`, `sleep`, `boost` etc.) are reset to `none` before every sync — active presets silently override all other commands on the Midea.
- **Temperature rounding**: W100 uses 0.5°C steps, Midea uses 1°C. Values are always rounded up when syncing W100→Midea, and clamped to the Midea's minimum of 16°C.
- **Loop prevention**: All triggers use a debounce (`for:`) and every sync branch checks for a difference before writing, preventing feedback loops.

## Files

| File | Description |
|---|---|
| `midea-w100-sync-blueprint.yaml` | The blueprint |
| `how.md` | Block-by-block explanation of the blueprint code |
