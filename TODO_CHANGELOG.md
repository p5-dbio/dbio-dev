# DBIO::ChangeLog — Design & Implementation Plan

## Goal

Integrated change tracking for DBIO as a core component. Combines the best ideas from
DBIx::Class::Events (state_at, custom events), Journal (automatic changesets via txn_do),
and AuditLog (normalized column-level diffs).

Ships as part of DBIO core — no extra distribution needed.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│ Schema                                                  │
│  load_components('Schema::ChangeLog')                   │
│  ├─ txn_do override → automatic ChangeSet creation      │
│  ├─ changelog_user / changelog_session accessors         │
│  └─ changelog_sources (opt: restrict tracked tables)     │
├─────────────────────────────────────────────────────────┤
│ Result (per tracked table)                              │
│  load_components('ChangeLog')                           │
│  ├─ insert/update/delete overrides                      │
│  ├─ log_event('name', \%details) — custom events        │
│  ├─ state_at($timestamp) — temporal reconstruction      │
│  ├─ changelog_exclude_columns(qw/.../)                  │
│  └─ has_many → per-table ChangeLog entries              │
├─────────────────────────────────────────────────────────┤
│ Driver (optional override layer)                        │
│  e.g. DBIO::PostgreSQL::ChangeLog                       │
│  ├─ override storage types (jsonb, native timestamps)   │
│  ├─ override deploy (partitioning, GIN indexes)         │
│  ├─ override write path (LISTEN/NOTIFY, triggers)       │
│  ├─ override query path (jsonb operators, DB functions)  │
│  └─ falls through to base ChangeLog via next::method    │
├─────────────────────────────────────────────────────────┤
│ Storage Tables (auto-generated, driver-customizable)    │
│  changelog_set     — transaction grouping               │
│  <source>_changelog — per-table change entries          │
└─────────────────────────────────────────────────────────┘
```

---

## Storage Schema

### `changelog_set` (one global table)

| Column       | Type         | Notes                              |
|--------------|--------------|------------------------------------|
| id           | integer PK   | auto-increment                     |
| user_id      | varchar(255) | nullable, from changelog_user()    |
| session_id   | varchar(255) | nullable, from changelog_session() |
| created_at   | datetime     | NOT NULL, set automatically        |

### `<source>_changelog` (one per tracked ResultSource)

| Column       | Type         | Notes                                         |
|--------------|--------------|-----------------------------------------------|
| id           | integer PK   | auto-increment                                |
| changeset_id | integer FK   | → changelog_set.id, nullable (custom events)  |
| row_id       | varchar(255) | serialized PK of tracked row                  |
| event        | varchar(64)  | 'insert', 'update', 'delete', or custom name  |
| changes      | text         | JSON: column diffs or custom event details    |
| created_at   | datetime     | NOT NULL, set automatically                   |

**`changes` format for built-in events:**

```json
// insert — all column values
{"name": "Miles Davis", "genre": "Jazz"}

// update — only changed columns with old+new
{"name": ["Miles Davis", "Miles Dewey Davis"]}

// delete — all column values at time of deletion
{"name": "Miles Dewey Davis", "genre": "Jazz"}

// custom event
{"approved_by": "admin", "reason": "verified"}
```

Update diffs use `[old, new]` arrays. This enables both state_at() reconstruction
and direct "what changed" queries.

---

## API Design

### Schema Level

```perl
package MyApp::Schema;
use base 'DBIO::Schema';

__PACKAGE__->load_components('Schema::ChangeLog');

# Optional: restrict which sources are tracked (default: all)
__PACKAGE__->changelog_sources([qw/Artist Album/]);
```

### Result Level

```perl
package MyApp::Schema::Result::Artist;
use base 'DBIO::Core';

__PACKAGE__->load_components('ChangeLog');
__PACKAGE__->table('artist');
__PACKAGE__->add_columns(
    id   => { data_type => 'integer', is_auto_increment => 1 },
    name => { data_type => 'varchar', size => 100 },
    password_hash => { data_type => 'varchar', size => 255 },
);

# Exclude sensitive columns from changelog
__PACKAGE__->changelog_exclude_columns(qw/password_hash/);
```

### Usage

```perl
# Automatic tracking — insert/update/delete just work
$schema->changelog_user($current_user->id);

$schema->txn_do(sub {
    my $artist = $schema->resultset('Artist')->create({ name => 'Coltrane' });
    $artist->update({ name => 'John Coltrane' });
    # Both changes grouped in one ChangeSet automatically
});

# Custom events
$artist->log_event(approved => { by => $admin->id, note => 'verified' });

# Temporal queries
my $state = $artist->state_at('2026-01-15 12:00:00');
# → { id => 1, name => 'Coltrane' }  (before the update)

# Query changelog directly
my @changes = $artist->changelog->search({ event => 'update' })->all;
```

### Disabling Tracking

```perl
# Disable for a transaction (bulk imports, migrations)
$schema->txn_do(sub {
    local $schema->changelog_disabled(1);
    # ... mass operations without changelog overhead
});

