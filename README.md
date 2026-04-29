# Trading Manager — 32-Agent Portfolio Management System

Multi-agent trading system using Claude + TradingView MCP + ForexFactory news feed.

## Architecture

32 agents across 6 asset departments + centralized executive branch.

### Data Sources (MCP Layer)
| MCP Server | Data | Used By |
|---|---|---|
| `tradingview` | Live OHLC, indicators, chart drawings, alerts | Monitor, Scanner, Artist |
| `forexfactory` | Economic calendar, high-impact news events | Scanner, Risk Manager |

### Asset Departments (30 agents)
Each of the 6 assets runs a dedicated 5-agent pipeline:

| Role | Count | Responsibility |
|---|---|---|
| Monitor | 6 | Real-time OHLC + volume surveillance via TradingView MCP |
| Scanner | 6 | Indicator rules + FF news filter (avoids red-flag events) |
| Analyst | 6 | Market bias: Bullish / Bearish / Neutral |
| Artist | 6 | Chart drawings, screenshots, visual evidence |
| Reporter | 6 | Packages findings into standardized briefing JSON |

### Executive Branch (2 agents)
| Role | Responsibility |
|---|---|
| Risk Manager | Validates trades against portfolio limits + upcoming FF events |
| Portfolio Manager | Capital allocation, position sizing, fires TV alerts |

### Assets
`BTCUSD` · `XAUUSD` · `CLOIL` · `EURUSD` · `NAS100` · `SP500`

### Currency → News Mapping
| Asset | Watch currencies on FF |
|---|---|
| BTCUSD | USD |
| XAUUSD | USD |
| CLOIL | USD (EIA inventory, NFP) |
| EURUSD | EUR, USD |
| NAS100 | USD (FOMC, CPI, NFP) |
| SP500 | USD (FOMC, CPI, NFP) |

### Workflow
```
[ForexFactory MCP] ──── news filter ────┐
[TradingView MCP]  ──── chart data  ────┤
                                        ↓
Monitor → Scanner → Analyst + Artist → Reporter → Risk Manager → Portfolio Manager
                                                        ↑
                                              checks FF for red events
                                              before approving trade
```

### News-Aware Behavior
- **Scanner**: Skips new entries within 30 min of a red-impact event
- **Risk Manager**: Tightens stops or rejects trades if high-impact event < 15 min away
- **Portfolio Manager**: Logs FF event context in every trade record

## Stack
- Claude (Anthropic)
- TradingView MCP — CDP bridge to TradingView Desktop (`port 9222`)
- ForexFactory MCP — economic calendar scraper via Playwright
- TradingView Desktop (Windows)

## MCP Config (`~/.claude/.mcp.json`)
```json
{
  "mcpServers": {
    "tradingview": {
      "command": "node",
      "args": ["C:/Users/Tenders/tradingview-mcp/src/server.js"]
    },
    "forexfactory": {
      "command": "uv",
      "args": ["run", "ffcal-server"],
      "cwd": "C:/Users/Tenders/forexfactory-mcp"
    }
  }
}
```
