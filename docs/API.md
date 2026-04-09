# GoodWe Python API Documentation

This document describes the public Python API exposed by the `goodwe` package for discovering, connecting to, and interacting with GoodWe inverters over the local network.

## Installation

```bash
pip install goodwe
```

## Requirements

- Python package: `goodwe`
- Network access to the inverter on the local network
- Supported protocols:
  - UDP on port `8899`
  - Modbus/TCP on port `502`

## Supported inverter families

The library can automatically detect and communicate with these inverter families:

- **ET family**: `ET`, `EH`, `BT`, `BH`
- **ES family**: `ES`, `EM`, `BP`
- **DT family**: `DT`, `MS`, `D-NS`, `XS`

It also includes support for some white-label GoodWe-manufactured models such as selected GE/GEH/GEP variants.

## Top-level package API

The main public API is exposed from `goodwe/__init__.py`.

### `goodwe.connect(...)`

```python
async def connect(
    host: str,
    port: int = 8899,
    family: str | None = None,
    comm_addr: int = 0,
    timeout: int = 1,
    retries: int = 3,
    do_discover: bool = True,
) -> Inverter
```

Connects to an inverter and returns an initialized inverter instance.

#### Parameters

- `host`: IP address or hostname of the inverter
- `port`: communication port
  - `8899` for UDP
  - `502` for Modbus/TCP
- `family`: optional explicit inverter family. Typical values:
  - `ET`, `EH`, `BT`, `BH`
  - `ES`, `EM`, `BP`
  - `DT`, `MS`, `NS`, `XS`
- `comm_addr`: communication address override
- `timeout`: request timeout in seconds
- `retries`: number of retries for failed communication
- `do_discover`: if `True`, attempt family auto-detection when `family` is not supplied

#### Returns

An instance of one of:

- `goodwe.ET`
- `goodwe.ES`
- `goodwe.DT`

all of which inherit from `goodwe.Inverter`.

#### Raises

- `goodwe.InverterError`

#### Example

```python
import asyncio
import goodwe

async def main():
    inverter = await goodwe.connect("192.168.1.14")
    data = await inverter.read_runtime_data()
    print(data["ppv"])

asyncio.run(main())
```

---

### `goodwe.discover(...)`

```python
async def discover(
    host: str,
    port: int = 8899,
    timeout: int = 1,
    retries: int = 3,
) -> Inverter
```

Attempts to identify the inverter type automatically and returns the matching inverter implementation.

Use this when you know the host but do not know the family.

#### Raises

- `goodwe.InverterError`

---

### `goodwe.search_inverters()`

```python
async def search_inverters() -> bytes
```

Broadcasts a discovery request on the local network and returns the raw response payload.

This is useful for locating inverters on the LAN.

#### Returns

- raw `bytes` response from the inverter discovery service

#### Raises

- `goodwe.InverterError`

## Core classes

## `Inverter`

Abstract base class for all inverter implementations.

Concrete implementations:

- `goodwe.ET`
- `goodwe.ES`
- `goodwe.DT`

### Common instance attributes

These attributes are populated after `read_device_info()` or during connection/discovery:

- `model_name: str | None`
- `serial_number: str | None`
- `rated_power: int`
- `ac_output_type: int | None`
- `firmware: str | None`
- `arm_firmware: str | None`
- `modbus_version: int | None`
- `dsp1_version: int`
- `dsp2_version: int`
- `dsp_svn_version: int | None`
- `arm_version: int`
- `arm_svn_version: int | None`

### Common instance methods

#### `await inverter.read_device_info()`

Loads model and firmware metadata from the inverter.

#### `await inverter.read_runtime_data() -> dict[str, Any]`

Reads current runtime sensor values and returns a dictionary keyed by sensor id.

Example keys may include:

- `ppv`
- `vpv1`
- `battery_soc`
- `active_power`
- `house_consumption`

The exact sensor set depends on inverter family and model.

#### `await inverter.read_sensor(sensor_id: str) -> Any`

Reads a single sensor value by id.

Raises `ValueError` if the sensor is not supported by the inverter.

#### `await inverter.read_setting(setting_id: str) -> Any`

Reads one configuration setting by id.

Raises `ValueError` if the setting is not supported.

#### `await inverter.write_setting(setting_id: str, value: Any)`

Writes a configuration setting.

Warning: this modifies inverter configuration and should be used carefully.

#### `await inverter.read_settings_data() -> dict[str, Any]`

Reads all supported settings and returns them as a dictionary.

#### `await inverter.send_command(command: bytes, validator: Callable[[bytes], bool] = lambda x: True)`

Sends a low-level command and returns the protocol response object.

Use this only for advanced protocol-level interactions.

#### `await inverter.get_grid_export_limit() -> int`

Returns the current grid export limit.

#### `await inverter.set_grid_export_limit(export_limit: int) -> None`

Sets the grid export limit.

Warning: this modifies inverter operational settings.

#### `await inverter.get_operation_modes(include_emulated: bool) -> tuple[OperationMode, ...]`

Returns supported operation modes for the inverter.

#### `await inverter.get_operation_mode() -> OperationMode`

Returns the active operation mode.

#### `await inverter.set_operation_mode(...) -> None`

```python
await inverter.set_operation_mode(
    operation_mode,
    eco_mode_power=100,
    eco_mode_soc=100,
)
```

Sets the inverter operation mode.

Special convenience modes:

- `OperationMode.ECO_CHARGE`
- `OperationMode.ECO_DISCHARGE`

These are helper modes implemented through eco-mode scheduling rather than native inverter mode values.

#### `await inverter.get_ems_mode() -> EMSMode`

Returns the current EMS mode.

