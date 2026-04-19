# Architectural Invariants

**Status.** v3.1 (2026-04-19).

**Supersedes.** v3. Substantive changes: STEP-4 elaborated from a four-phase to five-phase discipline with explicit reserve vs commit sub-phases; STEP-6 scope clarified (retires operation-level function pair, preserves intra-step sub-phases); STEP-8 and SIG-4 sharpened to reference visibility boundaries and commit points.

**Supersedes (v3 → v2).** v2. Substantive changes: reorganized around the five subsystem pillars (Bifurcation, Predicate, Witness, Obligation, Signal); step primitive and dispatch carved out as non-pillar categories; implementation-bound phrasing replaced with constraint-style language; universal claims relaxed to per-entity declarations; three-letter prefixes throughout.

**Purpose.** State the load-bearing invariants of txKernel's object, execution, and signalling model. This document is the rule set; [`CONCEPTS.md`](./CONCEPTS.md) is the vocabulary. Proposed changes are checked against these invariants; violations require revisiting the framework, not silently bending the rule.

**Audience.** Designers, reviewers, agents, and automated lint infrastructure. The prefixed rule identifiers (BIF-*, PRED-*, WIT-*, OBL-*, SIG-*, STEP-*, DISP-*, ARCH-*, EXC-*) are intended for direct citation in review comments, lint messages, and test names. They are greppable, unambiguous under case-insensitive search, and stable across document revisions.

**Companion documents.**

- [`CONCEPTS.md`](./CONCEPTS.md) — vocabulary and decompositions referenced here.
- [`object_model_0417.md`](./object_model_0417.md) — entity model, reference hierarchy, reclamation.
- [`LIVENESS.md`](./LIVENESS.md) — per-subsystem projection catalog.
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) — module layout, mutation discipline, import rules.
- [`SIGNAL_ATTACHMENTS.md`](./SIGNAL_ATTACHMENTS.md) — publication catalog. *(forthcoming)*
- [`BUS.md`](./BUS.md) — bus primitive APIs, static/temporal layering. *(forthcoming)*

---

## 1. Organizing principle

Subsystems are organized around **five pillars**. Every subsystem design must answer one question per pillar:

- **Bifurcation (BIF)** — How does this entity factor? Identity-only, Identity/Payload split, compound payload?
- **Predicate (PRED)** — What projections does this entity define, and how are they realized?
- **Witness (WIT)** — What evidence does authorization produce, and what is its scope?
- **Obligation (OBL)** — What do bindings to this entity promise at commit?
- **Signal (SIG)** — Which transitions publish externally, on which carriers?

Three further categories hold framework-level invariants that are not pillar-scoped:

- **Step (STEP)** — The execution primitive through which pillars are expressed.
- **Dispatch (DISP)** — The syscall-boundary control plane.
- **Architecture (ARCH)** — System-wide placement and composition rules.

A final category names normative exclusions from the publication model:

- **Exclusions (EXC)** — Mechanisms that are not signals and must not route through bus primitives.

Proposed changes are evaluated pillar-by-pillar. A subsystem author walks through BIF → PRED → WIT → OBL → SIG applying each pillar's invariants; a reviewer does the same.

### Rule annotations

Individual rules may carry annotations that affect how reviewers weight them:

- **(core)** — Load-bearing for a broad class of downstream reasoning. Weakening a core invariant requires systemic review, not a localized change.
- **(anti-TOCTOU)** — Part of the framework's concurrency-correctness guarantee. Violations open race windows that silently produce incorrect results.
- **(intentional deviation)** — Marks a deliberate departure from established patterns (Linux conventions, database transaction models, etc.). The rule's body explains why the deviation is principled and what it buys.

Annotations are normative aids, not severity tiers. Every invariant is binding regardless of annotation.

---

## 2. BIF — Bifurcation invariants

**BIF-1.** Each entity must explicitly declare whether it admits a `structural ⟂ payload` factoring. The declaration is architectural, not inferred. Ad hoc splitting or ad hoc unification violates the declaration requirement.

**BIF-2.** If factored, identity and payload must be independently retained and reasoned about. Each side has its own retention, its own reclamation point, and its own projection set.

**BIF-3.** Bindings with addressability semantics target identity. Bindings with operational semantics reach payload through identity's payload-capable retention field. No binding targets payload directly without passing through identity.

**BIF-4.** Co-located entities (no split) have identity and payload coincident; their operational evidence is identity retention.

**BIF-5.** Signal carriers must attach to exactly one retention domain — identity or payload — never a cross-domain composite. Cross-domain coordination must be modeled as explicit pairing, not as a shared carrier.

**BIF-6.** Strong retention on identity must not imply strong retention on payload. The two retention counts evolve independently; payload may reach zero while identity remains retained (zombie-like states).

---

## 3. PRED — Predicate invariants

**PRED-1.** Predicates must be pure: no mutation, no allocation, no I/O, no blocking. A predicate reads entity state and context under a guard-scoped observation and returns a boolean.

**PRED-2.** Predicates must be guard-scoped. Observations used by predicates are bounded by an epoch guard; predicates do not upgrade observations to retention evidence.

**PRED-3.** The canonical projection vocabulary is `{structural, namespace, payload}`. Each entity explicitly declares which projections apply to it. An entity is not required to expose all three; it must declare which it supports.

