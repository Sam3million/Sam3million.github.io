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

# Main idea

<!-- For simplicity, for most of this article, I am talking about the sun as light source: parallel light rays and  -->
Our goal is to determine how much a particular fragment is shadowed.
For now, let's focus on the sun: a sphere light source with parallel light rays in our scene.
Imagine sitting at the fragment and looking at the sun (with protective glasses, of course!).
You would see a disk, which we will assume has the same brightness at every point.

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