# Changelog — x3d-soplos

## 1.1.1 — 2026-07-25

### Fixed

- `patches/0001-sched-amd-x3d-vcache-7.1.patch`: the `dequeue_task_fair()`
  hunk in `kernel/sched/fair.c` needed `patch`'s fuzzy matching (fuzz 2) to
  apply against Linux 7.1.5 — a stable-tree backport removed
  `util_est_update(&rq->cfs, p, flags & DEQUEUE_SLEEP)` from that function
  (moved into `update_load_avg()` behind a new `UPDATE_UTIL_EST` flag),
  which was this hunk's anchor. Regenerated the hunk with the correct
  context (the surrounding `/* Must not reference @p ... */` comment and
  `return true; }`, both unchanged across the refactor) so it now applies
  with plain line-offset, zero fuzz, against 7.1.5. No change to the X3D
  atomic-counter logic itself (`amd_x3d_vcache_running`/`amd_x3d_other_running`
  increment/decrement calls are unchanged).
- The other two hunks in the same file (`enqueue_task_fair`,
  `select_task_rq_fair`) were unaffected — offset only, no fuzz, both before
  and after this fix.
- Re-verified the same patch file against Linux 7.1.0, 7.1.1, 7.1.3 and
  7.1.4 after the fix: still applies with offset only on all of them (the
  util_est refactor only landed in the 7.1.5 stable backport), so this is
  not a breaking change for earlier 7.1.x point releases.

---

## 1.1.0 — 2026-07-17

### Changed

- Replaced single `0001-sched-amd-x3d-vcache.patch` with three version-specific patch files:
  - `0001-sched-amd-x3d-vcache-6.x.patch` — Linux 6.12.x, 6.18.x
  - `0001-sched-amd-x3d-vcache-7.0.patch` — Linux 7.0.x
  - `0001-sched-amd-x3d-vcache-7.1.patch` — Linux 7.1.x, 7.2-rc
  - Necessary because `select_task_rq_fair()` fast path structure and the
    `topology.h` insertion point differ between kernel families.

### Improved

- Hot path performance: replaced `for_each_cpu()` load loops in
  `select_task_rq_fair()` with two `atomic_read()` calls — O(1) regardless of
  CPU count. Atomic counters (`amd_x3d_vcache_running`, `amd_x3d_other_running`)
  are maintained by new hooks at the end of `enqueue_task_fair` and before
  `return true` in `dequeue_task_fair`.
- Added `#include <linux/atomic.h>` and extern declarations for both atomic
  counters to `arch/x86/include/asm/topology.h`.

---

## 1.0.0 — 2026-07-16

Initial release.

### Added

- `0001-sched-amd-x3d-vcache.patch` — AMD X3D VCache scheduler preference for Linux 7.x.
  - New file `arch/x86/kernel/cpu/amd_x3d_sched.c`: detects asymmetric L3 cache
    topology on dual-CCD Ryzen X3D processors via the kernel cacheinfo subsystem
    at `late_initcall` time. Populates `amd_x3d_vcache_mask` with the cores
    belonging to the VCache CCD.
  - Hook in `kernel/sched/fair.c` (`select_task_rq_fair`): redirects waking
    tasks to an idle VCache CPU when the VCache CCD is not already carrying more
    average load than the non-VCache CCD (load threshold via cross-multiplication
    to avoid integer division).
  - `static_branch_unlikely` guard: zero overhead on non-AMD, non-Zen3+, and
    single-CCD X3D systems.
  - CPU counts (`amd_x3d_vcache_nr`, `amd_x3d_other_nr`) cached at boot to
    reduce per-wakeup work in the scheduler hot path.
  - Supports: Ryzen 9 7950X3D, 7900X3D, 9950X3D, 9900X3D (dual-CCD).
  - Automatically inactive on single-CCD X3D (5800X3D, 7800X3D, 9800X3D).
  - Targets Linux 7.1.x (Soplos Linux kernel).

### Notes

- Not submitted upstream. Soplos Linux patch.
- Signed-off-by: Sergi Perich <info@soploslinux.com>
