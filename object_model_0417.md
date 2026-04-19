# Object Model

**Date.** 2026-04-17 (last updated 2026-04-19 for pillar cross-references).
**Status.** Current.
**Supersedes.** `LIVENESS.md` core claims (projections, reclamation). Subsystem-anatomy details derive from this document and live in their own specs.
**Extends.** `tx-kernel-architecture-v11.md`'s entity and commit machinery.

This document is the consolidated statement of the kernel's object model: how entities exist, how they are named, how they persist, and how they are reclaimed. It commits to **EBR-based traversal composed with refcounted pinning** as the substrate, and develops the two halves — **lifecycle** (object / liveness / commit) and **resolution** (binding / obligation / reachability) — as one integrated framework.

**Relationship to pillar organization.** This document's §7 (bindings, obligations) and §8.1.1 (bifurcation) are the substrate of `INVARIANTS.md` v3's OBL and BIF pillars. §6 (reclamation) is the substrate of PRED-7 (anti-TOCTOU). The reference hierarchy in §4 grounds the five-phase step discipline (STEP-4 in `INVARIANTS.md`): observation under guard (phase 1) uses IdentRef; upgrade (phase 2) transitions to Cap or OperationalEvidence; commit points (phase 4) are the SENTINEL-guarded atomic transitions this document specifies.

Subsystem-specific application (VFS, process, VM, net, etc.) derives from the rules stated here. Specific projection catalogs, binding taxonomies, and substrate module layouts are separate documents.

---

## 1. The two-claim basis

v11's two basis claims are the architectural axioms:

- *A kernel maintains `(signifier, consistency, binding)` triples.*
- *Every entity decomposes into `(identity, capability, payload)` layers.*

The second claim is the **lifecycle half**: entities exist, carry state, transition through commit. The first claim is the **resolution half**: signifiers reach entities through bindings in namespaces. The two halves compose: resolution produces references with obligations; lifecycle's commit phase installs bindings and acquires retention.

---

## 2. Substrate commitment: EBR + pinning

The model is built on the composition of two mechanisms, each used strictly within its domain of strength.

### 2.1 EBR for traversal

Epoch-based reclamation (EBR) handles traversal memory safety. Under an epoch guard, a thread can safely dereference any pointer it obtains during a walk of the binding structure. Physical reclamation is deferred until all guards that might have observed the slot have released.

EBR's classical limitation — that long-lived references block global epoch advancement — is resolved **by scoping**: guards are type-bounded to stack scope via Rust lifetimes. An epoch-derived reference cannot escape the guard that produced it.

### 2.2 Pinning for retention

Refcounted pins handle long-lived retention. Pin types (`Cap<T>`, `PayloadCap<T>`, and domain-specific operational contributions) are `'static` — they survive arbitrary epochs, cross thread boundaries, and can be stored in struct fields.

Pins contribute to per-entity retention counters. When a counter drops to zero and the entity has no other contributors, the entity becomes eligible for reclamation.

### 2.3 Upgrade as the bridge

The single linearization point between the two mechanisms is the **upgrade**: taking a traversal-derived reference and producing a pin. Implementation is a CAS on the entity's retention count against a `SENTINEL_DEAD` threshold: succeeds if the entity has not been marked reclamation-bound; fails cleanly (returning an error) if a concurrent reclaimer has passed the no-upgrade barrier.

After upgrade, the pin is independent of the producing epoch. The guard can be released; the pin endures.

### 2.4 Why this composition

- Hot traversal paths (path resolution, fd lookup, pid lookup) are refcount-free. Each hop is an epoch-guarded pointer dereference; no atomic RMW.
- Long-lived references pay their cost once at upgrade and survive unchanged afterward. No retirement-queue accumulation from misuse.
- Epoch advancement is blocked only by active traversals, which are bounded by operation structure. A cold-cache traversal yields to the reactor, *releases its guard during the yield*, reacquires on resume.

Accepted costs: epoch retirement latency (two epoch advances between retention-drop and physical free), debug complexity (two possible reasons a slot is not yet free), guard-across-async discipline (guards must not span `.await`).

---

## 3. Entities

### 3.1 Identity / capability / payload

An entity decomposes into three layers.

**Identity.** What makes the entity *this one*. Its slot, its stable name, what bindings point at. Pinned by `Cap<T>`.

