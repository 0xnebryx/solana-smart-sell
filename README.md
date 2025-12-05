# 📈 Solana Smart Sell Bot

> Automated take-profit, stop-loss, and trailing TP for Solana tokens

[![Smart Sell](https://img.shields.io/badge/Smart%20Sell-Automated-success?style=for-the-badge)](https://obsidianbundler.com)
[![24/7](https://img.shields.io/badge/24%2F7-Protection-blue?style=for-the-badge)](https://obsidianbundler.com)
[![MEV](https://img.shields.io/badge/MEV-Protected-purple?style=for-the-badge)](https://obsidianbundler.com)

## 😴 Stop Watching Charts 24/7

You can't watch charts forever. But the market never sleeps.

Smart Sell automates your exit strategy:
- 📈 **Take Profit**: Automatically sell when target hit
- 📉 **Stop Loss**: Cut losses before they get worse
- 📊 **Trailing TP**: Follow profits up, sell on reversal
- 🎯 **MCAP Targets**: Sell based on market cap milestones

## The Problem

```
What Happens Without Automation:

3:00 AM: Token pumps to 5x while you sleep
4:00 AM: Token dumps back to 1x
8:00 AM: You wake up to see what could have been

With Smart Sell:

3:00 AM: Token hits your 4x take profit
3:00 AM: Bot automatically sells 50%
3:00 AM: Remaining 50% trails with 20% stop
4:00 AM: Trailing stop triggers, sells rest at 3.5x
8:00 AM: You wake up to realized profits
```

## Configuration Options

```typescript
interface SmartSellConfig {
  // Take Profit
  takeProfitPercent?: number;      // e.g., 100 = sell at 2x
  takeProfitSellPercent?: number;  // How much to sell (default: 100%)
  
  // Stop Loss
  stopLossPercent?: number;        // e.g., 50 = sell if down 50%
  
  // Trailing Take Profit
  trailingTpPercent?: number;      // e.g., 20 = trail 20% behind peak
  trailingActivation?: number;     // Activate after X% gain
  
  // MCAP Targets
  mcapTargets?: {
    mcap: number;                  // Target market cap
    sellPercent: number;           // % to sell at this target
  }[];
  
  // Advanced
  partialSells?: boolean;          // Allow multiple partial sells
  jitoProtection?: boolean;        // Use Jito for MEV protection
  notifyOnSell?: boolean;          // Send notifications
}
```

## Quick Start

```typescript
import { SmartSell } from '@obsidian/smart-sell';

const smartSell = new SmartSell({
  rpcUrl: process.env.RPC_URL,
  wallets: myWallets,
  jitoTip: 0.001
});

// Monitor a token
await smartSell.monitor({
  tokenMint: 'TOKEN_ADDRESS',
  entryPrice: 0.0001, // Your entry price
  
  // Take profit at 3x
  takeProfitPercent: 200,
  takeProfitSellPercent: 50, // Sell half
  
  // Stop loss at -40%
  stopLossPercent: 40,
  
  // Trail remaining with 15% stop
  trailingTpPercent: 15,
  trailingActivation: 200 // Activate after 2x
});
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   SMART SELL ENGINE                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              PRICE MONITOR                        │   │
│  │  (Birdeye API - 30 second intervals)              │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │                               │
│           ┌─────────────┼─────────────┐                │
│           ▼             ▼             ▼                │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│     │   Take   │  │   Stop   │  │ Trailing │          │
│     │  Profit  │  │   Loss   │  │    TP    │          │
│     │  Check   │  │  Check   │  │  Check   │          │
│     └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│          │             │             │                 │
│          └─────────────┼─────────────┘                 │
│                        │                               │
│                        ▼                               │
│              ┌──────────────────┐                      │
│              │  CONDITION MET?  │                      │
│              └────────┬─────────┘                      │
│                       │                                │
│            ┌──────────┴──────────┐                    │
│            ▼                     ▼                    │
│      ┌──────────┐         ┌──────────┐               │
│      │   YES    │         │    NO    │               │
│      │ Execute  │         │ Continue │               │
│      │   Sell   │         │ Monitor  │               │
│      └────┬─────┘         └──────────┘               │
│           │                                           │
│           ▼                                           │
│    ┌─────────────────┐                               │
│    │   Jito Bundle   │                               │
│    │ (MEV Protected) │                               │
│    └─────────────────┘                               │
│                                                       │
└───────────────────────────────────────────────────────┘
```

## Condition Examples

### Basic Take Profit + Stop Loss

```typescript
await smartSell.monitor({
  tokenMint: 'TOKEN',
  entryPrice: 0.0001,
  
  takeProfitPercent: 100,  // Sell at 2x
  stopLossPercent: 30      // Sell if down 30%
});
```

### Tiered Take Profit

```typescript
await smartSell.monitor({
  tokenMint: 'TOKEN',
  entryPrice: 0.0001,
  
  tiers: [
    { percent: 100, sellPercent: 25 },  // Sell 25% at 2x
    { percent: 200, sellPercent: 25 },  // Sell 25% at 3x
    { percent: 400, sellPercent: 25 },  // Sell 25% at 5x
    { percent: 900, sellPercent: 25 }   // Sell 25% at 10x
  ]
});
```

### Trailing Take Profit

```typescript
await smartSell.monitor({
  tokenMint: 'TOKEN',
  entryPrice: 0.0001,
  
  // Don't activate until 2x
  trailingActivation: 100,
  
  // Then trail 20% behind the peak
  trailingTpPercent: 20
});

// If token goes: 1x → 3x → 2.4x
// Trailing sells at 2.4x (3x - 20%)
// You capture most of the pump
```

### MCAP-Based Targets

```typescript
await smartSell.monitor({
  tokenMint: 'TOKEN',
  
  mcapTargets: [
    { mcap: 100000, sellPercent: 25 },    // $100K MCAP
    { mcap: 500000, sellPercent: 25 },    // $500K MCAP
    { mcap: 1000000, sellPercent: 25 },   // $1M MCAP
    { mcap: 5000000, sellPercent: 25 }    // $5M MCAP
  ]
});
```

## Multi-Wallet Support

```typescript
// Monitor token across all wallets
await smartSell.monitorAllWallets({
  tokenMint: 'TOKEN',
  
  // Same conditions apply to all wallets
  takeProfitPercent: 150,
  stopLossPercent: 40,
  
  // Sell from all wallets that hold this token
  coordinatedSell: true,
  
  // Use Jito for same-block execution
  useJito: true
});
```

## Notifications

```typescript
smartSell.on('sell', (event) => {
  console.log(`
    🚀 Smart Sell Executed!
    Token: ${event.tokenSymbol}
    Trigger: ${event.trigger} (TP/SL/Trail)
    Price: ${event.price}
    Amount: ${event.amount}
    Profit: ${event.profitPercent}%
    TX: ${event.signature}
  `);
});

smartSell.on('alert', (event) => {
  console.log(`⚠️ ${event.message}`);
});
```

## Production Solution

For a complete smart sell system with UI:

### 👉 [Obsidian Platform](https://obsidianbundler.com)

- ✅ All sell conditions in one interface
- ✅ Multi-wallet support
- ✅ MEV protection on all sells
- ✅ Real-time notifications
- ✅ Works while you sleep
- ✅ Free tier available

## Resources

- 💬 [Telegram](https://t.me/obsidianbundler)
- 🐦 [Twitter](https://x.com/obsidianbundler)
- 🌐 [Platform](https://obsidianbundler.com)

## Disclaimer

Educational code. Automated trading involves risk. Test with small amounts first.

---

⭐ **Star this repo** for more trading automation!

📈 **Automate your exits:** [obsidianbundler.com](https://obsidianbundler.com)
