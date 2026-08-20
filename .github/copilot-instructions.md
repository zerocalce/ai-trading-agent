# Copilot Instructions for Nocturne: AI Trading Agent

## Build, Test, and Run Commands

### Setup & Installation

**Install dependencies (Poetry):**
```bash
pip install poetry
poetry install
```

**Create environment file:**
```bash
cp .env.example .env
# Edit .env with your API keys and configuration
```

**Verify installation:**
```bash
poetry run python src/main.py --help
```

### Running the Agent

**Basic usage:**
```bash
poetry run python src/main.py --assets BTC ETH --interval 1h
```

**Single asset:**
```bash
poetry run python src/main.py --assets BTC --interval 4h
```

**Multiple intervals (runs parallel workers):**
```bash
poetry run python src/main.py --assets BTC ETH SOL --interval 1h --interval 4h --interval 1d
```

**Custom trading parameters:**
```bash
poetry run python src/main.py --assets BTC --interval 1h --max-position-size 0.5 --risk-per-trade 2.0
```

### Local API Server

The agent also runs a local API server:

**Access diary entries:**
```bash
curl http://localhost:3000/diary?limit=200
```

**View logs:**
```bash
curl http://localhost:3000/logs?path=llm_requests.log&limit=2000
```

**Configure server:**
```bash
# Set in .env
API_HOST=0.0.0.0
API_PORT=3000
```

### Docker Deployment

**Build image:**
```bash
docker build --platform linux/amd64 -t trading-agent .
```

**Run container:**
```bash
docker run --rm -p 3000:3000 --env-file .env trading-agent
```

**Verify container is running:**
```bash
curl http://localhost:3000/diary
```

### EigenCloud Deployment (TEE)

**Install EigenX CLI:**
```bash
# macOS/Linux
curl -fsSL https://eigenx-scripts.s3.us-east-1.amazonaws.com/install-eigenx.sh | bash

# Windows
curl -fsSL https://eigenx-scripts.s3.us-east-1.amazonaws.com/install-eigenx.ps1 | powershell -
```

**Authenticate:**
```bash
docker login
eigenx auth login
# Or generate new account:
eigenx auth generate --store
```

**Deploy agent:**
```bash
cp .env.example .env
# Edit .env with trading configuration
eigenx app deploy
```

**Monitor deployment:**
```bash
eigenx app info --watch
eigenx app logs --watch
```

**Update live deployment:**
```bash
# After code changes
eigenx app upgrade <app-name>

# Or after .env changes
# Re-deploy with updated environment
```

## High-Level Architecture

### System Overview

**Nocturne** is an **autonomous AI trading agent** that continuously monitors cryptocurrency markets and executes trades based on LLM-driven decision making. Unlike traditional bots with hardcoded rules, Nocturne uses frontier LLMs (OpenRouter: GPT-5 Pro, DeepSeek R1, Grok 4) to analyze market data and make trading decisions.

**Core Loop:**
```
1. Fetch current market data (from TAAPI)
2. Request LLM to analyze and decide (buy/sell/hold)
3. Execute trade on Hyperliquid (DEX)
4. Log decision + outcome
5. Sleep for interval (1h/4h/1d)
6. Repeat
```

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│     User Configuration (.env)               │
├─────────────────────────────────────────────┤
│ • Trading assets (BTC, ETH, SOL, etc.)      │
│ • Time intervals (1h, 4h, 1d)               │
│ • Position sizing rules                     │
│ • Risk parameters                           │
│ • API credentials                           │
└────────────────────┬────────────────────────┘
                     ↓
┌─────────────────────────────────────────────┐
│     main.py (Orchestrator)                  │
├─────────────────────────────────────────────┤
│ • Parse command-line arguments              │
│ • Initialize config                         │
│ • Start market monitoring loops             │
│ • Launch local API server                   │
└────────────────────┬────────────────────────┘
                     ↓
         ┌───────────┴──────────────┐
         ↓                          ↓
    [Worker Thread 1]         [Worker Thread N]
    [1h interval]         [4h/1d/custom interval]
         ↓                          ↓
    For each asset:           For each asset:
         ↓                          ↓
┌──────────────────────────────────────────┐
│ Market Data Fetcher (TAAPI)              │
├──────────────────────────────────────────┤
│ • Fetch OHLCV candles                    │
│ • Calculate technical indicators:        │
│   - EMA (Exponential Moving Average)     │
│   - RSI (Relative Strength Index)        │
│   - MACD (Moving Avg Convergence Div)    │
│   - Bollinger Bands                      │
│   - Volume profile                       │
│   - etc. (dynamic via tool calls)        │
└─────────────┬──────────────────────────────┘
              ↓
