# Signal Attachments

**Status.** v1 (2026-04-19).

**Purpose.** The per-subsystem publication catalog. Records which transitions publish externally, on which carriers, with what polarity, fired from which step-commits. Cross-references projection rows in [`LIVENESS.md`](./LIVENESS.md); does not duplicate them.

**Audience.** Subsystem authors declaring new publications, reviewers auditing that a proposed signal attachment satisfies SIG-* invariants, consumers (epoll, wait primitive) checking what a capability publishes.

**Companion documents.**

- [`INVARIANTS.md`](./INVARIANTS.md) — SIG-* rules this catalog realizes; BIF-5 single-carrier rule enforced here.
- [`CONCEPTS.md`](./CONCEPTS.md) §2.3 (publication plane) and §8 (bus layering).
- [`LIVENESS.md`](./LIVENESS.md) §3 — projection catalog this references.
- [`BUS.md`](./BUS.md) — bus primitive APIs and declaration machinery.
- [`STEP_MODEL.md`](./STEP_MODEL.md) §3.4 — publish phase of the in-step commit discipline.

---

## 1. Separation from LIVENESS.md

`LIVENESS.md` catalogs projections — the semantic truth that predicates evaluate. This document catalogs signal attachments — the opt-in publication hooks that fire when selected transitions occur. The two catalogs are linked but separate (SIG-3: signal attachment is opt-in, declared per transition):

- Not every projection has a signal attachment. Most projections are consulted only through require and need no external publication.
- Not every signal fires on a projection transition. Some publications (tracepoints, audit events) are observation hooks that do not correspond to a projection change.
- The projection catalog is the source of truth for predicates; this catalog is the source of truth for publication wiring.

Keeping them in separate documents preserves this distinction at the document level: changes to projections do not automatically imply signal-wiring changes, and vice versa.

---

## 2. Catalog schema

Each row in the per-subsystem tables records one signal attachment:

| Column | Meaning |
|---|---|
| **Entity** | The capability or entity type the attachment lives on. Must be a single retention-domain carrier (BIF-5). |
| **Carrier** | The bus primitive kind: `RawQueue` (level readiness), `RawPort` (edge lifecycle), `RawTrace` (passive recording). |
| **Wire** | Declared name of the wire on the carrier. Corresponds to a `readiness!`, `lifecycle!`, or `tracepoint!` declaration. |
| **Transition** | The state change that triggers firing. Phrased in terms of the entity's fields or projection transitions. |
| **Polarity** | `set` / `clear` (RawQueue), `fire` (RawPort), `emit` (RawTrace). |
| **Fired from** | The step-commit site that performs the linearizing write preceding this publication. Must be in `execution/*/step_*.rs` (SIG-6). |
| **Subscribers** | Who consumes this attachment. Informs whether the attachment is load-bearing or optional. |
| **Projection link** | Cross-reference to `LIVENESS.md §3` row if the transition corresponds to a projection change. `—` if standalone (e.g., tracepoint). |

The schema is enforced by lint: a row missing columns, or columns with inconsistent content (e.g., a RawPort wire with a `set` polarity), is flagged.

---

## 3. Per-subsystem catalogs

Catalogs per subsystem. Entries are listed for subsystems in scope for the initial framework; subsystems deferred to later phases note the deferral.

