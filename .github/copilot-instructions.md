**Project Overview**

This repository is a Home Assistant custom integration that exposes Google Home/Nest device alarms, timers and simple controls via the `google_home` integration in `custom_components/google_home`.

**Architecture & Data Flow**

- **Auth & discovery:** `custom_components/google_home/api.py` uses `glocaltokens` to obtain master/access tokens and discovers devices (local IPs) via Zeroconf.
- **Coordinator model:** `__init__.py` creates a `DataUpdateCoordinator` that polls devices (default 180s). The coordinator calls `GlocaltokensApiClient.update_google_devices_information` to fetch device state.
- **Platform split:** entity platforms are split into `sensor.py`, `switch.py`, and `number.py`. Shared device model and helpers are in `models.py`, `entity.py` and `api.py`.

**Key files to scan first**

- `custom_components/google_home/manifest.json` — declares dependencies (`glocaltokens`), `iot_class`, `after_dependencies` (e.g. `zeroconf`) and `version`.
- `custom_components/google_home/__init__.py` — integration setup, coordinator creation, and `async_forward_entry_setups` for platforms.
- `custom_components/google_home/api.py` — API client using `glocaltokens`; contains request patterns, error handling and token refresh behavior.
- `custom_components/google_home/services.yaml` — service signatures (delete alarm/timer, refresh, reboot).
- `pyproject.toml` — build tooling (`hatchling`) and dev dependencies (pytest, mypy, ruff, pylint, etc.). Also declares `homeassistant==2025.6.0` and `glocaltokens` requirement.

**Project-specific conventions & patterns**

- Uses Home Assistant integration patterns: `async_setup_entry`, `async_unload_entry`, `DataUpdateCoordinator`, and platform modules named after HA platforms (sensor/switch/number).
- Async/IO boundary: CPU-bound `glocaltokens` calls are executed via `hass.async_add_executor_job` in `api.py` (look for helper functions that wrap blocking calls).
- Logging: the module checks `_LOGGER.level == logging.DEBUG` to toggle verbose behavior in `GlocaltokensApiClient`.
- Polling vs setting: many API endpoints are used for both reading (polling) and writing (set) — see `update_alarm_volume` / `update_do_not_disturb` which set `polling=True` when reading.

**Developer workflows (how contributors run & test locally)**

- Build / packaging: `pyproject.toml` with `hatchling` — use `hatch build` / `hatch run` if needed.
- Static checks & tests: dev dependencies include `ruff`, `pylint`, `mypy`, and `pytest`. Run `hatch run pytest` or `pytest` in an environment from `pyproject.toml`.
- Running inside Home Assistant: repository README documents running inside a Home Assistant container and mentions tasks for starting/checking a container in the workspace. For local integration testing, install into your Home Assistant `custom_components` directory or use the provided Docker instructions in README.

**Integration & runtime notes (important for agents)**

- The integration expects real Google devices or tokens from `glocaltokens`. Some behaviors (device IP missing, unauthorized token) will clear `google_devices` and trigger token refresh — tests and mocks should simulate both success and token errors.
- Default polling interval changed historically to 180s; the coordinator uses `entry.options.get(CONF_UPDATE_INTERVAL, UPDATE_INTERVAL)` so tests can override via config entry options.
- After-dependencies include `zeroconf`, `cloud`, and `http` — ensure Zeroconf is available in environment when mocking discovery.

**Examples to reference when editing code**

- To add a new platform: mirror existing `sensor.py` pattern, export platform name in `PLATFORMS` constant (see `const.py`) and forward setup in `__init__.py`.
- To call device endpoints: use `GlocaltokensApiClient.request(...)` and follow existing headers (`HEADER_CAST_LOCAL_AUTH`) and timeout handling in `api.py`.

**Where to look for localization & UI strings**

- `translations/*.json` contains all front-end strings — update these when adding UI-visible text.

If anything is unclear or you want more detail (e.g., example tests, mocks for `glocaltokens`, or a short contributor checklist), tell me which area to expand and I'll update this file.
