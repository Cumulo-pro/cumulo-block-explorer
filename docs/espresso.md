# Espresso Explorer. Technical Documentation

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
| **API protocol** | CometBFT + Cosmos REST | Ethereum JSON-RPC | HotShot Query Service (REST) + Ethereum RPC |
| **Frontend refresh** | Polls collector JSON every 6s | Direct RPC every 6s | Polls collector JSON every 30s |
| **Backend required** | Yes | No | Yes |
| **Refresh interval** | 6s (collector) / 6s (frontend) | 6s (browser direct) | 30s (HotShot) / 6h (ETH contract) / 30s (frontend) |

### Request flow

```
Node.js collector - two independent loops
│
├── Every 30s - Espresso HotShot loop
│     ├── GET /v1/node/stake-table/current   ──► query.main.net.espresso.network
│     ├── GET /v1/status/block-height        ──► (same)
│     ├── GET /v1/hotshot-node/participation/vote/current  ──► (same)
│     ├── GET /v1/hotshot-node/block-reward  ──► (same)
│     ├── GET /v1/availability/block/{h}     ──► (same, last 50 blocks)
│     ├── GET /v1/availability/block/{h}/namespace/{ns}  ──► (same, per namespace)
│     └── GET {connect_info}/v1/node/identity  ──► individual validator nodes (optional)
│
├── Every 6h - Ethereum contract loop (at startup + interval)
│     └── POST JSON-RPC batch ──► eth-mainnet.g.alchemy.com  (1 HTTP request)
│           ├── eth_call validators(addr)          × N addresses
│           └── eth_call commissionTracking(addr)  × N addresses
│
└── atomic write ──► /var/lib/espresso-mainnet-collector/data.json
                            │
                    served as static file
                            │
User browser ──────────────►  https://data.espresso.cumulo.org.es/data.json
    │
    └── PHP frontend polls every 30s, renders in-page
```

### Atomic write

The collector writes to a `.tmp` file first, then renames it atomically to `data.json`, preventing
the frontend from reading a half-written file.

---

## Data Sources

### HotShot Query Service (`query.main.net.espresso.network`)

All Espresso chain data comes from a single public REST endpoint. No authentication is required.

| Endpoint | Data extracted |
|---|---|
| `GET /v1/node/stake-table/current` | Full active validator set with BLS keys, stake amounts (hex wei), connect_info |
| `GET /v1/status/block-height` | Latest block height (integer) |
| `GET /v1/hotshot-node/participation/vote/current` | Per-validator vote participation rate (0.0–1.0) |
| `GET /v1/hotshot-node/block-reward` | Current block reward (if available) |
| `GET /v1/availability/block/{height}` | Block header fields, hash, size, tx count, namespace table |
| `GET /v1/availability/block/{height}/namespace/{ns_id}` | Transactions in a specific namespace for a block |

**Timeouts:** Each request has a 12-second fetch timeout (`FETCH_TIMEOUT = 12000`). Requests run in
`Promise.allSettled()` - failed requests produce `null` and are skipped gracefully.

### Ethereum Staking Contract (every 6 hours)

Validator stake and commission are fetched live from the Espresso staking contract on Ethereum mainnet.
This eliminates all hardcoded stake/commission values and ensures data is always accurate.

| Contract | Address |
|---|---|
| Proxy (UUPS) | `0xcef474d372b5b09defe2af187bf17338dc704451` |
| Implementation | `0x36ad45A4931d0e226010be9a8477d400c8bb3d9c` (StakeTableV2) |

**Functions called:**

| Function | Selector | Returns |
|---|---|---|
| `validators(address)` | `0xfa52c7d8` | `(uint256 delegatedAmount, uint8 status)` |
| `commissionTracking(address)` | `0xac5c2ad0` | `(uint16 commissionBp, uint256 lastIncreaseTime)` |

- `status`: `0` = Unknown, `1` = Active (registered on L1), `2` = Exited
- `commissionBp`: commission in basis points - divide by 100 for percentage (e.g. 1000 → 10%)
- `delegatedAmount`: total delegated stake in wei - divide by 1e18 for ESP

**Transport:** A single JSON-RPC batch POST to Alchemy containing `2 × N` `eth_call` items
(one `validators()` + one `commissionTracking()` per known ETH address). This is **one HTTP request**
regardless of the number of validators. At ~67 addresses, the batch costs ~3,484 Alchemy CU
per 6-hour cycle (~70k CU/month total).

**RPC endpoint:** Alchemy Ethereum mainnet (`eth-mainnet.g.alchemy.com`)

### Validator identity probing (secondary)

For validators whose BLS key is **not** in `VALIDATOR_MAP`, the collector attempts to fetch identity
from the validator's own public node endpoint (the `connect_info` field from the stake table).
The probe uses a 3-second timeout:

```
GET {connect_info}/v1/node/identity
→ { node_name, company_name, company_website, country_code, latitude, longitude }
```

`company_name` is preferred as the validator name; `node_name` is used as fallback.

---

