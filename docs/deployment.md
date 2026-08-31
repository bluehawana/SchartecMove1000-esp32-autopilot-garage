# Deployment — where each piece lives

The split matters more than the parts list. Put the wrong box in the wrong room
and you get failures that only show up on the coldest or wettest week of the
year, which are the hardest kind to diagnose.

## The rule: rugged things outside, fragile things inside

```
  GARAGE                          INDOORS
  ──────                          ───────
  ESP32 + relay + sensors   ←→    Raspberry Pi 4  (Home Assistant)
  Schartec Move 1000              Mac mini        (camera processing)
                                  ASUS BD8        (network)
```

### Why the Pi does not go in the garage

Temperature ratings, which are not negotiable:

| Device | Rated range | Verdict for a Göteborg garage |
|---|---|---|
| **ESP32-WROOM** | −40 °C … +85 °C | ✅ Built for it |
| **Schartec Move 1000** | −20 °C … +40 °C | ✅ Designed for the room |
| **Raspberry Pi 4** | 0 °C … +50 °C | ❌ **Out of spec below freezing** |
| **USB SSD** | 0 °C … +70 °C | ❌ Same |

An unheated Swedish garage goes below zero. The Pi wouldn't die immediately —
it would misbehave on the coldest days of the year, which is worse.

Two more reasons, both independent of temperature:

- **Condensation.** Cars drag in snow and water; opening the door swaps warm
  air for cold. Repeated temperature swings plus moisture is what actually
  kills electronics, more reliably than cold alone.
- **Physical security.** If the Pi runs home-watch, presence lighting and
  camera alerts, it is the brain of the security system. The garage is the
  least secure room in the house. Putting the brain in the softest room —
  where it also controls the door — is backwards.

**None of this weakens the garage automation**, because the door logic lives
entirely on the ESP32. Wi-Fi down, HA down, Pi unplugged — the door still opens
and closes correctly. The Pi only adds arrival triggers and notifications.

## Siting the Pi: the laundry room

**The right room, and not as a compromise.** It is where the structured wiring
cabinet (*弱电箱*) already is — the old Telenor entry point, and where fibre
would terminate if Telia 1000 gets installed later. It is also a roomy utility
space that used to hold a printer and files, so there is shelf room to spare and
nobody cares about fan noise.

That gives you the two things that actually matter: **a wired Ethernet drop**,
and **one place to look** when something is wrong. Proximity to the garage is
irrelevant either way — the ESP32 reaches HA through the BD8 mesh regardless.

Open shelf, not inside the cabinet. Nothing to think about thermally.

The one habit worth keeping: **put it on a shelf, not on the floor.** The only
real laundry-room risk left is a washing machine leak, and being 40 cm up costs
nothing.

## Networking: Starlink + 5G means no port forwarding

Worth stating explicitly, because it rules out the usual advice.

The current setup — **Starlink plus Tele2 5G** — puts you behind **CGNAT** on
both links. You do not have a reachable public IP, so port forwarding is not
merely inadvisable here, it is **impossible**. Any guide telling you to forward
port 8123 cannot work on this connection.

That is fine, because the design never wanted it:

- **Remote access → Tailscale.** Works through CGNAT, needs no inbound port, and
  survives the WAN IP changing (which it will, on both links).
- **Backups → VPS pulls over Tailscale.** Pull, not push, so a compromised Pi
  cannot wipe the backups.
- **Departure trigger stays `local_only: true`.** The iOS Shortcuts webhook only
  ever needs to work from inside your own garage, on your own Wi-Fi.

If dual-WAN failover between Starlink and 5G is active, the WAN IP changes on
every switch — another reason nothing here should depend on a fixed address.

**When Telia fibre is installed, nothing on the Pi changes.** All of this is
LAN-side. You swap what feeds the cabinet; the Pi, the ESP32 and every
automation carry on unaware.

## Parts

| Item | Why | ~SEK |
|---|---|---|
| SanDisk **Max Endurance** microSD 256 GB | Endurance line, not Ultra/Extreme | 350 |
| Official USB-C PSU, 3 A | Undervolting causes bizarre, hard-to-trace faults | 150 |
| Case with heatsink | The Pi 4 runs warm | 120 |
| Ethernet cable | More reliable than Wi-Fi | 50 |

### Storage: a high-endurance card is fine

Common advice is "always boot from SSD". That is over-prescribed. What actually
kills cards is **write volume and the wrong card**, not the SD form factor.

**Pick the right SanDisk line** — this matters far more than capacity:

| Line | Designed for | For HA |
|---|---|---|
| Ultra | Cameras | ❌ Worst |
| Extreme / Extreme Pro | Fast video | ❌ Fast ≠ durable |
| **High Endurance** | Dashcams | ✅ Fine |
| **Max Endurance** | Security cameras, continuous write | ✅✅ Best |

High/Max Endurance are built for exactly our load: writing constantly, forever.
Extreme optimises sequential speed, which we never use.

**Buy bigger than you need — but for the right reason.** HA uses ~8 GB plus a
1–3 GB database, so 32 GB would do. A 256 GB card with 12 GB used has 240 GB of
spare blocks for wear levelling to rotate through, which spreads writes and cuts
per-block erase counts substantially. Buy big to spread the wear, not to fit.

