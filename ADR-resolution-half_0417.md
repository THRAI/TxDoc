# ADR: Bindings, Obligations, and Reachability — Completing the Kernel Model

**Status.** v2 (2026-04-17). Extends v1 with the terminology corrections in the audit checklist and the architecture/implementation bifurcation. v1 structural claims preserved.

**Extends.** `tx-kernel-architecture-v11.md`, `LIVENESS.md`, `SUBSYSTEM_ANATOMY.md`, `object_model_0417.md`.

**Refines v1.** Specific corrections in v2:
- "Queue pins FutexKey components" → "Queue retains the identity anchors required to keep the FutexKey stable" (§ futex tenant).
- "use Cap-identity keys, not reused integers" → "use stable identity keys for cross-subsystem registrations whenever the contract requires referential stability; do not use reusable namespace handles for internal registrations" (§ tenants).
- "our substrate-primitive approach fixes mmap UAF at the architectural level" → "the architecture identifies the need for an observer-safe primitive coordinating recipe withdrawal, PTE install, and TLB invalidation; this primitive remains to be specified precisely" (§ VM tenant).

---

## Part I — Foundation

### 1. Context

The v11 architecture rests on two basis claims:

- *A kernel maintains `(signifier, consistency, binding)` triples.*
- *Every kernel entity decomposes into `(identity, capability, payload)` layers.*

The second claim has been mechanized thoroughly by `LIVENESS.md` (projection framework with monotone transitions) and `SUBSYSTEM_ANATOMY.md` (four-module layout, three-phase commit). This body of work is the **lifecycle half**: what entities are, how they persist, how they change state.

The first claim was under-mechanized. This ADR develops the **resolution half**: how userspace signifiers reach entities, through what structures, with what guarantees. It formalizes three primitives parallel to object/liveness/commit:

- **Binding** — the structural element that maps signifiers to entities.
- **Obligation** — the typed declaration of what a binding promises.
- **Reachability** — the epoch-guarded traversal mechanism that composes bindings.

The two halves compose through projections (obligations require projections to hold) and through commit (bindings are established and removed at commit points). Together they describe the full namespace-to-payload chain.

### 2. The gap v11 leaves

Three symptoms motivated this work:

**Conflation of observation with retention.** v11's `Cap<T>` was used for both "I am in a binding container pointing at this entity" and "I am holding this entity to perform work." These are different. The first is structural presence; the second is retention. Object_model §4 separates them via the four-tier reference hierarchy (Weak / IdentRef / Cap / OperationalEvidence).

**No vocabulary for separable liveness.** Zombie-alive-for-waitpid-but-dead-for-kill required ad-hoc flags rather than typed obligations.

**Hot-path retention cost.** v11 traversal incurred refcount traffic per hop. Epoch-guarded observation (object_model §2) enables refcount-free traversal with upgrade-at-terminal.

This ADR introduces resolution-half primitives and specifies their interaction with the lifecycle half.

---

## Part II — The Resolution Half

### 3. Bindings

A **binding** is a structural entry in a container:

```
Binding<K, Obligation> {
    key:      K,                       // signifier
    evidence: Evidence<Obligation>,    // retention evidence, type derived from obligation
}
```

A binding:

- **Exists iff present in its container.** No independent lifetime; no refcount of its own. The container (B-tree, hash, radix, DLL) determines the binding's lifecycle.
- **Carries a key** locating it in the container's signifier space.
- **Carries evidence** — the retention mechanism realizing the binding's declared obligation. Evidence type is derived from obligation + target entity type (§6).

Bindings are not references in the refcount sense. They do not participate in retention management by themselves. What they *do* is **carry retention evidence as structural contents**: the evidence field stores a value whose lifetime is scoped to the binding's presence in its container. When the binding is removed, the evidence drops, which (if the evidence is a retention handle) decrements the appropriate count.

A **namespace** is a container of bindings.

### 4. Obligations

Every binding declares an **obligation** toward its target. The obligation is part of the binding's type — mandatory, not inferred.

**Resolution-only.** The binding exists to answer "what's at this key." Stale target is acceptable; resolvers handle the failure. Evidence: `Weak<T>` or `()`.

