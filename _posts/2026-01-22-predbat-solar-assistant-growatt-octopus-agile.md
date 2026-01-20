---
layout: post
title: Setting Up Predbat with Solar Assistant and Growatt SPH for Octopus Agile
excerpt: A complete guide to configuring Predbat for battery charging optimisation with a Growatt SPH inverter, Solar Assistant, and Octopus Agile tariff.
tags:
  - home-assistant
  - predbat
  - solar-assistant
  - growatt
  - octopus-agile
---

I've been running a Growatt SPH inverter with two GBLI 6.5kWh batteries (13kWh total) on Octopus Agile, using Predbat to optimise battery charging during the cheapest half-hour slots. Getting the configuration right took some trial and error, so here's the complete setup.

## The Hardware

- Growatt SPH hybrid inverter
- 2× GBLI 6.5kWh batteries (13kWh total)
- Solar Assistant (for Modbus communication with the inverter)
- Home Assistant with Predbat via AppDaemon

## Why Predbat?

Octopus Agile prices change every 30 minutes, with prices published day-ahead. Predbat analyses these prices alongside your solar forecast (via Solcast) and historical consumption to determine the optimal times to charge your battery from the grid. On Agile, this can mean charging at 5p/kWh overnight instead of 30p/kWh during peak hours.

## Solar Assistant Entities

Solar Assistant exposes your Growatt inverter to Home Assistant via MQTT. The key entities you'll need are:

### Sensors (Read-only)

| Purpose | Entity |
|---------|--------|
| Battery Power | `sensor.growatt_sph_battery_power` |
| Battery SoC | `sensor.growatt_sph_battery_state_of_charge` |
| Battery Voltage | `sensor.growatt_sph_battery_voltage` |
| Battery Temperature | `sensor.growatt_sph_battery_temperature` |
| PV Power | `sensor.growatt_sph_pv_power` |
| Grid Power | `sensor.growatt_sph_grid_power` |
| Load Power | `sensor.growatt_sph_load_power` |
| Load Energy Today | `sensor.growatt_sph_load_energy` |
| Grid Import Today | `sensor.growatt_sph_grid_energy_in` |
| Grid Export Today | `sensor.growatt_sph_grid_energy_out` |
| PV Energy Today | `sensor.growatt_sph_pv_energy` |

### Controls (Writable)

| Purpose | Entity |
|---------|--------|
| Work Mode | `select.growatt_sph_work_mode_priority` |
| Charge Rate % | `number.growatt_sph_battery_first_charge_rate` |
| Discharge Rate % | `number.growatt_sph_grid_first_discharge_rate` |
| Charge Target SoC | `number.growatt_sph_battery_first_stop_charge` |
| Discharge Stop SoC | `number.growatt_sph_grid_first_stop_discharge` |
| Reserve SoC | `number.growatt_sph_load_first_stop_discharge` |
| Charge Slot 1 Start | `select.growatt_sph_battery_first_slot_1_start` |
| Charge Slot 1 End | `select.growatt_sph_battery_first_slot_1_end` |
| Charge Slot Enable | `switch.growatt_sph_battery_first_slot_1_enabled` |

## The apps.yaml Configuration

Here's the complete Predbat configuration for this setup:

```yaml
pred_bat:
  module: predbat
  class: PredBat

  prefix: predbat
  timezone: Europe/London

  currency_symbols:
    - '£'
    - 'p'

  threads: auto

  # ===========================================
  # INVERTER CONFIGURATION
  # ===========================================
  inverter_type: "SA"
  num_inverters: 1

  # Battery capacity (2x GBLI 6.5kWh = 13kWh)
  soc_max:
    - 13

  # Max charge/discharge rate in Watts
  battery_rate_max:
    - 3600

  # Inverter limits
  inverter_limit: 3600
  inverter_limit_charge:
    - 3600
  inverter_limit_discharge:
    - 3600

  # ===========================================
  # ENERGY SENSORS (Daily totals)
  # ===========================================
  load_today:
    - sensor.growatt_sph_load_energy
  import_today:
    - sensor.growatt_sph_grid_energy_in
  export_today:
    - sensor.growatt_sph_grid_energy_out
  pv_today:
    - sensor.growatt_sph_pv_energy

  # ===========================================
  # POWER SENSORS (Real-time)
  # ===========================================
  battery_power:
    - sensor.growatt_sph_battery_power
  battery_voltage:
    - sensor.growatt_sph_battery_voltage
  battery_temperature:
    - sensor.growatt_sph_battery_temperature
  soc_percent:
    - sensor.growatt_sph_battery_state_of_charge
  pv_power:
    - sensor.growatt_sph_pv_power
  load_power:
    - sensor.growatt_sph_load_power
  grid_power:
    - sensor.growatt_sph_grid_power

  # ===========================================
  # GRID POWER DIRECTION
  # ===========================================
  # IMPORTANT: Check which way your sensor reports!
  # Predbat expects: negative = importing, positive = exporting
  # My Growatt reports positive when importing, so invert it
  grid_power_invert:
    - True

  # ===========================================
  # CHARGE/DISCHARGE RATE CONTROL
  # ===========================================
  charge_rate_percent:
    - number.growatt_sph_battery_first_charge_rate
  discharge_rate_percent:
    - number.growatt_sph_grid_first_discharge_rate

  # ===========================================
  # SOC LIMITS
  # ===========================================
  reserve:
    - number.growatt_sph_load_first_stop_discharge
  battery_min_soc:
    - number.growatt_sph_load_first_stop_discharge
  charge_limit:
    - number.growatt_sph_battery_first_stop_charge
  discharge_target_soc:
    - number.growatt_sph_grid_first_stop_discharge

  # ===========================================
  # TIME SLOT CONTROL
  # ===========================================
  charge_start_time:
    - select.growatt_sph_battery_first_slot_1_start
  charge_end_time:
    - select.growatt_sph_battery_first_slot_1_end
  scheduled_charge_enable:
    - switch.growatt_sph_battery_first_slot_1_enabled

  # ===========================================
  # SERVICE CALLS FOR MODE SWITCHING
  # ===========================================
  charge_start_service:
    service: select.select_option
    entity_id: select.growatt_sph_work_mode_priority
    option: "Battery first"

  charge_stop_service:
    service: select.select_option
    entity_id: select.growatt_sph_work_mode_priority
    option: "Load first"

  discharge_start_service:
    service: select.select_option
    entity_id: select.growatt_sph_work_mode_priority
    option: "Grid first"

  discharge_stop_service:
    service: select.select_option
    entity_id: select.growatt_sph_work_mode_priority
    option: "Load first"

  # ===========================================
  # INVERTER TIME
  # ===========================================
  inverter_time:
    - sensor.date_time

  # ===========================================
  # HISTORICAL DATA
  # ===========================================
  days_previous:
    - 7
    - 14
    - 21

  days_previous_weight:
    - 1.0
    - 0.7
    - 0.5

  # ===========================================
  # OCTOPUS AGILE
  # ===========================================
  metric_octopus_import: "re:sensor.octopus_energy_electricity_.*_current_rate"
  metric_octopus_export: "re:sensor.octopus_energy_electricity_.*_export.*_current_rate"

  octopus_intelligent_slot: "off"

  # Forecast 48 hours for Agile (prices change frequently)
  forecast_plan_hours: 48

  # ===========================================
  # SOLCAST
  # ===========================================
  pv_forecast_today: "re:sensor.solcast_pv_forecast_forecast_today"
  pv_forecast_tomorrow: "re:sensor.solcast_pv_forecast_forecast_tomorrow"
  pv_forecast_d3: "re:sensor.solcast_pv_forecast_forecast_day_3"
  pv_forecast_d4: "re:sensor.solcast_pv_forecast_forecast_day_4"

  # ===========================================
  # NO EV (covered in separate post)
  # ===========================================
  num_cars: 0
```

## Key Configuration Decisions

### Grid Power Inversion

This caught me out initially. Predbat expects grid power to be negative when importing and positive when exporting. My Growatt via Solar Assistant reports it the opposite way (positive = importing), so `grid_power_invert: True` is essential. Check your own sensor in Developer Tools → States while the grid is importing to see which way yours reports.

### Forecast Hours

I've set `forecast_plan_hours: 48` because Agile prices are published day-ahead. With 48 hours of planning horizon, Predbat can optimise across two days of known prices.

