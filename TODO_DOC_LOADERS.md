# TODO_DOC_LOADERS

Loader POD remains a separate cleanup track.

## Done

- factual cleanup and provenance fixes were already made where needed
- the PostgreSQL loader itself was rebuilt onto the new introspection path
- the main loader entrypoints in core, MySQL, PostgreSQL, and SQLite now
  describe their role more clearly

## Remaining

- only a broader family-wide modernization remains
- if touched again, modernize the deeper loader modules together to avoid mixed
  styles