**Addressability.** The binding promises identity stability: the target remains structurally alive, and resolving through the binding returns *this* entity (not a slot-reuse). The target may be payload-degraded. Evidence: `Cap<T>`.

**Operational.** The binding promises full usability: payload projection holds. Evidence: entity-specific operational retention (§7).

Per LIVENESS §2.4 obligation ↔ projection correspondence:

| Obligation       | Required projection | Evidence realizes via        |
|------------------|---------------------|------------------------------|
| Resolution-only  | (none — no guarantee)         | No pin. Fallible upgrade.    |
| Addressability   | structural                    | `Cap<T>`                     |
| Operational      | payload                       | Entity-specific (§7)         |

The obligation is the binding-author's declaration; the retention mechanism is derived from the entity's payload organization.

### 5. Reference hierarchy

Four reference types, per object_model §4:

```
        Weak<T>
        (nullable; epoch-independent; fallible upgrade)
            ↓ (observe under epoch)
    IdentRef<'g, T>
        (epoch-guarded; stack-bound; memory-safe observation)
            ↓ (pin under epoch)
         Cap<T>
        (refcounted; pins identity; arbitrary lifetime)
            ↓ (upgrade payload)
   T::OperationalEvidence
        (entity-specific; pins payload; implies Cap<T>)
```

Each downgrade is free; each upgrade is fallible.

### 6. Evidence derivation

The evidence type for a binding is derived from its obligation and target entity type:

```
derive(ResolutionOnly,  _ )  =  Weak<T>   |  ()
derive(Addressability,  T )  =  Cap<T>
derive(Operational,     T )  =  T::OperationalEvidence
```

Each entity type declares its operational evidence:

```rust
trait Entity {
    type OperationalEvidence;
}
```

Examples:

```rust
impl Entity for DEntry    { type OperationalEvidence = Cap<DEntry>; }
impl Entity for Mount     { type OperationalEvidence = Cap<Mount>;  }
impl Entity for OpenFile  { type OperationalEvidence = Cap<OpenFile>; }
impl Entity for Inode     { type OperationalEvidence = PayloadCap<Inode>; }
impl Entity for Frame     { type OperationalEvidence = MapPin; }
```

Binding designers choose obligations; the compiler derives evidence types. Changes to an entity's payload organization change the entity's `OperationalEvidence` type and propagate through derivation; binding declarations targeting that entity continue to compile with the same obligation.

### 7. Operational contributions (domain-specific pins)

Some entities have payload alive under a disjunction of independent sources. An Inode's payload is alive iff `nlinks > 0 ∨ open_refs > 0`. A Frame's payload is alive iff `map_count > 0 ∨ cache_ref > 0`. Each disjunct is an independently counted source.

For such entities, operational evidence is not a single type but any member of a family:

```rust
trait OperationalContribution<T: Entity> {
    // Typed pin contributing to one disjunct of T's payload projection.
    // Construction: epoch-guarded CAS with projection pre-check.
    // Drop: decrement the specific counter.
}

struct LinkPin { inode: Cap<Inode> }   // contributes to Inode.nlinks
struct OpenPin { inode: Cap<Inode> }   // contributes to Inode.open_refs

impl OperationalContribution<Inode> for LinkPin { /* ... */ }
impl OperationalContribution<Inode> for OpenPin { /* ... */ }

struct MapPin   { frame: Weak<Frame> }   // contributes to Frame.map_count
struct CachePin { frame: Weak<Frame> }   // contributes to Frame.cache_ref
```

Each contribution's existence implies the target entity's payload projection holds. Drop of all contributions (all disjuncts zero) lowers payload projection to false and triggers reclamation through the SENTINEL_DEAD mechanism.

Binding sites choose the appropriate contribution by semantic role:
- Filesystem directory entry → `LinkPin`.
- Per-open struct → `OpenPin`.
- PTE → `MapPin`.
- Page-cache radix entry → `CachePin`.

The obligation is "operational"; the specific contribution documents which operational role.

Entities with single-source payload (DEntry — only itself holds it; OpenFile — only FdEntries and epoll pin it) collapse to a single operational evidence type.

