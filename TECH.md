# hydro-monitor: technical plan

## Stack

- **Hardware:** Raspberry Pi 4 or 5, Raspberry Pi OS Bookworm 64-bit.
- **Sensors:** Atlas Scientific EZO pH, EZO EC (conductivity) and EZO RTD (temperature) circuits on an i4 InterLink shield, talking **I²C**. Hinged float switch on a GPIO pin.
- **Language:** Python 3.11 (the Bookworm system version). Don't use syntax newer than 3.11.
- **Libraries:**
  - `smbus2` for raw I²C to the EZO circuits.
  - `gpiozero` for the float switch. Use the `lgpio` pin factory, which works on both Pi 4 and Pi 5.
  - `paho-mqtt` 2.x for MQTT, using `CallbackAPIVersion.VERSION2`.
  - `PyYAML` for config.
  - Standard library `logging`, `dataclasses`, `argparse`, `json`. No web framework.
- **Tooling:** `pyproject.toml` with a hatchling build backend; `ruff` for lint and format; `pytest` for tests; GitHub Actions CI.
- **Runtime:** a long-running process managed by **systemd**.
- **Home Assistant:** runs on a separate device with an existing MQTT broker, such as the Mosquitto add-on. The Pi talks to Home Assistant only through MQTT.

## Architecture overview

One Python process, `hydro-monitor run`, with these components:

- **`config`** loads and validates `/etc/hydro-monitor/config.yaml` into typed dataclasses. Invalid config fails fast at startup with a clear message.
- **`ezo`** is a small driver for EZO circuits over I²C. `EzoDevice.command(cmd, wait_s)` writes an ASCII command, sleeps, reads the response, and parses the status byte. Subclasses or helpers exist for pH, EC and RTD (`read()`, `calibrate(...)`, `calibration_status()`).
- **`level`** reads the float switch through gpiozero and returns `water_low: bool` according to config (`low_when: closed|open`).
- **`mqtt`** wraps paho-mqtt. It handles connecting, reconnecting, the last-will message, publishing Home Assistant discovery, publishing state, and dispatching incoming command messages.
- **`discovery`** has pure functions that build Home Assistant discovery payloads. It's easy to unit test.
- **`calibration`** maps button commands, for example `ph_mid`, to EZO commands, runs them, and records the timestamp in the state file.
- **`service`** is the main loop. Every `read_interval_seconds` it reads each sensor, publishes values, and logs errors. A failure in one sensor never stops the others or the loop.
- **`cli`** is the entry point `hydro-monitor` with subcommands:
  - `--version`
  - `read` reads all sensors once and prints them. Useful for wiring checks.
  - `run` runs the service.

Data flow: sensors → I²C/GPIO → `service` → MQTT broker → Home Assistant. Calibration flows the other way: an HA button → MQTT command topic → `calibration` → EZO circuit.

I²C access is serialized with a single lock. Calibration commands and periodic reads must never overlap on the bus.

Design for later dosing control: keep hardware drivers and MQTT entity definitions in separate, data-driven modules so new actuators (switches, pumps) can be added as new entity types without restructuring.

### EZO I²C protocol notes

- EZO circuits ship in **UART mode** and must be switched to I²C mode once. This is a hardware setup step, documented in the setup guide.
- Default addresses: pH `0x63`, EC `0x64`, RTD `0x66`. They're configurable.
- EZO circuits don't use register addressing. Use `smbus2.i2c_msg.write` / `i2c_msg.read` with `SMBus.i2c_rdwr`. Read 31 bytes.
- Response byte 0: `1` success, `2` syntax error, `254` still processing, `255` no data. The rest is an ASCII string terminated by a null byte.
- Wait times: `R` about 900 ms for pH, about 600 ms for EC and RTD. Calibration commands about 900 ms (pH) and 600 ms (EC). Keep these as constants in one place.
- Commands used in v1:
  - Reading: `R`.
  - pH calibration: `Cal,mid,7.00`, `Cal,low,4.00`, `Cal,high,10.00`, `Cal,clear`, `Cal,?`.
  - EC calibration: `Cal,dry`, `Cal,low,<n>`, `Cal,high,<n>`, `Cal,clear`, `Cal,?`.
  - EC probe constant: `K,?` and `K,<n>`. Set K to 0.1 at startup if it differs from the config value.
  - Output selection: `O,?`. Only EC conductivity output is published in v1.
