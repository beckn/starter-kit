# Generic Devkit — Beckn Protocol v2.0.0

Goal of this devkit is to enable any developer to run a **complete, usecase-agnostic Beckn v2.0.0 round-trip** on their local machine within a few minutes — no domain knowledge of EV charging, retail, or any other sector required.

It covers all 11 Beckn protocol actions across the full transaction lifecycle:

```
discover → on_discover
select → on_select → init → on_init → confirm → on_confirm
→ status / on_status  →  track / on_track
→ update / on_update  →  cancel / on_cancel
→ rate / on_rate  →  support / on_support
(+ catalog/publish → catalog/on_publish)
```

---

## Prerequisites

1. [Docker Desktop](https://www.docker.com/products/docker-desktop) — installed and running
2. [Git](https://git-scm.com/downloads) — on your system path
3. [Postman](https://www.postman.com/downloads/) — for sending API calls

---

## Quick Start

### 1. Clone and navigate

```bash
git clone https://github.com/beckn/beckn-onix.git
cd beckn-onix
git checkout testnet
cd testnet/generic-devkit/install
```

### 2. Start containers

```bash
docker compose -f docker-compose-generic.yml up -d
docker ps
```

You should see five containers running:

| Container | Port | Role |
|---|---|---|
| `redis` | 6379 | Shared cache |
| `onix-bap` | 8081 | BAP adapter (caller + receiver) |
| `onix-bpp` | 8082 | BPP adapter (caller + receiver) |
| `sandbox-bap` | 3001 | Mock BAP application (receives callbacks) |
| `sandbox-bpp` | 3002 | Mock BPP application (processes requests) |

### 3. Import Postman collections

Open Postman → **Import** → select the entire folder:

```
testnet/generic-devkit/postman/
```

This imports two collections:

- **BAP — Beckn Protocol v2.0.0 Generic** — 11 outbound requests sent by the BAP
- **BPP — Beckn Protocol v2.0.0 Generic** — 11 inbound callbacks sent by the BPP

### 4. Run the flow

Use the **BAP collection** to drive the transaction lifecycle in order:

| Step | Action | Folder |
|---|---|---|
| 1 | `discover` | Discovery |
| 2 | `select` | Transaction |
| 3 | `init` | Transaction |
| 4 | `confirm` | Transaction |
| 5 | `status` | Fulfillment |
| 6 | `track` | Fulfillment |
| 7 | `update` | Fulfillment |
| 8 | `cancel` | Fulfillment |
| 9 | `rate` | Post-Fulfillment |
| 10 | `support` | Post-Fulfillment |
| 11 | `catalog/publish` | Catalog Publishing |

Each request returns an `ACK`. The corresponding `on_*` callback from the BPP arrives at `sandbox-bap` and can be viewed in the BAP logs:

```bash
docker logs -f onix-bap
docker logs -f sandbox-bap
```

Use the **BPP collection** to simulate BPP-initiated callbacks directly (e.g. unsolicited `on_status` push or `on_update`).

---

## Configuration Reference

### Collection Variables (Postman)

| Variable | Default value | Notes |
|---|---|---|
| `version` | `2.0.0` | Beckn protocol version |
| `network_id` | `beckn.one/testnet` | Matches `context.networkId` |
| `bap_id` | `bap.example.com` | BAP subscriber ID |
| `bap_uri` | `https://bap.example.com/bap/receiver` | BAP callback URL (in-container: `http://onix-bap:8081/bap/receiver`) |
| `bpp_id` | `bpp.example.com` | BPP subscriber ID |
| `bpp_uri` | `https://bpp.example.com/bpp/receiver` | BPP endpoint URL (in-container: `http://onix-bpp:8082/bpp/receiver`) |
| `bap_adapter_url` | `http://localhost:8081/bap/caller` | BAP collection target |
| `bpp_adapter_url` | `http://localhost:8082/bpp/caller` | BPP collection target |
| `transaction_id` | fixed UUID | Shared across the full flow; refresh for a new session |
| `auth_header` | _(empty)_ | Beckn HTTP Signature — auto-added by the adapter signer |

`messageId` and `timestamp` are auto-generated per request via Postman's `{{$guid}}` and `{{$isoTimestamp}}`.

### Adapter Endpoints

| Adapter | Path | Purpose |
|---|---|---|
| BAP | `http://localhost:8081/bap/caller/` | Send outbound action requests (Postman target) |
| BAP | `http://localhost:8081/bap/receiver/` | Receive inbound `on_*` callbacks from BPPs |
| BPP | `http://localhost:8082/bpp/caller/` | Send outbound `on_*` callbacks (Postman target) |
| BPP | `http://localhost:8082/bpp/receiver/` | Receive inbound action requests from BAPs |

### Config Files

| File | Purpose |
|---|---|
| `config/generic-bap.yaml` | BAP adapter config — ports, keys, schema validator, routing |
| `config/generic-bpp.yaml` | BPP adapter config — ports, keys, schema validator, routing |
| `config/generic-routing-BAPCaller.yaml` | Routes outbound BAP requests to BPP (registry) or CDS (URL) |
| `config/generic-routing-BAPReceiver.yaml` | Routes inbound `on_*` callbacks to `sandbox-bap` |
| `config/generic-routing-BPPCaller.yaml` | Routes outbound BPP callbacks to BAP (registry) |
| `config/generic-routing-BPPReceiver.yaml` | Routes inbound action requests to `sandbox-bpp` |

### Registry & Keys

The devkit ships with **testnet sandbox credentials** pre-registered on `beckn.one/testnet` via the DeDi registry (`https://api.dev.beckn.io/registry/dedi`).

To register your own subscriber IDs and keys, follow the [DeDi registration guide](https://developers.becknprotocol.io/) and update the `keyManager` block in `generic-bap.yaml` / `generic-bpp.yaml`.

### Schema Validation

Both adapters validate all payloads against the canonical Beckn v2.0.0 OpenAPI spec:

```
https://raw.githubusercontent.com/beckn/protocol-specifications-v2/refs/heads/draft/api/v2.0.0/beckn.yaml
```

Extended (domain-specific) schema validation is **disabled** by default (`extendedSchema_enabled: "false"`), keeping this devkit fully usecase-agnostic.

---

## Message Flow Diagram

```
┌──────────────┐   POST /bap/caller/discover    ┌──────────────┐
│   Postman    │──────────────────────────────► │  onix-bap    │
│  (BAP coll.) │                                │  :8081       │
└──────────────┘                                └──────┬───────┘
                                                       │ signs + routes → BPP (registry)
                                                       ▼
                                                ┌──────────────┐
                                                │  onix-bpp    │
                                                │  :8082       │
                                                └──────┬───────┘
                                                       │ forwards to sandbox-bpp
                                                       ▼
                                                ┌──────────────┐
                                                │ sandbox-bpp  │
                                                │  :3002       │
                                                └──────┬───────┘
                                                       │ generates on_discover response
                                                       ▼
                                                ┌──────────────┐
                                                │  onix-bpp    │  POST /bpp/caller/on_discover
                                                │  :8082       │──────────────────────────────►
                                                └──────────────┘  signs + routes → BAP (registry)
                                                                          │
                                                                          ▼
                                                                   ┌──────────────┐
                                                                   │  onix-bap    │
                                                                   │  :8081       │
                                                                   └──────┬───────┘
                                                                          │ forwards to sandbox-bap
                                                                          ▼
                                                                   ┌──────────────┐
                                                                   │ sandbox-bap  │
                                                                   │  :3001       │
                                                                   └──────────────┘
                                                                     (view response in logs)
```

---

## Stopping the Environment

```bash
docker compose -f docker-compose-generic.yml down
```

---

## Troubleshooting

**Container fails to start**
```bash
docker pull fidedocker/onix-adapter
docker pull fidedocker/sandbox-2.0:latest
```

**Registry lookup fails** — ensure internet connectivity to `https://api.dev.beckn.io`.

**Schema validation error** — verify the payload matches the v2.0.0 spec. Run:
```bash
docker logs onix-bap 2>&1 | grep -i "schema\|error"
```

**on_* callback not received** — check BAP receiver logs:
```bash
docker logs -f onix-bap
```

**Sandbox health check fails** — allow 10–15 seconds for sandbox containers to initialise:
```bash
docker logs sandbox-bap
docker logs sandbox-bpp
```
