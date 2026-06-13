> **avail.md** — Technical documentation for the Cumulo Block Explorer: Avail Mainnet instance.
> Covers the Node.js collector pipeline, Substrate JSON-RPC data sources, SCALE extrinsic decoding
> for DA detection, output schemas (data.json + stats.json), frontend module inventory, and deployment.

---

## Overview

| Property | Value |
|---|---|
| Instance | Avail Mainnet |
| URL | `https://avail-mainnet.cumulo.com.es` |
| Data endpoint (live) | `https://avail.explorer.cumulo.org.es/data.json` |
| Data endpoint (historical) | `https://avail.explorer.cumulo.org.es/stats.json` |
| Explorer proxy | `https://avail.explorer.cumulo.org.es/rpc/` |
| Public RPC (HTTP) | `https://avail.rpc.cumulo.me` |
| Public RPC (WebSocket) | `wss://avail.rpc.cumulo.me` |
| Collector version | `avail-mainnet-collector.js v3.0` |
| Collector interval | 12,000 ms |
| Stats flush interval | 1 hour (`STATS_INTERVAL = 60 × 60,000 ms`) |

---

## Architecture

### Comparison with Cosmos SDK Explorer

| Dimension | Cosmos SDK Explorer | Avail Explorer |
|---|---|---|
| Data collection | Node.js collector on server | Node.js collector on server |
| Data storage | JSON files on disk | Two JSON files on disk (`data.json` + `stats.json`) |
| RPC protocol | CometBFT + Cosmos REST | Substrate JSON-RPC |
| Frontend refresh | Polls collector JSON every 6s | Polls `data.json` every 30s, `stats.json` on demand |
| Backend required | Yes | Yes |
| Refresh interval | 6s (collector) / 6s (frontend) | 12s (collector) / 30s (frontend) |

### Request Flow

```
Node.js collector (server - every 12s)
    │
    ├── chain_getHeader / chain_getFinalizedHead  ──► avail.rpc.cumulo.me
    ├── state_getRuntimeVersion                  ──► (same)
    ├── system_properties / system_version       ──► (same)
    ├── chain_getBlockHash (last 20 blocks)      ──► (same, batched)
    ├── chain_getBlock (per hash)                ──► (same)
    ├── payment_queryInfo (per signed extrinsic) ──► (same, batched)
    ├── state_getStorage (staking, session)      ──► (same)
    └── SCALE decode extrinsics → DA detection
    │
    ├── atomic write ──► /var/lib/avail-collector/data.json  (live, every 12s)
    └── hourly flush ──► /var/lib/avail-collector/stats.json (once per hour)
                                │
                        served as static files
                                │
User browser ──────────────────► https://avail.explorer.cumulo.org.es/data.json
    │                            https://avail.explorer.cumulo.org.es/stats.json
    └── RPC calls (blocks.php)
            │
            └── rpc.php (PHP proxy) ──► avail.rpc.cumulo.me
```

The PHP/React frontend (Babel standalone + React 18 UMD) polls `data.json` every 30 seconds. `rpc.php` is a transparent PHP proxy that forwards browser RPC calls to the node, bypassing CORS restrictions.

---

## Data Sources

### Substrate RPC Methods

| Method | Data extracted |
|---|---|
| `chain_getHeader` | Latest block number, parent hash |
| `chain_getFinalizedHead` | Finalized block hash |
| `chain_getBlockHash([height])` | Block hash for a given height |
| `chain_getBlock([hash])` | Full block with extrinsics array |
| `state_getRuntimeVersion` | `specVersion`, `specName` |
| `system_properties` | `tokenSymbol`, `tokenDecimals`, `ss58Format` |
| `system_version` | Node software version |
| `payment_queryInfo(ext, blockHash)` | `partialFee` per signed extrinsic |
| `state_getStorage(key)` | `Session::Validators`, `staking::ErasStakers`, `staking::ErasValidatorPrefs`, `staking::CurrentEra` |

RPC transport: HTTP POST, batch supported (`rpcBatch`). All calls target `http://avail.rpc.cumulo.me`.

---

## Collector — Data Pipeline

### Step 0 — Load Identities

Reads `/var/lib/avail-collector/identidades.json` at startup and every hour. Supports both array and object formats. Builds a `Map` from SS58 address → identity fields (`moniker`, `legal`, `web`, `email`, `twitter`, `riot`).

### Step 1 — Chain Metadata

Fetches in parallel: `state_getRuntimeVersion`, `system_properties`, `system_version`, latest block header, finalized block hash and header. Computes:

