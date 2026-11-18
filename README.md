# SX1262 wM-Bus Watermeter Gateway

ESPHome firmware that receives encrypted [wM-Bus](https://en.wikipedia.org/wiki/Meter-Bus) telegrams from water meters (tested with Diehl **Hydrus**) over an SX1262 radio, decodes them via the [esphome-components](https://github.com/SzczepanLeon/esphome-components) external component, and exposes the readings in Home Assistant. Supports both the **Heltec Wireless Tracker** with its onboard display and the **Seeed Studio XIAO ESP32S3 + Wio-SX1262** without a display.

## Hardware

Two hardware variants are currently supported:

- **Heltec Wireless Tracker** (ESP32-S3 + SX1262 + 0.96" ST7735 display), tested on **V1.1** and **V1.2**. The display shows live meter totals and can be switched on with the onboard button.
- **Seeed Studio XIAO ESP32S3 + Wio-SX1262**. This compact variant has **no display**; it receives wM-Bus telegrams and makes the meter data available in Home Assistant.

Each board has its own pin configuration. Use the matching YAML file and do not copy the pinout from one board variant to the other.

## Files

| File | Description |
|------|-------------|
| `wmbus-heltec-tracker_ENG.yaml` | Config with English entity names and comments |
| `wmbus-heltec-tracker_DE.yaml` | Same config with German entity names and comments |
| [`wmbus-meter-x32sxg_DE.yaml`](https://github.com/BrAiNeeBug/WT-E32SXG/blob/cba27639d9d5e9c9ebd7d06bccbc7dbc6726bbaf/wmbus-meter-x32sxg_DE.yaml) | Seeed Studio XIAO ESP32S3 + Wio-SX1262, without display |

Both files are functionally identical. Use whichever you prefer; the rest of this README refers to the English one.

## Features

- Reads up to two wM-Bus meters (easily extendable to more)
- Derived "Wastewater" sensor (main meter minus garden meter, since garden water does not go into the sewer)
- Live values on the onboard display (Heltec variant only), auto-off after 60s, wake via button
- Runtime-adjustable log level (per tag) via a dropdown in the ESPHome web UI — no reflash needed to debug
- Battery life and RSSI diagnostics per meter
- Optional diagnostic sensors (WiFi signal, SSID, IP, uptime, clock), disabled by default

## Setup

### 1. Get your meter data

You need, per meter:
- the **meter ID** (serial number, often printed on the meter or extracted by sniffing a telegram)
- the **AES decryption key** (from the manufacturer, installer, or utility company)

### 2. Secrets

Create a `secrets.yaml` next to the config with these entries:

```yaml
wifi_ssid: "your-wifi"
wifi_password: "your-wifi-password"
ap_password: "fallback-ap-password"
webgui_password: "web-ui-password"
meter_htt11__encryption_key: "base64-api-encryption-key"
meter99_id: "12345678"
meter99_key: "00112233445566778899AABBCCDDEEFF"
meter15_id: "87654321"
meter15_key: "FFEEDDCCBBAA99887766554433221100"
```

`meter_htt11__encryption_key` is the ESPHome API encryption key (a 32-byte base64 string, e.g. generated at [esphome.io](https://esphome.io/components/api.html)).

### 3. Adjust the config

In the YAML file matching your hardware (for example `wmbus-heltec-tracker_ENG.yaml` for the Heltec or `wmbus-meter-x32sxg_DE.yaml` for the XIAO + Wio-SX1262):
- Set `localdomain` under `substitutions:` to your local DNS suffix (or remove it together with `domain:` under `wifi:`)
- Set your radio `frequency` if your region/meters use something other than `868.95MHz`
- Rename the `sensor:` entries (`Main Water`, `Garden Water`, `Wastewater`) to whatever makes sense for your setup
- If you only have one meter, delete the second `wmbus_meter` block and its sensors, and the `wastewater_total` template sensor (plus the `component.update` calls that reference it)
- For more than two meters, duplicate a `wmbus_meter` block + its `sensor:` entries

### 4. Flash

```bash
esphome run wmbus-heltec-tracker_ENG.yaml
```

First flash needs a USB connection; after that, OTA updates work via the `ota:` config (encrypted, ESPHome platform). The web server's own OTA upload is disabled.

### 5. Home Assistant

The device auto-integrates via the ESPHome integration once it's on your network (native API, encrypted with the API encryption key from your secrets). The web UI on port 80 is protected with digest authentication (user `admin`, password `webgui_password`).

## Local files

Your local `secrets.yaml` must not be committed to the repository. Make sure it is listed in `.gitignore`:

```gitignore
secrets.yaml
.esphome/
```

## Notes

- Log levels default to `NONE` at boot and can be raised per-tag (`wmbus`, `wmbusmeters`, `packet`) at runtime from the web UI — useful for debugging reception issues without needing verbose logs running permanently.
- `rx_gain: POWER_SAVING` trades a bit of sensitivity for lower power draw; switch to `BOOSTED` if you're having reception issues and power isn't a concern.
- The display-specific settings below apply only to the Heltec Wireless Tracker variant. The XIAO + Wio-SX1262 version has no display and does not need any display configuration.
- On the XIAO + Wio-SX1262, the radio uses the Wio-SX1262 B2B connector pins. The RX path needs `RXEN` (GPIO38) held HIGH; the configuration does this with an internal GPIO switch. `rf_switch: true` enables the SX1262's DIO2-controlled TX path. These board-specific settings are included in the [RX fix commit](https://github.com/BrAiNeeBug/WT-E32SXG/commit/cba27639d9d5e9c9ebd7d06bccbc7dbc6726bbaf).
- On the Heltec variant, the display uses the `mipi_spi` driver. The visible 80x160 panel sits inside the ST7735's larger 132x162 memory, so `offset_*` and `pad_*` values are required for a clean picture (no garbage pixels at the edges). Both the display backlight pin (GPIO21) and the display power pin (GPIO3) must be driven HIGH, which the config does at boot and on wake.
- Older ESPHome versions may not support `mipi_spi` or the `pad_width` / `pad_height` options. In that case use the deprecated `ili9xxx` platform with `swap_xy` / `mirror_x` and offsets 1 / 26 instead.

## Display
<img width="768" height="432" alt="image" src="https://github.com/user-attachments/assets/2542fe67-bebf-4a71-bfb5-64f51601acfd" />

