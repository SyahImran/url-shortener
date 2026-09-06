# URL Shortener Product Requirements

## Purpose

Build a high-scale URL shortening service that lets authenticated users create permanent short URLs and lets anyone resolve them quickly.

This document describes the desired product behavior and product-level constraints. Proposed technical design belongs under `.plans/`; implemented architecture belongs in `docs/architecture.md`.

## Goals

- Allow authenticated users to create short URLs.
- Allow public redirect resolution without authentication.
- Support both generated and user-selected Base62 short codes in one global namespace.
- Keep created mappings permanent and immutable.
- Provide best-effort reuse of an existing short URL for an exact destination string when no custom alias is requested.
- Support approximately 100 million creates per day and 1 billion redirects per day.
- Target less than 100 ms p95 redirect latency for clients near the deployment region.
- Support horizontal scaling and high availability within one region.

## Non-goals

- Expiration, deletion, mutation, or short-code reuse.
- User ownership, listing, or search APIs.
- Redirect analytics.
- URI normalization or destination reputation scanning.
- Application-level creation or redirect rate limits.
- Strong destination deduplication.
- API idempotency keys.
- Multi-region operation or regional disaster recovery.
- Distributed tracing.

## Functional requirements

### Create a short URL

- Only authenticated users may create short URLs.
- Any valid authenticated user is allowed to create one.
- The destination may use any syntactically valid URI scheme.
- The destination must be stored exactly as supplied; no URI canonicalization is performed.
- `customAlias` is optional.
- A custom alias may contain only `A-Z`, `a-z`, and `0-9`.
- Generated codes and custom aliases share one global namespace.
- Reserved application routes such as `health`, `api`, `admin`, and `metrics` cannot be used as aliases.
- An explicitly requested custom alias takes precedence over destination reuse.
- Successfully created mappings are permanent and immutable.

### Destination reuse

When no custom alias is requested, the service should attempt to reuse an existing short URL for the exact destination string.

This behavior is best effort rather than a uniqueness guarantee. Concurrent requests, cache eviction, or dependency failure may result in multiple permanent short codes for the same destination, and that remains valid product behavior.

### Redirect

- `GET /{shortCode}` is public.
- A successful lookup returns `302 Found` with the destination in the `Location` header.
- An unknown short code returns `404 Not Found` with a small JSON error response.
- `HEAD` support is not required.

### Health

- Expose `GET /health`.
- The endpoint reports minimal `healthy` or `unhealthy` status based on critical service dependencies.

## Scale requirements

| Operation | Daily volume | Average RPS | Approx. 2x peak |
| --- | ---: | ---: | ---: |
| Create | 100,000,000 | 1,157 | 2,315 |
| Redirect | 1,000,000,000 | 11,574 | 23,148 |
| Combined | 1,100,000,000 | 12,731 | 25,463 |

The service should be able to scale horizontally rather than relying on fixed excess capacity.

## Availability and consistency

- High availability is required, but no formal SLA is currently defined.
- The service is single-region; a full regional outage may make it unavailable until recovery.
- Eventual consistency is acceptable.
- Where possible, already-cached redirects should continue to work during authoritative-storage outages.
- Normal database durability guarantees are sufficient; no separate product durability SLA is defined.

## Performance

- Target redirect latency is less than 100 ms p95 for clients near the deployment region.
- Global latency is not guaranteed without future CDN or multi-region capability.
- The system should tolerate approximately 2x normal peak traffic and highly skewed access to hot short URLs.

## Observability

The product requires enough observability to operate the service safely, including structured logs, health checks, request/error/latency metrics, cache metrics, datastore metrics, and shard-level saturation metrics where applicable.

Distributed tracing is not required.

## Security and abuse boundaries

- Creation requires a valid externally issued JWT.
- Redirects remain public.
- Public traffic must use TLS in production.
- Credentials and full JWTs must not be logged.
- Reasonable request and destination-length limits are allowed.
- Infrastructure-level DDoS and connection protection is compatible with the absence of application-level rate limiting.

## Acceptance criteria

The initial product is acceptable when it can:

1. authenticate create requests and reject invalid authentication;
2. create generated permanent short URLs;
3. create valid custom Base62 aliases and reject conflicts or reserved aliases;
4. resolve known short codes with `302 Found`;
5. return `404 Not Found` for unknown short codes;
6. preserve the invariant that a successfully created short code never changes destination;
7. degrade without authoritative-data corruption when non-authoritative caching is unavailable;
8. meet the defined functional behavior under concurrency; and
9. demonstrate the stated redirect and creation load targets through measured load testing before production sizing is finalized.
