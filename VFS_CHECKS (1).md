# VFS Checks Spec

**Status.** v2.1 (2026-04-19). Terminology aligned with the step model: witnesses are consumed by the step's upgrade sub-phase (STEP-4 phase 2); `*_prepare` function references replaced with step-sub-phase references; `ADR-resolution-half.md` references removed.

**Supersedes (v2.1 → v2).** Terminology sync only; no semantic changes to the walker, trampoline, or require-function surface.

**Supersedes (v2 → v1).** v1. Witnesses corrected to `IdentRef`-carrying per object_model §7.6. Walker state Cap/IdentRef morphing at yield boundaries made explicit. Refinement wrappers extended per the UnlinkableNonDirChild / RmdirableDirChild discussion.

**Purpose**: Specify the internal structure of `vfs/checks/`. Define the walker as a shared transition kernel plus mode-specific acceptance, the sealed trampoline re-entry, and the require-function surface.

**Audience**: Implementers of `vfs/checks/`, reviewers auditing resolution purity, agents extending path-taking syscalls.

**Companion documents**:
- [`object_model_0417.md`](./object_model_0417.md) — authoritative for references, bindings, projections.
- [`LIVENESS.md`](./LIVENESS.md) — projection/require/witness framework.
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md) — four-module layout, five-phase step discipline, substrate.
- [`STEP_MODEL.md`](./STEP_MODEL.md) — execution primitive; the step function body within which vfs/checks witnesses are consumed.
- [`INVARIANTS.md`](./INVARIANTS.md) — PRED-*, WIT-*, STEP-* rules for resolution purity and witness scope.

This document does not re-derive those frameworks. It specifies the VFS-specific instance.

---

## 1. What `vfs/checks/` Is

VFS is a **representation-layer subsystem**. It owns the namespace — the DEntry tree and mount tree — through which paths are resolved into typed handles on subsystem-owned entities.

`vfs/checks/` has one externally-visible job: turn a path (plus context) into a **witness** consumed by a VFS step's upgrade sub-phase. The witness carries `IdentRef`-level observations sufficient to prove that at some recent moment the path resolved to the claimed structure. The step's upgrade sub-phase (STEP-4 phase 2) promotes the IdentRefs to retention evidence per the operation's obligations.

Internally, this is a **small-language interpreter**: a semi-Thue rewriting system over `WalkState` with bounded symlink budget, shared transition rules, and per-syscall acceptance/witness-construction.

---

## 2. Projections Consumed

Per LIVENESS.md §3:

| Entity | Projection | Context | Meaning | Mechanism |
|--------|-----------|---------|---------|-----------|
| DEntry | structural | — | Retention > 0 | identity retention |
| DEntry | namespace | MountNamespace | reachable from mnt_ns root | reachability |
| DEntry | payload (derived) | — | resolved RNode has payload_live | definitional via resolution |
| RNode  | structural | — | Retention > 0 | identity retention |
| RNode  | payload | — | nlinks-LinkPin-count ∨ open_refs-OpenPin-count (∨ projection-state-valid for synthetic) | payload retention (compound) |

All `require_*` consume **DEntry namespace** on every path component traversed. They consume **RNode payload** only at the terminal, and only for modes that need the entity to have live content (not `ParentAndName`, which wants proof of absence).

The walker consumes projections via **observation** (IdentRef + predicate). Retention upgrades are deferred to the step's upgrade sub-phase; the walker itself never upgrades.

---

## 3. Module Layout

```
vfs/checks/
├── mod.rs            — explicit export whitelist
├── predicates.rs     — pure bool predicates over IdentRef + context
├── witness.rs        — lifetime-parameterized witness types
├── require.rs        — public require_* surface
└── resolution/
    ├── state.rs      — WalkState, WalkMode, WalkCause, ResumeToken
    ├── step.rs       — kernel_step (mode-agnostic δ)
    ├── terminal.rs   — accepts, build_witness (α + ω per mode)
    ├── error.rs      — classify (WalkCause → Errno)
    └── driver.rs     — run_walker, resume_walker, walk_to_completion
```

