# The Step Model

**Status.** v1 (2026-04-19).

**Purpose.** Specify the step primitive: the synchronous, bounded unit of execution that replaces the prepare/commit phase pattern as txKernel's operation-building primitive. Define the step outcome algebra, the in-step commit discipline, witness scope in practice, and how steps compose with drivers and the wait primitive to form operations.

**Audience.** Subsystem authors writing execution code, reviewers evaluating step-function designs, lint-infrastructure authors encoding STEP-* invariants.

**Companion documents.**

- [`INVARIANTS.md`](./INVARIANTS.md) — STEP-* rules this document implements; PRED-*, WIT-*, OBL-*, SIG-* rules that steps must honor.
- [`CONCEPTS.md`](./CONCEPTS.md) — vocabulary (step, step outcome, commit discipline, upper/lower halves).
- [`object_model_0417.md`](./object_model_0417.md) — reference hierarchy, reclamation, bindings.
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) — module layout within which step functions live.
- [`BUS.md`](./BUS.md) — bus primitive APIs invoked from step commits. *(forthcoming)*

---

## 1. The primitive

A **step** is a synchronous, bounded, outcome-returning unit of work that a subsystem exposes for dispatch-layer invocation. Each step:

- Acquires its own epoch guard on entry and releases it on return.
- Consults predicates through require, producing witnesses scoped to the step.
- May upgrade witness observations to retention evidence for mutation.
- May mutate subsystem state via substrate primitives.
- May publish transitions via bus primitives.
- Returns exactly one outcome from the closed outcome algebra.

Steps are the subsystem's execution surface. Drivers in the dispatch layer invoke steps and compose their outcomes; reactors run the driver's composed future; but neither driver nor reactor calls into subsystem state directly — only step functions do.

---

## 2. The outcome algebra

Steps return values from a closed five-variant algebra. The specific enumeration is implementation-level detail; the structural distinctions are load-bearing (STEP-1).

```
StepOutcome<T> :=
    Advanced(Progress)
    Blocked(WakeCarrier, InterestConditions)
    AdvancedThenBlocked(Progress, WakeCarrier, InterestConditions)
    Done(T)
    Err(Errno)
```

Each variant expresses a distinct post-step situation:

| Variant | Meaning | Driver action |
|---|---|---|
| `Advanced(p)` | Made progress `p`; more progress may be possible without waiting | Accumulate `p`, re-invoke step |
| `Blocked(c, m)` | No progress possible; wait on carrier `c` with interests `m` | Compose wait future; on wake, re-invoke step |
| `AdvancedThenBlocked(p, c, m)` | Made progress `p`, then stalled; further progress requires waiting | Accumulate `p`, compose wait future; on wake, re-invoke step |
| `Done(v)` | Operation terminates with value `v` | Return `v` to caller |
| `Err(e)` | Operation terminates with error `e` | Return `e` to caller |

### 2.1 Why five variants

The algebra is the minimum that expresses every POSIX operation shape without ambiguity:

- **Advanced alone** is needed for operations that make bounded progress per call but aren't yet known to be blocked. A step may choose to yield after bounded work (STEP-2) even when more could be done immediately; the driver re-invokes.

- **Blocked alone** is needed for operations that cannot make any progress until a wake occurs. A `read` called on an empty pipe with no EOF signal returns `Blocked(pipe.read_wq, HasData | Broken)`.

- **AdvancedThenBlocked** is the composite case: a step moved some bytes and then observed the source go empty. Collapsing this into either pure Advanced (driver would re-call and discover Blocked on the next step) or pure Blocked (driver would lose the progress information) is lossy. The explicit composite reports both facts atomically, preserving POSIX partial-progress semantics (STEP-3 monotone progress applies to what was advanced; STEP-8 makes the advance externally visible; STEP-9 admits the composite as a peer outcome to pure Advanced and pure Blocked).

- **Done and Err** are terminal outcomes distinguishing success from failure. They do not carry progress (already accumulated by driver) or wake information (no further waiting).

A sixth variant (Deferred, Cancelled, Retried) would indicate either an implementation concern that belongs elsewhere or a framework-level extension requiring architectural review per ARCH-3.

### 2.2 Progress vocabulary