# Or per-operation
$artist->update({ name => 'New' }, { changelog => 0 });
```

---

## Implementation Details

### Component: `DBIO::Schema::ChangeLog`

- **Override `txn_do`**: Wrap in changeset creation (like Journal)
  - Use `local` scoping for current changeset ID (natural rollback safety)
  - Nested txn_do reuses parent changeset (no nesting of changesets)
- **`changelog_user($id)`**: Set user for current/future changesets
- **`changelog_session($id)`**: Set session identifier
- **`changelog_disabled`**: Flag to skip tracking
- **`deploy_changelog`**: Create changelog tables (separate from main deploy)
- **`changelog_sources`**: Class accessor, list of tracked source names

### Component: `DBIO::ChangeLog`

Row-level component. Loaded via `load_components('ChangeLog')`.

**Method overrides:**

- **`insert`**: After `next::method`, create changelog entry with all column values
  (excluding changelog_exclude_columns)
- **`update`**: Before `next::method`, capture `get_dirty_columns`. After success,
  create changelog entry with `{col => [old, new]}` diffs. Skip if no tracked
  columns changed.
- **`delete`**: Before `next::method`, capture all column values. Create changelog
  entry, then delete.

**New methods:**

- **`log_event($name, \%details)`**: Create custom changelog entry. No changeset
  required (changeset_id nullable for custom events outside transactions).
- **`state_at($timestamp)`**: Reconstruct row state at given time by replaying
  changelog entries backward (like Events). Returns hashref or undef if row
  didn't exist.
- **`changelog`**: Accessor for the has_many relationship to the changelog table.
- **`changelog_exclude_columns(@cols)`**: Class accessor for excluded columns.

### Auto-generated ResultSource for changelog tables

When `Schema::ChangeLog` initializes, it dynamically creates ResultSource classes
for `changelog_set` and each `<source>_changelog` table. No manual Result classes
needed.

Pattern: Use `DBIO::ResultSource::Table->new` + `register_source` at schema
composition time (similar to how Journal's `DB.pm` creates journal sources).

### Driver Override Layer

The ChangeLog is designed so drivers can customize behavior without replacing the
whole system. The base ChangeLog defines **hookable methods** that drivers override
via the standard component/MRO chain:

```perl
# Base methods that drivers can override:

changelog_column_definitions()
  # Returns column definitions for the changelog entry table.
  # Base: { changes => { data_type => 'text' } }
  # PG:   { changes => { data_type => 'jsonb' } }
  # MySQL: { changes => { data_type => 'json' } }

changelog_deploy_hook($source_name, $table_ddl)
  # Called after changelog table DDL is generated, before execution.
  # PG: add GIN index on changes column, set up partitioning by date
  # MySQL: add generated columns for indexed JSON paths

changelog_serialize_changes(\%changes)
  # Serialize the changes hash for storage.
  # Base: JSON::PP->new->encode(\%changes)
  # Driver can use native DB serialization or optimize format

changelog_deserialize_changes($raw)
  # Deserialize stored changes back to hashref.
  # Base: JSON::PP->new->decode($raw)

changelog_write_entry(\%entry)
  # Write a single changelog entry to storage.
  # Base: $self->create_related('changelog', \%entry)
  # Driver could batch writes, use COPY, or async insert

changelog_notify($event, \%entry)
  # Called after a changelog entry is written. No-op in base.
  # PG: pg_notify('changelog', $payload) for realtime listeners
  # Could also feed into message queues, webhooks, etc.

changelog_state_at_query($timestamp)
  # Build the query for state_at() reconstruction.
  # Base: Perl-side replay of changelog entries
  # PG: Could use window functions or jsonb_agg for DB-side reconstruction
```

**How drivers hook in:**

```perl
# In DBIO::PostgreSQL (the driver component)
# When ChangeLog is also loaded, the PG-specific overrides activate:

package DBIO::PostgreSQL::ChangeLog;
use base 'DBIO::ChangeLog';

sub changelog_column_definitions {
    my $self = shift;
    my %cols = $self->next::method(@_);
    $cols{changes}{data_type} = 'jsonb';
    return %cols;
}

