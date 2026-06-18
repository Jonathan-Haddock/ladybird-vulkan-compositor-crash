# Compositor aborts at startup on `VERIFY(context.submit(...))` after Vulkan `AllocBufferMemory` fails (no window shown)

> Ready to paste into https://github.com/LadybirdBrowser/ladybird/issues/new (Bug report template).
> Fields below map 1:1 to the template. Attach `logs/default.log`, `logs/amd.log`, `logs/nvidia.log`,
> `logs/triage/*`, and `logs/system-info.txt` to the issue.
>
> Filesystem paths in this repository copy are normalized to `/home/user` as an information-hygiene
> habit. The report and logs filed upstream use the **unmodified** backtrace, so maintainers see the
> exact output for reproduction.

### Summary

On a freshly built `Ladybird` (commit `849d528`), the browser fails to start. A window appears for a fraction of a second and then the process dies with `Illegal instruction (core dumped)`. The root failure is in the **Compositor** process: a Vulkan buffer allocation fails, Skia's `GrDirectContext::submit()` then returns `false`, and `LibGfx/SkiaBackendContext.cpp:70` hard-aborts on `VERIFY(context.submit(GrSyncCpu::kNo))`. Once the Compositor dies, the parent hits a second `VERIFY` on the broken IPC response (`LibIPC/Connection.h:74`) and the whole application goes down with `SIGILL`.

The first lines of output are the actual cause:

```
Failed vulkan call. Error: -2, skgpu::VulkanMemory::AllocBufferMemory (allocUsage:1, shouldPersistentlyMapCpuToGpu:0)
Could not allocate vertices
Failed vulkan call. Error: -1, CreateFence(gpu->device(), &fenceInfo, nullptr, &fSubmitFence)
VERIFICATION FAILED: context.submit(GrSyncCpu::kNo) at .../Libraries/LibGfx/SkiaBackendContext.cpp:70
```

In Vulkan, `-2` is `VK_ERROR_OUT_OF_DEVICE_MEMORY` and `-1` is `VK_ERROR_OUT_OF_HOST_MEMORY`. The failing allocation is a **small vertex buffer** during the first compositor frame on a 6 GB discrete GPU — this does not look like genuine memory exhaustion.

**The crash is GPU/driver-independent.** Forcing the Vulkan physical device with `MESA_VK_DEVICE_SELECT` (not `DRI_PRIME`, which only affects GL/EGL), both real GPUs fail identically:

| Forced Vulkan device | Driver | Result |
|---|---|---|
| `1002:1636` AMD Radeon Renoir | Mesa RADV 26.0.3 | crash — `AllocBufferMemory` -2 |
| `10de:2191` NVIDIA GTX 1660 Ti | NVIDIA proprietary 595.71.05 | crash — identical `AllocBufferMemory` -2 |

Two completely independent Vulkan drivers failing at the same small allocation argues against a single-driver/Mesa regression. **`--force-cpu-painting` launches normally**, isolating the fault to the Vulkan/Skia compositor path (and serving as a workaround).

**Related existing issues (this is not an exact duplicate):**
- **#6382** (OpenStreetMap iD editor) shows the same `skgpu::VulkanMemory::AllocBufferMemory` Error -2, but triggered by a *heavy page* — many repeated failures with `allocUsage:2` and persistent mapping, consistent with GPU-memory pressure under load. This report differs: a **single small** allocation (`allocUsage:1`) fails **at startup with no page loaded**, and is immediately fatal.
- **#9940** (incompatible Vulkan backend selection on multi-device Linux) shares the multi-GPU/Vulkan theme, but crashes in the WebGL `OpenGLContext`/`eglCreateImage` path triggered by a page, not in the Compositor at startup.

**What I ruled out / scope.** Skia here is the vcpkg-pinned build (`libskia.so`, `chromium_7258`), identical to upstream — the renderer is not built from anything on my system. With `VK_LAYER_KHRONOS_validation` (1.4.341) loaded at instance and device level, there are **no validation/usage errors** before the failure, so this is not an invalid Vulkan call from Ladybird/Skia — the driver returns `VK_ERROR_OUT_OF_DEVICE_MEMORY` for a small, valid allocation on an idle system. The unusual variable is the (very new) system Vulkan stack, so this may be a driver/Vulkan-stack interaction rather than a Ladybird renderer defect.

Separately, and **regardless of the underlying cause**: taking the whole browser down via `VERIFY` on a failed GPU submit — instead of falling back to CPU painting, which works (`--force-cpu-painting`) — may be worth treating as a robustness issue.

### Operating system

Linux — Ubuntu 26.04 LTS, kernel 7.0.0-22-generic (x86_64), Wayland session.

### Steps to reproduce

1. Build current `main` (commit `849d528`) in release: `./Meta/ladybird.py run`.
2. Launch `./Build/release/bin/Ladybird`.
3. A window flashes and disappears; the process exits with `Illegal instruction (core dumped)`.

No URL or interaction is required — it fails during compositor startup. The crash persists when forcing either real GPU:

```
MESA_VK_DEVICE_SELECT=1002:1636 ./Build/release/bin/Ladybird   # AMD Renoir
MESA_VK_DEVICE_SELECT=10de:2191 ./Build/release/bin/Ladybird   # NVIDIA GTX 1660 Ti
```

Workaround — CPU painting launches normally:

```
./Build/release/bin/Ladybird --force-cpu-painting
```

### Expected behavior

Ladybird launches and renders the default page (or falls back to CPU painting if the Vulkan compositor cannot initialize, rather than aborting).

### Actual behavior

A window appears briefly, the Compositor aborts on a failed Vulkan allocation, IPC to the Compositor breaks, and the application crashes with `SIGILL` before anything is rendered.

### URL for a reduced test case

N/A — the crash occurs at startup, before any page is loaded.

### HTML/SVG/etc. source for a reduced test case

```
N/A
```

### Log output and (if possible) backtrace

Full output of the default launch (`logs/default.log`; `logs/amd.log` and `logs/nvidia.log` are identical apart from PIDs):

```
Ladybird PID file '/run/user/1000/Ladybird.pid' exists with PID 683138, but process cannot be found
Failed vulkan call. Error: -2, skgpu::VulkanMemory::AllocBufferMemory (allocUsage:1, shouldPersistentlyMapCpuToGpu:0)
Could not allocate vertices
Failed vulkan call. Error: -1, CreateFence(gpu->device(), &fenceInfo, nullptr, &fSubmitFence)
VERIFICATION FAILED: context.submit(GrSyncCpu::kNo) at /home/user/ladybird/Libraries/LibGfx/SkiaBackendContext.cpp:70
Stack trace (most recent call first):
#0  0x000078ba6655511b at /home/user/ladybird/Build/release/lib/liblagom-gfx.so.0
#1  0x000078ba66555713 at /home/user/ladybird/Build/release/lib/liblagom-gfx.so.0
#2  0x000078ba682686de at /home/user/ladybird/Build/release/lib/liblagom-web.so.0
#3  0x00005e701474ee2e at /home/user/ladybird/Build/release/libexec/Compositor
#4  0x00005e701474f11f at /home/user/ladybird/Build/release/libexec/Compositor
#5  0x00005e7014782841 at /home/user/ladybird/Build/release/libexec/Compositor
#6  0x000078ba6a59ce57 at /home/user/ladybird/Build/release/lib/liblagom-core.so.0
#7  0x000078ba6a59c21f at /home/user/ladybird/Build/release/lib/liblagom-core.so.0
#8  0x000078ba6a5a2c7b at /home/user/ladybird/Build/release/lib/liblagom-core.so.0
#9  0x00005e701473a19b at /home/user/ladybird/Build/release/libexec/Compositor
#10 0x00005e7014792807 at /home/user/ladybird/Build/release/libexec/Compositor
#11 0x000078ba65c2a600 in __libc_start_call_main
#12 0x000078ba65c2a717 in __libc_start_main_impl
#13 0x00005e70147399a4 at /home/user/ladybird/Build/release/libexec/Compositor
288853.121 WebContent(683227): Failed to receive message_id: 33
288853.121 Ladybird(683200): Failed to receive message_id: 6
VERIFICATION FAILED: response at /home/user/ladybird/Libraries/LibIPC/Connection.h:74
Stack trace (most recent call first):
#0  (inlined) IPC::Connection<...>::send_sync<...CreateContext...> at Libraries/LibIPC/Connection.h:74:9
#1  (inlined) CompositorControlServerProxy<...>::create_context(...) at Services/Compositor/CompositorControlServerEndpoint.h:1057:96
#2  WebView::Application::register_compositor_context(...) at Libraries/LibWebView/Application.cpp:586:40
#3  WebView::WebContentClient::compositor_context_id_for_page(unsigned long) at Libraries/LibWebView/WebContentClient.cpp:97:51
#4  WebView::WebContentClient::allocate_compositor_context_id(...) at Libraries/LibWebView/WebContentClient.cpp:112:54
...  (Qt event loop frames omitted)
#35 WebView::Application::execute() at Libraries/LibWebView/Application.cpp:1056:30
#36 ladybird_main(Main::Arguments) at UI/Qt/main.cpp:109:25
#37 main at Libraries/LibMain/Main.cpp:50:6
```

#### Vulkan device enumeration (`MESA_VK_DEVICE_SELECT=list`)

```
GPU 0: 10de:2191 "NVIDIA GeForce GTX 1660 Ti" discrete GPU
GPU 1: 1002:1636 "AMD Radeon Graphics (RADV RENOIR)" integrated GPU
GPU 2: 10005:0   "llvmpipe (LLVM 21.1.8)" CPU
```

#### Environment

| | |
|---|---|
| Ladybird commit | `849d52822048d0ac395e424b8f5107540c2d842f` (2026-06-18) |
| OS / kernel | Ubuntu 26.04 LTS, 7.0.0-22-generic, x86_64, Wayland |
| Chrome | Qt 6.10.2 |
| Compiler / build | clang 21.1.8, CMake 4.2.3, Ninja 1.13.2 |
| CPU | AMD Ryzen 7 4800H (Renoir) |
| GPU 0 | AMD Radeon Graphics (RADV RENOIR), Mesa 26.0.3, Vulkan 1.4.335 |
| GPU 1 | NVIDIA GeForce GTX 1660 Ti, driver 595.71.05, Vulkan 1.4.329 |
| GPU 2 | llvmpipe (LLVM 21.1.8), Mesa 26.0.3 |
| Vulkan instance | 1.4.341 |

`vulkaninfo --summary` succeeds and enumerates all three devices (full output in `logs/system-info.txt`).