## Validator Map

The collector maintains `VALIDATOR_MAP` mapping full BLS verification keys (`BLS_VER_KEY~...`)
to known validator metadata:

```javascript
{
  "BLS_VER_KEY~<key>": {
    name:       "Blockdaemon",
    logo:       null,                      // filename in validatorsdata/vallogos/
    ethAddress: "0xfcA122749BD630d...",     // Ethereum staking contract address
    commission: 10.00,                     // FALLBACK only - live value fetched from contract
  },
  ...
}
```

**Commission field:** The `commission` value in `VALIDATOR_MAP` is used only as a fallback for
validators that do not have an ETH address. For all validators with a known ETH address, commission
is fetched live from the staking contract and takes precedence.

**Confirmed validators (top 14):** Identified by name on `stake.espresso.network`.  
**Ranks 15–43:** Cross-referenced by matching stake amounts to staking contract entries.

Validator logos are served from:
```
https://raw.githubusercontent.com/Cumulo-pro/validatorsdata/main/vallogos/<filename>
```

### Inactive validators

Validators registered on the Ethereum staking contract but below the active threshold (not present
in the HotShot stake table) are listed in `INACTIVE_VALIDATORS`. This array stores **only**
`ethAddress`, `name`, and `logo` - stake and commission are **fetched live from the contract**,
not hardcoded:

```javascript
const INACTIVE_VALIDATORS = [
  { ethAddress: "0xC86eb...", name: "Cypher Networks", logo: null       },
  { ethAddress: "0xc5052...", name: "Cumulo",           logo: "cumulo.jpg" },
  // ... (no stake, no commission - populated from Ethereum contract)
];
```

At output time, each inactive validator is enriched with live contract data:

```javascript
{
  ethAddress:  "0xC86eb...",
  name:        "Cypher Networks",
  logo:        null,
  stake:       5573.12,   // delegatedAmount from contract (ESP)
  commission:  10.00,     // commissionBp / 100 from contract
  status:      1,         // 0=Unknown, 1=Active/registered, 2=Exited
}
```

**Cross-reference:** At each collect cycle, the set of active ETH addresses (from the HotShot stake
table) is compared against `INACTIVE_VALIDATORS`. Any validator that appears in both lists
(graduated from inactive to active) is automatically removed from the inactive output. The collector
logs a warning when this happens:
```
⚑ N validator(s) moved from inactive → active (update INACTIVE_VALIDATORS)
```

**Sort order:** Inactive validators are sorted by stake descending. Validators with no contract
data yet (null stake) appear at the end.

---

## Collector - Data Pipeline

The collector runs **two independent asynchronous loops**:

### Loop A - Ethereum contract (every 6 hours)

Runs once at startup (before the first HotShot collect cycle), then every 6 hours.

1. Collects all unique ETH addresses from `VALIDATOR_MAP` values + `INACTIVE_VALIDATORS`
2. Builds a single JSON-RPC batch request (2 calls per address)
3. Decodes `delegatedAmount`, `status`, `commission` for each address
4. Stores results in module-level `_contractCache` (Map)

The 30-second HotShot loop reads `_contractCache` on every cycle to inject live
commission and stake values without making additional ETH calls.

### Loop B - HotShot collect (every 30 seconds)

#### Step 0 - Load previous rollupStats

Before fetching anything, the collector reads the existing `data.json` and restores the persistent
`rollupStats` accumulator. This allows rollup transaction history to span thousands of blocks
without losing data between collector restarts.

#### Step 1 - Parallel core fetch

Four API calls are fired simultaneously with `Promise.allSettled`:
- Stake table (active validator set)
- Block height
- Vote participation map
- Block reward

#### Step 2 - Process validators

For each entry in the HotShot stake table:
- Stake converted from hex wei to ESP: `Number(BigInt(hexAmount)) / 1e18`
- Metadata looked up in `VALIDATOR_MAP` (name, logo, ethAddress)
- **Commission:** read from `_contractCache` by ETH address → falls back to `VALIDATOR_MAP` value if no ETH address
- Vote participation rate cross-referenced: `missed = round((1 - voteRate) × 10000) / 100`
- Sorted by stake descending, assigned `rank` and `stakePct`

#### Step 3 - Identity probing

Unknown validators (name = null, connectInfo present) are probed in parallel for node identity.

#### Step 4 - Recent blocks

Last 50 blocks fetched in parallel. Namespace list extracted per block using three fallback
strategies (see Namespace System section).

#### Step 4b - Namespace transactions

For blocks with `txCount > 0` and a non-empty namespace list, each `(block, namespace)` pair
is fetched. Each transaction recorded as: `{ namespace, height, timestamp, builder, posInNs }`.

#### Step 4c - Persistent rollup accumulator

New transactions merged into `rollupStats`:
- Only transactions with `height > highWaterMark` are processed (prevents double-counting)
- Per-namespace counters: `txCount`, `lastHeight`, `lastTs`
- High-water mark advanced to current `blockHeight`
- Namespaces inactive for more than `ROLLUP_WINDOW = 2000` blocks are pruned (~17h)

