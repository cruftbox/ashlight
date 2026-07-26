# CLAUDE.md

## What this project is

Ashlight: an ESPHome device that polls a PurpleAir PA-II on the local network,
computes a corrected AQI, and displays it as a color on a 16-LED WS2812B ring.
The ESPHome `name:` should be `ashlight`, so it appears as `esphome.ashlight` in
Home Assistant.

Read `PLAN.md` for hardware, wiring, and the phase breakdown. Follow the phases
in order. Do not jump ahead to a full working config.

## Environment

- **Straylight**, Windows 11. The build and flash machine, and where this repo lives
  at `C:\Code\ashlight`. ESPHome CLI is installed here (2026.3.1). Build and flash
  with `esphome run ashlight.yaml`. No dashboard and no web flasher needed.
- **Wintermute**, Raspberry Pi 4. Runs Home Assistant as a plain Docker container
  (`homeassistant/home-assistant:stable`), config at
  `/home/michael/containers/ha/homeassistant/config`. This is HA Container, so there
  is no Supervisor and **add-ons cannot be installed**. The ESPHome Builder add-on is
  not available on this setup and there is no ESPHome instance anywhere on the LAN.
  Home Assistant still discovers the device over the network via the API, so nothing
  in the plan depends on having one.
- **Bandersnatch**, QNAP NAS running Container Station. Not used by this project.
- Target board: ESP32-C3 SuperMini. Use board `nologo_esp32c3_super_mini`, which sets
  variant `esp32c3` for you. Silicon confirmed by esptool: ESP32-C3 rev v0.4, 4MB
  flash, native USB Serial/JTAG, MAC `e8:3d:c1:83:59:fc`. It enumerates on COM8.
- PurpleAir PA-II at `<SENSOR_IP>` on the LAN. Ask for the IP if it isn't set yet.

## How to work with me

I have thirty years in broadcast and media engineering and I run a homelab. Skip
the explanations of what an IP address is. Do explain anything specific to
ESPHome, the ESP32-C3, or WS2812 timing, because that's where I'm newer.

- One phase at a time. Give me the config for the current phase only.
- Tell me what "done" looks like before I flash, so I can tell success from
  partial success.
- If something can fail in more than one way, say which failure looks like which.
- Push back if I'm about to do something in the wrong order.
- Dry and direct. No em dashes. No enthusiasm padding.
- When you're uncertain, say so plainly rather than hedging into vagueness.

## Verify ESPHome syntax against current docs

ESPHome's `http_request` component changed substantially in 2024. `capture_response`,
the `on_response` trigger, and the `json::parse_json` callback signature are all
different from older examples, and most blog posts and Stack Overflow answers you
will find use the obsolete form.

**Check the current ESPHome documentation before writing any config that touches
HTTP or JSON.** Do not write it from memory. If the installed ESPHome version
matters for a given feature, check what version the add-on is running first.

**The default response buffer is too small for this payload.** `http_request` caps
the captured body at a modest default. The PA-II `/json` response is several KB, so
it gets truncated, and truncated JSON surfaces as a parse failure rather than as an
obvious "response too large" error. Raise `max_response_buffer_size`. Confirm the
current option name and default in the docs, because both have moved.

Same applies to the LED platform. `neopixelbus`, `fastled_clockless`, and
`esp32_rmt_led_strip` have different support status on the C3 and this has
shifted over releases. Confirm which one is current and appropriate rather than
assuming.

## Technical constraints that are easy to get wrong

**GPIO.** Use GPIO3, 4, 5, or 10 for the LED data line. Avoid GPIO2, GPIO8, GPIO9,
which are strapping pins sampled at boot. GPIO8 typically also drives the onboard LED.

**The 330 Ω resistor goes at the ring end of the data run, not at the ESP32 end.**
Per Adafruit's NeoPixel Überguide. Its job is protecting the first pixel's input.

**Do not use the sensor's built-in AQI.** The fields `pm2.5_aqi`, `pm2.5_aqi_b`,
`p25aqic`, and `p25aqic_b` are uncorrected and use pre-2024 breakpoints. We compute
AQI ourselves.

**Field naming in the PurpleAir JSON is inconsistent.** AQI fields use dots
(`pm2.5_aqi`), particle fields use underscores (`pm2_5_cf_1`). Both are valid JSON
keys. This will bite in a lambda if you're not deliberate about it.

**Secrets.** WiFi credentials go in `C:\Code\ashlight\secrets.yaml`, next to
`ashlight.yaml`, referenced with `!secret`. Never inline them. That file is
gitignored; `ashlight.yaml` is not, and Phase 7 puts this repo on GitHub. The
PurpleAir payload contains precise lat/lon; keep it out of committed sample data,
blog screenshots, and pasted output.

**The sensor IP is not a secret.** It is a private LAN address. Put it in a
`substitutions:` block in `ashlight.yaml`, not in `secrets.yaml`. Secrets are for
things that would cause harm if published. Configuration that merely varies by site
belongs in substitutions, where it stays readable.

## The AQI math

### Step 1: average the channels

Mean of `pm2_5_cf_1` and `pm2_5_cf_1_b`. Use the cf_1 values, not atm.

Publish both channels as separate sensors too, so channel drift is visible. See
the channel A note in PLAN.md.

### Step 2: Barkjohn / EPA correction

```
corrected = 0.524 * pm2_5_cf_1_avg - 0.0862 * current_humidity + 5.75
```

