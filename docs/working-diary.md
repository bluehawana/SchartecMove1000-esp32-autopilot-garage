# Working Diary

## 2026-08-31 — Project start: from a purchase question to a full design

**Where this began.** Not with code. The question was whether a Schartec Move
1000 could be made to open when you walk up to it and close once you've driven
out — and, once that looked plausible, whether the Move 1000 was even the right
thing to buy. The whole point is friction: a garage you have to operate by
remote every single time is a garage you stop using.

**Read the manual before designing anything.** Pulled the actual
[Series 3 manual](https://www.schartec.de/mediafiles/pdf/Schartec-Move-Series-3-English-Manual.pdf)
and extracted the terminal map rather than guessing at it. Four findings shaped
everything downstream:

- There is an **O/S/C dry-contact terminal** (Fig. 17). This is the whole
  ballgame — a plain momentary button input on a screw terminal. Plenty of
  cheap openers bury this behind a proprietary bus. This one just hands it over.
- The radio is **433.92 MHz rolling code**, 20 remote slots. That kills every
  RF shortcut: no cloning, no replay capture, no generic 433 MHz transmitter.
  Contact closure is the only way in. Good to know *before* buying an SDR.
- **There is no auto-close timer.** Searched the whole menu section. It doesn't
  exist on the Series 3. So 100% of the "hold, then close" behaviour is ours.
- The photocell terminal (Fig. 18) **ships with a bridge fitted**, and the
  photocell itself is **not in the box**. Easy thing to discover too late.

**Buy verdict: yes, but it's a dumb-and-hackable opener, not a smart one.**
Recommended it on the strength of the O/S/C terminal alone. Flagged three
pre-purchase checks: door size limits (14 m², 140 kg), rail length (standard
K-rail only covers a sectional door to 2400 mm — M/L rails exist and
retrofitting is miserable), and budgeting ~€35 for the photocell as a separate
line item.

**Design: two directions, two threat models.** This was the decision that
shaped the architecture. Going *out* in the morning, you're already inside the
garage, therefore already authorised — a presence sensor is safe. Coming *home*
is completely different: any sensor that detects generic "approach" will open
the door for the postman, a cat, or whoever notices it does that. So arrival
needs an authenticated trigger, never a motion sensor. Split accordingly.

**The "hold until away" mechanic.** The neat part. A beam that breaks and then
clears is precisely the signal that a car or bike has physically passed
through — and it works identically for both, which a timer never would. Beats
a fixed countdown, which is what most commercial auto-close does.

**Logic on the ESP32, not in Home Assistant.** Deliberate. The door should not
stop working because HA is mid-update. HA only gets arrival and notifications.

**Two independent beams.** Also deliberate, and worth the extra €15. The
Schartec photocell stays wired *solely* to the opener's safety loop — the ESP32
never reads it and never switches it. A separate beam does passage detection.
Tapping a safety circuit to read its state is exactly how people accidentally
defeat one.

**Failure direction.** Every path leaves the door **open and stationary**: no
passage in 5 min → left open; beam blocked at re-check → aborted; someone still
in the bay → aborted; sensor wire cut → reads untriggered (NC + pullup +
inverted); reboot → relay forced off. An open garage is a security problem. A
closing garage with someone under it is a much worse one.

**On arrival, the default is a notification, not automation.** Phone geofences
drift 100–200 m, which means a pure GPS trigger can swing the door open while
you're still down the road and can't see it. One tap removes that entire
failure class. The fully-automatic version is written but ships
`initial_state: false` and requires two agreeing signals — geofence crossed
*and* phone on the car stereo, the Bluetooth check proving you're in the car
rather than the phone drifting on a kitchen counter.

### Not verified

Being explicit, because none of this has met reality yet:

- **No hardware exists.** Nothing is bought, nothing is wired, nothing has been
  flashed. This is a design, not a tested build.
- **`esphome config` has never run** — the CLI isn't installed on this machine.
  YAML syntax parses clean under a plain parser and the structure follows the
  component docs, but that is *not* schema validation. Expect to fix things.
- **The 200 cm approach threshold is a guess.** Needs measuring against the
  real bay depth.
- **Reed positions are unspecified** — they depend on the rail and carriage.
- **LD2410 mounting height (~1.2 m) is a starting point**, not a tested value.
- **Pinout is untested on real silicon.** Chose GPIO 25/26/27/32/33 to dodge
  strapping pins, and UART on 16/17 — which means an ESP32-**WROOM**, since
  WROVER uses those for PSRAM.

### Next step

Order the opener plus the photocell, checking rail length against the actual
door height first. Then `pip install esphome` and run `esphome config` to get a
real validation pass before any hardware arrives — cheapest possible place to
find the mistakes.

Commissioning order is in the README, and steps 1–3 happen with the door on
manual release. Do not skip them.

---

## 2026-08-31 (later) — The install video reframes the whole project

**The spark, properly identified.** Went looking for the Schartec install video
(it's on schartec.de, QR code in the manual). It's a fine video — rail,
brackets, limits, pair the remote — and it **ends with someone standing in
their garage pressing a button.** That's the thesis of this project in one
frame: the video ends exactly where our problem starts. Wrote it up in
`TLDR.md` because it's the clearest statement of why any of this exists.

**Found the number that breaks the naive design: 160 mm/s.** A 2400 mm door
takes **15 seconds**. Trigger the door at the moment you want to drive through
and you sit staring at it — worse than a remote, not better. Everything
restructured around one rule: **never trigger at the moment of need.**

**Speculative vs commanded opens.** The design change that made early
triggering affordable. Opening 20–30 s early means guessing, and guessing means
being wrong sometimes (you opened the house door to fetch a screwdriver, now
the garage is open to the street). So the firmware now tracks *why* the door
opened: a **speculative** open closes itself after 90 s; a **commanded** one is
left alone, because a person asked for it. Being wrong became cheap, which is
what unlocked aggressive early triggers everywhere else.

**"Is the car back?" — a real gap the user caught.** The arrival trigger keyed
on a phone geofence, which conflates *a person is home* with *a vehicle is
back*. Two concrete bugs: if your phone is already home the `enter` event never
fires again (door never opens for you), and a housemate walking home fires it
when no door is needed. Fixed by sensing the **vehicles**, not the people —
ultrasonic ranger at the bay for the car, BLE tag under the saddle for the
bike. Car arrival confirmed by phone-on-car-stereo, bike arrival by the
companion app's activity recognition. Walking home matches neither and
correctly opens nothing.

**Indoor early trigger, and why it needs a guard.** The user wanted a detector
further inside the house. Correct instinct, but a living-room sensor fires all
evening and you can't cycle the door every time someone crosses to the sofa.
Added a `button.garage_speculative_open` entry point so out-of-room sensors can
reach the node via HA, guarded by a "departure armed" window plus a
vehicle-actually-present check. Noted in the docs that the **hallway outside
the garage door** is the better mounting spot than the living room — fires far
less often, keeps most of the head start.

**iOS: CarPlay connect is the standout.** Best departure trigger found. It
fires as you start the car, ~15 s before you need the door, and it *cannot*
fire when you're not in the driver's seat — no GPS drift, no false positives.
Runs over a `local_only` webhook, so it stays unreachable from the internet.
Concluded no custom app is worth building: the HA Companion app already has
background location, activity recognition and push, and Shortcuts covers the
rest. Written up in `docs/ios-setup.md`.

**Speed variant: rejected.** User flagged it's ~30% more. Agreed and documented
the reasoning — 3 s per cycle you were never waiting on, *and* a lower weight
limit (100 kg vs 140 kg). Pay 30% more for less capacity, or spend the same
money on every sensor in the BOM. Anticipation is the right lever, not speed.

**Linked to villalyft.com.** Fits the Villalyftet model: publish real numbers
as we go. The energy angle is genuine rather than retrofitted — a garage door
standing open through a Göteborg winter is measurable heat loss, and "I forgot
to close it" is the usual cause. Close-after-passage caps the door's open time
at ~35 s per trip.

### Still not verified

Everything from the previous entry still stands (no hardware, `esphome config`
never run), plus:

- **Ultrasonic bay threshold (2.0 m) is a placeholder.** Must be measured
  against the actual bay depth and the actual car.
- **HC-SR04 echo is 5 V** — needs a divider or a 3.3 V-safe module. Not yet
  chosen.
- **`esp32_ble_tracker` alongside Wi-Fi, UART and everything else is RAM-heavy.**
  Flagged in the config as the first thing to drop if the node gets unstable.
  Untested.
- **The BLE MAC in the config is a placeholder** (`AA:BB:CC:DD:EE:FF`).
- **iOS activity-recognition entity names differ** between Android
  (`on_bicycle`) and iOS (`Cycling`) — needs checking against the real entity.
- **All head-start timings are estimates**, not stopwatch measurements.

### Next step

Order: Move 1000 (standard, **not** Speed) + photocell, checking rail length
against door height. Then `pip install esphome && esphome config` for a real
validation pass before any hardware lands.

### Corrections to the above (same session)

- **CarPlay was wrong.** The car is a **Toyota Prius+ (2016)** — no CarPlay.
  The trigger is plain **Bluetooth pairing**, which the Shortcuts automation
  supports identically ("When Bluetooth connects → Toyota"). Same ~15 s head
  start, same zero-false-positive property. Also noted the HA Companion app
  exposes `sensor.<phone>_bluetooth_connection`, so HA can watch the same event
  without Shortcuts — run both, they cover each other's failure modes.
- **Wi-Fi evaluated and mostly rejected as a trigger.** Joining home Wi-Fi means
  you're already ~20–50 m out, which is too late to cover 15 s of travel.
  Bluetooth wins because it identifies *which vehicle*, not just *roughly where*.
- **Single-person assumption was wrong.** Arrival automations now trigger on any
  household member and check *that specific person's* Bluetooth/activity sensor
  via a variables mapping. Checking the wrong person's phone would open the door
  for someone sitting in the living room.

## 2026-08-31 (later still) — The BD8 mesh as a presence layer

**User has three ASUS ZenWiFi BD8 nodes, one per floor.** That's a coarse
indoor positioning system they already own, so wrote up how to use it.

**Reframed the goal, which was the more useful contribution.** The ask was "not
15 seconds, maybe 5." But you can't make the Move 1000 travel faster — 160 mm/s
is a belt and physics. **The target isn't 5 seconds, it's zero**, and the way
there is starting earlier, never moving faster. A door that took 15 s but began
moving 20 s ago was never waited on.

**Built the trigger cascade** — four layers, each earlier and less certain:
floor change (40–60 s) → hallway (30 s) → house door (20 s) → Bluetooth (15 s).
They overlap deliberately; first to fire wins, cooldown stops them fighting,
wrong guesses self-close. Layer 3 alone already covers the travel time.

**Stated the limitation honestly rather than selling the clever thing.** Wi-Fi
roaming is sticky — iOS often won't leave an AP until ~−70 dBm, so a
three-floor descent might register in 5 seconds or 45. 802.11k/v/r helps and
should be on, but doesn't fix it. So mesh node tracking is good for *coarse
context* and bad as a precision timer. Recommended **Bermuda/ESPresense BLE**
(~€8/node, 1–2 s, room-level) as the real precision layer, reusing the same
BLE tag already planned for the bike.

**Recommended build order deliberately puts the clever thing last:** house-door
contact and Bluetooth first (cheap, certain, probably sufficient), live with it
a week, measure whether anyone is actually ever waiting, and only then add the
mesh/BLE layers if the data justifies it.

### Additionally unverified

- **AiMesh per-client node attribution** via the `asusrouter` HACS integration
  is the one thing here I couldn't check at all. Firmware-dependent. Flagged
  inline in the doc — verify before building on it.
- Node names in the template sensor (`BD8-Attic` etc.) are placeholders.
- All cascade head-start figures remain estimates, not stopwatch measurements.

### Correction: the arrival geofence was over-engineered

User mentioned they have **an off-street parking space in front of the garage**.
That invalidates the 250 m arrival geofence I specced.

The asymmetry I'd missed: **departure needs anticipation, arrival doesn't.**
Going out you're sitting in the car with the engine running and 15 s is
genuinely irritating. Arriving, you pull onto your own spot and line up, and the
door opens during that — **you are never blocking a neighbour's access, so the
wait costs nothing.**

I had sized the geofence to eliminate a wait that was already free, and paid for
it in exposure: a 250 m radius leaves the door open and unattended while you're
a block away, on a signal that can itself be 100–200 m wrong. Cut the arrival
radius to **~100–150 m**. Less exposure, fewer false opens, less heat lost in
February, and the only cost is seconds spent on your own parking spot anyway.

Noted the condition explicitly in the docs: if there's *no* off-street spot and
waiting means blocking traffic, size up and accept the exposure. The right
answer depends on which case you're in.
