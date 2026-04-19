# Subsystem Anatomy

**Status.** v2 (2026-04-19). Aligned with the step model (STEP-4, five-phase discipline). `*_prepare` + `*_commit` function pairs retired per STEP-6; sub-phase structure preserved.

**Purpose**: Define the uniform internal structure of a txKernel subsystem. Each subsystem decomposes into four modules with distinct roles, and every mutating *step* runs the five-phase commit discipline (observe → upgrade → reserve → commit → publish) within its function body. Syscalls may be single-step (atomic) or compositional (a sequence of steps across driver-mediated retries).

**Audience**: Designers adding new subsystems or operations, reviewers auditing cross-module dependencies, agents navigating the codebase.

**Companion documents**:
- [`object_model_0417.md`](./object_model_0417.md) — semantic object model (entities, projections, bindings, obligations, retention). Authoritative.
- [`INVARIANTS.md`](./INVARIANTS.md) — STEP-4 codifies the five-phase discipline stated here as invariants.
- [`STEP_MODEL.md`](./STEP_MODEL.md) §3 — the same discipline as seen from the execution-primitive view.
- [`CONCEPTS.md`](./CONCEPTS.md) §2.2 — sub-phase vocabulary (reserve, commit, commit point, visibility boundary).
- [`LIVENESS.md`](./LIVENESS.md) — application checklist and subsystem projection catalog.

This document is architecture-layer for §1–§4 and §6–§9 (what subsystems must be). §5 and §10 are illustrative. §4.5 identifies a substrate requirement whose precise realization is implementation-layer.

---

## 1. The Four-Module Layout

Every full subsystem has the following structure:

```
subsystem/
    structure/     SSoT for semantic indexes
                   data types, atomic flags, read accessors
                   holds published retention evidence (Cap<T> or operational)
                   no allocation logic, no resource reservation
                   pub(crate) fields; external access via checks/

    checks/        observation + witness production (step phase 1)
                   reads structure/ under epoch guard
                   produces IdentRef-carrying, lifetime-parameterized witnesses
                   may trampoline on informational I/O (emits IORequest; no mutation)
                   pub API: require_* functions, witness types

    execution/     mutating step functions (step phases 2–5)
                   consumes witnesses (by value)
                   upgrade:  IdentRef → Cap / operational evidence
                   reserve:  linear substrate tokens
                   commit:   visibility-boundary writes via commit primitives
                   publish:  bus primitive fires for attached signals
                   pub API: step_* functions

    project.rs    read-only projections for procfs/sysfs
                   reads structure/ directly
                   generates synthetic RNodes on demand (for pseudo-fs adapters)
                   pub API: projection definitions consumed by procfs adapter
```

Service subsystems (cred, rlim, trace, sched, time, dev) follow a reduced form without full entity resolution, but the module responsibilities are the same.

---

## 2. Module Responsibilities

### 2.1 structure/

The semantic ground truth. Contains DLLs, BTrees, Radixes, hash tables that *mean* something in the subsystem's domain:

- VFS: DEntry tree, RNode registry, mount tree
- PID: PidTable, process hierarchy DLL, process groups
- FD: per-process fd tables
- Net: socket hash, protocol tables

Structure modules hold **published** retention evidence (`Cap<T>` or operational evidence) — entities whose existence is semantically committed. Unpublished state (reservations, in-progress constructions) does not live in structure. That belongs in substrate.

Structure is:
- **Read by** `checks/`, `project.rs`, and (within the same subsystem) `execution/`
- **Written by** `execution/` only, at the commit sub-phase of a step (phase 4), through commit primitives
- **Never allocated from directly by external code** — external callers get published evidence through `checks/`'s require functions

### 2.2 checks/

The observation gate. Every external interaction with the subsystem enters through `checks/`.

**Purity invariant**: `checks/` never mutates observable state. It may yield (trampoline for cache fills), but any I/O it requests is fulfilled externally and fed back in — `checks/` itself remains a pure function over structure state.

Submodule layout:

```
checks/
    predicates.rs     pure predicate functions
    require.rs        require_* functions (witness producers)
    witness.rs        witness type definitions (lifetime-parameterized, !Send, !Sync)
    trampoline.rs     (optional) CheckStep state machines
    mod.rs            pub re-exports: require_* and witness types
```

Each projection has one predicate, defined exactly once. Each operation entry has one require function, which invokes predicates and returns either a witness or an errno. See [`object_model_0417.md`](./object_model_0417.md) §5, §8.6 for the projection framework.

**Witness discipline.** Witnesses produced by checks carry observation-level evidence (`IdentRef<'g, T>`), not retention (`Cap`). They are lifetime-parameterized by the epoch guard under which they were produced. Witnesses are `!Send`, `!Sync`; they cannot be stored in struct fields, static locals, or collections, and cannot cross step boundaries (WIT-3). Retention is acquired in the step's upgrade sub-phase (phase 2), after witness consumption.

Checks may:
- Observe structure/ via `IdentRef` under epoch guards (`pub(crate)` access)
- Construct witnesses (private constructors)
- Emit `IORequest` values when validation needs informational I/O fills
- Consume other subsystems' witnesses (for cross-subsystem predicates)

