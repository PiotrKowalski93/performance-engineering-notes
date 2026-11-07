# False Sharing

## Overview

False sharing occurs when multiple threads modify independent variables that reside on the same CPU cache line. Although threads access different memory locations, the cache coherence protocol operates at the cache-line granularity. When one thread modifies a value, the cache line may need to be invalidated in other cores, causing unnecessary cache-coherence traffic. This can significantly degrade performance in highly concurrent workloads.

## Example

### Baseline — Without Cache-Line Alignment

Each thread updates its own accumulator. However, the accumulators may be located next to each other in memory.

```cpp
struct Accumulator {
    std::atomic<uint32_t> value = 0;
};

std::vector<Accumulator> accumulators(thread_count);
```

Even though each thread accesses a separate atomic variable, multiple accumulators may share the same cache line. This cause false sharing.

### Optimized — Cache-Line Alignment

```cpp
struct alignas(64) Accumulator {
    std::atomic<uint32_t> value = 0;
};

std::vector<Accumulator> accumulators(thread_count);
```

`alignas(64)` aligns each `Accumulator` to a 64-byte boundary.

On a CPU with 64-byte cache lines, this ensures that each accumulator occupies its own cache line, preventing multiple threads from modifying different accumulators within the same cache line. The alignment also introduces padding, increasing the memory footprint in exchange for reduced cache-coherence traffic.

> Note: 64 bytes is a common cache-line size on modern x86 CPUs, but the actual size is architecture-dependent.

## Benchmark Results

| Configuration | Real Time | CPU Time |
|---|---:|---:|
| Without `alignas(64)` | 187 ms | 174 ms |
| With `alignas(64)` | 31.2 ms | 30.5 ms |

### Performance Improvement

- Real Time: approximately **6.0× faster**
- CPU Time: approximately **5.7× faster**
- Real Time reduction: approximately **83.3%**

The benchmark demonstrates a substantial performance improvement after eliminating false sharing.

## Why Does It Matter?

False sharing is particularly relevant when:

- Multiple threads frequently update independent counters
- Workloads use shared arrays of small structures
- Threads execute on different CPU cores
- The workload is sensitive to cache-coherence traffic

Common solutions include:

- `alignas(64)` for cache-line alignment
- Padding structures to prevent adjacent variables from sharing a cache line
- Using thread-local storage and aggregating results after parallel execution
- Reducing the frequency of shared writes