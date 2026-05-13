# Phase 3 — Protocol and Infrastructure Container Images

**Date:** 2026-05-13  
**Status:** Complete

## What was built

Phase 3 delivers all nine Docker container images that back the simulated ICS environment. Every device the canvas editor places on a scenario becomes one of these containers when the simulation runs. Images are built by GitHub Actions and pushed to `ghcr.io/iburres/`.

---

### Protocol containers

**`ics-sim-modbus`** (Alpine / Python 3.12)  
Implements a Modbus TCP server using pymodbus 3.7. Each device category gets a meaningful register map — a pressure transmitter starts at HR0=1013 (101.3 kPa), a pump at HR0=1 (running) with HR1=1500 (RPM). A background asyncio task drifts the primary process register every 2 seconds to produce data that looks like a live process rather than a frozen reading. Supports all device categories that use Modbus: sensor, pump, valve, flow-meter, pressure-transmitter, RTU, PLC.

**`ics-sim-dnp3`** (Alpine / Python 3.12)  
A full DNP3 Level 1 outstation written in pure Python — no C++ bindings, no compilation, runs on Alpine with zero dependencies beyond the standard library. Implements:
- CRC-16/DNP (reflected, polynomial 0xA6BC) for frame validation
- Link-layer frame encoding and parsing with 16-byte data blocks
- Transport-layer segmentation (FIR/FIN/SEQ)
- Application-layer Class 0 poll responses carrying Group 30, Var 5 (32-bit IEEE float) analog inputs

Four analog values drift every 2 seconds. Responds correctly to master READ requests.

**`ics-sim-openplc`** (Ubuntu 22.04)  
Builds OpenPLC Runtime from the official GitHub source. Exposes the web programming interface on port 8080 and a Modbus/TCP slave on port 502. The web UI lets instructors load ladder logic or Structured Text programs into the PLC container during a running simulation. Larger image than the others (~1 GB) due to the Ubuntu base and build toolchain.

---

### Security stack

**`ics-sim-suricata`** (jasonish/suricata base)  
Extends the jasonish Suricata image with:
- `ics-rules.rules` — 11 custom signatures covering Modbus write anomalies, DNP3 Direct Operate detection, cross-zone lateral movement (IT→OT), and attack-machine scanning
- `ics-sim.yaml` — Suricata overlay config that enables the `modbus` and `dnp3` app-layer decoders, defines OT/IT/DMZ/External address groups, and outputs Eve JSON to `/var/log/suricata/eve.json` for Loki ingestion
- Entrypoint runs `suricata-update` to pull the Emerging Threats Open + SCADA ruleset at container start, then launches in AF_PACKET mode

**`ics-sim-zeek`** (zeek/zeek base)  
Extends the official Zeek image with `ics-monitor.zeek`, a site policy script that:
- Loads `base/protocols/modbus` and `base/protocols/dnp3` analyzers
- Fires `Notice::Modbus_Write_Registers` when a WriteMultipleRegisters is observed
- Fires `Notice::DNP3_Direct_Operate` on DNP3 function code 3 (DirectOperate)
- Fires `Notice::Cross_Zone_Traffic` when any Modbus packet crosses zone boundaries
- Logs all ICS protocol connection summaries at teardown

---

### Network / infrastructure

**`ics-sim-firewall`** (Alpine + nftables)  
Entrypoint builds an nftables ruleset from environment variables. Default policy is configurable (`FW_DEFAULT_POLICY`). Comma-separated port lists (`FW_OT_TO_IT_PORTS`, `FW_IT_TO_OT_PORTS`) create per-port accept rules between zones. Always allows established/related and ICMP.

**`ics-sim-switch`** (Alpine + bridge-utils)  
Visible topology node representing a managed switch. Docker's own bridge network already handles L2 forwarding; this container adds a labeled presence in the SCADA diagram and provides `tcpdump` for traffic inspection during exercises.

**`ics-sim-router`** (Alpine + iproute2)  
Enables IP forwarding and shows the routing table at startup. Participates in multi-zone routing when the scenario requires cross-zone device communication.

---

### Attack machine

**`ics-sim-attack-base`** (kalilinux/kali-rolling)  
Pre-installed: nmap, masscan, Metasploit Framework, Scapy (Python), pymodbus, tshark, netcat. Container stays running via `tail -f /dev/null` so the xterm.js terminal panel (Phase 8) can attach with `docker exec -it`. Always placed on the External network segment.

---

## CI/CD pipeline

`.github/workflows/docker.yml` has three job groups:

| Job | Images | Timeout | Trigger |
|-----|--------|---------|---------|
| `build-images` (matrix, 7 parallel) | modbus, dnp3, firewall, switch, router, suricata, zeek | 40 min | push to main OR release |
| `build-openplc` | openplc | 60 min | push to main OR release |
| `build-attack-base` | attack-base | 60 min | main push + release only |

Each job uses GitHub Actions cache (`type=gha`) scoped per image to avoid redundant layer downloads on subsequent builds. Images are tagged `:latest` on main, `sha-{short}` on every build, and `{semver}` on tagged releases.

## Next

Phase 4: PLC ladder logic and Structured Text IDE — SVG rung editor in the canvas properties panel, ST source compilation, and deployment into the running OpenPLC container via its REST API.
