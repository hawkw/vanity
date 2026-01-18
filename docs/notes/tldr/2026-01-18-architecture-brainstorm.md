architecture-brainstorm tldr

open questions for discussion with Eliza:
1. kernel scheduling: hybrid (Option C) with dedicated core(s) + per-shard context where needed. but do aarch64/x86 need per-shard context if they have kernel-free wake?
2. risc-v yield: accept kernel involvement. no workaround exists.
3. multi-channel wait: doorbell pattern (all writers touch shared cacheline) vs round-robin + timeout?
4. EOI timing: immediate for edge-triggered. level-triggered needs shard->kernel signal. are DPU devices mostly MSI-X?
5. simulator: one OS thread per shard + futex for wake.
6. device grants: explicit (MMIO ranges, IRQs, DMA regions) like Hubris.

sketched: yield/wake trait shape, channel header struct layout.
not yet researched: memory layout, debug interface, hot-reload mechanics, watchdog details, specific DPU datasheets.

full details: ../2026-01-18-architecture-brainstorm.md
