(XXX: Credit JP and DH)


This is an article about doing raytracing on the CPU as fast as possible. Because we are focusing on
making it fast, the article will assume some knowledge of raytracing and the involved math and
programming. To not leave people behind, I'll try explaining the basics as we go, but to get deeper
understanding, I wholeheartedly recommend the Raytracing Weekend book series
(https://raytracing.github.io/).

The article stems from work I did in early 2023. As of the time of writing (December 2025), that
work is the most focused technical work I have done. This is quite the luxury for me, as I usually
don't get to do three months of mostly focused work on a single part of the system, and instead have
to prioritize what is the most valuable thing for me to work on, often times leaving programming
gems to other people on the team. This was a rare exception.

Also note that I am a generalist, and thus don't consider myself deeply knowledgeable in computer
graphics. I am painfully aware that I could have missed an obvious technique that could have made
the results better. If after reading the article you think of something I did not explore
thoroughly, I'll be delighted to get your email. Now, before we get to the core of the problem,
here's the last bit of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team. The goal of the company (as percieved back then) was to build
software to procedurally generate housing architecture meeting the developer's criteria, such as
yields, a correct mix of functions and apartment sizes, aesthetics, as well as spatial inputs
(e.g. build here, but not there). We assumed that to generate that architecture, there would be a
computer learning process, and as it is with learning processes, they need to get feedback on their
results that can be fed into the next iteration, gradually improving the results. Generating livable
architecture is a problem of very high dimensionality - before we even enter the realm of what is
architecture (and what is good architecture), there's structural engineering, business, regulatory
and legal criteria that a development project must satisfy. A house is definitely not just any
"house shaped" geometry. Thus we assumed that even a smartly designed learning algorithm will have
to do many iterations, getting feedback from a myriad of evaluators (structural, accoustic,
sunlight/daylight, business, legal) until it reaches a solution that even looks like it could have
been designed by a human, let alone a good one.

(TODO(jt): Include GIF here?)

So we generate stuff, evaluate, generate new stuff, evaluate, and so on, until eventually we stop
after either all criteria are satisfied, no significant improvement can be found, or we exceeded our
computational budget. And because the problem is going to be hard, we need to use that budget very
well to squeeze in as many iterations as possible. At the time we weren't quite sure whether we need
to be fast so that we have a shot at being quasi-realtime, or to at least finish the computation
overnight on a server farm [1].

[1]: The state as of me leaving the company is that for small scenes it took tens of seconds to get
something, and minutes to get something useful. This degraded to having to do overnight runs for
larger scenes. The problem is hard.

We are going to focus on one particularly computationally demanding part of the evaluation process:
the daylight evaluation. For the uninitiated, daylight evaluation is something that tells you how
much light accumulates at a point on the surface of the interior of a building for some chosen
stable lighting conditions outside. This is usually done for tens of thousands of points inside many
rooms and many buildings in a city block, both in the building you plan to build, and in the
buildings that surround them.

The way daylight evaluation is usually implememnted is ray tracing, although there are less
computationally intensive and less precise ways that were used before the wide adoption computers of
in architecture studios. Interestingly enough, all current solutions (Ladybug (XXX: link), Climate
Studio (XXX: link)) internally depend on just one piece of ray-tracing software called Radiance
(https://github.com/LBNL-ETA/Radiance/tree/master). This will be important later.

To bridge this closer to the videogame audience, the daylighting evaluation is very similar to what
game developers call light baking: computing light simulation for static scenes and baking the
results into planar (lightmaps) or spatial (light probes) textures, so that we know how a scene is
lit without having to compute it at runtime. The differences between daylight evaluation and light
baking come down to how the results are used, not how they are computed. Compared to light baking,
for daylight evaluation we didn't care about color, only intensity, nor did we need to produce
information about which direction the light is coming from for our light probes. We also only
simulated perfectly diffuse (Lambertian) materials, which made some localized parts of the raytracer
simpler. Additionally, there's a few peripheral bits about interpretting the results that are
specific to the AEC industry regulations, but I am going to purposefully ignore these, and instead
focus on the ray tracer core, which I believe looks the same as it would in a light baker.

# Baker Architecture, Appetizer

At a high level, a light baker operates with a geometric description of the scene, and a list of
raytracing tasks it needs to compute on that scene.

(XXX: Picture)



(XXX: explain ray tracing basics, including bounces)

Quite often, the list of tasks can change, while the scene remains the same, an example of this
being the simulation of real time global illumination (GI) only happening for the part of the scene
that a game will need to render its next frame. Another common situation is the scene changing ever
so slightly from frame to frame, in which case we can maybe reason about the static and dynamic
parts of the scene differently.

The simplest, most flexible, and sometimes sufficient way to represent the scene is to have arrays
of various geometric primitives: triangles, spheres, planes, boxes, etc. Raytracing the scene is
about going over each ray and testing it against all of these arrays, remembering information about
the closest hit, so that we know where to start our next bounce.

(XXX: Pseudocode)

Before addressing the elephant in the room and discarding this approach, I want to mention some of
its benefits. First, there is no need to build acceleration structures - you already have the arrays
of objects, or you can cheaply produce them, in case your representation does not exactly match what
the raytracer needs. This is not going to be true for the more sophisticated designs, where a part
of the raytracing cost will be spent on organizing data to accelerate raytracing. Another good thing
about these arrays is that the resulting code is very easy to SIMD. Going wide can happen over rays
or over geometries, both of which have their advantages and disadvantages. When going wide over
rays, problems arise with rays diverging, some rays potentially hitting the end of their journey
after less bounces than others, meaning we have dead rays in our wide registers, and either have to
do something about that, or accept the wasted work. This was the route taken by Casey Muratori in
his Handmade Ray miniseries: https://guide.handmadehero.org/ray/. When going wide over geometry, the
main downside is the cost of the memory trafic to the CPU. Each geometric primitive is loaded to be
tested against a ray, evicted by subsequent loads [2], only to be loaded again when the next ray is
going to need that exact memory [3].

[2]: The exact level of eviction (register file, L1, L2, ...) depends on (and gets worse with) the
size of the working set.

[3]: The wide-over-rays approach mitigates the cost of the memory traffic by doing more useful work
for each cache line it had to load.

The elephant is algorithmic in nature. We are visiting each geometry for each ray bounce, but most
of those rays have no way to reach most geometries. There are too many wasted loads and calculations
per bounce. To get away from the O(m*n), we use acceleration structures to help with eliminating
impossible hits, such as Bounding Volume Hierachies (BVH), k-d Trees, or Octrees. Choosing between
them depends on the character of your data. We went with the BVH, because it doesn't have many
assumptions about its contents and degrades gracefully with bad quality of the input data [4].

[4]: We didn't want to constrain the rest of what we are going to build by choosing an overly picky
datastructure. This also helped us productize the daylighting evaluator later.

(XXX: Explain BVH)

XXX: HERE Describe BVH raytracer: bounding boxes and triangles.

However, as we naively enter the land of computer science, we seem to have lost our ability to
utilize modern hardware, and have to do some thinking to recover it.

# Baker Architecture, Main Course

- Wide BVH: the array of box packs and the array of triangle packs
- SIMD box intersection
- SIMD triangle intersection (maybe have to explain moller trumbore first?)

# Dessert

- Naive triangle intersection -> Moller Trumbore
- Pre-baked sphere rays.
- SAH optimization
- targetting multiple SIMD backends within one binary: avx2, sse2, neon, fallback

# Future experiments

- Would avx512 help? 16-wide BVH means a lot of memory traffic. How many cache lines do we have to
  read on average for a sinle ray? And how many 512-bit loads?
- f16?
- Hiding indices in floats?

# Conclusion

- mention that this is 1000x faster than Climate Studio and Ladybug (for projects that Ladybug can even load)
- I might re-implement and opensource the generally useful parts
