---
layout: post
title: "Raytracing Demo"
tags: [programming, rendering, games]
date: 2022-01-27 16:11 -0800
description: A demo video featuring raytraced speheres
---

{% include video_embed.html file='rt_sphere' %}

Recently I've been reading through [Computer Graphics from Scratch](https://gabrielgambetta.com/computer-graphics-from-scratch/). The sections on raytracing have been a great refresher for me, and I've learned some new tricks.

I've written up a [C++ software raytracer](https://github.com/rivergillis/graphics-from-scratch) based on the book and now I've got an animation to show for it!
The above video combines about 2200 frames that were rendered as [PPM images](https://github.com/rivergillis/graphics-from-scratch/blob/b825be43fe96eb5ea1924d80dcee82d257da8651/image.cpp#L68) and stitched together using ffmpeg. Each 900x900 frame takes about 1 second to render on my M1 mac,
 and the renderer streams the images to an SDL2 window as they finish so that I can see the progress. Lowering the quality of the render allows us to get to real-time, but I prefer the look of the reflections.
