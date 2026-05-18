# Celestia — Missed Proposals Tracker

> **Network-specific feature** — This module goes beyond standard Cosmos SDK metrics. It applies the CometBFT weighted round-robin proposer selection algorithm to determine, with block-level precision, which validator missed their proposal turn at each consensus round.

---

## The Problem with Generic Uptime Metrics

Standard Cosmos block explorers track validator signing participation via the `missed_blocks_counter` from the slashing module. This tells you whether a validator **signed** a block — but not whether it **proposed** one when it was its turn.

Block proposal failures are a distinct failure mode:

- A validator can have 100% signing uptime and still repeatedly miss proposal slots
- Missed proposals add latency to block finalization (each skipped round ≈ +2–3 seconds)
- Without identifying *which* validator missed, operators have no actionable signal

Standard metrics provide no visibility into this. The Missed Proposals tracker closes that gap.

---

## How It Works

### Step 1 — Detecting extra-round blocks

Every block header includes `last_commit.round` — the consensus round at which the *previous* block was finally committed. A value of `0` means the first proposer succeeded. A value of `N > 0` means `N` validators were skipped before the block was accepted.

```
block[H].last_commit.round = 2
→ block[H-1] required 3 proposal attempts (rounds 0, 1, 2)
→ 2 validators missed their proposal turn before the block was committed
```

### Step 2 — Identifying who missed

To know *which* validators missed, the collector:

1. Fetches `/validators?height=H-2` — the CometBFT validator set **before** block `H-1`, including each validator's `proposer_priority` at that moment
2. Simulates the CometBFT weighted round-robin algorithm for `commitRound` rounds
3. The last simulated round should match the actual proposer of block `H-1` (sanity check)
4. All rounds before the last one = validators who missed

### Step 3 — CometBFT Proposer Selection Algorithm

Celestia uses the standard CometBFT weighted round-robin (`IncrementProposerPriority`). Each round follows this sequence:

```
1. Rescale priorities if spread exceeds 2 × totalVotingPower
2. Shift all priorities by the average (zero-center)
3. Increment every validator's priority by its voting power
4. Select the validator with the highest priority
   (tie-break: lexicographically smaller hex address wins)
5. Decrement the selected validator's priority by totalVotingPower
```

The algorithm requires `BigInt` arithmetic throughout — Celestia voting power values exceed JavaScript's safe integer limit (`Number.MAX_SAFE_INTEGER`).

**Key RPC endpoint used:**

```
GET /validators?height={H}&per_page=200
```

Returns `proposer_priority` per validator — the state **after** block `H` was committed, used as the starting point to simulate rounds for block `H+1`.

---

## Data Collection

### Collector integration

The missed proposals logic runs inside the main `collect()` cycle, after uptime data is fetched and validators are assembled. It operates on `blockData` — a per-block array built as a byproduct of the uptime batch fetch:

```js
blockData.push({ h, time, proposer, commitRound, txs })
```

For each block where `commitRound > 0` and the previous block is contiguous in the batch, the collector:

1. Fetches the validator set at `height - 2` (async, one RPC call per missed event)
2. Runs `cometBFTRoundProposers(valSet, commitRound + 1)`
3. Filters out the actual proposer from the missed list
4. Appends the event to the persistent history

### Persistent history file

Missed events accumulate in a dedicated JSON file, separate from `data.json`:

```
/var/lib/celestia-collector/missed-proposals.json        (mainnet)
/var/lib/celestia-mocha-collector/missed-proposals.json  (mocha testnet)
```

The file grows incrementally — new events are deduplicated by block height and merged on every cycle. It is never truncated; the full history since the first collector run is preserved.

**File structure:**

```json
{
  "startBlock": 11144453,
  "lastProcessedHeight": 11144900,
  "updatedAt": "2025-05-18T14:22:01.000Z",
  "events": [
    {
      "height": 11144619,
      "time": "2025-05-18T14:10:33.000Z",
      "round": 1,
      "txs": 13,
      "missed": [
        { "r": 0, "op": "celestiavaloper1...", "m": "alphab.ai" }
      ],
      "proposer": { "op": "celestiavaloper1...", "m": "B-Harvest" }
    }
  ],
  "validatorStats": {
    "celestiavaloper1...": { "moniker": "alphab.ai", "proposed": 1, "missed": 1 }
  }
}
```

