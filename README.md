# Trading Manager — 32-Agent Portfolio Management System

Multi-agent trading system using Claude + TradingView MCP.

## Architecture

32 agents across 6 asset departments + centralized executive branch.

### Asset Departments (30 agents)
Each of the 6 assets runs a dedicated 5-agent pipeline:

| Role | Count | Responsibility |
|---|---|---|
| Monitor | 6 | Real-time OHLC + volume surveillance |
| Scanner | 6 | Indicator-based signal detection |
| Analyst | 6 | Market bias: Bullish / Bearish / Neutral |
| Artist | 6 | Chart drawings, screenshots, visual evidence |
| Reporter | 6 | Packages findings into standardized briefing |

### Executive Branch (2 agents)
| Role | Responsibility |
|---|---|
| Risk Manager | Validates trades against global portfolio limits |
| Portfolio Manager | Capital allocation, position sizing, execution |

### Assets
`BTCUSD` · `XAUUSD` · `CLOIL` · `EURUSD` · `NAS100` · `SP500`

### Workflow
```
Monitor → Scanner → Analyst + Artist → Reporter → Risk Manager → Portfolio Manager
```

## Stack
- Claude (Anthropic)
- TradingView MCP (CDP bridge to TradingView Desktop)
- TradingView Desktop (port 9222)