sub changelog_deploy_hook {
    my ($self, $source_name, $ddl) = @_;
    $self->next::method(@_);
    # Add GIN index for efficient JSON queries
    $self->result_source->storage->dbh_do(sub {
        $_[1]->do("CREATE INDEX idx_${source_name}_cl_changes
                    ON ${source_name}_changelog USING GIN (changes)");
    });
}

sub changelog_notify {
    my ($self, $event, $entry) = @_;
    $self->result_source->storage->dbh_do(sub {
        $_[1]->do("SELECT pg_notify('changelog', ?)",
            undef, encode_json({ source => $source_name, event => $event }));
    });
}
```

**Loading order**: The driver component registers itself in the MRO chain. When a
Result class loads both `ChangeLog` and e.g. `PostgreSQL`, the driver's ChangeLog
methods take precedence but chain to the base via `next::method`. No special wiring
needed — this is just how DBIO's component system already works.

### PK Serialization

`row_id` stores the primary key as a string:
- Single-column PK: just the value (`"42"`)
- Multi-column PK: JSON array (`["US","42"]`)

This avoids Journal's limitation of single-column integer PKs only.

### state_at() Algorithm

```
1. Query changelog for this row where created_at <= $timestamp
   ORDER BY created_at DESC, id DESC
2. Walk backward through entries:
   - If most recent is 'delete' → return undef
   - If 'update': merge diffs (new values) into state hash
   - If 'insert': merge all values, stop (we have full state)
3. Return \%state
```

Update diffs store `[old, new]` — for reconstruction we use:
- Walking backward: take `old` value from diff (undoing changes)
- Walking forward from insert: take `new` value from diff

---

## File Layout

Core (in `dbio/lib/DBIO/`):

```
lib/DBIO/
├── ChangeLog.pm                    # Row-level component (base)
├── ChangeLog/
│   ├── Set.pm                      # ChangeSet result source (auto-generated base)
│   └── Entry.pm                    # ChangeLog entry result source (auto-generated base)
└── Schema/
    └── ChangeLog.pm                # Schema-level component
```

Driver overrides (in each driver distribution, Phase 4):

```
dbio-postgresql/lib/DBIO/PostgreSQL/ChangeLog.pm
dbio-mysql/lib/DBIO/MySQL/ChangeLog.pm
dbio-sqlite/lib/DBIO/SQLite/ChangeLog.pm
```

Driver ChangeLog files are optional — they only exist when a driver has
DB-specific optimizations. The base ChangeLog works with any database.

---

## Implementation Phases

### Phase 1: Core Tracking (MVP)

1. `DBIO::Schema::ChangeLog` — schema component with txn_do override, changeset
   creation, user/session tracking
2. `DBIO::ChangeLog` — row component with insert/update/delete overrides
3. Auto-generation of changelog ResultSources at schema composition time
4. `deploy_changelog()` for table creation
5. Tests using `DBIO::Test::Storage` (fake storage, no real DB)

### Phase 2: Temporal Queries

1. `state_at($timestamp)` implementation
2. `changelog` relationship accessor
3. Tests for state reconstruction across insert/update/delete sequences

### Phase 3: Custom Events & Polish

1. `log_event($name, \%details)`
2. `changelog_disabled` flag
3. `changelog => 0` per-operation opt-out
4. `changelog_exclude_columns` with `modify_changelog_value` callback for masking

### Phase 4: Driver Integration

Base ChangeLog defines hookable methods (see "Driver Override Layer" above).
Each driver overrides only what it needs via next::method.

**PostgreSQL** (`DBIO::PostgreSQL::ChangeLog`):
- `jsonb` for changes column with GIN index
- `pg_notify()` in `changelog_notify` for realtime event streaming
- DB-side `state_at` via window functions + `jsonb_agg` (optional fast path)
- Table partitioning by month on `created_at` for large-scale deployments
- `jsonb_path_query` for efficient "which rows changed column X?" queries

**MySQL** (`DBIO::MySQL::ChangeLog`):
- `JSON` column type for changes (falls back to `LONGTEXT` on old versions)
- `JSON_TABLE` / `JSON_EXTRACT` for indexed virtual columns
- Generated columns for frequently-queried JSON paths

**SQLite** (`DBIO::SQLite::ChangeLog`):
- `TEXT` column with JSON1 extension functions (`json_extract`, `json_each`)
- Lightweight — no special deploy hooks needed

**Other drivers**: Work out of the box with base ChangeLog (TEXT + Perl-side JSON).
Driver-specific optimizations can be added incrementally.

---

## Open Questions

- **Bulk operations**: `$rs->update(\%vals)` and `$rs->delete` bypass Row methods.
  Options: (a) don't track them (like Events/Journal), (b) override ResultSet
  methods too (like AuditLog), (c) track at Storage level. Recommendation: start
  with (a), document the limitation, add (b) later if needed.

- **Cascade deletes**: When a parent row is deleted and children cascade-delete in
  the DB, the changelog won't capture child deletions. Same limitation as all
  existing modules. Could add FK-aware cascade tracking later.

- **Separate DB storage**: AuditAny's collector pattern allows writing audit data to
  a different database. Worth adding? Recommendation: defer. Same-DB is simpler
  and sufficient for most cases. Can add a `changelog_storage` option later.

- **Retention/cleanup**: Should we provide built-in pruning? Recommendation: defer.
  Users can DELETE from changelog tables directly or add cron jobs.

- **Table naming**: `<source>_changelog` uses the source moniker lowercased.
  Alternative: configurable via `changelog_table_name` class accessor.

---

## Complexity Assessment

Phase 1 (MVP) is straightforward — roughly 300-400 lines across the two components
plus test scaffolding. The patterns are well-established in DBIO (component loading,
method overrides via next::method, dynamic source registration).

**Verdict: We can implement Phase 1 now.** Phases 2-4 can follow incrementally.
