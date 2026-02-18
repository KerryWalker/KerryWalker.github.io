---
layout: post
published: false
title: "Family Dashboard Part 3: Simplifying the Energy Gauge"
excerpt: "The concentric horseshoe looked great but was too cluttered. Replacing it with a clean 2×2 quad gauge layout inspired by Solar Assistant."
tags:
  - home-assistant
  - solar
  - growatt
---

The concentric horseshoe gauge from [Part 2](/2026/02/25/family-dashboard-part-2-energy-gauge.html) looked impressive but it was too much. Five rings crammed together with labels fighting for space — not ideal for a family dashboard where you want a quick glance, not a puzzle.

I use [Solar Assistant](https://solar-assistant.io/) to monitor my Growatt SPH and its dashboard has a simple 2×2 grid of gauges: Load, Solar, Grid, Battery. Each one is its own horseshoe with a single value in the centre. Clean and immediately readable.

So I rebuilt the card to match that layout.

## The New Design

Four horseshoe gauges in a 2×2 grid, all on one card using `custom:html-template-card`:

```
┌────────────┬────────────┐
│   410 W    │   560 W    │
│    Load    │   Solar    │
├────────────┼────────────┤
│    0 W     │   140 W    │
│    Grid    │  Battery   │
└────────────┴────────────┘
```

Each gauge is independent with its own arc, value and label. The dynamic behaviour from Part 2 is still there:

- **Solar** — orange, always fills left to right
- **Load** — blue, always fills left to right
- **Grid** — red filling left→right when importing, teal filling right→left when exporting. Shows "importing" or "exporting" underneath
- **Battery** — green filling left→right when charging, orange filling right→left when discharging. Shows battery percentage in pink underneath

The direction flip is what makes it work. You don't need to read the numbers to know what's happening — the arc direction and colour tell you instantly.

## Why Not Native Gauge Cards?

The built-in HA gauge card only supports one entity per card. You could put four gauge cards in a `grid` card, but you lose the dynamic colour and direction changes for grid and battery. The HTML template approach gives full control over all of it in a single card.

## The Card

Same approach as before — Jinja2 macros generating SVG paths, but now with a reusable `gauge` macro that takes position, radius, colour, value and label. Much cleaner code than the concentric version.

Full YAML: [`cards/energy-quad-gauge.yaml`](https://github.com/KerryWalker/Growatt-Solar-Assistant-Home-Assistant-Dash/blob/main/cards/energy-quad-gauge.yaml)

## Adapting It

Same as Part 2 — change the five entity IDs and max values at the top of the file. The sign conventions are: battery power positive = charging, grid power positive = importing.

## Next

Both gauge versions are in the [GitHub repo](https://github.com/KerryWalker/Growatt-Solar-Assistant-Home-Assistant-Dash) if you want to try either. The concentric version is still there as `cards/energy-horseshoe.yaml` if you prefer that look.

[Part 1: iCloud Calendars →](/2026/02/22/family-dashboard-part-1-calendars.html) · [Part 2: Concentric Energy Gauge →](/2026/02/25/family-dashboard-part-2-energy-gauge.html)
