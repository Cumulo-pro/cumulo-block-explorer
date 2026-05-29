# Celestia : Missed Proposals Tracker

> **Network-specific feature** - This module goes beyond standard Cosmos SDK metrics. It applies the CometBFT weighted round-robin proposer selection algorithm to determine, with block-level precision, which validator missed their proposal turn at each consensus round.

---

## The Problem with Generic Uptime Metrics

Standard Cosmos block explorers track validator signing participation via the `missed_blocks_counter` from the slashing module. This tells you whether a validator **signed** a block - but not whether it **proposed** one when it was its turn.

Block proposal failures are a distinct failure mode:

- A validator can have 100% signing uptime and still repeatedly miss proposal slots
- Missed proposals add latency to block finalization (each skipped round ≈ +2–3 seconds)
- Without identifying *which* validator missed, operators have no actionable signal

Standard metrics provide no visibility into this. The Missed Proposals tracker closes that gap.

---

## How It Works

### Step 1 - Detecting extra-round blocks

Every block header includes `last_commit.round` - the consensus round at which the *previous* block was finally committed. A value of `0` means the first proposer succeeded. A value of `N > 0` means `N` proposal rounds failed before the block was accepted.

```
block[H].last_commit.round = 1
→ block[H-1] required 2 proposal attempts (rounds 0 and 1)
→ 1 validator missed their proposal turn before the block was committed
```

