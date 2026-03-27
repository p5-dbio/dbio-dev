# AccessBroker First-Class Integration Design

## Goal

Make `DBIO::AccessBroker` a first-class connection input in DBIO core via
`My::Schema->connect($broker)`, without relying on a user-supplied coderef
workaround, and keep the broker contract valid for both DBI-based storages and
async storages such as `DBIO::PostgreSQL::Async`.

## Scope

This design covers:

- `dbio` core changes needed to accept an `AccessBroker` object directly
- the broker contract itself, so it is storage-aware rather than DBI-only
- DBI storage integration for connect and reconnect
- async-storage integration requirements, specifically for
  `DBIO::PostgreSQL::Async`
- test coverage in core for the new connect path

This design does not include:

- automatic SQL statement routing between read and write backends
- a mandatory built-in default broker for plain DSN usage

## Problem

The current `DBIO::AccessBroker` API claims to be storage-agnostic but exposes
`connect_info_for($mode)` in a DBI-shaped return format. That means the broker
model only fits `DBIO::Storage::DBI`, while async storage implementations such
as `DBIO::PostgreSQL::Async::Storage` use different connection information and
lifecycles.

The existing usage pattern also relies on a coderef passed to `connect()`,
which proves only that DBIO can accept a custom handle factory. It does not
make `AccessBroker` a first-class DBIO feature and does not give storage
implementations a shared broker contract.

## Design

### Public API

- Support `My::Schema->connect($broker)` where `$broker->isa('DBIO::AccessBroker')`
- Keep existing DSN, hashref, and coderef connect forms working unchanged
- Use `write` as the default initial broker mode

### Broker contract

`DBIO::AccessBroker` becomes storage-aware.

Instead of treating DBI-style connect args as the universal return shape, the
broker will expose a storage-facing method that accepts the concrete storage
instance and a mode:

- broker receives: `$storage`, `$mode`
- broker returns: storage-native connection information

The exact method name can be finalized in implementation, but the contract is:

- DBI storage receives DBI-style connect arguments
- async PostgreSQL storage receives EV::Pg/libpq-style connection info
- broker subclasses may branch on storage type
- the base broker provides the shared refresh lifecycle and default mode logic

The old DBI-specific method can remain as a compatibility shim inside the new
broker model if useful during implementation, but the first-class contract is
the storage-aware one.

### Core storage integration

`DBIO::Storage` will own the shared broker plumbing:

- accessor for attached `access_broker`
- accessor for current `access_broker_mode`
- helper to attach a broker and call `set_storage`
- helper to obtain current broker-derived connection information for a mode

This keeps the broker lifecycle in one place and avoids re-implementing the
same attachment rules in each storage class.

### DBI storage behavior

`DBIO::Storage::DBI` will:

- recognize broker-backed connection state during `connect_info`
- retain the broker on the storage instead of converting it into a one-shot
  coderef
- ask the broker for fresh DBI connect arguments when a handle is first built
- ask the broker again on reconnect, so rotated credentials are picked up

This removes the coderef workaround while preserving the normal DBI storage
responsibility for creating and validating handles.

### Async PostgreSQL behavior

`DBIO::PostgreSQL::Async` and `DBIO::PostgreSQL::Async::Storage` must support
the same broker-backed connect entry point.

Important constraint: the async storage currently freezes `conninfo` into the
pool when the pool is constructed. That defeats credential rotation, because
new async connections would continue using stale credentials.

Therefore the async design must use a broker-aware connection provider:

- `connect($broker)` attaches the broker to async storage directly
- storage does not reduce the broker to a one-time static conninfo snapshot
- the pool obtains conninfo through a callback/provider that can ask the broker
  again when creating a new EV::Pg connection
- dedicated non-pooled connections, such as LISTEN/NOTIFY connections, must
  also acquire fresh connection info through the same broker-aware path

Existing open async connections do not need forced mid-flight credential
replacement in this phase. The requirement is that newly created connections
use the broker’s current data.

### Boundaries

This phase provides a correct first-class connection source for brokers.

It does not attempt:

- automatic read/write statement routing inside a single schema instance
- replica balancing in core query execution

Those concerns belong to storage- or driver-level routing and should be solved
separately.

## Testing

### Core tests

Add focused tests in `dbio` for:

- `Schema->connect($broker)` acceptance
- broker attachment to storage
- default broker mode being `write`
- DBI reconnect fetching fresh broker-derived connect info
- backward compatibility for DSN and coderef connects

These tests should use fake or controlled storage/test infrastructure, not a
real SQLite integration test in core.

### Async tests

Add targeted tests in `dbio-postgresql-async` for:

- broker-backed connect setup
- pool connection creation via broker-aware conninfo provider
- fresh conninfo being requested again when a new async connection is created

The async tests can stay at the storage/pool contract level and do not need to
introduce full live-database coverage for the broker feature in this phase.

## Optional follow-up

A trivial built-in broker that wraps the standard single-connection case could
be added later if it meaningfully simplifies internal code or documentation.
It is not required for the first-class broker integration itself.

## Risks

- The broker API change is architectural, not cosmetic, and affects both core
  and async driver expectations.
- Async pooling needs a clean provider hook; a partial implementation that
  snapshots conninfo would silently undermine credential rotation.
- Some existing tests and drivers still assume older connection-info shapes and
  may need follow-up cleanup once the new broker contract lands.
