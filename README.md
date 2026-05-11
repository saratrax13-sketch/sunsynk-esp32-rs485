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

To make reads stable, this version uses `force_new_range: true` broadly. That makes ESPHome read most items in separate requests. It is a bit slower, but it avoids corrupt grouped reads.

The main scan interval is:

```yaml
update_interval: 5s
```

Slower-changing values use `skip_updates`:

| Setting | Approximate polling interval |
| --- | --- |
| no `skip_updates` | 5 seconds |
| `skip_updates: 5` | 30 seconds |
| `skip_updates: 11` | 60 seconds |
| `skip_updates: 59` | 5 minutes |
| `skip_updates: 119` | 10 minutes |

## Important MQTT Note

ESPHome MQTT switch state messages can be retained by default. Retained MQTT switch state caused Home Assistant to show old switch states being replayed after reconnects.

For this reason, writable inverter switches explicitly use:

```yaml
retain: false
```

on:

- Solar Export
- Use Timer
- Priority Load
- Grid Charge Enabled

If you previously used retained switch topics, clear the retained MQTT messages from your broker.

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

## Board Pin Notes

This configuration was written around GPIO labels, so the important part is that your board exposes GPIO16, GPIO17, GPIO18, GND, and power.

### ESP32-WROOM-32E DevKitC V4, 38-pin

This is the Espressif ESP32-DevKitC V4 style board fitted with an ESP32-WROOM-32E module.

Relevant pins:

| Function in YAML | GPIO | Common DevKitC label | Notes |
| --- | --- | --- | --- |
| UART TX to RS485 | GPIO16 | IO16 | Available on ESP32-WROOM/ESP32-SOLO DevKitC boards |
| UART RX from RS485 | GPIO17 | IO17 | Available on ESP32-WROOM/ESP32-SOLO DevKitC boards |
| RS485 flow control | GPIO18 | IO18 | Used to control DE/RE |
| Ground | GND | GND | Common ground |
| Power | 3V3 or 5V | 3V3 / 5V | Match your RS485 module |

The included YAML uses:

```yaml
esp32:
  board: esp32dev
  framework:
    type: esp-idf
```

### Ai-Thinker NodeMCU-32-S2 / ESP-12K, 42-pin

This is the Ai-Thinker NodeMCU-32-S2 development board based on the ESP-12K ESP32-S2 module.

Relevant GPIOs:

| Function in YAML | ESP32-S2 GPIO | Notes |
| --- | --- | --- |
| UART TX to RS485 | GPIO16 | General-purpose IO on ESP32-S2 |
| UART RX from RS485 | GPIO17 | General-purpose IO on ESP32-S2 |
| RS485 flow control | GPIO18 | General-purpose IO on ESP32-S2 |
| Ground | GND | Common ground |
| Power | 3V3 or 5V | Match your RS485 module |

If you use the ESP32-S2 board, change the board setting to the correct ESPHome board for your hardware, for example:

```yaml
esp32:
  board: nodemcu-32s2
  framework:
    type: esp-idf
```

Check your exact board silk-screen before wiring. Development boards with the same module can expose pins in different physical positions.

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

## Files

| File | Purpose |
| --- | --- |
| `sunsynk-inverter.yaml` | Main ESPHome configuration |
| `secrets.example.yaml` | Example ESPHome secrets file |
| `.gitignore` | Prevents local secrets/build files being committed |

## Setup

1. Copy `sunsynk-inverter.yaml` into ESPHome.
2. Create a `secrets.yaml` based on `secrets.example.yaml`.
3. Update Wi-Fi, MQTT, API, OTA, and web server secrets.
4. Update the static IP secrets or remove the `manual_ip` block from the YAML.
5. Check your board type and GPIO wiring.
6. Compile and flash from ESPHome Builder.
7. Watch logs for sane values.

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
