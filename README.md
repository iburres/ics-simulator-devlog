# ICS Simulator — Development Log

A public development log for the **ICS Simulator** project — an ICS/SCADA security research and education platform built for researchers, practitioners, and students.

---

## What is ICS Simulator?

ICS Simulator is a desktop application that allows users to build, configure, and interact with realistic simulated Industrial Control System environments. It supports drag-and-drop SCADA topology design, real protocol simulation, PLC programming, network security monitoring, and red team attack scenarios — all running locally on a single machine via Docker.

Designed for:
- Security researchers studying ICS/SCADA vulnerabilities
- Educators teaching industrial cybersecurity
- Students learning ICS protocols and defensive techniques

---

## Build Status

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 0 — Foundation | ✅ Complete | Project scaffold, Electron shell, Docker status check, first launch flow |
| Phase 1 — Orchestration Engine | ✅ Complete | Docker Compose generator, LevelDB store, resource estimator, scenario validator |
| Phase 2 — SCADA Canvas | ✅ Complete | React Flow canvas, ISA-5.1 device icons, zone regions, drag-drop palette, properties panel |
| Phase 3 — Container Images | ✅ Complete | All 9 protocol/security/infra Docker images; GitHub Actions CI/CD to GHCR |
| Phase 4 — PLC IDE | ✅ Complete | ST editor, SVG ladder viewer, variable bindings, OpenPLC HTTP deploy |
| Phase 5 — Security Stack UI | 🔜 Planned | Firewall rule editor, Suricata/Zeek alert panel |
| Phase 6 — Monitoring Panel | 🔜 Planned | Grafana + Loki + InfluxDB embedded dashboards |
| Phase 7 — HMI | 🔜 Planned | FUXA HMI container + Electron panel + sector templates |
| Phase 8 — Attack Terminal | 🔜 Planned | Kali xterm.js terminal, attack tool launcher |
| Phase 9 — Author/Student Modes | 🔜 Planned | Locked export, mission brief panel, scenario lifecycle UI |
| Phase 10 — Scenario Packs | 🔜 Planned | `.icspack` format, pack loader, license gate |
| Phase 11 — Sector Packs | 🔜 Planned | Oil & Gas, Power/Electric, Water Treatment, Automotive |
| Phase 12 — Packaging & Store | 🔜 Planned | License key system, auto-updater, final installer builds |

---

## Development Log Entries

| Date | Entry | Phase | Key Decisions |
|------|-------|-------|---------------|
| 2026-05-12 | [Phase 0 — Foundation](entries/2026-05-12-phase-0-foundation.md) | 0 | Tech stack, monorepo layout, Electron shell, first-launch Docker prompt |
| 2026-05-12 | [Phase 1 — Orchestration Engine](entries/2026-05-12-phase-1-orchestration-engine.md) | 1 | Docker Compose generator, 4-zone network model, LevelDB persistence, resource estimation |
| 2026-05-12 | [Phase 2 — SCADA Canvas](entries/2026-05-12-phase-2-scada-canvas.md) | 2 | React Flow v12, ISA-5.1 inline SVG icons, zone background nodes, drag-drop palette |
| 2026-05-13 | [Phase 3 — Container Images](entries/2026-05-13-phase-3-container-images.md) | 3 | Pure-Python DNP3 outstation, pymodbus 3.7, Suricata ICS rules, Zeek ICS scripts, GHCR CI/CD |
| 2026-05-13 | [Phase 4 — PLC IDE](entries/2026-05-13-phase-4-plc-ide.md) | 4 | ST editor, IEC 61131-3 variable bindings, ladder logic SVG, OpenPLC HTTP API deploy, INITIAL_PROGRAM_B64 pre-load |

---

## Architecture Overview

### Application Stack

