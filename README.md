# DTSU666-H meter emulator (ESPHome) — Victron → Huawei SUN2000

ESP32 firmware that emulates a **Chint DTSU666-H** smart meter (Modbus-RTU slave) on the
RS485 bus of a **Huawei SUN2000** inverter. The grid values are sourced from a **Victron
Cerbo GX** over its built-in MQTT broker — so the inverter's built-in **export limitation**
works without installing a separate physical meter.

In other words: the inverter thinks it is talking to a real Chint grid meter; in reality an
ESP32 mirrors the grid power that Victron already measures.

> **Status:** the config **builds under ESPHome 2026.5.1** (`config` valid + `compile`
> green) and has been validated on a real **SUN2000-15KTL-M2**: meter identified, sign/scale
> correct, 8N1 frame confirmed. See [`docs/reference/huawei-dtsu666-meter.md`](docs/reference/huawei-dtsu666-meter.md)
> for the register map and the reverse-engineered protocol details.

## How it works

- The inverter is the Modbus master and polls the meter (slave address **11 / 0x0B**, 9600 8N1).
- On startup the inverter probes register `0x07D1` (handshake) → the emulator answers `0x3B11`
  ("meter active"), then the inverter polls the data block `0x0836` (40×float32, engineering
  units) and the parameter block `0x08A6`.
- The firmware keeps a live snapshot of the grid (power, voltages, currents, frequency,
  power factor, energy) fed from Victron MQTT and serves it from those registers.
- **Sign:** `+` = import from grid, `−` = export. This matches the Victron convention, so no
  inversion is needed.
- **Scale:** values are float32 in plain engineering units (A, V, Hz, W, var, VA, cos φ, kWh)
  — no divisors, no ×10.
- **Fail-safe:** if the Victron feed goes stale (default 30 s) the emulator mutes Modbus
  (`disable_loop()`), so the inverter falls back to its own derating instead of acting on
  stale data. Set the inverter's "output limit for fail-safe" to 0% accordingly.

Full protocol notes, register table, sign/scale reasoning and the donor projects this is
based on: [`docs/reference/huawei-dtsu666-meter.md`](docs/reference/huawei-dtsu666-meter.md)
and [`docs/reference/sources.md`](docs/reference/sources.md).

## Tested hardware

Validated on **two Waveshare ESP32-S3 boards**, both on a Huawei **SUN2000-15KTL-M2** with a
**Victron Cerbo GX** as the data source, ESPHome **2026.5.1**:

| Board | Config | Notes |
|-------|--------|-------|
| **Waveshare ESP32-S3-RS485-CAN** | `dtsu666-emulator.yaml` | Isolated RS485, DIN-rail, 7–36 V. The plain meter emulator. |
| **Waveshare ESP32-S3-Relay-1CH** | `dtsu666-relay.yaml` | Same on-board RS485 + a 1-channel relay (CH1=GPIO47) used to switch an inverter cooling fan from Home Assistant. The author's production board. |

Both boards expose the same RS485 transceiver pinout, so the metering part is identical and
shared via `packages/meter_core.yaml`.

## RS485 wiring

ESP32-S3 (both boards) UART ↔ on-board RS485 transceiver:

| Signal | GPIO |
|--------|------|
| UART TX | `GPIO17` |
| UART RX | `GPIO18` |
| DE/RE flow control | `GPIO21` |

RS485 A/B to the inverter's COM port (SUN2000-M2, Table 5-3):

| Transceiver | Inverter COM |
|-------------|--------------|
| A (+) | `485A2` — pin 7 |
| B (−) | `485B2` — pin 9 |

After wiring, add the meter in the inverter's app (FusionSolar / installer quick-setup) as a
**DTSU666-H** power meter; identification takes a few minutes.

## Repository layout

- `dtsu666-emulator.yaml` / `dtsu666-relay.yaml` — entry points: substitutions (+ a gpio
  switch for the relay variant) and `packages`.
- `packages/meter_core.yaml` — shared part of both devices: esp32/wifi/api/ota/logger/
  web_server, uart + modbus (server), RS485 traffic counters.
- `packages/meter_registers.yaml` — snapshot globals + the `modbus_server` register map:
  handshake `0x07D1`→`0x3B11`, data block `0x0836` (40×FP32), parameter block `0x08A6`.
