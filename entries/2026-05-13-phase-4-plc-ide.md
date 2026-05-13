# Phase 4: PLC IDE — Structured Text Editor, Ladder Logic Viewer, OpenPLC Deployment Pipeline

**Date:** 2026-05-13  
**Commit:** `ac91664`  
**Branch:** `main`

---

## What Was Built

Phase 4 adds an integrated PLC programming environment that appears in the right-sidebar properties panel whenever a PLC device node is selected on the SCADA canvas. The goal: a researcher or student can write a complete IEC 61131-3 Structured Text program, map its I/O variables to Modbus registers, visualize it as a ladder logic diagram, and deploy it to a running OpenPLC container — all without leaving the application.

---

## Component Architecture

### `PlcIdePanel.tsx`

The IDE panel is a single React component that manages three pieces of local state:

- **`source`** — the raw Structured Text source string shown in the editor
- **`varRows`** — the variable binding table rows (local until Save is clicked)
- **`statusMsg`** — feedback string (save confirmation, deploy output, errors)

State is initialized from `device.plcProgram` if the scenario already has a saved program (decoded from base64), or from a built-in template otherwise. Changes are **not** propagated to the scenario document on every keystroke — only when the user explicitly clicks Save or Deploy. This prevents O(keystrokes) LevelDB writes.

The component is mounted in `PropertiesPanel.tsx` via a guard on `device.category === 'plc'`. Non-PLC devices still see the standard property rows. The panel widens to 320px when in PLC mode (`plc-mode` CSS class) to give the ST editor more horizontal space.

### Structured Text Editor

A `<textarea>` with:

- `font-family: 'Cascadia Code', 'Fira Code', monospace`
- `white-space: pre` / `overflow-x: auto` for correct indentation display
- `spellCheck={false}` (IEC identifiers should not trigger spell check)
- `resize: vertical` so users can expand the editor as needed
- `min-height: 200px` — enough for a typical single PROGRAM block

The template pre-loaded in the editor demonstrates a water treatment pump control scenario with six variables (level switches, flow rate, pump coil, inlet valve, alarm) to orient new users to the IEC 61131-3 variable declaration syntax.

### Variable Binding Table

Maps PLC internal variables to their field-bus protocol addresses:

| Column | Content |
|--------|---------|
| Name | IEC 61131-3 variable identifier |
| Type | BOOL / WORD / INT / DINT / REAL |
| IEC Addr | `%IX0.0` (input bit), `%QX0.0` (output bit), `%IW0` (input word), `%MW0` (memory word) |
| Proto | modbus-tcp / modbus-rtu / dnp3 / opc-ua / none |
| Reg | Modbus register number or DNP3 point index |

Rows are editable inline. Add/delete buttons maintain the list. The `VarRow` shape adds a stable React `id` key to the schema's `PLCProgramConfig.variables` type, keeping render reconciliation correct across mutations.

The helper `iecDirection(addr)` infers whether a variable is an input (`%I` prefix), output (`%Q` prefix), or memory (`%M` prefix) — used by both the ladder viewer and the save serializer.

### Ladder Logic SVG Viewer

A read-only SVG diagram generated from the variable binding table. This is pedagogically important: IEC 61131-3 defines five PLC programming languages, and the relationship between Structured Text (textual) and Ladder Diagram (graphical) is a core concept in ICS security courses.

Symbols rendered (per IEC 61131-3 §6.7.2):

```
Left power rail                Right power rail
     |                              |
     |---[  level_sw  ]---( pump_run )---|    ← Rung 1
     |---[  sensor_in ]---( alarm_out )--|    ← Rung 2
     |                              |
```

- **Normally-open contact** `--|[ ]|--` — BOOL inputs (`%IX` addresses), drawn in `--accent-teal`
- **Coil** `--( )--` — BOOL outputs (`%QX` addresses), drawn in `--accent-orange`