┌──────────────────────────────────────────┐
│ LLM Decision Maker (OpenRouter)          │
├──────────────────────────────────────────┤
│ • System prompt: Trading instructions    │
│ • Market context: Recent data + trends   │
│ • Tool available: Get any TAAPI indicat. │
│   (dynamic tool calling)                 │
│ • LLM decides: BUY / SELL / HOLD         │
│ • Returns: Trade action + reasoning      │
└─────────────┬──────────────────────────────┘
              ↓
      [Decision: BUY/SELL/HOLD]
              ↓
      ┌───────┴────────┐
      ↓                ↓
   [HOLD]          [BUY/SELL]
      ↓                ↓
   Log &          Execute Trade
   Continue       (Hyperliquid)
      ↓                ↓
      └────────┬──────┘
               ↓
┌─────────────────────────────────────────────┐
│ Trading Execution (Hyperliquid DEX)         │
├─────────────────────────────────────────────┤
│ • Place order at current market price       │
│ • Set stop-loss (risk management)           │
│ • Set take-profit levels                    │
│ • Monitor position until close              │
│ • Handle slippage & partial fills           │
└─────────────┬───────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│ Logging & Analytics (local files)           │
├─────────────────────────────────────────────┤
│ • diary.jsonl - Decision logs               │
│ • llm_requests.log - API calls              │
│ • trades.log - Execution results            │
│ • errors.log - Error tracking               │
└─────────────┬───────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│ Local API Server (monitoring)               │
├─────────────────────────────────────────────┤
│ • GET /diary - Recent decisions             │
│ • GET /logs - Tail log files                │
│ • Real-time monitoring dashboard            │
└─────────────────────────────────────────────┘
```

### Core Modules

**Location:** `src/`

1. **main.py** (27KB - orchestrator)
   - Entry point: parses CLI arguments
   - Loads configuration from `.env`
   - Initializes Hyperliquid wallet connection
   - Creates worker threads for each asset/interval combo
   - Runs local API server for monitoring
   - Main loop: fetch data → LLM → trade → log

2. **agent/decision_maker.py**
   - LLM integration via OpenRouter API
   - System prompt: Instructions on how to trade
   - Tool calling: Dynamic TAAPI indicator fetching
   - Trade decision logic: Analyzes response, extracts decision
   - Formats trade recommendations with reasoning

3. **indicators/taapi_client.py**
   - Wraps TAAPI.io REST API
   - Fetches technical indicators on demand
   - Supports: EMA, RSI, MACD, Bollinger Bands, Volume, Stoch, Williams %R, etc.
   - Used both for initial analysis and as tool callable by LLM
   - Error handling: retries on network failures

4. **trading/hyperliquid_api.py**
   - Wraps Hyperliquid Python SDK
   - Places orders (market orders primarily)
   - Manages positions (entry, stop-loss, take-profit)
   - Tracks P&L (profit/loss)
   - Executes risk management (position sizing, max drawdown)

5. **config_loader.py**
   - Loads configuration from `.env` file
   - Validates required keys (API keys, trading params)
   - Provides defaults for optional settings
   - Exposes config as singleton instance

6. **utils/** - Helper functions
   - Logging setup (diary.jsonl, structured logs)
   - Data formatting (convert TAAPI response to readable text)
   - Time utilities (interval parsing)
   - Error handling and retries

### Trading Decision Flow

```
┌─ Start Trading Loop ─┐
└──────────┬──────────┘
           ↓
[Interval elapsed?]
           │
      Yes ↓
┌──────────────────────────────────────┐
│ 1. Fetch Market Data (TAAPI)         │
│    • Last 20 candles (OHLCV)         │
│    • Basic indicators (EMA, RSI)     │
│    • Volume and trend analysis       │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│ 2. Prepare LLM Context               │
│    • Current price                   │
│    • Trends (short/medium/long term) │
│    • Recent decisions (history)      │
│    • Account balance & positions     │
│    • Risk parameters                 │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│ 3. Call LLM (OpenRouter)             │
│    • System prompt: "You are trader" │
│    • Market context in user message  │
│    • Tools available:                │
│      - get_taapi_indicator(name, ...) │
│    • Max 5 tool calls per decision   │
│    • LLM can fetch more indicators   │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│ 4. LLM Analysis & Tool Calls         │
│    Step 1: "Need RSI to confirm..."  │
│    → Tool: fetch RSI                 │
│    Step 2: "Check MACD divergence"   │
│    → Tool: fetch MACD                │
│    Step 3: Decision                  │
│    → "Buy BTC with 50% position"     │
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│ 5. Parse Decision                    │
│    • Extract action: BUY/SELL/HOLD  │
│    • Extract size: percentage        │
│    • Extract confidence level        │
│    • Validate against risk limits    │
└──────────────────┬───────────────────┘
                   ↓
        ┌──────────┴──────────┐
        ↓                     ↓
   [HOLD decision]      [BUY/SELL]
        ↓                     ↓
   [Log to diary]       [Execute Trade]
        ↓                     ↓
   [Continue]     [Place order on Hyperliquid]
                            ↓
                    [Set stop-loss]
                            ↓
                    [Set take-profit]
                            ↓
                    [Monitor position]
                            ↓
   ┌────────────────────────────────────────┐
   │ 6. Log Trade & Outcome                 │
   │    • Decision timestamp                │
   │    • LLM reasoning                     │
   │    • Trade execution details           │
   │    • Entry price / stop / target       │
   │    • Expected P&L                      │
   │    • Confidence score                  │
   └────────────────────────────────────────┘
                   ↓
            [Sleep for interval]
                   ↓
            [Loop continues]
