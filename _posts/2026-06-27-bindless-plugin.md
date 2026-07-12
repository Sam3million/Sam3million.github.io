---
layout: post
title: "Bindless Rendering Plugin for Unity"
date: 2026-06-27
thumbnail: "/assets/bindless-plugin/terrain.png"
category: gamedev
---

I created a native plugin for Unity that enables the use of bindless resources on Metal.
This can greatly reduce CPU overhead related to state changes when drawing. I use it
to improve terrain rendering performance for my survival game, as each chunk needs unique buffers
bound to the fragment stage. To eliminate state changes, the buffer addresses are stored in a single
global buffer, then accessed via the baseInstance offset provided to the draw call.

![Terrain](/assets/bindless-plugin/terrain.png)
*Thousands of chunks drawn at once - ignore gaps between LODS :)*

# The Issue
In my survival game, terrain chunks all use the same shader. However, to properly render the
different ground types of the terrain (e.g. dirt, grass, stone), I need access to each chunk's Ground Type Buffer
from within the fragment stage. This buffer stores a ground type ID for each vertex in the mesh, which can then
be used as an index into a texture array in the fragment shader. Some newer hardware supports access to the raw
per-vertex values of a primitive from within the fragment stage. However, on older hardware the only options seem to be
geometry shaders (which aren't supported in Metal) or manually binding and reading from the buffer. This manual
binding method works but introduces state change overhead because each buffer has to be manually bound before each draw call.

![Ground types](/assets/bindless-plugin/ground-types.png)
*Many ground types have to be rendered and smoothly blended*

# The Solution

Enter bindless rendering. This paradigm eliminates the need to manually bind buffers/resources to a shader. Instead,
resources can simply be marked resident on the GPU, and their addresses passed via a constant buffer. In the shader,
the address is dereferenced like a standard pointer. This comes in handy when you have many buffers that need to be bound (like in my case).
Rather than binding each explicitly, you can collect their GPU addresses in an argument buffer, then only make that one binding.
To access the proper address from within the shader, we pass in an index through the baseInstance parameter of the draw
call (essentially serving as a 'draw id').

![Grabbing GPU Address](/assets/bindless-plugin/solution.png)
*Grabbing GPU Address*

# The Plugin
The plugin is written in Objective-C++ and exposes a couple of functions to the C# world. The first is an initialization function
that takes in an MSL shader string and creates the pipeline state object (PSO), configures render descriptors, and sets up depth-stencil states. 
The second is a function that returns the GPU address for a given native buffer pointer. The final one is the actual rendering call.
It takes in my terrain draw information, marks the proper resources as resident, and issues the draw calls. This function is called
from a custom pass in my SRP with RasterCommandBuffer.IssuePluginEventAndData.

![Plugin snippet](/assets/bindless-plugin/plugin-code-snippet.png)
*Plugin Snippet*

# Limitations
This plugin is pretty manual and tailored to my terrain-rendering use case. It doesn't work with Unity Materials, so
I have to provide the MSL shader source and create the PSO myself. This is actually nice for my specific use case because it allows
me to use native barycentric coordinates in the fragment stage (Unity frustratingly seems to provide older shader model input into SPIRV-Cross,
so SV_Barycentrics doesn't properly compile into MSL). Having to manually make the PSO does hinder the range of use of the plugin in its current
state. The reason it doesn't work with Materials is that Unity doesn't provide any API to access the PSO for a given shader. There is
a way to hook into the rendering system and directly use Unity's PSO (see <a href="https://github.com/saivs/com.saivs.plugin.mdi">Saivs' MDI Plugin</a>), but it's kinda hacky. I may go back and write a more general-purpose plugin, but for now it does the job.