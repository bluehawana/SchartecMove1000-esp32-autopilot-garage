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

## 2026-08-31 (final) — The close rule was wrong for arrivals

**The bug the user caught.** The design closed the door 8 seconds after the
doorway beam cleared. That is right when you drive *out* and **wrong when you
drive in** — driving in leaves a person standing in the garage, so the door
would start closing while they were still climbing out of the car or carrying
shopping. Not just annoying: it's the exact scenario the whole safety story is
supposed to prevent.

**Rejected the obvious fix.** Direction detection — two beams a hand's width
apart, order of breaking tells you which way something went — works, and costs
an extra sensor plus a state machine with two cases that can be wired up
backwards.

**The rule that made it disappear:**

> Close when the doorway is clear **and there is no person in the garage.**

No direction sensing at all. Driving out, the garage empties the moment you do,
so it closes promptly. Driving in, occupancy holds the door until you've
unloaded and gone inside. **Same code path, both directions**, and it happens to
be the safe rule as well as the convenient one — the door simply never closes on
an occupied garage, which had to be guaranteed anyway.

**Made the occupancy sensor deliberately asymmetric.** `has_target` is instant
to say "someone is here" and carries a 20 s `delayed_off` before it will say
"nobody is", plus a further 10 s settle. Its job is to *veto* closing, so it
should be eager to veto and reluctant to release. A person standing still is the
case mmWave is worst at, so the debounce leans toward assuming occupied.

**Standoffs resolve open, never closed.** Someone still pottering after ten
minutes → the door gives up and stays **open**, with a notification. It never
breaks a deadlock by moving.

**Generalised the framing.** The user pointed out this applies to every villa
owner, not just this house — nobody should have to hurry into their garage
inside fifteen seconds. Rewrote README, TLDR and experience-design around that,
and the villalyft.com case page too.

### Repo state at end of session

Design complete and internally consistent. Still true: **no hardware exists,
`esphome config` has never run**, and every timing is an estimate. Full list of
unverified items in the entries above — the ultrasonic threshold, the BLE MAC
placeholder, AiMesh node attribution, iOS activity entity names, and reed
positions all need real measurement.

## 2026-08-31 (close-out) — Firmware validated, docs swept

**`esphome config` finally ran, and it passes.** Installed ESPHome 2026.8.2 into
a local venv and validated `garage-schartec.yaml`: **`Configuration is valid!`**,
no warnings, no strapping-pin complaints. That was the single biggest unverified
item across the whole project and it is now closed. Added the exact commands to
the README under "Validate before you buy anything", because the cheapest place
to find mistakes is before the parcels arrive.

A compile (which would additionally type-check the C++ lambdas) was started;
that is a deeper check than `config` and the remaining known gap on the firmware.

**Consistency sweep found four stale claims** the geofence correction had missed.
README, TLDR and ios-setup.md all still told the reader to set a **250 m** home
zone, contradicting the decision to shrink it to 100–150 m. Fixed all four.
Worth noting as a pattern: changing a design decision means grepping for the old
number, not just editing the doc where the reasoning lives.

### What is still unverified

Shorter than it was, and now accurate:

- **No hardware exists.** Nothing bought, wired or flashed.
- **Ultrasonic bay threshold (2.0 m)** — placeholder, must be measured.
- **HC-SR04 echo is 5 V** — needs a divider or a 3.3 V-safe module.
- ~~`esp32_ble_tracker` RAM headroom~~ — **measured: it fits.** Full compile
  gives RAM 56.9% (70,916/124,580 B), Flash 70.9% (1,300,347/1,835,008 B), so
  ~53 KB spare. Caveat: that is *static* allocation. Runtime heap under Wi-Fi/BLE
  coexistence is still unproven without hardware.
- **BLE MAC is a placeholder** (`AA:BB:CC:DD:EE:FF`).
- **AiMesh per-client node attribution** — firmware-dependent, unchecked.
- **iOS activity entity names** (`Cycling` vs `on_bicycle`) — unchecked.
- **All timings are estimates**, not stopwatch measurements.

