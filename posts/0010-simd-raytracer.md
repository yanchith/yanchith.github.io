(XXX: Credit JP and DH)


This is an article about raytracing on the CPU as fast as possible. Because we are focusing on fast,
the article will assume some knowledge of raytracing and the involved math and programming. To not
leave people behind, I'll try explaining the basics as we go, but to get deeper understanding, I
wholeheartedly recommend the Raytracing Weekend book series as a primer
(https://raytracing.github.io/).

The article stems from work I did in early 2023. As of the time of writing (December 2025), that
work is the most focused technical work I have done. This is quite the luxury for me, as I usually
don't get to concentracte for three months on a single part of the system, and instead have to
prioritize what is the most valuable thing for me to work on, often times leaving programming gems
to other people on the team. This was a rare exception.

Also note that I am a generalist, and thus don't consider myself deeply knowledgeable in computer
graphics. I am painfully aware that I could have missed an obvious technique that could have made
the results better. If after reading the article you think of something I did not explore
thoroughly, I'll be delighted to get your email. Now, before we get to the core of the problem,
here's the last bit of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team. That warmup happened by us building a small part of the system
first before spinning out wide. A small but important part we knew we needed. The goal of the
company (as percieved back then) was to build software to procedurally generate housing
architecture. This housing was supposed to meet the developer's criteria, such as yields, a correct
mix of functions and apartment sizes, aesthetics, as well as spatial inputs (e.g. build here, but
not there). We assumed that to generate that architecture, there would be a computer learning
process, and as it is with learning processes, they need to get feedback on their results that can
be fed into the next iteration, gradually improving the results. Generating livable architecture is
a problem of very high dimensionality - before we even enter the realm of what is architecture (and
what is good architecture), there's structural engineering, business, regulatory and legal criteria
that a development project must satisfy. A house is definitely not just any "house shaped"
geometry. Thus we assumed that even a smartly designed learning algorithm will have to do many
iterations, getting feedback from a myriad of evaluators (structural, accoustic, sunlight/daylight,
business, legal) until it reaches a solution that even looks like it could have been designed by a
human, let alone a good one.

(TODO(jt): Include GIF here?)

So we generate stuff, evaluate, generate new stuff, evaluate, and so on, until eventually we stop
after either all criteria are satisfied, no significant improvement can be found, or we exceeded our
computational budget. And because the problem is going to be hard, we need to use that budget well
to squeeze in as many iterations as possible. At the time we weren't quite sure whether we need to
be fast so that we have a shot at being quasi-realtime, or to at least finish the computation
overnight on a server farm [1].

[1]: The state as of me leaving the company is that for small scenes it took tens of seconds to get
something, and minutes to get something useful. This degraded to having to do overnight runs for
larger scenes. The problem is hard.

The team warmup focused on one particularly computationally demanding part of the evaluation
process: the daylight evaluation. For the uninitiated, daylight evaluation is something that tells
you how much light accumulates at a point on the surface of the interior of a building for some
chosen stable lighting conditions outside. This is then done for tens of thousands of points inside
many rooms and many buildings in a city block, both in the buildings you plan to build, and in the
buildings that surround them.

The way daylight evaluation is usually implememnted is ray tracing [radiosity]. Interestingly
enough, all current solutions (Ladybug (XXX: link), Climate Studio (XXX: link)) internally depend on
just one piece of ray-tracing software called Radiance
(https://github.com/LBNL-ETA/Radiance/tree/master). This will be important later.

[radiosity]: Although there are less computationally intensive and less precise ways that were used
before the wide adoption computers of in architecture studios.

To bridge this closer to the videogame audience, the daylighting evaluation is very similar to what
game developers call light baking: computing light simulation for static scenes and baking the
results into planar (lightmaps) or spatial (light probes) textures, so that we know how a scene is
lit without having to compute it at runtime. The differences between daylight evaluation and light
baking come down to how the results are used, not how they are computed. Compared to light baking,
daylight evaluation doesn't care about color, only intensity, nor does it need to produce
information about which direction the light is coming from for the light probes. We also only need
to simulate perfectly diffuse (Lambertian) materials, which made some parts of the raytracer
simpler. Additionally, there's a few peripheral bits about interpretting the results that are
specific to the AEC industry regulations, but I am going to purposefully ignore these, and instead
focus on ray tracing, which I believe looks the same as it would in a light baker.

# Ray Tracing

(XXX: Picture)

Ray tracing is a process of simulating light. For a point in space, for instance a pixel of a
digital camera, we want to determine the amount and color of light that point receives. We do that
by simulating paths that light could take through the scene from a light source to reach our
point. Such path can be either direct, or include one or more bounces from objects in the
scene. However, we are often interested in light reaching just a few select points in space (like
the chip of our digital camera), which rays going from the light sources have a miniscule chance to
reach, directly or via bounces. We would have to shoot a lot of rays from the light source for them
to reach our measurement points in sufficient quantities.

Instead, we lean on a property of the universe regarding small number of particles. If we were shown
a movie of elementary particles moving through space, sometimes colliding with each other, we would
have a hard time telling, whether the movie is playing forward or backward. This is because for a
small number of particles, it is impossible to tell. They behave exactly the same whether they
are moving forward or backward in time [oversimplification], and it is only possible to discern the
arrow of time once we have a large number of particles and probability enters the picture, pushing
particles towards states with high entropy [probability]. For us, this means that if we have a path between a
measurement point and a light source, a photon could have taken that exact path both ways, and it
would have transferred the same amount of energy.

[oversimplification]: This is a huge oversimplification on many levels. For instance photons do not
necessarily bounce of off all surfaces. Sometimes they are absorbed, and a new photon is emitted,
etc. However, there is a grain of truth in this model.

[probability]: Particles actually behave the same even in large numbers. The probabilistic behavior
is emergent.

The implication of this bidirectionality for our simulation is that we can trace rays in reverse:
from the relatively few points we are interested in towards light sources. The results are going to
be the same, but for most realistic scenes, we have higher chance of sucessfully completing the path
between the light source and the camera. With both forward and reverse ray tracing, we keep track of
how much energy we loose with each bounce. Because these energy losses (also called attenuation) are
multiplicative, and multiplication is commutative, this works out the same regardless of the
direction we trace the ray in.

When doing raytracing in software, we usually do reverse ray tracing. We shoot rays from our
measurement points, hoping they eventually reach sources of light. These rays go in a straight line
until they hit something. Depending on what was hit, various things happen. If the ray hits a light
reflecting surface, it bounces off of the object back into the scene. The new direction of the ray
depends on the physical properties of the surface. Some surfaces reflect the ray as a mirror would,
while other, diffuse surfaces deflect the ray in a seemingly random direction. This behavior is
described by the Bidirectional Reflectance Distribution Function (BRDF), and in general can be quite
complex, for instance the BRDF for animal fur behaves differently for various orientations of the
incoming and outgoing rays. Because each bounce attenuates the ray, we can decide to terminate the
ray early after a certain amount of bounces, as even if it did reach a light source eventually, the
light's contribution through that particular ray would have been close to zero. If the ray hits a
light source, it ends its journey. We read the light's value and apply the attenuation the ray has
accumulated when bouncing off of materials along its path.

Hopefully multiple rays shot from our point reach light sources. The light value at the point is the
sum of the individual ray contributions divided by the number of cast rays. Because scenes are not
always well lit, it may take shooting a lot of rays for us to form a coherent picture of what the
scene looks like. Low light scenes are also prone to noise, when various neighboring pixels in the
camera collect dramatically different values of light. As the number of rays goes to infinity, the
noise disappears, but because a large number of rays is not always practical, we often employ
denoising algorithms that reconstruct information from noisy pictures.

# Raytracer Architecture, Appetizer

At a high level, a light baker operates with a geometric description of the scene, and a list of
raytracing tasks it needs to compute on that scene. Quite often, the list of raytracing tasks can
change, while the scene remains the same, an example of this being the simulation of real time
global illumination (GI) only happening for the part of the scene that a game will need to render
its next frame. Another common situation is the scene changing ever so slightly from frame to frame,
in which case we can maybe reason about the static and dynamic parts of the scene differently.

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
tested against a ray, evicted by subsequent loads [eviction], only to be loaded again when the next
ray is going to need that exact memory [ray-wide-mitigation].

[eviction]: The exact level of eviction (register file, L1, L2, ...) depends on (and gets worse
with) the size of the working set.

[ray-wide-mitigation]: The wide-over-rays approach mitigates the cost of the memory traffic by doing
more useful work for each cache line it had to load.

The elephant is algorithmic in nature. We are visiting each geometry for each ray bounce, but most
of those rays have no way to reach most geometries. There are too many wasted loads and calculations
per bounce. To get away from the O(m*n), we use acceleration structures to help with eliminating
impossible hits, such as Bounding Volume Hierachies (BVH), k-d Trees, or Octrees. Choosing between
them depends on the character of your data. We went with the BVH, because it doesn't have many
assumptions about its contents and degrades gracefully with bad quality of the input data
[bvh-generality].

[bvh-generality]: We didn't want to constrain the rest of what we are going to build by choosing an
overly picky datastructure. This also helped us productize the daylighting evaluator later.

(XXX: Explain BVH)

XXX: HERE Describe BVH raytracer: bounding boxes and triangles.

However, as we naively enter the land of computer science, we seem to have lost our ability to
utilize modern hardware, and have to do some thinking to recover it.

# Raytracer Architecture, Main Course

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