**PRED-4.** Each projection must declare its realization mechanism. Admissible mechanisms include identity retention, binding-chain reachability, payload retention, and disjunctions of typed contributions. Mixing mechanisms for a single projection is not permitted.

**PRED-5.** Each projection must declare whether it is monotone under system evolution. Monotone projections transition only true → false, never back. Non-monotone projections require explicit justification and are expected to be rare; most projections are monotone.

**PRED-6.** Implications between projections on the same entity are per-subsystem theorems, derived from that entity's binding and retention structure. Universal cross-projection implications are not assumed.

**PRED-7 (core; anti-TOCTOU).** Concurrent mutation must not cause an operation to succeed against a different entity, a different context, or with different semantics than its arguments describe. Races must degrade to clean failure.

This is the framework's anti-TOCTOU guarantee: an observation made at check time cannot silently become a different observation at act time. The constraint is enforced by the composition of PRED-5 monotonicity (observations cannot transition back to true after being observed false), guard-scoped observation (the entity under observation cannot be reclaimed for the guard's duration), and upgrade-as-linearization (the transition from observation to retention is a single atomic point that either succeeds against the observed entity or fails cleanly).

Much of the rest of the framework — STEP-3 monotone progress, WIT-* witness discipline, SIG-1 signals-are-not-truth, DISP-2 dispatch state-blindness — leans on PRED-7 for its correctness arguments. Proposals that weaken PRED-7 (by allowing observations to silently retarget, or by introducing non-monotone intermediate states without explicit justification) invalidate a broad class of downstream reasoning and require systemic review.

**PRED-8.** Predicate evaluation must occur through a require function (see WIT-1). Direct predicate evaluation outside the require boundary bypasses authorization and is a framework violation.

---

## 4. WIT — Witness invariants

**WIT-1.** Witnesses must be produced only by require functions. A require function is the sole authorized producer of a witness and the sole authorized consumer of predicates at entry.

**WIT-2.** Witnesses carry guard-scoped observation evidence, not retention authority. A witness records that a predicate held at a point in time; it does not extend the lifetime or liveness of the observed entity.

**WIT-3.** Witnesses must be valid only within the guard and context under which they were produced. They do not cross guard boundaries, thread boundaries, or synchronous-step boundaries.

**WIT-4.** Witnesses must not be stored in long-lived state: no struct fields, no static bindings, no collections, no cross-step persistence. A witness that escapes its producing scope is a framework violation.

**WIT-5.** Cross-step or cross-guard continuation must carry strong retention evidence (identity retention or payload-capable retention), not witnesses. On resumption under a fresh guard, require is re-invoked to produce a fresh witness.

**WIT-6.** Witness construction is gated by its producing module. Require functions hold the private construction privilege for their witness types; code outside the producing module cannot fabricate witnesses.

---

## 5. OBL — Obligation invariants

**OBL-1.** Every externally meaningful binding to an entity must declare exactly one obligation from the set `{resolution-only, addressability, operational}`. Declaration is part of the binding type; it cannot be inferred, chosen at runtime, or omitted.

**OBL-2.** The strength of evidence a binding carries must match its declared obligation. Resolution-only bindings carry no retention. Addressability bindings carry identity retention. Operational bindings carry payload-capable retention. Weaker or stronger is a type violation.

**OBL-3.** Obligations cover evidence only. Whether a binding's install or withdrawal fires a signal is a separate, per-transition signal-attachment declaration (see SIG-*). Obligation typing does not imply publication.

**OBL-4.** The commit step of a mutating operation must upgrade witness observations to evidence matching the obligations of each outgoing binding. Upgrades are fallible at the reclamation sentinel; failure releases all already-acquired reservations and the operation fails cleanly.

**OBL-5.** Obligation declarations are stable across entity-implementation changes. A binding declared operational remains operational if the target entity's internal factoring changes; only the concrete evidence type varies.

---

## 6. SIG — Signal invariants

**SIG-1.** Signals are not truth. A signal fire does not define or guarantee truth at the moment of wake. Correctness is recovered by re-observation through a predicate under a fresh guard — which, in the step model, is the next step's require invocation.

**SIG-2.** Bus wires are not state. Stale signals, coalesced signals, and spurious wakes are admissible. Wires are publication channels; semantic truth lives in the canonical fields that predicates consult.

**SIG-3.** Signal attachment must be opt-in and declared per transition. Not every predicate transition publishes; attachments are selected for transitions with external subscribers. Declaration is per-subsystem.

**SIG-4.** Signal publication must occur after a commit point's visibility boundary, within the same step, on the carrier attached to that transition. Signal-before-boundary is forbidden. For steps with multiple commit points, each commit point's attached publications fire after that specific visibility boundary — not batched at step end — so that observers subscribed to earlier wires observe the correct pre-state of later commits.

**SIG-5.** Signal ordering is per-carrier, not global. The sequence write → signal → wake → re-observation holds for a single wire. Across different wires, no happens-before relation is implied.

**SIG-6.** Publication is not a substrate responsibility. Signal firing uses bus primitives called directly from commit-disciplined step code. The substrate enforces mutation linearization; the bus handles publication; the two are separate mechanisms.

**SIG-7.** Signal publication must match BIF-5: each attachment names exactly one retention-domain carrier. Cross-domain publications are a bifurcation violation.

**SIG-8 (per-step publication).** Streaming operations publish per step, not per operation. Each step that produces observable progress fires its declared signal attachments before returning. A multi-step operation that advances across N steps may fire a signal N times; it does not batch publications until the operation terminates.

