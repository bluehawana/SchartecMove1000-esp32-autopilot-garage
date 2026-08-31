# Measurements to take on site

Everything in this repo is currently built on estimates. These are the numbers
that turn it into a real design. Take this list to the garage on your phone.

Nothing should be ordered before section 1 is done.

---

## 1. Before ordering — the number that cannot be wrong

### ☐ Door height (mm)

Measure the **opening**, floor to the underside of the lintel.

| Height | Rail you need |
|---|---|
| ≤ 2400 mm | **K-rail** — the standard kit |
| 2400–2900 mm | **M-rail** (3655 mm total) |
| 2900–3400 mm | **L-rail** (4155 mm total) |

Retrofitting a longer rail later means dismantling the installation. This is
the single most expensive thing to get wrong.

### ☐ Door area (m²) — must be ≤ 14 m²
### ☐ Door weight, balanced (kg) — must be ≤ 140 kg

If you cannot weigh it: a balanced door should stay put when you lift it
halfway by hand. If it slams down or flies up, the springs need attention
**before** any opener goes on — and that is a job for a professional, because a
garage door spring under tension is genuinely dangerous.

### ☐ Headroom above the door (mm)

Space between the top of the opening and the ceiling, for the rail and carriage.

### ☐ Is there a mains socket in the garage?

If not, that is an electrician's job — budget for it.

---

## 2. The walk — decides how many triggers you need

Stand where the **hall sensor** would go. Start the timer. Walk normally,
through the laundry room, out its exterior door, and stop when you are standing
at the garage opening.

**Do it three times and take the fastest.**

### ☐ Hall → standing at the garage opening: ______ seconds

| Result | What it means |
|---|---|
| **> 15 s** | Hall sensor alone is enough. Skip the mesh layer entirely |
| **10–15 s** | Current design. Expect 1–5 s of waiting on a fast exit |
| **< 10 s** | You need the mesh floor-change trigger as well |

Also worth timing separately:

### ☐ Laundry exterior door → garage opening: ______ s  *(estimated 3–6)*
### ☐ Garage opening → engine running: ______ s

---

## 3. Inside the garage — sensor placement

### ☐ Internal length and width (mm)
Estimated ~3.1 × 6.1 m from the floor plan, never measured.

### ☐ Distance: back wall → front of the parked car (mm)

Sets the ultrasonic threshold. The firmware currently guesses **2.0 m** —
replace it with a value comfortably between "car parked" and "bay empty".

### ☐ Distance: LD2410 mounting point → where you park (mm)

Sets the `approach_zone` threshold, currently a placeholder **200 cm**.

> Your garage is long and narrow with the door at one end, which is not the
> geometry I assumed. Plan to mount the LD2410 on the **back wall looking down
> the length** so it covers both the walk-in path and the car.

### ☐ Where can the doorway beam mount? (both sides, 150–300 mm off the floor)
Low enough that a bike wheel breaks it.

### ☐ Where do the limit reeds go on the rail?
One at fully closed, one at fully open — they should make contact slightly
*before* the mechanical limit.

---

## 4. Network and power

### ☐ Can a cable reach the garage from the laundry room?
The house and garage are separate structures. If a conduit or route exists,
wired Ethernet (or PoE) beats Wi-Fi. If not, the BD8 mesh is fine.

### ☐ Wi-Fi signal strength inside the garage, door closed
Check on a phone standing where the ESP32 will go. A metal door can attenuate
badly.

### ☐ Is there an Ethernet drop in the laundry room?
For the Pi. Almost certainly yes, given the network cabinet is there.

---

## After measuring

Update these, then everything else follows:

| Value | Where |
|---|---|
| Ultrasonic threshold (`2.0`) | `esphome/garage-schartec.yaml`, `car_in_bay` lambda |
| Approach threshold (`200` cm) | same file, `approach_zone` lambda |
| All timings | [experience-design.md](experience-design.md) |
| Rail choice | [hardware.md](hardware.md) |

Then re-run:

```bash
.venv/bin/esphome config esphome/garage-schartec.yaml
```