```
lag = latestHeight − finalizedHeight
```

### Step 2 — Analyze Last 20 Blocks

Fetches the last 20 block hashes in parallel, then analyzes each block:

- SCALE-decodes every extrinsic to detect signed transactions.
- For each signed extrinsic: reads pallet index and call index.
- If `pallet = 0x1d` (DataAvailability) and `call = 0x01` (submitData):
  - Increment `daCount`
  - Read `AppId` as `compact<u32>` (Avail-specific, NOT fixed 4-byte u32)
  - Read data length as `compact<u32>` → `daBytes`
  - Sanity cap: if `daBytes > 8,388,608` (8 MB) → set to `0`
- Accumulates `txCount`, `daCount`, `daBytes`, `appIds`, `totalFeeAVAIL`.
- Fees via `payment_queryInfo` per signed extrinsic (batched in groups of 5).

### Step 3 — Hourly Accumulation

Tracks `_lastAccumulatedHeight` (global, persists across cycles). Each cycle accumulates all newly finalized blocks (from `_lastAccumulatedHeight + 1` to `finalizedHeight`) using a `metricsMap` built during Step 2. This ensures every finalized block is counted exactly once, regardless of whether the tip block is empty.

The `_hourAccum` bucket receives `txCount`, `daCount`, `daBytes`, `feeAvail`, `blockCount`, `uniqueAppIds`. After one hour it is flushed to `stats.json` and reset.

### Step 4 — Validator Set

- Active stash list from `Session::Validators` storage (SCALE `Vec<AccountId32>`).
- Stake data from `staking::ErasStakers` per stash (current era, paginated prefix scan).
- Commission from `staking::ErasValidatorPrefs` per stash.
- Identity from `identidades.json` via SS58 address lookup.
- SS58 encoding uses blake2b-512 (via `@noble/hashes`) for the checksum; falls back to SHA-512 if the package is unavailable.

#### ErasStakers Storage Format

Avail uses `PagedExposureMetadata` (not the older `Exposure` with full nominator Vec). Each entry is exactly **31 bytes**:

```
compact<Balance>   total          (variable bytes, ~12)
compact<Balance>   own            (variable bytes, ~11)
u32 LE             nominator_count  (4 bytes)
u32 LE             page_count       (4 bytes)
```

The collector detects the format by checking `remaining = buf.length − own.o`:
- `remaining === 8` → PagedExposureMetadata → read `nominator_count` as `u32 LE`
- `remaining > 8` → old `Exposure` → read `Vec<IndividualExposure>` length as compact

Storage key structure (Twox64Concat for both map keys):
```
twox128("Staking") + twox128("ErasStakers")   [64 hex chars = 32 bytes]
twox64(era_u32)                                [16 hex chars = 8 bytes]
era as u32 LE                                  [ 8 hex chars = 4 bytes]
twox64(validator_pubkey)                       [16 hex chars = 8 bytes]
validator_pubkey (AccountId32)                 [64 hex chars = 32 bytes]
```

### Step 5 — Computed Metrics

| Metric | Computation |
|---|---|
| `totalNetworkStake` | Sum of all validators' `totalPlanck` |
| `nakamotoCoefficient` | Minimum validators to cumulatively exceed 33% of total stake (GRANDPA liveness threshold) |
| `avgCommission` | Mean commission across validators with known commission |
| `commissionDistribution` | Counts in brackets: `0%`, `1–5%`, `6–10%`, `11–20%`, `21–100%` |
| `txsPerMinute` | `(txsLast20 / 20) × 5` (~12s/block → 5 blocks/min) |
| `daPerMinute` | Same formula for DA submissions |
| `bytesPerBlock` | `bytesLast20 / 20` (rounded) |

### Step 6 — Write Outputs

- **`data.json`**: atomic write every cycle (live block metrics + validator set).
- **`stats.json`**: hourly flush of `_hourAccum` into `hours` array, then recompute `days`/`weeks`/`months` by aggregation.

---

## Output Schemas

### data.json

