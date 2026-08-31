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

**This is the right room, and not as a compromise.** It is where the structured
wiring cabinet (*弱电箱*) already is — the old Telenor broadband entry point,
and where fibre would terminate if Telia 1000 gets installed later. Temperature
and fan noise are both non-issues there.

Siting the Pi with the network gear means a wired Ethernet drop instead of
Wi-Fi, and one place to look when something is wrong. Proximity to the garage is
irrelevant either way — the ESP32 reaches HA through the BD8 mesh regardless.

The cabinet also solves most of what would otherwise be laundry-room problems:
it is wall-mounted and enclosed, so the Pi is off the floor, above any plausible
flood line, and shielded from dryer lint and steam.

### The real risk is heat, not damp

A closed *弱电箱* with a router, an ONT or 5G modem and a Pi 4 all running is a
small warm box, and the Pi 4 throttles when it gets hot. So:

1. **Ventilate the cabinet** — vents top and bottom, or a small low-noise fan.
   Nobody hears it in a laundry room, which is one of the reasons this room
   works.
2. **Don't stack the Pi directly on the router.** Give it air on at least two
   sides.
3. **Heatsink case**, and check the temperature after a week:
   `cat /sys/class/thermal/thermal_zone0/temp` — under 60 °C idle is healthy,
   sustained 80 °C+ means it is throttling.

Still worth doing, cheaply: keep it toward the **top** of the cabinet (heat
rises, water doesn't), and not directly above the washing machine if the
cabinet placement gives you a choice.

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
| USB SSD 240 GB + USB3 enclosure | **Not an SD card** — see below | 350 |
| Official USB-C PSU, 3 A | Undervolting causes bizarre, hard-to-trace faults | 150 |
| Case with heatsink | The Pi 4 runs warm | 120 |
| Ethernet cable | More reliable than Wi-Fi | 50 |
| Cabinet vent or 40 mm fan | The *弱电箱* is a small warm box | 100 |

### ⚠️ Do not boot from an SD card

This is the single most common way a Home Assistant Pi dies. HA writes to its
database constantly; SD cards typically fail within months to a year. The
failure mode is that your garage stops recognising you one morning.

**Boot from a USB SSD.** A cheap 240 GB drive is plenty.

## Install

### 1. Enable USB boot (only if needed)

Pi 4 units made from roughly mid-2020 support USB boot out of the box. Older
ones need a bootloader update — Raspberry Pi Imager →
*Misc utility images → Bootloader → USB Boot* → write to a spare SD card, boot
once, done.

### 2. Write Home Assistant OS to the SSD

Use **Home Assistant OS**, not Docker on Raspberry Pi OS. HA OS gives you the
Supervisor and add-ons, and — importantly for this project — host networking,
so mDNS discovery and the HomeKit bridge actually work.

Raspberry Pi Imager → *Other specific-purpose OS → Home Assistant →
Home Assistant OS (RPi 4)* → write to the SSD over USB.

### 3. First boot

Connect Ethernet and power, then open:

```
http://homeassistant.local:8123
```

Create the account. Set the home zone radius to **100–150 m** — deliberately
small; see [experience-design.md](experience-design.md) for why.

### 4. Add ESPHome

*Settings → Add-ons → Add-on Store → **ESPHome Device Builder***.

From then on you can edit and reflash the garage node from the browser instead
of a laptop, which matters when it is mounted on a garage ceiling.

### 5. Adopt the garage node

The ESP32 announces itself over mDNS; HA should offer to add it. If not, add
the ESPHome integration manually by IP and paste the API key from
`esphome/secrets.yaml`.

You should end up with `cover.garage_door` plus the sensors listed in the
[README](../README.md).

### 6. HomeKit bridge (optional, and nice)

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

### 7. Backups to the VPS

The one job the VPSes are genuinely good for here. HA takes automatic backups;
get them **off the Pi**, so a dead SSD or a stolen Pi doesn't take the config
with it.

Simplest reliable setup: put the Pi and a VPS on **Tailscale**, then have the
VPS pull the backup directory on a schedule. Pull, not push — a compromised Pi
then can't wipe your backups.

Port forwarding isn't an option anyway behind Starlink/5G CGNAT, which is one
fewer bad idea available to you.

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