`Progress` is operation-specific. For byte-moving operations (read, write, splice) it is a byte count. For page-moving operations (mmap fault, copy_on_write) it is a page count. For enumeration-style operations (getdents) it is an entry count. The driver accumulates progress across steps by addition; the subsystem's step function determines the unit.

The only universal requirement is that progress is **additive and monotone** (STEP-3): no step undoes a previous step's reported progress.

### 2.3 WakeCarrier and InterestConditions

A `WakeCarrier` names a bus-level wake channel (RawQueue or RawPort) whose fire delivers a wake to subscribers. `InterestConditions` specify which fires on the carrier signify "worth trying again" — typically a mask of readiness bits or an event filter.

The carrier is subsystem-owned. A pipe read's carrier is `pipe.read_wq`; a socket write's carrier is `socket.send_wq`; a futex wait's carrier is the futex hash bucket. The step function knows which carrier governs its progress because it knows the subsystem's wake-publication discipline.

The driver treats the carrier opaquely (DISP-2): it does not inspect what the carrier refers to, only composes a wait future that parks on the carrier with the interest mask.

---

## 3. The in-step commit discipline

When a step mutates state, it follows a fixed four-phase internal discipline (STEP-4):

```
1. Observe      — require produces witnesses under epoch guard (WIT-*)
2. Upgrade      — IdentRef → Cap / PayloadCap / typed contribution (OBL-4)
3. Mutate       — substrate primitives install/remove/modify bindings
4. Publish      — bus primitives fire signals for attached transitions (SIG-4)
```

Each phase has a specific role and constraints. Skipping or reordering is a STEP-4 violation.

### 3.1 Observe

The step acquires an epoch guard as its first action. It invokes require functions to produce witnesses bearing guard-scoped observation evidence. Require evaluates the relevant predicates under the guard; if any predicate fails, require returns the appropriate errno and the step returns `Err` without further action.

Observation is side-effect-free (PRED-1). No mutation, allocation, I/O, or blocking happens during observation. The guard protects the observed entities from physical reclamation for the step's duration.

### 3.2 Upgrade

For bindings the step intends to install or modify, observations must be promoted to retention evidence matching each binding's declared obligation (OBL-4). The upgrade is a fallible CAS against the reclamation sentinel:

- Addressability bindings require `Cap<T>` — upgrade succeeds unless the entity has been marked dead.
- Operational bindings require `T::OperationalEvidence` — upgrade succeeds unless the payload has been released.

If any upgrade fails, the step releases all already-acquired reservations and returns `Err` (typically `ESTALE` or the appropriate errno for "target gone during resolution"). Partial upgrade is not allowed; either all required evidence is held, or none is.

### 3.3 Mutate

With evidence in hand, mutations apply through **substrate primitives** (`zone::sign`, `index::commit`, `index::withdraw_commit`, `index::swap_commit`, `credit::commit`). These primitives enforce observer-safety: concurrent walkers see either the pre-mutation or post-mutation state, never an intermediate.

Mutations are linearized at each substrate primitive's atomic transition. Multiple mutations within one step are sequenced by the step's code; each is its own linearization point, each observable independently (per-carrier ordering, SIG-5). Compound mutations that must be jointly atomic use `swap_commit` or similar substrate primitives that bundle the transition.

Direct field writes that bypass substrate primitives are STEP-4 violations and compromise the observer-safety guarantee.

### 3.4 Publish

After each linearizing write, the step fires signal attachments declared for that transition (SIG-4). The fire happens on the same carrier as the transition, within the same step's synchronous execution, after the write is visible to observers.

Not every mutation publishes (SIG-3). Only transitions with external subscribers need signal attachments. The attachment catalog (SIGNAL_ATTACHMENTS.md) records which transitions fire which wires.

Publication uses bus primitives directly (SIG-6). Substrate primitives do not fire signals; the subsystem's step code does, as the step's final action before returning.

### 3.5 Return

The step returns a step outcome reflecting what happened. A mutating step that completes its work returns `Done(value)`. A mutating step that made progress but could not finish returns `Advanced(progress)` or `AdvancedThenBlocked(progress, carrier, interests)`. A step that tried to mutate but hit a reclamation sentinel on upgrade returns `Err(errno)`.

