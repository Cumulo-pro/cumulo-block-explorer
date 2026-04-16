# Explorer Modules

Each supported network exposes the same seven modules. Data normalization happens in the collector layer, so every module works identically regardless of the underlying chain architecture.

---

## Table of Contents

- [Validators](#validators)
- [Blocks](#blocks)
- [Uptime](#uptime)
- [Governance](#governance)
- [Stats](#stats)
- [Consensus](#consensus)
- [Decentralization](#decentralization)

---

## Validators

**Purpose** — Complete, real-time view of the validator set for operators, delegators, and researchers.

### Data Displayed

| Field | Source | Notes |
|---|---|---|
| Moniker | Cosmos REST staking API | Validator self-reported name |
| Rank | Computed | Ordered by voting power descending |
| Operator Address | Cosmos REST staking API | Bech32-encoded (`<prefix>valoper1...`) |
| Voting Power | Cosmos REST staking API | Tokens delegated, in base denomination |
| Voting Power % | Computed | Validator tokens / bonded pool × 100 |
| Commission | Cosmos REST staking API | Current rate + max rate + max change rate |
| Status | Cosmos REST staking API | BONDED / UNBONDING / UNBONDED |
| Uptime | Computed (collector) | Signed blocks / total blocks, last 150 blocks |
| Jailed | Cosmos REST slashing API | True if validator has been jailed |
| Tombstoned | Cosmos REST slashing API | True if permanently slashed |
| Identity | Cosmos REST staking API | Keybase identity hash (16 hex chars) |
| Avatar | Keybase API | Resolved from identity hash, cached 24h |

### Frontend Features

- **Sortable table** — click any column header to sort ascending/descending
- **Filter** — live search by moniker or operator address
- **Status badges** — color-coded BONDED / UNBONDING / UNBONDED / JAILED
- **Keybase avatars** — validator logos resolved from on-chain identity hashes
- **Detail panel** — clicking a validator row opens an expanded card with full metrics
- **Real-time updates** — table refreshes every 6 seconds without page reload

### Data Flow

```
/cosmos/staking/v1beta1/validators  →  voting power, commission, status
/cosmos/slashing/v1beta1/signing_infos  →  jailed, tombstoned, signed blocks
/block?height=  (last 150 blocks)  →  uptime computation (see Uptime module)
Keybase /user/lookup.json  →  avatar URL (24h cache)
```

---

## Blocks

**Purpose** — Live stream of produced blocks for monitoring network activity and confirming transaction inclusion.

### Data Displayed

| Field | Source |
|---|---|
| Height | CometBFT `/block` |
| Block Hash | CometBFT `/block` |
| Timestamp (UTC) | CometBFT `/block` |
| Proposer | CometBFT `/block` → resolved to moniker |
| Transaction Count | CometBFT `/block` |
| Gas Used | CometBFT `/block` |
| Block Time (delta) | Computed from consecutive timestamps |

### Frontend Features

- **Live block cards** — new cards animate in at the top as blocks are produced
- **Block detail overlay** — click any card to see full block metadata and transaction list
- **Transaction list** — raw transaction hashes with type and gas details
- **Block time indicator** — running average block time display
- **Pagination** — "Load more" button to inspect historical blocks

### Data Flow

```
CometBFT /status  →  latest height
CometBFT /block?height=N  →  full block data
CometBFT /block?height=N-1, N-2, ...  →  recent block window (collector cache)
```

---

## Uptime

**Purpose** — Block-level signing history for every validator in the set, allowing operators and delegators to assess validator reliability.

### Data Displayed

| Field | Description |
|---|---|
| Validator | Moniker + avatar |
| Uptime % | Percentage of last N blocks signed |
| Signed | Count of signed blocks in window |
| Missed | Count of missed blocks in window |
| Block strip | Color-coded bar: green = signed, red = missed |
| Jailed | Red badge if currently jailed |
| Status | BONDED / UNBONDING / UNBONDED tag |

### Uptime Calculation

For each of the last `UPTIME_BLOCKS` (default: 150) blocks, the collector checks whether the validator's consensus address appears in `last_commit.signatures` with `block_id_flag === 2` (signed). Any other value (nil, absent) is counted as missed.

```javascript
// Pseudocode
for (const block of recentBlocks) {
  const sig = block.last_commit.signatures.find(
    s => s.validator_address === consensusAddr
  );
  if (sig?.block_id_flag === 2) {
    signed++;
  } else {
    missed++;
  }
  blockHistory.push({ height: block.height, signed: sig?.block_id_flag === 2 });
}
uptime = signed / (signed + missed) * 100;
```

### Window Size

The default window is **150 blocks**, which at a 6-second block time corresponds to approximately 15 minutes of history. This provides a responsive signal without being overly noisy. The window is configurable per chain via `UPTIME_BLOCKS` in the collector config.

### Frontend Features

- **Sortable** — by uptime percentage or missed block count
- **Block strip** — visual bar of the last N blocks; hover shows block height and signed/missed status
- **Color scale** — uptime above 99% is green, 95–99% is yellow, below 95% is red
- **Real-time updates** — refreshes as new blocks arrive

---

## Governance

**Purpose** — Complete governance proposal tracker covering the full proposal lifecycle from deposit through voting to final result.

### Proposal Lifecycle

```
DEPOSIT_PERIOD  →  VOTING_PERIOD  →  PASSED / REJECTED / FAILED
```

Each phase is tracked with timestamps, thresholds, and participation metrics.

### Data Displayed

| Field | Source |
|---|---|
| Proposal ID | Cosmos REST governance API |
| Title | Cosmos REST governance API |
| Description | Cosmos REST governance API |
| Status | Cosmos REST governance API |
| Deposit Amount | Cosmos REST governance API |
| Deposit Threshold | Cosmos REST governance API (params) |
| Voting Start / End | Cosmos REST governance API |
| YES votes | Cosmos REST governance API (tally) |
| NO votes | Cosmos REST governance API (tally) |
| ABSTAIN votes | Cosmos REST governance API (tally) |
| NO\_WITH\_VETO votes | Cosmos REST governance API (tally) |
| Tally percentages | Computed |
| Voter list | Cosmos REST governance API (paginated) |

### Tally Computation

```javascript
const total = BigInt(yes) + BigInt(no) + BigInt(abstain) + BigInt(veto);
const pctYes     = total > 0n ? Number(yes * 10000n / total) / 100 : 0;
const pctNo      = total > 0n ? Number(no * 10000n / total) / 100 : 0;
const pctAbstain = total > 0n ? Number(abstain * 10000n / total) / 100 : 0;
const pctVeto    = total > 0n ? Number(veto * 10000n / total) / 100 : 0;
```

BigInt arithmetic is used to avoid precision loss on large token amounts.

### API Compatibility

The collector supports both governance API versions:

| API Version | Route | Used By |
|---|---|---|
| v1beta1 | `/cosmos/gov/v1beta1/proposals` | Older chains (Cosmos Hub, some testnets) |
| v1 | `/cosmos/gov/v1/proposals` | Modern chains (Celestia, Story, etc.) |

The collector attempts v1 first and falls back to v1beta1 automatically.

### Frontend Features

- **Proposal cards** — each proposal is a card with status badge, title, and quick tally summary
- **Live tally bar** — horizontal stacked bar showing YES/NO/ABSTAIN/VETO percentages
- **Voter list** — expandable list of voter addresses and their vote option
- **Timeline** — visual timeline showing deposit end and voting end dates
- **Quorum indicator** — percentage of bonded tokens that have voted

---

## Stats

**Purpose** — Key network health metrics in a single dashboard for monitoring staking economics and network performance.

### Metrics Displayed

**Staking Economics**

| Metric | Computation |
|---|---|
| Bonded Tokens | From `/cosmos/staking/v1beta1/pool` |
| Total Supply | From `/cosmos/bank/v1beta1/supply` |
| Staking Ratio | Bonded / Total Supply × 100 |
| Inflation Rate | From `/cosmos/mint/v1beta1/inflation` |
| Community Pool | From `/cosmos/distribution/v1beta1/community_pool` |

**Network Performance**

| Metric | Computation |
|---|---|
| Latest Height | From CometBFT `/status` |
| Average Block Time | Mean of last 100 block intervals |
| Transactions (24h) | Sum of tx count in last 14,400 blocks (at 6s/block) |
| TPS | Txs in last block / avg block time |

**Validator Set**

| Metric | Computation |
|---|---|
| Active Validators | Count of BONDED validators |
| Inactive Validators | Count of UNBONDING + UNBONDED |
| Jailed Validators | Count of jailed validators |
| Average Commission | Mean of all active validator commission rates |

### Frontend Features

- **KPI cards** — large numeric display with metric label and unit
- **Trend indicators** — up/down arrows with percentage change vs. previous cycle
- **Real-time updates** — all figures refresh every 6 seconds

---

## Consensus

**Purpose** — Live view of the CometBFT consensus state machine, showing which validators have voted at each round step.

### Data Displayed

| Field | Description |
|---|---|
| Height | Current consensus height |
| Round | Current round within height |
| Step | NewHeight / Propose / Prevote / Precommit / Commit |
| Valid Round | Round in which the locked value was determined |
| Commit Round | Round that produced the most recent commit |
| Prevotes | Set of validators that have sent a prevote |
| Precommits | Set of validators that have sent a precommit |
| Participation % | Precommit weight / total bonded power |

### Data Source

CometBFT exposes the live consensus state at `/consensus_state` (RPC). The collector fetches this every 2 seconds — faster than block production — and writes `consensus.json`.

### Use Cases

- Monitor consensus liveness during network upgrades or incidents
- Identify validators that are lagging in the consensus process
- Observe round changes that indicate leader failures or network partitions

---

## Decentralization

**Purpose** — Quantitative analysis of validator set decentralization across geographic, hosting, and economic dimensions.

### Indices

Four independent decentralization indices are computed, all on a 0–100 scale where **100 = perfectly decentralized**:

#### Regional Index
```
Regional Index = (Unique regions / Total validators) × 100
```
Regions follow the standard continent taxonomy (Europe, North America, Asia, etc.).

#### Country Index
```
Country Index = (Unique countries / Total validators) × 100
```

#### Hosting Index
```
Hosting Index = (Unique ASNs (hosting providers) / Total validators) × 100
```
A high hosting index means validators are spread across many independent data centers rather than concentrated in a few providers (e.g., AWS, Hetzner).

#### Voting Power Index (HHI-based)
```
HHI = Σ (validator_power / total_power)²
Voting Power Index = (1 − HHI) × 100
```
The [Herfindahl-Hirschman Index](https://en.wikipedia.org/wiki/Herfindahl%E2%80%93Hirschman_index) is the standard economic measure of market concentration. A perfect HHI of 0 (all validators equal) gives a Voting Power Index of 100. A single validator controlling all stake gives an index of 0.

### Data Sources

- Validator IP addresses are retrieved from CometBFT `/net_info` (peer list)
- Each IP is geolocated using [ip-api.com](http://ip-api.com) (free tier: 45 req/min)
- Results are cached for 30 seconds to avoid rate limiting
- Pre-configured `validators_geo.json` overrides for validators with static IPs

### Frontend Features

- **World map** — interactive Leaflet.js map with one marker per validator node
- **Index gauges** — four colored gauges (0–100) for each decentralization dimension
- **Provider breakdown** — table of hosting providers with validator count and power share
- **Country breakdown** — table of countries with validator count and map highlighting
