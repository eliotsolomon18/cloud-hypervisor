# Fix: `pause; snapshot; resume` hangs guest virtio-fs I/O

## Problem

`ch-remote pause; ch-remote snapshot <dir>; ch-remote resume` against a
cloud-hypervisor VM with a vhost-user-fs device leaves any guest virtio-fs I/O
issued after the resume permanently hung in `D` state. Existing live-migration
and snapshot-then-restore-on-fresh-VMM paths are unaffected; the bug is
specific to in-place resume after a snapshot on the same VMM.

The chain of events that produces the hang:

1. `Fs::snapshot` → `VhostUserCommon::snapshot` → `state()` →
   `VhostUserHandle::save_backend_state` issues `GET_VRING_BASE` for each
   queue. The `vhost-user-backend` crate (used by virtiofsd) responds by
   setting `queue_ready=false`, **unregistering the kick FD from epoll, and
   dropping both the kick and call FDs** on the backend side.
   `save_backend_state` then sets `self.ready = false` on the handle and sends
   `SET_DEVICE_STATE_FD(SAVE, STOPPED)` to transfer state.
2. `Fs::resume` → `VhostUserCommon::resume` →
   `VhostUserHandle::resume_vhost_user` is gated on `self.ready`, so it is a
   no-op after a snapshot. Even without that gate, the only message it sends
   is `SET_VRING_ENABLE(1)` — insufficient by itself, because the backend has
   already dropped the kick FD and unregistered it from its event loop, so
   guest doorbell writes go nowhere.

The integration test added in the PR that introduced vhost-user
snapshot/restore (PR #7908: `test_snapshot_restore_virtio_fs`,
`cloud-hypervisor/tests/integration.rs:8695`) exercises only the
snapshot → kill source → restore-on-fresh-VMM → resume path. The in-place
pause/snapshot/resume path is uncovered, which is why this regression went
unnoticed.

## Proposed Solution

In the resume path, when the backend has been suspended via
`save_backend_state`, replay the per-queue tail of `setup_vhost_user`:
`SET_VRING_NUM`, `SET_VRING_ADDR`, `SET_VRING_BASE` (with the indices captured
at snapshot time), `SET_VRING_CALL`, `SET_VRING_KICK`, then
`SET_VRING_ENABLE(1)` for all queues. No `SET_DEVICE_STATE_FD(LOAD)` round-trip
is needed — the backend retains its filesystem-level state across SAVE.

To make this possible, two pieces of state are retained on `VhostUserHandle`,
captured at the points where they are naturally produced:

- `kick_evts: Vec<EventFd>` — clones of the per-queue kick eventfds, captured
  inside `setup_vhost_user`. Living on `VhostUserHandle` (rather than on
  `VhostUserCommon` one level up) means HUP-reconnect — which re-runs
  `setup_vhost_user` via `reinitialize_vhost_user` — automatically refreshes
  them in lockstep with the existing `vrings_info` and `queue_indexes`.
- `saved_vring_bases: Option<Vec<u64>>` — the non-empty per-queue indices
  captured by `save_backend_state`. Doubles as the discriminator at resume
  time: non-empty `Some(...)` ⇒ active vrings were stopped by snapshot (call
  new `restart_vrings`); `None` ⇒ plain pause or a pre-activation snapshot
  with no vrings to restart (call existing `resume_vhost_user`).

A new `restart_vrings(&mut self, virtio_interrupt: &dyn VirtioInterrupt)`
method on `VhostUserHandle` replays the wire sequence using the saved bases,
`self.kick_evts`, `self.vrings_info`, and `self.queue_indexes`. It leaves the
retained restart state untouched on error so resume can be retried, and clears
the saved bases only after the restart succeeds.
The four-call per-queue body is duplicated once with `setup_vhost_user`'s
inlined version (~8 LoC) — the alternative (extracting a shared helper)
increases the activate hot path edit surface without proportional benefit.

The call FD comes from `VirtioCommon::interrupt_cb`, which is already retained
one level up in `VhostUserCommon` and reachable from `VhostUserCommon::resume`
without plumbing changes.

