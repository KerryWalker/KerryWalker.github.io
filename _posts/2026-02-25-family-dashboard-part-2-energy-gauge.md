---
layout: post
published: false
title: "Family Dashboard Part 2: A Custom Concentric Energy Gauge"
excerpt: "Replacing three separate horseshoe cards with a single SVG gauge showing solar, battery, grid and load — built with Jinja2 templates."
tags:
  - home-assistant
  - solar
  - growatt
  - predbat
---

My original solar dashboard had three separate `flex-horseshoe-card` gauges — solar, battery and grid. They worked but took up a lot of space. For the family dashboard I wanted a single card that tells the whole energy story at a glance.

The result is five concentric horseshoe arcs rendered as inline SVG using `custom:html-template-card` (the [Lovelace HTML Jinja2 Template card](https://github.com/PiotrMachowski/Home-Assistant-Lovelace-HTML-Jinja2-Template-card)). No extra JS cards to install — just Jinja2 generating SVG paths.

## The Design

Five rings, outside to inside:

1. **Solar** (orange) — PV generation in watts, fills left to right
2. **Battery Level** (pink) — state of charge %, fills left to right
3. **Battery Power** (green/orange) — green and fills left→right when charging, orange and fills right→left when discharging
4. **Grid** (red/teal) — red left→right when importing, teal right→left when exporting
5. **Load** (blue) — household consumption, fills left to right

The direction flip is the key bit. When the battery switches from charging to discharging, the arc fills from the opposite side and changes colour. Same with the grid. You can immediately see which way energy is flowing without reading any numbers.

Above the rings: solar watts, battery percentage and load watts. Inside the rings: battery power and grid power, colour-matched to their arcs.

## The Entities

If you're running a Growatt SPH these are the five entities that drive it:

| Ring | Entity |
|------|--------|
| Solar | `sensor.growatt_sph_pv_total` |
| Battery Level | `sensor.growatt_sph_battery_state_of_charge` |
| Battery Power | `sensor.growatt_sph_battery_power` |
| Grid | `sensor.growatt_sph_grid_power` |
| Load | `sensor.growatt_sph_load_power` |

Battery power goes negative when discharging, grid power goes negative when exporting. That sign convention drives the direction and colour logic.

## Max Values

The arc maximums are tuned to my system so the rings scale realistically:

| Ring | Max | Why |
|------|-----|-----|
| Solar | 7000W | Array peak output |
| Battery charge | 5000W | Highest I've seen from solar |
| Battery discharge | 3500W | Inverter max output |
| Grid import | 8000W | House + battery + EV charger |
| Grid export | 3500W | Inverter limited |
| Load | 8000W | Everything on at once |

## How It Works

The card uses Jinja2's `cos` and `sin` functions to calculate SVG arc paths. Each horseshoe is a 270° arc opening at the bottom. The fill fraction is calculated against the appropriate max value, and a macro generates the SVG path data for either clockwise (charging/importing) or counter-clockwise (discharging/exporting) fill direction.

The full YAML is too long to paste inline — grab it from the repo: [`cards/energy-horseshoe.yaml`](https://github.com/KerryWalker/Growatt-Solar-Assistant-Home-Assistant-Dash/blob/main/cards/energy-horseshoe.yaml)

## Adapting It

To use this with a different inverter, change the five entity IDs and the max values. If your inverter uses opposite sign conventions for battery/grid power, flip the comparisons in the `charging` and `importing` variables.

## Next

[Part 1: iCloud Calendars →](/2026/02/22/family-dashboard-part-1-calendars.html)

More to come — weather, transport, chores. The full dashboard config is in the [GitHub repo](https://github.com/KerryWalker/Growatt-Solar-Assistant-Home-Assistant-Dash).
