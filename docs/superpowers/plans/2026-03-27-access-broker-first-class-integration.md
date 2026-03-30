# AccessBroker First-Class Integration Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `DBIO::AccessBroker` a first-class `Schema->connect($broker)` input across DBIO core, the three DBI drivers, and `dbio-postgresql-async`, without a coderef workaround.

**Architecture:** Move broker attachment and mode handling into shared storage-level plumbing, upgrade the broker contract to be storage-aware, then teach DBI and async storages to fetch fresh storage-native connection info from the broker when they establish new connections. DBI drivers should inherit the new behavior through `DBIO::Storage::DBI`, while async PostgreSQL must add a broker-aware pool/provider path so new EV::Pg connections do not freeze stale credentials.

**Tech Stack:** Perl 5, DBIO core, DBIO::Storage::DBI, DBIO::Storage::Async, EV::Pg, Future, Test::More, Test::Exception

---

## Chunk 1: Core broker contract

### Task 1: Add a failing core test for first-class broker connect

**Files:**
- Create: `dbio/t/access_broker/04-schema-connect.t`
- Modify: `dbio/t/access_broker/01-static.t`

- [ ] **Step 1: Write the failing test**

Add a focused core test that:
- creates a minimal fake broker subclass
- calls `DBIO::Test::Schema->connect($broker)`
- asserts the returned schema storage keeps the broker attached
- asserts the default broker mode is `write`
- avoids real SQLite deployment and avoids a user coderef

- [ ] **Step 2: Run test to verify it fails**

Run: `prove -l dbio/t/access_broker/04-schema-connect.t`
Expected: FAIL because `Schema->connect($broker)` is not recognized yet.

### Task 2: Make the broker contract storage-aware

**Files:**
- Modify: `dbio/lib/DBIO/AccessBroker.pm`
- Modify: `dbio/lib/DBIO/AccessBroker/Static.pm`
- Modify: `dbio/lib/DBIO/AccessBroker/ReadWrite.pm`
- Modify: `dbio/lib/DBIO/AccessBroker/Vault.pm`

- [ ] **Step 1: Add the new broker-facing API**

Implement a storage-aware broker method, with the base class defining the first-class contract and legacy DBI-shaped behavior only as an internal compatibility bridge if still needed.

- [ ] **Step 2: Update built-in broker subclasses**

Make Static/ReadWrite/Vault return storage-native connection information through the new broker contract for DBI storage and keep refresh semantics intact.

- [ ] **Step 3: Run the failing test again**

Run: `prove -l dbio/t/access_broker/04-schema-connect.t`
Expected: still FAIL, but later in storage handling rather than at argument parsing.

### Task 3: Add shared broker plumbing to storage base

**Files:**
- Modify: `dbio/lib/DBIO/Storage.pm`

- [ ] **Step 1: Write the failing expectation**

Extend `dbio/t/access_broker/04-schema-connect.t` so it checks for shared storage-level broker accessors/helpers rather than ad-hoc DBI-only state.

- [ ] **Step 2: Implement minimal shared storage helpers**

Add storage-level broker attachment, broker mode, and a helper to obtain current broker-derived connection information for a mode.

- [ ] **Step 3: Run the targeted test**

Run: `prove -l dbio/t/access_broker/04-schema-connect.t`
Expected: PASS for broker attachment semantics, while DBI reconnect tests still do not exist or fail.

## Chunk 2: DBI integration and driver coverage

### Task 4: Teach Schema and DBI storage to accept a broker directly

**Files:**
- Modify: `dbio/lib/DBIO/Schema.pm`
- Modify: `dbio/lib/DBIO/Storage/DBI.pm`

- [ ] **Step 1: Add a failing reconnect/refresh test**

Extend `dbio/t/access_broker/04-schema-connect.t` or add a second focused test so a broker with changing credentials is consulted again when DBI storage builds a new handle.

- [ ] **Step 2: Run test to verify it fails**

Run: `prove -l dbio/t/access_broker/*.t`
Expected: FAIL because DBI storage still snapshots classic connect args instead of re-querying the broker on handle creation/reconnect.

- [ ] **Step 3: Implement minimal DBI storage changes**

Make `Schema->connection` recognize a single `DBIO::AccessBroker` argument and make `DBIO::Storage::DBI` retain the broker, use broker-derived connect info during handle creation, and reuse the broker on reconnect.

- [ ] **Step 4: Run core broker tests**

Run: `prove -l dbio/t/access_broker/*.t`
Expected: PASS for broker connect semantics without relying on a coderef workaround.

### Task 5: Verify the three DBI drivers inherit the behavior

**Files:**
- Modify: `dbio-postgresql/lib/DBIO/PostgreSQL.pm`
- Modify: `dbio-mysql/lib/DBIO/MySQL.pm`
- Modify: `dbio-mysql/lib/DBIO/MySQL/MariaDB.pm`
- Modify: `dbio-sqlite/lib/DBIO/SQLite.pm`
- Create: `dbio-postgresql/t/01-access-broker-api.t`
- Create: `dbio-mysql/t/01-access-broker-api.t`
- Create: `dbio-sqlite/t/01-access-broker-api.t`

