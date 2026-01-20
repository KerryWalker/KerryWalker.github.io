---
layout: post
title: Predbat EV Charging with Tesla Fleet and Hypervolt on Octopus Agile
excerpt: How to configure Predbat to control Tesla charging via Home Assistant, using the cheapest Octopus Agile slots with a Hypervolt charger.
tags:
  - home-assistant
  - predbat
  - tesla
  - hypervolt
  - octopus-agile
---

Following on from my Predbat battery configuration, this post covers adding EV charging for a Tesla Model Y using the Tesla Fleet integration and a Hypervolt charger, all controlled by Predbat to charge during the cheapest Octopus Agile slots.

## The Setup

- Tesla Model Y 2025 Standard Range (~60kWh usable battery)
- Hypervolt EV charger (32A single phase, ~7.4kW)
- Tesla Fleet integration in Home Assistant
- Hypervolt charger integration in Home Assistant
- Predbat already configured for battery optimisation

## The Approach: Predbat-Led Charging

There are several ways to control EV charging, but I wanted to keep the Hypervolt in simple "plug and charge" mode and control everything via the Tesla. The flow looks like this:

```
Hypervolt: Always ready to provide power (plug & charge mode)
                    ↓
Predbat: Calculates cheapest Agile slots
         Sets binary_sensor.predbat_car_charging_slot ON/OFF
                    ↓
Home Assistant Automation:
         When slot ON  → Start Tesla charging
         When slot OFF → Stop Tesla charging
                    ↓
Tesla: Charges when instructed, draws power from Hypervolt
```

This keeps the Hypervolt configuration simple and puts all the intelligence in Home Assistant.

## apps.yaml Car Charging Configuration

Add this section to your existing Predbat `apps.yaml`:

```yaml
# ===========================================
# CAR CHARGING CONFIGURATION
# ===========================================

num_cars: 1

# Tesla Model Y Standard Range (~60kWh usable)
car_charging_battery_size:
  - 60.0

# Tesla charge limit sensor
car_charging_limit:
  - number.kerrys_tesla_charge_limit

# Tesla current battery SoC
car_charging_soc:
  - sensor.kerrys_tesla_battery_level

# Detect if car is plugged in (Hypervolt sensor is more reliable than Tesla when asleep)
car_charging_planned:
  - binary_sensor.hypervolt_car_plugged

car_charging_planned_response:
  - 'on'
  - 'true'
  - 'yes'

# Energy used for car charging (for filtering from house load)
car_charging_energy:
  - sensor.hypervolt_session_energy_total_increasing

# Only one car can charge at a time
car_charging_exclusive:
  - True

# ===========================================
# DO NOT set car_charging_now
# This would create a circular dependency with Predbat-led charging
# ===========================================
```

Note: I'm using the Hypervolt's `binary_sensor.hypervolt_car_plugged` rather than the Tesla's sensor for detecting if the car is plugged in. The Hypervolt updates immediately when you plug in, whereas the Tesla might be asleep and show stale data.

## Home Assistant Settings

After adding the car configuration and waiting for Predbat to reload, configure these settings in Home Assistant:

| Entity | Value | Reason |
|--------|-------|--------|
| `switch.predbat_octopus_intelligent_charging` | OFF | You're on Agile, not Intelligent GO |
| `switch.predbat_car_charging_hold` | ON | Filters car charging from historical load data |
| `switch.predbat_car_charging_plan_smart` | ON | Uses only the cheapest slots, not all low-rate slots |
| `input_number.predbat_car_charging_rate` | 7.4 | Hypervolt at 32A single phase ≈ 7.4kW |
| `input_number.predbat_car_charging_energy_scale` | 0.001 | Converts Hypervolt's Wh to kWh |
| `select.predbat_car_charging_plan_time` | Your choice | Time you need the car ready (e.g., 07:00) |
| `input_number.predbat_car_charging_plan_max_price` | Optional | Maximum price in pence you're willing to pay |

## The Charging Automations

You need two automations: one to control charging based on Predbat's slot sensor, and one to prevent the car charging immediately when plugged in.

### Automation 1: Predbat Tesla Charging Control

This starts and stops charging based on Predbat's calculated cheap slots:

```yaml
alias: "Predbat Tesla Charging Control"
description: "Start/stop Tesla charging based on Predbat's calculated cheap slots"
mode: single

triggers:
  - trigger: state
    entity_id: binary_sensor.predbat_car_charging_slot
    to: "on"
    id: start_charging
  - trigger: state
    entity_id: binary_sensor.predbat_car_charging_slot
    to: "off"
    id: stop_charging

conditions:
  - condition: state
    entity_id: binary_sensor.hypervolt_car_plugged
    state: "on"

actions:
  - choose:
      - conditions:
          - condition: trigger
            id: start_charging
        sequence:
          - action: button.press
            target:
              entity_id: button.kerrys_tesla_wake
          - delay:
              seconds: 30
          - action: switch.turn_on
            target:
              entity_id: switch.kerrys_tesla_charge
          - action: logbook.log
            data:
              name: "Predbat EV"
              message: "Started Tesla charging - cheap slot active"

      - conditions:
          - condition: trigger
            id: stop_charging
        sequence:
          - action: button.press
            target:
              entity_id: button.kerrys_tesla_wake
          - delay:
              seconds: 30
          - action: switch.turn_off
            target:
              entity_id: switch.kerrys_tesla_charge
          - action: logbook.log
            data:
              name: "Predbat EV"
              message: "Stopped Tesla charging - cheap slot ended"
```

The 30-second delay after waking allows the Tesla to fully wake up before sending the charge command. Teslas sleep aggressively to save battery, so this wake step is essential.

### Automation 2: Stop Charging on Plugin

The problem: when you plug in the Tesla, the Hypervolt immediately provides power and the Tesla starts charging - regardless of whether it's a cheap slot. This automation stops that immediate charging:

```yaml
alias: "Tesla - Stop Charging on Plugin (Let Predbat Control)"
description: "Prevent immediate charging when plugged in - Predbat will start it during cheap slots"
mode: single

triggers:
  - trigger: state
    entity_id: binary_sensor.hypervolt_car_plugged
    to: "on"
    id: hypervolt_plugged
  - trigger: state
    entity_id: binary_sensor.kerrys_tesla_charge_cable
    to: "on"
    id: tesla_cable
  - trigger: state
    entity_id: sensor.kerrys_tesla_charging
    to: "charging"
    id: tesla_charging

conditions:
  - condition: state
    entity_id: binary_sensor.predbat_car_charging_slot
    state: "off"
  - condition: or
    conditions:
      - condition: state
        entity_id: binary_sensor.hypervolt_car_plugged
        state: "on"
      - condition: state
        entity_id: binary_sensor.kerrys_tesla_charge_cable
        state: "on"

actions:
  - delay:
      seconds: 30
  - condition: state
    entity_id: binary_sensor.predbat_car_charging_slot
    state: "off"
  - action: switch.turn_off
    target:
      entity_id: switch.kerrys_tesla_charge
  - action: logbook.log
    data:
      name: "Predbat EV"
      message: "Stopped immediate charging - waiting for cheap slot"
```

This triggers on multiple sensors (Hypervolt plugged, Tesla cable connected, or Tesla starting to charge) because the timing between them can vary. It only stops charging if it's not already a cheap slot - so if you happen to plug in during a cheap window, it'll let it charge.

## Tesla Fleet API Costs

Tesla's Fleet API has usage limits, but there's a free tier that covers typical home use. You might send 10-20 commands per day (wake, start charging, stop charging), which is well within the free allowance. You can monitor your usage at [developer.tesla.com/dashboard](https://developer.tesla.com/dashboard).

## Handling "Unknown" Sensors

When you first set this up, you might see errors in Predbat showing Tesla sensors as "unknown". This is normal - when a Tesla is asleep, the Fleet API can't retrieve live data. The sensors will populate once the car wakes up (either when you use it, or when an automation wakes it for charging).

Predbat handles this gracefully and will work correctly once the car is plugged in and awake.

## The Complete Flow

1. Plug in Tesla → Hypervolt detects car
2. Tesla tries to charge immediately → "Stop on Plugin" automation stops it
3. Predbat sees car is plugged in, calculates cheapest Agile slots
4. When cheap slot starts → `binary_sensor.predbat_car_charging_slot` turns ON
5. "Predbat Tesla Charging Control" automation wakes car and starts charging
6. When cheap slot ends → automation stops charging
7. Repeat until target SoC reached

## Important: Tesla Charge Limit

Predbat uses your Tesla's charge limit setting (`number.kerrys_tesla_charge_limit`) to determine how much charging is needed. If this is set to 50%, Predbat will only plan charging up to 50% - even if the battery is at 20%.

Set this to your desired charge level in the Tesla app or via Home Assistant before plugging in.

## Results

With this setup, I plug in when I get home and forget about it. Predbat identifies the cheapest overnight slots (typically 1am-5am on Agile) and charges the car automatically. The car is ready at my specified time, charged at a fraction of peak electricity costs.

The combination of battery storage and EV charging optimisation means I'm using grid power almost exclusively during the cheapest periods, with solar and battery covering peak times.