| Field | Description |
|---|---|
| `startBlock` | First block processed — defines the tracking window start |
| `lastProcessedHeight` | Prevents double-counting proposed blocks across cycles |
| `events[]` | One entry per block committed at round > 0 |
| `events[].missed[]` | Validators who skipped, in round order (`r`: round index) |
| `events[].proposer` | Validator who ultimately committed the block |
| `validatorStats` | Cumulative proposed + missed counts per operator address |

### Atomic writes

The file is written atomically using the same pattern as `data.json`:

```js
await writeFile(MISSED_FILE + ".tmp", JSON.stringify(history), "utf8");
await rename(MISSED_FILE + ".tmp", MISSED_FILE);
```

This prevents partial reads by the frontend or concurrent processes.

---

## Output — `data.json` fields

The main collector output (`data.json`) includes two new top-level arrays and two new `meta` fields:

### `meta.missedSince`
Block height at which tracking began. `null` until the first cycle completes.

### `meta.missedTotal`
Total number of missed proposal events recorded in the persistent history.

### `proposalStats[]`
Per-validator array, sorted by `missed` descending. Only active (bonded) validators are included.

```json
{
  "operator":    "celestiavaloper1...",
  "moniker":     "alphab.ai",
  "avatar":      "https://...",
  "proposed":    1,
  "missed":      1,
  "missRate":    50.000,
  "votingShare": 0.69
}
```

| Field | Source | Description |
|---|---|---|
| `proposed` | `validatorStats` in history | Real blocks proposed since tracking start |
| `missed` | `validatorStats` in history | Real proposal slots missed since tracking start |
| `missRate` | `missed / (proposed + missed) × 100` | Percentage of turns missed |
| `votingShare` | Current validator set | Current cumulative voting power share |

### `missedPropEvents[]`
Last 50 events from the history, most recent first, enriched with current avatars.

```json
{
  "height": 11144619,
  "time":   "2025-05-18T14:10:33.000Z",
  "round":  1,
  "txs":    13,
  "missed": [
    { "r": 0, "op": "celestiavaloper1...", "m": "alphab.ai", "avatar": "https://..." }
  ],
  "proposer": { "op": "celestiavaloper1...", "m": "B-Harvest", "avatar": "https://..." }
}
```

Avatars are resolved from the current validator set on each cycle — not stored in the history file — keeping the history compact.

---

## Frontend Page

**Routes:**
- Mainnet: `https://cumulo.pro/services/celestia/missed-proposals`
- Mocha testnet: `https://cumulo.pro/services/celestia_mocha/missed-proposals`

**Data source:** `data.json` via `DATA_URL`, polled every 6 seconds (same as all other explorer pages).

**Key UI sections:**

| Section | Data source | Description |
|---|---|---|
| Status bar | `meta` | Chain ID, latest block, tracking start block, network health % |
| KPI strip | `proposalStats` aggregated | Total missed, network miss rate, health badge, blocks tracked |
| Warning banner | `proposalStats[0]` | Shown only when at least one validator has ≥ 1 real miss |
| Recent Missed Events | `missedPropEvents` | Block, time, round, who missed (with avatars), actual proposer |
| Proposal Statistics | `proposalStats` | Full per-validator table: proposed, missed, miss rate, vote share |

**Tech stack:** React 18 (UMD/Babel, no build step), Tailwind CSS via CDN — consistent with all other explorer pages.

---

## Limitations

- **Cold start** — Counts begin from the first collector run. There is no retroactive backfill from chain history.
- **Window dependency** — Missed events are only detected within the `UPTIME_BLOCKS` batch fetched each cycle (300 blocks, ~30 min). Blocks outside this window that had `commitRound > 0` are not captured.
- **Sanity check only** — The simulation is validated against the known actual proposer as a cross-check. If there is a mismatch (validator set changes mid-simulation, network upgrade, etc.), the event is still recorded but a warning is logged.
- **Inactive validators** — Only bonded validators appear in `proposalStats`. Jailed or unbonded validators are excluded from the stats table.

---

## Networks

| Network | Chain ID | Collector file | History file | Dashboard |
|---|---|---|---|---|
| Celestia Mainnet | `celestia` | `celestia-collector.js` | `/var/lib/celestia-collector/missed-proposals.json` | [cumulo.pro/services/celestia/missed-proposals](https://cumulo.pro/services/celestia/missed-proposals) |
| Celestia Mocha | `mocha-4` | `celestia-mocha-collector.js` | `/var/lib/celestia-mocha-collector/missed-proposals.json` | [cumulo.pro/services/celestia_mocha/missed-proposals](https://cumulo.pro/services/celestia_mocha/missed-proposals) |
