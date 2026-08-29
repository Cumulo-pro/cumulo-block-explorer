# Celestia : Missed Proposals Tracker

> **Network-specific feature**. This module goes beyond standard Cosmos SDK metrics. It applies the CometBFT weighted round-robin proposer selection algorithm to determine, with block-level precision, which validator missed their proposal turn at each consensus round.

---

## The Problem with Generic Uptime Metrics

Standard Cosmos block explorers track validator signing participation via the `missed_blocks_counter` from the slashing module. This tells you whether a validator **signed** a block, but not whether it **proposed** one when it was its turn.

Block proposal failures are a distinct failure mode:

- A validator can have 100% signing uptime and still repeatedly miss proposal slots
- Missed proposals add latency to block finalization (each skipped round ≈ +2–3 seconds)
- Without identifying *which* validator missed, operators have no actionable signal

Standard metrics provide no visibility into this. The Missed Proposals tracker closes that gap.

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

The algorithm requires `BigInt` arithmetic throughout. Celestia voting power values exceed JavaScript's safe integer limit (`Number.MAX_SAFE_INTEGER`).

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

---

## What "Missed" Actually Means

A missed proposal is an **inference**, not a direct on-chain observation. The blockchain records what *did* happen, never what *didn't*.

What is directly observable:
- A block was committed at round `N > 0` - **on-chain fact**
- The actual proposer (round `N`) - **on-chain fact**
- The validator scheduled for round `0..N-1` by the deterministic algorithm - **mathematically computable from public data**

What is inferred:
- The scheduled validator did not successfully propose in their round  **inference** (no explicit on-chain record)

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

### Extended catch-up scan

Detection normally piggybacks on the block data already fetched for the uptime grid (`UPTIME_BLOCKS`: 150 blocks on mainnet, 300 on mocha, roughly the last 30 minutes). That window alone is too short to survive a restart or a brief upstream gap without missing something, so on every cycle the collector also computes:

```
missedFromH = max(lastProcessedHeight + 1, latestHeight - MISSED_SCAN_CAP)
```

If `missedFromH` falls before the uptime window, it fetches the extra blocks in between (same batched `/block?height=H` calls, in groups of 20) purely for `proposer` + `commitRound`, without the signature analysis the uptime grid needs. `MISSED_SCAN_CAP` bounds how far back a single cycle will reach - 3,000 blocks (~10h) on mainnet, 5,000 (~8h) on mocha - so a collector that was down longer than that will resume detection from the cap boundary rather than trying to scan the entire gap in one cycle.

### Gap-Safe Height Tracking

**Fixed 2026-08-29.** The batched block fetches above use `Promise.allSettled`, and a single rejected promise (a timeout, a transient 5xx) was silently dropped from the result set. Because `lastProcessedHeight` used to advance to `latestHeight` unconditionally every cycle regardless of which heights actually got fetched, a height that failed to fetch was never retried - if that exact height happened to be a real missed proposal, it was lost permanently, with no error surfaced anywhere.

The collector now tracks which heights it actually has data for and finds the first gap in the scanned range before advancing:

```js
const presentHeights = new Set(blockData.map(b => b.h));
let scanGapAt = null;
for (let h = missedFromH; h <= latestH; h++) {
  if (!presentHeights.has(h)) { scanGapAt = h; break; }
}
history.lastProcessedHeight = scanGapAt !== null ? scanGapAt - 1 : latestH;
```

If a gap is found, `lastProcessedHeight` stops just before it instead of jumping past it, so the next cycle (12s later on mainnet) naturally retries the missing height. On the healthy path (no gap) this is a no-op - `lastProcessedHeight` still advances to `latestHeight` exactly as before.

### Early Latest-Block Detection

**Added 2026-08-29**, to close most of the gap with tools that report missed proposals within a second or two of the block landing. The header-based loop above can only ever confirm heights up to `latestHeight - 1` in a given cycle, since it needs `latestHeight`'s own header to read `commitRound` for the block before it - a miss at the current tip is structurally invisible until the *next* block exists, adding a full extra block-period of latency on top of the poll interval.

`/commit?height=H` returns the canonical commit round for that exact height directly, with no dependency on the following block - it's the same authoritative endpoint already used a few lines below as the false-positive cross-check (`fetchCommitRound`). Each cycle now also calls it once for the current tip:

```js
const latestBlockEntry = sortedBD[sortedBD.length - 1]; // == latestH
if (latestBlockEntry && latestBlockEntry.h === latestH) {
  const latestRound = await fetchCommitRound(latestH);
  if (latestRound > 0) {
    missedBlocks.push({ cur: { commitRound: latestRound }, prev: latestBlockEntry });
  }
}
```

This is purely additive - the pushed candidate flows through the exact same verification/simulation pipeline as every other candidate below. It can never double-count (the header-based loop structurally cannot confirm `latestHeight` in the same cycle, so the two paths never target the same height at once) and fails safe (if the extra call errors or finds nothing, the header-based loop still catches it one cycle later, exactly as it did before this change).