```

### Live Trading Agents

Three agents running continuously on public portfolios (seeded with initial capital):

1. **GPT-5 Pro** (funded: $200)
   - Most advanced reasoning
   - Slow (higher cost) but most thoughtful
   - Portfolio: https://hypurrscan.io/address/0xa049db4b3dfcb25c3092891010a629d987d26113
   - Live Logs: https://35.190.43.182/logs/0xC0BE8E55f469c1a04c0F6d04356828C5793d8a9D

2. **DeepSeek R1** (funded: $100, PAUSED)
   - Specialized reasoning model
   - Good cost/benefit ratio
   - Portfolio: https://hypurrscan.io/address/0xa663c80d86fd7c045d9927bb6344d7a5827d31db

3. **Grok 4** (funded: $100, PAUSED)
   - Fast inference
   - Potential for high-frequency trading
   - Portfolio: https://hypurrscan.io/address/0x3c71f3cf324d0133558c81d42543115ef1a2be79

## Key Conventions

### Python Code Style

**Configuration Management:**
```python
from config_loader import get_config

config = get_config()
taapi_key = config.get('TAAPI_API_KEY')
hyperliquid_key = config.get('HYPERLIQUID_PRIVATE_KEY')
model = config.get('LLM_MODEL', 'x-ai/grok-4')  # with default
```

**Async/Await Patterns:**
```python
import aiohttp
import asyncio

async def fetch_market_data(asset: str):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

# Usage in main loop
data = asyncio.run(fetch_market_data('BTC'))
```

**Error Handling with Retries:**
```python
def fetch_with_retries(func, max_retries=3, backoff=2):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(backoff ** attempt)
```

### LLM Integration Pattern

**Tool Calling with OpenRouter:**
```python
# System message tells LLM how to use tools
system = """You are a trading expert. Analyze market data and decide BUY/SELL/HOLD.
You have access to a tool: get_indicator(indicator_name, symbol, timeframe)
Common indicators: RSI, EMA, MACD, BBANDS. Always fetch indicators before deciding."""

# User message with market context
user = f"""Current BTC price: $43,200
Recent trend: +5% in last 4h, -2% in last 1h
Last 5 candles: [OHLCV data]
Position: None
Decision: ?"""

# Tools available
tools = [{
    "type": "function",
    "function": {
        "name": "get_indicator",
        "description": "Fetch technical indicator from TAAPI",
        "parameters": {
            "type": "object",
            "properties": {
                "indicator": {"type": "string", "description": "Indicator name (RSI, EMA, etc)"},
                "symbol": {"type": "string"},
                "timeframe": {"type": "string", "enum": ["1m", "5m", "1h", "4h", "1d"]}
            },
            "required": ["indicator", "symbol"]
        }
    }
}]

response = client.chat.completions.create(
    model="x-ai/grok-4",
    messages=[{"role": "system", "content": system}, {"role": "user", "content": user}],
    tools=tools,
    tool_choice="auto"
)

# Handle tool calls
while response.tool_calls:
    for tool_call in response.tool_calls:
        if tool_call.function.name == "get_indicator":
            indicator_data = fetch_indicator(**tool_call.function.arguments)
            # Continue conversation with indicator data
```

### Trading Decision Format

LLM should respond in format like:
```
Analysis:
- RSI shows oversold (28)
- EMA-20 below EMA-50 (bearish)
- Volume increasing on red candles
- Support at $42,000