No glob `pub use resolution::*`. `mod.rs` whitelists what leaves the module.

---

## 4. Walker Modes

Six modes. The set is closed.

```rust
pub enum WalkMode {
    Entity,                    // followed walk to terminal entity
    EntityUnfollowed,          // walk stops at final symlink; does not follow
    ParentAndName,             // walk to penultimate component; final may be absent
    ParentAndNamedChild,       // walk to parent; final must be present
    EntityOrParentAndName,     // walk to terminal; returns Present or Absent
    MountPoint,                // walk to terminal; terminal must be mount root
}
```

`RealPath` (canonical-path rendering) is not a distinct mode; it reuses `Entity` with post-walk rendering performed in the facade.

### 4.1 Mode-to-Facade mapping

| Mode | Core facade | Refinement facades | Representative syscalls |
|------|-------------|-------------------|-------------------------|
| Entity | `require_entity` | `require_directory`, `require_real_path` | open (no-create), stat, chmod, execve, chdir, chroot, realpath |
| EntityUnfollowed | `require_entity_unfollowed` | `require_symlink_for_readlink` | lstat, lchown, readlink, O_PATH\|O_NOFOLLOW |
| ParentAndName | `require_parent_and_name` | (none) | mkdir, symlink, mknod, link (new side), O_CREAT\|O_EXCL |
| ParentAndNamedChild | `require_parent_and_named_child` | `require_unlinkable_non_dir_child`, `require_rmdirable_dir_child` | unlink, rmdir, rename (old side) |
| EntityOrParentAndName | `require_entity_or_parent_and_name` | (none) | O_CREAT without O_EXCL |
| MountPoint | `require_mount_point` | (none) | umount, pivot_root |

Refinement facades wrap a core facade with a post-check predicate producing a refined witness type. See §10.2.

---

## 5. `WalkState`

`WalkState` is pure machine state. `WalkMode` is an interpreter parameter passed alongside, **not stored** in state.

### 5.1 Within a guard

```rust
pub struct WalkState<'g> {
    pub cursor: IdentRef<'g, DEntry>,
    pub remaining: Components<'a>,          // cursor into the path buffer
    pub symlink_budget: u8,
    pub root_ctx: RootCtxRef<'g>,
    pub trail: WalkTrail<'g>,
}
```

Under a live guard, walker state carries IdentRef (refcount-free observation). Trail elements are IdentRef too, anchored to the same guard.

### 5.2 Across yields

When `kernel_step` emits `NeedIO`, the walker must yield. IdentRef cannot cross `.await`. Yield protocol:

1. Before yielding, the walker upgrades `cursor`, `trail` elements, and `root_ctx` components to `Cap<DEntry>` via `substrate::pin::upgrade_to_cap`. Upgrades are fallible (SENTINEL_DEAD CAS); failure reports clean error.
2. The upgraded Caps are packaged into `ResumeToken` (`'static`).
3. The guard is released.
4. The thread yields.

On resume:

1. A fresh guard is acquired.
2. `ResumeToken`'s Caps are downgraded to IdentRef under the fresh guard (free; `IdentRef::observe(&cap, &guard)`).
3. `apply_io_result` applies the I/O result to produce a fresh `WalkState<'g2>` under the new guard.
4. Caps in the token drop (retention was temporary for crossing the yield).
5. `run_walker` re-enters with the new state.

**Walker's Cap/IdentRef morphing discipline:** IdentRef within a guard, Cap across yields. Warm walks (no NeedIO) pay zero refcount traffic. Cold walks pay at most one Cap acquisition per yield.

### 5.3 `Components<'a>`

Cursor into the path buffer. Zero allocation.

```rust
pub struct Components<'a> {
    buf: &'a [u8],
    cursor: usize,
}
```

Handles consecutive slashes, trailing slashes, empty components.

### 5.4 `WalkTrail<'g>`

Stack of ancestor DEntries and mount-boundary markers, for `..` traversal.

