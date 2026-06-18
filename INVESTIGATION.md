# Investigation log — Ladybird startup crash (Vulkan compositor)

A chronological record of how this crash was reproduced, triaged, and narrowed to a
root cause. Commands and raw outputs are preserved under `logs/`. The point of this
log is not just the conclusion but the **process** — including a deliberate step where
the initial conclusion was challenged before anything was reported upstream.

- **Target:** Ladybird, commit `849d52822048d0ac395e424b8f5107540c2d842f` (2026-06-18)
- **Machine:** HP OMEN 15, AMD Ryzen 7 4800H (Renoir) + NVIDIA GTX 1660 Ti, Ubuntu 26.04 LTS, Wayland
- **Full environment:** `logs/system-info.txt`
- **A note on paths:** filesystem paths in this published repo are normalized to `/home/user` as an
  information-hygiene habit. This is deliberate, not because the path is sensitive — it is a
  personal machine — but to keep the reflex. The version filed upstream uses the unmodified
  backtrace so maintainers have the exact output; normalizing the report a maintainer needs to
  reproduce from would be the wrong call.

---

## 1. Build from source

Built `main` in release via `./Meta/ladybird.py run`. Getting there required resolving a
chain of missing toolchain/dependency components one at a time:

| Failure | Fix |
|---|---|
| `cmake` not found | installed CMake (4.2.3) |
| `python3.14-venv` missing | `apt install python3.14-venv` |
| `nasm` not found | `apt install nasm` |
| `cargo`/`rustc` not found | `apt install cargo rustc` |
| `Qt6Config.cmake` not found | `apt install qt6-base-dev qt6-tools-dev` |

`libpulse` was absent (audio backend disabled) — a warning, not an error; build continued.
Build completed successfully and produced `Build/release/bin/Ladybird`.

## 2. First run — crash

A window appeared for a fraction of a second, then the process died with
`Illegal instruction (core dumped)`. Captured full output for the default launch and for
both GPUs (`logs/default.log`, `logs/amd.log`, `logs/nvidia.log`). The first lines are the
actual cause; the rest is fallout:

```
Failed vulkan call. Error: -2, skgpu::VulkanMemory::AllocBufferMemory (allocUsage:1, shouldPersistentlyMapCpuToGpu:0)
Could not allocate vertices
Failed vulkan call. Error: -1, CreateFence(gpu->device(), &fenceInfo, nullptr, &fSubmitFence)
VERIFICATION FAILED: context.submit(GrSyncCpu::kNo) at Libraries/LibGfx/SkiaBackendContext.cpp:70
```

**Failure chain:**
1. `skgpu::VulkanMemory::AllocBufferMemory` returns `-2` (`VK_ERROR_OUT_OF_DEVICE_MEMORY`)
   while allocating a small vertex buffer for the first compositor frame.
2. `CreateFence` then returns `-1` (`VK_ERROR_OUT_OF_HOST_MEMORY`).
3. Skia's `GrDirectContext::submit()` returns `false`, so the hard assert at
   `SkiaBackendContext.cpp:70` (`VERIFY(context.submit(GrSyncCpu::kNo))`) aborts the
   **Compositor** process.
4. With the Compositor gone, the parent's IPC `send_sync` for `CreateContext` fails its
   own `VERIFY` (`LibIPC/Connection.h:74`) and the app exits with `SIGILL`.

## 3. Critical review — *is this actually a bug, or my configuration?*

Before writing anything up, I challenged the conclusion. A `-2` out-of-device-memory on a
*small* vertex buffer, on a 6 GB discrete card, is not plausible real VRAM exhaustion — so
something was off, but the cause was not yet established. Two hypotheses:

- **(A) Ladybird-side bug** — bad Vulkan/Skia setup, and/or a robustness bug: hard-asserting
  (and taking down the whole browser) on a failed GPU submit instead of degrading to software.
- **(B) Environment** — an extremely bleeding-edge stack (Ubuntu 26.04, kernel 7.0, Mesa
  26.0.3, Vulkan 1.4.341, NVIDIA 595) with three Vulkan devices and several layers; the fault
  could live in the driver/loader or in Vulkan device selection.

I also caught a flaw in my own first draft: I had written that the crash "reproduces on both
GPUs via `DRI_PRIME`." But `DRI_PRIME` selects the **OpenGL/EGL** device, **not** the Vulkan
physical device — so those runs may all have used the same Vulkan device. The GPU-independence
claim was unproven. Three experiments were designed to settle it.

## 4. Triage experiments

### Experiment 1 — does Ladybird expose a software-render path?
`./Build/release/bin/Ladybird --help` → found **`--force-cpu-painting`** (output: `logs/triage/ladybird-help.txt`).

### Experiment 2 — software rendering
```
./Build/release/bin/Ladybird --force-cpu-painting about:blank
```
Result: **launches and stays alive** (no crash; ran until the 20s test timeout). Output:
`logs/triage/force-cpu-painting.log`.
→ The fault is specifically in the **Vulkan GPU painting path**, and `--force-cpu-painting`
is a working **workaround**.

