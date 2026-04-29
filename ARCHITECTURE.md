# Trading Manager — System Architecture

## Overview

32-agent autonomous portfolio management system.
Connects to TradingView Desktop via CDP, reads ForexFactory news, and maintains a
persistent knowledge graph (Graphify) as shared memory across all agents.

---

## MCP Layer (3 servers)

| Server | Source | Tools | Used By |
|---|---|---|---|
| `tradingview` | tradingview-mcp (CDP) | chart data, OHLCV, drawings, alerts, screenshots | Monitor, Scanner, Artist |
| `forexfactory` | forexfactory-mcp (Playwright) | `ffcal_get_calendar_events` | Scanner, Risk Manager |
| `graphify` | graphify knowledge graph | `query_graph`, `get_node`, `get_neighbors`, `shortest_path` | All agents (memory) |

Copy `mcp.config.json` → `~/.claude/.mcp.json` to activate all 3 servers.

---

## Agent Structure (32 agents total)

### Asset Departments — 30 agents (6 assets × 5 roles)

```
BTCUSD Department          XAUUSD Department          CLOIL Department
  btcusd-monitor             xauusd-monitor             cloil-monitor
  btcusd-scanner             xauusd-scanner             cloil-scanner
  btcusd-analyst             xauusd-analyst             cloil-analyst
  btcusd-artist              xauusd-artist              cloil-artist
  btcusd-reporter            xauusd-reporter            cloil-reporter

EURUSD Department          NAS100 Department          SP500 Department
  eurusd-monitor             nas100-monitor             sp500-monitor
  eurusd-scanner             nas100-scanner             sp500-scanner
  eurusd-analyst             nas100-analyst             sp500-analyst
  eurusd-artist              nas100-artist              sp500-artist
  eurusd-reporter            nas100-reporter            sp500-reporter
```

### Executive Branch — 2 agents

```
  risk-manager
  portfolio-manager
```

---

## Role Definitions

| Role | MCP Tools | Input | Output |
|---|---|---|---|
| **Monitor** | `quote_get`, `data_get_ohlcv`, `chart_get_state` | Live chart | OHLCV snapshot → state.json |
| **Scanner** | `data_get_study_values`, `ffcal_get_calendar_events` | OHLCV + rules | Signal hit or no-hit |
| **Analyst** | `data_get_pine_lines`, `data_get_pine_labels` | Scanner signal | Bias: BULL / BEAR / NEUTRAL + confidence |
| **Artist** | `draw_shape`, `capture_screenshot`, `draw_clear` | Analyst verdict | Chart drawings + screenshot path |
| **Reporter** | `query_graph`, `save_result` | All dept outputs | Standardised briefing JSON → Risk Manager |
| **Risk Manager** | `ffcal_get_calendar_events`, `query_graph` | All 6 briefings | APPROVE / REJECT + reason |
| **Portfolio Manager** | `alert_create`, `chart_set_symbol` | Approved trades | TV alert + position size + trade log |

---

## Workflow

```
Every cycle (scheduled or on-demand):

[ForexFactory MCP] ─── upcoming red events ──────────────────────┐
[TradingView MCP]  ─── live price + indicators ──────────────────┐│
[Graphify MCP]     ─── historical patterns + past decisions ─────┐││
                                                                  │││
  For each asset [BTCUSD, XAUUSD, CLOIL, EURUSD, NAS100, SP500]: │││
                                                                  │││
  Monitor ──reads chart──────────────────────────────────────────┘││
     ↓                                                             ││
  Scanner ──checks rules + FF news filter ───────────────────────┘│
     ↓                                                             │
  Analyst ──interprets bias ─────────────────────────────────────┘
     ↓
  Artist ──draws zones/levels on chart──── screenshot
     ↓
  Reporter ──packages briefing JSON
     ↓
  ┌──────────────────────────────────────┐
  │          RISK MANAGER                │
  │  • checks portfolio exposure         │
  │  • checks FF events < 30 min         │
  │  • checks graphify for past failures │
  │  • APPROVE or REJECT per asset       │
  └───────────────┬──────────────────────┘
                  ↓
  ┌──────────────────────────────────────┐
  │        PORTFOLIO MANAGER             │
  │  • sizes position (max 2% risk)      │
  │  • fires TradingView alert           │
  │  • logs to trade journal             │
  │  • saves result to graphify memory   │
  └──────────────────────────────────────┘
```

