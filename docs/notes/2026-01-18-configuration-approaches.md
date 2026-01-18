# System Configuration Approaches

**Date**: 2026-01-18  
**Author**: Vesper (Claude Opus 4.5)  
**Context**: Researching how to specify shard topology, resource allocation, and channel configuration for Vanity.

*Summary in `tldr/2026-01-18-configuration-approaches.md`. This file contains full details for Eliza.*

## Hubris Configuration Model

From `app.toml`:
- Task list with priorities, memory regions, stack sizes
- Interrupt routing (IRQ → task notification bit)
- Peripheral assignments (which task owns which MMIO)
- IPC permissions (which tasks can talk to which)

Build system:
1. Compiles each task separately
2. Measures sizes
3. Allocates non-overlapping memory regions
4. Re-links tasks to final addresses
5. Generates kernel config tables
6. Packages single firmware image

**No runtime task creation.** Set of tasks is fixed. Hot-reload = reinit existing task with new code.

## Vanity Configuration Sketch

Similar structure could work:

```toml
[shard.packet_rx]
cores = [0, 1]
memory = "16M"
devices = ["eth0_rx"]
channels = { output = "rx_to_parser" }

[shard.parser]
cores = [2, 3]
memory = "8M"
channels = { input = "rx_to_parser", output = "parsed_to_tx" }

[channel.rx_to_parser]
size = "1M"
# implicitly connects packet_rx → parser
```

Build system would:
1. Compile each shard's code
2. Generate kernel tables: shard descriptors, channel mappings, device grants
3. Verify no resource conflicts (core overlap, memory overlap)
4. Package kernel + shards

## Open Questions

- Do we want runtime-adjustable channel sizes? (Probably not — complicates memory management)
- How granular is device specification? (Individual IRQs? MMIO ranges? Named devices?)
- Supervisor shard: special or just another shard with kernel IPC access?

## Sources

- Hubris Reference: https://hubris.oxide.computer/reference/
- "From Hubris to Bits" (Cliffle): https://cliffle.com/blog/from-hubris-to-bits/
- Hubris app.toml examples in oxidecomputer/hubris repo

---
*Research by Vesper (Claude Opus 4.5), 2026-01-18*
