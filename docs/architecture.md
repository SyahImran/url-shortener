# URL Shortener System Design

## Status

Proposed

## Purpose

This document defines the durable system design for a high-scale URL shortening service. The service allows authenticated users to create permanent short URLs and anyone to resolve them. It prioritizes a fast redirect path, horizontal scalability, immutable data, and high availability within a single region.

Implementation plans belong under `.plans/`; investigations and audits belong under `.research/`.

## Goals

- Authenticated short-URL creation and public redirects.
- System-generated and user-selected Base62 codes in one global namespace.
- Permanent immutable mappings.
- Best-effort global reuse of an existing short URL for the exact destination string.
- Approximately 100 million creates/day and 1 billion redirects/day.
- Approximately 2x normal peak traffic.
- Less than 100 ms p95 redirect latency for in-region clients.
- Horizontal scaling in one region.
- Continue serving cached redirects during PostgreSQL outages where possible.

## Non-goals

- Expiration, deletion, mutation, or code reuse.
- User ownership, listing, or search APIs.
- Redirect analytics.
- URI normalization or destination reputation scanning.
- Application-level creation or redirect rate limits.
- Strong destination deduplication.
- API idempotency keys.
- Multi-region operation or regional disaster recovery.
- Distributed tracing.

# Requirements

## Functional requirements

### Creation

- Only authenticated users may create short URLs; any valid authenticated user is allowed.
- Destinations may use any syntactically valid URI scheme and are stored exactly as supplied.
- No URI canonicalization is performed.
- `customAlias` is optional and must contain only `A-Z`, `a-z`, and `0-9`.
- Custom aliases are globally unique and share the namespace with generated codes.
- Application routes such as `health`, `api`, `admin`, and `metrics` are reserved.
- An explicit custom alias overrides destination reuse.
- Created mappings are permanent and immutable.

### Destination deduplication

When no custom alias is requested, the service attempts to reuse an existing mapping for the exact destination string. Deduplication is global but best effort. Cache eviction, failure, or concurrent requests may create multiple permanent codes for one destination; this is valid behavior.

### Redirects

- `GET /{shortCode}` is public.
- Successful resolution returns `302 Found` with the destination in `Location`.
- Unknown codes return `404 Not Found` with a small JSON error.
- `HEAD` is not required.

# Non-functional requirements

## Scale

| Operation | Daily volume | Average RPS | Approx. 2x peak |
| --- | ---: | ---: | ---: |
| Create | 100,000,000 | 1,157 | 2,315 |
| Redirect | 1,000,000,000 | 11,574 | 23,148 |
| Combined | 1,100,000,000 | 12,731 | 25,463 |

The service relies on horizontal scaling and autoscaling rather than a fixed additional headroom target.

Permanent mappings produce approximately 3 billion rows/month and 36.5 billion rows/year. At an effective 500 bytes to 1 KB per row including indexes and overhead, this is roughly 18-36.5 TB/year before replication. Long-term data growth is the main reason for sharding.

## Availability and consistency

- High availability, without a formal SLA.
- Single region; a full regional outage may make the service unavailable until recovery.
- Eventual consistency is acceptable.
- Cached redirects continue during PostgreSQL failure where possible.
- Replica misses fall back to the shard primary to reduce false 404s caused by replication lag.
- PostgreSQL supplies normal database durability guarantees; no separate durability SLA is defined.

## Latency and bursts

- Redirect target: less than 100 ms p95 for clients near the deployment region.
- Global latency is not guaranteed without CDN or multi-region routing.
- Design for approximately 2x normal peak traffic and hot URLs.

## Observability

Provide structured logs, health checks, request/error/latency metrics, Redis hit/miss and eviction metrics, PostgreSQL query/connection/replication metrics, and shard-level saturation metrics. Distributed tracing is not required.

# High-level architecture

```text
                           +---------------------+
                           | External Auth       |
                           | JWT issuer / JWKS   |
                           +----------+----------+
                                      |
Client                                | signing keys
  |                                   v
  |                          +-------------------+
  +------------------------->| Load Balancer     |
                             +---------+---------+
                                       |
                                       v
                         +---------------------------+
                         | URL Shortener Service     |
                         | POST /short-urls          |
                         | GET /{shortCode}          |
                         | GET /health               |
                         | JWT validation            |
                         | Snowflake + Base62        |
                         | shard routing             |
                         +---------+-----------+-----+
                                   |           |
                                   v           v
                           +-----------+   +---------------------+
                           | Redis     |   | PostgreSQL shards   |
                           | Cluster   |   | primary + replicas  |
                           +-----------+   +---------------------+
```

