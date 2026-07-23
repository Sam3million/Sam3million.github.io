---
layout: post
title: "Hyper Marching Cubes"
date: 2026-03-28
modified: 2026-07-22
thumbnail: "/assets/marching-cubes/thumbnail.png"
category: gamedev
---

This post describes my variation on the Marching Cubes algorithm, which I call "Hyper Marching Cubes".
It allows for unique vertex output and faster triangulation. It doesn't use any Dictionary/HashMap
collections, making it well suited for both multithreaded CPU and compute shader GPU implementations. My SIMD CPU implementation
is able to mesh a 128x128x128 volume in 1.33 ms without duplicate vertices and with per-vertex normals.
My compute shader implementation is able to mesh the same volume in 0.88 ms. This does come at the cost of some memory.

<div align="center">
    <video width="80%" autoplay muted controls loop playsinline>
    <source src="/assets/marching-cubes/hyper-marching-cubes.mp4" type="video/mp4">
    </video>
</div>

<div align="center">
    <video width="80%" autoplay muted controls loop playsinline>
    <source src="/assets/marching-cubes/in-game-demo.mp4" type="video/mp4">
    </video>
</div>

<div class="tldr" markdown="1">
# 🔑 Key takeaways
- Variation of Marching Cubes algorithm that produces unique vertices and triangulation without Dictionary/HashMap
- Can be implemented easily on both the CPU and GPU
- CPU implementation meshes 128^3 volume in 1.33 ms
- GPU implementation meshes 128^3 volume in 0.88 ms
- Slightly higher memory cost compared to standard Marching Cubes
</div>

# Background

The marching cubes algorithm generates a mesh for a 3d noise volume. In technical terms, it reconstructs the isosurface.
Due to its capacity to produce arbitrary meshes, it is often used for procedurally generated terrain. This was the reason I first
became interested in the algorithm. I make heavy use it in my open world survival game. Terrain can be modified by altering the
underlying noise volume, then remeshing using the algorithm. For a great overview of the algorthm,
see <a href="https://www.youtube.com/watch?v=vTMEdHcKgM4">Sebastian Lague's Terraforming Video</a>.

# Hyper Marching Cubes

The crucial difference between the Hyper Marching Cubes algorithm and the standard version is that we can perform triangulation
at the same time as we output vertices, with the addition of some very cheap post-processing steps.

## Unique Edge Index

The key to this algorithm is the assignment of a unique index to each edge in the noise volume. We can then represent triangles
as connections between different edges rather than connections between vertices. Then, in a post-processing stage, once the
vertex count is known, we can remap to the actual vertex indices. To make this work, we need a way to assign a unique
number to each edge in the volume. For a 2x2x2 volume (1x1x1 cube), we can number the edges 0-11 easily. But how can we do it programmatically
for larger volumes?

![Each edge can be assigned a unique index](/assets/marching-cubes/single-cube-edges.svg)
*Each edge can be assigned a unique index. This is easy for the base case.*

![Each edge can be assigned a 3d position](/assets/marching-cubes/position-labeled-edges.svg)
*Each edge can be assigned a 3d position*

First we assign each edge a spacial position since this can be done easily for every cube in the volume.
This position encodes the same information as the index, so there should exist some function f(x,y,z) that takes in the edge position 
and outputs the index. I derived the following, which essentially just counts the number of edges before this one:

```c#
// Samuel Rose 2025.
// Each edge in an (R.x-1) * (R.y-1) * (R.z-1) volume of unit cubes can be assigned a unique index.
// index3D: The 3d position of the edge, where each edge in a single unit cube has xyz coords ranging from 0-2.
// resolution: R, the dimensions of the noise volume.
/// If R.x=R.y=R.z, returns an index in the range [0, 3*R^2*(R-1))
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
which confirms that the original function is correct.

Now we are able to represent triangles as connections between edges. We'll need 2 post-processing stages in which we remap these back to vertex indices.
The first stage assigns each vertex an index ranging from 0 to `vertexCount-1`, where vertexCount is the number of unique vertices created by marching cubes.
This is written as an IJobParallelFor in Unity.

```c#
// Outside of the job, allocate array
int MaxVertices = 3 * ChunkResolution * ChunkResolution * (ChunkResolution - 1);
NativeArray<int> indexByEdge = new (MaxVertices, Allocator.Persistent);
...
public void Execute(int index)
{
    // Assign each vertex a unique index in the range 0 to vertexCount-1:
    int volumeEdgeNumber = vertexEdges[index];
    indexByEdge[volumeEdgeNumber] = index;
}
```

The second stage runs for each triangle, and remaps the indices we stored in the first stage.

```c#
public void Execute(int index)
{
    // Remap the triangles to use the new vertex indices:
    int3 triangle = triangles[index];
    triangles[index] = new int3(indexByEdge[triangle.z], indexByEdge[triangle.y], indexByEdge[triangle.x]);
}
```

It might be hard to understand, so here's a visual breakdown of what's going on:

![Edge Connections](/assets/marching-cubes/edge-connections.png)

Let's say the mesh outputs 9 vertices and 3 triangles. In addition to the vertex positions,
we also output the vertex edge numbers to a separate array.

![Meshing Output 1](/assets/marching-cubes/meshing-output-1.svg)

In the first post-processing stage, we map vertex edge number to its position in the vertices array by
using the edge as an index into an array.

![Meshing Output 2](/assets/marching-cubes/meshing-output-2.svg)

In the second post-processing stage, we remap the triangles to use these values. Now you can see
that first triangle correctly connects the vertices at indices 8, 2, and 4.

![Meshing Output 3](/assets/marching-cubes/meshing-output-3.svg)

What we did was essentially replace a Dictionary/Hashmap with an array by having a perfect hashing function.

<!--
## Fast Per-Vertex Normals

This edge indexing technique also allows us to generate fast per-vertex normals. 

Since each vertex can only be used by 4 cubes, we allocate a float3 array of length `MaxVertices * 4`.
Each bucket of 4 float3s within this array describes the triangle normals that could possibly affect a
given vertex. We can simply use `edgeIndex % 4` to get the proper slot within this bucket, where edgeIndex
is the 0-11 index of an edge within a cube.

![4 Cubes Per Vertex](/assets/marching-cubes/normal-cubes.svg)
*Each vertex is only affected by 4 cubes*

When we calculate the triangles during meshing, we keep track of the per-vertex normal sums within this cube.
Then after processing all of its triangles, we write the normal contributions to the correct slot.
```c#
Span<float3> cubeNormalSums = stackalloc float3[12];
cubeNormalSums.Clear();
...
// Process vertices here, storing in vertexCache
...
// Process triangles here
for (int j = 0; j < indexCount; j+=3)
{
    int edge0 = remappedTriangulation[config * 16 + j];
    int edge1 = remappedTriangulation[config * 16 + j + 1];
    int edge2 = remappedTriangulation[config * 16 + j + 2];
    
    float3 vertexA = vertexCache[edge0];
    float3 vertexB = vertexCache[edge1];
    float3 vertexC = vertexCache[edge2];
    
    float3 ab = vertexB - vertexC;
    float3 bc = vertexA - vertexC;
    float3 ca = vertexA - vertexC;
    float3 cross = math.cross(-ca, ab);
    float crossLength = math.length(cross);
    
    if (crossLength > 0.0001f)
    {
        float3 normal = cross / crossLength;
        
        float angleA = math.atan2(crossLength, math.dot(ab, -ca));
        float angleB = math.atan2(crossLength, math.dot(bc, -ab));
        float angleC = math.atan2(crossLength, math.dot(ca, -bc));
        
        cubeNormalSums[edge0] += normal * angleC;
        cubeNormalSums[edge1] += normal * angleB;
        cubeNormalSums[edge2] += normal * angleA
    }
    if (crossLength > 0)
    {
        int3 triangle = new int3(edgeNumberCache[edge0], edgeNumberCache[edge1], edgeNumberCache[edge2]); 
        triangles[reservedTriangleRangeStartIndex + totalTriangleCount] = triangle;
        totalTriangleCount++;
    }
}
// After we've processed all triangles, we write the normal contributions from this cube to the correct slot for each vertex
for (int j = 0; j < 12; j++)
{
    // Only write if we did use this vertex:
    if (usedVertex[j])
    {
        int cubeIndex = j & 3;
        int volumeEdgeNumber = (edgeNumberCache[j] + xyzBase.x) * 4;
        normalsSumPerVertex[volumeEdgeNumber + cubeIndex] = cubeNormalSums[j];
        cubeNormalSums[j] = 0;
    }
}
```

Finally, in a post-processing stage, we sum the 4 float3s for each vertex to get the final normal.
We can actually do this in the same stage as when we map vertex edge to index.
```c#
public void Execute(int index)
{
    // Assign each vertex a unique index in the range 0 to vertexCount-1:
    int volumeEdgeNumber = vertexEdges[index];
    indexByEdge[volumeEdgeNumber] = index;
    
    // Calculate normal:
    float3 sum = float3.zero;
    for (int i = 0; i < 4; i++)
    {
        sum.x += normalSumByVertexIndex[volumeEdgeNumber * 12 + 0 + i];
        sum.y += normalSumByVertexIndex[volumeEdgeNumber * 12 + 4 + i];
        sum.z += normalSumByVertexIndex[volumeEdgeNumber * 12 + 8 + i];
    }
    UnsafeUtility.MemClear(addr, 12 * 4);
    
    (uniqueVerticesListPtr + index)->Normal = math.normalizesafe(sum, new float3(0, 1, 0));
}
```
-->

## CPU Implementation

In my CPU implementation I use Unity's Job System, the Burst compiler, and SIMD to get the most out of the hardware.
I run an IJobParalellFor where each thread is a different Y-Z location on the left side of the noise volume.
I then use SIMD for almost every task, including loading noise values, calculating cube configurations, and generating vertices.
To get a compact vertex and index output without using atomics, I do a prefix sum compaction stage. Then I run the remapping and
triangulation post processing stages as described above. Per-vertex normals are calculated from the noise value deltas at vertex generation time.

## Compute Shader Implementation

All the code snippets shown were from my CPU implementation, but this algorithm lends itself well
to using compute shaders. This allows for a fully GPU-driven mesh generation and rendering approach.

So, I went ahead and did just that. My compute shader version uses 5 different kernels to mesh the volume and prepare it for rendering.
The remapping and triangle post-processing kernels are dispatched indirectly based on the output vertex and index counts. I only did basic
optimization, so it could probably be improved a lot more (e.g. using Wave Intrinsics). Still, the results are impressive.

![Compute Shader](/assets/marching-cubes/compute-shader.png)

# Results

For the CPU-based implementations I used the Unity profiler to measure average time to fully recreate and upload the mesh.
Note this doesn't include the time spent modifying the underlying noise volume or rendering the mesh. For my compute shader 
implementation I used the Xcode debugger to capture frame data and summed the time spent on the mesh generation kernels.
This is running on a 2021 Macbook Pro with 10 Core CPU, 16 Core GPU.

| Volume Resolution: | 32^3 | 64^3 | 128^3 |
| :--- | :---: | :---: | :--- |
| Naive Approach | 7.40 ms | 39.82 ms | 206.41 ms |
| Sebastian Lague | 10.40 ms | 23.61 ms | 256.39 ms |
| Hyper Marching Cubes (SIMD Multithreaded) | 0.19 ms | 0.42 ms | 1.33 ms |
| Hyper Marching Cubes (Compute Shader) | 0.038 ms | 0.17 ms | 0.88 ms |

## Memory Usage
This runtime speed does come at the cost of memory, since we have to allocate the vertexEdges and indexByVertex arrays.
However, these are relatively cheap additions to standard marching cubes implementations that already have large buffers
for storing vertices, indices, normals, etc.

| Volume Resolution: | 32^3 | 64^3 | 128^3 |
| :--- | :---: | :---: | :--- |
| VertexEdges | 0.38 MB  | 3.10 MB | 24.97 MB |
| IndexByVertex | 0.38 MB  | 3.10 MB | 24.97 MB |
| **Total Additional Memory Usage** | **0.76 MB**  | **6.20 MB** | **49.94 MB** |

## Usage In Game

Here's some footage of me using Hyper Marching Cubes to deform the terrain in my open-world survival game.

<video width="100%;" autoplay muted controls loop playsinline>
  <source src="/assets/marching-cubes/in-game-demo.mp4" type="video/mp4">
</video>

# Future Work

- AVX Implementation for x86 CPUs
- Per-Triangle Normals (flat shading)
- Produce Meshlets Instead of Full Mesh
- Reduce Memory Cost (maybe bit packing?)
- Use it for Simulations