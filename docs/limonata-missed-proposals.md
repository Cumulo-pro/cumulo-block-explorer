# Limonata : Missed Proposals & Slow Proposers Tracker

> **Network-specific feature**. This module goes beyond standard Cosmos SDK metrics. It applies the CometBFT weighted round-robin proposer selection algorithm to determine, with block-level precision, which validator missed their proposal turn at each consensus round, and separately tracks real block latency per proposer.

---

## The Problem with Generic Uptime Metrics

Standard Cosmos block explorers track validator signing participation via the `missed_blocks_counter` from the slashing module. This tells you whether a validator **signed** a block, but not whether it **proposed** one when it was its turn.

Block proposal failures are a distinct failure mode:

- A validator can have 100% signing uptime and still repeatedly miss proposal slots
- Missed proposals add latency to block finalization
- Without identifying *which* validator missed, operators have no actionable signal

Standard metrics provide no visibility into this. The Missed Proposals tracker closes that gap. A related but distinct problem: a validator can propose successfully *every single time* and still be a network liability if its blocks are consistently slower than average. The **Slow Proposers** module (new to Limonata, see [below](#slow-proposers-proposer-latency)) closes that second gap.

---

## How It Works

### Step 1. Detecting extra-round blocks

Every block header includes `last_commit.round`, the consensus round at which the *previous* block was finally committed. A value of `0` means the first proposer succeeded. A value of `N > 0` means `N` proposal rounds failed before the block was accepted.

```
block[H].last_commit.round = 1
→ block[H-1] required 2 proposal attempts (rounds 0 and 1)
→ 1 validator missed their proposal turn before the block was committed
```

This value is cross-checked against the authoritative `/commit?height=H-1` endpoint before any event is recorded (see [False Positive Prevention](#false-positive-prevention)).

### Step 2. Identifying who missed

To know *which* validator missed, the collector:

1. Fetches `/validators?height=H-2`, which returns the CometBFT validator set **before** block `H-1`, including each validator's `proposer_priority` at that moment
2. Simulates the CometBFT weighted round-robin algorithm for `commitRound + 1` rounds
3. Verifies that round `commitRound` of the simulation matches the actual proposer of block `H-1` (sanity check)
4. Rounds `0` through `commitRound - 1` = validators who missed

### Step 3. CometBFT Proposer Selection Algorithm

#### What is weighted round-robin?

In a simple round-robin, each validator takes turns proposing one block at a time in a fixed cycle. In a **weighted** round-robin, validators with more voting power (stake) get proportionally more turns. A validator with 10% of the total stake should propose roughly 10% of all blocks over time.

CometBFT achieves this through a `proposer_priority` score maintained for every validator. The priority accumulates over rounds and is reset after each proposal. The key property is that **the system is deterministic and stateless**. Given the validator set and their priorities at any block height, anyone can compute exactly who will be selected as proposer for every future round without needing any additional information.

#### The `IncrementProposerPriority` algorithm

Each round follows this sequence (defined in the [CometBFT spec](https://github.com/cometbft/cometbft/blob/main/spec/consensus/proposer-selection.md)):

```
1. Rescale priorities if spread exceeds 2 × totalVotingPower
   (prevents priority overflow over long periods)
2. Shift all priorities by the average (zero-center the distribution)
3. Increment every validator's priority by its voting power
   (validators "earn" priority each round they don't propose)
4. Select the validator with the highest priority as proposer
   (tie-break: lexicographically smaller hex address wins)
5. Decrement the selected validator's priority by totalVotingPower
   (the proposer "spends" their accumulated priority)
```

Over time this guarantees each validator proposes in proportion to their stake. A high-stake validator will reach the top of the priority ranking more frequently because their priority grows faster (step 3).

The algorithm requires `BigInt` arithmetic throughout. Voting power values on Limonata (`aLIMO`, 18 decimals) exceed JavaScript's safe integer limit (`Number.MAX_SAFE_INTEGER`).

**Key RPC endpoint used:**

```
GET /validators?height={H}&per_page=200
```

Returns `proposer_priority` per validator. The state **after** block `H` was committed, used as the starting point to simulate rounds for block `H+1`.

---

## False Positive Prevention

Reading `last_commit.round` from the block RPC endpoint can yield transient stale values under certain conditions (pruning node state, parallel batch fetching). To prevent false positives, the collector performs a mandatory cross-check against the authoritative commit endpoint **before recording any event**:

```
GET /commit?height={H}
→ result.signed_header.commit.round  (canonical consensus round)
```

Logic applied on each candidate event:

```
canonicalRound = 0   → skip (false positive confirmed, block committed normally)
canonicalRound < 0   → fetch failed, proceed with block value + log warning
canonicalRound ≠ blockRound → use canonical value, log discrepancy
```

This guard eliminates false positives caused by RPC inconsistencies. Only events where `/commit` confirms `round > 0` are recorded.

**Empirically confirmed on Limonata testnet**: during initial testing, 3 out of the first 3 candidate blocks scanned (all showing `last_commit.round = 1` in the raw block header) turned out to be false positives, `/commit` confirmed `round = 0` for all three. Without this guard, the tracker would have recorded three phantom missed-proposal events against validators who had done nothing wrong. A wider sample of 20 additional blocks showed the block-header round field to be reliable outside of these transient cases.

---

## What "Missed" Actually Means

A missed proposal is an **inference**, not a direct on-chain observation. The blockchain records what *did* happen, never what *didn't*.

What is directly observable:
- A block was committed at round `N > 0` - **on-chain fact**
- The actual proposer (round `N`) - **on-chain fact**
- The validator scheduled for round `0..N-1` by the deterministic algorithm - **mathematically computable from public data**

What is inferred:
- The scheduled validator did not successfully propose in their round - **inference** (no explicit on-chain record)

This inference is the same methodology used by all CometBFT missed-proposal trackers. The important distinction is that a validator can "miss" their proposal turn for different reasons: being offline, network latency causing a timeout, or proposing a block that was rejected by peers. The tracker does not distinguish between these causes, it only records that the round was skipped.

Note: if a validator appears in the block's `last_commit.signatures` (i.e., signed the block proposed by the successor), it was online at the time, meaning the miss was due to a proposal timeout, not a full outage.

---

## Round Semantics

The `round` field in each event represents the consensus round at which the block was **finally committed**, not the round where the miss occurred.

For the most common case (a single validator miss):

```
Round 0 (scheduled proposer) → did not propose  ← the miss
Round 1 (next proposer)      → proposed and committed the block  ← round shown
```

A value of `Round 1` means one validator missed. `Round 2` means two validators missed consecutively before a third committed the block. The `missed[]` array in each event lists the validators for each failed round in order.

---

## End-to-End Verification

Every event recorded by this tracker is independently verifiable using only public RPC endpoints. The following is a complete worked example using block **#1,439,706** on Limonata testnet.

### Step 1 - Confirm the block required an extra round

```bash
curl -s "https://rpc-testnet.limonata.cumulo.me/commit?height=1439706" | python3 -m json.tool | grep round
# → "round": 1
```

### Step 2 - Confirm the actual proposer (round 1)

```bash
curl -s "https://rpc-testnet.limonata.cumulo.me/block?height=1439706" | python3 -m json.tool | grep proposer_address
# → "proposer_address": "AFB7DF203C928F2839F6344F28BD8CFE1266C3BC"
```

### Step 3 - Simulate the proposer selection algorithm

Fetch the validator set at height 1,439,705 (state before block 1,439,706) and run the CometBFT weighted round-robin for 2 rounds:

```js
// Reproducible script - requires Node.js with native fetch (v18+)
const RPC = "https://rpc-testnet.limonata.cumulo.me";
const HEIGHT = 1439705;
const ACTUAL_PROPOSER = "AFB7DF203C928F2839F6344F28BD8CFE1266C3BC"; // Vinjan.Inc

const d = await fetch(`${RPC}/validators?height=${HEIGHT}&per_page=200`).then(r => r.json());
const vals = d.result.validators;
const totalPower = vals.reduce((s, v) => s + BigInt(v.voting_power), 0n);
const diffMax = 2n * totalPower;

const prios = vals.map(v => ({
  address:  v.address.toUpperCase(),
  priority: BigInt(v.proposer_priority),
  power:    BigInt(v.voting_power),
}));

function runRound() {
  // 1. Rescale
  const maxP = prios.reduce((m, p) => p.priority > m ? p.priority : m, prios[0].priority);
  const minP = prios.reduce((m, p) => p.priority < m ? p.priority : m, prios[0].priority);
  const diff = maxP - minP;
  if (diff > diffMax && diff > 0n)
    prios.forEach(p => {
      const n = p.priority * diffMax;
      p.priority = n >= 0n ? (n + diff/2n)/diff : -((-n + diff/2n)/diff);
    });
  // 2. Shift by average
  const sum = prios.reduce((s, p) => s + p.priority, 0n);
  const n = BigInt(prios.length);
  const avg = sum >= 0n ? (sum + n/2n)/n : -((-sum + n/2n)/n);
  prios.forEach(p => p.priority -= avg);
  // 3. Increment by voting power
  prios.forEach(p => p.priority += p.power);
  // 4. Select highest priority (lexicographic tie-break)
  let best = prios[0];
  prios.forEach(p => {
    if (p.priority > best.priority || (p.priority === best.priority && p.address < best.address))
      best = p;
  });
  // 5. Decrement selected by totalPower
  best.priority -= totalPower;
  return best.address;
}

const round0 = runRound(); // → 3CF47F6336AEAE6F6312F2EA6C094F9B011E91E8
const round1 = runRound(); // → AFB7DF203C928F2839F6344F28BD8CFE1266C3BC  ✓ matches actual proposer
```

**Result:**

```
Round 0 proposer (scheduled, missed):  3CF47F6336AEAE6F6312F2EA6C094F9B011E91E8
Round 1 proposer (simulated):          AFB7DF203C928F2839F6344F28BD8CFE1266C3BC
Actual proposer (RPC):                 AFB7DF203C928F2839F6344F28BD8CFE1266C3BC  ✓ MATCH
```

The simulation round 1 matches the actual proposer - the algorithm is verified correct for this block. Therefore round 0 is also correct.

### Step 4 - Resolve the round 0 hex address to a validator

```bash
# Get the pubkey for the round-0 address
curl -s "https://rpc-testnet.limonata.cumulo.me/validators?height=1439705&per_page=200" \
  | python3 -m json.tool | grep -B2 -A3 "3CF47F6336AEAE6F6312F2EA6C094F9B011E91E8"
# → "value": "THhVVd2JBvqiRLsN4aRTNlkuTsLM8wkqXebSRheASGA="

# Map pubkey to operator address
curl -s "https://api-testnet.limonata.cumulo.me/cosmos/staking/v1beta1/validators?pagination.limit=300" \
  | python3 -m json.tool | grep -B5 "THhVVd2JBvqiRLsN4aRTNlkuTsLM8wkqXebSRheASGA="
# → "operator_address": "cosmosvaloper1tyrumv6hesw2rvf50qaj7v67tmr7f579m9symu"

# Get moniker
curl -s "https://api-testnet.limonata.cumulo.me/cosmos/staking/v1beta1/validators/cosmosvaloper1tyrumv6hesw2rvf50qaj7v67tmr7f579m9symu" \
  | python3 -m json.tool | grep moniker
```

### Verified proof chain

| Step | Claim | Source | Verifiable |
|---|---|---|---|
| 1 | Block 1,439,706 committed at round 1 | `/commit` RPC | ✓ Public |
| 2 | Actual proposer hex = `AFB7DF...C3BC` | `/block` RPC | ✓ Public |
| 3 | Algorithm round 1 = `AFB7DF...C3BC` | Simulation from public data | ✓ Reproducible |
| 4 | Algorithm round 0 = `3CF47F...E91E8` | Same simulation | ✓ Reproducible |
| 5 | `3CF47F...E91E8` pubkey = `THhVVd...ASGA=` | `/validators` RPC | ✓ Public |
| 6 | Pubkey maps to `cosmosvaloper1tyrumv6...symu` | Staking API | ✓ Public |
| 7 | `cosmosvaloper1tyrumv6...symu` and `cosmosvaloper1lv8dr4d...frgfgrs` monikers | Staking API | ✓ Public |

**Conclusion:** every step in this chain uses publicly accessible RPC endpoints and is independently reproducible by anyone, no data from the collector or `data.json` was used in this verification.

---

## Data Collection

### Collector integration

The missed proposals logic runs inside the main `collect()` cycle, after uptime data is fetched and validators are assembled. It operates on `blockData` - a per-block array built as a byproduct of the uptime batch fetch:

```js
blockData.push({ h, time, proposer, commitRound, txs })
```

For each block where `commitRound > 0` and the previous block is contiguous in the batch, the collector:

1. Verifies via `/commit?height=H` that the round is canonical (false positive guard)
2. Fetches the validator set at `height - 2` (one RPC call per missed event)
3. Runs `cometBFTRoundProposers(valSet, commitRound + 1)`
4. Sanity-checks that `roundAddrs[commitRound]` matches the actual block proposer
5. Records rounds `0..commitRound-1` as missed, appends the event to the persistent history

If the collector falls behind (restart, node downtime), it extends the scan window back to `lastProcessedHeight` (capped at `MISSED_SCAN_CAP` = 3,000 blocks, roughly 5 hours at Limonata's ~6s configured block time) so gaps don't silently drop events.

### Persistent history file

Missed events and proposer latency accumulate in a dedicated JSON file, separate from `data.json`:

```
/var/lib/limonata-testnet-collector/missed-proposals.json
```

The file grows incrementally - new events are deduplicated by block height and merged on every cycle. It is never truncated; the full history since the first collector run is preserved.

**File structure:**

```json
{
  "startBlock": 1436521,
  "lastProcessedHeight": 1440405,
  "updatedAt": "2026-08-01T18:05:00.000Z",
  "events": [
    {
      "height": 1439706,
      "time": "2026-08-01T17:20:00.000Z",
      "round": 1,
      "txs": 1,
      "missed": [
        { "r": 0, "op": "cosmosvaloper1tyrumv6...", "m": "Chicharito" }
      ],
      "proposer": { "op": "cosmosvaloper1lv8dr4d...", "m": "Vinjan.Inc" }
    }
  ],
  "validatorStats": {
    "cosmosvaloper1tyrumv6...": { "moniker": "Chicharito", "proposed": 2, "missed": 31 }
  },
  "latencyStats": {
    "cosmosvaloper1...": { "moniker": "Sr20de", "count": 9, "sum": 25.02, "max": 7.84 }
  }
}
```

| Field | Description |
|---|---|
| `startBlock` | First block processed - defines the tracking window start |
| `lastProcessedHeight` | Prevents double-counting proposed blocks across cycles |
| `events[]` | One entry per block committed at round > 0 |
| `events[].missed[]` | Validators who skipped, in round order (`r`: round index) |
| `events[].proposer` | Validator who ultimately committed the block |
| `validatorStats` | Cumulative proposed + missed counts per operator address |
| `latencyStats` | Cumulative block-time samples per operator address (`count`, `sum` in seconds, `max` in seconds), powers the Slow Proposers table |

### Atomic writes

The file is written atomically using the same pattern as `data.json`:

```js
await writeFile(MISSED_FILE + ".tmp", JSON.stringify(history), "utf8");
await rename(MISSED_FILE + ".tmp", MISSED_FILE);
```

This prevents partial reads by the frontend or concurrent processes.

### Backups

`saveMissedHistory()` also writes:
- A daily backup (`missed-proposals.json.bak`), overwritten once per day
- A permanent monthly backup (`missed-proposals.json.bak.YYYY-MM`), never overwritten

Both live on the same disk as the primary file, so they protect against corruption or accidental overwrites but not against total loss of `/var/lib/limonata-testnet-collector/`. Because the full history file is served publicly (same `/data/` Nginx location as `data.json`), it can also be pulled from an independent location on a schedule for off-server redundancy:

```bash
curl -s "https://explorer.limonata-testnet.cumulo.me/data/missed-proposals.json" \
  -o "missed-proposals-$(date +%Y%m%d).json"
```

---

## Output - `data.json` fields

The main collector output (`data.json`) includes new top-level arrays and `meta` fields:

### `meta.missedSince`
Block height at which tracking began. `null` until the first cycle completes.

### `meta.missedTotal`
Total number of missed proposal events recorded in the persistent history.

### `proposalStats[]`
Per-validator array, sorted by `missed` descending. Includes all bonded validators plus any jailed or unbonded validators that have accumulated missed events since tracking began - ensuring that validators who go offline do not disappear from the statistics table. A validator with `proposed + missed === 0` has simply not had a proposer turn yet (common for low voting-share validators in a short tracking window) and is rendered as "no data" rather than a false `0%`.

```json
{
  "operator":    "cosmosvaloper1...",
  "moniker":     "Chicharito",
  "avatar":      "https://...",
  "active":      true,
  "jailed":      false,
  "proposed":    2,
  "missed":      31,
  "missRate":    93.939,
  "votingShare": 0.81
}
```

| Field | Source | Description |
|---|---|---|
| `proposed` | `validatorStats` in history | Real blocks proposed since tracking start |
| `missed` | `validatorStats` in history | Real proposal slots missed since tracking start |
| `missRate` | `missed / (proposed + missed) × 100` | Percentage of turns missed |
| `votingShare` | Current validator set | Current cumulative voting power share |
| `active` | Current staking status | `false` if jailed or unbonded |
| `jailed` | Current staking status | `true` if currently jailed |

### `missedPropEvents[]`
Last 50 events from the history, most recent first, enriched with current avatars.

### `proposerLatency[]`
Per-validator array, sorted by average block time descending. See [Slow Proposers](#slow-proposers-proposer-latency) below.

Avatars are resolved from the current validator set on each cycle - not stored in the history file - keeping the history compact.

---

## Slow Proposers (proposer latency)

New module, not present in the Celestia version of this tracker. It answers a question `proposalStats` cannot: *is this validator's block consistently slow to produce, even when it never misses round 0?*

### How it works

For every block already processed for `proposalStats` (i.e. every block newer than `lastProcessedHeight`), the collector also computes:

```
dt = time(block[h]) − time(block[h-1])   (seconds, only if h and h-1 are contiguous heights)
```

`dt` is attributed to the proposer of `block[h]` and accumulated into `history.latencyStats[operator]` as `{ count, sum, max }`. Deltas outside `(0, 120)` seconds are discarded (collector restarts, node gaps) to avoid polluting the average with a single huge outlier.

This reuses the exact same `blockData`/`sortedBD` array already built for missed-proposals detection, no extra RPC calls are needed.

### Output

```json
{
  "operator":     "cosmosvaloper1...",
  "moniker":      "Sr20de",
  "avatar":       "https://...",
  "active":       true,
  "jailed":       false,
  "count":        9,
  "avgBlockTime": 2.78,
  "maxBlockTime": 7.84
}
```

The frontend compares `avgBlockTime` against `meta.blockTime` (the network-wide rolling average) to color-code outliers: ≥2× network average in red/orange, ≥1.3× in amber.

### Interpretation

Small `count` values are noisy, a validator with 1-2 turns and a single slow block will show a misleadingly high average. Treat rows with `count < 10` as low-confidence until more samples accumulate. This is a real limitation observed in practice on Limonata testnet: several validators initially named as "slow proposers" from manual spot-checks did not reproduce the pattern once `proposerLatency` accumulated more samples per validator, while at least one validator not originally flagged (`Sr20de`) emerged as a genuine outlier only after ~9 turns.

---

## Limitations

- **Cold start** - Counts begin from the first collector run. There is no retroactive backfill from chain history.
- **Window dependency** - Missed events are only guaranteed to be detected within `MISSED_SCAN_CAP` blocks of a gap (3,000 blocks, ~5 hours at Limonata's configured block time). Larger gaps than that are not backfilled.
- **Inference, not observation** - The tracker identifies *which* validator was scheduled to propose in each failed round. It cannot determine *why* the proposal failed (offline, timeout, proposal rejected). A validator that signed the committed block was online; one that did not sign may have been offline.
- **Validator set changes** - If the active validator set changes mid-simulation (e.g., immediately after a large stake change), the sanity check may fail. The event is still recorded with a warning logged.
- **Slow Proposers sample size** - See [Interpretation](#interpretation) above; low-turn validators need more data before their average is meaningful.

---

## Networks

| Network | Chain ID | Collector file | History file | Dashboard |
|---|---|---|---|---|
| Limonata Testnet | `limonata_10777-1` | `limonata-testnet-collector.js` | `/var/lib/limonata-testnet-collector/missed-proposals.json` | [cumulo.pro/services/limonata_testnet/missed-proposals](https://cumulo.pro/services/limonata_testnet/missed-proposals) |

---

## References

| Source | Description |
|---|---|
| [CometBFT - Proposer Selection Spec](https://github.com/cometbft/cometbft/blob/main/spec/consensus/proposer-selection.md) | Official specification of the weighted round-robin proposer selection algorithm (`IncrementProposerPriority`) |
| [CometBFT - validator_set.go](https://github.com/cometbft/cometbft/blob/main/types/validator_set.go) | Reference implementation of `IncrementProposerPriority` in Go |
| [CometBFT RPC - `/commit`](https://docs.cometbft.com/v0.38/rpc/#/Info/commit) | Authoritative endpoint for the canonical consensus round of a committed block |
| [CometBFT RPC - `/validators`](https://docs.cometbft.com/v0.38/rpc/#/Info/validators) | Returns the validator set with `proposer_priority` at a given height |