One horizontally scaled service exposes both creation and redirect endpoints. Instances retain no authoritative business state locally; Snowflake worker identity is ephemeral infrastructure state.

# Data model

The authoritative entity is intentionally minimal:

```sql
CREATE TABLE short_urls (
    short_code      VARCHAR(32) PRIMARY KEY,
    destination_uri TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`short_code` may be generated or user-selected. The database does not need to distinguish them after creation. `destination_uri` is the exact supplied string and is not unique. Fields such as `user_id`, `expires_at`, `updated_at`, `deleted_at`, `click_count`, and `custom_alias` are deliberately omitted.

The authoritative invariant is:

```text
Every successfully created short code permanently maps to exactly one destination URI.
```

Destination-to-short-code uniqueness is not an invariant.

# API design

## Create

```http
POST /short-urls
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "destination": "https://example.com",
  "customAlias": "Example123"
}
```

`customAlias` is optional.

New mapping: `201 Created`.

Existing mapping reused through best-effort destination lookup: `200 OK`.

Response:

```json
{
  "shortUrl": "https://sho.rt/Example123"
}
```

Errors:

| Condition | Status |
| --- | --- |
| Missing/invalid JWT | `401 Unauthorized` |
| Invalid URI | `400 Bad Request` |
| Invalid/reserved alias | `400 Bad Request` |
| Alias already exists | `409 Conflict` |
| Dependency/internal failure | `5xx` |

The endpoint is not API-idempotent.

## Redirect

```http
GET /{shortCode}
```

Success:

```http
HTTP/1.1 302 Found
Location: https://example.com
```

Unknown code returns `404` JSON.

## Health

`GET /health` verifies critical Redis and PostgreSQL connectivity and returns a minimal `healthy` or `unhealthy` status. Because cached redirects may still work during PostgreSQL failure, this dependency-sensitive endpoint must not blindly be the load balancer's only routing signal.

# Short-code generation

Generated codes use Snowflake-style distributed numeric IDs composed conceptually from timestamp, worker ID, and per-time-unit sequence. The numeric ID is encoded with the fixed Base62 alphabet:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

The alphabet order is a persisted contract and must never change after production data exists.

Each active generator must have a unique worker ID. Prefer a deployment-platform identity or lease-backed allocator; do not derive identity from ephemeral IP addresses. Worker IDs may be reused only after the previous holder is known to be inactive.

Clock rollback must be detected. Small configurable drift may wait for the clock to catch up; larger rollback stops ID generation rather than risking duplicates. Sequence exhaustion waits for the next time unit.

Generated codes are inserted without a pre-check. A primary-key conflict causes a bounded retry with a new ID. Generated/generated collisions indicate an ID-generation fault; normal conflicts are expected only when a custom alias already occupies the generated code.

Custom aliases are also inserted directly. A uniqueness conflict returns `409`, avoiding a check-then-insert race.

# PostgreSQL storage

Use application-level PostgreSQL sharding. The authoritative access direction is `short_code -> destination_uri`, so the short code is the shard key.

A deterministic consistent-hash ring maps `hash(short_code)` to a logical shard. Every application instance must use the same versioned shard map.

Each shard has one primary and one or more read replicas:

- creation writes to the primary;
- redirect cache misses query a healthy replica first;
- a replica miss falls back to the primary;
- if replicas are unavailable, reads use the primary.

Use bounded connection pools and timeouts. Keep separate logical read/write pools per shard. At large instance/shard counts, use a connection proxy such as PgBouncer when needed to control total PostgreSQL connections.

## Rebalancing

Consistent hashing reduces remapping but does not move rows. Adding/removing shards requires an explicit migration:

1. Define a new shard-map version.
2. Identify ranges that move.
3. Copy affected rows to new owners.
4. During migration, make routing aware of old and new placement.
5. Verify copied data.
6. Switch authoritative routing to the new map.
7. Remove old copies after a safety period.

Never change shard count by merely deploying a new hash ring.

# Redis design

Use Redis Cluster for two non-authoritative concerns.

Redirect cache:

```text
url:<short_code> -> <destination_uri>
```

It is populated lazily, has no explicit TTL, and uses LFU-oriented eviction under memory pressure.

Best-effort destination index:

```text
dest:<SHA-256(exact_destination_uri)> -> <short_code>
```

The hash gives a fixed-size key; it is not a security boundary. Entries may be evicted or overwritten and are not transactionally coordinated with PostgreSQL.

If Redis is unavailable, creation skips deduplication and writes an authoritative mapping; redirects fall back to PostgreSQL. This increases duplicates or latency but does not corrupt authoritative data.

# Creation flow

Generated code:

```text
POST /short-urls
  -> validate JWT locally
  -> validate exact destination URI
  -> SHA-256 destination
  -> Redis destination lookup
       hit  -> return existing short URL (200)
       miss -> generate Snowflake ID
            -> Base62 encode
            -> resolve shard
            -> INSERT on shard primary
                 conflict -> bounded regenerate/retry
            -> write destination hash -> short code to Redis
            -> return 201
