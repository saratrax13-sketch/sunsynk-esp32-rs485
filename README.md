# Sunsynk ESP32 RS485

ESPHome configuration for monitoring and controlling a Sunsynk 8kW single-phase inverter over RS485 using an ESP32 and Home Assistant.

Current active file:

```text
sunsynk-inverter-v4.18.yaml
```

This project was built around:

- Sunsynk 8kW single-phase inverter
- ESP32 + RS485 transceiver
- ESPHome
- Home Assistant
- MQTT and native ESPHome API

The configuration can read live inverter data and write selected inverter settings from Home Assistant.

## What This Does

The ESP32 connects to the inverter RS485 port and exposes Sunsynk Modbus registers to Home Assistant.

It provides:

- Battery SOC, voltage, current, power, and temperature
- PV1/PV2 solar power, voltage, and current
- Grid voltage, current, frequency, CT power, and grid power
- Inverter voltage, current, frequency, and inverter power
- Load power and UPS/essential load power
- Calculated total solar, home power, non-essential power, and essential load values for dashboard cards
- Generator/AUX power when the AUX port is configured as generator input
- Daily PV, battery charge/discharge, grid import/export, and load energy totals
- Inverter state code, raw state text, and friendly state text
- ESP32 diagnostic entities for Wi-Fi/IP/status/firmware version
- Writable inverter settings:
  - Solar Export
  - Use Timer
  - Priority Load
  - Grid Charge Enabled
  - Load Limit
  - Program charge source settings
  - Program time settings
  - Program power/capacity settings
  - Battery shutdown/restart/low capacity settings

## Why The YAML Uses Isolated Modbus Reads

This inverter behaved badly when ESPHome grouped too many nearby Modbus registers into one range read. The symptoms were shifted/corrupt values, for example:

- Battery SOC showing impossible values
- Battery voltage receiving inverter power bytes
- Timer/program values showing as large invalid values
- `not enough data for value` errors in the ESPHome logs

To make reads stable, this version uses `force_new_range: true` broadly. That makes ESPHome read most items in separate requests. It is slower, but it avoids corrupt grouped reads.

v4.14 used a faster 5s scan interval with stable 750ms RS485 request spacing. Earlier tests showed that 600ms could bring back shifted/corrupt values on the older RS485 board, while very conservative v4.2 timing was too slow in live use. It also filters impossible live values so dashboards keep the last valid reading instead of publishing corrupt spikes.

PV1/PV2 power values are checked against their voltage x current values and configurable PV string/total-solar limits. This rejects obvious shifted-register reads, such as PV power briefly showing another register value while the PV voltage and current indicate a different output.

Battery, grid, inverter, load, UPS/essential, and calculated power sensors also use sanity filters. v4.11 added a PV current cross-check, invalid state-code filtering, non-negative inverter power filtering, and two-sample confirmation for sudden high load/UPS readings.

v4.12 also confirms writable setting state reads before publishing them. This prevents a single shifted Modbus response from making `Use Timer`, `Solar Export`, `Grid Charge Enabled`, `Priority Load`, or `Load Limit` appear to change in Home Assistant when the inverter was not physically switched.

v4.13 fixes the PV current compile substitution and applies the same confirmation pattern to `Battery Shutdown Capacity`, `Battery Restart Capacity`, and `Battery Low Capacity` number reads.

v4.14 adds confirmed reads to `Prog1 Charge` through `Prog6 Charge`, preventing shifted values such as `100` from producing select mapping errors.

v4.15 moved to a Waveshare 3.3V RS485 board and 250ms request spacing after the cheap MAX485 module proved less tolerant.

v4.16 was a short-lived drift-test file with intentionally loose validation limits. It is not intended for normal dashboards.

v4.17 promoted the clean Waveshare test timing to the daily-driver settings: 5s scans, 100ms command throttle, and 250ms send wait. It also added `Grid CT Current Estimate`, a calculated current equivalent for `Grid CT Power`.

v4.18 moves the remaining Home Assistant dashboard helpers into ESPHome. Home Assistant no longer needs Sunsynk template sensors in `configuration.yaml` for this setup.

The generator/AUX power sensor uses a consecutive-sample confirmation filter. This lets a real generator appear automatically once it produces sustained valid power, while one-off shifted Modbus values are ignored when no generator is connected.

The current Modbus timing is controlled near the top of the YAML:

```yaml
modbus_update_interval: 5s
modbus_command_throttle: 100ms
modbus_send_wait_time: 250ms
```

Slower-changing values use `skip_updates`:

| Setting | Approximate polling interval |
| --- | --- |
| no `skip_updates` | 5 seconds |
| `skip_updates: 5` | 30 seconds |
| `skip_updates: 11` | 60 seconds |
| `skip_updates: 59` | 5 minutes |
| `skip_updates: 119` | 10 minutes |

Modbus is serial, so the values are read one after another. Dashboard values can therefore be a few seconds apart during one full scan.

Diagnostic Wi-Fi/IP/status/version entities do not use RS485 and do not change the inverter read timing.

## Important MQTT Note

ESPHome MQTT writable entity state messages can be retained by default. Retained MQTT state caused Home Assistant to show old writable states being replayed after reconnects.

For this reason, writable inverter switches, selects, and numbers explicitly use:

```yaml
retain: false
```

on the writable inverter controls, including:

- Solar Export
- Use Timer
- Priority Load
- Grid Charge Enabled
- Load Limit
- Program charge source settings
- Program time settings
- Program power/capacity settings
- Battery capacity settings

If you previously used retained writable topics, clear the retained MQTT messages from your broker.

## Hardware Wiring

### ESP32 To RS485 Converter

The YAML uses these pins:

```yaml
tx_pin: GPIO16
rx_pin: GPIO17
flow_control_pin: GPIO18
```

Wire the ESP32 to the RS485 converter like this:

| ESP32 pin | RS485 converter pin | Purpose |
| --- | --- | --- |
| GPIO16 | DI / TXD / DIN | ESP32 UART TX to RS485 driver input |
| GPIO17 | RO / RXD / R0 | RS485 receiver output to ESP32 UART RX |
| GPIO18 | DE and RE tied together | Direction/flow control |
| GND | GND | Common ground |
| 3V3 or 5V | VCC | Depends on your RS485 module |

Notes:

- Many MAX485-style modules expose separate `DE` and `/RE` pins. Tie them together and connect both to GPIO18.
- Use 3.3 V logic-safe RS485 modules with ESP32 boards.
- Some RS485 adapters label A/B differently. If reads fail completely, swap A and B.
- A 120 ohm termination resistor across A/B may help if the cable is long or noisy.

### RS485 Converter To Sunsynk Inverter RJ45

For the Sunsynk/Deye RS485 RJ45 pinout, using a T568B-crimped cable:

| Inverter RJ45 pin | T568B color | RS485 signal |
| --- | --- | --- |
| 1 | White/Orange | B / D- |
| 2 | Orange | A / D+ |
| 3 | White/Green | GND |

Most installs only need pins 1 and 2. Connect GND if your RS485 adapter and installation require it.

### RJ45 / T568B Color Reference

Full T568B order:

| RJ45 pin | T568B color |
| --- | --- |
| 1 | White/Orange |
| 2 | Orange |
| 3 | White/Green |
| 4 | Blue |
| 5 | White/Blue |
| 6 | Green |
| 7 | White/Brown |
| 8 | Brown |

## Board Pinout Layouts

This configuration is written around GPIO labels. Always confirm the silk-screen on your exact board before wiring, because clone boards can move physical header positions.

### ESP32-WROOM-32E DevKitC V4, 38-pin

This is the Espressif ESP32-DevKitC V4 style board fitted with an ESP32-WROOM-32E module.

The YAML uses:

```yaml
esp32:
  board: esp32dev
  framework:
    type: esp-idf
```

Typical 38-pin DevKitC-style header layout:

| Left header, top to bottom | Notes |
| --- | --- |
| 3V3 | 3.3 V output |
| EN | Enable/reset |
| VP / GPIO36 | Input-only ADC |
| VN / GPIO39 | Input-only ADC |
| GPIO34 | Input-only ADC |
| GPIO35 | Input-only ADC |
| GPIO32 | GPIO/ADC |
| GPIO33 | GPIO/ADC |
| GPIO25 | GPIO/DAC |
| GPIO26 | GPIO/DAC |
| GPIO27 | GPIO |
| GPIO14 | GPIO |
| GPIO12 | GPIO, strapping pin |
| GND | Ground |
| GPIO13 | GPIO |
| GPIO9 / SD2 | Flash pin on many boards, avoid |
| GPIO10 / SD3 | Flash pin on many boards, avoid |
| GPIO11 / CMD | Flash pin on many boards, avoid |
| 5V / VIN | 5 V input/output depending on board |

