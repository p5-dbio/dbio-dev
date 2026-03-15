# TODO_DOC

Current documentation cleanup status for the DBIO workspace.

## What Is Done

- major factual drift from the DBIC-to-DBIO split was cleaned
- replicated storage is documented as a core DBIO feature
- top-level API docs were polished for `DBIO`, `DBIO::Cake`, `DBIO::Candy`,
  `DBIO::ResultSet`, `DBIO::ResultSource`, and `DBIO::Test`
- the three primary driver components now describe their role and test story
  more consistently
- stale TODO items that described already-fixed architecture issues were
  removed from the split audit files

## What Is Left

- medium-priority prose polish in deeper API and manual modules
- a later style decision for the loader-family POD
- low-priority consistency work in tiny fixture and PostgreSQL leaf modules

See the split files for the current remaining backlog:

- `TODO_DOC_INVENTORY.md`
- `TODO_DOC_CORE.md`
- `TODO_DOC_API.md`
- `TODO_DOC_MANUALS.md`
- `TODO_DOC_DRIVERS.md`
- `TODO_DOC_LOADERS.md`
- `TODO_DOC_LOW_PRIORITY.md`
