channel-design tldr

DPDK ring: lockless SPSC/MPMC via head/tail pointer pairs. CAS for multi-producer/consumer. fixed size, power-of-two.
io_uring: SQ/CQ rings shared kernel<->userspace. SPSC per ring. memory barriers, no locks. optional eventfd for wake.
key insight: ring structure is separate from notification mechanism. ring doesn't solve wake.
for vanity: SPSC likely sufficient (one writer, one reader per channel). notification via platform yield/wake.
backpressure: DPDK has watermarks. vanity: probably just let userspace check head/tail.

full details: ../2026-01-18-channel-design.md
