# TODO_DOC_API

First-pass POD audit results for the main DBIO API and teaching surface.

## Files Reviewed

- `dbio/lib/DBIO/Cake.pm`
- `dbio/lib/DBIO/Candy.pm`
- `dbio/lib/DBIO/Core.pm`
- `dbio/lib/DBIO/ResultSet.pm`
- `dbio/lib/DBIO/ResultSource.pm`
- `dbio/lib/DBIO/Relationship.pm`
- `dbio/lib/DBIO/Storage.pm`
- `dbio/lib/DBIO/ResultSetColumn.pm`

## Main Findings

### `dbio/lib/DBIO/Cake.pm`

- High-value teaching surface and likely one of the most important modules to
  polish after `DBIO.pm`.
- The implementation is modernized, but the POD should be re-read for:
  - consistency with current type vocabulary
  - clarity around import options
  - consistency with how `DBIO::Candy` and vanilla style are described
- This module should probably become the benchmark for concise, confident
  DBIO language.

### `dbio/lib/DBIO/Candy.pm`

- Still reads more like inherited historical code than newly-shaped DBIO docs.
- Likely needs:
  - stronger top-level explanation of where Candy fits versus Cake and vanilla
  - clearer import option examples
  - consistency with current DBIO naming and tone

### `dbio/lib/DBIO/Core.pm`

- Very small POD, factually fine, but underpowered for a module many users will
  encounter directly.
- Could probably explain more explicitly why `DBIO::Core` is the normal base
  class and how it relates to Cake/Candy/vanilla.

### `dbio/lib/DBIO/ResultSet.pm`

- Rich and useful, but still carries older explanatory style.
- Specific doc debt:
  - the Moose/Moo subsection should be reviewed carefully now that the project
    is generally moving away from role-heavy framing
  - sentence rhythm and examples feel older than the newer driver/core docs
- This is a high-value polish target because it teaches the mental model.

### `dbio/lib/DBIO/ResultSource.pm`

- Strong conceptual content, but likely too dense for newcomers.
- A good candidate for structure tightening and cross-link cleanup.
- Should align terminology tightly with `Manual::ResultClass` and `Glossary`.

### `dbio/lib/DBIO/Relationship.pm`

- Helpful and example-heavy, but clearly older in voice.
- Could benefit from:
  - more modern examples
  - stronger distinction between declaration helpers and lower-level base docs
  - less historical “walk through tables” prose if the same idea is already in
    the manuals

### `dbio/lib/DBIO/Storage.pm`

- Mostly fine but dry and very base-class oriented.
- Check whether the split between `DBIO::Storage` and `DBIO::Storage::DBI`
  could be made clearer for readers who are not core contributors.

### `dbio/lib/DBIO/ResultSetColumn.pm`

- Abstract and description are technically fine, but the English is not at the
  same quality level as the stronger modules.
- Example: “helpful methods for messing with a single column of the resultset”
  is casual in a weak way rather than intentionally friendly.

## Cross-Module Opportunities

- The teaching surface should deliberately align three ways of writing result
  classes:
  - vanilla `DBIO::Core`
  - `DBIO::Candy`
  - `DBIO::Cake`
- `ResultSet`, `ResultSource`, and `Relationship` should use a more unified
  vocabulary for:
  - result class
  - result source
  - row/result object
  - relationship traversal

## Priority

- High:
  - `DBIO::Cake`
  - `DBIO::Candy`
  - `DBIO::ResultSet`
  - `DBIO::ResultSource`
- Medium:
  - `DBIO::Relationship`
  - `DBIO::Core`
  - `DBIO::ResultSetColumn`
  - `DBIO::Storage`
