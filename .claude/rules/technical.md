# Technical

Rust: no_std kernel, run cargo clippy after changes.

Platform abstraction:
- Kernel is a library that platforms call into
- Small focused capability traits (Timer, InterruptController), NOT monolithic HAL
- Each platform controls its own execution structure

Yield mechanism is platform-specific:
- aarch64: WFE/SEV
- RISC-V: kernel-mediated (WFI privileged)
- x86+WAITPKG: UMONITOR/UMWAIT
- x86 legacy: kernel-mediated
- Simulator: thread parking

Security model: trusted software handling untrusted inputs. Isolation = fault isolation = security isolation. Side channels in scope but not guaranteed.

Priority order when designs conflict: isolation > throughput > understandability > flexibility
