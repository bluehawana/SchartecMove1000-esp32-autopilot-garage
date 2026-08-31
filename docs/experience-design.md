# Experience Design — "smooth as butter"

The goal is a garage that never asks for anything. No remote, no app, no
looking back. This doc is the choreography, and the reasoning behind the
timings.

## The constraint everything bends around

**Move 1000: 160 mm/s.** A 2400 mm door = **~15 s** of travel.

```
   Move 1000        160 mm/s    2400mm door →  15.0 s
   Move 1000-Speed  200 mm/s    2400mm door →  12.0 s
```

Fifteen seconds is an eternity when you're sitting in a car waiting. It is
nothing at all if the door started moving while you were still putting your bag
in the boot.

**So: never trigger at the moment of need. Always trigger before it.**

That one rule generates the entire design. Every trigger below is chosen for
*how early it fires*, not for how accurately it detects intent.

## Drive out — morning

The house→garage door is the signal. Not the garage door, not the car.

```
T+0s    House→garage door opens          ← TRIGGER, speculative open
T+2s    Door starts moving
T+8s    You're at the car, loading
T+15s   Door fully open ........................ nobody was waiting on it
T+20s   You're in, engine on
T+25s   You drive out .......................... zero wait
T+27s   Beam breaks, then clears
T+35s   Door closes behind you
```

**You never see a closed door and you never press anything.**

### The screwdriver problem

You'll also open that house door to grab a screwdriver, and now the garage is
standing open to the street.

Solved by distinguishing *why* the door opened:

| Open reason | If nobody drives out | Why |
|---|---|---|
| **Speculative** (house door) | **Closes again after 90 s** | We guessed. We were wrong. Undo it |
| **Commanded** (remote / app / HA) | **Stays open** | A person asked. Don't second-guess them |

That's the `speculative_open` global in the firmware. It's what makes a
15-second head start affordable — being wrong is cheap, because it self-corrects.

## Drive in — arriving home

```
T-30s   Phone crosses the ~250m geofence + car Bluetooth connected  ← TRIGGER
T-28s   Door starts moving
T-15s   Door fully open
T+0s    You turn in and drive straight through .... never slow down
T+3s    Beam breaks, then clears
T+11s   Door closes behind you
```

### Sizing the geofence — smaller than you'd think

The obvious sizing is "big enough that the door finishes before I arrive":

```
   30 km/h  =  8.3 m/s   ×  15 s  =  125 m   → ~250 m with margin
   40 km/h  = 11.1 m/s   ×  15 s  =  167 m   → ~300 m
```

**Don't do this.** It optimises a problem you probably don't have.

There's a **parking space in front of this garage**, and that changes the whole
calculation. Arriving, you pull onto your own spot, put it in reverse, line up —
and the door opens during that. You are never blocking a neighbour's access
while you wait, so **the wait costs nothing.**

Going out is the opposite: you're sitting in the car with the engine running,
and 15 s is genuinely irritating. That's where anticipation earns its keep.

> **Departure needs anticipation. Arrival doesn't.**

So size the arrival geofence *down*, to **~100–150 m** — near enough that you're
almost home. A 250 m radius means the door stands open and unattended while
you're a block away, and a phone GPS can be 100–200 m out on its own. A smaller
radius means less exposure, fewer false opens, and less heat out the door in
February. You give up seconds you were spending on your own parking spot anyway.

If you don't have an off-street spot — if waiting means blocking traffic — then
size up and accept the exposure. Know which case you're in.

### Why not just GPS

Phone geofences drift 100–200 m. On GPS alone the door will eventually open
while the phone sits on the kitchen counter and you're nowhere near it.

Requiring **car Bluetooth as well** is what fixes this. It proves you're
physically in the car. Two independent signals that must agree — that's why
the automatic arrival automation ships disabled and asks for both.

## "Is the car back yet?" — the living-room question

You're inside. Someone is still out. You want to know whether they made it home.

A phone geofence **cannot** answer this. It tells you a *person* is in the
zone, which is a different fact from a *vehicle* being in the garage:

| Situation | Phone geofence says | Truth |
|---|---|---|
| You're in the living room | "home" | Says nothing about the car |
| Partner walks home from the bus | "home" (enter fires) | No vehicle arrived, no door needed |
| Partner drives home | "home" (enter fires) | Car arrived — door needed |
| Car in the garage, you're out | "away" | Car *is* home |

