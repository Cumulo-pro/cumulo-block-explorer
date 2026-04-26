# Monad Explorer — Technical Documentation

> This document covers the Monad-specific implementation of the Cumulo Block Explorer.
> Monad uses a **fundamentally different architecture** from the Cosmos SDK chains documented
> in `architecture.md` — there is no backend collector, no CometBFT RPC, and no Cosmos REST API.

---

## Overview

The Monad Explorer is a real-time EVM block explorer with two deployments sharing the same codebase:

| Instance | URL | Chain ID |
|---|---|---|
| Monad Testnet | `explorer.monad.cumulo.com.es` | 10143 |
| Monad Mainnet | `monad.cumulo.com.es` | 143 |

All data is fetched **directly from public Monad RPC endpoints** by the user's browser at page load and on a rolling refresh cycle. There is no server-side collector, no database, and no intermediate JSON files.

---

## Architecture

### How it differs from Cosmos SDK explorers

| | Cosmos SDK Explorer | Monad Explorer |
|---|---|---|
| **Data collection** | Node.js collector on server | Browser-side fetch (client-only) |
| **Data storage** | JSON files on disk | React state in memory |
| **RPC protocol** | CometBFT + Cosmos REST | Ethereum JSON-RPC |
| **Refresh** | Collector writes every 6s; frontend reads | Frontend fetches every 6s directly |
| **Backend required** | Yes (nginx + collector) | No (PHP serves HTML only) |
| **Rate limit risk** | None (local RPC) | Yes (public endpoints — mitigated by RPC failover) |

### Request flow

```
User browser
    │
    ├── PHP (layout + page shell — served once)
    │
    └── JavaScript / React (runs in browser)
            │
            ├── eth_getBlockByNumber (batch) ──► Monad RPC
            ├── eth_feeHistory              ──► Monad RPC
            ├── eth_getBlockByNumber("pending") ► Monad RPC
            │
            └── /api/v1/public/validators/* ──► gmonads.com API
                (validators page, geo map only)
```

### RPC failover

Each page maintains an ordered list of public RPC endpoints. On first call, the system probes each in sequence and caches the first responsive one (`_rpc` variable). On HTTP 429 (rate limit), the cache is cleared and the next call re-probes.

**Testnet RPC list:**
```
https://testnet-rpc.monad.xyz
https://rpc.ankr.com/monad_testnet
https://monad-testnet.gateway.tenderly.co
https://monad-testnet-rpc.huginn.tech
```

**Mainnet RPC list:**
```
https://rpc.monad.xyz
https://rpc1.monad.xyz
https://rpc2.monad.xyz
https://rpc3.monad.xyz
https://rpc-mainnet.monadinfra.com
```

---

## Data Sources

### 1. Ethereum JSON-RPC (primary — all pages except validators/geo)

Standard EVM JSON-RPC methods, sent as batched HTTP POST requests to minimize round trips.

| Method | Used by | Data extracted |
|---|---|---|
| `eth_getBlockByNumber("latest", false)` | All pages | Latest block number, timestamp, miner, tx count, gas |
| `eth_getBlockByNumber("safe", false)` | Performance | Voted (QC) block — MonadBFT stage 2 |
| `eth_getBlockByNumber("finalized", false)` | Performance | Finalized (QC²) block — MonadBFT stage 3 |
| `eth_getBlockByNumber("pending", false)` | Performance | Pending tx count (mempool depth) |
| `eth_getBlockByNumber(hex, true)` | Performance, Blocks | Full block with transactions (type, input data) |
| `eth_feeHistory(100, "latest", [10,50,90])` | Performance | Gas used ratio, priority fee percentiles |
| `eth_blockNumber` | Performance | Current chain head |

**Batch requests:** Multiple calls are sent as a JSON array in a single HTTP request (`rpcBatch`), reducing latency significantly — e.g., fetching 20 block headers costs 1 HTTP round trip instead of 20.

### 2. gmonads.com API (secondary — validators and geo pages only)

```
https://www.gmonads.com/api/v1/public/
```

| Endpoint | `?network=` | Used by | Data |
|---|---|---|---|
| `/validators/geolocations` | `testnet` / `mainnet` | geo.php | Validator IP geolocation |
| `/validators/epoch` | `testnet` / `mainnet` | validators.php | Validator set, stake, status |
| `/blocks/1m` | `testnet` / `mainnet` | stats.php, blocks.php | Block stats |
| `/blocks/aggregated-hourly` | `testnet` / `mainnet` | stats.php | Hourly aggregated block data |

