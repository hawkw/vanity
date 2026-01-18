# Interrupt Controller Configuration for Per-Shard Routing

**Date**: 2026-01-18  
**Context**: Understanding how Vanity can configure interrupt controllers to route
device interrupts to specific shards without kernel mediation on every interrupt.

## Problem Statement

Vanity shards own hardware devices and need to handle device interrupts. The goal
is to route interrupts directly to the appropriate shard's core with minimal kernel
involvement. However, hardware interrupt delivery is inherently privileged — the
CPU always enters a higher privilege mode when an interrupt fires.

The questions are:
1. How do we configure interrupt controllers to route to specific cores?
2. What's the minimal kernel involvement when an interrupt fires?
3. Can we avoid kernel involvement for inter-shard notifications (IPIs)?

## ARM GIC (Generic Interrupt Controller)

ARM systems use the GIC for interrupt management. GICv3/v4 are the modern versions.

### Interrupt Types

- **SGI (Software Generated Interrupt)**: IDs 0-15, for IPIs between cores
- **PPI (Private Peripheral Interrupt)**: IDs 16-31, per-core (timers, etc.)
- **SPI (Shared Peripheral Interrupt)**: IDs 32-1019, from devices
- **LPI (Locality-specific Peripheral Interrupt)**: IDs 8192+, message-based

### GIC Architecture (v3/v4)

Components:
- **GICD (Distributor)**: Global, manages SPIs, routes to redistributors
- **GICR (Redistributor)**: Per-core, manages PPIs/SGIs for that core
- **GICC (CPU Interface)**: Per-core, delivers interrupts to the PE

### Affinity Routing

GICv3 introduced **affinity routing** for SPIs. Each SPI can be configured with
a target affinity (which PE should receive it):

```
GICD_IROUTERn = Aff3.Aff2.Aff1.Aff0
```

This specifies exactly which PE (processing element) receives interrupt N. The
affinity values correspond to the MPIDR register of the target PE.

**Alternatively**, an SPI can be set to "1 of N" mode (Interrupt_Routing_Mode=1),
where any PE in the group can take it. But for Vanity, we want explicit routing.

### SGIs for IPIs

SGIs are triggered via the `ICC_SGI1R_EL1` system register (or memory-mapped
equivalent). The sender specifies:
- Target affinity (which cores to signal)
- SGI ID (0-15)

**Privilege requirement**: SGI generation requires EL1 (or higher). Userspace
cannot directly send SGIs.

**Implication for Vanity**: Inter-shard IPIs via SGI require kernel involvement.
This is why we prefer SEV for inter-shard wakeups — it's unprivileged.

### Interrupt Handling Flow

1. Device asserts interrupt line
2. GIC Distributor receives, routes to target Redistributor
3. Redistributor presents to CPU Interface
4. CPU Interface signals PE
5. PE takes exception to EL1 (kernel)
6. Kernel acknowledges interrupt (reads ICC_IAR1_EL1)
7. Kernel determines which shard owns this interrupt
8. Kernel returns to shard (or schedules shard wake)
9. Shard handles the device
10. Kernel writes EOI (ICC_EOIR1_EL1)

**Minimum kernel path**: Steps 5-10 always involve the kernel. The kernel must
at least acknowledge and EOI the interrupt, even if the shard does the actual
device handling.

### Per-Shard Configuration Strategy

At shard setup time, Vanity should:
1. Assign device interrupt IDs to shards
2. Configure GICD_IROUTERn to route each interrupt to the shard's core
3. Set interrupt priorities appropriately
4. Enable interrupts in GICD_ISENABLERn

At runtime:
- Kernel handles interrupt entry/exit
- Shard provides the driver logic for the device
- Kernel routes wakeup to correct shard after interrupt acknowledgment

### GICv4 Direct Virtual Interrupt Injection

GICv4 added direct injection of virtual interrupts to VMs without hypervisor
involvement. However, this is for **EL2→EL1** (hypervisor to VM), not **EL1→EL0**
(kernel to userspace). It doesn't help Vanity's use case.

## RISC-V PLIC (Platform-Level Interrupt Controller)

### PLIC Architecture

The PLIC connects interrupt sources to "contexts". A context is typically a
privilege mode on a hart, e.g., "Hart 0 M-mode" or "Hart 1 S-mode".

Components:
- **Interrupt Gateways**: One per source, handles edge/level conversion
- **PLIC Core**: Priority comparison, routing logic
- **Per-context registers**: Enable bits, threshold, claim/complete

### Per-Hart Routing

Each interrupt source has:
- **Priority register**: 0-7 (typically), 0 = disabled
- **Pending bit**: Set by gateway, cleared on claim

Each context has:
- **Enable bits**: Per-source enable mask
- **Threshold register**: Minimum priority to deliver
- **Claim/Complete register**: Claim returns highest-priority pending interrupt

### Routing Interrupts to Specific Harts