### 8. Reachability

Reachability is the mechanism by which resolvers traverse binding chains to reach entities. Three properties:

**Epoch-guarded.** Traversal runs under an epoch guard; the guard defers physical reclamation of any slot observed during the traversal. A resolver walking a container while another thread removes a binding sees either the binding or doesn't, but never sees freed memory.

**Composable.** A path resolution is a sequence of hops, each a binding lookup. Intermediate `IdentRef`s are held; the epoch guard persists across the entire composition within one step.

**Obligation-terminal.** Traversal ends when the resolver upgrades its final IdentRef to the retention level needed. Syscalls wanting to operate upgrade to operational evidence; registrations wanting only addressability upgrade to `Cap<T>`; cache populators may stay at IdentRef or downgrade to `Weak`.

Reachability itself contributes no retention. Walking from root to a 10-deep dentry touches 10 bindings, produces 10 IdentRefs, none bump any refcount. The retention claim is made only at the final upgrade, and only by the resolver that needs it.

This is the refcount-free hot path: *n* components become *n* epoch-guarded dereferences plus one terminal upgrade.

**Informational I/O.** Reachability may encounter cold caches. The resolver records a trampoline state, yields to the reactor, resumes under a fresh epoch guard once the cache is populated. Monotonicity of projections ensures re-predicated state on resume is consistent with pre-yield observations or cleanly fails.

Informational I/O is **interpreter-internal cache/state fill only; it does not publish bindings nor mutate semantic state**. Distinct from operational I/O (§ Part IV).

### 9. Reclamation ordering

Per LIVENESS §6 and object_model §6.2:

1. **Binding removal** — substrate linearization point (`index::withdraw_commit` or `swap_commit`).
2. **Evidence drop** — Rust Drop on the binding entry decrements retention contributions.
3. **Retention transition** — if retention falls to zero, SENTINEL_DEAD CAS commits reclamation intent.
4. **Enqueue epoch-deferred reclaim** — slot queued for destructor drain, physical free on epoch quiescence.

Reclamation requires **all** of:
- No binding presence where required (namespace projections false, pinning bindings withdrawn).
- No identity pins (Cap refcount zero).
- No payload-retention contributions (all disjuncts zero).
- Epoch quiescence reached.

No entity is physically reclaimed while an addressability or operational binding exists.

---

## Part III — Composition with the Lifecycle Half

### 10. Obligation-to-projection correspondence

Restated from §4 as a theorem:

| Obligation       | Projection required to hold    | Semantic reading                         |
|------------------|--------------------------------|------------------------------------------|
| Resolution-only  | (none; may be empty)           | "Look but do not commit."                |
| Addressability   | structural                     | "This is the same entity we named."      |
| Operational      | payload                        | "Operations against this will succeed."  |

Other projections (namespace, wait_addr, signal_addr, etc.) are **derived from binding presence across namespaces**. An entity's namespace projection is true iff there exists a chain of bindings from the relevant namespace root. This is a reachability query computed over container structure, not a flag read from the entity.

### 11. Full chain

```
Namespace (container of bindings)
    ↓
Binding (structural entry; key + evidence; declared obligation)
    ↓
Entity (identity + capability + payload)
    ↓
Payload
    ↓
Capability (vtable or dispatch)
    ↓
Effect (bytes moved, signal delivered, TLB flushed)
```

### 12. Integration with three-phase commit

**Checks.** Traversal under epoch guards produces IdentRef-witnesses. Require functions validate projections by reading entity state (through IdentRef) and binding structure (via container lookups under the same guard). Witnesses carry IdentRef, not Cap — the checks phase makes no retention claims. This is what enables `checks/` to remain pure even when operations create bindings: the witness proves past validation, not retention.

**Prepare.** Upgrades IdentRefs to the retention level the operation demands via `substrate::pin::upgrade_to_cap` or `substrate::pin::reserve_contribution`. A `*_prepare` for `open(2)` upgrades its witness's `IdentRef<Inode>` to `OpenPin` (acquiring operational evidence) via a fallible CAS. Reserves substrate resources (zone, index, credit) per SUBSYSTEM_ANATOMY §4. Upgraded evidence lives in the reservation bundle.

