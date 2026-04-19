# Architectural Concepts

**Status.** v1 (2026-04-19).

**Purpose.** The master vocabulary document for txKernel's object and execution model. Defines the terms, decompositions, and primitives that all other architecture documents reference. Read this first.

**Audience.** Designers, reviewers, agents. Consulted before writing subsystem specs, reviewing proposals, or reasoning about where a mechanism belongs.

**Relationship to other documents.**

- `INVARIANTS.md` states enforceable rules using the vocabulary defined here.
- `object_model_0417.md` provides the underlying entity model (identity, payload, bindings, evidence, obligations).
- `LIVENESS.md` catalogs per-subsystem projections using the semantic plane's vocabulary.
- Subsystem specs (VFS, process, futex, etc.) apply this vocabulary to their specific domains.

---

## 1. The two basis claims

The architecture rests on two claims from v11, which this document extends but does not replace:

**Resolution half.** A kernel maintains `(signifier, consistency, binding)` triples. Signifiers reach entities through bindings in namespace containers.

**Lifecycle half.** Every entity decomposes into `(identity, capability, payload)` layers. Identity is what the entity *is*; capability is what operations it supports; payload is its operational resources.

Everything below elaborates these two claims into a working architecture.

---

## 2. The three planes

The architecture decomposes into three interacting planes plus one structural constraint.

### 2.1 Semantic plane

**Defines truth.** Pure, guard-scoped, referentially transparent within an epoch.

- **Predicate.** Pure function `p(obj, ctx) → bool` over entity state and context. Defines what it means for a projection to hold.
- **Require.** The only authorized consumer of predicates at entry. Invokes predicates under an epoch guard; produces a witness on success; returns errno on failure.
- **Witness.** Proof that preconditions were satisfied at a point in time. Carries `IdentRef<'g, T>` values; `!Send + !Sync` by construction; lifetime-bound to the producing epoch guard.

### 2.2 Execution plane

**Makes transitions real.** Mutating, linearized, substrate-disciplined.

- **Step.** The execution primitive. A bounded synchronous unit of work that observes state, may mutate, and reports outcome. See §5.
- **Commit discipline.** A three-part internal pattern within a step that performs mutation: observe under guard, upgrade observations to retention, mutate via substrate primitives. Commit is not a phase; it is a classification of mutations within a step.
- **Obligation.** What a binding's commit makes true. Declared on binding type; determines evidence retention required at commit time.
- **Substrate primitives.** Observer-safe mutation primitives (`zone::sign`, `index::commit`, `index::withdraw_commit`, `index::swap_commit`, `credit::commit`) that enforce linearization at their call site.

### 2.3 Publication plane

**Makes waiting efficient.** Commit-disciplined, per-carrier-ordered, never authoritative.

- **Signal.** Commit-disciplined publication of selected transitions. Fires from within a mutating step, after the corresponding state write. Hints only.
- **Wire.** A bus carrier. Level-triggered (RawQueue) or edge-triggered (RawPort) or passive (RawTrace).
- **Signal attachment.** A per-subsystem declaration that a specific transition publishes on a specific wire. Selected, not automatic. Cataloged separately from projections.
- **Waiter contract.** Wake → re-observe via predicate or step → act. Signals never authorize action; fresh observation does.

### 2.4 Structural constraint: bifurcation

Per `object_model §8.1.1`. Entities whose partial order admits `structural ⟂ payload` (zombies, unlinked-but-open files, detached mounts, closed-but-queued sockets) factor into two zone-allocated types: `<n>Identity` and `<n>Payload`. Each zone reclaims independently.

**Signal attachment rule.** Each signal attachment names exactly one carrier slot (Identity or Payload), never a cross-layer composite. Cross-layer coordination requires explicit pairing, not shared wires.

---

## 3. The reference hierarchy

Four reference types form a hierarchy; each downgrade is free, each upgrade is fallible.

```
Weak<T>            — nullable, epoch-independent
    ↓ observe under epoch
IdentRef<'g, T>    — epoch-guarded, stack-bound, no retention
    ↓ pin under epoch (SENTINEL_DEAD CAS)
Cap<T>             — refcounted identity retention, 'static
    ↓ upgrade payload (payload-counter CAS)
T::OperationalEvidence — pins payload, entails Cap<T>, 'static
```

