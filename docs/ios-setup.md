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

Set your home zone radius to **~250 m**, not the default. See
[experience-design.md](experience-design.md) for the arithmetic — it needs to
cover the door's 15 s travel at your approach speed.

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

### CarPlay connect → open the garage ⭐

**The single best departure trigger available.**

Shortcuts → Automation → **When CarPlay connects** → Run immediately (turn
*off* "Ask Before Running") → Get Contents of URL:

```
POST  http://homeassistant.local:8123/api/webhook/<your-webhook-id>
```

Why this is so good:

- It fires the instant you start the car — while you're still doing up your
  seatbelt, ~15 s before you actually need the door
- It **cannot** fire when you aren't in the driver's seat. No GPS drift, no
  false positives, no "did the sofa open my garage" class of bug
- You're on home Wi-Fi in your own garage, so the webhook can stay
  `local_only: true` and unreachable from the internet

If you don't have CarPlay, **"When Bluetooth connects → \<car stereo\>"** works
identically.

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
| Leaving, by car | **CarPlay connects** | ~15 s |
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
