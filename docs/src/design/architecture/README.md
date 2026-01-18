# System Architecture

This section describes Vanity's system architecture at a high level. It covers
the core abstractions, how the kernel and userspace interact, and how the
system is structured across different hardware platforms.

> **Note:** This document captures design decisions made so far. Some sections
> are marked as open questions or areas requiring further design work.

## Overview

Vanity's architecture is organized around a few key principles derived from the
[goals](../goals.md):

1. **The kernel sets up resources, then gets out of the way.** During
   steady-state operation, userspace shards should run with minimal kernel
   involvement.

2. **Isolation is provided by hardware protection mechanisms.** Shards run in
   separate address spaces with distinct privilege levels from the kernel.

3. **Communication between shards uses shared memory channels.** The kernel
   establishes channel mappings but does not interpret or mediate channel
   contents at runtime.

4. **Platforms call into the kernel, not the other way around.** The
   cross-platform kernel is a library (`no_std` Rust crate) that platform-
   specific code invokes. There is no monolithic "HAL trait" that abstracts
   all hardware — instead, platforms provide small, focused capability objects
   where needed.

## Core Abstractions

### Shards

A **shard** is the fundamental unit of isolation and execution in Vanity. Each
shard is:

- An isolated address space (separate page tables / memory protection)
- Assigned one or more CPU cores for exclusive use during operation
- Granted access to specific device resources (MMIO regions, DMA buffers,
  interrupt lines)
- Connected to other shards via channels

Shards are defined at build time as part of the system configuration. The set
of shards is fixed — there is no dynamic process creation at runtime. However,
individual shards can be reloaded (torn down and restarted with new code)
without restarting the kernel or affecting other shards.

### Channels

A **channel** is a shared memory region used for communication between shards.
The kernel:

- Allocates the memory region
- Maps it into both shards' address spaces
- Provides a notification mechanism so shards can signal each other

The kernel does **not** interpret channel contents. The data format, flow
control, and protocol are determined by userspace. This allows channels to be
used for diverse purposes: packet descriptor rings, command queues, tensor
buffers, or any other shared-memory communication pattern.

### Device Grants

A **device grant** gives a shard direct access to hardware resources:

- MMIO regions mapped into the shard's address space
- DMA buffer memory that the device can read/write
- Interrupt lines routed to the shard

With a device grant, userspace code can interact with hardware directly —
reading/writing device registers, setting up DMA transfers, handling
completions — without kernel mediation on each operation. The shard brings its
own driver code.

## Kernel-Userspace Interaction Model

A key architectural question is: how do shards interact with the kernel, and
where does kernel code execute?

### Syscall Interface

For **infrequent operations** (configuration, setup, teardown, fault handling),
shards use traditional trap-based syscalls:

- Requesting device grants
- Creating/destroying channels
- Reporting fatal errors
- Hot-reload requests

These operations are rare and can afford the overhead of a full trap and
context switch.

### Yield and Wake Mechanism

For **yielding when waiting for work**, shards use a platform-specific
low-power wait mechanism. On aarch64, this is WFE (Wait For Event):

- Shard executes WFE when it has no work (input channels empty, devices idle)
- Shard can be woken by:
  - **Another shard** via SEV (Send Event) — no kernel involvement
  - **A device interrupt** — requires kernel to handle the interrupt, then
    resume the shard

This design means that pure inter-shard communication (e.g., pipeline stages
passing work to each other) can operate with **zero kernel involvement** in
steady state. Kernel involvement only occurs when:

- A device interrupt fires (unavoidable — hardware requires privileged handling)
- A shard needs to make a syscall (rare)
- A fault occurs (exceptional)

The cross-platform kernel crate should be **agnostic to the specific yield
mechanism**. Each platform implements an appropriate strategy:

- aarch64: WFE + SEV
- RISC-V: WFI or platform-specific mechanism (needs research)
- x86_64: MONITOR/MWAIT, PAUSE, or similar (needs research)
- Simulator: thread parking, condition variables, or similar

See [WFI/WFE research notes](../../../notes/2026-01-18-wfi-wfe-research.md) for
detailed analysis of the aarch64 mechanism.

### Kernel Scheduling Model

> **Status:** This is an open design question requiring further discussion.

We have identified several possible models for how the kernel is scheduled
relative to userspace shards:

**Option A: Dedicated kernel core(s)**
- One or more cores reserved exclusively for kernel operations
- Shards communicate with kernel via shared memory mailboxes
- Clean separation, but "wastes" cores

**Option B: Per-shard kernel context**
- Each shard's core also runs kernel code when needed
- Mode switch on same core for kernel operations
- No wasted cores, but more complex state management

**Option C: Hybrid**
- Dedicated core(s) for system-wide operations (boot, config, debug)
- Minimal per-shard kernel context for fast-path operations (yield, wake)

The choice depends on what kernel work actually needs to happen at runtime. If
the yield/wake mechanism avoids kernel involvement for inter-shard
communication (as described above), the kernel's steady-state responsibilities
are minimal:

- Handle device interrupts and route them to the appropriate shard
- Monitor watchdogs for stuck shards
- Service debug/introspection requests
- Handle faults and crashes

This suggests that dedicated kernel core(s) might be acceptable — they won't be
heavily loaded in steady state.

## Platform Integration

The kernel is structured as a cross-platform Rust library. Platform-specific
code (for aarch64, RISC-V, x86_64, or the simulator) calls into this library
rather than the library calling out through a HAL trait.

This means:

- Platforms control their own startup sequence and event loop structure
- Platforms provide small, focused traits for specific capabilities (timers,
  interrupt controllers) rather than one large HAL abstraction
- The kernel library exposes functions like `kernel::init()`,
  `kernel::handle_interrupt()`, `kernel::shard_yield()` that platforms invoke
  at appropriate times

This inversion gives platforms flexibility. An aarch64 bare-metal target and
the hosted simulator have very different execution structures, but both can use
the same kernel logic.

## Open Questions

The following architectural questions require further design work:

1. **Kernel scheduling model**: Which option (dedicated core, per-shard context,
   or hybrid) best fits our goals?

2. **Interrupt routing**: How do we configure interrupt controllers (GIC on ARM,
   PLIC on RISC-V) to route device interrupts to the appropriate shard's core?

3. **Channel notification mechanism**: What exactly does the notification look
   like? A doorbell write? An IPI? How does it integrate with the yield/wake
   mechanism?

4. **Memory layout**: How are shard address spaces laid out? How are shared
   regions (channels, device MMIO) mapped?

5. **System configuration format**: How is the set of shards, their resource
   allocations, and the channel topology specified? Build-time code generation
   (like Hubris)? A manifest file parsed at boot?

These will be addressed in subsequent sections as the design evolves.
