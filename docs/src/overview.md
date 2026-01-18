# Overview

Vanity is a dataplane operating system designed for high-throughput, low-latency
workloads on many-core systems with direct-attached hardware accelerators. Target
applications include network packet processing, high-performance storage, and AI
inference acceleration. For a detailed discussion of what Vanity is trying to
achieve, see [Goals and Non-Goals](./design/goals.md).

## What Vanity Is

Vanity is a small, focused operating system kernel that manages resources for a
fixed set of concurrent workloads. Its primary job is to *set up* resource
allocations — cores, memory regions, device access — and then get out of the way
during steady-state operation.

The design draws inspiration from several sources:

- **Exokernels** (MIT's ExOS/XOK): The idea that the OS should securely multiplex
  hardware rather than abstracting it away. Applications should have direct
  access to hardware where possible.

- **Akaros**: The concept of "many-core processes" that own dedicated cores and
  are treated as first-class entities, with asymmetric OS structure where some
  cores handle management while others run application code.

- **Linux io_uring**: The use of shared memory rings for communication between
  userspace and kernel, avoiding syscall overhead in the common case.

- **Hubris** (Oxide Computer): A static, predictable system model where the set
  of tasks is known at build time, enabling strong reasoning about resource
  usage and fault isolation.

## What Vanity Is Not

Vanity is **not a general-purpose operating system**. It does not aim to run
arbitrary software, support dynamic process creation, or provide POSIX
compatibility. It is not suitable for desktop computing, general server
workloads, or environments where flexibility is more important than throughput.

Vanity is also **not a unikernel**. There is a protection boundary between the
kernel and userspace workloads. This boundary exists to provide:

- **Isolation**: A bug, crash, or exploit in one workload should not affect
  others. The same mechanisms that contain faults also provide security — see
  the [Goals](./design/goals.md) for discussion of the threat model.
- **Hot-reload**: Workload code can be updated without restarting the kernel.
- **Introspection**: The kernel can observe and debug workload state.
- **Memory protection**: Workloads cannot accidentally corrupt each other's
  private state.

## The Shard Model

The central abstraction in Vanity is the **shard**: an isolated execution domain
with dedicated resources. A shard owns:

- One or more CPU cores (exclusive during operation)
- A private memory region
- Zero or more device resources (MMIO regions, interrupt lines, DMA buffers)
- Zero or more channels for communication with other shards

Shards communicate via **channels**: shared memory regions with a notification
mechanism. The kernel sets up channel mappings but does not interpret channel
contents — the protocol is determined by userspace.

This model maps naturally to dataplane workloads:

- In packet processing, each pipeline stage can be a shard, with channels
  carrying packets between stages.
- In storage, each device queue handler can be a shard, with channels connecting
  to backend processing.
- In AI inference, each model stage can be a shard, with channels passing tensor
  data through the pipeline.

## Design Philosophy

Several principles guide Vanity's design. These are explored in more detail in
the [Goals and Non-Goals](./design/goals.md) section.

**Performance over flexibility.** The system is optimized for a specific class
of workloads (pipeline-structured dataplane processing) rather than trying to be
good at everything. Design decisions that sacrifice throughput for generality
are avoided.

**Static over dynamic.** The set of shards and their resource allocations are
defined at build time and fixed at boot. This enables strong reasoning about
resource usage and avoids dynamic allocation overhead. Hot-reload provides
runtime flexibility where needed without sacrificing predictability.

**Cooperative over preemptive.** Shards yield control voluntarily when their
work is complete or when they're waiting for input. The kernel does not
preemptively timeslice between shards. A watchdog mechanism detects shards that
fail to yield in a timely manner, but this is for fault detection, not
scheduling.

**Direct access over abstraction.** Shards have direct access to hardware
resources (MMIO, DMA) rather than going through kernel abstractions. The kernel
provides the *mechanism* for granting access, not the *policy* for using it.
Userspace brings its own drivers.

**Understandability from day one.** The system should be understandable
conceptually (clear abstractions) and at runtime (visibility into system state,
debugging crashes and performance issues). Observability tools, structured
tracing, profiling, and crash dumps are designed in from the start, not added
later.

## Target Platforms

Vanity targets many-core systems with direct-attached hardware accelerators,
such as:

- Data processing units (DPUs) and smart NICs
- AI/ML accelerator cards
- High-performance storage controllers
- Network appliances and middleboxes

The primary target architectures are **aarch64** and **RISC-V (rv64)**, with
**x86_64** as a secondary target for development and certain deployment
scenarios.

Additionally, Vanity provides a **hosted simulator** that runs the kernel as a
normal userspace process on a development machine. The simulator enables rapid
iteration, debugging, and testing without requiring target hardware.

The kernel is structured as a cross-platform Rust library (`no_std`). Platform-
specific code calls into this library, providing the necessary hardware
interfaces. This inversion — platforms call kernel, rather than kernel calling
a monolithic HAL — gives platforms control over their execution structure while
sharing core logic.