```json
{
  "meta": {
    "updatedAt": "2026-06-13T16:23:01.773Z",
    "collectMs": 497,
    "chainId": "avail",
    "nodeVersion": "2.3.4-92228be58bc",
    "specName": "avail",
    "specVersion": 51,
    "tokenSymbol": "AVAIL",
    "tokenDecimals": 18,
    "ss58Format": 42,
    "currentEra": 709,
    "latestHeight": 3063216,
    "finalizedHeight": 3063214,
    "lag": 2,
    "blockTxs": 1,
    "blockDA": 1,
    "blockDABytes": 4584,
    "blockFeeAVAIL": 0.125048,
    "txsLast20Blocks": 23,
    "daLast20Blocks": 23,
    "bytesLast20Blocks": 46090,
    "feesLast20Blocks": 2.87447,
    "txsPerMinute": 5.75,
    "daPerMinute": 5.75,
    "bytesPerBlock": 2305,
    "validatorCount": 80,
    "identityCount": 73,
    "stakeCount": 80,
    "commCount": 80,
    "totalNetworkStake": "4789060544.740694",
    "avgCommission": 16.7,
    "nakamotoCoefficient": 26,
    "commissionDistribution": { "0": 0, "1-5": 8, "6-10": 5, "11-20": 66, "21-100": 1 }
  },
  "validators": [
    {
      "index": 0,
      "stash": "5C56cUbCpzuS221hitmx1dAFhSdXwwi6VUze26hQV911MeTN",
      "stashHex": "0x004c99e77bbc9a9e71f229629a2a38091ac1c7a68e7b47d891862a70a3da5f79",
      "moniker": null,
      "legal": null,
      "web": null,
      "email": null,
      "twitter": null,
      "riot": null,
      "commission": 20,
      "ownStake": "347053.487519",
      "totalStake": "57827632.342437",
      "nominators": 1
    }
  ]
}
```

**`blockTxs` / `blockDA` / `blockDABytes` / `blockFeeAVAIL`**: metrics from the last *finalized* block (not the tip, which is often empty in Avail).

**`nominators`**: number of accounts nominating this validator in the current era, decoded from `ErasStakers` (`PagedExposureMetadata.nominator_count`). Verified against Subscan for all 80 active validators (era 709) — exact match.

### stats.json

```json
{
  "updatedAt": "2026-06-13T10:00:00.000Z",
  "hours": [
    {
      "date": "2026-06-13T09:00:00Z",
      "txCount": 312,
      "daCount": 291,
      "daBytes": 892416,
      "feeAvail": 38.92,
      "blockCount": 180,
      "uniqueAppIds": [1, 2, 7]
    }
  ],
  "days": {
    "2026-06-13": { "date": "2026-06-13", "txCount": 2340, "daCount": 2180, "daBytes": 6291456, "feeAvail": 293.1, "blockCount": 4320, "uniqueAppIds": 5 }
  },
  "weeks": { "2026-06-08": { ... } },
  "months": { "2026-06": { ... } }
}
```

`days`, `weeks`, `months` are recomputed from the full `hours` array on every hourly flush. A maximum of 2160 hours (~90 days) are retained.

---

## Modules

### Chain Stats (`stats.php`)
Page sections: **Live Network · Historical Overview · Period Summary**

Historical overview with interactive charts. Live KPI cards (Transactions, DA Submissions, DA Throughput, Fee Revenue) updated every 30s from `data.json`. Charts draw from `stats.json` for the selected period.

**Period selector:** 7D / 30D / 90D / 1Y / All

**Historical Overview subsections** (`.st-subsec` style):
- Transactions — daily txs + fee bar charts
- Data Availability — DA submissions + throughput bar charts

**Period Summary table:** date, txs, DA subs, DA size, fees, blocks. DA Throughput uses a per-entry sanity cap of 100 MB to filter any corrupt historical entries.

---

### Consensus (`consensus.php`)
Page sections: **Finality Timeline · Block Producers · Recent Blocks**

GRANDPA/BABE consensus state. Displays live finalization lag, era/session progress, and block producer rotation.

---

### Validators (`validators.php`)
Page sections: **Network Stats · Active Set · Stake & Commission**

Active validator set sourced from `data.json → validators`. Sorted by total stake descending by default; all columns sortable.

**Table columns:** rank, moniker/identity (with avatar), stash address (SS58 truncated), total stake (AVAIL), commission %, **nominators**, own stake (AVAIL), network share bar.

**Detail panel** (click any row): commission, nominators, own stake, total stake, self-stake %, network share %, rank by total stake, stash address (full SS58 + hex), identity fields (legal, email, twitter, riot), web link, stake share bar.

**Avatar resolution:** tries logo images from `github.com/Cumulo-pro/validatorsdata/vallogos/` using multiple filename variants (slug, underscore, nospace); falls back to generic validator icon.

---

### Blocks (`blocks.php`)
Page sections: **Network KPIs · Recent Blocks**

