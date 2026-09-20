---
layout: post
title: A Samotech SM323, two wires, and a setting Hubitat doesn't have
excerpt: Replacing a dead dimmer on a UK 2-way circuit with only two cores between the switches, and the manufacturer-specific Zigbee attribute you need to write because no Hubitat driver exposes it.
tags:
  - hubitat
  - zigbee
  - homeassistant
  - lighting
---

The dimmer on our upstairs landing gave up. The dimming still worked, the push on and off didn't, so it needed replacing. I had a Samotech SM323 Zigbee rotary dimmer in a drawer from an earlier project, and the plan looked like a job for a wet afternoon: swap the dead dimmer for the SM323, leave the downstairs switch alone, pair it to Hubitat, done.

It took rather longer than that. Two things got in the way, and both are worth writing down, because nearly every answer I found online was wrong for this particular circuit.

**Before anything else.** I'm a time-served electrical engineer, so I'm competent to do this work and to judge when a circuit is safe to change. If you aren't, get a qualified electrician in — this is mains voltage, the wiring here is non-standard, and half the advice online about it is wrong. Whoever does it: isolate the lighting circuit at the consumer unit and prove dead at both boxes before touching a wire.

## What's actually in the wall

- A UK upstairs/downstairs 2-way circuit, with **no neutral** at either switch.
- Downstairs, a **2-gang** switch — one gang for the stairs, one for the hall. Not being replaced, not being disabled.
- Upstairs, the dimmer position, with two cables. One from above is the switched live to the lamp: a red core someone had extended with a bit of blue in a connector, so it's marked as the return. One from below runs to the downstairs switch.
- The link cable between the two switches is **2-core**, red and black. Not 3-core.

That last point is the one everything hinges on. It's an ordinary 2-way done the conversion way: permanent live into COM downstairs, switched live to the lamp at COM upstairs, two strappers between L1 and L2 at each end. Testing agreed — at the upstairs box the strapper read 240 V to earth, COM read 0 V.

## Why the obvious wiring doesn't work

A smart dimmer needs *permanent* power at L. With a 2-core link wired as a normal 2-way, which strapper is live depends on where the downstairs switch happens to be sitting. Flip it and the module loses power and drops off the network. Non-starter.

Three suggestions I was given, and why each fails:

- **Join both strappers into L and set the module to `three_way` mode.** If both cores land on L, the module has power whichever way the downstairs switch is thrown — so it can never tell that anything moved. The light just stays on.
- **Pull a third core through.** Not without taking the stairs wall apart.
- **Replace the downstairs switch with a wireless Zigbee button, or bypass it.** It's a 2-gang. The other gang has a job to do.

## The wiring that does work

Stop using the two cores as strappers. Make one a **permanent live** up to the module, and the other a **switch signal** into the module's S terminal. The downstairs gang stays exactly where it is and behaves like a normal toggle — it just switches a signal now, rather than the mains feed to the lamp.

**Downstairs, stairs gang only:**

- Permanent live stays in **COM**. Add the **red** link core into COM alongside it, or Wago them together with a jumper into COM. Red is now live all the time.
- **Black** link core into **L1**. L2 empty.

**Upstairs, the SM323:**

- Red → **L**
- A wire link between **L** and **N** (needed for no-neutral operation)
- Red/blue lamp wire → **L-OUT**, the lamp symbol
- Black → **S**

Every flip of the downstairs gang opens or closes live onto S, and the SM323 reads that as a toggle. The knob upstairs still dims, and Hubitat and Home Assistant keep full control.

There's a catch, and it turned into the second half of the afternoon: the SM323 ships expecting a momentary retractive switch. Feed a normal rocker into S without changing that, and it misbehaves. So the switch type has to be set to **normal on/off**.

## Pairing, before we get to that

Two things that cost me a while:

- Pressing the **knob** five times does not enter pairing. It switches the light on and off five times and convinces you the module is broken. Pull the knob off the spindle and press the small **RESET** button five times instead — the lamp flashes twice and it joins.
- If you've been hammering the breaker trying to reset it, it can sit there flashing at you. Leave the power off for a minute, then try again.

## The setting that isn't there

Paired, working, and nowhere to change the switch type. I went through the lot:

- **Generic Zigbee Dimmer** — on, off, level. Nothing else.
- **BirdsLikeWires Samotech SM323 Dimmer Module** (v1.10) — a good driver, but its preferences are electrical measurement, max level and logging. No switch type.
- A "Basic Z-Wave/Zigbee Tool", a Maker API `setDriver` URL, a config entity in Home Assistant. The Z-Wave tool is Z-Wave. Maker API doesn't expose preferences. The Hubitat to Home Assistant integration relays commands and attributes, not settings. All dead ends.

I wasn't missing a drop-down. There genuinely isn't one.

The reason is that the SM323 v2 is a **Sunricher**-based module, and the switch type lives in a **manufacturer-specific attribute** rather than a standard Zigbee cluster. From the Zigbee2MQTT converter:

- Cluster `0x0000` (Basic), attribute `0x8803`, type `uint8` (`0x20`)
- Manufacturer code `0x1224`
- Values: `0` = push_button, `1` = normal_on_off, `2` = three_way

