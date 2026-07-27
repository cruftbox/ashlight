# Ashlight

An ambient lamp that shows outdoor air quality as a colour.

It polls a PurpleAir PA-II on the local network, applies the EPA's own
correction for PurpleAir sensors, computes AQI against the 2024 breakpoints, and
renders the result on a 16-LED ring. There is no cloud service, no API key, and
no dependency on anything outside your LAN. If your internet drops, the lamp
keeps working.

It is named for what it warns about. Ash falling during smoke season is when it
earns its keep.

Built with [ESPHome](https://esphome.io/). The entire device is one YAML file.

> **Home Assistant is required.** The lamp reads `sun.sun` from Home Assistant to
> decide whether it is day or night, and that drives the brightness schedule.
> Without it the ring will still poll and show the correct colour, but it will
> sit at night brightness around the clock, which is too dim to read in daylight.
> Any Home Assistant install works, including Home Assistant Container. No
> add-ons are needed. See [Home Assistant](#home-assistant) below, including how
> to drop the dependency if you would rather not run it.

---

## Requirements

- A **PurpleAir PA-II** on your LAN, reachable by IP.
- **Home Assistant**, reachable on the same LAN. See the note above.
- **ESPHome** on a machine you can flash from. No dashboard needed.
- The hardware below.

---

## What it looks like in use

The ring shows a continuous colour gradient, not six discrete steps.

| AQI | Colour | Category |
|---|---|---|
| 25 | `#00E400` green | Good |
| 75 | `#FFFF00` yellow | Moderate |
| 125 | `#FF7E00` orange | Unhealthy for Sensitive Groups |
| 175 | `#FF0000` red | Unhealthy |
| 250 | `#8F3F97` purple | Very Unhealthy |
| 400 | `#7E0023` maroon | Hazardous |

Those are anchors at the **midpoint** of each EPA category, and the ring
interpolates between them. AQI 25 is pure green, AQI 75 is pure yellow, and the
50/51 category boundary is a 50/50 yellow-green. Below 25 and above 400 the
colour is flat rather than extrapolated.

Anchoring at midpoints rather than at category edges means a category reads as
itself when you are solidly inside it, while still showing proximity to the next
one. It also means the lamp can never flicker between two categories, which
matters because most houses sit near a boundary most of the time. An ambient
display should be calm; that is the entire point of the object.

**Brightness** follows a day/night schedule, read from Home Assistant's sun
entity so no coordinates live in the config.

**When the sensor is unreachable**, the ring shows three rotating red/green/blue
segments instead of a colour. It moves, it shows three colours at once, and blue
appears nowhere in the AQI palette, so it cannot be mistaken for a reading. This
is also what shows from power-on until the first successful poll.

---

## Hardware

| Item | Notes |
|---|---|
| ESP32-C3 SuperMini | Generic clone. Raw GPIO numbers on the silkscreen. |
| WS2812B ring, 16 LED, 45 mm | Confirm DI/DO direction before soldering. |
| 330 Ω resistor | Goes at the **ring** end of the data run. |
| 1000 µF electrolytic, 6.3 V or better | Across 5 V/GND close to the ring. |
| 5 V USB supply, 2 A | Headroom against brownout resets. |
| Wire, heat shrink, USB-A-to-C cable | See the note on cables below. |
| PurpleAir PA-II | On your LAN, reachable by IP. |

Total cost is dominated by the PurpleAir. Everything else is a few dollars.

### Wiring

```
ESP32-C3 SuperMini                          16 x WS2812B ring

        5V  ─────────────────────────────────  5V ──┐
                                                    ├─ 1000 µF
        GND ─────────────────────────────────  GND ─┘   (stripe to GND)

        GPIO3 ──────────────────[330Ω]───────  DI
                                    └─ at the ring end
```

Four connections. Three things about them are easy to get wrong:

**The 330 Ω goes at the ring end, not the MCU end.** It is on the data line, not
the power line, and it does not drop the supply. Its jobs are damping the
reflection off an unterminated wire, and limiting current through the first
pixel's ESD protection if the board drives data before the ring's 5 V has come
up. Both require it close to DI.

**Do not put a resistor in series with the ring's supply.** Each WS2812B has
constant-current drivers built into the package. There is no LED current to
limit.

**The capacitor's stripe is negative** and goes to GND. An electrolytic fitted
backwards will vent.

### Pin choice

Use GPIO3, 4, 5 or 10 for the data line. This config uses **GPIO3**.

Avoid GPIO2, GPIO8 and GPIO9. They are strapping pins sampled at boot. GPIO8
typically also drives the onboard LED on these clones.

### WiFi range, and where you can actually put this

Plan for this before you pick a location. The SuperMini's onboard ceramic
antenna is significantly worse than the phone or laptop you will instinctively
use to judge coverage. A spot where your phone shows full bars can still be
unusable for this board.

Measured on this build, same access point in both cases:

| RSSI | Behaviour |
|---|---|
| -32 to -35 dBm | 0% packet loss, every poll succeeds |
| -72 to -81 dBm | 33 to 40% packet loss, **zero** polls succeed |

The failure mode at the low end is worth recognising, because it does not look
like "no WiFi". The device stays associated, answers pings, and holds an open
ESPHome API connection, while every HTTP fetch fails with `Code: -1`. An
established connection carrying a trickle of data survives a lossy link; a fresh
TCP handshake, which needs several round trips to land in order, does not.

If polls fail while the device is plainly online, check RSSI before suspecting
the config, but do not stop there. See "when it is not the WiFi" below, because
a slow sensor produces a similar-looking result for an entirely different reason.

No wifi option rescues an out-of-range location. `power_save_mode` and 802.11k/v
are both in this config and neither recovered a -77 dBm link.

#### The antenna mod

The onboard ceramic antenna is the limiting factor, and a 31mm wire soldered to
the antenna feed is a well-travelled fix. 31mm is a quarter wavelength at
2.4 GHz. On this build it was worth about **4 dB**: mean -72.7 dBm over 147
samples in a location that previously measured -72 to -81.

Two honest caveats. First, an early 20 minute sample suggested 8 dB, and the
longer run did not support it. RSSI in one room spanned 14 dB across a morning,
so judge a location on hours of samples, not minutes. Second, 4 dB is real but
it is not transformative, and it did not turn out to be what was blocking the
polls in that room.

#### When it is not the WiFi

Weak signal and a slow sensor both show up as failed fetches, and they are worth
telling apart before you start moving furniture.

- **Weak signal** fails *fast*. `Code: -1` arrives well inside the timeout,
  because the TCP handshake never completes.
- **A slow sensor** fails at exactly the timeout. The request runs the full
  10000 ms and then gives up, because the sensor is going to answer, just not
  in time.

The second one is common. The PA-II is a single-threaded ESP8266 that uploads to
its own cloud service every 120 seconds and cannot serve its local web server
while it does. Measured from a wired host: responses of 0.1s, 6.5s, 11.4s and
13.2s, plus outright drops, all from a sensor whose own RSSI was -50 dBm. Nothing
on the ESP32 side can fix that.

This is why the poll interval is roughly 6 minutes and jittered rather than a
flat 2. Two unsynchronised 120 second cycles drift slowly against each other, so
the failures do not scatter, they arrive in long runs. A 2h29m capture caught 55
consecutive successes followed by 18 consecutive failures with one clean
transition and no reboot. A fresh random offset every cycle keeps the two cycles
from settling into phase, and cuts request load on the sensor at the same time.

### A note on USB cables

A known batch of SuperMini clones ships without the USB-C CC pulldown resistors.
With a C-to-C cable the board looks dead. **Use an A-to-C cable for the first
flash.** If the flasher still cannot see it, hold BOOT, tap RESET, release BOOT.

---

## Build

### 1. Install ESPHome

```
pip install esphome
```

Developed against ESPHome 2026.7.2. No dashboard or web flasher is needed; the
CLI does everything.

### 2. Create `secrets.yaml`

Copy `secrets.yaml.example` to `secrets.yaml` and fill it in. That file is
gitignored and must stay that way.

Generate the API encryption key with `esphome wizard`, or take any 32-byte
base64 string.

### 3. Set your sensor's address

In `ashlight.yaml`, under `substitutions:`:

```yaml
sensor_ip: "192.168.4.25"
```

Find your PA-II's address in your router's lease table, then confirm it with:

```
curl http://<SENSOR_IP>/json
```

You should get several KB of JSON back. If that command hangs, the sensor's
onboard web server is not responding, and nothing downstream will work.

A DHCP reservation for the sensor is worth setting up. The lamp resolves nothing
by name on the sensor side.

### 4. Flash

First flash must be over USB:

```
esphome run ashlight.yaml
```

After that, over the air:

```
esphome run ashlight.yaml --device OTA
```

**Leave the device powered for a full minute after an OTA reports success.**
ESPHome marks a boot successful after 60 seconds, and a device that resets
before then rolls back to the previous image. This is easy to trip over while
handling a device you are still assembling.

### 5. Watch it work

```
esphome logs ashlight.yaml --device OTA
```

A successful poll logs one line:

```
[I][ashlight]: AQI 53  corrected 10.21  smoothed 10.12  rgb 0.56/0.95/0.00  bright 0.15
```

---

## Configuration

Everything you are likely to want to change is in the `substitutions:` block at
the top of `ashlight.yaml`.

| Key | Default | What it does |
|---|---|---|
| `sensor_ip` | — | Your PA-II's LAN address. |
| `fail_threshold` | `4` | Consecutive failed polls before the ring gives up and shows the offline pattern. At roughly 6 minutes a poll, 4 is about 24 minutes. |
| `bright_day` | `0.15` | Ring brightness when the sun is up, 0.0 to 1.0. |
| `bright_night` | `0.03` | Ring brightness after dark. |

Brightness wants tuning against your particular diffuser, but `0.03` is a floor
rather than a taste. Brightness scales the 8-bit channels and `gamma_correct` is
1.0, so a low value leaves very few levels to carry a mixed hue. At `0.03` the
channels land near 8/255 and every anchor still reads as itself. Below that they
collapse in a way that puts out wrong information: at `0.02` the Hazardous
`#7E0023` loses its blue channel and reads as dark red, and at the 1/255 floor
the anchors collapse to primaries, where `#FF7E00` renders as pure red, a
category hotter than the air actually is.

If `0.03` is still too bright for your room, light fewer pixels rather than
lowering this. Total output falls with the pixel count while each lit pixel keeps
its full 8 bits.

`fail_threshold` is deliberately generous. The PA-II's web server drops requests
under no particular load, and a lamp that flips to the error pattern during
ordinary operation is worse than one showing a stale reading. Air quality does
not move that fast.

**It is counted in polls, not minutes.** So is the smoothing window below. If you
change the poll interval, rescale both or they silently change meaning.

---

## Home Assistant

**Required.** The brightness schedule depends on it.

The device is discovered over the network via the ESPHome API. Nothing needs to
be installed on the Home Assistant side, and the ESPHome add-on is not required.
This runs fine against Home Assistant Container, where add-ons do not exist.

It publishes:

| Entity | Notes |
|---|---|
| `AQI` | Computed here, not the sensor's own. |
| `PM2.5 Corrected` | Output of the Barkjohn correction, after smoothing. |
| `PM2.5 CF1 Channel A` | Raw channel A. |
| `PM2.5 CF1 Channel B` | Raw channel B. |
| `Sensor Humidity` | From the PA-II's onboard BME280. |
| `RSSI` | WiFi signal. |
| `Ring` | The light itself. |

The two PM2.5 channels are published separately on purpose. A PurpleAir has two
laser counters, and comparing them is how you notice one degrading. If channel A
and channel B drift apart over weeks, one of them is dying. PurpleAir sells
socketed replacements.

### What depends on Home Assistant, and how to drop it

Exactly one thing: the `sun_state` text sensor reads `sun.sun`, and every
brightness decision in the config tests it. Three places do this, all with the
same expression:

```cpp
id(sun_state).state == "above_horizon" ? ${bright_day}f : ${bright_night}f
```

If Home Assistant is absent or has not connected yet, that state is an empty
string and the test falls through to the night value. That is deliberate: erring
dim at boot is the right way round for an object that sits in a room. But it
also means a lamp with no Home Assistant is a permanently dim lamp.

To run without Home Assistant, replace the `text_sensor` block with ESPHome's
own [`sun`](https://esphome.io/components/sun.html) component and your
coordinates, then change those three expressions to test it. Note that this puts
your latitude and longitude in the config file, which is the reason this build
reads the sun from Home Assistant instead.

A `time`-based schedule works too and needs no coordinates, at the cost of not
tracking the seasons.

---

## The AQI math

The lamp does not use the sensor's own AQI numbers. The `pm2.5_aqi`,
`pm2.5_aqi_b`, `p25aqic` and `p25aqic_b` fields are uncorrected and use pre-2024
breakpoints. Expect this lamp to read lower than the sensor's own display, and
expect that to be correct.

### Step 1: average the channels

Mean of `pm2_5_cf_1` and `pm2_5_cf_1_b`. The `cf_1` values, not `atm`.

### Step 2: the Barkjohn correction

```
corrected = 0.524 × pm2_5_cf_1_avg − 0.0862 × humidity + 5.75
```

Clamped at 0. This is the US EPA's nationwide correction for PurpleAir sensors,
derived by Barkjohn et al. It matters most exactly when it matters most, which
is during wildfire smoke.

Use the sensor's humidity **raw**. The PA-II's own enclosure bias is already
baked into the fitted equation, so "correcting" humidity first makes the output
worse. The temperature field does read about 8 °F high, but temperature is not
an input here, so it does not matter.

This form is validated to roughly 343 µg/m³. There is a piecewise extension for
higher concentrations that this config does not implement.

### Step 3: smooth the concentration

A 3-sample moving average at roughly one poll every 6 minutes, so about an 18
minute window.

The window is counted in samples, so its length in minutes is set by the poll
interval. Leaving it at a larger sample count after slowing the poll would make
the lamp sluggish exactly during a smoke event, which is when it most needs to
move.

**Smooth the concentration, then compute AQI. Never the reverse.** The
concentration-to-AQI mapping is piecewise linear, so the mean of two AQI values
that straddle a breakpoint is not the AQI of the mean concentration.

### Step 4: 2024 EPA breakpoints

Standard piecewise linear interpolation:

```
AQI = (AQI_hi − AQI_lo) / (C_hi − C_lo) × (C − C_lo) + AQI_lo
```

| Concentration (µg/m³) | AQI | Category |
|---|---|---|
| 0.0 – 9.0 | 0 – 50 | Good |
| 9.1 – 35.4 | 51 – 100 | Moderate |
| 35.5 – 55.4 | 101 – 150 | Unhealthy for Sensitive Groups |
| 55.5 – 125.4 | 151 – 200 | Unhealthy |
| 125.5 – 225.4 | 201 – 300 | Very Unhealthy |
| 225.5 – 325.4 | 301 – 500 | Hazardous |

These are the May 2024 revised breakpoints. The older table put the
Good/Moderate boundary at 12.0 rather than 9.0. **If you find yourself writing
12.0, you have the wrong table.**

Concentration is truncated to one decimal before lookup, which is what closes
the gap between a band's top value and the next band's base. Above 325.4, AQI is
clamped at 500.

### Worked example

From a real payload: channels of 16.51 and 10.23, humidity 28%.

- Average: 13.37
- Corrected: 10.3 µg/m³
- AQI: **53**, Moderate

The sensor's own display read 61 for the same moment. That difference is
expected and correct.

---

## Enclosure

Not included here. The lamp is a ring, so any translucent diffuser over a 45 mm
ring will do.

If you adapt one of the Printables LED ring cases, note that most are cut for a
D1 mini and the controller cavity needs a remix for the SuperMini's 22.5 × 18 mm
footprint. Budget space for the 1000 µF capacitor, which is the largest single
object in the build and is easy to forget when you are sizing a pocket for the
board.

White PLA scatters hard and hides the individual LEDs but eats output.
Translucent transmits more and tends to show each LED as a hotspot. Either way,
retune `bright_day` and `bright_night` once you have a part you are happy with.

---

## Repository layout

| File | |
|---|---|
| `ashlight.yaml` | The entire device. |
| `secrets.yaml.example` | Template for the credentials file. |
| `PLAN.md` | Design notes and the phased build order. |
| `CLAUDE.md` | Working notes and constraints. |

`secrets.yaml` is gitignored and must stay out of the repository.

One privacy note if you publish anything about your own build: **the PurpleAir
JSON payload contains precise latitude and longitude.** Scrub it from pasted
output, screenshots and any sample data you commit.

---

## License

MIT. See `LICENSE`.