*Rationale.* Subscribers observing a streaming operation's signals must see progress in a timely fashion. A pipe writer that waits for data must be woken as bytes arrive, not only when the reader finishes. A process tracer observing syscall progress must see each step's observable effects, not only the final outcome. Batching publications until operation termination would violate the reactive contract subscribers rely on.

*Relationship to SIG-4.* SIG-4 orders signal publication relative to the state write within one step. SIG-8 orders publication relative to the operation's lifecycle across steps. Both must hold: each publication happens after its corresponding write (SIG-4), and publications for progressive operations fire per step rather than batched at operation end (SIG-8).

**SIG-9 (subscriber obligation on wake; pair with SIG-1).** Wake delivery does not grant truth. Subscribers must re-establish truth via predicate evaluation under a fresh guard before acting on any state the wake purportedly signaled.

SIG-1 states the publisher-side rule: signals do not guarantee truth. SIG-9 states the subscriber-side obligation: upon wake, subscribers must perform a fresh predicate evaluation. The two rules form a matched contract; a system honoring one without the other is incoherent.

*Consequences.* Subscriber code acting on wake alone — without invoking a predicate under a fresh guard — is a SIG-9 violation regardless of whether the signal was correctly published. The wait primitive's loop enforces this structurally: wake returns control to the condition closure, which evaluates under a fresh guard. Code that bypasses this loop (e.g., by consuming wake notifications directly without the wait primitive) must still honor SIG-9 by performing its own re-observation.

**SIG-10 (subscriber obligation on bus state; pair with SIG-2).** Subscribers must not infer truth from bus wire state. All semantics derive from predicate evaluation under a fresh guard, not from reading a wire's current bits or events.

SIG-2 states the publisher-side rule: bus wires are not state. SIG-10 states the subscriber-side obligation: code observing bus state (via diagnostic peek, via subscription list inspection, via any mechanism) must not treat what it observes as semantic truth. Truth lives in the fields that predicates consult.

*Consequences.* Subscriber optimizations that read a RawQueue's current bit set to decide whether to invoke a predicate are SIG-10 violations — they treat the bit as predictive of the predicate's outcome, which it is not (bits may be stale, coalesced, or spurious). The correct pattern is: predicate first; subscribe only on negative outcome; predicate again on wake.

**SIG-11 (cross-capability firing within a subsystem).** A step may fire wires on other capabilities within the same subsystem, provided each wire names a single carrier (BIF-5). Cross-capability firing within a subsystem is legal; cross-subsystem firing requires explicit coordination through the target subsystem's commit path.