- Note that pH `Cal,mid` clears the low and high points. The setup guide must say to calibrate mid first.

## Data model

No database. The Pi holds two pieces of state.

### Config file

At `/etc/hydro-monitor/config.yaml`, mode 600, owned by the service user. Example in `config.example.yaml`:

```yaml
mqtt:
  host: homeassistant.local
  port: 1883
  username: null        # optional
  password: null        # optional
  tls: false
  client_id: hydro-monitor
  base_topic: hydro-monitor
  discovery_prefix: homeassistant
i2c:
  bus: 1
  ph_address: 0x63
  ec_address: 0x64
  rtd_address: 0x66
float_switch:
  gpio_pin: 17          # BCM numbering
  low_when: closed      # closed | open
read_interval_seconds: 60
calibration:
  ph_low: 4.00
  ph_mid: 7.00
  ph_high: 10.00
  ec_probe_k: 0.1
  ec_low: 84            # µS/cm
  ec_high: 1413         # µS/cm
state_file: /var/lib/hydro-monitor/state.json
log_level: INFO
```

### State file

At `/var/lib/hydro-monitor/state.json`. Write it atomically: write a temp file, then rename.

```json
{"ph_last_calibrated": "2026-10-18T12:00:00+00:00", "ec_last_calibrated": null}
```

### MQTT topics

`<base>` defaults to `hydro-monitor`.

| Topic | Payload | Retained |
|---|---|---|
| `<base>/status` | `online` / `offline` (offline is the last-will message) | yes |
| `<base>/sensor/ph` | float, 2 decimals | no |
| `<base>/sensor/ec` | float µS/cm | no |
| `<base>/sensor/temperature` | float °C, 2 decimals | no |
| `<base>/sensor/water_low` | `ON` / `OFF` | no |
| `<base>/sensor/ph_last_calibrated` | ISO 8601 timestamp, or empty | yes |
| `<base>/sensor/ec_last_calibrated` | ISO 8601 timestamp, or empty | yes |
| `<base>/sensor/ph_cal_points`, `<base>/sensor/ec_cal_points` | int from `Cal,?` | yes |
| `<base>/diag/...` | diagnostic values (uptime, read errors, last successful read) | no |
| `<base>/cmd/calibrate` | one of `ph_mid`, `ph_low`, `ph_high`, `ph_clear`, `ec_dry`, `ec_low`, `ec_high`, `ec_clear` | n/a, subscribed |
| `<base>/cmd/calibrate/result` | JSON `{"command": ..., "ok": bool, "message": str}` | no |

### Home Assistant discovery

- Discovery messages go to `<discovery_prefix>/<component>/hydro_monitor/<object_id>/config`. They're retained.
- All entities share one `device` block: `identifiers: ["hydro_monitor"]`, name "Hydroponics", manufacturer "Atlas Scientific / DIY".
- Entities:
  - Sensor `ph`: `device_class: ph`, `state_class: measurement`.
  - Sensor `ec`: `device_class: conductivity`, unit `µS/cm`, `state_class: measurement`.
  - Sensor `temperature`: `device_class: temperature`, unit `°C`, `state_class: measurement`.
  - Binary sensor `water_low`: name "Reservoir water low", `device_class: problem`.
  - Buttons for each calibration command, `entity_category: config`. Each button publishes its payload to `<base>/cmd/calibrate`.
  - Sensors `ph_last_calibrated` and `ec_last_calibrated`: `device_class: timestamp`, `entity_category: diagnostic`.
  - Calibration point counts and diagnostics: `entity_category: diagnostic`.