**Compile passes too, not just config.** Ran a full `esphome compile` — exit 0,
`Successfully compiled program`, no errors. That type-checks every C++ lambda in
the file, which `config` does not: the cover's `COVER_OPEN` returns, the
`millis()` cooldown arithmetic, the `optional<bool>` return in the bay-distance
template. All good.

The build also answered the open BLE question. `esp32_ble_tracker` drags in the
entire Bluedroid stack — the compile spent ~600 of its 1524 objects on it — and
it still **fits**: RAM 56.9%, Flash 70.9%, ~53 KB spare. Being precise about what
that proves: the *image* fits. Runtime heap under simultaneous Wi-Fi and BLE is a
different question and needs real silicon. Noted in the config that this is the
first section to delete if the node misbehaves.

## 2026-08-31 (deployment) — Where each box lives

User has an always-on Mac mini, two VPSes and a Raspberry Pi 4, and asked where
Home Assistant should run. Answer: **the Pi, indoors.**

**Ruled out the VPS** — not on performance, on *location*. HA has to be on the
same LAN as the ESP32: the ESPHome API, the `local_only` iOS webhook (which is
`local_only` deliberately), BLE presence, and AiMesh node attribution all break
across the internet. More fundamentally, a garage door should not depend on an
internet round trip.

**Ruled out the Mac mini** for HA specifically: Docker on macOS has no host
networking, which breaks mDNS discovery and — the one that matters — the
**HomeKit bridge**, so no Siri or Control Center. Also it gets rebooted for
other work. Gave it the camera/Frigate role instead, where its CPU is genuinely
useful and a reboot costs nothing.

**Temperature is the load-bearing argument** for keeping the Pi out of the
garage: Pi 4 is rated 0–50 °C and a Swedish garage goes below zero, while the
ESP32 is −40…+85 °C. It wouldn't fail outright, it would misbehave on the
coldest week of the year, which is worse.

**Was wrong about the laundry room, and the user's reason settled it.** I had
argued for the upstairs storage room on humidity grounds. But the laundry room
is where the *弱电箱* is — the old Telenor entry point, and where Telia fibre
would terminate if it gets installed. That makes it the network hub, not a
compromise: wired Ethernet, one place to look when something breaks.

Then over-corrected: assumed the Pi would go *inside* the cabinet and wrote a
whole section on cabinet ventilation and thermal throttling. Wrong again — the
room is a roomy utility space that used to hold a printer and files, so the Pi
sits on an open shelf with space around it and thermals are a non-issue. Cut
that section and the fan from the parts list.

What actually survives as advice is one line: put it on a shelf rather than the
floor, because a washing machine leak is the only real risk left and 40 cm of
height is free. Twice in one section I invented a constraint the room did not
have.

**The SD card warning gets its own callout.** It's the most common way an HA Pi
dies, and the failure mode here is the garage not recognising you one morning.

**Best verification step in the doc:** pull the Pi's power and confirm the
garage still opens. That is the test that proves the ESP32 is genuinely
independent, which is the whole architectural claim.

**CGNAT rules out the usual advice.** The current WAN is Starlink + Tele2 5G,
and both are behind CGNAT — so there is no reachable public IP and port
forwarding is not merely inadvisable, it is **impossible**. Any guide saying to
forward 8123 simply cannot work here. Fortunately the design never wanted it:
Tailscale for remote access, VPS *pulls* backups over Tailscale, and the iOS
departure webhook stays `local_only` because it only ever fires from inside the
garage. Also noted that dual-WAN failover changes the WAN IP on every switch,
and that a later fibre swap changes nothing Pi-side since it is all LAN.

### Not verified

- Nothing here has been performed — no Pi imaged, no SSD bought, no HA
  installed. This is a plan.
- Whether this Pi 4 already has USB-boot firmware (units from ~mid 2020 do;
  older need an EEPROM update) — unchecked.
- **Thermals inside the 弱電箱 are a guess.** Depends on what else is in there
  and whether it vents. Measure `thermal_zone0` after a week before trusting it.
- Whether dual-WAN failover is actually configured, or the links are manual.

