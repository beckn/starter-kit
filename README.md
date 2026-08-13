# Getting Started with Beckn

A hands-on starter kit for running a live beckn network in your own environment — complete with a BAP, a BPP, and the Beckn Fabric services that tie them together. No prior beckn experience needed.

---

## Table of Contents

- [What Will You Learn?](#what-will-you-learn)
- [What is Beckn? — A Quick Recap](#what-is-beckn---a-quick-recap)
- [Prerequisites](#prerequisites)
- [Quick Start: Run the Network Locally](#quick-start-run-the-network-locally)
- [Public-Internet Testing (ngrok)](#public-internet-testing-ngrok)
- [Deployment: Cloud VPS](#deployment-cloud-vps)
- [Your First Transaction](#your-first-transaction)
- [Catalog Publishing (`catalog/publish`)](#catalog-publishing-catalogpublish)
- [How It All Works](#how-it-all-works)
  - [The Services in This Stack](#the-services-in-this-stack)
  - [Beckn Fabric: Registry and Discovery](#beckn-fabric-registry-and-discovery)
  - [The Full Transaction Flow](#the-full-transaction-flow)
  - [Tracing a Request End-to-End](#tracing-a-request-end-to-end)
- [Repository Structure](#repository-structure)
  - [config/ — How Each File Is Used](#config--how-each-file-is-used)
  - [postman/ — What the Collections Do](#postman--what-the-collections-do)
- [Customising the Starter Kit](#customising-the-starter-kit)
- [Troubleshooting](#troubleshooting)

---

## What Will You Learn?

By the end of this guide you will have:

1. **A running beckn network** — a BAP (consumer-side platform), a BPP (provider-side platform), and the ONIX adapters that connect them, all running in your own environment, local or cloud. You will also see how a crawler-fed **Discovery Service** serves `discover` requests from BAPs.

2. **A working understanding of Beckn Fabric's DeDi Registry** — how it's used for participant identity, signature verification, and dynamic routing, and how a BPP makes its own catalog independently discoverable by publishing plain files and registering a catalog index URL.

3. **An observable transaction flow** — you will fire real beckn API calls, see the messages get signed and routed, and watch the `discover → select → init → confirm` lifecycle play out end-to-end.

---

## What is Beckn? — A Quick Recap

**Beckn** is an open protocol that allows any buyer-side application to transact with any provider-side application across an open network — without either side being locked into a single platform. Think of it like HTTP for transactions: as long as both sides speak the beckn protocol, they can discover each other, negotiate, and transact, regardless of who built them.

Every beckn network has two kinds of application participants:

- **BAP (Beckn Application Platform)** — the consumer side. A BAP initiates transactions by sending actions: `discover`, `select`, `init`, `confirm`, and so on.
- **BPP (Beckn Provider Platform)** — the provider side. A BPP responds asynchronously: `on_discover`, `on_select`, `on_init`, `on_confirm`, and so on.

These two sides need not talk to each other directly. Every message travels through **ONIX adapters** — middleware that handles digital signing, schema validation, and protocol-level routing. It can also support observability at the network level, business policy enforcement, and much more. This keeps your application code focused on business logic.

Underpinning the network is Beckn Fabric's **DeDi Registry** — shared, trustless infrastructure every participant relies on for identity and routing lookups. Discovery in this starter kit is served by a **crawler-fed Discovery Service**: a BPP publishes its catalog to storage it controls, points its DeDi registry entry at that catalog's index, and a crawler independently discovers and indexes it — no direct call between a publishing BPP and a central catalog service is involved. This is covered in detail in [Beckn Fabric: Registry and Discovery](#beckn-fabric-registry-and-discovery) and [Catalog Publishing](#catalog-publishing-catalogpublish).

---

## Prerequisites

Ensure the following tools are installed before you begin:

- **Git** — to clone this repository
- **Docker** and **Docker Compose**
  - [Install Docker](https://docs.docker.com/engine/install/)
  - Docker Compose ships with Docker Desktop; for Linux see the [Compose plugin guide](https://docs.docker.com/compose/install/)
- **Postman** — to send test requests
  - [Download Postman](https://www.postman.com/downloads/)
- **ngrok** — a static domain is needed to run through the full flow, including making anything you publish actually crawlable (see [Public-Internet Testing (ngrok)](#public-internet-testing-ngrok))
  - [Download ngrok](https://ngrok.com/download) · requires a free ngrok account

For cloud VPS deployment you additionally need SSH access to a Linux server (Ubuntu 22.04 recommended) with ports 8081, 8082, and 9000 open in your firewall.

---

## Quick Start: Run the Network Locally

This is the fastest way to see a beckn network in action. Six Docker containers — two adapters, two application simulators, Redis, and a reverse proxy — start up on your laptop and form a complete, working network.

**Step 1 — Clone the repository**

```shell
git clone https://github.com/beckn/starter-kit.git
cd starter-kit/generic-devkit/install
```

**Step 2 — Start the stack**

```shell
docker compose -f docker-compose-generic.yml up
```

The first run pulls the required Docker images. This takes a few minutes once; subsequent starts are fast.

**Step 3 — Confirm all services are healthy**

```shell
docker compose -f docker-compose-generic.yml ps
```

Wait until all containers show `running` or `healthy`:

| Container | Port | Role |
|---|---|---|
| `beckn-router` | 9000 | Caddy reverse proxy — single entry point, routes `/bap/*` → `onix-bap`, `/bpp/*` → `onix-bpp` |
| `redis` | 6379 | Shared cache for both adapters |
| `onix-bap` | 8081 | BAP-side ONIX adapter (caller + receiver) |
| `onix-bpp` | 8082 | BPP-side ONIX adapter (caller + receiver + `catalog/publish`) |
| `sandbox-bap` | 3001 | Mock BAP application (receives callbacks) |
| `sandbox-bpp` | 3002 | Mock BPP application (processes requests) |

**Step 4 — (Optional) Watch the adapters in real time**

Open a second terminal and tail the adapter logs while you send requests:

```shell
docker compose -f docker-compose-generic.yml logs -f onix-bap onix-bpp
```

You will see each message being signed, validated, and routed as you work through the transaction steps.

**Stopping the stack**

```shell
docker compose -f docker-compose-generic.yml down
```

---

## Public-Internet Testing (ngrok)

When you send a `discover` request, the Discovery Service sends `on_discover` back to whatever URL is in `context.bapUri`. Both `bapUri`/`bppUri` callback URLs must be reachable from the public internet — which your laptop is not, by default. Publishing also depends on the tunnel: the catalog index and catalog files `catalog/publish` writes are only crawlable if they're served from a publicly reachable URL (see [Catalog Publishing](#catalog-publishing-catalogpublish)).

The stack includes **beckn-router**, a Caddy reverse proxy on port 9000 that sits in front of both adapters:

```
internet
    │
https://<your-static-domain>.ngrok-free.app
    │
ngrok → localhost:9000
    │
beckn-router (Caddy)
    ├── /bap/*     →  onix-bap:8081
    ├── /bpp/*     →  onix-bpp:8082
    └── /beckn/*   →  published catalog files (read-only, from ../data/beckn)
```

Tunnelling one port gives you a single stable public URL for the adapters and your published catalog. The `bapUri`/`bppUri` fields in your Postman requests are built from a single `public_url` collection variable, so there is only one value to change there — but note `catalogBaseURL` in `generic-bpp.yaml` is a **separate** setting that must be kept in sync with the same domain (see [Catalog Publishing](#catalog-publishing-catalogpublish)).

### One-time setup

**Step 1 — Get a static ngrok URL and configure the tunnel**

You need an authtoken and a static domain from [ngrok.com](https://ngrok.com) (both available on the free tier). Once you have them:

```shell
cd generic-devkit/install
cp ngrok.yml.example ngrok.yml
```

Edit `ngrok.yml` and fill in your authtoken and static domain:

```yaml
authtoken: <your-authtoken>
tunnels:
  beckn:
    proto: http
    addr: 9000
    domain: <your-static-domain>.ngrok-free.app
```

`ngrok.yml` is git-ignored — your token stays local.

**Step 2 — Update the Postman `public_url` variable**

In both Postman collections, set the `public_url` collection variable to your static domain:

```
public_url  →  https://<your-static-domain>.ngrok-free.app
```

`bapUri` and `bppUri` in every request are derived from this single variable — nothing else to change there.

**Step 3 — Update `catalogBaseURL` if you plan to publish**

If you're using [Catalog Publishing](#catalog-publishing-catalogpublish), also update `catalogBaseURL` in `generic-devkit/config/generic-bpp.yaml` to the same domain — this is what gets stamped into the URLs a crawler will try to fetch, and it is not derived from the Postman variable above.

### Running with the tunnel

Start the Docker stack as usual, then start ngrok in a separate terminal:

```shell
# Terminal 1 — Docker stack
docker compose -f docker-compose-generic.yml up

# Terminal 2 — ngrok tunnel
ngrok start --all --config ngrok.yml
```

Watch the tunnel at `http://localhost:4040`. Each `discover` flow will show the hops: your Postman request → adapter → external Discovery Service → callback arriving back through the tunnel.

**Stopping**

```shell
# Stop Docker
docker compose -f docker-compose-generic.yml down

# Stop ngrok (in its terminal)
Ctrl-C
# or, to kill it from another terminal:
pkill -f 'ngrok start'
```

### Local-only testing (no ngrok needed)

For the direct BAP↔BPP transaction flow (`select`, `init`, `confirm`, `on_select`, `on_init`, `on_confirm`), callbacks are routed internally via the DeDi Registry — no public internet required. Leave `public_url` at its default and skip ngrok entirely. `discover` and `catalog/publish` (if you want your own catalog to be crawlable) both need the tunnel, since both depend on being reachable from outside your machine.

---

## Deployment: Cloud VPS

The same Docker Compose file works on any Linux VPS. Follow these steps after provisioning your server.

**Recommended server spec:** 2 GB RAM, Ubuntu 22.04.

**Firewall — open these ports:**

| Port | Purpose |
|------|---------|
| 22 | SSH |
| 8081 | onix-bap (BAP ONIX adapter) |
| 8082 | onix-bpp (BPP ONIX adapter) |
| 9000 | beckn-router (reverse proxy) |
| 3001 | sandbox-bap (optional — for direct application access) |
| 3002 | sandbox-bpp (optional) |

**Install Docker on the server:**

```shell
ssh user@your-server-ip

curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

**Clone and start:**

```shell
git clone https://github.com/beckn/starter-kit.git
cd starter-kit/generic-devkit/install
docker compose -f docker-compose-generic.yml up -d
```

The `-d` flag runs the stack in the background. Verify with:

```shell
docker compose -f docker-compose-generic.yml ps
docker compose -f docker-compose-generic.yml logs --tail=50
```

**Make the stack survive reboots:**

The `onix-bap` and `onix-bpp` containers already have `restart: unless-stopped` in the compose file, as do the other services. Run `up -d` again after any changes to the compose file.

**Using a domain name and TLS:**

For a production-like setup, place a reverse proxy in front of the adapters. Example with Caddy (auto-TLS via Let's Encrypt):

```
bap.yourdomain.com {
    reverse_proxy localhost:8081
}

bpp.yourdomain.com {
    reverse_proxy localhost:8082
}
```

Once your domain is live, update the routing configs (`generic-routing-BAPCaller.yaml`, `generic-routing-BAPReceiver.yaml`, `generic-routing-BPPCaller.yaml`, `generic-routing-BPPReceiver.yaml`) with the public URLs, and see [Customising the Starter Kit](#customising-the-starter-kit) for registering your own participant identity with the DeDi Registry.

---

## Your First Transaction

With the stack running, import the Postman collections and walk through a beckn transaction.

**Import the collections**

1. Open Postman and click **Import**.
2. Navigate to `starter-kit/generic-devkit/postman/`.
3. Import both files:
   - `BAPBecknStarterKit.postman_collection.json` — the buyer side, 4 requests: `discover`, `select`, `init`, `confirm`
   - `BPPBecknStarterKit.postman_collection.json` — the provider side, 4 requests: `on_select`, `on_init`, `on_confirm`, and `publish`

**Set the collection variables**

Each collection has a `bap_adapter_url` / `bpp_adapter_url` variable that tells Postman where the ONIX adapters are reachable. For local deployment:

- `bap_adapter_url` → `http://localhost:8081/bap/caller`
- `bpp_adapter_url` → `http://localhost:8082/bpp/caller`

For a VPS, replace `localhost` with your server's IP or domain.

**Run the transaction flow**

Use the **BAP collection**, in order:

```
discover  →  select  →  init  →  confirm
```

`status`, `track`, `update`, `cancel`, `rate`, and `support` are not yet implemented in these collections. `discover` does **not** require you to have published anything — it's served from whatever a crawler has already indexed across the network, independent of your own publishing activity.

The `on_*` callbacks (`on_discover`, `on_select`, `on_init`, `on_confirm`) are sent asynchronously and received automatically by the sandbox applications — you do not need to trigger them manually. Use the **BPP collection** for two things outside this automatic flow:

- `on_select`/`on_init`/`on_confirm` — simulate a BPP-initiated callback directly (e.g. an unsolicited one), rather than one triggered by the matching BAP request.
- `publish` — see [Catalog Publishing](#catalog-publishing-catalogpublish) below, if you want your own catalog to become discoverable.

After each step, check the `onix-bap` and `onix-bpp` logs to see the message being processed.

---

## Catalog Publishing (`catalog/publish`)

`onix-bpp` exposes `/catalog/publish` — a DS-internal trigger that publishes one or more plain Beckn Catalog objects: it diffs each against what was last published (producing a fresh baseline, an incremental change file, or a no-op), signs the result, and writes a manifest + catalog index under the handler's `outputRoot` (`/beckn` in the container, `generic-devkit/data/beckn` on the host — see "Where the files get written" below). This is an unsigned, same-operator call, **not** the full signed, async `catalog/publish` Beckn action beckn.yaml describes (context/action envelope, routing, `on_publish` callback) — that is a materially larger scope this devkit does not implement yet. See [beckn-onix's catalogpublisher README](https://github.com/beckn/beckn-onix/blob/catalog-publisher/pkg/plugin/implementation/catalogpublisher/README.md) for the full design background.

### Prerequisite for your published catalogs to be discoverable

> **TBD.** Running `catalog/publish` writes files locally, but a crawler only picks them up once your node is a member of a networkId with a live DeDi registry entry pointing `meta.catalog_index_urls` at your published index. This is not needed to use `discover` itself (it already serves catalogs already indexed from other sources) — it only matters if you want catalogs you publish here to show up in `discover` results. The DeDi registration steps are not documented yet.

### Trigger it

Use the **`publish`** request under the BPP collection's **`2 — Catalog Publishing`** folder, or call it directly:

```bash
curl -X POST http://localhost:8082/catalog/publish \
  -H "Content-Type: application/json" \
  -d '{
    "context": { "action": "catalog/publish" },
    "message": {
      "catalogs": [
        { "id": "staging.p-node.fabric.nfh.global/CAT-GENERIC-001", "descriptor": { "name": "Generic Catalog" }, "provider": { "id": "PROV-EXAMPLE-01" }, "resources": [ /* ... */ ] }
      ],
      "publishDirectives": [
        { "catalogId": "staging.p-node.fabric.nfh.global/CAT-GENERIC-001", "visibleTo": ["beckn.one/testnet", "nfh.global/testnet"], "catalogType": "REGULAR" }
      ]
    }
  }'
```

Request body matches beckn.yaml's real `CatalogPublishAction` envelope shape (`context`/`message.catalogs[]`/`message.publishDirectives[]`) — `context` only carries `action` since every other `Context` field is optional and none are meaningful for this unsigned, same-operator call. `publishDirectives[]` entries are matched to a catalog by `catalogId`; `catalogType` (`MASTER`/`REGULAR`) is required by the spec, and `visibleTo` restricts which networks may fetch that catalog (empty/omitted means public) — both map straight onto the same-named fields in the published catalog index. Each catalog's own top-level `"id"` is used verbatim as its catalogId — it is not derived from a domain, so submit the full id you want published. `retire` (a list of catalogIds) and `forceBaseline` (bypass diffing, publish a fresh baseline) are this handler's own additions with no beckn.yaml equivalent — accepted as siblings of `context`/`message`, alongside or instead of `message.catalogs`.

### Sample response

```json
{
  "status": "COMPLETED",
  "results": [
    { "catalogId": "staging.p-node.fabric.nfh.global/CAT-GENERIC-001", "status": "ACCEPTED", "version": 1 }
  ]
}
```

`status` is always present (`COMPLETED`/`FAILED` for the call as a whole); each entry in `results` borrows beckn.yaml's `CatalogProcessingResult` vocabulary (`ACCEPTED`/`REJECTED`) per catalog — a bad submission (e.g. missing `id`) is `REJECTED` with a `reason`, without failing the rest of the batch:

```json
{
  "status": "COMPLETED",
  "results": [
    { "catalogId": "", "status": "REJECTED", "reason": "missing catalogId" }
  ]
}
```

A fatal failure (e.g. signing failure) returns `200` with `status: FAILED` and an `error` object instead. Publishing the same catalogId again with edited `resources`/`offers` produces an incremental change file and bumps its version instead of a fresh baseline; publishing it unchanged is a no-op. Inspect `generic-devkit/data/beckn/` on the host to see the generated manifest, catalog index, and versioned catalog files directly.

### Where the files get written

`catalogPublish`'s `outputRoot: /beckn` (set in `generic-bpp.yaml`) is bind-mounted to `generic-devkit/data/beckn/` on the host (`docker-compose-generic.yml`), so every published catalog lands there directly — no need to exec into the container to see it:

```
generic-devkit/data/beckn/
  index/
    becknCatalogs.index.json          # the catalog index -- one entry per catalogId, with baseline/changes/latest pointers
  catalogs/
    <localName>.v<version>.json.gz    # a baseline (e.g. CAT-GENERIC-001.v1.json.gz)
    <localName>.latest.json.gz        # overwritten-in-place pointer at the current version (publishLatest, default true)
    changes/
      <localName>.v<version>.changes.json.gz   # an incremental change file (e.g. CAT-GENERIC-001.v2.changes.json.gz)
```

`<localName>` is the catalog's `catalogId` with any `domain/` prefix stripped. Files are `.gz` by default (`gzip: true` in `catalogPublisher`'s config) — `digest`/`size` in the index are always computed against the decompressed content regardless.

The index's `baseline`/`changes[]`/`latest` entries carry full URLs, not local paths — each is `catalogBaseURL` (also set in `generic-bpp.yaml`, e.g. `https://your-tunnel.ngrok-free.dev/beckn`) plus the file's path under `outputRoot`. `catalogBaseURL` must match wherever `outputRoot` is actually being served from publicly (`beckn-router`'s Caddy `/beckn/*` route over your ngrok tunnel) — update it if your ngrok domain changes, or the URLs a crawler tries to fetch will 404.

### Migrating from the old catalog/publish API to the decentralized catalog

If you're publishing catalogs today via `catalog/publish` with ACK/NACK
responses, subscription CRUD (`catalog/subscription`), or a central
Cataloging Service, this section is for you. The model this plugin
implements is a different shape entirely: you publish plain files to your
own storage, and DeDi + a crawler do the rest. Nothing about your actual
catalog *content* (the `Catalog`, `Resource`, `Offer` schemas) changes --
what changes is how it gets from you to a Discovery Service.

#### The conceptual shift

**Before:** you called a network API (`catalog/publish`) and got an
ACK/NACK back. A central Cataloging Service stored your catalog, handled
subscriptions, and served `catalog/pull`/`catalog/search` to consumers.

**Now:** you publish immutable JSON files to storage you already control
(any CDN, object store, or static host) via this plugin's `Publish` call,
exposed here as a DS-internal `catalog/publish` trigger with no ACK/NACK
envelope at all -- see "Trigger it" above. Once your files are on your
storage and your DeDi record's `meta.catalog_index_urls` (a list of
`{url}` entries, per NFH-014 CON-TBD-33 -- a node may host more than one
catalog index) points at your index, crawlers discover and pull your
catalogs on their own schedule. There is no central service to call,
subscribe to, or wait on.

#### What you need to do

Short version: **pick some storage, call `catalog/publish` against your
own adapter instead of a central service, and set one field on a record
you already have.** That's the whole migration -- there's no server to
stand up, no subscription list to manage, and no ACK/NACK handshake to
get right.

1. **Pick storage you already have.** Any static host works -- S3, a CDN,
   GitHub Pages, even an ngrok tunnel for local testing. You're not
   building a new service; you're pointing this plugin at a folder.
2. **Call `catalog/publish` -- but against your own adapter, not a
   central Cataloging Service.** The request body (your catalog JSON) is
   unchanged, but the endpoint you hit is now this DS-internal,
   same-operator trigger on your own node instead of a network call to
   someone else's service, and there's no ACK/NACK to parse in response:
   a synchronous call returns the catalog files and index, ready to
   upload. No MERGE/FULL mode to pick either -- the plugin looks at what
   you last published and figures out on its own whether this is a fresh
   baseline or an incremental change; a resubmission of identical content
   is simply a no-op.
3. **Set one field on your existing DeDi Subscriber record:
   `meta.catalog_index_urls`** (a list of `{url}` entries, not a single
   string) -- that's the entire "registration" step. No separate
   pointer file, no new registry to onboard into. The plugin
   can even check this for you after every publish and warn you if it's
   missing (see the [beckn-onix catalogpublisher README](https://github.com/beckn/beckn-onix/blob/catalog-publisher/pkg/plugin/implementation/catalogpublisher/README.md) for the "Optional registry catalog-index link check").

Everything else -- subscriptions, restricted-catalog auth, a central
Cataloging Service, waiting on callbacks -- simply isn't part of this
model anymore, so there's nothing to configure for it, only things to
delete from your existing integration (see "What you no longer need,"
below).

#### What you no longer need

- **A `catalog/publish` call to a shared, network-facing Cataloging
  Service, with an ACK/NACK response.** You still call `catalog/publish`
  -- but it's now a DS-internal, same-operator trigger on your own
  adapter, not a network call to someone else's service, and it responds
  synchronously with your catalog files and index instead of an ACK/NACK
  envelope.
- **`catalog/subscription` CRUD.** A crawler's scope is its own
  configuration now -- you don't manage subscriber lists.
- **`catalog/search`.** Removed from the publish/pull surface; a
  Discovery Service may still offer search over its own store, but
  that's not something you interact with as a publisher.
- **`catalog/push`/`/on_pull` callbacks.** Consolidated into the crawler
  pulling from you and pushing into the Discovery Service's own `/push`
  -- you never receive a callback for this.
- **Restricted catalogs, download gates, `authMethods`.** Catalogs are
  public, unconditionally, in this design. If you relied on
  `publishDirectives.visibleTo` as an access gate, note that its
  replacement (`networkIds` in the index) is a **relevance filter only**,
  never an access control -- anyone with a file's URL can fetch it.

#### Field-by-field mapping

| Old (CATALG / DISCOVR) | New |
| :---- | :---- |
| `catalog/publish` with ACK/NACK | Files saved to storage; validation happens up front, results in a feedback log |
| `publishDirectives.visibleTo` | Per-catalog `networkIds` in the index -- relevance filter, not access gate |
| `publishDirectives.updateMode: MERGE` | A change file (id-keyed upserts/removals) |
| `publishDirectives.updateMode: FULL` | A fresh baseline |
| `catalog/pull`, mode FULL | The baseline file |
| `catalog/pull`, mode DELTA | Change files after the crawler's cursor |
| `downloadManifest` (sha256, sizeBytes) | `digest`/`size` in the index, verified against each self-signed file |
| Subscription filters (`networkIds`, `schemaTypes`) | Crawler-side filtering on the index |
| Subscription CRUD (`catalog/subscription`) | Not needed -- a crawler's scope is its own config |
| `catalog/search` | Removed from this surface |
| `catalog/push` | Crawler pull, with an optional change signal as an accelerator |
| `/on_pull` callback | Consolidated into the Discovery Service's internal `/push` |
| `subscriberId` | `nodeId`, a domain |
| Restricted catalogs / download gate / `authMethods` | **Removed.** Catalogs are public-only; no per-catalog auth exists |
| Offer-only catalogs, query-time attachment | Unchanged -- still lives behind `/discover` |

#### What stays exactly the same

- Your `Catalog`/`Resource`/`Offer` JSON content and its schema.
- `catalogType: MASTER`/`REGULAR` and `resourceDirectives[].extends` --
  unchanged, just resolved by the Discovery Service at index time instead
  of centrally at publish time.
- Offer-only catalogs and query-time attachment behind `/discover`.

---

## How It All Works

Now that you have seen the network in action, here is a deeper look at how the pieces fit together.

### The Services in This Stack

```
  ┌──────────────────────────────────────────────────────────────┐
  │                      Your Environment                        │
  │                                                              │
  │  ┌──────────────┐     ┌────────────────────────────────────┐ │
  │  │  sandbox-bap │◄───►│  onix-bap  (port 8081)             │ │
  │  │  BAP app     │     │  BAP-side ONIX adapter             │ │
  │  └──────────────┘     │  /bap/caller/   /bap/receiver/     │ │
  │                       └──────────────────────┬─────────────┘ │
  │                                              │               │
  │  ┌──────────────┐     ┌────────────────────────────────────┐ │
  │  │  sandbox-bpp │◄───►│  onix-bpp  (port 8082)             │ │
  │  │  BPP app     │     │  BPP-side ONIX adapter             │ │
  │  └──────────────┘     │  /bpp/receiver/  /bpp/caller/      │ │
  │                       │  /catalog/publish (writes to       │ │
  │                       │   ../data/beckn, served publicly   │ │
  │                       │   via beckn-router)                │ │
  │                       └──────────────────────┬─────────────┘ │
  │                                              │               │
  │  ┌──────────────────────────────────────┐    │               │
  │  │  redis  (shared cache, port 6379)    │    │               │
  │  └──────────────────────────────────────┘    │               │
  └─────────────────────────────────────────────┼───────────────┘
                                                │
                           ┌────────────────────┴──────────────────┐
                           ▼                                       ▼
          ┌─────────────────────────────┐     ┌─────────────────────────────┐
          │        Beckn Fabric         │     │      Discovery Service      │
          │                             │     │   (independent service,     │
          │  DeDi Registry              │     │    crawler-fed)             │
          │  · Identity lookups         │     │                             │
          │  · Dynamic routing          │     │  Receives discover from     │
          │    (resolves BAP/BPP URIs)  │     │  BAPs, serves results from  │
          │  · Holds each node's        │     │  whatever a crawler has     │
          │    catalog_index_urls       │     │  already indexed, returns   │
          │                             │     │  on_discover                │
          └─────────────────────────────┘     └─────────────────────────────┘
                           ▲
                           │ (independently, on its own schedule)
                           │
          A crawler reads catalog_index_urls from the DeDi Registry
          and pulls each node's published catalog files directly --
          no call from the publishing BPP to the Discovery Service.
```

| Service | Image | Port | Role |
|---------|-------|------|------|
| `beckn-router` | `caddy:alpine` | 9000 | Reverse proxy — single entry point for both adapters and published catalog files |
| `onix-bap` | `fidedocker/onix-adapter` | 8081 | BAP-side protocol adapter |
| `onix-bpp` | `fidedocker/catalog-publisher` | 8082 | BPP-side protocol adapter, includes the `catalogpublisher` plugin |
| `sandbox-bap` | `fidedocker/sandbox-2.0` | 3001 | Simulates a BAP application |
| `sandbox-bpp` | `fidedocker/sandbox-2.0` | 3002 | Simulates a BPP application |
| `redis` | `redis:alpine` | 6379 | Shared request/response cache |

**The ONIX adapter** (`fidedocker/onix-adapter` / `fidedocker/catalog-publisher`) is the core middleware from the [beckn-onix](https://github.com/beckn/beckn-onix) project. It is a plugin-based Go server that handles signing, signature validation, schema validation, and routing for every beckn message. Both `onix-bap` and `onix-bpp` run the same underlying binary — their behaviour is entirely determined by their config files and which plugins are compiled in.

**The application simulator** (`fidedocker/sandbox-2.0`) is a generic beckn application simulator. It exposes simple HTTP endpoints that receive forwarded messages and generate appropriate responses, so you can observe the full protocol flow without building your own BAP or BPP app yet.

### Beckn Fabric: Registry and Discovery

**Beckn Fabric** is the shared infrastructure layer that makes an open beckn network possible. Without Fabric, each participant would need bilateral agreements with every other participant. Fabric removes that requirement by providing shared, trustless services that the whole network relies on. This starter kit uses one Beckn Fabric service — the **DeDi Registry** — plus an independent, crawler-fed Discovery Service that runs alongside but outside of Fabric.

**DeDi Registry** (`fabric.nfh.global/registry/dedi`)

The DeDi (Decentralised Discovery) Registry is the source of truth for participant identity on the network. Every BAP and BPP registers their network ID, public key, and callback URI with the registry. The ONIX adapter consults the registry for two things:

- **Signature validation** — when an inbound message arrives, the adapter looks up the sender's public key in the registry to verify the digital signature. If the key is not found or the signature is invalid, the message is rejected.
- **Routing** — when a message needs to reach a BPP (e.g., `select`) or return to a BAP (e.g., `on_select`), the adapter performs a registry lookup by participant ID to resolve the correct callback URL. This is how the network routes messages without any static config between participants.

The registry is referenced in all four adapter config files under the `registry` plugin:
```yaml
registry:
  id: dediregistry
  config:
    url: https://fabric.nfh.global/registry/dedi
    registryName: subscribers.beckn.one
```

The same registry record is also where a BPP points crawlers at its own catalog — see `meta.catalog_index_urls` in [Catalog Publishing](#catalog-publishing-catalogpublish).

**Discovery Service** (crawler-fed)

The Discovery Service is an independent network service — not part of Beckn Fabric — that acts as the search engine for the network. It does **not** receive a direct call from a BPP's `publish`. Instead, a crawler reads `meta.catalog_index_urls` from each DeDi-registered node's record, independently fetches and verifies that node's published catalog files, and feeds the result into the Discovery Service's own index on its own schedule. When a BAP sends a `discover` request, the adapter routes it to the Discovery Service, which serves results from whatever the crawler has already indexed and responds asynchronously with `on_discover`. The BAP never contacts a BPP directly during discovery — direct BAP-to-BPP communication only begins at `select`.

The Discovery Service endpoint is configured in `generic-routing-BAPCaller.yaml`:
```yaml
target:
  url: "https://<discovery-service>/beckn"
endpoints:
  - discover
```

### The Full Transaction Flow

Here is the complete picture of how a beckn transaction flows through the network:

```
BPP side — catalog publishing (independent of any BAP transaction, optional)
─────────────────────────────────────────────────────────────────

sandbox-bpp ──publish──► onix-bpp ──writes files──► ../data/beckn
                                                          (served publicly via beckn-router)
sandbox-bpp ◄──synchronous response── onix-bpp
(no on_publish callback -- this is a same-operator, unsigned call)

                                                          │
                                          (independently, on its own schedule)
                                                          ▼
                                       A crawler reads this node's DeDi registry
                                       entry, finds catalog_index_urls, and pulls
                                       the published catalog files directly.


BAP side — discovery
─────────────────────────────────────────────────────────────────

sandbox-bap ──discover──► onix-bap ──discover──► Discovery Service
                                                    │
                                    (serves from whatever a crawler
                                     has already indexed, from any
                                     publishing node -- not only this one)
                                                    │
sandbox-bap ◄──on_discover── onix-bap ◄──on_discover───┘
(BAP receives list of matching offerings)


BAP ↔ BPP — transaction (select / init / confirm)
─────────────────────────────────────────────────────────────────

sandbox-bap ──select──► onix-bap
                        │ DeDi Registry resolves BPP URI
                        ▼
                    onix-bpp ──► sandbox-bpp
                        │
                    sandbox-bpp sends on_select response
                        │
                    onix-bpp
                        │ DeDi Registry resolves BAP URI
                        ▼
sandbox-bap ◄──on_select── onix-bap

     ... same pattern repeats for init / confirm ...
```

The key insight: **discovery is served from whatever a crawler has already indexed across the network**, independent of when or whether *this* BPP has published anything, and **post-discovery transactions flow directly between BAP and BPP** with the DeDi Registry providing dynamic routing. Publishing your own catalog (optional) makes it discoverable on the crawler's own schedule, asynchronously — it is never a prerequisite for `discover` to return results from other nodes.

### Tracing a Request End-to-End

Here is the step-by-step journey of a single `discover` call, showing what happens inside each service:

**1. Postman → onix-bap**
You trigger a `discover` by POSTing to onix-bap's caller endpoint (`/bap/caller/discover`). The `bapTxnCaller` module picks it up.

**2. addRoute** — the adapter reads `generic-routing-BAPCaller.yaml` and matches the `discover` action to the configured Discovery Service URL.

**3. sign** — the adapter signs the request body using the Ed25519 private key defined under `keyManager` in `generic-bap.yaml` (identity: `bap.example.com`).

**4. validateSchema** — the Beckn v2.0.0 OpenAPI spec is fetched from GitHub (cached for 1 hour) and the message is validated against it.

**5. Request → Discovery Service** — the signed, validated `discover` message is forwarded to the Discovery Service.

**6. Discovery Service → on_discover callback** — the Discovery Service searches its crawler-built catalog index and asynchronously POSTs `on_discover` back to onix-bap's receiver endpoint (`/bap/receiver/on_discover`).

**7. validateSign** — the `bapTxnReceiver` module verifies the Discovery Service's digital signature by looking up its public key in the DeDi Registry.

**8. addRoute** — reads `generic-routing-BAPReceiver.yaml`, which routes all `on_*` callbacks to the sandbox-bap webhook.

**9. sandbox-bap receives on_discover** — the response (list of available offerings) is now stored in the application and visible in logs.

For `select`, `init`, and `confirm`, the flow is similar but the adapter uses the DeDi Registry to resolve the BPP's URI dynamically (no hardcoded URL needed), and onix-bpp uses the registry to resolve the BAP's callback URI for the `on_*` responses.

---

## Repository Structure

```
starter-kit/
│
└── generic-devkit/
    │
    ├── config/                               # All adapter configuration
    │   ├── generic-bap.yaml                  # BAP adapter: modules, plugins, keys
    │   ├── generic-bpp.yaml                  # BPP adapter: modules, plugins, keys (incl. catalogPublish)
    │   ├── generic-routing-BAPCaller.yaml    # Where BAP sends outbound requests
    │   ├── generic-routing-BAPReceiver.yaml  # Where BAP delivers incoming callbacks
    │   ├── generic-routing-BPPCaller.yaml    # Where BPP sends outbound responses
    │   └── generic-routing-BPPReceiver.yaml  # Where BPP delivers incoming requests
    │
    ├── data/beckn/                            # catalog/publish output (see Catalog Publishing)
    │
    ├── install/
    │   ├── docker-compose-generic.yml        # Main compose file (pre-built adapter images)
    │   └── docker-compose-generic-local.yml  # Alternate compose file (locally built image)
    │
    └── postman/
        ├── BAPBecknStarterKit.postman_collection.json   # BAP-side flows
        └── BPPBecknStarterKit.postman_collection.json   # BPP-side flows
```

### config/ — How Each File Is Used

Each ONIX adapter instance loads one primary config file, which in turn references routing config files for each module.

**`generic-bap.yaml`** — loaded by onix-bap. Defines two modules:

- `bapTxnCaller` at `/bap/caller/` handles **outbound** BAP requests. It signs each message, routes it using `generic-routing-BAPCaller.yaml`, and validates the schema.
- `bapTxnReceiver` at `/bap/receiver/` handles **inbound** `on_*` callbacks. It validates the sender's signature (via DeDi Registry lookup), routes the response to sandbox-bap using `generic-routing-BAPReceiver.yaml`, and validates the schema.

**`generic-bpp.yaml`** — loaded by onix-bpp. Defines three modules:

- `bppTxnReceiver` at `/bpp/receiver/` handles **inbound** action requests from BAPs. It validates signatures and routes to sandbox-bpp using `generic-routing-BPPReceiver.yaml`.
- `bppTxnCaller` at `/bpp/caller/` handles **outbound** `on_*` responses. It signs messages and routes them using `generic-routing-BPPCaller.yaml`.
- `catalogPublish` at `/catalog/publish` — a separate module, **not** part of `bppTxnCaller` — handles `catalog/publish` directly: unsigned, same-operator, no routing config involved. See [Catalog Publishing](#catalog-publishing-catalogpublish).

**`generic-routing-BAPCaller.yaml`** — outbound routing for the BAP:
- `discover` → routes to the Discovery Service
- Transaction actions (`select`, `init`, `confirm`, plus not-yet-implemented `status`, `track`, `update`, `cancel`, `rate`, `support`) → `targetType: bpp`, resolved dynamically via DeDi Registry lookup

**`generic-routing-BAPReceiver.yaml`** — all inbound `on_*` callbacks are routed to sandbox-bap's webhook endpoint.

**`generic-routing-BPPReceiver.yaml`** — all inbound action requests are routed to sandbox-bpp's webhook endpoint.

**`generic-routing-BPPCaller.yaml`** — outbound routing for the BPP's `on_*` responses → `targetType: bap`, resolved dynamically via DeDi Registry lookup. `catalog/publish` does not route through this file at all — it's handled entirely by the separate `catalogPublish` module above.

Both config files embed a **`keyManager`** section with pre-generated Ed25519 key pairs for testnet participants `bap.example.com` and `bpp.example.com`. These are registered with the DeDi Registry on `beckn.one/testnet` and work out of the box — no changes needed to get started.

### postman/ — What the Collections Do

**`BAPBecknStarterKit.postman_collection.json`** — the buyer-side flows, organised in two folders:

- **1 — Discovery:** `discover` — sent to onix-bap (`/bap/caller/discover`), which routes it to the Discovery Service.
- **2 — Transaction:** `select` → `init` → `confirm` — sent to onix-bap, which routes each to the BPP via DeDi Registry lookup.

**`BPPBecknStarterKit.postman_collection.json`** — the provider-side flows, organised in two folders:

- **1 — Transaction:** `on_select`, `on_init`, `on_confirm` — sent to onix-bpp (`/bpp/caller/on_*`). Use these to manually simulate or inspect BPP responses. In a normal flow sandbox-bpp handles these automatically.
- **2 — Catalog Publishing:** `publish` — sent to onix-bpp at `/catalog/publish` directly (**not** `/bpp/caller/publish` — this is the separate `catalogPublish` module, see [Catalog Publishing](#catalog-publishing-catalogpublish)).

---

## Customising the Starter Kit

**Changing the Discovery Service endpoint**

Edit `config/generic-routing-BAPCaller.yaml` to point `discover` at a different Discovery Service.

**Running fully offline (no external dependencies)**

Change the `discover` target in `generic-routing-BAPCaller.yaml` to route directly to onix-bpp's receiver endpoint (`http://onix-bpp:8082/bpp/receiver/`). You will also need to remove or stub the DeDi Registry lookups for the routing to work without network access.

**Using your own participant identity**

The config files contain pre-generated testnet key pairs for `bap.example.com` and `bpp.example.com`, registered on `beckn.one/testnet`. To use your own identity:

1. Generate a new Ed25519 key pair.
2. Register your `networkParticipant` domain and public key with the DeDi Registry at `fabric.nfh.global/registry/dedi`.
3. Update the `keyManager` section in `generic-bap.yaml` and `generic-bpp.yaml` with your new participant ID, key ID, and key material.
4. Update the routing config files to use your `networkId` in place of `beckn.one/testnet`.
5. If you want your own published catalogs to be discoverable, see the (currently TBD) registry prerequisite in [Catalog Publishing](#catalog-publishing-catalogpublish).

**Replacing the application simulator with your own application**

The application simulator containers (`sandbox-bap` and `sandbox-bpp`) are simple simulators. Replace either with your own application by updating the service image and webhook URLs in the compose file and routing configs. Your application needs to accept POSTed beckn messages at the configured endpoints and be on the same Docker network.

**Using a different beckn domain**

Update the `domain` (or `networkId`) field in all four routing YAML files to match your domain. Point the `schemav2validator` in the adapter config to the appropriate OpenAPI spec URL for schema validation.

---

## Troubleshooting

**Containers fail to start**

Check for port conflicts on 8081, 8082, 9000, 3001, 3002, or 6379:

```shell
docker compose -f docker-compose-generic.yml logs
```

**sandbox-bap or sandbox-bpp stays in `starting` state**

The application containers health-check at `/api/health`. If they don't reach `healthy` within about a minute, inspect their logs:

```shell
docker compose -f docker-compose-generic.yml logs sandbox-bap
docker compose -f docker-compose-generic.yml logs sandbox-bpp
```

**Postman requests return connection errors**

Confirm the stack is fully up (`docker compose ps` shows all six containers as `running` or `healthy`) and that your Postman collection variables point to the right host and port.

**Signature validation failures (`validateSign` errors in logs)**

The most common cause is clock skew. Beckn signatures embed a timestamp with a short validity window. On Linux: `timedatectl` to check clock sync. On Docker Desktop, ensure the VM clock is synchronised.

**`discover` returns no results**

The Discovery Service must be reachable. Check the URL configured in `generic-routing-BAPCaller.yaml` and test connectivity with:

```shell
curl -s https://<discovery-service>/beckn
```

If it times out, the testnet may be temporarily unavailable, or your network may block outbound HTTPS. `discover` does **not** depend on you having called `publish` — it's served from whatever a crawler has already indexed across the network. If you expected to see your *own* published catalog specifically, check that your DeDi registry entry actually carries `meta.catalog_index_urls` (see [Catalog Publishing](#catalog-publishing-catalogpublish)) — a crawler has no other way to find it, and there's no immediate feedback if that field is missing.

**`catalog/publish` not working**

There is no `on_publish` callback in this model — `catalog/publish` responds synchronously. Check the response body directly: `status: FAILED` with an `error` object means a fatal failure (e.g. signing); a `REJECTED` entry in `results[]` means that specific catalog was invalid (check its `reason`). Also verify the BPP's signing keys are valid by reviewing the `keyManager` section in `generic-bpp.yaml`, and that `outputRoot`/`catalogBaseURL` are set correctly (see [Catalog Publishing](#catalog-publishing-catalogpublish)).

**Images fail to pull**

Ensure Docker has at least 2 GB RAM allocated and a stable internet connection for pulling the required images.

**Stopping and cleaning up**

```shell
# Stop containers
docker compose -f docker-compose-generic.yml down

# Stop and remove volumes and image cache
docker compose -f docker-compose-generic.yml down -v
```