**Effect:** combined with lowering the frontend's own poll interval (see [Frontend Page](#frontend-page)), worst-case end-to-end latency (chain event → visible on the page) drops from roughly one poll interval plus one full block period plus the old 6s frontend poll (~30s on mainnet, ~18s on mocha) to roughly one poll interval plus the new 2.5s frontend poll (~14.5s on mainnet, ~8.5s on mocha) - about half. These are worst-case estimates derived from the code's timing, not a stopwatch measurement of a live event. Going further (near-instant, WebSocket-driven detection matching a live-push model) was evaluated and deliberately deferred - it would require a persistent connection with its own reconnect logic, a concurrency guard against the existing interval loop, and a coordinated partial-write path into `data.json`, all of which meaningfully increase the surface area of a collector that today has none of those failure modes.

### RPC/API Automatic Fallback

**Added 2026-08-29**, after a Cumulo-node RPC pause caused a stall in data collection. Every RPC and API call (`rpc(path)` / `cosmos(path)`) is wrapped by `makeFallbackFetcher()`, backed by a short candidate list per endpoint type. Candidate `0` is always Cumulo's own node; the rest are third-party public endpoints, hand-verified (`/status` and `/cosmos/base/tendermint/v1beta1/node_info`, checking the `network` field) from the community-curated list at [`Cumulo-Front-Chain/Celestia/data`](https://github.com/Cumulo-pro/Cumulo-Front-Chain/tree/main/Celestia/data):

| Network | Fallback candidates (in order) |
|---|---|
| Mainnet | Polkachu, kjnodes, ITRocket, NodeStake, Validatus |
| Mocha-5 | Polkachu, ITRocket, NodesGuru, NodeStake |

Behavior:
- While the active candidate responds, there is no extra cost - the call is a single request, exactly as before this change.
- On failure, it walks forward through the remaining candidates (and, if none of those work either, back through the earlier ones) until one succeeds, and "sticks" with that candidate for subsequent calls.
- Every 30 calls made against a non-primary candidate, it retries Cumulo's own node once; if it responds, it switches back immediately.
- A `[fallback]` line is logged (visible via `journalctl`) every time the active candidate changes, in either direction - this is the first thing to check when a gap or an odd cluster of events shows up in the history around a specific time.