```rust
pub struct WalkTrail<'g> {
    entries: SmallVec<[TrailEntry<'g>; 8]>,
}

pub enum TrailEntry<'g> {
    DEntry(IdentRef<'g, DEntry>),
    MountBoundary { was_at: IdentRef<'g, DEntry> },
}
```

Trail is pushed on each forward step; popped on `..`. MountBoundary markers are pushed when the walker crosses into a mount; `..` across a mount pops the boundary and restores cursor to the mountpoint DEntry.

**DEntries do not carry parent pointers for walker purposes.** Ancestor tracking lives in the trail. Rename does not need to update descendants' parent pointers; each walker maintains its own view of "how I got here." Parent pointers may exist separately for getcwd rendering; they are a best-effort, stale-tolerant cache, not walker state.

### 5.5 `RootCtx`

```rust
pub struct RootCtxRef<'g> {
    pub mnt_ns_root: IdentRef<'g, DEntry>,
    pub chroot: Option<IdentRef<'g, DEntry>>,
    pub cwd: IdentRef<'g, DEntry>,
}
```

`*at` syscalls override `cwd` with the dirfd's DEntry. `AT_FDCWD` uses process `fs_context.cwd`.

Anchors may themselves be upgraded to Cap across yields (same discipline as walker state).

---

## 6. The Shared Kernel: `kernel_step`

Mode-agnostic δ. Reads WalkState and structure; produces KernelStep. No mutation.

```rust
pub(crate) fn kernel_step<'g>(
    state: WalkState<'g>,
    policy: FinalSymlinkPolicy,
    guard: &'g Guard,
) -> KernelStep<'g>;

pub(crate) enum KernelStep<'g> {
    Continue(WalkState<'g>),
    NeedIO(IORequest, ResumeToken),
    Error(WalkCause),
}
```

### 6.1 Transition rules

On entry, first re-predicate cursor:

```rust
if !namespace_live(&state.cursor, &state.root_ctx) {
    return Error(DetachedNamespace);
}
```

Then consume one component:

1. **Empty or `.`** — drop, Continue.
2. **`..`** — pop trail. If `TrailEntry::MountBoundary`, cursor becomes `was_at`. Otherwise cursor becomes popped DEntry. If trail empty and cursor at chroot or mnt_ns_root, cursor stays.
3. **Named component** — cache lookup on `state.cursor`. On hit: traverse permission check, mount crossing (push MountBoundary), symlink substitution (policy-gated for final):
   - Non-symlink, non-final: advance, Continue.
   - Symlink substitution: decrement budget; prepend target to remaining; absolute → cursor ← root. If budget zero: `Error(SymlinkBudgetExceeded)`.
4. **Cache miss** — emit IORequest; upgrade state-spanning references to Caps; package ResumeToken::LookupChild; release guard; return NeedIO.
5. **Permission denied** — `Error(TraverseDenied)`.
6. **Non-directory in non-final position** — `Error(NonDirectoryIntermediate)`.
7. **Component missing** — `Error(MissingComponent)`.
8. **Name too long** — `Error(NameTooLong)`.

### 6.2 `WalkCause`

```rust
pub(crate) enum WalkCause {
    MissingComponent,
    NonDirectoryIntermediate,
    SymlinkBudgetExceeded,
    TraverseDenied,
    NameTooLong,
    DetachedNamespace,
}
```

Terminal-type mismatches (readlink on non-symlink, chdir on non-directory, unlink on directory) are refinement-wrapper concerns in require.rs, not kernel_step's.

### 6.3 Final-symlink policy

Derived from mode by the driver; passed to kernel_step explicitly. Keeps kernel_step mode-agnostic.

```rust
pub(crate) enum FinalSymlinkPolicy { Follow, Stop }

fn final_symlink_policy(mode: WalkMode) -> FinalSymlinkPolicy {
    match mode {
        WalkMode::EntityUnfollowed => FinalSymlinkPolicy::Stop,
        _ => FinalSymlinkPolicy::Follow,
    }
}
```

---

## 7. Terminal: `accepts` and `build_witness`

```rust
pub(crate) fn accepts<'g>(mode: WalkMode, state: &WalkState<'g>) -> bool;

pub(crate) fn build_witness<'g>(
    mode: WalkMode,
    state: WalkState<'g>,
) -> Result<WalkWitness<'g>, Errno>;
```

### 7.1 Acceptance

| Mode | Additional α condition |
|------|-----------------------|
| Entity | cursor's RNode has payload_live |
| EntityUnfollowed | (none — symlink handling in kernel_step) |
| ParentAndName, ParentAndNamedChild, EntityOrParentAndName | driver short-circuits (§7.3) |
| MountPoint | cursor is a mount root |

### 7.2 Witness construction

Moves data from WalkState into typed witness. IdentRef lifetime `'g` preserved.

### 7.3 Penultimate-stop for parent-family modes

Driver short-circuits when `remaining` has exactly one component left:

```rust
fn is_parent_mode(mode: WalkMode) -> bool {
    matches!(
        mode,
        WalkMode::ParentAndName | WalkMode::ParentAndNamedChild | WalkMode::EntityOrParentAndName
    )
}

if is_parent_mode(mode) && state.remaining.count() == 1 {
    return terminal::build_penultimate_witness(mode, state);
}
```

`EntityOrParentAndName` penultimate logic attempts final-component lookup:
- Found + namespace_live → `Present(EntityAtPath)`.
- Negative cache → `Absent(ParentAndName)`.
- Cache miss → NeedIO via `ResumeToken::ProbeFinal`.

---

## 8. The Driver

```rust
pub(crate) fn run_walker<'g>(
    mode: WalkMode,
    state: WalkState<'g>,
    guard: &'g Guard,
) -> DriverStep<'g>;

pub(crate) fn resume_walker<'g>(
    mode: WalkMode,
    token: ResumeToken,
    io_result: IOResult,
    guard: &'g Guard,
) -> DriverStep<'g>;

pub(crate) async fn walk_to_completion(
    mode: WalkMode,
    path: &[u8],
    root_ctx: &RootCtx,
) -> Result<OwnedWitness, Errno>;
// OwnedWitness: lifetime-parameterized by the outer guard held during the async
// fn's frame; upgraded-ready for the step's commit sub-phase.

pub(crate) enum DriverStep<'g> {
    Accept(WalkWitness<'g>),
    Error(Errno),
    NeedIO(IORequest, ResumeToken),
}
```

`DriverStep` has no `Continue` variant. Continue is kernel_step's internal signal.

### 8.1 `run_walker` loop

```rust
pub(crate) fn run_walker<'g>(
    mode: WalkMode,
    mut state: WalkState<'g>,
    guard: &'g Guard,
) -> DriverStep<'g> {
    let policy = final_symlink_policy(mode);
    loop {
        if is_parent_mode(mode) && state.remaining.count() == 1 {
            return terminal::build_penultimate_witness(mode, state)
                .map_or_else(DriverStep::Error, DriverStep::Accept);
        }
        if accepts(mode, &state) {
            return match build_witness(mode, state) {
                Ok(w) => DriverStep::Accept(w),
                Err(e) => DriverStep::Error(e),
            };
        }
        match kernel_step(state, policy, guard) {
            KernelStep::Continue(next) => state = next,
            KernelStep::NeedIO(req, tok) => return DriverStep::NeedIO(req, tok),
            KernelStep::Error(cause) => {
                return DriverStep::Error(error::classify(mode, cause));
            }
        }
    }
}
```

### 8.2 `resume_walker`

```rust
pub(crate) fn resume_walker<'g>(
    mode: WalkMode,
    token: ResumeToken,
    io_result: IOResult,
    guard: &'g Guard,
) -> DriverStep<'g> {
    let state = match apply_io_result(token, io_result, guard) {
        Ok(s) => s,
        Err(errno) => return DriverStep::Error(errno),
    };
    run_walker(mode, state, guard)
}
```