Checks may not:
- Import from substrate/ (no reservation, no zone allocation, no credit accounting)
- Import from execution/ (any subsystem)
- Publish bindings (`set_*`, `emit_*`, `fire_*`, substrate commit operations)
- Write to structure/ fields
- Acquire retention beyond `IdentRef` observation

### 2.3 execution/

The mutation layer. Every operation that changes kernel state lives here as a `step_*` function. Each step function's body runs the five-phase commit discipline (STEP-4): observe, upgrade, reserve, commit, publish.

Submodule layout:

```
execution/
    step_open.rs           pipe::step_open, vfs::step_open, ...
    step_unlink.rs         vfs::step_unlink
    step_fork.rs           process::step_fork
    step_read.rs           pipe::step_read (multi-step)
    resume.rs              subsystem-private resume types for multi-step ops
    ...
```

Execution's **upgrade** sub-phase (STEP-4 phase 2) performs the SENTINEL_DEAD-guarded transition from `IdentRef<'g, T>` (witness-level observation) to `Cap<T>` or `T::OperationalEvidence` (retention). Upgrades are fallible; partial upgrades are not allowed (OBL-4). The **reserve** sub-phase (phase 3) acquires substrate slots as linear reservations; the **commit** sub-phase (phase 4) consumes reservations at visibility boundaries; the **publish** sub-phase (phase 5) fires attached signals.

Execution may:
- Read structure/ (to plan mutations)
- Upgrade witness observations to retention (phase 2)
- Call reserve primitives (phase 3): `zone::reserve`, `index::reserve_slot`, `credit::reserve`
- Call commit primitives (phase 4): `zone::sign`, `index::commit`, `index::{withdraw,swap}_commit`, `credit::commit`
- Fire bus events at publish sub-phase (phase 5)
- Consume witnesses (taking them by value)

Execution may not:
- Call without a witness (the step's observe sub-phase produces or consumes one)
- Bypass commit primitives for observer-visible field writes
- Publish before a commit point's visibility boundary
- Hold a reservation across step return

### 2.4 project.rs

The read-only projection layer for procfs/sysfs. Projections describe what the subsystem currently looks like from an observer's point of view.

```rust
#[projections]
impl ProcessIdentity {
    fn status(&self) -> Vec<u8> {
        // Reads identity fields always: pid, parent_pid, state-derived-from-payload-presence,
        // exit_status if payload is None.
        // Reads payload fields conditionally: uid/gid/rlimits/vm stats only if payload.is_some().
        //
        // For a zombie (payload is None):
        //   Name:           <taken from identity if cached there, else "?">
        //   State:          Z (zombie)
        //   Pid:            <pid>
        //   PPid:           <parent_pid>
        //   Threads:        0  (ThreadPayloads all reclaimed)
        //   ExitCode:       <exit_status>
        //   (no VmSize, VmRSS, Uid, Gid, SigPnd — those live in payload and payload is gone)
    }
    
    fn maps(&self) -> Vec<u8> {
        // Requires payload.frame.vm. For zombies: returns "" (empty) — vm is gone.
    }
    
    fn fd_dir(&self) -> DirProjection {
        // Requires payload.frame.fd_table. For zombies: empty directory.
    }
}
```

The projection reads `self.payload` atomically (it's a field of ProcessIdentity); if Some, the PayloadCap can be dereferenced to read ProcessPayload fields. If None, projection falls back to identity-only output. Zombies remain observable in procfs with a 'Z' state and exit code, but fields that required payload (VmSize, fd list, credentials) are absent. This is what POSIX `/proc/<pid>/status` does on Linux, realized here by the structural identity/payload split rather than by Option-guarded reads on a god-struct.

For the factoring convention determining which entities split and which stay co-located, see [`object_model_0417.md`](./object_model_0417.md) §8.1.1.

Projections are:
- **Read-only** — no mutation, no reservation, no side effects
- **Viewpoint-free** — they report current state, not validity for an operation
- **Generated on demand** — procfs synthesizes ephemeral RNodes when user code opens a projection file

A zombie process still appears in `/proc/<pid>/status` even though it is not signal-addressable. Projections describe; checks authorize. They read the same structure for different purposes.

---

## 3. The Five-Phase Commit Discipline (per Step)

Every mutating step's body runs this structure:

```
Phase 1: observe            Result<Witness<'g>, Errno>
                            ─────────────────────────
                            (under fresh epoch guard)
                            require_*(...) in checks/
                            pure: policy, reachability, permissions
                            may trampoline for informational I/O
                            produces IdentRef-carrying witness
                            no resource reservation, no mutation

Phase 2: upgrade            Result<Evidence, Errno>
                            ───────────────────────
                            IdentRef → Cap / operational evidence
                            (SENTINEL_DEAD CAS; fallible)
                            per OBL-4
                            no reservations yet; nothing private held

Phase 3: reserve            Result<Reservations, Errno>
                            ───────────────────────────
                            linear substrate tokens:
                              zone::reserve
                              index::reserve_slot
                              credit::reserve
                            values constructed into reserved slots
                            nothing observer-visible yet
                            Drop on early return = clean rollback