| Right header, top to bottom | Notes |
| --- | --- |
| GND | Ground |
| GPIO23 | GPIO / VSPI MOSI |
| GPIO22 | GPIO / I2C SCL commonly used |
| TX0 / GPIO1 | USB serial TX |
| RX0 / GPIO3 | USB serial RX |
| GPIO21 | GPIO / I2C SDA commonly used |
| GND | Ground |
| GPIO19 | GPIO / VSPI MISO |
| GPIO18 | RS485 flow control in this project |
| GPIO5 | GPIO / strapping pin |
| GPIO17 | RS485 RX in this project |
| GPIO16 | RS485 TX in this project |
| GPIO4 | GPIO |
| GPIO0 | Boot/strapping pin |
| GPIO2 | GPIO / strapping pin |
| GPIO15 | GPIO / strapping pin |
| GPIO8 / SD1 | Flash pin on many boards, avoid |
| GPIO7 / SD0 | Flash pin on many boards, avoid |
| GPIO6 / CLK | Flash pin on many boards, avoid |

Project wiring on this board:

| Project function | GPIO | Common label |
| --- | --- | --- |
| UART TX to RS485 | GPIO16 | IO16 |
| UART RX from RS485 | GPIO17 | IO17 |
| RS485 DE/RE flow control | GPIO18 | IO18 |
| Ground | GND | GND |
| Power to RS485 board | 3V3 or 5V | 3V3 / 5V |

### Ai-Thinker NodeMCU-32-S2 / ESP-12K, 42-pin

This is the Ai-Thinker NodeMCU-32-S2 development board based on the ESP-12K ESP32-S2 module.

If you use the ESP32-S2 board, change the board setting to the correct ESPHome board for your hardware, for example:

```yaml
esp32:
  board: nodemcu-32s2
  variant: esp32s2
  framework:
    type: esp-idf
```

ESP-12K module pin definition. The development board header order may differ, so use this as the GPIO reference and check the board silk-screen.

| ESP-12K pin | Signal |
| --- | --- |
| 1 | GND |
| 2 | 3V3 |
| 3 | EN |
| 4 | GPIO0 |
| 5 | GPIO1 |
| 6 | GPIO2 |
| 7 | GPIO3 |
| 8 | GPIO4 |
| 9 | GPIO5 |
| 10 | GPIO6 |
| 11 | GPIO7 |
| 12 | GPIO8 |
| 13 | GPIO9 |
| 14 | GPIO10 |
| 15 | GPIO11 |
| 16 | GPIO12 |
| 17 | GPIO13 |
| 18 | GPIO14 |
| 19 | GPIO15 |
| 20 | GPIO16 |
| 21 | GPIO17 |
| 22 | GPIO18 |
| 23 | GPIO19 / USB D- |
| 24 | GPIO20 / USB D+ |
| 25 | GPIO21 |
| 26 | GPIO26 |
| 27 | GPIO33 |
| 28 | GPIO34 |
| 29 | GPIO35 |
| 30 | GPIO36 |
| 31 | GPIO37 |
| 32 | GPIO38 |
| 33 | GPIO39 |
| 34 | GPIO40 |
| 35 | GPIO41 |
| 36 | GPIO42 |
| 37 | GPIO43 / U0TXD |
| 38 | GPIO44 / U0RXD |
| 39 | GPIO45 |
| 40 | GPIO46 |
| 41 | GND |
| 42 | GND |

Project wiring on this board:

| Project function | ESP32-S2 GPIO | Notes |
| --- | --- | --- |
| UART TX to RS485 | GPIO16 | General-purpose IO on ESP32-S2 |
| UART RX from RS485 | GPIO17 | General-purpose IO on ESP32-S2 |
| RS485 DE/RE flow control | GPIO18 | General-purpose IO on ESP32-S2 |
| Native USB | GPIO19/GPIO20 | Keep free if using native USB |
| Ground | GND | Common ground |
| Power to RS485 board | 3V3 or 5V | Match your RS485 module |

## Inverter Settings

On the inverter, check:

- Modbus ID: `1`
- Baud rate: `9600`
- RS485 wiring connected to the correct inverter RS485/Meter/BMS port for your setup

The YAML uses:

```yaml
uart:
  baud_rate: 9600

modbus_controller:
  address: 1
```

## Home Assistant Configuration

With `sunsynk-inverter-v4.18.yaml`, Sunsynk dashboard helper values are created by ESPHome. Home Assistant no longer needs a Sunsynk `template:` block for this project.

The included example file is:

```text
home-assistant-configuration.yaml
```

Minimal Home Assistant example:

```yaml
# Loads default set of integrations. Do not remove.
default_config:

# Load frontend themes from the themes folder.
frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml
```

ESPHome now provides the dashboard helper entities directly:

| Entity | Purpose |
| --- | --- |
| `sensor.sunsynk_inverter_total_solar_power` | Adds PV1 and PV2 power together |
| `sensor.sunsynk_inverter_home_power_final` | Card-friendly total home/load power |
| `sensor.sunsynk_inverter_non_essential_power` | Calculates non-essential load as total load minus UPS/essential load |
| `sensor.sunsynk_inverter_ups_essential_power` | UPS/essential load register from the inverter |
| `text_sensor.sunsynk_inverter_overall_state_text` | Converts raw inverter state into readable text |
| `binary_sensor.sunsynk_inverter_grid_connected` | Grid connected status from the inverter |
| `sensor.sunsynk_inverter_grid_ct_current_estimate` | Estimated CT current from `abs(Grid CT Power) / Grid Voltage` |

After flashing v4.18 and confirming these entities exist, remove the old Sunsynk template helpers from Home Assistant to avoid duplicate/legacy entities in cards.

## Dashboard Cards

Two dashboard examples are included:

| File | Card type | Notes |
| --- | --- | --- |
| `dashboard-cards/sunsynk-power-flow-card.yaml` | `custom:sunsynk-power-flow-card` | Best for a Sunsynk-style detailed inverter diagram |
| `dashboard-cards/power-flow-card-plus.yaml` | `custom:power-flow-card-plus` | Simpler generic power-flow card |

Install the matching custom card through HACS before pasting the YAML into a Lovelace dashboard. The card examples are written for v4.18 ESPHome-owned entities, so they do not depend on Home Assistant template helpers. If Home Assistant adds `_2` to any entity ID, update the card YAML to match your actual entity IDs.

## Files

| File | Purpose |
| --- | --- |
| `sunsynk-inverter-v4.18.yaml` | Current main ESPHome configuration |
| `sunsynk-inverter-v4.17.yaml` | Previous daily-driver version before ESPHome-owned dashboard helpers |
| `sunsynk-inverter-v4.16-drift-test.yaml` | Diagnostic drift-test version with loose validation limits; not for normal dashboards |
| `home-assistant-configuration.yaml` | Minimal Home Assistant configuration example with no Sunsynk templates |
| `secrets.example.yaml` | Example ESPHome secrets file |
| `dashboard-cards/sunsynk-power-flow-card.yaml` | Detailed Sunsynk dashboard card example |
| `dashboard-cards/power-flow-card-plus.yaml` | Generic power-flow dashboard card example |
| `.gitignore` | Prevents local secrets/build files being committed |

## Setup

1. Copy `sunsynk-inverter-v4.18.yaml` into ESPHome.
2. Create a `secrets.yaml` based on `secrets.example.yaml`.
3. Update Wi-Fi, MQTT, API, OTA, fallback AP, static IP, and web server secrets.
4. Update the static IP secrets or remove the `manual_ip` block from the YAML.
5. Check your board type and GPIO wiring.
6. Compile and flash from ESPHome Builder.
7. Confirm the v4.18 helper entities appear in Home Assistant.
8. Remove old Sunsynk template helpers from Home Assistant if you previously used them.
9. Add one of the dashboard cards from `dashboard-cards`.
10. Watch logs for sane values.

Useful log checks:

- Battery SOC should be a realistic percentage.
- Battery voltage should be realistic for your battery bank.
- Grid frequency should be around 50 Hz or 60 Hz depending on your region.
- PV voltage/current should match daylight conditions.
- No repeated `not enough data for value` errors.

## Safety

- Inverters and distribution boards contain dangerous voltages.
- Do not work inside the inverter or electrical panel unless you are qualified.
- RS485 is low voltage, but it is connected to equipment that may be near hazardous wiring.
- Do not plug the inverter RS485 RJ45 into normal Ethernet network equipment.

## References

- Sunsynk/Deye RS485 wiring reference: https://kellerza.github.io/sunsynk/guide/wiring
- ESP32-DevKitC V4 official Espressif documentation: https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32/esp32-devkitc/user_guide.html
- ESP32-S2 / ESP-12K pin references:
  - https://documentation.espressif.com/esp32-s2_datasheet_en.html
  - https://docs.ai-thinker.com/_media/esp32/docs/esp-12k_%E8%A7%84%E6%A0%BC%E4%B9%A6_en.pdf
- ESPHome Modbus controller documentation: https://esphome.io/components/modbus_controller
- Sunsynk Power Flow Card: https://github.com/slipx06/sunsynk-power-flow-card
- Power Flow Card Plus: https://github.com/flixlix/power-flow-card-plus
