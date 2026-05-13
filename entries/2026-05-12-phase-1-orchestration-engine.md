# Phase 1 — Docker Orchestration Engine

**Date:** 2026-05-12  
**Status:** Complete

## What was built

Phase 1 delivered the backend engine that translates a declarative `.icslab` scenario file into a live multi-container simulation. No UI changes — everything happens in the Electron main process and a new `@ics-sim/orchestrator` package.

### orchestrator package

New `packages/orchestrator/` with six focused modules:

- **compose-generator** — takes a parsed scenario and emits a Docker Compose YAML string. Handles per-device resource limits (memory + CPU), IP assignment from scenario subnets, and five fixed infrastructure services: Suricata, Zeek, InfluxDB 1.8, Loki, Grafana, and FUXA.
- **docker-client** — thin wrapper around `docker compose` CLI. Writes compose files to `userData/scenarios/{projectName}/`, then calls `up -d`, `down --volumes`, `ps --format json`, and `pull`. Parses the JSON-line output of `docker compose ps` into typed `ContainerStatus` objects.
- **schema-validator** — validates imported `.icslab` JSON against required fields before anything else runs.
- **resource-estimator** — estimates RAM and CPU the scenario will need, compares against current free system memory, and flags warning (>60%) or critical (>85%) thresholds so the UI can warn users before starting a simulation.
- **network-config** — canonical subnet/gateway defaults for the four network zones (OT `172.20.10.0/24`, IT `172.20.20.0/24`, DMZ `172.20.30.0/24`, External `172.20.40.0/24`).

### LevelDB store (db.ts)

Added `packages/app/src/main/db.ts` — a `classic-level` wrapper for the Electron main process working store. Persists the active scenario across restarts, plus arbitrary key/value settings.

### IPC wiring

The Electron main process now handles:
- `scenario:import` — validates, estimates, warns on high RAM, saves to LevelDB
- `simulation:start` — generates compose YAML, calls dockerClient.startScenario
- `simulation:stop` — tears down all containers, clears LevelDB
- `simulation:status` — polls container state via `docker compose ps`
- `system:meminfo` — exposes `os.totalmem/freemem/cpus` to the renderer

## Architecture notes

Each simulated ICS device gets its own Alpine container. The hard cap is 20 devices per scenario — chosen to keep resource requirements within reach of a developer laptop (16 GB RAM handles roughly 15 devices + full infrastructure stack). Infrastructure services (Suricata, Zeek, etc.) are always present regardless of scenario size.

Protocol container images (Modbus, DNP3, OPC UA, etc.) are referenced by GHCR image tag and will be built in Phase 3. They are placeholder references for now.

GPL/AGPL licensed tools (Grafana, Loki, Suricata) are pulled from Docker Hub at runtime — never bundled in the installer. This keeps the commercial distribution legally clean.

## Next

Phase 2: the React Flow canvas — drag-drop SCADA topology editor with ISA-5.1 P&ID device icons, network zone regions, and a device properties panel.
