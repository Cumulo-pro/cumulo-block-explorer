# Cumulo Block Explorer

> **Infrastructure-grade, self-hosted blockchain explorer suite** built by [Cumulo](https://cumulo.com.es) — a professional Proof-of-Stake validator operating across 15+ networks.

---

## Table of Contents

- [Why Cumulo Built Its Own Explorer](#why-cumulo-built-its-own-explorer)
- [Explorer Overview](#explorer-overview)
- [Supported Networks](#supported-networks)
- [Architecture](#architecture)
- [Modules](#modules)
- [Data Sources](#data-sources)
- [Metrics & Standards](#metrics--standards)
- [Documentation Index](#documentation-index)

---

## Why Cumulo Built Its Own Explorer

As a professional validator studio, Cumulo requires deep, real-time visibility into the networks it secures — visibility that public explorers simply do not provide.

### The Problem with Generic Explorers

Public block explorers (Mintscan, Ping.pub, Big Dipper, etc.) are built for general audiences. They excel at user-facing functions like browsing transactions and checking balances, but fall short for the operational needs of a validator:

- **Data freshness** — public explorers introduce indirection layers that delay metric updates by minutes
- **Validator-centric metrics** — uptime windows, signing histories, and jailing events are not surfaced in the granularity operators need
- **Governance monitoring** — tracking live tally data and vote participation across multiple networks simultaneously is not feasible on third-party tools
- **Decentralization analysis** — no public explorer computes regional, country, hosting-provider, or voting-power concentration indices in real time
- **Multi-network parity** — each network-specific explorer has a different UI and API surface; building our own ensures a uniform experience across all chains
- **Data ownership** — relying on third-party infrastructure for operational metrics is a single point of failure; self-hosted means we control uptime, latency, and access

### Why In-House

Cumulo runs its own validator nodes. The collectors that power this explorer run on the **same infrastructure** as the nodes themselves, with direct RPC access and zero intermediaries. This means:

- Sub-second data freshness (collection intervals of 3–12 seconds depending on block time)
- Complete control over what metrics are collected and how they are computed
- A platform that evolves with our operational requirements, not a third-party roadmap
- A showcase of technical depth for the delegators and chains that trust Cumulo

---

## Explorer Overview

The Cumulo Block Explorer is a **multi-chain, self-hosted monitoring and analytics platform** for Proof-of-Stake blockchain networks. It provides real-time and historical data across seven core modules:

| Module | Description |
|---|---|
| [Validators](docs/modules.md#validators) | Voting power, commission, uptime, jailing status |
| [Blocks](docs/modules.md#blocks) | Block stream, proposer, transactions, gas |
| [Uptime](docs/modules.md#uptime) | Per-validator signing history (block-level granularity) |
| [Governance](docs/modules.md#governance) | Proposals, live tally, voter list, deposit tracking |
| [Stats](docs/modules.md#stats) | Staking pool, inflation, supply, block time, TPS |
| [Consensus](docs/modules.md#consensus) | Live round state, prevote/precommit participation |
| [Decentralization](docs/modules.md#decentralization) | Geographic, hosting, and voting-power concentration indices |

Each module is served from **static JSON files** written by a background collector process, eliminating database overhead and enabling horizontal scaling through a plain nginx setup.

---

## Supported Networks

### Cosmos SDK (Mainnet)

| Network | Chain ID | Block Time | Status |
|---|---|---|---|
| Cosmos Hub | cosmoshub-4 | ~6s | Active |
| Celestia | celestia | ~6s | Active |
| XRPL EVM | xrplevm_1440000-1 | ~6s | Active |
| Story | story | ~3s | Active |


### Cosmos SDK (Testnet)

| Network | Chain ID | Status |
|---|---|---|
| Cosmos Testnet | provider | Active |
| Celestia Mocha | mocha-4 | Active |
| XRPL EVM Testnet | xrplevm_1449000-1 | Active |
| Story Aeneid | aeneid | Active |
| SEDA Testnet | seda-1-testnet | Active |

### Non-Cosmos Networks

| Network | Architecture | Status |
|---|---|---|
| Avail | Substrate (SCALE) | Building |
| Starknet | StarkWare / STARK proofs | Building |


> **Future networks** — the platform is designed to be extended to any PoS architecture. See [Architecture](docs/architecture.md) for details on adding new chains.

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        Blockchain Node                      │
│  CometBFT RPC (:26657)   Cosmos REST API (:1317)           │
└───────────────┬────────────────────────┬───────────────────┘
                │                        │
                ▼                        ▼
┌──────────────────────────────────────────────────────────┐
│                   Collector (Node.js)                     │
│  - Runs every 3–12 seconds (configurable per chain)       │
│  - Fetches, normalizes, enriches data                     │
│  - Writes atomic JSON snapshots to output directory       │
│  - Runs as systemd service on validator server            │
└─────────────────────────┬────────────────────────────────┘
                          │  writes JSON
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Static File Server (nginx)                  │
│  /data/data.json          – validators, blocks, stats    │
│  /data/governance.json    – proposals, tally, votes      │
│  /data/consensus.json     – live round state             │
└─────────────────────────┬───────────────────────────────┘
                          │  HTTP fetch (no CORS)
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   Web Frontend (PHP + React)             │
│  - Polls JSON endpoints every 6 seconds                  │
│  - Renders modules: Validators, Blocks, Uptime, etc.     │
│  - No server-side database; pure static data model       │
└─────────────────────────────────────────────────────────┘
```

Full architecture documentation → [docs/architecture.md](docs/architecture.md)

---

## Modules

Brief descriptions of each module; full details in [docs/modules.md](docs/modules.md).

**Validators** — Real-time table of all active, inactive, and jailed validators with voting power, commission rate, uptime score (last 150 blocks), token delegation, and Keybase-resolved identity avatars.

**Blocks** — Live stream of produced blocks including height, timestamp, proposer, transaction count, and gas consumed. Clickable cards reveal per-block detail.

**Uptime** — Block-level signing history for every validator rendered as a color-coded strip. Green = signed, red = missed. Sortable by uptime percentage or missed-block count.

**Governance** — Full proposal lifecycle tracking: deposit period, voting period, and final tally. Live YES/NO/ABSTAIN/NO\_WITH\_VETO percentages with voter address list.

**Stats** — Key network health indicators: staking ratio, inflation rate, community pool, circulating supply, average block time, and transaction throughput.

**Consensus** — Live CometBFT round state showing current height, round, step, and per-validator prevote/precommit participation.

**Decentralization** — Geographic map and numerical indices (regional, country, hosting, voting-power HHI) that measure how decentralized the validator set is.

---

## Data Sources

| Source | Protocol | Used For |
|---|---|---|
| CometBFT RPC | HTTP / JSON-RPC | Blocks, validators, consensus, net\_info |
| Cosmos REST API | HTTP / REST | Staking, governance, supply, inflation |
| Keybase API | HTTP / REST | Validator identity avatars |
| ip-api.com | HTTP / REST | Peer IP geolocation (decentralization) |
| Substrate JSON-RPC | HTTP + WebSocket | Avail block data (SCALE encoded) |

Full endpoint reference → [docs/data-sources.md](docs/data-sources.md)

---

## Metrics & Standards

| Metric | Standard | Notes |
|---|---|---|
| Uptime | Signed blocks / total blocks × 100 | 150-block rolling window |
| Voting Power % | Validator tokens / bonded pool × 100 | Live from staking pool |
| Staking Ratio | Bonded tokens / total supply × 100 | From mint + bank modules |
| HHI | Σ(market\_share²) | Voting power concentration |
| Decentralization Index | (1 − HHI) × 100 | Regional, country, hosting, power |
| Inflation | Direct from mint module | Chain-native value |
| TPS | Txs in last block / avg block time | Rolling approximation |

Full metric definitions and calculation methodology → [docs/metrics.md](docs/metrics.md)

---

## Documentation Index

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Collector pattern, deployment topology, nginx/systemd setup, adding new chains |
| [docs/modules.md](docs/modules.md) | Detailed description of each explorer module and its features |
| [docs/data-sources.md](docs/data-sources.md) | All API endpoints, request/response formats, normalization per network type |
| [docs/metrics.md](docs/metrics.md) | Metric definitions, calculation formulas, data windows, standards used |
| [docs/networks.md](docs/networks.md) | Per-network configuration: endpoints, block time, chain-specific adaptations |

---

## License

© Cumulo. All rights reserved.
