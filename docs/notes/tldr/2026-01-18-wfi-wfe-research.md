wfi-wfe-research tldr (aarch64)

WFE at EL0: allowed if SCTLR_EL1.nTWE=1. kernel can NOP it by clearing bit.
SEV at EL0: always allowed (unprivileged). broadcasts to all cores.
WFE wakes on: SEV from any core, interrupt pending, event register set, exception.
device interrupts: always trap to EL1. kernel involvement unavoidable.
inter-shard wake via SEV: no kernel involvement if WFE allowed at EL0.
GICv4 direct injection: only EL2->EL1 (hypervisor to VM), not useful for EL1->EL0.

design implication: aarch64 inter-shard comms can be kernel-free (SEV+WFE). device wakes need kernel. spurious wakes expected, shard must check channels and re-sleep if empty.

full details: ../2026-01-18-wfi-wfe-research.md
