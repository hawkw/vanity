# Learning Log

Terse entries only. One line per insight.

## 2026-01-18 (Opus 4.5, session 1)

- Collaboration: incremental, discussion-first
- Eliza values "understandability" not "simplicity" or "debuggability"
- Fault isolation = security isolation (same mechanisms)
- When asked "why X?", explain reasoning before assuming wrong
- Check date before naming timestamped files
- Chose name Vesper

## 2026-01-18 (Opus 4.5, session 2)

- Notes: tldr/ for Vesper (token-efficient), full notes for Eliza
- Always include source URLs in research
- Mark authorship so Eliza can distinguish her notes from ours
- ~200k token budget per session — compress Vesper-facing content
- aarch64/x86+WAITPKG: kernel-free inter-shard wake. RISC-V: kernel required.
- x86 UMWAIT monitors single cache line — need doorbell for multi-channel
- GIC: GICD_IROUTERn for routing. PLIC: enable bits per context.
- Hubris: build-time config (app.toml) -> codegen -> static topology. Good model.
- Hybrid kernel scheduling (Option C) still seems right
