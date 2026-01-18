# Interrupt Controller Configuration for Per-Shard Routing

**Date**: 2026-01-18  
**Author**: Vesper (Claude Opus 4.5)  
**Context**: How Vanity configures GIC (ARM) and PLIC (RISC-V) to route device interrupts to specific shards.

*Summary in `SUMMARY.md`. This file contains full details for Eliza.*

## Findings

### ARM GIC (v3/v4)

**Interrupt types**: SGI (0-15, IPIs), PPI (16-31, per-core), SPI (32+, devices), LPI (MSI-based).

**Routing SPIs**: Write target affinity to `GICD_IROUTERn`:
```
GICD_IROUTERn = Aff3.Aff2.Aff1.Aff0  // target PE's MPIDR
```

**Interrupt flow**:
1. Device → GICD → GICR → CPU interface → PE traps to EL1
2. Kernel reads `ICC_IAR1_EL1` (acknowledge)
3. Kernel looks up owner shard, wakes if needed
4. Shard handles device
5. Kernel writes `ICC_EOIR1_EL1` (EOI)

**GICv4 vLPI**: Direct injection is EL2→EL1 only (hypervisor to VM), not useful for EL1→EL0.

### RISC-V PLIC

**Routing**: Each interrupt source has priority register. Each context (hart+mode) has enable bitmap and threshold.

To route IRQ N to hart H S-mode only:
- `priority[N] > 0`
- `enable[H_smode][N] = 1`
- `enable[other][N] = 0`
- `threshold[H_smode] < priority[N]`

**Flow**: Device → gateway → PLIC core → context notification → hart traps to S-mode → software claims (`claim` register) → handles → completes (`complete` register).

### Vanity Design

**Setup phase** (kernel):
```
for shard in shards:
    for device in shard.devices:
        for irq in device.interrupts:
            route_interrupt(irq, shard.core)
            ownership_table[irq] = shard
```

**Runtime handler** (minimal kernel path):
```
irq = acknowledge()
shard = ownership_table[irq]
if shard.waiting: wake(shard)
// EOI timing depends on edge vs level
```

## Open Questions

- Level-triggered EOI: need shard→kernel signal when device cleared?
- MSI/MSI-X: simpler model (memory write), may not need same routing complexity
- Shared interrupts: probably not relevant for modern DPU hardware

## Sources

- ARM GICv3/v4 spec (IHI 0069): https://www.scs.stanford.edu/~zyedidia/docs/arm/gic_v3.pdf
- ARM GIC overview: https://documentation-service.arm.com/static/68077c770887e84c5ae973da
- GICv3/v4 OSDev wiki: https://wiki.osdev.org/Generic_Interrupt_Controller_versions_3_and_4
- RISC-V PLIC spec: https://five-embeddev.com/riscv-priv-isa-manual/Priv-v1.12/plic.html
- RISC-V PLIC PDF: https://courses.grainger.illinois.edu/ECE391/fa2025/docs/riscv-plic-1.0.0.pdf
- Linux GIC v3 devicetree bindings: https://www.kernel.org/doc/Documentation/devicetree/bindings/interrupt-controller/arm%2Cgic-v3.txt

---
*Research by Vesper (Claude Opus 4.5), 2026-01-18*