This also breaks the arrival trigger. If your phone is already in the zone, the
`enter` event never fires again — so the door never opens for you. And if a
housemate walks home, it fires when no door is needed.

**So we sense the vehicles directly, not the people.**

| Vehicle | Sensor | Why this one |
|---|---|---|
| **Car** | Ultrasonic ranger at the bay | Empty bay reads the far wall; a parked car reads ~2 m. Cheap, reliable, works in the dark |
| **Bike** | BLE beacon under the saddle + ESP32 BLE tracker | A bike has no Bluetooth and carries no phone. A €10 tag gives it an identity |

That yields two honest facts — `binary_sensor.garage_car_present` and
`binary_sensor.garage_bike_present` — which are what the living-room
notifications actually key off. "Car is back in the garage" means the car is in
the garage, not that a phone is nearby.

They also make the arrival triggers smarter: **don't open the door if the bay
is already occupied.** If the car is home, whatever crossed the geofence didn't
arrive by car.

### Telling car-arrival from bike-arrival

The vehicles announce themselves differently, so the triggers differ:

- **Car:** geofence enter **+ phone connected to the car stereo.** The
  Bluetooth check is the load-bearing part — it proves you're physically in the
  car, and it eliminates GPS drift as a failure mode in one step.
- **Bike:** no Bluetooth to lean on, so we use the companion app's **activity
  recognition** — the phone knows it's being carried on a bicycle
  (`on_bicycle` on Android, `Cycling` on iOS).
- **Walking home:** neither condition matches, so nothing opens. Correct — you
  don't need the garage door to walk in the front door.

## What breaks butter, and the fix

| Friction | Fix |
|---|---|
| 15 s of staring at a closed door | Trigger from the house door / geofence, not the garage door |
| Door left open after a false trigger | Speculative opens self-close after 90 s |
| Reopens as you walk back past the sensor | 60 s cooldown after any cycle |
| Cat, shadow, or someone crossing the bay | 1.5 s dwell requirement on the approach zone |
| Door starts closing while you're under it | Photocell (hardware) + beam + presence re-check |
| "Did it close?" anxiety | Push notification, and a nag if it's open 10 min |
| Working in the garage, door keeps opening | `Garage Auto-Open` switch — turn it off |
| Winter heat loss | Prompt close-after-passage; keep the dwell tight |

## When you should still press a button

Being honest about the limits — the remote isn't decoration:

- **Guests and deliveries.** No sensor should authenticate a stranger.
- **Anything unusual** — moving furniture, a trailer, a door half-blocked.
- **First-time-in-a-while** operation after ice, wind, or a power cut. Look at
  the door while it moves.
- **Bikes with a passenger or trailer** that might sit in the beam oddly.

Keep one remote in each car. You will want it the first time a sensor
misbehaves, and that day will come.

## Tuning against your actual bay

Every number here is a starting point. Measure and adjust:

| Value | Where | Measure this |
|---|---|---|
| `clear_dwell` (8 s) | firmware subs | Longest vehicle, tail to fully clear |
| `speculative_timeout` (90 s) | firmware subs | Slowest realistic walk-and-load |
| `cycle_cooldown_ms` (60 s) | firmware subs | Raise if it reopens on you |
| Approach threshold (200 cm) | `approach_zone` lambda | Sensor to where you park |
| Geofence radius (~250 m) | HA zone | Approach speed × 15 s, plus margin |

Time your own door with a stopwatch on first commissioning. If it's not
2400 mm, every timing above shifts.

## Verdict on the Speed variant: skip it

The **Move 1000-Speed** does 200 mm/s — about 3 s faster each way on a 2400 mm
door, ~6 s per round trip. It costs roughly **30% more**.

**Not worth it here**, for three reasons:

1. **Predictive triggering already absorbs the travel time.** The house-door
   trigger fires ~20 s ahead of when you need the door. Shaving 3 s off a
   15 s travel you were never waiting on buys you nothing.
2. **It carries a lower door weight limit** — 100 kg vs 140 kg. You'd pay 30%
   more for *less* capacity.
3. That 30% buys the photocell, the beam, the ESP32, the LD2410, the reeds and
   the bay sensor with change left over — i.e. the entire thing that actually
   makes the door feel instant.

Buy the standard **Move 1000** and spend the difference on sensors. Speed is
the wrong lever; **anticipation** is the right one.
