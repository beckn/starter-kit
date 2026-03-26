# Getting Started with Beckn

This document is a developer toolkit for building and testing a sample network on the **beckn network testnet**. This devkit (developer kit) provides a pre-configured adapter stack and ready-to-use Postman collections to help you get started quickly — whether you're running locally with Docker or deploying to a cloud VPS.

---

## Table of Contents

- [What is Beckn?](#what-is-beckn-and-ddm)  
- [How the Devkit Works](#how-the-devkit-works)  
  - [The Four Services](#the-four-services)  
  - [The Full Architecture](#the-full-architecture)  
  - [Tracing a Request End-to-End](#tracing-a-request-end-to-end)  
- [Repository Structure](#repository-structure)  
  - [config/ — How Each File Is Used](#config--how-each-file-is-used)  
  - [postman/ — What the Collections Do](#postman--what-the-collections-do)  
- [Prerequisites](#prerequisites)  
- [Deployment: Local with Docker](#deployment-local-with-docker)  
- [Deployment: Cloud VPS](#deployment-cloud-vps)  
- [Importing and Running Postman Collections](#importing-and-running-postman-collections)  
- [Understanding the Beckn Transaction Flow](#understanding-the-beckn-transaction-flow)  
- [Customising the Devkit](#customising-the-devkit)  
- [Troubleshooting](#troubleshooting)  
- [License](#license)

---

## What is Beckn?

**Beckn** is an open protocol that lets any application participate in any open network without being locked into a single platform. Think of it like HTTP for transactions: a consumer-side app (for a buyer of product or seeker of services)  and a provider-side app (for a seller of a product or provider of services) can discover each other, negotiate, and transact — even if they were built by completely different teams — as long as both speak the beckn protocol.

Every beckn network has two kinds of participants:

- **BAP (Beckn Application Platform)** — the buyer side. A BAP initiates transactions: it sends `discover`, `select`, `init`, `confirm`, and so on.  
- **BPP (Beckn Provider Platform)** — the seller/provider side. A BPP responds: it sends back `on_discover`, `on_select`, `on_init`, `on_confirm`, and so on.

Every message is digitally signed, schema-validated, and routed through middleware adapters called **ONIX adapters** (from the [beckn-onix](https://github.com/beckn/beckn-onix) project). These adapters handle all the protocol plumbing so your application code only needs to deal with business logic.

---

## How the Devkit Works

The devkit spins up a self-contained beckn network on your machine. It simulates both sides of a transaction — the buyer (BAP) and the seller (BPP) — so you can observe the full protocol flow without needing to connect to any external service.

### The Four Services

```
┌──────────────────────────────────────────────────────────────┐
│                     docker-compose stack                     │
│                                                              │
│  ┌─────────────────────┐    ┌─────────────────────────────┐  │
│  │   sandbox-bap       │    │         onix-bap            │  │
│  │   (port 3001)       │◄──►│         (port 8081)         │  │
│  │                     │    │                             │  │
│  │  BAP app simulator  │    │  BAP-side ONIX adapter:     │  │
│  │  Sends requests.    │    │  signs, validates, routes   │  │
│  │  Receives on_* cbs. │    │  outgoing calls; validates  │  │
│  └─────────────────────┘    │  sigs on incoming on_*      │  │
│                             └─────────────────────────────┘  │
│                                          │                   │
│                                     beckn_network            │
│                                          │                   │
│  ┌─────────────────────┐    ┌─────────────────────────────┐  │
│  │   sandbox-bpp       │    │         onix-bpp            │  │
│  │   (port 3002)       │◄──►│         (port 8082)         │  │
│  │                     │    │                             │  │
│  │  BPP app simulator  │    │  BPP-side ONIX adapter:     │  │
│  │  Receives requests. │    │  validates sigs on incoming │  │
│  │  Sends on_* resps.  │    │  calls; signs, routes       │  │
│  └─────────────────────┘    │  outgoing on_* responses    │  │
│                             └─────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  redis (port 6379) — shared caching layer            │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

| Service | Image | Port | Role |
| :---- | :---- | :---- | :---- |
| `onix-bap` | `fidedocker/onix-adapter` | 8081 | BAP-side protocol adapter (signs, routes, validates) |
| `onix-bpp` | `fidedocker/onix-adapter` | 8082 | BPP-side protocol adapter (signs, routes, validates) |
| `sandbox-bap` | `fidedocker/sandbox-2.0` | 3001 | Simulates a BAP application (sends requests, receives callbacks) |
| `sandbox-bpp` | `fidedocker/sandbox-2.0` | 3002 | Simulates a BPP application (receives requests, sends responses) |
| `redis` | `redis:alpine` | 6379 | Shared request/response cache |

**The ONIX adapter** (`fidedocker/onix-adapter`) is the core middleware that implements the beckn protocol. It is a plugin-based Go server from the [beckn-onix](https://github.com/beckn/beckn-onix) project. Both `onix-bap` and `onix-bpp` run the same adapter binary but with different config files — one configures it to behave as a BAP adapter, the other as a BPP adapter.

**The sandbox** (`fidedocker/sandbox-2.0`) is a network-aware application simulator. It provides HTTP endpoints that the Postman collection calls to trigger transactions, and the simulator also receives callback responses from the adapter.

### The Full Architecture

Here is how a transaction flows through the full stack:

```
You (Postman)
     │
     │  POST /bap/caller/discover
     ▼
┌─────────────────────────────────────────────────────┐
│  onix-bap  :8081  — module: bapTxnCaller            │
│                                                     │
│  Steps executed:                                    │
│    1. addRoute    → reads BAPCaller routing config  │
│    2. sign        → Ed25519-signs the message       │
│    3. validateSchema → checks against Beckn schema  │
└───────────────────────────┬─────────────────────────┘
                            │
            ┌───────────────▼──────────────────┐
            │  (discover only)                 │
            │  Live testnet BPP on internet    │
            │  https://<discover-service>      │
            └───────────────┬──────────────────┘
                            │  async: on_discover callback
                            ▼
┌─────────────────────────────────────────────────────┐
│  onix-bap  :8081  — module: bapTxnReceiver          │
│                                                     │
│  Steps executed:                                    │
│    1. validateSign  → verifies BPP's signature      │
│    2. addRoute      → reads BAPReceiver routing cfg │
│    3. validateSchema → checks response schema       │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  sandbox-bap  :3001                                 │
│  endpoint: /api/bap-webhook                         │
│  (receives and stores the on_discover response)     │
└─────────────────────────────────────────────────────┘


— For select / init / confirm (local BPP path) —

You (Postman)
     │
     │  POST /bap/caller/select
     ▼
┌─────────────────────────────────────────────────────┐
│  onix-bap  :8081  — module: bapTxnCaller            │
│  Routes to → onix-bpp via beckn_network             │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  onix-bpp  :8082  — module: bppTxnReceiver          │
│                                                     │
│  Steps executed:                                    │
│    1. validateSign  → verifies BAP's signature      │
│    2. addRoute      → reads BPPReceiver routing cfg │
│    3. validateSchema → checks request schema        │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  sandbox-bpp  :3002                                 │
│  endpoint: /api/webhook                             │
│  (receives the select; prepares on_select response) │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  onix-bpp  :8082  — module: bppTxnCaller            │
│                                                     │
│  Steps executed:                                    │
│    1. addRoute      → reads BPPCaller routing cfg   │
│    2. sign          → Ed25519-signs the on_select   │
│    3. validateSchema → checks response schema       │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  onix-bap  :8081  — module: bapTxnReceiver          │
│  Validates BPP signature, routes to sandbox-bap     │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  sandbox-bap  :3001  /api/bap-webhook               │
│  (receives and stores the on_select response)       │
└─────────────────────────────────────────────────────┘
```

### Tracing a Request End-to-End

To make this concrete, here is what happens when you fire a `discover` from Postman:

1. **Postman → onix-bap** (`POST localhost:8081/bap/caller/discover`) The `bapTxnCaller` module picks up the request.  
2. **addRoute** — the adapter reads `local-simple-routing-BAPCaller.yaml` and sees that `discover` maps to the live testnet URL (`https://<discovr>/beckn`).  
3. **sign** — the adapter signs the request body using the Ed25519 private key defined in `local-simple-bap.yaml` under `keyManager`.  
4. **validateSchema** — the Beckn schema is fetched from GitHub (or cache) and the message is validated.  
5. **Request → live testnet BPP** — the signed, validated `discover` is forwarded to the real BPP on the internet.  
6. **Async callback** — the testnet BPP sends an `on_discover` back to your adapter's receiver endpoint: `POST localhost:8081/bap/receiver/on_discover`.  
7. **validateSign** — the adapter verifies the BPP's digital signature on the callback.  
8. **addRoute** — reads `local-simple-routing-BAPReceiver.yaml`, which routes all `on_*` callbacks to `http://sandbox-bap:3001/api/bap-webhook`.  
9. **sandbox-bap receives the on\_discover** — the response is now available in the sandbox for inspection.

For `select`, `init`, and `confirm`, the routing target switches from the discover service to the local `onix-bpp` (port 8082), making the rest of the transaction loop fully self-contained within your environment.

---

## Repository Structure

```
<devkit-directory>/
│
├── config/                          # All adapter configuration
│   ├── local-simple-bap.yaml        # BAP adapter config (modules, plugins, keys)
│   ├── local-simple-bpp.yaml        # BPP adapter config (modules, plugins, keys)
│   ├── local-simple-routing-BAPCaller.yaml    # Where BAP sends outbound requests
│   ├── local-simple-routing-BAPReceiver.yaml  # Where BAP delivers incoming callbacks
│   ├── local-simple-routing-BPPCaller.yaml    # Where BPP sends outbound on_* responses
│   └── local-simple-routing-BPPReceiver.yaml  # Where BPP delivers incoming requests
│
├── install/
│   └── docker-compose-adapter.yml   # Single-file definition of the entire stack
│
├── postman/
│   ├── BAP-*.postman_collection.json   # BAP-side test flows
│   └── BPP-*.postman_collection.json   # BPP-side test flows
│
├── resources/
    └── architecture.png             # Architecture diagram image
```

### config/ — How Each File Is Used

Each ONIX adapter instance loads one primary config file and one or more routing files. The primary config file declares the adapter's **modules** (functional endpoints), and each module references a routing config file.

**`local-simple-bap.yaml`** — loaded by the `onix-bap` container. It defines two modules:

- `bapTxnCaller` listens at `/bap/caller/` and handles outgoing BAP requests (`discover`, `select`, `init`, `confirm`…). It **signs** requests and **routes** them using `local-simple-routing-BAPCaller.yaml`.  
- `bapTxnReceiver` listens at `/bap/receiver/` and handles incoming `on_*` callbacks. It **validates signatures** and **routes** responses to the sandbox using `local-simple-routing-BAPReceiver.yaml`.

**`local-simple-bpp.yaml`** — loaded by the `onix-bpp` container. It defines two modules:

- `bppTxnReceiver` listens at `/bpp/receiver/` and handles incoming BAP requests. It **validates signatures** and **routes** to the sandbox using `local-simple-routing-BPPReceiver.yaml`.  
- `bppTxnCaller` listens at `/bpp/caller/` and handles outgoing BPP responses. It **signs** responses and **routes** them back to the BAP using `local-simple-routing-BPPCaller.yaml`.

**`local-simple-routing-BAPCaller.yaml`** — routing rules for outgoing BAP requests:

- `discover` → routes to the live testnet Discovr (`https://<discovr>/beckn`)  
- All other actions (`select`, `init`, `confirm`…) → routes to the local `onix-bpp` on the Docker network

**`local-simple-routing-BAPReceiver.yaml`** — routes all incoming `on_*` callbacks to `http://sandbox-bap:3001/api/bap-webhook`.

**`local-simple-routing-BPPReceiver.yaml`** — routes all incoming requests (`select`, `init`, `confirm`…) to `http://sandbox-bpp:3002/api/webhook`.

**`local-simple-routing-BPPCaller.yaml`** — routes all outgoing `on_*` responses back to the BAP adapter (`onix-bap`) on the Docker network.

Each config also contains an embedded **keyManager** section with Ed25519 signing/encryption key pairs. These are pre-generated testnet keys registered with the Beckn registry — you don't need to change them to get started.

### postman/ — What the Collections Do

There are two collections, one for each side of the network:

**The BAP collection** simulates a buyer initiating a full transaction. It contains four requests that must be run in order:

1. `discover` — broadcasts intent to find available offerings; routed to the live testnet BPP  
2. `select` — selects a specific offering from the `on_discover` response  
3. `init` — initiates the order for the selected offering  
4. `confirm` — confirms and completes the transaction

Send these to `http://localhost:8081/bap/caller/{action}`.

**The BPP collection** simulates a provider responding to a transaction. It contains four requests:

1. `on_discover` — sends a discovery response back to the BAP  
2. `on_select` — sends a selection acknowledgement  
3. `on_init` — sends an order initiation acknowledgement  
4. `on_confirm` — sends a confirmation acknowledgement

Send these to `http://localhost:8082/bpp/caller/{on_action}`.

In a normal flow you only need the BAP collection — the sandboxes handle BPP responses automatically. The BPP collection is useful when you want to manually control or inspect specific response payloads.

---

## Prerequisites

Before you begin, ensure the following tools are installed:

- **Git** — to clone this repository  
- **Docker** and **Docker Compose** — to run the adapter stack  
  - [Install Docker](https://docs.docker.com/engine/install/)  
  - Docker Compose is included with Docker Desktop; for Linux, follow the [Compose plugin guide](https://docs.docker.com/compose/install/)  
- **Postman** — to import and run the test collections  
  - [Download Postman](https://www.postman.com/downloads/)

For cloud VPS deployment you additionally need SSH access to a Linux server (Ubuntu 22.04 recommended) with ports 8081 and 8082 open.

---

## Deployment: Local with Docker

This is the fastest path to a running network. Everything — both adapters, both sandbox applications, and Redis — runs in Docker containers on your laptop or desktop.

**Step 1 — Clone the repository**

```shell
git clone https://github.com/beckn/starter-kit.git
cd <devkit-directory>/install
```

**Step 2 — Start the full stack**

```shell
docker compose -f docker-compose-adapter.yml up --build
```

The first run will pull the `fidedocker/onix-adapter` and `fidedocker/sandbox-2.0` images. This may take a few minutes. Subsequent starts are fast.

**Step 3 — Verify all services are healthy**

```shell
docker compose -f docker-compose-adapter.yml ps
```

You should see five containers running (`redis`, `onix-bap`, `onix-bpp`, `sandbox-bap`, `sandbox-bpp`). The sandbox services have health checks configured; wait until they show `healthy` before sending requests.

**Step 4 — (Optional) Follow logs in real time**

```shell
docker compose -f docker-compose-adapter.yml logs -f onix-bap onix-bpp
```

This is very useful for understanding what the adapters are doing as you send requests.

**Step 5 — Import the Postman collections and start testing**

See [Importing and Running Postman Collections](#importing-and-running-postman-collections) below.

**Stopping the stack**

```shell
docker compose -f docker-compose-adapter.yml down
```

---

## Deployment: Cloud VPS

Deploying to a cloud VPS (AWS EC2, GCP Compute Engine, Azure VM, DigitalOcean Droplet, etc.) follows the same Docker Compose approach, but requires a few extra steps for network access and security.

**Step 1 — Provision a server**

Any Linux VPS with at least 2 GB RAM works. Ubuntu 22.04 is recommended. Open the following ports in your firewall/security group:

| Port | Purpose |
| :---- | :---- |
| 22 | SSH access |
| 8081 | BAP ONIX adapter |
| 8082 | BPP ONIX adapter |
| 3001 | sandbox-bap (optional — only if you want direct sandbox access) |
| 3002 | sandbox-bpp (optional) |

**Step 2 — Install Docker on the server**

```shell
# SSH into your server
ssh user@your-server-ip

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

**Step 3 — Clone the repository**

```shell
git clone https://github.com/beckn/starter-kit.git
cd <devkit-directory>/install
```

**Step 4 — Update routing configs for your public IP**

By default the routing configs reference `sandbox-bap` and `sandbox-bpp` by Docker network hostname. These work as-is for the internal Docker network. However, if you want external systems to reach your adapters, open `../config/local-simple-routing-BAPReceiver.yaml` and verify the webhook URL matches where your sandbox is reachable from the outside.

If you want to register your adapters with the Beckn testnet registry so real external BPPs can discover and call you, update the `networkParticipant` values in the config YAML files to your actual domain name and generate fresh key pairs (see [Customising the Devkit](#customising-the-devkit)).

**Step 5 — Start the stack**

```shell
docker compose -f docker-compose-adapter.yml up -d
```

The `-d` flag runs it in detached (background) mode.

**Step 6 — Verify**

```shell
docker compose -f docker-compose-adapter.yml ps
docker compose -f docker-compose-adapter.yml logs --tail=50
```

**Step 7 — Point Postman at your server**

In your Postman collection, change the base URL from `http://localhost` to `http://your-server-ip`. All ports remain the same.

**Running as a persistent service**

For a server that should survive reboots, add a restart policy. In `docker-compose-adapter.yml`, add `restart: unless-stopped` to each service definition, then:

```shell
docker compose -f docker-compose-adapter.yml up -d
```

Docker will automatically restart the stack on reboot.

**Using a domain name and TLS (production-like setup)**

For a more complete setup with HTTPS, place an Nginx or Caddy reverse proxy in front of the adapters:

```
Internet → Nginx (:443) → onix-bap (:8081)
                        → onix-bpp (:8082)
```

Caddy example (auto-TLS via Let's Encrypt):

```
bap.yourdomain.com {
    reverse_proxy localhost:8081
}

bpp.yourdomain.com {
    reverse_proxy localhost:8082
}
```

Update your routing configs to use `https://bap.yourdomain.com` and `https://bpp.yourdomain.com` as the external URLs, and register these with the Beckn testnet registry.

---

## Importing and Running Postman Collections

**Step 1 — Open Postman and import**

1. Launch Postman.  
2. Click **Import** (top-left).  
3. Select **File**, then navigate to the `postman/` directory.  
4. Import both JSON collection files (the BAP collection and the BPP collection).

**Step 2 — Set the base URL**

Each collection has a `BASE_URL` collection variable. Set it to:

- `http://localhost` for local deployment  
- `http://your-server-ip` for VPS deployment

**Step 3 — Run the BAP collection in sequence**

The four requests in the BAP collection must be run in order: `discover` → `select` → `init` → `confirm`. This mirrors a complete transaction on the beckn network.

After sending `discover`, check the `onix-bap` logs to watch the adapter sign and forward the request. When the testnet BPP responds with `on_discover`, you'll see the callback arrive at the `bapTxnReceiver` module and get delivered to the sandbox.

---

## Understanding the Beckn Transaction Flow

A beckn transaction in the Generic domain follows this standard lifecycle:

```
BAP (Buyer)                  Network / BPP             BPP (Provider)
    │                              │                         │
    │──── discover ───────────────►│──── discover ──────────►│
    │                              │                         │
    │◄─── on_discover ─────────────│◄─── on_discover ────────│
    │    (catalog of offerings)    │                         │
    │                              │                         │
    │──── select ────────────────────────────────────────-►  │
    │    (choose an offering)      │                         │
    │                              │                         │
    │◄─── on_select ─────────────────────────────────────--  │
    │    (pricing / terms)         │                         │
    │                              │                         │
    │──── init ────────────────────────────────────────-──►  │
    │    (initiate order)          │                         │
    │                              │                         │
    │◄─── on_init ──────────────────────────────────────-─-  │
    │    (draft order details)     │                         │
    │                              │                         │
    │──── confirm ─────────────────────────────────────-──►  │
    │    (confirm order)           │                         │
    │                              │                         │
    │◄─── on_confirm ───────────────────────────────────-─-  │
    │    (order confirmation)      │                         │
```

Every arrow in this diagram is a signed, schema-validated HTTP POST routed through the ONIX adapters. The devkit pre-configures all of this so you can observe the full flow without writing any code.

---

## Customising the Devkit

**Changing which BPP the BAP connects to**

Edit `config/local-simple-routing-BAPCaller.yaml`. The `target.url` under the `discover` endpoint points to the live testnet BPP. Replace it with any beckn-compliant BPP URL.

**Running fully offline (no live testnet)**

Change the `discover` target in `local-simple-routing-BAPCaller.yaml` from the live testnet URL to `http://onix-bpp:8082/bpp/receiver/`. This routes discovery to your local BPP, making the stack fully self-contained with no internet dependency.

**Using your own participant identity (key pairs)**

The config files contain pre-generated testnet key pairs for two pre-registered sandbox participants. These are registered with the Beckn testnet registry and work out of the box.

To register your own participant:

1. Generate a new Ed25519 key pair.  
2. Register your `networkParticipant` domain and public key with the Beckn testnet registry.  
3. Update the `keyManager` section in `local-simple-bap.yaml` and `local-simple-bpp.yaml` with your new identity and keys.

**Swapping the sandbox application**

The `sandbox-bap` and `sandbox-bpp` services are the default application simulators. You can replace either with your own application — once you have the applications ready, the application endpoints need to be mapped in ONIX routing configuration and be on the same Docker network (`beckn_network`).

**Connecting to a different beckn domain**

The routing configs use `<domain>` as the identifier. To test a different beckn domain, update the `domain` field in all four routing YAML files and point the schema validator to the appropriate OpenAPI spec URL.

---

## Troubleshooting

**Containers fail to start**

Check for port conflicts on 8081, 8082, 3001, 3002, or 6379\. Inspect logs with:

```shell
docker compose -f docker-compose-adapter.yml logs
```

**sandbox-bap or sandbox-bpp stays in `starting` state**

The sandboxes have a health check that polls `/api/health`. If they don't become healthy within a minute, check their logs:

```shell
docker compose -f docker-compose-adapter.yml logs sandbox-bap
```

**Postman requests return connection errors**

Ensure the Docker stack is fully running (`docker compose ps` shows all containers as `running` or `healthy`) and that the `BASE_URL` collection variable is set correctly.

**Signature validation errors in adapter logs**

If you see `validateSign` failures, the most common cause is clock skew. Beckn signatures include a timestamp and have a short validity window. Ensure your system clock is accurate (`timedatectl` on Linux, or check Docker's time sync settings).

**`discover` returns no results from the live testnet**

The live testnet BPP at `34.93.141.21.sslip.io` must be reachable from your machine. Test with `curl -s https://34.93.141.21.sslip.io/beckn`. If it times out, the testnet may be temporarily unavailable, or your network may block outbound HTTPS to that IP.

**Images fail to pull**

Ensure Docker has sufficient resources (at least 2 GB RAM) and a stable internet connection for pulling `fidedocker/onix-adapter` and `fidedocker/sandbox-2.0`.

**Stopping and cleaning up**

```shell
# Stop all containers
docker compose -f docker-compose-adapter.yml down

# Stop and remove volumes and cached data
docker compose -f docker-compose-adapter.yml down -v
```