**Capability.** The operations the entity supports. FileOps, InodeOps, protocol handlers. Injected at open time for per-instance capability (tty master/slave, device nodes).

**Payload.** The operational resources. Page cache, socket buffers, protocol state. Pinned by `T::OperationalEvidence` — which is `Cap<T>` for co-located entities, `PayloadCap<T>` for entities with indirected payload boxes, or typed contributions for entities with compound payload predicates.

### 3.2 Co-located vs indirected payload

Entities with small, bounded payload and no zombie-semantics requirement (DEntry, FdEntry, OpenFile) co-locate identity and payload in a single slot. For these, `Cap<T>` and operational retention coincide.

Entities with large or unbounded payload (Inode with its page cache, SocketData with protocol buffers) indirect payload to a separately allocated box. For these, `Cap<T>` pins identity only; `PayloadCap<T>` pins the payload box. `PayloadCap<T>` entails `Cap<T>` by structural invariant: the payload box internally references its identity slot and cannot outlive it.

### 3.3 Compound payload predicates

Some entities have payload that is live under a disjunction of independent contributions:

```
Inode.payload_live  ⇔  nlinks > 0 ∨ open_refs > 0
Frame.payload_live  ⇔  map_count > 0 ∨ cache_ref > 0
```

Each disjunct is counted by its own typed pin (`LinkPin`, `OpenPin`, `MapPin`, `CachePin`). Each pin is acquired via an epoch-guarded CAS with a projection pre-check; drop decrements its specific counter. The entity's payload becomes reclaimable when all disjuncts are simultaneously zero and no operation holds a reservation.

A binding site chooses the appropriate contribution based on its semantic role: a hard-link dentry carries a `LinkPin`, an open file carries an `OpenPin`, a PTE carries a `MapPin`. The obligation remains "operational"; the specific pin documents which disjunct the binding participates in.

---

## 4. Reference layering

Four reference types form a hierarchy. Each downgrade is free; each upgrade is fallible.

```
   Weak<T>
   (nullable, epoch-independent)
       ↓  observe under epoch
  IdentRef<'g, T>
   (epoch-guarded, stack-bound, memory-safe observation)
       ↓  pin under epoch
   Cap<T>
   (refcounted, pins identity, arbitrary lifetime)
       ↓  upgrade payload
  T::OperationalEvidence
   (entity-specific, pins payload, entails Cap<T>)
```

**`Weak<T>`** is a zone-slot reference with a generation tag. Contributes no retention. Upgrade compares the stored generation against the slot's current generation; mismatch (slot was reused) returns None.

**`IdentRef<'g, T>`** is an epoch-guarded pointer. Produced by traversal. Lifetime `'g` is tied to the guard; the borrow checker prevents escape. Contributes no retention. Safe to dereference because the guard defers reclamation.

**`Cap<T>`** is a refcounted pin on identity. Upgrade from `IdentRef` via SENTINEL_DEAD CAS. `'static` with respect to epochs. Can be stored, sent, held across `.await`.

**`T::OperationalEvidence`** is entity-specific operational retention. For co-located entities, equal to `Cap<T>`. For indirected entities, a `PayloadCap<T>` or typed contribution that additionally pins payload. Upgrade from `Cap<T>` via a CAS on the relevant payload counter.

The hierarchy makes structural monotonicity (§5) a type-level property: holding operational evidence implies holding identity retention implies having a safe dereference path implies having a weak reference. Downgrade is always available without cost.

---

## 5. Projections and monotonicity

### 5.1 The three projections

Every entity has (at least) three independent projections over its state. These are the semantic predicates that the lifecycle half uses to classify operations.

**Structural projection.** True iff some retention exists. Mechanically: retention count > 0 (any Cap-or-stronger reference held). This is the "entity is semantically alive" predicate; it is independent of physical reclamation timing.

**Namespace projection.** True iff the entity is reachable through a chain of bindings from a namespace root. This is a pure structural query over binding containers (§7). The entity itself carries no flag; the predicate is "the parent's children container holds a binding targeting this entity," recursively up to a root.

**Payload projection.** True iff the entity's operational state is available. For co-located entities, equivalent to structural. For indirected entities, true iff payload box exists and its retention is nonzero. For compound-predicate entities, the disjunction of independent contributions.

### 5.2 Monotonicity

