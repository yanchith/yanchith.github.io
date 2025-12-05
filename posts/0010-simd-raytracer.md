At the moment of writing this (December 2025), the work I am about to talk about is the most focused
technical work I have done. This was quite the luxury for me, as I usually don't get to dive deep
for 3 months in a single technical topic, and instead spend my days putting out fires. While I don't
think I have impostor syndrome anymore (the same way I can't say I am exactly young anymore), I am
well aware of my limitations. This was me doing reasearch and programming punching well above my
weight class. It was a lot of fun, and I learned a lot, but I am painfully aware that I could have
missed an obvious technique that could have made the results better. If, after reading the article,
you think of anything like that, I'll be delighted to get your email. Now, before we get to the core
of the problem, here's a little bit of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team. The goal of the company as percieved back then was to procedurally
generate housing architecture meeting the developer's criteria, such as yields, aesthetics, a
correct mix of functions and apartment sizes, etc. (This is still the goal of the company today, but
things are necessarilly more nuanced.) We assumed that to generate that architecture, there is going
to be a computer learning process, and as it goes with learning processes, they need to get feedback
on their results that can be fed into the next iteration. We didn't know for certain back then, but
we assumed the problem has very high dimensionality, and that even a smartly designed learning
algorithm will have to do many iterations until it reaches a solution that even looks like it could
have been designed by a human, let alone a good one.

So the loop is generate stuff, evaluate, generate new stuff, evaluate, and eventually stop after
either all criteria are satisfied, no significant improvement can be found, or we ran out of
computational budget. And because the problem is going to be hard, we need to use that budget very
well. The rest of this post is going to focus on one particular part of the evaluation process. One
that is computationally demanding: the daylighting evaluation, something that tells you how much
light does a point on the surface of the interior of a building get, for (usually) tens of thousands
of points.

To bridge this closer to game developer audience, the daylighting evaluation is very similar to
light baking, except we don't care about the light color, only intensity, nor do we need to produce
information about which direction the light is coming from - all we need is the light intensity at
the light probe's location. There's a few boring bits specific to the AEC industry we had to do as
well, but I am going to purposefully ignore these for the rest of the article, and instead focus on
the ray tracer core, which I believe looks the same as it would in a light baker.

# Baker Architecture, First pass


---

TODO(jt): SIMD raytracing on the CPU is something I spend months on, and it is relatively
interesting. Maybe I should write about, while I still remember?

- context, company start, team warmup


- architecture, naive pass
- Naive triangle intersection -> Moller Trumbore
- Wide BVH
- SAH optimization
- SIMD box intersection
- SIMD Moller-Trumbore
- wide on bboxes and triangles, not on rays
- targetting multiple SIMD backends within one binary: avx2, sse2, neon, fallback
- Would avx512 help? 16-wide BVH means a lot of memory traffic. How many cache lines do we have to
  read on average for a sinle ray?
- f16?

- mention that this is 1000x faster than Climate Studio (for projects that Climate Studio can load)
- company going under -> opensource the thing?
