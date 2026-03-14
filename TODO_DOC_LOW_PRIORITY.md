# TODO_DOC_LOW_PRIORITY

Second-pass audit notes for lower-priority installed module POD.

## Files Reviewed

### DBIO test-support modules

- `dbio/lib/DBIO/Test/Schema.pm`
- `dbio/lib/DBIO/Test/Storage.pm`
- `dbio/lib/DBIO/Test/DateTimeParser.pm`
- `dbio/lib/DBIO/Test/Kubernetes.pm`

### PostgreSQL feature leaf modules

- `dbio-postgresql/lib/DBIO/PostgreSQL/Introspect/Columns.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Introspect/Tables.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Introspect/Policies.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Diff/Column.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Diff/Table.pm`
- `dbio-postgresql/lib/DBIO/PostgreSQL/Diff/Policy.pm`

## Main Findings

### `dbio/lib/DBIO/Test/Kubernetes.pm`

- This is the strongest remaining factual/naming outlier in the low-priority
  group.
- The module still carries obvious `dbic` naming in user-visible places:
  - generated namespace: `dbic-test-$suffix`
  - default database names like `dbic_test`
- The synopsis comment also says “PostgreSQL tests moved to
  dbio-postgresql distribution”, which is plausible but should be re-read for
  current accuracy and tone.
- POD structure still uses `=head1 NAME`.
- This file should get an explicit DBIC-to-DBIO terminology cleanup pass.

### `dbio/lib/DBIO/Test/Storage.pm`

- Good and useful POD.
- A small polish pass would help:
  - sentence rhythm
  - consistency with `DBIO::Test`
  - maybe an explicit note about interaction with hybrid driver storage classes
- Not urgent.

### `dbio/lib/DBIO/Test/Schema.pm`

- Solid and concise.
- Could perhaps mention more explicitly that this schema is the shared fixture
  base used by external driver distributions through `DBIO::Test`.

### `dbio/lib/DBIO/Test/DateTimeParser.pm`

- No real POD issue; it only has an abstract and is small enough that this is
  fine.

## PostgreSQL Leaf Module Findings

### General

- These leaf modules are concise, modern, and mostly consistent.
- No major stale architecture or namespace drift found in the sampled leaf set.
- Their docs are intentionally narrow, which is acceptable because the family
  entrypoints already carry the conceptual overview.

### Minor consistency issues

- The `Diff::*` and `Introspect::*` leaf docs should eventually be checked for:
  - quoting style consistency in examples
  - uniform use of `schema.table` terminology
  - whether one or two leaf modules should mention limitations or assumptions
    around SQL rendering and identifier quoting

### `dbio-postgresql/lib/DBIO/PostgreSQL/Diff/Policy.pm`

- Summary examples and wording are acceptable, but the human-readable summary
  strings shown in POD should probably be normalized against the actual output
  style used across the whole diff family.

### `dbio-postgresql/lib/DBIO/PostgreSQL/Diff/Table.pm`

- The “empty shell, columns added separately” concept is clear, but should stay
  aligned with whatever final explanation `DBIO::PostgreSQL::Diff` uses.

### `dbio-postgresql/lib/DBIO/PostgreSQL/Introspect/*`

- Sampled modules are fine.
- This family does not need a rewrite-first strategy; it only needs a light
  consistency pass after top-level docs are stabilized.

## Bulk Fixture Namespaces

The following installed namespaces were surveyed only at a metadata level:

- `DBIO::Test::Schema::*`
- `DBIO::Test::Namespace::*`
- `DBIO::Test::Taint::*`

Findings:

- Their abstracts are mostly straightforward and appropriately fixture-oriented.
- They are not a good use of prose-polish effort until the main public docs are
  settled.
- They should only be touched if:
  - stale DBIC naming leaks into visible POD, or
  - broader style work requires a consistency cleanup.

## Conclusion

The remaining installed-module POD surface is now largely mapped.

The only clearly actionable low-priority factual cleanup item is
`DBIO::Test::Kubernetes.pm`. Everything else in this pass is primarily
consistency and polish, not urgent architecture repair.