Projections are **monotone**: each can only transition from true to false, never back. Operations that lower a projection (unlink, close, exit) cannot be reversed by later operations — a new entity must be created instead.

Monotonicity is the load-bearing property. It guarantees:

- A failed check is genuinely a failure (not racing a revival).
- Any "true" observed at some point is permanent evidence; "false" observed later is a clean transition.
- Commit-phase reservations that succeed linearize at a point where all required projections held, and remain valid until the operation's commit completes.

### 5.3 Within-entity entailment

`payload_live ⇒ structural_live`. Any operational evidence entails identity retention, because operational evidence is either (a) a `Cap<T>` directly, for co-located entities, or (b) holds a `Cap<T>` field internally. This is enforced at the type level: downgrade from any operational evidence to `Cap<T>` is free and always available.

The contrapositive: `¬structural_live ⇒ ¬payload_live`. Once identity retention drops to zero, payload cannot survive — all payload pins carry identity pins, so structural zero implies all payload pins have already dropped.

### 5.4 Cross-axis independence

`namespace ⟂ payload`. Namespace membership and payload liveness are governed by independent mechanisms (binding containers vs. refcounts). All four combinations are valid states:

| Namespace | Payload | State                                       |
|-----------|---------|---------------------------------------------|
| true      | true    | Fully alive, normally named                 |
| true      | false   | *(rare; transient during commit phases)*    |
| false     | true    | Unlinked-but-open file, zombie with payload |
| false     | false   | Zombie, fully closed unlinked file          |

The zombie case (ProcessIdentity persisting after ProcessPayload reclaimed) and the unlinked-but-open-inode case are not special states in the framework — they are natural points in this cross-axis grid, realized structurally by the identity/payload split (object_model §8.1.1).

### 5.5 Race degradation

Under monotonicity and the upgrade-as-linearization-point discipline, concurrent mutation can cause an operation to fail, but cannot cause it to succeed against a different entity, a different context, or with different semantics.

Proof sketch. (1) Projections decrease monotonically; any observation that fails after initial success means a genuine transition occurred, not a stale-then-revived state. (2) The SENTINEL_DEAD CAS is the single linearization point for upgrade; either the upgrade wins (reclamation waits) or the upgrade sees the sentinel (clean failure). (3) Binding removal is a container-level linearization point; resolvers either see the binding or don't, never see partial removal. (4) The epoch guard prevents use-after-free regardless of upgrade outcome.

---

## 6. Reclamation

### 6.1 Semantic death

An entity is **semantically dead** when all its projections are false. For a typical entity, this means no retention is held (no Cap, no operational evidence) *and* the entity is unreachable from any namespace (no binding chain leads to it).

Semantic death is the predicate that determines "no further operations on this entity are possible." It is the trigger for reclamation intent: the entity's slot is marked for reclaim, SENTINEL_DEAD is CAS'd into the retention counter, and the slot enters the reclaim queue.

### 6.2 Physical reclamation

Physical reclamation requires semantic death *plus* epoch quiescence:

```
physically reclaimable  ⇔  semantically dead  ∧  all observing epochs have passed
```

This is the happens-before chain:

1. Binding removed (container-level linearization point).
2. Binding's evidence field drops (Rust Drop semantics; no out-of-order possible).
3. Retention counter decrements; if zero, SENTINEL_DEAD CAS attempted.
4. SENTINEL_DEAD succeeds → reclaim queue enqueue.
5. Reclaim queue drains under bounded-work discipline, running destructors.
6. Slot returned to zone allocator after current epoch has advanced past the retirement point.

### 6.3 Bounded-work reclamation

Dropping the last pin on an entity with large payload (an inode with a gigabyte page cache, a socket with large buffers) cannot run its destructor synchronously on the dropping thread — the latency would be unbounded.

The reclaim queue accepts reclaim requests and drains them under a bounded-work budget per tick. Oversized reclaim items (an inode with a large page cache) split themselves: the destructor releases a bounded subset of payload, re-enqueues the remainder, yields. This preserves latency bounds on the hot path at the cost of slightly deferred physical return.

Synchronous drain is available under resource pressure (zone allocator exhausted, needing to free slots). Threads hitting allocation pressure pay the drain cost directly.

### 6.4 The two-tier free

"Freed" has two meanings:

- **Semantically freed**: no operation can reach the entity. Immediate on binding removal + retention drop to zero.
- **Physically freed**: zone slot returned to the allocator. Deferred by reclaim queue and epoch quiescence.