This does not by itself guarantee zero data loss - if every candidate is unreachable at once, calls still fail - but combined with [Gap-Safe Height Tracking](#gap-safe-height-tracking) above, a single node's outage (the common case) no longer stalls or corrupts collection.

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

### Backups

`saveMissedHistory()` also writes:
- A daily backup (`missed-proposals.json.bak`), overwritten once per day
- A permanent monthly backup (`missed-proposals.json.bak.YYYY-MM`), never overwritten

The monthly backup's "never overwritten" guarantee is enforced by checking `existsSync(monthFile)` on disk before writing it, not an in-memory flag - a systemd restart mid-month (routine during deploys) does not re-trigger and silently destroy that month's snapshot.

A companion manifest file, `missed-proposals-backups.json`, indexes every monthly backup present on disk (`{ months: [{ month, file, sizeBytes, createdAt }] }`) so the frontend can list and offer them for download without directory listing or guessing filenames. On first startup after this feature was added, `backfillMissedBackupsManifest()` scans `OUTPUT_DIR` once for any pre-existing `.bak.YYYY-MM` files and registers them retroactively, so backups written before the manifest existed are not orphaned.

Both live on the same disk as the primary file, so they protect against corruption or accidental overwrites but not against total loss of the output directory. Because the full history file and the manifest are served publicly (same static-file location as `data.json`), they can also be pulled from an independent location on a schedule for off-server redundancy:

```bash
curl -s "https://celestia.explorer.cumulo.org.es/missed-proposals.json" \
  -o "missed-proposals-$(date +%Y%m%d).json"
```

### Manual backfill (ops tool)

`merge-missed-backfill.mjs`, kept alongside the mainnet collector, is a one-off recovery script for the rare case where a specific height is confirmed (via `/commit?height=H`) to be a real missed proposal but is absent from the history - for example after the gap described in [Gap-Safe Height Tracking](#gap-safe-height-tracking) above was found and fixed retroactively. It re-derives the event with the exact same algorithm (`cometBFTRoundProposers` + validator-set + moniker resolution), then merges it into the live file by height (skipping any height already present) and takes a timestamped backup of the file before writing. It is not run automatically - it is a manual last resort, invoked only after independently confirming the gap against `/commit`.

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

**Data source:** `data.json` via `DATA_URL`, polled every 2.5 seconds. Lowered from 6s on 2026-08-29, once the collector-side detection latency was cut (see [Early Latest-Block Detection](#early-latest-block-detection)) - the page's own poll interval had become the largest remaining piece of end-to-end latency. Client-side only; the underlying data can never change faster than the collector's own write cadence (12s mainnet, 6s mocha), so this is a reasonable floor, not pushed lower.

**Tabs:** the page is split into **Live** and **Archive & Stats**, so the monthly-backups list and the aggregate charts below don't compete for space with the live event feed at the top of the page.

**Key UI sections (Live tab):**

| Section | Data source | Description |
|---|---|---|
| Status bar | `meta` | Chain ID, latest block, tracking start block, network health % |
| KPI strip | `proposalStats` aggregated | Total missed, network miss rate, health badge, blocks tracked |
| Warning banner | `proposalStats[0]` | Shown only when at least one validator has ≥ 1 real miss |
| Recent Missed Events | `missedPropEvents` | Block, time, round, who missed (with avatars), actual proposer, tx count |
| Proposal Statistics | `proposalStats` | Full per-validator table: proposed, missed, miss rate, vote share |

**Key UI sections (Archive & Stats tab):**

| Section | Data source | Description |
|---|---|---|
| Health Gauge | `proposalStats` aggregated | Circular gauge of overall network proposal-success rate |
| Top Missed Chart | `proposalStats` | Horizontal bar chart, top 10 validators by missed count |
| Miss Rate Distribution | `proposalStats` | Histogram of validators bucketed by miss-rate severity |
| Monthly Archives | `missed-proposals-backups.json` | One card per permanent monthly backup, with size and download link |

**Export:** both the live event history and the aggregate per-validator stats can be exported as JSON or CSV (`Export JSON` / `Export CSV` buttons), client-side, from whatever is currently loaded - no extra collector endpoint involved.

Jailed or unbonded validators with accumulated misses appear in the Proposal Statistics table with a **JAILED** / **INACTIVE** badge and reduced opacity, ensuring their historical data remains visible.

**Tech stack:** React 18 (UMD/Babel, no build step), Tailwind CSS via CDN - consistent with all other explorer pages.

---

## Limitations

- **Cold start** - Counts begin from the first collector run. There is no automatic retroactive backfill from chain history (a manual one-off tool exists for confirmed gaps, see [Manual backfill](#manual-backfill-ops-tool)).
- **Bounded catch-up window** - The [extended catch-up scan](#extended-catch-up-scan) reaches back up to `MISSED_SCAN_CAP` blocks (~10h mainnet, ~8h mocha) past `lastProcessedHeight`. A collector down longer than that resumes from the cap boundary, not from where it left off - anything older is not retroactively scanned.
- **Fallback is not infinite redundancy** - the [RPC/API fallback](#rpcapi-automatic-fallback) protects against one endpoint (including Cumulo's own) being down, not against every candidate being down simultaneously.
- **Inference, not observation** - The tracker identifies *which* validator was scheduled to propose in each failed round. It cannot determine *why* the proposal failed (offline, timeout, proposal rejected). A validator that signed the committed block was online; one that did not sign may have been offline.
- **Validator set changes** - If the active validator set changes mid-simulation (e.g., immediately after a network upgrade or a large stake change), the sanity check may fail. The event is still recorded with a warning logged.

---

## Networks

| Network | Chain ID | Collector file | History file | Dashboard |
|---|---|---|---|---|
| Celestia Mainnet | `celestia` | `celestia-collector.js` | `/var/lib/celestia-collector/missed-proposals.json` | [cumulo.pro/services/celestia/missed-proposals](https://cumulo.pro/services/celestia/missed-proposals) |
| Celestia Mocha | `mocha-5` | `celestia-mocha-collector.js` | `/var/lib/celestia-mocha-collector/missed-proposals.json` | [cumulo.pro/services/celestia_mocha/missed-proposals](https://cumulo.pro/services/celestia_mocha/missed-proposals) |

---

## References

| Source | Description |
|---|---|
| [CometBFT - Proposer Selection Spec](https://github.com/cometbft/cometbft/blob/main/spec/consensus/proposer-selection.md) | Official specification of the weighted round-robin proposer selection algorithm (`IncrementProposerPriority`) |
| [CometBFT - validator_set.go](https://github.com/cometbft/cometbft/blob/main/types/validator_set.go) | Reference implementation of `IncrementProposerPriority` in Go |
| [CometBFT RPC - `/commit`](https://docs.cometbft.com/v0.38/rpc/#/Info/commit) | Authoritative endpoint for the canonical consensus round of a committed block |
| [CometBFT RPC - `/validators`](https://docs.cometbft.com/v0.38/rpc/#/Info/validators) | Returns the validator set with `proposer_priority` at a given height |
| [Celestia Docs - Consensus](https://docs.celestia.org/learn/how-celestia-works/consensus) | Overview of how Celestia uses CometBFT consensus |
| [Cumulo Medium - Beyond Uptime](https://medium.com/cumulo-pro/beyond-uptime-how-celestia-validators-performance-metrics-changed-after-matcha-v6-9cbbcbdcfec1) | Context on why proposal metrics matter specifically for Celestia after Matcha v6 |
