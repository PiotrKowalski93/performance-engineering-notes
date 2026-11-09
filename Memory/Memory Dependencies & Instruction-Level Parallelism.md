# Memory Dependencies & Instruction-Level Parallelism – Breaking Memory Dependencies

## 1. Problem

```cpp
std::array<uint32_t, 256> computeHistogram(
    const GrayscaleImage& image) {

    std::array<uint32_t, 256> hist{};
    
    for (int i = 0; i < image.width * image.height; ++i) {
        hist[image.data[i]]++;
    }

    return hist;
}
```

Each iteration performs a read-modify-write operation on the histogram:

1. Read the pixel value
2. Use it as an index into the histogram
3. Read the current counter
4. Increment the counter
5. Write the updated counter back

The problem is that consecutive iterations may update the same histogram entry. For example:

```cpp
hist[42]++;
hist[42]++;
hist[42]++;
```

Each update depends on the result of the previous update. This creates a loop-carried memory dependency. The CPU cannot freely execute these updates in parallel because the final result of one update may be required by the next one. Additionally, the compiler cannot assume that consecutive iterations access different histogram entries because the index is data-dependent.

## 2. Baseline

The original implementation uses a single histogram.

| Image | Execution time |
|---|---:|
| bird | 77.4 us |
| coins | 35.2 us |
| pepper | 30.6 us |
| pixabay | 14.7 ms |

This implementation has a dependency between updates whenever multiple pixels map to the same histogram entry.

## 3. Optimization – Multiple Independent Histograms

To break the dependency chain, I introduced four independent histograms. Instead of updating a single histogram, each iteration stream updates a separate array:

```cpp
hist_1[image.data[i]]++;
hist_2[image.data[i + 1]]++;
hist_3[image.data[i + 2]]++;
hist_4[image.data[i + 3]]++;
```

Each histogram has its own counters, so updates to one histogram do not depend on updates to another. This allows the CPU to execute multiple independent memory operations without waiting for the previous update to complete. After processing the pixels, the partial histograms are merged into a final histogram:

```cpp
for (int i = 0; i < 256; ++i) {
    hist_final[i] =
        hist_1[i] + hist_2[i] +
        hist_3[i] + hist_4[i];
}
```

The remaining pixels that do not fit into a complete group of four are processed separately.

### Results – 4 independent histograms

| Image | Execution time | Improvement |
|---|---:|---:|
| bird | 56.2 us | 27.4% |
| coins | 27.4 us | 22.2% |
| pepper | 24.8 us | 19.0% |
| pixabay | 11.5 ms | 21.8% |

The optimization reduced execution time across all tested images.

## 4. Experiment – Increasing the Number of Histograms

I also tested six independent histograms instead of four. The idea was to expose even more independent memory operations and potentially increase instruction-level parallelism.

### Results – 6 independent histograms

| Image | Execution time |
|---|---:|
| bird | 56.0 us |
| coins | 27.3 us |
| pepper | 24.4 us |
| pixabay | 11.7 ms |

The results are almost identical to the four-histogram implementation. Increasing the number of independent histograms did not provide a meaningful performance improvement. This suggests that four independent update streams were already sufficient to hide a significant part of the dependency-related latency. More independent operations do not necessarily translate into better performance. Other bottlenecks, such as instruction throughput, cache behavior and memory access patterns, can limit further improvements.

## 5. Experiment – SIMD Reduction

The next experiment was to optimize the merging phase using AVX2 intrinsics. Instead of merging individual counters sequentially, I processed eight 32-bit counters at a time:

```cpp
__m256i h1 = _mm256_loadu_si256(
    reinterpret_cast<const __m256i*>(&hist_1[i]));

__m256i h2 = _mm256_loadu_si256(
    reinterpret_cast<const __m256i*>(&hist_2[i]));

__m256i h3 = _mm256_loadu_si256(
    reinterpret_cast<const __m256i*>(&hist_3[i]));

__m256i h4 = _mm256_loadu_si256(
    reinterpret_cast<const __m256i*>(&hist_4[i]));

__m256i sum = _mm256_add_epi32(h1, h2);
sum = _mm256_add_epi32(sum, h3);
sum = _mm256_add_epi32(sum, h4);

_mm256_storeu_si256(
    reinterpret_cast<__m256i*>(&hist_final[i]), sum);
```

Each AVX2 register contains eight 32-bit integers, allowing eight histogram entries to be merged simultaneously.

### Results – AVX2 reduction

| Image | Execution time |
|---|---:|
| bird | 56.0 us |
| coins | 27.3 us |
| pepper | 24.4 us |
| pixabay | 11.7 ms |

There was no measurable performance improvement compared to the scalar reduction. This is an interesting result because the compiler had already vectorized the scalar reduction. The generated assembly contained instructions such as:

```asm
movdqa  1024(%rsp,%rax), %xmm0
paddd   (%rsp,%rax), %xmm0
paddd   3072(%rsp,%rax), %xmm0
paddd   2048(%rsp,%rax), %xmm0
movups  %xmm0, (%rbx,%rax)
```

`paddd` performs packed integer addition on multiple 32-bit integers. The compiler was already using SIMD instructions to merge the partial histograms, so manually introducing AVX2 intrinsics did not provide an additional benefit.

**Important:** SIMD was used to accelerate the reduction phase, not the actual histogram accumulation. The data-dependent histogram updates remained scalar.

## 6. Key Takeaways

- Data-dependent memory accesses can introduce loop-carried dependencies
- Read-modify-write operations on the same memory location cannot be freely parallelized
- Splitting accumulation across multiple independent data structures can break dependency chains
- Independent operations expose more instruction-level parallelism (ILP)
- Increasing the number of independent accumulators does not guarantee better performance
- Always inspect generated assembly before introducing manual SIMD optimizations
- Compilers can already vectorize simple reduction loops, making manual intrinsics unnecessary

## Final Results

| Implementation | bird | coins | pepper | pixabay |
|---|---:|---:|---:|---:|
| Baseline | 77.4 us | 35.2 us | 30.6 us | 14.7 ms |
| 4 histograms | 56.2 us | 27.4 us | 24.8 us | 11.5 ms |
| 6 histograms | 56.0 us | 27.3 us | 24.4 us | 11.7 ms |
| 4 histograms + AVX2 reduction | 56.0 us | 27.3 us | 24.4 us | 11.7 ms |

The biggest improvement came from breaking the dependency between histogram updates, not from SIMD. The experiment demonstrates that exposing independent operations to the CPU can be more beneficial than manually optimizing arithmetic instructions.