Clamp the result at 0. This form is validated to roughly 343 µg/m³; there is a
piecewise extension for higher concentrations that we can add later if a smoke
event makes it relevant. Note the correction matters most exactly when it matters
most, which is during wildfire smoke, and this is Southern California.

### Step 3: 2024 EPA breakpoints for PM2.5

Standard piecewise linear interpolation:

```
AQI = (AQI_hi - AQI_lo) / (C_hi - C_lo) * (C - C_lo) + AQI_lo
```

| Concentration (µg/m³) | AQI | Category | Color |
|---|---|---|---|
| 0.0 – 9.0 | 0 – 50 | Good | `#00E400` |
| 9.1 – 35.4 | 51 – 100 | Moderate | `#FFFF00` |
| 35.5 – 55.4 | 101 – 150 | Unhealthy for Sensitive Groups | `#FF7E00` |
| 55.5 – 125.4 | 151 – 200 | Unhealthy | `#FF0000` |
| 125.5 – 225.4 | 201 – 300 | Very Unhealthy | `#8F3F97` |
| 225.5 – 325.4 | 301 – 500 | Hazardous | `#7E0023` |

These are the May 2024 revised breakpoints. The older table put the Good/Moderate
boundary at 12.0 rather than 9.0. If you find yourself writing 12.0, you have the
wrong table.

The table ends at 325.4. Above that, clamp AQI at 500 and stay in Hazardous. Write
that branch now rather than leaving it undefined. This is Southern California and
the project is named for ash, so it is reachable, not theoretical.

### The colors are gradient anchors, not discrete steps

As of Phase 5 the ring does not step between the six colors above. Each color is
anchored at the **midpoint** of its category and the ring interpolates between
anchors, so AQI 25 is pure green, AQI 75 is pure yellow, and the 50/51 boundary
is a 50/50 yellow-green. Anchoring at midpoints rather than category edges keeps
a category reading as itself when you are solidly inside it, while still showing
proximity to the next one.

| AQI | Anchor |
|---|---|
| 25 | `#00E400` Good |
| 75 | `#FFFF00` Moderate |
| 125 | `#FF7E00` Unhealthy for Sensitive Groups |
| 175 | `#FF0000` Unhealthy |
| 250 | `#8F3F97` Very Unhealthy |
| 400 | `#7E0023` Hazardous |

Below 25 and above 400 the color is flat rather than extrapolated.

**This replaced category hysteresis.** A continuous color has no category to flip
between, so the overshoot threshold described below is no longer needed. The only
damping left is the moving average on the corrected concentration.

### Known-good check

From the payload captured 2026-07-22: channels of 16.51 and 10.23, humidity 28.

- Average: 13.37
- Corrected: 10.3 µg/m³
- AQI: 53, Moderate

The sensor reported 61 for the same moment. That difference is expected and correct.

## Behavior requirements

**Hysteresis is not optional.** The house sits near the Good/Moderate boundary a
lot of the time. Without it the lamp will visibly flip between green and yellow
all afternoon. An ambient display should be calm; that is the entire point of the
object.

Phase 5 solved this with the gradient rather than with an overshoot threshold. A
continuous color cannot flip, so what remains is a moving average on the
corrected concentration, roughly 18 minutes. If the gradient is ever reverted to
discrete steps, the overshoot requirement comes back with it.

The window is `window_size: 3` and is counted in **samples**, so its length in
minutes is whatever the poll interval makes it. It was 8 samples at a 2 minute
poll, also about 16 to 18 minutes. Change the poll interval and this number has
to move with it or the lamp goes sluggish.

**Smooth in the right order.** Average the corrected concentration, then compute
AQI from that average. Do not average AQI values. The concentration-to-AQI mapping
is piecewise linear, so the mean of two AQI numbers that straddle a breakpoint is
not the AQI of the mean concentration. Apply hysteresis to the resulting category,
not to the AQI number.

**Brightness schedule.** 16 LEDs at full output is unpleasant at night. Sun-based
or time-based, dimmed after dark.

**Failure state.** Decide what the lamp does when the fetch fails or the sensor is
unreachable. It must be visually distinct from a valid reading. A slow dim pulse,
or off, not "hold the last color indefinitely" and certainly not a color that
means something in the AQI scale.

This covers the window between power-on and the first successful poll, which is up
to a full poll interval. Show the failure state during it. Not black, and not a
default color that could be read as a valid measurement.

**Poll interval.** Roughly 6 minutes, jittered to land 5 to 7 minutes apart.
Deliberately not 2, and the jitter is the point.

2 minutes matched the sensor's own averaging window, which was the original
reasoning and is why the config said 2 for six phases. It turned out to be the
wrong number for a different reason. The PA-II is a single-threaded ESP8266 that
uploads to its own cloud every 120 seconds and cannot serve the local web server
while it does. Two unsynchronised 120 second cycles drift slowly against each
other, so the failures did not scatter, they arrived in long runs. Measured
2026-07-26 over 2h29m: 55 consecutive successes, then 18 consecutive failures,
one clean transition, no reboot. A fresh random offset every cycle means the two
cycles can never settle into phase.

Freshness is not the constraint. This is an ambient lamp, not instrumentation.
Slower also cuts the request load on a sensor that is visibly struggling to
answer, from about 720 a day to 230.

**Anything counted in polls has to be rescaled when the interval moves.** Both
the moving average window and the failure threshold are in samples, not minutes,
so slowing the poll silently stretches them. See the two settings above.