Userspace-observable events (a file disappearing, a process reaping) correspond to semantic freeing. The physical reclamation is interpreter-internal and not observable through any binding.

---

## 7. Bindings, obligations, reachability

### 7.1 Bindings

A **binding** is a structural slot in a container:

```
Binding<K, Obligation> {
    key:      K,                       // signifier
    evidence: Evidence<Obligation>,    // retention evidence, type derived from obligation
}
```

A binding exists iff present in its container. It has no independent lifetime, no refcount of its own. A **namespace** is a container of bindings.

### 7.2 Obligations

Every binding declares one of three obligations. Declaration is part of the binding's type; it cannot be inferred or omitted.

**Resolution-only.** The binding exists to answer "what's at this key." No retention promise. Evidence: `Weak<T>` or `()`. Resolvers that reach a target through a resolution-only binding must handle stale-reference outcomes.

**Addressability.** The binding promises identity stability: the target remains structurally alive, and resolving through the binding returns *this* entity (not a slot-reuse). The target may be payload-degraded. Evidence: `Cap<T>`.

**Operational.** The binding promises full usability: payload projection holds. Evidence: `T::OperationalEvidence`.

### 7.3 Evidence derivation

Evidence type is derived from the obligation and the target entity:

```
derive(ResolutionOnly,  T)  =  Weak<T>  |  ()
derive(Addressability,  T)  =  Cap<T>
derive(Operational,     T)  =  T::OperationalEvidence
```

The binding designer chooses an obligation; the compiler derives the evidence type; the entity's payload organization (co-located vs indirected vs compound) determines what the operational evidence concretely is. Binding declarations are therefore stable across entity-implementation changes: if Inode later changes from indirected to co-located payload, existing operational bindings to Inode continue to compile with the same declared obligation, only the underlying evidence type changes.

### 7.4 Reachability

Reachability is epoch-guarded traversal of binding chains. Properties:

- **Refcount-free.** Traversal produces `IdentRef<'g, T>` values along the way. No retention is acquired until the final upgrade.
- **Composable.** A multi-hop walk (path resolution through dentries, fd-to-file-to-inode chain) is a sequence of binding lookups under a single guard.
- **Obligation-terminal.** The walk ends when the resolver upgrades its final `IdentRef` to the retention level needed. Syscalls wanting to operate upgrade to operational evidence; registrations wanting only addressability upgrade to `Cap<T>`; cache populators may stay at `IdentRef` or downgrade to `Weak`.

Reachability may yield to the reactor for informational I/O (cache fill). On yield, the guard is released; on resume, a fresh guard is acquired and the trampoline state is re-validated against current structure. Monotonicity ensures safety: predicates that held before the yield may have lapsed (clean failure) but cannot have been silently replaced by a different target.

### 7.5 Obligation ↔ projection correspondence

Each obligation declares which projection must hold for the binding to be honored:

| Obligation       | Required projection | Realized via                    |
|------------------|---------------------|----------------------------------|
| Resolution-only  | (none)              | No pin; fallible upgrade at use |
| Addressability   | structural          | `Cap<T>`                         |
| Operational      | payload             | `T::OperationalEvidence`         |

Predicates on namespace projections are binding-chain reachability queries, computed over the container structure, not over entity state. The flag fields sometimes seen on entities (`removed` on a dentry, `detached` on a mount) are optimization hints for fast-path staleness detection, not primary state — the canonical predicate consults the container.

### 7.6 Bindings under commit

The step model's five-phase commit discipline (STEP-4 in `INVARIANTS.md`) integrates with bindings as follows:

- **Observe** (phase 1) produces `IdentRef`-carrying witnesses by traversal under the epoch guard.
- **Upgrade** (phase 2) promotes `IdentRef` to `Cap` or operational evidence as the obligation requires. Upgrades are fallible CASes against `SENTINEL_DEAD`; failure returns `Err` with no reservations yet held.
- **Reserve** (phase 3) acquires substrate resources (zone slots, index slots, credit) as linear tokens. Failure drops already-acquired reservations through Rust's Drop.
- **Commit** (phase 4) installs the new bindings by inserting the reserved-and-upgraded evidence into their target containers, atomically. Each substrate commit call is a commit point — one visibility boundary at which private reservation state becomes observer-visible. The commit sub-phase's infallibility rests on phase 3 having successfully acquired all evidence and reservations.
- **Publish** (phase 5) fires signal attachments declared on each commit point.

