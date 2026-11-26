# Counting objects and timing the GPU

Two tools to see what a program holds on the GPU and how long the GPU spends on a frame.
[scene's profiling guide](https://github.com/eskiu-packages/scene/blob/develop/docs/profiling.md)
builds per-stage frame statistics on them.

## Live objects

`gpu_counts()` returns a `GpuCounts`: the GPU objects made through this package that are
still alive, in the whole process.

| Field | What it counts |
|---|---|
| `buffers`, `buffer_bytes` | `GpuBuffer`s (`Gpu_buffer`, `storage`, `uniform`, `vertices`, `indices`) and read-back staging buffers, with their sizes |
| `textures`, `texture_bytes` | `GpuTexture`s, with `GpuTexture_bytes`: every mip level of every layer, times the sample count |
| `render_pipelines` | `GpuPipeline`s |
| `compute_pipelines` | `GpuKernel`s and `NnProgram` pipelines |
| `bind_groups` | WebGPU bind groups alive: a `GpuBindGroup` or a pipeline's group 0 once used in a pass (a group replaced mid-pass is released when its pass or encoder is submitted) |
| `samplers` | `GpuSampler`s |

Counting is two additions per object made or released; it is always on. The numbers are
not atomic, so read them on the thread that makes and releases the objects.

To find a leak, read the counts at a quiet point (between frames), do the thing that should
leave nothing behind (open and close a screen, load and unload a level) several times, and
read them again: what is still higher grows with each round. Caches that fill once (a
pipeline per shader variant) rise on the first round and stay flat after it, so compare the
second reading with the last one, not with the very first.

```eskiu
GpuCounts before = gpu_counts();
for (i in 0..10) enter_and_leave(&gpu)?;
GpuCounts after = gpu_counts();
if (after.textures > before.textures) printf("textures grew by %d\n", after.textures - before.textures);
```

The bytes are what the objects ask for, not what the driver allocates: drivers round
sizes up, compress some textures and keep their own pools, so a memory tool (Xcode's
Memory Graph or the Metal resource list in a GPU capture) shows somewhat more. Textures
made outside wgpu (`Gpu_texture_from_metal`, `Gpu_texture_from_vulkan`) count with the bytes
their size and format would take, although their memory belongs to whoever made them; the
textures of a window surface's frames are not counted.

## GPU time

When the adapter has timestamp queries, `gpu_open` turns them on and sets `Gpu.timestamps`.
wgpu-native has them on Metal, Vulkan and DirectX 12; a browser's WebGPU may expose them (Chrome
rounds them to 100 microseconds unless its flags say otherwise); the WebGL2 backend has none.
`Gpu_timer` fails when they are missing: show the GPU time as unavailable then.

```eskiu
GpuTimer timer = gpu.timer(16)?;          // frames of up to 16 render passes
defer timer.release();
...each frame:
GpuEncoder enc = gpu.encoder();
enc.time(&timer);                          // this encoder's passes are timed
... passes ...
enc.submit()?;
if (timer.last_frame >= 0) printf("GPU %.2f ms\n", timer.last_ms);
```

The GPU writes its clock at the start and the end of each render pass of the encoder.
`last_ms` is from the start of the first pass to the end of the last, `pass_ms` holds each
pass. Results come back through three read-back buffers, so nothing waits for the GPU: they
arrive one to three frames later (`last_frame` says which frame), and `GpuEncoder_time`
skips a frame (`dropped`) when all three are still on their way. Work outside the encoder's
render passes (copies, compute, a `Gpu_pass` of its own) is not included, and neither is
the time between passes or the compositor's work after present.

Cost: a 64-pass timer holds a query set and four buffers of 1 KB; each timed frame adds one
resolve and one copy of 16 bytes per pass and a buffer map.
scene's guide has measurements of the cost.

`tests/counts_timer.esk` checks both.