> **Note:** The `?network=` parameter is always specified explicitly. Without it, gmonads defaults to testnet — both instances must pass the parameter to avoid cross-contamination.

---

## Modules

### Validators (`validators.php`)
Validator set from `gmonads.com/api/v1/public/validators/epoch`. Displays active validators with stake, status, and identity. Data is fetched server-side (PHP `file_get_contents`) to avoid CORS restrictions.

### Blocks (`blocks.php`)
Live block stream using `eth_getBlockByNumber`. New blocks animate in at the top. Shows block number, timestamp, proposer (miner), transaction count, gas used, and fullness percentage.

### Uptime (`uptime.php`)
Validator signing history. Analyses which validators appear as block proposers over a rolling window, calculating an uptime proxy from block production rate.

### Consensus (`consensus.php`)
Live MonadBFT consensus state. Shows current round, vote stages (prevote/precommit), and participation rates. Updates every 2–6 seconds.

### Search (`search.php`)
Accepts block numbers and transaction hashes. Resolves via `eth_getBlockByNumber` and `eth_getTransactionByHash`. Linked from the block producers table in performance.php (`search?q=<blockNumber>`).

### Stats (`stats.php`)
Chain-level statistics combining gmonads aggregated data with direct RPC calls. Shows total transactions, validator count, block time averages, and network growth trends.

