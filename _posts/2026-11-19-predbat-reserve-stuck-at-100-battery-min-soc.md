---
layout: post
title: "Predbat Battery Stuck at 100% and Not Discharging: The battery_min_soc Trap"
pillar: energy
excerpt: My home battery charged to full overnight and then refused to power the house all day. I had blamed one Predbat feature for months. The real cause was two settings in apps.yaml pointing at the same inverter entity.
tags:
  - predbat
  - growatt
  - solar-assistant
  - home-assistant
  - octopus-energy
---

The car charged overnight, and in the morning the house battery was stuck. Not low. Stuck. Sitting at 99%, not charging, not discharging, while the house bought every kettle and every light from the grid through a dark November morning.

The setup is the one from my [Predbat and Solar Assistant post]({% post_url 2026-01-22-predbat-solar-assistant-growatt-octopus-agile %}): a Growatt SPH inverter read through Solar Assistant over MQTT, Predbat doing the optimising, and a Tesla charging overnight on [Octopus Intelligent Go]({% post_url 2026-02-05-switching-octopus-agile-to-intelligent-go-predbat-tesla %}). It normally runs itself. This time it had decided the battery's floor was 100%.

## What it looked like

Everything agreed. The reserve, which is the state of charge Predbat will not discharge below, had jumped to 100% at 01:20 and stayed there. So had a Predbat setting that should never move on its own.

| Reading | Value |
|---|---|
| Reserve | 100% |
| `set_reserve_min` | 100% |
| Battery power | 0 W |
| State of charge | 99% |

A battery at 99% with a floor of 100% cannot discharge a single watt. It had been frozen for six and a half hours by the time I looked.

There was one more detail that made it worse. The slider for `set_reserve_min` had its own minimum raised to 100 as well, so it could not even be dragged back down by hand.

## The feature I blamed

The obvious suspect was a switch called `set_reserve_enable`. It lets Predbat hold the battery by raising the reserve, which is how it stops the house battery emptying itself into the car during an overnight charge.

I had history with it. I used to run with it on, noticed the battery sticking at 100% and refusing to discharge, traced it to the switch, and turned it off. That cured the sticking, and I thought no more about it.

Months later the opposite happened. The car was lower than it had been in a long time, so it needed to charge for longer than the house battery did. The battery finished, the inverter dropped back to load first, and with the switch off there was nothing holding it. It emptied from 100% to 36% straight into the car. That is exactly what the switch exists to prevent, and it had not shown up before only because the car had never needed that long.

So I turned the switch back on. The next time the car charged, the battery was stuck at 100% again.

That seemed to settle it: on, it sticks, and off, it drains into the car. Except this time the switch had done the part it was supposed to do. The battery did not drain. The hold worked. What failed was the release afterwards.

And Predbat did try to release. Its status read `Demand`, meaning discharge to cover the house, on every run from 07:00 when the car slot ended until I intervened an hour later. It wanted the battery to work. A floor at 100% would not let it, and it was a floor Predbat had set itself.

## Two settings, one entity

The cause was in `apps.yaml`, and it was almost certainly my own mistake from when I first set the system up:

```yaml
reserve:
  - number.growatt_sph_load_first_stop_discharge
battery_min_soc:
  - number.growatt_sph_load_first_stop_discharge
```

The same inverter entity, twice. The second one should have been this:

```yaml
battery_min_soc:
  - number.growatt_sph_grid_first_stop_discharge
```

Put the two names side by side and the only difference is one word in the middle, `load` against `grid`. I had looked at that file many times while chasing this and read straight past it every time, because both lines look right and both are plausibly about a minimum state of charge.

`reserve` is a value Predbat **writes**. During a hold it writes the current state of charge there, which is how the hold works.

`battery_min_soc` is a value Predbat **reads**. It is meant to be the lowest state of charge the battery can go to, and on every run Predbat raises `set_reserve_min` to match it. Raises only. It never lowers it.

Point both at the same entity and you get a loop that can only climb:

```
hold writes the reserve entity = current SoC (100)
  -> battery_min_soc reads the same entity = 100
  -> set_reserve_min is raised to 100, and never lowered
  -> reserve = the higher of the plan and 100 = 100, for good
```

Predbat was reading back its own temporary hold and treating it as the battery's permanent minimum.

## Why it only happened sometimes

A config mistake that broke things every night would have been found in a week. This one worked most of the time, and that is what made it worse. The loop only fires when a hold drives the reserve high, which in practice means a car charge that fills the house battery before dawn. Any night the battery stayed below a full hold, nothing ratcheted, and everything behaved.

So with the switch on it failed on some car-charging nights and not others, and I blamed the switch, because turning it off made the problem go away. It did go away, but only because with no hold there was nothing to ratchet, and the price was a battery that could drain into the car instead. The switch was doing its job. The config was turning its output into a permanent floor.

## The night, in order

- **23:09** The car starts charging on Intelligent Go.
- **23:09 to 01:19** Predbat charges the house battery from the grid, 60% to 100%.
- **01:20** `set_reserve_min` goes from 14% to 100%. The reserve is pinned and battery power drops to 0 W.
- **07:00** The car slot ends. Predbat's status changes to `Demand`.
- **07:00 to 08:05** Status stays `Demand` on every run. The floor stays at 100%. Nothing moves.
- **About 08:05** I turn the hold off and set the reserve back to 12% by hand. The battery starts discharging within seconds.

## The fix

Point `battery_min_soc` at the entity it should have had all along, which is what the current Solar Assistant template for the Growatt SPH uses:

```yaml
battery_min_soc:
  - number.growatt_sph_grid_first_stop_discharge
```

The template now carries a warning next to that line never to use the reserve entity there. The rule behind it is simple: `battery_min_soc` must not name anything Predbat writes during a hold. If you would rather not depend on an entity at all, a fixed number such as `10` does the same job and cannot be ratcheted.

Changing the config is not enough on its own, because the ratchet only goes up. After restarting Predbat:

1. Check the `set_reserve_min` slider's minimum has dropped back. Mine reset from 100 to 14 on the restart.
2. Set `set_reserve_min` itself back down by hand. Mine went from 100 to 14.
3. Turn `set_reserve_enable` back on if you had given up on it, as I had. With the loop gone it is safe.

## It is already fixed upstream

This turned out to be a known problem. A Predbat [issue describing the same thing](https://github.com/springfall2008/batpred/issues/4013), a reserve that climbed to 97% over one overnight charge, was closed the day before mine happened. Predbat v9.7.11 adds a guard: if `battery_min_soc` names a reserve entity it is ignored, the slider minimum is reset, and a warning is shown.

I was on v9.7.6, so I had no guard, and on any older version the only protection is the config itself. Upgrading is worth doing either way. So is fixing the config, because a guard that ignores a setting is not the same as the setting being right.

## In short

If a Predbat battery sits at 100% and will not discharge, and `set_reserve_min` has crept up to match, check whether `reserve` and `battery_min_soc` in `apps.yaml` point at the same entity.

One is written by Predbat and the other is read back, and when they share an entity every hold ratchets the minimum up for good. The two names differ by a single word, `load` against `grid`, so read them character by character. Point `battery_min_soc` at the grid-first entity or give it a fixed value, restart, put `set_reserve_min` back down by hand, and upgrade to v9.7.11 or later for the guard.

The wider lesson is the one I should have applied sooner: when something fails only some of the time, the feature that is switched on when it fails is not necessarily the cause. Mine was innocent for months.
