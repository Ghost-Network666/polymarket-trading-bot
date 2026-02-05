# Build Prompt: Self-Hosted Polymarket Trading System

**Goal:** Create a complete, self-hosted Polymarket trading skill that works like Bankr but runs entirely in my VM using the official Polymarket CLOB API.

## What I Need

A skill named `polymarket` that lets me trade prediction markets via natural language, similar to how Bankr worked but without relying on their API. I want complete control and no external dependencies beyond the official Polymarket infrastructure.

## Core Requirements

### 1. Directory Structure
```
~/.config/polymarket/
├── config.json          # Wallet config (private key, signature type, funder address)
└── markets_cache.json   # Optional cache for faster market lookups

/home/claude/polymarket/
├── SKILL.md            # Main skill documentation
├── scripts/
│   ├── trade.py        # Execute trades via CLI
│   ├── market_search.py # Find markets by keyword
│   ├── positions.py    # View positions and orders
│   ├── setup.py        # Initial wallet setup and allowances
│   └── interactive.py  # Interactive trading mode (optional)
└── references/
    └── examples.md     # Common usage examples
```

### 2. Configuration Format

`~/.config/polymarket/config.json`:
```json
{
  "private_key": "0x...",
  "funder_address": "0x...",
  "signature_type": 1,
  "rpc_url": "https://polygon-rpc.com",
  "auto_approve": false,
  "max_slippage": 0.02
}
```

**Signature Types:**
- `0`: EOA wallet (MetaMask, direct private key)
- `1`: Email/Magic wallet (most common)
- `2`: Proxy wallet (Safe, etc)

### 3. Core Functionality

The system should support these natural language commands:

**Trading:**
- "Buy $50 of YES on [market question]"
- "Sell my position in [market]"
- "Place limit order: buy 100 YES at 0.65 on [market]"
- "Market buy $20 NO on [market]"

**Market Discovery:**
- "Find markets about Trump"
- "Show me election markets"
- "What are the hottest markets right now?"
- "Search for AI markets"

**Portfolio Management:**
- "Show my positions"
- "What are my open orders?"
- "Cancel all orders"
- "Cancel order [order_id]"
- "What's my balance?"

**Market Info:**
- "What's the current price for [market]?"
- "Show order book for [market]"
- "Get market details for [condition_id]"

### 4. Technical Implementation

**Python Dependencies:**
```bash
pip install py-clob-client web3 --break-system-packages
```

**Key Components:**

1. **Client Initialization:**
```python
from py_clob_client.client import ClobClient

client = ClobClient(
    host="https://clob.polymarket.com",
    chain_id=137,  # Polygon
    key=config['private_key'],
    signature_type=config['signature_type'],
    funder=config['funder_address']
)
client.set_api_creds(client.create_or_derive_api_creds())
```

2. **Market Search:**
```python
markets = client.get_simplified_markets()
# Filter by keyword, sort by volume/activity
```

3. **Order Placement:**
```python
from py_clob_client.clob_types import OrderArgs
from py_clob_client.order_builder.constants import BUY, SELL

order = OrderArgs(
    token_id="...",
    price=0.65,
    size=10.0,
    side=BUY
)
result = client.create_and_post_order(order)
```

4. **Position Tracking:**
```python
balances = client.get_balances(market_id)
open_orders = client.get_orders()
```

### 5. Smart Features I Want

**A. Fuzzy Market Matching:**
When I say "Buy YES on Trump election", the system should:
1. Search for markets containing "Trump" and "election"
2. Rank by relevance and volume
3. If multiple matches, ask me to confirm which one
4. Extract the correct token_id automatically

**B. Price Protection:**
- Warn if my price is >5% away from current best bid/ask
- Calculate implied probability from price
- Show potential profit/loss before confirming

**C. Smart Defaults:**
- Limit orders by default (safer)
- Use `price = best_ask + 0.01` for market buys
- Use `price = best_bid - 0.01` for market sells

**D. Caching:**
- Cache market list for 5 minutes to reduce API calls
- Store frequently traded markets for faster lookup

### 6. Error Handling

Handle these common errors gracefully:

- **Invalid funder address:** Guide user to find correct proxy address at polymarket.com/settings
- **Insufficient allowance:** Run approval script automatically (EOA only)
- **Ambiguous market:** Show top 3 matches and ask user to pick
- **Order would cross:** Explain POST_ONLY limitation
- **Network errors:** Retry with exponential backoff
- **Insufficient balance:** Show current balance and required amount