Decision: SELL
Reasoning: Bearish crossover + volume confirmation
Position Size: 50% of available capital
Stop Loss: $43,500 (above recent swing high)
Target: $41,000 (previous support)
Confidence: 7/10
```

**Parser extracts:**
- Action: SELL
- Size: 0.5 (50%)
- Stop: $43,500
- Target: $41,000
- Confidence: 0.7

### Logging Structure (diary.jsonl)

Each trade decision logged as JSON line:
```json
{
  "timestamp": "2025-08-20T14:32:00Z",
  "asset": "BTC",
  "interval": "1h",
  "action": "BUY",
  "position_size": 0.5,
  "entry_price": 43200,
  "stop_loss": 41500,
  "take_profit": 45000,
  "confidence": 0.75,
  "reasoning": "RSI 28 + EMA crossover + volume surge",
  "market_context": {
    "current_price": 43200,
    "4h_trend": "neutral",
    "rsi": 28,
    "ema_20_50": "bearish"
  },
  "execution_result": {
    "order_id": "order_12345",
    "status": "filled",
    "fill_price": 43205,
    "slippage": 0.012
  }
}
```

### Position Management

**Risk Parameters (from .env):**
```bash
MAX_POSITION_SIZE=0.5          # Max 50% of account in one trade
MAX_DAILY_LOSS=-0.05           # Stop if down 5% in a day
MIN_STOP_DISTANCE=0.02         # At least 2% from entry
MAX_CONCURRENT_POSITIONS=3     # Hold max 3 open trades
```

**Automated Risk Management:**
```python
def place_trade(symbol, action, size, entry_price):
    # Calculate position sizing
    account_balance = get_account_balance()
    max_size = account_balance * MAX_POSITION_SIZE
    actual_size = min(size * account_balance, max_size)
    
    # Calculate stops
    stop_distance = entry_price * MIN_STOP_DISTANCE
    stop_loss = entry_price - stop_distance if action == 'BUY' else entry_price + stop_distance
    take_profit = entry_price * (1 + 0.03) if action == 'BUY' else entry_price * (1 - 0.03)
    
    # Execute order
    order = hyperliquid_api.place_order(
        symbol=symbol,
        action=action,
        size=actual_size,
        entry=entry_price,
        stop_loss=stop_loss,
        take_profit=take_profit
    )
    
    # Log
    log_trade(order, reason="LLM decision")
```

## Common Development Tasks

### Adding a New Trading Indicator

1. Add to TAAPI client:
   ```python
   def get_atr(self, symbol, timeframe):
       return self.fetch_indicator('atr', symbol, timeframe)
   ```

2. Update LLM tool list in decision_maker.py

3. Include in system prompt examples

### Changing LLM Model

Edit `.env`:
```bash
LLM_MODEL=x-ai/grok-4  # or any OpenRouter model
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
```

Test with:
```bash
poetry run python src/main.py --assets BTC --interval 1h
```

### Monitoring Live Agent

**Check recent decisions:**
```bash
tail -f diary.jsonl | grep BTC
```

**View API costs:**
```bash
grep "llm_cost" llm_requests.log | awk '{sum+=$2} END {print sum}'
```

**Dashboard (local):**
```bash
curl http://localhost:3000/diary?limit=50 | jq '.[] | {timestamp, asset, action, confidence}'
```

### Debugging Failed Trades

1. Check diary.jsonl for decision
2. Check trades.log for execution
3. Check errors.log for network/API errors
4. Verify TAAPI rate limits not exceeded
5. Verify Hyperliquid balance sufficient
6. Test with small position size

### Backtesting Strategy

Currently no built-in backtest, but you can:
1. Export diary.jsonl decisions
2. Calculate hypothetical P&L with historical prices
3. Adjust LLM system prompt based on results
4. Re-run with new model/parameters

### Deployment Monitoring

**For EigenCloud deployment:**
```bash
eigenx app info --watch    # Real-time status
eigenx app logs --watch    # Live logs
```

**For Docker deployment:**
```bash
docker logs -f container_id
curl http://localhost:3000/logs?path=diary.jsonl&limit=100
```

### Cost Optimization

**Main costs:**
1. **LLM API calls** (OpenRouter) - Largest cost
   - ~$0.01-0.05 per decision (varies by model)
   - Solution: Use cheaper models, cache decisions, longer intervals

2. **Hyperliquid trading fees** - 0.02% per trade
   - Solution: Reduce trade frequency, optimize position sizes

3. **TAAPI indicator fetches** - Free tier has limits
   - Solution: Cache indicators, reuse across models

**Cost reduction strategies:**
- Use cheapest viable model (Grok 4 vs GPT-5 Pro)
- Increase interval (4h/1d instead of 1h)
- Reduce asset count
- Implement decision caching
- Batch indicator fetches

## Production Considerations

**Before going live:**
1. Start with small position sizes (test capital)
2. Paper trade first (simulate only)
3. Set conservative risk parameters
4. Monitor for 1 week before increasing sizes
5. Set up alerts for large losses
6. Implement circuit breaker (stop at -X%)

**Key risks:**
- LLM hallucinations / bad decisions
- API failures (TAAPI, Hyperliquid, OpenRouter)
- Network latency / slippage
- Regulatory/exchange downtime
- Account compromise (private key exposure)

**Best practices:**
- Never commit .env (contains private keys)
- Use environment variables for secrets
- Regularly rotate API keys
- Monitor account balance daily
- Review trade logs for anomalies
- Have manual override capability
