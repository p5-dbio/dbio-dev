# TODO_DOC_API

Current remaining POD work for the main API surface.

## Done

- `DBIO.pm` got a top-level prose pass.
- `DBIO::Cake` and `DBIO::Candy` now state their role in the three-style
  result-class story more clearly.
- `DBIO::ResultSet` and `DBIO::ResultSource` were tightened up at the entry
  sections and no longer lean as hard on old wording.
- `DBIO::Test` now reads more like a shared harness doc and less like migration
  notes glued to API details.

## Remaining

- `dbio/lib/DBIO/Core.pm`
  - could still say a little more about why it is the normal vanilla base class
- `dbio/lib/DBIO/Relationship.pm`
  - likely still older in tone than the refreshed top-level docs
- `dbio/lib/DBIO/ResultSetColumn.pm`
  - small module, but the wording could be sharpened later
- `dbio/lib/DBIO/Storage.pm`
  - serviceable, but still fairly dry compared with the refreshed entrypoints