Phase 4: commit             Observable writes at visibility boundaries
                            ──────────────────────────────────────────
                            consumes reservations via commit primitives:
                              zone::sign               → Cap<T>
                              index::commit
                              index::withdraw_commit
                              index::swap_commit
                              credit::commit
                            each call is a commit point;
                            each commit point is a visibility boundary;
                            STEP-8 attaches from here

Phase 5: publish            Signal fires on attached wires
                            ──────────────────────────────
                            for each commit point with attachments:
                              wire.fire(event)
                            per SIG-4 (after boundary),
                            per SIG-5 (per-carrier ordering)
```

### 3.1 Failure Semantics

- **Phase 1 failure (observe)**: no side effects. Witness never produced. Error propagates through `?`.
- **Phase 2 failure (upgrade)**: `SENTINEL_DEAD` detected on CAS. No reservations held yet; nothing to roll back. Error propagates through `?`.
- **Phase 3 failure (reserve)**: reservations partially acquired are dropped through Rust's Drop. No commit points have executed; no observer can see anything. Error propagates through `?`.
- **Phase 4 cannot fail by construction.** Commit primitives consume a valid reservation token; their input is already pre-conditioned on phase 3's success. They return the published evidence (e.g., `Cap<T>`), not `Result<Cap<T>, _>`.
- **Phase 5 cannot fail by construction.** Fire on a bus carrier is infallible (queues may coalesce or drop; the fire call itself always succeeds).

The no-rollback property of committed writes depends on phase 4's infallibility: once a commit primitive has returned, the write is visible, and STEP-8 locks it in.

### 3.2 Linearity of Reservations

Reservation types are **linear** — used exactly once, either committed or dropped:

- Not `Copy`, not `Clone`
- Consumption methods (sign, commit, mutation-commit) take `self` by value
- Drop is the rollback path

Linearity is enforced by Rust's move semantics.

Reservation bundles (structs grouping multiple reservations for a single step) are also linear. Moving the bundle moves all fields atomically. Dropping the bundle runs each field's Drop independently. Destructuring inside the commit sub-phase consumes the fields one by one.

```rust
pub struct PipeReservations {
    pipe_slot: ZoneReservation<PipeData>,
    read_slot: ZoneReservation<PipeReadEnd>,
    write_slot: ZoneReservation<PipeWriteEnd>,
    fd_read:   IndexReservation<RawFd, FdEntry>,
    fd_write:  IndexReservation<RawFd, FdEntry>,
    charge:    CreditReservation,
    pipe_data: PipeData,
    read_end:  PipeReadEnd,
    write_end: PipeWriteEnd,
}
// No Clone. No Copy. Linear by construction.
```

**Do not `mem::forget` a reservation.** That leaks the resource. The only sanctioned use of `mem::forget` is inside substrate's own sign/commit implementations.

### 3.3 Witness Lifetime Across Phases

The witness produced by phase 1 holds `IdentRef<'g, T>` for its target entities. `'g` is the lifetime of the epoch guard acquired at phase 1's start.

Phase 2 performs upgrade: each `IdentRef<'g, T>` in the witness is promoted to `Cap<T>` or `T::OperationalEvidence` via the SENTINEL_DEAD-guarded CAS. After upgrade, the evidence is `'static`. The guard may still be held (it has not been explicitly released), but no observation made under it is semantically required after this point; the step now carries retention evidence instead of observation evidence.

Unlike the old `*_prepare` function (which could `.await` after upgrade because its results were `'static`), a step cannot `.await` at any sub-phase — STEP-2 forbids it. Reserve and commit in phases 3 and 4 are synchronous. If a step needs to "wait" for substrate (cache fill, informational I/O), it returns `Blocked` and leaves reserve/commit for the next invocation after a driver-managed wait.

```rust
// Conceptual shape:
fn op_step<'ctx>(
    ctx: &'ctx ThreadContext,
    ...,
) -> StepOutcome<OpResult> {
    // Phase 1: observe
    let guard = epoch::guard();
    let witness = require_op(..., &guard)?;

    // Phase 2: upgrade
    let target_cap = witness.target.upgrade_to_cap()?;
    let other_cap  = witness.other.upgrade_to_cap()?;

    // Phase 3: reserve
    let zone_slot = substrate::zone::reserve(&zones.foo)?;
    let idx_slot  = substrate::index::reserve_slot(&ctx.table)?;

    // Phase 4: commit  (each call is a commit point / visibility boundary)
    let foo_cap = substrate::zone::sign(zone_slot, FooData { ... });
    substrate::index::commit(&ctx.table, idx_slot, foo_cap.clone());

    // Phase 5: publish
    foo_cap.readiness.fire(ReadinessMask::READY);

    StepOutcome::Done(OpResult { ... })
}
```

Note the `?` operator on reservations: any failure in phase 3 drops all earlier phase-3 reservations through Rust's unwinding. Once phase 4 begins, commit primitives are infallible, so there is no in-phase-4 rollback path to reason about.

### 3.4 What Becomes Observable at Commit

