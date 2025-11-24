TODO(jt): SIMD raytracing on the CPU is something I spend months on, and it is relatively
interesting. Maybe I should write about, while I still remember?


- architecture
- Wide BVH
- SAH optimization
- SIMD box intersection
- SIMD Moller-Trumbore
- wide on bboxes and triangles, not on rays
- targetting multiple SIMD backends within one binary: avx2, sse2, neon, fallback
- Would avx512 help? 16-wide BVH means a lot of memory traffic.
- f16
