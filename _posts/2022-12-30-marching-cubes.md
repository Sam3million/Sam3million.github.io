---
layout: post
title: "Hyper Marching Cubes"
date: 2026-03-28
thumbnail: "/assets/aridsun/arid-sun-1.png"
category: gamedev
---

This post describes my journey with the Marching Cubes algorithm. I also describe my variation on the algorithm, which I call "Hyper Marching Cubes".
This version allows for unique vertex output, faster triangulation, and faster per-vertex normal calculation. It doesn't use any Dictionary/HashMap
collections, making it well suited for both multithreaded CPU and compute shader GPU implementations. My CPU implementation is able to mesh a 32x32x32
volume in around 0.2 ms without duplicate vertices and with angle-weighted per-vertex normals. This does come at the cost of some memory.

<video width="100%;" autoplay muted controls loop playsinline>
  <source src="/assets/marching-cubes/hyper-marching-cubes.mp4" type="video/mp4">
</video>

<div class="tldr" markdown="1">
# 🔑 Key takeaways
- Variation of Marching Cubes algorithm that directly produces unique vertices and per-vertex normals
- Can be implemented easily on both the CPU and GPU
- CPU implementation meshes 32^3 volume in 0.2 ms
- Higher (but fixed) memory cost compared to standard Marching Cubes
</div>

# Background

The marching cubes algorithm generates a mesh for a 3d noise volume. In technical terms, it reconstructs the isosurface.
Due to its capacity to produce arbitrary meshes, it is often used for procedurally generated terrain. Terrain can be modified
by altering the underlying noise volume, then remeshing using the algorithm. For a great overview of the algorthm,
see <a href="https://www.youtube.com/watch?v=vTMEdHcKgM4">Sebastian Lague's Terraforming Video</a>.

# Hyper Marching Cubes

This section describes the current state of my implementation, with all the tricks I've come up with.
It runs multithreaded using Burst-compiled Jobs.
I feel that I've changed the core algorithm enough that I get to call it something different.
So, I am giving it the name "Hyper Marching Cubes".
The crucial difference between the Hyper Marching Cubes algorithm and the standard version are that we can perform triangulation
and per-vertex normal calculation at the same time as we output vertices, with the addition of some very cheap post-processing steps.

## Unique Edge Index

In standard implementations of marching cubes, you first generate "Vertex Groups," which are vertex triplets
that represent a triangle. After all vertex groups have been generated, you can then generate the triangle indices. 
I wanted to find a way to generate the triangle indices immediately alongside the vertices.

First, what is the triangle index data actually saying? Well, it's telling the GPU to connect certain vertices.
For example, vertices 0, 3, and 8 form a triangle.

put image here 

As I stated before, we can't generate the triangulation right away because we don't know what index in the output vertex array
a given vertex will end up. But I thought that there must be a different way of representing the triangulation that didn't rely on this.
After thinking about this problem for a couple of weeks, I realized the solution: edges. Rather than outputting
triangle data as connections between vertices, I could output them as connections between edges in the volume! Then, in a post-processing
stage, once the vertex count was known, I could remap them to the actual indices. To make this work, I would need
a way to assign a unique number to each edge in the volume.

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
which confirms that the original function is correct.

Now, instead of outputting a vertex group, we can output an int3 with the three edges to connect.
We'll need 2 post-processing stages in which we remap edge to index.
To do this, we must allocate an int array of size `MaxUniqueVertices`, which is equal to that `3R^2*(R-1)` from before. We'll run the first post-processing stage
over the number of unique vertices output by marching cubes. We then write the current index to this int array at the index
of this vertex's unique edge index. 

```c#
public void Execute(int index)
{
    // Assign each vertex a unique index in the range 0 to vertexCount-1:
    int vertexEdge = vertexEdges[index];
    indexByEdge[vertexEdge] = index;
}
```

The second stage runs for each triangle, and reads in the indices we stored in the first stage.

```c#
public void Execute(int index)
{
    // Remap the triangles to use the new vertex indices:
    int3 triangle = triangles[index];
    triangles[index] = new int3(indexByEdge[triangle.z], indexByEdge[triangle.y], indexByEdge[triangle.x]);
}
```

It might be hard to understand, so here's a visual breakdown of what's going on:

![Each edge can be assigned a 3d position](/assets/marching-cubes/edge-connections.png)

Let's say the mesh outputs 9 vertices and 3 triangles. In addition to the vertex positions,
we also output the vertex edge numbers.

![Each edge can be assigned a 3d position](/assets/marching-cubes/meshing-output-1.svg)

In the first post-processing stage, we map vertex edge number to its position in the vertices array by
using the edge as an index into an array.

![Each edge can be assigned a 3d position](/assets/marching-cubes/meshing-output-2.svg)

In the second post-processing stage, we remap the triangles to use these values.

![Each edge can be assigned a 3d position](/assets/marching-cubes/meshing-output-3.svg)

## Fast Per-Vertex Normals

