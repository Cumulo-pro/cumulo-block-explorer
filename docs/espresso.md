# Espresso Explorer - Technical Documentation

> This document covers the Espresso-specific implementation of the Cumulo Block Explorer.
> Espresso uses a **fundamentally different architecture** from both the Cosmos SDK chains documented
> in `architecture.md` and the Monad Explorer documented in `monad.md` - it uses a server-side
> Node.js collector writing a shared JSON file, not browser-side RPC fetching, and not CometBFT/Cosmos REST.

---

## Overview

| Instance | URL | Chain |
|---|---|---|
| Espresso Mainnet | `espresso.explorer.cumulo.org.es` | espresso-mainnet-1 |

All data is collected **server-side** by a Node.js daemon that polls the Espresso HotShot Query API
every 30 seconds and writes an atomic JSON snapshot. The PHP/HTML frontend fetches that snapshot
from a CDN-like data endpoint and re-renders the page every 30 seconds - **no direct API calls
from the browser**.

---

## Architecture

### How it differs from other Cumulo explorers

| | Cosmos SDK Explorer | Monad Explorer | Espresso Explorer |
|---|---|---|---|
| **Data collection** | Node.js collector on server | Browser-side fetch (client-only) | Node.js collector on server |
| **Data storage** | JSON files on disk | React state in memory | Single JSON file on disk |
| **API protocol** | CometBFT + Cosmos REST | Ethereum JSON-RPC | HotShot Query Service (REST) |
| **Frontend refresh** | Polls collector JSON every 6s | Direct RPC every 6s | Polls collector JSON every 30s |
| **Backend required** | Yes | No | Yes |
| **Refresh interval** | 6s (collector) / 6s (frontend) | 6s (browser direct) | 30s (collector) / 30s (frontend) |

### Request flow

```
Node.js collector (server - every 30s)
    │
    ├── GET /v1/node/stake-table/current   ──► query.main.net.espresso.network
    ├── GET /v1/status/block-height        ──► query.main.net.espresso.network
    ├── GET /v1/hotshot-node/participation/vote/current  ──► (same)
    ├── GET /v1/hotshot-node/block-reward  ──► (same)
    ├── GET /v1/availability/block/{h}     ──► (same, last 50 blocks)
    ├── GET /v1/availability/block/{h}/namespace/{ns}  ──► (same, per namespace)
    └── GET {connect_info}/v1/node/identity  ──► individual validator nodes (optional)
    │
    └── atomic write ──► /var/lib/espresso-mainnet-collector/data.json
                                │
                        served as static file
                                │
User browser ──────────────────►  https://data.espresso.cumulo.org.es/data.json
    │
    └── PHP frontend polls every 30s, renders in-page
```

### Atomic write

The collector writes to a `.tmp` file first, then renames it atomically to `data.json`, preventing
the frontend from reading a half-written file.

---

## Data Sources

### HotShot Query Service (`query.main.net.espresso.network`)

All data comes from a single public REST endpoint. No authentication is required.

| Endpoint | Data extracted |
|---|---|
| `GET /v1/node/stake-table/current` | Full validator set with BLS keys, stake amounts (hex wei), connect_info |
| `GET /v1/status/block-height` | Latest block height (integer) |
| `GET /v1/hotshot-node/participation/vote/current` | Per-validator vote participation rate (0.0–1.0) |
| `GET /v1/hotshot-node/block-reward` | Current block reward (if available) |
| `GET /v1/availability/block/{height}` | Block header fields, hash, size, tx count, namespace table |
| `GET /v1/availability/block/{height}/namespace/{ns_id}` | Transactions in a specific namespace for a block |

**Timeouts:** Each request has a 12-second fetch timeout (`FETCH_TIMEOUT = 12000`). Requests run in
`Promise.allSettled()` - failed requests produce `null` and are skipped gracefully.

### Validator identity probing (secondary)

For validators whose BLS key is **not** in the hardcoded `VALIDATOR_MAP`, the collector attempts to
fetch identity from the validator's own public node endpoint (the `connect_info` field from the
stake table). The probe uses a 3-second timeout and reads:

```
GET {connect_info}/v1/node/identity
→ { node_name, company_name, company_website, country_code, latitude, longitude }
```

If successful, `company_name` is used as the validator name; otherwise `node_name`. This allows
previously unknown validators to appear with a real name without requiring a manual map update.

---

## Validator Map

The collector maintains a hardcoded `VALIDATOR_MAP` mapping full BLS verification keys
(`BLS_VER_KEY~...`) to known validator metadata:

```javascript
{
  "BLS_VER_KEY~<key>": {
    name:       "Blockdaemon",
    logo:       null,                        // filename in validatorsdata/vallogos/
    ethAddress: "0xfcA122749BD630d...",       // Ethereum staking contract address
    commission: 10.00,                       // percentage
  },
  ...
}
```