### Historical Weighting

The `days_previous` setting tells Predbat to look at your consumption from 7, 14, and 21 days ago to predict today's usage. The weights (1.0, 0.7, 0.5) give more importance to recent data. Make sure you have as many weights as you have days.

### Time Slot Control vs Service Calls

The Growatt SPH supports both approaches. Time slot control (`charge_start_time`, `charge_end_time`, `scheduled_charge_enable`) lets Predbat set specific charging windows. Service calls (`charge_start_service`, etc.) switch the inverter's work mode. Using both gives Predbat maximum flexibility.

## Enabling the Date/Time Sensor

If `inverter_time` shows red in the Predbat web interface, you need to enable the time_date sensor. Add this to your `configuration.yaml`:

```yaml
sensor:
  - platform: time_date
    display_options:
      - 'date_time'
```

Then restart Home Assistant.

## Home Assistant Settings (Predbat Dashboard)

Beyond the `apps.yaml` file, Predbat exposes many settings as Home Assistant entities that you configure via the UI or the Predbat dashboard. Here are the key ones:

### Core Settings

| Setting | Value | Reason |
|---------|-------|--------|
| `select.predbat_mode` | Control charge | Discharge disabled - only optimising charging |
| `switch.predbat_set_reserve_enable` | ON | Allows Predbat to set the reserve/minimum SoC |
| `input_number.predbat_set_reserve_min` | 20 | Minimum battery SoC to maintain |
| `switch.predbat_set_charge_freeze` | ON | Holds SoC during cheap periods if already charged |
| `input_number.predbat_best_soc_keep` | 0.5 | Buffer in kWh to keep above target |

### Agile-Specific Settings

| Setting | Value | Reason |
|---------|-------|--------|
| `input_number.predbat_forecast_plan_hours` | 48 | Two days of planning for day-ahead Agile prices |
| `switch.predbat_combine_charge_slots` | ON | Combines consecutive cheap slots into one charge window |
| `switch.predbat_combine_export_slots` | ON | Same for export (when enabled) |
| `switch.predbat_calculate_inday_adjustment` | ON | Adjusts plan as Agile prices update |

### Load Prediction

| Setting | Value | Reason |
|---------|-------|--------|
| `switch.predbat_load_filter_modal` | ON | Uses modal filtering for load prediction |
| `input_number.predbat_metric_self_sufficiency` | 1.0 | Adds 1p/kWh virtual value to self-consumed energy |

### Optional but Recommended

| Setting | Value | Reason |
|---------|-------|--------|
| `switch.predbat_expert_mode` | ON | Exposes additional settings |
| `switch.predbat_inverter_hybrid` | ON | Correct for Growatt SPH |
| `switch.predbat_metric_pv_calibration_enable` | ON | Improves solar predictions over time |
| `switch.predbat_set_export_freeze` | ON | Holds SoC during high export rate periods |

### Understanding `switch.predbat_active`

This confused me initially - `switch.predbat_active` is a **status indicator**, not a control. It shows ON when Predbat is actively executing a charge/discharge plan (i.e., you're within a scheduled charging window). You don't turn it on manually; it turns on automatically when Predbat is controlling the inverter.

## Verifying the Configuration

After saving your `apps.yaml`:

1. Wait a few minutes for Predbat to reload (it runs every 5 minutes)
2. Access the Predbat web interface
3. Click **apps.yaml** to see any mismatched entities (shown in red)
4. Check `predbat.status` for errors

## What I'm Not Covering Here

I've deliberately excluded EV charging configuration from this post - that's covered separately as it adds significant complexity with Tesla Fleet integration and charging automations.

I've also disabled discharge/export (`switch.predbat_set_discharge_enable` = off) for now. On Agile there are periods where exporting makes sense, but I wanted to get charging working reliably first.

## Results

With this configuration, Predbat successfully identifies the cheapest overnight slots on Agile and charges the battery accordingly. On a typical day, it might charge at 2am-4am when prices are around 7p/kWh, then the battery powers the house during the 4pm-7pm peak when prices hit 30p+/kWh.

The Predbat dashboard shows the planned charging windows and predicted costs, making it easy to verify it's working as expected.