Non-mutating steps skip phases 2, 3, and 4 and return outcomes based on observation alone. A step consulting a predicate to answer `fstat` observes, formats the result into `Done(stat_result)`, and returns.

---

## 4. Witness scope in practice

The witness-scope rules (WIT-3, WIT-4, WIT-5) are strict: witnesses must not cross step boundaries. In practice this means:

- Witnesses are local variables within the step function body.
- Witnesses must not be stored in `self`-fields, returned in `StepOutcome`, passed to other threads, or held across `.await` points (there are no `.await` points within a step per STEP-2, but the rule also forbids such structure).
- Any operation that requires cross-step continuation must carry retention evidence (`Cap<T>` or `T::OperationalEvidence`), not witnesses.

### 4.1 Resumption pattern

A multi-step operation that resumes after blocking follows this pattern:

```
step N:
    acquire guard
    require_X(...) → WitnessX { ref: IdentRef<'g, X> }
    ... do work ...
    if need to continue and can't now:
        extract Cap<X> from witness (upgrade to 'static)
        store Cap in driver-visible resume state
        return Blocked(carrier, interests)
    [guard released on return]

driver:
    parks on carrier, waits for wake

step N+1:
    acquire fresh guard (new 'g')
    downgrade Cap<X> → IdentRef<'g', X> under fresh guard
    require_X(...) again → fresh WitnessX (under new 'g')
    ... continue work ...
```

The `Cap<X>` carries forward because it is `'static` (OBL-2, object_model §4); it keeps the target identity alive across the blocking interval. On resumption, the fresh guard gives a new `'g` lifetime, and require re-runs to produce a fresh witness under the new guard.

The re-require is not bookkeeping — it is the anti-TOCTOU re-check (PRED-7). Between the Cap's acquisition and the resumption, other operations may have executed; the entity's projections may have changed. The fresh require evaluates predicates against current state, potentially returning a different errno than the initial require.

### 4.2 What the driver holds

The driver retains whatever `'static` values the subsystem hands it for resume purposes. Typically:

- An opaque `ResumeState` enum whose variants are subsystem-defined, carrying `Cap<T>`, `PayloadCap<T>`, or typed contributions, plus any per-operation accumulators (byte count, cursor position).
- The `WakeCarrier` from the last step's `Blocked` or `AdvancedThenBlocked` outcome.
- The accumulated `Progress` from all `Advanced*` outcomes so far.

None of this crosses into subsystem internals from the driver's perspective (DISP-2). The driver holds opaque handles; the next step downgrades and re-requires.

### 4.3 What the step receives on resumption

The step function's signature accepts resume state alongside its normal arguments. For a `read` operation:

```
pipe::step_read(
    ctx: &ThreadContext,
    handle: OpaqueHandle,       // identifies the pipe capability
    buf: &mut UserBuf,           // caller's buffer with accumulated advance
    resume: Option<&mut ReadResume>,  // subsystem-specific resume state
) -> StepOutcome<usize>
```

On first invocation, `resume` is `None`. On subsequent invocations after Blocked, it carries the subsystem's state from the previous step. The step either completes its work (returning `Done(total_read)`) or stalls again (returning `Blocked(carrier, mask)` and updating `resume`).

---

## 5. One-step operations

Operations with no progressive behavior — `open`, `unlink`, `mkdir`, `rmdir`, `rename`, `fork`, `dup`, `close`, `mmap` (reservation, not population), `setsockopt` — are steps that terminate in `Done` or `Err` on first call.

### 5.1 Example: `open`

Open resolves a path, performs authorization, and returns an fd. Its step:

```rust
pipe::step_open(ctx, path, flags, mode) -> StepOutcome<Fd>:
    [observe]
    guard = epoch::guard()
    path_witness = require_path_resolvable(path, ctx.cwd, &guard)?
    parent_witness = require_parent_exists(path_witness, &guard)?
    lookup_witness = require_lookup(parent_witness, path.name, flags, &guard)?
    perm_witness = require_open_perm(lookup_witness, flags, ctx.cred, &guard)?

    [upgrade]
    target_cap = upgrade_cap(lookup_witness.target())?    // Cap<RNode>

    [mutate]
    openfile_slot = zone::reserve::<OpenFile>()?
    fd_slot = ctx.fd_table.reserve_slot()?
    openfile = OpenFile { rnode: target_cap, offset: 0, flags, ... }
    zone::sign(openfile_slot, openfile)                  // publishes OpenFile
    index::commit(ctx.fd_table, fd_slot, Cap(openfile_slot))  // publishes fd

    [publish]
    // most opens publish nothing; epoll-watched dirs may fire on content change
    // but not on open itself

    [return]
    Done(Fd(fd_slot.as_raw()))
```

