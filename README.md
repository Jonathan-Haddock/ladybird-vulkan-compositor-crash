# Ladybird — Vulkan compositor startup crash (build, reproduce, triage, report)

[![Filed upstream: issue #10172](https://img.shields.io/badge/filed%20upstream-LadybirdBrowser%2Fladybird%20%2310172-brightgreen?logo=github)](https://github.com/LadybirdBrowser/ladybird/issues/10172)

> **⤷ Where this work landed:** filed upstream as
> **[LadybirdBrowser/ladybird#10172](https://github.com/LadybirdBrowser/ladybird/issues/10172)** —
> a reproducible bug report with root cause and workaround for the Vulkan compositor startup crash.

Investigation of a startup crash in the [Ladybird](https://github.com/LadybirdBrowser/ladybird)
web browser, built from source on a current Linux laptop: building a large C++/Vulkan project from
source, reproducing a runtime crash, narrowing it to a root cause, and confirming a workaround.

## TL;DR finding

Ladybird (commit `849d528`) crashes on launch before showing a page. The **Compositor** process
hits `VERIFY(context.submit(GrSyncCpu::kNo))` (`LibGfx/SkiaBackendContext.cpp:70`) after a Vulkan
`AllocBufferMemory` call fails (`VK_ERROR_OUT_OF_DEVICE_MEMORY`) on a *small* vertex buffer; the
Compositor dies and the parent crashes via a broken IPC `VERIFY`, ending in `SIGILL`.

- **GPU/driver-independent:** forcing the Vulkan device with `MESA_VK_DEVICE_SELECT`, **both** the
  AMD (Mesa RADV) and NVIDIA (proprietary) drivers crash identically — not a single-driver regression.
- **Workaround:** `--force-cpu-painting` launches normally, isolating the fault to the Vulkan/Skia
  compositor path.
- **Duplicate check:** the same Vulkan `-2` allocation failure already appears under heavy page load
  in #6382, but this *startup, no-page, GPU-independent* scenario is not on the tracker. Closest
  themes: #6382 and #9940 (see `INVESTIGATION.md`).

## Repository contents

| File | What it is |
|---|---|
| [`BUG_REPORT.md`](BUG_REPORT.md) | The upstream bug report, in Ladybird's issue-template format, ready to file or post as a comment on #6382/#9940. |
| [`INVESTIGATION.md`](INVESTIGATION.md) | Chronological log: build, crash, the "is this really a bug?" review, the triage experiments, and the duplicate check. |
| `logs/default.log`, `amd.log`, `nvidia.log` | Full crash output of the default and per-GPU runs. |
| `logs/system-info.txt` | OS, CPU, GPUs, drivers, Vulkan summary, Ladybird commit. |
| `logs/triage/` | Outputs of the triage experiments (`--help`, `--force-cpu-painting`, `MESA_VK_DEVICE_SELECT` runs). |

## Environment (summary)

Ubuntu 26.04 LTS · kernel 7.0.0-22 · AMD Ryzen 7 4800H (Renoir) + NVIDIA GTX 1660 Ti · Mesa 26.0.3 ·
NVIDIA 595.71.05 · Vulkan 1.4.341 · Qt 6.10.2 · clang 21.1.8. Full details in `logs/system-info.txt`.

## What this demonstrates

Building a modern browser engine from source (resolving a chain of toolchain/dependency failures),
reading multi-process crash backtraces, distinguishing root cause from fallout, **testing a
hypothesis instead of assuming it**, using the correct Vulkan device-selection mechanism, checking
the issue tracker before filing, finding a workaround, and writing a clear, reproducible bug report.
