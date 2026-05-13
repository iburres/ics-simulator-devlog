# Phase 2 — SCADA Canvas Editor

**Date:** 2026-05-12  
**Status:** Complete

## What was built

Phase 2 delivers the visual authoring layer — the drag-drop SCADA topology canvas that turns the app from a shell into a working editor.

### Canvas architecture

Built on **@xyflow/react v12** (React Flow). The canvas has two node types and one edge type:

- **ZoneNode** — four fixed background regions (OT Network, IT Network, DMZ, External), each with a distinct border color and low-opacity fill. Non-interactive; they establish the security zone geography of every scenario.
- **DeviceNode** — a compact card showing the ISA-5.1 device icon, node label, and IP address. Border color matches the zone the device belongs to. Handles on top and bottom for drawing connections.
- **ProtocolEdge** — a bezier edge with a floating protocol badge (modbus-tcp, dnp3, opc-ua, etc.). Each protocol has an assigned accent color. Edges default to `modbus-tcp` when drawn.

### ISA-5.1 icons

All 16 `DeviceCategory` types have hand-drawn SVG icons following ISA-5.1 / IEC 5417 P&ID conventions:
- PLC/RTU/IED: rectangular enclosure symbols with distinguishing internal marks
- Sensor / Pressure TX: ISA instrument bubble (circle with category code)
- Pump / Valve / Flow Meter: ISA process equipment symbols
- HMI: monitor shape; Historian: database cylinder
- Firewall: brick-wall pattern; IDS/IPS: shield with eye; Switch/Router: network equipment
- Attack Machine: monitor with targeting crosshair

Icons render at any size via inline SVG (no raster assets required).

### Device palette

Left sidebar groups all 16 device types into five sections (Control, Field Devices, Monitoring, Network, Red Team). Items are HTML5 draggable — drop onto the canvas to place a new device. Zone is auto-detected by drop coordinates (2×2 grid: OT top-left, IT top-right, DMZ bottom-left, External bottom-right). Default IP and protocol are assigned per category.

### Properties panel

Right panel shows selected device properties: node ID, zone, IP address, protocol list, and any Modbus/DNP3/OPC-UA configuration present. Attack-machine devices display a resource warning note. Panel shows an empty state when nothing is selected.

### App shell redesign

The app now opens to a **Launch Screen** with Docker status, two entry points (New Scenario / Open .icslab), and version info. Once in the editor, the full shell layout activates:

- **Toolbar** — scenario name, device count, simulation status badge (Idle / Starting / Running / Stopping), New/Open buttons, Run Simulation / Stop Simulation controls
- **Workspace** — Device Palette (188px) + Canvas (flex) + Properties Panel (260px)
- **Status bar** — Docker version, container count when running, container health pills (up to 6)

Simulation start/stop wires through to the Phase 1 orchestrator (Docker Compose generator). The status bar polls container health from `docker compose ps` via the existing IPC channel.

### Scenario import → canvas

When a `.icslab` file is imported:
- If the scenario has a `visual` layer (CanvasNode[] positions), devices render at their saved positions.
- If `visual.nodes` is empty (hand-crafted file), devices are auto-laid out by category into the appropriate zone quadrant (3-per-row grid).

## Next

Phase 3: Protocol container images — seven Alpine Docker images pushed to GHCR (Modbus, DNP3, OPC UA, BACnet, EtherNet/IP, IEC 61850, OpenPLC Runtime).
