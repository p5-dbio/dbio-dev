# TODO_DOC_CORE

Current remaining POD work for core DBIO modules.

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

- `dbio/lib/DBIO/Storage/DBI.pm`
  - very large POD; still worth a later clarity pass around replication hooks
    and where conceptual material should live
- `dbio/lib/DBIO/Compat/DBIxClass.pm`
  - still worth a tone pass so the compatibility story reads intentionally,
    not like leftover migration text
- `dbio/lib/DBIO/Row.pm`
- `dbio/lib/DBIO/Schema.pm`
  - only light prose cleanup remains around core cross-references
