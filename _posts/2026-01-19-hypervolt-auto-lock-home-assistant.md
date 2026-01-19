---
layout: post
title: Auto-locking a Hypervolt EV Charger with Home Assistant and Tesla
tags:
  - home-assistant
  - hypervolt
  - tesla
---

I recently set up some Home Assistant automations to automatically lock and unlock my Hypervolt EV charger when my Tesla plugs in or unplugs. The goal was simple: keep the charger locked and secure when not in use, but have it automatically unlock when I plug in my car.

## The Problem

The Hypervolt charger has a lock feature that prevents unauthorised use - handy if your charger is accessible to others. But manually locking and unlocking it every time you charge is tedious. I wanted it to just work: unlock when I plug in my Tesla, lock when I unplug.

## The Solution

The key insight is that we can detect when *my* Tesla is plugged in by combining two sensors:

- `binary_sensor.hypervolt_car_plugged` - the Hypervolt knows a car is connected
- `binary_sensor.kerrys_tesla_charge_cable` - the Tesla knows its cable is connected

If both are true, my Tesla must be plugged into my Hypervolt. No GPS tracking needed - if the Hypervolt at my house has a car plugged in, and my Tesla confirms its cable is connected, it can only be my Tesla at my house.

## The Automations

### Unlock When Tesla Plugs In

```yaml
alias: "Hypervolt Unlock - Tesla Plugged In"
description: "Unlock Hypervolt when Tesla is plugged in"
mode: single

triggers:
  - trigger: state
    entity_id: binary_sensor.hypervolt_car_plugged
    to: "on"
  - trigger: state
    entity_id: binary_sensor.kerrys_tesla_charge_cable
    to: "on"

conditions:
  - condition: state
    entity_id: binary_sensor.hypervolt_car_plugged
    state: "on"
  - condition: state
    entity_id: binary_sensor.kerrys_tesla_charge_cable
    state: "on"

actions:
  - action: switch.turn_off
    target:
      entity_id: switch.hypervolt_lock_state
```

The automation triggers when either sensor turns on, but only fires if both conditions are met. This handles the slight timing difference between the Hypervolt detecting a car and the Tesla reporting its cable status.

### Lock When Tesla Unplugs

```yaml
alias: "Hypervolt Lock - Tesla Unplugged"
description: "Lock Hypervolt when Tesla unplugs"
mode: single

triggers:
  - trigger: state
    entity_id: binary_sensor.hypervolt_car_plugged
    to: "off"
  - trigger: state
    entity_id: binary_sensor.kerrys_tesla_charge_cable
    to: "off"

conditions:
  - condition: state
    entity_id: binary_sensor.hypervolt_car_plugged
    state: "off"
  - condition: state
    entity_id: binary_sensor.kerrys_tesla_charge_cable
    state: "off"

actions:
  - action: switch.turn_on
    target:
      entity_id: switch.hypervolt_lock_state
```

Same logic in reverse - lock when both sensors show disconnected.

## Why This Works

The dual-sensor approach handles several edge cases nicely:

| Scenario | Hypervolt | Tesla Cable | Result |
|----------|-----------|-------------|--------|
| Tesla plugs into Hypervolt | on | on | Unlocks ✓ |
| Tesla unplugs from Hypervolt | off | off | Locks ✓ |
| Tesla unplugs at public charger | unchanged | off | Nothing ✓ |
| Random car plugs into Hypervolt | on | off | Nothing ✓ |
| Random car unplugs from Hypervolt | off | off | Locks ✓ |

The last case is actually fine - if a random car was using the charger (which I'd have manually unlocked), it locks again when they finish.

## Guest Charging

If I want to let someone else use the charger, I just manually unlock it. When they unplug, it automatically locks again. No need to remember to secure it afterwards.

## Prerequisites

You'll need:

- [Hypervolt Home Assistant integration](https://github.com/gndean/home-assistant-hypervolt-charger) for the charger sensors
- [Tesla Fleet integration](https://www.home-assistant.io/integrations/tesla_fleet/) for the Tesla sensors

Both are straightforward to set up and expose the sensors used in these automations.