#### `await inverter.set_ems_mode(ems_mode: EMSMode, ems_power_limit: int | None = None) -> None`

Sets the inverter EMS mode.

#### `await inverter.get_ongrid_battery_dod() -> int`

Returns on-grid battery depth-of-discharge setting.

#### `await inverter.set_ongrid_battery_dod(dod: int) -> None`

Sets on-grid battery depth of discharge.

#### `inverter.sensors() -> tuple[Sensor, ...]`

Returns the list of supported runtime sensor definitions for this inverter.

#### `inverter.settings() -> tuple[Sensor, ...]`

Returns the list of supported settings definitions for this inverter.

#### `inverter.set_keep_alive(keep_alive: bool) -> None`

Enables or disables persistent protocol connection behavior where supported.

## Inverter family classes

## `ET`

Hybrid inverter implementation for ET/EH/BT/BH and related platform variants.

Typical runtime data includes:

- PV voltage/current/power
- on-grid values for up to 3 phases
- backup output metrics
- battery voltage/current/power
- battery SoC/SoH
- inverter temperature
- import/export energy
- diagnostic and warning/error flags

Typical settings and operations include:

- grid export limit
- battery depth of discharge
- operation mode
- EMS mode
- eco mode configuration

## `ES`

Single-phase hybrid inverter implementation for ES/EM/BP family.

Typical runtime data includes:

- PV channel voltage/current/power
- battery metrics
- on-grid voltage/current/frequency
- backup output values
- inverter temperature
- total/ daily energy generation and load
- work mode and diagnostic status

Typical settings include:

- backup supply
- off-grid charge
- shadow scan
- export limit
- battery charge/discharge voltage/current
- depth of discharge
- eco mode groups

## `DT`

Grid-only inverter implementation for DT/MS/D-NS/XS and related variants.

Typical runtime data includes:

- multiple PV strings
- line/grid voltage and current
- per-phase power
- total inverter power
- apparent/reactive power
- temperatures
- daily and lifetime generation
- meter communication status
- RSSI and derating information

Typical settings include:

- inverter time
- shadow scan
- start/stop/restart
- grid export enable/limit

## Sensor model

## `Sensor`

A sensor definition describes a measurable value or configurable setting.

### Dataclass fields

- `id_`: machine-readable identifier
- `offset`: register offset or data offset
- `name`: human-readable label
- `size_`: sensor data size
- `unit`: engineering unit such as `V`, `A`, `W`, `%`
- `kind`: optional `SensorKind`

### Common usage

```python
runtime_data = await inverter.read_runtime_data()

for sensor in inverter.sensors():
    if sensor.id_ in runtime_data:
        print(sensor.id_, sensor.name, runtime_data[sensor.id_], sensor.unit)
```

## Enumerations

## `SensorKind`

Logical sensor grouping.

Values:

- `SensorKind.PV`
- `SensorKind.AC`
- `SensorKind.UPS`
- `SensorKind.BAT`
- `SensorKind.GRID`
- `SensorKind.BMS`

## `OperationMode`

Supported inverter operation modes.

Important values include:

- `GENERAL`
- `OFF_GRID`
- `BACKUP`
- `ECO`
- `PEAK_SHAVING`
- `SELF_USE`
- `ECO_CHARGE`
- `ECO_DISCHARGE`

Not every inverter supports every operation mode.

## `EMSMode`

Energy management system control modes.

Values include:

- `AUTO`
- `CHARGE_PV`
- `DISCHARGE_PV`
- `IMPORT_AC`
- `EXPORT_AC`
- `CONSERVE`
- `OFF_GRID`
- `BATTERY_STANDBY`
- `BUY_POWER`
- `SELL_POWER`
- `CHARGE_BATTERY`
- `DISCHARGE_BATTERY`

Availability depends on inverter family and firmware.

## Exceptions

The package exposes these exception types:

### `InverterError`

Base exception for inverter communication and protocol problems.

### `RequestFailedException`

Raised when a request fails after retries.

Attributes:

- `message`
- `consecutive_failures_count`

### `RequestRejectedException`

Raised when the inverter explicitly rejects a request.

Attributes:

- `message`

### `PartialResponseException`

Raised when response data is incomplete.

Attributes:

- `length`
- `expected`

### `MaxRetriesException`

Raised when retry count is exhausted.

## Usage patterns

## Read all runtime data

```python
import asyncio
import goodwe

async def main():
    inverter = await goodwe.connect("192.168.1.14")
    runtime_data = await inverter.read_runtime_data()

    for sensor in inverter.sensors():
        if sensor.id_ in runtime_data:
            print(f"{sensor.id_}: {sensor.name} = {runtime_data[sensor.id_]} {sensor.unit}")

asyncio.run(main())
```

## Read a single sensor

```python
import asyncio
import goodwe

async def main():
    inverter = await goodwe.connect("192.168.1.14")
    battery_soc = await inverter.read_sensor("battery_soc")
    print("Battery SoC:", battery_soc)

asyncio.run(main())
```

## Read all settings

```python
import asyncio
import goodwe

async def main():
    inverter = await goodwe.connect("192.168.1.14")
    settings = await inverter.read_settings_data()
    print(settings)

asyncio.run(main())
```

## Set operation mode

```python
import asyncio
import goodwe
from goodwe import OperationMode

async def main():
    inverter = await goodwe.connect("192.168.1.14")
    await inverter.set_operation_mode(OperationMode.ECO)

asyncio.run(main())
```

## Important notes

- All library interaction is asynchronous.
- Always use `await` with API methods that perform I/O.
- Sensor and setting availability varies by inverter family, hardware platform, and firmware.
- Writing settings can alter inverter behavior. Validate values carefully before changing configuration.
- UDP communication may require retries because packet delivery is not guaranteed.