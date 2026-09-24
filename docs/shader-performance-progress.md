# Shader Performance Progress

## Requirements and environment

The current shader-performance pass depends on a working host toolchain and Vulkan/Qt runtime stack. The environment has been set up with the following requirements:

- Nix + flakes enabled (`nix-command` + `flakes`)
- `nix develop` dev shell for the repo, with:
  - `clang`, `lld`, `cmake`, `ninja`, `pkg-config`, `git`, `glslang`
  - Vulkan headers/loader/tools/validation layers
  - Qt 6 base stack for launcher support
  - X11, Wayland, libdecor, udev/systemd, ALSA, PulseAudio, DBus integration
- Runtime libraries included in the shell for the built emulator and launcher
- Build validation path uses serial/targeted compilation for very large shader-heavy tests because the full `kyty_tests` aggregate can be killed by host resource limits during compilation of `shader_cfg_tests.cpp`

Recommended working configuration:

```bash
. /home/codespace/.nix-profile/etc/profile.d/nix.sh
cd /workspaces/KytyPS5
nix develop --command bash -c '
  cmake -S . -B _Build/linux -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_MAKE_PROGRAM="$(which ninja)" \
  && cmake --build _Build/linux --target audio_out2_port_tests --target page_manager_tests --target image_page_table_tests -j1
'
```

## Completed

- Removed runtime Vulkan pipeline cache checkpoints; cache saves on shutdown only. The scheduler no longer captures pipeline-cache data at frame/command-buffer boundaries, avoiding driver-side cache capture stalls during gameplay.
- Reused Vulkan pipeline cache across emulator revisions when device/driver/cache UUID match.
- Pipeline-cache signatures no longer include the emulator commit; compatible rebuilds now reuse the same driver cache instead of invalidating it on every workflow build.
- Added persistent SPIR-V cache under `_ShaderCache/<TITLE_ID>/<revision>/`.
- SPIR-V cache keys include stage, shader hash, static state, specialization, push layout, wave size, user-data base, and back shader code.
- Deferred SPIR-V cache writes until `ProgramCache` shutdown to avoid render-thread disk I/O.
- Added shader timing logs: `translate_ms`, `backend_ms`, `disk_hits`, `disk_misses`.
- Reduced shader recompiler logging in silent mode.
- Disabled shader validation by default in launcher configuration.
- Release builds no longer compile Vulkan debug printf code.
- Same-size/same-format presentation uses `vkCmdCopyImage` instead of filtered blit.
- Increased command buffer pool growth step from 4 to 8.
- Increased descriptor pool capacity from 1024 to 2048 sets.
- Increased stream buffer from 64 MiB to 128 MiB.
- Release GCC/Clang builds no longer force frame pointers.
- Added slow-operation telemetry around the unavoidable synchronous graphics pipeline lookup/creation path.
  - Added bounded VideoOut pacing telemetry: every 120 present-thread samples report late-frame count, average lateness, and maximum lateness without changing vblank scheduling behavior.
  - Added bounded `FlushAndWait` GPU-wait telemetry: every 128 waits reports the cumulative wait count and average wait duration, allowing CPU/GPU synchronization stalls to be compared with present pacing.
  - Added visible slow-operation telemetry for shader compilation and graphics/compute pipeline creation (8 ms threshold), allowing isolated frame spikes to be correlated with a specific shader or pipeline.
  - Integrated SPIRV-Tools performance passes for newly recompiled shaders, with safe fallback to the original module when optimization fails and a cache format bump for invalidation.
  - Added optimizer workload metrics to shader-cache logs: successful optimizer runs, SPIR-V words saved, and validation/optimization fallbacks.
  - Added descriptor image reuse/rebind counters to measure snapshot-cache opportunities without changing PS5 resource visibility or Vulkan image-layout transitions.
  - Added `FlushAndWait` frequency and average GPU-wait telemetry without changing guest synchronization semantics.
  - Reserved capacity for shader and pipeline lookup tables to avoid render-thread hash-table rehashes
    as a workload discovers new permutations.
  - Narrowed the pipeline-cache mutex scope so synchronous Vulkan pipeline creation does not hold the
    cache lock; lookup and insertion remain protected while the single render-thread creation path
    preserves pipeline ordering.
