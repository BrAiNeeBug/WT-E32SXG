# SX1262 wM-Bus Water Meter Gateway

ESPHome firmware that receives encrypted [wM-Bus](https://en.wikipedia.org/wiki/Meter-Bus) telegrams from water meters (tested with Diehl **Hydrus**) over an SX1262 radio, decodes them via the [esphome-components](https://github.com/SzczepanLeon/esphome-components) external component, and exposes the readings in Home Assistant. Includes a small ST7735 display showing live totals.

## Hardware

- **Heltec Wireless Tracker 1.1** (ESP32-S3 + SX1262) — or any ESP32-S3 board wired to an SX1262 module
- ST7735 SPI display (160x80), wired via a second SPI bus
- Optional: physical button on GPIO0 to wake the display

Pinout is defined in the YAML under `spi:` / `wmbus_radio:` / `display:` — adjust if your wiring differs.

## Features

- Reads up to two wM-Bus meters (easily extendable to more)
- Derived "difference" sensor (e.g. main meter minus sub-meter, for leak/wastewater calculation)
- Live values on the onboard display, auto-off after 60s, wake via button
- Runtime-adjustable log level (per tag) via a dropdown in the ESPHome web UI — no reflash needed to debug
- Battery life and RSSI diagnostics per meter

## Setup

### 1. Get your meter data

You need, per meter:
- the **meter ID** (serial number, often printed on the meter or extracted by sniffing a telegram)
- the **AES decryption key** (from the manufacturer, installer, or utility company)

### 2. Secrets

Enter your meter IDs and AES decryption keys in `secrets.yaml`.

### 3. Adjust the config

In `gateway-wmbus-heltec-tracker.yaml`:
- Set your radio `frequency` if your region/meters use something other than `868.95MHz`
- Rename the `sensor:` entries (`Meter 1`, `Meter 2`, `Difference Total`) to whatever makes sense for your setup
- If you only have one meter, delete the second `wmbus_meter` block and its sensors, and the `difference_total` template sensor
- For more than two meters, duplicate a `wmbus_meter` block + its `sensor:` entries

### 4. Flash

```bash
esphome run gateway-wmbus-heltec-tracker.yaml
```

First flash needs a USB connection; after that, OTA updates work via the `ota:` config.

### 5. Home Assistant

The device auto-integrates via the ESPHome integration once it's on your network (native API, encrypted via `api_key`).

## Local files

Your local `secrets.yaml` should not be committed to the repository. Make sure it is listed in `.gitignore`:

```gitignore
secrets.yaml
.esphome/
```

## Notes

- Log levels default to `NONE` at boot and can be raised per-tag (`wmbus`, `wmbusmeters`, `packet`) at runtime from the web UI — useful for debugging reception issues without needing verbose logs running permanently.
- `rx_gain: POWER_SAVING` trades a bit of sensitivity for lower power draw; switch to `BOOSTED` if you're having reception issues and power isn't a concern.
