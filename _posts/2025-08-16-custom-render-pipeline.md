---
layout: post
title: "Custom Render Pipeline"
date: 2025-08-16
thumbnail: "/assets/render-pipeline/stars.png"
category: gamedev
---

I wrote a custom scriptable render pipeline in Unity, implementing deferred rendering,
forward rendering, render passes, realtime lighting, shadow mapping, bloom, tone mapping,
and more. I mainly did this to give me complete control over rendering and to maximize
performance.

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

![Image of a graphical error](/assets/render-pipeline/error.png)

The primary rendering approach I focused on for this pipeline was deferred + forward transparent, though I did add basic
support for forward opaque passes. I use 3 GBuffers; the first is for color + translucency, the second is for normals + smoothness,
and the third is for emission, metallic, occlusion, and height. The frame buffer is stored in a B10G11R11 unsigned float format for HDR support.

# Skybox

I used an implementation of Bruneton's precomputed atmospheric scattering. This procedural skybox model
allows for realistic and performant dynamic skyboxes that change with sun position. I built upon the base implementation,
adding support for precomputed ambient lighting.

![Image of the skybox](/assets/render-pipeline/skybox.png)

## Celestial Bodies

Celestial bodies like the sun, moon, and stars are implemented as a separate render passes from the skybox.
The sun and moon are rendered as a fullscreen fragment shader, while the stars are actually rendered with a tiled approach
through a compute shader. Stars can be randomly generated, or actual datasets can be used. The image below
shows the bright star catalogue being rendered, which is a dataset of the stars visible to the naked eye. I may
write a separate post about this because it's pretty cool.

![Celestial Bodies](/assets/render-pipeline/stars.png)
*Bright star catalogue rendered in my custom render pipeline*

# Lighting

## Directional Lights

Directional lights are far away light sources like the sun and moon. I have support for up to 4 directional lights in my pipeline.

![Directional Lights](/assets/render-pipeline/directional-lights.png)

## Point Lights

Deferred rendering allows for many point lights to affect the scene at once. The performance cost is lower than in standard forward rendering,
as we only have to calculate the lighting once per pixel.

![Point Lights](/assets/render-pipeline/point-lights.png)

## Ambient Lighting

As I mentioned before, the ambient lighting is precomputed from the skybox. I use a compute shader to calculate spherical
harmonics at various times of day. At runtime, these can be interpolated between for a smooth day-night ambient lighting transition.
I also added color overrides for more artistic control.

![Ambient Lighting](/assets/render-pipeline/ambient-lighting.png)

## Shadows

Shadows were implemented using the typical cascaded shadow mapping technique. I looked into other approaches, such as
shadow volumes, but they were too complex or performance-heavy to implement. I may revisit this in the future though
because I don't quite like the look of shadow mapping.

![Shadows](/assets/render-pipeline/shadows.png)

# Post Processing Effects

## Bloom

Bloom was implemented using a compute shader approach, based on Unity's HDRP bloom implementation.
The image is progressive downsampled and upsampled, with blur applied at each stage. Using a compute shader
allows for better performance because pixels can be loaded into groupshared memory instead of each thread having
to do many reads.

![Bloom](/assets/render-pipeline/bloom.png)

## Tonemapping

Tonemapping is just a function that transforms each pixel's color slightly. This allows for more artist control over
the final look. I used an ACES tonemapping function, and also added parameters like hue, saturation, and contrast to this render pass.

![Without Tonemapping](/assets/render-pipeline/without-tonemapping.png)
*Without tonemapping*

![With Tonemapping](/assets/render-pipeline/with-tonemapping.png)
*With tonemapping*

## Skybox Fog

This effect is like standard fog, but samples the skybox color rather than a solid color. This gives a nice
fade out for distant objects.

![Skybox Fog](/assets/render-pipeline/skybox-fog.png)

# Future Work

- Realtime dynamic global illumination
- Raytracing
- Ambient occlusion
- Clouds
- Liquids
- Per-triangle shadow volumes