**Confirmed validators (top 14):** Identified by name on `stake.espresso.network`.  
**Ranks 15–43:** Cross-referenced by matching stake amounts to staking contract entries.

Validator logos are fetched from:
```
https://raw.githubusercontent.com/Cumulo-pro/validatorsdata/main/vallogos/<filename>
```

### Inactive validators

Validators registered on the Ethereum staking contract but below the active threshold are maintained
as a static `INACTIVE_VALIDATORS` array with: `ethAddress`, `name`, `logo`, `stake`, `commission`.
These appear on the Validators page with an "Inactive" badge and are excluded from all stake
calculations.

---

## Collector - Data Pipeline

The collector (`espresso-mainnet-collector.js`) runs on a 30-second interval and executes
these steps on each cycle:

### Step 0 - Load previous rollupStats

Before fetching anything, the collector reads the existing `data.json` and restores the persistent
`rollupStats` accumulator. This allows rollup transaction history to span thousands of blocks
without losing data between collector restarts.

### Step 1 - Parallel core fetch

Four API calls are fired simultaneously with `Promise.allSettled`:
- Stake table (validator set)
- Block height
- Vote participation map
- Block reward

### Step 2 - Process validators

For each entry in the stake table:
- Stake amount is converted from hex wei to ESP: `Number(BigInt(hexAmount)) / 1e18`
- Metadata is looked up in `VALIDATOR_MAP` (or left null for unknown validators)
- Vote participation rate is cross-referenced to compute `missed` slots percentage:
  `missed = round((1 - voteRate) * 10000) / 100`
- Validators are sorted by stake descending and assigned `rank` and `stakePct`

### Step 3 - Identity probing

Unknown validators (name = null, connectInfo present) are probed in parallel for node identity.
Results update `name`, `website`, `location`, and `country` fields in-place.

### Step 4 - Recent blocks

The last 50 blocks are fetched in parallel (`/v1/availability/block/{h}`). Each block provides:
- `height`, `timestamp`, `hash`, `size`, `txCount`, `l1Head`, `builder`, `payloadCommitment`
- Namespace list (`_nsList`): extracted from `payload.ns_table` using three fallback strategies (see below)

### Step 4b - Namespace transactions

For blocks that have both `txCount > 0` and a non-empty `_nsList`, the collector fetches each
`(block, namespace)` pair: `/v1/availability/block/{h}/namespace/{ns_id}`.

Each transaction is recorded as:
```javascript
{ namespace, height, timestamp, builder, posInNs }
```

Transaction hashes (`TX~...`) are **not stored** - the official hash scheme could not be reliably
reproduced client-side, so transactions are identified by `height·posInNs` instead.

### Step 4c - Persistent rollup accumulator

New transactions are merged into `rollupStats`:
- Only transactions with `height > highWaterMark` are processed (prevents double-counting)
- Per-namespace counters track: `txCount`, `lastHeight`, `lastTs`
- High-water mark is advanced to the current `blockHeight`
- Namespaces not seen within the last `ROLLUP_WINDOW = 2000` blocks are pruned

**Window size:** ~2000 blocks ≈ 16–17 hours at current Espresso block cadence.

### Step 5 - Atomic write

The final JSON is written to a `.tmp` file and renamed atomically. `_nsList` is stripped from blocks
before writing.

### Output schema (`data.json`)

```json
{
  "meta": {
    "chainId": "espresso-mainnet-1",
    "latestHeight": 123456,
    "epoch": 42,
    "totalStake": 95000.0,
    "validatorCount": 43,
    "inactiveCount": 24,
    "nakamoto": 5,
    "blockReward": null,
    "updatedAt": "2025-01-01T00:00:00.000Z"
  },
  "validators": [ ... ],
  "inactiveValidators": [ ... ],
  "recentBlocks": [ ... ],
  "recentTransactions": [ ... ],
  "rollupStats": {
    "highWaterMark": 123456,
    "namespaces": {
      "33139": { "txCount": 412, "lastHeight": 123450, "lastTs": "2025-01-01T..." },
      "8453":  { "txCount": 87,  "lastHeight": 123448, "lastTs": "2025-01-01T..." }
    }
  }
}
```

---

## Namespace System

Espresso uses **namespaces** (integer IDs) to separate transactions from different rollups.
Namespace IDs correspond to EVM chain IDs:

| Namespace ID | Rollup |
|---|---|
| 33139 | ApeChain |
| 8453 | Base |
| 42161 | Arbitrum One |
| 10 | OP Mainnet |
| 7777777 | Zora |
| 34443 | Mode |
| 1135 | Lisk |
| 5000 | Mantle |
| 59144 | Linea |
| 167000 | Taiko |
| 324 | zkSync Era |
| 1101 | Polygon zkEVM |
| 534352 | Scroll |
| 81457 | Blast |
| 204 | opBNB |
| 11155111 | Sepolia (testnet) |
| 84532 | Base Sepolia (testnet) |

