This is an article about doing raytracing on the CPU as fast as possible. Because we are focusing on
making it fast, the article assumes some knowledge of raytracing, and the involved math and
programming. To not leave people behind, I'll try explaining the basics as we go, but to get deeper
understanding, I wholeheartedly recommend the excellent Raytracing Weekend book series
(https://raytracing.github.io/).

The article stems from work I did in early 2023. As of the time of writing (December 2025), that
work is the most focused technical work I have done. This is quite the luxury for me, as I usually
don't get to spend three months on a single focused part of the system, and instead have to
prioritize what is the most valuable thing for me to work on, often times having to leave
programming gems to other people on the team. This was a rare exception.

Also please note that I consider myself a generalist, and don't have the deep knowledge of someone
who specializes in raytracing. It was a lot of fun and I learned a lot, but I am painfully aware
that I could have missed an obvious technique that could have made the results better. If, after
reading the article, you think of any area I did not explore thoroughly, I'll be delighted to get
your email. Now, before we get to the core of the problem, here's the last bits of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team. The goal of the company (as percieved back then) was to build
software to procedurally generate housing architecture meeting the developer's criteria, such as
yields, a correct mix of functions and apartment sizes, aesthetics, etc. We assumed that to generate
that architecture, there is going to be a computer learning process, and as it is with learning
processes, they need to get feedback on their results that can be fed into the next
iteration. Generating livable architecture is a problem of very high dimensionality, and we assumed
that even a smartly designed learning algorithm will have to do many iterations until it reaches a
solution that even looks like it could have been designed by a human, let alone a good one.

(TODO(jt): Include GIF here?)

So we generate stuff, evaluate, generate new stuff, evaluate, and so on, until eventually we stop
after either all criteria are satisfied, no significant improvement can be found, or we exceeded our
computational budget. And because the problem is going to be hard, we need to use that budget very
well to squeeze in as many iterations as possible. At the time we weren't quite sure whether we need
to be fast so that we have a shot at being quasi-realtime, or to at least finish the computation
overnight on a server farm.

We are going to focus on one particular part of the evaluation process, one that is computationally
demanding: the daylight evaluation. For the uninitiated, daylighting is something that tells you how
much light accumulates at a point on the surface of the interior of a building, usually for tens of
thousands of points inside many rooms and many buildings in a city block. The way it is usually
implememnted is ray tracing, although there are less computationally intensive and less precise ways
that were used before the wide adoption computers. Interestingly enough, all current solutions
(Ladybug, Climate Studio) internally depend on a rather dated piece of ray-tracing software called
Radiance (https://github.com/LBNL-ETA/Radiance/tree/master). This will be important later.

To bridge this closer to videogames, the daylighting evaluation is very similar to what game
developers call light baking: the act of computing precise light simulation for static scenes and
baking the results into planar (lightmaps) or spatial (light probes) textures. The differences here
are minute, and can be summed up as we didn't have to care about color, only intensity, nor did we
need to produce information about which direction the light is coming from, and we only simulated
perfectly diffuse (Lambertian) materials. There's also a few peripheral bits of nonscientific nature
about interpretting the ray tracing results that are specific to the AEC industry regulations, but I
am going to purposefully ignore these, and instead focus on the ray tracer core, which I believe
looks the same as it would in a light baker.

# Baker Architecture, First pass

At a high level, a light baker operates with a geometric description of the scene, and a list of
raytracing tasks it needs to compute on that scene. Quite often, the list of tasks can change, while
the scene remains the same, an example of this being the simulation of real time global illumination
(GI) only happening for the part of the scene that a game will need to render its next
frame. Another common situation is the scene changing ever so slightly from frame to frame, in which
case we can maybe reason about the static and dynamic parts of the scene differently.

The simplest, most flexible, and sometimes even sufficient way to represent the scene is to have
arrays of various geometric primitives: triangles, spheres, planes, boxes, etc. The act of
raytracing the scene is then to go over each ray we want to evaluate and test it against all of
these arrays, remembering information about the closest hit.

(XXX: Explaing naive array raytracer)
(XXX: explain bounces)

(TODO(jt): Pseudocode)

Before addressing the elephant in the room and discarding this approach, I do want to go over some
of its benefits. First, there are no acceleration structures - you already have the arrays of
objects, or you can cheaply produce them, in case your representation is not compact. This is not
going to be true for the more sophisticated designs, where a part of the raytracing cost will be
spent on building acceleration structures. Another good thing here is the resulting code is very
easy to SIMD. Going wide can happen over rays, or over geometries, both of these have their
advantages and disadvantages. When going wide over rays, problems arise with rays diverging, some
potentially hitting the end of their journey after less bounces than others, which now means we have
dead rays in our wide registers, and either have to do something about that, or accept the wasted
work. This was the route taken by Casey Muratori in his Handmade Ray miniseries:
https://guide.handmadehero.org/ray/. When going wide over geometry, the main cost is in the memory
trafic to the CPU. Each geometric primitive is loaded to be tested against a ray, evicted by
subsequent loads [1], only to be loaded again when the next ray is going to need that exact
memory [2]. The last benefit of the array approach is that because there are no additional
datastructures necessary, it may just be that for your problem set, the data may be small enough to
fit in a CPU cache.

[1]: The exact level of eviction (register file, L1, L2, ...) depends on the size of the working set.

[2]: The wide-over-rays approach mitigates the cost of this by doing more useful work for the data
it had to load.


Now, the elephant is of algorithmic nature. We are visiting each geometry for each ray bounce, but
most of those rays have no way to reach most geometries. There are too many wasted loads and
calculations per bounce. There are acceleration structures that help with eliminating impossible
hits: Bounding Volume Hierachies (BVH), k-d trees, octrees. Which one is best depends on the
character of your data. We picked the BVH, because it assumes the least, and we didn't want to
constrain the rest of what we are going to build by choosing an overly picky datastructure. The
BVH can deal with any, even degenerate data.

(XXX: Explain BVH)

XXX: HERE Describe BVH raytracer: bounding boxes and triangles.

However, as we naively enter the land of computer science, we seem to have lost our ability to
utilize modern hardware, and have to do some thinking to recover it.

---

TODO(jt): SIMD raytracing on the CPU is something I spend months on, and it is relatively
interesting. Maybe I should write about, while I still remember?


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

- mention that this is 1000x faster than Climate Studio and Ladybug (for projects that Ladybug can even load)
- company going under -> opensource the thing?
