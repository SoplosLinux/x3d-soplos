# x3d-soplos — AMD X3D VCache Scheduler Patch for Soplos Linux

Scheduler patch for Linux 7.x that biases task wake-up towards the VCache CCD
on dual-CCD AMD Ryzen X3D processors, improving gaming and latency-sensitive
workload performance.

> Applied automatically by **soplos-kernel-installer** when compiling the `soplos-x3d` kernel variant (Stock/developer mode).

---

## Supported hardware

| Processor | CCDs | VCache CCD L3 | Standard CCD L3 | Notes |
|-----------|------|---------------|-----------------|-------|
| Ryzen 9 7950X3D | 2 | 96 MB | 32 MB | Supported |
| Ryzen 9 9950X3D | 2 | 96 MB | 32 MB | Supported |
| Ryzen 9 7900X3D | 2 | 64 MB | 32 MB | Supported |
| Ryzen 9 9900X3D | 2 | 64 MB | 32 MB | Supported |
| Ryzen 7 5800X3D | 1 | 96 MB | — | Single-CCD, patch inactive |
| Ryzen 7 7800X3D | 1 | 96 MB | — | Single-CCD, patch inactive |
| Ryzen 7 9800X3D | 1 | 96 MB | — | Single-CCD, patch inactive |

Single-CCD X3D processors are automatically excluded at runtime: all cores
report the same L3 size, so the asymmetry check fails and the feature stays
disabled with zero overhead.

---

## How it works

### Detection

A `late_initcall` runs after all CPUs are online and reads each CPU's L3 cache
size via the kernel `cacheinfo` subsystem. If `max_l3 > 2 * min_l3` the
topology is asymmetric — one CCD carries 3D VCache stacking. The cores with
the larger L3 are recorded in `amd_x3d_vcache_mask`.

The detection requires no firmware cooperation, no ACPI methods, and no
userspace tool — it works purely from CPU cache topology data that the kernel
already has.

### Scheduling

The hook lives in `select_task_rq_fair()` in `kernel/sched/fair.c`, just after
`select_idle_sibling()` has chosen a candidate CPU. If the candidate is not on
the VCache CCD, the patch checks two conditions before redirecting:

1. **Load threshold** — the VCache CCD must not already carry more average load
   than the non-VCache CCD (cross-multiplication avoids integer division).
   Prevents VCache CCD saturation in mixed workloads.

2. **Idle CPU available** — `available_idle_cpu()` confirms a VCache CPU is
   actually idle before redirecting the task there.

If both conditions hold, the task is steered to an idle VCache CPU.

The `static_branch_unlikely` used to guard the entire block compiles to a
single no-op NOP on systems where the feature is inactive (non-AMD, non-Zen3+,
or single-CCD X3D). Zero overhead.

### Modified files

| File | Change |
|------|--------|
| `arch/x86/include/asm/topology.h` | Export `amd_x3d_vcache_active`, `amd_x3d_vcache_mask`, `amd_x3d_vcache_nr`, `amd_x3d_other_nr` |
| `arch/x86/kernel/cpu/Makefile` | Compile `amd_x3d_sched.o` when `CONFIG_CPU_SUP_AMD=y` |
| `arch/x86/kernel/cpu/amd_x3d_sched.c` | New file — detection logic and exported symbols |
| `kernel/sched/fair.c` | Hook in `select_task_rq_fair()` |

---

## Applying the patch manually

```bash
cd /path/to/linux-7.x.y
patch -p1 < /path/to/0001-sched-amd-x3d-vcache.patch
```

The patch applies cleanly to Linux 7.1.x. Minor context adjustments may be
needed for other 7.x minor versions.

---

## Known limitations

- **CPU hotplug** — `amd_x3d_vcache_mask`, `amd_x3d_vcache_nr` and
  `amd_x3d_other_nr` are computed once at boot and not updated if CPUs are
  brought online or offline afterwards. Not relevant for desktop gaming systems.
- **No per-process control** — all tasks are eligible for VCache CCD
  preference, not just gaming processes. The load threshold mitigates this in
  practice. Per-process control via sysfs is planned for a future revision.

---

## Kernel variants using this patch

This patch is used exclusively by the `soplos-x3d` kernel variant compiled and
distributed by Soplos Linux. It is not submitted upstream.

| Kernel name | Patches | March levels |
|-------------|---------|--------------|
| `soplos-x3d` | bore + ntsync + x3d | v3, v4 |

---

## License

GPL-2.0
