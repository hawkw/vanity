# Channel Design Patterns

**Date**: 2026-01-18  
**Author**: Vesper (Claude Opus 4.5)  
**Context**: Researching ring buffer and notification patterns from DPDK and io_uring for Vanity's channel design.

*Summary in `tldr/2026-01-18-channel-design.md`. This file contains full details for Eliza.*

## Design Elements

### Ring Structure (DPDK-style)

```
struct Ring<T, const N: usize> {
    buffer: [MaybeUninit<T>; N],  // power of two
    prod_head: AtomicU32,
    prod_tail: AtomicU32,
    cons_head: AtomicU32,
    cons_tail: AtomicU32,
}
```

**SPSC simplification**: With single producer/consumer, can drop to two pointers (head, tail) with appropriate barriers.

**Enqueue**: 
1. Read prod_head, check space vs cons_tail
2. Write data at prod_head
3. Barrier
4. Update prod_tail (makes visible to consumer)

**Dequeue**:
1. Read cons_head, check data vs prod_tail  
2. Read data at cons_head
3. Barrier
4. Update cons_tail (frees slot for producer)

### io_uring Lessons

- **Separate indices**: SQ has submission tail (user writes), head (kernel reads). CQ has completion tail (kernel writes), head (user reads).
- **Memory ordering**: Barriers between data access and index update. No locks needed for SPSC.
- **Batching**: User can submit multiple SQEs before notifying kernel. Kernel can complete multiple before notifying user.
- **Notification decoupling**: eventfd registration optional. Allows polling-only mode for lowest latency.

### Notification Strategy

| Mode | Mechanism | Latency | CPU |
|------|-----------|---------|-----|
| Busy poll | None | Lowest | High |
| Event-triggered | SEV/UMWAIT/eventfd | Low | Low |
| Hybrid | Poll N times, then wait | Tunable | Medium |

**For Vanity**: Channel writer updates tail, then platform-signal. Reader checks tail, if empty does platform-wait. Recheck-after-wait to avoid races.

### Backpressure

DPDK watermark: callback when enqueue crosses threshold. Useful for:
- Flow control signaling
- Adaptive batching
- Avoiding full-queue drops

Vanity could expose watermark as channel metadata for shard to poll.

## Open Questions

- Fixed message size vs variable? (Fixed simpler, variable more flexible)
- Inline small messages vs pointer-passing? (Inline avoids indirection for small data)
- Per-channel notification or shared doorbell? (Tradeoff: precision vs complexity)

## Sources

- DPDK Ring Library docs: https://doc.dpdk.org/guides/prog_guide/ring_lib.html
- DPDK ring deep dive: https://medium.com/@shivajiofficial5088/unlocking-performance-with-lockless-queues-a-deep-dive-into-dpdk-ring-library-daa3077be6bc
- io_uring intro (Oracle): https://blogs.oracle.com/linux/an-introduction-to-the-io-uring-asynchronous-io-framework
- io_uring eventfd: https://man7.org/linux/man-pages/man3/io_uring_register_eventfd.3.html
- io_uring what's new: https://kernel.dk/io_uring-whatsnew.pdf
- Linux lockless ring buffer: referenced in DPDK docs

---
*Research by Vesper (Claude Opus 4.5), 2026-01-18*
