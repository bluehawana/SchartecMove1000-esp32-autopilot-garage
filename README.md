# Schartec Move 1000 — ESP32 Autopilot Garage

Turn a Schartec Move 1000 (Series 3) into a garage door that opens when you
walk up to it from inside, holds while you drive or cycle out, and closes once
you're clear — without touching a remote.

The goal is simple: **remove the friction so the garage actually gets used.**

```
leaving:                              arriving:
  you head for the garage               you turn in, door already open
    → door opens, holds                   → you drive in
    → beam breaks, then clears            → beam breaks, then clears
    → garage is empty (you left)          → you are STILL IN THE GARAGE
    → closes                              → door waits...
                                          → you walk into the house
                                          → closes
```

One rule produces both: **close when the doorway is clear and no person is in
the garage.** No direction detection, no second beam — and it is the safe rule
as well as the convenient one.

All of that logic runs **on the ESP32**, not in Home Assistant. The door keeps
working when Wi-Fi or HA is down.

## Where this came from

We started by watching the [Schartec installation video](https://www.schartec.de)
— the one the QR code in the manual points to. Rail, brackets, motor head, set
the limits, pair the remote, done. It's a good video.

It ends with someone standing in their garage, pressing a button.

**That's exactly where our problem starts.** Following it perfectly gets you a
motor on a door. It doesn't get you a garage you'll actually use — because a
remote still costs you a small ritual every time: find it, press it, wait,
drive through, wonder later whether it closed. None of that is hard. It's just
enough friction that the garage becomes a storage room and the car lives on the
driveway.

So the functionality we actually need isn't remote control. It's **no control**.

See [TL;DR.md](TLDR.md) for the short version, or
[docs/experience-design.md](docs/experience-design.md) for the full
choreography and timings.

## The number that shapes the whole design

The Move 1000 travels at **160 mm/s**. A 2400 mm door takes **~15 seconds**.

That rules out the obvious approach. Trigger the door when you *want* to drive
through and you sit staring at it for fifteen seconds — worse than a remote,
not better. So every trigger fires **before** the moment of need:

- **Going out:** a cascade — phone **roams to the ground-floor mesh node**
  (~40–60 s), **hallway** presence (~30 s), the internal **house→garage door**
  contact (~20 s), and the phone **pairing to the car's Bluetooth** (~15 s,
  and certain — it can't fire when you're not in the car)
- **Coming home:** a **~250 m geofence**, ~30 s out at residential speed
- **Closing:** a beam that **breaks then clears** — the exact signal that a car
  or bike has passed through, and it works the same for both

Opening early is only affordable if being wrong is cheap, so the firmware
tracks *why* the door opened. A **speculative** open (house door) that nobody
uses **closes itself after 90 s**. A **commanded** open (remote/app) is left
alone — a person asked for that, so we don't second-guess it.

## Why this is needed at all

The Move 1000 is a good base — it exposes an **O/S/C dry-contact terminal**,
which is the universal way into any opener. But out of the box it has:

- ❌ No auto-close timer (no such setting exists in the Series 3 menu)
- ❌ No Wi-Fi, no app, no cloud
- ❌ No door position feedback
- ❌ Rolling code at 433.92 MHz — remotes **cannot** be cloned or replayed

So 100% of the autopilot is bolted on. The opener is a motor with a button
input. That's all we need it to be.

## Repo layout

```
TLDR.md                    the initiative: spark, gap, and how we get there
esphome/
  garage-schartec.yaml     firmware — sensors, relay, all close logic
  secrets.yaml.example     copy to secrets.yaml (gitignored)
homeassistant/
  automations.yaml         arrival, safety nets, night guard
docs/
  experience-design.md     choreography, timings, what breaks "butter"
  ios-setup.md             Companion app, HomeKit, Shortcuts automations
  presence-network.md      using the ASUS BD8 mesh for floor-level presence
  hardware.md              BOM, terminal map, pinout, wiring
  safety.md                EN 12453, failure modes — read before enabling
  working-diary.md         session log
```

