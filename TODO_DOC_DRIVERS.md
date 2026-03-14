# TODO_DOC_DRIVERS

First-pass POD audit results for driver module documentation.

## Files Reviewed

- `dbio-mysql/lib/DBIO/MySQL.pm`
- `dbio-mysql/lib/DBIO/MySQL/Storage.pm`
- `dbio-mysql/lib/DBIO/MySQL/MariaDB.pm`
- `dbio-mysql/lib/DBIO/MySQL/Storage/MariaDB.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL.pm`
- `dbio-sqlite/lib/DBIO/SQLite.pm`
- `dbio-sqlite/lib/DBIO/SQLite/Storage.pm`
- `dbio-sqlite/lib/DBIO/SQLite/Test.pm`

## Priority 0

### `dbio-mysql/lib/DBIO/MySQL/Storage.pm`

- `is_replicating` still points to `DBIO::Storage::DBI::Replicated`.
- This should be updated to `DBIO::Replicated::Storage`.

## Priority 1

### `dbio-mysql/lib/DBIO/MySQL.pm`

- Generally solid and current.
- Migration notes are useful, but should be checked for tone and whether they
  belong in quite this much detail in the component module.
- Testing section includes replicated coverage and is a useful model.

### `dbio-postgresql/lib/DBIO/PostgreSQL.pm`

- Similar quality level to MySQL.
- Good factual coverage.
- Should be compared with MySQL and SQLite for a more deliberate shared style.

### `dbio-sqlite/lib/DBIO/SQLite.pm`

- Now looks thinner than MySQL/PostgreSQL in the testing section.
- Should likely mention offline shared-harness replicated coverage too, or
  intentionally explain why SQLite is documented differently.

### `dbio-sqlite/lib/DBIO/SQLite/Test.pm`

- Good candidate for expansion.
- It now sits at an important boundary between generic `DBIO::Test` behavior
  and SQLite-specific setup, but the POD still reads mainly as a small wrapper.

### `dbio-mysql/lib/DBIO/MySQL/MariaDB.pm`

- Fine and concise.
- Could be aligned more explicitly with the style of the main `DBIO::MySQL`
  component.

### `dbio-mysql/lib/DBIO/MySQL/Storage/MariaDB.pm`

- Solid overall.
- Should be checked for consistency with the replicated terminology cleanup in
  the MySQL storage family.

### `dbio-sqlite/lib/DBIO/SQLite/Storage.pm`

- Reasonably solid technical description.
- Still more implementation-centric than user-guiding; likely okay, but worth
  a sentence-level polish pass.

## Cross-Driver Findings

- MySQL and PostgreSQL component POD now expose a stronger test story than
  SQLite.
- Migration sections are similar enough that they should either:
  - share a more intentional template, or
  - be reduced and pushed into deeper docs
- Driver component modules should all make the split clear between:
  - schema component responsibilities
  - storage activation
  - testing guidance
  - migration context

## Out-Of-Scope But Noted

- `dbio-sqlite/t/lib/DBIOTest.pm` still appears to carry old replicated naming.
- That is not installed-module POD, but it should be cleaned separately to
  avoid historical namespace leakage in test scaffolding.
