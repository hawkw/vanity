# aarch64 WFI/WFE Research for Shard Yield Mechanism

**Date**: 2026-01-18
**Context**: Designing the shard yield/wake mechanism for Vanity

## Background

While designing Vanity's system architecture, we needed to determine how shards
should yield when waiting for work, and whether we could avoid kernel context
switches when a shard is woken by another shard (via IPI/SEV) rather than by a
hardware device interrupt.

The question: Can an aarch64 userspace process use WFI/WFE to wait for events,
and can it be woken by inter-shard notifications without a kernel context switch?

## Findings

### WFI and WFE at EL0 (User Mode)

Both WFI (Wait For Interrupt) and WFE (Wait For Event) can be executed at EL0,
but whether they execute or trap to EL1 is controlled by system registers:

- `SCTLR_EL1.nTWI` (bit 16): if 1, WFI at EL0 executes without trapping
- `SCTLR_EL1.nTWE` (bit 18): if 1, WFE at EL0 executes without trapping

If these bits are set to 0, the instructions trap to EL1 and the kernel can
emulate or skip them. Linux historically allowed these instructions but later
changed to trapping and skipping them (effectively making them NOPs).

**Source**: https://stackoverflow.com/questions/64502103/aarch64-esr-trapped-wfi-or-wfe-instruction-execution

### WFE Wake Sources

WFE can be woken by:
- An exception (which causes EL0→EL1 transition)
- An interrupt becoming pending (if SEVONPEND in System Control Register is set)
- A SEV (Send Event) instruction from another core
- An external event signal
- The event register already being set

Crucially, **SEV is an unprivileged instruction** that can be executed from EL0.

### Device Interrupts Require Kernel Involvement

When a hardware interrupt fires, the CPU takes an exception to EL1 (or higher).
This is architecturally required — there is no mechanism on aarch64 to deliver
hardware interrupts directly to EL0 userspace without kernel involvement.

Flow for device interrupt wake:
1. Shard executes WFE, enters low-power wait
2. Device interrupt fires
3. CPU wakes, takes exception to EL1 (kernel)
4. Kernel handles/acknowledges interrupt
5. Kernel returns to EL0 (shard resumes)

### Inter-Shard Notification Can Avoid Kernel

For shard-to-shard communication using SEV:
1. Shard A writes data to shared channel
2. Shard A executes SEV (unprivileged)
3. Shard B (waiting in WFE) wakes up without exception
4. Shard B checks channel, finds work, processes it

If `SCTLR_EL1.nTWE = 1`, this flow involves **no kernel context switch**.

### GICv4/4.1 Direct Virtual Interrupt Injection

ARM's GICv4/4.1 supports "direct injection of virtual interrupts" to virtual PEs
without hypervisor intervention. However, this is specifically for EL2→EL1
delivery (hypervisor to guest VMs), not EL1→EL0 (kernel to userspace). It does
not help our use case.

**Source**: https://neoverse-reference-design.docs.arm.com/en/main/features/virtualization/gicv4_1-vlpi-vsgi.html

## Design Implications

On aarch64, if a shard doesn't know whether its next wake will come from hardware
or from another shard, it can use WFE and:

- If woken by SEV from another shard: no context switch, immediate resume
- If woken by device interrupt: context switch to kernel is unavoidable, but
  happens only when the interrupt actually fires

This is better than immediately trapping to the kernel on yield, because the
common case (inter-shard communication in a pipeline) avoids kernel involvement.

**Important**: This behavior is aarch64-specific. Other architectures (RISC-V,
x86_64) may have different mechanisms or constraints. The cross-platform kernel
crate should be agnostic to the specific yield mechanism, allowing each platform
to implement the optimal strategy for its architecture.

## Open Questions

- What is the equivalent mechanism on RISC-V? Does WFI have similar properties?
- On x86_64, what are the options? MONITOR/MWAIT? PAUSE? Ring-0 requirement?
- Should we expose the yield mechanism choice to userspace, or hide it entirely?