The linearization points are the commit primitive calls. Before any commit primitive call, no structure change is observable. After each commit primitive call, that specific binding change is observable. These linearization points are the commit points of STEP-4 phase 4; each is a visibility boundary, and the boundary is the precise moment STEP-8 attaches.

For steps with multiple commit points, the commit sequence is a series of observer-safe transitions. Each transition lands in a valid state. Concurrent observers may witness intermediate states *between* transitions, but each transition itself is atomic (per §4.5).

### 3.5 Failure Has No Rollback

If phases 1–3 fail (observe / upgrade / reserve), reservations drop via Drop — rollback is "don't commit." If phase 4 (commit) begins, each commit primitive runs to completion. There is no "commit failed partway" path for single-step syscalls. Errors detected after a commit point are deferred to the next step invocation per STEP-8.

### 3.6 Compositional Syscalls

A syscall may be composed of multiple steps, each of which satisfies §3.1–§3.5 individually. The syscall as a whole need not be atomic; it is a composition of steps that are individually atomic.

Simple syscalls (open, unlink, pipe2, close) are single-step. Complex syscalls (execve, splice, sendfile, epoll_wait, waitpid, clone3-with-CLONE_VFORK) are compositional across multiple steps.

When a syscall is compositional:

- Each step within it satisfies §3.1–§3.5.
- Failure at any step reports accumulated progress to userspace per the syscall's POSIX semantics (short read/write, EINTR with partial progress, etc.).
- Rollback of prior steps is neither expected nor generally possible (STEP-8).
- Authorization is established by each step's observe sub-phase under a fresh guard (STEP-7); retained `Cap<T>` carries identity stability across step boundaries but not predicate truth.

Steps within a compositional syscall may interleave with reactor yields. Observers may see the syscall in intermediate committed states. This is normal for syscalls that are not specified atomic.

### 3.7 Point of No Return

Some compositional syscalls have a "point of no return" — a first commit point after which clean failure is no longer possible. Beyond this point, failure in a subsequent step may require process termination (execve-family) or is accepted as partial progress (splice-family).

Syscall scripts should structure their step sequence such that all fallible phases (observe, upgrade, reserve) occur before any commit point that constitutes a point of no return. This is a convention, not a framework requirement.

---

## 4. Substrate: Publication and Mutation Primitives

`substrate/` provides primitives for two concerns:

- **Publication**: making previously-absent state newly visible. Handles creation. Three families: zone, index, credit.
- **Mutation**: transforming already-visible state while preserving observer-safety. Handles update and removal. One family: mutation (withdraw, swap, conditional-commit).

All primitives share a two-phase shape (reserve + commit, with drop-rollback) but do not implement a common trait.

```
substrate/
    zone/        object substrate — zone-backed slots for kernel entities
    index/       index substrate — claimed keys in hashes/radixes/tables (publication)
    credit/      accounting substrate — refundable rlimit/cgroup charges
    mutation/    transition substrate — observer-safe removal, swap, conditional-commit
```

### 4.1 Zone Reservation (publication)

Objects allocated from a zone. The reservation holds the slot; signing writes the value and produces a `Cap`.

```rust
// substrate/zone.rs
pub struct ZoneReservation<T> { slot: ZoneSlot, _marker: PhantomData<T> }

pub fn reserve<T>(zone: &Zone<T>) -> Result<ZoneReservation<T>, Errno>;
pub fn sign<T>(r: ZoneReservation<T>, value: T) -> Cap<T>;

// Drop on ZoneReservation releases the slot back to the zone.
```

Used for: all fixed-size kernel entities.

### 4.2 Index Reservation (publication)

Slots in a subsystem index. The reservation guards the key; commit publishes the value at the key.

```rust
// substrate/index.rs
pub struct IndexReservation<K, V> { /* guard + key */ }

pub fn reserve_in<K, V>(index: &Index<K, V>, key: K) -> Result<IndexReservation<K, V>, Errno>;
pub fn commit<K, V>(r: IndexReservation<K, V>, value: V) -> Cap<V>;

// Drop on IndexReservation releases the key guard without publishing.
```

Used for: fd slot allocation, pid number allocation, any namespace-indexed publication of previously-absent state.

### 4.3 Credit Reservation (publication of a charge)

Refundable accounting against a resource budget. Reservation decrements the budget; commit finalizes the charge; drop refunds.

```rust
// substrate/credit.rs
pub struct CreditReservation { /* bag, kind, amount */ }

pub fn reserve(bag: &RLimitBag, kind: RLimitKind, amount: u64)
    -> Result<CreditReservation, Errno>;
pub fn commit(r: CreditReservation);

// Drop on CreditReservation refunds the amount to the bag.
```

Used for: `RLIMIT_NOFILE`, `RLIMIT_NPROC`, `RLIMIT_AS`, `RLIMIT_STACK`, cgroup accounting.

### 4.4 Convention, Not Trait

Substrate families share the reserve/commit/drop shape but do not implement a common trait. Each family has its own reservation type, its own commit function signature, and its own Drop semantics. Subsystems use them by name, not through generic traits.

