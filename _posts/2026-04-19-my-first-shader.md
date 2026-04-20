---
layout: post
title: "Unity Grass Shader"
date: 2026-04-19
---

In this post I describe the GPU-driven grass rendering technique I implemented for my ongoing survival game project.
Grass blades are generated, culled, compacted, rendered, and animated all on the GPU using compute shaders and indirect instanced draw calls.
The results look great, with immersive wind sway and realistic height, width, and bend variation.
The performance is great too, with my M1 Macbook Pro (10 Core CPU, 16 Core GPU) achieving 144+ FPS at 1080p in a fully grass covered scene.
Memory usage is relatively cheap, as I make use of bit arrays, indexing, and data packing. More on that later.

<video width="100%;" muted controls loop playsinline>
  <source src="/assets/grass/grass.mp4" type="video/mp4">
</video>

<div class="tldr" markdown="1">
# 🔑 Key takeaways
- Grass blades are generated per terrain chunk in a compute shader.
- 4 types of culling (Chunk-Frustum, Blade-Frustum, Distance-Stochastic, and Occlusion) minimize the number of blades we need to render
- GPU Scan-And-Compact algorithm with Wave Intrinsics is used to compactify the blades into an output buffer
- Grass geometry generated in vertex shader with cubic bezier curve, animated with noise texture, and rendered indirectly.
- 144+ FPS at 1080p with ~100,000 blades (culled down from ~1,000,000)
- Can be easily extended to include different models (e.g. flowers, bushes)
</div>

<br />
*Context*:
This was created as part of my ongoing open-world survival game project. Things like chunked grass blade buffer management were implemented but
are not discussed in depth here. 

![My survival game project](/assets/grass/landscape.png)
*My survival game project*
<br />

# Introduction

<!-- For simplicity, for most of this article, I am talking about the sun as light source: parallel light rays and  -->
Grass rendering is hot topic in computer graphics. There are many different techniques, but the most widely used approach is
alpha-masked quads. This gives decent and cheap results, but loses out on realistic blade deformation and wind sway.

<!--Put image of grass quad rendering here -->

Another technique is to individually render each blade. This allows for each blade to have real geometry and actually be there in the world.
Games like Ghost of Tsushima go all in on this approach, and the results speak for themselves.

<!--Put image of GoT grass rendering here -->

So why don't we see it more often in games? Well, unless it is implemented very carefully, the performance cost is massive.
In this post I'm going to describe my (mostly careful) implementation of per-blade grass rendering.

# Main Idea
To render a field of grass that looks full enough, we need hundreds of thousands to millions of blades.
We could place a million blade objects in our scene, press play, and call it a day, but that would cause most computers to explode.
To make this even remotely possible, we need to use a technique known as GPU Instancing. This technique allows us to draw
many copies of a mesh at different position offsets with a single draw call. This alone still isn't enough to make it performant, though.
To take it to the next level, we need Indirect Instancing. What sets indirect instancing apart is that the number of instances is not 
known by the CPU when it makes the draw call. That information lives completely in buffers on the GPU. What makes this so powerful
is that we can perform culling on the grass blades in a compute shader prior to drawing them. That way, we're not wasting time drawing
blades that aren't even on screen. We can also do things like separate the blades into LODs to further increase performance.


# Blade Generation
First off, we need to generate a buffer with grass blade positions. It should live on the GPU for later culling and rendering,
so populating it with compute shader seems fitting. We want our grass blades to be positioned exactly on top of the terrain, so 
we're going to need the terrain mesh data. We can bind each mesh's vertex and index buffers to our compute shader for access.
The approach I used is as follows:
- For each mesh triangle, dispatch `triangleSampleFactor` threads, each trying to produce a blade
- For every thread
    - Generate a random hash
    - Using that hash, generate random barycentric coordinates describing a position on the triangle
    - Create another hash from that position, and use it to generate a random float in the range 0-1
    - If the float is less than some predefined value, atomically write the blade to the buffer

I also added a weighting to the number of blades to spawn based on the area of the triangle, because tiny triangles would have 
the same number of blades as large ones, causing a non-uniform distribution across the mesh. Triangles with area 1 receive
about `triangleSampleFactor` blades, triangles with area 0.5 receive about half as many, and so on.
```cpp
  [numthreads(64,1,1)]
  void CSMain (uint3 id : SV_DispatchThreadID)
  {
    if(id.x >= triangleCount * triangleSampleFactor) return;
    uint triangleIndex = id.x / triangleSampleFactor;
    uint bladeIndex = id.x % triangleSampleFactor;
    int3 indexGroup = triangleBuffer[triangleIndex];
    vertdata vertex0 = vertexBuffer[indexGroup.x];
    vertdata vertex1 = vertexBuffer[indexGroup.y];
    vertdata vertex2 = vertexBuffer[indexGroup.z];
      
    float area = length(Normal(vertex0.vertex, vertex1.vertex, vertex2.vertex));
    if(bladeIndex > round(area * triangleSampleFactor)) return;
  ...
```

If a blade survives the position hash, we should generate some per blade properties before writing it out.
This includes things like height, width, bend, wind offset, and facing direction. I use a combination of
hashing and simplex noise for these properties.

```cpp
  // Generate random facing direction
  float3 u = normalize(float3(normal.y, -normal.x, 0));
  // This should be a unit vector:
  float3 v = cross(normal, u);
  float theta = Random(posSeed ^ 382173) * 2.0f * 3.141592653589793f;
  // This should also be a unit vector:
  float3 grassDirection = cos(theta) * u + sin(theta) * v;
  
  // Generate random height and width
  float height = max((SimplexNoise(pos * 0.1) * 0.5 + 0.5), 0.25);
  float width  = max(SimplexNoise(pos * 0.1 + float3(-602.4912, -998.21, 412.145)) * 0.5 + 0.5, 0.25);
  float bend   = SimplexNoise(pos * 0.1 + float3(591.12, -123.44, -123.321)) * 0.5 + 0.5;
  float wind   = SimplexNoise(pos * 0.1 + float3(223.981, -111.95, 223.45)) * 0.5 + 0.5;

  GrassInfo g;
  
  // Store properties and write
  ...
```



![](/assets/tssv/sun.svg)
*Left: unoccluded sun. Right: Sun partially occluded by scene geometry.*

Naturally, scene geometry could partially or fully block the circle, stopping some light from reaching the fragment.
We need to determine what fraction of that light source disk is blocked by scene geometry.
That directly gives us the shadow amount for that fragment.

- For each fragment:
    - `total_sun_occlusion := not occluded`
    - For each shadow-casting triangle:
        - Project onto that fragment's view of the sun disk
        - Determine occlusion and add to `total_sun_occlusion`
    - Return `total_sun_occlusion` as shadow amount

Of course, doing this naively and looping through all scene triangles for each fragment would be extremely slow.
To speed this up, I quickly cull large number of triangles in various stages so that each fragment only has to consider a small number of triangles.
The culling step is explained in [this chapter](#culling) below.