### 3.1 VFS — DEntry / RNode

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| DEntry (parent) | RawPort | `fsnotify_port` | Unlink: binding removed from children | `fire(Unlink{name})` | `vfs::step_unlink` | fsnotify, fanotify | DEntry.namespace → false (for child) |
| DEntry (parent) | RawPort | `fsnotify_port` | Create: binding installed | `fire(Create{name, rnode})` | `vfs::step_mkdir`, `vfs::step_create` | fsnotify, fanotify | — (adds new DEntry, not a transition of existing projection) |
| DEntry (parent) | RawPort | `fsnotify_port` | Rename-from: binding withdrawn | `fire(MovedFrom{name})` | `vfs::step_rename` | fsnotify, fanotify | DEntry.namespace → false (for source) |
| DEntry (parent) | RawPort | `fsnotify_port` | Rename-to: binding installed | `fire(MovedTo{name})` | `vfs::step_rename` | fsnotify, fanotify | — |
| RNode | RawQueue | `readable` | Content written (for regular files via cache) | `set(HasData)` | `vfs::step_write_commit` (via cache) | poll/select, epoll | RNode.payload state |
| RNode | RawQueue | `readable` | Writers closed, no more data coming | `set(Broken)` | `vfs::step_close` (last writer on pipe-like) | poll/select, epoll | RNode.payload state |
| RNode | RawTrace | `vfs_ops` | Open, read, write, unlink, rename | `emit({syscall, args})` | each corresponding step | ftrace, perf, BPF | — |

Notes:

- The `fsnotify_port` on a parent DEntry fires on any child-binding mutation: unlink, create, rename (both sides). This matches Linux's fsnotify hook placement.
- RNode's `readable` wire is a general-purpose readiness queue shared by regular files (via the page cache's content-available state), pipes (via ring fill level), sockets (via receive queue), and character devices (via driver-specific readiness).
- The `vfs_ops` tracepoint is emitted on every VFS step for observability; nop-patched when no tracers are attached.

### 3.2 VFS — OpenFile

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| OpenFile | RawPort | `close_port` | Last reference dropped, file closing | `fire(Closing)` | `vfs::step_close` | inotify (for watched files), epoll (for watched fd) | OpenFile.structural → false |