- `packages/victron_mqtt.yaml` — subscription to the Victron grid topics
  (`N/<portalID>/grid/<instance>/Ac/...`) + keepalive every 30 s.
- `packages/failsafe_ha.yaml` — freshness watchdog + fail-safe (mutes Modbus via
  `disable_loop()` when data goes stale) + Home Assistant entities.
- `secrets.yaml.example` → copy to `secrets.yaml` (gitignored).

## ESPHome version

Pinned to **2026.5.1** (= `ghcr.io/esphome/esphome:stable` as of 2026-05-31). The
`modbus_server` component is version-sensitive — re-check `config`/`compile` on any version bump.

## Build & flash (Docker; run from the repo directory)

```bash
cp secrets.yaml.example secrets.yaml          # fill in your values
IMG=ghcr.io/esphome/esphome:2026.5.1
docker run --rm -v "$PWD":/config "$IMG" config  dtsu666-emulator.yaml   # -> Configuration is valid!
docker run --rm -v "$PWD":/config "$IMG" compile dtsu666-emulator.yaml   # -> Successfully compiled program

# OTA flash (device already on the network):
docker run --rm -v "$PWD":/config "$IMG" upload dtsu666-emulator.yaml --device <device-ip>

# First USB flash on macOS: Docker Desktop does NOT pass through USB-serial
# (`--device=/dev/ttyUSB0` does not work). Working path — compile in Docker, then flash the
# factory image from the host:
#   pip install esptool
#   esptool --port /dev/cu.usbmodem* write-flash 0x0 \
#     .esphome/build/dtsu666-emulator/.pioenvs/dtsu666-emulator/firmware.factory.bin
```

Use `dtsu666-relay.yaml` instead if you are on the relay board.

## Configuration (substitutions)

Field values confirmed on hardware:

| Substitution | Purpose | Value | Note |
|--------------|---------|-------|------|
| `meter_address` | slave address on the bus | `11` | confirmed on the inverter |
| `handshake_code` | reply to 0x07D1 | `0x3B11` | `0x3F80` on older meter firmware |
| `uart_tx_pin`/`uart_rx_pin` | UART pins to the transceiver | `GPIO17`/`GPIO18` | Waveshare ESP32-S3 (both boards) |
| `rs485_de_re_pin` | transceiver DE/RE pin | `GPIO21` | Waveshare |
| `uart_stop_bits` | stop bits | `1` (8N1) | confirmed on the inverter; `2` is the fallback |
| `victron_mqtt_host` | Cerbo GX MQTT broker host | `!secret victron_mqtt_host` | set in `secrets.yaml` |
| `victron_portal_id` | VRM portal ID / Cerbo GX serial | `!secret victron_portal_id` | set in `secrets.yaml` |
| `victron_grid_instance` | grid-meter device instance | `0` | from the live broker |
| `freshness_timeout_ms` | staleness threshold + boot grace | `30000` | over → fail-safe; at boot → defaults window |
| `param_2217/2220/2221/2223` | 0x08A6 fallback | `0`/`0`/`0`/`0` | if identification stalls: `0`/`3`/`11`/`1` |
| `relay_pin` | cooling relay (relay variant only) | `GPIO47` | Waveshare demo code, HIGH=ON |

Secrets in `secrets.yaml` (see `secrets.yaml.example`): `wifi_ssid`, `wifi_password`,
`ota_password`, `ap_fallback_password`, `victron_mqtt_host`, `victron_portal_id`,
(optional) `api_encryption_key`, `victron_mqtt_user`/`victron_mqtt_password`.

## Credits

Built on the reverse-engineering work of several projects (full list with links in
[`docs/reference/sources.md`](docs/reference/sources.md)) — notably
[`salakrzy/DTSU666_CHINT_to_HUAWEI_translator`](https://github.com/salakrzy/DTSU666_CHINT_to_HUAWEI_translator),
[`Vuko791/sun2000-dtsu666h-sniffer`](https://github.com/Vuko791/sun2000-dtsu666h-sniffer) and
[`pazzero/esphome-dtsu666h-modbus-proxy`](https://github.com/pazzero/esphome-dtsu666h-modbus-proxy).

## License

[MIT](LICENSE).