Unknown namespace IDs are displayed as `NS {id}` with a deterministic warm-palette color
assigned by `id % 5`.

### Namespace extraction

Block payloads use a binary namespace table format. The collector tries three strategies to parse it:

```
Strategy A - ns_table is an array of objects:
  payload.ns_table[i].namespace_id  (or .ns_id / .namespace / .id)

Strategy B - ns_table has a base64-encoded binary 'bytes' field:
  buf[0..3] = numNss (LE uint32)
  Entries at stride 8 (ns_id u32 + end_offset u32)
  or stride 16 (ns_id u64 + end_offset u64)

Strategy C - payload object has numeric keys:
  Object.keys(payload).filter(k => /^\d+/.test(k))
```

---

## Modules

### Stats (`stats.php`)
Stake distribution, Nakamoto coefficient, top 10 validators, commission breakdown. All data from
`data.json/meta` and `data.json/validators`. Refreshes every 30s.

**Key metrics displayed:**
- Epoch, latest block height
- Total ESP staked, active/inactive validator counts
- Nakamoto coefficient (validators needed to control >33% of stake)
- Stake distribution bar (one segment per validator, colored amber → orange palette)
- Top 10 by stake with mini bar charts
- Nakamoto set with cumulative stake display
- Commission breakdown: ≤1% / 1–5% / 5–10% / 10%+

### Validators (`validators.php`)
Full validator table with: rank, logo/avatar, name, ETH address, BLS key (truncated),
stake (ESP + %), commission, missed slots badge. Clickable rows link to `validator?key={blsKey}`.
Active and inactive sets displayed separately.

**Search:** name, ETH address, BLS key (client-side filter, no re-fetch).

### Validator profile (`validator.php`)
Single validator detail page. Query param: `?key={blsKey}`. Shows all available fields
including full BLS key, staking contract link, and participation history.

### Rollups (`rollups.php`)
Live rollup activity aggregated from the persistent `rollupStats` accumulator. Shows all
namespaces active in the last ~2000 blocks (~17h).

**Stats strip:** Active rollup count, total transactions in window, most active rollup name,
block window size.

**Card grid:** One card per active namespace, showing:
- Initials avatar (first letters of each word, or first 2 chars for single-word names)
- Transaction count with relative activity bar
- Last block height and relative timestamp (`Xs ago`)
- Warm-palette color coding per rollup

**Sort modes:** By activity (tx count desc), by name (alphabetical), by namespace ID.

**Search:** Client-side filter by rollup name or namespace ID.

### Blocks (`blocks.php`)
Latest 50 blocks from `data.json/recentBlocks`. Shows height, timestamp, builder (fee_info.account),
transaction count, block size, L1 head anchor, payload commitment (truncated).

Clicking a block navigates to `block?height={h}`.

### Block detail (`block.php`)
Query param: `?height={h}`. Fetches the block directly from the HotShot Query API
(`/v1/availability/block/{h}`) client-side for full detail. Displays all header fields and
lists transactions grouped by namespace.

### Transactions (`transactions.php`)
Recent transaction feed from `data.json/recentTransactions`. Displays up to the last 200
transactions with rollup badge, block height, and position-in-namespace.

**Transaction identifier format:** `#height · Tx posInNs` (e.g. `#123,456 · Tx 3`).  
Transaction hashes are not shown - users are directed to the official Espresso explorer for hashes.

**Namespace filter tab:** Shows all known namespaces from recent transactions; clicking a tab
filters to that rollup only.

**Search:** name of rollup, namespace ID, or block height.

### Transaction detail (`transaction.php`)
Query params: `?height={h}&ns={namespace}&index={posInNs}`. Fetches the namespace transaction
list for the block directly from the API client-side, extracts the transaction at the given index,
and displays: namespace/rollup, position, raw payload (hex dump), payload size, builder.

A prominent link bar directs the user to the official Espresso explorer for the canonical TX hash.

### Search (`search.php`)
Accepts: block height (numeric), validator name, ETH address, BLS key fragment.
Resolves blocks via `data.json/recentBlocks` or direct API call; validators via in-memory search
of `data.json/validators`.

---

## HotShot Consensus - Key Properties

Espresso uses **HotShot**, a BFT consensus protocol developed by Espresso Systems:

- **Validator set:** BLS key-based, managed via Ethereum staking contract on L1
- **Epochs:** Validators are grouped into epochs; the stake table is epoch-scoped
- **Vote participation:** Tracked per-validator as a float (0.0–1.0); `missed = (1 - voteRate) × 100`
- **Block builder:** The `fee_info.account` field identifies the block builder (sequencer)
- **L1 anchor:** Each block records the latest confirmed Ethereum L1 block height (`l1_head`)
- **Payload commitment:** A cryptographic commitment to the block payload (`payload_commitment`)
- **Namespaces:** Rollup transactions are segregated by namespace ID (EVM chain ID convention)

