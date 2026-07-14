---
layout: post
affiliate: true
title: Updating Predbat Home Battery Charging for Octopus Intelligent Go
excerpt: After switching from Octopus Agile to Intelligent Go and setting up Tesla charging, I'd missed updating Predbat's apps.yaml for home battery charging. Here's what needed changing.
tags:
  - home-assistant
  - predbat
  - octopus-intelligent-go
  - growatt
---

In my [previous post](/2026/02/05/switching-octopus-agile-to-intelligent-go-predbat-tesla/) I covered switching from Octopus Agile to [Intelligent Go](https://share.octopus.energy/lush-cliff-190) and getting Octopus to control my Tesla charging. That all went smoothly, but I'd missed something - I hadn't updated Predbat's `apps.yaml` to properly handle home battery charging on the new tariff.

On Agile, Predbat was doing all the heavy lifting - calculating the cheapest half-hourly slots and charging the Growatt batteries accordingly. With Intelligent Go the rates are fixed (7p off-peak, 30.5p peak), but Predbat still needs to know about the Intelligent Go dispatch slots so it can charge the home battery during cheap periods, including any bonus slots Octopus grants outside the standard 23:30-05:30 window.

## The Missing Piece: octopus_intelligent_slot

The key setting I was missing was `octopus_intelligent_slot`. This tells Predbat about the Intelligent Go dispatch slots from the Octopus Energy integration. Without it, Predbat could see the rates but had no awareness of bonus cheap slots that Octopus might grant for car charging - slots that the home battery should also be charging during.

Adding this to `apps.yaml`:

```yaml
octopus_intelligent_slot: 're:(binary_sensor.octopus_energy([0-9a-z_]+|)_intelligent_dispatching)'
```

## The Regex Match Problem

After adding the line and restarting Predbat, I got this warning in the log:

```
Warn: Regular expression argument: octopus_intelligent_slot unable to match
re:(binary_sensor.octopus_energy([0-9a-z_]+|)_intelligent_dispatching), now will disable
```

The regex wasn't matching because the Octopus Energy integration hadn't created the `intelligent_dispatching` entity yet. Because I'd linked my Tesla directly to Octopus (rather than linking the Hypervolt charger), the integration needed a refresh to pick up the Intelligent Go device and create the dispatching sensor.

The fix was simple - go into **Settings → Devices & Services → Octopus Energy → Configure** and refresh the integration. After that, the entity appeared:

```
binary_sensor.octopus_energy_xxxxxxxx_xxxx_xxxx_xxxx_xxxxxxxxxxxx_intelligent_dispatching
```

Note the entity name uses the Tesla's device UUID rather than the electricity meter serial number, which is why the default regex didn't initially match - the entity simply didn't exist yet. The regex in the default Predbat template matched it fine once the entity existed.

## Preventing Off-Peak Cycling

On Agile, Predbat might see profitable opportunities to charge and discharge within a single cheap window. On Intelligent Go this doesn't make sense - you want to charge during the off-peak window and hold that charge. Two settings prevent this cycling behaviour:

```yaml
combine_charge: true
calculate_export_during_charge: false
```

`combine_charge` tells Predbat to treat the cheap charging window as one block rather than individual slots. `calculate_export_during_charge` being false stops Predbat from trying to export during that same window, which it might otherwise see as economically sensible given the 15p export rate versus 7p import.

## Forecast Plan Hours

On Agile I had `forecast_plan_hours` set to 30 to match the roughly 30 hours of known Agile prices available at any time. With Intelligent Go the rates are fixed and repeat every 24 hours, so this can come down to 24. It gives quicker plan updates since there's less to calculate:

```yaml
forecast_plan_hours: 24
```

`forecast_hours` stays at 48 - that's for solar and load prediction which still benefits from looking further ahead.

## All the apps.yaml Changes

Here's a summary of what changed in `apps.yaml` from my Agile configuration:

| Setting | Agile Value | Intelligent Go Value |
|---------|-------------|---------------------|
| Section comment | `# OCTOPUS ENERGY RATES (Agile)` | `# OCTOPUS ENERGY RATES (Intelligent Go)` |
| `octopus_intelligent_slot` | Not present | `'re:(binary_sensor.octopus_energy([0-9a-z_]+|)_intelligent_dispatching)'` |
| `combine_charge` | Not present | `true` |
| `calculate_export_during_charge` | Not present | `false` |
| `forecast_plan_hours` | 30 | 24 |

The rate sensors (`metric_octopus_import`, `metric_octopus_export`, `metric_standing_charge`) didn't need changing - the Octopus Energy integration automatically provides the correct Intelligent Go rates using the same regex patterns.

## Home Assistant UI Settings

These were already set when I switched the car charging over, but for completeness:

| Setting | Value |
|---------|-------|
| `switch.predbat_octopus_intelligent_charging` | ON |
| `switch.predbat_car_charging_plan_smart` | OFF |
| `input_number.predbat_best_soc_keep` | 2.0 |
| `input_number.predbat_best_soc_min` | 0 |
| `input_number.predbat_forecast_plan_hours` | 24 |

## The Result

After making these changes, Predbat's log showed it successfully finding the Intelligent slot sensor. The plan now shows battery charging scheduled during the 23:30-05:30 off-peak window, and if Octopus grants any bonus slots for car charging, the home battery will take advantage of those too.

The combination of Octopus controlling the Tesla and Predbat controlling the home battery is working well. Each does what it's best at - Octopus handles the car charging schedule and negotiates bonus slots, while Predbat optimises the home battery around those slots and the solar forecast.