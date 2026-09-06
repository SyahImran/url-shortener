# ADR 0001: Use Snowflake-style IDs encoded as Base62

## Status

Accepted

## Context

The service must support approximately 100 million creates per day, generate short codes across horizontally scaled application instances, and avoid a central database sequence or dedicated ID service on the critical write path. Generated codes and user-selected aliases share one global Base62 namespace.

## Decision

Generate globally unique numeric IDs using a Snowflake-style scheme composed from time, worker identity, and a per-time-unit sequence. Encode generated IDs using one fixed Base62 alphabet:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

Each active generator must hold a unique worker identity. Clock rollback must be detected and must not be allowed to produce potentially duplicate IDs.

Generated codes are inserted directly. A primary-key conflict triggers a bounded retry with a newly generated ID.

## Consequences

- Application instances can generate IDs independently without a central sequence service.
- Generated/generated collisions should not occur when worker allocation and clock handling are correct.
- Worker-ID allocation and clock rollback become operational concerns.
- Base62 alphabet order becomes a durable persisted encoding contract.
- Encoded values may reveal structural properties of the underlying generated IDs.