```
┌─────────────────────────────────────────────────┐
│                Electron Shell                   │
│  ┌──────────────┐  ┌────────────────────────┐  │
│  │  Main Process│  │   Renderer (React)     │  │
│  │  (Node.js)   │  │   SCADA Canvas         │  │
│  │  Orchestrator│  │   Device Palette       │  │
│  │  LevelDB     │  │   Properties Panel     │  │
│  │  DockerClient│◄─┤   Simulation Controls  │  │
│  └──────┬───────┘  └────────────────────────┘  │
└─────────┼───────────────────────────────────────┘
          │  docker compose up/down/ps
          ▼
┌─────────────────────────────────────────────────┐
│              Docker Network (per scenario)       │
│                                                 │
│  OT (172.20.10.0/24)   IT (172.20.20.0/24)     │
│  ┌────┐ ┌────┐ ┌────┐  ┌──────────┐ ┌──────┐  │
│  │PLC │ │RTU │ │IED │  │Historian │ │FUXA  │  │
│  └────┘ └────┘ └────┘  │(InfluxDB)│ │(HMI) │  │
│                         └──────────┘ └──────┘  │
│  DMZ (172.20.30.0/24)  External (172.20.40.0)  │
│  ┌──────────┐ ┌──────┐ ┌──────────────────────┐│
│  │Suricata  │ │Zeek  │ │  Attack Machine       ││
│  │(IDS/IPS) │ │      │ │  (Kali Linux)         ││
│  └──────────┘ └──────┘ └──────────────────────┘│
└─────────────────────────────────────────────────┘
```

### Scenario File Format (`.icslab`)

Each scenario is a JSON document with four layers:
- **Visual** — React Flow node positions and edge connections
- **Network** — subnet definitions per zone, static routes
- **Devices** — per-device configuration (IP, protocols, register maps)
- **Security** — firewall ACL rules, IDS ruleset selection, logging config

Locked scenarios (for student distribution) omit the Visual and Security layers, preventing topology extraction.

### Protocol Support

| Protocol | Container | Port |
|----------|-----------|------|
| Modbus TCP | `ics-sim-modbus` | 502 |
| Modbus RTU/ASCII | `ics-sim-modbus` | configurable |
| DNP3 | `ics-sim-dnp3` | 20000 |
| OPC UA | Phase 4+ | 4840 |
| BACnet | Phase 4+ | 47808 |
| EtherNet/IP | Phase 4+ | 44818 |
| IEC 61850 | Phase 4+ | 102 |

---

## Technology Stack

| Layer | Technology | License |
|-------|-----------|---------|
| Desktop shell | Electron 31 | MIT |
| UI framework | React 18 + React Flow v12 | MIT |
| Build tool | electron-vite 2 / Vite 5 | MIT |
| Local store | classic-level (LevelDB) | MIT |
| Compose generator | js-yaml | MIT |
| Modbus simulation | pymodbus 3.7 | BSD |
| DNP3 simulation | Pure Python (custom) | — |
| PLC runtime | OpenPLC Runtime | GPLv3 |
| IDS/IPS | Suricata + ET rules | GPLv2 |
| Network analysis | Zeek | BSD |
| Firewall | nftables | GPLv2 |
| HMI | FUXA | MIT |
| Historian | InfluxDB 1.8 | MIT |
| Dashboards | Grafana + Loki | AGPLv3 |
| Attack machine | Kali Linux | Various |

GPL/AGPL licensed tools (Grafana, Loki, Suricata, OpenPLC) are pulled from Docker Hub at simulation runtime — never bundled in the installer binary. This keeps the commercial distribution legally clean.

---

## About This Log

This devlog documents progress, design decisions, and architectural choices. It contains no source code, no credentials, and no configuration details that could be used to compromise real systems.

Each entry covers: what was built, why that approach was chosen, and what comes next.

---

## Contact

**Ian Burres** — Professor of Practice, The University of Texas at San Antonio  
ORCID: [0009-0006-1320-9956](https://orcid.org/0009-0006-1320-9956)
