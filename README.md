# GoodWe

[![PyPi](https://img.shields.io/pypi/v/goodwe.svg)](https://pypi.python.org/pypi/goodwe/)
[![Python Versions](https://img.shields.io/pypi/pyversions/goodwe.svg)](https://github.com/marcelblijleven/goodwe/)
[![Build Status](https://github.com/marcelblijleven/goodwe/actions/workflows/publish.yaml/badge.svg)](https://github.com/marcelblijleven/goodwe/actions/workflows/publish.yaml)
![License](https://img.shields.io/github/license/marcelblijleven/goodwe.svg)

Library for connecting to GoodWe inverter over local network and retrieving runtime sensor values and configuration
parameters.

It has been reported to work with GoodWe ET, EH, BT, BH, ES, EM, BP, DT, MS, D-NS, and XS families of inverters. It
should work with other inverters as well, as long as they listen on UDP port 8899 or TCP port 502 and respond to one of supported communication protocols.
In general, if you can connect to your inverter with the official mobile app (SolarGo/PvMaster) over Wi-Fi (not
bluetooth), this library should work.

White-label (GoodWe manufactured) inverters may work as well, e.g. General Electric GEP (PSB, PSC) and GEH models are
know to work properly.

Since v0.4.x the library also supports standard Modbus/TCP over port 502.
This protocol is supported by the V2.0 version of LAN+WiFi communication dongle (model WLA0000-01-00P).

(If you can't communicate with the inverter despite your model is listed above, it is possible you have old ARM firmware
version. You should ask manufacturer support to upgrade your ARM firmware (not just inverter firmware) to be able to
communicate with the inverter.)

## Installation

```bash
pip install goodwe
```

## Quick start

```python
import asyncio
import goodwe


async def get_runtime_data():
    ip_address = "192.168.1.14"

    inverter = await goodwe.connect(ip_address)
    runtime_data = await inverter.read_runtime_data()

    for sensor in inverter.sensors():
        if sensor.id_ in runtime_data:
            print(f"{sensor.id_}: \t\t {sensor.name} = {runtime_data[sensor.id_]} {sensor.unit}")


asyncio.run(get_runtime_data())
```

## Python API overview

The package exposes an asynchronous Python API for inverter discovery, runtime data access, settings access, and inverter
control.

### Top-level functions

- `await goodwe.connect(host, port=8899, family=None, comm_addr=0, timeout=1, retries=3, do_discover=True)`
  - Connect to an inverter and return a concrete inverter instance
- `await goodwe.discover(host, port=8899, timeout=1, retries=3)`
  - Auto-detect the inverter family for a known host
- `await goodwe.search_inverters()`
  - Broadcast on the local network to find inverter devices

### Main classes

- `goodwe.Inverter`
  - Abstract base class implemented by all inverter families
- `goodwe.ET`
  - Hybrid ET/EH/BT/BH family support
- `goodwe.ES`
  - Single-phase hybrid ES/EM/BP family support
- `goodwe.DT`
  - Grid-only DT/MS/D-NS/XS family support
- `goodwe.Sensor`
  - Sensor/setting definition metadata

### Common inverter methods

After connecting, the returned inverter instance typically supports:

- `await inverter.read_device_info()`
- `await inverter.read_runtime_data()`
- `await inverter.read_sensor(sensor_id)`
- `await inverter.read_setting(setting_id)`
- `await inverter.read_settings_data()`
- `await inverter.write_setting(setting_id, value)`
- `await inverter.get_grid_export_limit()`
- `await inverter.set_grid_export_limit(export_limit)`
- `await inverter.get_operation_mode()`
- `await inverter.set_operation_mode(...)`
- `await inverter.get_ems_mode()`
- `await inverter.set_ems_mode(...)`
- `inverter.sensors()`
- `inverter.settings()`

Availability of sensors, settings, and control methods depends on the inverter family and firmware.

### Enumerations

The package exports these useful enums:

- `goodwe.SensorKind`
- `goodwe.OperationMode`
- `goodwe.EMSMode`

### Exceptions

The package exports these exception types:

- `goodwe.InverterError`
- `goodwe.RequestFailedException`

## Detailed Python documentation

For full Python API documentation, usage examples, classes, enums, and exception reference, see:

- [docs/API.md](docs/API.md)

## References and useful links

- https://github.com/mletenay/home-assistant-goodwe-inverter