OpenFile itself publishes little; most observable state lives on its RNode. The `close_port` notifies subscribers (notably epoll's target-fd tracking) that the file is going away so they can clean up subscriptions.

### 3.3 Process — ProcessIdentity / ProcessPayload

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| ProcessIdentity | RawPort | `exit_port` | Payload transitioned to None (exit) | `fire(Exited{status})` | `process::step_exit_commit` | waitpid (parent), pidfd subscribers, ptrace tracer | ProcessPayload.payload → false |
| ProcessIdentity | RawPort | `signal_port` | Signal delivered (generated for this process) | `fire(SignalGenerated{sig})` | `process::step_signal_deliver` | signalfd subscribers, ptrace tracer | — (not a projection change) |
| ProcessIdentity | RawPort | `ptrace_port` | Ptrace stop point reached | `fire(Stop{reason})` | dispatch/intercept/ptrace via `process::step_intercept_commit` | attached tracer | — |
| ProcessIdentity | RawTrace | `sched_trace` | Scheduling events, context switches | `emit({from, to, reason})` | scheduler (in reactor), via process wire | ftrace, perf | — |
| ThreadIdentity | RawPort | `thread_exit_port` | ThreadPayload transitioned to None | `fire(ThreadExited)` | `thread::step_thread_exit_commit` | pthread_join, clear_child_tid futex wake | ThreadPayload.payload → false |

Notes on BIF-5 compliance: every row names a single entity (ProcessIdentity or ProcessPayload or ThreadIdentity). No wire spans both Identity and Payload. The `exit_port` fires when Payload transitions to None; the wire lives on Identity (because ProcessPayload has been reclaimed by the time subscribers observe the event — only Identity survives the zombie window). Placing the wire on Identity is the BIF-5-compliant choice.

### 3.4 Process — signalfd, pidfd

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| SignalFd | RawQueue | `signalfd_readable` | New signal matching mask enqueued | `set(HasSignal)` | `process::step_signal_enqueue` (when signal routes to a signalfd) | poll/select, epoll | — |
| PidFd | RawQueue | `pidfd_readable` | Target process exited (pidfd becomes readable with exit info) | `set(HasExit)` | `process::step_exit_commit` | poll/select, epoll | ProcessPayload.payload → false |

PidFd's `HasExit` and ProcessIdentity's `exit_port` fire on the same underlying transition (process exit) but target different subscriber shapes: pidfd for poll-based readers holding an fd, exit_port for waitpid-style direct observers. Both fire from the same step-commit; the step publishes to both wires as separate `fire` calls.

### 3.5 Pipe

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| Pipe (read end) | RawQueue | `read_wq` | Bytes written to ring | `set(HasData)` | `pipe::step_write_commit` | blocking reader, poll/select, epoll | Pipe.readable projection |
| Pipe (read end) | RawQueue | `read_wq` | All writers closed | `set(Broken)` | `pipe::step_close` (last write-side close) | blocking reader, poll/select, epoll | Pipe.writers_exist → false |
| Pipe (write end) | RawQueue | `write_wq` | Bytes drained from ring | `set(Space)` | `pipe::step_read_commit` | blocking writer, poll/select, epoll | Pipe.writable projection |
| Pipe (write end) | RawQueue | `write_wq` | All readers closed | `set(Broken)` | `pipe::step_close` (last read-side close) | blocking writer, poll/select, epoll | Pipe.readers_exist → false |

Note: a pipe is two capability ends (read end, write end), each with its own RawQueue. Writing to the pipe fires on the read end's `read_wq` (readers care); draining fires on the write end's `write_wq` (writers care). This is cross-capability firing within one subsystem — legal per SIG-6 because both capabilities are in the pipe subsystem's commit path. BIF-5 is satisfied: each wire attaches to exactly one capability.

### 3.6 Socket

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| SocketIdentity | RawQueue | `recv_wq` | Data arrived in receive queue | `set(HasData)` | `socket::step_deliver_data_commit` (from protocol stack) | poll/select, epoll, blocking recv | SocketPayload.readable |
| SocketIdentity | RawQueue | `recv_wq` | Connection closed remotely | `set(Broken)` | `socket::step_close_remote_commit` | poll/select, epoll, blocking recv | SocketPayload.payload transitions |
| SocketIdentity | RawQueue | `send_wq` | Send buffer drained below high watermark | `set(Space)` | `socket::step_ack_commit` (from protocol stack) | poll/select, epoll, blocking send | SocketPayload.writable |
| SocketIdentity (listening) | RawQueue | `accept_wq` | New connection arrived in accept queue | `set(HasPending)` | `socket::step_enqueue_accept_commit` | blocking accept, poll/select, epoll | — |
| SocketIdentity | RawPort | `urgent_port` | Urgent data (TCP OOB) arrived | `fire(Urgent)` | `socket::step_deliver_urgent_commit` | SIGURG delivery, poll OOB tracking | — |

Notes:

- All socket readiness wires attach to SocketIdentity rather than SocketPayload. This is deliberate: per BIF-5 and the Identity/Payload split, readiness is an addressability concern (observers can still poll a socket that has payload-degraded); placing wires on Identity preserves observability through the closed-but-fd-held window.
- Protocol-specific state transitions (TCP FIN received, UDP dgram arrived) are subsystem-internal commits that fire on these generic readiness wires. Protocol state machines are subsystem implementation, not catalog entries.

### 3.7 Timerfd, Eventfd, Signalfd

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| Timerfd | RawQueue | `timer_wq` | Timer expired (counter incremented) | `set(Expired)` | `timer::step_expire_commit` (from reactor's timer subsystem) | poll/select, epoll, blocking read | Timerfd.expirations_pending projection |
| Eventfd | RawQueue | `event_wq` | Counter written (either via write(2) or peer) | `set(HasCount)` | `eventfd::step_write_commit` | poll/select, epoll, blocking read | — |
| Signalfd | RawQueue | `signalfd_wq` | Signal matching mask received | `set(HasSignal)` | `process::step_signal_enqueue` (routes through signalfd filter) | poll/select, epoll, blocking read | — |

These are Linux-specific fd-ified notification mechanisms; each is a capability whose whole purpose is to expose an event stream as a readable fd. The readiness wires are straightforward RawQueue attachments.

### 3.8 Mount

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| MountIdentity | RawPort | `umount_port` | Mount detached from tree (MNT_DETACH or final umount) | `fire(Detached)` | `mount::step_umount_commit` | resolvers holding Cap<MountIdentity> can observe that their reference is now a dangling reservation | MountIdentity.namespace → false |

Most mount-lifecycle events are observable through VFS's fsnotify rather than a mount-specific wire. The `umount_port` exists primarily to inform in-flight resolvers that a mount they hold a Cap to has been detached, so they can fail gracefully.

### 3.9 Device / Driver

Device and driver hot-plug events are cataloged here but depend on the device-subsystem specification, which is out of scope for the initial framework documents.

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| Device | RawPort | `uevent_port` | Add, remove, change | `fire(UEvent{kind, attrs})` | `device::step_*_commit` | udev, subsystem consumers | Device.namespace, DevicePayload.payload |
| DeviceNode (char/block) | RawQueue | `device_readable` | Driver-dependent | `set(HasData)` | driver's `step_*_commit` path | poll/select, epoll | — |

The Device/Driver subsystem split (when DeviceIdentity/DevicePayload factoring is specified) will slot into this catalog. Current rows are placeholders indicating the shape; detailed specification deferred.

### 3.10 Tracing / Observability

All tracepoints across all subsystems are aggregate catalog entries. Each subsystem's step files emit tracepoints as declared by `tracepoint!`:

| Entity | Carrier | Wire | Transition | Polarity | Fired from | Subscribers | Projection link |
|---|---|---|---|---|---|---|---|
| (any) | RawTrace | `<subsystem>_<op>` | Operation invoked | `emit({args})` | each step function | ftrace, perf, BPF programs, tx-observe | — |

Per the `tracepoint!` macro model: declarations are per-subsystem; emissions are per-step. Nop-patched when no subscribers; jump-to-handler when any subscriber attaches.

### 3.11 Deferred

These subsystems will receive catalog entries in subsequent phases:

- **SysV IPC** (msg, sem, shm). Each carries publication state (queue has messages, semaphore has available permits); catalog entries deferred pending SysV subsystem specification.
- **POSIX IPC** (mq_open, sem_open, shm_open). Mostly delegated to VFS (mqueue/shm via tmpfs); specific mq-notify registration requires a dedicated wire, catalog entry deferred.
- **Cgroup** events. Cgroup-v2 exposes various attention-drawing events (memory pressure, cpu pressure, oom). Wire catalog deferred pending cgroup subsystem specification.
- **Futex** does not appear in this catalog. Per the futex subsystem's design, futex waitqueues are subsystem-internal and do not publish to external subscribers — futex is pure kernel-internal synchronization.
- **rt_mutex** does not appear in this catalog. Deferred per INVARIANTS EXC-3 pending priority semantics work.

---

## 4. Review checklist for new attachments

When proposing a new signal attachment, verify against SIG-* and BIF-5:

1. **SIG-3.** Is this attachment declared per transition, for a transition with genuine external subscribers? If no subscribers would use it, the attachment is unnecessary — the transition may still be a projection change, but does not require publication.

2. **SIG-4.** Does the firing happen after the linearizing write, in the same step-commit? Specify which substrate primitive call precedes the bus primitive call.

3. **SIG-5.** Does the carrier choice respect per-carrier ordering expectations? If subscribers need to observe transitions in a specific order, they must be on the same carrier.

4. **SIG-6.** Is the fire called from an `execution/*/step_*.rs` publish phase? Not from checks, not from substrate, not from project, not from structure.

5. **SIG-7 / BIF-5.** Does the carrier attach to exactly one retention domain — identity or payload — never both? For split entities, pick the domain whose lifetime matches the subscriber's needs.

6. **SIG-11 (cross-capability firing).** If the fire targets a different capability's wire, is the target capability within the same subsystem? Cross-subsystem firing requires coordination through the target subsystem's step, not direct wire access.

7. **SIG-12 (AdvancedThenBlocked publication).** If the step returns a composite outcome, is publication performed normally during the mutation portion? The composite outcome does not defer or prevent publication.

8. **STEP-9.** If the firing is part of a composite outcome (progress + block), both the publish and the block-outcome portions must be honored.

9. **Subscriber-side (SIG-9, SIG-10).** For documenting subscribers in the Subscribers column: confirm the subscriber's design honors the re-observation contract (wake does not grant truth; bus state does not imply truth). Subscribers documented here must conform to SIG-9 and SIG-10; if a proposed subscriber would act on wake alone or on bus state alone, the subscriber's design is buggy and the attachment should not be added to support it.

10. **Projection link.** Does this transition correspond to a projection change? If yes, link the projection row in `LIVENESS.md §3`. If no, document why the attachment is standalone (tracepoint, audit event, etc.).

11. **Carrier kind.** Is the chosen kind appropriate?
   - RawQueue for multi-subscriber level readiness.
   - RawPort for edge-triggered lifecycle events, possibly with consumption.
   - RawTrace for passive recording with optional subscribers.

12. **BIF-5 enforcement for split entities.** For entities factored per object_model §8.1.1 (e.g., ProcessIdentity/ProcessPayload), the carrier must name one zone slot. Signals whose subscribers need to observe through the zombie window go on Identity; signals whose subscribers only care during payload-live windows may go on either.

---

## 5. Anti-patterns

Patterns that violate SIG-* or BIF-5, with catalog-level diagnostics:

**S-1. Attachment with no subscribers.** A catalog entry with empty "Subscribers" column. *Violates SIG-3 (opt-in per transition).* Fix: remove the attachment, or identify the intended subscriber and record it.

**S-2. Carrier spans Identity and Payload.** An attachment whose wire fires from a transition on both zones. *Violates BIF-5, SIG-7.* Fix: split into two attachments (one per zone), or determine which zone's lifetime matches the subscriber contract and attach there only.

**S-3. Fire from non-step code.** Catalog entry whose "Fired from" cites checks, substrate, project, or structure. *Violates SIG-6.* Fix: move the fire into a step-commit publish phase.

**S-4. Fire before write.** Catalog entry where the fire precedes the substrate commit it publishes. *Violates SIG-4.* Fix: reorder the step-commit code so the fire follows the write.

**S-5. Publication batched at operation end.** A multi-step operation that publishes only at terminal `Done`, not per progressive step. *Violates SIG-8.* Fix: publish in each step that makes subscriber-observable progress; typically in each step's publish phase.

**S-6. Attachment replaces projection.** A proposal to remove a projection from `LIVENESS.md` and replace it with a signal attachment. *Violates SIG-1 (signals are not truth).* Fix: projections define truth; signals only publish transitions. Both are needed for predicates requiring monotonicity reasoning.

**S-7. Carrier kind mismatched to semantics.** Using RawPort for multi-subscriber level readiness, or RawQueue for single-consumer lifecycle events. *Semantic mismatch, not a specific invariant violation, but leads to consumer confusion and incorrect wake behavior.* Fix: choose the carrier kind appropriate to the subscriber contract per §1.1 of `BUS.md`.

**S-8. Subscriber acts on wake alone.** A documented subscriber whose design acts on wake delivery without re-evaluating predicates. *Violates SIG-9.* Fix: the subscriber's design must include a predicate re-evaluation step on wake. If the subscriber cannot include this step (e.g., legacy API constraint), the attachment should not be added to support it.

**S-9. Subscriber reads bus state as truth.** A documented subscriber whose design peeks at wire state to decide whether a condition holds. *Violates SIG-10.* Fix: predicates evaluate entity fields under an epoch guard, not bus wire values. Subscriber optimizations that shortcut predicates by reading bus state are incorrect regardless of how correct they appear in testing.

---

## 6. Lints

Lints that enforce catalog consistency:

**L-1. Row completeness.** All columns present and non-empty for each row. Exception: "Projection link" may be `—` for standalone attachments.

**L-2. Entity-carrier consistency.** An entity declared in BIF as Identity/Payload-split must have its wires attached to exactly one of the two. Rows violating this are flagged.

**L-3. Source-site verification.** "Fired from" entries must point to existing `execution/*/step_*.rs` functions. A catalog row naming a non-existent step site is a stale entry.

**L-4. Polarity-carrier consistency.** `set`/`clear` polarities belong to RawQueue entries; `fire` polarities to RawPort; `emit` to RawTrace. Mismatches are flagged.

**L-5. Projection-link freshness.** If "Projection link" cites a LIVENESS row, that row must exist. Renames or deletions in LIVENESS that orphan catalog rows are flagged.

**L-6. No orphan declarations.** Every `readiness!` / `lifecycle!` / `tracepoint!` declaration in code should have at least one catalog row (either a direct subscriber or a tracepoint with documented purpose). Orphan declarations (wire declared but never fired or never subscribed) are flagged for cleanup or documentation.

---

## 7. Versioning

Signal attachments are part of the kernel's stable API surface — external observers (epoll clients, fsnotify listeners, ftrace consumers) depend on wire declarations and their semantics. Versioning rules:

- **Addition** of a new attachment is API-compatible. Subscribers who do not request the new wire are unaffected.
- **Removal** of an attachment is an API break. Subscribers expecting the wire will fail. Removal requires a deprecation cycle.
- **Semantic change** (same wire, different firing conditions) is an API break. Subscribers may observe wakes that no longer match their interest, leading to wasted re-checks or missed transitions.
- **Carrier kind change** (e.g., changing a RawPort to a RawQueue) is always an API break. Consumer code structure differs fundamentally.

The catalog is the source of record for versioning decisions. Rows marked with `(deprecated: <version>)` are scheduled for removal in a future release.

---

## 8. Open questions

**8.1. Cross-subsystem attachment ownership.** When a fire on one subsystem's wire is triggered by a commit in another subsystem's step (e.g., pipe's write_commit fires on the pipe's read_wq, which the reader's driver observes), the catalog row lives in the pipe subsystem's section. For less symmetric cases (e.g., a VFS operation firing fsnotify on a directory that the inotify subsystem subscribes to across module boundaries), the placement is less clear. Current convention: the row is cataloged in the subsystem owning the entity the wire attaches to.

**8.2. Aggregate tracepoints.** Tracepoint entries are currently aggregated as a single catalog row ("any entity, tracepoint, per-op"). A more granular catalog would record each tracepoint declaration separately. Whether the granularity is worth the catalog size is deferred.

**8.3. Conditional publication.** Some attachments fire only under specific conditions (e.g., ptrace_port fires only if the process is traced). The catalog currently records the publication without the predicate that gates it. A "fires-when" column may be added if this becomes a source of consumer confusion.

**8.4. Catalog generation.** A future enhancement could generate this catalog from source annotations (each `fire` call site annotated with catalog metadata, catalog regenerated by build tool). Currently the catalog is maintained manually. Automated generation would enforce row-source consistency at build time.

---

## References

- [`INVARIANTS.md`](./INVARIANTS.md) §6 (SIG-*), §2 (BIF-5), §7 (STEP-9 composite).
- [`CONCEPTS.md`](./CONCEPTS.md) §2.3 (publication plane), §8 (bus static/temporal).
- [`LIVENESS.md`](./LIVENESS.md) §3 (projection catalog cross-referenced here).
- [`BUS.md`](./BUS.md) §1 (primitives), §3 (declarations), §5 (firing).
- [`STEP_MODEL.md`](./STEP_MODEL.md) §3.4 (publish phase), §3.5 (return), §5.x and §6.x (per-subsystem step examples with fire calls).
