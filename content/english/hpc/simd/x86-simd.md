---
title: Intrinsics and Vector Extensions
weight: 1
---

The most direct and low-level way to use SIMD is to use the vector instructions in assembly — they aren't different from their scalar equivalents at all — but we are not going to do that. Instead, we will use *intrinsic* functions mapping to these instructions that are available in all modern C/C++ compilers.

## Setup

To use them, we need to do a little groundwork.

First, we need to determine which extensions are supported by the hardware. On Linux, you can call `cat /proc/cpuinfo`, and on other platforms you'd better go to [WikiChip](https://en.wikichip.org/wiki/WikiChip) and look it up there using the name of the CPU. In either case, there should be a `flags` section that lists the codes of all supported vector extensions.

There is also a special [CPUID](https://en.wikipedia.org/wiki/CPUID) assembly instruction that lets you query various information about the CPU, including the support of particular extensions. It is primarily used to get such information in runtime in order to avoid distributing a separate binary for each microarchitecture. Its output information is returned very densely in the form of feature masks, so compilers provide built-in methods to make sense of it. Here is an example:

```c++
#include <iostream>
using namespace std;

int main() {
    cout << __builtin_cpu_supports("sse") << endl;
    cout << __builtin_cpu_supports("sse2") << endl;
    cout << __builtin_cpu_supports("avx") << endl;
    cout << __builtin_cpu_supports("avx2") << endl;
    cout << __builtin_cpu_supports("avx512f") << endl;

    return 0;
}
```

Second, we need to include a header file that contains the subset of intrinsics we need. Similar to `<bits/stdc++.h>` in GCC, there is `<x86intrin.h>` header that contains all of them, so we will just use that.

And last, we need to tell the compiler that the target CPU actually supports these extensions. This can be done either with `#pragma GCC target(...)` [as we did before](../), or with `-march=...` flag in the compiler options. If you are compiling and running the code on the same machine, you can set `-march=native` to auto-detect the microarchitecture.

## SIMD Registers

The most notable distinction between SIMD extensions is the support for wider registers:

- SSE added 16 128-bit registers called `xmm0` through `xmm15`.
- AVX added 16 256-bit registers called `ymm0` through `ymm15`.
- AVX512[^mask] added 16 512-bit registers called `zmm0` through `zmm15`.

[^mask]: AVX512 also added 8 so-called *mask registers* named `k0` through `k7`, which are used for masking and blending data. We are not going to cover them and will mostly use AVX2 and previous standards.

As you can guess from the naming, and also from the fact that 512 bits already occupy a full cache line, x86 designers are not planning to add wider registers anytime soon.

C/C++ compilers implement special *vector types* that refer to the data stored in these registers:

- 128-bit `__m128`, `__m128d` and `__m128i` types for single-precision floating-point, double-precision floating-point and various integer data respectively;
- 256-bit `__m256`, `__m256d`, `__m256i`;
- 512-bit `__m512`, `__m512d`, `__m512i`.

Registers themselves can hold data of any kind: these types are only used for type checking. To convert a variable to another type, you can do it the same way you would convert any other type, and it won't cost you anything.

## SIMD Intrinsics

*Intrinsics* are C-style functions that do something with vector data types, usually by just calling the associated assembly instruction.

For example, here is a cycle that adds together two arrays of 64-bit floating-point numbers using AVX intrinsics:

```c++
double a[100], b[100], c[100];

// iterate in blocks of 4,
// because that's how many doubles can fit into a 256-bit register
for (int i = 0; i < 100; i += 4) {
    // load two 256-bit segments into registers
    __m256d x = _mm256_loadu_pd(&a[i]);
    __m256d y = _mm256_loadu_pd(&b[i]);

    // add 4+4 64-bit numbers together
    __m256d z = _mm256_add_pd(x, y);

    // write the 256-bit result into memory, starting with c[i]
    _mm256_storeu_pd(&c[i], z);
}
```

Note that, in general, we may have a problem here if the length of the array is not divisible by the block size. There are two common solutions to this:

1. We can "overshoot" by iterating over the last incomplete segment either way. To make sure sure we don't segfault by trying to read from or write to a memory region we don't own, we need to pad the arrays to the nearest block size (typically with some "neutral" element, e. g. zero).
2. Make one iteration less and write a little loop in the end that calculates the remainder normally (with scalar operations).

Humans prefer #1, because it is simpler and results in less code. Compilers prefer #2, because they don't really have another legal option.

### Instruction References

Most SIMD intrinsics follow a naming convention similar to `_mm<size>_<action>_<type>`, and are relatively self-explanatory once you get used to assembly naming conventions.

Here are a few more examples, just so that you get the gist of it:

- `_mm_add_epi16`: add two 128-bit vectors of 16-bit *extended packed integers*, or simply said, `short`s.
- `_mm256_acos_pd`: calculate elementwise $\arccos$ for 4 *packed doubles*.
- `_mm256_broadcast_sd`: broadcast (copy) a `double` from a memory location to all 4 elements of the result vector.
- `_mm256_ceil_pd`: round up each of 4 `double`s to the nearest integer.
- `_mm256_cmpeq_epi32`: compare 8+8 packed `int`s and return a mask that contains ones for equal element pairs.
- `_mm256_blendv_ps`: pick elements from one of two vectors according to a mask.

As you may have guessed, there is a combinatorially very large number of intrinsics. A very helpful reference for x86 intrinsics is the [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/), which has groupings by categories and extensions, descriptions, pseudocode, associated assembly instructions, and their latency and throughput on Intel microarchitectures. You may want to bookmark that page.

The Intel reference is useful when you know that a specific instruction exists and just want to look up its name or performance info. When you don't know whether it exists, this [cheat sheet](https://db.in.tum.de/~finis/x86%20intrinsics%20cheat%20sheet%20v1.0.pdf) may do a better job.

Also note that compilers do not necessarily pick the exact instruction that you specify. Similar to the scalar `c = a + b` we [discussed before](/hpc/analyzing-performance/assembly), there is a fused vector addition instruction too, so instead of using 2+1+1=4 instructions per loop cycle, compiler [rewrites the code above](https://godbolt.org/z/dMz8E5Ye8) with blocks of 3 instructions like this:

```nasm
vmovapd ymm1, YMMWORD PTR a[rax]
vaddpd  ymm0, ymm1, YMMWORD PTR b[rax]
vmovapd YMMWORD PTR c[rax], ymm0
```

Also, some of the intrinsics are not direct instructions, but short sequences of instructions. One example is the `extract` group of instructions, which are used to get individual elements out of vectors (e. g. `_mm256_extract_epi32(x, 0)` returns the first element out of 8-integer vector); it is quite slow (~5 cycles) to move data between "normal" and SIMD registers in general.

## Vector Extensions

If you feel like the design of C intrinsics is terrible, you are not alone. I've spent hundreds of hours writing SIMD code and reading the Intel Intrinsics Guide, and I still can't remember whether I need to type `_mm256` or `__m256`.

Intrinsics are not only hard to use but also neither portable nor maintainable. In good software, you don't want to maintain different procedures for each CPU: you want to implement it just once, in an architecture-agnostic way.

One day, compiler engineers from the GNU Project thought the same way and developed a way to define your own vector types that feel more like arrays with some operators overloaded to match the relevant instructions.

In GCC, here is how you can define vector of 8 integers packed into a 256-bit (32-byte) register:

```c++
typedef int v8si __attribute__ (( vector_size(32) ));
// type ^   ^ typename          size in bytes ^ 
```

Unfortunately, this is not a part of the C or C++ standard, so different compilers use different syntax for that.

There is somewhat of a naming convention, which is to include size and type of elements into the name of the type: in the example above, we defined a "vector of 8 signed integers". But you may choose any name you want, like `vec`, `reg` or whatever. The only thing you don't want to do is to name it `vector` because of how much confusion there would be because of `std::vector`.

### Examples

The main advantage of using these types is that for many operations you can use normal C++ operators instead of looking up the relevant intrinsic.

```c++
v4si a = {1, 2, 3, 5};
v4si b = {8, 13, 21, 34};

v4si c = a + b;

for (int i = 0; i < 4; i++)
    printf("%d\n", c[i]);

c *= 2; // multiply by scalar

for (int i = 0; i < 4; i++)
    printf("%d\n", c[i]);
```

With vector types we can greatly simplify the "a + b" loop we implemented with intrinsics before:

```c++
typedef double v4d __attribute__ (( vector_size(32) ));
v4d a[100/4], b[100/4], c[100/4];

for (int i = 0; i < 100/4; i++)
    c[i] = a[i] + b[i];
```

Now, armed with a nicer syntax, consider a slightly more complex example: calculating the sum an array.

```c++
int sum_naive(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s;
}
```

The naive approach is not so straightforward to vectorize, because the state of the loop (sum $s$ on the current prefix) depends on the previous iteration.

The way to overcome this is to split a single scalar accumulator $s$ into 8 separate ones, so that $s_i$ would contain the sum $\sum_{j=0}^{n / 8} a_{8 \cdot j + i }$, that is, the sum of every 8th element of the original array, shifted by $i$. If we store these 8 accumulators in a single 256-bit vector, we can update them all at once by adding consecutive 8-elements segments of the array:

```c++
int sum_simd(v8si *a, int n) {
    //       ^ you can just cast a pointer normally, like with any other pointer type
    v8si s = {0};

    for (int i = 0; i < n / 8; i++)
        s += a[i];
    
    int res = 0;
    
    // sum 8 accumulators into one 
    for (int i = 0; i < 8; i++)
        res += s[i];

    // add the remainder of a
    for (int i = n / 8 * 8; i < n; i++)
        res += a[i];
        
    return res;
}
```

The last part, where we sum up the 8 accumulators into a single scalar to get the total sum, can be done a bit faster by what's called "horizontal summation", which is the repeated use of [special instructions](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#techs=AVX,AVX2&text=_mm256_hadd_epi32&expand=2941) that add together pairs of adjacent elements:

```c++
int hsum(__m256i x) {
    __m128i l = _mm256_extracti128_si256(x, 0);
    __m128i h = _mm256_extracti128_si256(x, 1);
    l = _mm_add_epi32(l, h);
    l = _mm_hadd_epi32(l, l);
    return _mm_extract_epi32(l, 0) + _mm_extract_epi32(l, 1);
}
```

Although this is roughly how compilers vectorize it, this is not the fastest way to sums and other array reductions. We will come back to this problem in the last chapter.

---

As you can see, vector extensions are much cleaner compared to the nightmare we have with intrinsic functions. But some things that we may want to do are just not expressible with native C++ constructs, so we will still need intrinsics. Luckily, this is not an exclusive choice, because vector types support zero-cost conversion to the `_mm` types and back. We will, however, try to avoid doing so as much as possible and stick to vector extensions when we can.