One guard, one require-chain, two coordinated substrate commits, no publications. If any require fails, the step returns `Err(errno)`. If either substrate reservation fails, all prior reservations drop cleanly (linear reservation types enforce this at the type level).

There is no `open_prepare` + `open_commit` split. The four-phase discipline happens within the single step. The "prepare" language in prior iterations corresponded to the observe+upgrade phases; "commit" to mutate+publish; the split into two functions was a v11/v12 convenience, not a framework requirement.

### 5.2 Example: `unlink`

Unlink withdraws a directory binding. One step:

```rust
vfs::step_unlink(ctx, path) -> StepOutcome<()>:
    [observe]
    guard = epoch::guard()
    resolve_witness = require_path_exists(path, ctx.cwd, &guard)?
    parent_witness = require_parent_writable(resolve_witness, ctx.cred, &guard)?
    target_witness = require_not_dir_or_empty(resolve_witness, &guard)?

    [upgrade]
    parent_cap = upgrade_cap(parent_witness.parent())?
    binding_cap = upgrade_cap(resolve_witness.binding())?

    [mutate]
    index::withdraw_commit(parent_cap.children, binding_cap.key)
    // Drop of binding_cap's evidence decrements LinkPin on RNode;
    // if nlinks→0 and no open refs, triggers reclaim.

    [publish]
    // fsnotify subscribers observe unlink through a RawPort on the parent:
    parent_cap.fsnotify.fire(FsEvent::Unlink { name: path.last_component() })

    [return]
    Done(())
```

One substrate commit (withdraw), one publication (fsnotify). The target RNode's reclamation cascade happens through evidence-drop semantics per object_model §6, not through explicit code in this step.

### 5.3 Example: `fork`

Fork creates a new process by duplicating the current one. Despite its complexity (must atomically publish new process and thread identities plus payload), it is a single step:

```
process::step_fork(ctx, flags) -> StepOutcome<Pid>:
    [observe]
    guard = epoch::guard()
    parent_witness = require_fork_permitted(ctx, flags, &guard)?
    clone_witness = require_clone_resources(ctx, flags, &guard)?

    [upgrade]
    parent_cap = upgrade_cap(parent_witness.process())?

    [mutate]
    child_identity_slot = zone::reserve::<ProcessIdentity>()?
    child_payload_slot = zone::reserve::<ProcessPayload>()?
    thread_identity_slot = zone::reserve::<ThreadIdentity>()?
    thread_payload_slot = zone::reserve::<ThreadPayload>()?

    child_payload = clone_payload_per_flags(&parent_cap, flags, clone_witness)
    child_identity = new_identity(child_pid, parent_cap, payload_cap)
    // ... similarly for thread

    zone::sign(child_payload_slot, child_payload)
    zone::sign(thread_payload_slot, thread_payload)
    zone::sign(child_identity_slot, child_identity)
    zone::sign(thread_identity_slot, thread_identity)

    index::commit(parent_cap.children, child_pid, Cap(child_identity_slot))
    index::commit(ctx.pid_ns.pid_hash, child_pid, Cap(child_identity_slot))

    [publish]
    // waitpid subscribers on parent observe new child (no fire; addition is polled)
    // ptrace subscribers may observe fork event:
    if parent_cap.traced() {
        parent_cap.ptrace_port.fire(PtraceEvent::ForkChild { child_pid })
    }

    [return]
    Done(Pid(child_pid))
```

Multiple substrate commits (four zone signs, two index commits) within one step, each a linearization point. Substrate commit primitives provide observer-safety: concurrent walkers on `parent_cap.children` see either no child or the fully-formed child, never a half-formed state. The sign-then-commit ordering ensures that when the child appears in a namespace, its identity slot is already fully initialized.

Fork is never a multi-step operation — it cannot be, because no intermediate state could safely be observed by other parties. The entire fork is a bounded synchronous step.

