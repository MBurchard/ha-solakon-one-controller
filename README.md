# Solakon One Charge Controller

Home Assistant blueprint for automated charge control of the Solakon One balcony power station with battery storage.

## Features

- Five charge modes with automatic SOC-based transitions
- Demand-based PV feed-in with dynamic power limit
- Modbus-safe queued execution (no overlapping writes)
- All hardware values use "set only if changed" guards to minimise API calls

## Charge Modes

| Mode | Trigger | Behaviour |
|---|---|---|
| disabled | Manual | Resets hardware to defaults, stops automation |
| safety_charge | SOC < 10% | Grid + PV charging, no discharge, stays until SOC >= 15% |
| pv_protect | SOC >= 10% (or >= 15% from safety_charge) | PV-only charging, no discharge, no feed-in |
| eco_feed | SOC > 20% | PV feed-in to house, demand-based, no discharge |
| power_boost | SOC > 70% (auto) or manual from >= 20% | Like eco_feed with battery discharge |

## Prerequisites

Before importing this blueprint you need to create two helpers manually:

### 1. Input Select (Charge Mode)

Create an input select helper (Settings > Devices & Services > Helpers > Input Select) with exactly these options:

- `disabled`
- `safety_charge`
- `pv_protect`
- `eco_feed`
- `power_boost`

### 2. Statistics Sensor (Demand Smoothing)

Create a statistics sensor (Settings > Devices & Services > Helpers > Statistics) on your energy consumption sensor with the following recommended settings:

| Setting | Value |
|---|---|
| Statistic characteristic | Minimum |
| Sample size | 50 |
| Max age | 15 seconds |

This smooths out load spikes from appliances (washing machines, induction hobs, dryers, etc.) and prevents unnecessary grid feed-in. Use this statistics sensor as the "Demand Sensor" input in the blueprint, not the raw consumption sensor.

## Installation

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FMBurchard%2Fha-solakon-one-controller%2Fblob%2Fmain%2Fblueprints%2Fsolakon_one.yaml)

Or import manually: Settings > Automations & Scenes > Blueprints > Import Blueprint, then paste:

```
https://github.com/MBurchard/ha-solakon-one-controller/blob/main/blueprints/solakon_one.yaml
```

## Configuration

After importing, create a new automation from the blueprint. The most important settings:

### Entities

You need to assign all Solakon One entities (sensors, selects, numbers) from your integration. The blueprint will guide you through entity selection.

### Key Parameters

| Parameter | Default | Description |
|---|---|---|
| Max Active Power | 1200 W | Maximum feed-in power with EPS load (external socket) |
| Max House Feed | 800 W | Maximum house grid feed-in power (German legal limit) |
| Max Grid Charge | 600 W | Maximum power when charging from grid |
| Min SOC | 10% | Below this SOC, grid charging starts |
| Min SOC Feed | 20% | Below this SOC, battery is protected (no feed-in) |
| Min SOC Power Boost | 70% | SOC threshold for automatic power boost |
| Max Discharge Current | 40 A | Battery discharge current in power boost mode |

### Remote Control Mode Values

The default mode values (0 = off, 1 = inverter export, 3 = PV priority charge) can be adjusted if the Solakon One firmware changes the mapping.

## License

[MIT](LICENSE)