```

If Redis is unavailable, skip deduplication and continue to PostgreSQL.

Custom alias:

```text
validate JWT
  -> validate destination
  -> validate Base62 alias
  -> reject reserved alias
  -> resolve shard
  -> INSERT directly
       conflict -> 409
  -> optionally update destination index
  -> 201
```

An explicit custom alias does not reuse a different existing generated code for the destination.

# Redirect flow

```text
GET /{shortCode}
  -> Redis redirect cache
       hit  -> 302
       miss -> resolve shard
            -> read replica
                 hit  -> populate Redis -> 302
                 miss -> primary lookup
                      hit  -> populate Redis -> 302
                      miss -> 404
```

No negative caching is used, so repeated nonexistent codes continue to reach PostgreSQL.

# Authentication

An external authentication system issues JWTs. The service validates signature, issuer, audience where applicable, expiration, and not-before claims locally. Cache JWKS/signing keys and refresh periodically or when an unknown key ID appears. Do not call the authentication service synchronously for every create request.

This permits creation to continue during a temporary auth-service outage while known signing material remains valid. Redirects perform no authentication.

# Load balancing and scaling

Use a layer-7 load balancer in front of horizontally scaled application instances. Useful autoscaling signals include request rate, CPU, latency, active connections, and runtime saturation.

The application tier is not expected to be the hardest scaling problem at the stated ~25k combined 2x peak RPS; long-term PostgreSQL storage growth and skewed hot-key traffic are more significant.

# Hot URLs

Frequently accessed mappings should remain in Redis and avoid PostgreSQL. A viral short code can still become a hot Redis key. The baseline design relies on Redis and measurement first. If hot-key pressure becomes material, add a small application-local cache ahead of Redis; do not add that layer before measurements justify it.

# Failure behavior

## Redis unavailable

Redirects fall back to PostgreSQL. Creation skips best-effort deduplication and may create additional destination duplicates.

## PostgreSQL primary unavailable

New creation for that shard fails. Cached redirects continue. Replicas may still serve existing mappings, but replica misses cannot reliably fall back to the unavailable primary.

## Replica unavailable

Use another replica, then the primary if necessary.

## PostgreSQL entirely unavailable

Cached redirects continue; uncached redirects and creation fail.

## Authentication service unavailable

Creation continues for JWTs verifiable with cached/current signing keys. Unknown rotated keys may fail until JWKS access returns.

## Snowflake allocation unavailable

Existing instances with valid worker leases may continue; new instances must not generate IDs without a unique worker identity.

## Clock rollback

The affected instance stops issuing IDs until safe rather than risking duplicate identifiers.

# Observability

Structured logs should include request ID, route, status, latency, shard ID, cache hit/miss, database target (`replica`/`primary`), retry reason, and Snowflake worker ID where relevant. Never log full JWTs. Avoid logging complete destination URIs by default because query strings may contain sensitive data.

Metrics should cover HTTP throughput/errors/latency, Redis hit rates/latency/evictions/memory, PostgreSQL latency/errors/connections/replication lag/primary fallback, Snowflake rollback or sequence events, and per-shard throughput/errors/map version.

# Security

- Use TLS for public traffic.
- Validate JWTs locally for creation.
- Keep database and Redis topology private.
- Validate aliases before routing.
- Centralize the reserved-alias list.
- Use parameterized SQL.
- Apply reasonable request and destination-length limits.
- Do not log credentials or full JWTs.
- Infrastructure-level DDoS/connection protection remains compatible with the requirement for no application-level rate limiting.

# Implementation conventions

Use one deployable service with clear internal boundaries:

```text
HTTP transport
  -> application service
       -> authentication adapter
       -> short-code generator / Base62 codec
       -> shard router
       -> Redis adapter
       -> PostgreSQL repository