### 5.4 When to one-step

A step that completes in its first call — returning `Done` or `Err` — is a one-step operation. Characteristics:

- No inherent partial-progress notion. Open either gives an fd or doesn't; there's no "partially opened."
- No wait dependency. Open does not need to wait on kernel state to transition.
- Work is bounded by the step-bound constraint (STEP-2). If bounded work completes the operation, one step suffices.

Most creation, removal, and atomic-reconfiguration operations are one-step. Most data-plane operations are not.

---

## 6. Multi-step operations

Operations with progressive behavior — `read`, `write`, `pread`, `pwrite`, `splice`, `sendfile`, `copy_file_range`, `getdents`, `readv`, `writev` — are steps that return `Advanced*` outcomes across multiple invocations.

### 6.1 Example: `read` on a pipe

A pipe read drains the ring until the user buffer is full or the ring is empty:

```
pipe::step_read(ctx, handle, buf, resume) -> StepOutcome<usize>:
    [observe]
    guard = epoch::guard()
    pipe_witness = require_pipe_readable(handle, &guard)?
    buf_witness = require_user_buf_writable(buf, ctx.space, &guard)?

    pipe = pipe_witness.ident();
    remaining = buf.remaining();

    [non-mutating observation of pipe state]
    available = pipe.ring.available_read();
    writers_closed = pipe.writers_closed();

    if available > 0 {
        [upgrade — briefly; for reading, we just need Cap]
        pipe_cap = upgrade_cap(pipe_witness)?
        //   (only upgrade if we mutate; but read mutates the cursor)

        [mutate — consume from ring]
        n = min(available, remaining);
        pipe.ring.pop_into(&mut buf.slice_mut(n));   // substrate-protected ring op
        pipe.ring.advance_read(n);                   // substrate index::commit

        [publish — writers may be waiting for space]
        pipe.write_wq.fire(WriteReadiness::Space);   // opt-in per SIG-3

        if buf.remaining() == 0 {
            return Done(buf.filled());
        }
        // more buffer, but ring now empty:
        if pipe.ring.available_read() == 0 && !writers_closed {
            return AdvancedThenBlocked(
                Progress(n),
                pipe.read_wq,
                ReadInterests::HasData | ReadInterests::Broken
            );
        }
        // more buffer and writers closed → can drain remainder next step,
        // or EOF if truly empty:
        if writers_closed {
            return Done(buf.filled());  // EOF via short read
        }
        return Advanced(Progress(n));
    }

    // ring empty
    if writers_closed {
        return Done(buf.filled());  // typically filled()==0; EOF
    }

    return Blocked(
        pipe.read_wq,
        ReadInterests::HasData | ReadInterests::Broken
    );
```

The four outcomes `Advanced`, `Blocked`, `AdvancedThenBlocked`, `Done` all occur in this single step function depending on ring state and buffer state. The driver composes them:

```
driver (drive_blocking):
    loop {
        match pipe::step_read(ctx, handle, &mut buf, resume.as_mut()) {
            Advanced(p) => { buf.advance(p.0); /* loop */ }
            Blocked(c, m) => {
                wait::wait_event(c, m, Interruptible, ctx).await?;
                /* loop */
            }
            AdvancedThenBlocked(p, c, m) => {
                buf.advance(p.0);
                wait::wait_event(c, m, Interruptible, ctx).await?;
                /* loop */
            }
            Done(total) => return Ok(total),
            Err(e) => return Err(e),
        }
    }
```

The driver's loop is small, subsystem-agnostic (DISP-2), and handles the five outcomes uniformly. This is the realization of the retry-is-recheck identity: each loop iteration's step invocation is itself the re-observation for the previous iteration's wake.

### 6.2 Example: `splice`

Splice moves data between two capabilities without a user-space buffer. Its step must observe both source and sink:

```
splice::step_move(ctx, src_handle, sink_handle, count, resume) -> StepOutcome<usize>:
    guard = epoch::guard()

    src_witness = require_readable(src_handle, &guard)?
    sink_witness = require_writable(sink_handle, &guard)?

    src = src_witness.ident();
    sink = sink_witness.ident();

    src_available = src.readable_bytes();
    sink_space = sink.writable_bytes();
    remaining = count - resume.accumulated();

    if src_available == 0 && !src.eof() {
        return Blocked(src.read_wq, ReadInterests::HasData | ReadInterests::Broken);
    }
    if src.eof() {
        return Done(resume.accumulated());
    }
    if sink_space == 0 && !sink.closed() {
        return Blocked(sink.write_wq, WriteInterests::Space | WriteInterests::Broken);
    }
    if sink.closed() {
        return Err(EPIPE);
    }

    // both have something; move a bounded chunk
    chunk = min(src_available, sink_space, remaining, STEP_CHUNK_LIMIT);
    [upgrade src + sink caps]
    [mutate: move chunk bytes from src to sink via substrate]
    [publish: src write_wq (space freed), sink read_wq (data available)]

    advanced = Progress(chunk);

    if resume.accumulated() + chunk >= count {
        return Done(resume.accumulated() + chunk);
    }

    // more to move; check if we can continue immediately
    if src.readable_bytes() > 0 && sink.writable_bytes() > 0 {
        return Advanced(advanced);
    }

    // progressed but now stalled on one side
    if src.readable_bytes() == 0 {
        return AdvancedThenBlocked(advanced, src.read_wq, ReadInterests::HasData);
    }
    return AdvancedThenBlocked(advanced, sink.write_wq, WriteInterests::Space);
```

Splice is the test case that motivated the step model: it is fundamentally progressive (moves bounded chunks across many invocations), it may block on either of two different carriers depending on which side stalls, and it must publish on the opposite side of each move (source write_wq for space freed, sink read_wq for data available).

Under the prepare/commit pattern this was awkward; under the step model it fits naturally as a step that observes both sides, performs a bounded move, and reports outcome with the correct wake carrier.

### 6.3 Bounded work within a step

Every step is upper-bounded in latency (STEP-2). For progressive operations, the bound is typically a chunk size: move up to N pages, read up to M bytes, enumerate up to K directory entries. The bound is chosen so that one step's work completes within a small, predictable latency envelope — typically sub-microsecond for pure memory operations, microsecond-scale for operations that touch caches but not I/O.

If the operation's overall work exceeds the bound, the step makes partial progress (reports `Advanced(progress)`), and the driver loops. Reactor-observable yield points occur between steps, never within. This is what makes latency analysis tractable: the reactor knows that any scheduled step will return within the bound, and any long operation is a series of such steps.

### 6.4 When to multi-step

A step is part of a multi-step operation when:

- The operation has inherent partial-progress semantics (POSIX `ssize_t` return).
- The operation may block on availability of kernel state (readiness, space, queue depth).
- The operation's work exceeds a single step's bound and must be chunked.

Data-plane operations almost always multi-step. Creation/removal/reconfiguration almost always one-step.

---

## 7. How steps compose with wait and drivers

The upper/lower half division (CONCEPTS §6) places steps in the lower half (synchronous worker code) and composition in the upper half (async driver code).

### 7.1 The three driver modes

The dispatch layer offers three closed driver modes (ARCH-3, CONCEPTS §9.1):

**Nonblocking.** Call step once; on `Blocked` or `AdvancedThenBlocked` with zero progress, return `EAGAIN`. On `AdvancedThenBlocked` with nonzero progress, return the accumulated progress (POSIX partial-progress convention).

**Waiting** (covers blocking and timed variants via WaitProtocol). Loop: call step; handle each outcome per §6.1's loop. On `Blocked*`, invoke wait primitive with `WaitProtocol::Interruptible` (or Killable, or a timeout variant).

**Selecting** (epoll-shape). Register on the wake carrier without invoking step. On carrier fire, return which capability fired; caller may then invoke step (or not).

### 7.2 Wait primitive invocation

When a driver in waiting mode handles a `Blocked(c, m)` outcome, it invokes the wait primitive with the outcome's carrier and interests plus its configured protocol:

```
wait::wait_event(
    wq: c,                      // carrier from Blocked outcome
    condition: || step_would_advance(),   // the re-observation predicate
    protocol: Interruptible,    // driver-configured protocol
    ctx,
).await
```

Under the identity "retry is recheck" from CONCEPTS §9.4, the `condition` closure is a lightweight predicate check that returns true if step would now return `Advanced*` or `Done` or `Err`. In practice, the condition often matches the step's early-observation phase: check the same fields the step would check first.