*Example.* A pipe's `step_write_commit` fires on the read end's `read_wq` after writing bytes to the ring. The read end is a distinct capability from the write end, but both are within the pipe subsystem. The step's publish phase fires on the other capability's wire; BIF-5 is satisfied because the wire (`read_wq`) attaches to exactly one retention domain (the read end's Identity slot).

*Boundary.* Cross-subsystem firing (subsystem A's step fires on subsystem B's wire) is a different concern. It requires the fire to happen within subsystem B's own step commit path — typically by subsystem A invoking subsystem B's step function as a nested operation, letting B's step perform the fire. Direct cross-subsystem fires from A into B's wires bypass B's commit discipline and are a framework violation.

**SIG-12 (publication allowed in AdvancedThenBlocked).** A step that returns `AdvancedThenBlocked(progress, carrier, interests)` may publish signals for the progress portion before returning. The composite outcome does not prevent or defer publication; the publish phase runs normally during the step's mutation portion, and the step then reports the composite outcome.

*Rationale.* The advance portion of an AdvancedThenBlocked outcome is indistinguishable from a pure Advanced outcome with respect to publication (SIG-8 per-step publication applies). The blocking portion is a separate concern — it names the wake carrier for the driver's wait composition. Forbidding publication in composite-outcome steps would force subsystems to split otherwise-unified steps into separate steps (one that advances and publishes, one that declares blocking), adding unnecessary complexity.

*Consequences.* Subsystem authors may freely publish in the advance portion of a composite outcome. The publication is subject to SIG-4 (after the linearizing write) and SIG-6 (from step publish phase) as usual. No additional rules apply because the outcome is composite.

---

## 7. STEP — Execution primitive invariants

**STEP-1.** Execution must be driven by a closed outcome algebra defined by the dispatch contract. The algebra distinguishes:

- completed progress,
- no-progress conditions requiring wait,
- partial progress followed by wait,
- terminal success,
- terminal error.

The specific variants are enumerated in [`CONCEPTS.md §5.1`](./CONCEPTS.md); the algebra is closed (see ARCH-3).

**STEP-2.** Steps must be synchronous and bounded. A step does not suspend, does not construct futures, does not interact with the executor. Work exceeding the step's bound must split across multiple steps.

**STEP-3.** Step progress must be monotone with respect to committed state transitions, relying on PRED-5. No step undoes a previous step's published progress. Retry-after-wait proceeds from the accumulated state, never from an earlier state.

**STEP-4 (core).** Within a mutating step, the commit discipline is five sub-phases in fixed order:

1. **Observe** under a fresh epoch guard; require produces witnesses bearing `IdentRef<'g, T>`.
2. **Upgrade** each required observation to the evidence its target binding demands (`Cap<T>` or `T::OperationalEvidence`) via the SENTINEL_DEAD-guarded CAS. Partial upgrade is forbidden: either all required evidence is held or the step returns `Err` with prior reservations dropped.
3. **Reserve** substrate slots (`zone::reserve`, `index::reserve_slot`, `credit::reserve`). Reservations are linear, private, and carry no observer-visible state. Failure to reserve drops all prior reservations through Drop and returns `Err`.
4. **Commit** each reservation at a **commit point** (`zone::sign`, `index::commit`, `index::withdraw_commit`, `index::swap_commit`, `credit::commit`). Each commit point is a **visibility boundary**: the write becomes observer-visible atomically at the substrate primitive's linearization point. Once a commit succeeds, STEP-8 attaches to whatever progress it represents.
5. **Publish** declared signal attachments for each commit point, after the write, on the same carrier (SIG-4, SIG-5).

Violations:

- Skipping upgrade before any commit (mutation without matching retention).
- Reserving without a corresponding commit or drop (leaked reservation).
- Writing fields directly (bypassing the reserve → commit pair).
- Firing a signal before its paired commit point.
- Firing a signal attached to a commit point that never executed.

*Sub-phase rationale.* The reserve/commit split names the cancellation-safety structure: pre-reserve nothing is allocated; post-reserve private slots exist but are droppable; post-commit the write is irreversible (STEP-8). Without this naming, "mutate via substrate primitives" elides the cleanup-on-failure property that `zone::reserve` + `zone::sign` enforce through linear types.

*Substrate primitives named by sub-phase:*

| Sub-phase | Primitives |
|---|---|
| Reserve | `zone::reserve`, `index::reserve_slot`, `credit::reserve` |
| Commit  | `zone::sign`, `index::commit`, `index::withdraw_commit`, `index::swap_commit`, `credit::commit` |

**STEP-5.** A no-progress outcome must name its wake carrier and interest conditions. The driver composes the wait on that carrier; the next step re-observes under a fresh guard. No-progress is not failure; it is a deferred-readiness outcome (see SIG-1: the wake itself is not truth).

**STEP-6 (intentional deviation).** There is no operation-level commit phase, and no `*_prepare` + `*_commit` function-pair split. One-shot operations are steps that terminate in success on first call; multi-step operations progress through intermediate outcomes until terminating. Commit discipline applies per-mutation within a step (STEP-4's five sub-phases), not per-operation.

*Scope clarification.* STEP-6 retires two things and preserves one thing. It retires the operation-level commit phase (no BEGIN/COMMIT/ABORT boundary around an operation) and the `*_prepare` + `*_commit` function-pair convention (the pair collapses into one step function body). It preserves the *intra-step* reserve/commit sub-phase distinction (STEP-4, phases 3–4) because the reserve-then-commit structure is how substrate primitives enforce cancellation safety at the type level. Reviewers checking STEP-6 look for operation-level phase boundaries or function-pair splits, not for intra-step reserve/commit sub-structure.

*Why this departs from established systems.* Linux syscall implementations commonly wrap work in a do-once-and-commit shape: prepare arguments, acquire subsystem locks, perform work, publish results, release. Databases and transaction managers use explicit BEGIN/COMMIT/ABORT boundaries to demarcate operation lifetimes. Both treat operation-level commit as a first-class phase with its own vocabulary and its own rollback semantics.

txKernel intentionally does not. Making commit a per-mutation discipline within steps, rather than a per-operation phase, buys three properties that the operation-commit pattern forfeits:

- **Progressive operations express naturally.** Operations that make bounded progress across many invocations (`read`, `write`, `splice`, `sendfile`, `copy_file_range`) do not fit the commit-once pattern without awkwardness. Under the step model, they are the common case, not a special carve-out.

- **No rollback required.** Because STEP-3 mandates monotone progress, there is no operation-level state to unwind when a step returns a no-progress outcome. Each already-published mutation stands; the operation resumes from where it was, not from where it began. Systems with operation-commit typically need rollback machinery to handle failures mid-operation; txKernel replaces this with monotone-progress-under-retry.

- **Partial progress is reportable.** Outcomes like "made partial progress, then stalled" (STEP-1's partial-progress-followed-by-wait variant) fit naturally alongside "completed" and "no progress." Under operation-commit, this trichotomy forces either coarse rollback (discard partial work) or awkward straddling (commit the prepared but not-yet-published fraction). The step model reports both facts atomically.

The trade-off is that operations cannot be "aborted cleanly after partial commit." This is deliberate: kernel operations that reach partial completion have published effects that other observers may already have acted on, and retracting them would violate PRED-7. POSIX's `ssize_t` return convention reflects exactly this — partial completion is reported, not undone.

Proposals to add an operation-level commit phase, a transactional abort mechanism, a two-phase operation boundary, or to re-split step functions into `*_prepare` + `*_commit` pairs are STEP-6 violations. Proposals to name, lint, or elaborate the intra-step reserve/commit sub-phase distinction fall under STEP-4 and do not conflict with STEP-6.

**STEP-7 (fresh observation).** All semantic decisions within a step must be based on require executed under the current guard. Prior observations — results carried from earlier steps, cached predicate results, fields read on a previous invocation, assumptions implicit in incoming arguments — do not constitute grounds for action in the current step.

The rule is stronger than witness scope (WIT-3). Witness scope prohibits *witnesses* from crossing step boundaries; STEP-7 additionally prohibits *any semantically-loaded observation* from doing so. A step that reads a field during setup, returns Blocked, and on resumption acts on the previously-read value — without re-requiring — violates STEP-7 even though no witness crossed the boundary.

*Enforcement mechanism.* Steps acquire their guards afresh on entry (per STEP-2 and STEP-4). All state-sensitive decisions must happen within require invocations that observe current entity state under the current guard. Retained retention evidence (Cap, OperationalEvidence) is admissible cross-step data because it confers identity stability without asserting any projection's current truth.

*Relationship to PRED-7.* STEP-7 is the step-level manifestation of PRED-7's anti-TOCTOU guarantee. PRED-7 mandates that races cannot silently mislead an operation; STEP-7 operationalizes this by requiring each step's actions to be predicated on fresh observation, so that between-step mutations by other actors are detected at the next step's require, not masked by stale knowledge.

**STEP-8 (progress irreversibility).** Reported progress is observable and must not be undone. Progress becomes externally visible at the **visibility boundary** of each commit point (STEP-4 phase 4); pre-commit reservations carry no such visibility. Once any commit point within a step has executed, whatever progress it represents is irreversible: subscribers to the subsystem's signal attachments, readers of its projections, and other operations observing the affected fields may already have acted on the post-boundary state. A step that returns an outcome containing progress — `Advanced(p)` or `AdvancedThenBlocked(p, ...)` — has crossed at least one visibility boundary attributable to `p`.

This strengthens STEP-3 (monotone progress across steps) with the externality corollary: not only does the subsystem not roll back its accounting, but other parties have structural dependencies on the advance having occurred. Rolling back reported progress would violate not only the subsystem's internal consistency but observers' assumptions.

*Consequences.* A step that publishes progress and then encounters an error must not attempt to undo the published progress. If the error is unrecoverable, the step still returns `Advanced(p)` (or `AdvancedThenBlocked(p, ...)`) with the successful progress, and surfaces the error on the *next* step invocation as `Err(errno)`. One-shot errors that prevent any progress return `Err` directly; errors after partial progress are deferred one step.

Drivers translate this to POSIX semantics: accumulated progress before an error returns `Ok(progress)` (partial success); error without progress returns `Err(errno)`. The `ssize_t` convention in POSIX is exactly this translation.

**STEP-9 (composite outcome).** A step may both advance progress and declare blocking in a single outcome. The `AdvancedThenBlocked(progress, carrier, interests)` variant is the structural expression of this: the step's publish phase fires signals for the advance; the same outcome reports the wait carrier.

*Rationale.* A step that drains part of a ring, observes the ring become empty, and finds no EOF is reporting two facts: progress occurred, and further progress requires waiting. Forcing these into separate step invocations (one to report progress, another to discover blocking) would duplicate predicate work, complicate driver loops, and obscure the atomic nature of "partial progress, then stall."

The framework admits composite outcomes as a peer to pure `Advanced` and pure `Blocked`. Splitting into separate steps is permitted but not required; subsystems choose based on their internal structure.

*Interaction with other invariants.* The advance portion of a composite outcome is subject to STEP-8 (irreversible) and SIG-4 (signal publishes after the write). The blocking portion is subject to STEP-5 (names the wake carrier). Drivers handle composites by accumulating the progress and then composing the wait — not by treating them as anomalous.

---

## 8. DISP — Dispatch invariants

**DISP-1.** Dispatch must operate over a closed set of driver modes. Dispatch does not invent wait policies, retry policies, or composition patterns beyond the declared set.

**DISP-2.** Dispatch must remain state-blind. It observes syscall metadata, step outcomes, wake events, and thread context; it does not read canonical subsystem state. Optimizations that inspect subsystem state to bypass a step are DISP-2 violations.

**DISP-3.** Dispatch composes wait futures and execution attempts but does not define truth. The truth is determined by predicates within steps; dispatch only schedules and composes.

**DISP-4.** Dispatch responsibilities decompose into disjoint classes. The classes are enumerated in [`CONCEPTS.md §11`](./CONCEPTS.md) and are closed (see ARCH-3). No class implements another class's decisions: observers do not admit; gates do not park; interceptors do not admit; wait adapters do not admit; drivers do not admit or transfer control.

**DISP-5.** Subsystem-internal publications are not dispatch mechanisms. Publications fire from step commit under SIG-4 discipline; the term "hook" is reserved for dispatch-boundary mechanisms.

**DISP-6 (driver isolation).** Drivers must not inspect subsystem state; all semantics flow through `StepOutcome`. No peeking at capability fields for optimization, no reading shared handles to short-circuit step invocation, no consulting external tables to decide whether to call step.

DISP-6 is the contract-level complement to DISP-2. DISP-2 prohibits the driver from inspecting subsystem state; DISP-6 prohibits the subsystem from offering state-dependent information through any channel other than `StepOutcome`. Together they close the "driver could technically see this state" loophole: the state must not only be off-limits to the driver, it must not be reachable from the driver's vocabulary at all.

*Consequences.* Capability objects exposed to dispatch must not carry state-dependent fields that drivers might read. Handles passed between step invocations must be opaque from the driver's perspective — the driver may store them, pass them back, but not interpret them. Any state-dependent decision (should we wait? can we retry? is there more work?) must surface through `StepOutcome` variants and their parameters, never through fields the driver inspects directly.

*Enforcement.* Capability types exposed at the dispatch/subsystem boundary have narrow APIs: typically a handle token and a function pointer. Fields with semantic content are private to the subsystem. Lints flag driver-layer code that imports subsystem-internal types beyond the narrow interface.

---

## 9. ARCH — Architectural invariants

**ARCH-1.** The system is structured into three planes:

- **Semantic plane** — predicates, require, witnesses.
- **Execution plane** — obligations, commit discipline, step.
- **Publication plane** — signals, wait contracts.

Bifurcation is a structural constraint that applies across planes.

**ARCH-2.** Every mechanism must have coordinates on three axes:

- **Plane** — which plane(s) the mechanism operates on.
- **Dispatch class** — observe, intercept, gate, wait-adapt, drive, or subsystem-internal.
- **Middleware vs fixed** — whether the mechanism is a protocol combinator over caller-supplied semantics or has hard-coded policy.

A mechanism lacking coordinates on any axis is either misclassified or genuinely novel; the latter requires architectural review.

**ARCH-3.** Extensions must not introduce new primitives without revisiting invariants. Catalogs whose membership derives from framework constraints (step outcome algebra, driver modes, wait protocols, bus primitives, dispatch classes) are closed; proposed additions require re-examination of the invariants that determined their current membership.

**ARCH-4.** Middleware must be a higher-order protocol combinator: it takes caller-supplied semantic operations or predicates as parameters and supplies execution protocol as implementation. Mechanisms that hard-code policy — observers, interceptors, gates, subsystem operations, bus publications — are not middleware and must not be implemented as one.

---

## 10. EXC — Normative exclusions

The following mechanisms are not signals and must not be implemented using bus publication primitives:

**EXC-1. Fault injection** (synchronous SIGSEGV, SIGFPE, SIGBUS, SIGILL from faulting context). Thread-local mechanism at trap-return; specified by the signal subsystem, not by bus publication.

**EXC-2. Cross-core barriers** (TLB shootdown, IPI synchronization). Synchronous coordination where the issuing thread blocks on acknowledgment; specified by the reactor's synchronous coordination API. Not a predicate transition.

**EXC-3. Single-waiter handoff.** Subsystem-owned one-waiter protocols. Do not model shared predicate publication; not part of the publication catalog.

Reusing bus publication primitives for any of these mechanisms is an architectural regression, not a convenience. Each has its own documented primitive elsewhere in the system.

---

## 11. The one-liner

The model in one sentence:

> **Predicates define truth. Attempts express progress. Drivers schedule progress. Signals make waiting efficient. Only fresh observation authorizes action.**

Five roles, in causal order. Suitable for the top of architecture documents, review checklists, and subsystem authorship guides.

---

## 12. The execution-publication cycle

The central cycle, anchoring PRED-*, STEP-*, and SIG-*:

```
step ──► state write ──► signal ──► wake ──► step
          (truth)         (hint)     (hint)   (re-observation)
                                                 │
                                                 └─ fresh guard
                                                    fresh require
                                                    fresh witness

  write            = linearization of the transition (STEP-4, SIG-4)
  signal           = commit-disciplined publication (SIG-3, SIG-4, SIG-6)
  step             = bounded synchronous observation + progress (STEP-2, STEP-3)
  wake             = hint to re-observe (SIG-1, SIG-2)
  re-observation   = next step's require under fresh guard (WIT-3, WIT-5)
```

The arrow from wake to the next step is not a retry in the failure-recovery sense. It is continuation of the operation's progression, and the inner step call is the re-observation by construction (WIT-1 makes require the sole predicate consumer; WIT-5 makes re-invocation the sole re-observation path after a guard release).

---

## 13. Per-invariant one-liners

For use in review comments, lint messages, and test names.

### BIF

| ID | One-liner |
|---|---|
| BIF-1 | Entities must declare their factoring. |
| BIF-2 | If factored, identity and payload are independently retained. |
| BIF-3 | Bindings target identity; payload is reached through identity. |
| BIF-4 | Co-located entities have coincident identity and payload retention. |
| BIF-5 | Each signal carrier attaches to exactly one retention domain. |
| BIF-6 | Identity retention does not imply payload retention. |

### PRED

| ID | One-liner |
|---|---|
| PRED-1 | Predicates are pure. |
| PRED-2 | Predicates are guard-scoped; they do not upgrade. |
| PRED-3 | Canonical projections are structural, namespace, payload; per-entity declared. |
| PRED-4 | Each projection declares its realization mechanism; no mixing. |
| PRED-5 | Each projection declares monotonicity; most are monotone. |
| PRED-6 | Cross-projection implications are per-subsystem theorems. |
| PRED-7 | **(core; anti-TOCTOU)** Races degrade to failure, not to silent incorrect success. |
| PRED-8 | Predicate evaluation occurs through require; direct evaluation bypasses authorization. |

### WIT

| ID | One-liner |
|---|---|
| WIT-1 | Require is the sole producer of witnesses. |
| WIT-2 | Witnesses carry observation evidence, not retention. |
| WIT-3 | Witnesses are valid only within their producing guard and context. |
| WIT-4 | Witnesses must not be stored in long-lived state. |
| WIT-5 | Cross-step continuation carries retention evidence; witnesses are re-produced. |
| WIT-6 | Witness construction is gated by its producing module. |

### OBL

| ID | One-liner |
|---|---|
| OBL-1 | Every binding declares exactly one obligation. |
| OBL-2 | Evidence strength must match declared obligation. |
| OBL-3 | Obligations cover evidence only, not publication. |
| OBL-4 | Commit upgrades observations to evidence matching obligations; upgrade is fallible. |
| OBL-5 | Obligation declarations are stable across implementation changes. |

### SIG

| ID | One-liner |
|---|---|
| SIG-1 | Signals are not truth; fresh observation authorizes action. |
| SIG-2 | Bus wires are not state. |
| SIG-3 | Signal attachment is opt-in and declared per transition. |
| SIG-4 | Signals publish after the linearizing write, same commit, same carrier. |
| SIG-5 | Signal ordering is per-carrier, not global. |
| SIG-6 | Publication is not a substrate responsibility. |
| SIG-7 | Each signal attachment names exactly one retention-domain carrier. |
| SIG-8 | Streaming operations publish per step, not batched at operation end. |
| SIG-9 | Wake delivery does not grant truth; subscribers must re-observe (pair with SIG-1). |
| SIG-10 | Subscribers must not infer truth from bus state (pair with SIG-2). |
| SIG-11 | Cross-capability firing within a subsystem is permitted (BIF-5 still holds). |
| SIG-12 | Publication is allowed in steps returning AdvancedThenBlocked. |

### STEP

| ID | One-liner |
|---|---|
| STEP-1 | Execution runs on a closed outcome algebra. |
| STEP-2 | Steps are synchronous and bounded. |
| STEP-3 | Progress is monotone across steps. |
| STEP-4 | **(core)** Five-phase in-step discipline: observe → upgrade → reserve → commit → publish. |
| STEP-5 | No-progress outcomes name their wake carrier. |
| STEP-6 | **(intentional deviation)** No operation-level commit phase; commit is per-mutation within a step. |
| STEP-7 | Steps act only on fresh observation; prior truth is not carried forward. |
| STEP-8 | Reported progress is observable and must not be undone. |
| STEP-9 | A step may both advance progress and declare blocking in one outcome. |

### DISP

| ID | One-liner |
|---|---|
| DISP-1 | Dispatch operates over a closed set of driver modes. |
| DISP-2 | Dispatch is state-blind. |
| DISP-3 | Dispatch composes; it does not define truth. |
| DISP-4 | Dispatch classes are disjoint. |
| DISP-5 | Subsystem publications are not dispatch hooks. |
| DISP-6 | State-dependent information flows through `StepOutcome`, not through side channels. |

### ARCH

| ID | One-liner |
|---|---|
| ARCH-1 | The system decomposes into semantic, execution, and publication planes. |
| ARCH-2 | Every mechanism has coordinates on three axes. |
| ARCH-3 | Closed catalogs admit no ad hoc extension. |
| ARCH-4 | Middleware is a higher-order protocol combinator. |

### EXC

| ID | One-liner |
|---|---|
| EXC-1 | Fault injection is not a signal. |
| EXC-2 | Cross-core barriers are not signals. |
| EXC-3 | Single-waiter handoff is not a signal. |

Example review usage:

- *"This violates DISP-2; move the pipe-empty check into the step function."*
- *"This violates WIT-3; use a retention handle to carry across the step boundary and re-require on entry."*
- *"This is a SIG-4 violation: the signal fires before the state write."*
- *"This is an EXC-2 violation: TLB shootdown must use the reactor's synchronous coordination API."*

---

## 14. Relationship to other documents

This document **defines** the rules. Other documents **depend on** them. References are directional: other documents cite rule identifiers from here; this document does not duplicate their content.

- [`CONCEPTS.md`](./CONCEPTS.md) — the vocabulary these rules use. Concepts defines terms; invariants states rules using them.

- [`object_model_0417.md`](./object_model_0417.md) — entity factoring (§8.1.1) is the basis for BIF-1. Obligation declarations (§7) are the basis for OBL-*. Reference semantics are the substrate on which WIT-* rely.

- [`LIVENESS.md`](./LIVENESS.md) — per-subsystem projection catalogs realize PRED-3, PRED-4, PRED-5. Signal attachments are cataloged separately (SIGNAL_ATTACHMENTS.md) per OBL-3.

- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) — the four-module layout, the in-step commit discipline (STEP-4), the import rules enforcing DISP-2, BIF-5, WIT-3, and related boundary constraints.

- [`SIGNAL_ATTACHMENTS.md`](./SIGNAL_ATTACHMENTS.md) *(forthcoming)* — the publication catalog. Records per-transition attachments per SIG-3; subject to SIG-4, SIG-5, SIG-7.

- [`BUS.md`](./BUS.md) *(forthcoming)* — bus primitive APIs, static/temporal layering, wire declaration and subscription semantics. Implementation details kept out of this document per SIG-6.

- [`ADR-resolution-half.md`](./ADR-resolution-half.md) — the reactor's role. EXC-2 references the synchronous coordination API defined there.

Implementation-specific detail — struct layouts, reference-type names, macro expansions, bus-primitive APIs, driver-mode enumerations — lives in the documents listed above, not here. This document constrains what must hold; the others describe what is.

---

## 15. Reviewing against the invariants

A proposal, architecture edit, or code change is checked pillar-by-pillar:

1. **BIF.** Does the entity declare its factoring? Are bindings aimed at the correct retention domain? Do signals attach to exactly one domain?
2. **PRED.** Are predicates pure, guard-scoped, consulted only through require? Are projections declared with realization mechanisms and monotonicity properties?
3. **WIT.** Are witnesses produced only by require, guard-scoped, non-escaping, non-persistent?
4. **OBL.** Do bindings declare obligations? Does evidence strength match? Is publication treated as separate from obligation?
5. **SIG.** Is the publication opt-in, declared per transition, attached to one carrier, fired after the linearizing write? For streaming operations, does publication fire per step rather than batched? For subscribers, is truth re-established via predicate evaluation rather than inferred from wake or bus state? For cross-capability firing, does the step stay within its own subsystem's commit path?
6. **STEP.** Are steps synchronous and bounded? Is progress monotone? Is the five-phase discipline observed — observe, upgrade, reserve, commit, publish — with reservations dropped cleanly on any pre-commit failure and each commit point's publication paired correctly? Does a no-progress outcome name its wake carrier? Does the step act only on fresh observation under the current guard? Is advanced progress treated as externally visible from its visibility boundary onward and non-retractable?
7. **DISP.** Is dispatch state-blind? Are classes disjoint? Is composition via closed driver modes? Does the subsystem/driver contract communicate state-dependent information exclusively through `StepOutcome`?
8. **ARCH.** Does the mechanism have coordinates on all three axes? Does it respect closed catalogs? If it is middleware, is it a protocol combinator?
9. **EXC.** If the mechanism handles fault injection, cross-core coordination, or single-waiter handoff, is it routed through the non-bus primitive specified for its class?

A proposal surviving all nine checks is admitted by the framework. Failure on any pillar is a concrete, citable violation.

---

## 16. Changelog from v2

Principal changes:

- **Reorganized by pillar.** The twelve flat v2 invariants (I-1 through I-12) are redistributed across the five pillars plus STEP, DISP, ARCH, EXC categories. Pillar organization matches how subsystems are actually designed and reviewed.

- **Three-letter prefixes.** Rule identifiers are three-letter prefixed (BIF-*, PRED-*, etc.) for greppability, unambiguous citation in lint messages, and stability for test-name conventions. Single-letter prefixes have been retired.

- **Constraint-style phrasing.** Rules are stated as constraints ("must", "must not", "is defined as") rather than implementation descriptions. Struct layouts, concrete reference types, and API details have been moved to [`object_model_0417.md`](./object_model_0417.md), [`BUS.md`](./BUS.md), and subsystem specs.

- **Relaxed universal claims.** PRED-3 no longer states "every entity exposes all three canonical projections"; entities declare which projections apply. PRED-5 treats monotonicity as a per-projection declaration rather than a universal property, although most projections are expected to be monotone.

- **Added pillar-level invariants.** Previously under-specified rules — witness scoping (WIT-3, WIT-4, WIT-5, WIT-6), obligation-evidence separation from publication (OBL-3), in-step commit discipline (STEP-4), bifurcation-publication matching (SIG-7) — are now named and numbered.

- **Elevated three-axis placement.** The three-axis placement rule (ARCH-2) is now load-bearing, not a descriptive tool in CONCEPTS.

- **Exclusions as a numbered category.** Previously a prose section, carve-outs are now EXC-1, EXC-2, EXC-3 with citable identifiers.

Mapping from v2 identifiers:

| v2 | v3 |
|---|---|
| I-1 | SIG-1 |
| I-2 | SIG-2 |
| I-3 | STEP-5 |
| I-4 | STEP-3 |
| I-5 | DISP-2 |
| I-6 | SIG-5 |
| I-7 | SIG-4 |
| I-8 | DISP-4 |
| I-9 | WIT-3 (plus WIT-4, WIT-5) |
| I-10 | *(moved to BUS.md; static-declaration property, not a framework invariant)* |
| I-11 | *(moved to BUS.md; subscription-liveness property, not a framework invariant)* |
| I-12 | STEP-2 |

v2's I-10 and I-11 have been moved to BUS.md because they describe static-layer implementation properties (wire declaration lifecycle, subscription semantics) rather than constraints subsystems must honor. The load-bearing content — that wires are declared per-capability and that subscription does not guarantee delivery — remains in BUS.md as part of the bus primitive specification.

New invariants added in v3: BIF-1, BIF-2, BIF-4, BIF-6; PRED-2, PRED-3, PRED-4, PRED-6, PRED-8; WIT-1, WIT-2, WIT-4, WIT-5, WIT-6; OBL-1, OBL-2, OBL-4, OBL-5; SIG-3, SIG-6, SIG-7, SIG-8, SIG-9, SIG-10, SIG-11, SIG-12; STEP-1, STEP-4, STEP-6, STEP-7, STEP-8, STEP-9; DISP-1, DISP-3, DISP-5, DISP-6; ARCH-1, ARCH-2, ARCH-4; EXC-1 through EXC-3 (elevated from prose).

Total: 53 numbered invariants across 9 categories (up from 12 flat + 3 carve-outs + 5 catalog clauses in v2).

---

## References

- [`CONCEPTS.md`](./CONCEPTS.md) — vocabulary and decompositions.
- [`object_model_0417.md`](./object_model_0417.md) §7 (bindings, obligations), §8.1.1 (bifurcation).
- [`LIVENESS.md`](./LIVENESS.md) §3 (projection catalog).
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) §2 (module layout), §3 (mutation discipline).
- [`ADR-resolution-half.md`](./ADR-resolution-half.md) §14 (reactor).
