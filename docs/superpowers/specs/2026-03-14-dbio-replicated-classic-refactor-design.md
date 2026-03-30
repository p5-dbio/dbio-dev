# DBIO::Replicated Classic Refactor Design

## Goal

Refactor `dbio-replicated` away from Moose/MooseX::Types and rename the public API to the driver-style namespace used by the other DBIO driver distributions.

## Scope

This phase is limited to `dbio-replicated`. Core `dbio` changes are explicitly deferred until the user confirms that work can start there.

## Design

### Public structure

- Add `DBIO::Replicated` as the schema component.
- Add `DBIO::Replicated::Storage` as the public storage class.
- Move pool and balancer code under `DBIO::Replicated::*`.
- Remove the legacy `DBIO::Storage::DBI::Replicated*` namespace instead of wrapping it.

### Class model

- Replace Moose roles with normal classes.
- `DBIO::Replicated::Storage` remains the coordinator and delegates read/write paths.
- `DBIO::Replicated::Pool` manages replicant registration, connection, validation, and status.
- `DBIO::Replicated::Balancer` becomes a normal abstract base class with shared balancing logic.
- `DBIO::Replicated::Balancer::First` and `DBIO::Replicated::Balancer::Random` inherit from that base.
- Add backend wrapper classes that hold a concrete driver storage plus replicated metadata instead of mutating driver classes at runtime with roles.

### Backend wrappers

- `DBIO::Replicated::Backend` wraps a concrete `DBIO::Storage::DBI` instance.
- `DBIO::Replicated::Backend::Master` identifies the master side.
- `DBIO::Replicated::Backend::Replicant` adds replicant state such as `active`, `dsn`, `id`, and `master`.
- Wrapper classes provide DSN-aware trace output and any replicated-only helper methods while forwarding DBI storage behavior to the wrapped storage object.

### Constraints

- Do not modify `dbio` yet.
- Keep the current replicated behavior working from `dbio-replicated` alone as far as possible.
- Update tests to target the new namespace and the wrapper-based architecture.

## Risks

- Some behavior currently depends on runtime class mutation after `_determine_driver`.
- Without the deferred core work, there may be edge cases where a cleaner storage factory hook would simplify the implementation.
- Test coverage must confirm that read/write routing, balancing, debug output, and replicant validation still work.