### Nakamoto coefficient

Computed by sorting validators by stake descending and counting how many are needed to
cumulatively reach 33% of total stake. A higher coefficient = more decentralised.

```javascript
function nakamotoCoeff(validators) {
  let cum = 0, n = 0;
  for (const v of validators) {
    cum += v.stakePct;
    n++;
    if (cum >= 33) break;
  }
  return n;
}
```

---

## Design Decisions

**Why a server-side collector (not browser-direct like Monad)?**
The Espresso API involves multiple chained requests per cycle (stake table → blocks → per-namespace
transactions). Doing this from the browser on every page load would impose high latency and risk
rate limiting. The collector aggregates everything server-side and exposes a single pre-computed
JSON file that any number of browser clients can consume instantly.

**Why 30s refresh instead of 6s?**
Espresso's block cadence is slower than Monad's ~0.4s. At the time of writing, new blocks appear
every ~30 seconds, so a 30s collector interval is well-matched to actual chain activity.

**Why is the rollup accumulator persistent?**
Each collector cycle only fetches the last 50 blocks. Without persistence, rollup statistics would
reset every 30 seconds. The `highWaterMark` pattern ensures each block is counted exactly once
across all cycles, allowing cumulative statistics to span up to 2000 blocks (~17h) before pruning.

**Why no TX hash display?**
Espresso's canonical transaction hash scheme (`TX~...`) uses `blake3` over specific serialisations
of namespace ID, payload length, and payload bytes. The exact byte layout could not be reproduced
reliably client-side to match the official explorer's hashes. Rather than display incorrect hashes,
the UI uses `height·posInNs` as an unambiguous transaction identifier and provides a direct link
to the official explorer for canonical hash lookups.

**Why static VALIDATOR_MAP instead of on-chain lookup?**
The Ethereum staking contract holds validator metadata, but calling it from the backend requires
an Ethereum RPC connection, ABI parsing, and an ethers.js/viem dependency. The static map covers
all currently active validators and is updated manually as new validators join. The identity-probing
fallback (`connect_info` → `/v1/node/identity`) provides a zero-maintenance path for new validators
that expose their own node endpoint.

---

## File Structure

```
services/
└── espresso/
    ├── layout-explorer.php       # HTML shell, meta tags, sidebar, brand tokens
    ├── menuexplorer.php          # Sidebar navigation (Stats → Validators → Rollups → Blocks → Transactions → Search)
    ├── components/
    │   ├── nav.php               # Top header navigation
    │   └── menu-top-expl.php     # Explorer sub-navigation (mobile/tablet)
    ├── stats.php                 # Network statistics page
    ├── validators.php            # Validator set table
    ├── validator.php             # Individual validator profile
    ├── rollups.php               # Rollup activity dashboard
    ├── blocks.php                # Recent blocks feed
    ├── block.php                 # Block detail page
    ├── transactions.php          # Recent transactions feed
    ├── transaction.php           # Transaction detail page
    └── search.php                # Multi-entity search

/var/lib/espresso-mainnet-collector/
└── data.json                     # Output file (atomic write, served as static)

/path/to/collector/
└── espresso-mainnet-collector.js # Node.js daemon (systemd service)
```

### Collector deployment

```bash
# Install
node --version  # requires Node.js 18+ (native fetch)

# Run manually
node espresso-mainnet-collector.js

# Run as systemd service
sudo systemctl start espresso-mainnet-collector
sudo systemctl enable espresso-mainnet-collector

# Verify output
cat /var/lib/espresso-mainnet-collector/data.json | python3 -m json.tool | head -30

# After updating the collector, restart to pick up changes
sudo systemctl restart espresso-mainnet-collector
```

The collector logs each cycle to stdout:
```
[2025-01-01T00:00:00.000Z] Collecting Espresso mainnet data...
  ✓ 43 validators (41 identified) | epoch 42 | block #123,456 | 50 blocks | 87 txs | 3 rollups (hwm #123,456) | 2341ms
```

---

## External Links

| Resource | URL |
|---|---|
| Official Espresso Explorer | `https://explorer.espresso.network` |
| HotShot Query API | `https://query.main.net.espresso.network` |
| Espresso Staking Portal | `https://stake.espresso.network` |
| Espresso Network | `https://espresso.network` |
| Cumulo validator data (logos) | `https://github.com/Cumulo-pro/validatorsdata/tree/main/vallogos` |
| Collector data endpoint | `https://data.espresso.cumulo.org.es/data.json` |
