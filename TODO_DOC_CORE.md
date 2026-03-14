# TODO_DOC_CORE

First-pass POD audit results for core DBIO module documentation.

## Files Reviewed

- `dbio/lib/DBIO.pm`
- `dbio/lib/DBIO/Test.pm`
- `dbio/lib/DBIO/Replicated.pm`
- `dbio/lib/DBIO/Replicated/Storage.pm`
- `dbio/lib/DBIO/Storage/DBI.pm`
- `dbio/lib/DBIO/Manual/DocMap.pod`
- `dbio/lib/DBIO/Manual/Features.pod`
- `dbio/lib/DBIO/Row.pm`
- `dbio/lib/DBIO/Schema.pm`
- `dbio/lib/DBIO/Compat/DBIxClass.pm`

## Priority 0

### `dbio/lib/DBIO/Manual/DocMap.pod`

- Driver overview needs a current architecture pass.
- Replicated storage is now core functionality and should be described as such,
  not only by omission from the driver list.
- "A boatload of DBIO features" should be replaced with stronger language.

### `dbio/lib/DBIO/Replicated.pm`

- Too thin for a core feature.
- Needs a real user-facing explanation of:
  - master vs replicant connect info
  - balancer selection
  - read vs write routing
  - relationship to `DBIO::Test`

### `dbio/lib/DBIO/Replicated/Storage.pm`

- Public POD is much smaller than the feature surface.
- Needs a conceptual section that explains:
  - backend wrapper model
  - pool responsibilities
  - balancer responsibilities
  - forced reliable reads vs balanced reads

## Priority 1

### `dbio/lib/DBIO.pm`

- Top-level English is uneven.
- Several intro sentences read like migrated source text rather than polished
  product documentation.
- Good candidate for a full prose polish pass after factual cleanup.

### `dbio/lib/DBIO/Test.pm`

- Strong content, but too much policy, migration history, and API contract are
  mixed into one flow.
- The `replicated => 1` story is there, but it should be easier to skim.
- Shared-driver-test guidance should likely become more example-driven.

### `dbio/lib/DBIO/Storage/DBI.pm`

- Large POD block likely carries too much conceptual weight in one place.
- Replication hooks like `is_replicating` and `lag_behind_master` need clearer
  framing now that `DBIO::Replicated::Storage` is first-class core API.

### `dbio/lib/DBIO/Manual/Features.pod`

- Contains stale dated claims and old community metrics.
- This looks like it needs reframing, not line edits.

## Priority 2

### `dbio/lib/DBIO/Row.pm`

- Replicated storage cross-reference should be checked in context to ensure the
  surrounding explanation still reads naturally after the architecture move.

### `dbio/lib/DBIO/Schema.pm`

- Same issue as `DBIO::Row`: the reference may be factually fine, but the
  surrounding text should be re-read for coherence.

### `dbio/lib/DBIO/Compat/DBIxClass.pm`

- Probably correct, but should be reviewed so migration-heavy explanation reads
  like intentional compatibility documentation, not leftover transition text.

## Notes

- Core docs should explicitly distinguish:
  - core capabilities
  - DB-specific drivers
  - migration guidance
  - testing guidance
- The replicated move back into core is the single biggest documentation gap.
