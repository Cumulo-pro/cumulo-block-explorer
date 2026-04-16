# Supported Networks

This document provides the configuration reference for each network supported by the Cumulo Block Explorer, including RPC/API endpoints, block time, chain-specific adaptations, and any deviations from the standard Cosmos SDK model.

---

## Table of Contents

- [Cosmos Hub (Mainnet)](#cosmos-hub-mainnet)
- [Celestia (Mainnet)](#celestia-mainnet)
- [XRPL EVM (Mainnet)](#xrpl-evm-mainnet)
- [Story (Mainnet)](#story-mainnet)
- [Axone (Mainnet)](#axone-mainnet)
- [Dymension (Mainnet)](#dymension-mainnet)
- [Warden (Mainnet)](#warden-mainnet)
- [Avail (Mainnet)](#avail-mainnet)
- [Starknet (Mainnet)](#starknet-mainnet)
- [Testnets](#testnets)
- [Network Compatibility Matrix](#network-compatibility-matrix)

---

## Cosmos Hub (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `cosmoshub-4` |
| Consensus | CometBFT |
| SDK | Cosmos SDK v0.47+ |
| Block Time | ~6 seconds |
| Bond Denom | `uatom` (1 ATOM = 10^6 uatom) |
| Bech32 Prefix | `cosmos` |
| Collector Interval | 6,000 ms |
| Uptime Window | 150 blocks |
| RPC Endpoint | `https://rpc.cosmos.cumulo.com.es` |
| API Endpoint | `https://api.cosmos.cumulo.com.es` |

**API Version:** Governance v1beta1 (legacy) and v1 (both supported, v1 preferred)

**Notable characteristics:**
- Reference implementation of Cosmos SDK; all standard endpoints available
- Ed25519 consensus keys (standard)
- IBC hub — high governance activity

---

## Celestia (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `celestia` |
| Consensus | CometBFT |
| SDK | Cosmos SDK (Celestia fork) |
| Block Time | ~12 seconds |
| Bond Denom | `utia` (1 TIA = 10^6 utia) |
| Bech32 Prefix | `celestia` |
| Collector Interval | 12,000 ms |
| Uptime Window | 150 blocks (~30 min) |
| RPC Endpoint | `https://celestia.cumulo.org.es` |
| API Endpoint | `https://celestia.api.cumulo.org.es` |

**Notable characteristics:**
- Data Availability (DA) layer — blocks contain data blobs in addition to standard transactions
- Longer block time (12s) means the 150-block uptime window covers ~30 minutes instead of ~15
- Governance v1 API

---

## XRPL EVM (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `xrplevm-mainnet` |
| Consensus | CometBFT |
| SDK | Cosmos SDK (EVM-adapted) |
| Block Time | ~6 seconds |
| Bond Denom | `axrp` |
| Bech32 Prefix | `ethm` (EVM-compatible) |
| Collector Interval | 6,000 ms |
| Uptime Window | 150 blocks |
| RPC Endpoint | `https://rpc.xrpl.cumulo.org.es` |
| API Endpoint | `https://api.xrpl.cumulo.org.es` |

**Notable characteristics:**
- EVM-compatible chain (accounts are Ethereum-style `0x` addresses)
- Cosmos SDK staking and governance are fully functional
- Bech32 prefix `ethm` maps to `0x` EVM addresses
- Keybase identity resolution works normally

---

## Story (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `story` |
| Consensus | CometBFT |
| SDK | Story Protocol (custom Cosmos fork) |
| Block Time | ~2–3 seconds |
| Bond Denom | `atto` (1 IP = 10^18 atto) |
| Bech32 Prefix | N/A — operator addresses are `0x` hex |
| Collector Interval | 3,000 ms |
| Uptime Window | 150 blocks (~7 min) |
| RPC Endpoint | `http://92.42.106.179:26647` |
| API Endpoint | `http://92.42.106.179:1317` |

**API Deviations from Standard Cosmos SDK:**

Story wraps all REST responses in a custom envelope:
```json
{ "code": 200, "msg": { <actual_data> }, "error": "" }
```

Additional differences:
| Field | Standard | Story |
|---|---|---|
| `operator_address` | `<prefix>valoper1...` (bech32) | `0x...` (EVM hex) |
| `consensus_pubkey.@type` | `ed25519.PubKey` | `secp256k1.PubKey` |
| `status` | `"BOND_STATUS_BONDED"` (string) | `3` (integer) |
| API base path | `/cosmos/staking/v1beta1/validators` | `/staking/validators` |

**Normalization in collector:**
```javascript
// Status integer → string mapping
const STATUS_MAP = { 1: 'UNBONDED', 2: 'UNBONDING', 3: 'BONDED' };

// Unwrap response envelope
const data = response.msg ?? response;
```

**Denomination note:** Story uses `atto` = 10^-18, equivalent to `wei` in Ethereum. Display conversions use `BigInt` to avoid precision loss.

---

## Axone (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `axone-1` |
| Consensus | CometBFT |
| SDK | Cosmos SDK |
| Block Time | ~6 seconds |
| Bond Denom | `uaxone` |
| Bech32 Prefix | `axone` |
| Collector Interval | 6,000 ms |

**Notable characteristics:**
- AI-focused blockchain for decentralized AI resource sharing
- Standard Cosmos SDK API; no normalization required

---

## Dymension (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `dymension_1100-1` |
| Consensus | CometBFT |
| SDK | Cosmos SDK (EVM-compatible) |
| Block Time | ~6 seconds |
| Bond Denom | `adym` |
| Bech32 Prefix | `dym` |
| Collector Interval | 6,000 ms |

**Notable characteristics:**
- RollApp hub — hosts settlement for application-specific rollups
- EVM-compatible address space

---

## Warden (Mainnet)

| Parameter | Value |
|---|---|
| Chain ID | `warden-mainnet` |
| Consensus | CometBFT |
| SDK | Cosmos SDK |
| Block Time | ~6 seconds |
| Bond Denom | `uward` |
| Bech32 Prefix | `warden` |
| Collector Interval | 6,000 ms |

**Notable characteristics:**
- Intent-based blockchain with on-chain key management (Keychain)
- Standard Cosmos SDK API

---

## Avail (Mainnet)

| Parameter | Value |
|---|---|
| Chain | Avail |
| Consensus | GRANDPA (Substrate) |
| Framework | Substrate |
| Block Time | ~20 seconds |
| Bond Denom | `AVL` |
| Collector Interval | 20,000 ms |
| RPC Port | 9944 (HTTP + WebSocket) |

**Architecture difference:** Avail does not use Cosmos SDK. It is built on the [Substrate](https://substrate.io/) framework (Polkadot ecosystem).

**Key differences from Cosmos chains:**

| Concept | Cosmos SDK | Avail (Substrate) |
|---|---|---|
| Transactions | `Tx` | Extrinsics |
| Consensus | CometBFT (BFT) | BABE + GRANDPA |
| Finality | Instant (single-slot) | Probabilistic + GRANDPA finality |
| RPC | HTTP REST + CometBFT RPC | JSON-RPC (HTTP + WebSocket) |
| Data encoding | JSON | SCALE codec (binary) |
| Validator rotation | Staking unbonding | Session-based (era ~24h) |
| Account format | bech32 | SS58 |

**SCALE codec decoding:** The Avail collector implements custom decoders for:
- `Vec<AccountId32>` — compact-length-prefixed list of 32-byte account IDs
- `u64` — little-endian 64-bit integer
- `Option<u32>` — optional 32-bit integer with presence byte

---

## Starknet (Mainnet)

| Parameter | Value |
|---|---|
| Chain | Starknet |
| Consensus | Proof of Stake + STARK proofs |
| Framework | StarkWare |
| Block Time | ~30 seconds |
| Collector Interval | 30,000 ms |

**Architecture difference:** Starknet is a ZK-rollup on Ethereum. Its "validators" are sequencers (block proposers) rather than a traditional PoS validator set. The explorer monitors sequencer activity, block production, and proof submission.

---

## Testnets

All testnets mirror their mainnet counterpart in architecture but use separate endpoints, chain IDs, and token denominations.

| Testnet | Chain ID | Mainnet Counterpart |
|---|---|---|
| Cosmos Testnet | `theta-testnet-001` | Cosmos Hub |
| Celestia Mocha | `mocha-4` | Celestia |
| XRPL EVM Testnet | `xrplevm-testnet` | XRPL EVM |
| Story Aeneid | `aeneid` | Story |
| SEDA Testnet | `seda-1-testnet` | SEDA (future mainnet) |
| Axone Dendrite-2 | `axone-dentrite-2` | Axone |

Testnet collectors are identical to mainnet but point to testnet RPC/API endpoints and use testnet token denominations (which have no real monetary value).

---

## Network Compatibility Matrix

| Module | Cosmos SDK | Substrate (Avail) | StarkWare | Story (EVM) |
|---|---|---|---|---|
| Validators | Full | Session-based | Sequencers | Full (normalized) |
| Blocks | Full | Full | Full | Full |
| Uptime | Full | GRANDPA slots | N/A | Full |
| Governance | Full | OpenGov (referenda) | N/A | Full |
| Stats | Full | Partial | Partial | Full |
| Consensus | CometBFT round state | GRANDPA rounds | N/A | CometBFT round state |
| Decentralization | Full | Full | Partial | Full |

**Legend:** Full = all data available from standard endpoints. Partial = some fields not available from this network's APIs. N/A = concept does not apply to this architecture.
