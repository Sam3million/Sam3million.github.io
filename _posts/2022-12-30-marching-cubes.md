---
layout: post
title: "Hyper Marching Cubes"
date: 2026-03-28
thumbnail: "/assets/aridsun/arid-sun-1.png"
---

This post describes my multi-year long journey with procedural terrain generation and the Marching Cubes algorithm.
I describe some techniques I used to hyper optimize the algorithm, including SIMD, multithreading, and the Burst compiler.
I also describe some novel techniques I came up with to further optimize it and allow for immediate triangulation and unique vertex output.
I'm able to mesh a 32x32x32 volume in around 0.05 ms without duplicate vertices and with angle-weighted per-vertex normals. This runs in
parallel on the CPU so many chunks can be meshed at the same time. For example, meshing 8 chunks takes about 0.2 ms 

# Background

In 2022, I had an idea for a "capture the flag" type game, where two teams would start out on different islands separated by water.
The goal was to get to the other side, capture the opposing team's flag, and bring it back to your side.
In thinking about the different ways to get to the other team's island, one method seemed particularly interesting to me: tunneling.
What if players could sneak over to the other side through a secret tunnel they dug?
To do this, however, players would need a way to deform the terrain at runtime. Enter marching cubes.

The marching cubes algorithm generates a mesh for a 3d noise volume. In technical terms, it reconstructs the isosurface.
Terrain can be modified by altering the underlying noise volume, then remeshing using the algorithm.
I had seen Sebastian Lague's video on Marching Cubes so I was at least somewhat familiar with the concept.

After working with the algorithm and terrain generation, I was inspired to instead create an open-world survival game.
I've been working on the survival game for a few years now on and off. In that time, I feel confident in saying
I've mastered the algorithm and perhaps even developed new techniques to speed it up.

This post retraces my journey with terrain generation and Marching Cubes to the point where I am at now.

# Noise Volumes Explained

A noise volume is essentially a 3D volume of numbers. At each point in space, the noise value varies according
to some function. We can divide the space into two regions--the region where the function is less than a certain value,
and the region where the function is greater than (or equal to) a certain value. The interface between these two regions
defines a surface. This value can thus be described as a *surfaceLevel*. A common choice for the surfaceLevel is 0.
You can think of all noise value points that are greater than or equal to 0 as being "solid" and all noise value points
less than 0 as being "empty/air."

# Marching Cubes Explained

The marching cubes algorithm reconstructs the surface between these two regions.

# Populating the Noise Volume - Part 1

The noise volume can be very easily populated with 3D simplex noise. Just dispatch a compute shader for every
volume grid point, sample the simplex noise function at that grid, and write it to an output buffer. Getting good
looking terrain can take some more work though. More on that later.

# CPU Based Meshing - Part 1

To first gain a better understanding of how the algorithm worked, I implemented a naive CPU-based version.
This was very slow but it functioned and allowed me to fully grasp the algorithm.

# GPU Based Meshing

In Sebastian Lague's video, he used a compute shader to generate the mesh. This gave him speedups, so I thought
it was the logical next step. I implemented it, and it was faster than my terrible CPU implementation.
However, it was still sort of slow, as getting the mesh data back from the GPU had a large delay (and was just annoying).

If we were using a fully GPU driven terrain rendering approach, GPU meshing might have more merit. But if the CPU for any reason
needs the data (e.g. for Physics, Networking), I realized that it's better to just do it on the CPU.

# Burst Marching Cubes

So now it was time to make an actually good implementation of the algorithm. My single-threaded, naive version from before
wouldn't cut it. It was time for multithreading.

As this project was made in Unity, the best way to multithread it was to use Unity's job system. This allows for
workloads to be distributed to multiple threads and thus reduce the load on any one thread. Unity's Burst compiler
can also be leveraged with the job system for extreme optimization of code.

# Hyper Marching Cubes

This section describes the current state of my implementation, with all the tricks I've come up with.
I feel that I've changed the core algorithm enough that I get to call it something different.
So, I am giving it the name "Hyper Marching Cubes".
The crucial differences between the Hyper Marching Cubes algorithm and the standard version are that we can directly output unique vertices
and we can perform triangulation at the same time.

# First Discovery - Unique Vertices

In implementations of marching cubes that don't produce duplicate vertices, people often use some sort of Dictionary/HashMap as a filter.
I never liked this approach because you have to hash floating point values, and HashMaps are also clunky to use in parallel. I wanted a way to
produce unique vertices directly without any hashmaps.

The reason marching cubes produces duplicate vertices is that adjacent cubes share certain edges. So, what if we had some kind of filter
that only lets us take vertices that no other cube has taken? Let's investigate. Imagine every cube only outputs vertices generated on
its bottom-leftmost edges.

