# TODO_DOC_CORE

Current status for the core-module POD cleanup.

## Done

- `DBIO::Replicated` and `DBIO::Replicated::Storage` were expanded earlier to
  explain the core replicated model.
- `DBIO::Manual::DocMap` now distinguishes driver distributions from core
  capabilities such as replicated storage and `DBIO::Test`.
- obvious stale names such as old replicated namespaces, `dbicdump`, and
  `DBIC_TRACE` were already cleaned in the manuals and core POD.
- `DBIO::Test::Kubernetes` no longer exposes DBIC-era naming in user-visible
  defaults.

## Remaining

- no factual core-module cleanup remains in this tracked set
- future work here is optional sentence-level polish only
