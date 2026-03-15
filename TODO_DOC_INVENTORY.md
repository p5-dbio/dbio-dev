# TODO_DOC_INVENTORY

Coverage map for the split POD audit files in this workspace root.

## Audit Artifacts

- `TODO_DOC.md`
  - master briefing, rubric, and audit strategy
- `TODO_DOC_CORE.md`
  - top-level core modules, replicated docs, major core manuals references
- `TODO_DOC_LOADERS.md`
  - loader family across core and drivers
- `TODO_DOC_DRIVERS.md`
  - MySQL/PostgreSQL/SQLite driver entrypoints and selected storage docs
- `TODO_DOC_API.md`
  - user-facing API surface such as `DBIO::Core`, `DBIO::ResultSet`,
    `DBIO::ResultSource`, `DBIO::Relationship`, `DBIO::Storage`,
    `DBIO::Cake`, `DBIO::Candy`
- `TODO_DOC_MANUALS.md`
  - manual and teaching surface tone/clarity review
- `TODO_DOC_PG_FEATURES.md`
  - PostgreSQL-native DDL/deploy/introspect/diff module family
- `TODO_DOC_LOW_PRIORITY.md`
  - remaining installed test-support POD and sampled PostgreSQL leaf modules

## Coverage Status

### Reviewed and mostly acted on

- core public modules
- replicated modules
- top-level driver modules
- key manuals and teaching docs
- main API entrypoints
- PostgreSQL feature entrypoints
- shared test fixtures that had user-visible POD

### Still mostly backlog, not rewrite work

- the wider loader family
- every tiny fixture/result helper under `dbio/lib/DBIO/Test/...`
- every PostgreSQL leaf module beyond the sampled set

## Known Low-Priority Namespaces

These namespaces are installed, but much of their POD is fixture-oriented or
internal enough that they should not block the main documentation pass:

- `DBIO::Test::*`
- `DBIO::Test::Schema::*`
- `DBIO::Test::Namespace::*`
- `DBIO::Test::Taint::*`
- many tiny helper modules under `DBIO::Relationship::*`
- many tiny helper modules under `DBIO::Replicated::*`

## Recommended Sequence

1. Leave the finished high-level docs alone unless facts change again.
2. Decide separately whether loader POD should get a family-wide modernization.
3. Do a later consistency pass over leaf modules only if release time allows.