> Don't pay extra for A2. It helps random IOPS in theory, but the Pi doesn't
> fully use command queuing, so the real gain is small. Spend that money on
> endurance instead.

### Cut the writes at the source — this matters more than the card

The firmware already throttles the noisy sensors: the LD2410 reports ~10× per
second, and publishing that raw would write **~864,000 rows a day** for one
sensor. Throttled to 1 s it is ~86,000 — a **10× reduction**, for free.

Then exclude what you don't need to keep history for. In `configuration.yaml`:

```yaml
recorder:
  purge_keep_days: 7
  commit_interval: 30        # default is 1s; batching cuts write ops hard
  exclude:
    entities:
      - sensor.garage_moving_distance   # commissioning only
      - sensor.garage_still_distance
      - sensor.garage_bay_distance
      - sensor.garage_wifi_signal
```

Keep the *binary* sensors and the cover — those are low-rate and are the ones
you actually want history for. It is the continuous numeric sensors that hurt.

### And back up off the device

With backups landing off the Pi, a dead card costs an hour, not a weekend. And
the garage keeps working throughout, because the door logic is on the ESP32.

## Install

### 1. Write Home Assistant OS to the card

Use **Home Assistant OS**, not Docker on Raspberry Pi OS. HA OS gives you the
Supervisor and add-ons, and — importantly for this project — host networking,
so mDNS discovery and the HomeKit bridge actually work.

Raspberry Pi Imager → *Other specific-purpose OS → Home Assistant →
Home Assistant OS (RPi 4)*.

### 2. First boot

Connect Ethernet and power, then open:

```
http://homeassistant.local:8123
```

Create the account. Set the home zone radius to **100–150 m** — deliberately
small; see [experience-design.md](experience-design.md) for why.

### 3. Add ESPHome

*Settings → Add-ons → Add-on Store → **ESPHome Device Builder***.

From then on you can edit and reflash the garage node from the browser instead
of a laptop, which matters when it is mounted on a garage ceiling.

### 4. Adopt the garage node

The ESP32 announces itself over mDNS; HA should offer to add it. If not, add
the ESPHome integration manually by IP and paste the API key from
`esphome/secrets.yaml`.

You should end up with `cover.garage_door` plus the sensors listed in the
[README](../README.md).

### 5. HomeKit bridge (optional, and nice)

```yaml
# configuration.yaml
homekit:
  - name: Garage Bridge
    filter:
      include_entities:
        - cover.garage_door
```

Now Siri, Control Center, Apple Watch. **This is the part that does not work on
macOS Docker** — it needs mDNS on host networking, which is why the Pi wins
over the Mac mini for HA.

### 6. Remote access and backups

Behind Starlink/5G CGNAT there is no public IP, so port forwarding is not an
option — which rules out the usual bad advice for free.

**Remote access → Cloudflare Tunnel.** Free, needs no inbound port and no fixed
IP, so it works through CGNAT and survives WAN failover.

> ⚠️ **Put Cloudflare Access in front of it** (free up to 50 users). A Tunnel
> without Access publishes your Home Assistant login page to the internet.

Tailscale is the alternative if you would rather not expose a hostname at all —
it needs a client on each device, but nothing is reachable from the open web.

**Backups → Cloudflare R2.** 10 GB free and no egress fees, S3-compatible so
HA's backup add-ons can target it directly. Get backups **off the Pi**, so a
dead card or a stolen Pi doesn't take the configuration with it.

### What not to use Cloudflare for

**D1 cannot replace HA's recorder**, despite being free and SQLite-flavoured:

- HA's recorder speaks SQLAlchemy — SQLite, MariaDB, PostgreSQL. D1 is an
  **HTTP API**. There is no driver.
- D1's free tier allows **100,000 row writes/day**. The throttled LD2410 alone
  produces ~86,000. One garage would exhaust it.
- Every write would be an internet round trip over CGNAT — the recorder would
  fall behind and the UI would stall.

Where D1 *does* fit: **daily aggregates**, one row a day instead of 86,000.
That is a good match for publishing real energy numbers on villalyft.com.

## Where the Mac mini fits

Keep it, but give it the compute work rather than the automation:

- **Camera object detection** (Frigate). A Pi 4 struggles with more than one or
  two streams; the Mac mini has the CPU for it.
- **Recording storage.**

> **Pi = automation brain** (light, dedicated, always up)
> **Mac mini = heavy lifting** (cameras, recording)

The point of the split is that when you reboot the Mac mini for your own work,
the garage and the house automations don't notice.

## Verify before you trust it

- [ ] Pi boots from SSD, not SD — `ls /dev/mmcblk*` should find nothing
- [ ] Survives a power cut: pull the plug, confirm it comes back unattended
- [ ] `cover.garage_door` responds from the HA app
- [ ] **Pull the Pi's power and confirm the garage door still works** — this is
      the test that proves the ESP32 is genuinely independent
- [ ] A backup has actually landed on the VPS, and you have restored one once
