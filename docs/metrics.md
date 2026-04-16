# Metrics & Standards

This document defines every metric produced by the Cumulo Block Explorer: its formula, data window, the standard or convention it follows, and any caveats about interpretation.

---

## Table of Contents

- [Validator Metrics](#validator-metrics)
- [Network Performance Metrics](#network-performance-metrics)
- [Staking Economics](#staking-economics)
- [Governance Metrics](#governance-metrics)
- [Decentralization Indices](#decentralization-indices)
- [Data Windows and Freshness](#data-windows-and-freshness)
- [Precision and Denomination Handling](#precision-and-denomination-handling)

---

## Validator Metrics

### Uptime

**Definition:** Percentage of blocks in the observation window that were signed by the validator.

**Formula:**
```
uptime = signed_blocks / (signed_blocks + missed_blocks) × 100
```

**Window:** Last `UPTIME_BLOCKS` blocks (default: **150 blocks**, ~15 minutes at 6s block time).

**Signed block detection:** A block is considered signed by a validator if and only if the validator's consensus address appears in `block.last_commit.signatures` with `block_id_flag === 2` (COMMIT). Flags 1 (ABSENT) and 3 (NIL) are counted as missed.

**Important distinctions:**
- This is a **short-window** uptime metric, designed to surface recent signing behavior quickly. It is not equivalent to the chain's native `missed_blocks_counter` from the slashing module, which uses a different sliding window defined in the chain's slashing parameters.
- A validator can have 100% explorer uptime but have been jailed previously — jailing history is a separate field.
- Tombstoned validators are always displayed as 0% uptime regardless of recent signing.

**Industry context:** Most explorers (Mintscan, Ping.pub) display uptime over a 100–10,000 block window. The 150-block window is deliberately short to give validators and delegators a real-time health signal.

---

### Voting Power %

**Definition:** The fraction of total bonded stake controlled by a single validator.

**Formula:**
```
voting_power_pct = validator_tokens / bonded_pool_tokens × 100
```

**Data sources:**
- `validator_tokens` — from `/cosmos/staking/v1beta1/validators` (field: `tokens`)
- `bonded_pool_tokens` — from `/cosmos/staking/v1beta1/pool` (field: `bonded_tokens`)

**Note:** `tokens` and `bonded_tokens` are raw amounts in the base denomination (e.g., `uatom` for Cosmos Hub, 1 ATOM = 1,000,000 uatom). Division is performed in floating point after converting to integers, which is safe for percentage display but should not be used for on-chain computations.

---

### Commission Rate

**Definition:** The fraction of staking rewards a validator retains before distributing the remainder to delegators.

**Values tracked:**
| Field | Description |
|---|---|
| `commission_rate` | Current rate (e.g., 0.05 = 5%) |
| `max_rate` | Maximum rate the validator can ever set |
| `max_change_rate` | Maximum rate change per 24-hour period |

**Source:** `/cosmos/staking/v1beta1/validators` → `commission.commission_rates`

**Standard:** Defined by the Cosmos SDK `x/staking` module spec. Rates are stored as 18-decimal-precision strings and displayed as percentages rounded to 2 decimal places.

---

### Jailed Status

**Definition:** A validator is jailed when it violates a slashing condition and is temporarily removed from the active set.

**Sources:**
- `jailed: true/false` — from `/cosmos/staking/v1beta1/validators`
- `jailed_until` — from `/cosmos/slashing/v1beta1/signing_infos`; zero value (`1970-01-01`) means not jailed

**Slashing conditions** (Cosmos SDK standard):
- **Downtime slash:** Missing more than `signed_blocks_window × (1 - min_signed_per_window)` blocks in the slashing window → jail + small slash (default 0.01% on Cosmos Hub)
- **Double sign (equivocation):** Signing two conflicting blocks at the same height → tombstone + large slash (default 5% on Cosmos Hub)

**Tombstoned:** A permanently jailed validator. Cannot be unjailed. Displayed with a distinct indicator.

---

## Network Performance Metrics

### Average Block Time

**Definition:** The mean time between consecutive block productions over a recent window.

**Formula:**
```
avg_block_time = Σ(block[i].time - block[i-1].time) / (N-1)
```

**Window:** Last 100 blocks (configurable).

**Unit:** Seconds, displayed to 2 decimal places.

**Note:** Block time is not perfectly constant in Tendermint/CometBFT-based chains. It varies with validator connectivity, round changes, and consensus timeouts. The 100-block window smooths transient spikes while remaining responsive to real changes.

---

### Transactions Per Second (TPS)

**Definition:** Approximate transaction throughput, computed as an instantaneous estimate from the most recent block.

**Formula:**
```
tps = tx_count_in_last_block / avg_block_time
```

**Caveat:** TPS is highly variable. A block with zero transactions gives TPS = 0; a busy block spikes TPS upward. This metric should be read alongside the 24h transaction count for a more meaningful picture of network activity.

**Industry note:** There is no universal standard for blockchain TPS measurement. Some explorers report peak TPS, others report averages over varying windows. Our approach (per-block / avg block time) is transparent and reproducible from the raw data.

---

### Transactions (24h)

**Definition:** Total number of transactions included in blocks produced in the last 24 hours.

**Formula:**
```
tx_24h = Σ(tx_count) for all blocks in last (86400 / avg_block_time) blocks
```

At 6s/block this is approximately the last 14,400 blocks.

---

## Staking Economics

### Staking Ratio (Bonding Ratio)

**Definition:** The fraction of total token supply that is currently bonded (staked) with active validators.

**Formula:**
```
staking_ratio = bonded_tokens / total_supply × 100
```

**Significance:** A higher staking ratio generally indicates stronger economic security of the network, as more tokens are at risk of being slashed for misbehavior. Most Cosmos chains target a staking ratio between 50–67% through their inflation adjustment mechanism.

**Standard:** This metric is aligned with the Cosmos SDK `x/mint` module's concept of "bonded ratio" used internally to compute the current inflation rate.

---

### Inflation Rate

**Definition:** The annualized rate at which new tokens are being minted.

**Source:** `/cosmos/mint/v1beta1/inflation` (direct read, not computed).

**Context:** In Cosmos SDK chains using the standard mint module, inflation adjusts automatically:
- If staking ratio < target → inflation increases (to incentivize staking)
- If staking ratio > target → inflation decreases

The target and min/max bounds are chain-specific governance parameters.

---

### Community Pool

**Definition:** Tokens accumulated in the community pool from the distribution module's `community_tax`.

**Source:** `/cosmos/distribution/v1beta1/community_pool`

**Note:** Values are returned as decimal strings with 18-digit precision. The displayed value is rounded to the nearest whole token for readability.

---

## Governance Metrics

### Tally Percentages

**Definition:** The fraction of cast votes for each option.

**Formula:**
```
total_votes = yes + no + abstain + no_with_veto
pct_yes     = yes / total_votes × 100
pct_no      = no / total_votes × 100
pct_abstain = abstain / total_votes × 100
pct_veto    = no_with_veto / total_votes × 100
```

**Implementation note:** Token amounts are very large integers (up to 10^24 on some chains). All arithmetic uses JavaScript `BigInt` to avoid precision loss.

**Standard:** Cosmos SDK governance requires:
- `quorum` — minimum participation (default 33.4% of bonded tokens)
- `threshold` — YES must exceed 50% of non-ABSTAIN votes
- `veto_threshold` — NO_WITH_VETO must not exceed 33.4% of all votes

These thresholds are governance parameters and vary by chain.

---

### Turnout

**Definition:** Fraction of bonded tokens that have participated in voting.

**Formula:**
```
turnout = total_votes / bonded_tokens × 100
```

**Significance:** A proposal can reach quorum (33.4%) while representing a minority of token holders. Turnout contextualizes the tally.

---

## Decentralization Indices

All indices are on a **0–100 scale** where **100 = perfectly decentralized** and **0 = fully centralized** (one entity controls everything).

### Regional Index

**Formula:**
```
Regional Index = (unique_regions / total_validators) × 100
```

**Regions used:** Continental taxonomy — Europe, North America, South America, Asia, Africa, Oceania, Middle East.

**Interpretation:** A value of 50 means validators are spread across half the possible regions. A value of 100 means each validator is in a different region (maximum geographic spread relative to validator count).

---

### Country Index

**Formula:**
```
Country Index = (unique_countries / total_validators) × 100
```

---

### Hosting Provider Index

**Formula:**
```
Hosting Index = (unique_ASNs / total_validators) × 100
```

**ASN (Autonomous System Number)** identifies the network operator (hosting provider, ISP, data center). Validators sharing an ASN are vulnerable to the same provider outage.

**Significance:** A low hosting index indicates concentration in a few cloud/data center providers (e.g., Hetzner, OVH, AWS), which represents a correlated failure risk even if validators are geographically distributed.

---

### Voting Power Index (HHI-based)

**Definition:** Economic decentralization of the validator set, measured by the inverse of the Herfindahl-Hirschman Index (HHI).

**Formula:**
```
HHI = Σ ( validator_i_power / total_power )²

Voting Power Index = (1 - HHI) × 100
```

**The HHI** is the standard measure of market concentration used in antitrust economics. It ranges from near 0 (many equal competitors) to 1 (monopoly).

**Examples:**
| Scenario | HHI | Voting Power Index |
|---|---|---|
| 180 equal validators | 0.006 | 99.4 |
| Top 10 validators control 33% each | ~0.11 | ~89 |
| Single validator controls 51% | ~0.26 | ~74 |
| Single validator controls 100% | 1.0 | 0 |

**Industry context:** The HHI is used by the US Department of Justice to evaluate market concentration in merger reviews. Applying it to validator voting power is a methodologically rigorous way to quantify stake concentration beyond simple "top-N controls X%" statistics.

---

## Data Windows and Freshness

| Metric | Update Frequency | Data Window |
|---|---|---|
| Latest block | 6s (per chain block time) | Single block |
| Uptime | 6s | Last 150 blocks |
| Validator set | 6s | Real-time snapshot |
| Staking pool | 6s | Real-time snapshot |
| Inflation | 6s | Real-time snapshot |
| Governance tally | 30s | Real-time snapshot |
| Governance votes | 30s | All votes (paginated) |
| Consensus state | 2s | Real-time snapshot |
| Peer geolocation | 30s | Cached per IP |
| Keybase avatars | 24h | Per-identity cache |

---

## Precision and Denomination Handling

### Denomination Conversion

All Cosmos SDK chains store token amounts in a **base (micro) denomination**:

| Network | Base Denom | Display Denom | Factor |
|---|---|---|---|
| Cosmos Hub | uatom | ATOM | 10^6 |
| Celestia | utia | TIA | 10^6 |
| XRPL EVM | axrp | XRP | 10^6 |
| Story | atto | IP | 10^18 |

The collector stores raw base-denomination values in JSON and the frontend applies the conversion for display:

```javascript
const displayAmount = (rawAmount, decimals) =>
  (BigInt(rawAmount) / BigInt(10 ** decimals)).toLocaleString();
```

### Percentage Display

All percentages are rounded to 2 decimal places for display. Internal computations use full floating-point precision.

### BigInt Usage

Token amounts on large chains can exceed `Number.MAX_SAFE_INTEGER` (2^53). The collector uses JavaScript `BigInt` for:
- Governance tally vote counts
- Voting power comparisons
- Supply and pool totals

Frontend display converts to `Number` only after scaling down by the denomination factor.