## Quick start

```bash
cp esphome/secrets.yaml.example esphome/secrets.yaml
```

Fill it in — generate the API key with:

```bash
openssl rand -base64 32
```

Then flash:

```bash
esphome run esphome/garage-schartec.yaml
```

Wire it up per [docs/hardware.md](docs/hardware.md), then copy
`homeassistant/automations.yaml` into your HA config and fix the entity names.

## Commissioning order

Do not skip steps 1–3. Test with the **door disconnected from the opener**
(manual release pulled) until step 4.

1. Set the opener's travel limits normally, per the Schartec manual
2. Fit the photocell, remove the factory bridge, **verify a closing door
   reverses when you break the beam by hand**
3. Flash the ESP32, confirm both reeds and the doorway beam read correctly in
   Home Assistant — with the relay disconnected
4. Reconnect the door. Test `cover.open_cover` / `close_cover` manually
5. Enable `Garage Auto-Close`. Test it a dozen times on foot
6. Only then enable `Garage Auto-Open`

## Entities exposed

| Entity | Type | Notes |
|---|---|---|
| `cover.garage_door` | Cover | Open / close / stop |
| `switch.garage_auto_open` | Switch | **Turn off when working in the garage** |
| `switch.garage_auto_close` | Switch | Kill switch for close-after-passage |
| `button.garage_speculative_open` | Button | Entry point for iOS / other-room triggers |
| `binary_sensor.garage_closed_limit` | Binary | Reed |
| `binary_sensor.garage_open_limit` | Binary | Reed |
| `binary_sensor.garage_doorway_beam_blocked` | Binary | Independent beam |
| `binary_sensor.garage_house_door` | Binary | House→garage door — the early trigger |
| `binary_sensor.garage_inner_approach` | Binary | LD2410, < 2 m, 1.5 s dwell |
| `binary_sensor.garage_occupancy` | Binary | LD2410 presence |
| `binary_sensor.garage_car_present` | Binary | Ultrasonic — is the car actually back? |
| `binary_sensor.garage_bike_present` | Binary | BLE beacon — is the bike actually back? |

## Tuning

Everything worth adjusting is in `substitutions:` at the top of the YAML:

| Key | Default | Raise it if… |
|---|---|---|
| `pulse_length` | `400ms` | The opener misses presses |
| `clear_dwell` | `8s` | A long car's tail is still under the door |
| `passage_timeout` | `5min` | You linger before pulling out |
| `speculative_timeout` | `90s` | Your walk-and-load takes longer |
| `occupancy_timeout` | `10min` | You potter in the garage a lot |
| `empty_settle` | `10s` | Raise if it ever closes while you're inside |
| `cycle_cooldown_ms` | `60000` | It reopens as you walk back past |

The 200 cm approach threshold is in the `approach_zone` lambda — set it to
comfortably less than the distance from the sensor to where you park.

## Safety

**Read [docs/safety.md](docs/safety.md) before enabling auto-close.**

The short version: a door that closes with nobody watching is a different
category of machine under EN 12453 than one you drive by remote while looking
at it. The photocell is mandatory, not optional. Every failure path in this
firmware leaves the door **open and stationary** rather than moving.

## Part of Villalyftet

This project is documented as part of **[villalyft.com](https://villalyft.com)**
— the Askim neighbourhood group-buy initiative for home energy improvements —
and follows the same rule the site sets for itself: publish the real numbers,
including the ones that didn't work out.

The energy angle is not a stretch. A garage door left standing open through a
Göteborg winter is a measurable heat loss, and "I forgot to close it" is the
most common way it happens. Close-after-passage removes that failure mode
entirely — the door is open for the ~35 seconds it takes to drive through, and
not a minute longer.

If neighbours want the same setup, the parts list is the parts list. That's
the Villalyftet model.

## Status

Design complete, not yet built. Hardware on order.
