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

### Reviewed at first-pass level

- core public modules
- replicated modules
- loader family
- top-level driver modules
- selected driver storage/test docs
- manuals and teaching docs
- key API surface modules
- PostgreSQL new feature modules

### Not reviewed in the same depth yet

- every single tiny fixture/result class under `dbio/lib/DBIO/Test/...`
- every single PostgreSQL leaf module individually beyond the sampled set
- low-level helper modules with tiny or no user-facing POD

These are mostly lower-value for prose polish than the main entrypoints and
conceptual docs. The remaining risk here is now low and concentrated in a few
special cases rather than an unknown broad area.

## Known Low-Priority Namespaces

These namespaces are installed, but much of their POD is fixture-oriented or
internal enough that they should not block the main documentation pass:

- `DBIO::Test::*`
- `DBIO::Test::Schema::*`
- `DBIO::Test::Namespace::*`
- `DBIO::Test::Taint::*`
- many tiny helper modules under `DBIO::Relationship::*`
- many tiny helper modules under `DBIO::Replicated::*`

One notable exception: `DBIO::Test::Kubernetes` is not low-value anymore,
because it still appears to expose DBIC-era naming in user-visible text and
defaults.

## Recommended Sequence

1. Finish factual cleanup in reviewed files.
2. Rewrite the high-level teaching/docs surface.
3. Re-run a lighter consistency pass over leaf modules.
4. Only then touch low-value fixture/test namespaces if needed.
