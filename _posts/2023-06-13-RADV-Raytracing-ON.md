---
title: "RADV Ray Tracing: Now ON by default"
layout: "post"
---

Yes, you heard that right.

Ray Tracing Pipelines.

On RADV.

Enabled by default.

Now merged in Mesa `main`.



This has been in the works for a loooooooooong time. Probably the longest of
any RADV features so far.

But what makes ray tracing pipelines so complex that it takes this long to implement?
Let's take a short look at what it took for RADV to get its implementation off the ground.

## Ray Tracing basics

For the purposes of this blog, ray tracing is the process of finding intersections between rays and some geometry.

Most of the time, this geometry will be made up of lots of triangles. We don't want to test every single triangle for
intersection separately, so Bounding Volume Hierarchies (BVHs) are used to speed up the process by skipping entire
groups of triangles at once.

## Hardware acceleration

Nowadays, GPUs have dedicated hardware to speed up the ray tracing process.

AMD's hardware acceleration for ray tracing is very simple: It consists of a single instruction called `image_bvh_intersect_ray` (and its 64-bit variant).[^1]

Why is it called `image_bvh_intersect_ray`? Because the hardware sees the BVH as a 1D image and uses its memory subsystem for textures to fetch BVH data, of course.

This instruction takes care of calculating intersections between a ray and a single node in the BVH. But intersecting one node isn't good enough:
In order to find actual intersections between the ray and geometry, we need to traverse the BVH and check lots of nodes.
The traversal loop that accomplishes this is implemented in software[^2].

## Ray Tracing Pipelines

In Vulkan, you can use ray tracing pipelines to utilize your GPU's hardware-accelerated ray tracing capabilities. It might not seem like it, but ray tracing pipelines
actually bring a whole lot of new features with them that make them quite complex to implement.

Ray tracing pipelines introduce a set of new shader stages:
- Ray generation shaders calculate origins and directions of rays to trace and call `traceRayEXT` to start tracing
- Any-hit shaders are responsible or confirming or rejecting potential intersections
- Intersection shaders can be used to run custom ray-primitive intersection code, which can be used to do raytracing on non-triangle geometry
- Closest-hit shaders are responsible for handling rays that have hit geometry, calculating things like lighting for the traced ray
- Miss shaders handle the case where no accepted intersections were results (either there were no intersections, or all intersections were rejected).
- Callable shaders can be invoked from the ray generation shader and can do arbitrary calculations, including recursive calls (calling callable shaders from callable shaders)

That's right, as a small side effect, ray tracing pipelines also introduced full proper recursion from shaders. This doesn't just apply to callable shaders:
You can also trace new rays from a closest-hit shader, which can recursively invoke more closest-hit shaders, etc.

Also, ray tracing pipelines introduce a very dynamic, GPU-driven shader dispatch process: In traditional graphics and compute pipelines, once you bind a pipeline,
you know exactly which shaders are going to execute once you do a draw or dispatch. In ray tracing pipelines, this depends on something called the Shader Binding Table,
which is a piece of memory containing so-called "shader handles". These shader handles identify the shader that is *actually* launched when vkCmdTraceRaysKHR is called.

In both graphics and compute pipelines, the concept of pipeline stages was quite simple: You have a bunch of shader stages (for graphics pipelines, it's
usually vertex and fragment, for compute pipelines it's just compute). Each stage has exactly one shader: You don't have one graphics pipeline with
many vertex shaders. In ray tracing pipelines, there are no restrictions on how many shaders can exist for each stage.

In RT pipelines, there is also the concept of shaders dispatching other shaders: Every time `traceRayEXT` is called, more shaders (any-hit, intersection, closest-hit or miss shaders)
are launched.

That's lots of changes just for some ray tracing!

## Hardware limitations

RT pipelines aren't really a fitting representation of AMD hardware. There is no such thing as reading a memory location to determine which shader to launch, and the hardware has
no concept of a callstack to implement recursion. RADV therefore has to do a bit of magic to transform RT pipelines in a way that will actually run.

### Shader stages: All-in-one

The first approach RADV used to implement these ray tracing pipelines was essentially to pretend that the whole ray tracing pipelines a normal compute shader:
All shaders from the pipeline are assigned a unique ID. Then, all shaders are inserted into a humongous chain of `if (idx == shader_id) { (paste shader code here) }` statements.

If you wanted to call a shader, it was as simple as setting `idx` to the ID of the shader you wanted to call. You could even implement recursion by storing the ID of the shader
to return to on a call stack.

Launching shaders according to the shader binding table wasn't a problem either: You just read the shader binding table at the start and set `idx` to whatever value is in there.

But there was a problem.

#### Oh God there's so many of them

As it turns out, if you don't put any restrictions on how many shaders can exist in a stage, there's going to be apps that use LOTS of them. We're talking almost a thousand shaders
in some cases. Ludicrously large code like that resulted in lots of ludicrous results (games spending over half an hour compiling shaders!). Clearly, the megashader solution wasn't
sustainable.

#### Also I forgot an important addition

Ray Tracing Pipelines also add pipeline libraries. You might have heard of them in the context of Graphics Pipeline Libraries, which was also really painful to implement in RADV.

Pipeline libraries essentially allow you to create parts of your ray tracing pipeline beforehand, and then re-use these created parts all over other ray tracing pipelines. But
if we just paste all shaders into one chonker compute shader, we can't compile it yet when creating a pipeline library, because other shaders will be added once a real pipeline
is created from it!

This basically meant that we couldn't do anything but copy the source code around, and start compiling only when the real pipeline is created. It also turned out that it's valid
behaviour to query the stack size used for recursion from pipeline libraries, but because RADV didn't compile any code yet, it didn't even know what stack size the shaders from that
pipeline used.

### Separate shader compilation

This is where separate shader compilation comes in. As the name suggests, most[^3] shaders are compiled independently. Instead of using shader IDs to select what shader is called,
we store the VRAM addresses of the shaders and directly jump to whatever shaders we want to execute next.

Directly jumping to a shader is still impossible because reading the shader binding table is required. Instead, RADV creates a small piece of shader assembly that sets up necessary
parameters, reads the shader binding table, and then directly jumps to the selected shader (like it is done for shader calls).

This allows us to compile shaders immediately when creating pipeline libraries. It also pretty much resolves the problem of chonker compute shaders taking ludicrously long to compile.
It also required basically reworking the entire ray tracing compilation infrastructure, but I think it forms a great basis for future work in the performance area.

## FAQ

### What apps/games does RADV ray tracing run?

Everything runs.

In case you disagree, please [open an issue](https://gitlab.freedesktop.org/mesa/mesa/-/issues).

### How well do ray queries run?

Pretty competitive with AMDVLK/the AMD Windows drivers! You'll generally see similar, if not better, performance on RADV.

### How well do pipelines run?

Not well (expect significantly less performance compared to AMDVLK/Windows drivers). This is being worked on.

## Footnotes

[^1]: RDNA3 introduces another instruction that helps with BVH traversal stack management, but RADV doesn't use it yet.
[^2]: This is also what makes it so easy to support ray tracing even when there is no hardware acceleration (using `RADV_PERFTEST=emulate_rt`): Most of the traversal code can be reused, only `image_bvh_intersect_ray` needs to be replaced with a software equivalent.
[^3]: Any-hit and Intersection shaders are still combined into a single traversal shader. This still shows some of the disadvantages of the combined shader method, but generally compile times aren't that ludicrous anymore.
