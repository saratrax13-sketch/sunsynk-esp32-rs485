# Sunsynk ESP32 RS485 Agent Guide

Use this guide for future development sprints in this repository.

## Current Baseline

- Active ESPHome file: `sunsynk-inverter-v4.19-readonly-registers.yaml`
- Previous daily driver: `sunsynk-inverter-v4.18.yaml`
- Earlier daily driver: `sunsynk-inverter-v4.17.yaml`
- Diagnostic only: `sunsynk-inverter-v4.16-drift-test.yaml`
- Dashboard cards live in `dashboard-cards/`.
- Home Assistant no longer needs Sunsynk template helpers when using v4.18 or newer.
- v4.19 adds read-only battery/AGM voltage diagnostics and does not change write behavior.

## User Preferences

- Keep changes conservative and versioned.
- Create a new YAML version for material changes instead of mutating the prior stable file.
- Keep revision notes at the top of each YAML file.
- Preserve existing Home Assistant entities/cards unless a migration is clearly discussed.
- For all new files in this project, use:

```yaml
ssid: !secret wifi_ssid2
password: !secret wifi_password2
```

## Hardware Baseline

Recommended RS485 hardware is the Waveshare RS485 Board (3.3V). The cheap MAX485-style board worked, but was less tolerant and needed slower timing.

Known wiring:

- Waveshare `VCC` -> ESP32 `3V3`
- Waveshare `GND` -> ESP32 `GND`, ideally Sunsynk RJ45 pin 3 GND
- Waveshare `DI` -> ESP32 `GPIO16`
- Waveshare `RO` -> ESP32 `GPIO17`
- Waveshare `RSE` -> ESP32 `GPIO18`
- Waveshare `A` -> Sunsynk RJ45 pin 2 A/D+
- Waveshare `B` -> Sunsynk RJ45 pin 1 B/D-

## Known-Good Timing

```yaml
modbus_update_interval: 5s
modbus_command_throttle: 100ms
modbus_send_wait_time: 250ms
```

Keep `writable_state_confirm_samples: "3"` for normal operation.

## Daily Driver Guardrails

Do not leave diagnostic drift-test values in daily driver files. Normal values are:

```yaml
generator_min_valid_power_w: "300"
generator_max_valid_power_w: "10000"
generator_confirm_samples: "3"
pv_string_max_power_w: "5500"
total_solar_max_power_w: "11000"
pv_current_min_when_power_a: "1"
inverter_power_max_w: "10000"
load_power_max_w: "10000"
load_power_spike_confirm_w: "5000"
```

Only loosen these in a file explicitly named as a drift/test version.

## Log Review

When reviewing logs, check:

- `[E]`, `No option found`, `CRC`, `timeout`, and Modbus errors.
- MQTT/safe_mode warnings near boot are usually not RS485 failures.
- `Overall State` should normally be `0002`; `Overall State Code` should be `2`.
- Writable states should not flicker after startup: `Use Timer`, `Solar Export`, `Grid Charge Enabled`, `Priority Load`, `Load Limit`, and `Prog* Charge`.
- PV power should match voltage/current trends, and PV1 + PV2 should match total solar.
- `Grid Current` is diagnostic and does not map cleanly to `Grid CT Power`; use `Grid CT Current Estimate` for CT-current comparison.

## ESPHome-Owned Dashboard Helpers

v4.18 and newer own these helper entities in ESPHome:

- `sensor.sunsynk_inverter_total_solar_power`
- `sensor.sunsynk_inverter_home_power_final`
- `sensor.sunsynk_inverter_non_essential_power`
- `sensor.sunsynk_inverter_ups_essential_power`
- `text_sensor.sunsynk_inverter_overall_state_text`
- `binary_sensor.sunsynk_inverter_grid_connected`
- `sensor.sunsynk_inverter_grid_ct_current_estimate`

Avoid reintroducing old Home Assistant template helpers:

- `sensor.sunsynk_total_pv_power`
- `sensor.sunsynk_ups_essential_final`
- `sensor.sunsynk_non_essential_final`
- `sensor.sunsynk_home_power_final`
- `sensor.sunsynk_overall_state_text`

## Validation

- Use `rg` for searches.
- Use `apply_patch` for manual edits.
- Run textual/static checks for entity names and substitutions.
- If local `esphome` is unavailable, say so and ask the user to compile in ESPHome Builder.
- Do not claim a compile passed unless ESPHome actually compiled the file.