**Commit.** Publishes new bindings by inserting the prepared evidence into the appropriate namespaces. Uses substrate primitives:
- New binding: `substrate::index::commit`.
- Removal: `substrate::index::withdraw_commit` (mutation substrate, §4.5 of ANATOMY).
- Replacement: `substrate::index::swap_commit`.
- Conditional: `substrate::index::install_if_match` (e.g., futex register-if-check).

Both are committed atomically from external observers' perspective: before commit, no namespace contains the new bindings; after commit, all are present with evidence.

### 13. Two trace examples

**Read from an unlinked file.** User has `fd` open on an inode that was unlinked after opening.

- *Checks.* Resolve fd under epoch guard: binding present (operational obligation), evidence is `Cap<OpenFile>`. Produce `IdentRef<'g, OpenFile>`. OpenFile holds `OpenPin` referencing the unlinked Inode. Inode's namespace projection is false; payload projection is true (OpenPin keeps open_refs > 0). Witness produced; operation authorized.
- *Prepare.* Upgrade IdentRef<OpenFile> to Cap<OpenFile>. Resolve user pages. No new substrate reservations for read-from-memory.
- *Commit.* Copy from inode's page cache to resolved user pages. Update atime. Return bytes-read count.

Unlinked-but-open is no special case: the inode's nlinks disjunct is zero but open_refs disjunct is nonzero; operational projection holds; read proceeds.

**Reap zombie.** Parent calls `waitpid(child_pid, &status, 0)`. Child has exited earlier; parent hasn't yet reaped.

*Before waitpid (state at start of trace).* At `exit_commit` earlier, the child's `ProcessPayload` was dropped: `ProcessIdentity.payload` transitioned from `Some(PayloadCap<ProcessPayload>)` to `None`. The PayloadCap's Drop decremented ProcessPayload's retention; SENTINEL_DEAD fired on the ProcessPayload slot; the ProcessPayload zone slot reclaimed. Exit status was written into `ProcessIdentity.exit_status` before the payload drop (exit_status lives in identity precisely so reap can read it without needing payload). ProcessIdentity persists because parent's children container holds a binding with `Cap<ProcessIdentity>` evidence.

- *Checks.* Resolve child_pid under epoch guard via parent's children binding. Binding obligation: **addressability**; evidence: `Cap<ProcessIdentity>`. Produce `IdentRef<'g, ProcessIdentity>`. Predicates: ProcessIdentity.structural — live (binding evidence extant). ProcessIdentity.wait_addr — true (binding present in parent.children container). ProcessIdentity.payload.is_some() — false (payload dropped at exit). Witness validates wait-semantics and captures an IdentRef to read exit_status from.
- *Prepare.* Upgrade `IdentRef<'g, ProcessIdentity>` to `Cap<ProcessIdentity>` via `substrate::pin::upgrade_to_cap`. Read `ProcessIdentity.exit_status` through the Cap — no payload access required; this is what the identity/payload split buys.
- *Commit.* Withdraw child binding from parent's children container via `substrate::index::withdraw_commit`. Withdraw from pid hash via `withdraw_commit`. Both binding evidences drop. `Cap<ProcessIdentity>` retention decrements. If this was the last retention on ProcessIdentity, SENTINEL_DEAD CAS fires on its slot; reclamation enqueued. Return exit_status to parent.

The zombie state is not modeled by Option-wrapped fields on a god-struct. It is **two zone slots with independent lifetimes**: ProcessPayload reclaimed early (at exit_commit); ProcessIdentity reclaimed late (at reap_commit). Two SENTINEL_DEAD transitions; two reclamation events. The framework's `ProcessIdentity.wait_addr ⟂ ProcessPayload.payload` partial order is a literal statement about which slots exist. waitpid and kill differ cleanly without special flags: `kill(pid, sig)` requires `signal_addr` which depends on `ProcessIdentity.payload.is_some()` (false for zombies, so ESRCH); `waitpid(pid, ...)` requires `wait_addr` which depends only on the children-binding being present (true for zombies).

---

## Part IV — The Architectural Picture

### 14. The two halves and the reactor

