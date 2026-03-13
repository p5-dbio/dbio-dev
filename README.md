# DBIO Development Workspace

Development workspace for **DBIO** — a modern fork of [DBIx::Class](https://metacpan.org/pod/DBIx::Class), the Perl ORM.

## Getting Started

```bash
git clone https://github.com/Perl5/dbio-dev.git
cd dbio-dev

# Clone all DBIO repositories
git clone https://github.com/Perl5/DBIO.git dbio
git clone https://github.com/Perl5/DBIO-PostgreSQL.git dbio-postgresql
git clone https://github.com/Perl5/DBIO-MySQL.git dbio-mysql
git clone https://github.com/Perl5/DBIO-SQLite.git dbio-sqlite
# ... etc.

# Install DBIO core dependencies
cd dbio && cpanm --installdeps . && cd ..

# Test a driver
cd dbio-postgresql
prove -l -I../dbio/lib t/
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

## What Changed from DBIx::Class

- **Namespace**: `DBIO::` instead of `DBIx::Class::`
- **SQL::Abstract**: Replaces SQL::Abstract::Classic
- **Integrated extensions**: TimeStamp, Helpers merged into core
- **Database-specific drivers**: Each database is a separate CPAN distribution
- **SQL::Translator optional**: PostgreSQL driver uses pg\_catalog introspection instead

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
