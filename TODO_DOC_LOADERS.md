# TODO_DOC_LOADERS

First-pass POD audit results for Loader-related module documentation.

## Files Reviewed

- `dbio/lib/DBIO/Loader.pm`
- `dbio/lib/DBIO/Loader/DBI.pm`
- `dbio/lib/DBIO/Loader/Table.pm`
- `dbio/lib/DBIO/Loader/RelBuilder.pm`
- `dbio/lib/DBIO/Loader/DBObject.pm`
- `dbio/lib/DBIO/Loader/DBI/Writing.pm`
- `dbio/lib/DBIO/Loader/DBI/Component/QuotedDefault.pm`
- `dbio/lib/DBIO/Loader/RelBuilder/Compat/*`
- `dbio-mysql/lib/DBIO/MySQL/Loader.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Loader.pm`
- `dbio-sqlite/lib/DBIO/SQLite/Loader.pm`

## Main Findings

### Style Drift

- The Loader family still uses older POD structure much more heavily than the
  rest of the modernized codebase.
- Repeated old-style sections include:
  - `=head1 NAME`
  - `AUTHORS`
  - `LICENSE`
- This may be acceptable if deliberate, but right now it reads more like a
  partly-unmigrated documentation style.

### Consolidation Opportunity

- The same high-level explanation pattern repeats across core and driver loader
  modules:
  - what loader does
  - where driver-specific subclasses live
  - where to find more options
- This should be reviewed for consolidation and stronger cross-linking.

### Driver Loader English

- The driver loader descriptions are serviceable, but read older and flatter
  than the newer top-level driver modules.
- MySQL/PostgreSQL/SQLite loader POD should be normalized to a common quality
  level if these files are touched at all.

## File-Specific Notes

### `dbio/lib/DBIO/Loader.pm`

- Overall factual framing looks current.
- Still reads like a direct descendant of older DBIx::Class loader docs.
- Worth checking for sentence-level polish and modern DBIO naming rhythm.

### `dbio/lib/DBIO/Loader/DBI.pm`

- Same issue as above: likely correct, but stylistically older.
- Good candidate for a targeted cleanup if the Loader family gets a full pass.

### `dbio-mysql/lib/DBIO/MySQL/Loader.pm`

- Uses old-style POD structure.
- Description is minimal and could probably say more about the role of the
  driver-specific loader versus core `DBIO::Loader`.

### `dbio-postgresql/lib/DBIO/PostgreSQL/Loader.pm`

- Same structure/style issue as MySQL loader.
- PostgreSQL introspection complexity suggests there may be room for better
  explanation than the current brief description.

### `dbio-sqlite/lib/DBIO/SQLite/Loader.pm`

- Same old-style POD structure.
- The `rescan` behavior is actually interesting and worth keeping, but the
  surrounding file should still be checked for style modernization.

## Decision Needed

Before editing, decide whether Loader POD should:

1. stay largely in legacy structure with only factual fixes, or
2. be modernized as a family to match the current DBIO style.

Mixing both approaches will likely produce inconsistent docs.