```
                ┌────────────────────┐
                │   Resolution half  │
                │  (binding, reach)  │
                └─────────┬──────────┘
                          │
                          │  informational I/O (cache/state fill only)
                          ▼
                ┌────────────────────┐
                │                    │
                │      Reactor       │
                │ (temporal core)    │
                │                    │
                └─────────┬──────────┘
                          ▲
                          │  operational I/O (realization of committed semantics)
                          │
                ┌─────────┴──────────┐
                │   Lifecycle half   │
                │  (entity, commit)  │
                └─────────┬──────────┘
                          │
                          ▼
                         HAL
                (hardware-specific primitives,
                 accessed only through reactor
                 temporal coordination)
```

**Semantic lives in the two halves. Time lives in the reactor.**

The halves are *semantic*. They reason about state, invariants, promises. Checks, prepare, commit, projection predicates, obligation assertions — these do not depend on *when* things happen, only on *what* and *in what order* relative to other semantic events.

The reactor is *temporal*. It mediates asynchrony, submits I/O to hardware, delivers completions, drives timers, coordinates cross-core invalidation. It does not know what a binding is, what an obligation declares, or what a projection witnesses. It polls futures and runs continuations.

### 15. Informational vs operational vs observational I/O

**Informational I/O — from the resolution half.** Triggered when traversal encounters cold caches. Resolver records trampoline state, dispatches coroutine to reactor (submits block read or metadata load), yields. On completion, the coroutine populates the cache and signals the resolver to resume under a fresh epoch guard.

Informational I/O **has no binding-visible side effects**. The cache is populated (interpreter-internal state change); no new bindings are published; no entities transition semantically. Checks phase purity is preserved: checks is referentially transparent, not synchronous. It may yield for informational coroutines; it produces the same witness on resumption that it would have produced had caches been warm.

**Operational I/O — from the lifecycle half.** Triggered in commit when semantic state is already committed and the physical effect must be realized. A write's commit synchronously updates the page cache and mtime, then dispatches a writeback coroutine for the disk transfer. A close's commit may schedule orphan cleanup. Operational I/O is the physical realization of committed semantics.

**Observational effects — also from the lifecycle half.** Tracepoints (per-CPU ring writes), ptrace boundary events, wait-queue wakes. Structurally resemble operational I/O but carry notifications, not payload I/O.

### 16. Coroutine as boundary object

A coroutine is the unit of exchange between semantic half and reactor:

- A half **authors** the coroutine: async function instance carrying the work.
- The reactor **runs** the coroutine: polls it, handles `.await` by parking on completion events, resumes on event fire.
- The coroutine **returns** a result to its author, synchronously or via the author's completion handler.

Coroutines carry semantic content (binding/entity vocabulary) with temporal placeholders (`.await` points). Neither half reaches reactor scheduling internals directly.

### 17. Reactor responsibilities, separated

Per the checklist, reactor APIs divide into at least two semantic classes:

**Deferred async execution.** Submit coroutine; yield; resume on completion. The standard reactor pattern for read, write, sleep, timer-based operations.

**Synchronous coordination.** Cross-core invalidation (TLB shootdown), IPI-based barriers, CPU affinity adjustments. These are not "async" in the yield-and-wait sense; they are synchronous-from-issuing-thread but cross-core. The reactor exposes these as distinct APIs because their semantics differ (the issuing thread blocks until all targets ack).

Hardware interaction occurs through reactor temporal services. No semantic half directly manipulates hardware state (PTEs, page tables, device registers) except through reactor-provided coordination primitives.

### 18. Thread futures

Userspace threads are reactor tenants. A thread is factored into `ThreadIdentity` (zone-allocated identity slot, with projections and pins) and `ThreadPayload` (computation state: trap frame, signal mask, pending signals). Scheduling is the reactor's concern.

When a thread is runnable, its `ThreadIdentity.payload` is Some, pointing at a ThreadPayload slot that carries the saved trap frame and scheduling-relevant state. When a thread exits, ThreadPayload drops (SENTINEL_DEAD on the payload slot); ThreadIdentity persists until it is detached from its owning `ProcessIdentity`'s thread chain and the last Cap releases. Cross-boundary interaction is disciplined through published evidence (reactor holds `Cap<ThreadIdentity>` while scheduling and a `PayloadCap<ThreadPayload>` to access the trap frame) and handoff points (fork publishes new identity + payload; exit drops the payload and detaches identity from the ready set).

