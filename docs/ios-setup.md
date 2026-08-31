# iOS Setup

Everything here uses apps that already exist. Nothing custom to build, nothing
to submit to the App Store.

## 1. Home Assistant Companion (free, App Store)

The workhorse. Install it on both phones and it provides:

- **`device_tracker.phone`** — the geofence used by the arrival automations
- **`sensor.phone_activity`** — activity recognition, which is how we know
  you're **cycling** rather than driving
- **Actionable notifications** — the "Open the garage?" prompt with a button
- **Home screen + lock screen widgets**, and an Apple Watch complication
- **Siri** — "Hey Siri, open the garage"

Set your home zone radius to **~100–150 m**. Deliberately smaller than the
~250 m you'd need to have the door finished on arrival — with a parking space
out front you can wait a few seconds for free, and a tighter radius means the
door spends far less time open and unattended. See
[experience-design.md](experience-design.md).

## 2. Apple Home (HomeKit bridge)

Expose the door to the native Home app so it appears in Control Center, on the
Watch, and in CarPlay. In `configuration.yaml`:

```yaml
homekit:
  - name: Garage Bridge
    filter:
      include_entities:
        - cover.garage_door
```

Now it's a real HomeKit garage door: Siri, Control Center tile, Watch, and
"Hey Siri, is the garage open?"

## 3. Shortcuts personal automations — the good part

This is where iOS beats Android for our use case.

### Bluetooth connect → open the garage ⭐

**The best departure trigger available**, and it needs no CarPlay — the
**Toyota Prius+ (2016)** pairs over plain Bluetooth hands-free/audio, which is
all we need.

Shortcuts → Automation → **When Bluetooth connects** → pick your Toyota → Run
Immediately (turn **off** "Ask Before Running") → Get Contents of URL:

```
POST  http://homeassistant.local:8123/api/webhook/<your-webhook-id>
```

Why this is the right trigger:

- It fires the moment the car powers up and your phone pairs — while you're
  still doing up your seatbelt, ~15 s before you actually need the door
- It **cannot** fire when you aren't in the car. No GPS drift, no false
  positives, no "did the sofa open my garage" class of bug
- You're on home Wi-Fi in your own garage, so the webhook stays
  `local_only: true` and unreachable from the internet

> Check the exact pairing name first — Settings → Bluetooth. It's usually
> `Toyota` or `CAR MULTIMEDIA` on this generation. The automation and the HA
> template below both match on that string.

**Belt and braces:** the HA Companion app also exposes
`sensor.<phone>_bluetooth_connection`, so Home Assistant can watch for the same
event without Shortcuts. Run both — Shortcuts fires immediately, the HA sensor
catches it if a Shortcut automation gets disabled or an iOS update resets it.

### Wi-Fi as a trigger

Shortcuts → Automation → **Wi-Fi** also works, but mind the geometry:

- **Joining** home Wi-Fi means you're already within ~20–50 m of the house.
  At driving speed that's only a few seconds — **too late** to cover the door's
  15 s travel. Useful as a *confirmation*, not as the opener.
- **Leaving** home Wi-Fi is a decent "we've actually gone" signal, and the
  away-guard automation can use it.

Bluetooth beats Wi-Fi here because it tells you *which vehicle*, not just
*roughly where*.

### NFC tag by the house door

Stick a €1 NFC tag beside the internal door. Shortcuts → Automation → **NFC**.

Tap the phone on the way past and the door starts opening. Highest possible
intent, zero false positives — but it needs a free hand, which is exactly what
you don't have with a bike. Treat it as a nice manual override, not the primary
path.

### Arrive home

Shortcuts → Automation → **Arrive** → Home. Apple's geofence is independent of
HA's, so this is a useful second opinion. Keep "Ask Before Running" **on** for
arrivals — an accidental open while you're not there is exactly the failure
we're avoiding.

## Recommended combination

| Moment | Trigger | Head start |
|---|---|---|
| Leaving, by car | **Bluetooth connects to the Toyota** | ~15 s |
| Leaving, by car or bike | House→garage door contact | ~20 s |
| Leaving, armed window | Hallway sensor | ~30 s |
| Coming home, by car | HA zone + car Bluetooth | ~30 s |
| Coming home, by bike | HA zone + activity = cycling | ~30 s |
| Anything unusual | The remote in your pocket | — |

These overlap on purpose. Whichever fires first wins, the 60 s cooldown stops
them fighting each other, and a speculative open that nobody uses closes itself
after 90 s. Redundant early triggers are cheap; waiting at a closed door is not.

## Do you need a custom app?

No — and building one would be worse. The Companion app already has background
location, activity recognition and push notifications, all of which need
entitlements and battery-optimisation work you'd otherwise be doing yourself.
Shortcuts covers the rest.

## Security

- Keep the webhook `local_only: true`. A departure trigger only ever needs to
  work from inside your own garage.
- Generate a long random webhook id and keep it out of git:
  ```bash
  python3 -c "import secrets; print(secrets.token_urlsafe(32))"
  ```
  Put it in Home Assistant's `secrets.yaml` as `garage_webhook_id`.
- Don't expose Home Assistant to the internet just for this. If you want remote
  access, use HA Cloud or a VPN — not a port forward.
