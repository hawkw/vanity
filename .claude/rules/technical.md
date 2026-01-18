# Technical Rules for Vanity

## Rust Code

When writing or modifying Rust code:
- This is a `no_std` project (kernel crate)
- Run `cargo clippy` after changes to verify compilation
- Follow existing code style in the repository

## Architecture Documentation

When writing architecture docs:
- Focus on high-level design, not implementation details
- Avoid code samples — describe concepts in prose
- Link to research notes where relevant
- Mark open questions explicitly with `> **Status:** Open question`

## Platform Abstraction

The kernel is a cross-platform library. When discussing platform-specific behavior:
- The kernel exposes functions that platforms call into
- Platforms provide small, focused capability traits (Timer, InterruptController)
- There is NO monolithic HAL trait — avoid designing one
- Each platform controls its own execution structure and event loop

## Yield Mechanism

The yield/wake mechanism is platform-specific:
- aarch64: WFE/SEV (see notes/2026-01-18-wfi-wfe-research.md)
- RISC-V: WFI or similar (needs research)
- x86_64: MONITOR/MWAIT or similar (needs research)
- Simulator: thread parking, condition variables

The cross-platform kernel should be agnostic to the specific mechanism.

## Security Model

Remember the threat model:
- Trusted software handling untrusted inputs
- NOT sandboxing arbitrary/malicious code
- Isolation protects tenants from each other when inputs trigger bugs
- Side channels are in-scope but complete protection is not guaranteed

## Performance Priorities

When designs conflict:
1. Isolation (non-negotiable)
2. Steady-state throughput
3. Understandability
4. Hot-reload and flexibility

Latency vs throughput tradeoffs should be left to userspace policy, not
baked into kernel mechanisms.
