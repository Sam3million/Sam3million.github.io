---
layout: post
title: "Custom Render Pipeline"
date: 2025-08-16
thumbnail: "/assets/render-pipeline/thumbnail.png"
category: gamedev
---

I wrote a custom scriptable render pipeline in Unity, implementing deferred rendering,
forward rendering, render passes, realtime lighting, shadow mapping, bloom, tone mapping,
and more. Deferred rendering for many lights. Bloom implemented using compute shaders to
leverage LDS, giving higher performance and quality than raster based bloom.

# Background

The need for a custom render pipeline arose from limitations with Unity's built in render pipeline.
For my survival game, in order to perform occlusion culling on my GPU-driven grass, I needed read
access to the depth buffer AND the ability to render the grass to it. For some reason, this was not possible.
In addition, to receive lighting on my grass, I needed to use deferred rendering. So, for these reasons
and more I decided to write my own render pipeline.

# Setup

I used Unity's Scriptable Render Pipeline documentation and CatlikeCoding's render pipeline series to get started.
After getting basic rendering working with command buffers, I switched to Unity's RenderGraph for a more modern approach.
The documentation is not great, so this process involved a lot of trial and error.

The primary rendering approach I focused on for this pipeline was deferred + forward transparent, though I did add basic
support for forward opaque passes. I use 3 GBuffers; the first is for HDR color, the second is for normals + smoothness, and the third is for
emission, metallic, occlusion, and height.

# Skybox

I used an implementation of Bruneton's precomputed atmospheric scattering. This procedural skybox model
allows for realistic and performant dynamic skyboxes that change with sun position. I built upon the base implementation,
adding things like weather variants and precomputed ambient lighting.

# Lighting

## Directional Lights



## Point Lights

Deferred rendering allows for many point lights to affect the scene at once. The performance cost is minimal.

## Ambient Lighting

As I mentioned before, the ambient lighting is precomputed from the skybox. I use a compute shader to calculate spherical
harmonics at various times of day. At runtime, these can be interpolated between for a smooth day-night ambient lighting transition.
I also added color overrides for more artistic control.

## Shadows

Shadows were implemented using the age-old cascaded shadow mapping technique.

# Post Processing Effects

## Bloom

Bloom was implemented using a compute shader approach, based on Unity's HDRP bloom implementation.
The image is progressive downsampled and upsampled, with blur applied at each stage. Using a compute shader
allows for better performance because pixels can be loaded into groupshared memory instead of each thread having
to do many reads.

## Tonemapping

Tonemapping is quite simple, it's just a function applied to each pixel's color, transforming it slightly.
I used an ACES tonemapping function, and also added parameters like hue, saturation, and contrast to this render pass.

# Future Work

- Realtime dynamic global illumination
- Raytracing
- Ambient occlusion
- 