### 4.5 Observer-Safe Mutation Primitives

Publication primitives (§4.1–§4.3) handle operations that make absent state newly visible. They are observer-safe by construction: before commit, the target key has no value; after commit, it has the value; no intermediate state is visible.

Many operations mutate *already-visible* state. Rename replaces a binding at a key with a different binding at a different key. Unlink withdraws a binding from a key. These operations cannot be realized as sequential "remove then insert" without creating observable windows where namespace invariants fail (e.g., rename's "destination must always resolve to either old or new target" would be violated by a brief absent window).

The mutation substrate family provides primitives whose commit is a single observer-visible transition:

```rust
// substrate/mutation.rs (conceptual; exact API is implementation-layer)

// Atomic removal: key goes from Some(expected) to absent.
pub struct WithdrawReservation<K, V> { /* guard + key + expected reference */ }

pub fn withdraw<K, V>(
    index: &Index<K, V>,
    key: K,
    expected: &Cap<V>,
) -> Result<WithdrawReservation<K, V>, Errno>;

pub fn withdraw_commit<K, V>(r: WithdrawReservation<K, V>) -> Cap<V>;
// Returns the withdrawn evidence for teardown; operation is observer-atomic.


// Atomic replacement: key goes from Some(expected_old) to Some(new).
pub struct SwapReservation<K, V> { /* ... */ }

pub fn swap<K, V>(
    index: &Index<K, V>,
    key: K,
    expected_old: &Cap<V>,
) -> Result<SwapReservation<K, V>, Errno>;

pub fn swap_commit<K, V>(r: SwapReservation<K, V>, new_value: V) -> (Cap<V>, Cap<V>);
// Returns (new, old). Atomic from observer's perspective.


// Conditional commit: insert only if a condition holds, atomically.
pub fn register_if<K, V, F>(
    queue: &Queue<K, V>,
    key: K,
    entry: V,
    condition: F,
) -> RegisterOutcome<V>
where F: FnOnce() -> bool;
// Used for futex-style "check and register" atomicity.
```

**The architectural invariant**: a substrate primitive is observer-safe iff its commit moment is a single linearization point, and the structure satisfies its steady-state invariants at every moment outside the substrate's internal atomic window.

**Execution selects primitives by operation type.** Publication primitives for create/open/fork-style operations; mutation primitives for unlink/rename/replace-style operations; conditional-commit primitives for check-and-register operations. The framework requires that execution never synthesize a transition from sequential publication + un-publication when a mutation primitive exists, because sequential composition loses observer-safety.

**Flag coupling.** Flags co-located with containers (like `removed` on a DEntry, `detached` on a Mount) are set by the relevant mutation primitive atomically with the container-level change. Two separate stores (withdraw from container; then set `removed = true`) permit observable inconsistency; the substrate primitive fuses them.

**Cross-subsystem coordination.** Some operations (notably mmap/munmap with PTE teardown and TLB invalidation) require coordinated atomicity across more than one substrate. The architecture identifies the need for such coordinated primitives; the precise shape is implementation-layer. The requirement: from any concurrent observer's perspective, the binding and its derived artifacts (PTEs, cached translations) are consistent — either the binding holds and accesses succeed, or the binding is gone and accesses fail cleanly.

### 4.6 Substrate Is Not a Subsystem

Substrate primitives know nothing about kernel semantics. They know about zones, keys, atomicity, and refcounts. Naming, obligation-typing, and evidence derivation live in `object_model_0417.md` and are enforced at the subsystem layer via type discipline.

---

## 5. Worked Example: Create Pipe

Full five-phase structure of `pipe(2)` shown as one `step_pipe` function. The worked example below factors the body into two helper functions (`pipe_prepare`, `pipe_commit`) for readability — the separation is purely syntactic and internal to the step; both helpers execute synchronously within one step invocation, together covering phases 2 through 5 of STEP-4. A conforming implementation may inline the helpers, keep them separate, or structure the body any other way that preserves the phase ordering and reservation linearity. Illustrative; exact substrate primitive signatures are implementation-layer.

### 5.1 Dispatch Entry

```rust
// dispatch/syscalls/pipe.rs — calls step_pipe; helpers are internal to the step.
async fn sys_pipe(ctx: &ThreadContext, fds_out: UserAddr) -> Result<(), Errno> {
    // Phase 1: observe
    let witness = cred::checks::require_fd_creation(
        &ctx.cred, &ctx.fd_table, 2, &ctx.guard
    )?;

    // Phases 2 + 3: upgrade + reserve (internal helper, still within the step)
    let reservations = pipe_subsys::execution::pipe_prepare(
        witness,
        &ctx.rlimits, &ctx.fd_table, &ctx.zones, &ctx.heap,
    )?;

    // Phases 4 + 5: commit + publish (internal helper, still within the step)
    let published = pipe_subsys::execution::pipe_commit(reservations);

    vm::copy_to_user(&ctx.space, fds_out, &[published.read_fd, published.write_fd])?;
    Ok(())
}
```

### 5.2 The Check

```rust
// subsystems/cred/checks/require.rs
pub fn require_fd_creation<'g>(
    cred: &'g CredentialSnapshot,
    fd_table: &'g FdTable,
    count: u32,
    guard: &'g Guard,
) -> Result<FdCreationAuthorized<'g>, Errno> {
    if !rlimits_permit_fd_creation(cred, count) {
        return Err(EMFILE);
    }
    Ok(FdCreationAuthorized {
        fd_table: IdentRef::observe(fd_table, guard),
        cred:     IdentRef::observe(cred, guard),
    })
}

// Witness lifetime-parameterized; IdentRef-carrying; !Send + !Sync.
pub struct FdCreationAuthorized<'g> {
    pub(crate) fd_table: IdentRef<'g, FdTable>,
    pub(crate) cred:     IdentRef<'g, CredentialSnapshot>,
}
```

### 5.3 The Upgrade + Reserve Helper (phases 2 + 3)

```rust
// subsystems/pipe/execution/pipe.rs
pub fn pipe_prepare<'g>(
    witness:  FdCreationAuthorized<'g>,
    rlimits:  &RLimitBag,
    fd_table: &FdTable,
    zones:    &ZoneBundle,
    heap:     &Heap,
) -> Result<PipeReservations, Errno> {
    // Upgrade observations to retention. After this, 'g is gone.
    let fd_table_cap = witness.fd_table.upgrade_to_cap()?;
    // cred moves out of witness as a value; no upgrade needed.
    
    // Reserve zone slots
    let pipe_slot  = substrate::zone::reserve(&zones.pipe_data)?;
    let read_slot  = substrate::zone::reserve(&zones.pipe_read_end)?;
    let write_slot = substrate::zone::reserve(&zones.pipe_write_end)?;
    
    // Reserve fd slots (may fail if table full despite policy permitting)
    let fd_read  = substrate::index::reserve_in(&fd_table_cap, NextAvailable)?;
    let fd_write = substrate::index::reserve_in(&fd_table_cap, NextAvailable)?;
    
    // Reserve rlimit credit (may fail on race despite check passing)
    let charge = substrate::credit::reserve(rlimits, RLimitKind::NOFILE, 2)?;
    
    // Construct values into owned memory (not yet in slots)
    let pipe_data = PipeData::new(heap);
    let read_end  = PipeReadEnd::new(/* references resolved at sign time */);
    let write_end = PipeWriteEnd::new(/* ... */);
    
    Ok(PipeReservations {
        pipe_slot, read_slot, write_slot,
        fd_read, fd_write, charge,
        pipe_data, read_end, write_end,
    })
}
```

If any reserve fails, previously-acquired reservations drop → slot released, credit refunded, key guard released. Zero side effects.

### 5.4 The Commit + Publish Helper (phases 4 + 5)

```rust
pub fn pipe_commit(r: PipeReservations) -> PipePublished {
    // Sign the zone reservations → Caps / operational evidence
    let pipe_cap = substrate::zone::sign(r.pipe_slot, r.pipe_data);
    
    // Construct OpenPins (operational contributions) for the two ends.
    // This is where the RNode's open_refs disjunct receives its contribution.
    let read_pin  = OpenPin::construct(&pipe_cap);
    let write_pin = OpenPin::construct(&pipe_cap);
    
    let read_end  = substrate::zone::sign(r.read_slot,
        PipeReadEnd { open_pin: read_pin, /* ... */ });
    let write_end = substrate::zone::sign(r.write_slot,
        PipeWriteEnd { open_pin: write_pin, /* ... */ });
    
    // Publish into fd table via observer-safe index commits.
    let read_fd  = substrate::index::commit(r.fd_read,  FdEntry::for_pipe_read(read_end));
    let write_fd = substrate::index::commit(r.fd_write, FdEntry::for_pipe_write(write_end));
    
    // Finalize rlimit charge
    substrate::credit::commit(r.charge);
    
    // Fire tracepoints
    pipe_subsys::trace_pipe_created(read_fd.raw(), write_fd.raw());
    
    PipePublished { read_fd, write_fd }
}
```

No `Result`. Every substrate call here is observer-safe publication; each is an atomic transition; their aggregate commits the pipe atomically from external observers' perspective.

---

## 6. Dependency Graph

```
foundation/
    zone, persistent_bt, persistent_rx, dll, mm, heap, epoch, pin, binding
    ↑
substrate/
    zone, index, credit, mutation — two-phase primitives with observer-safe commit
    knows nothing about subsystems or semantics
    ↑
bus/
    queue, port, trace, pollable — notification substrate
    ↑
subsystem/structure/     ← SSoT for semantic indexes (holds published evidence)
    ↑
subsystem/checks/        ← observation: predicates, require, witnesses, trampoline
                         ← reads structure/ under guard; no writes, no substrate, no bus publication
    ↑
subsystem/execution/     ← effectful: composes substrate + structure + bus
                         ← consumes witnesses; upgrades; installs bindings
    ↑                     ↑
dispatcher/          pipeline/
    drives steps         orchestrates cross-subsystem
    trampoline,          observe → upgrade → reserve
    invokes step_*       → commit → publish
    functions            across subsystems
    with witnesses
```

No cycles. Each module depends only on strictly lower-level concerns.

---

## 7. Import Rules (Lint Specification)

### 7.1 checks/ imports

**May import from:**
- `foundation::{zone, heap, ...}` — primitive types
- `bus::{Waker, PollMask, ...}` — passive types only, not setters
- Own `subsystem::structure::*` — read-only access
- Other subsystems' `checks::{require_*, witness types}` — for cross-subsystem predicates

**May NOT import from:**
- `substrate/` — no reservation, no zone allocation, no credit accounting, no mutation primitives
- Any subsystem's `execution/` — checks cannot call into mutation code
- `bus::{set, emit, fire, publish}` — no observable notifications

### 7.2 execution/ imports

**May import from:**
- All lower layers: foundation, substrate, bus
- Own subsystem's structure/, checks/ (for witness types)
- Other subsystems' execution/ — only within pipeline scripts

**May NOT import from:**
- Another subsystem's structure/ directly (must go through that subsystem's checks/)
- Another subsystem's private internals

