# Goals and Non-Goals

This section establishes what Vanity is trying to achieve and, equally
importantly, what it is explicitly *not* trying to achieve. These goals should
guide architectural decisions throughout the project.

## Goals

### Maximize steady-state throughput

The primary goal of Vanity is to maximize the amount of time spent executing
userspace workload code. Every cycle spent in the kernel, waiting for
synchronization, or performing bookkeeping is a cycle not spent processing
packets, storage commands, or inference requests.

This goal has several implications:

- The kernel should be *uninvolved* in steady-state operation wherever possible.
- Context switches, syscalls, and other transitions should be minimized.
- When kernel involvement is necessary, it should be as cheap as possible.

### Support both latency-sensitive and throughput-oriented workloads

Different applications have different performance profiles. A packet forwarding
application might prioritize low latency (respond to each packet as quickly as
possible), while a bulk data transfer might prioritize throughput (maximize
bytes per second, even if individual operations take longer).

Vanity should support both use cases by providing flexible mechanisms and
leaving policy decisions to userspace:

- **Polling or yielding**: Shards can busy-poll their input channels for lowest
  latency, or yield and wait for notifications to save cycles.
- **Batching is userspace policy**: The kernel does not batch or coalesce
  operations on behalf of userspace. Applications choose whether to process
  items one at a time or in batches.
- **No kernel-imposed queuing**: Channels provide the mechanism for inter-shard
  communication; queue depths and flow control are userspace decisions.

The kernel provides primitives that work well for both patterns; the application
decides which trade-offs to make.

### Provide strong isolation between workloads

Workloads must be isolated from each other. A bug, crash, or exploit in one
workload should not bring down the entire system, corrupt other workloads, or
leak information across boundaries. The same mechanisms that contain faults also
provide security — there is no meaningful distinction between "fault isolation"
and "security isolation" at the mechanism level.

An important distinction: Vanity runs *trusted software* that handles *untrusted
inputs*. The userspace applications are developed and deployed by a known,
trusted source — we are not trying to sandbox arbitrary or malicious code.
However, those trusted applications are exposed to untrusted data from the
network, storage devices, or other external sources. A bug in packet processing
code could be exploited by a malicious input.

The kernel provides isolation facilities that applications use to protect
themselves and each other from the consequences of such exploits. Consider a DPU
in a multi-tenant server hosting virtual machines for different customers. Each
customer's virtual NIC is handled by a separate workload. Even though all the
workload code comes from the same trusted developers, an exploit triggered by
malicious network traffic from one tenant must not be able to access another
tenant's data.

Isolation enables:

- Continued operation of unaffected workloads when one fails
- Meaningful error reporting and crash diagnostics
- Confidence that workloads cannot accidentally interfere with each other
- Multi-tenancy where untrusted inputs cannot breach tenant boundaries
- Defense in depth against exploits triggered by malicious inputs

This isolation model must also consider hardware side channels. Cache timing
attacks, speculative execution vulnerabilities, and other microarchitectural
side channels can leak information across isolation boundaries.

Our threat model — trusted software handling untrusted inputs — makes side-channel
exploitation significantly harder than scenarios where attackers run their own
code. Traditional attacks like Flush+Reload or Prime+Probe require the attacker
to execute probing code that measures microarchitectural state. When attackers
can only provide inputs, they cannot directly probe caches or branch predictors.
However, this does not eliminate side-channel risks entirely: input-dependent
timing variations in trusted code can still leak information if an attacker can
correlate their inputs with observable response times.

Where hardware provides mechanisms for mitigating these attacks (such as cache
partitioning or controlled speculation), Vanity should expose them to workloads.
Where mitigations require software cooperation (such as constant-time
algorithms), Vanity should not interfere with them. Complete protection against
all side channels is not guaranteed — the threat landscape evolves and perfect
protection may be impractical — but side channels are within our security model,
not outside it.

### Support hot-reload of workload code

It should be possible to deploy a new version of a workload without restarting
the kernel or disrupting other workloads. This enables:

- Faster development iteration
- Zero-downtime updates in deployment
- Recovery from workload crashes without full system restart

Hot-reload operates at the granularity of entire workloads (shards), not
fine-grained replacement of individual functions or pipeline stages.

### Enable direct hardware access for workloads

Workloads should be able to interact with hardware devices (NICs, accelerators,
storage controllers) directly, without kernel mediation on every operation. The
kernel grants access to hardware resources; it does not abstract them.

This "exokernel-style" approach means:

- Workloads bring their own drivers
- The kernel does not impose a particular device abstraction
- Hardware features are not hidden behind compatibility layers

