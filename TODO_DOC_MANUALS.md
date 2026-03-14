# TODO_DOC_MANUALS

First-pass POD audit results for the DBIO manuals and broader teaching docs.

## Files Reviewed

- `dbio/lib/DBIO/Manual/QuickStart.pod`
- `dbio/lib/DBIO/Manual/Troubleshooting.pod`
- `dbio/lib/DBIO/Manual/Glossary.pod`
- `dbio/lib/DBIO/Manual/Intro.pod`
- `dbio/lib/DBIO/Manual/Reading.pod`
- `dbio/lib/DBIO/Manual/ResultClass.pod`
- plus previously noted:
  - `dbio/lib/DBIO/Manual/DocMap.pod`
  - `dbio/lib/DBIO/Manual/Features.pod`

## Main Findings

### Abstract And Tone Problems

- `QuickStart`: abstract starts lowercase and should be normalized.
- `Troubleshooting`: “Got a problem? Shoot it.” is not the right tone for the
  current documentation standard.
- `Glossary`: “Clarification of terms used.” is harmless but weak.
- `Features`: “A boatload of DBIO features...” is dated and informal in a way
  that lowers perceived quality.

### Historical Drift

- `Troubleshooting` still documents `DBIC_TRACE` rather than the DBIO naming.
- `QuickStart` still refers to `dbicdump`; this likely needs to become
  `dbiodump` or otherwise be explicitly explained if the old name is still
  intentionally present.
- `Features` includes dated claims and community metrics from 2010.

### Teaching Surface Consistency

- `Intro`, `QuickStart`, and `ResultClass` overlap enough that they should be
  reviewed together instead of independently polished.
- `Glossary` terminology should be checked against the wording in:
  - `DBIO::ResultSet`
  - `DBIO::ResultSource`
  - `DBIO::Row`
  - `DBIO::Core`

### `dbio/lib/DBIO/Manual/Intro.pod`

- Useful, but strongly inherited from older DBIx::Class teaching voice.
- Likely needs a modern DBIO-first rewrite rather than only cleanup.

### `dbio/lib/DBIO/Manual/QuickStart.pod`

- Good onboarding value.
- Needs a factual cleanup pass and a language polish pass.
- Should align with modern tooling names and the new sugar-story.

### `dbio/lib/DBIO/Manual/Troubleshooting.pod`

- Probably the highest factual-risk manual after `Features`.
- Needs a dedicated stale-terminology pass.

### `dbio/lib/DBIO/Manual/Reading.pod`

- Meta-doc for writers is useful, but it itself reads old.
- Should be checked against the current PodWeaver / abstract-driven style.

### `dbio/lib/DBIO/Manual/ResultClass.pod`

- Important bridge between conceptual docs and API docs.
- Should be aligned closely with `DBIO::Core`, `DBIO::Cake`, and
  `DBIO::Candy`.

## Recommended Work Order

1. `DocMap`
2. `QuickStart`
3. `Troubleshooting`
4. `Intro`
5. `ResultClass`
6. `Glossary`
7. `Features`
8. `Reading`
