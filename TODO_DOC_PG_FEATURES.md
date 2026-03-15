# TODO_DOC_PG_FEATURES

Remaining documentation work for the PostgreSQL-native feature family.

## Current Status

This family no longer has notable factual drift. The loader and introspection
work is current enough that the remaining tasks are all consistency work.

## Remaining

- `dbio-postgresql/lib/DBIO/PostgreSQL/DDL.pm`
  - could still say a little more clearly when to use it directly versus
    through higher-level deploy helpers
- `dbio-postgresql/lib/DBIO/PostgreSQL/Deploy.pm`
  - good baseline doc already; only a light polish pass remains
- `dbio-postgresql/lib/DBIO/PostgreSQL/Diff.pm`
  - could use a slightly stronger note on review-oriented output and operation
    ordering
- `dbio-postgresql/lib/DBIO/PostgreSQL/Introspect.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Storage.pm`
  - both are in decent shape and only need sentence-level consistency work
