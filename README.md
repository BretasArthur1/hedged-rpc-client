# Hedged RPC Client

A high-performance Solana RPC client that implements **request hedging** to reduce tail latency in distributed systems.

## The Problem

In distributed systems, even reliable RPC providers occasionally have slow responses (tail latency). Waiting for a single slow provider can degrade your application's performance.

## The Solution: Request Hedging

Instead of waiting for one provider, **race multiple providers** and use the fastest response:

```
Time ────────────────────────────────────────────▶

Provider A:  ████████████████████░░░░ (400ms - slow)
Provider B:  ████████░░░░ (200ms - WINNER! ✓)
Provider C:  ████████████████░░░░ (300ms)

Result: 200ms response time (instead of 400ms)
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                       Your Application                           │
│                    (hedged-rpc-client)                           │
└─────────────┬────────────────────────────────────────────────────┘
              │
              │ Single Logical Request
              │
              ▼
    ┌─────────────────────┐
    │  Hedging Strategy   │
    │  • Initial: 1-3     │ ◄── Configurable
    │  • Delay: 20-100ms  │
    │  • Timeout: 1-3s    │
    └─────────┬───────────┘
              │
              │ Fan-out to multiple providers
              │
     ┌────────┼────────────────┐
     │        │                │
     ▼        ▼                ▼
┌─────────┐ ┌─────────┐ ┌──────────┐
│ Helius  │ │ Triton  │ │QuickNode │
│  RPC    │ │  RPC    │ │   RPC    │
└────┬────┘ └────┬────┘ └─────┬────┘
     │           │            │
     │ 200ms     │ 150ms ✓    │ 400ms
     │           │            │
     └───────────┴────────────┘
                 │
                 │ First successful response wins
                 ▼
         ┌──────────────────┐
         │  Return Result   │
         │  Provider: Triton│
         │  Latency: 150ms  │
         └──────────────────┘
```

## Features

### Core Library
- **Hedged Requests**: Race multiple RPC providers, return fastest response
- **Performance Tracking**: Per-provider statistics (wins, latency, errors)
- **Flexible Strategies**: Conservative, balanced, or aggressive hedging
- **Slot Validation**: Reject stale responses based on slot freshness
- **Fully Async**: Built on Tokio and Solana's nonblocking RPC client

### Interactive TUI Dashboard
- **Real-time Charts**: Latency sparklines, win rate bars, session analytics
- **Live Testing**: Single calls, batch mode, or continuous stress testing
- **Performance Metrics**: Calls/sec, success rate, average latency
- **Provider Comparison**: See which provider performs best in real-time

## Quick Start

### As a Library

```rust
use hedged_rpc_client::{HedgedRpcClient, HedgeConfig, ProviderConfig, ProviderId};
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let providers = vec![
        ProviderConfig {
            id: ProviderId("helius"),
            url: "https://mainnet.helius-rpc.com".to_string(),
        },
        ProviderConfig {
            id: ProviderId("triton"),
            url: "https://api.mainnet-beta.solana.com".to_string(),
        },
    ];

    let config = HedgeConfig::low_latency(providers.len());
    let client = HedgedRpcClient::new(providers, config);

    // Races both providers, returns fastest response
    let (provider, blockhash) = client.get_latest_blockhash().await?;
    println!("Winner: {} with blockhash: {}", provider.0, blockhash);

    Ok(())
}
```

### Interactive Dashboard

```bash
# Set your RPC endpoints
export HELIUS_RPC_URL="https://mainnet.helius-rpc.com/?api-key=YOUR_KEY"
export TRITON_RPC_URL="https://api.mainnet-beta.solana.com"
export QUICKNODE_RPC_URL="https://your-endpoint.quiknode.pro/YOUR_KEY"

# Launch the TUI
cargo run --release
```

**Controls:**
- `↑/↓` - Select provider
- `Space` - Quick test selected provider
- `Tab` - Toggle between Hedged/Single mode
- `r` - Run single call
- `b` - Toggle batch mode (auto-run multiple calls)
- `+/-` - Adjust provider count (Hedged mode)
- `,/.` - Adjust batch count
- `s` - Reset statistics
- `q` - Quit

## Hedging Strategies

### Conservative (Default)
- Queries 1 provider initially
- Hedges after 100ms if no response
- Best for: Production with cost constraints

### Low Latency
- Races 2 providers immediately  
- Hedges after 20ms
- Best for: Performance-critical applications

### Aggressive
- Races 3 providers immediately
- Hedges after 20ms
- Best for: Maximum speed, low latency requirements

### Custom
```rust
let config = HedgeConfig {
    initial_providers: 2,
    hedge_after: Duration::from_millis(50),
    max_providers: 3,
    min_slot: None,
    overall_timeout: Duration::from_secs(2),
};
```

## When to Use Hedging

**Good for:**
- High-traffic applications where latency matters
- Systems with multiple available RPC providers
- Trading bots, MEV, or time-sensitive operations
- Applications that can tolerate slightly higher RPC costs

**Not ideal for:**
- Single provider setups
- Cost-extremely-sensitive operations
- Non-time-critical background jobs

## Examples

See `examples/` directory:
- `basic_get_account.rs` - Simple hedged request example
- `rpc_race.rs` - Many-call stress test with statistics
- `dual_race.rs` - Compare two concurrent runners

## Why Hedging Works

Traditional approach (sequential):
```
Request → Provider A (slow: 800ms) → Timeout → Retry Provider B → Success
Total time: 800ms+ (or timeout)
```

Hedged approach (parallel):
```
Request → Provider A (slow: 800ms) ──┐
       → Provider B (fast: 150ms) ──┼→ Success! 
       → Provider C (medium: 300ms)─┘
Total time: 150ms (fastest wins)
```

## Implementation Details

For Rust/Solana developers interested in the internals:

### Core Racing Logic
- **`FuturesUnordered` + `tokio::select!`**: Races provider futures efficiently without spawning tasks per provider
- **First successful wins**: The first `Ok(T)` response completes the call; remaining in-flight futures are dropped automatically
- **No cancellation tokens needed**: Tokio's select! handles cleanup when one branch completes

### Stats Collection
- **Shared state**: `Arc<Mutex<HashMap<ProviderId, ProviderStats>>>` tracks wins, latency, and errors
- **Lock-free reads**: Stats snapshots are created on-demand without blocking the hot path
- **Per-provider metrics**: Each provider accumulates independent performance data

### Timeout & SLA
- **Outer timeout**: `tokio::time::timeout()` enforces the `overall_timeout` as a hard SLA
- **Graceful degradation**: If all providers fail before timeout, returns `HedgedError::AllFailed`
- **No retry logic**: Fails fast and returns control to the caller

### Hedging Strategy
```rust
// Phase 1: Query initial_providers immediately
for provider in initial_providers {
    futures.push(call_provider(provider));
}

// Phase 2: If no response after hedge_after, fan out
tokio::select! {
    Some((id, Ok(val))) = futures.next() => return Ok((id, val)),
    _ = sleep(hedge_after) => {
        // Add remaining providers to the race
        for provider in remaining_providers {
            futures.push(call_provider(provider));
        }
    }
}
```

This design ensures minimal overhead while maximizing responsiveness.

## Resources

- [Request Hedging Pattern](https://medium.com/javarevisited/request-hedging-a-concurrency-pattern-every-senior-engineer-should-know-bdfaa2da8d40)
- [The Tail at Scale (Google Research)](https://cacm.acm.org/research/the-tail-at-scale/)

## License

MIT
