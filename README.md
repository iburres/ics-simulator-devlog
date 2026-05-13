# ICS Simulator — Development Log

A public development log for the **ICS Simulator** project — an ICS/SCADA security research and education platform built for researchers, practitioners, and students.

## What is ICS Simulator?

ICS Simulator is a desktop application that allows users to build, configure, and interact with realistic simulated Industrial Control System environments. It supports drag-and-drop SCADA topology design, real protocol simulation, PLC programming, network security monitoring, and red team attack scenarios — all running locally on a single machine.

Designed for:
- Security researchers studying ICS/SCADA vulnerabilities
- Educators teaching industrial cybersecurity
- Students learning ICS protocols and defensive techniques

## About This Log

This devlog documents progress, design decisions, and release milestones. It contains no source code, no configuration details, and no information that could be used to compromise real systems.

Each entry covers:
- What was built or decided
- Why that approach was chosen
- What comes next

## Entries

| Date | Entry | Summary |
|------|-------|---------|
| 2026-05-12 | [Phase 0 — Foundation](entries/2026-05-12-phase-0-foundation.md) | Project scaffold, tech stack decisions, monorepo setup |

## Technology Overview

Built on open-source foundations:
- **Electron** — cross-platform desktop shell (Windows, macOS, Linux)
- **React + React Flow** — drag-drop canvas and UI
- **Docker** — isolated virtual network for each simulation
- **Suricata + Zeek** — real IDS/IPS and network protocol analysis
- **FUXA** — open-source HMI designer and runtime
- **Grafana + InfluxDB** — monitoring dashboards and process historian
- Real protocol simulation: Modbus, DNP3, OPC UA, BACnet, EtherNet/IP, IEC 61850

## Contact

Developed by Ian Burres — Professor of Practice, UTSA  
ORCID: [0009-0006-1320-9956](https://orcid.org/0009-0006-1320-9956)
