# Ashlight

An ambient light that shows outdoor air quality by color, driven by the PurpleAir
sensor already running at the house. Named for what it warns about: ash falling
during smoke season is when the lamp earns its keep. No cloud services, no API keys, no dependency
on anything outside the LAN.

## Hardware

| Item | Notes |
|---|---|
| ESP32-C3 SuperMini | Generic clone. Raw GPIO numbers on silkscreen. |
| WS2812B ring, 16 LED, 45mm | Confirm DI/DO direction before soldering. |
| 330 Ω resistor | Goes at the **ring** end of the data run, not the MCU end. |
| 1000 µF electrolytic, 6.3V+ | Across 5V/GND, close to the ring. Stripe = negative. |
| 5V USB supply, 2A | Headroom against brownout resets. |
| Wire, heat shrink, USB-A-to-C cable | A-to-C matters, see Phase 1. |

## Wiring

```
SuperMini 5V   ──────────────────────────────  Ring 5V
SuperMini GND  ──────────────────────────────  Ring GND
SuperMini GPIO3 ──────────────[330Ω]─────────  Ring DIN
                                       └─ resistor here, at the ring
1000 µF across Ring 5V / Ring GND, negative stripe to GND
```

Avoid GPIO2, GPIO8, GPIO9. They are strapping pins sampled at boot. GPIO8 also
usually drives the onboard LED on these clones.

## Data source

Local endpoint on the PA-II, no auth:

- `http://<SENSOR_IP>/json` — 2 minute average. **Use this one.**
- `http://<SENSOR_IP>/json?live=true` — instantaneous, jittery.

Sensor is a PA-II, firmware 7.02, hardware 2.0, dual PMSX003 laser counters plus
a BME280. Confirmed working payload captured 2026-07-22.

### Fields that matter

| Field | Use |
|---|---|
| `pm2_5_cf_1` | Channel A raw PM2.5. Input to the correction. |
| `pm2_5_cf_1_b` | Channel B raw PM2.5. |
| `current_humidity` | Input to the correction. Use raw. Do **not** offset it. |
| `current_temp_f` | Reads high, BME280 bakes inside the enclosure. Subtract ~8°F. |
| `rssi` | Log it. WiFi is the SuperMini's weak point. |

The temperature offset applies to temperature only. The Barkjohn correction was
derived against the PA-II's own onboard RH with the enclosure bias already baked
in, so "correcting" humidity before feeding the formula makes the output worse.
The two rows sit next to each other and the mistake is an easy one.

### Fields to ignore

- `pm2.5_aqi`, `pm2.5_aqi_b` — uncorrected and computed with pre-2024 breakpoints.
- `p25aqic`, `p25aqic_b` — the ready-made `rgb(r,g,b)` strings. Same problem.

Do the AQI math ourselves. See CLAUDE.md for the formulas.

## Known issue: channel A

At capture time channel A read 16.51 cf_1 against channel B's 10.23, a 6.3 µg/m³
spread. Channel A also showed 0.00 in both the 5.0µm and 10.0µm particle bins
while B saw particles there, and A's `pm10_0_cf_1` was bit-identical to its
`pm2_5_cf_1`. That pattern suggests a degrading or obstructed laser counter.

Not blocking. Log both channels separately from day one and watch the gap over a
few weeks. PurpleAir sells socketed replacement laser counters if A keeps drifting.

## Phases

Each phase has exactly one thing that can be wrong. Do not skip ahead.

### Phase 0 — done
Payload captured and field names confirmed.

### Phase 1 — bare board
Flash ESPHome with a minimal config. WiFi, api, ota, fallback AP, nothing else.
No LEDs attached, nothing soldered. Set the ESPHome `name:` to `ashlight` so it
lands cleanly in Home Assistant as `esphome.ashlight`.

Done when: device appears in the ESPHome dashboard and in Home Assistant, and
the RSSI sensor reports a value.

Gotcha: a known batch of SuperMini clones ships without the USB-C CC pulldown
resistors. With a C-to-C cable the board looks dead. Use A-to-C for the first
flash. If the flasher can't see it, hold BOOT, tap RESET, release BOOT.

### Phase 2 — LED proof of concept
Ring on clip leads or breadboard. No resistor, no capacitor, low brightness,
short wires. Add a `light:` block and drive it from the Home Assistant color
picker.

Done when: colors set from HA appear correctly on the ring.

Two failure modes here, and they look different:

- **Nothing lights at all.** Data direction or power. Confirm DI/DO before
  suspecting the config.
- **First pixel flickers, colors come out wrong, or it works on the bench and
  fails once the wires get longer in Phase 6.** Logic level. The C3 drives 3.3V
  and a 5V WS2812B wants roughly 0.7 x VDD, which is 3.5V, to read a logic high.
  Short wires and low brightness hide this, which is why it tends to show up late.
  Fix with a level shifter, or drop the ring's supply to ~4.3V through a series
  diode so its threshold comes down to meet the C3.

### Phase 3 — polling
Add the HTTP fetch and JSON parse. Publish `pm2_5_cf_1`, `pm2_5_cf_1_b`, and
`current_humidity` as template sensors. No AQI math yet, no color changes.

Done when: raw values in HA match what a browser shows at the same moment.

### Phase 4 — the math
Barkjohn correction, then 2024 EPA breakpoints, then a corrected AQI sensor.

Done when: the computed AQI is sane and differs from `pm2.5_aqi` in the expected
direction. Spot-check against the PurpleAir map, which applies the same correction.

### Phase 5 — behavior
Map AQI to color. Add hysteresis so the lamp doesn't strobe on a boundary, and a
brightness schedule so it isn't painful at 2am.

Done when: it has run a full day without visibly flickering between two colors.

### Phase 6 — permanence
Solder, print the enclosure, assemble. Only now.

The Printables models are cut for a D1 mini, so the controller cavity needs a
remix for the SuperMini footprint. STEP files are included on the "Ultimate 3D
printed case for LED Ring and D1mini" model.

### Phase 7 — write it up
Blog post at cruftbox.com, repo on GitHub.

## Before publishing anything

The PurpleAir JSON payload contains precise lat/lon. Scrub it from any pasted
output, screenshots, or committed sample data. WiFi credentials live in
`secrets.yaml` and stay out of the repo.
