# Schartec Move 1000 — ESP32 Autopilot Garage

Turn a Schartec Move 1000 (Series 3) into a garage door that opens when you
walk up to it from inside, holds while you drive or cycle out, and closes once
you're clear — without touching a remote.

The goal is simple: **remove the friction so the garage actually gets used.**

```
morning, leaving:
  presence detected inside, within 2 m, dwelling > 1.5 s
    → pulse O/S/C ......................... door opens, holds
    → doorway beam breaks ................. car or bike passing through
    → beam clears
    → wait 8 s, re-confirm clear + bay empty
    → pre-warn 3 s, pulse O/S/C ........... door closes
```

All of that logic runs **on the ESP32**, not in Home Assistant. The door keeps
working when Wi-Fi or HA is down.

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
esphome/
  garage-schartec.yaml     firmware — sensors, relay, all close logic
  secrets.yaml.example     copy to secrets.yaml (gitignored)
homeassistant/
  automations.yaml         arrival, safety nets, night guard
docs/
  hardware.md              BOM, terminal map, pinout, wiring
  safety.md                EN 12453, failure modes — read before enabling
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
| `binary_sensor.garage_closed_limit` | Binary | Reed |
| `binary_sensor.garage_open_limit` | Binary | Reed |
| `binary_sensor.garage_doorway_beam_blocked` | Binary | Independent beam |
| `binary_sensor.garage_inner_approach` | Binary | LD2410, < 2 m, 1.5 s dwell |
| `binary_sensor.garage_occupancy` | Binary | LD2410 presence |

## Tuning

Everything worth adjusting is in `substitutions:` at the top of the YAML:

| Key | Default | Raise it if… |
|---|---|---|
| `pulse_length` | `400ms` | The opener misses presses |
| `clear_dwell` | `8s` | A long car's tail is still under the door |
| `passage_timeout` | `5min` | You linger before pulling out |
| `cycle_cooldown_ms` | `60000` | It reopens as you walk back past |

The 200 cm approach threshold is in the `approach_zone` lambda — set it to
comfortably less than the distance from the sensor to where you park.

## Safety

**Read [docs/safety.md](docs/safety.md) before enabling auto-close.**

The short version: a door that closes with nobody watching is a different
category of machine under EN 12453 than one you drive by remote while looking
at it. The photocell is mandatory, not optional. Every failure path in this
firmware leaves the door **open and stationary** rather than moving.

## Status

Design complete, not yet built. Hardware on order.
