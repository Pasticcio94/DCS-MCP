# DCS MCP — AI Battlefield Commander for DCS World

An [MCP](https://modelcontextprotocol.io) server and Lua bridge that lets an LLM (e.g. Claude) act as a **high-level battlefield commander** in live [DCS World](https://www.digitalcombatsimulator.com/) missions.

The AI wakes on a timer or on a field event, reads the tactical situation, issues orders, then hibernates until the next trigger.

> **Status:** early development — Step 1 MVP (transport + basic tools) is complete. See the [Roadmap](#roadmap).

---

## Concept

The AI operates at **command level** (logistics, attack, defense, objective control, patrols). It does **not** micro-manage single units (fuel, ammo, etc.).

Wake-up conditions:

- **Timed cadence** — e.g. every 30 minutes of mission time
- **Field events** — e.g. enemy aircraft detected in a zone

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ 1. TRIGGERS         timer cadence · field events        │
└───────────────────────────┬─────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────┐
│ 2. AI DECISION      Commander Runtime  +  LLM (Claude)  │
└───────────────────────────┬─────────────────────────────┘
                            ▼  MCP (stdio)
┌─────────────────────────────────────────────────────────┐
│ 3. DCS EXECUTION    Python MCP server (FastMCP)         │
│                          │  TCP · newline-delimited JSON│
│                     Lua bridge inside DCS               │
└─────────────────────────────────────────────────────────┘
```

| Layer | Language | Responsibility |
|---|---|---|
| Lua bridge (`bridge.lua`) | Lua | In-DCS logic, non-blocking TCP listener, command handlers |
| MCP server (`server.py`) | Python | Exposes MCP tools, async client to the bridge |
| Commander Runtime | Python *(planned)* | Wake/hibernate orchestration, trigger handling |

**Why a separate Commander Runtime?** MCP is pull-based: a server cannot wake the model by itself. Scheduling and event-driven wake-ups therefore live *above* MCP.

## Features

- [x] Custom TCP transport — DCS is the server, Python is the client
- [x] Newline-delimited JSON protocol with request/response ID matching
- [x] Non-blocking Lua listener driven by `timer.scheduleFunction` (no server stalls)
- [x] Self-contained Lua JSON codec (no external dependencies)
- [x] MCP tools: `ping`, `get_mission_time`
- [ ] `get_sitrep` — zone-based situation report
- [ ] `exec_order` — high-level order execution
- [ ] Commander Runtime (timer + event triggers)
- [ ] Headless test suite

## Repository layout

```
.
├── lua/
│   └── bridge.lua              # DCS-side TCP bridge
├── python/
│   ├── dcs_bridge_client.py    # Async TCP client
│   └── server.py               # FastMCP server
├── docs/
│   └── setup.md                # De-sanitization & mission load order
└── pyproject.toml
```

> Adjust paths to match your actual tree.

## Requirements

- DCS World (a **dedicated server instance** is recommended)
- Python ≥ 3.10
- [MIST](https://github.com/mrSkortch/MissionScriptingTools) and/or [MOOSE](https://github.com/FlightControl-Master/MOOSE) loaded in the mission
- An MCP-capable client (Claude Desktop, Claude Code, …)

## Installation

**1. De-sanitize DCS** — the bridge needs socket access, which DCS blocks by default in the mission scripting environment. Edit `MissionScripting.lua` in your DCS install as described in [`docs/setup.md`](docs/setup.md).

> ⚠️ De-sanitizing removes DCS security restrictions. Use it only on a server you control and with missions you trust.

**2. Load the bridge in your mission** — add `bridge.lua` via a *DO SCRIPT FILE* trigger action, **after** MIST/MOOSE (load order matters, see setup guide).

**3. Install the MCP server**

```bash
git clone https://github.com/<your-user>/<your-repo>.git
cd <your-repo>
pip install -e .
```

**4. Register it in your MCP client**

```json
{
  "mcpServers": {
    "dcs": {
      "command": "python",
      "args": ["python/server.py"]
    }
  }
}
```

## Usage

1. Start DCS and load the mission (the bridge starts listening).
2. Start your MCP client — it connects to the bridge over TCP.
3. Ask the model to call `ping` or `get_mission_time` to verify the link.

### Wire protocol (illustrative)

```json
→ {"id": 1, "method": "ping"}
← {"id": 1, "result": "pong"}
```

## Design decisions

- **MVP first** — complexity is deferred until simpler approaches are validated.
- **Custom TCP over DCS-gRPC** — simpler and sufficient for the MVP.
- **Lightweight state dumps** — with ~1000 units, a full dump must not block the server, so state is dumped **progressively, split by zone**.
- **Testable without DCS GUI** — DCS needs a live GUI, so the bridge is designed for dependency injection (fake socket / timer) to enable headless tests.

## Roadmap

1. ~~**Step 1** — TCP transport, `ping`, `get_mission_time`~~ ✅
2. **Step 2** — `get_sitrep`, `exec_order`
3. **Testing** — headless harness with injected fake socket/timer
4. **Live testing** — full transport test with DCS running
5. **Commander Runtime** — timer cadence, event-driven wake-up, full AI loop

## Contributing

Issues and PRs are welcome. Please open an issue first for larger changes.

## License

_TBD — add a `LICENSE` file (e.g. MIT)._

## Disclaimer

Not affiliated with Eagle Dynamics. *DCS World* is a trademark of Eagle Dynamics SA.
