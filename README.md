# Trading Manager — Autonomous Multi-Agent Portfolio System

AI trading system using Claude + TradingView MCP + ForexFactory + Graphify memory.

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| AI | Claude (Anthropic) | All agent reasoning |
| Charts | TradingView Desktop + MCP | Live data, drawings, alerts, Pine Script |
| News | ForexFactory MCP | Economic calendar, red-flag event filter |
| Memory | Graphify | Knowledge graph, trade history, pattern recall |

## Agent Structure (8 agents)

```
orchestrator
    ├── btcusd-specialist
    ├── xauusd-specialist
    ├── cloil-specialist
    ├── eurusd-specialist
    ├── nas100-specialist
    ├── sp500-specialist
    └── risk-portfolio-manager
```

### Specialist Agents (6)
Each runs a **3-phase Deep Intelligence cycle**:

| Phase | What it does |
|---|---|
| Phase 1 — The Eye | 3-timeframe scan (Daily trend · 4H structure · 15m entry) + ForexFactory news check |
| Phase 2 — The Brain | SMC/ICT analysis · volatility · sentiment · institutional narrative |
| Phase 3 — Pine Script | Writes + applies bespoke Pine Script to live chart (FVGs, OBs, liquidity zones) |

Output per specialist:
```json
{
  "symbol": "BTCUSD",
  "market_narrative": "...",
  "institutional_bias": "Bullish/Bearish/Neutral",
  "liquidity_targets": ["level1", "level2"],
  "probability_score": 8,
  "action_plan": "...",
  "entry": 0.0, "sl": 0.0, "tp": 0.0, "r_ratio": 2.3,
  "news_risk": "Low/Med/High"
}
```
If no setup: `STATUS: SCANNING - NO SETUP`

### Risk & Portfolio Manager (1)
Receives all 6 proposals and applies:

| Rule | Limit |
|---|---|
| Risk per trade | 1% of account |
| Max portfolio heat | 5% total |
| Max open trades | 4 |
| Max USD-correlated | 2 simultaneous |
| NAS100 + SP500 together | Blocked (correlated) |
| High news risk | Auto-reject |
| Min probability score | 7 / 10 |
| Min R ratio | 1.5R |
| Daily loss cap | −3% halts all trades |

### Orchestrator (1)
Runs the full cycle: checks state → fires 6 specialists → collects proposals → risk manager → updates state.json.

## Assets

| Symbol | Character | Session |
|---|---|---|
| BTCUSD | Crypto · 24/7 · halving cycles | 24/7 |
| XAUUSD | Gold · DXY/yields inverse · safe-haven | London + NY |
| CLOIL | WTI Oil · OPEC/EIA · geopolitical | London + NY |
| EURUSD | Forex · ECB/Fed divergence · highest liquidity | London + NY |
| NAS100 | Tech index · rate-sensitive · gap fills | NY only |
| SP500 | Broad market · OPEX pins · sector rotation | NY only |

## News Filter

ForexFactory checked every cycle. Red-impact events block new entries:
- Within **30 min** → Scanner blocks entry
- Within **15 min** → Risk Manager tightens SL on open trades
- **Active now** → Portfolio Manager pauses all alerts

## Memory (Graphify)

Knowledge graph auto-rebuilds on every `git commit` (post-commit hook).
Agents query past decisions, win rates, and setup patterns across sessions.

## Setup

```bash
# 1. Launch TradingView with CDP
Start-Process "$env:LOCALAPPDATA\TradingView\TradingView.exe" -ArgumentList "--remote-debugging-port=9222"

# 2. Copy MCP config
cp mcp.config.json ~/.claude/.mcp.json

# 3. Restart Claude Code — all 3 MCP servers load automatically

# 4. Run the cycle
# Type: use orchestrator
```

## MCP Servers

See `mcp.config.json` — copy to `~/.claude/.mcp.json`:
- `tradingview` — CDP bridge to TradingView Desktop
- `forexfactory` — economic calendar via Playwright scraper
- `graphify` — knowledge graph memory server

## Files

```
Trading-manager/
├── .claude/
│   ├── agents/
│   │   ├── orchestrator.md
│   │   ├── btcusd-specialist.md
│   │   ├── xauusd-specialist.md
│   │   ├── cloil-specialist.md
│   │   ├── eurusd-specialist.md
│   │   ├── nas100-specialist.md
│   │   ├── sp500-specialist.md
│   │   └── risk-portfolio-manager.md
│   └── settings.json
├── state.json
├── mcp.config.json
├── ARCHITECTURE.md
├── CLAUDE.md
└── README.md
```