### Lifecycle of `saved_vring_bases`

- `setup_vhost_user`: clears (`= None`) at the end, alongside replacing
  `kick_evts`.
- `save_backend_state`: writes (`= Some(bases)`) only when `GET_VRING_BASE`
  captured at least one active queue; leaves it `None` for pre-activation
  snapshots with no vrings to restart.
- `restart_vrings`: clones/borrows retained restart state while replaying the
  setup sequence; clears `saved_vring_bases` and sets `ready = true` only after
  success.
- `pause_vhost_user` / `resume_vhost_user`: untouched (plain pause/resume path
  is unaffected).

### One pre-existing latent bug fixed in passing

`setup_vhost_user` previously called `self.queue_indexes.push(...)` without
clearing first. Re-runs (HUP-reconnect via `reinitialize_vhost_user`)
accumulated duplicates, which would have caused snapshot-after-HUP →
restore-on-fresh-VMM to fail length-validation in the existing
`VringBasesCountMismatch` check. Adding `self.queue_indexes.clear();` at the
top of the per-queue loop fixes this as a side benefit and is required for
`kick_evts.len() == queue_indexes.len()` to hold under our new code.

## Assumption

The fix is built on one assumption about vhost-user back-end behavior:

> After `SET_DEVICE_STATE_FD(SAVE, STOPPED)` followed by
> `CHECK_DEVICE_STATE`, a conformant vhost-user back-end retains enough
> internal state (i.e. is non-destructive on SAVE) that the source can resume
> processing by re-issuing the standard ring-arming sequence (`SET_VRING_NUM`
> → `SET_VRING_ADDR` → `SET_VRING_BASE` → `SET_VRING_KICK` → `SET_VRING_CALL`
> → `SET_VRING_ENABLE(1)`), without a corresponding `LOAD` operation.

This assumption is necessary for the fix to work as designed. If a back-end
genuinely destroys state on SAVE, the only correct path on resume would be
LOAD with the saved blob — which is what the existing restore-on-fresh-VMM
path does. The evidence below establishes that the assumption holds for
virtiofsd specifically and is required by the vhost-user spec for any
conformant back-end.

## Evidence

### vhost-user spec

The QEMU vhost-user protocol spec ("Migrating back-end state" section) is
explicit:

> *"If the migration fails, then the source can transparently resume
> operation until another migration attempt is made."*

For this guarantee to be meaningful, conformant back-ends must preserve
internal state across a SAVE that does not lead to a successful migration.
This is normative spec language, not implementation detail.

The same spec describes the start/stop mechanics:

> *"The back-end must start a ring upon receiving a kick … and stop a ring
> upon receiving `VHOST_USER_GET_VRING_BASE`."*

So the canonical way to bring a ring back from "stopped" is to re-establish
the kick FD and let the next kick re-arm processing — exactly the sequence
this fix replays.

### QEMU reference implementation

`hw/virtio/vhost.c`, in `vhost_virtqueue_start` (called by `vhost_dev_start`),
restarts a previously-stopped ring by re-running the full setup sequence:
`SET_VRING_NUM` → `SET_VRING_BASE` (using the saved `last_avail_idx`) →
`SET_VRING_ADDR` → `SET_VRING_KICK` → `SET_VRING_CALL`. No
`SET_DEVICE_STATE_FD(LOAD)` is involved. This is the same operation the new
`restart_vrings` performs, against the same protocol.

### virtiofsd documentation

`virtiofsd-source/doc/migration.md` explicitly treats source-resumption as a
first-class scenario:

> *"the destination instance's `--migration-on-error` switch; `abort` will
> abort migration (on the destination) when any error occurs, **allowing
> execution to be resumed on the source side, with any FD still open**."*

> *"`abort`: When any error is encountered (e.g. destination cannot find a
> file that is open in the guest), abort migration altogether. **You can then
> generally resume execution on the source; the source virtiofsd instance
> will retain all open file descriptors until it is quit.**"*

`virtiofsd-source/src/vhost_user.rs:531-538` contains an explicit author note
about this scenario:

> *"… QEMU will clear F_LOG_ALL only when the VM is running, i.e. when the
> source resumes after a cancelled migration, which is exactly what we
> want…"*

### virtiofsd implementation

`virtiofsd-source/src/passthrough/device_state/mod.rs:72-88` shows that
`serialize()` is non-destructive: it constructs `serialized::PassthroughFs::V2`
from `&self`, writes the bytes, and clears only the bookkeeping
`migration_info` annotations. The actual state — `inodes`, `handles`,
`next_inode`, `next_handle`, `cfg` — is untouched. The corresponding
`From<&PassthroughFs> for serialized::PassthroughFsV2` in
`serialization.rs:49-107` is a pure read of the live tables.

In other words, after `SAVE` the FUSE server is fully alive in process memory
with all its filesystem state intact. Only the rings have been torn down (by
the prerequisite `GET_VRING_BASE`). Re-arming the rings is sufficient to
resume.

### Backend-side teardown on `GET_VRING_BASE`

`vhost-user-backend-0.21.0/src/handler.rs:422-454` (the version pinned by
virtiofsd's `Cargo.toml`) shows that `GET_VRING_BASE` does three things to
each queue:

1. `vring.set_queue_ready(false)` — disarms the queue.
2. `unregister_event(fd.as_raw_fd(), EventSet::IN, ...)` — removes the kick FD
   from the epoll handler.
3. `vring.set_kick(None); vring.set_call(None)` — drops both FDs entirely.

Step 3 is why `SET_VRING_ENABLE(1)` alone is insufficient on resume:
`set_vring_enable` (line 517-534) only flips a flag; it does not re-register
the kick FD with epoll. Re-registration happens only inside `initialize_vring`
(line 206-225), which is called from `set_vring_kick` and `set_vring_call`.
Therefore the resume path **must** re-issue `SET_VRING_KICK`, which is
exactly what the new `restart_vrings` does.

## Changes

### `virtio-devices/src/vhost_user/vu_common_ctrl.rs`

- Removed `#[derive(Clone)]` from `VhostUserHandle`. Verified unused — the
  only `vu.clone()` site in `mod.rs` clones the wrapping `Arc`. The derive is
  incompatible with the new `Vec<EventFd>` field because `EventFd` is not
  `Clone`.
- Added `kick_evts: Vec<EventFd>` and `saved_vring_bases: Option<Vec<u64>>`
  fields, initialized in both `connect_vhost_user` constructors.
- `setup_vhost_user`:
  - Added `self.queue_indexes.clear();` at the top of the per-queue loop
    (latent bug fix; see "One pre-existing latent bug" above).
  - Captured `queue_evt.try_clone()?` for each queue into a local
    `kick_clones: Vec<EventFd>` (new error variant `CloneKickEventFd`).
  - At the end of the function, alongside the existing
    `self.vrings_info = Some(vrings_info); self.ready = true;` writes:
    `self.kick_evts = kick_clones; self.saved_vring_bases = None;`.
- `save_backend_state`: stashes `self.saved_vring_bases = Some(vring_bases.clone())`
  only when at least one vring base was captured; pre-activation snapshots keep
  it as `None` so in-place resume does not require an `interrupt_cb`.
- Added accessor `pub fn has_saved_vring_bases(&self) -> bool` next to
  `supports_device_state`; it returns true only for non-empty saved bases.
- Added new method `pub fn restart_vrings(&mut self, virtio_interrupt: &dyn VirtioInterrupt)`
  after `restore_backend_state`. Body: clones `saved_vring_bases`, clones
  `vrings_info`, borrows `kick_evts`, clones `queue_indexes`; length-validates
  all four; issues `SET_VRING_NUM` for every queue first, then per-queue
  `SET_VRING_ADDR` / `SET_VRING_BASE` / `SET_VRING_CALL` (gated on
  `virtio_interrupt.notifier(...)`) / `SET_VRING_KICK`; calls existing
  `enable_vhost_user_vrings(true)`; clears `saved_vring_bases` and sets
  `ready = true` only after success. If any step fails, the retained bases and
  kick FDs remain available for a retry.

### `virtio-devices/src/vhost_user/mod.rs`

- Added three error variants:
  `MissingSavedVringBases`, `MissingVringsInfo`,
  `CloneKickEventFd(#[source] io::Error)`.
- `VhostUserCommon::resume` now branches on `vu_locked.has_saved_vring_bases()`:
  on `true` (non-empty saved bases), calls `restart_vrings` with
  `interrupt_cb.as_ref()` borrowed (not cloned) from
  `VirtioCommon::interrupt_cb`; otherwise keeps the existing
  `resume_vhost_user` call. The trailing per-queue `trigger_interrupt` loop is
  unchanged.

### `cloud-hypervisor/tests/integration.rs`

- Added `test_pause_snapshot_resume_virtio_fs` in `common_sequential`
  immediately after `test_snapshot_restore_virtio_fs`, gated by
  `#[test] #[cfg(not(feature = "mshv"))]`. Boots a VM with virtio-fs, mounts
  it in the guest, writes a marker file, calls
  `snapshot_restore_common::snapshot_and_check_events` (existing helper),
  resumes via `remote_command(&api_socket, "resume", None)` against the
  **same** API socket (no kill, no fresh VMM), waits for the `resumed`
  event, then reads/writes through the virtio-fs mount. Post-resume guest
  commands are wrapped in `timeout 10` so a hang surfaces as
  `NonZeroExitStatus(124)` and a clean test failure rather than blocking
  indefinitely.

### Untouched

- `VhostUserCommon` struct: no new fields.
- `VhostUserCommon::activate`, `state`, `reset`, `shutdown`: bodies unchanged.
- `VringInfo` struct.
- `Fs::resume`, `Blk::resume`, `Net::resume`, `GenericVhostUser::resume`: the
  fix is centralized in `VhostUserCommon::resume`, which all four call.
- HUP-reconnect path (`VhostUserEpollHandler::reconnect` →
  `reinitialize_vhost_user` → `setup_vhost_user`): inherits the fix
  automatically because `setup_vhost_user` repopulates `kick_evts` and clears
  `saved_vring_bases`.

## Risk Analysis

- **Live migration (PR #7908):** unchanged. `VhostUserCommon::snapshot` calls
  `self.shutdown()` when `migration_started == true`, dropping `self.vu` and
  the new fields with it. The destination constructs a fresh handle from the
  snapshot blob via `Fs::new` — additions are invisible to the receiving
  side.
- **Plain pause/resume (no snapshot):** unchanged. `pause_vhost_user` only
  sends `SET_VRING_ENABLE(0)`, `saved_vring_bases` stays `None`, and the
  existing `resume_vhost_user` branch runs.
- **Pre-activation pause/snapshot/resume:** unchanged. `save_backend_state`
  still returns `vring_bases = Some(vec![])` in the snapshot state for
  consistency with `backend_state`, but the live handle keeps
  `saved_vring_bases = None`, so resume does not require an `interrupt_cb`.
- **Resume retry after backend restart error:** improved. `restart_vrings`
  leaves `saved_vring_bases` and `kick_evts` intact on error, so a later
  `resume` attempt can retry the full ring-arming sequence instead of falling
  through to the `ready=false` no-op path.
- **Source-side resume after a *failed* live migration:** today silently
  broken (the same `ready=false` no-op); with this fix, works correctly per
  spec. Net positive.
- **HUP-reconnect mid-life:** `setup_vhost_user` re-runs via
  `reinitialize_vhost_user`, repopulating `kick_evts` / `vrings_info` /
  `queue_indexes` (now correctly cleared first). Any stale `saved_vring_bases`
  is also cleared, so a subsequent resume takes the plain branch as expected.
- **HUP during pause-with-snapshot:** unrecoverable regardless of this fix
  — the backend state captured by SAVE no longer corresponds to the freshly
  reconnected backend. The user must shut down and start from a fresh
  snapshot.

## Verification

- `cargo fmt --check`: clean (stable rustfmt emits warnings about ignored
  nightly-only import options).
- `git diff --check`: clean.
- `cargo check -p virtio-devices`: clean.
- `cargo test -p virtio-devices vhost_user`: clean; the filter matches no
  unit tests, but the test binary compiles.
- The new integration test was **not** executed locally — it requires the
  `~/workloads/` test environment (Ubuntu Jammy image, virtiofsd, etc.) and
  is run via `./scripts/dev_cli.sh tests --integration --test-filter test_pause_snapshot_resume_virtio_fs`.
  Maintainer CI is expected to validate.

### Manual reproduction recipe

```sh
# Start virtiofsd
virtiofsd --socket-path=/tmp/vfsd.sock --shared-dir=/tmp/share --cache=auto

# Boot a VM with virtio-fs
cloud-hypervisor \
    --api-socket /tmp/ch.sock \
    --kernel <vmlinux> \
    --disk path=<rootfs> \
    --memory size=512M,shared=on \
    --fs socket=/tmp/vfsd.sock,tag=myfs,num_queues=1,queue_size=1024

# In the guest:
mount -t virtiofs myfs /mnt
dd if=/dev/zero of=/mnt/pre bs=4k count=1   # baseline I/O succeeds

# Pause, snapshot, resume in place:
ch-remote --api-socket /tmp/ch.sock pause
ch-remote --api-socket /tmp/ch.sock snapshot file:///tmp/snap
ch-remote --api-socket /tmp/ch.sock resume

# In the guest, *before* this fix:
dd if=/dev/zero of=/mnt/post bs=4k count=1   # hangs in D state forever

# In the guest, *after* this fix:
dd if=/dev/zero of=/mnt/post bs=4k count=1   # completes immediately
ls /tmp/share/post                            # confirms write reached the host
```

## Relevant PRs / Issues

- **#7908** (`Fill out snapshot/restore support for vhost-user devices`,
  merged 2026-03-27, author: rbradford). Introduced
  `SET_DEVICE_STATE_FD(SAVE/LOAD)` support in cloud-hypervisor and added the
  one existing integration test (`test_snapshot_restore_virtio_fs`). The new
  bug originates here: `save_backend_state` issues the protocol but no symmetric
  resume path exists for the in-place case.
- **#7850** (`Pause/resume & snapshot/restore with vhost-user devices`,
  open, author: rbradford — same author as #7908). Parent tracking issue.
  Explicitly states: *"I've also identified some issues with pause/resume
  with vhost-user devices. We should add some tests to our integration test
  suite for vhost-user devices (at least virtiofsd) and doing
  snapshot/restore with those."* This PR addresses the in-place
  pause/snapshot/resume case directly and adds the requested test.
- **#6931** (`Unable to restore a snapshot of vm using virtiofs root`,
  closed). A different but related bug in the snapshot+restore flow — the
  underlying cause was a missing reply in the `vhost` crate's REPLY_ACK
  handling, fixed upstream by **rust-vmm/vhost PR #290**
  (`vhost_user: fix replies without GET_PROTOCOL_FEATURES`, merged
  2025-06-03 by Alyssa Ross). The issue closed once virtiofsd picked up a
  vhost crate version that included the fix. Useful background for the
  snapshot-on-virtio-fs interaction; not a code dependency of this PR.
- **#7104** (`Fix Resolve Snapshot Restore Failure with virtiofsd`,
  closed unmerged). An earlier attempt to address #6931's symptoms by
  re-ordering protocol-feature negotiation in cloud-hypervisor itself.
  Author moved away from virtiofsd in their downstream use case; the
  upstream fix instead landed via rust-vmm/vhost PR #290. The issue chased
  there is orthogonal to this one.

## Spec / external references

- QEMU vhost-user protocol spec, "Migrating back-end state" section:
  https://qemu-project.gitlab.io/qemu/interop/vhost-user.html#migrating-back-end-state
- QEMU reference frontend implementation:
  `hw/virtio/vhost.c::vhost_virtqueue_start` — the same per-queue restart
  pattern this fix replays.
- virtiofsd migration documentation: `virtiofsd-source/doc/migration.md`.