- Removed the same cache lock around synchronous shader translation and shader-module creation;
    graphics and compute shader refresh are render-thread-only, so the long compiler path no longer
    holds an unrelated pipeline-cache mutex.

## Remaining candidates

1. Async compute queue separation.
2. Direct rendering to swapchain when layouts/formats allow it.
3. Present/vblank pacing investigation and tuning based on the new runtime measurements.
4. Robust Vulkan feature cost profiling and optional reduction.
5. SPIR-V optimizer tuning based on the new workload metrics.
6. Descriptor binding/image snapshot cache expansion based on reuse/rebind measurements.
7. Remove or reduce only proven `FlushAndWait` waits using the new wait-cost telemetry.
8. AMD-specific runtime measurements on RADV/Windows driver.

The current pass is continuing with low-risk render-thread overhead reductions first. The next
measurement should compare the number and duration of first-use shader/pipeline stalls after the
lookup-table reservation. This does not replace the larger async pipeline-prewarm design required
to remove driver compilation stalls entirely.

## Current status

`glslangValidator` is installed, CMake configures successfully, and the earlier Vulkan-Hpp compatibility breakage in the renderer/presentation stack has been fixed by replacing implicit brace assignments with explicit `vk::Extent*`/`vk::Offset*` constructors and by normalizing the mixed 2D/3D extent comparisons. The audio-side union default-construction issue was also fixed for the current compiler toolchain.

The host Vulkan path still uses monolithic graphics pipeline creation. A graphics-pipeline-library implementation remains future architecture work; the current slow-operation telemetry confirms that first-use `vkCreateGraphicsPipelines` calls can block the render thread for hundreds of milliseconds.

The first two candidates have an important host-architecture constraint. Device selection currently requires one queue family that supports graphics, compute, and presentation, and the command scheduler submits all guest work through that queue. Splitting async compute therefore requires a second scheduler plus explicit Vulkan semaphore and resource-ownership tracking; enabling another queue opportunistically would be unsafe for PS5 guest ordering. Direct swapchain rendering is also not a drop-in replacement: guest video-out images are produced before swapchain acquisition, then the presentation path applies format/extent conversion and an optional system overlay. The current copy/blit path is consequently retained until those ownership and presentation contracts are redesigned.

The first low-risk present/vblank measurement is now implemented in the VideoOut present thread, and shader-cache logs now expose optimizer effectiveness and fallback counts. The next step is to collect measurements under real workloads and tune only after observing the late-frame and shader patterns, followed by an explicit design for a second host queue rather than a partial async-compute toggle.

## Remaining work and completion estimate

The low-risk infrastructure work is essentially done: persistent shader caches, telemetry, optimizer integration, and present-thread pacing instrumentation are in place. A real graphics-pipeline-library or asynchronous pipeline-prewarm path remains architecture-sensitive work rather than a completed feature.

At this point, roughly 60-70% of the shader-performance task package is complete. The remaining 30-40% is concentrated in:

1. Collecting and interpreting the new present/vblank pacing measurements under real workloads.
2. Tuning only after seeing the late-frame and shader patterns instead of guessing.
3. Designing a safe second host queue for async compute, with explicit ownership/semaphore tracking.
4. Validating swapchain direct-rendering viability without breaking guest ownership/presentation contracts.
5. Targeted AMD/RADV/Windows driver validation and optimizer tuning based on measured workload metrics.

## Handoff

To continue from another Copilot Chat session, open this repository/Codespace and ask the agent to read this file, then name the next item to implement. The next concrete task is to collect and interpret the new present/vblank pacing measurements; async compute and direct swapchain rendering remain larger architectural projects with the constraints documented above.
