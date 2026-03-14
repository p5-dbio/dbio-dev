# DBIO Development Workspace

Development workspace for **DBIO** (DBI Objects) — Relational mapping. Joins itself. Fully native. Everything included.

A modern fork of [DBIx::Class](https://metacpan.org/pod/DBIx::Class).

## Workspace Setup

All DBIO repositories are checked out as subdirectories of this workspace.
The workspace itself is a git repo that holds shared configuration
(`.editorconfig`, `CLAUDE.md`, etc.) — the actual distributions live in their
own repos underneath.

```bash
# 1. Clone the workspace
git clone git@github.com:p5-dbio/dbio-dev.git
cd dbio-dev

# 2. Clone all repositories (copy-paste block)
git clone git@github.com:p5-dbio/dbio.git
git clone git@github.com:p5-dbio/dbio-dzil.git
git clone git@github.com:p5-dbio/dbio-postgresql.git
git clone git@github.com:p5-dbio/dbio-mysql.git
git clone git@github.com:p5-dbio/dbio-sqlite.git
git clone git@github.com:p5-dbio/dbio-replicated.git
git clone git@github.com:p5-dbio/dbio-db2.git
git clone git@github.com:p5-dbio/dbio-firebird.git
git clone git@github.com:p5-dbio/dbio-informix.git
git clone git@github.com:p5-dbio/dbio-mssql.git
git clone git@github.com:p5-dbio/dbio-oracle.git
git clone git@github.com:p5-dbio/dbio-sybase.git

# 3. Install core dependencies
cd dbio && cpanm --installdeps . && cd ..
```

After setup your directory should look like:

```
dbio-dev/
  .editorconfig        # Shared editor settings (2-space indent, UTF-8, LF)
  CLAUDE.md            # Central AI assistant instructions
  dbio/                # Core ORM
  dbio-dzil/           # Shared Dist::Zilla build system
  dbio-postgresql/     # PostgreSQL driver
  dbio-mysql/          # MySQL + MariaDB driver
  dbio-sqlite/         # SQLite driver
  ...
```

## Repositories

| Directory | Distribution | Description |
|-----------|-------------|-------------|
| `dbio/` | DBIO | Core ORM (Schema, ResultSet, Row, Storage) |
| `dbio-dzil/` | Dist-Zilla-PluginBundle-DBIO | Shared build system for all DBIO dists |
| `dbio-postgresql/` | DBIO-PostgreSQL | PostgreSQL driver (introspection, deploy, enums, schemas) |
| `dbio-mysql/` | DBIO-MySQL | MySQL + MariaDB driver |
| `dbio-sqlite/` | DBIO-SQLite | SQLite driver |
| `dbio-replicated/` | DBIO-Replicated | Master/slave replication storage |
| `dbio-db2/` | DBIO-DB2 | IBM DB2 driver |
| `dbio-firebird/` | DBIO-Firebird | Firebird/InterBase driver |
| `dbio-informix/` | DBIO-Informix | Informix driver |
| `dbio-mssql/` | DBIO-MSSQL | Microsoft SQL Server driver |
| `dbio-oracle/` | DBIO-Oracle | Oracle driver |
| `dbio-sybase/` | DBIO-Sybase | Sybase/ASE driver |

## Working with Drivers

Drivers link against the local DBIO core via `-I../dbio/lib`:

```bash
# Test a driver against local core
cd dbio-postgresql
prove -l -I../dbio/lib t/

# Build a driver
dzil build

# Test core
cd dbio
prove -l t/
```

## What Changed from DBIx::Class

- **Namespace**: `DBIO::` instead of `DBIx::Class::`
- **SQL::Abstract**: Replaces SQL::Abstract::Classic
- **Integrated extensions**: TimeStamp, Helpers merged into core
- **Sugar syntax**: `DBIO::Candy` (from DBIx::Class::Candy) and `DBIO::Cake` (from DBIx::Class::ResultDDL) included
- **Database-specific drivers**: Each database is a separate CPAN distribution
- **SQL::Translator optional**: PostgreSQL driver uses pg_catalog introspection instead

## Building

All driver distributions use `[@DBIO]` — a [Dist::Zilla](https://metacpan.org/pod/Dist::Zilla) plugin bundle:

```ini
# dist.ini — that's all you need
name = DBIO-DriverName
author = DBIO & DBIx::Class Authors
license = Perl_5

[@DBIO]
```

Version comes from git tags. Copyright is dual: DBIO Authors + DBIx::Class Authors.

## Documentation

POD uses inline `=attr` and `=method` commands (collected by PodWeaver):

```perl
has name => ( is => 'ro', required => 1 );

=attr name

The name. Required.

=cut
```

## Contributing

1. Fork the relevant repository
2. Create a feature branch
3. Write tests
4. Submit a pull request

Each repository is independent — PRs go to the specific distribution's repo, not here.

## Copyright

Copyright (C) 2026 DBIO Authors
Portions Copyright (C) 2005-2025 DBIx::Class Authors

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
