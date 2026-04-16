# Architecture

This document describes the technical architecture of the Cumulo Block Explorer: how data flows from blockchain nodes to the web frontend, how collectors are deployed, and how new chains are integrated.

---

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [System Components](#system-components)
- [Collector](#collector)
- [Static File Server](#static-file-server)
- [Web Frontend](#web-frontend)
- [Deployment](#deployment)
- [Adding a New Chain](#adding-a-new-chain)

---

## Design Philosophy

The architecture follows three core principles:

1. **No database** — All state is held in JSON files on disk. The collector overwrites them on every cycle. The frontend reads them over HTTP. This eliminates a class of operational failures (database outages, schema migrations, replication lag) and makes the entire stack trivially portable.

2. **Collector co-location** — Each collector runs on the same server as the RPC node it monitors. This means RPC calls are local-network or loopback, with zero public internet latency or rate-limit risk.

3. **Uniform module surface** — Every supported network exposes the same seven modules (Validators, Blocks, Uptime, Governance, Stats, Consensus, Decentralization) regardless of the underlying chain architecture. Network-specific data normalization happens inside the collector, not in the frontend.

---

## System Components

```
┌──────────────────────────────────────────────────────────────────┐
│  BLOCKCHAIN NODE LAYER                                           │
│                                                                  │
│  ┌────────────────────┐    ┌─────────────────────────────────┐  │
│  │  CometBFT RPC      │    │  Cosmos REST API                │  │
│  │  :26657            │    │  :1317                          │  │
│  │  /status           │    │  /cosmos/staking/v1beta1/...    │  │
│  │  /block            │    │  /cosmos/gov/v1/...             │  │
│  │  /validators       │    │  /cosmos/mint/v1beta1/...       │  │
│  │  /net_info         │    │  /cosmos/bank/v1beta1/...       │  │
│  └────────────────────┘    └─────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────┘
                                        │ direct RPC (loopback / LAN)
                                        ▼
┌──────────────────────────────────────────────────────────────────┐
│  COLLECTION LAYER                                                │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Collector (Node.js)                                      │  │
│  │                                                           │  │
│  │  1. Fetch block data (RPC)         every 6s              │  │
│  │  2. Fetch validator data (REST)    every 6s              │  │
│  │  3. Fetch governance (REST)        every 30s             │  │
│  │  4. Enrich: Keybase avatars        every 24h             │  │
│  │  5. Normalize to canonical schema                        │  │
│  │  6. Write atomic JSON snapshot                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Runs as: systemd service (auto-restart on failure)             │
│  Output:  /var/lib/<chain>-collector/                           │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ file system writes
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│  SERVING LAYER                                                   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  nginx                                                    │  │
│  │                                                           │  │
│  │  location /data/ {                                        │  │
│  │      alias /var/lib/<chain>-collector/;                   │  │
│  │      add_header Access-Control-Allow-Origin *;            │  │
│  │      add_header Cache-Control "no-cache";                 │  │
│  │  }                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  TLS termination via Let's Encrypt (certbot)                    │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ HTTPS GET /data/*.json
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                              │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Web Frontend (PHP + React)                               │  │
│  │                                                           │  │
│  │  - PHP renders page shell + navigation                    │  │
│  │  - React components fetch & render JSON data              │  │
│  │  - Polling interval: 6s (configurable per network)        │  │
│  │  - No SSR / no database reads                             │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Collector

### Responsibilities

The collector is a Node.js process with a single responsibility: **produce a consistent JSON snapshot on every cycle**. It does not serve HTTP, does not maintain a database, and does not do any rendering.

Each collection cycle:

1. **Fetch block data** from CometBFT RPC — latest block height, block hash, proposer, transactions, timestamps
2. **Fetch validator set** from REST API — operator addresses, voting power, commission, status, slashing info
3. **Fetch staking pool** — bonded and unbonded token totals
4. **Fetch inflation** — current inflation rate from the mint module
5. **Fetch supply** — total and circulating supply per denomination
6. **Fetch governance** — proposals and live tally (on a longer cycle, e.g., 30s)
7. **Fetch consensus state** — live CometBFT round state
8. **Enrich** — resolve Keybase avatars (cached for 24h), decode bech32 addresses
9. **Write** — atomically write `data.json`, `governance.json`, and `consensus.json`

### Atomic Writes

To prevent the frontend from reading a partially-written file, the collector writes to a temporary file and renames it:

```javascript
const tmpPath = path.join(OUTPUT_DIR, 'data.tmp.json');
const finalPath = path.join(OUTPUT_DIR, 'data.json');

await fs.writeFile(tmpPath, JSON.stringify(payload, null, 2));
await fs.rename(tmpPath, finalPath);
```

`rename` on the same filesystem is atomic on Linux (POSIX guarantee).

### Error Handling

- All parallel fetches use `Promise.allSettled()` so a single API failure does not abort the entire cycle
- HTTP requests have an 8-second timeout via `AbortController`
- The collector catches all errors, logs them, and continues on the next interval — it never exits on a transient failure
- systemd `Restart=always` with `RestartSec=10` handles process crashes

### Collection Intervals

| Network Type | Interval | Rationale |
|---|---|---|
| Cosmos Hub | 6s | ~6s block time |
| Celestia | 12s | ~12s block time |
| Story | 3s | ~2–3s block time |
| Avail | 20s | ~20s block time |
| Governance | 30s | Low-frequency changes |
| Keybase avatars | 24h | Rate limit compliance |

---

## Static File Server

nginx is configured to serve the collector output directory over HTTPS with permissive CORS headers (since the frontend is served from a separate subdomain):

```nginx
server {
    listen 443 ssl;
    server_name data.cosmos.cumulo.com.es;

    ssl_certificate /etc/letsencrypt/live/data.cosmos.cumulo.com.es/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/data.cosmos.cumulo.com.es/privkey.pem;

    location /data/ {
        alias /var/lib/cosmos-mainnet-collector/;
        add_header Access-Control-Allow-Origin "*";
        add_header Cache-Control "no-cache, must-revalidate";
        default_type application/json;
    }
}
```

No backend application server is needed. nginx serves files directly from disk at several thousand requests per second if required.

---

## Web Frontend

### Technology Stack

| Library | Version | Role |
|---|---|---|
| PHP | 8.x | Page routing, layout rendering, server-side includes |
| React | 18 (CDN) | Reactive UI components |
| Babel (standalone) | Latest | JSX transpilation in-browser |
| Tailwind CSS | CDN | Utility-first styling |
| Leaflet.js | 1.9 | Geographic map (decentralization module) |
| Alpine.js | 3.x | Lightweight interactivity for non-React elements |

### Frontend Fetch Pattern

Every React component uses a consistent polling pattern:

```javascript
const [data, setData] = React.useState(null);
const [loading, setLoading] = React.useState(true);

React.useEffect(() => {
  const fetchData = async () => {
    const response = await fetch('https://data.cosmos.cumulo.com.es/data/data.json');
    const json = await response.json();
    setData(json);
    setLoading(false);
  };

  fetchData();
  const interval = setInterval(fetchData, 6000);
  return () => clearInterval(interval);
}, []);
```

This pattern ensures:
- Initial load on mount
- Automatic refresh every 6 seconds
- No stale data accumulates across navigations
- Clean teardown on component unmount

### PHP API Proxy

Some REST calls require server-side proxying to avoid CORS or to aggregate data. These use a lightweight `api-proxy.php` that forwards requests and returns the raw response:

```php
<?php
$url = $_GET['url'] ?? '';
if (!$url) { http_response_code(400); exit; }

$response = file_get_contents($url);
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');
echo $response;
```

---

## Deployment

### Server Layout (Per Network)

```
Server (OVH / Hetzner)
├── /opt/<chain>-collector/
│   ├── <chain>-collector.js       # Collector script
│   ├── package.json
│   └── node_modules/
├── /var/lib/<chain>-collector/    # Output directory
│   ├── data.json
│   ├── governance.json
│   └── consensus.json
└── /etc/systemd/system/<chain>-collector.service
```

### systemd Unit File

```ini
[Unit]
Description=<Chain> Block Explorer Collector
After=network.target

[Service]
Type=simple
User=collector
WorkingDirectory=/opt/<chain>-collector
ExecStart=/usr/bin/node <chain>-collector.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl enable <chain>-collector
systemctl start <chain>-collector
journalctl -fu <chain>-collector   # follow logs
```

---

## Adding a New Chain

To add a new network to the explorer:

### Step 1 — Create the Collector

Copy the closest existing collector (e.g., `cosmos-mainnet-collector.js`) and adjust:

```javascript
const CONFIG = {
  RPC_URL:      'https://rpc.newchain.example.com',
  API_URL:      'https://api.newchain.example.com',
  OUTPUT_DIR:   '/var/lib/newchain-collector',
  INTERVAL:     6000,        // ms, match block time
  UPTIME_BLOCKS: 150,        // signing history window
  CHAIN_ID:     'newchain-1',
  BECH32_PREFIX: 'newchain', // address prefix
  BOND_DENOM:   'unewtoken', // staking denomination
  DECIMALS:     6,
};
```

If the chain uses a **non-standard API** (e.g., custom wrapper, different field names), add normalization in the `normalizeValidators()` function.

### Step 2 — Deploy the Collector

```bash
# On the target server
mkdir -p /opt/newchain-collector /var/lib/newchain-collector
cp newchain-collector.js /opt/newchain-collector/
cd /opt/newchain-collector && npm install
cp newchain-collector.service /etc/systemd/system/
systemctl enable --now newchain-collector
```

### Step 3 — Configure nginx

Add a new server block to nginx, pointing to `/var/lib/newchain-collector`, with TLS from certbot.

### Step 4 — Create Frontend

Copy the closest existing explorer folder (e.g., `EXPLORER COSMOS MAINNET/`) and update the data endpoint URLs and chain-specific labels (denomination, bech32 prefix, etc.).

### Non-Cosmos Networks

For networks that do **not** use Cosmos SDK (Substrate, EVM, UTXO, etc.), the collector replaces the REST API calls with the appropriate protocol:

| Architecture | Protocol | Key Difference |
|---|---|---|
| Substrate (Avail) | JSON-RPC + SCALE | Custom SCALE codec decoder needed for complex types |
| EVM (Story) | EVM JSON-RPC | Operator addresses in hex (0x...), numeric status codes |
| StarkWare | Starknet RPC | Proof-based consensus, different finality model |
| UTXO (Fuel) | FuelVM RPC | No validator set in traditional PoS sense |

The frontend modules remain identical; only the collector changes.
