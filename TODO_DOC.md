# TODO_DOC

## Goal

Create a focused POD audit backlog for the DBIO ecosystem after the recent
DBIC-to-DBIO migration work and the move of replicated storage back into the
DBIO core.

This file is for analysis and prioritization, not for drafting replacement
text yet.

Related split result files:

- `TODO_DOC_INVENTORY.md`
- `TODO_DOC_CORE.md`
- `TODO_DOC_LOADERS.md`
- `TODO_DOC_DRIVERS.md`
- `TODO_DOC_API.md`
- `TODO_DOC_MANUALS.md`
- `TODO_DOC_PG_FEATURES.md`

## Scope

In scope:
- Module POD in installed modules under `lib/` across `dbio`, `dbio-mysql`,
  `dbio-postgresql`, and `dbio-sqlite`
- Factual drift after recent architecture changes
- Weak or non-idiomatic English
- Underexplained complex behavior
- Duplicate explanations that should be consolidated
- Places where the docs should say more, not just say it better

Out of scope for this pass:
- `README.md`
- `CLAUDE.md`
- generated `AGENTS.md`
- historical notes under `docs/`
- test-only helper modules under `t/lib/` unless they expose an obvious naming
  or terminology leak worth fixing later

## Reference Baseline

Use these recent shifts as the mental baseline when judging POD quality:

- `DBIO::*` is the primary namespace. Historical `DBIx::Class::*` references
  should either be clearly marked as migration history or removed.
- Drivers are split into dedicated distributions such as `DBIO::MySQL`,
  `DBIO::PostgreSQL`, and `DBIO::SQLite`.
- Replicated storage is no longer a separate driver distribution in active
  architecture; it is now core `DBIO` functionality under
  `DBIO::Replicated::*`.
- Replicated storage is no longer a Moose/Moo-driven composition story. The
  public docs should describe normal classes and core behavior, not old role
  mechanics.
- Shared tests should document `DBIO::Test` as the common harness, including
  fake storage, driver-specific offline SQL generation, and `replicated => 1`
  coverage.
- Sugar styles (`DBIO::Cake`, `DBIO::Candy`, vanilla `DBIO::Core`) are now
  part of the DBIO teaching surface and should read like current first-class
  usage, not like side notes.
- Avoid stale dates, stale release numbers, and historical community metrics
  unless they are explicitly framed as historical context.

## Audit Rubric

When reviewing a module POD, classify issues into one or more of these buckets:

- `FACT`: architecture or naming is outdated
- `ENGLISH`: wording is awkward, weak, or non-idiomatic
- `CLARITY`: complex behavior is mentioned but not explained well enough
- `REDUNDANCY`: the same explanation appears in too many places
- `EXPAND`: the topic deserves more documentation, not only cleanup
- `STYLE`: old POD structure conflicts with the newer `# ABSTRACT` /
  PodWeaver-oriented style

## Priority 0: Factual Drift Or Missing Core Story

- `dbio/lib/DBIO/Manual/DocMap.pod`
  - `FACT`: the driver-distribution section no longer reflects the current
    picture cleanly because replicated storage is now core, not a separate
    driver distribution.
  - `ENGLISH`: "A boatload of DBIO features" is weaker than the newer tone we
    want.
  - `CLARITY`: the manual map should distinguish DB-specific drivers from core
    capabilities like replicated storage.

- `dbio-mysql/lib/DBIO/MySQL/Storage.pm`
  - `FACT`: `is_replicating` still says it is intended for
    `DBIO::Storage::DBI::Replicated`; this must point to
    `DBIO::Replicated::Storage`.

- `dbio/lib/DBIO/Replicated.pm`
  - `EXPAND`: too thin for a core feature.
  - Missing public explanation of:
    - how master vs replicants are configured
    - how balancers fit into the flow
    - when reads go to master vs balanced backends
    - how `DBIO::Test` interacts with replicated mode

- `dbio/lib/DBIO/Replicated/Storage.pm`
  - `EXPAND`: the implementation is substantial, but the POD surface is still
    too small for the importance of the feature.
  - `CLARITY`: needs a public model of pool, balancer, backend wrapper, and
    read/write routing.

## Priority 1: High-Value Quality Improvements

- `dbio/lib/DBIO.pm`
  - `ENGLISH`: top-level wording is uneven. Example: "This code in the next
    step can be generated automatically..." reads clumsily.
  - `CLARITY`: this is the entrypoint for the whole project and should read
    like polished product documentation.
  - `REDUNDANCY`: check whether introductory teaching overlaps too much with
    `Manual::QuickStart` and `Manual::Intro`.

