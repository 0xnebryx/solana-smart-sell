# solana-smart-sell

> Automated take-profit, stop-loss, and trailing exits for Solana positions — patterns, not products.

Notes on building reliable exit automation for on-chain positions, with the failure modes that show up only in production.

---

## Why "naive TP/SL" doesn't work

The textbook take-profit/stop-loss looks like:

```
loop:
  price = fetchPrice()
  if price >= takeProfit: sell()
  if price <= stopLoss: sell()
  sleep(N)
```

This breaks on Solana for several reasons:

1. **Price feed lag** — DEX-derived prices update at slot speed, but your RPC may be 1–2s behind. The price you act on may already be stale.
2. **MEV sandwiches** — by the time your sell TX is in the mempool, your exit becomes a target.
3. **Slippage on the way out** — illiquid tokens move 5–15% per N-SOL trade, so a 10% TP can collapse to 0% realized.
4. **Bonding-curve impact** — on pump.fun pre-graduation, large sells crash the curve. Your fill price is much worse than the spot.
5. **Partial fills** — for routes through aggregators, partial fills can occur and your "did I exit" check fails.

---

## Better patterns

### 1. Trailing TP with peak tracking

Track the peak price since entry, sell on a percentage drop from peak:

```
peak = max(peak, current_price)
if current_price < peak * (1 - trail_pct): sell()
```

This lets winners run but locks in gains if reversion starts.

### 2. Multi-step ladders (DCA exit)

Instead of one sell at +50%, ladder it:

```
+25% → sell 30% of position
+50% → sell 30% of position
+100% → sell 30% of position
+200% → sell remainder
```

Reduces sandwich impact (smaller TXs = less worth front-running) and smooths exit through reversals.

### 3. Slippage-aware exits

Quote the sell route via aggregator and check the EXPECTED price before submitting. If the route shows worse than your target — split the order, route differently, or wait.

### 4. MEV-protected submission

Send sell TXs through Jito with a tip — this gets you priority inclusion and (for some Jito modes) protection from being front-run by validators running modified clients.

### 5. Failure-recovery loop

Sells fail for many reasons (slippage exceeded, pool drained, RPC error). The exit loop must:
- Distinguish retryable errors (RPC timeout) from terminal (insufficient balance)
- Backoff exponentially on retryable
- Surface terminal errors immediately to the user
- Never silently swallow a failed exit — that defeats the purpose

---

## Sell venues on Solana

| Venue              | When to use                                              |
|--------------------|----------------------------------------------------------|
| pump-fun SDK       | Token still on bonding curve (pre-graduation)            |
| PumpSwap (Raydium) | Graduated tokens, deeper liquidity                       |
| Jupiter v6         | General routing across all DEX liquidity                 |
| Direct CLMM        | Specific Orca/Raydium CLMM pools when slippage matters   |

A smart router picks based on graduation status and pool depth.

---

## Footguns

- **Don't poll RPC every 500ms** — you'll hit rate limits and get IP-banned on free tiers. Use websockets or 2–5s polling.
- **Beware tokens with transfer hooks (Token-2022)** — extensions like `transferHook` and `permanentDelegate` can block or steal your sell. Whitelist the safe extensions you'll touch.
- **Network congestion is binary** — during peak congestion, your TXs may never land. Detect this and pause the exit loop rather than burning fees.
- **Ground truth is on-chain** — your local state of "did I sell" can desync. Always reconcile against on-chain reality before acting.

## Reading list

- [Jupiter v6 swap API](https://station.jup.ag/docs/apis/swap-api)
- [pump-fun SDK on npm](https://www.npmjs.com/package/@nirholas/pump-sdk)
- [Token-2022 extensions](https://spl.solana.com/token-2022/extensions)

## License

MIT