#### Step 5 - Build output and atomic write

- Active ETH addresses extracted into a Set
- `INACTIVE_VALIDATORS` filtered (remove any now in active set), enriched from `_contractCache`, sorted by stake desc
- Final JSON written atomically via `.tmp` → rename

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
  "validators": [
    {
      "blsKey": "BLS_VER_KEY~...",
      "name": "Blockdaemon",
      "logo": null,
      "ethAddress": "0xfcA1...",
      "commission": 10.00,
      "stake": 12345.67,
      "stakePct": 13.02,
      "rank": 1,
      "missed": 0.12,
      "voteRate": 0.9988,
      "active": true,
      "epoch": 42
    }
  ],
  "inactiveValidators": [
    {
      "ethAddress": "0xC86eb...",
      "name": "Cypher Networks",
      "logo": null,
      "stake": 5573.12,
      "commission": 10.00,
      "status": 1
    }
  ],
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

Block payloads use a binary namespace table format. The collector tries three strategies:

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
stake (ESP + %), commission (live from contract), missed slots badge.
Active and inactive sets displayed separately. Inactive section includes a notice that
stake and commission values are fetched live from the Ethereum staking contract.

**Sort:** Active validators sorted by stake desc. Inactive validators sorted by stake desc
(nulls last, shown when contract cache is not yet populated).

**Search:** name, ETH address, BLS key (client-side filter, no re-fetch).

### Validator profile (`validator.php`)
Single validator detail page. Query param: `?address={ethAddress}`. Shows all available fields
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
Latest 50 blocks from `data.json/recentBlocks`. Shows height, timestamp, builder (`fee_info.account`),
transaction count, block size, L1 head anchor, block hash (truncated).

> **Note on builder:** The `builder` field is the sequencer address that constructed the block,
> not the consensus validator. In the current Espresso mainnet, one or very few sequencers are
> active so the same address repeats across most blocks - this is expected network behavior,
> not a display bug.

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
Espresso's block cadence is slower than Monad's ~0.4s. New blocks appear roughly every 30 seconds,
so a 30s collector interval is well-matched to actual chain activity.

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

**Why fetch validator data from the Ethereum contract?**
The HotShot stake table API only returns data for **active** validators (those above the activation
threshold). Inactive validators - registered on the Ethereum staking contract but not yet active -
have no presence in the HotShot API. Additionally, the stake table does not include commission rates.
Querying the Ethereum staking contract (`StakeTableV2`) directly provides:
- Commission rates for all validators (active and inactive)
- Stake amounts for inactive validators
- On-chain registration status (`Unknown / Active / Exited`)

This is done as a single JSON-RPC batch every 6 hours (validator registrations and commissions
change rarely), keeping Ethereum RPC usage at ~70k Alchemy CU/month.

**Why keep `VALIDATOR_MAP` and `INACTIVE_VALIDATORS` as static lists?**
The Ethereum staking contract has no enumeration function - it cannot return the full list of
registered validators. The only way to discover all validators is to parse historical
`RegisterValidator` events from the contract. The static lists serve as the known-address registry.
`name` and `logo` are off-chain data with no on-chain equivalent, so they always require manual
maintenance. Only `stake` and `commission` - the on-chain data - are now fetched dynamically.

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
# Requires Node.js 18+ (native fetch, no external dependencies)
node --version

# Run manually
node espresso-mainnet-collector.js

# Run as systemd service
sudo systemctl start espresso-mainnet-collector
sudo systemctl enable espresso-mainnet-collector

# After updating the collector, restart to pick up changes
sudo systemctl restart espresso-mainnet-collector

# Verify output
cat /var/lib/espresso-mainnet-collector/data.json | python3 -m json.tool | head -40
```

The collector logs each startup and cycle to stdout:
```
[ETH] ✓ 67 addresses · 43 registered on L1 · 1 batch call
[2025-01-01T00:00:00.000Z] Collecting Espresso mainnet data...
  ✓ 43 active (41 identified) | 24 inactive | epoch 42 | block #123,456 | 50 blocks | 87 txs | 3 rollups (hwm #123,456) | 2341ms
```

If a validator graduates from inactive to active between ETH fetches:
```
  ⚑ 1 validator(s) moved from inactive → active (update INACTIVE_VALIDATORS)
```

---

## External Links

| Resource | URL |
|---|---|
| Official Espresso Explorer | `https://explorer.espresso.network` |
| HotShot Query API | `https://query.main.net.espresso.network` |
| Espresso Staking Portal | `https://stake.espresso.network` |
| Espresso Network | `https://espresso.network` |
| Staking contract (Etherscan) | `https://etherscan.io/address/0xcef474d372b5b09defe2af187bf17338dc704451` |
| Cumulo validator data (logos) | `https://github.com/Cumulo-pro/validatorsdata/tree/main/vallogos` |
| Collector data endpoint | `https://data.espresso.cumulo.org.es/data.json` |
