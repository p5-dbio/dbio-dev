---
name: dbio-driver
description: "How DBIO database drivers work: component system, storage classes, SQLMaker, and driver conventions"
user-invocable: false
allowed-tools: Read, Grep, Glob
model: sonnet
---

DBIO database drivers are separate CPAN distributions that provide database-specific
functionality. Each driver connects DBIO's generic ORM to a specific database engine.

## How Drivers Plug Into DBIO

DBIO uses a component system. A driver provides a **Schema component** that gets
loaded via `load_components`:

```perl
package MyApp::DB;
use base 'DBIO::Schema';
__PACKAGE__->load_components('PostgreSQL');  # loads DBIO::PostgreSQL
```

The component typically overrides `connection()` to set the correct storage class:

```perl
sub connection {
  my $self = shift;
  $self->storage_type('+DBIO::PostgreSQL::Storage');
  return $self->next::method(@_);
}
```

## Driver Structure

A minimal driver needs:

### 1. Schema Component (`DBIO::DriverName`)

The main module that users `load_components()`. Sets storage type on connection.

```perl
package DBIO::DriverName;
# ABSTRACT: DriverName support for DBIO
use base 'DBIO::Schema';

sub connection {
  my $self = shift;
  $self->storage_type('+DBIO::DriverName::Storage');
  return $self->next::method(@_);
}

1;
```

### 2. Storage Class (`DBIO::DriverName::Storage`)

Extends `DBIO::Storage::DBI` with database-specific behavior:

- Connection handling (DSN parsing, driver-specific connect options)
- Transaction semantics (savepoints, nested transactions)
- Last insert ID retrieval
- Bind attribute mapping for data types
- Database version detection

```perl
package DBIO::DriverName::Storage;
# ABSTRACT: Storage for DriverName databases
use base 'DBIO::Storage::DBI';

sub _rebless { ... }           # Auto-detect and rebless to correct storage
sub last_insert_id { ... }     # How to get the last auto-increment value
sub _svp_begin { ... }         # Savepoint support
sub _svp_release { ... }
sub _svp_rollback { ... }
sub with_deferred_fk_checks { ... }  # Deferred FK constraints

1;
```

### 3. SQLMaker (optional, `DBIO::DriverName::SQLMaker`)

Extends `DBIO::SQLMaker` for SQL dialect differences:

- LIMIT/OFFSET syntax
- String concatenation operator
- Quote characters
- Type casting
- Database-specific functions

```perl
package DBIO::DriverName::SQLMaker;
# ABSTRACT: SQL dialect for DriverName
use base 'DBIO::SQLMaker';

sub _build_limit { ... }    # e.g., LIMIT X OFFSET Y vs FETCH FIRST X ROWS
sub _quote_char { ... }     # e.g., backticks for MySQL, double-quotes for PG

1;
```

### 4. Result Component (optional, `DBIO::DriverName::Result`)

For database-specific column/table annotations on Result classes:

```perl
package MyApp::DB::Result::User;
use base 'DBIO::Core';
__PACKAGE__->load_components('DriverName::Result');
# Now has access to database-specific features
```

## Driver Naming Convention

| Part | Naming |
|------|--------|
| Distribution | `DBIO-DriverName` (e.g., `DBIO-PostgreSQL`) |
| Schema component | `DBIO::DriverName` |
| Storage class | `DBIO::DriverName::Storage` |
| SQLMaker | `DBIO::DriverName::SQLMaker` |
| Result component | `DBIO::DriverName::Result` |
| DBI driver | `DBD::DriverName` (e.g., `DBD::Pg`, `DBD::mysql`) |

## Key Storage Methods to Override

| Method | Purpose |
|--------|---------|
| `_rebless` | Auto-detect database type from DSN |
| `last_insert_id` | Retrieve auto-increment value |
| `_svp_begin/release/rollback` | Savepoint support |
| `with_deferred_fk_checks` | Defer FK constraint checking |
| `connect_call_*` | Connection-time setup (encoding, timezone) |
| `sqlt_type` | SQL::Translator type name (legacy) |
| `datetime_parser_type` | Which DateTime parser to use |
| `bind_attribute_by_data_type` | DBI bind attributes per column type |

## Testing Drivers

- Unit tests (no DB needed): SQLMaker tests using `DBIO::Test`
- Integration tests: require env vars like `DBIOTEST_PG_DSN`, `TEST_DBIO_POSTGRESQL_DSN`
- Always provide offline tests that verify SQL generation without a live database
- Use `prove -l -I../dbio/lib t/` to test with local DBIO core

## Build System

All drivers use `[@DBIO]` in dist.ini. Dependencies go in cpanfile.

```ini
name = DBIO-DriverName
author = DBIO & DBIx::Class Authors
license = Perl_5

[@DBIO]
```