Resume re-enters run_walker under a fresh guard. kernel_step's first action is re-predicating cursor, so concurrent mutation during the yield is caught on resume (race-degradation theorem preserved).

### 8.3 `walk_to_completion`

```rust
pub(crate) async fn walk_to_completion(
    mode: WalkMode,
    path: &[u8],
    root_ctx: &RootCtx,
) -> Result<OwnedWitness, Errno> {
    let outer_guard = epoch::pin();
    let initial = WalkState::init(path, root_ctx, &outer_guard)?;
    let mut step = run_walker(mode, initial, &outer_guard);
    loop {
        match step {
            DriverStep::Accept(w) => return Ok(OwnedWitness::from_identref(w, outer_guard)),
            DriverStep::Error(e) => return Err(e),
            DriverStep::NeedIO(req, tok) => {
                drop(outer_guard);
                let result = yield_for_io(req).await;
                outer_guard = epoch::pin();
                step = resume_walker(mode, tok, result, &outer_guard);
            }
        }
    }
}
```

`OwnedWitness` is the returned value: still IdentRef-bearing, but anchored to a guard the returning function will keep alive through its stack until the syscall script upgrades.

### 8.4 `ResumeToken`: continuation, not snapshot

```rust
pub(crate) enum ResumeToken {
    LookupChild {
        suspended: SuspendedState,
        parent: Cap<DEntry>,
        name: NameOwned,
    },
    ReadSymlink {
        suspended: SuspendedState,
        link_rnode: Cap<RNode>,
    },
    ProbeFinal {
        suspended: SuspendedState,
        parent: Cap<DEntry>,
        name: NameOwned,
    },
}

pub(crate) struct SuspendedState {
    cursor_cap: Cap<DEntry>,
    remaining: OwnedComponents,
    symlink_budget: u8,
    root_ctx_caps: RootCtxCaps,
    trail_caps: SmallVec<[TrailEntryCap; 8]>,
}
```

Token is sealed. Only kernel_step produces; only `apply_io_result` consumes.

`apply_io_result` downgrades SuspendedState Caps to IdentRef under the fresh guard, applies I/O result, returns fresh WalkState. Caps in SuspendedState drop at end of scope.

---

## 9. Walker Invariants

### 9.1 Purity

kernel_step, accepts, build_witness, classify, apply_io_result perform **no mutation of published structure**. They read structure, construct new values, emit IORequests.

`apply_io_result` may populate the dcache with a newly-looked-up DEntry via observer-safe substrate primitive. Dcache population is informational I/O per ADR §15 — within the race-degradation envelope.

### 9.2 Monotonicity preservation

The walker does not restore any lowered projection. Post-yield resume re-predicates; lowered projections during the yield cause clean `Error(DetachedNamespace)`.

### 9.3 Mount-boundary discipline

Crossing forward pushes `TrailEntry::MountBoundary { was_at: cursor }`; cursor becomes mount root. `..` popping MountBoundary restores cursor to `was_at` (the mountpoint DEntry).

### 9.4 Symlink budget

Decrement-only. Zero-budget substitution is `Error(SymlinkBudgetExceeded)` → ELOOP.

### 9.5 Witness scoping

Witnesses are `!Send + !Sync` via IdentRef. Steps do not `.await` at all (STEP-2); the no-`.await`-on-witness rule is consequently structural, not a runtime discipline. Do not store witnesses in structs, statics, or collections; do not carry witnesses across step boundaries (WIT-3, WIT-4). Cross-step continuation carries `Cap<T>` extracted from the witness before return, not the witness itself.

---

## 10. `require.rs`: Public Surface

### 10.1 Core facades

