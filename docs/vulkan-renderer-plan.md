# melonDS Vulkan Renderer — Research & Implementation Plan

*Status: proposal. Grounded in the tree at `d3cd616` (post-blackmagic3, compute renderer with tile-size scaling).*

This document summarizes research into how melonDS renders today, what prior Vulkan work exists, how comparable emulators structured their Vulkan backends, and lays out a phased implementation plan.

---

## Part 1 — Research summary

### 1.1 How melonDS renders today

The core owns one top-level `Renderer` (2D + 3D + compositing, `src/GPU.h:833-876`) which owns two `Renderer2D` units and one `Renderer3D` (`src/GPU.h:873-875`). Three stacks exist:

| Stack | Parent | 3D | Output |
|---|---|---|---|
| Software | `SoftRenderer` (`GPU_Soft.*`) | `SoftRenderer3D` | RAM, 256×192 BGRA, per-scanline pull via `GetLine` |
| Classic GL | `GLRenderer` (`GPU_OpenGL.*`) | `GLRenderer3D` (`GPU3D_OpenGL.*`) | GL texture, polygon-based, GL 3.2+ (3.1 lowering pending upstream, PR #2642) |
| GL compute | `GLRenderer` | `ComputeRenderer3D` (`GPU3D_Compute.*`) | GL texture, tile-based compute rasterizer, GL 4.3 |

Facts a new backend must respect (all verified in-tree):

- **All rendering happens on the emu thread.** `RenderFrame()` is invoked at VCount==215 HBlank (`GPU.cpp:1163-1166`), `FinishRendering()` at the *next* frame's VCount==192 (`GPU.cpp:1289`) — ~240 scanlines (≈15 ms) before *full completion* is demanded. The consumer deadline is much shorter, though: a software-composited pairing starts pulling output via `GetLine` at the next frame's VCount==0, only ~48 scanlines (≈3 ms) after submission — the threaded software rasterizer tolerates that short lead because it syncs *per scanline* (`GPU3D_Soft.cpp:1730-1823`).
- **Two output contracts** (`Renderer::GetFramebuffers`, `GPU.h:860-862`): return `true` + RAM 256×192 BGRA pointers (software), or `false` + a backend handle. The `false` path is currently a raw `*(GLuint*)` cast in the frontend (`Screen.cpp:1150`) — a type-unsafe seam that must be generalized for Vulkan.
- **The software compositor pulls the 3D output line-by-line** via `Renderer3D::GetLine(line)` with `RenderXPos` scroll applied (`GPU_Soft.cpp:115`, contract `GPU3D.h:336-338`); GL renderers return `nullptr` and composite GPU-side instead. `GetLine` for a new frame can be called *before* that frame's `FinishRendering`, so a RAM-output renderer must sync on first use.
- **Renderer selection/hot-swap**: `EmuThread::updateRenderer()` (`EmuThread.cpp:858-889`) switches on config `3D.Renderer` and calls `NDS::SetRenderer`; `GPU::SetRenderer` (`GPU.cpp:315-344`) syncs VRAM captures, Init/Resets the new renderer, and **falls back to software if `Init()` fails** — free fallback plumbing for a Vulkan backend on unsupported hardware.
- **Renderer state is not savestated.** `PreSavestate`/`PostSavestate` bracket serialization (`GPU.cpp:209,311`); the software renderer drains its render thread there — a Vulkan renderer waits its in-flight work.
- **VRAM display capture requires GPU→CPU readback**: `AllocCapture`/`SyncVRAMCapture` (`GPU.h:857-858`) — GL uses `glReadPixels` of a downscaled capture buffer (`GPU_OpenGL.cpp:843-912`). This is the only mandatory readback; everything else stays on-GPU.
- **Frontend/windowing**: Qt is *not* coupled through `QOpenGLWidget`. `ScreenPanelGL` is a native, Qt-never-paints widget (`WA_NativeWindow`, `WA_PaintOnScreen`, null `paintEngine()`, `Screen.cpp:868-878, 1350-1353`); contexts come from the vendored DuckStation layer (`src/frontend/duckstation/gl/`) created against native handles collected into `WindowInfo` (`window_info.h` — Win32/X11/Wayland/macOS). Up to 4 windows share one context tree; the emu thread renders and presents to each sequentially. **`WindowInfo` already carries every native handle Vulkan WSI needs.**
- **Build system**: GL support is gated by `ENABLE_OGLRENDERER` → `OGLRENDERER_ENABLED` (`src/CMakeLists.txt:97-109`), consumed in only 6 files. Shaders are embedded as strings at build time via `cmake/MakeEmbed.cmake` (`src/OpenGL_shaders/CMakeLists.txt:30-47`) and compiled by the driver at runtime. No shader compiler exists in the build today. There is **zero Vulkan code anywhere in the tree**.
- **Config hazard**: the `renderer3D_*` enum values are positional ints persisted to TOML *and* wrapped in `#ifdef OGLRENDERER_ENABLED` (`EmuInstance.h:71-79`) — adding entries under a new ifdef silently re-maps users' stored settings when build flags differ. Must be fixed with explicit values.

### 1.2 The compute renderer is the porting asset

`ComputeRenderer3D` is the DS software rasterizer translated to GPU compute: CPU builds per-polygon span setups and per-scanline work indices, then ~9 compute dispatch stages per frame do span interpolation, coarse+fine tile binning, per-variant rasterization (one indirect dispatch per active variant, up to 256), depth/blend resolve in submission order (with the DS's ±0x200/±0xFF depth-equal tolerances), and a final pass (edge marking, fog, AA) into an `rgba8` image (`GPU3D_Compute.cpp:1059-1216`, `GPU3D_Compute_shaders.h`).

It is nearly Vulkan-shaped already:

- Born on the Switch under **deko3d** (a Vulkan-like API), ported to GL 4.3 (PR #2041) — dead deko3d references remain in-tree (`GPU3D_Compute.cpp:1222-1252`).
- Uses **SSBO atomics only** — no image atomics, no subgroup ops (the `ballotARB` path is `#if 0`), no 64-bit ints. This is exactly the subset that is solid on MoltenVK/Metal and mobile.
- Shader variants are generated by prepending `#define`s (33 programs, recompiled on scale change through the incremental `ShaderCompileStep` loop) — in Vulkan these become **specialization constants** plus a small set of build-time permutations, eliminating runtime compilation entirely.
- Known port frictions (all mechanical): GL binding indices are reused across pipelines (binding 2 = YSpanSetups *and* ColorTiles) and need per-pipeline set layouts; `bool` members in std430 blocks must become `uint`; loose `layout(location=N)` uniforms become push constants; `glMemoryBarrier` calls become explicit `vkCmdPipelineBarrier2` chains; the bin-result buffer needs `INDIRECT_BUFFER` usage.

The classic GL renderer, by contrast, leans on GL-isms that translate poorly (stencil bit-plane mid-pass clears, per-batch texture-wrap mutation, `GL_MAX` blend tricks) and its edge-marking/AA is partly non-functional today. It stays as the weak-GPU fallback (upstream is even lowering it to GL 3.1, PR #2642); porting it buys nothing.

The texture cache is already API-generic: `Texcache<TexLoaderT, TexHandleT>` (`GPU3D_Texcache.h:43`) does CPU decode of all DS formats to RGB6A5 with xxhash-based VRAM-dirty invalidation; the GL-specific part is ~30 lines (`GPU3D_TexcacheOpenGL.cpp`). A Vulkan instantiation is a small new loader.

### 1.3 Prior art (melonDS × Vulkan)

- **Requested since 2020** (#807, #984, #1060, #2085). Issue #984 led to PR #990, which created the modular `Renderer3D` interface explicitly so Vulkan could plug in.
- **PR #2700** (July 2026, closed): a near-complete port of the compute renderer to Vulkan incl. MoltenVK on macOS and frame pipelining — closed under the project's **no-AI-code policy**, not on technical grounds. Its existence proves the port is tractable.
- **melonDualDS** (`SapphireRhodonite/melonDS-android`): a *shipped* Vulkan compute backend on Android — the compute renderer demonstrably ports to Vulkan on mobile drivers.
- **Maintainer direction is OpenGL** (blackmagic3 2D-GL rework, GL 3.1 lowering). No maintainer has committed to Vulkan. Any upstreaming attempt must be discussed first and hand-written to house style; this plan is structured so the work is valuable in a fork regardless.
- **paraLLEl-RDP** (N64) is the strongest external existence proof of a production Vulkan compute rasterizer, and the pattern source for timeline-semaphore sync and specialization-constant variants.

### 1.4 Lessons from other emulators

- **DuckStation/PPSSPP/Dolphin** all abstract over APIs, at the cost of tens of kLOC grown over years. Citra's experience (closest analogue: fixed-function handheld, small framebuffers, hi-res scaling): the dominant cost is **infrastructure** — swapchain, sync, descriptors, driver quirks — not the rasterizer.
- **melonDS does not have the shader-explosion problem** that forced Dolphin's ubershaders and async-pipeline machinery. The DS shader set is closed (~35 compute + a handful of graphics pipelines); **everything can be precompiled at init**. Do not build async shader infrastructure.
- **Qt + Vulkan**: `QVulkanWindow` is the wrong shape (owns device/swapchain, drives rendering from Qt's event loop — incompatible with an emu-thread renderer). Dolphin/DuckStation pattern: backend creates `VkSurfaceKHR` from native window handles; **Wayland is the known trap** (Qt owns the `wl_surface`) and needs either careful reuse of the existing native-widget approach or a dedicated child `QWindow`.
- **Ryujinx-style runtime SPIR-V emission is overkill** for a static shader set; build-time compilation removes the runtime dependency entirely.

---

## Part 2 — Headline decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Vulkan baseline** | **Vulkan 1.3** (dynamic rendering, synchronization2, timeline semaphores all core). No pre-1.3 fallback — software/GL remain the fallback renderers. Enable `VK_KHR_portability_enumeration`/`portability_subset` for MoltenVK (≥1.3 — bundled by us in Phase 5; macOS ships nothing itself). | 1.3 covers all hardware this backend targets. A known legacy class (NVIDIA Fermi/Kepler, Windows Haswell/Broadwell, Windows GCN 1/2) runs the GL 4.3 compute path but can never reach Vulkan 1.3 — those users stay on the existing GL renderers, and the selection UI must say so explicitly rather than fail silently. The prize is macOS via MoltenVK, where GL caps at 4.1 and the compute renderer is currently **impossible**. Avoids an extension-permutation matrix. Android's mostly-1.1 installed base is explicitly out of scope; the design (SSBO atomics only, no subgroups, no image atomics) keeps a later 1.1+extensions mobile backend feasible. |
| **First (and only) 3D port target** | **`ComputeRenderer3D`**, not the classic GL renderer. | Software-rasterizer accuracy; already survived deko3d→GL (and GL→Vulkan twice in prior art); its GL feature set maps *cleaner* to Vulkan; unlocks macOS. Classic GL stays as the weak-GPU fallback. |
| **Shader toolchain** | **Build-time GLSL→SPIR-V with glslangValidator** (`--target-env vulkan1.3`), embedded via a binary-capable extension of `cmake/MakeEmbed.cmake`. **Specialization constants** for all config-derived values (screen dims, TileSize, MaxWorkTiles; workgroup sizes via `local_size_x_id`; derived globals as spec-constant expressions — verified that none of the injected `#define`s size arrays, so nothing blocks this); the fixed rasterizer-variant matrix as build-time permutations. Scale change = re-specializing pipelines from cached SPIR-V modules. No runtime glslang/shaderc. | The variant space is closed. Removes MBs of runtime dependency (matters for `BUILD_STATIC` MinGW/BSD), matches the existing shader-embed pattern, and replaces the 33-program recompile-on-scale-change (`NeedsShaderCompile` machinery becomes unnecessary — scale changes are cheap pipeline rebuilds from cached SPIR-V). |
| **Loader / bootstrap / memory** | Vendored **volk** (dlopen — no hard link dep, same philosophy as glad), **vk-bootstrap** (instance/device selection), **VMA** (single header, `VMA_MEMORY_USAGE_AUTO`). All under a new `src/vulkan/` (they're needed by core-side renderers; precedent: everything GPU-related is vendored). | Allocation profile is tiny and static (a dozen SSBOs, texcache arrays, two screen images, a staging ring). No reason to hand-roll; no reason to link the loader. |
| **WSI / Qt integration** | **No `QVulkanWindow`, no QRhi.** A `ScreenPanelVK` sibling of `ScreenPanelGL` (reusing base `ScreenPanel` layout/OSD/input; `getWindowInfo()` and the native-widget setup live on `ScreenPanelGL` today, `Screen.h:184-192` / `Screen.cpp:868-878`, and get hoisted into the base), plus a presenter shaped like the vendored `GL::Context` interface, creating `VkSurfaceKHR` from `WindowInfo` native handles (`vkCreateWin32/Xlib/Wayland/MetalSurfaceKHR`; macOS needs a small `CAMetalLayer` ObjC++ shim like `context_agl.mm`). All rendering and presentation stay on the **emu thread**, preserving today's threading contract; `MakeCurrent`/borrow hooks become no-ops. | Minimum-diff, Dolphin/DuckStation-proven. `WindowInfo` already carries the handles. Wayland ships gated (see Phase 5) with the child-`QWindow`/`createWindowContainer` fallback prepared. |
| **Descriptors / sync** | One descriptor-set layout per pass; **push constants** for per-dispatch scalars. The one hard binding problem is the rasterization loop: up to 256 back-to-back indirect dispatches, each binding a different texcache array image plus one of 9 samplers, with the image set changing every frame as the cache allocates/evicts. Strategy: split the combined sampler into separate image + sampler bindings, bake the 9 wrap-mode samplers as an immutable-sampler array indexed via the existing push-constant block, and allocate per-variant sets from a per-frame-reset pool (worst case 256 × frames-in-flight) or a texture-keyed set cache; destruction of evicted texcache images is deferred past GPU frame retirement. No descriptor indexing, no bindless. One **timeline semaphore** as the frame-progress clock; two frames in flight bounded by the core's own pacing. | Simplest robust model at this pipeline count; MoltenVK/mobile-safe. Baking 9 immutable samplers into one layout avoids 9× duplication of the rasterizer pipeline layouts. |
| **Cross-API RHI layer** | **No.** Self-contained backend (parallel-rdp style), plus three cheap permanent seams: (1) a tagged output handle replacing the `*(GLuint*)` convention; (2) a presenter interface shaped like `GL::Context` so `EmuInstance`/`EmuThread` stop naming GL; (3) the texcache template instantiation. | An RHI only pays if the GL renderers are to be replaced — not the goal. The three seams are where an abstraction would grow later if Metal-native/Android ever materialize. |
| **Pipeline cache / validation** | Single `VkPipelineCache` persisted per-deviceUUID with header validation (politeness — all pipelines build at init anyway). `VK_LAYER_KHRONOS_validation` + **sync validation** behind a config/env toggle from day one; every object named via `VK_EXT_debug_utils`; per-pass command labels. | Sync validation is the highest-value tool for a compute-rasterizer port — every `glMemoryBarrier` becomes an explicit dependency that will be wrong the first time. |

---

## Part 3 — Phased implementation plan

Every phase lands independently, is inert with `ENABLE_VKRENDERER=OFF` (default until Phase 5), and leaves the tree shippable. Phases 2 and 3 are independent of each other and can proceed in parallel.

### Phase 1 — Build scaffolding, device layer, headless self-test (M)

**Goal:** Vulkan infrastructure compiles across the CI matrix, off by default; device bring-up is provable headlessly.

- `CMakeLists.txt`: `option(ENABLE_VKRENDERER)` beside `ENABLE_OGLRENDERER` (line 68).
- `src/CMakeLists.txt`: gated block mirroring the GL one (lines 97-109), defining `VKRENDERER_ENABLED`.
- New `src/vulkan/`: vendored `volk`, `vk_mem_alloc.h`, vk-bootstrap; `VulkanDevice.{h,cpp}` — instance (frontend supplies surface extensions when presentation is wanted; headless otherwise), physical-device selection (require 1.3; check `maxPerStageDescriptorStorageBuffers` ≥ 10; record `maxComputeWorkGroupInvocations`, `maxComputeWorkGroupCount[2]`, `maxStorageBufferRange` for ScaleFactor clamping later — GL 4.3 guarantees ≥1024 workgroup invocations but Vulkan's floor is 128, and TileSize² hits 256 at ≥5× scale, 1024 at ≥9×, while the tile SSBOs approach ~800 MB at 16×), device + queues, VMA, timeline-semaphore helper, staging ring, debug-utils naming, validation toggle (`3D.Vk.Validation` config + env var), portability-subset handling.
- `cmake/MakeEmbedBinary.cmake` (emits `uint32_t[]`) + `src/Vulkan_shaders/CMakeLists.txt` with glslangValidator custom commands modeled on `src/OpenGL_shaders/CMakeLists.txt:30-47`.
- `vcpkg.json`: `vulkan-headers`, `vulkan-memory-allocator`, host `glslang`; CI workflows: `glslang-tools`, `libvulkan-dev`, and **`mesa-vulkan-drivers` (lavapipe) on the Ubuntu runner**.
- Other documented build paths, itemized here because they are hand-maintained: `flake.nix` (official path per BUILD.md) gains `vulkan-headers`/`vulkan-loader`/`glslang`, gated like the Wayland optionals; each BSD job in `build-bsd.yml` either gets loader+glslang packages or pins `ENABLE_VKRENDERER=OFF` explicitly; BUILD.md gains the MSYS2/apt/brew package names. The "degrades cleanly" behavior is itself a work item: `ENABLE_VKRENDERER=ON` must fail-or-disable gracefully at configure time when `glslangValidator` is absent.
- **Validation:** a hidden `--vk-selftest`: create device, run one embedded compute dispatch (buffer fill), read back, compare bytes — runs headless on lavapipe in CI forever after. CI builds both flag states.
- **Risks:** toolchain availability on BSD/MSYS2 → the option degrades to OFF cleanly; volk's dlopen means a missing loader is a runtime no-device, not a link failure.

### Phase 2 — Vulkan compute rasterizer at 1×, software-composited (XL)

**Goal:** the full compute rasterizer on Vulkan, correct and selectable on every platform — including macOS, where this is the first compute-renderer availability at all. No WSI dependency: output rides the existing software compositor and display paths (works even with `ScreenPanelNative`).

- New `src/GPU3D_Vulkan.{h,cpp}` (`VkRenderer3D : Renderer3D`): CPU geometry/span setup copied near-verbatim from `GPU3D_Compute.cpp:633-984`; the 9 dispatch stages recorded into one command buffer with `vkCmdPipelineBarrier2` chains; per-variant loop uses push constants + per-variant texture descriptor; indirect dispatches from the bin-result buffer (`INDIRECT_BUFFER` usage).
- New `src/Vulkan_shaders/*.comp`, forked from `GPU3D_Compute_shaders.h` snippets with the mechanical fixes: renumbered bindings into per-pipeline set layouts, `bool`→`uint` in SSBO structs, location-uniforms→push constants, config `#define`s→`constant_id` (workgroup sizes via `local_size_x_id`, derived globals as spec-constant expressions), `uimageBuffer`→storage texel buffer, output→storage image — plus one *non*-mechanical change: splitting the combined `usampler2DArray CurrentTexture` into separate image + sampler bindings so the 9 wrap-mode samplers can be an immutable-sampler array indexed by push constant (see the descriptor decision in Part 2).
- New `src/GPU3D_TexcacheVulkan.{h,cpp}`: `Texcache<TexcacheVulkanLoader, VkTexHandle>`; array images; the 3×3 wrap-mode sampler matrix as immutable samplers (replaces GL per-draw wrap mutation).
- Modify `src/GPU_Soft.{h,cpp}` + `src/GPU3D.h`: constructor variant accepting an injected `Renderer3D` (today it hardcodes `SoftRenderer3D`, `GPU_Soft.cpp:38`) — **and** break the house `dynamic_cast` idiom: `SoftRenderer::PreSavestate/PostSavestate/SetRenderSettings` cast `Rend3D` to `SoftRenderer3D*` and dereference unchecked (`GPU_Soft.cpp:73-92`; `SetRenderSettings` fires on the first frame via `EmuThread.cpp:888`, the savestate pair via `GPU.cpp:209/311` — instant null-deref with an injected renderer). Add equivalent virtuals to `Renderer3D` (it has none of these today, `GPU3D.h:321-346`) and dispatch through them, which is also what gives the Vulkan renderer its savestate-drain and settings hooks.
- Modify the frontend's "non-software ⇒ OpenGL" hard-wiring **in this phase, not later**: `EmuInstance::usesOpenGL()` (`EmuInstance.cpp:375-379`), `MainWindow::createScreenPanel` (`Window.cpp:817-822`), the `videoRenderer` forcing in `EmuThread::run` (`EmuThread.cpp:121-132, 236-246`), and `VideoSettingsDialog::UsesGL()` all treat any non-software renderer as GL — left alone they would create a GL panel/context for a Vulkan selection, and when GL context creation fails (precisely the machines Vulkan-without-GL serves), silently reset the stored config to software (`Window.cpp:831-841`). Replace with a "display needs GL" predicate that excludes Vulkan-3D-with-soft-output, and stop the failure path from clobbering a Vulkan selection.
- Modify `src/frontend/qt_sdl/EmuInstance.h` (**fix the enum: explicit values for all entries**, add `renderer3D_Vulkan`), `EmuThread.cpp:858-889` (construction arm: `SoftRenderer` parent + `VkRenderer3D`), `Config.cpp` (`3D.Vk.*` keys, clamp), `VideoSettingsDialog.cpp` (radio; platform-gating precedent at :85-87).
- **Output contract:** the compute renderer's final pass writes display-normalized `rgba8` (`result /= vec4(63,63,63,31)`, `GPU3D_Compute_shaders.h:1673-1675`) — but `GetLine` consumers require the software renderer's packed format: 6-bit R/G/B in bits 0-21 and 5-bit alpha at bits 24+ (`GPU3D_Soft.cpp:562`; `DrawBG_3D` tests `(c >> 24) == 0` and blends in 6-bit space, `GPU2D_Soft.cpp:371-382`; `DoCapture` extracts the same lanes, `GPU_Soft.cpp:310-399`). Phase 2 therefore adds a readback variant of the final pass emitting the un-normalized packed u32 into a host-visible ring buffer (the deko3d original's native format — cf. the commented-out packing at `GPU3D_Compute_shaders.h:1665-1668`); the `rgba8` image path returns in Phase 4. `FinishRendering()` waits the frame's timeline value; `GetLine(line)` (with `RenderXPos` scroll) serves from the mapped ring, waiting once on first use per frame. **Timing honesty:** first `GetLine` arrives at the next frame's VCount==0 — ~48 scanlines (≈3 ms) after submission, not the ~240-scanline `FinishRendering` window; GPU+copy time beyond that stalls the emu thread. Acceptable at 1× on real GPUs; measure on lavapipe and on MoltenVK (whose timeline host-waits are layered over MTLSharedEvent and carry extra latency); the prepared mitigation is chunked line-range readback signaled at increasing timeline values, recovering the soft rasterizer's incremental overlap. The `RenderFrameIdentical` early-out submits nothing on some frames — the ring/wait logic must keep serving the previous slot instead of waiting on a value that will never signal. Scale locked to 1× this phase (the soft compositor is 1×-only, and 1× native accuracy is precisely this renderer's value). `PreSavestate`/`RestartFrame` = wait/drop in-flight work via the new `Renderer3D` virtuals.
- **Validation:** (a) sync-validation + GPU-assisted validation clean over a stress matrix (shadow-heavy: Zelda PH/ST; translucency; SM64DS); (b) **golden-hash diffing against GL `ComputeRenderer3D`** — identical algorithm, so per-frame output hashes should match or diffs be explainable (the GL twin hashes `rgba8`, this path hashes the packed format — normalize one side before comparing) — plus spot pixel-diffs vs `SoftRenderer3D`; (c) a captured polygon-RAM snapshot replayed through the renderer on lavapipe in CI; (d) savestate loops, VCOUNT-write titles (`RestartFrame`), hot-swap soak (Vulkan↔soft↔GL) under `renderLock`; (e) measured first-`GetLine` wait times on lavapipe and MoltenVK.
- **Risks:** barrier bugs (sync validation from day one, per-pass labels, RenderDoc); MoltenVK quirks (design is SSBO-atomics-only — Metal's solid path; `Init()` failure falls back to software via existing `GPU::SetRenderer` plumbing; timeline host-wait latency measured, chunked readback as fallback); emu-thread stall from the ~3 ms first-`GetLine` deadline on slow GPUs; descriptor/texture lifetime — in-flight sets referencing evicted texcache images require deferred destruction tied to frame retirement.

### Phase 3 — Vulkan presentation path (M)

**Goal:** swapchain-backed display, decoupled from renderer risk: initially presents software-composited RAM frames (the Vulkan analogue of `Screen.UseGL`), so it is testable with any renderer, and is independently useful on macOS where GL is deprecated.

- New `src/frontend/qt_sdl/VulkanPresenter.{h,cpp}` implementing a `GL::Context`-shaped interface: `Create(WindowInfo)`→surface+swapchain (FIFO default), `SwapBuffers`→acquire/draw screens/present with `OUT_OF_DATE`/`SUBOPTIMAL` handled on both acquire *and* present, `SetSwapInterval`→present-mode change (full chain recreation; IMMEDIATE/MAILBOX for fast-forward where available), `CreateSharedContext`→same `VkDevice`, per-window swapchain (4-window case). Cheap-resize vs full-recreate split per Dolphin's `VKSwapChain`.
- New `ScreenPanelVK` in `Screen.{h,cpp}`: sibling of `ScreenPanelGL`, reuses `ScreenPanel` (layout/OSD/input), with `getWindowInfo()` + the native-widget setup hoisted from `ScreenPanelGL` into the base; RAM frames→staging ring→sampled image; screen quads via `screenMatrix[]`; OSD textures via staging. Teardown on `QPlatformSurfaceEvent::SurfaceAboutToBeDestroyed`; resize signaled UI-side, executed render-side, `OUT_OF_DATE` treated as truth; multiply by `devicePixelRatio`. Two Wayland inversions to design for now even though Wayland ships gated: Wayland surfaces generally never return `OUT_OF_DATE` (`currentExtent` is `0xFFFFFFFF` — resize must work from the Qt-side signal alone), and Mesa's Wayland WSI blocks acquire/present for occluded windows in FIFO mode — with up to 4 windows presented sequentially from the emu thread, one hidden window must not stall the rest (skip-present on occlusion; MAILBOX or `VK_EXT_swapchain_maintenance1` where available).
- Modify `Window.cpp:811-916` (`createScreenPanel` three-way), `EmuInstance.cpp:375-434` (generalize `usesOpenGL/initOpenGL/setVSyncGL/makeCurrentGL/drawScreen` behind a presenter-kind switch; GL borrow protocol not needed for Vulkan), `EmuThread.{h,cpp}` (generalize `msg_InitGL/msg_DeInitGL`), `Config` (`Screen.UseVK`).
- **Wayland gated off at first ship** (falls back to native/GL panel); the child-`QWindow` + `createWindowContainer` fallback is the prepared alternative (see Phase 5).
- **Validation:** resize/rotate/fullscreen/DPI-change stress; vsync + fast-forward toggles; 2–4 windows; multi-instance; validation clean; X11/Windows/macOS-MoltenVK.

### Phase 4 — Full Vulkan compositor: 2D, capture, hi-res, zero readback (XL)

**Goal:** `VkRenderer : Renderer` parent (Vulkan 2D units + Phase-2 3D), eliminating the CPU round-trip; hi-res scaling with `HiresCoordinates`; GPU display capture. Becomes the default Vulkan mode. Mode selection stays one `renderer3D_Vulkan` enum value (reserved with these semantics in the Phase-2 enum fix): the full compositor is used when a Vulkan display is active; `Init()` failure or a non-Vulkan display drops automatically to Phase 2's soft-composited mode, mirroring the existing software fallback in `GPU::SetRenderer` — no second enum value, no TOML churn.

- New `src/GPU_Vulkan.{h,cpp}`: port of `GPU_OpenGL.cpp` — batched scanline-range compositing, final pass into a double-buffered 2-layer array image, `DoCapture`/`DownscaleCapture` pipelines, `SyncVRAMCapture` via download buffer + fence (replaces `glReadPixels`; small but correctness-critical — reuse the exact `SyncAllVRAMCaptures`-before-swap ordering `GPU::SetRenderer` enforces).
- New `src/GPU2D_Vulkan.{h,cpp}`: mechanical port of the blackmagic3 `GPU2D_OpenGL` compositor/layer/sprite pipelines — **the largest untranslated surface in the whole plan**; no redesign, pass-for-pass.
- Modify `src/GPU.h` + `Screen.cpp:1150` + `EmuThread.cpp`: replace the `*(GLuint*)` convention with a small tagged output handle; `ScreenPanelVK` gains the zero-copy path (same device: image view + timeline value). (The `useOpenGL` gating decoupling already landed in Phase 2.)
- Modify `GPU3D_Vulkan.cpp`: drop readback when parented by `GPU_Vulkan`; enable the ScaleFactor/TileSize matrix (spec-constant re-specialization; `NeedsShaderCompile` stays default-false), with the offered ScaleFactor clamped from the device limits recorded in Phase 1 (`maxComputeWorkGroupInvocations`, `maxComputeWorkGroupCount[2]`, `maxStorageBufferRange`, available device memory — GL 4.3-era minimums must not be assumed); capture-texture sampling for capture-as-texture polygons.
- **Validation:** scale sweep 1–16× incl. TileSize transitions and limit-clamp behavior on a minimal-limits device; display-capture torture titles (dual-screen 3D via capture; capture→texture feedback loops) with **VRAM-content diff vs the GL renderer** after `SyncVRAMCapture`; golden-hash vs GL compute at multiple scales; multi-window; hot-swap soak.

### Phase 5 — Hardening & release (M)

- MoltenVK bundling in macOS packaging (volk dlopen keeps non-Vulkan macOS working); enable the Vulkan radio on macOS (per-platform gating precedent: `VideoSettingsDialog.cpp:85-87`).
- Wayland enablement after dedicated testing — either the existing native-`wl_surface` path (the panel is already a Qt-never-paints native window, same surface EGL uses today) or the child-`QWindow` fallback; whichever survives testing on GNOME/KDE/wlroots ships.
- `VkPipelineCache` persistence per-deviceUUID with header validation + corruption fuzzing.
- Perf pass: overlap 3D compute with 2D composite on the timeline; evaluate async-compute queue (parallel-rdp precedent); frame-time telemetry vs GL compute (expect parity or better; Vulkan's win is frame pacing and macOS existence, not raw throughput at DS resolutions).
- Docs (BUILD.md, `docs/vulkan-renderer.md` maintainer notes: barrier structure, binding maps); CI with `ENABLE_VKRENDERER=ON` as default where toolchain permits; driver-quirk switches only as encountered — no speculative quirk table.

---

## Part 4 — Cross-cutting risks & non-goals

| Risk | Mitigation |
|---|---|
| **Upstream acceptance**: maintainers' direction is OpenGL; strict no-AI-code policy killed the one near-complete Vulkan PR (#2700). | Treat upstreaming as a separate, explicit decision. The plan's value stands in a fork (precedent: melonDualDS shipped exactly this on Android). Discuss with maintainers before investing Phase-4 effort upstream. |
| **Wayland surface ownership** (Dolphin's multi-year trap). | Gated at first ship; two prepared fallbacks; presentation is isolated in `ScreenPanelVK`/`VulkanPresenter`, nothing core-side depends on it. |
| **Sync/barrier bugs in the compute port**. | Sync validation + GPU-AV from day one; per-pass debug labels; golden-hash oracle against the GL twin makes correctness regressions cheap to detect. |
| **In-flight GPU resources vs. texcache eviction / renderer teardown**. | Deferred destruction keyed by timeline value (per-frame trash lists); device wait-idle on Reset, hot-swap, and the savestate brackets. |
| **Two GLSL dialect copies of the rasterizer** (GL + Vulkan). | Deliberate, bounded tech debt: the snippet-assembled shader structure keeps hand-porting algorithm fixes tractable. Revisit single-source (Vulkan-GLSL + SPIRV-Cross→GL) only if divergence hurts. |
| **2D compositor churn** (blackmagic3 is fresh, more graphical work announced). | Phase 4 is a mechanical pass-for-pass port pinned to a tag; Phases 1–3 ship value regardless. |
| **Enum/config compatibility** when `VKRENDERER` and `OGLRENDERER` flags differ across builds. | Fix the enum with explicit values in Phase 2; clamp unknown values to software at load. |

**Non-goals:** porting the classic GL renderer; ubershaders/async pipeline compilation; descriptor indexing/bindless; a cross-API RHI; Android (needs a 1.1+extensions variant — nothing here contorts for it, and the SSBO-atomics-only design keeps the door open); replacing the GL or software renderers.

---

## Part 5 — Key references

- Renderer plumbing: `src/GPU.h`, `src/GPU.cpp` (timing at :1163-1166, :1289), `src/GPU3D.h:321-346`
- Compute renderer: `src/GPU3D_Compute.{h,cpp}`, `src/GPU3D_Compute_shaders.h`; texcache `src/GPU3D_Texcache.*`
- Presentation: `src/GPU_OpenGL.{h,cpp}`, `src/frontend/qt_sdl/Screen.cpp`, `src/frontend/duckstation/`
- Prior art: melonDS #984 / PR #990 (interface), PR #2041 (compute renderer), PR #2700 (closed Vulkan port), PR #2642 (GL 3.1 lowering), melonDualDS 0.7.0 RCs (shipped Android Vulkan backend)
- External: paraLLEl-RDP (github.com/Themaister/parallel-rdp); Dolphin `VKSwapChain.cpp`; DuckStation `src/util/vulkan_device.cpp`; vkguide.dev; vk-bootstrap; volk; VMA; Zeux on robust pipeline caches (zeux.io/2019/07/17/serializing-pipeline-cache)
