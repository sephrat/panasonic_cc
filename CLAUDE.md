# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Home Assistant custom component (`panasonic_cc`) for controlling Panasonic Comfort Cloud devices (air conditioners, heat pumps) and Aquarea water heaters via cloud polling. It is installed into `<ha-config>/custom_components/panasonic_cc/`.

## Development Environment

The recommended dev environment is the VS Code dev container (`.devcontainer/`), which spins up a standalone Home Assistant instance pre-configured with `config/configuration.yaml`. Open in VS Code and use the "Reopen in Container" option.

## Code Style

Format with `black`. There is no automated test suite — verify changes by running the integration in the dev container.

## Architecture

All code lives in `custom_components/panasonic_cc/`.

**Two device families, one integration:**
- **Panasonic Comfort Cloud** — air conditioners and heat pumps, using `aio-panasonic-comfort-cloud`
- **Aquarea** — water heaters and zone-based heat pumps, using `aioaquarea`

Each family follows the same layered pattern:

```
coordinator.py          → DataUpdateCoordinator subclasses per device family
base.py                 → CoordinatorEntity base classes (one per family)
climate/sensor/etc.py   → Platform entity classes extending base classes
```

**Coordinators (`coordinator.py`):**
- `PanasonicDeviceCoordinator` — polls Comfort Cloud device state (default 120 s)
- `PanasonicDeviceEnergyCoordinator` — polls energy data (default 300 s)
- `AquareaDeviceCoordinator` — polls Aquarea device state

**Entity platforms:** `climate`, `sensor`, `switch`, `select`, `number`, `button`, `water_heater`. Each file uses Home Assistant's entity description dataclass pattern with lambda callbacks for reading state from coordinator data.

**`__init__.py`** handles `async_setup_entry` / `async_unload_entry`, creates coordinators, and forwards setup to all platforms.

**`config_flow.py`** drives the UI setup flow (credentials + options like fetch intervals, energy sensor toggle, preset naming).

**`const.py`** is the single source of truth for domain constants, sensor type keys, HVAC mode mappings, and component type identifiers.

## Key Constraints

- Minimum Home Assistant version: 2024.12.1 (see `manifest.json`)
- `integration_type` is `hub` — one config entry manages all devices under a single Panasonic account
- `iot_class` is `cloud_polling` — no local API; all state comes from Panasonic's cloud
- 2FA (SMS) must be configured in the Panasonic app before the integration can authenticate