Implementation notes:
- SVG `viewBox="0 0 220 H"` scales to 100% panel width
- One rung per output variable; all input variables appear as series contacts on that rung
- This is a simplified topology (all inputs → all outputs) that is educationally accurate for the majority of introductory lab scenarios
- Analog variables (WORD/INT/REAL) appear in a chip row below the ladder rail since they cannot be represented as contacts/coils

### Save Workflow

`handleSave()`:
1. Encodes the ST source to base64 with `btoa(source)` — matching the `PLCProgramConfig.source` schema field type
2. Builds a `PLCProgramConfig` object with `language: 'st'`, the encoded source, and the variable row array
3. Calls `onProgramChange(device.nodeId, program)` which propagates to `App.handleProgramChange`
4. `handleProgramChange` does a deep-merge update into `scenario.devices.devices[nodeId]`
5. The updated scenario flows to LevelDB via the existing `saveActiveScenario` persistence path (on next simulation start)

The "saved" program is then picked up at simulation launch time via `INITIAL_PROGRAM_B64` (see compose generator section).

### Deploy Workflow

`handleDeploy()`:
1. Calls `handleSave()` first (always synchronizes scenario with container state)
2. Invokes `window.electronAPI.plc.deploy(nodeId, source)` via IPC
3. Main process `plc:deploy` handler:
   - Looks up `activePlcPorts.get(nodeId)` — the published host port for this container
   - If no simulation running: returns a "save succeeded, restart to deploy" notice
   - Otherwise: calls `deployToOpenPLC(hostPort, source, nodeId)`
4. Displays compiler output or error in the `.plc-deploy-output` pre block

Deploy button is disabled when `simRunning === false` (the `disabled` prop) with a tooltip explaining why.

---

## Compose Generator Changes

Two additions to `compose-generator.ts`:

### Port Publishing

```typescript
const PLC_WEB_PORT_BASE = 18080
let plcPortIndex = 0
// ...inside device loop:
if (device.category === 'plc') {
  services[serviceName].ports = [`${PLC_WEB_PORT_BASE + plcPortIndex}:8080`]
  plcPortIndex++
}
```

Each PLC container's OpenPLC web interface (port 8080) is published on the host at `18080`, `18081`, etc. The port assignment is deterministic — the `activePlcPorts` map in `main/index.ts` is built using the identical `Object.entries` iteration order, so the IPC handler can always resolve `nodeId → host port` without parsing the compose file.

Port 18080+ was chosen to avoid collision with common development ports (3000, 4200, 5173, 8080, 8443).

### INITIAL_PROGRAM_B64 Environment Variable

```typescript
if (device.plcProgram?.source) {
  env.push(`INITIAL_PROGRAM_B64=${device.plcProgram.source}`)
  env.push(`PLC_VAR_COUNT=${device.plcProgram.variables.length}`)
}
```

Added to `buildDeviceEnv()`. The `source` field is already base64 (stored that way in `PLCProgramConfig`), so it drops straight into the compose env without re-encoding.

---

## OpenPLC Entrypoint Changes

`containers/openplc/entrypoint.sh` now handles pre-load at container startup:

```bash
if [ -n "${INITIAL_PROGRAM_B64}" ]; then
  echo "${INITIAL_PROGRAM_B64}" | base64 -d > "${PROGRAM_DIR}/${DEVICE_ID}.st"
  sqlite3 openplc.db \
    "UPDATE Settings SET Value='${DEVICE_ID}.st' WHERE Key='Prog_Name';"
fi
exec ./openplc "$@"
```

The SQLite write registers the uploaded file as the active program in OpenPLC's persistent settings database. On startup, the runtime reads `Prog_Name` from the Settings table, compiles the ST file with the IEC-to-C transpiler (matiec), and begins the PLC scan cycle. The result: the PLC is executing the user's custom program from the first Docker container heartbeat.

---

## Main Process IPC Handlers

### `plc:deploy`

Full authentication + upload workflow using Node.js `http` module:

1. `POST /login` — URL-encoded form data (`username=openplc&password=openplc`)
2. Extract `Set-Cookie` session cookie from the 302 redirect response
3. `POST /upload-program` — multipart/form-data body with the ST file
4. `GET /start_plc` — triggers IEC-to-C compilation and restarts the scan cycle
5. Scrape `<pre>` compiler output from the HTML response for display in the IDE

The Node.js `http` module is used directly (rather than global `fetch`) because it gives us individual `Set-Cookie` header access without the WHATWG cookie API's cross-origin restrictions.

```typescript
const rawCookie = loginResp.headers['set-cookie']  // string[] | undefined in Node.js
const sessionCookie = rawCookie
  ? rawCookie.map(c => c.split(';')[0]).join('; ')
  : ''
```

Default OpenPLC credentials (`openplc`/`openplc`) are the install-script defaults. In production ICS environments, credentials would be rotated — this is intentionally documented in the code as a lab-context design choice.

### `plc:status`

Polls `GET /dashboard`, scrapes the "PLC Status: Running" text for a lightweight health check. Returns `running: false` gracefully on timeout or connection refused (e.g., container still starting).

### `activePlcPorts` Map

```typescript
const activePlcPorts = new Map<string, number>()
```

Populated in `simulation:start` before calling `generateCompose()`. Cleared in `simulation:stop`. The map is module-level (not per-handler) because it represents simulation-scoped state shared between `plc:deploy` and `plc:status` calls.

---

## Schema / IPC Types

Two new types added to `packages/schema/src/ipc.ts`:

```typescript
interface PLCDeployResult {
  ok: boolean
  output?: string   // IEC compiler output lines
  error?: string
}

interface PLCRuntimeStatus {
  nodeId: string
  running: boolean
  program?: string  // filename of loaded program
  error?: string
}
```

And two new channels in `IPCChannels`:

```typescript
'plc:deploy': [{ nodeId: string; source: string }, PLCDeployResult]
'plc:status': [{ nodeId: string }, PLCRuntimeStatus]
```

Both channels follow the existing `ipcMain.handle` / `ipcRenderer.invoke` pattern documented in `preload/index.ts`.

---

## Key Design Decisions

**Why base64 for ST source storage?**  
`PLCProgramConfig.source` was defined as base64 in Phase 0 schema design. Base64 avoids JSON escaping issues with ST source code (which contains `(*`, `*)`, and quote characters). `btoa/atob` in the browser and `base64 -d` in the shell handle encode/decode symmetrically.

**Why multipart upload instead of docker cp?**  
Using the OpenPLC web API keeps the deployment path identical for both local and future remote deployments. `docker cp` would require a running Docker CLI call from the main process and would bypass OpenPLC's compilation pipeline (the file needs to be compiled before the scan cycle can pick it up). The HTTP API triggers the built-in `matiec` compiler and restarts the PLC atomically.

**Why `http` module instead of global `fetch`?**  
`IncomingMessage.headers['set-cookie']` returns `string[] | undefined` in Node.js — each `Set-Cookie` header is a separate array element. Global `fetch`'s `Headers.get('set-cookie')` concatenates them with commas, which would corrupt cookies containing comma-separated attributes. The `http` module gives us the raw array.

**Why deterministic port assignment instead of `docker port` lookup?**  
`docker port` requires a Docker CLI invocation, which introduces latency and a subprocess dependency. Since the compose generator and the main process use identical `Object.entries` ordering (guaranteed by ECMAScript spec for string keys in insertion order), the port mapping can be computed without any I/O.

---

## Next Phase

**Phase 5: DNP3 IED configuration and DNP3 outstation simulation**  
The `containers/dnp3/` directory exists but has no implementation yet. Phase 5 will:
- Write the pure-Python DNP3 outstation server (using `pydnp3` or a compatible library)
- Add a DNP3 configuration panel in the PropertiesPanel for IED devices
- Add Zeek-based DNP3 traffic analysis (`.zeek` script for function code logging)
- Document NERC CIP-005 and IEC 62351 context for the DNP3 security lab scenarios
