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

- **Going out:** every trigger must come from **inside the house** — there is
  no internal door to the garage, so the door has to be open before you walk
  round to it. Mesh floor-change (~40–60 s), **hall** presence (~30 s), and the
  exterior house door (~10 s, late but cheap)
- **Coming home:** a deliberately *small* **~100–150 m geofence** — with a
  parking space out front, waiting costs nothing, so a tight radius wins
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
  measurements.md          ← what to measure on site, before ordering
  deployment.md            where each box lives — Pi indoors, ESP32 in the garage
  experience-design.md     choreography, timings, what breaks "butter"
  ios-setup.md             Companion app, HomeKit, Shortcuts automations
  presence-network.md      using the ASUS BD8 mesh for floor-level presence
  hardware.md              BOM, terminal map, pinout, wiring
  safety.md                EN 12453, failure modes — read before enabling
  working-diary.md         session log
```

## Next step: measure

Every number in this repo is an estimate. Two of them decide real money and a
real design, and both need a tape measure and a stopwatch:

1. **Door height** → which rail to order. Getting this wrong means dismantling
   the installation later.
2. **Hall → garage walk time** → how many trigger layers you actually need.

Checklist to take to the garage: **[docs/measurements.md](docs/measurements.md)**

## Validate before you buy anything

The cheapest place to find mistakes is before the parcels arrive. This needs no
hardware at all:

```bash
python3 -m venv .venv && .venv/bin/pip install esphome
```

```bash
cp esphome/secrets.yaml.example esphome/secrets.yaml
```

Fill it in — generate the API key with:

```bash
openssl rand -base64 32
```

Then check the firmware:

```bash
.venv/bin/esphome config esphome/garage-schartec.yaml
```

> ✅ Validated against **ESPHome 2026.8.2** — `Configuration is valid!`, no
> warnings, no strapping-pin complaints. Re-run it after any edit; it catches
> schema mistakes in seconds.

A full compile also passes, which additionally type-checks the C++ lambdas:

```bash
.venv/bin/esphome compile esphome/garage-schartec.yaml
```

> ✅ **Compiles clean.** `RAM 56.9%` (70,916 / 124,580 B) ·
> `Flash 70.9%` (1,300,347 / 1,835,008 B).
>
> The RAM figure is the one to watch: enabling `esp32_ble_tracker` pulls in the
> whole Bluedroid stack alongside Wi-Fi. It **fits**, with ~53 KB spare — but
> that is static allocation, not runtime heap under Wi-Fi/BLE coexistence. If
> the node turns out to be unstable on real hardware, dropping the BLE section
> is the first thing to try; the bike tag is the only feature that needs it.

## Flashing

```bash
.venv/bin/esphome run esphome/garage-schartec.yaml
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

## Where things live

The door logic runs **entirely on the ESP32**, which is rated −40 °C … +85 °C
and belongs in the garage. Home Assistant runs on a **Raspberry Pi indoors** —
a Pi is only rated to 0 °C and an unheated Swedish garage goes below that.

The upshot worth knowing: **pull the Pi's power and the garage still works.**
HA only adds arrival triggers and notifications. See
[docs/deployment.md](docs/deployment.md).

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

**Design complete and validated in software; nothing built, nothing bought.**

- Firmware passes `esphome config` **and** a full compile (ESPHome 2026.8.2)
- Every physical dimension and timing is still an **estimate** — see
  [docs/measurements.md](docs/measurements.md)
- Next action: measure the door height and time the walk