## 2026-08-31 (floor plan) — Two corrections from the actual house

**The garage has no internal door.** The user sent the floor plan and then
stated it plainly: the garage's only opening is the main door. You leave the
house, walk round outside, and in through the garage opening.

That invalidated the trigger the whole departure design rested on:

| Trigger | Why it died |
|---|---|
| Internal house→garage door contact | **No such door exists** |
| Bluetooth pairing to the Toyota | **Too late** — in the car means it was already open |
| mmWave inside the garage | You cannot be inside before it opens |

So **every useful departure trigger must come from inside the house.** What I
had filed as "layer 1 and 2, nice-to-have margin" is now the entire mechanism.
Rewrote the cascade around hall presence, with the exterior house door as a
cheap but late backup, and added a stopwatch step — time the real walk, because
every number in that section depends on this specific house.

**What survived without changes: the close logic.** Worth noting because it was
not luck. The rule is "close when the doorway is clear *and* nobody is in the
garage", with no direction sensing. Under the new route the beam now breaks
twice on the way out (walk in, then drive out) and twice on the way in — and
the rule still behaves correctly in all four cases, because it waits on *people
leaving*, not on beam events. Refusing to build a direction-detecting state
machine is what made it survive a change to the building.

### Then the better question: why write to the database at all

User asked what the frequent database writes are actually *for*. Correct
instinct, and the honest answer is: for this project, almost nothing.

The distinction I had not made explicit: **HA's state machine lives in RAM and
is what every automation reads. The recorder database exists only for history
graphs and the logbook.** So the test is simply *will anyone ever look at a
graph of this?*

Applied to our sensors, the answer is no for every high-rate one. The raw
mmWave distance exists to set a threshold **once**; `bay_distance` in
centimetres is not what you want, `car_present` is. Meanwhile the genuinely
useful history — when the door opened, when the car came home — is naturally
tiny: a garage opens maybe ten times a day.

So throttling was treating the symptom. Changed the diagnostics to
`disabled_by_default: true` with `entity_category: diagnostic`: enable in the
UI while tuning, disable afterwards, and they are **not recorded at all**. The
`approach_zone` lambda is unaffected because it reads on-device state, never the
database.

**Withdrew the SSD recommendation.** "Always boot from SSD" was over-prescribed.
With the write volume removed at source, a SanDisk **Max Endurance** card is
fine — and bigger helps for a non-obvious reason: 256 GB with 12 GB used gives
wear levelling far more spare blocks to rotate through. Buy big to spread the
wear, not to fit.

### Cloudflare: right instinct, wrong component

User asked why we weren't using Cloudflare's free D1. Checked it properly: D1
**cannot** back HA's recorder — the recorder speaks SQLAlchemy (SQLite/MariaDB/
PostgreSQL) and D1 is an HTTP API with no driver, its free tier allows 100k row
writes/day which one throttled LD2410 would exhaust, and every write would be an
internet round trip over CGNAT.

But the instinct was right, and three Cloudflare pieces do fit: **Tunnel** for
remote access through CGNAT (with **Access** in front, or it publishes the HA
login to the internet), **R2** for off-device backups, and **D1 for daily
aggregates** — one row a day, which suits publishing real numbers on
villalyft.com.

### Still not verified

- The floor plan suggests a door between the stair landing and the laundry
  room; whether that is one threshold or two is **unconfirmed** and decides
  where the exterior door contact goes.
- Garage dimensions (~3.1 × 6.1 m) are **scaled off the drawing**, not measured.
- The walk time from hall to garage opening is the number the whole cascade
  depends on and **has not been timed**.

## 2026-08-31 (measurements) — The walk is 3-6 seconds, and that is a problem

User confirmed the route: the laundry room has **its own exterior door**, then
you walk outside to the garage's main door. **Two thresholds, neither internal.**
And the walk itself is only **3-6 seconds**.

That number is the difficulty. The door needs 15 s, so an exterior door contact
gives nowhere near enough. It splits into two cases:

- **Last person out** (locking up, checking phone): 15-20 s → zero wait. Fine.
- **Not last out** (straight through): 3-6 s → **~10 s standing at the garage.**

Ten seconds standing outside is the exact problem this project exists to remove,
so the trigger had to move further into the house. Hall presence is now the
primary: ~10-14 s of head start, since it is the single corridor to the exit and
does not fire all evening the way a living-room sensor would.

**Recorded the honest residual.** Even hall-triggered, a fast exit still leaves
1-5 s of waiting. That is arithmetic, not a tuning problem: 15 s of travel
against an ~11 s walk. Wrote it into the doc rather than quietly hoping, along
with the mitigating context — you are walking throughout rather than sitting in
a car, and the most common case (locking up) is zero. Also added an explicit
"do not walk under a moving door" line, because that is the obvious wrong way to
close the gap.

**Architecture consequence: a second node.** The departure triggers are all
inside the house while the door hardware is in the garage, and the two are
separate structures. So a small indoor ESP32 in the laundry room takes the
exterior door contact — conveniently right beside the network cabinet — and the
hall sensor is easier as an off-the-shelf wireless unit straight into HA. BOM
up from ~EUR110 to ~EUR140.

### Still unmeasured

- **Garage dimensions** — user confirmed these are not known and need measuring
  on site. My ~3.1 x 6.1 m was scaled off the drawing.
- **The walk time is the user's estimate, not a stopwatch reading.** Every
  timing in experience-design.md scales from it, so it is the first thing to
  measure and the most consequential.

### Session close — what turns estimates into a design

Added `docs/measurements.md`: a checklist to take to the garage, ordered so that
nothing gets bought before the one irreversible number is known.

The two that matter most:

1. **Door height** — decides K/M/L rail. Retrofitting a longer rail means
   dismantling the installation, so it is the most expensive thing to get wrong.
2. **Hall → garage walk time** — decides how many trigger layers are needed.
   Over 15 s and the mesh layer can be dropped entirely; under 10 s and it
   becomes mandatory.

Also included a safety note that does not belong to the automation at all: if
the door does not stay put when lifted halfway, the springs need professional
attention **before** any opener is fitted. A garage door spring under tension is
dangerous, and no amount of correct firmware compensates for an unbalanced door.

Every placeholder in the firmware is cross-referenced to the measurement that
replaces it, so there is no guessing about which number goes where.

## 2026-09-01 — Filled the hole yesterday's redesign left

Yesterday's floor-plan correction moved every departure trigger indoors and
added an indoor ESP32 to the BOM — but `esphome/` still held only the garage
node. The repo described a two-node system and shipped one. Fixed.

**`esphome/indoor-node.yaml`** — laundry exterior door contact, plus an optional
wired hall input for anyone who would rather pull cable than buy a wireless
sensor. Validated and compiled: **RAM 25.3%, Flash 48.2%**. Much lighter than
the garage node, which carries the whole Bluedroid stack for the bike tag.

Applied yesterday's database lesson from the start rather than retrofitting it:
diagnostics are `entity_category: diagnostic` + `disabled_by_default`, so
nothing high-rate is ever recorded.

**Rewrote the HA automations** that were still placeholders — hall presence as
the primary trigger, the laundry door as an honest backup. Both press
`button.garage_speculative_open`; a double-fire is harmless because the garage
node ignores a second press inside its 60 s cooldown.

**Recorded a dependency rather than hiding it.** The open trigger now routes
through Home Assistant, which is a step back from "works without HA". Written
into the config header and the README instead of glossed: if HA is down you use
the remote, and the *close* logic — the half where failing badly actually
matters — stays entirely on the garage node.

A separate API key for the indoor node, not a shared one.

### Not verified

- Neither node has run on hardware.
- **The wired hall input is speculative** — the recommendation is still an
  off-the-shelf wireless sensor, since the hall is a different room from the
  laundry. The GPIO exists for whoever finds a cable run convenient.
- `binary_sensor.hall_presence` in the automations is a placeholder name; it
  must be swapped for whatever the real sensor turns out to be.
- Pin choices on the indoor node are untested on real silicon.
