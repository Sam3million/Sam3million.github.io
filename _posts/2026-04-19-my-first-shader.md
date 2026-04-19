---
layout: post
title: "Unity Grass Shader"
date: 2026-04-19
---

### The Project
This is a compute-shader driven grass system.

<video width="100%" autoplay loop muted playsinline>
  <source src="/assets/videos/grass.mp4" type="video/mp4">
</video>

```cpp
// This is where your HLSL code goes
float4 frag (v2f i) : SV_Target {
    return _Color;
}