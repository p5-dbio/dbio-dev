# TODO_DOC_LOW_PRIORITY

Low-priority installed-module POD work that still exists after the main pass.

## Done

- `DBIO::Test::Kubernetes` has already been cleaned enough that it is no
  longer the standout naming outlier from the audit.
- `DBIO::Test::Schema` and `DBIO::Test::Storage` now read like shared fixture
  docs instead of throwaway internal notes.
- sampled PostgreSQL `Diff` / `Introspect` / `Storage` feature modules were
  given their last small consistency pass.

## Remaining

- only unsampled leaf helpers remain, and only if a later release wants every
  tiny internal module to share the exact same tone
