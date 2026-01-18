# Cross-Architecture Yield/Wake Mechanisms

**Date**: 2026-01-18  
**Author**: Vesper (Claude Opus 4.5)  
**Context**: Extending WFI/WFE research to RISC-V and x86_64 for cross-platform shard yield/wake.

*Summary in `SUMMARY.md`. This file contains full details for Eliza.*

## Findings

### RISC-V

WFI traps or faults in U-mode per privileged spec §3.3.3. `mstatus.TW` controls timeout behavior but doesn't enable true user-mode wait. No SEV equivalent exists — IPIs require CLINT `msip` writes (M-mode only).

**Options for Vanity**:
1. Trap to kernel on yield, kernel WFIs, return on any wake
2. Busy-poll with fences (wastes power)
3. Hybrid: short poll then kernel yield

AIA/IMSIC may improve this but still requires privilege.

### x86_64 WAITPKG

CPUID.7.0:ECX[5] indicates support. Instructions:
- `UMONITOR rax` — set up cache-line monitor
- `UMWAIT` — wait until write to monitored line, timeout, or interrupt
- `TPAUSE` — timed wait without monitoring

OS can limit max wait time via `IA32_UMWAIT_CONTROL` MSR. If OS timeout hit, RFLAGS.CF set.

Key advantage: wake is *targeted* to cache-line write, not broadcast. Writer updates channel tail → reader wakes automatically.

### Cross-Platform Abstraction

| Arch | User wait | User wake | Targeted | Kernel-free inter-shard |
|------|-----------|-----------|----------|-------------------------|
| aarch64 | WFE | SEV | No (broadcast) | Yes |
| RISC-V | No | No | N/A | No |
| x86+WAITPKG | UMWAIT | cache write | Yes | Yes |
| x86 legacy | No | No | N/A | No |

Proposed trait shape:
```rust
trait YieldMechanism {
    fn prepare(&mut self, hints: YieldHints);
    fn wait(&mut self) -> WakeReason;
    fn signal(&self, target: ShardId);
}
```

### Channel Wake Protocol

Sender: write data → barrier → update tail → platform signal (SEV on ARM, no-op on x86)  
Receiver: check channel → if empty: setup monitor → recheck → wait → loop

The recheck-after-setup is critical to avoid lost-wake races.

## Sources

- RISC-V Privileged Spec §3.3.3: https://riscv.github.io/riscv-isa-manual/snapshot/privileged/
- RISC-V WFI U-mode discussion: https://github.com/riscv/riscv-isa-manual/issues/1224
- Intel UMWAIT reference: https://www.felixcloutier.com/x86/umwait
- DPDK UMWAIT power management guide (Intel): https://d2pgu9s4sfmw1s.cloudfront.net/UAM/Prod/Done/a062E00001fDjZZQA0/...
- Stack Overflow UMONITOR/UMWAIT example: https://stackoverflow.com/questions/74956482/

---
*Research by Vesper (Claude Opus 4.5), 2026-01-18*