- `dbio/lib/DBIO/Test.pm`
  - `CLARITY`: good content, but the structure is dense and mixes policy,
    migration notes, and API details.
  - `EXPAND`: add a crisper story for shared driver tests, especially
    `storage_type => ...` plus `replicated => 1`.
  - `REDUNDANCY`: decide what belongs here versus in driver test helper POD.

- `dbio/lib/DBIO/Storage/DBI.pm`
  - `CLARITY`: replication hooks such as `is_replicating` and
    `lag_behind_master` exist, but their relationship to
    `DBIO::Replicated::Storage` is not obvious enough.
  - `REDUNDANCY`: this POD is very large; some conceptual material may belong
    in cross-references or narrower manuals instead of one giant narrative.

- `dbio-sqlite/lib/DBIO/SQLite.pm`
  - `CLARITY`: testing section is now thinner than MySQL/PostgreSQL and does
    not mention the new replicated shared-harness story.
  - `REDUNDANCY`: decide whether all drivers should share a more consistent
    testing subsection template.

- `dbio-sqlite/lib/DBIO/SQLite/Test.pm`
  - `EXPAND`: should explain more clearly how it layers on top of `DBIO::Test`
    and which options remain generic versus SQLite-specific.

- `dbio-postgresql/lib/DBIO/PostgreSQL.pm`
- `dbio-mysql/lib/DBIO/MySQL.pm`
- `dbio-sqlite/lib/DBIO/SQLite.pm`
  - `REDUNDANCY`: migration/test sections are close enough that they may want a
    more deliberate shared style.
  - `ENGLISH`: compare the tone and sentence rhythm across all three driver
    modules and normalize where needed.

## Priority 2: Systematic Style Cleanup

- Loader family in core and drivers:
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
  - `STYLE`: many still use old `=head1 NAME`, `AUTHORS`, and `LICENSE`
    sections rather than the newer abstract-driven style.
  - `CLARITY`: decide whether this whole family should be modernized together
    or intentionally left in legacy style for now.

- `dbio/lib/DBIO/Manual/Features.pod`
  - `FACT`: contains stale dated claims and historical release/community
    numbers from 2010.
  - `ENGLISH`: the title and tone are dated.
  - `CLARITY`: may need a modern reframe rather than line edits.

- `dbio/lib/DBIO/Row.pm`
- `dbio/lib/DBIO/Schema.pm`
  - `CLARITY`: both cross-reference replicated storage, but the surrounding
    explanation should be checked now that replicated is a core feature again.

- `dbio/lib/DBIO/Compat/DBIxClass.pm`
  - `CLARITY`: historical/migration-heavy docs may be correct, but should be
    reviewed so they read as deliberate compatibility docs rather than leftover
    migration prose.

## Secondary But Worth Tracking

- `dbio/lib/DBIO/Manual/DocMap.pod` and `dbio/lib/DBIO/Manual/Features.pod`
  should be reviewed together so naming, tone, and information hierarchy stay
  consistent.

- `dbio-sqlite/t/lib/DBIOTest.pm` still appears to contain old replicated
  naming (`::DBI::Replicated`). This is outside the installed-module-POD scope,
  but worth a separate cleanup check.

## Suggested Audit Order For Tomorrow

1. Fix factual drift first:
   - `DocMap`
   - stale replicated references
   - any old architecture wording
2. Expand the replicated story in core:
   - `DBIO::Replicated`
   - `DBIO::Replicated::Storage`
   - cross-links from `DBIO::Test` and `DBIO::Storage::DBI`
3. Polish the top-level teaching surface:
   - `DBIO.pm`
   - driver component modules
   - `DBIO::Test`
4. Decide whether the Loader family gets a broad style modernization or only
   targeted factual fixes in this round.
5. Re-read the manuals after the module pass to consolidate duplicated
   explanations instead of letting them drift further apart.

## Agent Briefing For Tomorrow

If this audit is handed to another agent, the review should not be a grep-only
rename pass. It should read each POD section as prose and ask:

- Is this still true after the current architecture?
- Does this sound like native, confident English?
- Does this explain enough for a user who does not already know the internals?
- Is this explanation repeated somewhere better?
- Would a short cross-link be better than another paragraph here?