Binding removal at a commit point runs the evidence's Drop, which decrements retention. The happens-before chain is guaranteed by Rust's drop order plus the container's linearization primitive; §6.2's six-step sequence applies.

---

## 8. Cross-cutting rules

### 8.1 Entity factoring

Subsystem authors factor their entities as follows:

1. Determine identity/payload separation (see 8.1.1 below). If the entity's partial order admits `structural ⟂ payload` — that is, identity persisting after payload has dropped (zombies, unmounted-but-held mounts, closed-but-queued sockets) — factor into two types: `<Name>Identity` and `<Name>Payload`, each with its own zone.
2. Identify the entity type's payload organization: co-located (no split), indirected (identity + payload via PayloadCap), or compound-predicate (multiple typed contributions).
3. Define `T::OperationalEvidence` accordingly: `Cap<T>` / `PayloadCap<T>` / typed contributions.
4. Enumerate projections: structural (identity retention > 0), namespace (reachability through binding containers), payload (operational evidence presence or disjunction of contributions).
5. Identify external pinners if any (nlinks, map_count, etc.) and define their typed contributions.

#### 8.1.1 Identity / payload split convention

**Rule.** Where an entity's partial order admits `structural ⟂ payload` — where identity must survive while payload has already released — factor into two distinct zone-allocated types:

```rust
struct <Name>Identity {
    meta: SlotMeta,
    // identity fields: stable names, DLL entries, exit status, event queues,
    // observation slots, cross-subsystem addressability handles
    payload: Option<PayloadCap<<Name>Payload>>,
}

struct <Name>Payload {
    meta: SlotMeta,
    // payload fields: session state, computation context, buffers,
    // large per-instance state that is reclaimable independently of identity
}
```

Bindings with **addressability** obligation target the identity type: `Cap<<Name>Identity>`. Bindings with **operational** obligation target through the identity's payload field, taking a `PayloadCap<<Name>Payload>` or a typed `OperationalContribution<<Name>Payload>`.

**Why a structural split, not just Option<payload_fields>:** the god-struct pattern (identity fields + Option-wrapped payload fields in one allocation) forces all zombies to carry payload-sized slots. The structural split allocates identity and payload in separate zones with independent reclamation: payload reclaims at exit/umount/shutdown while identity persists in a small slot until reaped. Linux's single-allocation `task_struct` carries the full weight through the zombie window; txKernel does not.

**Reclamation is independent.** ProcessPayload reaches SENTINEL_DEAD and its slot reclaims at exit_commit. ProcessIdentity remains alive (binding evidence in parent.children keeps `Cap<ProcessIdentity>` retained) until reap_commit withdraws the binding, drops the Cap, and triggers ProcessIdentity's SENTINEL_DEAD. Two zone reclamations, two linearization points.

