# TODO_DOC_LOW_PRIORITY

Low-priority installed-module POD work that still exists after the main pass.

## Done

- `DBIO::Test::Kubernetes` has already been cleaned enough that it is no
  longer the standout naming outlier from the audit.

## Remaining

- tiny `DBIO::Test::*` fixture modules only need attention if stale naming
  leaks back into visible POD
- small PostgreSQL `Diff::*` / `Introspect::*` leaf modules can wait for a
  later consistency pass