### 7.3 structure/ imports

**May import from:**
- foundation/, substrate/ (for Cap types, primitive collections)
- bus/ passive types

**May NOT import from:**
- Other subsystems
- checks/ or execution/ (lower layer — inversion)

### 7.4 project.rs imports

**May import from:**
- Subsystem's own structure/ (read-only)
- foundation/ types

**May NOT import from:**
- execution/ (projections never mutate)
- checks/ (projections are not authorized operations)
- substrate/

### 7.5 Enforcement Approach

Rust module visibility (pub / pub(crate) / private) handles most cases. AST-based lints catch the rest:

- Forbidden import paths per module.
- `checks/*.rs` calling methods matching `set_*`, `emit_*`, `fire_*`, `insert_*`, `remove_*`, `mutate_*`.
- `checks/*.rs` importing `substrate::`.
- Witness types stored in struct fields, static bindings, `Vec`/`HashMap`, or returned/held across `.await` boundaries.
- `mem::forget` or `ManuallyDrop::new` on reservation types outside `substrate/`.
- Raw predicate reimplementation: any external code reading subsystem-internal flags (`dentry.removed`, `process.state`, `mount.detached`) outside the canonical predicate function.

These run as pre-commit hooks or CI gates.

---

## 8. Service Subsystem Variant

