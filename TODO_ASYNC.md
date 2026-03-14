# DBIO Async Database Support — Design & Implementation Plan

## Goal

Event-loop-agnostic async database support for DBIO. The core defines abstract
interfaces and async-aware API surfaces. Actual async implementations ship as
separate distributions (e.g. `DBIO::EV::Pg`, `Net::Async::DBIO`, `Mojo::DBIO`)
so DBIO core never depends on any specific event loop.

The first target is PostgreSQL via [EV::Pg](https://metacpan.org/pod/EV::Pg),
which provides non-blocking libpq access with pipeline mode (124k queries/sec),
prepared statements, COPY, and LISTEN/NOTIFY — bypassing DBI entirely.

---

## Architecture Overview

```
DBIO::Schema → DBIO::ResultSet → DBIO::Row
                    ↓
              DBIO::Storage (abstract interface)
              ├── DBIO::Storage::DBI (sync, DBI-based — current)
              │   ├── DBIO::PostgreSQL::Storage
              │   ├── DBIO::MySQL::Storage
              │   └── DBIO::SQLite::Storage
              └── DBIO::Storage::Async (async base — new)
                  ├── DBIO::EV::Pg::Storage      (separate dist, uses EV::Pg)
                  ├── Net::Async::DBIO::Storage   (separate dist, uses IO::Async)
                  └── Mojo::DBIO::Storage         (separate dist, uses Mojo::IOLoop)
```

Key insight: `DBIO::Storage` already defines abstract methods (`select`,
`insert`, `update`, `delete`, `select_single`) as virtual methods. The async
path does NOT replace these — it adds parallel `*_async` variants that return
Futures instead of immediate results.

---

## What the Core Needs to Provide

### 1. Refined Abstract Storage Interface

Currently `DBIO::Storage` defines `select`, `insert`, `update`, `delete` as
virtual methods, and `DBIO::Storage::DBI` implements them with DBI-specific
internals (`_dbh_execute`, `dbh_do`, `$sth->execute()`, etc.).

The refactoring goal is NOT to change the sync path, but to:

- Document the abstract contract more precisely (argument signatures, return values)
- Add async method signatures to the abstract base
- Ensure `Storage::DBI` remains the only sync implementation (no breakage)

**New virtual methods on `DBIO::Storage`:**

```perl
# Async variants — return DBIO::Future objects
# Default implementation: throw "not implemented" (like current virtual methods)

sub select_async       { die "Virtual method!" }
sub select_single_async { die "Virtual method!" }
sub insert_async       { die "Virtual method!" }
sub update_async       { die "Virtual method!" }
sub delete_async       { die "Virtual method!" }
```

These have the same argument signatures as their sync counterparts but return
a Future instead of immediate results.

### 2. Future Interface (Not Implementation)

DBIO core defines `DBIO::Future` as a **role/interface specification**, not a
concrete implementation. This avoids depending on `Future.pm` while ensuring
all async distributions return compatible objects.

```perl
package DBIO::Future;
# ABSTRACT: Interface specification for async result objects

# A DBIO::Future must support these methods:
#
#   ->then(sub { my @result = @_; ... })   # success callback, returns new Future
#   ->catch(sub { my $error = shift; ... }) # error callback, returns new Future
#   ->get()                                 # block until resolved, return result
#   ->is_ready()                            # true if resolved (success or failure)
#   ->is_failed()                           # true if resolved with error
#
# Async distributions map to their event loop's native Future:
#   DBIO::EV::Pg       → Future.pm (Future::XS or Future::PP)
#   Net::Async::DBIO   → Future.pm (native)
#   Mojo::DBIO         → Mojo::Promise (wrapped to match interface)
```

**Architecture: Three layers**

1. **`DBIO::Future`** — Interface/contract definition. Documents the required
   methods (`then`, `catch`, `get`, `is_ready`, `is_failed`). Optionally
   validates that an object implements the contract. Not a base class —
   async distributions bring their own Future (Future.pm, Mojo::Promise, etc.).

2. **`DBIO::Test::Future`** — Synchronous mock Future for the test suite.
   Resolves immediately, no event loop needed. Just like `DBIO::Test::Storage`
   is a fake storage, this is a fake Future. Allows core to test async method
   signatures without installing any async framework.

3. **Real Futures** — Provided by async distributions via `future_class`:

```perl
# DBIO::Future — Interface contract (in core)
package DBIO::Future;
# ABSTRACT: Future interface contract for async DBIO operations

# A DBIO-compatible Future must implement:
#   ->then(sub { my @result = @_; ... })   # success chain, returns new Future
#   ->catch(sub { my $error = shift; ... }) # error chain, returns new Future
#   ->get()                                 # block until resolved, return result
#   ->is_ready()                            # true if resolved (success or failure)
#   ->is_failed()                           # true if resolved with error

sub validate {
  my ($class, $obj) = @_;
  for (qw(then catch get is_ready is_failed)) {
    croak "$obj does not implement $_" unless $obj->can($_);
  }
}
```

```perl
# DBIO::Test::Future — Synchronous mock (in core, for testing only)
package DBIO::Test::Future;
# ABSTRACT: Synchronous mock Future for DBIO test suite

sub done { bless { ready => 1, result => [@_[1..$#_]] }, $_[0] }
sub fail { bless { ready => 1, failed => 1, error => $_[1] }, $_[0] }
sub is_ready  { $_[0]->{ready} }
sub is_failed { $_[0]->{failed} }
sub get       { die $_[0]->{error} if $_[0]->{failed}; @{ $_[0]->{result} } }
sub then  { ... }  # immediate chaining
sub catch { ... }  # immediate error handling
```

Each storage declares its Future class:

```perl
# Default in DBIO::Storage (and DBIO::Test::Storage)
sub future_class { 'DBIO::Test::Future' }

# In DBIO::EV::Pg::Storage (separate distribution)
sub future_class { 'Future' }  # uses Future.pm from CPAN

# In Mojo::DBIO::Storage (separate distribution)
sub future_class { 'Mojo::Promise' }
```

This means core tests work without any async framework:

```perl
# t/test/async_interface.t — no EV, no IO::Async, no Mojo
my $schema = DBIO::Test->init_schema;
my $future = $schema->resultset('Artist')->all_async;
isa_ok($future, 'DBIO::Test::Future');
ok($future->is_ready, 'test future resolves immediately');
my @artists = $future->get;
```

### 3. Async-Aware ResultSet Methods

Separate methods rather than modal flag. The `_async` suffix is explicit,
discoverable, and avoids the complexity of a ResultSet that changes behavior
based on construction-time options.

```perl
# Sync (unchanged — these stay exactly as they are)
my @rows   = $rs->all;
my $row    = $rs->first;
my $count  = $rs->count;
my $row    = $rs->single;
my $row    = $rs->next;    # cursor iteration

# Async equivalents
my $f = $rs->all_async;
$f->then(sub {
    my @rows = @_;
    # process rows
});

my $f = $rs->first_async;
$f->then(sub {
    my $row = shift;
    # process row
});

my $f = $rs->count_async;
$f->then(sub {
    my $count = shift;
    # ...
});
```

**Implementation approach for ResultSet async methods:**

```perl
# In DBIO::ResultSet

sub all_async {
    my $self = shift;
    my $storage = $self->result_source->storage;

    # Delegate to storage's async select
    my $attrs = $self->_resolved_attrs;
    return $storage->select_async(
        $attrs->{from}, $attrs->{select}, $attrs->{where}, $attrs,
    )->then(sub {
        my @rows_data = @_;
        # Inflate rows (same logic as sync all, but deferred)
        return map { $self->_construct_results($_) } @rows_data;
    });
}

sub first_async {
    my $self = shift;
    return $self->search(undef, { rows => 1 })->all_async->then(sub {
        return $_[0];  # first element
    });
}

sub count_async {
    my $self = shift;
    my $storage = $self->result_source->storage;
    my $attrs = $self->_resolved_attrs;
    return $storage->select_single_async(
        $attrs->{from}, [{ count => '*' }], $attrs->{where}, $attrs,
    )->then(sub {
        return $_[0]->[0];
    });
}
```

**Why separate methods instead of `{ async => 1 }`?**

- No ambiguity — caller always knows whether they get data or a Future
- No risk of forgetting the flag and getting a Future where rows were expected
- Grep-friendly: searching for `all_async` finds all async usage
- Sync code path untouched — zero risk of regressions

### 4. Connection Pool Abstraction

Sync storage uses one connection per schema (current behavior, unchanged).
Async storage needs a pool because multiple queries can be in-flight
simultaneously, and transactions pin a connection.

```perl
package DBIO::Storage::Pool;
# ABSTRACT: Abstract connection pool interface

# Interface:
#   ->acquire()           # Returns a Future that resolves to a connection
#   ->release($conn)      # Return connection to pool
#   ->acquire_txn()       # Acquire a connection pinned for transaction use
#   ->size()              # Current pool size
#   ->available()         # Number of idle connections
#   ->max_size()          # Configured maximum
```

The pool lives in the async storage, not in core. Core only defines the
interface shape. Each async distribution implements pooling appropriate to
its event loop.

**Sync backward compatibility:**

```perl
package DBIO::Storage::Pool::Single;
# Trivial "pool" wrapping a single connection — used by Storage::DBI
# acquire() always returns the same $dbh
# No actual pooling — just satisfies the interface
```

### 5. Transaction Semantics for Async

Transactions are inherently connection-scoped. In async mode, a transaction
must pin a specific connection from the pool for all queries within it.

```perl
# Async transaction — the sub receives a transaction-bound storage
my $f = $schema->txn_do_async(sub {
    my ($txn_storage) = @_;  # This storage is pinned to one connection

    # All queries through $txn_storage use the same connection
    $txn_storage->insert_async('artist', { name => 'Coltrane' })->then(sub {
        $txn_storage->insert_async('album', {
            title => 'A Love Supreme',
            artist_id => $_[0]->id,
        });
    });
});

# $f resolves after COMMIT, rejects after ROLLBACK
# The connection is released back to the pool after completion

# Scope guard variant
my $f = $schema->txn_scope_guard_async->then(sub {
    my $guard = shift;
    # ... do async work ...
    $guard->commit_async;
});
```

**Key rules:**

- `txn_do_async` acquires a connection, issues BEGIN, runs the sub, issues
  COMMIT on success or ROLLBACK on Future failure
- The sub receives a storage object bound to that connection — queries through
  any other storage object go to different pool connections
- Nested `txn_do_async` uses savepoints (same as sync nested txn_do)
- On Future rejection (error), rollback is automatic
- The pinned connection returns to the pool after COMMIT or ROLLBACK

**BlockRunner retry for async:**

```perl
# Storage::BlockRunner already handles retry on disconnect.
# Async equivalent:
#   1. Attempt the transaction
#   2. If connection dies mid-transaction, the Future rejects
#   3. BlockRunner catches, reconnects, retries once (same as sync)
#   4. This is handled inside txn_do_async — transparent to caller
```

---

## What Async Distributions Provide

### `DBIO::EV::Pg` (first implementation target)

Implements `DBIO::Storage::Async` using EV::Pg directly — no DBI, no DBD::Pg.
EV::Pg speaks libpq's async protocol natively.

```perl
# Schema setup
my $schema = MyApp::Schema->connect(
    'DBIO::EV::Pg',        # storage_type
    {
        host     => 'localhost',
        dbname   => 'myapp',
        user     => 'myapp',
        pool_size => 10,    # connection pool
    },
);

# Simple async query
$schema->resultset('Artist')->search({ genre => 'Jazz' })->all_async->then(sub {
    my @artists = @_;
    say $_->name for @artists;
});

# Pipeline mode — batch multiple independent queries
my $f = $schema->storage->pipeline(sub {
    my $storage = shift;
    my @futures;
    push @futures, $storage->insert_async('artist', { name => 'Miles' });
    push @futures, $storage->insert_async('artist', { name => 'Coltrane' });
    push @futures, $storage->insert_async('artist', { name => 'Monk' });
    return Future->needs_all(@futures);
});
# All three INSERTs sent in a single network round-trip

# LISTEN/NOTIFY integration
$schema->storage->listen('changelog', sub {
    my ($channel, $payload) = @_;
    # React to ChangeLog notifications in real-time
    my $event = decode_json($payload);
    say "Table $event->{source} had $event->{event}";
});

# COPY support for bulk loading
$schema->storage->copy_in('artist', [qw/name genre/], sub {
    my $put = shift;
    $put->(['Miles Davis', 'Jazz']);
    $put->(['John Coltrane', 'Jazz']);
    # ... thousands more rows at wire speed
});
```

**What `DBIO::EV::Pg` contains:**

| Module | Role |
|--------|------|
| `DBIO::EV::Pg::Storage` | Implements `DBIO::Storage::Async`, wraps EV::Pg |
| `DBIO::EV::Pg::Pool` | Connection pool management with EV watchers |
| `DBIO::EV::Pg::Cursor` | Async cursor for streaming large result sets |
| `DBIO::EV::Pg::Pipeline` | Pipeline mode batch execution |

**Connection info — NOT DBI DSN format:**

```perl
# DBI style (Storage::DBI) — NOT used by EV::Pg
# 'dbi:Pg:dbname=myapp;host=localhost'

# EV::Pg style — libpq connection parameters
{
    host      => 'localhost',
    dbname    => 'myapp',
    user      => 'myapp',
    password  => $pw,
    pool_size => 10,
    # Any libpq conninfo parameter works
}
```

### `Net::Async::DBIO` (IO::Async based)

```perl
use IO::Async::Loop;
use Net::Async::DBIO;

my $loop = IO::Async::Loop->new;
my $schema = MyApp::Schema->connect(
    'Net::Async::DBIO::Storage',
    {
        dsn  => 'dbi:Pg:dbname=myapp',
        loop => $loop,
        pool_size => 5,
    },
);

# Uses IO::Async's Future natively
my $f = $schema->resultset('Artist')->all_async;
my @artists = $f->get;  # blocks in IO::Async style
```

### `Mojo::DBIO` (Mojolicious based)

```perl
use Mojolicious::Lite;
use Mojo::DBIO;

helper schema => sub {
    state $schema = MyApp::Schema->connect(
        'Mojo::DBIO::Storage',
        { dsn => 'postgresql:///myapp', pool_size => 5 },
    );
};

get '/artists' => sub {
    my $c = shift;
    $c->schema->resultset('Artist')->all_async->then(sub {
        $c->render(json => { artists => [map { $_->name } @_] });
    });
};
```

---

## SQLMaker Considerations

`DBIO::SQLMaker` generates SQL strings and bind values. It is already
storage-agnostic — it doesn't execute anything, just builds queries. Both
sync and async storage consume its output identically:

```
SQLMaker->select(...)  →  ($sql, @bind)
                              ↓
            Storage::DBI          Storage::Async
            $sth->execute(@bind)  $pg->exec_params($sql, @bind)
```

No changes to SQLMaker are needed for async support.

---

## Cursor and Streaming

### Sync Cursor (current)

```perl
# DBIO::Cursor wraps a DBI $sth
# $cursor->next calls $sth->fetchrow_arrayref — blocking
while (my @row = $cursor->next) { ... }
```

### Async Cursor

Async cursors support both pull-based (Future per chunk) and push-based
(callback per row/chunk) patterns:

```perl
# Pull-based: get next chunk as a Future
my $cursor = $rs->cursor_async;
$cursor->next_async(100)->then(sub {  # fetch 100 rows
    my @rows = @_;
    if (@rows) {
        # process, then fetch more
        return $cursor->next_async(100);
    }
});

# Push-based: streaming with backpressure
$rs->stream(sub {
    my ($row, $stream) = @_;
    # process each row as it arrives
    # call $stream->pause / $stream->resume for backpressure
}, on_done => sub {
    say "All rows processed";
});
```

The async cursor is implemented by the async storage distribution, not core.
Core only defines the interface expectation.

---

## Error Handling

Async errors propagate through Future rejection, not exceptions:

```perl
# Sync (current)
eval {
    $schema->txn_do(sub { ... });
};
if ($@) { handle_error($@) }

# Async
$schema->txn_do_async(sub { ... })->catch(sub {
    my $error = shift;
    handle_error($error);
});

# Errors are DBIO::Exception objects (same as sync)
# Transaction rollback is automatic on rejection (same as sync)
```

**Important:** An unhandled rejected Future should warn, similar to an
unhandled exception. Async distributions should ensure this (Future.pm does
this by default via `$Future::WARN_ON_DESTROY`).

---

## ChangeLog Integration

The ChangeLog system (see `TODO_CHANGELOG.md`) integrates naturally with async:

- **Write path**: `changelog_write_entry` already has a driver hook point.
  Async storage overrides it to return a Future instead of blocking.
- **LISTEN/NOTIFY**: `changelog_notify` in `DBIO::PostgreSQL::ChangeLog`
  already calls `pg_notify()`. With `DBIO::EV::Pg`, this becomes non-blocking
  and the notification is received via `$storage->listen('changelog', sub {...})`.
- **Real-time changelog streaming**: An async storage can subscribe to changelog
  notifications and push events to application code without polling.

```perl
# Real-time changelog watching with DBIO::EV::Pg
$schema->storage->listen('changelog', sub {
    my ($channel, $payload) = @_;
    my $event = decode_json($payload);
    websocket_broadcast($event);  # Push to connected clients
});
```

---

## File Layout

### Core additions (in `dbio/lib/DBIO/`)

```
lib/DBIO/
├── Future.pm                      # Interface docs only (no implementation)
├── Storage.pm                     # Add *_async virtual methods
└── Storage/
    ├── Async.pm                   # Base class for async storage implementations
    └── Pool.pm                    # Abstract connection pool interface
```

### ResultSet additions (in `dbio/lib/DBIO/`)

```
lib/DBIO/
└── ResultSet.pm                   # Add all_async, first_async, count_async etc.
```

### Separate distribution: `dbio-ev-pg/`

```
dbio-ev-pg/
├── dist.ini                       # [@DBIO]
├── cpanfile                       # depends on EV::Pg, Future, DBIO
└── lib/DBIO/EV/Pg/
    ├── Storage.pm                 # Main async storage implementation
    ├── Pool.pm                    # EV-based connection pool
    ├── Cursor.pm                  # Async cursor with streaming
    └── Pipeline.pm                # Pipeline mode batch execution
```

---

## Implementation Phases

### Phase 1: Refactor Storage Interface (core — no async yet)

1. Add `select_async`, `insert_async`, `update_async`, `delete_async`,
   `select_single_async` as virtual methods on `DBIO::Storage`
   (just `die "Virtual method!"` — same pattern as existing methods)
2. Add `txn_do_async` and `txn_scope_guard_async` to `DBIO::Storage`
   (virtual methods)
3. Create `DBIO::Future` as interface documentation (pod-only module)
4. Create `DBIO::Storage::Async` base class with shared async infrastructure:
   pool integration, transaction pinning logic, Future chaining helpers
5. Create `DBIO::Storage::Pool` interface specification
6. Verify zero impact on sync path — all existing tests must pass unchanged

**Estimated scope:** ~200 lines of new code, mostly virtual method stubs and
documentation. The sync path is completely untouched.

### Phase 2: ResultSet Async API (core)

1. Add `all_async`, `first_async`, `count_async`, `single_async` to
   `DBIO::ResultSet`
2. These delegate to `storage->select_async` etc. and inflate rows in the
   Future chain
3. Add `cursor_async` and streaming interface definitions
4. Tests using mock async storage (returns pre-resolved Futures)

**Estimated scope:** ~300 lines. ResultSet methods are thin wrappers that
delegate to storage and handle row inflation.

### Phase 3: `DBIO::EV::Pg` (separate distribution)

First real async driver. This is where the bulk of async complexity lives.

1. `DBIO::EV::Pg::Storage` — implements all async storage methods using EV::Pg
2. `DBIO::EV::Pg::Pool` — connection pool with EV watchers
3. `DBIO::EV::Pg::Cursor` — async cursor for streaming result sets
4. `DBIO::EV::Pg::Pipeline` — pipeline mode for batch operations
5. LISTEN/NOTIFY support — integrates with ChangeLog notifications
6. Prepared statement caching
7. COPY IN/OUT support for bulk data transfer
8. Integration tests against real PostgreSQL

**Estimated scope:** ~1500-2000 lines. This is the substantial implementation.

### Phase 4: Connection Pool Polish (core + drivers)

1. Implement `DBIO::Storage::Pool::Single` for sync backward compat
2. Pool health checking (connection validation, reconnect)
3. Pool metrics (active/idle/waiting counts)
4. Configurable pool behavior (max wait time, overflow policy)
5. Statement-level load balancing (read queries to replicas)
   — integrates with `DBIO::Replicated`

### Phase 5: Additional Async Drivers

1. `Net::Async::DBIO` — IO::Async integration
2. `Mojo::DBIO` — Mojolicious integration
3. Generic `AnyEvent::DBIO` if demand exists

---

## User-Facing API Summary

```perl
use MyApp::Schema;

# ── Sync (unchanged, current behavior) ──────────────────────

my $schema = MyApp::Schema->connect('dbi:Pg:dbname=myapp', $user, $pw);
my @artists = $schema->resultset('Artist')->all;            # blocking
my $row = $schema->resultset('Artist')->find(1);            # blocking
$schema->txn_do(sub { $row->update({ name => 'New' }) });   # blocking

# ── Async with EV::Pg ───────────────────────────────────────

my $schema = MyApp::Schema->connect(
    'DBIO::EV::Pg',
    { host => 'localhost', dbname => 'myapp', pool_size => 10 },
);

# Async queries return Futures
my $f = $schema->resultset('Artist')
    ->search({ genre => 'Jazz' })
    ->all_async;

$f->then(sub {
    my @artists = @_;
    say $_->name for @artists;
})->catch(sub {
    warn "Query failed: $_[0]";
});

# Async transactions
$schema->txn_do_async(sub {
    my $storage = shift;
    $schema->resultset('Artist')->create_async({ name => 'Miles' })->then(sub {
        my $artist = shift;
        $schema->resultset('Album')->create_async({
            title     => 'Kind of Blue',
            artist_id => $artist->id,
        });
    });
})->then(sub {
    say "Transaction committed";
})->catch(sub {
    say "Transaction rolled back: $_[0]";
});

# Pipeline mode — batch queries in one round-trip
$schema->storage->pipeline(sub {
    my @futures = map {
        $schema->resultset('Artist')->create_async({ name => $_ })
    } @names;
    Future->needs_all(@futures);
})->then(sub {
    my @artists = @_;
    say "Inserted " . scalar(@artists) . " artists in one round-trip";
});

# Mix sync and async — sync methods still work
# (they block the event loop, but are useful for scripts/migrations)
my @all = $schema->resultset('Artist')->all;  # blocks, returns rows

# LISTEN/NOTIFY for real-time events
$schema->storage->listen('new_order', sub {
    my ($channel, $payload) = @_;
    process_order(decode_json($payload));
});
```

---

## Open Questions

- **Row-level async methods**: Should `$row->update_async`,
  `$row->delete_async`, `$row->insert_async` exist? Recommendation: yes, but
  Phase 5. Start with ResultSet-level and storage-level async, add Row-level
  later. Row methods are thin wrappers around storage anyway.

- **`create_async`**: ResultSet's `create` does insert + return inflated row.
  The async version needs to chain insert_async → inflate. Recommendation:
  yes, add `create_async` alongside `all_async` in Phase 2.

- **Prefetch/join behavior**: Complex queries with prefetch generate multiple
  SQL statements or use JOINs. Async prefetch could pipeline the multiple
  queries. Recommendation: defer. Start with single-query async, add prefetch
  optimization later.

- **Which Future?**: Different event loops have different Future
  implementations. Should we standardize on `Future.pm`? Recommendation:
  document the interface, let each distribution pick. `DBIO::EV::Pg` and
  `Net::Async::DBIO` use `Future.pm` naturally. `Mojo::DBIO` wraps
  `Mojo::Promise`. If interop is needed, add adapter later.

- **Sync fallback**: Should `all_async` work on a sync storage by returning an
  already-resolved Future? Recommendation: yes. This makes it easy to write
  code that works with both sync and async storage. The sync storage's
  `select_async` would do the blocking query and return a resolved Future.
  Requires a minimal Future shim in core OR making Future.pm a core dependency.
  Decision deferred to implementation time.

- **Schema-level `connect` API**: How does the user specify async storage?
  Current proposal uses `storage_type` string. Alternative: detect from DSN.
  Recommendation: explicit `storage_type` is clearer and doesn't require
  parsing magic.

- **Testing async code**: Async tests need an event loop running. Each async
  distribution handles its own test setup. Core async tests (Phase 2) use
  mock storage with pre-resolved Futures — no event loop needed.

---

## Complexity Assessment

Phase 1 (storage interface) and Phase 2 (ResultSet async API) are low-risk
additions to core — they add new methods without changing existing ones.
Estimated ~500 lines total, all additive.

Phase 3 (`DBIO::EV::Pg`) is the real work — a complete storage implementation
using a non-DBI client library. This is ~2000 lines but lives in its own
distribution with no risk to core stability.

**Verdict: Phase 1 can happen alongside other core work. Phase 3 is the
critical path — it proves the design works with a real async client.**