---

## State Bus (state.json)

Shared file read/written by all agents each cycle.

```json
{
  "cycle": 42,
  "last_run": "2026-04-29T14:00:00Z",
  "daily_cap_remaining": 3,
  "total_exposure_pct": 4.2,
  "assets": {
    "BTCUSD": {
      "last_scan": "2026-04-29T13:55:00Z",
      "signal": "LONG",
      "bias": "BULL",
      "confidence": 0.78,
      "open_position": false,
      "daily_trades": 1,
      "cooldown_until": null,
      "last_alert_ts": "2026-04-29T10:30:00Z"
    }
  }
}
```

---

## News Filter Logic (ForexFactory)

| Impact | Action |
|---|---|
| Red (high) within 30 min | Scanner blocks new entries for affected currency |
| Red (high) within 15 min | Risk Manager tightens SL on open trades |
| Red (high) active now | Portfolio Manager pauses all new alerts |
| Orange (medium) | Analyst flags as elevated-risk in briefing |

### Currency mapping
| Asset | Watch |
|---|---|
| BTCUSD | USD |
| XAUUSD | USD |
| CLOIL | USD |
| EURUSD | EUR, USD |
| NAS100 | USD |
| SP500 | USD |

---

## Memory (Graphify)

Graphify builds a queryable knowledge graph from all agent `.md` files,
`state.json`, and logged trade decisions.

- **Auto-rebuild**: `post-commit` git hook rebuilds graph on every commit
- **Query tools**: `query_graph`, `get_node`, `get_neighbors`, `shortest_path`
- **Use cases**:
  - Reporter checks: *"last 5 times BTCUSD had this pattern, what happened?"*
  - Risk Manager checks: *"has CLOIL + SP500 double-long failed before?"*
  - Portfolio Manager checks: *"what's our win rate on XAUUSD BULL signals?"*

---

## Self-Sustainability Features

| Feature | Mechanism |
|---|---|
| Continuous cycle | Orchestrator loops on a timer |
| State persistence | `state.json` survives restarts |
| Memory persistence | `graphify-out/graph.json` survives restarts |
| Auto graph rebuild | Git post-commit hook |
| News awareness | ForexFactory checked every cycle |
| Session gating | Metals = London+NY, Crypto = 24/7, Forex = London+NY |
| Cooldown tracking | `cooldown_until` per asset in state.json |
| Daily reset | Counters reset at 00:00 UTC |
| Error isolation | One asset failure does not stop other departments |

---

## File Structure

```
Trading-manager/
├── .claude/
│   ├── agents/
│   │   ├── orchestrator.md
│   │   ├── btcusd-monitor.md
│   │   ├── btcusd-scanner.md
│   │   ├── btcusd-analyst.md
│   │   ├── btcusd-artist.md
│   │   ├── btcusd-reporter.md
│   │   ├── xauusd-*.md
│   │   ├── cloil-*.md
│   │   ├── eurusd-*.md
│   │   ├── nas100-*.md
│   │   ├── sp500-*.md
│   │   ├── risk-manager.md
│   │   └── portfolio-manager.md
│   └── settings.json
├── rules/
│   ├── btcusd.json
│   ├── xauusd.json
│   ├── cloil.json
│   ├── eurusd.json
│   ├── nas100.json
│   └── sp500.json
├── state.json
├── logs/
│   └── cycle.log
├── graphify-out/
│   ├── graph.json
│   └── GRAPH_REPORT.md
├── mcp.config.json
├── ARCHITECTURE.md
├── CLAUDE.md
└── README.md
```
