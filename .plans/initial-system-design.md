# Initial URL Shortener System Design

## Status

Proposed

## Purpose

This plan captures the proposed technical design for the initial implementation of the URL shortener. It is non-authoritative working material. Product requirements are defined in `docs/prd.md`; `docs/architecture.md` should describe only the architecture that has actually been implemented.

## Design goals

- Authenticated short-URL creation and public redirects.
- System-generated and user-selected Base62 codes in one global namespace.
- Permanent immutable mappings.
- Best-effort global reuse of an existing short URL for the exact destination string.
- Approximately 100 million creates/day and 1 billion redirects/day.
- Approximately 2x normal peak traffic.
- Less than 100 ms p95 redirect latency for in-region clients.
- Horizontal scaling in one region.
- Continue serving cached redirects during PostgreSQL outages where possible.

## Proposed high-level architecture

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

Use one horizontally scaled service for both creation and redirect endpoints. Keep application instances stateless with respect to authoritative business data.

## Proposed data model

```sql
CREATE TABLE short_urls (
    short_code      VARCHAR(32) PRIMARY KEY,
    destination_uri TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`short_code` may be generated or user-selected. `destination_uri` is stored exactly as supplied and is not unique.

The authoritative invariant is:

```text
Every successfully created short code permanently maps to exactly one destination URI.
```

Destination-to-short-code uniqueness is intentionally not an invariant.

## API design

### Create

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

- New mapping: `201 Created`.
- Existing mapping reused through best-effort destination lookup: `200 OK`.
- Missing/invalid JWT: `401 Unauthorized`.
- Invalid URI or reserved/invalid alias: `400 Bad Request`.
- Alias conflict: `409 Conflict`.
- Dependency/internal failure: `5xx`.

The endpoint is not API-idempotent.

### Redirect

```http
GET /{shortCode}
```

Success returns `302 Found` with the destination in `Location`. Unknown codes return `404 Not Found` with a small JSON error.

### Health

`GET /health` verifies critical Redis and PostgreSQL connectivity and returns a minimal `healthy` or `unhealthy` status. Because cached redirects may still work during PostgreSQL failure, dependency-sensitive health should not automatically be the load balancer's only routing signal.

## Short-code generation

Use Snowflake-style distributed numeric IDs composed from a timestamp, worker ID, and per-time-unit sequence. Encode the numeric ID with the fixed Base62 alphabet:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

The alphabet order becomes a persisted contract once production data exists.

Each active generator must have a unique worker ID. Prefer a deployment-platform identity or lease-backed allocator rather than ephemeral IP-derived identity. Detect clock rollback; small configured drift may wait for recovery, while larger rollback must stop ID generation rather than risk duplicates.

Insert generated codes without a pre-check. On a primary-key conflict, perform a bounded retry with a new ID. Insert custom aliases directly; a uniqueness conflict returns `409`.

See ADR 0001 for the adopted code-generation decision.

## PostgreSQL storage

Use application-level PostgreSQL sharding. Route by `short_code`, the authoritative lookup key.

Use a deterministic consistent-hash ring from `hash(short_code)` to logical shard. All application instances must share the same versioned shard map.

Each shard should have one primary and one or more read replicas:

- writes go to the primary;
- redirect cache misses query a healthy replica first;
- replica misses fall back to the primary to reduce false `404`s from replication lag;
- if replicas are unavailable, reads use the primary.

Use bounded connection pools and timeouts. Keep logical read/write pools per shard. Add a connection proxy such as PgBouncer if total connection counts require it.

### Rebalancing

Consistent hashing reduces remapping but does not move data. Adding or removing shards requires an explicit migration process:

1. define a new shard-map version;
2. identify moving ranges;
3. copy affected rows to new owners;
4. make routing aware of old and new placement during migration;
5. verify copied data;
6. switch authoritative routing to the new map; and
7. remove old copies only after a safety period.

Never change shard count by merely deploying a different ring.

See ADR 0002 for the adopted authoritative-storage decision.

## Redis design

Use Redis Cluster for two non-authoritative concerns.

Redirect cache:

```text
url:<short_code> -> <destination_uri>
```

Populate it lazily. Use memory-pressure eviction rather than treating Redis as authoritative.

Best-effort destination index:

```text
dest:<SHA-256(exact_destination_uri)> -> <short_code>
```

This index enables best-effort exact-string destination reuse. It is not transactionally coordinated with PostgreSQL and may be evicted or overwritten.

If Redis is unavailable, creation skips destination deduplication and writes a new authoritative mapping; redirects fall back to PostgreSQL. This may increase duplicate destinations or latency but must not corrupt authoritative mappings.

See ADR 0003 for the adopted Redis decision.

## Creation flow

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

An explicit custom alias never reuses a different generated code for the same destination.

## Redirect flow

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

Do not use negative caching initially.

## Authentication

An external authentication system issues JWTs. Validate signature, issuer, audience where applicable, expiration, and not-before claims locally. Cache JWKS/signing keys and refresh periodically or when an unknown key ID appears. Do not call the authentication service synchronously for each create request.

Redirects require no authentication.

## Load balancing and scaling

Use a layer-7 load balancer in front of horizontally scaled application instances. Useful autoscaling signals include request rate, CPU, latency, active connections, and runtime saturation.

The application tier is unlikely to be the hardest scaling problem at the target traffic. Long-term PostgreSQL growth and skewed hot-key traffic are more significant risks.

## Hot URLs

Rely on Redis first. If measured hot-key pressure becomes material, consider a small application-local cache ahead of Redis. Do not add this layer before measurements justify it.

## Failure behavior

- **Redis unavailable:** redirects fall back to PostgreSQL; creation skips best-effort deduplication.
- **PostgreSQL primary unavailable:** new creation for that shard fails; cached redirects continue; replicas may still serve existing rows.
- **Replica unavailable:** use another replica, then the primary.
- **PostgreSQL entirely unavailable:** cached redirects continue; uncached redirects and creation fail.
- **Authentication service unavailable:** creation may continue for JWTs verifiable with cached signing keys.
- **Snowflake allocation unavailable:** existing valid workers may continue; new instances must not generate IDs without a unique worker identity.
- **Clock rollback:** the affected generator stops issuing IDs until safe.

## Observability

Use structured logs with request ID, route, status, latency, shard ID, cache hit/miss, database target, retry reason, and Snowflake worker ID where useful. Never log full JWTs. Avoid full destination URIs by default because query strings may contain sensitive information.

Measure HTTP throughput/errors/latency, Redis hit rate/latency/evictions/memory, PostgreSQL latency/errors/connections/replication lag/primary fallback, Snowflake rollback/sequence events, and per-shard throughput/errors/map version.

## Security

- Use TLS for public traffic.
- Validate JWTs locally for creation.
- Keep PostgreSQL and Redis topology private.
- Validate aliases before routing.
- Centralize the reserved-alias list.
- Use parameterized SQL.
- Apply reasonable request and destination-length limits.
- Keep secrets out of the repository and logs.

## Implementation boundaries

Use clear internal boundaries:

```text
HTTP transport
  -> application service
       -> authentication adapter
       -> short-code generator / Base62 codec
       -> shard router
       -> Redis adapter
       -> PostgreSQL repository
```

Keep HTTP handlers thin. Externalize configuration for base URL, JWT/JWKS settings, Redis, shard map, PostgreSQL, Snowflake worker allocation, timeouts, and reserved aliases.

Every external operation must have bounded timeouts. Use bounded retries only for safe transient cases. Manage database schema through versioned migrations once implementation begins.

## Testing strategy

Unit tests should cover Base62, URI/alias validation, reserved aliases, Snowflake uniqueness and rollback, shard routing, and error mapping.

Integration tests should cover PostgreSQL uniqueness, custom alias conflicts, Redis destination hits/misses, redirect cache hit/miss, replica-to-primary fallback, missing codes, Redis failure, replica failure, and JWT validation with cached keys.

Concurrency tests should verify that duplicate destinations are allowed under races, only one concurrent custom-alias insert wins, generated IDs remain unique across workers, and generated-code conflict retries are bounded.

Load tests should exercise approximately 23k redirect RPS and 2.3k create RPS, skewed hot-key traffic, Redis failure, replica failure, primary fallback, and autoscaling behavior. Production sizing must use measured p95/p99 latency and saturation rather than arithmetic averages alone.

## Open implementation risks

1. Snowflake worker-ID lease safety and clock rollback.
2. Shard-map rollout and rebalancing.
3. PostgreSQL physical growth at tens of billions of rows.
4. Redis hot-key behavior.
5. Database connection counts as shards and application instances grow.
6. Replica lag and primary-fallback frequency.
7. Dependency-sensitive `/health` behavior with load balancing.
8. Final maximum URI and alias lengths.
9. Operational changes to reserved aliases after permanent codes exist.

## Completion criteria

The initial design is implemented when the repository contains working application code, migrations, configuration, tests, and operational setup that satisfy `docs/prd.md`; the implementation has been verified against the testing strategy; and `docs/architecture.md` has been updated to describe the resulting current system rather than this proposal.
