# TODO_DOC

Current documentation cleanup status for the DBIO workspace.

## What Is Done

- major factual drift from the DBIC-to-DBIO split was cleaned
- replicated storage is documented as a core DBIO feature
- top-level API docs were polished for `DBIO`, `DBIO::Cake`, `DBIO::Candy`,
  `DBIO::ResultSet`, `DBIO::ResultSource`, `DBIO::Test`, `DBIO::Core`,
  `DBIO::Relationship`, `DBIO::ResultSetColumn`, `DBIO::Storage`,
  `DBIO::Row`, and `DBIO::Schema`
- the three primary driver components now describe their role and test story
  more consistently
- the main loader entrypoints now describe their scope more clearly
- the key manuals called out in the first audit have had their high-value text
  cleanup pass
- stale TODO items that described already-fixed architecture issues were
  removed from the split audit files

## What Is Left

- optional family-wide loader modernization beyond the entrypoints
- later leaf-level polishing only if a future release wants a fully uniform
  voice across every helper module

See the split files for the current remaining backlog:

- `TODO_DOC_INVENTORY.md`
- `TODO_DOC_CORE.md`
- `TODO_DOC_API.md`
- `TODO_DOC_MANUALS.md`
- `TODO_DOC_DRIVERS.md`
- `TODO_DOC_LOADERS.md`
- `TODO_DOC_LOW_PRIORITY.md`
