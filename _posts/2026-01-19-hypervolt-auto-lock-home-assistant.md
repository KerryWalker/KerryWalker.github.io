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

## First Attempt: Location-Based Automation

My initial approach seemed logical: use the Tesla's location to detect when it's home, then unlock the charger. The automation checked three things:

1. Tesla is at home (via `device_tracker.kerrys_tesla_location`)
2. Hypervolt detects a car plugged in
3. Tesla reports its cable is connected

Simple enough, right? Except it didn't work reliably.

### Why Tesla Location Tracking Is Unreliable

The problem is that Teslas sleep aggressively to conserve battery. When asleep, the Tesla Fleet API can't get live data, so the location sensor shows stale information or simply "unknown". I'd pull into my driveway, plug in, and... nothing. The Tesla was still reporting "not_home" because it hadn't woken up to update its location yet.

You can work around this with a more forgiving zone check:

```yaml
template:
  - binary_sensor:
      - name: "Tesla Home"
        state: >
          {{ distance('device_tracker.kerrys_tesla_location', 'zone.home') | float < 0.5 }}
        availability: >
          {{ states('device_tracker.kerrys_tesla_location') not in ['unknown', 'unavailable'] }}
```

This checks if the Tesla is within 0.5km of home, which is more tolerant of GPS drift. But it still relies on the location actually updating, which doesn't happen reliably when the car is asleep.

## The Compromise

I had to accept that reliable location tracking wasn't going to happen without constantly waking the car, which defeats the purpose of letting it sleep. The trade-off became clear: I could either have accurate location data and a car that drains its battery polling for updates, or I could find another way.

The compromise was to drop location entirely and rely on what I *could* trust - the physical connection. If my Hypervolt (which is fixed at my house) detects a car plugged in, AND my Tesla confirms its cable is connected, then it must be my Tesla at my house. The Hypervolt itself becomes the location check. It's not as elegant as "Tesla arrives home, charger unlocks", but it's bulletproof.


## The Final Solution

Two sensors, no GPS:

- `binary_sensor.hypervolt_car_plugged` - the Hypervolt knows a car is connected
- `binary_sensor.kerrys_tesla_charge_cable` - the Tesla knows its cable is connected

If both are true, my Tesla is plugged into my Hypervolt. Done.

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

I initially had it only trigger on the Hypervolt sensor, with the Tesla cable as a condition. But the timing was inconsistent - sometimes the Hypervolt would report first, sometimes the Tesla would. Triggering on both and checking both as conditions made it much more reliable.

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

## Lessons Learned

1. **Don't trust Tesla location data** - the car sleeps too aggressively for real-time location tracking to be reliable
2. **Physical sensors beat GPS** - the Hypervolt knowing a car is plugged in is more reliable than GPS coordinates
3. **Trigger on multiple sensors** - when two things need to happen, trigger on both and use conditions to ensure both are true
4. **Simple is better** - removing the location check made the automation both simpler and more reliable

## Prerequisites

You'll need:

- [Hypervolt Home Assistant integration](https://github.com/gndean/home-assistant-hypervolt-charger) for the charger sensors
- [Tesla Fleet integration](https://www.home-assistant.io/integrations/tesla_fleet/) for the Tesla sensors

Both are straightforward to set up and expose the sensors used in these automations.
