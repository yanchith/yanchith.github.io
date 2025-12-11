This is an article about raytracing on the CPU as fast as possible. Because we are going to be
focusing on fast for most of the article, there will assumptions of raytracing, math and
programming. To not leave people behind, I'll try explaining some basics as we go, but to get deeper
understanding, I wholeheartedly recommend the Raytracing Weekend book series as a primer
(https://raytracing.github.io/). The second topic for which there will be assumptions is SIMD, which
I will not be explaining at all. Here, my long-form recommendation for the patient viewer is Casey
Muratori's Handmade Hero (https://guide.handmadehero.org/) and Computer Enhance
(https://computerenhance.com) series, which both touch on way more than just SIMD.

The article stems from work I did in early 2023. As of the time of writing (December 2025), that
work is the most focused technical work I have done. This was quite the luxury for me, as I usually
don't get to concentracte for weeks on doing single part of the system well, and instead have to
prioritize what is the most valuable thing for me to work on. This was a happy and rare exception.

Also note that I am a generalist, and don't consider myself deeply knowledgeable in computer
graphics. I am painfully aware that I could have missed an obvious technique that could have made
the results better. If after reading the article you think of something I did not explore
thoroughly, I'll be delighted to get your email. Now, before we get to the core of the problem,
here's the last bit of context.

In late 2022 and early 2023, a startup company I worked for was about to get funding, and we started
warming up the programming team, which meant building a small part of the system together first,
before fanning out out wide. A small but important part we knew we needed.

The goal of the company as percieved back then was to give real estate developers tools to
procedurally generate housing architecture. This housing was supposed to meet the developer's
criteria, such as yields, a correct mix of functions and apartment sizes, aesthetics, as well as
spatial inputs (e.g. build here but not there, build in this shape...). We assumed that to generate
that architecture, there would be a computer learning process. As it is with learning processes,
they need to get feedback on their results from one iteration that can be fed into the next,
gradually improving the results. Generating livable architecture is a very high-dimensional
problem. A house is definitely not just any "house shaped" geometry, so before we can enter the
realm of architecture (and maybe even good architecture?), there's structural engineering, business,
regulatory and legal criteria that a development project must satisfy. For such a tall order, we
assumed that even a smartly designed learning algorithm will have to do many iterations, getting
feedback from a myriad of evaluators (structural, accoustic, sunlight/daylight, business, legal)
until it reaches a solution.

(TODO(jt): Include GIF here?)

So we generate stuff, evaluate, generate new stuff, evaluate, and so on, until eventually we stop
after either all criteria are satisfied, no significant improvement can be found, or we exceeded our
computational budget. And because the problem is going to be hard, we need to use that budget well
to squeeze in as many iterations as possible. At the time we weren't quite sure whether we need to
be fast so that we have a shot at being quasi-realtime, or to at least finish the computation
overnight on a powerful hardware [1].

[1]: The state as of me leaving the company is that for small scenes it took seconds to get
something, and minutes to get something useful. This degraded to having to do overnight runs for
large and huge scenes. The problem is hard.

Our team warmup focused on one particularly computationally demanding part of the evaluation
process: the daylight evaluation. For the uninitiated, daylight evaluation is something that tells
you how much light accumulates on surfaces in the interior of a building for some stable lighting
conditions outside. This is then done for tens of thousands of surfaces inside many rooms and many
buildings in a city block, both in the buildings you plan to build, and in the already existing
buildings that surround them [regulations].

[regulations]: It turns out light is essential for the human wellbeing, so there is a minimal amount
of daylight a dwelling must receive defined by regulations.

The way daylight evaluation is usually implemented is raytracing [radiosity]. Interestingly enough,
all current software solutions (Ladybug: https://www.ladybug.tools/ladybug.html, Climate Studio:
https://www.solemma.com/climatestudio) internally depend on a raytracing package called Radiance
(https://github.com/LBNL-ETA/Radiance/tree/master). This will be important later.

[radiosity]: Although there are analytical ways used for simpler scenes that were used before the
wide adoption computers of in architecture studios.

To bridge this closer to the videogame audience, the daylighting evaluation is very similar to what
game developers call light baking: computing light simulation for static scenes and baking the
results into planar (lightmaps) or spatial (light probes) textures, so that we know how a scene is
lit without having to compute it at runtime. The differences between daylight evaluation and light
baking come down to how the results are used, not how they are computed. Compared to light baking,
daylight evaluation doesn't care about color, only intensity, nor does it need to produce
information about which direction the light is coming from for the light probes. Daylighting also
only needs to simulate perfectly diffuse (Lambertian) materials, making some parts of the raytracer
simpler. Additionally, there's a few peripheral bits about interpretting the results that are
specific to the AEC industry regulations that I am going to purposefully ignore, and instead focus
on raytracing, which I believe looks the same as it would in a computer graphics application.

# Ray Tracing

(XXX: Picture)

Ray tracing is a process of simulating light. For a point in space, for instance a pixel of a
digital camera, we want to determine the intensity and color of light that point receives. We do
that by simulating paths that light could take through the scene from a light source to reach our
point. Such paths can be either direct, or include one or more bounces from objects in the
scene. However, we are often interested in light reaching just a few select points in space, like
the chip of our digital camera. Compared to all the places the light can go, light rays have only a
miniscule chance of reaching the points we are interested in, directly or via bounces. We would have
to shoot a lot of rays from light sources for them to trickle down to our measurement points in
sufficient quantities.

Instead, we lean on a laws of the universe regarding (the absence of) the arrow of time for the
behavior of particles. If we were shown a movie of elementary particles moving through space,
sometimes colliding with each other, we would have a hard time telling whether the movie is playing
forward or backward. This is because it is impossible to tell - they behave exactly the same whether
they are moving forward or backward in time, and it is only possible to discern the direction of
time once we have a large number of particles and probability enters the picture, pushing particles
towards states with high entropy [probability]. For us, this means that if we have a path between a
measurement point and a light source, a photon could have taken that exact path both in both
directions [oversimplificatoin].

[probability]: The fundamental laws for particles do not change when there's many of them. The
probabilistic behavior emerges from our inability to keep track large data, leading us to reason
about macrostates and entropy instead.

[oversimplification]: This is an oversimplification on many levels. Photons do not necessarily
bounce of off all surfaces. Sometimes they are absorbed, and a new photon is emitted. However, there
is a grain of truth in this model, and imagining photons as "zillions of billiard balls" bouncing
around doesn't disrupt our high level simulation.

The implication of bidirectionality for our simulation is that we can trace rays in reverse: from
the relatively few points we are interested in towards light sources. For infinite number of rays
the results would have been the same, but with our finite limitations, we have higher chance of
sucessfully completing the path between the light source and the camera going in backwards
[big-lights]. With both forward and reverse raytracing, we keep track of how much energy we loose
with each bounce. Because these energy losses (also called attenuation) multiply, and multiplication
is commutative, this works out the same regardless of the direction we trace the ray in.

[big-lights]: Because light sources are usually bigger than our camera, and even if the ray escapes
the scene, we can sample a background skybox to get some ambient value of light.

So we trace rays from our measurement points, hoping they eventually reach sources of light. These
rays go in a straight line until they hit something. Depending on what was hit, various things
happen. If the ray hits a light reflecting surface, it bounces off of the it back into the
scene. The new direction of the ray depends on the physical properties of the surface. Some surfaces
reflect the ray as a mirror would, others deflect the ray in a random direction, and there are
usually more complicated behaviors for realistic materials. This is described by the Bidirectional
Reflectance Distribution Function (BRDF). A BRDF for a real material can be quite complex
[anisotropy], and cannot always be defined by a simple mathematical formula. For many realistic
materials, the BRDF is actually a lookup table [merl]

[anisotropy]: For instance, the BRDF for animal fur behaves differently for various orientations of
the incoming and outgoing rays relative to the surface.

[merl]: For instance, see the MERL library of materials:
https://www.merl.com/research/downloads/BRDF

If the ray hits a light source, it ends its journey. We read the light's value and apply the
attenuation the ray has accumulated when bouncing off of materials along its path. Because each
bounce attenuates the ray, we can decide to terminate the ray early after a certain amount of
bounces, as even if it did reach a light source eventually, the light's contribution through that
particular ray would have been close to zero.

Hopefully multiple rays shot from our camera reach light sources. We compute the light value at the
camera's pixel as the sum of the individual ray contributions divided by the number of cast
rays. Because scenes are not always well lit, it may take a lot of rays for us to form a coherent
picture of what the scene looks like from the perspective of the camera. Low light scenes are prone
to noise, when various neighboring pixels in the camera collect dramatically different values of
light. As the number of rays increases to infinity, the noise becomes weaker, but because a large
number of rays is not always practical, we often employ denoising algorithms that reconstruct
information from noisy pictures.

# Raytracer Architecture, Appetizer

At a high level, a raytracer operates on a geometric description of the scene and a list of ray
tracing tasks it needs to compute on that scene. These do not necessarilly change at the same rate
from frame to frame, and many real-time raytracers reason about that, but we won't.

The simplest and most flexible way to represent the scene is to have arrays of various geometric
primitives: triangles, spheres, planes, boxes... Raytracing the scene is about going over each ray
and testing [intersection] it against all of these arrays, remembering information about the closest
hit, so that we know where to start our next bounce. In fact, this is so simple that almost all of
it fits in pseudocode.

[intersection]: Testing rays against geometry means solving an equation for the two parametric
geometries, e.g. ray and triangle. The solve provides us with both the distance to the intersection
point and the normal of the intersected surface, both of which we need to procede.

```
Sphere :: struct {
    position: Vec3;
    radius:   float32;
}

Plane :: struct {
    normal: Vec3;
    d: float32;
}

AABox :: struct {
    min: Vec3;
    max: Vec3;
}

Material_Type :: enum u32 {
    LIGHT_SOURCE :: 0;
    LAMBERTIAN   :: 1;
    // ...
}

Material :: struct {
    type: Material_Type;
    p0: float32;
    p1: float32;
    p2: float32;

    #overlay (p0) color: Vec3;
}

Scene :: struct {
    spheres:          [..] Sphere;
    sphere_materials: [..] Material;

    planes:          [..] Plane;
    plane_materials: [..] Material;

    aaboxes:         [..] AABox;
    aabox_materials: [..] Material;
}

raytrace :: (scene: Scene, primary_ray_origin: Vec3, primary_ray_direction: Vec3, max_bounces: s64) -> Vec3 {
    ray_origin    := primary_ray_origin;
    ray_direction := primary_ray_direction;
    ray_color     := Vec3.{ 1, 1, 1 };

    for 0..max_bounces - 1 {
        hit_distance: float32 = FLOAT32_MAX;
        hit_normal:   Vec3;
        hit_material: Material;

        for sphere: scene.spheres {
            hit, distance, normal := ray_sphere_intersection(ray_origin, ray_direction, sphere);
            if hit && distance < hit_distance {
                hit_distance = distance;
                hit_normal   = normal;
                hit_material = sphere_materials[it_index];
            }
        }

        for plane: scene.planes {
            hit, distance, normal := ray_plane_intersection(ray_origin, ray_direction, plane);
            if hit && distance < hit_distance {
                hit_distance = distance;
                hit_normal   = normal;
                hit_material = plane_materials[it_index];
            }
        }

        for aabox: scene.aaboxes {
            hit, distance, normal := ray_aabox_intersection(ray_origin, ray_direction, aabox);
            if hit && distance < hit_distance {
                hit_distance = distance;
                hit_normal   = normal;
                hit_material = aabox_materials[it_index];
            }
        }

        if hit_distance == FLOAT32_MAX {
           // The ray didn't hit anything. Multiply by background color. We could sample a skybox instead...
           ray_color *= .{ 0.2, 0.1, 0.4 };
           return ray_color;
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

// We'll define some of these later.
ray_sphere_intersection :: (ray_origin: Vec3, ray_direction: Vec3, sphere: Sphere) -> hit: bool, hit_distance: float32, hit_normal: Vec3;
ray_plane_intersection  :: (ray_origin: Vec3, ray_direction: Vec3, plane: Plane) -> hit: bool, hit_distance: float32, hit_normal: Vec3;
ray_aabox_intersection  :: (ray_origin: Vec3, ray_direction: Vec3, aabox: AABox) -> hit: bool, hit_distance: float32, hit_normal: Vec3;

// We won't really be defining these later, but let's pretend they exist.
attenuate :: (color: Vec3, direction: Vec3, normal: Vec3, material: Material);
bounce    :: (ray_origin: Vec3, ray_direction: Vec3, hit_distance: float32, hit_normal: Vec3, hit_material: Material);
```

Before addressing the elephant in the room and moving on from this approach, I want to mention some
of its benefits.

Firstly, there is no need to build acceleration structures - you already have the arrays of objects,
or you can cheaply produce them, in case your representation does not already match what the
raytracer needs. This is not going to be true for the more sophisticated designs, where a part of
the raytracing cost will be spent on organizing data to accelerate raytracing.

Another virtue of the array approach is that the resulting code is very straightforward to
SIMD. Going wide can happen either over rays or geometries, both of which have their respective
strengths and weaknessess. When going wide over rays, we get a lot of value for each geometry we
load, as we test it against 4/8/16 rays at a time. Problems arise with rays diverging, some rays
hitting the end of their journey after less bounces than others, meaning we end up with finished
rays in our wide registers, and either have to do something about that [hot-swapping-rays], or
accept the wasted work. Going wide over rays was the route taken by Casey Muratori in his Handmade
Ray educational miniseries: https://guide.handmadehero.org/ray/. Unlike paralellizing on rays, going
wide over geometry doesn't ameliorate the cost of memory trafic to the CPU. Each geometric primitive
is loaded to be tested against a ray, evicted by subsequent loads [eviction], only to be loaded
again when the next ray is going to need that exact same memory. It still is an improvement over the
non-SIMD version, as we at least compute more intersections at a time, and it does not need to care
about dead rays.

[hot-swapping-rays]: After we have made a pass over all geometry and we know which rays hit what, we
could write out the results for finished rays and load in new ones into our wide registers. This
would incur some management and complexity overhead, but would also mean we utilize SIMD to the
fullest right until the very end where we run out of rays to swap in.

[eviction]: The exact level of eviction (register file, L1, L2, ...) gets worse with the size of the
working set.

Now back to the elephant, which is algorithmic in nature. We are visiting each geometry for each ray
bounce, but most of those rays have no way to reach most geometries. This incurs many wasted loads
and calculations per bounce. To get away from the O(m*n), we use acceleration structures to help
with eliminating impossible hits, such as Bounding Volume Hierachies (BVH) or Octrees. Choosing a
datastructure depends on the character of your data. We went with the BVH, because it doesn't have
many assumptions and degrades gracefully with bad quality of input data [bvh-generality].

[bvh-generality]: We didn't want to constrain the rest of what we are going to build by choosing an
overly picky datastructure. This also helped us productize the daylighting evaluator later.

The BVH is a tree where each node has bounding volume geometrically containing the node's
contents. This bounding volume is usually a parametric shape, e.g. an axis-aligned box or a
sphere. The node's contents are either links to child nodes, or in case of leaf nodes actual scene
geometries. Note that a node's bounding volume only has a relation to the node's contents, but there
is no direct relation between the bounding volumes of two sibling tree nodes. Two sibling nodes can
very well have overlapping bounding volumes, both of which are contained by the parent's bounding
volume. This means that traversing a BVH can take multiple paths at the same time, as they aren't
necessarilly mutually exclusive.

(XXX: Picture)

Ray tracing against a BVH boils down to a tree traversal. We test a ray against a node's bounding
volume first, and if that intersects, we test against its contents. Once we get to a leaf node, we
test our ray against the node's geometry, keeping track of the closest hit distance and normal, as
we would in unaccelerated array approach above. Once we know the closest hit distance, it can help
us eliminate entire branches of the tree, because we can skip over nodes not just when we miss their
bounding volume, but also if the distance to that bounding volume is greater than the recorded
closest hit distance. For this reason (and others), it is worthwhile to traverse the tree depth
first, so that we record our closest hit as soon as possible, eliminating further tests down the
line.

Building a BVH is a little more complicated than using it. I mentioned previously that in a BVH, a
node's bounding volume contains the objects of that node, including child nodes, if any, but
otherwise has no relation to bounding volumes of sibling nodes. This makes it a little easier to
build a BVH. For a good BVH, we'd also like to minimize the volume each node takes, and make
bounding volumes of sibling nodes overlap less. We'll work on building a good BVH later. Starting
with an array of geometries (say triangles), we organize them into a tree, such that the bounding
volume of a node (say axis-aligned bounding box) contains all down-tree triangles and bounding
boxes.

We start with a single BVH node that contains all triangles, and has a bounding box that contains
all these triangles. We build the tree by recursively splitting the node:

- Pick a direction
- Sort triangles along the picked direction
- Split the array in the middle, giving each part to a newly created BVH node
- Link the created BVH nodes to the parent
- Recurse to both child nodes

We stop recursing, if the number of triangles in our current node is below a threshold, say 8.

(XXX: bvh build psudocode)

```
Triangle :: struct {
    v0: Vec3;
    v1: Vec3;
    V2: Vec3;
}

AABox :: struct {
    min: Vec3;
    max: Vec3;
}

Node_Type :: enum {
    UNINITIALIZED :: 0;

    NODE :: 1;
    LEAF :: 2;
}

Node :: struct {
    left_type:  Node_Type; // If .UNINITIALED, index is invalid.
    left_index: s64;
    right_type: Node_Type; // If .UNINITIALED, index is invalid.
    right_index: s64;
}

Leaf :: struct {
    triangle_count: s64;
    triangles: [LEAF_TRIANGLE_COUNT] Triangle;
}

BVH :: struct {
    nodes:  [..] Node;
    leaves: [..] Leaf;
}

build_bvh :: (triangles: [] Triangle) -> BVH {

}

```


Now let's go over our initial, naive implementation. The raytracer's bounding volumes were
axis-aligned boxes, and scene geometries were triangles.


The intersection math for the boxes was the "slab" method.

(XXX: Pseudocode, https://tavianator.com/2022/ray_box_boundary.html)

Our initial prototype did triangle intersections as plane intersections combined three containment
tests (for each triangle arm), but we eventually moved to Möller–Trumbore, which produces the same
answer faster by having less instructions and shorter dependency chains.

(XXX: Pseudocode)

Now our speed scales logarithmically with the size of the scene. However, as we naively entered
the land of computer science, we have temporarily lost our ability to utilize modern hardware, and have
to do some thinking to recover it.

# Raytracer Architecture, Main Course

- Wide BVH: the array of box packs and the array of triangle packs
- Wide BVH also means less pointers, so it is smaller.
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
- what about metropolis, monte carlo estimation, etc...

# Conclusion

- mention that this is 1000x faster than Climate Studio and Ladybug (for projects that Ladybug can even load)
- I might re-implement and opensource the generally useful parts