This value is cross-checked against the authoritative `/commit?height=H-1` endpoint before any event is recorded (see [False Positive Prevention](#false-positive-prevention)).

### Step 2 - Identifying who missed

To know *which* validator missed, the collector:

1. Fetches `/validators?height=H-2` - the CometBFT validator set **before** block `H-1`, including each validator's `proposer_priority` at that moment
2. Simulates the CometBFT weighted round-robin algorithm for `commitRound + 1` rounds
3. Verifies that round `commitRound` of the simulation matches the actual proposer of block `H-1` (sanity check)
4. Rounds `0` through `commitRound - 1` = validators who missed

### Step 3 - CometBFT Proposer Selection Algorithm

Celestia uses the standard CometBFT weighted round-robin (`IncrementProposerPriority`). Each round follows this sequence:

```
1. Rescale priorities if spread exceeds 2 × totalVotingPower
2. Shift all priorities by the average (zero-center)
3. Increment every validator's priority by its voting power
4. Select the validator with the highest priority
   (tie-break: lexicographically smaller hex address wins)
5. Decrement the selected validator's priority by totalVotingPower
```

The algorithm requires `BigInt` arithmetic throughout - Celestia voting power values exceed JavaScript's safe integer limit (`Number.MAX_SAFE_INTEGER`).

**Key RPC endpoint used:**

```
GET /validators?height={H}&per_page=200
```

Returns `proposer_priority` per validator - the state **after** block `H` was committed, used as the starting point to simulate rounds for block `H+1`.

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

---

## What "Missed" Actually Means

A missed proposal is an **inference**, not a direct on-chain observation. The blockchain records what *did* happen, never what *didn't*.

What is directly observable:
- A block was committed at round `N > 0` - **on-chain fact**
- The actual proposer (round `N`) - **on-chain fact**
- The validator scheduled for round `0..N-1` by the deterministic algorithm - **mathematically computable from public data**

What is inferred:
- The scheduled validator did not successfully propose in their round - **inference** (no explicit on-chain record)

This inference is the same methodology used by all CometBFT missed-proposal trackers. The important distinction is that a validator can "miss" their proposal turn for different reasons: being offline, network latency causing a timeout, or proposing a block that was rejected by peers. The tracker does not distinguish between these causes - it only records that the round was skipped.

Note: if a validator appears in the block's `last_commit.signatures` (i.e., signed the block proposed by the successor), it was online at the time - meaning the miss was due to a proposal timeout, not a full outage.

---

## Round Semantics

Different tools display the `round` field with different semantics:

| Dashboard | Value shown | Meaning |
|---|---|---|
| **This tracker** | `Round 1` | Round at which the block was finally committed |
| **Krews** | `Round 0` | Round at which the missed validator was scheduled |

Both refer to the same event. For a single-validator miss (the most common case):
- Our `Round 1` = the block needed one extra round
- Krews' `Round 0` = the validator who missed was scheduled for the first round

They are complementary views of the same consensus failure.

---

## End-to-End Verification

Every event recorded by this tracker is independently verifiable using only public RPC endpoints. The following is a complete worked example using block **#11,297,453** on Celestia mainnet.

### Step 1 - Confirm the block required an extra round

```bash
curl -s "https://celestia.cumulo.org.es/commit?height=11297453" | python3 -m json.tool | grep round
# → "round": 1
```

Block 11,297,453 was committed at round 1. Independently confirmed by Mintscan (`Round 1` displayed on the block page).

### Step 2 - Confirm the actual proposer (round 1)

```bash
curl -s "https://celestia.cumulo.org.es/block?height=11297453" | python3 -m json.tool | grep proposer_address
# → "proposer_address": "3DEA7F647851564D6764306F108921BBFC29ADCE"
```

Mintscan independently shows `Proposer: senggigi` for this block.

### Step 3 - Simulate the proposer selection algorithm

Fetch the validator set at height 11,297,452 (state before block 11,297,453) and run the CometBFT weighted round-robin for 2 rounds:

```js
// Reproducible script - requires Node.js with native fetch (v18+)
const RPC = "https://celestia.cumulo.org.es";
const HEIGHT = 11297452;
const ACTUAL_PROPOSER = "3DEA7F647851564D6764306F108921BBFC29ADCE"; // senggigi

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

const round0 = runRound(); // → 126403A2CAA36DBDE0FB64A7AE72ED82979366F7
const round1 = runRound(); // → 3DEA7F647851564D6764306F108921BBFC29ADCE  ✓ matches actual proposer
```

**Result:**
```
Round 0 proposer (scheduled, missed):  126403A2CAA36DBDE0FB64A7AE72ED82979366F7
Round 1 proposer (simulated):          3DEA7F647851564D6764306F108921BBFC29ADCE
Actual proposer (RPC + Mintscan):      3DEA7F647851564D6764306F108921BBFC29ADCE  ✓ MATCH
```

The simulation round 1 matches the actual proposer - the algorithm is verified correct for this block. Therefore round 0 is also correct.

### Step 4 - Resolve the round 0 hex address to a validator

```bash
# Get the pubkey for the round-0 address
curl -s "https://celestia.cumulo.org.es/validators?height=11297452&per_page=200" \
  | python3 -m json.tool | grep -A3 "126403A2CAA36DBDE0FB64A7AE72ED82979366F7"
# → "value": "1NfzJ7SfbqNONm8nVvSkvUhl9srbpv1NrOerb3LFJmI="

# Map pubkey to operator address
curl -s "https://celestia.api.cumulo.org.es/cosmos/staking/v1beta1/validators?pagination.limit=300&status=BOND_STATUS_BONDED" \
  | python3 -m json.tool | grep -B5 "1NfzJ7SfbqNONm8nVvSkvUhl9srbpv1NrOerb3LFJmI="
# → "operator_address": "celestiavaloper1v987evnk7hsqct7smdqpxqprhvlcxgt43kyewc"

# Get moniker
curl -s "https://celestia.api.cumulo.org.es/cosmos/staking/v1beta1/validators/celestiavaloper1v987evnk7hsqct7smdqpxqprhvlcxgt43kyewc" \
  | python3 -m json.tool | grep moniker
# → "moniker": "alphab.ai"
```

### Verified proof chain

| Step | Claim | Source | Verifiable |
|---|---|---|---|
| 1 | Block 11,297,453 committed at round 1 | `/commit` RPC + Mintscan | ✓ Public |
| 2 | Actual proposer = senggigi (`3DEA7F...ADCE`) | `/block` RPC + Mintscan | ✓ Public |
| 3 | Algorithm round 1 = `3DEA7F...ADCE` | Simulation from public data | ✓ Reproducible |
| 4 | Algorithm round 0 = `126403...66F7` | Same simulation | ✓ Reproducible |
| 5 | `126403...66F7` pubkey = `1NfzJ7...JmI=` | `/validators` RPC | ✓ Public |
| 6 | Pubkey maps to `celestiavaloper1v987...ewc` | Staking API | ✓ Public |
| 7 | `celestiavaloper1v987...ewc` moniker = **alphab.ai** | Staking API | ✓ Public |

**Conclusion:** alphab.ai was the CometBFT-scheduled proposer for round 0 of block 11,297,453 and did not successfully propose. Every step in this chain uses publicly accessible RPC endpoints and is reproducible by anyone.

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

### Persistent history file

Missed events accumulate in a dedicated JSON file, separate from `data.json`:

```
/var/lib/celestia-collector/missed-proposals.json        (mainnet)
/var/lib/celestia-mocha-collector/missed-proposals.json  (mocha testnet)
```

The file grows incrementally - new events are deduplicated by block height and merged on every cycle. It is never truncated; the full history since the first collector run is preserved.

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
| `startBlock` | First block processed - defines the tracking window start |
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

## Output - `data.json` fields

The main collector output (`data.json`) includes two new top-level arrays and two new `meta` fields:

### `meta.missedSince`
Block height at which tracking began. `null` until the first cycle completes.

### `meta.missedTotal`
Total number of missed proposal events recorded in the persistent history.

### `proposalStats[]`
Per-validator array, sorted by `missed` descending. Includes all bonded validators plus any jailed or unbonded validators that have accumulated missed events since tracking began - ensuring that validators who go offline do not disappear from the statistics table.

```json
{
  "operator":    "celestiavaloper1...",
  "moniker":     "alphab.ai",
  "avatar":      "https://...",
  "active":      true,
  "jailed":      false,
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
| `active` | Current staking status | `false` if jailed or unbonded |
| `jailed` | Current staking status | `true` if currently jailed |

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

Avatars are resolved from the current validator set on each cycle - not stored in the history file - keeping the history compact.

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

Jailed or unbonded validators with accumulated misses appear in the Proposal Statistics table with a **JAILED** / **INACTIVE** badge and reduced opacity, ensuring their historical data remains visible.

**Tech stack:** React 18 (UMD/Babel, no build step), Tailwind CSS via CDN - consistent with all other explorer pages.

---

## Limitations

- **Cold start** - Counts begin from the first collector run. There is no retroactive backfill from chain history.
- **Window dependency** - Missed events are only detected within the `UPTIME_BLOCKS` batch fetched each cycle (150 blocks, ~30 min). Blocks outside this window that had `commitRound > 0` are not captured.
- **Inference, not observation** - The tracker identifies *which* validator was scheduled to propose in each failed round. It cannot determine *why* the proposal failed (offline, timeout, proposal rejected). A validator that signed the committed block was online; one that did not sign may have been offline.
- **Validator set changes** - If the active validator set changes mid-simulation (e.g., immediately after a network upgrade or a large stake change), the sanity check may fail. The event is still recorded with a warning logged.

---

## Networks

| Network | Chain ID | Collector file | History file | Dashboard |
|---|---|---|---|---|
| Celestia Mainnet | `celestia` | `celestia-collector.js` | `/var/lib/celestia-collector/missed-proposals.json` | [cumulo.pro/services/celestia/missed-proposals](https://cumulo.pro/services/celestia/missed-proposals) |
| Celestia Mocha | `mocha-4` | `celestia-mocha-collector.js` | `/var/lib/celestia-mocha-collector/missed-proposals.json` | [cumulo.pro/services/celestia_mocha/missed-proposals](https://cumulo.pro/services/celestia_mocha/missed-proposals) |