To route interrupt N to hart H (S-mode context):
1. Set `priority[N]` > 0
2. Set `enable[H_smode][N]` = 1
3. Clear `enable[other_contexts][N]` = 0
4. Set `threshold[H_smode]` < `priority[N]`

This ensures only hart H's S-mode context will receive interrupt N.

### Interrupt Handling Flow

1. Device asserts interrupt
2. Gateway converts to pending request
3. PLIC compares priorities, selects highest for each context
4. If highest > threshold, PLIC signals context
5. Hart takes trap to S-mode (or M-mode)
6. Software claims interrupt (read claim register)
7. Software handles interrupt
8. Software completes interrupt (write complete register)

**Similar to GIC**: Kernel involvement is required at minimum for claim/complete.

### RISC-V ACLINT/IMSIC (Advanced Interrupt Architecture)

The RISC-V AIA introduces:
- **IMSIC (Incoming MSI Controller)**: Message-signaled interrupts per hart
- **APLIC (Advanced PLIC)**: Updated PLIC with MSI support

These are newer and provide more flexibility, but the fundamental privilege model
remains — interrupt handling requires S-mode (or M-mode).

## Vanity Interrupt Routing Design

### Configuration Phase (Kernel)

At boot/shard-setup:

```
for each shard:
    for each device grant:
        for each interrupt in device:
            configure_interrupt_route(interrupt_id, shard.core)
            enable_interrupt(interrupt_id)
            record_ownership(interrupt_id -> shard)
```

### Runtime Interrupt Path

The kernel interrupt handler needs to be **minimal**:

```
fn handle_interrupt():
    irq_id = acknowledge_interrupt()
    shard = lookup_owner(irq_id)
    
    if shard.state == Running:
        // Shard is already running on its core, just return
        // The shard will poll its device or was in WFE
        return_to_shard()
    else:
        // Shard was yielded, need to wake it
        wake_shard(shard)
        // Schedule shard to run
        
    // EOI after shard handles, or immediately if shard will poll
```

**Design choice**: When should we EOI?
- **Immediate EOI**: Allows new interrupts while shard processes. Risk of re-entry.
- **Delayed EOI**: Wait for shard to signal completion. Blocks further interrupts.

For edge-triggered: immediate EOI is usually fine.
For level-triggered: shard must clear the device condition before EOI.

### Interrupt Ownership Table

The kernel maintains a table mapping interrupt IDs to shards:

```
struct InterruptOwnership {
    interrupt_id: u32,
    owner_shard: ShardId,
    // Cached routing info for fast lookup
}
```

This should be:
- Immutable after setup (no runtime changes)
- Compact and cache-friendly (looked up on every interrupt)
- Possibly per-core for NUMA locality

### SGI/IPI Usage

Vanity could use SGIs for:
- **Watchdog kicks**: Kernel pings shard to check liveness
- **Forced yield**: Kernel preempts stuck shard (via kernel, not userspace)
- **Debug interrupts**: Debugger halts shard

Since SGI generation is privileged, these all go through the kernel, which is
appropriate for these use cases.

## Comparison of Approaches

| Aspect | ARM GIC | RISC-V PLIC |
|--------|---------|-------------|
| Per-core routing | Affinity in GICD_IROUTER | Enable bits per context |
| IPI mechanism | SGI (privileged) | CLINT msip (privileged) |
| Minimum kernel path | Acknowledge + EOI | Claim + Complete |
| User-mode handling | Not possible | Not possible |

**Conclusion**: Both architectures require kernel involvement for device interrupts.
The kernel role is:
1. Configure routing at setup time
2. Fast acknowledge/claim at interrupt time
3. Wake shard if necessary
4. Manage EOI/complete

The kernel does NOT need to:
- Parse or handle the interrupt content
- Interact with the device
- Make complex routing decisions at runtime

## Open Questions

1. **Interrupt coalescing**: Should shards be able to request batched delivery
   (e.g., poll-mode where kernel doesn't immediately wake)?

2. **Level vs edge**: How do we handle level-triggered interrupts where the shard
   must clear the condition before EOI?

3. **Shared interrupts**: What if a device's interrupt is shared (legacy PCI)?
   Probably not a concern for modern DPU hardware.

4. **MSI/MSI-X**: These are message-based, not level/edge. How does this affect
   the model? (Probably simpler — write to memory, kernel sees it, wakes shard.)

5. **Priority inversion**: If a high-priority shard is waiting on a channel from
   a low-priority shard, and a device interrupt for the low-priority shard fires,
   should kernel prioritize that interrupt?

## References

- ARM GICv3/v4 Architecture Specification (ARM IHI 0069)
- ARM Cortex-A Series Programmer's Guide for ARMv8-A
- RISC-V PLIC Specification
- RISC-V Advanced Interrupt Architecture Specification
- Linux GIC driver source (`drivers/irqchip/irq-gic-v3.c`)
- Linux PLIC driver source (`drivers/irqchip/irq-sifive-plic.c`)

---

*Research notes by Vesper (Claude Opus 4.5), 2026-01-18*