Some subsystems implement `condition` by calling into a dedicated predicate function that the step itself also uses internally — keeping the re-check and the step in sync by construction.

### 7.3 Signal delivery into waits

When a waiter wakes due to signal delivery (SIGKILL for Killable, any unblocked signal for Interruptible), the wait primitive returns a classified outcome (Interrupted, Killed). The driver translates to `EINTR` or partial-progress-return per POSIX rules:

- If no progress yet accumulated: return `EINTR`.
- If progress accumulated: return `Ok(accumulated)` (partial success, POSIX `ssize_t` convention).

This translation is at the dispatch layer, not in the step. Steps never see signal state directly.

---

## 8. Subsystem module layout for steps

Under SUBSYSTEM_ANATOMY §2, a subsystem's `execution/` module houses step functions. The concrete file layout:

```
execution/pipe/
    step_read.rs        // pipe::step_read(...)
    step_write.rs       // pipe::step_write(...)
    step_open.rs        // pipe::step_open(...)
    step_close.rs       // pipe::step_close(...)
    step_ioctl.rs       // pipe::step_ioctl(...)
    resume.rs           // PipeReadResume, PipeWriteResume types
    mod.rs              // re-exports

execution/vfs/
    step_open.rs
    step_close.rs
    step_unlink.rs
    step_rename.rs
    step_mkdir.rs
    ...
```

One file per step function is the common pattern; related steps may share a file (e.g., `step_read` and `step_write` on a simple ring-based entity might be co-located). Resume types are subsystem-private; they do not leak into the driver layer beyond their opaque handle.

The prior `*_prepare` + `*_commit` pair convention is retired. A step function does both halves (observe+upgrade, then mutate+publish) within one function body, reflecting STEP-6's rejection of an operation-level commit phase.

---

## 9. Anti-patterns

Patterns that violate the step model, with the violated invariants:

**A-1.** Returning a witness in a `StepOutcome`. *Violates WIT-3, WIT-4.* Fix: extract `Cap<T>` from witness before return; store in subsystem-private resume state.

**A-2.** Storing a witness in `self` or a static. *Violates WIT-4.* Fix: treat witness as strictly stack-local within the step function body.

**A-3.** Calling `.await` inside a step. *Violates STEP-2.* Fix: if the step cannot complete synchronously, return `Blocked` with the carrier the async operation would wake on, and let the driver compose the wait.

**A-4.** Inspecting subsystem state from a driver (e.g., "check pipe buffer size before calling step_read"). *Violates DISP-2.* Fix: the check belongs in step_read's observation phase.

**A-5.** Firing a signal before the corresponding mutation. *Violates SIG-4.* Fix: always fire after `substrate::*_commit` returns, never before.

**A-6.** Firing a signal from outside a step's publish phase (e.g., from checks/, or from a projection). *Violates SIG-6 and the commit discipline.* Fix: signals only fire from the publish phase of a step.

**A-7.** Writing fields directly to mutate state, bypassing substrate primitives. *Violates STEP-4 and observer-safety.* Fix: all binding mutations go through `substrate::index::*`; all zone writes go through `substrate::zone::sign`.

**A-8.** A step that makes progress, then mutates further after detecting it will block, then returns `Blocked` without the earlier progress. *Violates STEP-3, STEP-4, STEP-8.* Fix: if progress was made and the step then blocks, return `AdvancedThenBlocked` carrying the progress. Advanced progress is externally visible (STEP-8); it cannot be rolled back silently.