Zigbee2MQTT exposes that as `external_switch_type` and you get it for free. Hubitat has no idea it exists. (Worth noting the converter only attaches this to the **v2** — the original SM323 doesn't have it at all.)

So I wrote a throwaway driver to write the attribute directly.

## The driver

Hubitat → **Drivers Code** → **New Driver**, paste, Save:

```groovy
metadata {
    definition(name: "Samotech SM323 Switch Type Tool", namespace: "kerry", author: "Kerry") {
        capability "Actuator"
        command "setSwitchType", [[name: "type*", type: "ENUM", constraints: ["push_button", "normal_on_off", "three_way"]]]
        command "readSwitchType"
    }
}

def configure() {}

def setSwitchType(String type) {
    def lookup = ["push_button": 0, "normal_on_off": 1, "three_way": 2]
    int val = lookup[type]
    log.info "SM323: writing external switch type ${type} (${val})"
    return zigbee.writeAttribute(0x0000, 0x8803, 0x20, val, [mfgCode: "0x1224"]) +
           zigbee.readAttribute(0x0000, 0x8803, [mfgCode: "0x1224"])
}

def readSwitchType() {
    return zigbee.readAttribute(0x0000, 0x8803, [mfgCode: "0x1224"])
}

def parse(String description) {
    def map = zigbee.parseDescriptionAsMap(description)
    if (map.cluster == "0000" && map.attrId == "8803") {
        def names = ["00": "push_button", "01": "normal_on_off", "02": "three_way"]
        log.info "SM323: external switch type is ${names[map.value] ?: map.value}"
    } else {
        log.debug "SM323: ${map}"
    }
}
```

Then:

1. On the SM323's device page, change **Type** to `Samotech SM323 Switch Type Tool` and Save Device.
2. Open Logs and click **readSwitchType**. Mine came back `push_button`, which is the factory default and proof the manufacturer-specific read is getting through.
3. Run **setSwitchType** with `normal_on_off`. The log shows the write, a write response with status `00`, and a read-back of `normal_on_off`.
4. **Power cycle the circuit** at the consumer unit for about ten seconds. Samotech are clear that the mode only takes effect after a restart, and they're right. You'll see the device announce and the on/level reports as it rejoins; a fresh read still said `normal_on_off`.
5. Change **Type** back to the BirdsLikeWires driver and Save Device. The setting lives in the module, not in the driver.

If you leave out the empty `configure()` you'll get a harmless `MissingMethodException` when Hubitat runs configure automatically after the driver swap. It's noise.

## Where that left it

The downstairs gang toggles the light from either position. The knob upstairs dims and toggles. Hubitat reports physical and digital changes correctly, and Home Assistant gets the lot through Maker API. New hardware: one short wire link. Wiring changed downstairs: one core moved into COM.

## Then I found a neutral

That should have been the end of it, except for something I'd been ignoring for years.

The landing lamp is halogen, thirty-odd watts, and that turns out to be why the no-neutral setup has been so well behaved. A no-neutral dimmer has to keep itself alive by leaking a trickle of current through the lamp. Halogen just soaks that up as a bit of extra warmth and says nothing.

An LED driver does not. It sees that trickle as power and does one of the classic things: glows faintly when it's supposed to be off, flickers at low brightness, pops on and off, or refuses to start at all if the total load is under the module's minimum — usually somewhere around 5 to 10 watts. A single 5 W lamp sits right on that line.

Which explains something that has been following me round the house. I've got two more of these dimmers in other rooms, both no-neutral, and both hopeless with LEDs. I'd assumed I kept buying bad bulbs. I wasn't — it was the wiring.

Upstairs I had a spare core going begging, so the fix was straightforward: bring a neutral down, drop the L to N link, put the spare core into N, and sleeve it blue. With a real neutral the module powers itself properly from L and N, and the lamp output becomes an ordinary trailing-edge dimmed feed. At that point whether an LED dims nicely is a question about the bulb, not about the circuit.

Where there's no spare core, the fallbacks in order:

1. **A bypass at the fitting.** Samotech sell one — an RC network wired across live and neutral in the rose or the fitting. It gives the module a path to feed itself that doesn't run through the LED driver, and it usually cures the glow and the flicker outright. One per circuit, not one per lamp.
2. **More load.** Several downlights together, say 15 to 20 watts of dimmable LED, often clear the minimum between them. One 5 W lamp never will.
3. **Choose the lamp carefully.** Anything sold as trailing-edge or smart-dimmer compatible copes better with the leakage. On its own it's the least reliable fix, but it helps alongside the others.

Neutral where the wiring allows it, bypass where it doesn't. Between the two, the LED problem should stop being a mystery.

## What I'd tell someone starting this

With a 2-core link and no neutral, there's exactly one layout that works: constant live on one core, switch signal on the other into S. The alternating-loop idea sounds reasonable and can't work, because the module can't see a change it's being powered by either way.

Set the SM323 to `normal_on_off` if the far switch is a rocker. Hubitat has no interface for that, and the `0x8803` write with manufacturer code `0x1224` above is the whole trick.

And the knob doesn't pair it. The reset button does.

## References

- [Zigbee2MQTT Samotech definitions](https://github.com/Koenkk/zigbee-herdsman-converters/blob/master/src/devices/samotech.ts) — SM323_v2 pulls in the Sunricher external switch type, v1 doesn't
- [Zigbee2MQTT Sunricher library](https://github.com/Koenkk/zigbee-herdsman-converters/blob/master/src/lib/sunricher.ts) — attribute `0x8803`, manufacturer code `0x1224`, value map
- [BirdsLikeWires Hubitat drivers](https://github.com/birdslikewires/hubitat)
- [Samotech SM323 product page](https://www.samotech.co.uk/products/zigbee-dimmer-switch/) — including the power cycle requirement after a mode change
