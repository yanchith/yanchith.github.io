This is an article about raytracing on the CPU as fast as possible. Because we are going to be
focusing on fast, the article will assume some knowledge of raytracing and the involved math and
programming. To not leave people behind, I'll try explaining some basics as we go, but to get deeper
understanding, I wholeheartedly recommend the Raytracing Weekend book series as a primer
(https://raytracing.github.io/). The second topic for which there are some knowledge assumptions is
SIMD. Here, my long-form recommendation for the patient is Casey Muratori's Handmade Hero
(https://guide.handmadehero.org/) and Computer Enhance (https://computerenhance.com) series, which
both touch on way more topics than just SIMD.

The article stems from work I did in early 2023. As of the time of writing (December 2025), that
work is the most focused technical work I have done. This is quite the luxury for me, as I usually
don't get to concentracte for three months on a single part of the system, and instead have to
prioritize what is the most valuable thing for me to work on. This was a happy and rare exception.

Also note that I am a generalist, and thus don't consider myself deeply knowledgeable in computer
graphics. I am painfully aware that I could have missed an obvious technique that could have made
the results better. If after reading the article you think of something I did not explore
thoroughly, I'll be delighted to get your email. Now, before we get to the core of the problem,
here's the last bit of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team, which basically meant building a small part of the system together
first before spinning out wide. A small but important part we knew we needed. The goal of the
company (as percieved back then) was to build software to procedurally generate housing
architecture. This housing was supposed to meet the developer's criteria, such as yields, a correct
mix of functions and apartment sizes, aesthetics, as well as spatial inputs (e.g. build here, but
not there). We assumed that to generate that architecture, there would be a computer learning
process, and as it is with learning processes, they need to get feedback on their results that can
be fed into the next iteration, gradually improving the results. Generating livable architecture is
a problem of very high dimensionality. A house is definitely not just any "house shaped" geometry,
so before we enter the realm of what is architecture (and what is good architecture), there's
structural engineering, business, regulatory and legal criteria that a development project must
satisfy. For such a tall order, we assumed that even a smartly designed learning algorithm will have
to do many iterations, getting feedback from a myriad of evaluators (structural, accoustic,
sunlight/daylight, business, legal) until it reaches a solution.

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
already existing buildings that surround them.

The way daylight evaluation is usually implememnted is raytracing [radiosity]. Interestingly
enough, all current solutions (Ladybug (XXX: link), Climate Studio (XXX: link)) internally depend on
just one piece of raytracing software called Radiance
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
specific to the AEC industry regulations that I am going to purposefully ignore, and instead focus
on raytracing, which I believe looks the same as it would in a light baker.

# Ray Tracing

(XXX: Picture)

Ray tracing is a process of simulating light. For a point in space, for instance a pixel of a
digital camera, we want to determine the amount and color of light that point receives. We do that
by simulating paths that light could take through the scene from a light source to reach our
point. Such paths can be either direct, or include one or more bounces from objects in the
scene. However, we are often interested in light reaching just a few select points in space, like
the chip of our digital camera. Compared to all the places the light can go, light rays have only a
miniscule chance of reaching the points we are interested in, directly or via bounces. We would have
to shoot a lot of rays from light sources for them to tricke down to our measurement points in
sufficient quantities.

Instead, we lean on a property of the universe regarding (the absence of) the arrow of time for
small number of particles. If we were shown a movie of elementary particles moving through space,
sometimes colliding with each other, we would have a hard time telling, whether the movie is playing
forward or backward [oversimplification]. This is because it is impossible to tell - they behave
exactly the same whether they are moving forward or backward in time, and it is only possible to
discern the direction of time once we have a large number of particles and probability enters the
picture, pushing particles towards states with high entropy [probability]. For us, this means that
if we have a path between a measurement point and a light source, a photon could have taken that
exact path both ways.

[oversimplification]: This is a huge oversimplification on many levels. For instance photons do not
necessarily bounce of off all surfaces. Sometimes they are absorbed, and a new photon is emitted,
etc. However, there is a grain of truth in this model.

[probability]: Particles actually behave the same even in large numbers. The probabilistic behavior
is emergent.

The implication of this bidirectionality for our simulation is that we can trace rays in reverse:
from the relatively few points we are interested in towards light sources. For infinite number of
rays the results would have been the same, but with our finite limitations, we have higher chance of
sucessfully completing the path between the light source and the camera going in reverse. With both
forward and reverse raytracing, we keep track of how much energy we loose with each bounce. Because
these energy losses (also called attenuation) are multiplicative, and multiplication is commutative,
this works out the same regardless of the direction we trace the ray in.

So we shoot rays from our measurement points, hoping they eventually reach sources of light. These
rays go in a straight line until they hit something. Depending on what was hit, various things
happen. If the ray hits a light reflecting surface, it bounces off of the it back into the
scene. The new direction of the ray depends on the physical properties of the surface. Some surfaces
reflect the ray as a mirror would, while others deflect the ray in a seemingly random
direction. This behavior is described by the Bidirectional Reflectance Distribution Function (BRDF),
and in general can be quite complex, for instance the BRDF for animal fur behaves differently for
various orientations of the incoming and outgoing rays. Because each bounce attenuates the ray, we
can decide to terminate the ray early after a certain amount of bounces, as even if it did reach a
light source eventually, the light's contribution through that particular ray would have been close
to zero. If the ray hits a light source, it ends its journey. We read the light's value and apply
the attenuation the ray has accumulated when bouncing off of materials along its path.

Hopefully multiple rays shot from our point reach light sources. The light value at the point is the
sum of the individual ray contributions divided by the number of cast rays. Because scenes are not
always well lit, it may take shooting a lot of rays for us to form a coherent picture of what the
scene looks like. Low light scenes are prone to noise, when various neighboring pixels in the camera
collect dramatically different values of light. As the number of rays increases to infinity, the
noise disappears, but because a large number of rays is not always practical, we often employ
denoising algorithms that reconstruct information from noisy pictures.

# Raytracer Architecture, Appetizer

At a high level, a raytracer operates on a geometric description of the scene and a list of ray
tracing tasks it needs to compute on that scene. Both the scene and the list of tasks can change
from frame to frame, the example of the first being objects moving around on the scene, and of the
second that the camera can move around, and we perhaps do not need all the raytracing information
for places the camera does not see.

The simplest and most flexible way to represent the scene is to have arrays of various geometric
primitives: triangles, spheres, planes, boxes, etc. Raytracing the scene is about going over each
ray and testing [intersection] it against all of these arrays, remembering information about the
closest hit, so that we know where to start our next bounce.

[intersection]: Testing rays against geometry means solving an equation for the two parametric
geometries (e.g. ray and triangle). The solve usually provides us with both the distance to
intersection point and the normal of the intersected surface, both of which we need to procede.

```
raytrace :: (scene: Scene, primary_ray_origin: Vec3, primary_ray_direction: Vec3, bounce_count: s64) -> Vec3 {
    ray_origin    := primary_ray_origin;
    ray_direction := primary_ray_direction;
    ray_color     := Vec3.{ 1, 1, 1 };

    for 0..bounce_count - 1 {
        hit_distance: f32 = FLOAT32_MAX;
        hit_normal:   Vec3;
        hit_material: Material;

        for sphere: scene.spheres {
            hit, distance, normal := ray_sphere_intersection(ray_origin, ray_direction, sphere);
            if hit && distance < hit_distance {
                hit_distance = distance;
                hit_normal   = normal;
                hit_material = sphere.material;
            }
        }

        for plane: scene.planes {
            hit, distance, normal := ray_plane_intersection(ray_origin, ray_direction, plane);
            if hit && distance < hit_distance {
                hit_distance = distance;
                hit_normal   = normal;
                hit_material = plane.material;
            }
        }

        for aabox: scene.aaboxes {
            hit, distance, normal := ray_aabox_intersection(ray_origin, ray_direction, aabox);
            if hit && distance < hit_distance {
                hit_distance = distance;
                hit_normal   = normal;
                hit_material = aabox.material;
            }
        }

        if hit_distance == FLOAT32_MAX {
           // The ray didn't hit anything. Return background color.
           return .{ 0, 0, 0 };
        }

        if hit_material.type == .LIGHT_SOURCE {
            ray_color *= hit_material.color;
            return ray_color;
        } else {
            ray_color = attenuate(ray_color, ray_direction, hit_normal, hit_material);
            ray_origin, ray_direction = bounce(ray_origin, ray_direction, hit_distance, hit_normal, hit_material);
        }
    }

    // We ran out of bounces and haven't hit a light.
    return .{ 0, 0, 0 };
}

```

Before addressing the elephant in the room and discarding this approach, I want to mention some of
its benefits.

Firstly, there is no need to build acceleration structures - you already have the arrays of objects,
or you can cheaply produce them, in case your representation does not exactly match what the
raytracer needs. This is not going to be true for the more sophisticated designs, where a part of
the raytracing cost will be spent on organizing data to accelerate raytracing.

Another virtue of the array approach is that the resulting code is very straightforward to
SIMD. Going wide can happen either over rays or geometries, both of which have their respective
strengths and weaknessess. When going wide over rays, we get a lot of value for each geometry we
load, as we test it against 4/8/16 rays at a time. Problems arise with rays diverging, some rays in
our SIMD registers potentially hitting the end of their journey after less bounces than others,
meaning we end up with finished rays in our wide registers, and either have to do something about
that [hot-swapping-rays], or accept the wasted work. Going wide over rays was the route taken by
Casey Muratori in his Handmade Ray educational miniseries:
https://guide.handmadehero.org/ray/. Unlike paralellizing on rays, going wide over geometry doesn't
ameliorate the cost of memory trafic to the CPU. Each geometric primitive is loaded to be tested
against a ray, evicted by subsequent loads [eviction], only to be loaded again when the next ray is
going to need that exact memory. It still is an improvement over the non-SIMD version, as we at
least compute more intersections at a time, and it does not need to care about dead rays.

[hot-swapping-rays]: After we have made a pass over all geometry and we know which rays hit what, we
could write out the results for finished rays and load in new ones into our wide registers. This
would incur some management and complexity overhead, but would also mean we utilize SIMD to the
fullest right until the very end where we run out of rays.

[eviction]: The exact level of eviction (register file, L1, L2, ...) depends on (and gets worse
with) the size of the working set.

Now back to the elephant, which is algorithmic in nature. We are visiting each geometry for each ray
bounce, but most of those rays have no way to reach most geometries. This incurs many wasted loads
and calculations per bounce. To get away from the O(m*n), we use acceleration structures to help
with eliminating impossible hits, such as Bounding Volume Hierachies (BVH) or Octrees. Choosing a
datastructure depends on the character of your data. We went with the BVH, because it doesn't have
many assumptions about its contents and degrades gracefully with bad quality of input data
[bvh-generality].

[bvh-generality]: We didn't want to constrain the rest of what we are going to build by choosing an
overly picky datastructure. This also helped us productize the daylighting evaluator later.

The BVH is a tree where each node has bounding volume geometrically containing the node's
contents. This bounding volume can be e.g. an axis-aligned box or a sphere. The node's contents are
either links to child nodes, or, in case of leaf nodes actual scene geometries. Note that a node's
bounding volume only has a relation to the node's contents, but there is no direct relation between
the bounding volumes of two sibling tree nodes. Two sibling nodes can very well have overlapping
bounding volumes, both of which are contained by the parent's bounding volume. This means that
traversing a BVH can take multiple paths at the same time, as they aren't mutually exclusive.

(XXX: Picture)

Ray tracing against a BVH boils down to a tree traversal. We test a ray against a node's bounding
volume first, and only after that hits we test against its contents. If we don't hit the bounding
volume, there is no point in continuing down this way. Once we get to a leaf node, we test our ray
against the node's geometry, keeping track of the closest hit distance and normal, as we would in
unaccelerated array approach above. Once we know it, the closest hit distance can help us eliminate
entire branches of the tree, because we can skip them not just when we miss the bounding volume, but
also if the distance to the bounding volume is greater than are recorded closest hit distance. For
this reason (and others), it is worthwhile to traverse the tree depth first, so that we record our
closest hit against geometry sooner.

Our raytracer's bounding volumes were axis-aligned boxes, and scene geometries were triangles.

(XXX: Explain basic BVH build)

The intersection math for the boxes was the "slab" method.

(XXX: Pseudocode)

Our initial prototype did triangle intersections as plane intersections combined three containment
tests (for each triangle arm), but we eventually moved to Möller–Trumbore, which produces the same
answer faster.

(XXX: Pseudocode)

Now our speed can scale logarithmically with the size of the scene. However, as we naively entered
the land of computer science, we seem to have lost our ability to utilize modern hardware, and have
to do some thinking to recover it.

# Raytracer Architecture, Main Course

- Wide BVH: the array of box packs and the array of triangle packs
- SIMD box intersection
- SIMD triangle intersection (maybe have to explain moller trumbore first?)

# Dessert

- Naive triangle intersection -> Moller Trumbore
- Pre-baked sphere rays.
- SAH optimization (Credit DH)
- targetting multiple SIMD backends within one binary: avx2, sse2, neon, fallback

# Future experiments

- Would avx512 help? 16-wide BVH means a lot of memory traffic. How many cache lines do we have to
  read on average for a sinle ray? And how many 512-bit loads?
- f16?
- Hiding indices in floats?
- Going wide over rays AND having a BVH... if we can do a smart gather.

# Conclusion

- mention that this is 1000x faster than Climate Studio and Ladybug (for projects that Ladybug can even load)
- I might re-implement and opensource the generally useful parts
