---
layout: post
title: Project proposal
subtitle: A review of hardware-based acceleration of Ray tracing
# gh-repo: daattali/beautiful-jekyll
# gh-badge: [star, fork, follow]
tags: [proposal]
comments: true
author: A. Vuong & N. Nguyen
---

{: .box-success}
This short post serves as a proposal of our project topic for ECE570 course - taught by Dr. Ben Lee at Oregon State University in Winter 2024.

# Table of Contents
1. [Abstract](#abstract)
2. [Short introduction](#shortintroduction)
3. [What is ray tracing?](#introraytracing)

## Abstract <a name="abstract"></a>

Ray tracing is an old problem in graphics processing. In this setting, rays are emitted from a source, then its reflecting and refracting rays from other objects are then collected and processed. Using this information, a detailed and accurate simultion of the environment can be constructed. This technique has been widely applied to many fields such as: film making, physics simulation of light and sound wave, etc. First demonstrated successfully in the 1960s, real-time ray tracing on consumer devices still remains a holy grail for computer scientists up until 2018, when Nvidia released the RTX 2000 series with the push for this feature. Since then, a multitude of research in both software and hardware has been devoted to accelerate this process. In this work, we will review the basic of ray tracing, its evolution over the year, and the current state of the art technique.

## Short introduction <a name="shortintroduction"></a>
### What is ray tracing? <a name="introraytracing"></a>

While ray tracing has many applications, here we will constraint ourselves to the problem of simulating light. In the real world, we see corlor of different objects because the light rays coming from some sources (the sun, lamps, monitor, etc.) hitting the objects and get either diffused, reflected, or refracted. The combination of these effects create the colorful world we see every day. In ray tracing based simulation, we will follow exactly this procedure. We begin by shooting a bunch of rays, when these rays hit an object, using a physical model, accurate calculations of the reflecting and refracting rays are computed, then we will follow these resulting rays until they hit the camera/eyes, then summing these rays up will give a precise picture of the world we are simulating. 

In practice, instead of shooting rays from the light source, we actually shoot those rays from the camera/eyes and keep track of all the rays that hit the objects, this works thanks to the bi-directional of light. This is demonstrated below:
![image](/assets/img/raytracing/1920px-PathOfRays.svg.png "Ray tracing")

{:.image-caption}
*Simple model of ray tracing. Source: Wikipedia*

### What we did before ray tracing? Rasterization!