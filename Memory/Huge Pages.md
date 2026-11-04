# Huge Pages: Reducing DTLB Misses in Memory-Intensive Workloads

## Overview

This experiment explores how Transparent Huge Pages (THP) can improve the performance of memory-intensive workloads by reducing the number of Data Translation Lookaside Buffer (DTLB) misses. The benchmark applies a matrix-free operator to a large array of doubles.
The baseline uses a conventional dynamic memory allocation. The optimized version uses `mmap()` and `madvise(MADV_HUGEPAGE)` to request Transparent Huge Pages.

The main objective is to investigate the relationship between:

- Memory page size
- Virtual-to-physical address translation
- DTLB misses and page-table walks

---

## 1. Background

### What is a memory page?

Linux uses virtual memory. A process accesses memory through virtual addresses, which are translated into physical addresses.
Memory is managed in fixed-size units called pages. On a typical x86-64 Linux system:

| Page type | Size |
|---|---:|
| Standard page | 4 KiB |
| Huge Page | 2 MiB |
| Huge Page (optional) | 1 GiB |

The actual supported page sizes depend on the architecture and system configuration.

### What is the TLB?

The Translation Lookaside Buffer (TLB) is a hardware cache that stores recently used virtual-to-physical address translations.
Without a TLB hit, the processor may need to perform a page-table walk to resolve the address. Page-table walks can be expensive, particularly in workloads that access large memory regions with poor TLB locality.

### Why can Huge Pages help?

Consider a 1 GiB memory region.

Using 4 KiB pages:

```text
1 GiB / 4 KiB = 262,144 pages
```

Using 2 MiB pages:

```text
1 GiB / 2 MiB = 512 pages
```

A 2 MiB page covers 512 times more memory than a 4 KiB page. This can significantly increase TLB reach and reduce the number of page-table entries required to map a large memory region. Huge Pages do not make the data itself faster to access. They primarily reduce address-translation overhead.

---

## 2. Huge Pages vs Transparent Huge Pages

Linux supports different mechanisms for using Huge Pages.

### Explicit Huge Pages ( hugetlb )

Explicit Huge Pages require a reserved pool of huge pages.
They can be requested through mechanisms such as:

```cpp
MAP_HUGETLB
```

### Transparent Huge Pages (THP)

Transparent Huge Pages allow Linux to manage huge-page allocation transparently. An application can request that a memory region be considered for THP:

```cpp
madvise(ptr, size, MADV_HUGEPAGE);
```

The kernel may use huge pages when conditions permit. This is the approach used in this experiment.
Important: `MADV_HUGEPAGE` is a hint. It does not guarantee that the entire region will be backed by huge pages.

---

## 3. Enabling Transparent Huge Pages on Linux

### Check the current configuration

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Example output:

```text
always [madvise] never
```

The value inside brackets is the currently selected policy.

Common policies:

| Policy | Description |
|---|---|
| `always` | THP is considered for eligible memory mappings by default |
| `madvise` | THP is considered for mappings explicitly marked with `MADV_HUGEPAGE` |
| `never` | THP is disabled |

For this experiment, the `madvise` policy is a useful choice because the application explicitly requests THP.

### Enable the madvise policy

```bash
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

This changes the active policy until the setting is changed again or the system restarts.

### Verify the setting

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

### Check whether THP is actually being used

The system policy alone does not prove that a particular allocation uses huge pages.

Inspect the process memory mappings:

```bash
cat /proc/<PID>/smaps
```

Look for fields such as:

```text
AnonHugePages
KernelPageSize
MMUPageSize
```

For example:

```text
AnonHugePages:      2048 kB
```

This indicates that 2 MiB of anonymous memory in that mapping is backed by anonymous huge pages.
The exact output depends on the kernel and mapping.

---

## 4. Baseline Implementation

The baseline allocates a large array of doubles using `new[]`.

```cpp
inline auto allocateDoublesArray(size_t size) {
    double* alloc = new double[size];

    auto deleter = [](double* ptr) {
        delete[] ptr;
    };

    return std::unique_ptr<double[], decltype(deleter)>(
        alloc,
        std::move(deleter)
    );
}
```

The allocation uses the conventional C++ dynamic memory allocation mechanism. The underlying physical memory may still benefit from Linux memory-management features, including THP, depending on the system configuration. The baseline does not explicitly request THP for the allocated region.

---

## 5. Optimized Implementation

The optimized implementation uses `mmap()` to allocate a memory region and `madvise()` to request Transparent Huge Pages.

```cpp
inline auto allocateDoublesArray(size_t size) {
    size_t total_array_size = size * sizeof(double);

    constexpr size_t page_2mb = 1UL << 21;

    auto pages_needed =
        total_array_size / page_2mb +
        (total_array_size % page_2mb != 0);

    size_t total_to_alloc = pages_needed * page_2mb;

    void* ptr = mmap(
        nullptr,
        total_to_alloc,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS,
        -1,
        0
    );

    if (ptr == MAP_FAILED) {
        throw std::bad_alloc{};
    }

    madvise(ptr, total_to_alloc, MADV_HUGEPAGE);

    auto deleter = [total_to_alloc](double* p) {
        munmap(p, total_to_alloc);
    };

    return std::unique_ptr<double[], decltype(deleter)>(
        static_cast<double*>(ptr),
        std::move(deleter)
    );
}
```

### Key implementation details

#### 1. Round allocation size up to a multiple of 2 MiB

```cpp
constexpr size_t page_2mb = 1UL << 21;
```

2 MiB equals \(2^{21}\) bytes.

The allocation size is rounded up to a multiple of 2 MiB. This is useful when working with huge-page-sized regions, although `madvise(MADV_HUGEPAGE)` does not require the mapping size to be an exact multiple of 2 MiB.

#### 2. mmap()

```cpp
mmap(
    nullptr,
    total_to_alloc,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
);
```

`mmap()` creates a virtual memory mapping.

The mapping is:

- Private
- Anonymous
- Readable and writable

The returned address is page-aligned.

#### 3. madvise()

```cpp
madvise(ptr, total_to_alloc, MADV_HUGEPAGE);
```

This tells Linux that the region is a candidate for Transparent Huge Pages. The kernel may use 2 MiB pages instead of standard 4 KiB pages. The return value should be checked in production code because `madvise()` can fail.

#### 4. Custom deleter

```cpp
auto deleter = [total_to_alloc](double* p) {
    munmap(p, total_to_alloc);
};
```

Unlike `new[]`, `mmap()` memory must be released with `munmap()`.

The custom deleter captures the allocation size so that the mapping can be released correctly. This is an example of using RAII to manage a resource other than ordinary heap memory.

---

## 6. Benchmark Results

### Baseline

Google Benchmark:

| Benchmark | Time | CPU | Iterations |
|---|---:|---:|---:|
| Apply matrix-free operator | 6.41 s | 6.41 s | 1 |

Performance counters:

```text
415,902,857  dtlb_load_misses.walk_completed
 53,665,767  dtlb_store_misses.walk_completed
