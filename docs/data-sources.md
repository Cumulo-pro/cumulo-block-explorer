# Data Sources

This document catalogs every external data source used by the Cumulo Block Explorer collectors: the API endpoints, their protocols, the data extracted from each, and any normalization applied.

---

## Table of Contents

- [CometBFT RPC](#cometbft-rpc)
- [Cosmos REST API](#cosmos-rest-api)
- [Keybase API](#keybase-api)
- [IP Geolocation API](#ip-geolocation-api)
- [Substrate JSON-RPC (Avail)](#substrate-json-rpc-avail)
- [EVM-Adapted APIs (Story)](#evm-adapted-apis-story)
- [Data Normalization](#data-normalization)

---

## CometBFT RPC

**Default port:** 26657  
**Protocol:** HTTP JSON-RPC  
**Authentication:** None (public endpoint)

CometBFT (formerly Tendermint) is the consensus engine used by all Cosmos SDK chains. Its RPC interface is the primary source of block and consensus data.

### Endpoints Used

#### `/status`

Returns the node's current state including the latest block height and chain ID.

**Used for:** Latest height, chain ID, node moniker  
**Fetch frequency:** Every cycle (6s default)

```json
{
  "result": {
    "node_info": { "network": "cosmoshub-4", "moniker": "my-node" },
    "sync_info": {
      "latest_block_height": "21504821",
      "latest_block_time": "2024-11-20T14:32:10.123Z",
      "catching_up": false
    }
  }
}
```

#### `/block?height=<N>`

Returns the full block at height N. If height is omitted, returns the latest block.

**Used for:** Block hash, timestamp, proposer address, transaction list, previous commit signatures  
**Fetch frequency:** Every cycle; last 150 blocks fetched in parallel for uptime calculation

```json
{
  "result": {
    "block": {
      "header": {
        "height": "21504821",
        "time": "2024-11-20T14:32:10.123Z",
        "proposer_address": "ABE3D9F8C14B..."
      },
      "data": {
        "txs": ["base64tx1", "base64tx2"]
      },
      "last_commit": {
        "height": "21504820",
        "signatures": [
          { "block_id_flag": 2, "validator_address": "ABE3D9F8...", "timestamp": "..." },
          { "block_id_flag": 1, "validator_address": "C29A1B45...", "timestamp": "..." }
        ]
      }
    }
  }
}
```

`block_id_flag` values:
- `1` — ABSENT (did not vote)
- `2` — COMMIT (voted and signed)
- `3` — NIL (voted nil)

#### `/validators?height=<N>&per_page=100&page=<P>`

Returns the validator set at a given height with consensus public keys and voting power.

**Used for:** Consensus address mapping, voting power at block level  
**Fetch frequency:** Every cycle

```json
{
  "result": {
    "validators": [
      {
        "address": "ABE3D9F8C14B...",
        "pub_key": { "type": "tendermint/PubKeyEd25519", "value": "base64pubkey" },
        "voting_power": "1234567",
        "proposer_priority": "0"
      }
    ]
  }
}
```

#### `/net_info`

Returns the list of connected peers including IP addresses.

**Used for:** Decentralization module — peer IP geolocation  
**Fetch frequency:** Every 30s

```json
{
  "result": {
    "peers": [
      {
        "node_info": { "id": "abc123...", "moniker": "peer-node" },
        "remote_ip": "1.2.3.4"
      }
    ]
  }
}
```

#### `/consensus_state`

Returns the live consensus round state machine.

**Used for:** Consensus module  
**Fetch frequency:** Every 2s

```json
{
  "result": {
    "round_state": {
      "height": "21504822",
      "round": 0,
      "step": 6,
      "valid_round": -1,
      "locked_round": -1,
      "votes": [...]
    }
  }
}
```

---

## Cosmos REST API

**Default port:** 1317  
**Protocol:** HTTP REST (gRPC-gateway)  
**Authentication:** None (public endpoint)  
**Spec:** [Cosmos SDK Swagger](https://v1.cosmos.network/rpc)

The Cosmos REST API (also called the LCD — Light Client Daemon) exposes staking, governance, distribution, and bank data via gRPC-gateway REST endpoints.

### Endpoints Used

#### `/cosmos/staking/v1beta1/validators`

Returns all validators with staking parameters. Paginated (`pagination.limit`, `pagination.key`).

**Used for:** Validator moniker, operator address, commission, status, tokens, delegator shares, description, identity

```json
{
  "validators": [
    {
      "operator_address": "cosmosvaloper1...",
      "consensus_pubkey": { "@type": "/cosmos.crypto.ed25519.PubKey", "key": "base64..." },
      "jailed": false,
      "status": "BOND_STATUS_BONDED",
      "tokens": "12345678901234",
      "delegator_shares": "12345678901234.000000000000000000",
      "description": {
        "moniker": "Cumulo",
        "identity": "45A2E210C9F6EFEC",
        "website": "https://cumulo.com.es",
        "details": "Professional PoS validator"
      },
      "commission": {
        "commission_rates": {
          "rate": "0.050000000000000000",
          "max_rate": "0.200000000000000000",
          "max_change_rate": "0.010000000000000000"
        }
      }
    }
  ]
}
```

#### `/cosmos/slashing/v1beta1/signing_infos`

Returns slashing parameters for all validators.

**Used for:** Jailing status, tombstoning, missed block count (chain-native counter)

```json
{
  "info": [
    {
      "address": "cosmosvalcons1...",
      "validator_signing_info": {
        "address": "cosmosvalcons1...",
        "start_height": "0",
        "index_offset": "21504821",
        "jailed_until": "1970-01-01T00:00:00Z",
        "tombstoned": false,
        "missed_blocks_counter": "3"
      }
    }
  ]
}
```

#### `/cosmos/staking/v1beta1/pool`

Returns the bonded and not-bonded token pools.

**Used for:** Staking ratio, voting power normalization

```json
{
  "pool": {
    "not_bonded_tokens": "500000000000",
    "bonded_tokens": "4500000000000"
  }
}
```

#### `/cosmos/bank/v1beta1/supply`

Returns the total token supply per denomination.

**Used for:** Circulating supply, staking ratio

```json
{
  "supply": [
    { "denom": "uatom", "amount": "1159712345678" }
  ]
}
```

#### `/cosmos/mint/v1beta1/inflation`

Returns the current inflation rate.

**Used for:** Stats module inflation display

```json
{
  "inflation": "0.082000000000000000"
}
```

#### `/cosmos/distribution/v1beta1/community_pool`

Returns the accumulated community pool balance.

**Used for:** Stats module community pool display

```json
{
  "pool": [
    { "denom": "uatom", "amount": "12345678901.234567890000000000" }
  ]
}
```

#### `/cosmos/gov/v1/proposals` (or `/cosmos/gov/v1beta1/proposals`)

Returns governance proposals. The collector tries v1 first, falls back to v1beta1.

**Used for:** Governance module — proposal list, status, tally

#### `/cosmos/gov/v1/proposals/<id>/tally`

Returns the live tally for an active proposal.

**Used for:** Real-time YES/NO/ABSTAIN/VETO counts

#### `/cosmos/gov/v1/proposals/<id>/votes`

Returns individual votes for a proposal. Paginated.

**Used for:** Voter address list and vote option per proposal

---

## Keybase API

**Base URL:** `https://keybase.io/_/api/1.0`  
**Protocol:** HTTP REST  
**Authentication:** None  
**Rate limit:** ~10 req/s (unofficial)

Keybase is used to resolve validator identity hashes to profile pictures and names.

### Endpoint Used

#### `/user/lookup.json?key_suffix=<identity>&fields=pictures`

```json
{
  "them": [
    {
      "pictures": {
        "primary": {
          "url": "https://s3.amazonaws.com/keybase_processed_uploads/..."
        }
      }
    }
  ]
}
```

**Fetch strategy:**
- Resolved once per validator identity
- Cached in memory for 24 hours
- Requests batched in groups of 5 (parallel) to respect rate limits
- Validators without a Keybase identity display a generated avatar

---

## IP Geolocation API

**Base URL:** `http://ip-api.com/json`  
**Protocol:** HTTP REST  
**Authentication:** None  
**Rate limit:** 45 requests/minute (free tier)

Used by the Decentralization module to map validator peer IPs to geographic locations.

### Endpoint Used

#### `/json/<ip>?fields=country,regionName,city,lat,lon,org,as`

```json
{
  "country": "Germany",
  "regionName": "Bavaria",
  "city": "Munich",
  "lat": 48.1374,
  "lon": 11.5755,
  "org": "AS24940 Hetzner Online GmbH",
  "as": "AS24940 Hetzner Online GmbH"
}
```

**Fetch strategy:**
- Results cached for 30 seconds per IP
- Requests throttled to stay within 45 req/min limit
- Failed geolocations fall back to known static mappings in `validators_geo.json`

---

## Substrate JSON-RPC (Avail)

**Default port:** 9944  
**Protocol:** HTTP + WebSocket JSON-RPC (same port)  
**Encoding:** SCALE codec

Avail uses the Substrate framework, which has a fundamentally different API surface from Cosmos SDK. The collector implements a SCALE codec decoder to handle binary-encoded responses.

### Methods Used

| Method | Returns | Used For |
|---|---|---|
| `chain_getHeader` | Latest block header | Block height, parent hash |
| `chain_getBlockHash` | Block hash at height N | Block retrieval |
| `chain_getBlock` | Full block with extrinsics | Block data, tx count |
| `session_validators` | Vec\<AccountId32\> | Current session validator set |
| `grandpa_authorities` | Grandpa authority set | Finality validator set |
| `system_peers` | Connected peers | Decentralization data |

### SCALE Decoding

Substrate encodes all data in [SCALE codec](https://docs.substrate.io/reference/scale-codec/) — a compact binary format. The collector implements decoders for the types used:

```javascript
// Vec<AccountId32>: 4-byte compact length + 32-byte account IDs
function decodeVecAccountId32(hexStr) {
  const bytes = Buffer.from(hexStr.slice(2), 'hex');
  const count = bytes.readUInt32LE(0);
  const accounts = [];
  for (let i = 0; i < count; i++) {
    accounts.push(bytes.slice(4 + i * 32, 4 + (i + 1) * 32).toString('hex'));
  }
  return accounts;
}
```

---

## EVM-Adapted APIs (Story)

Story Protocol uses a Cosmos-compatible API but with EVM-specific field formats. The collector handles these differences explicitly.

### Field Differences

| Field | Standard Cosmos | Story (EVM-adapted) |
|---|---|---|
| `operator_address` | `cosmosvaloper1...` (bech32) | `0x...` (EVM hex) |
| `consensus_pubkey.@type` | `ed25519.PubKey` | `secp256k1.PubKey` |
| `status` | `"BOND_STATUS_BONDED"` (string) | `3` (integer: 1=unbonded, 2=unbonding, 3=bonded) |
| API wrapper | Direct JSON | `{ "code": 200, "msg": {...}, "error": "" }` |
| API routes | `/cosmos/staking/v1beta1/validators` | `/staking/validators` |

### Status Normalization

```javascript
// Story uses numeric status codes
const STATUS_MAP = { 1: 'UNBONDED', 2: 'UNBONDING', 3: 'BONDED' };
const normalizedStatus = typeof validator.status === 'number'
  ? STATUS_MAP[validator.status]
  : validator.status;
```

---

## Data Normalization

Regardless of source chain, the collector writes a **canonical JSON schema** that the frontend can consume uniformly.

### Canonical `data.json` Schema

```json
{
  "meta": {
    "chainId": "cosmoshub-4",
    "latestHeight": 21504821,
    "blockTime": 6.23,
    "validatorCount": 180,
    "fetchedAt": "2024-11-20T14:32:10.123Z"
  },
  "validators": [
    {
      "rank": 1,
      "moniker": "Cumulo",
      "operatorAddress": "cosmosvaloper1...",
      "consensusAddress": "cosmosvalcons1...",
      "avatarUrl": "https://...",
      "votingPower": "12345678901234",
      "votingPowerPct": 1.23,
      "commission": 0.05,
      "maxCommission": 0.20,
      "status": "BONDED",
      "jailed": false,
      "tombstoned": false,
      "uptime": 99.3,
      "signed": 149,
      "missed": 1,
      "blocks": [
        { "height": 21504821, "signed": true },
        { "height": 21504820, "signed": true }
      ]
    }
  ],
  "recentBlocks": [
    {
      "height": 21504821,
      "hash": "ABC123...",
      "time": "2024-11-20T14:32:10.123Z",
      "proposer": "Cumulo",
      "txCount": 3,
      "gasUsed": 185000
    }
  ],
  "stats": {
    "bondedTokens": "4500000000000",
    "totalSupply": "1159712345678",
    "stakingRatio": 38.8,
    "inflation": 0.082,
    "communityPool": "12345678901",
    "avgBlockTime": 6.23,
    "tps": 0.48
  }
}
```

### Canonical `governance.json` Schema

```json
{
  "proposals": [
    {
      "id": "942",
      "title": "Update community spend",
      "description": "...",
      "status": "PROPOSAL_STATUS_VOTING_PERIOD",
      "depositTotal": "512000000",
      "depositThreshold": "512000000",
      "votingStart": "2024-11-18T12:00:00Z",
      "votingEnd": "2024-11-25T12:00:00Z",
      "tally": {
        "yes": "450000000000",
        "no": "12000000000",
        "abstain": "5000000000",
        "veto": "1000000000",
        "pctYes": 95.3,
        "pctNo": 2.5,
        "pctAbstain": 1.1,
        "pctVeto": 0.2
      },
      "votes": [
        { "voter": "cosmos1...", "option": "VOTE_OPTION_YES" }
      ]
    }
  ]
}
```