### 19. Dependency implications

```
HAL
 ↑
exec::reactor  ← temporal core; depends on HAL, foundation
 ↑                                          ↑
 │                                          │
 │  (both halves depend on reactor)         │
 │                                          │
registry/*  ← resolution half          execution/*  ← lifecycle half
 ↑                                          ↑
 │                                          │
 └──────────── depend on tx-fnd/ ────────────┘
              (epoch, pin, binding substrate)
```

The reactor is *peer* to both halves, not a sub-layer of either. Both halves depend on reactor APIs; neither depends on the other (data flows from registry's outputs to execution's inputs, but this is not module dependency). HAL is reactor's dependency; no semantic code reaches HAL directly.

---

## Part V — Tenants

Subsystems instantiate the framework over their own domain. A full catalog is deferred; this ADR names the tenants at a high level with the v2 corrections applied.

Entities subject to the identity/payload split per the factoring convention ([`object_model.md`](./object_model_0417.md) §8.1.1) — Process, Thread, Mount, Socket — appear below as two-type tenants with the canonical `<n>Identity` / `<n>Payload` factoring. Co-located entities (DEntry, FdEntry, OpenFile) stay as single types.

### VFS

The largest and most instructive tenant. Namespaces: directory children containers (one per DEntry), fd tables (one per process, shareable via CLONE_FILES), mount tree, name cache. Bindings: path components (addressability or operational within directory containers), fd slots (operational on OpenFiles), mount points (operational on Mounts), name cache entries (resolution-only, with Weak evidence). Exhibits all three obligation types and both co-located and indirected payload. Connects to reactor for informational I/O (cache fills) and via capabilities (dispatch vectors) for operational I/O at commit.

### Process/signal

Entities are split: `ProcessIdentity` (pid, parent, children DLL, siblings, exit_status, events, observe, payload: Option<PayloadCap<ProcessPayload>>) and `ProcessPayload` (frame — typed environment with vm/fd_table/sig_actions/fs_context; policy — cred, signal_mask, rlimits). Two zones with independent reclamation.

Similarly, `ThreadIdentity` (tid, owning_identity, next_thread, events, observe, payload) and `ThreadPayload` (saved trap frame, signal mask, pending signals, stop state, tls_base, robust_list_head).

Namespaces: pid hashes (one per PidNamespace), children containers (one per ProcessIdentity), process group member lists, thread chains rooted at ProcessIdentity.first_thread. Bindings: all addressability — pids must stably name the ProcessIdentity across zombie states; children entries in parent's container must survive child exit; pgrp memberships must survive transient emptiness. All these bindings carry `Cap<ProcessIdentity>` or `Cap<ThreadIdentity>` as evidence, never Cap to a payload type.

Exemplar of the identity/payload split. At `exit_commit`: drop `ProcessIdentity.payload` (dropping the PayloadCap releases retention on ProcessPayload; SENTINEL_DEAD on ProcessPayload slot); write exit_status into ProcessIdentity; ProcessIdentity persists via children-binding Cap. At `reap_commit`: withdraw children binding (`substrate::index::withdraw_commit`); withdraw pid-hash binding; Cap<ProcessIdentity> refcount drops; SENTINEL_DEAD on ProcessIdentity slot. Two zones, two reclamations, no god-struct.

### Mount

Entities are split: `MountIdentity` (mnt_id, mountpoint Cap<DEntry>, parent_mount Option<Cap<MountIdentity>>, child mounts DLL, flags, payload: Option<PayloadCap<MountPayload>>) and `MountPayload` (superblock, backing device, filesystem-specific state, root_rnode Cap<RNode>).

Namespace: mount tree per MountNamespace. Bindings: mount-point entries (operational evidence to MountIdentity), parent-mount references in child MountIdentities (addressability).

