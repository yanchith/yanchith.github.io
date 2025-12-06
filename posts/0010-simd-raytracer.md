(XXX: say somewhere here, maybe just in title, that this is a post about making CPU raytracing fast)

As of the moment of writing this (December 2025), the work I am about to talk about is the most
focused technical work I have done. This is quite the luxury for me, as I usually don't get to dive
deep for 3 months into a single focused part of the system, and instead each day have to prioritize
what is the most valuable thing for me to work on, often times having to leave programming gems to
the younger people on the team. This was a rare exception.

Also please note that this was me doing research and programming well above my weight class. It was
a lot of fun, and I learned a lot, but I am painfully aware that I could have missed an obvious
technique that could have made the results better. If, after reading the article, you think of any
area I did not explore thorougly, I'll be delighted to get your email. Now, before we get to the
core of the problem, here's the last bits of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team, as only some of us have worked together before. The goal of the
company (as percieved back then) was to build software to procedurally generate housing architecture
that met the developer's criteria, such as yields, a correct mix of functions and apartment sizes,
aesthetics, etc. This is still the goal of the company today, but things are necessarilly more
nuanced. We assumed that to generate that architecture, there is going to be a computer learning
process, and as it is with learning processes, they need to get feedback on their results that can
be fed into the next iteration. Generating livable architecture is a problem of very high
dimensionality, and we assumed that even a smartly designed learning algorithm will have to do many
iterations until it reaches a solution that even looks like it could have been designed by a human,
let alone a good one.

(TODO(jt): Include GIF here?)

So the loop is generate stuff, evaluate, generate new stuff, evaluate, and so on, until eventually
we stop after either all criteria are satisfied, no significant improvement can be found, or we
exceeded our computational budget. And because the problem is going to be hard, we need to use that
budget very well to squeeze in as many iterations as possible. At the time we didn't weren't quite
sure whether we need to be fast so that we have a shot at being quasi-realtime, or to at least
finish the computation overnight on a server farm.

We are going to focus on one particular part of the evaluation process, one that is computationally
demanding: the daylight evaluation. For the uninitiated, daylighting is something that tells you how
much light accumulates at a point on the surface of the interior of a building, usually for tens of
thousands of points inside many rooms and many buildings. The way it is usually implememnted is ray
tracing, although there are less computationally intensive and less precise ways that were used last
century. Interestingly enough, all current solutions (Ladybug, Climate Studio) internally depend on
a rather old piece of software called Radiance
(https://github.com/LBNL-ETA/Radiance/tree/master). This will be important later.

To bridge this closer to videogames, the daylighting evaluation is very similar to what we call
light baking: the act of computing precise light simulation for static scenes and baking the results
into planar (lightmaps) or spatial (light probes) textures. The differences here are minute, and can
be summed up as we don't care about the light color, only intensity, nor do we need to produce
information about which direction the light is coming from. All we need is the light intensity at
the light probe's location. There's also a few peripheral bits of nonscientific nature about
interpretting the ray tracing results that are specific to the AEC industry regulations, but I am
going to purposefully ignore these for the rest of the article, and instead focus on the ray tracer
core, which I believe looks the same as it would in a light baker.

# Baker Architecture, First pass

Because we are going to be focusing on how to make CPU raytracing fast, this is the time to mention
that the article assumes some knowledge of raytracing, and the involved math and programming. An
excellent resource for this is the Raytracing Weekend book series (https://raytracing.github.io/).

At a high level, a light baker operates with a geometric description of the scene, and a list of
raytracing tasks it needs to compute on that scene. Quite often, the list of tasks can change, while
the scene remains the same, an example of this being real time global illumination (GI) computation
only happening for the part of the scene that the engine will need to render its next frame. Another
common situation is the scene changing ever so slightly from frame to frame, in which case we can
maybe reason about the static and dynamic parts of the scene differently.

The simplest, most flexible, and sometimes even sufficient way to represent the scene is to have
arrays of various geometric primitives: triangles, spheres, planes, boxes, etc. The act of
raytracing the scene is then to go over each ray we want to evaluate, and check it against all of
these arrays. Before addressing the obvious elephant in the room and discarding this approach, I do
want to go over the benefits. First off, there is (almost) no build step for the scene - you already
have the arrays of objects, or you can cheaply produce them. This is not going to be true for other
approaches, and a part of the raytracing cost is going to be the build of the acceleration
structures. Another good thing here is the resulting code is very easy to both SIMD and
multithread. The SIMDification can happen over the rays, or over the geometries. Both of these have
their sets of advantages and disadvantages. When going wide over rays, we have to deal with the
situation of rays diverging after the first bounce, some potentially hitting the end of their
journey, which now means we have to somehow deal with dead rays in our wide registers, or accept the
wasted work they mean. This was the route taken by Casey Muratori in his Handmade Ray miniseries:
https://guide.handmadehero.org/ray/.

- XXX: Going wide over geometires
- XXX: fit in cache, because there's no additional data




- probes, triangles, bounding boxes


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