### Experiment 3 — force the Vulkan device with the *correct* mechanism
`MESA_VK_DEVICE_SELECT` (the `VK_LAYER_MESA_device_select` layer) selects the actual Vulkan
physical device. Enumeration (`logs/triage/vk-device-list.log`):

```
GPU 0: 10de:2191 "NVIDIA GeForce GTX 1660 Ti" discrete GPU
GPU 1: 1002:1636 "AMD Radeon Graphics (RADV RENOIR)" integrated GPU
GPU 2: 10005:0   "llvmpipe (LLVM 21.1.8)" CPU
```

| Forced device | Driver | Result |
|---|---|---|
| `MESA_VK_DEVICE_SELECT=1002:1636` | AMD RADV (Mesa) | **crash**, `AllocBufferMemory` -2 (`logs/triage/vk-amd.log`) |
| `MESA_VK_DEVICE_SELECT=10de:2191` | NVIDIA proprietary | **crash**, identical `AllocBufferMemory` -2 (`logs/triage/vk-nvidia.log`) |

## 5. Reconfirmation, dependency scope, and Vulkan validation

**Reconfirmed** immediately before reporting: the default launch still crashes deterministically with
the identical signature (`AllocBufferMemory` -2 → `SkiaBackendContext.cpp:70` → `SIGILL`).

**What is mine vs. Ladybird's.** `libskia.so` is the **vcpkg-pinned** build
(`Build/release/vcpkg_installed/x64-linux-dynamic/lib/libskia.so`, `chromium_7258`); Ladybird's
renderer (Skia + `LibGfx`/Compositor) is identical to what upstream CI and other contributors build —
it is not compiled from anything on my system. The unusual variables are all from the OS: Mesa
26.0.3 / RADV, NVIDIA 595.71.05, Vulkan loader + ICDs 1.4.341, kernel 7.0. Combined with "no one
reports a *startup* crash," the differing factor is the **system Vulkan stack**.

**Vulkan validation** (`VK_LAYER_KHRONOS_validation` 1.4.341) was installed and confirmed loaded at
both instance and device level in the process that creates the Vulkan device (loader debug:
`logs/triage/validation-layer-loaded.txt`). The run produced **no VUID / usage errors** before the
failure (`logs/triage/validation-run.log`). Interpretation: Ladybird/Skia is **not making an invalid
Vulkan call** — `vkAllocateMemory` returns `VK_ERROR_OUT_OF_DEVICE_MEMORY` for a small, valid
allocation on an idle system, a runtime refusal validation does not flag. Not fully exonerating
(Skia could request a valid-but-unsatisfiable memory type), but there is no API misuse.

## 6. Conclusion

- The crash is **genuinely GPU-independent**, now proven with the correct mechanism: **two
  completely independent Vulkan drivers** (Mesa RADV and the NVIDIA proprietary driver) fail
  **identically** at the same small allocation. A single-driver/Mesa regression cannot explain
  the NVIDIA driver reproducing it.
- **CPU painting works**, isolating the fault to the Vulkan/Skia compositor path and providing
  a workaround (`--force-cpu-painting`).
- Evidence points to a **driver / Vulkan-stack interaction** on a very new stack: Skia is pinned and
  identical to upstream, validation found no API misuse, and the device refuses a small valid
  allocation. So this is likely partly environmental (Mesa 26 / Vulkan 1.4.341 / NVIDIA 595), not a
  pure Ladybird logic bug — and not proven to be a Ladybird *defect* in the renderer.
- **Independently of the cause**, hard-asserting on a failed GPU submit — taking the whole browser
  down with `SIGILL` instead of falling back to CPU painting (which works) — is a Ladybird-side
  robustness issue. This is the part squarely within Ladybird's control and worth reporting.
### Duplicate check (open + closed issues)

Searched the tracker for the exact assertion strings and the Compositor path:

- **#6382** (OpenStreetMap iD editor) — **same `AllocBufferMemory` Error -2**, but page-triggered
  under heavy load (many repeated failures, `allocUsage:2`, persistent-mapped, plus
  `AllocImageMemory`). Looks like real GPU-memory pressure. Differs from this report, which fails
  on a *single small* allocation (`allocUsage:1`) **at startup with no page**.
- **#9940** — incompatible Vulkan backend selection on multi-device Linux; related theme, different
  crash (WebGL `eglCreateImage`, page-triggered).
- **#9882** — a different Compositor assertion (`compatible_visual_context_tree_version`), macOS,
  page-triggered. #7025 — page/game-triggered crash.

**Conclusion:** the underlying Skia/Vulkan `-2` allocation failure is already known (#6382), but this
exact scenario — immediate, GPU-independent failure at startup on the first small allocation, on a
current Vulkan 1.4.341 stack — is not on the tracker. Best handled as a **new data point on #6382**
(and/or #9940) or a focused new issue that cross-references both, rather than a stand-alone
"novel bug." Final call left to a human before anything is posted.