```

Keep HTTP handlers thin. Externalize configuration for base URL, JWT/JWKS settings, Redis, shard map, PostgreSQL, Snowflake epoch/worker allocation, timeouts, and reserved aliases. Keep secrets out of the repository.

Every external operation must have bounded timeouts. Use bounded retries only for safe transient cases such as generated-code uniqueness conflicts, JWKS fetches, or connection establishment. Do not hide sustained dependency failures behind long synchronous retry loops.

Manage database schema through versioned migrations once implementation begins. Because mappings are permanent, destructive migrations require particular care.

# Testing strategy

Unit tests should cover Base62, URI/alias validation, reserved aliases, Snowflake uniqueness and rollback, shard routing, and error mapping.

Integration tests should cover PostgreSQL uniqueness, custom alias conflicts, Redis destination hits/misses, redirect cache hit/miss, replica-to-primary fallback, missing codes, Redis failure, replica failure, and JWT validation with cached keys.

Concurrency tests should verify that duplicate destinations are allowed under races, only one concurrent custom alias insert wins, generated IDs remain unique across workers, and generated-code conflict retry is bounded.

Load tests should exercise at least ~23k redirect RPS and ~2.3k create RPS, skewed hot-key traffic, Redis failure, replica failure, primary fallback, and autoscaling behavior. Production sizing must use measured p95/p99 latency and saturation rather than arithmetic averages alone.

# Key trade-offs

## PostgreSQL over a distributed KV store

Provides familiar relational constraints and tooling but makes the application responsible for shard routing, connection management, and rebalancing.

## Snowflake over random codes

Avoids probabilistic generated/generated collisions and central sequences, but introduces worker-identity and clock-safety concerns.

## Redis destination index over a persistent secondary index

Avoids a massive persistent destination index and matches best-effort semantics, but eviction/concurrency can produce duplicate destinations.

## Lazy caching over write-through

Keeps creation independent of redirect-cache population and only caches accessed URLs, at the cost of a database lookup on the first redirect.

## One service over separate read/write services

Keeps deployment and shared logic simple, while sacrificing independent deployment/scaling of the two paths. They can be separated later without changing the public API or storage model.

# Open implementation risks

1. Snowflake worker-ID lease safety and clock rollback.
2. Shard-map rollout and rebalancing.
3. PostgreSQL physical growth at tens of billions of rows.
4. Redis hot-key behavior.
5. Database connection counts as shards and app instances grow.
6. Replica lag and primary-fallback frequency.
7. Dependency-sensitive `/health` behavior with load balancing.
8. Final maximum URI and alias lengths.
9. Operational changes to reserved aliases after permanent codes exist.

# Evolution path

Potential future changes include separate read/write deployables, application-local hot-key caching, CDN/edge redirects, multi-region reads, stronger destination deduplication, a persistent destination index, an alternative distributed datastore, analytics, lifecycle management, and abuse detection. Add these only when requirements or measurements justify them.

# Final design summary

The baseline uses one horizontally scaled URL-shortener service, locally validated JWTs for creation, public 302 redirects, permanent immutable Base62 codes, optional globally unique Base62 aliases, Snowflake-style generated IDs, application-level PostgreSQL sharding with consistent hashing and read replicas, Redis Cluster for lazy redirect caching and best-effort destination deduplication, replica-first reads with primary fallback, direct insert for uniqueness, no negative caching, no analytics or expiration, no application-level rate limits, single-region high availability, and basic logs/metrics/health checks.
