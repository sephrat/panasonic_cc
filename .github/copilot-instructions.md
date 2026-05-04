# Copilot Instructions

## What This Is

A Home Assistant custom component (`panasonic_cc`) for controlling Panasonic Comfort Cloud devices (air conditioners, heat pumps) and Aquarea water heaters via cloud polling. Installed into `<ha-config>/custom_components/panasonic_cc/`.

- Minimum HA version: 2024.12.1
- `integration_type: hub` — one config entry per Panasonic account, managing all devices
- `iot_class: cloud_polling` — no local API; all state comes from Panasonic's cloud

## Development

**No automated test suite.**

**Formatter:** `black`
**Lint:** `scripts/lint`

## Architecture

All code lives in `custom_components/panasonic_cc/`. Two device families share one integration:

| Family | Devices | Library |
|--------|---------|---------|
| Panasonic Comfort Cloud | Air conditioners, heat pumps | `aio-panasonic-comfort-cloud` |
| Aquarea | Water heaters, zone heat pumps | `aioaquarea` |

### Layered pattern (same for both families)

```
coordinator.py   → DataUpdateCoordinator subclasses — own all API calls and device state
base.py          → CoordinatorEntity base classes — implement _handle_coordinator_update
climate/sensor/  → Platform entity classes — extend base classes, declare entity descriptions
```

**Coordinators (`coordinator.py`):**
- `PanasonicDeviceCoordinator` — polls device state (default 120 s)
- `PanasonicDeviceEnergyCoordinator` — polls energy data (default 300 s); only created when `enable_daily_energy_sensor` is `True`
- `AquareaDeviceCoordinator` — polls Aquarea device state; only created when `api.has_unknown_devices` is `True` (i.e., devices not recognized as Comfort Cloud)

**`__init__.py`** runs `async_setup_entry`: authenticates, creates coordinators, stores them in `hass.data[DOMAIN]` under the keys `DATA_COORDINATORS`, `ENERGY_COORDINATORS`, `AQUAREA_COORDINATORS`, then forwards setup to all platforms in `COMPONENT_TYPES`.

**`const.py`** is the single source of truth for domain constants, sensor type keys, HVAC mode mappings, component type identifiers, and fetch interval defaults.

**`config_flow.py`** drives the UI setup flow (credentials) and options flow (fetch intervals, energy sensor toggle, preset naming).

## Key Conventions

### Entity description pattern
All platform files use frozen dataclasses extending HA's `*EntityDescription`, with `get_state` and `is_available` lambda callbacks that receive the device object:

```python
@dataclass(frozen=True, kw_only=True)
class PanasonicSensorEntityDescription(SensorEntityDescription):
    get_state: Callable[[PanasonicDevice], Any] | None = None
    is_available: Callable[[PanasonicDevice], bool] | None = None
```

### `_async_update_attrs` pattern
Every entity base class calls `_async_update_attrs()` in `__init__` and in `_handle_coordinator_update`. Concrete entity classes implement this abstract method to push coordinator data into `_attr_*` fields.

### Entity naming and translation
- `_attr_has_entity_name = True` on all base classes
- `_attr_translation_key` set to the entity description `key`
- `_attr_unique_id` = `f"{coordinator.device_id}-{key}"`
- Display strings live in `strings.json` and `translations/*.json` — **never** hardcode user-visible names in Python

### Applying device changes (Comfort Cloud)
Use `coordinator.get_change_request_builder()` to get a `ChangeRequestBuilder`, set only the changed fields, then call `coordinator.async_apply_changes(builder)`. Do not call the API client directly from entity code.

### Aquarea demo mode
`AQUAREA_DEMO = False` in `__init__.py` can be toggled locally to test Aquarea entities against the library's built-in demo environment without real hardware.

### Model quirks
`MODELS_WITHOUT_HORIZONTAL_SWING` in `const.py` lists model prefixes that falsely report horizontal swing support in the API. Check `coordinator.device_has_horizontal_swing` (not the raw device property) when deciding whether to expose the horizontal swing select entity.
