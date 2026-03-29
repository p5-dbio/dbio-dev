# CLAUDE.md — DBIO Development Workspace

## Overview

This workspace contains all DBIO distributions — a fork of DBIx::Class. Each subdirectory is a separate git repository released independently to CPAN.

## Key Differences from DBIx::Class

- Namespace: `DBIO::` replaces `DBIx::Class::`
- SQL::Abstract replaces SQL::Abstract::Classic
- Integrates DBIx::Class::TimeStamp, DBIx::Class::Helpers into core
- SQL::Translator is OPTIONAL — being replaced by DB-specific modules
- LIMIT/OFFSET: No more `sql_limit_dialect` string dispatch or `emulate_limit()`. Each driver's SQLMaker provides an `apply_limit` method. Default is `LIMIT ? OFFSET ?`. If you had custom `emulate_limit()` or `limit_dialect` workarounds, override `apply_limit` on your SQLMaker subclass instead.

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

- `copyright_holder` set automatically: `heritage = 1` → `DBIO & DBIx::Class Authors`, otherwise → `DBIO Authors`. Override via `copyright_holder = ...` in `[@DBIO]`.
- `LICENSE` file must be committed in the repo. Heritage repos use the original DBIx::Class license with DBIO attribution header. Non-heritage repos use a standard Perl_5 license. No `[License]` plugin — the file is gathered from git as-is.
- PodWeaver with `=attr` and `=method` collectors; `heritage = 1` uses `@DBIO::Heritage` which adds DBIx::Class attribution block.

**Exception**: `dbio/` core uses `[@DBIO] core = 1` — VersionFromMainModule, MakeMaker::Awesome, ExecDir, MetaResources etc.

## Versioning — NEVER bump $VERSION manually

`$VERSION` in driver modules is managed by `RewriteVersion::Transitional` (part of
`@Git::VersionManager`). It is rewritten automatically during `dzil release` based
on the git tag. **Never edit `$VERSION` by hand.** The value in source between
releases is the previous released version — that is correct and intentional.

`{{$NEXT}}` in `Changes` is filled in by `NextRelease` during `dzil release`.
Just add bullet points under `{{$NEXT}}` — never write a version line manually.

`dzil release` must be run by the user, not by AI.

## Standard Commands

```bash
# From any driver directory
dzil build          # Build
dzil test           # Test
dzil release        # Release to CPAN (user only, not AI)
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
- **Driver-specific test result classes** live in the driver's `lib/` namespace (e.g. `DBIO::PostgreSQL::Test::SequenceTest`), NOT in `t/lib/`. Load them via hashref syntax: `DBIO::Test::Schema->load_classes({ 'DBIO::PostgreSQL::Test' => ['SequenceTest'] })`. Never use `FindBin` or `use lib` in tests.

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
