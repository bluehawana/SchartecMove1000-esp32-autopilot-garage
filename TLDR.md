# TL;DR

## The spark

We started where everyone starts: watching the
[Schartec installation video](https://www.schartec.de) — the one the QR code in
the manual points to. It's a good video. Rail, brackets, motor head, set the
travel limits, pair the remote, done.

And then it ends with someone standing in their garage, pressing a button.

That's the moment the idea landed. **The video ends exactly where our problem
starts.** Following it perfectly gets you a motor on a door. It does not get
you a garage you'll actually use.

## The gap

A remote-operated garage still costs you a small ritual, every single time:

- Find the remote. It's in the other car. It's in yesterday's jacket.
- Press it. Wait. Watch.
- Drive through.
- Turn around, check it closed. Or don't, and wonder about it all day.

None of that is *hard*. It's just enough friction that the garage quietly turns
into a storage room and the car lives on the driveway. Multiply by two people,
twice a day, through a Swedish winter, holding a bike or a coffee or a kid.

**The functionality we need isn't "remote control." It's "no control."**

## What that means in our reality

| Situation | What must happen |
|---|---|
| Morning, walking out with the car | Door is **already open** when we get there |
| Wheeling the bike out | Same — no free hand for a remote |
| Driving out | Door closes behind us, unasked |
| Coming home | Door is open before we reach the driveway |
| Driving in | Closes **after we've got out and gone inside** — not while we're unloading |
| Someone's under the door | It stops. Every time. No exceptions |

## The number that shapes everything

The Move 1000 travels at **160 mm/s**. A 2400 mm door takes **~15 seconds**.

That single fact rules out the obvious design. If you trigger the door when you
*want* to drive through, you sit and stare at it for fifteen seconds — which is
worse than the remote, not better.

**So the trigger has to fire before the moment of need.** Everything else in
this project falls out of that constraint:

- Going out, the trigger is the **internal house→garage door**, not the garage
  door. It fires ~20s before you're ready to move. By the time you're in the
  car, the door has been open for five seconds.
- Coming home, the trigger is a **geofence ~250 m out** — about 30 seconds at
  residential speed. Enough to arrive at an open door without slowing down.
- Closing is triggered by a **beam that breaks and then clears**, which is the
  precise signal that a car or bike has physically passed through. Works
  identically for both. A fixed timer never could.
- And it closes only once **nobody is left in the garage** — so driving out it
  shuts straight away, while driving in it waits for you to unload and go
  inside. One rule, both directions, no direction sensing needed.

## How we get there

The Move 1000 has no Wi-Fi, no app, no auto-close timer, and rolling-code
remotes that can't be cloned. But it has one thing that matters: an **O/S/C
dry-contact terminal**. A plain momentary button input on a screw terminal.

So we drive that terminal with a relay, and add the senses the opener lacks:

```
ESP32
 ├── relay ──────────── O/S/C terminal        (the only way to command it)
 ├── 2× reed switch ─── open / closed limits  (the opener never tells us)
 ├── IR beam ────────── doorway passage       (the "you're clear" signal)
 ├── LD2410 mmWave ──── who's in the bay      (the "don't close" signal)
 └── door contact ───── house→garage door     (the 15-second head start)
```

All the logic lives **on the ESP32**, not in Home Assistant — the door keeps
working when the network doesn't. Home Assistant only handles arrival and
notifications.

Separately and untouched, the **Schartec photocell** stays wired solely to the
opener's own safety loop. The ESP32 never reads it, never switches it, and
cannot compromise it. That's the part that has to work even if everything in
this repo is on fire.

## Knowing the vehicle is back

A phone geofence says *a person* is in the zone. That is not the same fact as
*the car is in the garage* — and if your phone is already home, the geofence
never fires again, so the door never opens for you.

So we sense the vehicles, not the people: an **ultrasonic ranger** at the bay
for the car, and a **BLE tag under the saddle** for the bike, which has no
Bluetooth and carries no phone. Car arrival is confirmed by the phone being on
the car stereo; bike arrival by the phone's activity recognition saying you're
cycling. Walking home matches neither, and correctly opens nothing.

## Cost

~**€110** of parts on top of the opener. See [docs/hardware.md](docs/hardware.md).

**Skip the Speed variant.** ~30% more for 3 seconds per cycle, and a lower
weight limit (100 kg vs 140 kg). Anticipation beats speed — and that 30% pays
for every sensor above.

## Part of Villalyftet

Documented under **[villalyft.com](https://villalyft.com)**, same as the other
Askim projects: real numbers, published as we go. The energy case is real too —
a garage standing open through a Göteborg winter costs heat, and
close-after-passage means it's open for ~35 seconds instead of all afternoon.

## The honest part

A door that closes with nobody watching is a genuinely different machine from
one you drive by remote while looking at it — different enough that EN 12453
treats them as separate categories. The photocell isn't optional here.

Every failure path in this design leaves the door **open and stationary**
rather than moving. An open garage is a security problem. A closing garage with
someone under it is a much worse one.

Read [docs/safety.md](docs/safety.md) before enabling anything.