### 7. SKILL.md Structure

The SKILL.md should include:

**Frontmatter:**
```yaml
---
name: polymarket
description: Self-hosted Polymarket trading via natural language using official CLOB API. Use when user wants to trade prediction markets, place bets, check positions, view market odds, buy/sell outcome tokens, cancel orders, research markets, or execute any Polymarket operations. Direct API integration with no external dependencies. Supports limit orders, market orders, position management, fuzzy market search, and automated token allowances.
---
```

**Sections:**
1. Quick Start (5-min setup)
2. Configuration Guide
3. Trading Commands & Examples
4. Market Discovery
5. Position Management
6. Understanding Polymarket (prices, odds, binary markets)
7. Error Handling & Troubleshooting
8. Advanced Features (batch orders, POST_ONLY, etc)
9. Security Best Practices
10. API Reference Links

### 8. Helper Scripts

**trade.py** - Main trading interface:
```bash
# Examples:
python trade.py buy "Trump wins" 0.65 10
python trade.py sell "Biden wins" 0.35 5
python trade.py cancel abc123
python trade.py cancel-all
```

**market_search.py** - Find markets:
```bash
python market_search.py "election"
python market_search.py "AI" --sort volume
python market_search.py "Trump" --limit 5
```

**positions.py** - View portfolio:
```bash
python positions.py
python positions.py --market 0x123...
python positions.py --export csv
```

**setup.py** - First-time setup:
```bash
python setup.py --wizard  # Interactive setup
python setup.py --approve-tokens  # Set allowances (EOA only)
```

### 9. Interactive Mode (Optional but Nice)

```bash
python interactive.py
```

```
> search Trump election
Found 3 markets:
[1] Will Trump win 2024 election? (65% YES)
[2] Will Trump debate Biden? (45% YES)
[3] Trump popular vote over 50%? (38% YES)

> buy 1 50 10
Buying $10 of YES on "Will Trump win 2024 election?" at 0.65
Current best ask: 0.648
Confirm? (y/n): y
✓ Order placed! ID: abc123

> positions
Open Positions:
- Trump wins 2024: 15.4 YES tokens ($10 invested, worth $10.01)

Open Orders:
- None

> exit
```

### 10. What NOT to Include

- Don't implement a web interface (CLI only)
- Don't create a database (use JSON files for cache)
- Don't add leverage/margin (Polymarket doesn't support it)
- Don't implement custom contracts (use official CLOB only)
- Don't add social features (just trading)

### 11. Testing Checklist

Before packaging the skill, test:
- [ ] Setup new wallet with wizard
- [ ] Search for markets by keyword
- [ ] Place limit buy order
- [ ] Place limit sell order
- [ ] View positions
- [ ] Cancel individual order
- [ ] Cancel all orders
- [ ] Handle ambiguous market search
- [ ] Error handling for invalid inputs
- [ ] Token approval for EOA wallet

### 12. Documentation Style

- Keep SKILL.md under 500 lines
- Use code examples liberally
- Assume user is technical but new to Polymarket
- Include troubleshooting section
- Link to official Polymarket docs
- Provide copy-paste commands

### 13. Security Requirements

- Never log private keys
- Store config in ~/.config (not in skill directory)
- Warn about signature_type implications
- Remind about Polygon gas fees
- Note that API creds are tied to private key
- Explain funder address security model

## Expected Output

After running this prompt with OpenClaw, I should have:

1. **Working skill** at `/home/claude/polymarket/` that I can package into `polymarket.skill`
2. **Tested scripts** that actually execute trades
3. **Complete documentation** in SKILL.md
4. **Configuration template** ready to fill with my wallet details

## Final Notes

- Focus on reliability over features
- The official `py-clob-client` is the source of truth
- Keep it simple - this replaces Bankr, not extends it
- Make error messages actually helpful
- Test on Polygon mainnet (Polymarket is production only)

Build me this complete system, test it thoroughly, and package it as `polymarket.skill` so I can upload it to OpenClaw and start trading independently.

---

## References for Builder

- Polymarket CLOB API: https://docs.polymarket.com/
- Python Client: https://github.com/Polymarket/py-clob-client
- Example Agents: https://github.com/Polymarket/agents
- Polygon RPC: https://polygon-rpc.com

## Success Criteria

When complete, I should be able to:
1. Say "buy $20 YES on Trump wins" and it works
2. View my positions with one command
3. Search markets in <1 second
4. Never worry about Bankr API being down
5. Understand exactly what's happening under the hood
