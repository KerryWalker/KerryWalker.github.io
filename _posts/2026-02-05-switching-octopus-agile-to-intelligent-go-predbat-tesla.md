---
layout: post
title: Switching from Octopus Agile to Intelligent Go with Predbat and Tesla
excerpt: How to reconfigure Home Assistant and Predbat when moving from Octopus Agile (Predbat-led charging) to Intelligent Go (Octopus-led charging) while keeping preheating working.
tags:
  - home-assistant
  - predbat
  - tesla
  - hypervolt
  - octopus-intelligent-go
---

I've been running [Octopus Agile](https://share.octopus.energy/lush-cliff-190) with [Predbat](https://springfall2008.github.io/batpred/) controlling both my home battery (Growatt SPH with 13kWh storage) and Tesla Model Y charging via a Hypervolt charger. It's worked brilliantly, but I decided to switch to [Intelligent Go](https://share.octopus.energy/lush-cliff-190) for the simpler pricing structure and potential bonus charging slots.

This post covers the changes needed to move from Predbat-led charging on Agile to Octopus-led charging on Intelligent Go.

## Why Switch?

**Agile** gives you half-hourly variable pricing - great when prices are low, but in winter those rates can spike significantly. During cold snaps with low wind and high demand, Agile prices regularly hit 30-40p/kWh even overnight, and peak hours can exceed 50p/kWh. Predbat handles the optimisation brilliantly, but it can only work with the prices it's given.

**Intelligent Go** gives you:
- Fixed off-peak rate (currently ~7.5p/kWh) from 23:30-05:30 - guaranteed, regardless of wholesale prices
- Fixed peak rate (~24.5p/kWh) the rest of the time
- Bonus cheap slots outside the window when Octopus has excess renewable energy
- Octopus controls your EV charging directly

During winter, having a guaranteed 7.5p overnight rate is valuable. On Agile, those overnight slots might be 7p one night and 25p the next. With Intelligent Go, I know exactly what I'm paying for overnight charging - both for the car and the home battery.

## The Key Difference: Who Controls the Car?

| Aspect | Agile + Predbat | Intelligent Go |
|--------|-----------------|----------------|
| Who calculates cheap slots? | Predbat | Octopus |
| Who controls Tesla charging? | Home Assistant automation | Octopus via Tesla API |
| Charger mode | Plug and charge | Plug and charge (no change) |
| Bonus slots? | No | Yes |
| Predbat's role for car | Plans and executes charging | Just predicts around Octopus slots |

## The Preheating Problem

My main concern with Intelligent Go was preheating. If Octopus controlled my Hypervolt charger's schedule, it would block power outside charging windows - meaning no preheating the cabin before my morning commute.

The solution: **connect the Tesla directly to Octopus, not the Hypervolt**.

When you set up Intelligent Go, you can choose to link either your car or your charger. By linking the Tesla:
- Octopus controls charging via Tesla's API (start/stop commands)
- The Hypervolt stays in "plug and charge" mode, always ready to provide power
- When you preheat, the Tesla draws power from the Hypervolt - Octopus only controls *charging*, not all power draw
- Preheating works normally

## Setting Up Intelligent Go with Tesla

### In the Octopus App

1. Go to **Devices** → **Add device** → **Electric vehicle**
2. Select **Tesla** and your model
   - **Important**: I initially selected "Model Y 2025 RWD Premium" which gave a "we don't support your tech" error
   - Changing to **"Model Y 2025 RWD"** (without Premium) worked fine
   - If you get the unsupported error, try a slightly different model variant
3. When asked about your charger, select a dumb charger option:
   - **"No charger commando plug"** (7kW option), or
   - **"Tesla 7.5kW"** - which is what I chose since it matches the car
   - This tells Octopus your charging rate without connecting your smart charger
4. Sign in to your Tesla account and grant permissions:
   - Vehicle Information
   - Vehicle Charging Management
5. Complete the test charge

### In Home Assistant

Once Intelligent Go is active, make these changes:

#### 1. Enable Octopus Intelligent Charging

Set `switch.predbat_octopus_intelligent_charging` to **ON**.

This was OFF for Agile. Turning it ON tells Predbat to look for Intelligent Go slots from the Octopus Energy integration rather than calculating its own.

#### 2. Disable Predbat Car Charging Planning

Set `switch.predbat_car_charging_plan_smart` to **OFF**.

Predbat no longer needs to plan car charging - Octopus handles that now.

#### 3. Disable the Tesla Charging Automation

If you followed my previous post, you'll have an automation that starts/stops Tesla charging based on `binary_sensor.predbat_car_charging_slot`. 

**Disable this automation** - otherwise you'll have both Predbat and Octopus trying to control your car.

Go to **Settings** → **Automations & Scenes**, find the automation (mine was called "Start/stop Tesla charging based on Predbat's calculated cheap slots"), and toggle it off.

Don't delete it - keep it disabled in case you ever switch back to Agile.

### apps.yaml Changes

Your existing Octopus integration sensors should continue to work. The key sensor Predbat uses for Intelligent Go is:

```yaml
octopus_intelligent_slot: 're:binary_sensor.octopus_energy_.*_intelligent_dispatching'
```

This is already in the default Predbat config. It detects when you're in a cheap slot (either the 23:30-05:30 window or a bonus slot).

The car charging configuration in apps.yaml can stay the same - Predbat still uses this information for predictions:

```yaml
car_charging_soc:
  - sensor.kerrys_tesla_battery_level

car_charging_limit:
  - number.kerrys_tesla_charge_limit

car_charging_planned:
  - binary_sensor.hypervolt_car_plugged

car_charging_energy:
  - sensor.hypervolt_session_energy_total_increasing
```

## How It Works Now

1. **Plug in the Tesla** - Hypervolt is ready, car might briefly start charging
2. **Octopus detects** the car and stops charging within a minute
3. **Set your preferences** in the Octopus app - target SoC and ready-by time
4. **Octopus schedules** charging during cheap slots (23:30-05:30 plus any bonus slots)
5. **Predbat sees** these slots via the Octopus integration
6. **Home battery charges** during the same cheap windows
7. **Preheating works** - just use the Tesla app as normal, the Hypervolt provides power

## Summary of Changes

| Setting/Component | Before (Agile) | After (Intelligent Go) |
|-------------------|----------------|------------------------|
| `switch.predbat_octopus_intelligent_charging` | OFF | **ON** |
| `switch.predbat_car_charging_plan_smart` | ON | **OFF** |
| Tesla charging automation | Enabled | **Disabled** |
| Hypervolt mode | Plug and charge | Plug and charge |
| Who controls Tesla | Predbat automation | Octopus app |
| apps.yaml | No changes needed | No changes needed |

## Things to Watch

- **Bonus slots**: Check the Octopus app to see if you're getting bonus cheap slots outside the main window. Setting your "ready by" time later (e.g., 10am instead of 6am) increases your chances.
- **Battery predictions**: Predbat should now show the Intelligent Go rate structure in its predictions. Check the Predbat plan to verify it's seeing the cheap overnight window.
- **Export rates**: Intelligent Go export rates are typically less favourable than Agile. If you were exporting during Agile price spikes, you'll lose that benefit.

## Results

After a night on Intelligent Go, everything worked as expected. The Octopus app showed the charging schedule, the car charged during the cheap window, and the home battery topped up overnight too. 

Most importantly, preheating worked perfectly the next morning - the Hypervolt provided power for cabin heating while the car was plugged in, with no interference from Octopus.

The setup is simpler now: I just plug in and set my target in the Octopus app. No more watching Agile prices or worrying about whether Predbat picked the right slots.