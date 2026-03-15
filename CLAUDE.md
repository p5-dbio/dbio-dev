# CLAUDE.md — DBIO Development Workspace

## Overview

This workspace contains all DBIO distributions — a fork of DBIx::Class. Each subdirectory is a separate git repository released independently to CPAN.

## Key Differences from DBIx::Class

- Namespace: `DBIO::` replaces `DBIx::Class::`
- SQL::Abstract replaces SQL::Abstract::Classic
- Integrates DBIx::Class::TimeStamp, DBIx::Class::Helpers into core
- SQL::Translator is OPTIONAL — being replaced by DB-specific modules

## Distributions

### Core

- **dbio/** — DBIO core ORM. Has its own complex dist.ini (NOT `[@DBIO]`). Includes replicated storage support in core.
- **dbio-dzil/** — `Dist::Zilla::PluginBundle::DBIO` + `Pod::Weaver::PluginBundle::DBIO`. Self-bootstrapping via `[Bootstrap::lib]`.

### Actively Supported Drivers

- **dbio-postgresql/** — PostgreSQL driver. Most advanced: introspection via pg_catalog, deploy via test-and-compare, enums, schemas, RLS, indexes. Uses Moo.
- **dbio-mysql/** — MySQL + MariaDB driver.
- **dbio-sqlite/** — SQLite driver.

### Async

- **dbio-postgresql-async/** — Async PostgreSQL via EV::Pg. Bypasses DBI entirely, uses libpq directly. Pipeline mode, LISTEN/NOTIFY, COPY. Returns Future objects.

### Extracted Drivers (not yet actively developed)

- **dbio-db2/** — IBM DB2
- **dbio-firebird/** — Firebird/InterBase
- **dbio-informix/** — Informix
- **dbio-mssql/** — Microsoft SQL Server
- **dbio-oracle/** — Oracle
- **dbio-sybase/** — Sybase/ASE

### Not distributions

- **dbio-admin/** — Admin tools (no dist.ini yet)
- **dbio-diffs/** — Patch archive, not a distribution

## Build System

All driver distributions use `[@DBIO]` plugin bundle:

```ini
name = DBIO-DriverName
author = DBIO & DBIx::Class Authors
license = Perl_5

[@DBIO]
```

- Version from git tags (first release: 0.900)
- PodWeaver with `=attr` and `=method` collectors
- Custom copyright: DBIO Authors + DBIx::Class Authors

**Exception**: `dbio/` core uses its own dist.ini (VersionFromMainModule, MakeMaker::Awesome, MetaNoIndex etc.)

## Standard Commands

```bash
# From any driver directory
dzil build          # Build
dzil test           # Test
dzil release        # Release to CPAN
prove -l t/         # Quick test
prove -l -I../dbio/lib t/   # Test with local DBIO core

# Dependencies
cpanm --installdeps .
```

## POD Conventions

- `# ABSTRACT:` required on every .pm file
- `=attr name` after `has` → collected into ATTRIBUTES section
- `=method name` after `sub` → collected into METHODS section
- Never write NAME, VERSION, AUTHORS, COPYRIGHT sections (auto-generated)
- POD is inline (after code), not at end of file
- Use `L<DBIO::Module>` for cross-references, never manual URLs

## Testing

- **DBIO core tests MUST use `DBIO::Test::Storage` (fake/virtual storage), NEVER `dbi:SQLite` or any real database connection.** The fake storage captures SQL and supports mocks via `$storage->mock(qr/.../, \@rows)`. Real database testing belongs in the driver distributions (dbio-sqlite, dbio-postgresql, etc.).
- Driver integration tests use env vars:
  - PostgreSQL: `TEST_DBIO_POSTGRESQL_DSN` or `DBIOTEST_PG_DSN`
  - MySQL: `DBIOTEST_MYSQL_DSN`
  - etc.
- `DBIO::Test` provides shared test utilities
- `DBIO::Test->init_schema` without arguments uses `DBIO::Test::Storage` — this is the correct default for core tests

## Architecture

```
DBIO::Schema → DBIO::ResultSource → DBIO::ResultSet → DBIO::Row
DBIO::Storage → DBIO::Storage::DBI (base for drivers)
DBIO::SQLMaker (SQL generation)

Drivers plug in via load_components:
  __PACKAGE__->load_components('PostgreSQL');
  → sets storage_type to DBIO::PostgreSQL::Storage
```

## Copyright

- DBIO: Copyright (C) 2026 DBIO Authors
- Based on: DBIx::Class, Copyright (C) 2005-2025 DBIx::Class Authors