- Every entity uses `availability_topic: <base>/status`.
- Measurement sensors set `expire_after` to 3 × `read_interval_seconds`. If a single sensor fails repeatedly, it goes unavailable in HA while the others keep working.
- Discovery is republished on every (re)connect and whenever `homeassistant/status` receives `online`, which is HA's birth message.

## External services

- **MQTT broker**, the owner's existing one on the Home Assistant device. Optional username and password, and optional TLS.
- **Home Assistant**, through MQTT discovery and the automation blueprint. No HA API calls from the Pi.
- No cloud services.

## Repository layout

```
hydro-monitor/
├── pyproject.toml
├── README.md
├── PRODUCT.md
├── TECH.md
├── config.example.yaml
├── src/hydro_monitor/
│   ├── __init__.py          # __version__
│   ├── cli.py
│   ├── config.py
│   ├── ezo.py
│   ├── level.py
│   ├── discovery.py
│   ├── mqtt.py
│   ├── calibration.py
│   ├── state.py
│   └── service.py
├── tests/
│   ├── fakes.py             # FakeI2CBus, fake MQTT client
│   └── test_*.py
├── deploy/
│   ├── hydro-monitor.service
│   └── install.sh
├── blueprints/automation/hydro_monitor/
│   └── hydroponics_alerts.yaml
├── docs/
│   ├── hardware-setup.md
│   └── calibration.md
└── .github/workflows/ci.yml
```

## Testing approach

- Unit tests with `pytest` run in CI on `ubuntu-latest` with Python 3.11. No real hardware is needed in CI.
- **I²C:** `ezo.py` depends on a small bus interface. Tests use a `FakeI2CBus` that records written commands and returns scripted responses, including error codes 2, 254 and 255.
- **GPIO:** tests use gpiozero's `MockFactory`.
- **MQTT:**
  - Discovery payload builders are pure functions with snapshot-style assertions.
  - The MQTT wrapper is tested against a fake client object. There is no broker in CI.
- **Blueprint:** a test loads the YAML with a loader that tolerates `!input`, and checks the required inputs exist.
- **Hardware smoke test** is manual: `hydro-monitor read` on the Pi, documented in the setup guide.
- `ruff check` and `ruff format --check` must pass in CI.

## Deployment

- `deploy/install.sh`, run with sudo on the Pi:
  - Creates a system user `hydro-monitor` in groups `i2c` and `gpio`.
  - Creates a venv at `/opt/hydro-monitor/venv`, created with `--system-site-packages` so the system `lgpio` package is available, and pip-installs the package from the repo.
  - Creates `/etc/hydro-monitor/config.yaml` from the example if missing (mode 600) and `/var/lib/hydro-monitor/`.
  - Installs and enables the systemd unit.
- The script is idempotent. Rerunning it upgrades in place and never overwrites an existing config.
- systemd unit: `Restart=always`, `RestartSec=10`, `After=network-online.target`, `Wants=network-online.target`. Logs go to journald and are viewed with `journalctl -u hydro-monitor`.
- Prerequisites, documented: I²C enabled with `raspi-config`, EZO circuits switched to I²C mode, and `python3-lgpio` installed with apt.

## Open questions

- **Temperature compensation.** Should RTD readings feed pH and EC through `RT,<temp>` reads or `T,<temp>`? It's not in v1 until the owner confirms. It's tracked under the Later milestone.
- **Float switch.** Pin and polarity are unknown. Both are configurable, defaulting to BCM 17 and `closed`.
- **MQTT auth/TLS.** Both are supported through config. The real broker settings are unknown.
- **Smoothing.** v1 publishes raw single readings every 60 s. Averaging is deferred.
- **Flow Connector Board.** Its role next to the i4 InterLink shield is unconfirmed. It doesn't affect the software as long as the circuits are reachable over I²C.
