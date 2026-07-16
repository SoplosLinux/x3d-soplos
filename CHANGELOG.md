# Changelog — x3d-soplos

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
