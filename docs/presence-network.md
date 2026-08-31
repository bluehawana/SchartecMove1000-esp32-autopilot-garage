# Using the ASUS BD8 Mesh as a Presence Network

Three ZenWiFi BD8 nodes across three floors is already a coarse indoor
positioning system. This is how to read it, what it's genuinely good for, and
where it will let you down.

## First, reframe the target

The goal isn't "5 seconds instead of 15." **The goal is zero.**

You can't make the Move 1000 travel faster than 160 mm/s — that's physics and a
belt. What you *can* do is start it earlier. A door that took 15 s but began
moving 20 s ago is a door you never waited for at all.

So the question is never "how do I shorten the travel?" It's **"what is the
earliest honest signal that we're leaving?"**

## The trigger cascade

Each layer fires earlier and is less certain than the one below it. They
overlap on purpose — whichever fires first wins, the 60 s cooldown stops them
fighting, and a wrong guess closes itself after 90 s.

| Layer | Signal | Head start | Confidence |
|---|---|---|---|
| 1 | **Floor change** — phone roams to the ground-floor BD8 | 40–60 s | Low |
| 2 | **Hallway presence** — mmWave outside the garage door | ~30 s | Medium |
| 3 | **House→garage door** contact opens | ~20 s | High |
| 4 | **Bluetooth** — phone pairs to the Toyota | ~15 s | Certain |

Layer 3 alone already covers the 15 s travel. Layers 1–2 are margin for the
case where you walk straight to the car and pull away fast.

**Anything at layer 1 or 2 must be a speculative open** — that's the whole
reason the mechanism exists.

## Reading the mesh

Home Assistant's built-in AsusWRT integration gives you device trackers, but
not *which node* a client is on. For AiMesh node attribution use the
**`asusrouter`** integration from HACS (Vaskivskyi), which exposes the
connected access point per client.

> Verify this against your firmware before building on it — AiMesh client
> attribution varies by firmware version, and it's the one thing here I
> couldn't test.

Then map nodes to floors:

```yaml
# configuration.yaml
template:
  - sensor:
      - name: "Owner Floor"
        state: >
          {% set ap = state_attr('device_tracker.owner_phone', 'connected_to') %}
          {% if   ap == 'BD8-Attic'   %} 3
          {% elif ap == 'BD8-Middle'  %} 2
          {% elif ap == 'BD8-Ground'  %} 1
          {% else %} unknown {% endif %}
```

And trigger on **descending toward the garage**:

```yaml
- alias: "Garage - early open on descent to ground floor"
  id: garage_floor_descent
  mode: single
  triggers:
    - trigger: state
      entity_id: sensor.owner_floor
      to: "1"
  conditions:
    - condition: template
      value_template: "{{ trigger.from_state.state in ['2', '3'] }}"
    - condition: state
      entity_id: input_boolean.garage_departure_armed
      state: "on"
    - condition: state
      entity_id: cover.garage_door
      state: "closed"
    - condition: or
      conditions:
        - condition: state
          entity_id: binary_sensor.garage_car_present
          state: "on"
        - condition: state
          entity_id: binary_sensor.garage_bike_present
          state: "on"
  actions:
    - action: button.press
      target:
        entity_id: button.garage_speculative_open
```

Note it triggers on the **transition** `3→1` or `2→1`, not on merely *being* on
the ground floor. Direction is the signal. Sitting in the kitchen all evening
never fires it.

## The honest limitation: Wi-Fi roaming is sticky

This is the part to understand before you rely on it.

Phones cling to an access point well past the point where a closer one would be
better. iOS is especially conservative — it often won't roam until the current
AP drops below roughly −70 dBm. So walking from the third floor to the ground
floor may show up as a node change **in 5 seconds, or in 45, or not until you
open the front door.**

The BD8s support 802.11k/v/r fast roaming, which helps. Turn it on. It does not
make the problem go away.

**What this means practically:** mesh node tracking is good for *coarse
context* — "is anyone on the ground floor" — and bad as a precision timer. Use
it as layer 1, where being early and occasionally wrong is fine by design.
Don't use it as your only trigger.

## If you want real room-level precision

Use BLE, not Wi-Fi. You're already buying ESP32s and a BLE tag for the bike, so
the marginal cost is small.

**Bermuda** (HACS) or **ESPresense** turns a handful of ESP32s into a
room-level positioning system by trilaterating BLE signal strength:

- ~€8 per node, one per floor plus one near the garage
- Room-level accuracy, updating in **1–2 seconds** rather than 5–45
- Tracks the same BLE tag you're putting on the bike, and Apple Watches and
  iPhones via HA's own BLE transmitter

This is the right tool for layer 1. The mesh is the free approximation of it;
BLE is the thing that actually works to a couple of seconds.

## Recommended build order

1. **Start with layer 2** (hall presence). With no internal door to the garage,
   an indoor trigger is the mechanism, not an optimisation — see
   [experience-design.md](experience-design.md).
2. **Add the exterior door contact** as a cheap backup. It fires late, but it
   costs almost nothing.
3. **Time your actual walk** with a stopwatch: hall → out of the house → at the
   garage opening. If that is under 15 s, you need layer 1 too.
4. **Only then** add the mesh layer, and expect it to be coarse.

Bluetooth-to-car is deliberately absent from departures: with no internal door,
by the time you are in the car the door had to be open already.
