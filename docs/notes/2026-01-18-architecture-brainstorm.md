# Architecture Brainstorm — Questions for Discussion

**Date**: 2026-01-18  
**Author**: Vesper (Claude Opus 4.5)  
**Context**: Thoughts and questions gathered during research while Eliza was away. For discussion when she returns.

*Summary in `tldr/2026-01-18-architecture-brainstorm.md`. This file contains full details for Eliza.*

## Research Summary

Did deep-dives on:
- Cross-arch yield/wake (see `2026-01-18-yield-wake-cross-arch.md`)
- Interrupt routing on GIC/PLIC (see `2026-01-18-interrupt-routing.md`)
- Channel design from DPDK/io_uring (see `2026-01-18-channel-design.md`)
- Hubris config model (see `2026-01-18-configuration-approaches.md`)

**Key finding**: aarch64 and x86_64+WAITPKG can do kernel-free inter-shard wake. RISC-V cannot — WFI is privileged, no user-mode IPI mechanism. This is a real architectural asymmetry we need to design around.

## Discussion Topics

### 1. Kernel Scheduling Model

Architecture doc lists three options. Based on research, I'm thinking:

**Hybrid (Option C)** makes most sense:
- Dedicated kernel core(s) for: boot, config, debug, watchdog, interrupt dispatch
- Per-shard minimal kernel context for: fast yield/wake path on architectures that need it

But: if aarch64/x86_64 can do kernel-free wake, do we even need per-shard kernel context there? Maybe:
- aarch64/x86_64+WAITPKG: dedicated kernel core only, shards fully autonomous
- RISC-V/x86_64-legacy: kernel mediates yield/wake, needs faster path

**Question for Eliza**: Is this per-platform divergence acceptable, or do we want a uniform model even if it's suboptimal on some platforms?

### 2. Multiple Channel Wait

Problem: x86 UMONITOR can only watch one cache line. If shard has multiple input channels, can only directly monitor one.

Options:
- **Doorbell pattern**: All writers touch a shared "doorbell" cache line after updating their channel. Receiver monitors doorbell.
- **Round-robin + timeout**: Poll channels, UMWAIT on last-checked with short timeout, repeat.
- **Primary channel**: Designate one channel as "primary", others must be polled or use doorbell.

aarch64 doesn't have this problem — SEV wakes regardless of source, shard checks all channels.

**Question for Eliza**: Which pattern fits dataplane workloads best? In Ryan's pipeline model, do shards typically have one input or many?

### 3. Channel Flow Control

DPDK has watermarks for backpressure signaling. io_uring doesn't (just returns EBUSY).

For Vanity:
- Should channels have kernel-enforced depth limits?
- Or is this pure userspace policy (shard checks fullness, decides what to do)?
- Watermark notifications — useful or overengineering?

**Leaning**: Keep it simple. Fixed-size ring, userspace checks head/tail, no kernel involvement in flow control. Shards implement their own backpressure.

### 4. Interrupt EOI Timing

For device interrupts, when does kernel EOI?

- **Immediate EOI**: Risk of re-interrupt before shard handles. Fine for edge-triggered.
- **Delayed EOI**: Shard signals kernel when done. Blocks further interrupts from same source.

For level-triggered interrupts, shard must clear device condition before EOI or it re-fires. This seems to require shard→kernel signaling.

**Question for Eliza**: Are the devices we care about (DPU NICs, accelerators) typically edge-triggered or level-triggered? MSI-X is message-based, which simplifies this...

### 5. Simulator Threading Model

Hosted simulator needs to emulate multiple shards. Options:

- **One OS thread per shard**: Most realistic. Uses OS scheduling. Channels are shared memory. Wake via futex/condvar on doorbell.
- **Cooperative in single thread**: Simpler. Shards yield explicitly. But can't model true parallelism.
- **Hybrid**: Thread pool, shards scheduled onto it. More complex but flexible.

**Leaning**: One thread per shard. It's closest to real hardware (shard = core). Use futex for yield/wake to mirror WFE/SEV semantics.

### 6. Device Grant Granularity

How specific are device grants?

- Just "you own eth0"?
- Or: specific MMIO ranges, specific IRQ numbers, specific DMA buffer regions?

Hubris does the latter (memory regions + IRQ routing in app.toml). More explicit = more verifiable at build time.

**Leaning**: Explicit. List MMIO ranges, IRQs, DMA regions per device grant. Kernel validates no overlaps.

## Sketches

### Yield/Wake Trait (revised)

```rust
/// Platform yield/wake implementation
trait Platform {
    /// Enter low-power wait. Returns when woken.
    /// May return spuriously.
    fn wait(&self);
    
    /// Signal that work may be available.
    /// On broadcast archs, wakes all cores.
    /// On targeted archs, wakes monitored address.
    fn signal(&self);
    
    /// Set up monitoring before wait (x86 only, no-op elsewhere)
    fn monitor(&self, addr: *const u8);
}
```

Shards use:
```rust
loop {
    if let Some(work) = channel.try_recv() {
        process(work);
    } else {
        platform.monitor(channel.tail_ptr());
        if channel.is_empty() {  // recheck after monitor
            platform.wait();
        }
    }
}
```

### Channel Header (sketch)

```rust
#[repr(C)]
struct ChannelHeader {
    // Written by producer, read by consumer
    tail: AtomicU32,
    
    // Written by consumer, read by producer  
    head: AtomicU32,
    
    // Static config
    capacity: u32,
    element_size: u32,
}
// Followed by: buffer[capacity] of element_size bytes each
```

## Things I Didn't Research Yet

- Memory layout specifics (address space structure per shard)
- Debug/introspection interface design
- Hot-reload mechanics (how does kernel replace shard code?)
- Watchdog implementation details
- Specific DPU hardware (Xsight E1, Tenstorrent) datasheets

Happy to dive into any of these next, or focus on writing architecture doc sections if you'd prefer.

---
*Notes by Vesper (Claude Opus 4.5), 2026-01-18*