- [ ] **Step 1: Add failing driver smoke tests**

For each DBI driver distribution, add a compile/API-level test that loads the driver component, calls `->connection($broker)` or `->connect($broker)` on a tiny schema, and verifies the schema ends up with the expected storage class and attached broker.

- [ ] **Step 2: Run the three new smoke tests**

Run: `prove -l dbio-postgresql/t/01-access-broker-api.t dbio-mysql/t/01-access-broker-api.t dbio-sqlite/t/01-access-broker-api.t`
Expected: PASS if the inherited DBI path is sufficient; if any driver-specific override still breaks broker input, FAIL with the driver-specific gap.

- [ ] **Step 3: Apply only necessary driver fixes**

If the driver components already delegate cleanly via `next::method`, keep changes minimal. If they need broker-specific normalization, patch only the affected component.

- [ ] **Step 4: Re-run the DBI driver smoke tests**

Run: `prove -l dbio-postgresql/t/01-access-broker-api.t dbio-mysql/t/01-access-broker-api.t dbio-sqlite/t/01-access-broker-api.t`
Expected: PASS

## Chunk 3: Async PostgreSQL integration

### Task 6: Add failing async storage/pool tests for broker-backed connect

**Files:**
- Create: `dbio-postgresql-async/t/02-access-broker.t`
- Modify: `dbio-postgresql-async/t/01-storage-api.t`

- [ ] **Step 1: Write the failing async tests**

Add tests that:
- connect an async schema with a broker
- assert the async storage retains the broker
- assert pool connection creation obtains conninfo through a broker-aware path
- assert a second connection request can see changed broker output instead of frozen initial conninfo

- [ ] **Step 2: Run the async tests to verify they fail**

Run: `prove -l dbio-postgresql-async/t/01-storage-api.t dbio-postgresql-async/t/02-access-broker.t`
Expected: FAIL because async storage and pool only understand static conninfo today.

### Task 7: Implement broker-aware async connection providers

**Files:**
- Modify: `dbio-postgresql-async/lib/DBIO/PostgreSQL/Async.pm`
- Modify: `dbio-postgresql-async/lib/DBIO/PostgreSQL/Async/Storage.pm`
- Modify: `dbio-postgresql-async/lib/DBIO/PostgreSQL/Async/Pool.pm`

- [ ] **Step 1: Add minimal async storage broker plumbing**

Teach the async schema component and storage to accept a broker directly and to expose a broker-backed conninfo provider instead of a one-time static conninfo capture.

- [ ] **Step 2: Update the pool**

Make pool connection creation call the provider each time a new EV::Pg connection is created, rather than freezing a string at pool construction time.

- [ ] **Step 3: Cover dedicated connections**

Update any dedicated connection creation path, especially LISTEN/NOTIFY setup, to use the same broker-aware conninfo retrieval.

- [ ] **Step 4: Run async tests**

Run: `prove -l dbio-postgresql-async/t/01-storage-api.t dbio-postgresql-async/t/02-access-broker.t`
Expected: PASS

## Chunk 4: Cleanup and verification

### Task 8: Remove the coderef-only integration test path from core and update docs

**Files:**
- Modify: `dbio/t/access_broker/04-integration.t`
- Modify: `dbio/lib/DBIO/AccessBroker.pm`
- Modify: `dbio/ERROR.md`

- [ ] **Step 1: Align or replace the old integration test**

Remove the misleading real-SQLite coderef test from core or rewrite it so it validates the new first-class broker path instead of the old workaround.

- [ ] **Step 2: Update docs**

Document `Schema->connect($broker)` as the primary path, explain the storage-aware broker contract, and note that statement routing is explicitly out of scope here.

- [ ] **Step 3: Run focused verification**

Run: `prove -l dbio/t/access_broker/*.t`
Expected: PASS

Run: `prove -l dbio-postgresql/t/01-access-broker-api.t dbio-mysql/t/01-access-broker-api.t dbio-sqlite/t/01-access-broker-api.t`
Expected: PASS

Run: `prove -l dbio-postgresql-async/t/01-storage-api.t dbio-postgresql-async/t/02-access-broker.t`
Expected: PASS

- [ ] **Step 4: Run a syntax sweep on touched modules**

Run: `perl -c dbio/lib/DBIO/AccessBroker.pm`
Expected: syntax OK

Run: `perl -c dbio/lib/DBIO/Storage.pm`
Expected: syntax OK

Run: `perl -c dbio/lib/DBIO/Storage/DBI.pm`
Expected: syntax OK

Run: `perl -c dbio-postgresql-async/lib/DBIO/PostgreSQL/Async/Storage.pm`
Expected: syntax OK