### Prioritize understandability

The system should be understandable at two levels: conceptually (how does it
work?) and at runtime (what is it doing, and why?).

Conceptual understandability means:

- Clear, coherent abstractions that can be explained simply
- Predictable behavior that follows from the design
- Documentation that is part of the project, not separate from it
- A design simple enough that one person can hold it in their head

Runtime understandability means:

- Introspection via debug probes (JTAG/SWD) and host-side tools
- Visibility into shard state, channel contents, and resource usage
- Structured tracing with minimal runtime overhead (no string formatting in the
  hot path)
- Crash dumps and post-mortem analysis for debugging failures
- Profiling support for understanding where time is spent
- Access to hardware performance counters (cache misses, branch mispredicts,
  etc.) for low-level optimization
- Timing measurements for channel operations and shard execution
- Tools for identifying latency outliers and throughput bottlenecks

### Support a hosted simulator for development

It should be possible to run Vanity as a normal userspace process on a
development machine. The simulator enables:

- Rapid iteration without target hardware
- Easier debugging with host-native tools
- Automated testing in CI environments
- Development before hardware is available

The simulator is a first-class target, not a second-class afterthought.

## Non-Goals

### General-purpose computing

Vanity is not trying to be Linux, Windows, or even a minimal POSIX system. It
does not aim to:

- Run arbitrary applications
- Provide a shell or interactive environment
- Support dynamic loading of unknown code
- Offer broad hardware compatibility

### Dynamic process creation

Unlike traditional operating systems, Vanity does not support spawning new
processes at runtime. The set of workloads is defined at build time. This
restriction enables simpler resource management and stronger guarantees about
system behavior.

### Preemptive multitasking

Vanity uses cooperative scheduling. Workloads yield control voluntarily; the
kernel does not forcibly preempt them on a timer. A watchdog mechanism detects
workloads that fail to yield, but this is for fault detection, not scheduling.

This is a deliberate trade-off: we give up fairness guarantees in exchange for
lower scheduling overhead and more predictable latency.

### Invasive physical attacks

Attacks requiring intensive physical access and specialized equipment are outside
our threat model. This includes chip decapping, probing with electron microscopes,
power analysis requiring oscilloscopes attached to the board, and similar
techniques that require extended uninterrupted access with laboratory equipment.

We assume that physical security measures prevent an attacker from having a week
of uninterrupted access with a probe station. However, we may eventually consider
attacks requiring only *casual* physical access — such as a malicious USB device,
a compromised SFP transceiver, or other attacks that could be performed by
someone posing as a datacenter technician with a few minutes of access. These
casual physical access attacks are not a day-one concern but may be addressed in
future work.

### POSIX or Linux compatibility

Vanity does not provide POSIX APIs, Linux syscall compatibility, or any
expectation that existing software will run unmodified. Workloads are written
specifically for Vanity using its native interfaces.

## Future Goals

The following are goals we intend to pursue, but which are not day-one
requirements. They should inform the design (avoiding decisions that would make
them impossible) but are not required for an initial working system.

### Cryptographic verification of workload binaries

For production deployments, it may be desirable to ensure that only signed,
trusted binaries can be loaded as workloads. The initial implementation may load
unsigned binaries; signature verification can be added later as the hot-reload
and update mechanisms mature.

### Broader architecture support

The initial implementation targets aarch64 and RISC-V, with x86_64 as a
secondary target for development. Future work may expand to additional
architectures as use cases arise, including 32-bit systems, microcontrollers, or
other accelerator architectures. The design should not preclude such expansion,
but it is not a near-term priority.

### Secure boot and attestation

In high-security deployments, it may be valuable to support a secure boot chain
and remote attestation — proving to external parties that the system is running
known, trusted code. This builds on cryptographic verification but adds
additional requirements around hardware root of trust and attestation protocols.

## Tension and Trade-offs

Some goals are in tension with each other. When conflicts arise, the following
priority order applies:

1. **Isolation** — Workloads must not be able to violate each other's
   boundaries, whether through bugs or exploits. This is non-negotiable for
   multi-tenant use cases. (Isolation encompasses both "fault isolation" and
   "security isolation" — the mechanisms are the same.)

2. **Steady-state throughput** — This is the primary performance goal. Other
   goals should not compromise it significantly.

3. **Understandability** — A design that is simple and comprehensible is
   preferred over a complex design with marginally better performance.

4. **Hot-reload and flexibility** — Nice to have, but not at the cost of
   steady-state performance or significant complexity.