### Chain Performance (`performance.php`)
The most technically distinctive page — 100% direct RPC, zero third-party APIs. See the [Metrics Reference](#metrics-reference) section below for full detail.

### Node Map (`geo.php`)
World map of validator node locations from `gmonads.com/api/v1/public/validators/geolocations`. Fetched server-side to avoid CORS.

---

## Chain Performance — Metrics Reference

The Performance page (`performance.php`) has two tabs: **Live Data** and **Metrics Reference** (also documented here for the repository).

### Refresh cycles

| Cycle | Interval | Metrics |
|---|---|---|
| Fast | 6 seconds | Block pipeline, producers table, TPS, pending txs, finality time, all Row 1–3 KPIs |
| Slow | 30 seconds | Capacity chart, tx type breakdown, priority fee percentiles |

### Block Pipeline · MonadBFT

Monad uses **MonadBFT**, a BFT consensus protocol that decouples execution from consensus. A block progresses through three stages, each mapped to a standard Ethereum RPC tag:

| Stage | RPC tag | Description |
|---|---|---|
| **Proposed** | `"latest"` | Block produced by current slot validator. Has not yet received a QC. |
| **Voted** | `"safe"` | Received a **Quorum Certificate** (QC) — 2/3+ validators signed. ~1 block lag. |
| **Finalized** | `"finalized"` | Received **QC²** (two consecutive QCs). Irreversible. ~2 blocks / <1s from proposal. |

The lag badges show both the block distance and estimated time (`lag × avgBlockTime`). Raw timestamps are not used for time calculation — EVM block timestamps are integer seconds, too coarse for Monad's ~0.4s block cadence.

### KPI Cards

#### Row 1 — Core throughput

| Metric | Formula | Source | Notes |
|---|---|---|---|
| **Live TPS** | `Σ txCount / (latest.ts − oldest.ts)` | Block headers (last 20) | Wall-clock denominator, not block-time target |
| **Avg Block Time** | `Δ timestamp / Δ blockNumber` | Block headers (last 20) | ~0.40–0.42s on testnet |
| **Avg Block Fullness** | `Σ gasUsed / Σ gasLimit × 100` | Block headers (last 20) | Gas limit: 200M per block |
| **Base Fee** | Fixed constant | Protocol | Always 100 Gwei on Monad — does not adjust with demand |

#### Row 2 — Network health

| Metric | Formula | Source | Notes |
|---|---|---|---|
| **Unique Proposers** | `distinct(miner)` | Block headers (last 20) | 20/20 = perfect rotation |
| **Empty Blocks** | `count(txs=0) / total × 100` | Block headers (last 20) | Rare on Monad |
| **Contract Calls** | `count(input≠0x) / total × 100` | Full blocks (last 20) | ~100% on testnet — DeFi-native chain |
| **Priority Fee** | `median(reward[][1]) / 1e9` | `eth_feeHistory(100)` | Blocks with 0 reward filtered out; uniform or zero on Monad |

**Priority fee behaviour:** On Monad, the fixed base fee removes the incentive to outbid for block space. Tips are typically either zero (RPC doesn't return `reward[]`) or a uniform fixed value across all transactions.

#### Row 3 — Advanced metrics

| Metric | Formula | Source | Notes |
|---|---|---|---|
| **Finality Time** | `finalityBlocks × avgBlockTime` | Derived | ~840ms at 2 blocks × 0.42s |
| **Avg Txs / Block** | `Σ txCount / blockCount` | Block headers (last 20) | ~4 txs/block at current activity |
| **Pending Txs** | `eth_getBlockByNumber("pending").transactions.length` | RPC | Near-zero on Monad — blocks absorb txs faster than they arrive |
| **Avg Gas / Tx** | `mean(gasUsed / txCount)` over non-empty blocks | Block headers (last 20) | ~300k gas/tx reflects contract-heavy workload |

### Capacity & Traffic

| Chart | Source | Detail |
|---|---|---|
| **Block Capacity** | `eth_feeHistory(100, "latest", [])` → `gasUsedRatio[]` | Bar chart, 100 blocks, color-coded by utilisation |
| **Tx Type Breakdown** | `eth_getBlockByNumber(n, true)` → `tx.type` | Donut chart: Type 0 (Legacy), 1 (EIP-2930), 2 (EIP-1559), 4 (EIP-7702) |
| **Contract vs Transfer** | `tx.input !== "0x"` | Stacked bar — complement to tx type donut |

### Block Producers Table

Last 20 block headers fetched in a single batched RPC request. On first load, 20 blocks are fetched at once. On subsequent refreshes, only the gap since the last seen block is filled (incremental fill, up to 25 blocks per cycle). Each block number links to `search?q=<blockNumber>`.

---

## MonadBFT — Key Properties

- **~0.4s block time** — approximately 2.5 blocks per second
- **<1s to finality** — QC² achieved within 2 blocks of proposal
- **No reorgs** — finalized blocks are irreversible (BFT safety guarantee)
- **Fixed base fee** — 100 Gwei, does not adjust (unlike Ethereum EIP-1559)
- **Parallel EVM** — optimistic parallel execution; contract state access conflicts resolved at commit time
- **200M gas limit** — ~10× Ethereum's limit; currently <1% utilised
- **EIP-7702 active** — Type 4 transactions (account abstraction) on mainnet

---

## File Structure

```
services/
├── monad_testnet/          # Testnet explorer (Chain ID: 10143)
│   ├── layout-explorer.php # HTML shell, meta tags, sidebar
│   ├── menuexplorer.php    # Sidebar navigation
│   ├── validators.php
│   ├── blocks.php
│   ├── uptime.php
│   ├── consensus.php
│   ├── search.php
│   ├── stats.php
│   ├── performance.php     # Chain Performance page (RPC-only)
│   └── geo.php
│
└── monad/                  # Mainnet explorer (Chain ID: 143)
    └── (identical structure — generated via sed substitution)
```

### Mainnet generation

The mainnet files are derived from testnet by substituting chain-specific values:

```bash
sed \
  -e 's|10143|143|g' \
  -e 's|https://testnet-rpc.monad.xyz|https://rpc.monad.xyz|g' \
  -e 's|https://rpc.ankr.com/monad_testnet|https://rpc1.monad.xyz|g' \
  -e 's|https://monad-testnet.gateway.tenderly.co|https://rpc2.monad.xyz|g' \
  -e 's|https://monad-testnet-rpc.huginn.tech|https://rpc3.monad.xyz|g' \
  -e 's|explorer.monad.cumulo.com.es/performance|monad.cumulo.com.es/performance|g' \
  -e 's|Monad Testnet|Monad Mainnet|g' \
  monad_testnet/performance.php > monad/performance.php

# Add 5th mainnet RPC
sed -i 's|"https://rpc3.monad.xyz",|"https://rpc3.monad.xyz",\n  "https://rpc-mainnet.monadinfra.com",|' \
  monad/performance.php
```

---

## Design Decisions

**Why no collector for Monad?**
Monad's public RPC endpoints are fast and stable enough for direct browser fetching at 6s intervals. The data volume per cycle is small (20 block headers = 1 batch request), and the stateless nature of the metrics means no aggregation state needs to persist between cycles.

**Why gmonads for validators/geo and not direct RPC?**
The validator set on Monad is managed through on-chain staking contracts. Without the ABI and contract addresses, querying stake data directly via `eth_call` is not feasible from the frontend. gmonads provides a clean REST API over this data. All other pages bypass gmonads entirely.

**Why `display:none` instead of unmounting for the docs tab?**
The Live Data tab contains Chart.js instances and React state that accumulates over time (rolling block buffer). Unmounting and remounting would destroy this state and require a fresh RPC load. Using `display:none` keeps the live data running in the background while the user reads the Metrics Reference tab.