`'static` forms (Cap, OperationalEvidence) may cross thread boundaries, be held in struct fields, and survive across `.await`. `'g`-forms (IdentRef, witnesses) may not.

---

## 4. Projections and monotonicity

Every entity has at least three independent projections:

- **Structural.** Retention > 0; entity is semantically alive.
- **Namespace / addressability.** Reachable through a chain of bindings from a namespace root.
- **Payload.** Operational state is available; for indirected entities, payload box exists; for compound-predicate entities, disjunction of typed contributions.

**Monotonicity.** Each projection transitions only from true to false, never back. This is the load-bearing property for race-degradation (§7.3) and waiter correctness.

Subsystem projection catalogs (in `LIVENESS.md`) enumerate projections per entity type and state the realization mechanism (identity retention / binding-chain reachability / payload retention / compound disjunction).

---

## 5. The step model

**The execution primitive is `step`, not `commit`.** Every kernel operation is a sequence of bounded synchronous steps.

### 5.1 Step outcome

```rust
enum StepOutcome<T> {
    Advanced(Progress),                            // made progress, may step again
    Blocked(Channel, Mask),                        // no progress, wait on channel
    AdvancedThenBlocked(Progress, Channel, Mask),  // made progress, then stalled
    Done(T),                                       // terminal with value
    Err(Errno),                                    // terminal with error
}
```

### 5.2 Step contract

A step:

- Runs **synchronously.** No `.await`, no future construction inside. If a step cannot proceed synchronously, it returns `Blocked` (or `AdvancedThenBlocked`).
- Is **bounded.** Completes in upper-bounded time. Work that exceeds the bound splits across multiple steps.
- **Observes under a fresh epoch guard.** Acquires its own guard; releases at step end.
- **May mutate** via substrate primitives, following the three-part in-step commit discipline.
- **May fire signals** via the temporal bus, after each linearizing write (I-7).
- **Returns one outcome.** Never partial, never ambiguous.

### 5.3 Witness scope

**Witnesses do not cross step boundaries.** A witness produced by `require_*` within a step is valid only for that step's synchronous execution. Between steps, the epoch guard is released; any witness is treated as expired.

Cross-step continuation carries `'static` retention evidence (Cap, OperationalEvidence), not witnesses. On resume, the next step downgrades Caps to IdentRef under a fresh guard and re-runs `require_*` to produce a fresh witness.

### 5.4 Monotone progress under retry

**Operation progress is monotone across steps.** No step undoes a previous step's published progress. A step that returns `Advanced(n)` has committed `n` units of progress that subsequent steps cannot retract.

This is the generalization of projection monotonicity to operation progression. It ensures that `Blocked → wait → retry` is safe: the retry begins from the progress state the previous step left, never from an earlier state.

### 5.5 Trajectories

Operations exhibit characteristic trajectories:

- **One-step operations.** `open`, `unlink`, `fork`, `dup`. Step returns `Done(value)` or `Err(errno)` immediately. No intermediate outcomes. Commits atomically as its single action.
- **Multi-step operations.** `read`, `write`, `splice`, `sendfile`, `copy_file_range`. Step returns `Advanced`, `Blocked`, or `AdvancedThenBlocked` repeatedly until terminating with `Done` or `Err`. Progress accumulates across steps.

The distinction is a property of the operation, not a separate pattern. Both are step functions; they differ only in which outcomes they produce.

---

## 6. The upper/lower halves

Steps are synchronous; operations are asynchronous. The division is formal:

**Lower half (per-step).** Synchronous, bounded, mutation-capable under epoch guard. Produces `StepOutcome` synchronously. No future creation, no awaiting, no reactor awareness.

**Upper half (per-operation).** Asynchronous composition of step outcomes with wait futures. Produces the operation's overall future. Owned by drivers (§9.1).

**Executor (reactor).** Runs the composed future. Handles temporal services (sleep, timeout, signal delivery). Does not see step functions directly.

Each layer has a single concurrency responsibility. Lower half is worker code. Upper half is composition code. Executor is scheduling code.

---

## 7. Bindings, obligations, reachability

### 7.1 Bindings

A **binding** is a structural slot in a container:

```
Binding<K, Obligation> {
    key:      K,
    evidence: Evidence<Obligation>,
}
```

Bindings have no independent lifetime; existence is governed by the container.

### 7.2 Obligations

Every binding declares one of three obligations:

- **Resolution-only.** Evidence: `Weak<T>`. No retention promise.
- **Addressability.** Evidence: `Cap<T>`. Target remains structurally alive.
- **Operational.** Evidence: `T::OperationalEvidence`. Payload projection holds.

**Publication is not part of obligation typing.** Whether a binding-install/withdraw fires a signal is declared separately in the signal attachment catalog, not by the binding's obligation.

### 7.3 Race degradation

Under monotonicity, container-linearized binding mutation, and SENTINEL_DEAD-guarded upgrade, concurrent mutation may cause operations to fail but cannot cause them to succeed against a different entity, a different context, or with different semantics. **Races degrade to failure, not to silent incorrect success.**

---

## 8. The bus: static and temporal layers

The bus splits into two layers with distinct responsibilities.

### 8.1 Static bus

**What wires exist and who subscribes.** Compile-time-rooted declarations; type-checked subscriptions; no temporal behavior.

- **Wire declarations.** `readiness!`, `lifecycle!`, `tracepoint!` macros declare what a capability-type publishes. Fixed at type definition.
- **Subscription graph.** Who has registered interest on which wires. Queried by epoll at `EPOLL_CTL_ADD`; queried by wait primitive at step `Blocked` outcome.
- **Type-level safety.** Firing a mask bit not in the declared set is a compile error.

### 8.2 Temporal bus

**Runtime fire-and-wake.** The hot path; minimal.

- **Fire.** Called from within a step's mutating portion, after the state write. Delivers wakes to subscribers on the affected wire.
- **Wake delivery.** Per-carrier ordered (I-6, I-7). Wake recipients re-enter their drivers, which invoke the next step.
- **Diagnostic peek.** Read-only wire inspection for debugging. Never authoritative (I-2).

### 8.3 Three primitives

```
RawQueue    — level-triggered readiness, multi-subscriber, non-consuming
RawPort     — edge-triggered lifecycle events, possibly consuming
RawTrace    — passive diagnostic recording, no waiters
```

These are the complete set. Mechanisms requiring structured waitqueues (priority ordering, ownership tracking, counting) are subsystems, not bus primitives.

---

## 9. The wait primitive

**One loop, parameterized by (channel, condition, protocol).**

```
loop:
    check condition under fresh guard
    if ready: return ConditionTrue
    register waiter on channel
    sleep per protocol
    on wake: (loop: recheck)
    on interrupt: return classified outcome
```

### 9.1 The three modes

All operations involving potential waiting fall into three modes:

- **Nonblocking.** Call step once. If `Blocked`, return EAGAIN. No wait primitive invocation.
- **Waiting.** Call step in a loop; on `Blocked`, invoke wait primitive with `WaitProtocol`; resume. Covers both blocking and timed variants via protocol enum.
- **Selecting.** Register on channel without stepping. Used by epoll; returns fired-channel information to caller for further action.

### 9.2 WaitProtocol

Closed enum of named protocols:

```
Uninterruptible
Interruptible
Killable
InterruptibleTimeout(Duration)
KillableTimeout(Duration)
```

Each protocol specifies task state, interrupt policy, and timeout behavior. The set is closed; extension is an architectural decision.

### 9.3 Wait outcomes

```
WaitOutcome ::= ConditionTrue | Interrupted | Killed | TimedOut
```

Subsystems translate outcomes to POSIX errno per their operation's semantics (EINTR, ETIMEDOUT, partial-progress return, etc.).

### 9.4 The retry-is-recheck identity

The driver's retry loop and the wait primitive's recheck loop are the same loop. Calling `step()` after a wake *is* the re-evaluation of the condition; there is no separate retry mechanism. This is structural, not a discipline the driver enforces.

### 9.5 Specialized protocol objects

Generic wait_event takes (channel, condition, protocol) as parameters. Specialized protocol objects (e.g., `Completion`) ossify (condition, channel) into a named type for recurring patterns. Completion is the degenerate wait where condition is "done counter ≥ 1" and channel is the completion's internal queue. Specialized objects are justified when the pair recurs, the condition is simple and fixed, and type-level guarantees beat documented convention.

---

## 10. Middleware as protocol combinator

**Middleware is a higher-order protocol combinator.** It takes caller-supplied semantic operations or predicates as parameters and supplies execution protocol as implementation.

Protocol combinators:

- **Wait adapters.** `wait_event(channel, condition, protocol, ctx) → WaitOutcome`.
- **Drivers.** `drive_*(step_fn, wait_protocol, ctx) → Result<T, Errno>`. Three modes (§9.1).
- **Future combinators.** `with_timeout(fut, deadline)`, `with_cancel(fut, token)`.
- **Specialized protocol objects.** `Completion`, and others if added.

Not middleware (hard-coded policy or fixed mechanism):

- Observers, interceptors, gates (§11).
- Subsystem step functions.
- Bus primitives (fire, subscribe).
- Subsystem commit publications (fire from steps).

Protocol catalogs are closed. Extension requires architectural review.

---

## 11. Dispatch classes

The dispatch layer decomposes into five disjoint classes, each with a distinct control-flow role.

### 11.1 Observe

Passive boundary observation. Hard-coded behavior: record, publish, count. No blocking, no control transfer, no admission decisions.

Examples: syscall tracepoints (enter/exit), audit subsystem publication, credential snapshot prelude, signal postlude check.

### 11.2 Intercept

Boundary control transfer with resumption. May park the thread; may transfer control to an external observer (tracer); requires recovery protocol.

Examples: ptrace entry/exit stops. The intercept class is strictly for control-transfer protocols, not for observation.

### 11.3 Gate

Universal admission decisions at the syscall boundary. Evaluates policy predicates over syscall metadata; admits or rejects. If rejected, the syscall never reaches subsystem execution.

Examples: seccomp BPF filters, LSM syscall-boundary hooks. Gates operate on syscall metadata only — not on subsystem-internal state.

### 11.4 Wait-adapt

The wait primitive (§9). Composes channel, condition, protocol into a blocking operation. Consumed by the waiting mode of drivers.

### 11.5 Drive

The three modes (§9.1) that compose step functions with wait futures into operation futures. Drivers are state-blind: they see step outcomes, wake events, and thread context; they never inspect subsystem-internal state.

### 11.6 Disjointness

The five classes are disjoint. No class mechanism implements another class's decisions: observers do not admit; gates do not park; interceptors do not admit; wait adapters do not admit; drivers do not admit or transfer control. Subsystem-internal publications are not dispatch hooks; they fire through bus primitives from step functions under I-7 discipline.

---

## 12. The three orthogonal decompositions

Every mechanism in the architecture has coordinates on three orthogonal axes. A mechanism that doesn't fit cleanly on all three is either misclassified or genuinely novel.

**Axis 1: Plane.** Semantic / execution / publication. Answers: what kind of information does this mechanism operate on?

**Axis 2: Dispatch class.** Observe / intercept / gate / wait-adapt / drive, or "subsystem-internal" for mechanisms not at the dispatch boundary. Answers: what control-flow role does this play?

**Axis 3: Middleware vs fixed.** Protocol combinator (parameterized over semantics) or hard-coded (fixed behavior) or semantics carrier (consumed by combinators). Answers: is this parameterized or specialized?

The three axes together place any mechanism. Examples:

- `wait_event` — publication plane (consumes hints), wait-adapt class, middleware.
- `seccomp filter` — semantic plane (evaluates policy predicate), gate class, fixed.
- `pipe write_commit signal fire` — publication plane, subsystem-internal (not dispatch), fixed.
- `Completion` — publication plane, wait-adapt class, middleware (specialized protocol object).
- `ptrace entry stop` — execution plane (pauses control flow), intercept class, fixed.

---

## 13. Carve-outs

Three notification mechanisms are **not** publication-plane signals and must not be implemented through bus primitives:

- **Fault injection** (SIGSEGV, SIGFPE, SIGBUS, SIGILL). Thread-local AST at trap-return. Specified by the signal subsystem.
- **Cross-core barriers** (TLB shootdown, IPI sync). Synchronous reactor coordination; issuing thread blocks on ack. Specified by the reactor's sync-coordination API.
- **Single-waiter handoff.** Subsystem-owned waitqueue with one-waiter protocol. Not a publication-catalog mechanism.

These are documented here so they are not accidentally forced through the bus model.

---

## 14. The closed catalogs

Two catalogs are closed by framework constraint; extension requires revisiting invariants.

### 14.1 Step outcomes

```
Advanced(Progress)
Blocked(Channel, Mask)
AdvancedThenBlocked(Progress, Channel, Mask)
Done(T)
Err(Errno)
```

