---
layout: post
title: Project proposal
subtitle: A review of hardware-based acceleration of Ray tracing
# gh-repo: daattali/beautiful-jekyll
# gh-badge: [star, fork, follow]
tags: [proposal]
comments: true
author: An Vuong 
---
{: .box-success}
Updated Feb 21, 2024. The page for Midreport is created at [here]({{ site.baseurl }}{% link _posts/2024-02-21-project-mid-report.md %})

{: .box-success}
This short post serves as a proposal of our project topic for ECE570 course - taught by Dr. Ben Lee at Oregon State University in Winter 2024.

# Table of Contents
1. [Abstract](#abstract)
2. [Short introduction](#shortintroduction)
    1. [What is ray tracing?](#introraytracing)
    2. [What we did before ray tracing? Rasterization!](#rasterization)
    3. [Some comparison examples between rasterization and ray tracing](#comparsion)
3. [What we want to include in the project?](#conclusion)
4. [References](#references)

## Abstract <a name="abstract"></a>

Ray tracing is an old problem in graphics processing. In this setting, rays are emitted from a source, then its reflecting and refracting rays from other objects are then collected and processed. Using this information, a detailed and accurate simultion of the environment can be constructed. This technique has been widely applied to many fields such as: film making, physics simulation of light and sound wave, etc. First demonstrated successfully in the 1960s, real-time ray tracing on consumer devices still remains a holy grail for computer scientists up until 2018, when Nvidia released the RTX 2000 series with the push for this feature. Since then, a multitude of research in both software and hardware has been devoted to accelerate this process. In this work, we will review the basic of ray tracing, its evolution over the year, and the current state of the art technique.

## Short introduction <a name="shortintroduction"></a>
### What is ray tracing? <a name="introraytracing"></a>

While ray tracing has many applications, here we will constraint ourselves to the problem of simulating light. In the real world, we see color of different objects because the light rays coming from some sources (the sun, lamps, monitor, etc.) hitting the objects and get either diffused, reflected, or refracted. The combination of these effects create the colorful world we see every day. In ray tracing based simulation, we will follow exactly this procedure. We begin by shooting a bunch of rays, when these rays hit an object, using a physical model, accurate calculations of the reflecting and refracting rays are computed, then we will follow these resulting rays until they hit the camera/eyes, then summing these rays up will give a precise picture of the world we are simulating. 

In practice, instead of shooting rays from the light source, we actually shoot those rays from the camera/eyes and keep track of all the rays that hit the objects, this works thanks to the bi-directional property of light. This is demonstrated below:
![image](/ece570page/assets/img/raytracing/1920px-PathOfRays.svg.png "Ray tracing")

{:.image-caption}
*Simple model of ray tracing. Source: Wikipedia*

In 2018, with the release of the RTX 2000 series, Nvidia introduced the RT cores, whose sole purpose is to accelerate the ray tracing calculations. Since then, the architecture has been refined and new algorithms have been introduced, culminating in something we now call Ray Reconstruction, which is currently state-of-the-art for real-time ray tracing. We'll put our main focus on the RT cores, ray tracing pipeline, and the Ray Reconstruction technique in our project.

![image](/ece570page/assets/img/raytracing/rtcores.png "RT Cores")

{:.image-caption}
*New RT Cores in Nvidia Turing - Source: Nvidia white paper*

### What we did before ray tracing? Rasterization! <a name="rasterization"></a>

Before the commoditization of ray tracing, in order to render a scence, be it a movie scene, a 3D object, or a video game frame, the most popular approach is rasterization. While rasterization pipelines are different for different architectures, the general idea is to divide the scene into a bunch of polygons (usally triangles) with connected edges. Then shaders are used to color these polygons based on some pre-defined rules set by the designers. Because of this, the rendering of the scene is actually based on some carefully chosen rules in order to best mimic reality, not the actual physical rules. While experienced designers can make a rasterized scene looks excellent, it is still an emulation. In constrast, ray tracing strives to render a scene based on the actual physical laws of light. In other words, rasterization tries to emulate the world, while ray tracing tries to simulate it. 

While rasterization will not be the main focus of our project, we think it is needed in order to understand how impactful ray tracing is, so we will briefly introduce this in our projects. We will also give some examples to make comparison between these two techniques, some of them are shown in the next section.

### Some comparison examples between rasterization and ray tracing <a name="comparison"></a>
![image](/ece570page/assets/img/raytracing/cyberpunk.png "Cyberpunk 2077")

{:.image-caption}
*A scene from Cyberpunk 2077, left is with RT, right is with rasterization*

![image](/ece570page/assets/img/raytracing/farcry.png "Far Cry 6")

{:.image-caption}
*A scene from Far Cry 6, left is with RT, right is with rasterization*

We can see that the difference ray tracing provides is quite massive. The reflections are actually based on the physical model of the world, since their constructions are from the information provided by the traced rays. On the other hand, rasterization has to rely on heuristic techniques called baked lighting and cube map, while this can work well, it has to be manually tuned for each scene and requires a high level of artistic skills.

### Performance concerns <a name="performance"></a>

While ray tracing does provide a massive uplift in visual quality, it also comes with one of the biggest hit in performance ever seen in a feature in the industry. At release, the performance deficit of using ray tracing on an RTX 2080 can be as much as 50%, which made many people concerned whether it was worth it. Having anticipated this, Nvidia also introduced an image reconstruction technique called DLSS that took advantage of the improved tensor cores to balance this performance hit. The general idea is to applying ray tracing to a low resolution frame, then apply DLSS to upscale that rendered frame to high resolution. In this project, we will also review about the tensor cores, while this is also not the main focus, it is one of the main components that makes ray tracing works in practice!

## What we want to include in the project <a name="conclusion"></a>

To summarize, we are working to include the following in our project:
- Review of rasterization 
- Review of ray tracing
- How RT cores accelerate ray tracing
- How Tensor cores play a part in this process
- Comparison between rasterization and ray tracing
- Some performance benchmarks between software-based and hardware-based ray tracing

## Some references <a name="references"></a>
- [Ray Reconstruction in DLSS 3.5](https://www.nvidia.com/en-us/geforce/news/nvidia-dlss-3-5-ray-reconstruction/)
- [Nvidia Turing white paper](https://images.nvidia.com/aem-dam/en-zz/Solutions/design-visualization/technologies/turing-architecture/NVIDIA-Turing-Architecture-Whitepaper.pdf)
- [Nvidia Ampere white paper](https://www.nvidia.com/content/PDF/nvidia-ampere-ga-102-gpu-architecture-whitepaper-v2.pdf)