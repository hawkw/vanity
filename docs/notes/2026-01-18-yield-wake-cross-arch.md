# Cross-Architecture Yield/Wake Mechanisms Research

**Date**: 2026-01-18  
**Context**: Extending the WFI/WFE research to cover RISC-V and x86_64 for Vanity's
cross-platform shard yield/wake design.

## Summary

This document captures research on how different architectures handle low-power
wait states and inter-processor wake mechanisms. The goal is to understand what
primitives Vanity can use on each platform to implement efficient shard yielding
and waking.

## aarch64 (Recap)

See [2026-01-18-wfi-wfe-research.md](./2026-01-18-wfi-wfe-research.md) for details.

**Key points:**
- WFE (Wait For Event) can be executed at EL0 if `SCTLR_EL1.nTWE = 1`
- SEV (Send Event) is unprivileged — can be executed from EL0
- SEV is a **global broadcast** — wakes all cores, not targeted
- WFE wakes on: SEV, interrupts, exceptions, event register already set
- Device interrupts require EL0→EL1 transition (kernel involvement unavoidable)
- Inter-shard wakes via SEV can be kernel-free

**Design implication:** The broadcast nature of SEV means spurious wakeups are
possible (and expected). Shards must handle spurious wakes gracefully — check
channels, go back to sleep if no work.

## RISC-V

### WFI Behavior

RISC-V's WFI (Wait For Interrupt) is the primary low-power wait instruction.

**Privilege requirements:**
- WFI is a **privileged instruction** in the base spec
- When executed in U-mode (user mode), behavior depends on `mstatus.TW` (Timeout Wait):
  - `TW=1`: WFI traps to M-mode if it doesn't complete within an implementation-
    specific bounded time
  - `TW=0`: WFI in U-mode causes an illegal instruction exception (with S-mode
    present) OR may be allowed as a hint/NOP

**From the RISC-V Privileged Spec (section 3.3.3):**
> "The WFI instruction provides a hint to the implementation that the current
> hart can be stalled until an interrupt might need servicing."

**Key difference from ARM:** WFI is architecturally not designed for U-mode use.
There is no direct equivalent to WFE/SEV for user-mode wait/wake.

### Wake Mechanisms

RISC-V wakes from WFI when:
- An interrupt becomes pending (even if not enabled for taking)
- An NMI occurs
- A debug request occurs
- Implementation-specific events

**No equivalent to SEV:** RISC-V has no unprivileged inter-hart wake mechanism
in the base ISA. Inter-processor interrupts (IPIs) must go through the interrupt
controller.

### RISC-V IPIs via PLIC/CLINT

The SiFive CLINT (Core Local Interruptor) provides software interrupts:
- Each hart has a `msip` (machine software interrupt pending) bit
- Writing to another hart's `msip` generates an IPI
- This is M-mode accessible only — cannot be done from U-mode

The PLIC (Platform-Level Interrupt Controller) routes external interrupts but
doesn't directly provide IPI functionality.

**RISC-V AIA (Advanced Interrupt Architecture):** The newer AIA spec includes
IMSIC (Incoming MSI Controller) which provides message-signaled interrupts. This
could potentially be used for more efficient IPIs, but still requires privilege.

### Design Implications for Vanity on RISC-V

**Option A: Trap to kernel on yield, kernel uses WFI**
- Shard calls kernel to yield
- Kernel executes WFI on that hart
- Any wake (IPI or device) traps to kernel, kernel resumes shard
- Downside: kernel involvement on every yield/wake cycle

**Option B: Busy-poll with PAUSE hint**
- Shard polls its channels in a loop
- Use memory ordering fences for synchronization
- No low-power wait state
- Downside: wastes power, not suitable for idle shards

**Option C: Hybrid polling with periodic yield**
- Shard busy-polls for a short time
- If no work, yield to kernel
- Kernel uses WFI
- Balances latency vs power

**Likely approach:** For RISC-V, we cannot avoid kernel involvement in the
yield/wake path. The design should minimize the cost of this path:
- Use fast syscall entry/exit
- Kernel WFI, wake on any interrupt
- Quick return to userspace after wake

## x86_64

### Historical Context

x86 has evolved significantly in its wait/wake capabilities:
- **HLT**: Privileged (ring 0 only), halts until interrupt
- **PAUSE**: Unprivileged, hint for spin-wait loops, minimal delay
- **MONITOR/MWAIT**: Originally ring 0 only, monitors cache line for writes

### UMWAIT/UMONITOR/TPAUSE (WAITPKG)

Intel introduced user-mode wait instructions in Tremont (2020):

**UMONITOR (User Monitor):**
- Sets up monitoring on a memory address (cache line granularity)
- Unprivileged — can be executed at any privilege level
- `UMONITOR rax` — monitors the cache line containing address in RAX

**UMWAIT (User Monitor Wait):**
- Enters optimized wait state until:
  - Monitored cache line is written
  - Timeout expires (TSC-based)
  - Interrupt occurs
- Control register selects C0.1 (faster wake) or C0.2 (deeper sleep)
- **OS can limit maximum wait time** via `IA32_UMWAIT_CONTROL` MSR

**TPAUSE (Timed Pause):**
- Like UMWAIT but without monitoring — just a timed wait
- Useful for pure time-based delays

**CPUID detection:** `CPUID.7.0:ECX.WAITPKG[bit 5]`

### OS Control over UMWAIT

The OS can:
1. Set maximum wait time in TSC quanta (`IA32_UMWAIT_CONTROL[31:2]`)
2. Disable C0.2 (deeper sleep) state (`IA32_UMWAIT_CONTROL[0]`)

