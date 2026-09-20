# dusty-air

A DIY indoor air quality sensor: a Nova Fitness **SDS011** laser dust sensor and a **DHT11**
temperature/humidity sensor read by an **ESP32-WROOM-32D**, running [ESPHome](https://esphome.io)
and reporting to Home Assistant over the encrypted native ESPHome API.

This is a hobby build published for reference. It works, but it is a proof of concept rather than
a product — there is no calibration story, no weatherproofing, and no long-term validation.

## Hardware

| Part | Notes |
|------|-------|
| Nova Fitness SDS011 | Laser scattering PM2.5 / PM10 sensor, UART output |
| DHT11 | Temperature and humidity, single-wire protocol |
| Espressif ESP32-WROOM-32D | Wi-Fi + BLE microcontroller, 3.3 V logic |
| Micro-USB cable and 5 V adapter | 500 mA or better |

A bare 4-pin DHT11 also needs a 10 kΩ resistor between DATA and VCC. Three-pin breakout modules
have it on board.

### Wiring

The SDS011 needs 5 V power but its UART lines are 3.3 V, so it connects directly to the ESP32 with
no level shifter.

| SDS011 pin | ESP32 pin |
|------------|-----------|
| 5V | 5V / VIN |
| GND | GND |
| TXD | GPIO16 (UART2 RX) |
| RXD | GPIO17 (UART2 TX) |

| DHT11 pin | ESP32 pin |
|-----------|-----------|
| VCC (+) | 3V3 |
| DATA (out / S) | GPIO4 |
| GND (-) | GND |

An interactive schematic of the wiring and data path is at
[`docs/dusty-air-schematic.html`](docs/dusty-air-schematic.html). Note that it still depicts the
retired MQTT topology — see [Status](#status).

## Setup

### 1. Install ESPHome

```bash
uv tool install esphome      # or: pipx install esphome
```

### 2. Create your secrets file

```bash
cp firmware/secrets.example.yaml firmware/secrets.yaml
```

Fill in the values. `secrets.yaml` is git-ignored and must never be committed. You need:

- **`wifi_ssid` / `wifi_password`** — the ESP32 is 2.4 GHz only; a 5 GHz-only SSID will not connect.
- **`fallback_password`** — at least 8 characters, for the recovery hotspot.
- **`api_key`** — generate with `openssl rand -base64 32`. Home Assistant asks for this same key
  when it adopts the device.
- **`ota_password`** — anything you like, required for over-the-air updates.

### 3. First flash, over USB

Connect the ESP32 by USB and run:

```bash
esphome run firmware/dusty-air.yaml
```

On macOS the board usually appears as `/dev/cu.usbserial-XXXX` (CP2102) or `/dev/cu.usbmodemXXXX`.
If ESPHome offers several options, pick the serial port rather than OTA for this first flash.

### 4. Adopt it in Home Assistant

Home Assistant discovers the device over mDNS. Go to **Settings → Devices & Services**; ESPHome
should offer `dusty-air`. Paste the `api_key` from step 2 when prompted. Seven entities are created:
PM2.5, PM10, temperature, humidity, Wi-Fi signal, uptime, and a status binary sensor.

If discovery does not fire, add the ESPHome integration manually using the device's IP address and
port 6053.

### 5. Later updates go over the air

```bash
esphome run firmware/dusty-air.yaml --device dusty-air.local
```

## Operating notes

- **Warm-up.** The SDS011 fan spins for ~30 s before a reading is trustworthy. The firmware sets
  `update_interval: 5min`, which drives the sensor's built-in duty cycle: fan on 30 s, read, sleep.
  At roughly 10% duty the rated 8000-hour fan life stretches to years rather than months.
- **Humidity skews readings.** The SDS011 over-reports above ~70% relative humidity because water
  droplets scatter light. Treat humid-air readings with suspicion rather than trying to correct them.
- **Self-heating.** If the boards share an enclosure, the DHT11 reads warmer and drier than the room.
  Give it ~10 minutes to equilibrate after handling before trusting temperature or humidity.
- **Mains power only.** Battery operation was evaluated and rejected; the fan makes long battery life
  impractical. The reasoning and the arithmetic are in [`docs/PLAN.md`](docs/PLAN.md).

## Debugging

```bash
# Over the air
esphome logs firmware/dusty-air.yaml --device dusty-air.local

# Over USB
esphome logs firmware/dusty-air.yaml --device /dev/cu.usbserial-0001
```

Sensor values are logged at **DEBUG** level. The shipped config uses `logger.level: INFO`, so
readings will *not* appear until you raise it:

```yaml
logger:
  level: DEBUG
```

## SDS011 serial protocol

9600 baud, 8N1, 3.3 V TTL. The sensor emits a ten-byte frame roughly once per second while active.

| Byte | Meaning |
|------|---------|
| 0 | `0xAA` header |
| 1 | `0xC0` command id |
| 2 | PM2.5 low byte |
| 3 | PM2.5 high byte |
| 4 | PM10 low byte |
| 5 | PM10 high byte |
| 6 | Device ID byte 1 |
| 7 | Device ID byte 2 |
| 8 | Checksum: sum of bytes 2–7, low 8 bits |
| 9 | `0xAB` tail |

```
PM2.5 (µg/m³) = ((byte3 << 8) | byte2) / 10
PM10  (µg/m³) = ((byte5 << 8) | byte4) / 10
```

ESPHome's built-in `sds011` platform handles all of this; the table is here for reference if you
write your own driver.

## Repository layout

- `firmware/` — ESPHome config. `dusty-air.yaml` is the device; `secrets.example.yaml` is the
  template for the git-ignored `secrets.yaml`.
- `docs/` — `PLAN.md` holds the design decisions and reasoning; the schematic renders the data path.
- `models/` — enclosure. `dusty-air.scad` is the source of truth; the STLs are rendered from it with
  OpenSCAD. Edit parameters in the `.scad` and re-render rather than hand-editing STLs.
- `dashboard/` — a retired standalone MQTT dashboard, kept for reference only (see below).

## Status

Working. Reports PM2.5, PM10, temperature, humidity, Wi-Fi signal, and uptime to Home Assistant over
the native ESPHome API.

Two things in this repo are deliberately stale:

- **`dashboard/index.html`** is a development-only page that spoke MQTT directly over WebSocket. The
  firmware moved from MQTT to the native API, which is a binary protocol a browser cannot consume,
  so the page is non-functional. It is kept for reference with a header comment explaining why. Read
  the sensors from Home Assistant instead.
- **`docs/dusty-air-schematic.html`** still draws the old MQTT data path (Mosquitto, discovery
  topics, WebSocket dashboard). The wiring half of it is still accurate.

The `mqtt:` block is left commented out in `firmware/dusty-air.yaml`. If you re-enable it, read the
comments there first — in particular, clear any retained discovery topics, or Home Assistant will
end up holding two copies of every entity.

## License

MIT — see [LICENSE](LICENSE).
