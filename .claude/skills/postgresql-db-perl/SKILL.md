---
name: postgresql-db-perl
description: "PostgreSQL database knowledge for Perl driver development (DBD::Pg, pg_catalog, PG-specific types and features)"
user-invocable: false
allowed-tools: Read, Grep, Glob
model: sonnet
---

PostgreSQL knowledge relevant for Perl database driver development.

## DBD::Pg (Perl DBI Driver)

- `DBD::Pg` is the DBI driver for PostgreSQL
- Connection: `DBI->connect("dbi:Pg:dbname=mydb;host=localhost", $user, $pass)`
- Supports: COPY, LISTEN/NOTIFY, prepared statements, server-side cursors
- Array columns return/accept Perl arrayrefs
- JSONB columns: send as JSON string, receive as JSON string (decode yourself)
- Bytea: automatic escaping/unescaping via `DBD::Pg`

## pg_catalog Introspection

Key system catalogs for driver development:

| Catalog | Purpose |
|---------|---------|
| `pg_namespace` | Schemas (namespaces) |
| `pg_class` | Tables, views, indexes, sequences |
| `pg_attribute` | Columns |
| `pg_type` | Data types (including enums, composites) |
| `pg_enum` | Enum values |
| `pg_index` | Index definitions |
| `pg_am` | Access methods (btree, gin, gist, brin, hash) |
| `pg_constraint` | Constraints (PK, FK, UNIQUE, CHECK, EXCLUDE) |
| `pg_trigger` | Triggers |
| `pg_proc` | Functions/procedures |
| `pg_extension` | Installed extensions |
| `pg_policy` | Row Level Security policies |
| `pg_sequence` | Sequence parameters |

### Useful Functions

```sql
pg_get_indexdef(oid)        -- Full CREATE INDEX statement
pg_get_constraintdef(oid)   -- Constraint definition
pg_get_triggerdef(oid)      -- Trigger definition
pg_get_expr(node, relid)    -- Expression from node tree
format_type(type_oid, mod)  -- Human-readable type name
```

## PostgreSQL Type System

### Native Types

| Category | Types |
|----------|-------|
| Integer | `smallint`, `integer`, `bigint`, `serial`, `bigserial` |
| Float | `real`, `double precision`, `numeric(p,s)` |
| String | `text`, `varchar(n)`, `char(n)` |
| Binary | `bytea` |
| Boolean | `boolean` |
| Date/Time | `timestamp`, `timestamptz`, `date`, `time`, `timetz`, `interval` |
| UUID | `uuid` |
| JSON | `json`, `jsonb` |
| Array | `anytype[]` (e.g., `text[]`, `integer[]`) |
| Network | `inet`, `cidr`, `macaddr` |
| Geometric | `point`, `line`, `box`, `circle`, `polygon`, `path` |
| Range | `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange` |

### Extension Types

| Extension | Types |
|-----------|-------|
| `pgvector` | `vector(n)` — AI embeddings |
| `postgis` | `geometry`, `geography` |
| `timescaledb` | hypertables (not a type, a table variant) |

### User-Defined Types

- **Enums**: `CREATE TYPE status AS ENUM ('active', 'inactive')`
- **Composites**: `CREATE TYPE address AS (street text, city text)`
- **Ranges**: `CREATE TYPE floatrange AS RANGE (subtype = float8)`
- **Domains**: `CREATE DOMAIN email AS text CHECK (VALUE ~ '@')`

## Index Types

| Access Method | Use Case | Operators |
|---------------|----------|-----------|
| `btree` | Equality, range, sorting (default) | `<`, `<=`, `=`, `>=`, `>` |
| `hash` | Equality only | `=` |
| `gin` | Arrays, JSONB, full-text, trigram | `@>`, `<@`, `?`, `?&`, `?\|`, `@@` |
| `gist` | Geometry, range, full-text | `<<`, `>>`, `&&`, `@>`, `<@` |
| `brin` | Large sequential tables | Same as btree, physically ordered |
| `ivfflat` | pgvector approximate NN | `<->`, `<=>`, `<#>` |
| `hnsw` | pgvector approximate NN (better) | `<->`, `<=>`, `<#>` |

### Index Features

- **Partial indexes**: `WHERE active = true`
- **Expression indexes**: `ON lower(name)`
- **INCLUDE columns**: Non-key columns in index
- **CONCURRENTLY**: Non-blocking index creation

## Schemas (Namespaces)

PostgreSQL schemas are first-class namespaces:

```sql
CREATE SCHEMA auth;
CREATE TABLE auth.users (...);
SET search_path TO auth, public;
```

- Default schema: `public`
- System schemas: `pg_catalog`, `information_schema`
- `search_path` controls unqualified name resolution

## Row Level Security (RLS)

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;  -- applies to table owner too
CREATE POLICY user_policy ON users
    FOR ALL
    USING (user_id = current_setting('app.user_id')::integer);
```

## Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;      -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";   -- uuid_generate_v4()
CREATE EXTENSION IF NOT EXISTS pg_trgm;       -- trigram similarity
CREATE EXTENSION IF NOT EXISTS pgvector;      -- vector similarity search
CREATE EXTENSION IF NOT EXISTS postgis;       -- geographic data
```

## Testing with PostgreSQL

- Use `TEST_DBIO_POSTGRESQL_DSN` or `DBIOTEST_PG_DSN` env vars
- Each test should use its own temp database or schema
- `CREATE DATABASE` / `DROP DATABASE` for isolation
- Use transactions with rollback for quick cleanup
- `pg_prove` for pgTAP (SQL-level testing)
