Specification: Pstryk → GbbOptimizer Sync Tool (for Home Assistant Add-on)

### Overview

Create a Python service (runs inside this Home Assistant add-on) that periodically queries a Pstryk Energy Meter and publishes selected metrics to GbbOptimizer over MQTT.

### Objectives

- Query Pstryk meter every 4 minutes plus a random delay of 0–60 seconds.
- Extract forward and reverse active energy totals from the meter.
- Publish a JSON payload to GbbOptimizer via MQTT with required fields.
- Run continuously as a service in the add-on container with robust logging and error handling.

### Tech/Runtime

- Language: Python 3
- Runs inside this repository’s Home Assistant add-on boilerplate
- Long-running looped service

### Configuration (Add-on options)

Provide and read the following settings from add-on configuration:

1. `pstryk_address`: Pstryk Meter address. Default: `_pstryk._tcp` (requires mDNS resolution).
2. `gbb_mqtt_host`: GbbOptimizer MQTT broker hostname or IP. See the list in the GbbOptimizer manual: [Supported MQTT servers](https://gbboptimizer.gbbsoft.pl/Manual?PageNo=14).
3. `gbb_plant_id`: PlantID (use as MQTT username). Examples: A069, J420, B666
4. `gbb_secret_token`: Secret Token (use as MQTT password).
5. `log_level`: Verbosity, e.g., `INFO` (default), `DEBUG`, `WARNING`, `ERROR`.

The script must validate the presence/format of required options on startup and log clear errors if invalid.

### Constants and Structure

- Centralize constants in a single dictionary near the top of the main module (e.g., `CONST = { ... }`).
- Include keys for default intervals, endpoints, timeouts, MQTT topics, payload schema versions, etc.

### Pstryk Meter Integration

- If `pstryk_address` is an mDNS name (e.g., `_pstryk._tcp`), resolve it to an IP before use. Prefer standard Python libraries or lightweight mdns lookup; fall back to logging an error and retry if resolution fails.
- Query endpoint: `http://$METER_IP/state` using HTTP GET with a sensible timeout.
- Reference sample: `example-pstryk-response.json` in this repo.
- Extract fields:
  - Energy imported from grid (`fromgrid_total_kWh`): `.multiSensor.sensors[] | select(.type=="forwardActiveEnergy" and .id==0) | .value`
  - Energy exported to grid (`togrid_total_kWh`): `.multiSensor.sensors[] | select(.type=="reverseActiveEnergy" and .id==0) | .value`
- On any retrieval/parse issue, use `-1` for those values and log the reason.

### MQTT (GbbOptimizer)

- Connect to the MQTT broker using:
  - Host: `gbb_mqtt_host`
  - Username: `gbb_plant_id`
  - Password: `gbb_secret_token`
- Publish to MQTT topic: `${gbb_plant_id}/ha_gbb/sensor` a payload matching `example-gbb-payload.json` in this repo.
- Include these two mapped fields:
  - `fromgrid_total_kWh` ← Pstryk forwardActiveEnergy (id=0)
  - `togrid_total_kWh` ← Pstryk reverseActiveEnergy (id=0)
- If MQTT connection fails, log and retry on next cycle; do not crash the service.

### Scheduling

- Main loop cadence: every 4 minutes plus a random delay between 0 and 60 seconds.
- Jitter must be re-generated each cycle.
- Use graceful sleep so service can exit quickly on termination signals.

### Logging

- Respect `log_level` from configuration.
- Log at startup: configuration summary (omit secrets), resolved Pstryk IP, MQTT host, and schedule.
- Log each cycle: start, success (with values), failures (with reason), and publish result.

### Error Handling and Resilience

- Network and JSON parsing errors must not crash the service.
- On any Pstryk data issue, publish payload with `-1` values as specified.
- Use reasonable timeouts and retries (single attempt per cycle; rely on next cycle for retry).

### Deliverables (in this repo)

- Python service module(s) implementing the above behavior.
- Read configuration from add-on options (and/or environment if that’s the convention in this boilerplate).
- Use `CONST` dict for constants.
- Minimal dependencies; prefer standard library.
- AppArmor profile (see apparmor.txt file)

### Validation Aids

- Use `example-pstryk-response.json` to validate parsing paths.
- Use `example-gbb-payload.json` to validate publish format.

### Notes

- Keep the code clear and maintainable per clean code guidelines.
- Avoid deep nesting; prefer early returns.
- Do not log secrets.