**Co-located (no split) is reserved for entities where identity and payload have no independent lifetimes.** DEntry (namespace presence coincides with the Cap's existence; no degraded-but-addressable state). FdEntry (per-fd-table slot; no degraded state). OpenFile (gone when last holder drops; no zombie semantics). These stay as single types; their `OperationalEvidence` is `Cap<T>` directly.

**Entities that must split under this convention:**

- `ProcessIdentity` / `ProcessPayload` — zombie semantics (payload drops at exit; identity drops at reap).
- `ThreadIdentity` / `ThreadPayload` — thread-exit semantics (computation state drops at thread exit; identity drops when detached from thread chain and last Cap released).
- `MountIdentity` / `MountPayload` — force-umount / MNT_DETACH-like semantics (superblock and backing device release at umount_commit; identity persists until no resolver holds a Cap).
- `SocketIdentity` / `SocketPayload` — post-close / FIN'd-drained semantics (protocol state and buffers release when drained; identity persists for any remaining fd holders, returning ENOTCONN / EPIPE on operations).

**Entities that stay co-located:**

- DEntry, Mount-point (if different from MountIdentity — see Mount subsystem spec), OpenFile, FdEntry.
- RNode is a special case: identity is `RNode` itself; payload depends on backing. For persistent-backed RNodes, the page cache is a compound-payload contribution. For synthetic (procfs/sysfs), there is no independent payload — the RNode is the projection. For struct-backed (pipe, socket, tty), the struct type (PipeRing, SocketPayload, TtyData) plays the role of the payload half; the RNode is effectively the identity half but is named RNode for historical clarity. Think of RNode as always-identity; its payload varies by Backing.

**Naming convention is part of the architecture, not cosmetic.** Field names, zone names, and type names must use the Identity/Payload suffix for split entities. Reviewers and lints check that bindings in containers never carry Cap to a payload type (always to identity); operational obligations go through a PayloadCap or typed contribution field on the identity.

### 8.2 Binding design

Binding designers:

1. Identify the signifier (key type) and target entity.
2. Choose the obligation based on the binding's semantic promise:
   - If the binding should invalidate cleanly on target death: resolution-only.
   - If the binding must name the same entity even after payload-degradation: addressability.
   - If the binding enables operations that require payload: operational.
3. Declare the binding type. Evidence type is derived.

### 8.3 Reference site rules

- **In checks/predicate code**: use `IdentRef<'g, T>` produced by traversal. Never store, never hold across yield points.
- **In structure/**: store published `Cap<T>` or operational evidence (these are `'static`).
- **In execution/, during the step's upgrade sub-phase** (STEP-4 phase 2): upgrade `IdentRef` to `Cap` or operational evidence under the substrate pin family's CAS discipline.
- **In execution/, during the step's reserve sub-phase** (phase 3): acquire linear substrate tokens; failure drops prior reservations cleanly.
- **In execution/, during the step's commit sub-phase** (phase 4): install evidence into bindings at commit points; evidence Drop on later binding removal decrements retention.
- **In projection/**: read structure via published references; never upgrade further (projections are observational).

### 8.4 What bindings are not

- Not pointers. A raw pointer cannot declare an obligation; bindings carry typed evidence that matches their obligation.
- Not references in the refcount sense. Bindings do not increment anything by existing; they *carry* evidence, and the evidence's existence-within-the-binding-slot is what contributes to retention.
- Not all cross-entity references. Some long-lived references are free-standing pins (an OpenFile's internal reference to its Inode, not stored in a namespace container) — these are structurally `Cap` or operational-evidence fields, not bindings.

---

## 9. Appendices

### 9.1 Glossary

- **EBR** — epoch-based reclamation. Defers physical free of structures until all observers have released.
- **Pin / pinning** — refcounted retention. A pin's existence contributes to a counter; reclamation is blocked while counters are nonzero.
- **Upgrade** — transition from a weaker reference (IdentRef, Weak) to a stronger one (Cap, operational evidence), via a fallible CAS.
- **SENTINEL_DEAD** — the counter value indicating reclamation is irreversible; upgrades observing it fail.
- **Semantic death** — state where all projections are false; reclamation intent can fire.
- **Physical reclamation** — the slot is returned to the allocator. Requires semantic death plus epoch quiescence.
- **Reclaim queue** — bounded-work drainer for large payload destruction. Splits oversized items to preserve latency bounds.

### 9.2 Relationship to v11 and related documents

- v11's entity model (§2, §3, §4 of `tx-kernel-architecture-v11.md`) is preserved and sharpened here. `Cap<T>` in v11 corresponds to this document's `Cap<T>` plus (where applicable) operational evidence.
- The five-phase step commit discipline (observe → upgrade → reserve → commit → publish) is specified in `INVARIANTS.md` (STEP-4) and `STEP_MODEL.md` §3; this document integrates it with bindings in §7.6.
- Projection-based liveness was introduced in `LIVENESS.md`. This document supersedes its core claims (§3–§6) with ADR-aligned versions; LIVENESS.md's §7 (predictions) and §8 (checklist) remain useful.
- The reactor's role and the informational/operational I/O distinction live in `tx-kernel-architecture-v11.md` Part IV and are not repeated here.

### 9.3 Open questions

- Exact guard-registration discipline under async (per-CPU vs per-task) is deferred to reactor specification.
- `Shared<T>` from v11 needs placement: a COW-capable specialization of `PayloadCap<T>` is the likely answer, detailed design deferred.
- Weak back-references for future rmap (Frame → VmEntries) are a natural extension; not required before Phase 2.
- Name-cache eviction and staleness discipline for resolution-only bindings is implementation choice; framework accommodates both lazy and eager strategies.