```

Perf statistics:

```text
8.855646327 seconds time elapsed
8.524874000 seconds user
0.329995000 seconds sys
```

### Optimized — Transparent Huge Pages

Google Benchmark:

| Benchmark | Time | CPU | Iterations |
|---|---:|---:|---:|
| Apply matrix-free operator | 5.80 s | 5.80 s | 1 |

Performance counters:

```text
50,202,925  dtlb_load_misses.walk_completed
 6,465,025  dtlb_store_misses.walk_completed
```

Perf statistics:

```text
8.115809081 seconds time elapsed
7.898769000 seconds user
0.215993000 seconds sys
```

### Results comparison

| Metric | Baseline | THP | Change |
|---|---:|---:|---:|
| Google Benchmark time | 6.41 s | 5.80 s | -9.5% |
| Throughput | 228.476 MiB/s | 252.272 MiB/s | +10.4% |
| DTLB load misses | 415.9 M | 50.2 M | -87.9% |
| DTLB store misses | 53.7 M | 6.5 M | -88.0% |
| `perf` elapsed time | 8.856 s | 8.116 s | -8.4% |

The Google Benchmark runtime improved by approximately 9.5%. The number of completed DTLB page-table walks decreased by approximately 88%.

---

## 7. Why Did the Performance Improve?

### Reduced DTLB page-table walks

The most significant change is the reduction in DTLB misses. The workload accesses a large memory region. With standard 4 KiB pages, the processor needs to manage a much larger number of address translations. Using 2 MiB pages increases TLB reach and reduces the frequency of page-table walks.

This is consistent with the observed reduction in:

```text
dtlb_load_misses.walk_completed
dtlb_store_misses.walk_completed
```

### Reduced address-translation overhead

Page-table walks require additional memory accesses. Even when the actual data is available in cache, the processor may still need to resolve the virtual address.
Reducing the number of page-table walks can therefore improve performance independently of data-cache behavior.

### Why was the speedup only around 9.5%?

A reduction in DTLB misses does not translate directly into an equivalent reduction in execution time.

The workload may still be limited by:

- Memory bandwidth
- Data-cache misses
- Arithmetic operations
- Instruction throughput
- Other memory-system bottlenecks

The CPU can also overlap some page-table walks with other work. Huge Pages optimize address translation, not every aspect of memory access.

---

## 8. Important Considerations

### Huge Pages are not always beneficial

Huge Pages can introduce trade-offs:

- Increased memory fragmentation
- Potentially higher memory consumption
- More expensive allocation or compaction
- Possible latency spikes during page promotion or allocation

For latency-sensitive systems, these effects should be measured.

### THP behavior is kernel-dependent

The Linux kernel may use, split, or collapse huge pages depending on memory pressure and configuration. The presence of `MADV_HUGEPAGE` does not guarantee that all pages will be huge pages.

---

## 9. Key Takeaways

- Huge Pages reduce the number of virtual-to-physical address translations required for large memory regions
- Transparent Huge Pages allow Linux to manage huge-page allocation automatically
- `mmap()` and `madvise(MADV_HUGEPAGE)` provide a way to request THP for a memory region
- The observed reduction in DTLB misses was approximately 88%
- The measured runtime improved by approximately 9.5%
- A large reduction in hardware counters does not necessarily translate into an equally large speedup
- Huge Pages are most useful when address-translation overhead is a meaningful bottleneck
- Always verify actual huge-page usage and benchmark the workload