Live block stream polling every 6s via Substrate RPC through `rpc.php` proxy. Shows the last 50 blocks.

**Columns:** block number, time ago, extrinsic count, DA submissions count (highlighted in orange if > 0), DA bytes, fee.

DA detection uses pallet `0x1d` / call `0x01`, decoded client-side from raw extrinsic hex with the same SCALE logic as the collector.

---

### Transactions (`txs.php`)
Page sections: **Network KPIs · Transactions · DA · Transfers · Staking**

Recent signed extrinsics from the latest blocks, decoded client-side. Classifies by pallet: DataAvailability, Balances, Staking, etc.

---

### Search (`search.php`)

Accepts block numbers and validator SS58 addresses. Resolves block data via `chain_getBlock` and validator info from `data.json`.

---

## Avail DA — Key Properties

### SCALE Extrinsic Layout

Avail signed extrinsics carry an extra signed extension (`CheckAppId`) not present in standard Substrate:

```
compact<len>
u8 version (0x84 = signed v4)
[if signed:]
  u8 addr_tag (0xFF = AccountId32)
  [32 bytes] public_key
  u8 sig_tag (0x01 = Sr25519)
  [64 bytes] signature
  era (1 byte if immortal, 2 bytes if mortal)
  compact<u32> nonce
  compact<u128> tip
  compact<u32> appId      ← Avail-specific CheckAppId signed extension
pallet_index (u8)
call_index   (u8)
[call args...]
```

`readCompact(b, o)` decodes SCALE compact integers:

| Mode | bits 0–1 | Encoding |
|---|---|---|
| 0 | `00` | Single byte: value = `b[o] >> 2` |
| 1 | `01` | Two bytes LE: value = 14-bit |
| 2 | `10` | Four bytes LE: value = 30-bit |
| 3 | `11` | Big-integer: next `(b[o] >> 2) + 4` bytes LE |

### DA Pallet Constants

| Constant | Value |
|---|---|
| DataAvailability pallet index | `0x1d` (29) — confirmed from live extrinsic inspection |
| `submitData` call index | `0x01` |
| AppId encoding | `compact<u32>` (NOT fixed 4-byte u32) |

### Chain Constants

| Constant | Value |
|---|---|
| Block time | ~12 seconds |
| SS58 prefix | 42 |
| PLANCK | 10^18 (1 AVAIL = 10^18 planck) |
| Token | AVAIL |
| Consensus | GRANDPA (finality) + BABE (block production) |
| `specName` | `avail` |
| Active validators | 80 (era 709) |
| Active nominators | up to 317 per validator |

---

## Data Verification

All 80 validators (era 709) were verified against:
1. **On-chain `staking::ErasStakers`** via Avail public RPC — `totalStake` and `ownStake` match exactly (diff = 0.000 AVAIL across all 80).
2. **Subscan** (`avail.subscan.io/validator/[stash]`) — `totalStake`, `ownStake`, `commission`, and `nominators` verified for a representative sample (Cumulo, Ruby Nodes, Stakeway, Allnodes, Foundation 2, SubWallet Validator, STAKINGCABIN, StakerHouse, Staking4All) — all exact matches.

