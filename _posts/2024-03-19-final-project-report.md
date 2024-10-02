---
layout: post
title: Final Project Report
subtitle: A review of hardware-based acceleration of Ray tracing
# gh-repo: daattali/beautiful-jekyll
# gh-badge: [star, fork, follow]
tags: [report]
comments: true
author: An Vuong 
---
## Table of contents 
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Abstract ](#abstract)
- [Background ](#background)
   * [Graphics rendering pipeline ](#graphics-rendering-pipeline)
   * [Rasterization](#rasterization)
   * [Ray tracing](#ray-tracing)
      + [What is ray tracing? ](#what-is-ray-tracing)
      + [Ray tracing as a replacement for Rasterization](#ray-tracing-as-a-replacement-for-rasterization)
      + [Performance concerns ](#performance-concerns)
- [Hardware acceleration](#hardware-acceleration)
   * [Overview of Turing architecture](#overview-of-turing-architecture)
      + [More details on SMs](#more-details-on-sms)
      + [Memory hierarchy](#memory-hierarchy)
   * [RT cores for Ray tracing](#rt-cores-for-ray-tracing)
      + [Acceleration of BVH traversal](#acceleration-of-bvh-traversal)
      + [Acceleration by upscaling](#acceleration-by-upscaling)
      + [An example](#an-example)
- [Conclusion](#conclusion)
- [References ](#some-references)

<!-- TOC end -->

<!-- TOC --><a name="abstract"></a>
## Abstract 
Ray tracing is an old problem in graphics processing. In this setting, rays are emitted from a source, then its reflecting and refracting rays from other objects are then collected and processed. Using this information, a detailed and accurate simultion of the environment can be constructed. This technique has been widely applied to many fields such as: film making, physics simulation of light and sound wave, etc. First demonstrated successfully in the 1960s, real-time ray tracing on consumer devices still remains a holy grail for computer scientists up until 2018, when Nvidia released the RTX 2000 series with the push for this feature. Since then, a multitude of research in both software and hardware has been devoted to accelerate this process. In this work, we will review the basic of ray tracing, its evolution over the year, and the current state of the art technique.

<!-- TOC --><a name="background"></a>
## Background 
This section reviews the basics of graphics rendering pipeline. Some details on Rasterization and Raytracing are also given. The information presented here mostly follows the standard from OpenGL.
<!-- TOC --><a name="graphics-rendering-pipeline"></a>
### Graphics rendering pipeline 
In order to display a scene on a monitor, a 3D scene needs to be correctly transformed into a 2D representation, which can then be passed to to the display buffer and shown. This process of transforming a 3D scene to a 2D representation is often referred to as the **graphics rendering pipeline**. At a very high level, this rendering pipeline can be broken down into 3 major steps:
![image](/ece570page/assets/img/rendering/Graphics_pipeline_2_en.svg.png "Rendering pipeline")

{:.image-caption}
*General rendering pipeline. Source: Wikipedia*

Briefly, what happens in each step is:
- **Application**: The information of 3D scene is generated, usually by CPU. In video games, this is when the game reacts to the inputs from the players. Animations, textures, collisions of objects, and other physical/environmental simulations are calculated according to the inputs. This information is then passed from the CPU to the graphics processors (GPU) for the next step.
- **Geometry**: Usually this is the actual beginning of of the rendering pipeline. This step often comprises of Vertex pipeline and Geometry pipeline, these processes will transform the 3D scene information based on the camera and the viewport information, lighting and textures are also being applied here. Many optimizations are done in this step such as clipping and frustum culling. By the end of the stage, a complete scene with color and texture information is ready to be shown, but the information is still in 3D space, thus the need for the next step.
- **Rasterization**: Here, the 3D scene/objects provided from the previous step will be projected onto the 2D display. Next section deals with this stage in more detail.

Going a bit deeper, OpenGL graphics APIs define the rendering pipeline as the following:
<!-- ![image](/ece570page/assets/img/rendering/opengl-RenderingPipeline.png "OpenGL Rendering pipeline") -->
<p align="center" width="100%">
    <img width="33%" src="/ece570page/assets/img/rendering/opengl-RenderingPipeline.png " > 
</p>

{:.image-caption}
*OpenGL Rendering pipeline. Source: Khronos Group*

The steps in blue are accessible and programmable by developers, whereas steps in yellow are fixed functions provided by abstractions from OpenGL depending on the underlying hardware supports. Things might look different in DirectX12 and Vulkan graphics APIs since these libraries offer much more granular access to the hardware, where programmers can customize almost everything. For the purpose of this work, we will assume the model of OpenGL. Next section will focus on the Rasterization stage, the one after that will discuss how it can be replaced by Raytracing and what are the costs of doing so.

![image](/ece570page/assets/img/rendering/exmaple-graphics-pipeline.png "Example Rendering pipeline")

{:.image-caption}
*An example of rendering pipeline. Source: Graphics Compendium*
<!-- TOC --><a name="rasterization"></a>
### Rasterization
At a high level, rasterization (or raytracing later on) deals with the problem of projecting a 3D object onto a 2D surface (our displays). 

![image](/ece570page/assets/img/rendering/projection_3d_to_screen.png "Example rasterization")

{:.image-caption}
*Rasterization of a simple triangle. Source: Mariano Trebino*

As shown in the figure above, this problem roughly translates to the problem of coloring each pixels depending on whether the object falls onto that position or not. This problem, simple as first glance, turns out to be very expensive to compute. A naive approach is to iterate through all the pixels and check whether it overlaps with the object or not. This might be very inefficient if the object is small.

<p align="center" width="100%">
    <img width="50%" src="/ece570page/assets/img/rendering/iterate_screen_pixels.png " > 
</p>

{:.image-caption}
*Rasterization of a simple triangle. Source: Mariano Trebino*

A simple yet effective optimization is to first calculate the bounding box of the triangle, then only iterate over the pixels that fall within the bounding box.

<p align="center" width="100%">
    <img width="50%" src="/ece570page/assets/img/rendering/optimized_iteration.png " > 
</p>

{:.image-caption}
*Optimized rasterization of a simple triangle. Source: Mariano Trebino*

The next question is what if there are multiple objects (triangles)? If we simply apply the same idea, there will be issues with the rendering order, since a 3D scene has depth (one object can be behind another), this geometry needs to be preserved when projecting onto the screen. A nice trick in the graphics processing is to use the depth buffer (also known as the Z-buffer). The Z-buffer stores the distances (Z-axis values) from the projection center to the objects, this in turns help the pipeline to determine the correct order.

<p align="center" width="100%">
    <img width="55%" src="/ece570page/assets/img/rendering/depth_problem.png" > 
</p>

{:.image-caption}
*Rasterization of a 2 triangles. Source: Mariano Trebino*

The Z-buffer also plays critical role in the parallelization of rasterization. Without it, the coloring process needs to be applied in an order that respects the geometry of the 3D scene. With Z-buffer, now we can process all pixels concurrently and then apply a merge step, where information from the Z-buffer will be used to rearrange the order. This parallelization is partially why rasterization, despite being mostly a bruteforce approach, has been the most popular rendering technique since its inception. 

Since rasterization is so important to graphics rendering, modern GPUs usually dedicate large amount of their sillicon budget to implement dozens of rasterization units. For e.g, the RTX 2080 has 64 Rasterization Operation Processors (ROPs) which can be executed in parallel to create a frame. As can be seen from the diagram below, Rasterization engines play a big part in the Turing architecture, Nvidia dedicated 16 ROPs to each of 6 Graphics Processing Clusters (GPCs). These raster engines take data produced by the Streaming Multiprocessing Units (SMs) and render the frame onto the screen. The SMs effectively process the scene according to the pipeline mentioned earlier.

![image](/ece570page/assets/img/rendering/2080diagram.png "Nvidia Turing TU102")

{:.image-caption}
*Nvidia Turing TU102 High level diagram. Source: Nvidia*

To make the comparison with Raytracing easier later on, at a very high level, the compute loop for rasterization looks like this:
```python
for object in all_objects:
    for pixel in possible_pixels:
        test_if_pixel_in_object_by_position_check()
```
<!-- TOC --><a name="ray-tracing"></a>
### Ray tracing
<!-- TOC --><a name="what-is-ray-tracing"></a>
#### What is ray tracing? 

While ray tracing has many applications, here we will constraint ourselves to the problem of simulating light. In the real world, we see color of different objects because the light rays coming from some sources (the sun, lamps, monitor, etc.) hitting the objects and get either diffused, reflected, or refracted. The combination of these effects create the colorful world we see every day. In ray tracing based simulation, we will follow exactly this procedure. We begin by shooting a bunch of rays, when these rays hit an object, using a physical model, accurate calculations of the reflecting and refracting rays are computed, then we will follow these resulting rays until they hit the camera/eyes, then summing these rays up will give a precise picture of the world we are simulating. 

In practice, instead of shooting rays from the light source, we actually shoot those rays from the camera/eyes and keep track of all the rays that hit the objects, this works thanks to the bi-directional property of light. This is demonstrated below:

<p align="center" width="100%">
    <img width="65%" src="/ece570page/assets/img/raytracing/1920px-PathOfRays.svg.png" > 
</p>

{:.image-caption}
*Simple model of ray tracing. Source: Wikipedia*

<!-- TOC --><a name="ray-tracing-as-a-replacement-for-rasterization"></a>
#### Ray tracing as a replacement for Rasterization
Recall that rasterization can be thought as a problem of checking whether a pixel overlaps with an object or not. We can apply the idea of raytracing to this problem as follows: starting from the camera (center of projection), we shoot a ray at each pixel, then we trace this ray to see if it hits any object. If it does not hit anything, then the pixel should not be colored, on the other hand, if the ray hits something, we can then query the properties of that object to render the pixel.

<p align="center" width="100%">
    <img width="55%" src="/ece570page/assets/img/rendering/raytracediagram.png" > 
</p>

{:.image-caption}
*Pixel testing using ray tracing. Source: Scratch a pixel*

The compute loop for ray tracing can be thought of as:
```python
for pixel in possible_pixels:
    for object in all_objects:
        test_if_pixel_in_object_by_shooting_ray()
```

Note that the iterators of objects and pixels are swapped for ray tracing compared to rasterization. Similar to rasterization, the computation will be extremely inefficient if we just naively shoot rays at every pixel. A trick to optimize this process is to pre-calculate a tree of objects such that the childs are contained inside the parents from the perspective of the camera. There are many different data structures that try to tackle this, one of them is the Bounding Volume Hierarchy (BVH), which is being used by all the major players like Nvidia, Intel, and AMD.

As an example, the figure below shows a possible BVH structure of a rabbit. When we trace a ray, we keep going only if the ray hits the bigger bounding volume first. If not, the tracing process for that pixel is stopped. It turns out traversing the BVH tree is of O(log n) which is very efficient. Notice that by using ray tracing, we can bypass the problem of Z-buffer, since the ray will always hit the front objects first.

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/rendering/bvh.png" > 
</p>

{:.image-caption}
*BVH algorithm. Source: Nvidia*

 At this stage, it seems like we are complicating the problem by using ray tracing. But the magic happens when we keep on tracing the ray after it hits the object, i.e, allowing the rays to bounce. Every time a ray bounces off an object, we can utilize the material information and simulate the effect of lights, we can then use this newly acquired information to better coloring the pixels. The more bounces we allow a ray to have, the better the rendered image will look. So in this sense, ray tracing not only replaces rasterization as rendering engine, it actually also makes the frame looks more realistic. An example is:

 <p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/rendering/raster.png" > 
</p>

{:.image-caption}
*Rasterization. Source: Wikipedia*

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/rendering/rt.png" > 
</p>

{:.image-caption}
*Ray tracing. Source: Wikipedia*

<!-- TOC --><a name="performance-concerns"></a>
#### Performance concerns 

As mentioned earlier, thanks to the bidirectionality of light, instead of shooting rays from the light source, we can shoot rays from the camera and the scene will still be rendered accurately, this is well demonstrated in:
<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/rendering/rtdiagram.jpg" > 
</p>

{:.image-caption}
*Ray tracing diagram. Source: Nvidia*

Notice that, from the diagram, to get accurate lighting, we need to trace each ray to the light source in order to have correct lighting and shadow information. This makes ray tracing a very expensive process, thus while ray tracing does provide a massive uplift in visual quality, it also comes with one of the biggest hit in performance ever seen in a feature in the industry. 

At release, the performance deficit of using ray tracing on an RTX 2080 can be as much as 50%, which made many people concerned whether it was worth it. Having anticipated this, Nvidia also introduced an image reconstruction technique called DLSS that took advantage of the improved tensor cores to balance this performance hit. The general idea is to applying ray tracing to a low resolution frame, then apply DLSS to upscale that rendered frame to high resolution. In the following sections, we will take a look at Turing architecture and how it accelrates the processing of ray tracing.

<!-- TOC --><a name="hardware-acceleration"></a>
## Hardware acceleration
<!-- TOC --><a name="overview-of-turing-architecture"></a>
### Overview of Turing architecture
Focusing on the Ampere TU102 die from Nvidia, from a high level, this chip is organized into 6 Graphics Processing Clusters (GPCs), each GPC contains:

- a dedicated Raster Engine: this unit setups the whole rasterization process as well as performing the Z-buffer merge in the end (called Z-Cull)
- 2 raster operator partitions, each partition contains 8 ROP units: assemble the GPU output into a bitmapped image ready for display.
- 6 Texture Processing Clusters (TPCs): each TPC includes 2 Streaming Multiprocessors (SMs) and 1 PolyMorph Engine, these units mostly handle geometry processing.
- All GPCs share an L2 cache of 6MB

<p align="center" width="100%">
    <img width="100%" src="/ece570page/assets/img/arch/gpcs.png" > 
</p>

{:.image-caption}
*Zoomed in into 2 GPCs. Source: Nvidia*

The meat of a GPU usually lies in the SMs, where most of the compute happens. For this reason, TU102 organizes each SM as follow:

- 64 CUDA cores
- 8 Tensor cores to accelerate tensor processing (more on this later)
- 1 RT cores to accelerate rays processing (more on this later)
- 4 Texture units
- 256 KB register file
- 96 KB L1 cache (shared within a GPC)

In total, along with the memory controller and communication bus, the TU102 die has a massive 18.6 billion transistors, a 55% increase compared to previous generation (the GTX 1080Ti). Compare to GTX 1080Ti (PA102), each SM of the TU102 contains less CUDA cores, but the total number of SMs is more than double, which yields a 21% increase the amount of CUDA cores. This is mainly because each SM now has to give space to accomodate the newly introduced Tensor and RT cores, which takes a lot of sillicon budget to implement.

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/arch/comparetable.png" > 
</p>

{:.image-caption}
*Comparison with Pascal architecture. Source: Nvidia*

<!-- TOC --><a name="more-details-on-sms"></a>
#### More details on SMs

Going deeper into the implementation of SMs, we found the following setup:
- Warp schedulers: each SM has 4 warp schedulers, this unit handle a static set of warps (each warp is a collection of fixed number of threads, usually 32) and issues instructions to a dedicated set of arithmetic instruction units. 
- Instructions are performed over 2 cycles, independent instructions can be issued every cycle (pipelined)
- For core FMA (fused-multiply add) math operations, dependent instructions issue latency is 4 clock cycles. Since there are 4 warp schedulers within an SM, this latency can be hidden by using 4-way instruction-level parallelism per warp.

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/arch/sm.png" > 
</p>

{:.image-caption}
*Organization of each SM. Source: Fabien Sanglard*

An interesting aspect of the TU102 SMs is what Nvidia counts as CUDA cores. The listed 64 CUDA cores are, in fact, 128 units. Half of them can only execute INT32 operations, while the other half only does FP32. All of these cores can be executed in parallel, so the best way to utilize the TU102 is to use mixed precision computing, otherwise half of the CUDA cores will be effectively sleeping during the computation. For FP64 operations, TU102 only has 2 dedicated units, so high-precision calculation is not recommended on this device. Besides the CUDA cores, which can be thought of general computing units, there are also 4 Special Functions Units (SFUs) and 8 Tensor cores. 

SFUs are utilized when dealing with special functions like logarithms, sinusoidals, transcendentals, etc. This helps accelerate complex computations and free the CUDA cores for other jobs. As for the Tensor cores, this is an entirely new component of TU102. These units can also be thought of as another kind of SFUs, but instead of some special functions, the tensor cores' whole purpose is to maximize high-dimensional matrix (tensor) multiplications. The reason for the existence of this is because Nvidia predicted the rise of deep learning, which, at its core, is all about matrix multiplications. So they placced a huge bet on this functionality, and it seems to be paying off nicely. It turns out Tensor cores also play a critical role in accelerating ray tracing, which we will go into later.

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/arch/tensorcore.gif" > 
</p>

{:.image-caption}
*Acceleration of matrix multiplcation using Tensor cores. Source: Nvidia*

<!-- TOC --><a name="memory-hierarchy"></a>
#### Memory hierarchy
In order to feed all of these computation units with data, TU102 also made big changes in the design of memory compared to the previous generations. In Turing, the memory is now unified, this creates a single path for texture caching and memory loads and frees up L1 for other data. An interesting thing is the amount of unified/shared memory can now be set up at run time, applications can decide whether they need more L1 cache or more shared memory, for e.g, from the total of 96 KB, 64 KB can be shared and 32 KB can be L1, or vice versa. The L2 cache size also doubles from 3 MB in Pascal to 6 MB to help feeding the increase number of computing cores.

<p align="center" width="100%">
    <img width="50%" src="/ece570page/assets/img/arch/memory.png" > 
</p>

{:.image-caption}
*TU102 memory hierarchy. Source: Nvidia*

With Turing, Nvidia was also the first manufacturer to introduce GDDR6 on a consumers' device. This new memory delivers a signaling rate of 14 Gbps, a 27% increase from last generation, also with 20% better power efficiency.

<p align="center" width="100%">
    <img width="50%" src="/ece570page/assets/img/arch/eyediagram.png" > 
</p>

{:.image-caption}
*Memory signal eye diagram. Source: Nvidia*

Finally, TU102 also introduces new memory compression algorithm that  reduces the amount of data written out to memory and transferred from memory to the L2 cache, and reduces the amount of data transferred between clients (such as the texture unit) and the frame buffer. This compression is based on different characteristics of the data, effectively this feature translates to 50% increase in effective bandwidth compared to Pascal.

<p align="center" width="100%">
    <img width="100%" src="/ece570page/assets/img/arch/memorycomp.png" > 
</p>

{:.image-caption}
*Traffic reduction improvement. Source: Nvidia*

<!-- TOC --><a name="rt-cores-for-ray-tracing"></a>
### RT cores for Ray tracing
To better appreciate the hardware acceleration of ray tracing, let us first recall a general ray tracing pipeline, which was briefly mentioned in the Background section. Assuming the BVH structured is already available, generally speaking, the ray tracing pipeline consists of:
- Ray generation/casting: A ray is shoot from a camera perspective, this is usually done by shaders located inside SMs.
- Ray trace: A ray is then traced by traversing the BVH structure. From the API, this function is named TraceRay().
- Target hit or miss: whenever a ray hits something, the information on textures, materials, etc. are queried to perform the rendering. What's interesting is that at this stage, the TraceRay() functionality can be called recursively, this effectively changes the number of bounces each ray has. This features give developer controls over the tradeoff between image quality and performance.

BVH structure traversal is the most expensive stage of the pipeline. Traditionally this is done by software and is hence very slow. RT cores are specialized hardware whose purpose is to accelerate this traversal process. Thus, we can think of RT cores as another kind of functional units whose job is solely to traverse the BVH structure as fast as possible.

<p align="center" width="100%">
    <img width="50%" src="/ece570page/assets/img/arch/raytrace_01-625x630.png" > 
</p>

{:.image-caption}
*Ray tracing pipeline. Source: Nvidia*

<!-- TOC --><a name="acceleration-of-bvh-traversal"></a>
#### Acceleration of BVH traversal
In order to mitigate the bottleneck in this step. Turing architecture introduced an efficient data structure for spatial searching, simply named *Acceleration Structure* (AS), this refers to a paper in 1983 by Fujimoto "Accelerated Ray Tracing". Unlike traditional vertex data used for rasterization, there is no standard AS layout suitable for all implementations/scenes. Thus, AS format is rather opaque, and it is up to the driver at runtime to determine the exact details. This step is hidden inside Nvidia's proprietary driver, so we do not have much information. But we do know some properties, AS has:
- Opaque geometry format optimized for ray traversal (e.g. BVH)
- Layout determined by driver and hardware
- Built at runtime on the GPU
- Immutable except for incremental in-place updates

Using this, every objects in the scene will be represented by two levels of acceleration structures. *Bottom-level* are built from geometric primitives like triangles and rectangles. These primitives are constructed from vertices and index buffers. *Top-level* accleration structures are in turn built from references to bottle-level structures, along with these points, top-level AS also contains transformation matrix to position in in the scene, and an offset to locate information in the shader table.

<p align="center" width="100%">
    <img width="70%" src="/ece570page/assets/img/arch/as.png" > 
</p>

{:.image-caption}
*Acceleration structures. Source: Nvidia*

Utilizing this acceleration structures, the RT core, upon receiving a ray probe from the SM, proceeds to automously traverse the BVH and perform ray-intersection test. This is a fixed-function, and thus RT core is implemented as an ASIC, this circuit functionality is to tell if a ray intersects a geometric primitive (rectangle or triangle) or not. It will then return any hits to the SM and let the shaders implement the result.

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/arch/control.png" > 
</p>

{:.image-caption}
*Performance of Ray Tracing in Control. Source: Digital Foundry*

Even with these software optimizations and dedicated hardware (RT cores account for 25% sillicon budget of Turing), the performance when using ray tracing is still concerning. At release, the RTX 2080 Ti showed a near 50% percent drop when using Ray Tracing, this motivated Nvidia to introduce another trick: Deep Learning Super Sampling (DLSS).
<!-- TOC --><a name="acceleration-by-upscaling"></a>
#### Acceleration by upscaling - DLSS
Recall that in order to perform ray tracing, we need to shoot a ray from the camera through each pixel of the viewport. In practice, doing this just once will create a very noisy image of the scene, multiple rays need to be traced for each pixel in order to reduce the noise. This introduces even more computation overhead. To combat this, we can perform ray tracing at a lower resolution, which requires smaller total number of rays, and then upscale the resulting image back to the original resolution. While traditional methods for upscaling like Temporal Anti-Aliasing (TAA) and Contrast Adaptive Sharpening (CAS) can and have been used for similar tasks, their results still leave much to be desired, they often either introduce a lot of ghosting/blurriness or oversharpen the image, which heavily affects the final result.

<p align="center" width="100%">
    <img width="70%" src="/ece570page/assets/img/arch/generativeai.png" > 
</p>

{:.image-caption}
*Generative models. Source: Lilian Weng*

With the advance of deep learning, especially generative modelling in the past decade, instead of simply using TAA or CAS, Nvidia decided to use a neural network (NN) to upscale the image. The low-resolution output of the SM will be fed into this NN to get a higher-resolution result, which will then be sent to the buffers and displayed. Note that in order to achieve 60 frames per second (FPS), the GPU only has at most 16.6ms to process each frame, and the DLSS can only take up a small fraction of that. Thankfully, since neural network, at least in the inference phase, is just a lot of matrix multiplications, this can be greatly accelerated by tensor cores (introduced earlier). With the RTX 2080Ti and DLSS2.0, it takes less than 1ms to upscale to a 1440p frame (2560x1440 pixels).

<p align="center" width="100%">
    <img width="100%" src="/ece570page/assets/img/arch/dlsstime.png" > 
</p>

{:.image-caption}
*Upscaling latency DLSS 2.0. Source: Nvidia*

Recently, Nvidia also introduce a new technique called Ray Reconstruction in DLSS3.5, this is built specifically to upscale ray traced images. This produces much better image quality compared to DLSS2, but it also requires new hardware functionalities limited to later generations, so we will not discuss it more in this paper.

<!-- TOC --><a name="an-example"></a>
#### An example

<p align="center" width="100%">
    <img width="75%" src="/ece570page/assets/img/blender/ece570_viewport.png" > 
</p>

{:.image-caption}
*Simple scene configuration in Blender*

To quickly demonstrate the performance of ray tracing on GPU. We created a simple scene in Blender with some translucent and reflective objects. We then rendered this scene using pure rasterization and path tracer (a more intensive and much more accurate version of ray tracing). The result (run on AMD 3900X + Nvidia RTX 3080) is as follow:

 <table style="margin: 0px auto;">
  <tr>
    <th></th>
    <th>CPU</th>
    <th>GPU</th>
  </tr>
  <tr>
    <td>Rasterization</td>
    <td>-</td>
    <td>0.51s</td>
  </tr>
  <tr>
    <td>RT 32 rays/pixel</td>
    <td>13.89s</td>
    <td>6.9s</td>
  </tr>
  <tr>
    <td>RT 256 rays/pixel</td>
    <td>100.43s</td>
    <td>29.68s</td>
  </tr>
  <tr>
    <td>RT 512 rays/pixel</td>
    <td>198.7s</td>
    <td>55.9s</td>
  </tr>
  <tr>
    <td>RT 1024 rays/pixel</td>
    <td>442.4s</td>
    <td>112.7s</td>
  </tr>
</table> 

{:.image-caption}
*Table 1: Performance of simple ray traced scene.*

<div class="juxtapose">
    <img src="/ece570page/assets/img/blender/ece570_rasterization.png" />
    <img src="/ece570page/assets/img/blender/ece570_1024samples.png" />
</div>


{:.image-caption}
*Rasterization vs. Path tracing resulting images.*

We can see that as the number of rays increases, the gap between using CPU and GPU gets more significant. The resulting of ray tracing approach, as expected, is much more realistic than that of rasterization. We want to note that this experiment is not truly representative of what Nvidia do, since their approach is hidden within the driver. Furthermore, for real time applications such as video games, the number of rays is usually set at 2 or 4, this helps speed up the performance but will result in a much noisier image (demonstrated below). For this reason, a denoiser step is usually utilized after the ray tracing stage. It helps remove the noise from the final image, the better the denoiser, the smaller number of rays per pixel we need to use, thus making this an important step in the pipeline.

<div class="juxtapose">
    <img src="/ece570page/assets/img/blender/ece570_16samples.png" />
    <img src="/ece570page/assets/img/blender/ece570_1024samples.png" />
</div>
<script src="https://cdn.knightlab.com/libs/juxtapose/latest/js/juxtapose.min.js"></script>
<link rel="stylesheet" href="https://cdn.knightlab.com/libs/juxtapose/latest/css/juxtapose.css">

{:.image-caption}
*16 rays/pixel vs. 1024 rays/pixel.*

The computing process can be summarized by the figure below. The steps in blue and yellow boxes are greatly accelerated by hardware introduced in this paper (RT cores and Tensor cores). We note that the stages in yellow boxes are not inherent to the ray tracing pipeline, they are more like optimization tricks and can be skipped if one desires. In fact, we did not employ these steps in our example above. This example also concludes our discussion on accelerating ray tracing.

<p align="center" width="100%">
    <img width="100%" src="/ece570page/assets/img/blender/diagram.png" > 
</p>

{:.image-caption}
*Ray tracing computing process*

<!-- TOC --><a name="conclusion"></a>
## Conclusion
In this short paper, we have reviewed the basics of ray tracing, Nvidia Turing architecture, as well as how it accelerates the calculation of ray tracing. In the last 5 years, the industry has made great strides in real time ray tracing rendering, and the results have been impressive. That being said, we note that the current performance, even with all the tricks and optimizations, still fall far below what rasterization can produce (in our example, 0.5s compared to 112.7s, more than 200x faster). In our opinions, this means there are still much work to be done with this alternative rendering approach. We expect better results coming from the Denoisers and Upscalers, since the advancement in generative modeling has been unprecedented, integrated these new models into the pipeline is not a trivial task, but it is definitely an exciting one. We look forward to seeing what the industry will bring to this problem in the next few years.

<!-- TOC --><a name="some-references"></a>
## References 
- Kay, T. L., & Kajiya, J. T. (1986). Ray tracing complex scenes. *ACM SIGGRAPH computer graphics*, 20(4), 269-278.
- Fujimoto, A., & Iwata, K. (1985). Accelerated ray tracing. In *Computer Graphics: Visual Technology and Art Proceedings of Computer Graphics Tokyo′ 85* (pp. 41-65). Tokyo: Springer Japan.
- Nah, J. H., Park, J. S., Park, C., Kim, J. W., Jung, Y. H., Park, W. C., & Han, T. D. (2011, December). T&I engine: Traversal and intersection engine for hardware accelerated ray tracing. In *Proceedings of the 2011 SIGGRAPH Asia Conference* (pp. 1-10).
- Karras, T., & Aila, T. (2013, July). Fast parallel construction of high-quality bounding volume hierarchies. In *Proceedings of the 5th High-Performance Graphics Conference* (pp. 89-99).
- [DLSS 2.0 - Image Reconstruction for Real-time Rendering with Deep Learning](https://developer.nvidia.com/gtc/2020/video/s22698-vid)
- [Ray Reconstruction in DLSS 3.5](https://www.nvidia.com/en-us/geforce/news/nvidia-dlss-3-5-ray-reconstruction/)
- [Nvidia Turing white paper](https://images.nvidia.com/aem-dam/en-zz/Solutions/design-visualization/technologies/turing-architecture/NVIDIA-Turing-Architecture-Whitepaper.pdf)
- [Nvidia Ampere white paper](https://www.nvidia.com/content/PDF/nvidia-ampere-ga-102-gpu-architecture-whitepaper-v2.pdf)
- [A history of NVIDA GPU architecture](https://fabiensanglard.net/cuda/)
- [Nvidia Developers blog](https://developer.nvidia.com/blog)
- [Microsoft's DirectX12](https://devblogs.microsoft.com/directx/announcing-microsoft-directx-raytracing/)
        