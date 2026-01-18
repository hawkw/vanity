configuration-approaches tldr

Hubris model: app.toml at build time -> codegen -> static task topology. no runtime creation/destruction.
benefits: peak memory known at compile time. predictable addresses (helps debugger). no dynamic alloc in kernel.
build process: compile tasks -> measure sizes -> allocate non-overlapping regions -> re-link -> generate kernel tables -> package image.
for vanity: similar approach fits. shards defined in config, kernel tables generated at build. hot-reload changes code, not topology.

full details: ../2026-01-18-configuration-approaches.md
