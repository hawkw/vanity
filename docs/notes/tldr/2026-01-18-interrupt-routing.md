interrupt-routing tldr

fundamental: hardware interrupts always trap to privileged mode. kernel involvement unavoidable.
ARM GIC: GICD_IROUTERn sets per-SPI affinity. SGIs (IPIs) require EL1.
RISC-V PLIC: per-context enable bits + priority threshold per hart.
kernel job: configure routing at setup, fast ack/EOI at runtime, wake shard if needed.
ownership table: static IRQ->Shard mapping, O(1) lookup on interrupt entry.
EOI timing: edge-triggered=immediate. level-triggered=wait for shard to clear device.

full details: ../2026-01-18-interrupt-routing.md
