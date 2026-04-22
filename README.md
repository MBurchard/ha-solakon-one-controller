# Solakon One Charge Controller [v008]

Home Assistant blueprint for automated charge control of the Solakon One balcony power station with battery storage.

## Features

- Five charge modes with automatic SOC-based transitions
- Demand-based PV feed-in with dynamic power limit
- Modbus-safe queued execution (no overlapping writes)
- All hardware values use "set only if changed" guards to minimize API calls

## Charge Modes

| Mode          | Trigger                                   | Behaviour                                                |
|---------------|-------------------------------------------|----------------------------------------------------------|
| disabled      | Manual                                    | Resets hardware to defaults, stops automation            |
| safety_charge | SOC < 10%                                 | Grid + PV charging, no discharge, stays until SOC >= 15% |
| pv_protect    | SOC >= 10% (or >= 15% from safety_charge) | PV-only charging, no discharge, no feed-in               |
| eco_feed      | SOC > 20%                                 | PV feed-in to house, demand-based, no discharge          |
| power_boost   | SOC > 70% (auto) or manual from >= 20%    | Like eco_feed with battery discharge                     |

## Solar Overflow Regulator

At and above the overflow SOC threshold (default 98%) a direct regulator takes over `active_power`. The Solakon
One never reports its real PV potential when the battery is near full (the MPPT curtails internally), so the
regulator uses the battery charge and discharge sensors as feedback:

- **Charge > 0, discharge == 0:** PV produces more than we feed in, the surplus charges the battery. Add exactly
  that surplus to `active_power`.
- **Discharge > 0, charge == 0:** PV cannot cover the feed-in, battery closes the gap. Subtract exactly that
  amount from `active_power`.
- **Both zero:** battery idle but PV might still have headroom (typical at 100 % SOC where the battery cannot
  accept charge). Probe upwards by `power_step` (default 25 W). Oscillation around the ceiling is bounded by the
  write throttle.
- **Both non-zero** (rare quantisation edge): hold.

Writes on `active_power` inside the overflow zone are throttled to once per `overflow_throttle` seconds (default
5 s). The Solakon One briefly drains the battery when active_power rises before its PV ramps up, so reacting too
fast bounces. Outside the overflow zone (SOC < overflow threshold), the normal demand-based feed-in applies
without throttle. `active_power` is clamped between `demand - min_net_power` (the house stays covered, discharge
there is expected power_boost behaviour) and `max_house_feed` (legal limit). RC mode stays on 1 (inverter export)
throughout, discharge current stays mode-specific. Works in both eco_feed and power_boost.

## Prerequisites

Before importing this blueprint, you need to create two helpers manually:

### 1. Input Select (Charge Mode)

Create an input select helper (Settings > Devices & Services > Helpers > Input Select) with exactly these options:

- `disabled`
- `safety_charge`
- `pv_protect`
- `eco_feed`
- `power_boost`

### 2. Statistics Sensor (Demand Smoothing)

Create a statistics sensor (Settings > Devices & Services > Helpers > Statistics) on your energy consumption sensor with
the following recommended settings:

| Setting                  | Value      |
|--------------------------|------------|
| Statistic characteristic | Minimum    |
| Sample size              | 50         |
| Max age                  | 15 seconds |

This smooths out load spikes from appliances (washing machines, induction hobs, dryers, etc.) and prevents unnecessary
grid feed-in.
Use this statistics sensor as the "Demand Sensor" input in the blueprint, not the raw consumption sensor.

## Installation

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FMBurchard%2Fha-solakon-one-controller%2Fblob%2Fmain%2Fblueprints%2Fsolakon_one.yaml)

Or import manually: Settings > Automations & Scenes > Blueprints > Import Blueprint, then paste:

```
https://github.com/MBurchard/ha-solakon-one-controller/blob/main/blueprints/solakon_one.yaml
```

## Configuration

After importing, create a new automation from the blueprint. The most important settings:

### Entities

You need to assign all Solakon One entities (sensors, selects, numbers) from your integration. The blueprint will guide
you through entity selection.

### Key Parameters

| Parameter              | Default | Description                                           |
|------------------------|---------|-------------------------------------------------------|
| Max Active Power       | 1200 W  | Maximum feed-in power with EPS load (external socket) |
| Max House Feed         | 800 W   | Maximum house grid feed-in power (German legal limit) |
| Max Grid Charge        | 600 W   | Maximum power when charging from grid                 |
| Min SOC                | 10%     | Below this SOC, grid charging starts                  |
| Min SOC Feed           | 20%     | Below this SOC, battery is protected (no feed-in)     |
| Min SOC Power Boost    | 70%     | SOC threshold for automatic power boost               |
| Max Discharge Current  | 40 A    | Battery discharge current in power boost mode         |
| Overflow SOC Threshold | 98%     | SOC at and above which the overflow regulator runs    |
| Overflow Write Throttle| 5 s     | Minimum seconds between active_power writes in overflow |
| Regulator Probe Step   | 25 W    | Step size when both charge and discharge are zero     |

### Remote Control Mode Values

The default mode values (0 = off, 1 = inverter export, 3 = PV priority charge) can be adjusted if the Solakon One
firmware changes the mapping.

## Logging

The blueprint fires `custom_log` events at key decision points (mode transitions, power adjustments, charging
decisions).
These events are completely optional. If no listener is configured, they are silently discarded without any errors or
log entries.

### Event Structure

Each `custom_log` event contains the following data:

| Field   | Description                                    |
|---------|------------------------------------------------|
| channel | Log channel identifier (always `solakon`)      |
| level   | Log level (`debug`)                            |
| message | Human-readable description of the action taken |

### Enabling Log Output

To capture these events, create a trigger-based template sensor. Add the following block to your `template.yaml`
(or the Templates section in your HA configuration):

```yaml
- trigger:
    - platform: event
      event_type: custom_log
  sensor:
    - name: "User Log"
      unique_id: sensor.user_log
      state: "{{ trigger.event.data.message }}"
      attributes:
        channel: "{{ trigger.event.data.channel }}"
        level: "{{ trigger.event.data.level }}"
        timestamp: "{{ now().isoformat() }}"
```

The sensor state will always show the most recent log message. You can inspect it in Developer Tools > States or use it
in further automations (e.g. to forward messages to a notification service).

## License

[MIT](LICENSE)