The split enables **force-umount / MNT_DETACH semantics**. Normal umount drops both at once: withdraw mount-tree binding, drop MountIdentity.payload (SENTINEL_DEAD on MountPayload slot, superblock flushed and backing released), last Cap on MountIdentity drops, SENTINEL_DEAD on MountIdentity. Force-umount drops MountIdentity.payload eagerly while MountIdentity persists for any resolver still holding a Cap; operations on such a stale mount return ESTALE / ENOTCONN-equivalent. When the last Cap drops, MountIdentity reclaims.

### Socket

Entities are split: `SocketIdentity` (addressability handle, events, observe, payload: Option<PayloadCap<SocketPayload>>) and `SocketPayload` (protocol FSM state, send/recv buffers, smoltcp handle or equivalent).

Namespace: protocol tables keyed by socket tuple or listen-queue reference. Bindings: protocol-table entries are addressability evidence to SocketIdentity; fd-to-socket bindings in the fd subsystem are operational.

The split supports **clean close with lingering fds**: on FIN-ACK drain complete, SocketPayload drops (large buffers released); SocketIdentity persists for any fd still open; subsequent operations through such fds return ENOTCONN/EPIPE. Parallels unlinked-but-open files, but here the payload drops earlier than the identity because protocol-FSM completion is distinct from handle release.

### VM

**The architecture identifies the need for an observer-safe primitive coordinating recipe withdrawal, PTE install/teardown, and TLB invalidation; this primitive remains to be specified precisely.** See SUBSYSTEM_ANATOMY §4.5.1.

