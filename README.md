# Sunsynk ESP32 RS485

ESPHome configuration for monitoring and controlling a Sunsynk 8kW single-phase inverter over RS485 using an ESP32 and Home Assistant.

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
- Generator/AUX power when the AUX port is configured as generator input
- Daily PV, battery charge/discharge, grid import/export, and load energy totals
- Inverter state code and raw state text
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

v4.7 uses a faster 5s scan interval with stable 750ms RS485 request spacing. Earlier tests showed that 600ms could bring back shifted/corrupt values, while very conservative v4.2 timing was too slow in live use. It also filters impossible live values so dashboards keep the last valid reading instead of publishing corrupt spikes.

The generator/AUX power sensor uses a consecutive-sample confirmation filter. This lets a real generator appear automatically once it produces sustained valid power, while one-off shifted Modbus values are ignored when no generator is connected.

The current Modbus timing is controlled near the top of the YAML:

```yaml
modbus_update_interval: 5s
modbus_command_throttle: 750ms
modbus_send_wait_time: 750ms
```

Slower-changing values use `skip_updates`:

| Setting | Approximate polling interval |
| --- | --- |
| no `skip_updates` | 10 seconds |
| `skip_updates: 5` | 60 seconds |
| `skip_updates: 11` | 2 minutes |
| `skip_updates: 59` | 10 minutes |
| `skip_updates: 119` | 20 minutes |

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

## Home Assistant Templates

These templates go in Home Assistant's `configuration.yaml`, normally at:

```text
/config/configuration.yaml
```

If you already have a top-level `template:` section, do not create a second one. Merge the `- sensor:` and `- binary_sensor:` blocks into the existing `template:` section.

After saving, go to **Developer Tools > YAML > Check Configuration**. Then reload Template Entities or restart Home Assistant.

What these templates create:

| Entity | Purpose |
| --- | --- |
| `sensor.sunsynk_total_pv_power` | Adds PV1 and PV2 power together |
| `sensor.sunsynk_ups_essential_final` | Clean UPS/essential load value from the inverter UPS register |
| `sensor.sunsynk_non_essential_final` | Calculates non-essential load as total load minus UPS/essential load |
| `sensor.sunsynk_home_power_final` | Total home/load power for dashboard cards |
| `sensor.sunsynk_overall_state_text` | Converts the raw inverter state code into readable text |
| `binary_sensor.sunsynk_grid_voltage_connected` | Optional voltage-based grid connected helper; keep the ESPHome `binary_sensor.sunsynk_inverter_grid_connected` entity for dashboard cards |

Template block:

```yaml
template:
  - sensor:
      - name: "Sunsynk Total PV Power"
        unique_id: sunsynk_total_pv_power
        unit_of_measurement: "W"
        device_class: power
        state_class: measurement
        state: >
          {{
            (
              states('sensor.sunsynk_inverter_pv1_power') | float(0) +
              states('sensor.sunsynk_inverter_pv2_power') | float(0)
            ) | round(0)
          }}

      - name: "Sunsynk UPS Essential Final"
        unique_id: sunsynk_ups_essential_final
        unit_of_measurement: "W"
        device_class: power
        state_class: measurement
        state: >
          {{
            states('sensor.sunsynk_inverter_ups_essential_power') | float(0) | abs | round(0)
          }}

      - name: "Sunsynk Non Essential Final"
        unique_id: sunsynk_non_essential_final
        unit_of_measurement: "W"
        device_class: power
        state_class: measurement
        state: >
          {% set total = states('sensor.sunsynk_inverter_load_power') | float(0) | abs %}
          {% set essential = states('sensor.sunsynk_inverter_ups_essential_power') | float(0) | abs %}
          {{ [total - essential, 0] | max | round(0) }}

      - name: "Sunsynk Home Power Final"
        unique_id: sunsynk_home_power_final
        unit_of_measurement: "W"
        device_class: power
        state_class: measurement
        state: >
          {{
            states('sensor.sunsynk_inverter_load_power') | float(0) | abs | round(0)
          }}

      - name: "Sunsynk Overall State Text"
        unique_id: sunsynk_overall_state_text
        state: >
          {% set raw = states('sensor.sunsynk_inverter_overall_state') %}
          {% set code = raw | int(base=16, default=0) %}
          {{ {
            0: 'Standby',
            1: 'Self-test',
            2: 'Normal',
            3: 'Alarm',
            4: 'Fault'
          }.get(code, 'Unknown (' ~ code ~ ')') }}

  - binary_sensor:
      - name: "Sunsynk Grid Voltage Connected"
        unique_id: sunsynk_grid_voltage_connected
        device_class: connectivity
        state: >
          {{ states('sensor.sunsynk_inverter_grid_voltage') | float(0) > 100 }}
```

## Dashboard Cards

Two dashboard examples are included:

| File | Card type | Notes |
| --- | --- | --- |
| `dashboard-cards/sunsynk-power-flow-card.yaml` | `custom:sunsynk-power-flow-card` | Best for a Sunsynk-style detailed inverter diagram |
| `dashboard-cards/power-flow-card-plus.yaml` | `custom:power-flow-card-plus` | Simpler generic power-flow card |

Install the matching custom card through HACS before pasting the YAML into a Lovelace dashboard. If Home Assistant adds `_2` to any entity ID, update the card YAML to match your actual entity IDs.

## Files

| File | Purpose |
| --- | --- |
| `sunsynk-inverter-v4.7.yaml` | Main ESPHome configuration |
| `secrets.example.yaml` | Example ESPHome secrets file |
| `dashboard-cards/sunsynk-power-flow-card.yaml` | Detailed Sunsynk dashboard card example |
| `dashboard-cards/power-flow-card-plus.yaml` | Generic power-flow dashboard card example |
| `.gitignore` | Prevents local secrets/build files being committed |

## Setup

1. Copy `sunsynk-inverter-v4.7.yaml` into ESPHome.
2. Create a `secrets.yaml` based on `secrets.example.yaml`.
3. Update Wi-Fi, MQTT, API, OTA, fallback AP, static IP, and web server secrets.
4. Update the static IP secrets or remove the `manual_ip` block from the YAML.
5. Check your board type and GPIO wiring.
6. Compile and flash from ESPHome Builder.
7. Add the Home Assistant templates from this README to `configuration.yaml`.
8. Add one of the dashboard cards from `dashboard-cards`.
9. Watch logs for sane values.

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
