# ADR 0002: Use application-sharded PostgreSQL for authoritative storage

## Status

Accepted

## Context

The dominant operation is lookup from `short_code` to `destination_uri`. The workload targets approximately 100 million creates and 1 billion redirects per day, while permanent mappings produce tens of billions of rows over time. The design requires familiar relational guarantees for authoritative mappings while allowing horizontal storage growth.

## Decision

Use PostgreSQL as the authoritative datastore and shard mappings at the application layer by short code using a deterministic consistent-hash ring.

Each logical shard has one primary and one or more read replicas. Creation writes go to the primary. Cache-miss redirects read from replicas first and fall back to the primary on a replica miss to mitigate replication lag.

Shard-map configuration is versioned. Adding or removing shards requires an explicit data-rebalancing procedure; changing the hash ring alone is not sufficient.

## Consequences

- Redirect lookups route to one shard without fan-out.
- PostgreSQL remains the authoritative source for permanent mappings.
- Storage and write capacity can grow by adding shards.
- The application owns shard routing, shard-map rollout, migration, and connection-management complexity.
- Replica lag is acceptable because the system permits eventual consistency and primary fallback.