Namespace: recipes container per AddressSpace. Bindings: VA range → VmEntry (operational — fault handling requires the VmEntry's fields). VmEntries hold operational contributions (MapPin or equivalent) on their backing inodes. The fault handler is a functor that reads VmEntries and installs PTEs; PTEs are themselves bindings (VA → Frame, operational via MapPin contribution).

Architectural requirement: from any concurrent thread's perspective, either the binding (VA range → VmEntry) holds and PTE-cached access is valid, or the binding is withdrawn and access faults cleanly to SIGSEGV. No silent use-after-free is observable. The primitive realizing this requirement is an open design concern.

### Futex

Smaller tenant. Namespace: futex table, keyed by FutexKey. Bindings: FutexKey → Queue (the wait queue).

**The Queue retains the identity anchors required to keep the FutexKey stable.** For MAP_SHARED, anchors include the inode reference (kept via appropriate evidence on the inode). For MAP_PRIVATE, the AddressSpace reference. No silent retarget of a FutexKey's meaning across its queue's lifetime.

Waiter registration uses the `install_if_match` conditional-commit primitive: atomically "check memory value + insert waiter into queue" as a single commit. Lost wakeups are prevented by this atomicity, not by two-step register-then-verify logic.

### Epoll / signalfd / eventfd

Registrations keyed by stable identity (the target per-open-struct's Cap or equivalent), not by reusable namespace handles (fd number). Per the contract: **use stable identity keys for cross-subsystem registrations whenever referential stability is required; do not use reusable namespace handles for internal registrations.**

The fd-number appears only at the userspace ABI boundary; kernel-internal registration is by stable identity. This prevents fd-reuse retargeting bugs that afflict integer-keyed registrations.

### Observation (ptrace, tracepoints)

Cross-cutting. Ptrace ObserveSlots are addressability bindings on observed processes (identity anchor required; the binding must name the same ProcessIdentity across its lifetime — including through the observed process's zombie window, since a tracer may collect wait status after tracee's payload has dropped). Tracepoint subscriptions are reactor-side future registrations for events fired by commit phases; not bindings in the structural sense.

---

## Part VI — Architecture vs Implementation

The architecture layer specifies:

- The semantic object model (entity / identity / payload).
- Binding as structural namespace entry.
- Obligation as typed promise.
- Reachability as epoch-guarded traversal.
- Projections (namespace, structural, payload) and their mechanisms.
- Monotonicity and the race-degradation theorem.
- The observer contract (LIVENESS §2.6).
- Publication and mutation substrate contracts (ANATOMY §4).
- Reactor responsibilities.

The implementation layer chooses:

- Specific container data structures (persistent radix vs B-tree vs hash).
- Specific epoch scheme (global vs per-CPU).
- Specific refcount layout (packed u32 vs separate words).
- Specific stable identity key representation (Cap vs generation-tagged slot id).
- Binding entry struct layouts.
- Reactor scheduling policy.
- Cache invalidation strategy.
- Concrete observer-safe primitive realizations.
- Shootdown protocol details.
- Coroutine runtime internals.

Implementation is free as long as architectural contracts are preserved.

### Substrate modules the implementation layer must provide

**`tx-fnd/epoch`.** Epoch-based reclamation. Guard, IdentRef, AtomicSlot, defer_zone_reclaim. Kernel-specific specialization of classical EBR (no-std, zone-slot reclaim, per-reactor-task registration).

**`tx-fnd/pin`.** Refcounted pin primitives. Cap<T>, PayloadCap<T>, Weak<T>, OperationalContribution trait, SENTINEL_DEAD CAS, reclaim queue integration.

**`tx-fnd/binding`.** Binding containers with obligation typing. `Binding<K, Obligation>`, obligation-parameterized insertion helpers, the Entity trait with associated OperationalEvidence.

**`substrate/mutation`** (extension of binding substrate). `index::withdraw_commit`, `index::swap_commit`, `index::install_if_match`. Observer-safe.

Migration of v11 `Cap<T>` sites proceeds after substrate is in place. Each site audits against the binding catalog: structural reference within a namespace (binding evidence), free-standing holder (OpenFile's internal inode reference), or traversal handle (becomes IdentRef). Mechanical audit, not architectural change.

---

## Part VII — Open Questions and Status

**VM coordination primitive.** As stated: the observer-safe primitive coordinating recipe withdrawal, PTE install/teardown, and TLB invalidation remains to be specified. Its precise shape depends on pmap substrate design. Candidate approaches: atomic bundle; sequence with explicit barriers; install_pte validating recipe-binding currency at commit. Architectural requirement: no silent UAF visible to concurrent threads.

**Conditional-commit generalization.** `install_if_match` handles futex. Whether signalfd, eventfd, or other register-if-condition semantics need related primitives warrants consolidation. Candidate for a named substrate abstraction.

**`Shared<T>` specialization.** v11's `Shared<T>` for Frame slots (vm, fd_table, sig_actions, fs_context) with COW-on-mutate. Under the framework, likely a specialization of `PayloadCap<T>` for COW payload. Placement deferred.

**Back-references for rmap.** Future rmap (Frame → mapping VmEntries) needs non-pinning back-pointers. `Weak<VmEntry>` is the candidate. Detailed design deferred until VM requires it.

**Name-cache staleness discipline.** Resolution-only bindings with `Weak<DEntry>` become stale when targets reclaim; lookup through stale entries fails. Eviction policy (LRU vs pressure-driven) and staleness detection (lazy vs eager invalidation) are implementation choices; both are framework-compatible.

**Thread-reactor handoff.** ThreadIdentity and ThreadPayload are lifecycle entities whose scheduling is reactor-managed. Fork publishes both; exit drops ThreadPayload and eventually detaches ThreadIdentity. Synchronization between commit-phase state changes and reactor scheduling is where semantic/temporal separation is most tested. Specification deferred to a companion process-subsystem anatomy document.

**Status.** This ADR is *proposed*. Structural claims preserved from v1; terminology and phrasing corrections applied per the audit checklist. Implementation follows substrate (tx-fnd/epoch, tx-fnd/pin, tx-fnd/binding, substrate/mutation) first, then subsystem migration.

---

## Summary

v11 gave us **object, liveness, commit** — the lifecycle half. This ADR adds **binding, obligation, reachability** — the resolution half. The two halves compose through projections (obligations require projections) and through commit (bindings are established and removed at commit points). Together they describe the full namespace-to-payload chain.

The reactor mediates time: informational I/O for resolution, operational I/O for lifecycle, coroutines as the boundary object, synchronous coordination for cross-core work. HAL grounds the reactor in hardware. Semantic lives in the two halves; time lives in the reactor.

The architecture specifies semantic contracts and mechanism requirements; implementation supplies the substrate, runtime, and data structures that realize them.