Service subsystems (cred, rlim, trace, sched, time, dev) provide utility functions rather than owning entities:

- `structure/` — shared state (credential bag, rlimit bag, tracepoint table)
- `checks/` — pure queries (policy permission, capability lookup)
- Mutation functions organized per the five-phase step discipline where applicable
- `project.rs` — projections if user-visible

Service subsystems are called *by* full subsystems. `pipe::execution::step_pipe`'s reserve sub-phase calls `credit::substrate::reserve` (for RLIMIT_NOFILE charge) and consumes `cred::checks::require_fd_creation`'s witness during its observe sub-phase. This direction is fine: services sit below full subsystems in the dependency graph.

---

## 9. Cross-Subsystem Pipelines

Some operations span multiple subsystems. These live in `pipeline/`:

```
pipeline/
    fork.rs         proc → vm → vfs → sig → proc
    execve.rs       vfs → vm → sig → proc → vfs   (compositional; see §3.6)
    mmap.rs         vfs → vm
    splice.rs       vfs × vfs                     (compositional; streaming)
    accept.rs       net → vfs → net
    mount.rs        mount → vfs
    ...
```

A single-step pipeline script follows the five-phase discipline and aggregates across subsystems:

```rust
// pipeline/fork.rs (simplified)
fn step_fork_pipeline(
    parent_ctx: &ThreadContext,
    flags: CloneFlags,
) -> StepOutcome<Pid> {
    let guard = epoch::guard();

    // Phase 1: observe — gather witnesses from each subsystem's checks/ under one guard
    let proc_w = proc::checks::require_fork_permitted(parent_ctx, &guard)?;
    let vm_w   = vm::checks::require_address_space_clonable(parent_ctx, flags, &guard)?;
    let sig_w  = sig::checks::require_sigactions_clonable(parent_ctx, flags, &guard)?;
    let vfs_w  = vfs::checks::require_fd_table_clonable(parent_ctx, flags, &guard)?;

    // Phase 2: upgrade — promote each subsystem's IdentRefs to retention evidence
    let proc_cap = proc_w.upgrade()?;
    let vm_cap   = vm_w.upgrade()?;
    let sig_cap  = sig_w.upgrade()?;
    let vfs_cap  = vfs_w.upgrade()?;

    // Phase 3: reserve — each subsystem's reserve sub-phase
    let proc_r = proc::execution::fork_reserve(proc_cap, ...)?;
    let vm_r   = vm::execution::fork_reserve(vm_cap, ...)?;
    let sig_r  = sig::execution::fork_reserve(sig_cap, ...)?;
    let vfs_r  = vfs::execution::fork_reserve(vfs_cap, ...)?;

    let bundle = ForkReservations { proc_r, vm_r, sig_r, vfs_r };

    // Phase 4: commit — each subsystem's commit sub-phase, in order
    proc::execution::fork_commit(bundle.proc_r);
    vm::execution::fork_commit(bundle.vm_r);
    sig::execution::fork_commit(bundle.sig_r);
    let pid = vfs::execution::fork_commit(bundle.vfs_r);

    // Phase 5: publish — each subsystem fires signals for its commit points
    //   (fires are attached to specific commit points above and happen inline
    //    with each subsystem's commit; shown here conceptually)

    StepOutcome::Done(pid)
}
```

