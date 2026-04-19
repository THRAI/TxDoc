# Liveness Framework

**Status.** v2.1 (2026-04-19). Terminology aligned with the step model: witness consumption is at STEP-4 phase 2 (upgrade); reservation is at phase 3; `ADR-resolution-half.md` references removed in favor of `object_model_0417.md`.

**Supersedes (v2.1 → v2).** Terminology sync only; no semantic changes.

**Supersedes (v2 → v1).** LIVENESS v1 core claims where the two disagree; `object_model_0417.md` is authoritative for vocabulary.

**Purpose.** Define the projection / require / witness framework used within `checks/` modules. State monotonicity, race-degradation, and binding-preservation as architectural invariants. Provide the catalog of projections per subsystem with explicit realization mechanism for each.

**Audience.** Designers adding new kernel entity types, reviewers auditing concurrency properties, agents extending subsystems.

**Companion documents.**
- [`object_model_0417.md`](./object_model_0417.md) — semantic object model (entities, references, projections, bindings). Authoritative for terminology. Read first.
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) — four-module layout, five-phase step discipline, substrate primitives.
- [`INVARIANTS.md`](./INVARIANTS.md) — PRED-* invariants realize the framework stated here; STEP-4 names the sub-phases this catalog references.
- [`STEP_MODEL.md`](./STEP_MODEL.md) — step primitive; witness lifetime and upgrade phase.

---

## 1. Motivation

A kernel entity does not have a single "alive" bit. An unlinked-but-open file is operationally alive for I/O but namespace-gone. A zombie process is addressable for `waitpid` but not signal-deliverable. A detached mount is reachable by existing resolvers but not by new walks. A closed fd is gone from its process's table but the underlying open-file description may persist via dup or via another fd.

Previous designs scattered these distinctions across subsystem-specific flags (`removed`, `zombie`, `detached`) checked at individual call sites. The result was semantic duplication, inconsistent errno selection, and races that produced "incorrect success" — operations succeeding against entities in degraded states.

This framework lifts validity into a first-class concern: a finite set of **projections** per entity type, each realized by a specific retention or reachability mechanism, all consumed exclusively through **require functions** that produce **witnesses**.

The vocabulary distinctions this framework depends on:

- **Binding**: a structural entry in a namespace container. Carries a key and retention evidence (whose type is derived from the binding's declared obligation). Existence is governed by the container; no independent refcount.
- **Observation**: a guard-scoped view of an entity via `IdentRef<'g, T>`. Contributes no retention. Memory-safe because the epoch guard defers physical reclamation.
- **Identity retention**: a `Cap<T>`. Pins the entity's slot against physical reclamation; does not guarantee payload liveness.
- **Payload retention**: `PayloadCap<T>` or a typed operational contribution (LinkPin, OpenPin, MapPin, CachePin, ...). Pins one disjunct of the payload projection.
- **Obligation**: a typed promise a binding makes toward its target — resolution-only, addressability, or operational.

Reachability through a chain of bindings to an entity is **not** retention. An entity's namespace projection being true (reachable) is a reachability claim over the container structure, which does not by itself keep the entity's payload alive and does not by itself keep the entity's identity alive. The entity's identity and payload are kept alive by whatever evidence the bindings along the chain carry — which depends on each binding's declared obligation.

---

## 2. The Framework

### 2.1 Layers

```
Type layer        IdentRef<'g, T>, Cap<T>,           structural safety
                  PayloadCap<T>, contributions       (from object_model §4)
                  ↓ enables
Predicate layer   p(obj, ctx) -> bool                semantic validity
                  ↓ invoked by
Require layer     require_*() -> Witness             entry enforcement
                  ↓ produces
Witness layer     Witness<'g> { ... }                observation-level evidence
```

Each layer has exactly one job:

- **Type layer.** Memory safety and retention vocabulary. Four reference tiers per object_model §4. `IdentRef<'g, T>` is the traversal primitive: epoch-scoped observation, no retention. `Cap<T>` is identity retention. `PayloadCap<T>` and the typed operational contributions are payload retention. Downgrade is free; upgrade is fallible via the SENTINEL_DEAD CAS.

- **Predicate layer.** Semantic definition. Pure functions from `(entity, context) → bool`. Encode what it means for a projection to hold. Read entity state and container structure under an epoch guard; do not mutate anything.

- **Require layer.** Authorization boundary. The only sanctioned way to consume a projection. Handles errno selection, invokes predicates, produces witnesses carrying IdentRefs to the checked entities.

- **Witness layer.** Observation-level evidence of past validity, scoped to the producing epoch guard. Witnesses are `!Send + !Sync` by construction of their IdentRef fields. Consumed by the step's upgrade sub-phase (STEP-4 phase 2), which promotes `IdentRef<'g, T>` to retention evidence (`Cap<T>` or `T::OperationalEvidence`) per the operation's obligations.

### 2.2 Projections

Each entity type exposes one or more **projections** — derived predicates over its state and system context. Projections are not stored fields; they are computed from entity-local data, container structure, external pinners, and the observation viewpoint.

Projections are:

- **Finite per entity type.** Typically 2–4.
- **Viewpoint-dependent.** `p(entity, context) → bool`.
- **Monotone.** Transition only from true to false.
- **Realized by a specific mechanism** — each projection is defined in terms of exactly one of: binding-chain reachability, identity retention, or payload retention (§2.4).

### 2.3 Monotonicity

> Projections transition only from true to false, never back.

Operations that lower a projection (`unlink`, `close`, `exit`, `umount`) move it downward. No operation restores a lost projection — new entities are created instead. This is the load-bearing property guaranteeing that a failed check is genuinely a failure (not racing a revival), and that a past "true" observation is permanent evidence of having been true at a prior moment.

### 2.4 Mechanism per projection kind

Per object_model §5, projections correspond to one of three distinct mechanisms. The following table is the **architectural rule**: every projection declared in §3 must state which mechanism realizes it.

| Projection kind | Realizing mechanism | Concrete evidence |
|---|---|---|
| Structural | Identity retention | Entity's retention count > 0 (some `Cap<T>` extant in the system) |
| Namespace / addressability | Binding-chain reachability | Chain of bindings from a namespace root reaches the entity |
| Payload | Payload retention | `PayloadCap<T>` extant, or a disjunction of typed operational contributions is nonzero |

Each subsystem's projection catalog states its realization mechanism per projection. **No claim of the form "namespace implies structural" is made generically.** Such implications are derived per-subsystem where they hold (because the binding carrying reachability also carries identity-retention evidence due to its declared obligation), and stated explicitly as theorems — never assumed.

Namespace projections are reachability queries computed by walking binding containers. Flag fields that appear on entities (`removed` on a DEntry, `detached` on a Mount) are fast-path staleness hints for common cases; the canonical predicate consults the container.

### 2.5 Race-degradation theorem

> Under monotonicity, container-linearized binding mutation, and SENTINEL_DEAD-guarded upgrade, concurrent mutation may cause operations to fail but cannot cause them to succeed against a different entity, a different context, or with different semantics than the operation's arguments describe. **Races degrade to failure, not to silent incorrect success.**

This is an **architectural invariant**. It is enforced by:

1. Projections transition only downward (§2.3).
2. Checks phase is side-effect-free; a stale `true` observed in checks that becomes `false` before the step's upgrade sub-phase is benign — the upgrade fails cleanly through SENTINEL_DEAD CAS, and since phase 3 (reserve) has not yet executed, no reservations exist to roll back.
3. Binding-mutation substrate primitives linearize at single atomic transitions (see `SUBSYSTEM_ANATOMY.md` §4.5).
4. Upgrade from observation to retention is a single linearization point.
5. Witnesses hold IdentRefs tied to their producing guard; subsequent operations proceed strictly through the upgraded evidence, which is by construction the entity the witness observed.

Identity cannot be silently redirected; validity may lapse cleanly; semantics remain fixed to the operation's arguments.

### 2.6 The observer contract

Extending race-degradation to cross-subsystem binding observation:

> **Binding-preservation monotonicity.** No binding-preserving operation may cause an in-flight walker or reader to observe a different target for a binding (parent, name) than the target that held when the walker last observed it. Concurrent binding-preserving operations preserve the originally observed key-target relation. Binding-mutating operations may cause in-flight walkers to fail, but may not cause them to silently retarget.

*Binding-preserving operations.* Cache population, metadata updates, atime writes, content reads and writes, flushes. These do not modify the namespace graph's binding structure.

*Binding-mutating operations.* Rename, unlink / rmdir, mount / umount, create (adds binding). These modify binding structure, and must go through observer-safe substrate mutation primitives.

The contract constrains the mutation mechanism (§2.5.3) and evidence-upgrade linearization (§2.5.4). It does **not** constrain cache strategy, content-operation ordering, or commit sub-step ordering within a binding mutation — those remain subsystem freedoms (§2.7).

### 2.7 Architecture vs implementation

This framework is the **architecture layer**. It defines *what must hold*:

- Projections are monotone.
- Each projection is realized by exactly one of (identity retention, binding-chain reachability, payload retention).
- Witnesses are observation-level evidence (carry IdentRef, not Cap).
- Require functions are the sole entry point for projection consumption.
- Cross-subsystem binding observation satisfies the contract in §2.6.

It does **not** prescribe:

- Container data structures — radix, B-tree, hash, DLL are all admissible provided they satisfy the substrate linearization contract.
- Cache strategy — LRU, pressure-driven, eager, lazy are all admissible.
- Commit sub-step ordering within substrate-primitive atomicity.
- Sync policy for durable state — write-through, write-back, journaling are all admissible.

The implementation layer (subsystem authors + substrate implementers) supplies concrete realizations meeting these invariants.

---

## 3. Per-Subsystem Projection Catalog

Each row states the realizing mechanism explicitly.

| Subsystem | Projection | Context | Meaning | Mechanism |
|---|---|---|---|---|
| **VFS — DEntry** | structural | — | Retention > 0 | identity retention |
| | namespace | MountNamespace | reachable from mnt_ns root via binding chain | reachability |
| | payload (derived) | — | resolved RNode has payload_live | derived via resolution arrow to RNode.payload |
| **VFS — RNode** | structural | — | Retention > 0 | identity retention |
| | payload | — | LinkPin-count > 0 ∨ OpenPin-count > 0 (persistent) or projection-state-valid (synthetic) | payload retention (compound) |
| **PID — ProcessIdentity** | structural | — | Retention > 0 on identity zone slot | identity retention |
| | wait_addr | parent relation | in parent's children binding | reachability (within parent-children container) |
| | signal_addr | PidNamespace | pid binding present ∧ identity.payload.is_some() | reachability + payload-presence |
| **PID — ProcessPayload** | structural | — | Retention > 0 on payload zone slot | identity retention |
| | payload | — | `ProcessIdentity.payload` field is Some (PayloadCap extant) | payload retention |
| **PID — ThreadIdentity** | structural | — | Retention > 0 | identity retention |
| | in_chain | ProcessIdentity | reachable through owning process's thread chain | reachability |
| **PID — ThreadPayload** | structural | — | Retention > 0 | identity retention |
| | payload | — | `ThreadIdentity.payload` field is Some | payload retention |
| **FD — entry** | structural | FdTable | slot installed | reachability in fd_table |
| | payload | — | per-open struct reachable via Cap | operational evidence chain |
| **MountIdentity** | structural | — | Retention > 0 | identity retention |
| | namespace | MountNamespace | attached to mount tree | reachability |
| **MountPayload** | structural | — | Retention > 0 | identity retention |
| | payload | — | superblock valid, backing responsive; `MountIdentity.payload` is Some | payload retention |
| **SocketIdentity** | structural | — | Retention > 0 | identity retention |
| | addressable | Protocol | bound (and listening if applicable) in protocol table | reachability |
| **SocketPayload** | structural | — | Retention > 0 | identity retention |
| | payload | — | protocol state not CLOSED and not fully drained; `SocketIdentity.payload` is Some | payload retention |
| **VM — VmEntry** | structural | — | Retention > 0 | identity retention |
| | namespace | AddressSpace | present in recipes binding container | reachability |
| | payload | — | backing accessible | payload retention |
| **Frame** | structural | — | Retention > 0 | identity retention |
| | payload (compound) | — | map_count-MapPin > 0 ∨ cache_ref-CachePin > 0 | payload retention (compound) |

Subsystem-local vocabulary is accepted (namespace for VFS, addressable for sockets, wait_addr for processes). Uniform terminology is not required.

**Why multiple entities per subsystem (Process, Thread, Mount, Socket each appear as two rows):** entities whose partial order admits `structural ⟂ payload` are factored into `<Name>Identity` and `<Name>Payload` with independent zones and independent reclamation lifetimes, per the factoring convention in [`object_model.md`](./object_model_0417.md) §8.1.1. The two-row presentation in this catalog reflects the two zone-allocated types. Bindings in containers target Identity; payload is reached through `Identity.payload: Option<PayloadCap<Payload>>`.

### 3.1 Partial orders per subsystem

Partial orders express **implications between projections within one entity type**. They are per-subsystem theorems, not generic properties. Each implication is derived from the concrete binding and retention structure of that subsystem.

**VFS DEntry.**

```
structural ⇐ namespace
```

*Derivation.* A namespace projection being true means a chain of bindings reaches the DEntry. Along that chain, the binding directly referencing the DEntry carries operational evidence (`Cap<DEntry>` or stronger) because DEntry bindings declare operational obligation. Therefore identity retention is nonzero, so structural is true. This implication is an honest derived theorem, not a generic property.

```
payload ⟂ namespace
```

*Unlinked-but-open* violates both directions: namespace may be false while payload holds (OpenPin keeps inode payload alive even after nlinks = 0 and the DEntry is no longer reachable).

**VFS RNode.**

```
structural ⇐ payload
```

*Derivation.* payload retention (a LinkPin or OpenPin extant) implies identity retention per object_model §5.3 (operational evidence implies `Cap<T>`).

**PID ProcessIdentity.**

```
structural ⇐ wait_addr         (binding in parent.children carries Cap<ProcessIdentity>)
structural ⇐ signal_addr       (binding in pid_hash carries Cap<ProcessIdentity>)
signal_addr ⇒ wait_addr        (subsystem-specific; signal-addressable implies wait-addressable)
```

*Note the absence of a `payload` projection on ProcessIdentity.* Payload is a projection on `ProcessPayload`, not on the identity. `ProcessIdentity.payload: Option<PayloadCap<ProcessPayload>>` being Some is observable through the identity and gates signal_addressability (`signal_addr` requires both pid-hash binding AND ProcessIdentity.payload.is_some()).

**PID ProcessPayload.**

```
structural ⇐ payload
```

*Derivation.* Operational evidence (any contribution pinning ProcessPayload's disjuncts) implies identity retention on the ProcessPayload slot. Trivial.

**PID — the split in operation.**

```
ProcessIdentity.wait_addr ⟂ ProcessPayload.payload
```

*Zombie:* ProcessIdentity.wait_addr is true (still in parent.children), ProcessPayload no longer exists (payload slot reclaimed at exit_commit). The two orthogonal projections live on two separate entities; there is no "zombie flag" to check — the presence or absence of ProcessIdentity.payload is the state.

**MountIdentity / MountPayload.**

```
MountIdentity.namespace ⟂ MountPayload.payload
```

*Force-umount / MNT_DETACH:* MountIdentity.namespace may be false (detached from tree) while MountPayload.payload held is still Some for active resolvers; or payload may be None (superblock flushed and released at umount_commit) while identity persists for Caps held by resolvers in flight.

**SocketIdentity / SocketPayload.**

```
SocketIdentity.addressable ⟂ SocketPayload.payload
```

*Closed socket with lingering fds:* SocketPayload.payload may be false (protocol drained, buffers released) while SocketIdentity persists for fds that have not yet closed (operations on such fds return ENOTCONN/EPIPE).

Partial orders are diagnostic, not directive. They document the coherent state combinations an entity may inhabit. Transition rules must respect them.

### 3.2 Derived projections

Some projections are defined **by reference to other entities' projections via resolution**. Example: DEntry.payload is "the resolved RNode has RNode.payload_live." This is a **definitional dependency**, not a retention dependency: the DEntry doesn't pin the RNode's payload; it inherits the predicate via the resolution arrow.

Definitional dependencies compose cleanly because each predicate has one canonical source. Implementation: DEntry.payload calls RNode.payload_live through the DEntry's resolution.

### 3.3 Operational contributions (pinning disjuncts)

Where multiple independent sources can pin payload, each source provides a typed contribution implementing `OperationalContribution<T>` per ADR §7:

- **RNode.payload** = LinkPin-count-nonzero ∨ OpenPin-count-nonzero (∨ CachePin-count-nonzero for cache retention if applicable).
- **Frame.payload** = MapPin-count-nonzero ∨ CachePin-count-nonzero.

Each contribution is constructed via an epoch-guarded CAS with a projection pre-check; drop decrements the specific counter. The entity's payload projection fires false when all disjunct counters drop to zero.

Contributions are a realization mechanism, not a new framework concept. They appear in the predicate as a disjunction over typed counters.

---

## 4. The Checks Module

Each subsystem's `checks/` submodule contains its predicates, require functions, and witness types. See `SUBSYSTEM_ANATOMY.md` §2.2 for module layout rules.

### 4.1 Predicates

```rust
fn namespace_live<'g>(d: IdentRef<'g, DEntry>, ctx: &ResolveCtx, g: &'g Guard) -> bool;
fn payload_live<'g>(rn: IdentRef<'g, RNode>, g: &'g Guard) -> bool;
fn signal_addressable<'g>(p: IdentRef<'g, ProcessIdentity>, ns: &PidNamespace, g: &'g Guard) -> bool;
```

Predicates are:

- **Pure.** No side effects, no allocation, no blocking.
- **Reusable.** Called from multiple require functions and multiple check points.
- **Single source of truth.** The semantic meaning of a projection lives here.
- **Guard-scoped.** Take `IdentRef<'g, T>`, which carries the epoch guard's lifetime.
- **Non-retaining.** Predicates never upgrade references or acquire contributions.

### 4.2 Require functions

```rust
pub fn require_namespace_child<'g>(
    parent: IdentRef<'g, DEntry>,
    name: &NameRef,
    ctx: &ResolveCtx,
    g: &'g Guard,
) -> Result<NamespaceChild<'g>, Errno>;
```

Require functions:

- Are the **only authorized consumer of projections.** All step-authorization logic enters through require, whose output is consumed by the step's upgrade sub-phase.
- Know POSIX errno conventions for the operation they gate.
- Acquire derived data (IdentRef to located entity) into the witness.
- Are fallible: `Result<Witness<'g>, Errno>`.

### 4.3 Witnesses

```rust
pub struct NamespaceChild<'g> {
    pub(in crate::vfs::checks) child: IdentRef<'g, DEntry>,
}

pub struct SignalTarget<'g> {
    pub(in crate::pid::checks) proc: IdentRef<'g, ProcessIdentity>,
}
```

Witnesses are:

- **IdentRef-carrying.** Observation-level evidence. Per object_model_0417 §7.6, checks produces IdentRef; the step's upgrade sub-phase promotes it.
- **Constructible only by require functions.** `pub(in crate::<subsystem>::checks)` fields; no public constructors.
- **Lifetime-bound.** `'g` ties them to the producing guard.
- **`!Send + !Sync`.** By transitivity through IdentRef.
- **Transient.** Consumed by the step's upgrade sub-phase; never stored in structure; never held across step boundaries (WIT-3).

### 4.4 Scoping discipline

Witnesses represent **past validity under a specific guard**, not ongoing guarantees. The following disciplines apply:

- Witnesses do not appear in struct fields, static bindings, or collections.
- Witnesses do not cross `.await` boundaries.
- Witnesses do not cross thread boundaries.
- The step's upgrade sub-phase (STEP-4 phase 2) promotes witness IdentRefs to Caps or operational evidence via `substrate::pin`. After upgrade, evidence is `'static`; the guard may still be held for the remaining sub-phases (reserve, commit, publish), all synchronous per STEP-2.

Resumable traversal tokens (see `VFS_CHECKS.md` §5.2 for the walker's ResumeToken) are **informational only with respect to retention**: they carry Cap handles across yields purely to ensure the cursor's slot is not reclaimed during the yield, not because any binding has been committed. On resume, the Caps are downgraded to IdentRef under a fresh guard, and the cursor's semantic validity is re-predicated from scratch. Resumable tokens are not witnesses and do not carry commitment.

### 4.5 Controlled redundancy

A predicate may be invoked multiple times per operation:

- Once by require in checks (authorization, witness production).
- Again during a trampolined check after an I/O yield (re-verification after cache fill).
- During substrate reservation where the reservation invariant overlaps the semantic predicate.
- During multi-step operations that re-require at each step's observe phase (STEP-7 fresh observation).

This is **temporal redundancy under monotonicity**, not duplication. Each invocation uses the same predicate (defined once in `predicates.rs`). What is forbidden is reimplementing the semantic condition inline at call sites. Reading `dentry.removed` directly outside a predicate is a lint violation.

### 4.6 Policy check vs reservation

- **Policy checks** are predicates in `checks/`: "would this operation be permitted in principle given current policy?" (RLIMIT_NOFILE permits this process to create more fds; the predicate reads counter and limit without modifying.)
- **Reservation** is substrate work in a step's reserve sub-phase (STEP-4 phase 3): the CAS that decrements budget and yields a refundable reservation.

Both are needed. A policy check can pass while reservation fails (concurrent thread consumed the last slot → EMFILE from reservation). The TOCTOU gap between policy permission and actual availability is architecturally explicit: `checks/` answers permission; `substrate::credit::reserve` provides the consumption linearization.

---

## 5. Enforcement Layer

### 5.1 Module visibility

Subsystem-internal fields are accessible outside the subsystem only via require functions:

```rust
pub use self::checks::{require_namespace_child, NamespaceChild};
// DEntry.children, DEntry.removed stay pub(crate) within vfs/
```

Rust's module system enforces most of this. `checks/` re-exports require functions and witness types; underlying fields stay `pub(crate)` or private.

### 5.2 Lints

1. **No raw predicate reimplementation.** Code reading subsystem-internal flags (`dentry.removed`, `process.state`, `mount.detached`) outside a predicate function is flagged.
2. **Witnesses are stack-only.** Types bearing `'g` (lifetime-derived from IdentRef) cannot appear in struct fields, static bindings, or collections. Rust's borrow checker largely enforces this automatically; lints add backup for witness types carrying owned data alongside IdentRef.
3. **Require at entry points.** Functions marked as consuming projection X must call the corresponding `require_*` at entry before any operation depending on X.
4. **Module boundaries.** Full rules in `SUBSYSTEM_ANATOMY.md` §7. Checks-specific: may not import substrate, may not call bus publication, may not write structure.

---

## 6. Reclamation

Reclamation is a specific ordered sequence, per object_model §6.2:

1. **Binding removal.** Substrate linearization point — `substrate::index::withdraw_commit` or `substrate::index::swap_commit`. The binding ceases to be present in its container atomically from observers' perspective.
2. **Co-located evidence drop.** Rust Drop on the removed binding's contents decrements whatever retention contributions it carried (LinkPin, OpenPin, MapPin, identity Caps, ...).
3. **Retention transition.** If the decrement takes the entity's payload disjuncts to zero AND identity retention to zero, SENTINEL_DEAD CAS is attempted on the retention counter. Success commits reclamation intent.
4. **Epoch-deferred reclaim.** Slot is enqueued for destructor drain (bounded-work per object_model §6.3); once epoch-quiesced, slot is returned to its zone.

Reclamation requires **all** of:

- No binding presence where the operation would reach the entity (namespace projections false, or pinning bindings withdrawn).
- No identity pins (Cap refcount zero).
- No payload-retention contributions (all disjunct counters zero).
- Epoch quiescence (all guards that may have observed pre-reclamation state have released).

Physical reclamation does not happen while any of these remain. An entity with an extant `Cap<T>` or still-present addressability / operational binding is not reclaimed, regardless of other bindings' state. The ordering is non-negotiable: binding removal must precede evidence drop must precede retention transition must precede epoch-deferred reclaim.

---

## 7. Predictions

The framework generates design answers without explicit per-case derivation.

**Prediction 1: Zombie processes have two addressability projections.** Before examining POSIX, the framework predicts `waitpid` and `kill` target different projections: they have different transitions. POSIX confirms: zombies are reapable (wait_addr true) but not signal-receivable (signal_addr false). Falls out; need not be specified.

**Prediction 2: Unmount needs a detached-equivalent flag or container transition.** Analogous to `removed` for DEntry. An unmounted-but-still-held mount is analogous to an unlinked-but-open file.

**Prediction 3: The step model's no-rollback property.** Under monotonicity, failures before a step reaches phase 4 (commit) require no compensating action because no visibility boundary has been crossed. Monotone decrease means preconditions weren't met during observe or upgrade; no external state was modified, and linear reservations drop cleanly if phase 3 runs partially.

---

## 8. Application Checklist

When adding a new kernel entity type:

1. **Enumerate projections** (2–4 typical): structural; one or two namespace/addressability; one payload. Name per domain vocabulary.

2. **State each projection's mechanism** per §2.4 — identity retention, binding-chain reachability, or payload retention. This is an architectural requirement.

3. **Identify the partial order** (per-subsystem theorem). Draw the Hasse diagram. Each implication must be an honest derivation from the binding/retention structure, not a generic property.

4. **Identify viewpoint dependencies.** Does the projection depend on context (mnt_ns, pid_ns, fd_table)?

5. **Identify pinning contributions** for payload. Which sources can keep payload alive? Define typed contributions implementing `OperationalContribution<T>` per ADR §7 if payload is compound.

6. **Write predicates.** Pure functions over `IdentRef<'g, T>` + context. One per projection. Atomic reads for concurrent flags.

7. **Write require functions.** One per operation entry point. Errno per POSIX. Witness carries IdentRef to the located entity.

8. **Define witness types.** Lifetime-parameterized by `'g`. Private constructors. Contain only what the step's upgrade sub-phase needs.

9. **Enumerate reclamation conditions.** All projections false; all retention zero; epoch quiescence. Note any background reaper vs opportunistic-on-drop.

10. **Verify race-degradation theorem for this entity.** Walk through (checks, concurrent mutation, step's upgrade). Confirm either success against correct target, or clean failure.

11. **Verify the observer contract applies.** If the entity participates in binding-preservation relationships (as walker targets, resolver cursors, reader targets), confirm that binding-preserving operations cannot silently retarget in-flight observations, and binding-mutating operations go through observer-safe substrate primitives.

12. **Document in §3 projection catalog.**

---

## 9. Open Questions

- **Pager / Filesystem trampoline interaction.** Filesystem's `FsStep` is a sibling of `ResolutionStep`. Whether FsResumeTokens carry retention evidence or are purely informational remains to be confirmed (likely purely informational — filesystems operate on physical blocks, not namespace entities).

- **Predicate-level async.** Most predicates are pure synchronous reads. Some (FUSE-backed, network-filesystem permission checks) might yield for remote validation. The trampolined check pattern handles this; interaction with predicate purity deserves a closed-form spec.

- **VM payload predicate shape.** VmEntry.payload is "backing accessible"; the exact predicate and its interaction with the (not-yet-specified) VM coordination primitive is deferred to the VM subsystem spec.

---

## References

- [`object_model_0417.md`](./object_model_0417.md) — authoritative for entities, references, projections, bindings.
- [`INVARIANTS.md`](./INVARIANTS.md) — PRED-* realize the framework stated here.
- [`STEP_MODEL.md`](./STEP_MODEL.md) — witness consumption at the step's upgrade sub-phase.
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) — four-module layout, five-phase step discipline, substrate §4.
- v11 §6 (DEntry/RNode/Entity layering), §7 (Resolution Registry), §16 (Cancellation Safety).