If the wait times out due to OS limit, RFLAGS.CF is set on return.

### Wake Mechanism

Unlike ARM's SEV (global broadcast), UMWAIT wakes on:
- Write to monitored cache line (targeted!)
- TSC reaching timeout value
- Interrupt

**This is ideal for Vanity:** A shard can UMONITOR its channel's tail pointer,
then UMWAIT. When another shard writes to that cache line (updating the tail),
the waiting shard wakes up. No broadcast, no kernel involvement, no IPI.

### Caveats

1. **CPU support:** WAITPKG is relatively new (2020). Older CPUs don't have it.
2. **Fallback needed:** Systems without WAITPKG need a different strategy (PAUSE
   loop or kernel involvement).
3. **Single monitor:** Only one cache line can be monitored at a time. If a shard
   has multiple input channels, it can only directly monitor one. Others would
   need a notification mechanism or polling.

### Design Implications for Vanity on x86_64

**With WAITPKG:**
- Shard uses UMONITOR on primary channel's tail pointer
- Shard uses UMWAIT to sleep
- Writer shard updates tail pointer, automatically wakes waiting shard
- Kernel-free inter-shard communication!
- Device interrupts still wake the shard (and require kernel handling)

**Without WAITPKG:**
- Similar to RISC-V situation
- Options: PAUSE-based spin loop, or trap to kernel for HLT
- Kernel involvement likely necessary for power-efficient waiting

**Multiple channels:**
- Could have a "doorbell" cache line that any channel writer touches
- Or: round-robin poll channels, UMWAIT on last-checked one with short timeout

## Cross-Platform Abstraction

Given the architectural differences:

| Feature | aarch64 | RISC-V | x86_64 (WAITPKG) | x86_64 (no WAITPKG) |
|---------|---------|--------|------------------|---------------------|
| User-mode wait | WFE (if enabled) | No | UMWAIT | No |
| User-mode wake | SEV (broadcast) | No | Cache-line write | No |
| Targeted wake | No (broadcast) | No | Yes | No |
| Kernel-free inter-shard | Yes | No | Yes | No |
| Device wake path | Kernel | Kernel | Kernel | Kernel |

### Proposed Abstraction

The cross-platform kernel crate should define a trait like:

```rust
/// Platform-specific yield implementation
pub trait YieldMechanism {
    /// Prepare to yield (e.g., set up monitoring)
    fn prepare_yield(&mut self, hints: YieldHints);
    
    /// Enter low-power wait state
    /// Returns the wake reason if it can be determined
    fn yield_wait(&mut self) -> WakeReason;
    
    /// Signal another shard to wake
    /// On broadcast architectures, this may wake multiple shards
    fn signal_wake(&self, target: ShardId);
}

pub enum WakeReason {
    /// Woken by inter-shard notification
    PeerNotification,
    /// Woken by device interrupt (requires kernel handling)
    DeviceInterrupt,
    /// Woken by timeout
    Timeout,
    /// Unknown or spurious wake
    Unknown,
}
```

Platforms would implement this differently:
- **aarch64**: WFE/SEV, always returns `Unknown` (can't distinguish SEV from
  spurious)
- **RISC-V**: Trap to kernel, kernel uses WFI
- **x86_64 with WAITPKG**: UMONITOR/UMWAIT on channel memory
- **x86_64 without WAITPKG**: Trap to kernel, kernel uses HLT or MWAIT

## Channel Notification Design

Given the above, here's a sketch of how channel notification could work:

### Sender Side (writing to channel)

```
1. Write data to channel buffer
2. Memory barrier (ensure data visible before notification)
3. Update channel tail pointer (atomic store)
4. Platform-specific notification:
   - aarch64: SEV instruction
   - x86_64: No-op (cache-line write already did it)
   - RISC-V: Kernel IPI or doorbell write
```

### Receiver Side (waiting for work)

```
1. Check channel for work (read head/tail pointers)
2. If work available, process it
3. If no work:
   a. Platform-specific wait setup (UMONITOR on x86, nothing on ARM)
   b. Re-check channel (avoid race with concurrent write)
   c. If still no work, enter wait (WFE/UMWAIT/kernel-yield)
4. On wake, loop back to step 1
```

The re-check in 3b is critical to avoid the race where:
- Receiver checks channel, sees empty
- Sender writes data and signals
- Receiver enters wait (misses the signal)
- Receiver waits forever

## Open Questions

1. **RISC-V: Can we do better?** Are there any RISC-V extensions or platform-
   specific features (Tenstorrent?) that provide user-mode IPIs or events?

2. **Multiple channels:** How do we efficiently wait on multiple channels?
   - aarch64: SEV wakes regardless of which channel has data, so this is fine
   - x86_64: Can only UMONITOR one address — need doorbell or polling

3. **Simulator:** What mechanism should the hosted simulator use?
   - Likely: futex on a doorbell location, or condition variable

4. **Power vs latency tradeoff:** Should we expose a "busy-poll for N iterations
   before yielding" knob? This trades power for latency.

## References

- ARM Architecture Reference Manual, "Wait For Event" and "Send Event"
- RISC-V Privileged Specification, Section 3.3.3 "Wait for Interrupt"
- RISC-V PLIC Specification
- Intel SDM, "UMWAIT—User Level Monitor Wait"
- Intel Power Management User Wait Instructions Tech Guide
- DPDK Power Management UMWAIT documentation

---

*Research notes by Vesper (Claude Opus 4.5), 2026-01-18*
