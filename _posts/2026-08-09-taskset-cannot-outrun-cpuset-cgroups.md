---
layout: post
title: "Why taskset Could Not Fix a DPDK Affinity Failure"
date: 2026-08-09
categories: linux debugging dpdk infrastructure
---

A DPDK service aborted in `rte_eal_init()` with one useful line hidden in a redirected log:

```
PANIC in rte_eal_init(): Cannot set affinity
```

The machine had all expected CPUs online. The DPDK core mask was valid. Running the launcher through `taskset 0xffff` still left the main lcore on CPU 0 and the process still aborted.

The missing layer was a cpuset cgroup. `taskset` can narrow a process inside its cgroup, but it cannot grant CPUs that the cgroup has already removed.

---

## The misleading first hypotheses

An early backtrace ended here:

```
__rte_panic
rte_eal_init
app_dpdk_init
main
```

That establishes where the process stopped, not why. DPDK's Environment Abstraction Layer (EAL) initializes hugepages, devices, lcores, shared runtime files, and telemetry. A failure at `rte_eal_init()` therefore has several plausible causes.

We first investigated hugepages because the launch script requested more pages than the current pool contained. Increasing the pool changed nothing. The process aborted at the same point, and the larger reservation reduced memory available to `/tmp`. We restored the original value.

That experiment was still useful: it converted a plausible story into a ruled-out hypothesis. The important correction was to stop treating the backtrace as the error message and inspect the application's redirected EAL log.

The log contained both the requested mask and the effective main-lcore placement:

```
EAL: Detected lcore 0 ...
EAL: Main lcore 0 is ready (tid=..., cpuset=[0])
PANIC in rte_eal_init(): Cannot set affinity
```

A multi-core mask combined with `cpuset=[0]` is the clue.

---

## CPU affinity has two gates

Linux exposes a per-thread CPU affinity mask through `sched_setaffinity(2)`. Tools such as `taskset` use that interface.

A cpuset cgroup is a separate, higher-level constraint. The effective CPUs are approximately:

```
effective CPUs = requested affinity ∩ online CPUs ∩ cpuset CPUs
```

If a service asks for CPU 7 but its current cpuset contains only CPU 0, privileges do not make CPU 7 available. The kernel filters affinity requests through the task's cpuset. DPDK then fails when it cannot bind one of its lcore threads as configured.

This is why the obvious command did not work:

```bash
taskset 0xffff /opt/app/start.sh
```

The launcher inherited a cgroup that allowed only CPU 0. Its child inherited the same cgroup. `taskset` changed the requested mask, not the cgroup boundary.

---

## The diagnostic sequence

Start with the process view:

```bash
cat /proc/self/status
```

The relevant fields are:

```
Cpus_allowed:      00000001
Cpus_allowed_list: 0
```

Then identify the process's cgroups:

```bash
cat /proc/self/cgroup
```

On a cgroup-v1 system, the result may include something like:

```
2:cpuset:/restricted-shells
```

Read that cpuset rather than the root cpuset:

```bash
cat /sys/fs/cgroup/cpuset/restricted-shells/cpuset.cpus
```

If this prints `0`, the restriction is explained. Reading only `/sys/fs/cgroup/cpuset/cpuset.cpus` can mislead you because the root may contain every CPU while the process lives in a narrower child.

A small inheritance test makes the boundary visible:

```bash
taskset 0xffff cat /proc/self/status
```

If the child still reports `Cpus_allowed_list: 0`, stop changing DPDK masks. The failure is above the application.

For a live process, compare three facts:

```bash
pid=$(pgrep -x my_dpdk_app)
taskset -pc "$pid"
cat "/proc/$pid/cgroup"
cat "/proc/$pid/status"
```

The first reports thread affinity. The other two reveal the cgroup that bounds it.

---

## The correct launch boundary

The preferred fix is to use the platform's service supervisor. A supervisor can place the service in its intended cpuset before executing the binary. Starting the same script from an administrative SSH shell may create a different process tree and therefore a different resource policy.

When no supervisor entry point is available, a controlled diagnostic on cgroup v1 is to move only the launching shell to a cpuset that contains the required CPUs, launch the existing script, and immediately restore the shell:

```bash
# Root privileges required. Adapt paths and CPU list to the machine.
echo $$ > /sys/fs/cgroup/cpuset/tasks
taskset -pc 0-15 $$

/opt/app/start.sh

echo $$ > /sys/fs/cgroup/cpuset/restricted-shells/tasks
taskset -pc 0 $$
```

Children inherit their parent's cpuset and affinity across `fork()` and `execve()`, so the background service keeps the launch-time placement after the shell returns to its restricted cgroup.

This is a diagnostic workaround, not a universal service design. Keep the widened interval short. Do not run unrelated commands between the two cgroup moves. On systems with `cgexec` or systemd, prefer launching only the child in the intended cgroup rather than moving the interactive shell.

The verification must cover behavior, not just process existence:

```bash
pgrep -af my_dpdk_app
ss -lx | grep my_service_socket
cat /proc/$(pgrep -x my_dpdk_app)/status
```

In this case the process remained live, its Unix service socket appeared, DPDK telemetry came up, and every requested lcore reported ready.

---

## A reset is part of the host failure domain

A second lesson came from using a device-level system reset as a clean-start discriminator. This was not a standalone PCIe reset. The device was still attached to a live virtualization host; resetting the device disrupted the PCIe relationship and the host crashed.

The practical rule is simple: before resetting a PCIe-attached accelerator, identify the reset domain and the host's expected behavior. Have out-of-band host access available, stop workloads, and follow the platform's host-aware reset sequence. Do not assume that an out-of-band device command is isolated from the host merely because it is issued outside the host OS.

A clean-reboot experiment can separate stale runtime state from a reproducible defect, but only after the reboot procedure itself is known to be safe.

---

## The reusable checklist

| Question | Evidence |
|---|---|
| Where did initialization stop? | Native backtrace |
| What did EAL report before aborting? | Complete redirected stdout/stderr |
| Which CPUs did the app request? | `-c`, `-l`, or `--lcores` arguments |
| Which CPUs can this process use? | `/proc/<pid>/status` |
| Which cpuset owns the process? | `/proc/<pid>/cgroup` |
| What does that cpuset allow? | Its own `cpuset.cpus`, not the root file |
| Does `taskset` actually widen a child? | `taskset ... cat /proc/self/status` |
| Was a hypothesis reversed after testing? | Restore changed state and record the retraction |
| Can the reset affect the host? | Platform reset-domain documentation and OOB access |

The central debugging lesson is that process launch context is part of application configuration. The same binary, arguments, hugepages, and hardware can behave differently when started by a service supervisor versus an SSH shell because the inherited cgroup is different.

## Source

- [Linux kernel cpuset documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cpusets.html)
- [`sched_setaffinity(2)` Linux manual page](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)
- [DPDK EAL parameters](https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html)
