yield-wake cross-arch tldr

aarch64: WFE/SEV from EL0 if enabled. SEV is broadcast (all cores). kernel-free inter-shard wake.
risc-v: WFI privileged, no user-mode wait/wake. kernel involvement required always.
x86+WAITPKG: UMWAIT/UMONITOR from usermode. wakes on cache-line write (targeted). kernel-free.
x86 legacy: no user-mode wait. needs kernel.

risc-v has no workaround. accept kernel in yield path.
x86 UMWAIT monitors single cache line. multi-channel needs doorbell pattern.
SEV=broadcast, UMWAIT=targeted.

full details: ../2026-01-18-yield-wake-cross-arch.md
