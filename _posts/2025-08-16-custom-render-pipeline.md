---
layout: post
title: "Custom Render Pipeline"
date: 2025-08-16
thumbnail: "/assets/aridsun/arid-sun-1.png"
category: gamedev
---

I wrote a custom scriptable render pipeline in Unity, implementing deferred rendering,
forward rendering, render passes, realtime lighting, shadow mapping, bloom, tone mapping,
and more. Deferred rendering for many lights. Bloom implemented using compute shaders to
leverage LDS, giving higher performance and quality than raster based bloom.