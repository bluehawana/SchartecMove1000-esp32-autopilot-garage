# Safety

Read this before enabling auto-close. It is short, and it is the part that
matters most.

## What changes when you automate

With a remote you press a button and watch the door move. With autopilot the
door moves **with nobody watching**. Under **EN 12453** those are different
categories of operation with different requirements, and the Schartec manual
itself references EN 12453, EN 13241-1 and EN 60335-2-95.

The practical version: unattended closing needs working safety devices. The
photocell is not a nice-to-have here, it is the thing that makes this legal and
safe to run.

## Non-negotiables

1. **Fit the Schartec photocell** and remove the factory bridge from the
   photocell terminal. Verify a closing door reverses when you break the beam,
   by hand, before any software is enabled.
2. **Fit the warning light** on the 24–28 V hazard terminal. The board flashes
   it during travel for free.
3. **Leave the opener's force setting at factory (value 2).** The manual is
   blunt about this and it is right. Raising it to mask a badly balanced door
   is how people get hurt. If the door needs more force, fix the door.
4. **Consider a safety edge** on the bottom of the door. A photocell sees an
   object in the beam plane; a safety edge catches what the beam misses.

## Two independent beams, on purpose

This project uses **two** beams and it is worth understanding why.

- The **Schartec photocell** is wired only to the opener's safety loop. The
  ESP32 never reads it, never switches it, and cannot affect it. It works even
  if the ESP32 is unplugged, bricked, or on fire.
- The **second beam** is ours, for detecting that a car or bike has passed
  through.

Tapping a safety circuit to read its state is a classic way to accidentally
defeat it. An extra €15 keeps the certified safety path completely independent
of anything in this repo. If your photocell receiver happens to have a spare
dry NO contact, use that instead and skip the second pair.

## How the firmware fails

Deliberate choices in `garage-schartec.yaml`:

| Situation | Behaviour |
|---|---|
| Nobody passes through within 5 min | **Door left open.** Never blind-closes |
| Beam blocked at the re-check | Close aborted, door left open |
| Someone still detected in the bay | Close aborted, door left open |
| Wi-Fi or HA down | Fully functional — logic is on-device |
| ESP32 reboots | Relay forced off in `on_boot` |
| Sensor wire cut | Reads as untriggered; door is not moved |
| Mid-travel state | Never pulses unless a limit reed is definite |

Every failure path leaves the door **open and stationary** rather than moving.
An open garage is a security problem; a closing garage with someone under it is
a much worse one.

## Things that are still your job

- **Don't put an outdoor motion sensor on the open trigger.** Anything that
  detects generic "approach" will open your garage for the postman, a cat, or
  whoever notices it does that. Use an authenticated trigger for arrival —
  geofence plus car Bluetooth, a BLE beacon, or just the remote.
- **Turn off `Garage Auto-Open`** when working in the garage, or the door opens
  every time you walk near the front.
- **Check your insurance and, if applicable, your BRF/samfällighet rules**
  before running a self-closing garage door.
- **Re-test the photocell every few months.** It is the only thing standing
  between this project and a genuinely bad day.

## Cycle life

Schartec advertise the Move 1000 as tested to **20,000 cycles**. Autopilot means
more cycles — that is the point. At ~6/day that is roughly 9 years; at ~12/day,
closer to 4–5. It is a belt drive at a budget price point. Plan for it rather
than being surprised by it.
