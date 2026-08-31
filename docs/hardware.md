# Hardware & Wiring

## Bill of materials

| Item | Purpose | Approx |
|---|---|---|
| Schartec Move 1000 Series 3 | The opener (2 remotes + K-rail included) | ~€200 |
| **Schartec photocell set** | Opener's own safety beam. **Not included.** Mandatory | ~€35 |
| Independent IR beam pair, 12–24 V | Passage detection for the automation | ~€15 |
| ESP32-WROOM devkit | The brain | ~€8 |
| Opto-isolated relay module (1ch) | Dry contact onto O/S/C | ~€3 |
| HLK-LD2410 (or LD2410C) | mmWave inner-approach sensor | ~€8 |
| 2× NC reed switch + magnet | Open / closed limits | ~€5 |
| Piezo buzzer 3–5 V | Pre-close warning | ~€2 |
| 24 V flashing warning light, ≤100 mA | Travel warning, driven by the opener | ~€15 |
| 5 V USB PSU **or** 24 V→5 V buck | Powers ESP32 + LD2410 | ~€6 |

Roughly **€90 on top of the opener**.

> Buying the opener: the standard kit ships a **K-rail (~3155 mm total), good for a
> sectional door up to 2400 mm high**. Taller doors need the M-rail (3655 mm) or
> L-rail (4155 mm) variant. Get the right one up front — swapping a rail later is
> miserable. Limits: 14 m², 140 kg balanced.

## What the Move 1000 control board gives you

From the Series 3 manual:

| Terminal | Manual ref | Notes |
|---|---|---|
| **O/S/C** | Fig. 17 | Momentary dry contact. Open / Stop / Close **toggle** |
| **Photocell** | Fig. 18 | Ships with a **bridge fitted** — remove it when adding beams |
| **Warning light** | Fig. 19 | 24–28 VDC, max 100 mA. Board flashes it during travel |
| **Wicket door** | Fig. 20 | Also bridged from factory. Leave alone if unused |
| **Accessory supply** | — | 24–28 VDC, **max 300 mA total** |

Radio: 433.92 MHz, **rolling code**, 20 remote slots.

### Two consequences that shape the whole design

1. **Rolling code means no RF shortcuts.** You cannot clone a remote, replay a
   capture, or transmit with a generic 433 MHz module. You need a physical
   contact closure. That is why this project drives O/S/C directly.
2. **O/S/C is a single toggle, not open/close commands.** One pulse from closed
   opens; from open it closes; mid-travel it stops. The board never reports
   door position, so without the reed switches the automation is flying blind
   and will eventually send the door the wrong way. The reeds are not optional.

Also worth knowing: **the Series 3 has no auto-close timer.** There is no such
setting in the menu. Every bit of "hold, then close" behaviour is ours.

## Pinout (ESP32-WROOM)

| GPIO | Connects to | Mode |
|---|---|---|
| 26 | Relay module IN | Output |
| 27 | Piezo buzzer | Output |
| 32 | Closed-limit reed | Input, pullup, inverted |
| 33 | Open-limit reed | Input, pullup, inverted |
| 25 | Doorway beam receiver | Input, pullup, inverted |
| 16 | LD2410 RX | UART TX |
| 17 | LD2410 TX | UART RX |

> Use an **ESP32-WROOM**, not a WROVER — GPIO16/17 are tied up by PSRAM on
> WROVER boards.

Reeds and the beam are wired as **NC to GND**. With `pullup` + `inverted`, a
severed wire reads as "not triggered", which fails toward leaving the door
alone rather than moving it.

## Wiring

**Unplug the opener at the wall first.** The O/S/C and photocell terminals are
SELV low voltage, but there is mains inside the head. The manual notes that
installation should be done by a competent person per EN 12635.

1. **Relay → O/S/C.** Relay COM and NO across the two O/S/C terminals.
   **Potential-free only — never feed voltage into this input.**
2. **Schartec photocell → photocell terminal.** Remove the factory bridge.
   Test that breaking the beam reverses a closing door *before* enabling
   anything in software.
3. **Warning light → hazard terminal.** The opener flashes it during travel
   on its own; nothing to configure.
4. **Reeds.** One magnet at the fully-closed position, one at fully-open, each
   reed on the rail/frame so the carriage magnet lines up. Adjust so they make
   contact slightly *before* the hard limit.
5. **Independent beam.** Mount across the doorway, roughly 150–300 mm off the
   floor so a bike wheel breaks it. Receiver output to GPIO25.
6. **LD2410.** Inside, facing the door, ~1.2 m up. Tune the 200 cm threshold in
   the YAML to your bay.
7. **Power.** Simplest is a 5 V USB adapter in a nearby socket. If you tap the
   24–28 V accessory rail through a buck converter, remember the whole rail is
   only 300 mA and the photocells draw from it too.

## No-solder alternative

Swap the ESP32 + relay for a **Shelly Plus 1** (~€20) across O/S/C in
potential-free mode, and use its SW input for the closed-limit reed. You lose
the second reed and the on-device logic — everything moves into Home Assistant
automations, so the door stops automating when HA is down. Fine as a first step.

## If you don't want to open the opener head

Sacrifice one of the two included remotes: solder the relay contacts across its
button pads and power it from its own cell. You are pressing a real, paired
remote, so rolling code is untouched. You still need the reed switches, since
this gives you no state feedback either.

Keep the other remote in the car regardless. You will want a manual override
the first time a sensor misbehaves.