Total network stake: **4,789.061 M AVAIL** (matches Subscan's "Total Staking 4.789B").

---

## Design Decisions

**Q: Why a server-side collector (not browser-direct)?**

Avail's Substrate RPC does not send CORS headers, blocking cross-origin browser requests. The collector runs server-side and exposes pre-computed JSON files. `blocks.php` and `txs.php` use `rpc.php`, a PHP proxy, for direct RPC calls needed for the live block stream.

---

**Q: Why two output files (`data.json` + `stats.json`)?**

`data.json` is overwritten every ~12s with live block and validator data. `stats.json` accumulates hourly buckets that survive restarts and grow over time (days/weeks/months). Merging them would require reading the full historical dataset on every write cycle.

---

**Q: Why hardcode the DA pallet index (`0x1d`) instead of parsing runtime metadata?**

Runtime metadata parsing is fragile — the binary format changed between Substrate versions and is complex to decode. The pallet index for DataAvailability has been 29 (`0x1d`) since genesis. It is validated via live extrinsic inspection and documented. Any future runtime upgrade that changes it requires a one-line update.

---

**Q: Why use `_lastAccumulatedHeight` instead of always using the tip block (`i=0`)?**

The most recent block (tip) often arrives with no transactions — Avail's block production is ahead of finalization by 1–2 blocks, and transactions appear in finalized blocks. Using the tip for accumulation would systematically undercount DA submissions. The `_lastAccumulatedHeight` tracker ensures every finalized block is counted exactly once.

---

**Q: Why compact-decode `AppId` instead of reading 4 bytes?**

In Avail's runtime, `AppId` is declared with `#[codec(compact)]`, meaning it is SCALE compact-encoded in extrinsics (1–4 bytes, not fixed 4). Reading 4 bytes shifts the offset and causes the subsequent data-length compact decode to read garbage, producing astronomically large byte counts (confirmed: values of ~1.8×10^149 GB before the fix).

---

**Q: Why detect `PagedExposureMetadata` vs old `Exposure` format?**

Avail's staking pallet migrated to paged exposure storage. All current-era entries use `PagedExposureMetadata { total: compact, own: compact, nominator_count: u32, page_count: u32 }` (31 bytes total). The old `Exposure { total, own, others: Vec }` format may still exist for historical eras. The collector detects the format by checking remaining bytes after `own` (8 → paged, otherwise → Vec).

---

## File Structure

```
services/avail/
├── layout-explorer.php    # HTML shell, brand tokens
├── menuexplorer.php       # Sidebar navigation
├── stats.php              # Historical chain statistics (stats.json)
├── consensus.php          # GRANDPA/BABE consensus view
├── validators.php         # Active validator set with nominators column
├── blocks.php             # Live block stream with DA detection
├── txs.php                # Recent transactions
├── search.php             # Block and validator search
└── rpc.php                # Transparent Substrate RPC proxy (CORS workaround)

/var/lib/avail-collector/
├── avail-mainnet-collector.js   # Node.js collector daemon
├── data.json                    # Live output (atomic write, every ~12s)
├── stats.json                   # Historical hourly buckets (flushed once per hour)
└── identidades.json             # Validator identity map (SS58 → moniker, web, twitter…)
```

### Deployment

```bash
# Dependencies
cd /var/lib/avail-collector
npm init -y
npm install @noble/hashes

# Run manually
node avail-mainnet-collector.js

# Install as systemd service
sudo cp avail-collector.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable avail-collector
sudo systemctl start avail-collector

# Verify output
cat /var/lib/avail-collector/data.json | python3 -m json.tool | head -30

# Follow logs
sudo journalctl -u avail-collector -f

# Clean corrupt daBytes from stats.json (one-time fix after appId compact bug)
node -e "
const fs = require('fs');
const s = JSON.parse(fs.readFileSync('/var/lib/avail-collector/stats.json'));
const fix = obj => obj && Object.values(obj).forEach(d => { if (d.daBytes > 8388608000) d.daBytes = 0; });
fix(s.days); fix(s.weeks); fix(s.months);
fs.writeFileSync('/var/lib/avail-collector/stats.json', JSON.stringify(s, null, 2));
console.log('OK');
"
```

### Collector Log Format (per cycle)

```
[blake2b] ✓
[identity] 80 entradas, 73 con moniker
Avail Mainnet Collector v3.0
  RPC: http://avail.rpc.cumulo.me
  data.json  → /var/lib/avail-collector/data.json
  stats.json → /var/lib/avail-collector/stats.json
[accum] +1 DA en bloques 3063213–3063213
[2026-06-13T16:23:01.773Z] blk=3063216 lag=2 txs=1 da=1(fin)/23(20) bytes=4584 fee=0.1250AVAIL vals=80 497ms
```

---

## External Links

| Resource | URL |
|---|---|
| Cumulo Avail Explorer | `https://avail-mainnet.cumulo.com.es` |
| Collector data (live) | `https://avail.explorer.cumulo.org.es/data.json` |
| Collector stats (historical) | `https://avail.explorer.cumulo.org.es/stats.json` |
| Public RPC (HTTP) | `https://avail.rpc.cumulo.me` |
| Public RPC (WebSocket) | `wss://avail.rpc.cumulo.me` |
| Subscan (via Cumulo RPC) | `https://avail.subscan.io/?rpc=wss://avail.rpc.cumulo.me` |
| Avail Explorer (via Cumulo RPC) | `https://explorer.avail.so/?rpc=wss://avail.rpc.cumulo.me` |
| Avail Network | `https://www.availproject.org` |
| Avail docs | `https://docs.availproject.org` |
| Avail GitHub | `https://github.com/availproject` |
| Cumulo validator data | `https://github.com/Cumulo-pro/validatorsdata` |
