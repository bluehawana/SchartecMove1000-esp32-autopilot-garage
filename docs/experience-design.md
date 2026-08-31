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

## The house has no internal door to the garage

This constraint shapes everything, so it goes first.

**The garage's only opening is the main door.** There is no house→garage
internal door. To reach the car you leave the house, walk round outside, and in
through the garage opening — which means **the door has to be open before you
get there**, or you cannot get in at all.

Three consequences, all of which killed an earlier design:

| Trigger | Verdict |
|---|---|
| Internal house→garage door contact | ❌ **No such door exists** |
| Phone pairing to the car's Bluetooth | ❌ **Too late** — if you're in the car, it was already open |
| mmWave inside the garage | ❌ You can't be inside before it opens |

> **Every useful departure trigger has to come from inside the house.**

The indoor sensors are therefore not a refinement, they are the mechanism. What
was "layer 1 and 2, nice to have" is now the whole thing.

## Drive out — morning

```
T+0s    Hall presence, heading for the door       ← TRIGGER (indoor)
T+2s    Door starts moving
T+12s   You're out of the house, walking round
T+17s   Door fully open
T+20s   You reach the garage opening ......... it was open before you arrived
T+35s   In the car, engine on
T+45s   You drive out
T+48s   Beam breaks, then clears
T+56s   Garage is empty — you left in the car — so it closes
```

The exterior house door contact is a **backup** trigger, not the primary one:
it fires as you step outside, and the walk from there to the garage opening is
shorter than the door's 15 s travel. It is better than nothing and it is very
cheap, but on its own you would stand and wait.

### Measure your own walk

Every number above depends on your house. Time these three with a stopwatch
before setting anything:

1. Hall sensor → out of the house
2. House door → standing at the garage opening
3. Garage opening → engine running

If (1)+(2) is under 15 s, you need the earlier mesh/floor trigger as well.

## Drive in — arriving home

```
T-20s   Geofence + car Bluetooth agree              ← TRIGGER
T-18s   Door starts moving
T-3s    Door fully open — you were parking anyway
T+0s    You drive in
T+3s    Beam breaks, then clears
T+11s   Doorway clear ......... but you are still in the car
T+40s   You get out, unload the shopping
T+55s   You walk into the house
T+65s   Garage reads empty, settles
T+75s   Door closes behind you
```

**The door does not close at T+11s**, and that is the whole point of this
section. Closing on a cleared doorway is correct when you drove *out* and wrong
when you drove *in* — because driving in leaves a person in the garage.

## The rule that covers both directions

The naive fix is to detect direction: two beams a hand's width apart, and the
order they break tells you which way something went. It works, and it's an extra
sensor and a pile of edge cases.

You don't need it. **One rule covers both:**

> Close when the doorway is clear **and there is no person in the garage.**

Watch what that does in each case, with no direction sensing at all:

| | Drove out | Drove in |
|---|---|---|
| Doorway clear? | Yes, after the dwell | Yes, after the dwell |
| Person in the garage? | **No** — they left with the vehicle | **Yes** — climbing out of the car |
| Result | Closes after the ~8 s dwell | **Waits** until they walk inside |

Same code path. No second beam, no direction logic, no state machine that can
get its two cases the wrong way round. And it is the *safe* rule as well as the
convenient one: the door never closes while a person is in the garage, which is
what you would have had to guarantee anyway.

The mmWave sensor's `has_target` gets a deliberately **asymmetric** debounce —
instant to say "someone is here", 20 seconds slow to say "nobody is". Its job is
to veto closing, so it should be eager to veto and reluctant to release. After
the garage first reads empty there's a further 10 s settle, because a person
standing still is exactly the case mmWave is worst at.

If someone is still pottering about after ten minutes, the door gives up and
**stays open**, and you get a notification. It never resolves that standoff by
closing.

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
| `occupancy_timeout` (10 min) | firmware subs | How long you potter before it gives up |
| `empty_settle` (10 s) | firmware subs | Raise if it ever closes while you're still inside |
| `cycle_cooldown_ms` (60 s) | firmware subs | Raise if it reopens on you |
| Approach threshold (200 cm) | `approach_zone` lambda | Sensor to where you park |
| Geofence radius (~100–150 m) | HA zone | Keep it small if you have off-street parking |

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