![Each edge can be assigned a unique index](/assets/marching-cubes/single-cube-edges.svg)
*Each edge can be assigned a unique index*

Then there are no duplicates produced. However, we are missing some edges on the front, right, and top faces. That is because
there are no other cubes that will output those edges. So, depending on which face the cube is on, we output certain extra edges.

## Second Discovery - Unique Edge Index

In standard implementations of marching cubes, you first generate "Vertex Groups," which are vertex triplets
that represent a triangle. After all vertex groups have been generated, you can filter out duplicate vertices,
then generate the triangle indices. I wanted to find a way to generate the triangle indices immediately alongside the vertices.

First, what is the triangle index data actually saying? Well, it's telling the GPU to connect certain vertices.
For example, vertices 0, 3, and 8 form a triangle.

put image here 

As I stated before, we can't generate the triangulation right away because we don't know what index in the output vertex array
a given vertex will end up. But I thought that there must be a different way of representing the triangulation that didn't rely on this.
After thinking about this problem for a couple of weeks, I realized the solution: edges. Rather than outputting
triangle data as connections between vertices, I could output them as connections between edges in the volume! Then, in a post-processing
stage, once the vertex count was known, I could remap them to the actual indices. To make this work, I would need
a way to assign a unique number to each edge in the volume. Luckily, this problem is solvable.


![Each edge can be assigned a unique index](/assets/marching-cubes/single-cube-edges.svg)
*Each edge can be assigned a unique index*

For a 2x2x2 volume (1x1x1 cube), we can number the edges 0-11 easily. I did this manually up to 4x4x4 to get an idea of the pattern.
Now how do we do this programmatically?

![Each edge can be assigned a 3d position](/assets/marching-cubes/position-labeled-edges.svg){: width="100px" }
*Each edge can be assigned a 3d position*

First we assign each edge a spacial position since this can be done easily for every cube in the volume.
This position encodes the same information as the index, so there should exist some function f(x,y,z) that takes in the edge position 
and outputs the index. I derived the following, which essentially just counts the number of edges before this one:

```c#
/// <summary>
/// Each edge in an (R.x-1) * (R.y-1) * (R.z-1) volume of unit cubes can be assigned a unique index.
/// Samuel Rose 2025.
/// </summary>
/// <param name="index3D">The 3d position of the edge, where each edge in a single unit cube has xyz coords ranging from 0-2.</param>
/// <param name="resolution">R, the dimensions of the volume of points.</param>
/// <returns>If R.x=R.y=R.z, returns an index in the range [ 0, 3*R^2*(R-1) )</returns>
public static int UniqueEdgeIndex(int3 index3D, int3 resolution)
{
    return (index3D.x >> 1) + 
        resolution.x * (index3D.z >> 1) + 
        (resolution.x - 1) * ((index3D.z + 1) >> 1) * (1 - (index3D.y & 1)) + 
        resolution.x * resolution.z * (index3D.y >> 1) + 
        (resolution.x * (resolution.z - 1) + resolution.z * (resolution.x - 1)) * ((index3D.y + 1) >> 1);
}
```

Setting `resolution.x = resolution.y = resolution.z = R` and plugging in the max edge position of `((R-1)*2-1, (R-1)*2, (R-1)*2)`, we can see that the maximum value
produced by the function is `3 * R^2 * (R-1) - 1`. Adding 1, this polynomial outputs the sequence 12, 54, 144, 300, 540, ...
for R >= 2. Inputting this into the online encyclopedia of integer sequences, we see <a href="https://oeis.org/A059986">A059986</a>,
which confirms that the original function is correct. The actual meaning of this is that each volume can produce at most
`3 * R^2 * (R-1)` unique vertices. I save this as a variable `MaxUniqueVertices`.

Now, instead of outputting a vertex group, we can instead output an int3 with the three edges to connect.
We'll need 2 post-processing stages in which we remap edge to index.
To do this, we must allocate an int array of size `MaxUniqueVertices`. We'll run the first post-processing stage
over the number of unique vertices output by marching cubes. We then write the current index to this int array at the index
of this vertex's unique edge index. 

```c#
public void Execute(int index)
{
    // Assign each vertex a unique index in the range 0 to vertexCount-1:
    int vertexEdgeIndex = uniqueEdgeIndices[index];
    indexByEdge[vertexEdgeIndex] = index;
}
```

The second stage runs for each triangle, and reads in the indices we stored in the first stage.

```c#
public void Execute(int index)
{
    // Remap the triangles to use the new vertex indices:
    int3 vertexGroup = vertexGroups[index];
    triangles[index] = new int3(indexByEdge[vertexGroup.z], indexByEdge[vertexGroup.y], indexByEdge[vertexGroup.x]);
}
```

It might be hard to understand, so here's a visual breakdown of what's going on: