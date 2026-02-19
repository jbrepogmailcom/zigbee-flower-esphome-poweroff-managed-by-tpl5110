# ESPHome Zigbee Laser Flower Monitor - Poweroff Managed by TPL5110

This project publishes:
- distance (mm)
- battery voltage (V)
- battery (%)

This variant is designed for external power gating with **TPL5110**.
The MCU is fully powered off between wake cycles.

## Hardware
- Seeed Studio XIAO nRF52840 Sense (`xiao_ble` in ESPHome)
- VL53L0X ToF sensor
- TPL5110 power timer

I2C pins used here:
- `SDA: P1.13`
- `SCL: P1.14`

TPL5110 control wiring:
- `DONE` from TPL5110 -> `P0.29` (XIAO pin 029-D3)
- `P1.12` did not work reliably for DONE switching in this setup.
- Firmware keeps DONE pin LOW while running and sets it HIGH to shut power off.
- Added **47uF capacitor between `GND` and `DRV/VOUT`** on the TPL5110 module to stabilize MCU power-up.

## ESPHome file
Main firmware config:
- `zigbee_flower_tpl5110_v6.yaml`

Previous baseline:
- `zigbee_flower_tpl5110_v3.yaml`

Compile example:
```bash
esphome compile zigbee_flower_tpl5110_v6.yaml
```

## What Changed vs v3
- Measurement is done **before Zigbee join** and cached.
- Publish happens **only after joined** (`battery`, `%`, `distance`), then DONE is asserted.
- Added fast reconnect strategy (`REJOIN` first, periodic `STEERING` fallback).
- Added ToF cycle diagnostics:
  - update calls count
  - filter input count
  - filter output count
- ToF averaging tuned to `20` samples with `60ms` spacing (`timing_budget 50ms + 10ms margin`).
- Distance publish path decoupled:
  - internal raw ToF sensor (`Distance_Raw`)
  - exposed cached publish sensor (`Distance`).

## Zigbee2MQTT converter
Converter file:
- `zigbee2mqtt/esphome_nrf52840_tof_converter.js`

### Install in Zigbee2MQTT
1. Copy converter to Zigbee2MQTT data directory, e.g.:
```bash
cp zigbee2mqtt/esphome_nrf52840_tof_converter.js /opt/zigbee2mqtt/data/external_converters/
```

2. Add to `/opt/zigbee2mqtt/data/configuration.yaml`:
```yaml
external_converters:
  - external_converters/esphome_nrf52840_tof_converter.js
```

3. Restart Zigbee2MQTT.

4. Reconfigure the device once (MQTT):
```bash
mosquitto_pub -h <MQTT_HOST> -u <MQTT_USER> -P <MQTT_PASS> \
  -t zigbee2mqtt/bridge/request/device/configure \
  -m '{"id":"<YOUR_FRIENDLY_NAME>"}'
```

## Power behavior
Using TPL5110 power gating gives the lowest standby drain, because the board is fully switched off between wakeups.

Expected battery life on **CR2032** is approximately **5 years** (depends on wake interval, Zigbee join time, RF conditions, and battery quality).

Without external power gating, measured current with ESPHome Zigbee on this board was about **~1.33 mA** in the best tested setup.

## Home Assistant hint
`distance` can be used for automation, e.g. trigger alert when flower leaves are dropping too fast.
