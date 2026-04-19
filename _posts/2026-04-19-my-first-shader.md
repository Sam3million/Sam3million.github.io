---
layout: default
title: "Unity Grass Shader"
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