# Beckn Discovr — Deployment & Configuration Reference

> **Why this lives here, temporarily.** The Discover service's source code
> (`catalog-discover-job`, `catalog-publish-job`, `response-dispatcher`) isn't open-sourced
> yet. Until it is, this directory hosts the full deployment stack, config, and this guide
> as part of the starter kit so anyone can stand up a working Discovr instance from public
> Docker Hub images alone. **Once the Discover source repos are open-sourced, this
> directory (and this document) will move to live alongside that source code**, and this
> location will be removed or redirected.

This document is for a DevOps/cloud engineer standing up the **Beckn Discovr** stack — a
catalog discovery service fronted by the **Onix adapter** — as a single docker-compose
deployment on a VM. It covers the full topology, every container, and every configuration
setting in `discovr-stack.yml` and `onix-discover/discover-adapter.yaml`.

All application images are published publicly on Docker Hub under the `fidedocker` org —
no registry authentication is required to pull them.

---

## 1. Architecture

```
                              Internet
                                 │
                     ┌───────────────────────┐
                     │  Nginx Proxy Manager   │  (ports 80/443, admin UI on 81)
                     │  (jc21/nginx-proxy-    │
                     │   manager)             │
                     └───────────┬───────────┘
                                 │ forwards /beckn/* to onix-discover:8080
                                 ▼
                     ┌───────────────────────┐
                     │      onix-discover     │  Beckn protocol gateway
                     │  (fidedocker/          │  — signature verification, schema
                     │   onix-crawler:1.0.0)  │    validation, signing, routing
                     │                        │  — 4 modules in one process:
                     │  /receiver/            │    discoverReceiver, discoverReceiverOnPull,
                     │  /receiver-on-pull/    │    discoverCaller, crawl
                     │  /caller/              │
                     │  /crawl                │
                     └───┬───────────────┬────┘
                         │               │
              validated  │               │  signs + forwards
              discover   │               │  on_discover to
              requests   │               │  context.bapUri
                         ▼               │
              ┌─────────────────────┐    │
              │ catalog-discover-job │    │
              │ (fidedocker/         │    │
              │  catalog-discover-   │    │
              │  job:v1.6.0)         │    │
              │ — runs the query,    │    │
              │   publishes result   │    │
              │   to Kafka           │    │
              └──────────┬──────────┘    │
                         │ Kafka topic     │
                         │ discovr.discover│
                         │ .out.responses  │
                         ▼                 │
              ┌─────────────────────┐    │
              │ response-dispatcher  │────┘  POST /caller (STATIC_CALLBACK_URL)
              │ (fidedocker/         │
              │  response-dispatcher:│
              │  v1.6.0)             │
              │ — consumes Kafka,     │
              │   posts on_discover   │
              │   to onix-discover    │
              └─────────────────────┘

              ┌─────────────────────┐
              │  catalog-publish     │  Receives catalog/push and catalog/on_pull
              │  (fidedocker/        │  from BPPs/Catalgs, indexes into ES + Postgres.
              │   catalog-publish-   │  Also the target of onix-discover's `crawl`
              │   job:v1.6.0)        │  module's CRAWLER_PUSH_ENDPOINT.
              └─────────────────────┘

Infra (all internal-only, 127.0.0.1-bound): Postgres+PostGIS, Elasticsearch, Kafka, Redis.
```

**Two request flows through `onix-discover`:**

- **Inbound (public)** — a BAP's `discover` request or a Catalg's `catalog/push` /
  `catalog/on_pull` hits `discoverReceiver` (or `discoverReceiverOnPull` for the
  schema-relaxed `on_pull` case), gets signature + schema validated, then routed
  internally to `catalog-discover-job` or `catalog-publish`.
- **Outbound (internal)** — `response-dispatcher` posts the query result to
  `discoverCaller`, which signs it and forwards to the requesting BAP's
  `context.bapUri` (read from the request body — no BAP URLs are hardcoded).
- **Crawl (internal, optional)** — the `crawl` module independently pulls catalog
  manifests from other network participants' DeDi index URLs and pushes verified
  results into `catalog-publish`. Disabled by leaving `CRAWLER_ENABLED: "false"`.

---

## 2. Images

| Service | Image | Notes |
|---|---|---|
| `catalog-publish` | `fidedocker/catalog-publish-job:v1.6.0` | Spring Boot. Ingests + indexes catalogs. |
| `catalog-discover-job` | `fidedocker/catalog-discover-job:v1.6.0` | Spring Boot. Runs discover queries. |
| `response-dispatcher` | `fidedocker/response-dispatcher:v1.6.0` | Spring Boot. Delivers `on_discover` callbacks. |
| `onix-discover` | `fidedocker/onix-crawler:1.0.0` | Go (Beckn-ONIX). **Not** the plain `fidedocker/onix-adapter` — this build carries the crawler plugin (module type `crawl`), which the stock adapter release doesn't have. |
| `postgres` | `postgis/postgis:15-3.3` | Shared by catalog-discover-job, catalog-publish, and the crawl module's own state tables. |
| `elasticsearch` | `docker.elastic.co/elasticsearch/elasticsearch:9.3.1` | Single-node, security disabled — internal-only. |
| `discovery-kafka` | `bitnamilegacy/kafka:3.9.0` | Single-broker KRaft mode. |
| `redis` | `redis:7-alpine` | Backs onix-discover's `cache` plugin (key/signature caching, payload correlation). |
| `nginx-proxy-manager` | `jc21/nginx-proxy-manager:latest` | Only public-facing entry point (ports 80/443/81). |

All four application images are digest-pinned in `discovr-stack.yml` — a `docker compose pull`
will always fetch the exact same bytes, even if the mutable tag is later re-pushed.

`onix-discover` runs under `platform: linux/amd64` — set this explicitly if deploying on an
ARM host (it'll run under emulation, which works but is slower to start).

---

## 3. Directory layout

```
discovr-stack.yml                       # this compose file
onix-discover/
  discover-adapter.yaml                 # Onix config — 4 modules (see §5)
  routing-discover-receiver.yaml        # inbound routing rules (discover → catalog-discover-job, etc.)
  routing-discover-caller.yaml          # outbound routing rules (on_discover → BAP via context.bapUri)
elasticsearch/
  es-index-template.json                # catalog index mapping, mounted into catalog-publish
  es-synonyms.txt                       # search synonym rules, mounted into elasticsearch
```

---

## 4. Prerequisites

| Item | Where it comes from |
|---|---|
| A registered **subscriber identity** on DeDi (subscriberId, e.g. `discover.example.com`) | Your network's DeDi registry |
| **Ed25519 signing keypair** + **X25519 encryption keypair**, base64-encoded | Generated and registered against the subscriberId above |
| A **DeDi key ID** for that keyset | Assigned when you register the keys |
| A **public base URL** for this deployment (DNS name or `<ip>.sslip.io`) | Your VM's public IP / DNS |
| A **Postgres password** — generate a strong one, don't use the default `discover_password` | — |
| Docker + docker compose installed on the VM | — |
| `docker network create beckn-network` run once | — |

No cloud registry auth is needed — all app images pull anonymously from `fidedocker` on
Docker Hub.

---

## 5. `discovr-stack.yml` — service-by-service config reference

### 5.1 Infrastructure services

#### `postgres` (container: `discovery-postgres`)
| Env var | Value | Purpose |
|---|---|---|
| `POSTGRES_DB` | `discover_db` | Database name used by catalog-discover-job / catalog-publish. |
| `POSTGRES_USER` | `discover_user` | App DB user. |
| `POSTGRES_PASSWORD` | `discover_password` (**change this**) | Must match every other service's Postgres password reference below. |
| `POSTGRESQL_POSTGRES_PASSWORD` | same as above | Superuser password (postgis image convention). |
| `POSTGRES_INITDB_ARGS` | `--encoding=UTF-8` | — |

Bound to `127.0.0.1:5434` — not reachable from the internet.

#### `elasticsearch` (container: `discovery-elasticsearch`)
Single-node, `xpack.security.enabled=false` (internal-network-only, no auth). Heap capped
at 512Mi via `ES_JAVA_OPTS`. Mounts `es-synonyms.txt` for search synonym rules. Bound to
`127.0.0.1:9200`.

#### `discovery-kafka` (container: `discovery-kafka`)
Single-broker KRaft-mode Kafka. Named `discovery-kafka` (not `kafka`) specifically to avoid
a DNS collision if you also run the separate Catalg/catalog stack on the same
`beckn-network`. Message size ceiling raised to 10 MiB
(`KAFKA_CFG_MESSAGE_MAX_BYTES` / `KAFKA_CFG_REPLICA_FETCH_MAX_BYTES`) to match the
producer-side `CATALOG_MAX_PAYLOAD_SIZE` used elsewhere in the stack — don't lower one
without lowering the other. Bound to `127.0.0.1:9093`.

#### `redis` (container: `discovery-redis`)
Backs onix-discover's `cache` plugin — key/signature caching and cross-module payload
correlation (see §5.5). Bound to `127.0.0.1:6379`.

### 5.2 `catalog-publish`

Handles inbound `catalog/push` and `catalog/on_pull`, indexes into Elasticsearch, tracks
Flyway-managed schema migrations against Postgres.

Key settings:
- `APP_DATASOURCE_URL` / `_USERNAME` / `APP_DATASOURCE_PASSWORD` / `DB_PASSWORD` — Postgres connection; **the password fields must all match `postgres`'s `POSTGRES_PASSWORD`**.
- `APP_MESSAGING_BROKER_SERVERS` — Kafka bootstrap (`discovery-kafka:29092`).
- `KAFKA_MAX_REQUEST_SIZE` / `CATALOG_MAX_PAYLOAD_SIZE` (10 MiB) — must stay ≤ the broker's `KAFKA_CFG_MESSAGE_MAX_BYTES`.
- `APP_CATALOG_VALIDATION_ENABLED=false` — schema validation for published catalogs is done upstream by onix-discover, not here.
- `BECKN_PROTOCOL_API_SCHEMA_URL` — fetched once at startup and cached (`SCHEMA_CACHE_TTL_HOURS`); requires outbound internet access to `raw.githubusercontent.com`. See §8 (Troubleshooting) if this fetch fails.
- `SIGNATURE_AUTH_ENABLED=false` — inbound signature verification is done by onix-discover in front; don't enable this here too (double verification / header-stripping mismatches).
- `APP_CATALOG_ELASTICSEARCH_*` — ES connection, index naming, retry/backoff, and `APP_CATALOG_ELASTICSEARCH_DEFAULT_SCHEMA_TYPE=GenericResource`: resources published **without** a `schemaType` land in the `beckn-catalog-genericresource` fallback index instead of being silently dropped.
- `APP_CATALOG_PULL_*` — hardening for the `catalog/on_pull` download path (size caps, DNS-rebinding protection via a short-TTL positive-resolution cache, timeouts, dedicated executor pool).
- `ES_MAPPING_TEMPLATE_FILE=/config/es-index-template.json` — mounted from `./elasticsearch/es-index-template.json`.

Healthcheck hits `/actuator/health` (not under `/beckn` — Spring's actuator endpoints are excluded from the Beckn API prefix).

### 5.3 `catalog-discover-job`

Runs the actual discover query against Postgres/Elasticsearch and publishes the result to
Kafka for `response-dispatcher` to deliver.

Key settings:
- `DISCOVERY_BPP_ID` / `DISCOVERY_BPP_URI` — **this deployment's own public identity**, stamped onto outbound `on_discover` responses. `DISCOVERY_BPP_ID` **must equal** the `subscriberId` configured in `discover-adapter.yaml`'s `simplekeymanager` (§6.3) — onix-discover's caller keyset lookup is keyed by `context.bppId` and will fail signing if these don't match.
- `DISCOVERY_SPATIAL_ENGINE=elasticsearch` — spatial queries run against ES rather than PostGIS directly; also gates the ES→PostgreSQL "chain" queries (`DISCOVERY_CHAIN_*`) used for combined text+spatial cases.
- `DISCOVERY_FILTER_ACTIVE_CATALOG` / `DISCOVERY_FILTER_VALID_CATALOGS` (`true`) — only return catalogs currently marked active/valid by default; overridable per-request via `?active=`/`?validity=` query params.
- `DISCOVERY_DEDUP_CACHE_TTL_SECONDS` if set — idempotency cache keyed by `messageId`.
- `ES_MIN_SCORE`, `ES_RELATIVE_SCORE_THRESHOLD`, `ES_MULTI_MATCH_FIELDS`, `ES_TIE_BREAKER`, `ES_FUZZINESS` — search relevance tuning.
- `SIGNATURE_AUTH_ENABLED=false` — same reasoning as catalog-publish; onix-discover verifies inbound signatures.
- `LEGACY_ACK_NACK_SUPPORT=false` — leave `false` unless integrating with an ONA-style client expecting the old flat ACK/NACK envelope.
- `BECKN_PROTOCOL_API_SCHEMA_URL` / `SCHEMA_CACHE_TTL_HOURS` — same schema-fetch dependency as catalog-publish.

Healthcheck: `/actuator/health`.

### 5.4 `response-dispatcher`

Consumes `discovr.discover.out.responses` from Kafka and POSTs the result on to
`onix-discover`'s caller module, which then signs and forwards it to the requesting BAP.

Key settings:
- `SPRING_KAFKA_LISTENER_CONCURRENCY=4` — one listener thread per Kafka partition, so a slow/unreachable BAP callback on one partition doesn't block the others.
- `STATIC_CALLBACK_ENABLED=true` / `STATIC_CALLBACK_URL=http://onix-discover:8080/caller` — every outbound `on_discover` is routed through onix-discover (which resolves `context.bapUri` per-request) rather than this service doing its own DeDi lookup.
- `SIGNING_ENABLED=false` — outbound signing is done by onix-discover's caller module, not here.
- `CALLBACK_URL_VALIDATION_ENABLED=false` — the SSRF guard is disabled specifically because `onix-discover` is an internal container hostname, which would otherwise be flagged; do not disable this if you point `STATIC_CALLBACK_URL` at anything else.
- `HTTP_CLIENT_CONNECTION_TIMEOUT` / `HTTP_CLIENT_TIMEOUT` / `HTTP_RETRY_MAX_ATTEMPTS` — fail-fast tuning for the (unused while STATIC_CALLBACK_ENABLED=true) DeDi-lookup fallback path.

Depends on `catalog-discover-job` (healthy) and `onix-discover` (started).

### 5.5 `onix-discover`

The only public-facing application container (everything else sits behind it). Config is
entirely in the mounted `onix-discover/discover-adapter.yaml` — see §6 for the full
breakdown. The compose-level settings are:

- `image: fidedocker/onix-crawler:1.0.0` — the crawler-capable build (see §2).
- `platform: linux/amd64`.
- `command: ["./server", "--config=/app/config/discover-adapter.yaml"]` — overrides the image's default CMD (which reads `$CONFIG_FILE`) to point at the mounted config directly.
- `depends_on: redis (healthy)`.
- No host port binding — only reachable via the `nginx-proxy-manager` container on the shared `beckn-network`, or from other containers on that network directly at `onix-discover:8080`.

### 5.6 `nginx-proxy-manager`

The single public ingress. Exposes `80`/`443` (traffic) and `81` (admin UI — **restrict
this to your own IP** at the VM firewall, not just at NPM). See §7 for the proxy-host and
rewrite-rule setup once the containers are up.

---

## 6. `onix-discover/discover-adapter.yaml` — full config reference

Four modules run in a single onix-discover process:

| Module | Path | Role | Purpose |
|---|---|---|---|
| `discoverReceiver` | `/receiver/` | `bpp` | Public inbound: validates signature + schema, routes `discover`/`catalog/push` to the internal services. |
| `discoverReceiverOnPull` | `/receiver-on-pull/` | `bpp` | Identical to `discoverReceiver` except `validateSchema` is omitted — `catalog/on_pull` payloads carry `publishDirectives` that don't fit the strict Beckn v2.0 OpenAPI schema. |
| `discoverCaller` | `/caller/` | `bpp` | Internal: signs outbound `on_discover` and routes to the requesting BAP via `context.bapUri`. |
| `crawl` | `/crawl` | `bap` | Internal, no public exposure: independently pulls catalog manifests from other participants' DeDi index URLs and pushes verified results into `catalog-publish`. |

### 6.1 Top-level

```yaml
appName: "onix-discover"
log:
  level: info            # bump to "debug" for troubleshooting
http:
  port: 8080
  timeout: { read: 30, write: 30, idle: 30 }
pluginManager:
  root: ./plugins
```

### 6.2 Identity — `subscriberId` (appears 3×, plus once more inside each module's `keyManager`)

Every module's handler-level `subscriberId` and every `keyManager.config.subscriberId`
**must be the identical string** — the DeDi-registered subscriber ID for this deployment
(e.g. `discover.example.com`). This value must also equal `DISCOVERY_BPP_ID` on
`catalog-discover-job` (§5.3). A mismatch breaks signature verification and outbound
signing.

### 6.3 Signing/encryption keys — `keyManager` (`simplekeymanager`, appears 3×)

```yaml
keyManager:
  id: simplekeymanager
  config:
    subscriberId: <your-subscriber-id>
    keyId: <your-DeDi-key-id>
    signingPrivateKey: <base64 Ed25519 private key>
    signingPublicKey: <base64 Ed25519 public key>
    encrPrivateKey: <base64 X25519 private key>
    encrPublicKey: <base64 X25519 public key>
```

**Swap these for real, DeDi-registered keys before going live.** The committed
`discover-adapter.yaml` ships with dummy 32-byte zero-seed values (`keyId:
local-test-key-id`, keys all `AAAA...=`) so the stack boots and responds out of the box for
local testing — inbound requests will correctly get rejected at `validateSign` (they're not
actually signed by anything DeDi recognizes), but nothing crashes on startup. The same four
key values are used identically across all three `std`-handler modules; there is one
keyset per subscriber, not one per module.

### 6.4 Registry — `dediregistry` (appears 4×)

```yaml
registry:
  id: dediregistry
  config:
    # allowedNetworkIDs: "network.a,network.b"   # optional — see below
    timeout: 10
    retry_max: "3"
    retry_wait_min: "100ms"
    retry_wait_max: "500ms"
```

The registry `url` itself is **not** set here — it's a canonical value injected internally
from the adapter's own beckn-constants and will error at startup on any mismatch if you try
to override it. `allowedNetworkIDs` is commented out by default (accepts a signature from
any subscriber the registry returns, no network-membership filtering) — uncomment and set
a comma-separated list to restrict verification to specific network memberships.

### 6.5 Cache — `cache` (appears 4×)

```yaml
cache:
  id: cache
  config:
    addr: redis:6379
```

Backs both the `payloadStore` plugin and general key/signature caching. Must point at the
`redis` service on the shared network.

### 6.6 Payload store — `payloadStore` (appears 3×, receiver/receiverOnPull/caller — not crawl)

```yaml
payloadStore:
  id: payloadstore
  config:
    ttl: "24h"
    indexTTL: "25h"
    maxBodyBytes: "1048576"
    storeBody: "true"
    storeSignature: "true"
    compress: "true"
```

Enables messageId correlation for 4-line solicited callback signatures — the receiver
caches the inbound request by `messageId`, and the caller's later signing of the matching
`on_discover` finds it via the same Redis namespace (hardcoded to `"onix"` internally, so
this works across modules without extra config).

### 6.7 Schema validator — `schemaValidator` (`schemav2validator`, appears 3×)

```yaml
schemaValidator:
  id: schemav2validator
  config:
    cacheTTL: "3600"
    extendedSchema_enabled: "false"
    extendedSchema_cacheTTL: "86400"
    extendedSchema_maxCacheSize: "100"
    extendedSchema_downloadTimeout: "30"
    extendedSchema_allowedDomains: "beckn.org,example.com,raw.githubusercontent.com"
```

The base schema type/location aren't set here either — injected from beckn-constants.
`extendedSchema_*` governs optional domain-specific schema extensions; disabled by default.

### 6.8 Router — `router` (appears 3×, different config file per module)

```yaml
router:
  id: router
  config:
    routingConfig: ./config/routing-discover-receiver.yaml   # receiver + receiverOnPull
    # routingConfig: ./config/routing-discover-caller.yaml   # caller
```

See the separate `routing-discover-receiver.yaml` / `routing-discover-caller.yaml` files
for the actual routing rules (inbound endpoint → internal service; outbound `on_discover` →
`context.bapUri`).

### 6.9 Middleware — `reqpreprocessor` (appears 3×)

```yaml
middleware:
  - id: reqpreprocessor
    config:
      contextKeys: transactionId,messageId
      role: bpp
```

Populates request context (transaction/message IDs) for logging correlation, per module.

### 6.10 Module pipelines (`steps:`)

| Module | Steps |
|---|---|
| `discoverReceiver` | `validateSign` → `validateSchema` → `addRoute` → `storePayload` → `signAck` |
| `discoverReceiverOnPull` | `validateSign` → `addRoute` → `storePayload` → `signAck` (no `validateSchema`) |
| `discoverCaller` | `addRoute` → `sign` → `validateSchema` → `storePayload` → `validateAckSign` |

`signAck`/`validateAckSign` are recognized by the handler's internal step-initialization
logic when present in `steps:` — there is no separate `responseSteps:` key.

### 6.11 Crawl module — `catalogcrawler`

```yaml
- name: crawl
  path: /crawl
  handler:
    type: crawl
    role: bap
    plugins:
      registry:
        id: dediregistry
        config: { timeout: "10", retry_max: "3", retry_wait_min: "100ms", retry_wait_max: "500ms" }
      crawler:
        id: crawler
        config:
          CRAWLER_ENABLED: "true"
          CRAWLER_DB_DSN: "postgres://discover_user:<password>@postgres:5432/discover_db?sslmode=disable"
          CRAWLER_PUSH_ENDPOINT: "http://catalog-publish:8080/beckn/catalog/push"
          CRAWLER_BPP_URI: "<this deployment's public BPP URI — matches DISCOVERY_BPP_URI>"
          CRAWLER_INDEX_URLS: "http://catalog-source.invalid/none"   # placeholder; ignored once CRAWLER_REGISTRY_URL is set
          CRAWLER_NETWORK_IDS: "network.a,network.b"                 # networks to discover peers on via the registry
          CRAWLER_REGISTRY_URL: "https://<your-dedi-registry>/registry/dedi"
          CRAWLER_INDEX_INTERVAL: "5m"
          CRAWLER_CATALOG_INTERVAL: "1m"
          CRAWLER_FETCH_TIMEOUT: "30s"
          CRAWLER_MAX_ARTIFACT_BYTES: "10485760"
          CRAWLER_MAX_DECOMPRESSED_BYTES: "104857600"
          CRAWLER_MAX_PUSH_BYTES: "10485760"
          CRAWLER_MAX_ATTEMPTS: "5"
          CRAWLER_MERGE_ONLY: "true"
          CRAWLER_LOG_LEVEL: "info"
```

- `CRAWLER_DB_DSN` — the crawler maintains its own state tables (crawl cursor, queued
  syncs) in the same Postgres instance the rest of the stack uses; no separate database
  needed.
- `CRAWLER_PUSH_ENDPOINT` — internal container-DNS route to `catalog-publish`'s
  `/beckn/catalog/push`, bypassing the public ingress entirely.
- `CRAWLER_NETWORK_IDS` — once `CRAWLER_REGISTRY_URL` is set, the crawler discovers peer
  catalog index URLs by querying the DeDi registry for each network ID listed here; only
  records the registry marks `state=="live"` with a non-empty `catalog_index_urls[]` are
  crawled. `CRAWLER_INDEX_URLS` becomes a no-op placeholder once registry discovery is
  active — leave it as the non-resolving default.
- Set `CRAWLER_ENABLED: "false"` to disable crawling entirely without removing the module.

---

## 7. First-time setup

```bash
docker network create beckn-network
# Fill in real subscriberId/keys/passwords/URIs in discover-adapter.yaml and
# discovr-stack.yml per §4 and §6 above.
docker compose -f discovr-stack.yml pull
docker compose -f discovr-stack.yml up -d
docker compose -f discovr-stack.yml ps    # all healthy except onix-discover ("running" is fine, no healthcheck defined)
```

### NPM proxy host setup

Open `http://<vm-ip>:81` → first-boot login `admin@example.com` / `changeme` → **change
the password immediately**.

**Proxy Hosts → Add Proxy Host:**

| Field | Value |
|---|---|
| Domain Names | `<vm-ip>` or your DNS name (`<ip>.sslip.io` works for a quick Let's Encrypt cert on an IP-only deploy) |
| Scheme | `http` |
| Forward Hostname/IP | `onix-discover` |
| Forward Port | `8080` |
| Block Common Exploits | **OFF** — injects headers that break Beckn signature verification |
| Websockets Support | OFF |

**Custom Locations** (gives BAPs a clean `/beckn/*` public path while keeping onix's
internal module paths hidden):

Add location `/beckn`, forward to `onix-discover:8080`, then under that location's
**Advanced** tab add:

```nginx
if ($request_uri ~ "^/beckn/catalog/on_pull") {
    rewrite ^/beckn/(.*)$ /receiver-on-pull/$1 break;
}
if ($request_uri ~ "^/beckn/(on_discover|catalog/subscription|catalog/pull)") {
    rewrite ^/beckn/(.*)$ /caller/$1 break;
}
if ($request_uri !~ "^/beckn/(on_discover|catalog/subscription|catalog/pull|catalog/on_pull)") {
    rewrite ^/beckn/(.*)$ /receiver/$1 break;
}
```

| External URL | Routed to |
|---|---|
| `<host>/beckn/discover` | `discoverReceiver` |
| `<host>/beckn/catalog/push` | `discoverReceiver` |
| `<host>/beckn/catalog/on_pull` | `discoverReceiverOnPull` (schema check skipped) |
| `<host>/beckn/on_discover` | `discoverCaller` |
| `<host>/beckn/catalog/subscription`, `/catalog/pull` | `discoverCaller` |

Then enable SSL (**SSL tab → Request a new certificate → Force SSL**) once your domain
resolves to the VM.

### Firewall

Allow inbound TCP `80`, `443`, `81` (restrict `81` to your own IP), `22`. Block everything
else.

### Register in DeDi

Your subscriber record (subscriberId, public base URL, keyId, both public keys) must exist
in DeDi before signature verification/signing will work.

---

## 8. Troubleshooting

- **`catalog-discover-job` / `catalog-publish` crash-loop on startup with
  `Connect timed out` fetching `BECKN_PROTOCOL_API_SCHEMA_URL`**: both services fetch and
  cache the Beckn v2.0 API schema from `raw.githubusercontent.com` at startup and will not
  start without it. If this fails consistently even though `curl` to the same URL succeeds
  from the host or from inside the container, it is very likely the **JVM** getting stuck
  on a broken/blackholed IPv6 path before ever falling back to IPv4 — a known symptom
  particularly under emulated (non-native-architecture) container runtimes. Add
  `-Djava.net.preferIPv4Stack=true` to that service's `JAVA_OPTS`. If it persists, check
  outbound connectivity/firewall rules specifically for the JVM process, not just `curl`.
- **`onix-discover` fails with `invalid module: crawl`**: the image doesn't have the
  crawler plugin compiled in — confirm you're running `fidedocker/onix-crawler:1.0.0`, not
  `fidedocker/onix-adapter`.
- **Signature verification fails for all inbound requests**: check that `subscriberId`
  is identical across every module's handler config, every `keyManager.config`, and
  `DISCOVERY_BPP_ID` on `catalog-discover-job` — and that the DeDi record for that
  subscriber actually exists with matching public keys.
- **Outbound `on_discover` never reaches the BAP**: check `response-dispatcher` logs for
  `event=CALLBACK_RESOLVED`, then `onix-discover` logs for `POST /caller/on_discover` —
  if the caller module never receives it, check `STATIC_CALLBACK_URL` and that
  `onix-discover` is reachable from `response-dispatcher` on `beckn-network`.

## 9. Rollback

```bash
# Revert image tags in discovr-stack.yml to the previous known-good version, then:
docker compose -f discovr-stack.yml up -d
```

Postgres/Elasticsearch/Kafka/Redis data persists across container restarts via named
volumes — a plain `up -d` after a rollback does not lose data. To wipe everything
(destructive): `docker compose -f discovr-stack.yml down -v`.
