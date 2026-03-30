# DBIO::Replicated Classic Refactor Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Moose-based replicated storage with normal classes in `dbio-replicated` and rename the public API to `DBIO::Replicated::*` without touching `dbio` yet.

**Architecture:** `DBIO::Replicated::Storage` stays the orchestration point, while backend wrapper classes hold concrete driver storages and replicated metadata. Pool and balancer logic move to normal inheritance-based classes under the new driver namespace.

**Tech Stack:** Perl 5.20+, DBIO, DBI, Test::More, Test::Exception, Test::Warn

---

## Chunk 1: Namespace and test pivot

### Task 1: Rename test targets to the new namespace

**Files:**
- Modify: `dbio-replicated/t/10-replicated.t`

- [ ] **Step 1: Write the failing test**

Update the test file so it loads `DBIO::Replicated` and `DBIO::Replicated::Storage` names instead of the legacy namespace, while preserving existing replicated behavior assertions.

- [ ] **Step 2: Run test to verify it fails**

Run: `prove -l dbio-replicated/t/10-replicated.t`
Expected: FAIL because the new namespaces do not exist yet.

### Task 2: Add the new top-level component and storage shell

**Files:**
- Create: `dbio-replicated/lib/DBIO/Replicated.pm`
- Create: `dbio-replicated/lib/DBIO/Replicated/Storage.pm`

- [ ] **Step 1: Write minimal implementation**

Add the component and storage class with the new package names and enough structure for the renamed test to compile.

- [ ] **Step 2: Run test to verify progress**

Run: `prove -l dbio-replicated/t/10-replicated.t`
Expected: FAIL later in execution, not at module load time.

## Chunk 2: Replace Moose object model

### Task 3: Port pool and balancers to normal classes

**Files:**
- Create: `dbio-replicated/lib/DBIO/Replicated/Pool.pm`
- Create: `dbio-replicated/lib/DBIO/Replicated/Balancer.pm`
- Create: `dbio-replicated/lib/DBIO/Replicated/Balancer/First.pm`
- Create: `dbio-replicated/lib/DBIO/Replicated/Balancer/Random.pm`

- [ ] **Step 1: Write failing assertions**

Extend the existing replicated test with namespace and behavior checks for the new pool and balancer classes.

- [ ] **Step 2: Implement minimal class/accessor behavior**

Port the current logic without Moose traits or roles.

- [ ] **Step 3: Run targeted test**

Run: `prove -l dbio-replicated/t/10-replicated.t`
Expected: balancing and pool assertions pass or fail only on backend integration.

## Chunk 3: Wrapper-based backends

### Task 4: Add backend wrapper classes and integrate them

**Files:**
- Create: `dbio-replicated/lib/DBIO/Replicated/Backend.pm`
- Create: `dbio-replicated/lib/DBIO/Replicated/Backend/Master.pm`
- Create: `dbio-replicated/lib/DBIO/Replicated/Backend/Replicant.pm`
- Modify: `dbio-replicated/lib/DBIO/Replicated/Storage.pm`
- Modify: `dbio-replicated/lib/DBIO/Replicated/Pool.pm`

- [ ] **Step 1: Write failing test updates**

Adjust the test expectations so master/replicant-specific behavior is asserted through wrapper instances.

- [ ] **Step 2: Implement wrappers**

Forward DBI storage behavior through wrapped concrete storages and keep replicated metadata on the wrapper classes.

- [ ] **Step 3: Run targeted test**

Run: `prove -l dbio-replicated/t/10-replicated.t`
Expected: read/write routing, balancing, and validation pass.

## Chunk 4: Distribution cleanup

### Task 5: Remove Moose dependencies and update docs

**Files:**
- Modify: `dbio-replicated/cpanfile`
- Modify: `dbio-replicated/README.md`
- Modify: `dbio-replicated/lib/DBIO/Storage/DBI/Replicated/Introduction.pod` or replacement docs

- [ ] **Step 1: Update metadata and docs**

Drop Moose-related prereqs and rename examples to the new namespaces.

- [ ] **Step 2: Run distribution tests**

Run: `prove -l dbio-replicated/t/10-replicated.t`
Expected: PASS

- [ ] **Step 3: Run a syntax sweep**

Run: `perl -I dbio-replicated/lib -c dbio-replicated/lib/DBIO/Replicated.pm`
Expected: syntax OK
