# Data Schema

This document provides the human-readable data model for the URL shortener. Once executable migrations or schema definitions exist, they are authoritative for exact database structure.

## ShortUrl

The authoritative persistent entity is intentionally minimal.

| Field | Type | Constraints | Purpose |
| --- | --- | --- | --- |
| `short_code` | `VARCHAR(32)` | Primary key, Base62 | Permanent generated code or custom alias |
| `destination_uri` | `TEXT` | Not null | Exact destination URI supplied by the client |
| `created_at` | `TIMESTAMPTZ` | Not null, server default | Creation timestamp |

Conceptual schema:

```sql
CREATE TABLE short_urls (
    short_code      VARCHAR(32) PRIMARY KEY,
    destination_uri TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Deliberately omitted fields

The design does not require `user_id`, `expires_at`, `updated_at`, `deleted_at`, `click_count`, or a persistent `custom_alias` flag.

Authentication gates creation but stored mappings have no user ownership. URLs are permanent and immutable. Multiple short codes may legally point to the same exact destination because destination deduplication is best effort.

## Redis data

Redis is not authoritative storage.

Redirect cache:

```text
url:<short_code> -> <destination_uri>
```

Best-effort destination lookup:

```text
dest:<SHA-256(exact_destination_uri)> -> <short_code>
```

Redis entries may be evicted without affecting the authoritative mapping.