Five variants. No sixth. Extension would require a new execution-primitive concept.

### 14.2 Driver modes

```
nonblocking     — step once; Blocked → EAGAIN
waiting         — loop step with WaitProtocol variant
selectable      — register without stepping; epoll-shape
```

Three modes. Blocking and timed are WaitProtocol variants of the waiting mode, not separate modes.

### 14.3 Wait protocols

```
Uninterruptible
Interruptible
Killable
InterruptibleTimeout(Duration)
KillableTimeout(Duration)
```

Closed enum. Addition is an architectural decision.

### 14.4 Bus primitives

```
RawQueue, RawPort, RawTrace
```

Three primitives. Mechanisms requiring structured waitqueues are subsystems, not bus primitives.

### 14.5 Dispatch classes

```
observe, intercept, gate, wait-adapt, drive
```

Five classes. Disjoint. Adding a sixth requires architectural review.

---

## 15. The stacked one-liner

The model in one sentence, revised for the step primitive:

> **Predicates define truth. Steps advance state monotonically under retry. Drivers compose steps with wait primitives into operations. Signals make waiting efficient. Only fresh observation authorizes action.**

Five roles, in causal order:

- **Predicates** define truth (semantic plane).
- **Steps** advance state monotonically (execution plane).
- **Drivers** compose (upper half).
- **Signals** publish hints (publication plane).
- **Fresh observation** authorizes action (waiter contract; re-run of step's require under fresh guard).

---

## 16. Vocabulary summary table

For quick reference:

| Term | Plane | Role |
|---|---|---|
| Predicate | Semantic | Defines truth |
| Require | Semantic | Entry-point consumer, produces witness |
| Witness | Semantic | Proof bound to epoch guard, one-step scope |
| Projection | Semantic | Monotone predicate over entity state |
| Obligation | Execution | Binding-declared commit responsibility (evidence only) |
| Step | Execution | Bounded synchronous execution primitive |
| StepOutcome | Execution | Five-variant outcome enum |
| Progress monotonicity | Execution | No step undoes published progress |
| Commit discipline | Execution | Three-part mutation pattern within a step |
| Substrate primitive | Execution | Observer-safe mutation linearization |
| Signal | Publication | Commit-disciplined hint after linearizing write |
| Wire | Publication | Bus carrier (RawQueue/RawPort/RawTrace) |
| Signal attachment | Publication | Per-transition opt-in publication declaration |
| Waiter contract | Publication | Wake → re-observe → act |
| Bifurcation | Structural | Identity/Payload split per §8.1.1 |
| Cap<T> | Reference | Identity retention, 'static |
| IdentRef<'g, T> | Reference | Epoch-guarded observation, bounded lifetime |
| OperationalEvidence | Reference | Payload retention, entails Cap |
| Binding | Structural | Key+evidence slot in a container |
| Reachability | Structural | Binding-chain traversal |
| Driver | Dispatch | Upper-half composition of steps + waits |
| Wait primitive | Dispatch | check → register → sleep → wake → recheck |
| Wait protocol | Dispatch | Closed enum of task-state + interrupt policy |
| Observer / Interceptor / Gate | Dispatch | Three boundary classes (non-middleware) |
| Middleware | Composition | Higher-order protocol combinator |

---

## 17. What this document does not cover

- **Per-subsystem projection catalogs.** See `LIVENESS.md`.
- **Per-subsystem signal attachments.** See `SIGNAL_ATTACHMENTS.md` (forthcoming).
- **Enforceable rules.** See `INVARIANTS.md`.
- **Entity factoring details.** See `object_model_0417.md §8.1.1`.
- **Module layout, import rules, lints.** See `SUBSYSTEM_ANATOMY.md`.
- **Reactor internals, HAL interfaces.** Out of scope for concepts; see reactor and HAL specs.
- **Specific syscall implementations.** Out of scope; see per-subsystem specs.

---

## References

- `object_model_0417.md` — entities, references, bindings, obligations, evidence.
- `INVARIANTS.md` — enforceable rules using this vocabulary.
- `LIVENESS.md` — projection catalog and predicate framework.
- `SUBSYSTEM_ANATOMY.md` — four-module layout and mutation discipline.
- `ADR-resolution-half.md` — reactor role, informational vs operational I/O.
