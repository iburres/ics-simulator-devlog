# Phase 0 — Project Foundation

**Date:** 2026-05-12  
**Status:** Complete

## What Was Built

The foundational scaffold for ICS Simulator was established today. This covers the full project structure, build pipeline, and cross-platform packaging configuration — everything needed before feature development begins.

## Key Decisions Made

### Architecture
After evaluating several approaches, the project settled on a single-user Electron desktop application. Scenarios are portable `.icslab` files that instructors create and students import — no server infrastructure required, no subscription, fully offline.

### Why Electron + React
Foundry VTT (a popular virtual tabletop application) served as the architectural reference. It uses a similar Electron + Node.js stack. However, where Foundry VTT uses PixiJS (a game renderer) for its canvas, ICS Simulator uses **React Flow** — a purpose-built library for interactive node/edge graph editors. This was the right call: SCADA topology diagrams are network graphs, not game maps.

### Why Docker for Simulation
Every simulated device (PLC, RTU, sensor, firewall) runs as its own Docker container on a virtual network. This gives students real network isolation — the attack machine lives on a genuinely separate network segment. IT/OT separation is enforced at the OS networking layer, not simulated in software.

### Protocol Stack
Six protocols are supported with real open-source implementations:
- **Modbus** TCP/RTU/ASCII — the most common ICS protocol
- **DNP3** — widely used in power and water utilities
- **OPC UA** — the modern ICS integration standard
- **BACnet** — building automation and HVAC
- **EtherNet/IP (CIP)** — manufacturing and automation
- **IEC 61850** — power grid substations (replaced Profinet, which has no viable open-source implementation)

### Security Stack
The simulator ships with both **Suricata** (inline IPS with ICS-specific rulesets) and **Zeek** (deep protocol analysis). This combination is already proven in production ICS security environments and gives students both enforcement and forensic depth simultaneously.

### Business Model
One-time license purchase plus optional sector-specific scenario packs. No subscription. Fully offline after activation. This directly addresses the main frustration with existing tools in this space.

## What Was Completed Today

- npm workspaces monorepo initialized
- Electron + React + TypeScript + Vite build pipeline configured
- Cross-platform packaging configured (Windows installer, macOS DMG, Linux AppImage)
- GitHub Actions CI/CD pipeline — matrix build across all three platforms on every push
- ESLint + Prettier + Husky pre-commit hooks
- Typed IPC contract between Electron main process and React renderer
- Complete `.icslab` JSON schema defined in TypeScript — all four layers (Visual, Network, Device, Security)
- Docker availability check on startup with actionable error messaging
- Phase 0 shell UI running: status panel, Docker detection, scenario import stub

## Challenges

**Node.js PATH on Windows:** The PowerShell execution policy blocked npm scripts. Fixed with `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`.

**Docker PATH in Electron:** Electron's `exec()` spawns child processes with a minimal inherited environment — Docker Desktop's binary directory wasn't in it. Fixed by explicitly prepending known Docker installation paths per platform before each Docker command.

**Dependency version conflict:** `@vitejs/plugin-react` v6 requires Vite 8, but `electron-vite` 2.x uses Vite 5. Pinned to `@vitejs/plugin-react@^4.3.2` which is compatible with Vite 5.

## What Comes Next

**Phase 1** — `.icslab` schema validation, Docker Compose generator, and LevelDB working store. This is the engine that turns a scenario file into a running Docker topology.

**Phase 2** — React Flow canvas with drag-drop device palette, ISA-5.1 standard P&ID symbols, network zone visualization, and device properties panels.