The bundle struct composes per-subsystem reservations. Linearity is preserved because each field is non-Copy. If any phase-3 reserve fails, earlier-acquired reservations drop → each subsystem rolls back independently.

The commit sub-phase is infallible end-to-end. Each subsystem's commit helper is infallible; the composition is infallible.

**Ordering within phase 4 matters for observability but not for correctness.** A second thread might observe proc changes before vfs changes. Under the concurrency contract (see `object_model_0417.md` §6), this is a linearization detail: the partial state is not semantically invalid, just incomplete. POSIX does not require cross-subsystem fork to appear atomic to concurrent observers. Each commit call is a distinct commit point with its own visibility boundary; STEP-8 attaches per boundary, not to the bundle as a whole.

For compositional syscalls (§3.6), the pipeline script executes a sequence of steps rather than a single five-phase step. Each step follows the five-phase discipline individually.

---

## 10. Testing Boundaries

The four-module split creates crisp test boundaries.

**structure/ tests:** construct data, assert accessors return expected values. No mocks.

**checks/ tests:** construct structure state under a synthetic guard, call predicates, assert bool results. Construct contexts, call require functions, assert witness produced or errno returned. No mocks beyond structure construction.

**execution/ tests:** mock heap + waker + bus. Construct witnesses directly (witness types have pub(crate) constructors for testing under a test guard). Invoke step functions; assert outcomes of each phase — observe produces expected witnesses, upgrade succeeds or fails cleanly, reserve holds linear tokens, commit points publish to mocked bus, publish phase fires expected signals.

**project.rs tests:** construct structure, call projections, assert output text.

**pipeline/ tests:** full integration — construct a thread context, run the pipeline, verify end-to-end behavior. The only level requiring multi-subsystem coordination.

---

## 11. Adding a New Subsystem

Checklist:

1. **Identify entity types.** What entities does this subsystem own? Per `object_model_0417.md` §3, determine each entity's payload organization (co-located / indirected / compound-predicate).
2. **Draft structure/.** Indexes and flags. pub(crate) fields.
3. **Enumerate projections.** See `LIVENESS.md` §3 for the catalog format.
4. **Write checks/predicates.rs.** One predicate per projection.
5. **Write checks/witness.rs.** Witness types, lifetime-parameterized, !Send + !Sync, private constructors, contain the IdentRefs that the step's upgrade sub-phase will consume.
6. **Write checks/require.rs.** One require per operation entry point.
7. **For each operation, write execution/step_*.rs with one step function per operation.**
   - Observe: acquire fresh guard, consume witness.
   - Upgrade: promote IdentRefs to retention (OBL-4); fallible.
   - Reserve: acquire linear substrate tokens; fallible.
   - Commit: consume reservations at visibility boundaries via commit primitives; infallible.
   - Publish: fire declared signal attachments for each commit point.
8. **If projections are user-visible, write project.rs.**
9. **Register in dispatch/route.rs** for syscalls that land directly in this subsystem.
10. **Add the subsystem to any pipeline scripts** it participates in.
11. **Audit cross-subsystem registrations for stable identity keys** (per `object_model_0417.md` §6.4).
12. **Document in `LIVENESS.md`'s projection catalog.**

---

## 12. Why This Structure

- **checks/ purity** (observation only, never retention) preserves the reachability model and the concurrency contract.
- **execution/ centralization** makes mutation auditable. Every observer-visible write crosses a commit point inside a step, and every commit point is authorized by a witness the same step has upgraded.
- **substrate/ separation** makes resource management uniform. Every resource follows the two-phase protocol with linear reservations.
- **Publication vs. mutation split** (§4.5) makes observer-safety enforceable at the primitive layer, not at per-subsystem discretion.
- **project.rs isolation** makes procfs non-intrusive. Read-only projection never interferes with authorization or mutation paths.
- **Witness lifetime discipline** prevents accidental retention leaks from checks into long-lived state.
- **Linear reservations** eliminate rollback code. The type system ensures every reservation reaches exactly one terminal state (commit or drop-to-rollback).

---

## References

- [`object_model_0417.md`](./object_model_0417.md) — semantic object model: entities, projections, bindings, obligations, retention, concurrency contract
- [`INVARIANTS.md`](./INVARIANTS.md) — STEP-4 codifies the five-phase discipline stated here as invariants
- [`STEP_MODEL.md`](./STEP_MODEL.md) §3 — the same discipline as seen from the execution-primitive view
- [`CONCEPTS.md`](./CONCEPTS.md) §2.2 — sub-phase vocabulary
- [`LIVENESS.md`](./LIVENESS.md) — application checklist and per-subsystem projection catalog
