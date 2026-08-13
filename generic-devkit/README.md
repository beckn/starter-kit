# Generic Devkit — Beckn Protocol v2.0.0

Goal of this devkit is to enable any developer to run a **usecase-agnostic Beckn v2.0.0 round-trip** on their local machine within a few minutes — no domain knowledge of EV charging, retail, or any other sector required.

It currently covers the front half of the transaction lifecycle, plus catalog publishing:

```
discover → on_discover
select → on_select → init → on_init → confirm → on_confirm
(+ catalog/publish -- a DS-internal trigger, not part of the signed transaction flow above)
```

`status`/`on_status`, `track`/`on_track`, `update`/`on_update`, `cancel`/`on_cancel`,
`rate`/`on_rate`, and `support`/`on_support` are not yet implemented in this devkit's
Postman collections.

---

## Prerequisites

1. [Docker Desktop](https://www.docker.com/products/docker-desktop) — installed and running
2. [Git](https://git-scm.com/downloads) — on your system path
3. [Postman](https://www.postman.com/downloads/) — for sending API calls

---

## Quick Start

### 1. Clone and navigate

```bash
git clone https://github.com/beckn/starter-kit.git
cd starter-kit/generic-devkit/install
```

### 2. Start containers

```bash
docker compose -f docker-compose-generic.yml up -d
docker ps
```

You should see six containers running:

| Container | Port | Role |
|---|---|---|
| `beckn-router` | 9000 | Caddy reverse proxy — single entry point, routes `/bap/*` → `onix-bap`, `/bpp/*` → `onix-bpp` |
| `redis` | 6379 | Shared cache |
| `onix-bap` | 8081 | BAP adapter (caller + receiver) |
| `onix-bpp` | 8082 | BPP adapter (caller + receiver) |
| `sandbox-bap` | 3001 | Mock BAP application (receives callbacks) |
| `sandbox-bpp` | 3002 | Mock BPP application (processes requests) |

### 3. Import Postman collections

Open Postman → **Import** → select the entire folder:

```
generic-devkit/postman/
```

This imports two collections:

- **BAP Beckn Starter Kit** — 4 outbound requests sent by the BAP
- **BPP Beckn Starter Kit** — 4 requests: 3 BPP-initiated callbacks + `catalog/publish`

### 4. Run the flow

Use the **BAP collection** to drive the transaction lifecycle in order:

| Step | Action | Folder |
|---|---|---|
| 1 | `discover` | Discovery |
| 2 | `select` | Transaction |
| 3 | `init` | Transaction |
| 4 | `confirm` | Transaction |

`status`, `track`, `update`, `cancel`, `rate`, and `support` are not yet
implemented in this devkit's collections (see the note at the top of this
README).

Each request returns an `ACK`. The corresponding `on_*` callback from the BPP arrives at `sandbox-bap` and can be viewed in the BAP logs:

```bash
docker logs -f onix-bap
docker logs -f sandbox-bap
```

Use the **BPP collection** for two things outside the automatic BAP-driven flow above:

- **`1 — Transaction`** — simulate BPP-initiated callbacks directly (`on_select`, `on_init`, `on_confirm`), e.g. to test an unsolicited/out-of-band callback rather than one triggered by the matching BAP request.
- **`2 — Catalog Publishing`** — trigger `catalog/publish` (see [Catalog Publisher](#catalog-publisher-catalogpublish) below). This is new: if you are publishing catalogs, use this flow. Publishing is not a prerequisite for `discover` to work -- `discover` already serves catalogs a crawler has indexed from other sources, whether or not you've published anything yourself.

---

## Catalog Publisher (`catalog/publish`)

`onix-bpp` exposes `/catalog/publish` — a DS-internal trigger that publishes one or more plain Beckn Catalog objects: it diffs each against what was last published (producing a fresh baseline, an incremental change file, or a no-op), signs the result, and writes a manifest + catalog index under the handler's `outputRoot` (`/beckn` in the container, `generic-devkit/data/beckn` on the host — see `docker-compose-generic-local.yml`). This is an unsigned, same-operator call, **not** the full signed, async `catalog/publish` Beckn action beckn.yaml describes (context/action envelope, routing, `on_publish` callback) — that is a materially larger scope this devkit does not implement yet. See [beckn-onix's catalogpublisher README](https://github.com/beckn/beckn-onix/blob/catalog-publisher/pkg/plugin/implementation/catalogpublisher/README.md) for the full design background.

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

The devkit ships with **testnet sandbox credentials** pre-registered on `beckn.one/testnet` via the DeDi registry (`https://fabric.nfh.global/registry/dedi`).

To register your own subscriber IDs and keys, follow the [DeDi registration guide](https://developers.becknprotocol.io/) and update the `keyManager` block in `generic-bap.yaml` / `generic-bpp.yaml`.

### Schema Validation

Both adapters validate all payloads against the canonical Beckn v2.0.0 OpenAPI spec:

```
https://raw.githubusercontent.com/beckn/protocol-specifications-v2/refs/tags/core-v2.0.0-lts/api/v2.0.0/beckn.yaml
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
