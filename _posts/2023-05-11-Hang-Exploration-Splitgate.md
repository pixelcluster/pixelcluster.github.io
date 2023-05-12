---
title: "GPU Hang Exploration: Splitgate"
layout: "post"
---

GPU hangs are one of the most common results of pretty much anything going
wrong GPU-side, and finding out why they occur isn't always easy. In this blog
post, I'll document my journey towards finding the cause of one specific
hang in the game "Splitgate".


Right off the bat, I noticed a few oddities with this particular hang.
Firstly, the actual game always ran completely fine. The only place where
it hung was on the first startup where the game automatically configures
graphics settings (I'll call it autoconfiguration from here).

Additionally, while I could reproduce the hang on my Steam Deck, I couldn't
get the hang to appear on my desktop. I have an RDNA2 graphics card, which
is the same architecture as the Deck, so it seemed unlikely that specifics
about the hardware architecture were the problem here.

## API Validation

As a first step, I tried running the game with the Vulkan Validation Layers. If
the game is using the API in an invalid way and that is the cause of the hangs, there's
a rather good chance the Validation Layers will catch it.

Even though there were a few errors from the validation layers,
it seemed like none of the errors were actually relevant to the hang. 
Most importantly, the errors with autoconfiguration on were the same as the
errors with autoconfiguration off.

As any software, the Validation Layers aren't perfect and can't detect every
possible invalid behaviour. At this point I was still unsure whether I'd have to
search for the bug on the application side or on the driver side.

## API dumping

With the validation layers being unable to detect any invalid behaviour by the app
during the autoconfiguration phase, another question comes to mind:
What *is* the application doing, actually?

To answer that, I utilized the API Dump Vulkan layer by LunarG. When this layer is
activated, it dumps all the commands made by the application, including every parameter
and return value to standard output.

While API dumps are good to have for debugging, large API dumps from large engines
are often difficult to navigate (not just because it's an 800MB large file and
your text editor dies trying to scroll through them). Instead, it's often best to
extract just the work that hangs for further debugging. But what frame is this?

#### Finding the hanging submission

The CPU and GPU do work asynchronously, which means that the CPU is free to
do more work while GPU starts with its work. Somewhat unfortunately, this also
means the CPU can do more Vulkan calls which will show up in the API dump after
the app already submitted the hanging frame to the GPU. This means that I
couldn't just look at the last command in the API dump and assume that command
caused the hang. Luckily, there were other hints towards what caused the hang.

In Vulkan, when you want to know when a particular work submission finishes,
you give a `VkFence` to the submit function. Later, you can wait for
the submission to finish with `vkWaitForFences`, or you can query whether
the submission has already finished with `vkGetFenceStatus`.

I noticed that after work was submitted, the app seemed to call
`vkGetFenceStatus` from time to time, polling whether that submission was
finished. Usually, `vkGetFenceStatus` would return `VK_SUCCESS` after a
few calls, indicating that the submission finished. However, there was one
submission where `vkGetFenceStatus` seemed to always return `VK_NOT_READY`.
It seemed very likely that the GPU was hanging while executing that submission.

To test my theory, I modified the implementation of `vkQueueSubmit`, which
you call for submitting work, to call `vkDeviceWaitIdle` immediately after
submitting the work. `vkDeviceWaitIdle` waits for *all* outstanding GPU work to
finish. When the GPU hangs, the `vkQueueSubmit` which caused the hang should
be the last line in the API dump[^1].

This time, the API dump cut off at the `vkQueueSubmit` for which
`vkGetFenceStatus` always returned `VK_NOT_READY`.

Bingo.

{:refdef: style="text-align: center;"}
![mr incredible](/assets/memes/uncanny-mr-incredible-0.png){: width="250" }
{: refdef}

## Going lower-level

Now we know which submission hangs, but that submission still contains a lot of
commands. Even though the text editor now survives scrolling through the
commands, finding what is wrong by inspection is highly unlikely.

Instead, I tried to answer the question: "What specific command is making the
GPU hang?"

In order to find the answer, I needed to find out as much as possible about
what state the GPU is in when it hangs. There are a few useful tools which
helped me gather info:

#### umr

The first thing I did was use [umr](https://gitlab.freedesktop.org/tomstdenis/umr)
to query if any waves were active at the time of the hang. Waves (or Wavefronts)
are groups of 32 or 64 shader invocations (or threads in DX terms) that the GPU
executes at the same time[^2]. There were indeed quite a few waves currently
executing. For each wave, umr can show a disassembly of the GPU code that is
currently executing, as well as the values of all registers, and more.

In this case, I was especially interested in the `halt` and `fatal_halt` status
bits for each wave. These bits are set when the wave encounters a fatal
exception (for example dereferencing invalid pointers) and won't continue
execution. These bits were not set for any waves I inspected, so it was
unlikely that exceptions in a shader were causing the hang.

Aside from exceptions, the other common way for shaders to trigger GPU hangs is
by accidentally executing infinite loops. But the shader code currently executing
was very simple and didn't even have a jump instruction anywhere, so the hang
couldn't be caused by infinite loops either.

#### RADV\_DEBUG=hang

Shaders aren't the only thing that the GPU executes, and as such shaders aren't
the only thing that can cause GPU hangs.

In RADV, command buffers recorded in Vulkan are translated to a
hardware-specific command buffer format called `PKT3`[^3]. Commands encoded
in this format are written to GPU-accessible memory, and executed by the
GPU's command processor (CP for short) when the command buffer is submitted.

These commands might also be involved in the hang, so I tried finding out which
commands the CP was executing when the hang happened. RADV has integrated debug
functionality that can help with exactly this, which can be enabled by setting
an environment variable named `RADV_DEBUG` to `"hang"`. But when I tried
triggering the hang with this environment variable in place, it started up just
fine!

This isn't the first time I've seen this. `RADV_DEBUG=hang` has a funny side
effect: It also inserts commands to wait for draws or dispatches to complete
immediately after the dispatch is triggered. This immensely helps with
figuring out which shader is faulty if there are multiple shaders executing
concurrently. But it also prevents certain hangs from happening: Where things
executing concurrently *causes* the hang in the first place.

In other words, we seem to be looking at a synchronization issue.

{:refdef: style="text-align: center;"}
![uncanny mr incredible](/assets/memes/uncanny-mr-incredible-1.png){: height="250" }
{: refdef}

## Synchronization boogaloo

Even though we know we're dealing with a synchronization issue, the original
question remains unsolved: What command causes the hang?

The "sync after every draw/dispatch" method of `RADV_DEBUG=hang` fixes the
issue, but it has a very broad effect. Since the issue seems to reproduce
very reliably (which in itself is a rarity for synchronization bugs), we
can apply that sync selectively to only some draws or dispatches to narrow
down what commands exactly cause the hangs.

First, I tried restricting the synchronization to only apply to dispatches
(so no draws were synchronized). This made the hang appear again. Testing
the other way around (restricting the synchronization to only draws) confirmed:
All compute dispatches were fine, the issue was about draw synchronization
only.

Next, I tried only synchronizing at the end of renderpasses. This also fixed
the hang. However, synchronizing at the start of renderpasses fixed nothing.
Therefore it was impossible that missing synchronization across renderpasses
was the cause of the hang.

The last likely option was that there was missing synchronization in between
the draws and something in between renderpasses.

At this point, the API dump of the hanging submission proved very helpful.
Upon taking a closer look, it became clear that the commands in the submitted
command buffer had a very simple pattern (some irrelevant commands omitted for brevity):

1. `vkCmdBeginRenderPass` to begin a new renderpass
2. `vkCmdDraw`
3. `vkCmdEndRenderPass`, ending the renderpass
5. `vkCmdWriteTimestamp`, writing the current elapsed time

What stuck out to me was that `vkCmdWriteTimestamp` was called with a
`pipelineStage` of `VK_PIPELINE_STAGE_TOP_OF_PIPE`. In simple terms, this means
that the timestamp can be written before the preceding draw finished.[^4]

Further testing confirmed: If I insert synchronization before writing the
timestamp, the hang is fixed. Inserting synchronization immediately
after writing the timestamp makes the hang re-appear.

## How hard can writing a timestamp be?

By now, it has become pretty clear that timestamp queries are the problem here.
But it just didn't really make sense that the timestamp write itself would
hang.

Timestamp writes on AMD hardware don't require launching any shaders.
They can be implemented using one PKT3 command called `COPY_DATA`[^5], which
accepts many data sources other than memory. One of these data sources is the
current timestamp. RADV uses `COPY_DATA` to write the timestamp to memory.
The memory for these timestamps is managed by the driver, so it's exceedingly
unlikely the memory write would fail.

From the wave analysis with umr earlier I also knew that the in-flight shaders
didn't actually write or read any memory that might interfere with the
timestamp write (somehow). The timestamp write itself being the cause of the
hang seemed impossible.

## Taking a step back

If timestamp writes can't be the problem, what else can there be that might
hang the GPU?

There is one other part to timestamp queries aside from writing the timestamp
itself: In Vulkan, timestamps are always written to opaque "query pool"
objects. In order to actually view the timestamp value, an app has to copy the
results stored in the query pool to a buffer in CPU or GPU memory. Splitgate
uses Unreal Engine 4, which has a known bug related to query pool copies that
RADV has to [work around](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_query.c#L1555-1559).

It isn't too far-fetched to think there might be other bugs in UE's Vulkan
RHI regarding query copies. Synchronizing the query copy didn't do anything,
but just commenting out the query copy fixed the hang as well.

{:refdef: style="text-align: center;"}
![uncannier mr incredible](/assets/memes/uncanny-mr-incredible-2.png){: height="250" }
{: refdef}

## ????

Up until this point, I was pretty sure that something about the timestamp write
must be the cause of the problems. Now it seemed like query copies might also
influence the problem somehow? I was pretty unsure how to reconcile these two
observations, so I tried finding out more about how exactly the query copy
affected things.

Query copies on RADV are implemented using small compute shaders written
directly in [NIR](https://docs.mesa3d.org/nir/index.html). Having the simple
driver-internal shaders in NIR is a nice and simple way of storing them inside
the driver, but they're a bit hard to read for people not used to the syntax.
For demonstration purposes I'll use a GLSL translation of the shader[^6].
The copy shader for timestamp queries looks like this:

```c
location(binding = 0) buffer dst_buf;
location(binding = 1) buffer src_buf;

void main() {
    uint32_t result_size = flags & VK_QUERY_RESULT_64_BIT ? sizeof(uint64_t) : sizeof(uint32_t);
    uint32_t dst_stride = result_size;
    if (flags & VK_QUERY_RESULT_WITH_AVAILABILITY_BIT)
        dst_stride += sizeof(uint32_t);
    uint32_t src_stride = 8;

    uint64_t result = 0;
    bool available = false;
    uint64_t src_offset = src_stride * global_id.x;
    uint64_t dst_offset = dst_stride * global_id.x;
    uint64_t timestamp = src_buf[src_offset];
    if (timestamp != TIMESTAMP_NOT_READY) {
        result = timestamp;
        available = true;
    }
    if ((flags & VK_QUERY_RESULT_PARTIAL_BIT) || available) {
        if (flags & VK_QUERY_RESULT_64_BIT) {
            dst_buf[dst_offset] = result;
        } else {
            dst_buf[dst_offset] = (uint32_t)result;
        }
    }
    if (flags & VK_QUERY_RESULT_WITH_AVAILABILITY_BIT) {
        dst_buf[dst_offset + result_size] = available;
    }
}
```

At first, I tried commenting out the stores to `dst_buf`, which resulted in the
hangs disappearing again. This can indicate that `dst_buf` is the problem, but
it's not the only possibility. The compiler can also optimize out the load
because it isn't used further down in the shader, so this could also mask
an invalid read as well. When I commented out the read and always stored a
constant instead - it also didn't hang!

But could it be that the shader was reading from an invalid address? Splitgate
is by far not the only app out there using timestamp queries, and those apps
all work fine - so it can't just be fundamentally broken, right?

To test this out, I modified the timestamp write command once again. Remember
how `PKT3_COPY_DATA` is really versatile? Aside from copying memory and
timestamps, it can also copy a 32/64-bit constant supplied as a parameter.
I undid all the modifications to the copy shader and forced a constant to be
written instead of timestamps. No hangs to be seen.

{:refdef: style="text-align: center;"}
![even uncannier mr incredible](/assets/memes/uncanny-mr-incredible-3.png){: height="250" }
{: refdef}

## ?????????

It seems like aside from the synchronization, the value that is written as the
timestamp influences whether a hang happens or not. But that also means neither
of the two things already investigated can actually be the source of the hang,
can they?

It's essentially the same question as in the beginning, still unanswered:\
"What the heck is hanging here???"

### RADV\_DEBUG=hang (but useful this time)

Stabbing in the dark with more guesses won't help here. The only thing that can
is more info. I already had a small GPU buffer that I used for some other
debugging I skipped over. To get definitive info on whether it hangs because of
the timestamp write, the timestamp copy, or something else entirely, I modified
the command buffer recording to write some magic numbers into that debug buffer
whenever these operations happened. It went something along the lines of:
- write `0xAAAAAAAA` if timestamp write is complete
- write `0xBBBBBBBB` if timestamp copy is complete

However, I still needed to ensure I only read the magic numbers after the
GPU had time to execute them (without waiting forever during GPU hangs)..
This required a different intricate and elaborate synchronization algorithm.

```c
// VERY COMPLICATED SYNCHRONIZATION
sleep(1);
```

With that out of the way, let's take a look at the magic number of the hanging
submission.
```
Magic: 0x0
```

*what???* this means *neither write nor copy* have executed? Alright, what if
I add another command writing a magic number right at the beginning of the
command buffer?

```
Magic: 0x0
```

So... the hang happens before the command buffer starts executing? Something
can't be right here.[^7]

{:refdef: style="text-align: center;"}
![even more uncannier mr incredible](/assets/memes/uncanny-mr-incredible-4.png){: height="250" }
{: refdef}

At this point I started logging all submits that contained either timestamp
writes or timestamp copies, and I noticed that there was another submission
with the same pattern of commands right before the hanging one. 

## Multi-submit madness

This previous submission had executed just fine - all timestamps were written,
all shaders finished without hangs. This meant that neither the way timestamps
were written nor the way they were copied could be direct causes of hangs,
because they worked just one submission prior.

I verified this theory by forcing full shader synchronization to happen before
the timestamp write, but only for the submission that actually hangs. To my
surprise, this did nothing to fix the hangs.

When I applied the synchronization trick to the previous submit (that always
worked fine!), the hangs stopped appearing.

It seems like the cause of the hang is not in the hanging submission, but in a
completely separate one that completed successfully.

{:refdef: style="text-align: center;"}
![most uncanny mr incredible](/assets/memes/uncanny-mr-incredible-5.png){: height="250" }
{: refdef}

## What is the app doing?

Let's rewind to the question that started this whole mess. "What is the app
doing?"

Splitgate (as of today) uses Unreal Engine 4.27.2. Luckily, Epic Games make the
source code of UE available to anyone registering for it with their Epic Games
account. There was hope that the benchmark code they were using was built into
Unreal, where I could examine what exactly it does.

Searching in the game logs from a run with the workaround enabled, I found this:
```
LogSynthBenchmark: Display: Graphics:
LogSynthBenchmark: Display:   Adapter Name: 'AMD Custom GPU 0405 (RADV VANGOGH)'
LogSynthBenchmark: Display:   (On Optimus the name might be wrong, memory should be ok)
LogSynthBenchmark: Display:   Vendor Id: 0x1002
LogSynthBenchmark: Display:   Device Id: 0x163F
LogSynthBenchmark: Display:   Device Revision: 0x0
LogSynthBenchmark: Display:   GPU first test: 0.06s
LogSynthBenchmark: Display:          ... 3.519 s/GigaPix, Confidence=100% 'ALUHeavyNoise' (likely to be very inaccurate)
LogSynthBenchmark: Display:          ... 2.804 s/GigaPix, Confidence=100% 'TexHeavy' (likely to be very inaccurate)
LogSynthBenchmark: Display:          ... 2.487 s/GigaPix, Confidence=100% 'DepTexHeavy' (likely to be very inaccurate)
LogSynthBenchmark: Display:          ... 8.917 s/GigaPix, Confidence=100% 'FillOnly' (likely to be very inaccurate)
LogSynthBenchmark: Display:          ... 0.330 s/GigaPix, Confidence=100% 'Bandwidth' (likely to be very inaccurate)
LogSynthBenchmark: Display:          ... 0.951 s/GigaVert, Confidence=100% 'VertThroughPut1' (likely to be very inaccurate)
LogSynthBenchmark: Display:          ... 6.053 s/GigaVert, Confidence=100% 'VertThroughPut2' (likely to be very inaccurate)
LogSynthBenchmark: Display:   GPU second test: 0.54s
LogSynthBenchmark: Display:          ... 4.186 s/GigaPix, Confidence=100% 'ALUHeavyNoise' (likely to be inaccurate)
LogSynthBenchmark: Display:          ... 3.118 s/GigaPix, Confidence=100% 'TexHeavy' (likely to be inaccurate)
LogSynthBenchmark: Display:          ... 2.844 s/GigaPix, Confidence=100% 'DepTexHeavy' (likely to be inaccurate)
LogSynthBenchmark: Display:          ... 9.127 s/GigaPix, Confidence=100% 'FillOnly' (likely to be inaccurate)
LogSynthBenchmark: Display:          ... 0.339 s/GigaPix, Confidence=100% 'Bandwidth' (likely to be inaccurate)
LogSynthBenchmark: Display:          ... 0.983 s/GigaVert, Confidence=100% 'VertThroughPut1' (likely to be inaccurate)
LogSynthBenchmark: Display:          ... 6.422 s/GigaVert, Confidence=100% 'VertThroughPut2' (likely to be inaccurate)
LogSynthBenchmark: Display:   GPU Final Results:
LogSynthBenchmark: Display:          ... 4.186 s/GigaPix, Confidence=100% 'ALUHeavyNoise'
LogSynthBenchmark: Display:          ... 3.118 s/GigaPix, Confidence=100% 'TexHeavy'
LogSynthBenchmark: Display:          ... 2.844 s/GigaPix, Confidence=100% 'DepTexHeavy'
LogSynthBenchmark: Display:          ... 9.127 s/GigaPix, Confidence=100% 'FillOnly'
LogSynthBenchmark: Display:          ... 0.339 s/GigaPix, Confidence=100% 'Bandwidth'
LogSynthBenchmark: Display:          ... 0.983 s/GigaVert, Confidence=100% 'VertThroughPut1'
LogSynthBenchmark: Display:          ... 6.422 s/GigaVert, Confidence=100% 'VertThroughPut2'
```

`FSynthBenchmark` indeed appears in the UE codebase as a
benchmark tool to auto-calibrate settings. From reading its code, it seemed
like it does 3 separate benchmark ru...

wait. 3?? 

We can clearly see from the logs there are only two benchmark runs. Maybe the
third run hangs the GPU somehow?

## Hang? Well yes, but actually no

While thinking about this, another possibility came to my mind. The GPU driver
can't actually detect if the GPU is hung because of some fatal error or
if it just takes an obscenely long amount of time for some work. No matter what
it is, if it isn't finished in 10 seconds, the GPU will be reset.[^8]

So what if the hang I've been chasing all this time isn't actually a hang? How
do I even find out?

The amdgpu kernel driver has a parameter named `lockup_timeout` for this exact
purpose: You can modify this parameter to change the amount of time after which
the GPU is reset if a job doesn't finish, or disable this GPU reset entirely.
To test this theory, I went with disabling the GPU reset.

After setting all the parameters up and rebooting, I started the game another
time.

And it worked! It took a really long time, but eventually, the game started
up fully. It was indeed just hammering the poor Deck's GPU with work that
took way too long.

## Why does my workaround work?

Finally, things start clearing up a bit. There is still an open question,
though: What does the workaround do to prevent this?

The code that runs the 3 benchmark passes doesn't always run them
unconditionally. Instead, the 3 benchmarks have an increasingly larger
workload (each roughly 10x as much as the previous one). Comments nearby
explain that this choice was made because the larger benchmark runs cause
driver resets on low-end APUs (hey, that's exactly the problem we're
having!). It measures the time it takes for the benchmark workloads to
complete using the timestamp queries, and if the total benchmark time is beyond
a certain point, it skips the other benchmark runs.

If you've been paying extremely close attention all the way until here, you
might notice a small problem. UE4 interprets the timestamp values as the
time until the benchmark workload completes. But as I pointed out all the way
near the beginning, the timestamp can be written before the benchmark workload
is even finished!

If the timestamp is written before the benchmark workload finishes, the
measured benchmark time is much less than the workload actually took.
In practice, this results in the benchmark results indicating a much faster GPU
than there actually is. I assume this led to the third benchmark (which was too
heavy for the Deck GPU) to be launched. My desktop GPU seems to be powerful
enough to get through the benchmark before the lockup timeout, which is why
I couldn't reproduce the issue there.

In the end, the hack I originally found to work around the issue turned out
to be a
[fitting workaround](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/22823).

And I even got to make my first bugreport for Unreal Engine.

{:refdef: style="text-align: center;"}
![canny mr incredible](/assets/memes/canny-mr-incredible.png){: width="250" }
{: refdef}


## Footnotes

[^1]: If an app is using Vulkan on multiple threads, this might not always be the case. This is a rare case where I'm grateful for Unreal Engine to have a single RHI thread.
[^2]: Nvidia also calls them "warps".
[^3]: Short for "Packet 3". Packet 2, 1 and 0 also exist, although they aren't widely used on newer AMD hardware.
[^4]: If you insert certain pipeline barriers, writing the timestamp early would be disallowed, but these barriers weren't there in this case. 
[^5]: This command writes the timestamp immediately when the CP executes it. There is another command which waits for previous commands to finish before writing the timestamp.
[^6]: You can also view the original NIR and the GLSL translation [here](https://gitlab.freedesktop.org/mesa/mesa/-/blob/0b251d43/src/amd/vulkan/radv_query.c#L531)
[^7]: As it turned out later, the debugging method was flawed. In actuality, both timestamp writes and copies completed successfully, but the writes indicating this seemed to be still in the write cache. Forcing the memory containing the magic number to be uncached solved this.
[^8]: Usually, the kernel driver can also command the GPU to kill whatever job it is doing right now. For some reason, it didn't work here though.