This edge indexing technique also allows us to generate per-vertex normals much faster.
Normally, you would have to go through all triangles, calculate their normals,
and sum them for each vertex. We can instead use the edges as an index into a local sum array.
Since each vertex can only be used by 4 cubes, we allocate a float array of length `MaxVertices * 4 * 3`.
The `* 3` is because each normal has 3 components. Each bucket of 12 floats within this array describes the
triangle normals that could affect a vertex.

We'll need another formula to tell us which 0-3 slot a cube is in relative to a given edge.
This can be calculated from the 3D edge offset again.
```c#
// edgeOffset3D has components in the range 0-2.
// returns an index [0-3].
int RelativeCubeIndex(int3 edgeOffset3D)
{
    return ((edgeOffset3D.z >> 1) + edgeOffset3D.y) * (edgeOffset3D.x & 1) +
            ((edgeOffset3D.x >> 1) + edgeOffset3D.z) * (edgeOffset3D.y & 1) +
            ((edgeOffset3D.y >> 1) + edgeOffset3D.x) * (edgeOffset3D.z & 1);
}
```
When we calculate the triangles during meshing, we keep track of the per-vertex normal sums within this cube.
Then after processing all of its triangles, we write the normal contributions to the correct slot.
```c#
Span<float3> cubeNormalSums = stackalloc float3[12];
cubeNormalSums.Clear();
...
for (int j = 0; j < triangleCount; j+=3)
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
        cubeNormalSums[edge2] += normal * angleA;
        
        int3 triangle = new int3(edgeNumberCache[edge0], edgeNumberCache[edge1], edgeNumberCache[edge2]); 
        triangles[reservedTriangleRangeStartIndex + totalTriangleCount] = triangle;
        totalTriangleCount++;
    }
}
// After we've processed all triangles, we write the normal contributions from this cube to the correct slot for each vertex
for (int j = 0; j < 12; j++)
{
    // Only write if we did use this vertex:
    if (edgeNumberCache[j] != -1)
    {
        int cubeIndex = RelativeCubeIndex(offset(j));
        float3 sum = cubeNormalSums[j];
        normalsSumPerVertex[edgeNumberCache[j] * 12 + 0 + cubeIndex] = sum.x;
        normalsSumPerVertex[edgeNumberCache[j] * 12 + 4 + cubeIndex] = sum.y;
        normalsSumPerVertex[edgeNumberCache[j] * 12 + 8 + cubeIndex] = sum.z;
    }
}
```

Finally, in a post-processing stage, we sum the 4 float3s for each vertex to get the final normal.
We can actually do this in the same stage as when we map vertex edge to index.
```c#
public void Execute(int index)
{
    // Assign each vertex a unique index in the range 0 to vertexCount-1:
    int vertexEdge = vertexEdges[index];
    indexByEdge[vertexEdge] = index;
    
    // Calculate normal:
    float3 sum = float3.zero;
    for (int i = 0; i < 4; i++)
    {
        sum.x += normalSumByVertexIndex[specialIndex * 12 + 0 + i];
        sum.y += normalSumByVertexIndex[specialIndex * 12 + 4 + i];
        sum.z += normalSumByVertexIndex[specialIndex * 12 + 8 + i];
    }
    UnsafeUtility.MemClear(addr, 12 * 4);
    
    (uniqueVerticesListPtr + index)->Normal = math.normalizesafe(sum, new float3(0, 1, 0));
}
```

# Results

## CPU Implementation

## GPU Implementation

# Naive CPU Implementation

To first gain a better understanding of how the algorithm worked, I implemented a naive CPU-based version.
This was very slow but it functioned and allowed me to fully grasp the algorithm.

<video width="100%;" muted controls loop playsinline autoplay>
  <source src="/assets/marching-cubes/digging.mp4" type="video/mp4">
</video>

# Slow GPU Implementation

In Sebastian Lague's video, he used a compute shader to generate the mesh. This gave him speedups, so I thought
it was the logical next step. I implemented it, and it was faster than my terrible CPU implementation.
However, it was still sort of slow, as getting the mesh data back from the GPU had a large delay (and was just annoying).

If we were using a fully GPU driven terrain rendering approach, GPU meshing might have more merit. But if the CPU for any reason
needs the data (e.g. for Physics, Networking), I realized that it's better to just do it on the CPU.

<video width="100%;" muted controls loop playsinline autoplay>
  <source src="/assets/marching-cubes/gpu-based.mp4" type="video/mp4">
</video>

# Future Work

A common downside of marching cubes in compute shader approaches is that you can't really produce unique vertices without sending the data back to the CPU.
Since Hyper Marching Cubes produces unique vertices directly, it is actually quite GPU friendly. It would be interesting to implement it with a series of 
compute shaders and use fully GPU-based rendering. The first dispatch would do the meshing, the second would be an indirect dispatch to remap the indices
and calculate normals, and the third would be another indirect dispatch to triangulate. All could be done on the GPU without any special data structures.