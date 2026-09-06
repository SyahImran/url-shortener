# ADR 0003: Use Redis for redirect caching and best-effort destination deduplication

## Status

Accepted

## Context

The service is read-heavy, must handle hot URLs, and targets less than 100 ms p95 redirect latency for in-region clients. It should also attempt to reuse an existing short URL for an exact destination string, but destination uniqueness is explicitly best effort rather than an invariant.

## Decision

Use Redis Cluster for two non-authoritative concerns:

1. Lazy redirect cache: `url:<short_code> -> <destination_uri>`.
2. Best-effort destination lookup: `dest:<SHA-256(exact_destination_uri)> -> <short_code>`.

Redirect-cache entries have no explicit TTL and use LFU-oriented eviction under memory pressure. Creation does not synchronously populate the redirect cache. Destination-index entries may be evicted or overwritten and do not participate in a distributed transaction with PostgreSQL.

If Redis is unavailable, creation may proceed without deduplication and redirects fall back to PostgreSQL.

## Consequences

- Hot and frequently accessed URLs usually avoid PostgreSQL reads.
- Only accessed mappings consume redirect-cache capacity.
- Redis loss or eviction does not lose authoritative URL mappings.
- Destination deduplication can miss under eviction, restart, failure, or concurrency, producing multiple permanent short codes for one destination; this is accepted behavior.
- Extremely hot Redis keys may eventually justify an additional application-local cache, but that is not part of the baseline design.