```rust
pub async fn require_entity<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<EntityAtPath<'g>, Errno>;
pub async fn require_entity_unfollowed<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<EntityAtPath<'g>, Errno>;
pub async fn require_parent_and_name<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<ParentAndName<'g>, Errno>;
pub async fn require_parent_and_named_child<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<ParentAndNamedChild<'g>, Errno>;
pub async fn require_entity_or_parent_and_name<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<EntityOrParentAndName<'g>, Errno>;
pub async fn require_mount_point<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<MountPointAtPath<'g>, Errno>;
pub async fn require_real_path<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<RealPath<'g>, Errno>;
```

Each facade initializes WalkState with its mode, runs walk_to_completion, unpacks the WalkWitness variant.

### 10.2 Refinement wrappers

Core require + post-check predicate, producing a refined witness.

```rust
pub async fn require_directory<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<DirectoryAtPath<'g>, Errno> {
    let e = require_entity(path, ctx).await?;
    if !predicates::is_directory(&e.rnode) { return Err(ENOTDIR); }
    Ok(DirectoryAtPath(e))
}

pub async fn require_symlink_for_readlink<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<SymlinkAtPath<'g>, Errno> {
    let e = require_entity_unfollowed(path, ctx).await?;
    if !predicates::is_symlink(&e.rnode) { return Err(EINVAL); }
    Ok(SymlinkAtPath(e))
}

pub async fn require_unlinkable_non_dir_child<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<UnlinkableNonDirChild<'g>, Errno> {
    let base = require_parent_and_named_child(path, ctx).await?;
    if predicates::is_directory(&base.child.child_rnode()) { return Err(EISDIR); }
    Ok(UnlinkableNonDirChild(base))
}

pub async fn require_rmdirable_dir_child<'g>(path: &[u8], ctx: &ResolveCtx) -> Result<RmdirableDirChild<'g>, Errno> {
    let base = require_parent_and_named_child(path, ctx).await?;
    if !predicates::is_directory(&base.child.child_rnode()) { return Err(ENOTDIR); }
    if !predicates::is_empty_directory(&base.child).await? { return Err(ENOTEMPTY); }
    Ok(RmdirableDirChild(base))
}
```

Refinement wrappers are **not syscall-neutral**. Their errno choices are specific to their use case. If another syscall needs a differently-errno'd refinement, add a new wrapper.

---

## 11. Witness Types

All lifetime-parameterized by `'g`; private constructors via `pub(in crate::vfs::checks)`; `!Send + !Sync` via IdentRef fields.

```rust
pub struct EntityAtPath<'g> {
    pub dentry: IdentRef<'g, DEntry>,
    pub rnode:  IdentRef<'g, RNode>,
}

pub struct DirectoryAtPath<'g>(EntityAtPath<'g>);
pub struct SymlinkAtPath<'g>(EntityAtPath<'g>);

pub struct ParentAndName<'g> {
    pub parent: IdentRef<'g, DEntry>,
    pub name:   NameOwned,
}

pub struct ParentAndNamedChild<'g> {
    pub parent: IdentRef<'g, DEntry>,
    pub child:  IdentRef<'g, DEntry>,
    pub name:   NameOwned,
}

pub struct UnlinkableNonDirChild<'g>(ParentAndNamedChild<'g>);
pub struct RmdirableDirChild<'g>(ParentAndNamedChild<'g>);

pub enum EntityOrParentAndName<'g> {
    Present(EntityAtPath<'g>),
    Absent(ParentAndName<'g>),
}

pub struct MountPointAtPath<'g> {
    pub dentry: IdentRef<'g, DEntry>,
    pub rnode:  IdentRef<'g, RNode>,
}

pub struct RealPath<'g> {
    pub dentry: IdentRef<'g, DEntry>,
    pub path: CanonicalPath,
}

pub(crate) enum WalkWitness<'g> {
    Entity(EntityAtPath<'g>),
    EntityOrParent(EntityOrParentAndName<'g>),
    ParentAndName(ParentAndName<'g>),
    ParentAndNamedChild(ParentAndNamedChild<'g>),
    MountPoint(MountPointAtPath<'g>),
    RealPath(RealPath<'g>),
}
```

Notes:

- **No dyn-entity fields.** Entity-type resolution happens in subsystem checks/ after VFS witness is consumed.
- **No Cap\<Mount\> in MountPointAtPath.** Walker reports the structural observation; mount subsystem's checks/ extracts the Cap.
- **Parent-and-name witnesses carry NameOwned.** Path buffer lifetime ends at syscall boundary; witness may outlive it.

---

## 12. Predicates

```rust
pub fn namespace_live<'g>(d: &IdentRef<'g, DEntry>, ctx: &RootCtxRef<'g>) -> bool;
pub fn payload_live<'g>(r: &IdentRef<'g, RNode>) -> bool;

pub fn is_directory<'g>(r: &IdentRef<'g, RNode>) -> bool;
pub fn is_symlink<'g>(r: &IdentRef<'g, RNode>) -> bool;
pub fn is_regular_file<'g>(r: &IdentRef<'g, RNode>) -> bool;
pub fn is_mount_root<'g>(d: &IdentRef<'g, DEntry>) -> bool;

pub fn traverse_permitted<'g>(d: &IdentRef<'g, DEntry>, cred: &Credential) -> bool;

pub async fn is_empty_directory<'g>(d: &IdentRef<'g, DEntry>) -> Result<bool, Errno>;
// May yield for directory-block read (informational I/O).
```

Single source of truth for projections. Reading `rnode.mode` and testing bits outside this module is a lint violation.

---

## 13. Observer Contract at the Walker

Per LIVENESS §2.6, the walker inherits the observer contract by construction:

- kernel_step re-predicates cursor on every step entry.
- Dcache population uses observer-safe substrate primitives.
- Binding-mutating operations (rename, unlink) use substrate::index::withdraw_commit or swap_commit; walkers that had resolved through a now-mutated binding carry a DEntry whose namespace_live is now false; next re-predicate catches it; clean ENOENT.

Concrete scenarios:

- **unlink(D) commits while walker is at D:** walker's cursor IdentRef is safe to observe (EBR); re-predicate on next kernel_step sees `removed = true` (or substrate has withdrawn D from parent's children container); `DetachedNamespace`; ENOENT.

- **rename(A/B/C, X/Y/Z) commits while walker is mid-traversal through A/B/C/D/E:** walker's trail contains DEntry for C (from forward traversal). `..` pop returns to C's IdentRef; next kernel_step re-predicates C; sees removed; ENOENT. New location X/Y/Z is reachable by fresh walks; in-flight walker does not silently retarget.

---

## 14. Open Questions

- **Parent-pointer rendering for getcwd.** Walker uses trail; rendering needs ancestor traversal. Specification deferred to VFS representation spec.

- **Hot-path specialization.** Fully-cached walks loop without NeedIO; no Cap upgrades. Confirm in implementation.

- **`*at` with AT_EMPTY_PATH.** Handled at syscall-script level; not a WalkMode concern.

- **Canonical-path rendering.** RealPath facade does post-walk rendering. Allocation in facade, tolerated.

- **Magic-link /proc/self/fd/N.** Handled in procfs's InodeOps::lookup; walker is unaware.

---

## 15. Implementation Order

1. `witness.rs` — nail down witness shapes.
2. `predicates.rs` — pure bool functions.
3. `resolution/state.rs` — types only.
4. `resolution/terminal.rs` — accepts, build_witness.
5. `resolution/error.rs` — classify.
6. `resolution/step.rs` — kernel_step.
7. `resolution/driver.rs` — run_walker, resume_walker, walk_to_completion, apply_io_result.
8. `require.rs` — facades.
9. `mod.rs` — exports.

Tests follow each step per SUBSYSTEM_ANATOMY §10.

---

## References

- [`object_model_0417.md`](./object_model_0417.md)
- [`INVARIANTS.md`](./INVARIANTS.md)
- [`STEP_MODEL.md`](./STEP_MODEL.md)
- [`LIVENESS.md`](./LIVENESS.md)
- [`SUBSYSTEM_ANATOMY.md`](./SUBSYSTEM_ANATOMY.md)