**A-9.** Returning `Err` after substrate commits have succeeded. *Violates STEP-3, STEP-8 (if the commits published progress) or indicates the commits should not have been issued (if they shouldn't count as progress).* Fix: error detection must precede commits; once substrate commits succeed, the step must return a success outcome (possibly `Advanced*`). An error after successful progress-publication surfaces on the *next* step invocation as `Err`, not retroactively on the current step.

**A-10.** Carrying a witness across a `Blocked`-then-resume boundary. *Violates WIT-3.* Fix: stash a `Cap<T>` in resume state; on resumption, downgrade to fresh `IdentRef` under fresh guard and re-invoke require.

**A-11.** Acting on a value read or computed during a previous step without re-verifying under the current guard. *Violates STEP-7.* Fix: any state-sensitive decision within a step derives from require invoked under the current guard. Retained Caps carry identity stability across steps; they do not carry predicate truth. Reading a field during setup, returning `Blocked`, and then acting on the previously-read field on resumption is the canonical bug this rule prevents — the field's truth may have changed during the wait.

**A-12.** A driver inspecting subsystem state through a "convenient" side channel — reading a capability's internal field to decide whether to call step, or consulting a subsystem's shared table for optimization. *Violates DISP-2 and DISP-6.* Fix: the state-dependent decision must surface through `StepOutcome`. If the driver needs to know "can I skip this step," the subsystem must encode that as a step outcome (e.g., `Done` returned immediately on the first call when nothing to do). Drivers see outcomes and nothing else.

Each anti-pattern has a canonical fix. Lints can detect many of these structurally (e.g., witness types bearing `'g` appearing in non-stack positions, `.await` inside functions signatured as step, imports from subsystem internals into dispatch-layer code).

---

## 10. Relationship to prior patterns

For readers familiar with earlier iterations:

- **v11 prepare/commit pairs** are now single step functions. The prepare phase corresponds to observe+upgrade; the commit phase to mutate+publish. One function, four internal phases, STEP-6 explicitly rejects operation-level phase separation.

- **v11 attempt functions** (as used in blocking I/O path) are now step functions with multi-step trajectories. The former "attempt" is one invocation of a step; the former "driver retry loop" is unchanged but its primitive is now "call step" rather than "call attempt."

- **v12 `poll_read`-style ops** with `&mut Option<WakerGuard>` parameters threaded through subsystems are retired. The WakerGuard plumbing is internal to the wait primitive; step functions never see it. `Blocked` outcomes name the carrier abstractly; the wait primitive handles subscribe/unsubscribe.

- **Witness discipline in prior iterations** assumed witnesses would be bounded by scope but did not rigorously enforce "one step." The step model makes this structural: each step acquires and releases its own guard, so witnesses cannot survive across invocations even by accident. The lints formalizing this (WIT-3, WIT-4) are direct specifications of the prior convention.

- **ProcessProjections reading conditionally on payload presence** is now a natural consequence of BIF-2 (identity/payload independently retained) plus PRED-3 (projections declared per entity). The `#[projections]` macro pattern from v12 remains useful for generating read-only projection views over structure/; it is not affected by the step model.

---

## 11. Open questions

A few concerns remain to be closed in subsequent specification work:

**11.1. Bounded-work granularity.** STEP-2 requires steps to be bounded, but does not specify a concrete bound. Different operation classes (byte-copy, page-fault, directory-enumeration) have different natural granularities. A companion document or per-subsystem convention will fix the canonical bounds.

**11.2. Resume state versioning.** If a subsystem revises its step functions between kernel versions, an in-flight operation's resume state may become invalid. For initial implementation this is moot (resume state is ephemeral, doesn't persist across reboots), but for future checkpoint/restart work it becomes relevant.

**11.3. Progress type uniformity.** Currently `Progress` is operation-specific (bytes, pages, entries). A typed-progress approach (`Progress<Bytes>`, `Progress<Pages>`) would catch category errors at compile time. Whether this is worth the API complexity is deferred.

**11.4. Step-local tracing.** Tracepoints that fire within a step currently use the publish-phase primitives. A separate "in-step observability" pattern (emit tracepoints at observe phase entry, upgrade phase, mutation points) may be useful for debugging. Specification deferred.

---

## References

- [`INVARIANTS.md`](./INVARIANTS.md) §7 (STEP-*), §3 (PRED-*), §4 (WIT-*), §5 (OBL-*), §6 (SIG-*).
- [`CONCEPTS.md`](./CONCEPTS.md) §5 (step model), §6 (upper/lower halves), §9 (wait primitive).
- [`object_model_0417.md`](./object_model_0417.md) §4 (reference hierarchy), §6 (reclamation), §7 (bindings, obligations).
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) §2 (module layout), §3 (three-phase commit — to be revised for step).
- [`LIVENESS.md`](./LIVENESS.md) §3 (projection catalog).
- [`ADR-resolution-half.md`](./ADR-resolution-half.md) §14 (reactor, informational I/O as Blocked).
