---
layout: post
title: "CPU Raytracer"
date: 2022-02-01
thumbnail: "/assets/cpu-raytracer/frame-output.png"
category: gamedev
---

This was a project I would work on in my free time during my senior year of high school.
I thought it would be fun to write a raytracer from scratch without relying too much on outside information.
I wrote it in Java because that was what we were using in my AP CSA class. I watched a bunch of videos on linear algebra,
and wrote a library based on my newly acquired knowledge (thank you 3Blue1Brown). I derived ray intersection
functions for various shapes, implemented texturing, reflections, shadows, and skybox. It ran super slow of
course, but it gave me a deep understanding of rendering fundamentals.

![CPU Raytraced Scene](/assets/cpu-raytracer/frame-output.png)
*CPU Raytraced Scene*
<br />

# Demo Video
Here's a demo of the raytracer where I move the camera and light around and adjust the reflection count.
The window is only 250x250 pixels in order for it to run in real time.
<video width="100%;" muted controls loop playsinline autoplay>
  <source src="/assets/cpu-raytracer/demo.mp4" type="video/mp4">
</video>