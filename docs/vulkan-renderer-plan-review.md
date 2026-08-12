# Adversarial Review — `docs/vulkan-renderer-plan.md`

*Target: the Vulkan renderer research & implementation plan on branch `claude/vulkan-renderer-plan-gfv41x` (commit `52a1acf`), reviewed against the tree at `d3cd616` — the exact commit the plan claims to be grounded in.*

## Method

Nine independent reviewers attacked the plan, each confined to one lens: core renderer plumbing, the compute renderer and its shaders, frontend/build, timing & threading, Vulkan spec correctness, MoltenVK & hardware portability, build/toolchain/CI/licensing, external prior-art claims, and plan structure. Each lens's findings then went to a separate skeptic whose instructions were to *kill* them — re-opening every cited file, spec section and URL independently, and discarding anything that was misread, quoted out of context, already mitigated in the plan, or merely line-number drift. A completeness critic then asked what all nine had missed.

**89 findings raised → 55 survived refutation → 7 added by the critic → 24 after dedupe.**

Two findings that reached the report unrefuted (#1, #13) were subsequently verified by hand; both are confirmed below. One (#19) remains marked PLAUSIBLE.

| Severity | Count |
|---|---|
| CRITICAL | 1 |
| MAJOR | 17 |
| MINOR | 6 |

---

## Verdict

**The research is excellent, the judgement calls are mostly right, and the plan as written would not survive contact with an implementer.** All three are true at once, and separating them is the point of this review.

Part 1 is the strongest part of the document. Across ten lenses, essentially every file-and-line citation in the research section was opened and checked, and they hold. Reviewers found mis-statements of *meaning* — the threading contract, the fallback semantics, what a commented-out shader block does — but almost no fabricated citations. The Part 2 judgement calls (port `ComputeRenderer3D` not the classic GL renderer; no RHI; no `QVulkanWindow`; no async pipeline infrastructure; no runtime GLSL; ship the rasterizer under the software compositor before touching WSI) are correct and well argued. Keep all of them.

The failure is concentrated and has a single root cause: **the plan repeatedly treats behaviour the OpenGL driver provides for free as if it were a property of the code being ported.** GL's implicit in-order buffer guarantee is why the compute renderer can rewrite single-instanced SSBOs every frame with no fences — which makes "CPU setup copied near-verbatim" plus "two frames in flight" a data race (finding 1), the one thing nine reviewers walked past. GL's driver-side compilation is why "eliminating runtime compilation entirely" reads plausible. GL's per-draw state mutation is why the Phase-4 2D port looks "mechanical". GL's `glBufferSubData`-into-a-live-buffer is why the memory profile looks "tiny and static" when it is ~2.6 GiB at 16× before any frame versioning, with a single buffer 6× over Vulkan's guaranteed `maxStorageBufferRange` floor.

Would it work if followed literally? No, and it fails early:

- **Phase 1 does not configure on Windows or macOS** — vcpkg's `glslang` port ships no tools without the `tools` feature.
- **Phase 2 cannot meet its own macOS goal or run its own macOS validation**, because nothing before Phase 5 puts a Vulkan implementation on a Mac.
- **Phase 2's advertised safety net does not exist** for the parent/child pairing Phase 2 actually builds: `SoftRenderer::Init()` returns `true` unconditionally and never Inits its children.
- **Phase 2's correctness gate** depends on a capture/replay + golden-hash harness that is a substantial standalone project priced at zero, scheduled to run in the one CI job already marked `continue-on-error: true` that has no run step at all.
- **Phases 2 and 3 are declared parallel** while editing the same four frontend functions.
- **Phase 4's one-enum auto-degrade** is not delivered by the plumbing it cites.

The two external existence proofs that justify the XL sizings do not survive: PR #2700 was self-closed by its author after ~24 hours with zero maintainer comment (there is no "no-AI-code policy" in the repo at all), and melonDualDS lists Vulkan and compute as two *separate* renderers. What remains is paraLLEl-RDP — different console, different renderer, written by a graphics specialist. The two XL phases are currently sized as ports. They are ports **plus** an unwritten synchronization design **plus** an unwritten test harness **plus** an unwritten macOS packaging subsystem.

The 18 CRITICAL/MAJOR findings are not padding — they collapse into four root causes: GL-implicit-behaviour assumptions; an absent error and ownership model; validation and macOS packaging priced at zero; and an evidence base that does not hold.

### Recommendation

Do not discard this plan. Revise it in this order:

1. **Fix the frames-in-flight and buffer-versioning model first** — it simultaneously changes the memory table, the descriptor design, the Phase-4 clamp and the perf story.
2. **Move loader + MoltenVK + ICD packaging and `VkPipelineCache` persistence from Phase 5 into Phase 1**, since Phase 2's stated goals depend on both.
3. **Make the validation harness a real, sized Phase 1.5** with its own deliverables — or downgrade every "golden-hash" claim to "eyeball" and say so.
4. **Delete both feasibility existence proofs and re-estimate honestly.**
5. **Write down the device/surface ownership model** before Phase 1 freezes the `VulkanDevice` API.

The plan's most persuasive quality is its confidence. That is also the thing most likely to carry an implementer six months in before any of the above surfaces.

---

## What holds up

**The research (Part 1) is close to publishable.** `GPU.cpp:1163-1166`, `GPU3D.h:321-346`, `GPU_Soft.cpp:115`, `Screen.cpp:868-878/1150`, `EmuThread.cpp:858-889`, `src/CMakeLists.txt:97-109`, `GPU3D_Compute.cpp:633-984` — all real, all saying what the plan says. The `WindowInfo`-already-carries-the-handles observation, the `*(GLuint*)` type-unsafe seam, the positional-enum-under-`#ifdef` config hazard, and the "GL renderers return `nullptr` from `GetLine`" contract are correctly identified and genuinely load-bearing. That grounding is what made this review productive rather than speculative.

**`ComputeRenderer3D` as the only 3D port target is the right call, and nothing here touches it.** Verified: SSBO atomics, no image atomics, no int64, the `ballotARB` path really is `#if 0`, a closed 256-variant space, a genuinely API-generic texcache template. The classic GL renderer really does lean on stencil bit-plane tricks and `GL_MAX` blend hacks that translate badly.

**Other decisions that survive intact:** no cross-API RHI (and the three named seams are exactly the right three); refusing `QVulkanWindow`; refusing ubershaders and async pipeline compilation; build-time GLSL→SPIR-V with no runtime shader compiler; vendoring volk/vk-bootstrap/VMA; sync validation as the highest-value tool for this port.

**The Phase-2 shape is the plan's best structural idea.** Shipping the Vulkan rasterizer under the *software* compositor, with no WSI dependency, before touching swapchains, isolates rasterizer risk from presentation risk. Most Phase-2 findings below are about things bolted to that idea, not the idea itself. It should survive the revision.

**The plan is honest where it could have been vague.** It flags the `RenderFrameIdentical` never-signalled-value hazard, the `dynamic_cast` null-deref, the enum/TOML trap, the Wayland `currentExtent == 0xFFFFFFFF` inversion, and the Mesa occluded-window FIFO stall. It labels the two-GLSL-dialect problem as deliberate tech debt rather than pretending it away.

---

## Findings

### 1. CRITICAL · WRONG · "Two frames in flight" plus "CPU setup copied near-verbatim" is a data race

*Part 2 "Descriptors / sync" and "Loader / bootstrap / memory" rows; Phase 2 bullet 1.*

Every buffer the compute renderer touches is single-instanced and reused every frame, and the code relies entirely on GL's implicit in-order guarantee to make that safe. CPU-written per frame: `glBufferSubData` into `YSpanSetupMemory`, `YSpanIndicesTextureMemory`, `RenderPolygonMemory` (`GPU3D_Compute.cpp:974-983`) and `MetaUniformMemory` (`:1055-1057`). GPU read-modify-written across the 9-stage chain, also single-instanced: `TileMemory[3]`, `FinalTileMemory`, `BinResultMemory`, `WorkDescMemory`, `XSpanSetupMemory` — all allocated once in `SetRenderSettings` (`:361-397`). There is no `glFenceSync`, no `glFinish`, no `glClientWaitSync`, no `glMapBufferRange` anywhere in the file; `ComputeRenderer3D` does not override `FinishRendering()`; every buffer is `GL_DYNAMIC_DRAW` with no orphaning. Vulkan provides no equivalent guarantee.

With two frames in flight, frame N+1's staging writes and stage-1 dispatch land on memory frame N is still reading: write-after-read corruption presenting as intermittent geometry from the previous frame, wrong toon/fog tables, garbage tiles — and easily mistaken for a barrier bug and chased for weeks. Phase 2 survives only by accident (the `GetLine` readback drains frame N before N+1 is submitted); Phase 4 explicitly "drops readback when parented by `GPU_Vulkan`", removing the only thing making it safe.

Separately, the std140 `MetaUniform` block (`GPU3D_Compute_shaders.h:355`, binding 0) is one of these per-frame-rewritten buffers and appears nowhere in the plan — not in the Part 1.2 friction list, not in the descriptor decision, not in the Phase 1 limits checklist.

**Correction.** State the frames-in-flight number as a design constraint with named consequences. Add a Phase-2 work item: every CPU-written buffer becomes a per-frame-slot suballocation from the staging ring Phase 1 budgets but never wires up (with `MetaUniform` as `UNIFORM_BUFFER_DYNAMIC` or a per-slot descriptor aligned to `minUniformBufferOffsetAlignment`); every GPU-side intermediate is either duplicated per in-flight frame or the renderer is documented as strictly one-frame-in-flight with a timeline wait at the top of `RenderFrame`. Recompute the Phase-4 clamp from N × the corrected footprint. Replace the "tiny and static" parenthetical with a real table.

### 2. MAJOR · WRONG · Tile SSBO footprint is understated 3×, and one tile buffer is 6× the guaranteed `maxStorageBufferRange`

*Phase 1 VulkanDevice bullet; Phase 4 clamp bullet; Part 2 memory rationale.*

There are three tile memory layers, not one: `GPU3D_Compute.h:83-91` defines `tilememoryLayer_Color/Depth/Attr` with `tilememoryLayer_Num = 3`, and `GPU3D_Compute.cpp:361-365` loops allocating `4*TileSize*TileSize*MaxWorkTiles` for each. At 16× (TileSize 32, TilesPerLine 128, TileLines 96, MaxWorkTiles 196608) each layer is 768 MiB → **2.25 GiB for the tile layers alone**, plus `FinalTileMemory` at 288 MiB (`:367-368`) plus `XSpanSetupMemory` — roughly **2.6 GiB** before texcache, framebuffers and swapchain. The plan says "~800 MB".

Worse, a single `TileMemory[i]` at 768 MiB is **6× Vulkan's guaranteed `maxStorageBufferRange` floor of 2²⁷ = 128 MiB** — a hard scale cap above roughly 6–7× *independent of available memory*, which the plan never mentions. `ScaleFactor` range is `{1,16}` (`Config.cpp:82`).

**Correction.** Replace "~800 MB" with the computed ~2.6 GiB and show the derivation. Note the `maxStorageBufferRange` cap explicitly — either split the tile buffers or clamp on it. Turn the "tiny and static" claim into a memory table parameterized by ScaleFactor and frames-in-flight.

### 3. MAJOR · UNSUPPORTED · "Eliminating runtime compilation entirely" is false

*Part 2 "Shader toolchain" and "Pipeline cache / validation" rows; Part 1.2; Phase 4.*

`vkCreateComputePipelines` **is** runtime compilation on every driver — specialization constants are consumed at pipeline-creation time, not module-creation time. On MoltenVK it is worse: SPIR-V→MSL conversion happens at `vkCreateShaderModule` and again per pipeline, and MoltenVK's own runtime guide documents pipeline-cache serialization specifically to avoid re-converting. There are 33 programs (`GPU3D_Compute.cpp:78`), with workgroup size itself specialization-driven (`local_size_x = TileSize`, `GPU3D_Compute_shaders.h:1063, :1277`), so a scale change genuinely invalidates every pipeline.

The plan deletes the existing incremental `ShaderCompileStep`/OSD machinery (`EmuThread.cpp:891-905`) — which exists precisely to keep `SetRenderSettings` (fired on the first frame after any settings change, inside the `renderLock` block at `EmuThread.cpp:235-251`) from freezing the UI — and schedules the one documented replacement mitigation into Phase 5 as "politeness". Result: cold start and every hi-res slider change performs 33 synchronous pipeline builds on the emu thread with no time-slicing and no progress OSD. That will be reported as a hang.

**Correction.** Rewrite as "no runtime *GLSL* compilation; pipeline creation remains a real init and scale-change cost." Keep the time-slicing machinery, re-targeted at pipeline creation. Move `VkPipelineCache` persistence to Phase 1 and reclassify it as required. Add a Phase-2 measurement of cold- and warm-cache pipeline build time on desktop and MoltenVK.

### 4. MAJOR · INCOMPLETE · macOS has no Vulkan implementation until Phase 5, yet Phase 2 requires one

*Phase 5 bullet 1; Part 2 baseline row; contradicted by Phase 2's goal and validation item (e).*

Four defects, found by four lenses:

- **Ordering.** Phase 2's goal says "correct and selectable on every platform — including macOS" and its validation demands "measured first-`GetLine` wait times on … MoltenVK". Nothing in Phases 1–4 puts a Vulkan implementation on a Mac.
- **`VK_KHR_portability_enumeration` is a *loader* extension.** MoltenVK's `MVKExtensions.def` lists only `KHR_portability_subset`, and MoltenVK "does not load Vulkan Layers on its own". volk's `__APPLE__` path dlopens `libvulkan.dylib` → `libvulkan.1.dylib` → `/usr/local/lib` → `libMoltenVK.dylib`, so shipping only `libMoltenVK.dylib` gives an ICD-less driver where requesting `portability_enumeration` returns `VK_ERROR_EXTENSION_NOT_PRESENT` — hard instance-creation failure — and where `VK_LAYER_KHRONOS_validation` cannot load at all.
- **Packaging.** `tools/mac-libs.rb` discovers dylibs only via `otool -L` on the built binary (`:20-38`), so it cannot see a dlopen'd loader; `MACOS_BUNDLE_LIBS` defaults OFF (`qt_sdl/CMakeLists.txt:244-253`); `build-macos.yml` never sets it and codesigns with `--deep` at `:99`; release presets set `BUILD_STATIC=ON`. Someone must add MoltenVK acquisition, an install rule into `Contents/Frameworks`, a search path volk's leaf-name dlopen actually consults, a `MoltenVK_icd.json`, and codesign ordering.
- **Deployment target.** `CMakeLists.txt:12` and the `x64-osx-1015` overlay triplet target macOS 10.15, but MoltenVK 1.4.1 requires macOS 11 and 1.4.2 requires macOS 12 — dyld refuses a dylib whose `minos` exceeds the running OS.

Also: the proposed `flake.nix` gating `optionals (!isDarwin)` gives Nix macOS no Vulkan at all, and on Nix Linux `volkInitialize` fails at runtime unless `vulkan-loader` joins the `qtWrapperArgs` `LD_LIBRARY_PATH` prefix that already exists for dlopen'd libpcap/wayland (`flake.nix:60-64`).

macOS is the plan's entire strategic justification. As sequenced, the prize is unreachable until the last phase, the phase meant to prove correctness on it cannot run there, and the validation tooling the plan calls its highest-value asset is unavailable there by construction. And Phase 5 is sized (M).

**Correction.** Move loader + MoltenVK + ICD manifest + bundling + codesigning into Phase 1 as a named, sized deliverable (weeks, not a line), or delete macOS from Phase 2's goal and validation. Ship a real loader alongside MoltenVK; if shipping MoltenVK bare, drop the `portability_enumeration` request or enable it only when present. Pick a MoltenVK version, state the resulting minimum macOS, and reconcile it with `CMakeLists.txt:12` and the `x64-osx-1015` triplet. Fix the `flake.nix` bullet for both Darwin and the Linux runtime library path.

### 5. MAJOR · WRONG · The "free software fallback" does not exist for the pairing Phase 2 builds

*Part 1.1 hot-swap bullet; Phase 2 Risks; Phase 4 goal; Part 2 baseline rationale.*

All four halves verified:

- **It does not reach the Phase-2 pairing.** `GPU.cpp:323/337` call `Init()` only on the top-level `Renderer`, and `SoftRenderer::Init()` is `bool Init() override { return true; }` (`GPU_Soft.h:34`) — it constructs its children at `GPU_Soft.cpp:36-38` and never Inits them. `GLRenderer` is the only renderer that propagates (`GPU_OpenGL.cpp:237-239`). So a `SoftRenderer` parent wrapping a failing `VkRenderer3D` installs successfully, and the failure surfaces as a crash or hang at the first `RenderFrame()`/`GetLine()` — on exactly the unsupported hardware the fallback was meant to protect.
- **It is silent.** `GPU::SetRenderer` is `noexcept` with `// TODO: report error to platform` at `:328-331`; there is no renderer-kind accessor; `EmuThread.cpp:878` has already latched `lastVideoRenderer`. The user gets software rendering with the UI still showing Vulkan and no retry.
- **Phase 4's auto-degrade is not delivered.** The fallback at `GPU.cpp:334-338` hardcodes `std::make_unique<SoftRenderer>(NDS)` — a failing `VkRenderer` lands in *full software*, losing the Vulkan 3D renderer entirely, not in Phase 2's mode.
- **Runtime device loss appears nowhere** in the plan or the risk table; the `Renderer` interface has no runtime error return.

The plan's own hard requirement — that excluded-hardware users be told explicitly rather than fail silently — is unimplementable on top of the plumbing it calls free.

**Correction.** Three new core work items: (1) propagate `Init()` from `SoftRenderer` to its children; (2) add a renderer-kind accessor or out-param on `GPU::SetRenderer` so the frontend can learn the fallback fired, un-latch `lastVideoRenderer`, and reflect reality in the UI; (3) add a real degrade hook for Phase 4 (fallback-factory parameter, or two-stage construction in `updateRenderer`, currently ending in `default: __builtin_unreachable();` at `EmuThread.cpp:864-877`). Add a device-loss row to the risk table with a stated policy — which needs a runtime error path on the `Renderer` interface that does not exist today.

### 6. MAJOR · INCOMPLETE · There is no `VkDevice`/`VkInstance` ownership model

*Phase 2 validation (d); Phase 3 presenter bullet and validation; Part 4 risk table.*

Three scenarios need incompatible answers:

- **Hot-swap.** `GPU::SetRenderer` does `Rend = std::move(renderer)` at `:322`, destroying the old renderer *before* Init and with no wait-idle. If `VkRenderer3D` owns the device/VMA allocator, switching Vulkan→software leaves Phase 3's still-live `ScreenPanelVK` presenter holding a dangling device — a crash on the next `drawScreen`. Note `ScreenPanelGL::drawScreen` (`Screen.cpp:1104-1264`) is not among the `renderLock` holders, so "under `renderLock`" does not cover the presenter.
- **Instance/process singleton.** If that is the answer, the plan must say so and define teardown ordering, since two Vulkan renderers briefly coexist during `SetRenderer`.
- **Multi-instance.** `main.cpp:75-76/117-145` and `EmuInstance.h:32` allow up to 16 `EmuInstance`s, each with its own emu thread. Either 16 `VkDevice`s (16× validation overhead, 16 staging rings and descriptor pools, and 16 concurrent writers to one per-deviceUUID pipeline-cache file), or one shared device — which then requires external synchronization of a shared `VkQueue` across up to 16 emu threads, a submission serialization point the plan never mentions. volk additionally documents `volkLoadDevice` as single-device; multiple devices need `volkLoadDeviceTable`.

"Multi-instance" and "hot-swap soak" appear only as Phase-3 validation line items, but they are design decisions that determine the Phase-1 `VulkanDevice` API. Discovering them during Phase 3 means re-plumbing Phase 1 after Phases 2 and 3 have both built on it.

**Correction.** Add an explicit ownership section to Phase 1, before the `VulkanDevice` API is written: instance/device scope, who outlives whom across `SetRenderer`, whether `volkLoadDeviceTable` is needed, how a shared queue is serialized, how the pipeline-cache file is guarded against concurrent writers. Add wait-idle-before-teardown to the `SetRenderer` path.

### 7. MAJOR · RISKY · Emu-thread-only presentation with no-op borrow hooks deadlocks the UI

*Part 2 "WSI / Qt integration" row; Phase 3.*

- **Lost synchronization.** `Window.cpp:813-815` deletes the old panel *outside* the emu-thread park, and the park at `:826-828/:843-844` brackets only `createContext` and only when `windowID != 0`. So swapchain/surface creation and destruction happen on the UI thread while the emu thread is using the queue and swapchain — undefined behaviour on every window add/remove and every video-settings change (`Window.cpp:2263-2320`).
- **Hard deadlock.** If the emu thread blocks in `vkAcquireNextImageKHR` or `vkQueuePresentKHR` — which the plan itself predicts for occluded Wayland windows in FIFO — it never reaches `handleMessages` (`EmuThread.cpp:563-566, 661-666, 692-696`), so the UI thread's `waitMessage` blocks forever and the application becomes unkillable through its own UI, including the teardown paths that would fix it.

Today's GL path has the same shape but keeps the borrow park as the escape valve. The plan removes it and calls that a simplification.

**Correction.** Do not declare the borrow hooks no-ops. Either keep a borrow/park protocol for Vulkan — a small generalization of what exists — or write down an explicit device-and-surface ownership model with named cross-thread synchronization for every create/destroy/resize path, including the panel delete at `Window.cpp:813-815`. Require a bounded/timeout or skip-present policy so a blocked acquire cannot starve `handleMessages`, and state that the presenter must take `renderLock`.

### 8. MAJOR · INCOMPLETE · The validation harness Phase 2 rests on is unbuilt, unsized, and lands in a job whose failures are ignored

*Phase 1 Validation; Phase 2 Validation (b) and (c); Phase 4; Phase 5; Part 4 risk row 3.*

None of this infrastructure exists and none is costed. `.github/workflows/build-ubuntu.yml` declares `continue-on-error: true` on the build job (line 19) across an x86_64/aarch64 matrix, installs no Vulkan packages, and has **no run or ctest step at all** — configure, build, install, upload artifact, done. Same for macOS, Windows and BSD. There is no `add_test` or `enable_testing` anywhere outside `src/teakra/CMakeLists.txt:79`. `CLI.cpp:37/52` shows argument parsing runs after `QApplication` is constructed (`main.cpp:332`), so even `--vk-selftest` needs design work to run headless.

The golden-hash oracle requires: deterministic polygon-RAM capture hooked into GPU3D, a serializer, a replay driver decoupled from NDS timing, a hash-compare mode in two renderers, format normalization between them, legally distributable test content, and CI infrastructure that runs a built binary. Phase 4's VRAM-content diff and Phase 5's cache-corruption fuzzing are two more unscoped tools.

The plan rates sync/barrier bugs as its top technical risk and names this oracle as the mitigation that "makes correctness regressions cheap to detect".

**Correction.** Make it a real phase — Phase 1.5 — with its own deliverables and sizing. Add `mesa-vulkan-drivers` and `libvulkan-dev` to *both* Ubuntu matrix arches and add an actual run step. Remove `continue-on-error: true`, or move the Vulkan tests into a blocking job. If it will not be built, downgrade every "golden-hash" and "VRAM-content diff" claim to "manual comparison" and re-rate the barrier-bug risk accordingly.

### 9. MAJOR · WRONG · `RestartFrame` semantics are backwards

*Phase 2 "Output contract", final sentence.*

`RestartFrame` does not mean discard; it means **re-render**. `SoftRenderer3D::RestartFrame` (`GPU3D_Soft.cpp:1768-1772`) calls `SetupRenderThread`, which waits, resets and *re-issues* the frame (`:48-88`), then `EnableRenderThread` posts (`:90-95`) and the thread renders again (`:1774-1805`). The call site is `if (GPU3D.AbortFrame) { Rend->Restart3DRendering(); GPU3D.AbortFrame = false; }` (`GPU.cpp:1191-1194`).

The abort path also has a second behaviour the plan never mentions: while `AbortFrame` is set (`GPU.cpp:1389`), `SoftRenderer3D::GetLine` returns a **black line without waiting** (`GPU3D_Soft.cpp:1807-1814`) and `FinishRendering` skips its wait (`:1740-1743`). And `AbortFrame` is savestated (`GPU3D.cpp:535`), so it can be true immediately on state load, before any `RenderFrame` has been submitted.

A Vulkan renderer that drops work on `RestartFrame` destroys the only copy of the frame the readback ring must serve — black or stale 3D on every VCount-write title, and combined with the plan's own never-signalled-value hazard, a permanent hang once `GetLine` waits on a dropped submission.

**Correction.** Distinguish the three operations. `PreSavestate` = drain. `RestartFrame` = re-submit from already-uploaded geometry state, mirroring `SetupRenderThread`'s wait-then-re-issue. Add the third behaviour: while `AbortFrame` is set, `GetLine` returns a valid black line *without* waiting and `FinishRendering` skips its wait. State that `AbortFrame` is savestated and make it a Phase-2 test case.

### 10. MAJOR · WRONG · Phase 2 is neither inert with the flag off nor parallel with Phase 3

*Part 3 preamble; Phase 2 bullets 4 and 5.*

- `GPU_Soft.cpp` and `GPU3D_Soft.cpp` are compiled unconditionally (`src/CMakeLists.txt:37,41`), unlike the GL block at `:97-115`. Phase 2's refactor of `SoftRenderer`'s savestate/threading handoff (`GPU_Soft.cpp:71-92`, called from `GPU.cpp:209/311`) ships to **every** user regardless of the flag.
- The new `Renderer3D` virtuals land on the GL stack too. `GLRenderer::SetRenderSettings` uses the same `dynamic_cast` idiom with two *mutually incompatible* 3D signatures — `ComputeRenderer3D::SetRenderSettings(ScaleFactor, HiresCoordinates)` vs `GLRenderer3D::SetRenderSettings(ScaleFactor, BetterPolygons)`, selected by an `IsCompute` flag (`GPU_OpenGL.cpp:322-341`). A unified virtual requires rewriting both GL 3D renderers — landing on the stack 100% of hi-res users run.
- Phase 2 bullet 5 and Phase 3 bullet 3 both rewrite `EmuInstance::usesOpenGL` (`:375-434`), `MainWindow::createScreenPanel` (`Window.cpp:811-866`), EmuThread's renderer arm (`:229-251`) and `VideoSettingsDialog::UsesGL`. Phase 4 then asserts the decoupling "already landed in Phase 2", so if Phase 3 lands first the ordering breaks.

**Correction.** Replace the preamble sentence with the truth: Phase 2 refactors shared core code and is *not* inert with the flag off — it requires GL-stack and software-stack regression testing (scale change, better-polygons toggle, hi-res-coords toggle, savestate round-trip) before merge. Name `GPU_OpenGL.cpp:322-341`, `GPU3D_Compute.{h,cpp}` and `GPU3D_OpenGL.{h,cpp}` as edited files, and specify the unified signature as `SetRenderSettings(RendererSettings&)`. Make Phase 3 explicitly depend on Phase 2's frontend decoupling, or extract that decoupling into its own small shared phase.

### 11. MAJOR · UNSUPPORTED · Phase 1's device-acceptance checklist is wrong in every direction

*Phase 1 VulkanDevice bullet; Part 2 descriptor row; Phase 2 shader bullet; Phase 4 clamp.*

- **The gate is bogus.** `maxPerStageDescriptorStorageBuffers ≥ 10` is both above the spec floor (4) and above the actual need: counting distinct std430 blocks per program in `GPU3D_Compute_shaders.h`, the maximum in one program is **7** (Rasterise), 8 distinct bindings overall. Conformant devices reporting 8 or 9 are rejected for nothing.
- **`maxTexelBufferElements` is never checked**, though the plan converts `uimageBuffer` to a storage texel buffer. `YSpanIndicesTextureMemory` is sized `64*2048*ScaleFactor` (`GPU3D_Compute.cpp:389-400`) — 131072 texels **at 1×**, against a spec floor of 65536. `vkCreateBufferView` would violate `VUID-VkBufferViewCreateInfo-range-00930` at the very scale Phase 2 ships.
- **`maxPerStageDescriptorUniformBuffers` and `minUniformBufferOffsetAlignment` are absent**, though `MetaUniform` is a std140 UBO at binding 0.
- **No device features are enabled anywhere in Phase 1**, yet the design needs at least two: `shaderSampledImageArrayDynamicIndexing` (guaranteed only from Vulkan 1.4 or with `descriptorIndexing`, **not core 1.3**) for `Samplers[pushConstantIdx]`, and `independentBlend` for the Phase-4 sprite pass. The immutable-sampler design is explicitly sold as needing "no descriptor indexing"; the omission hides a hard feature dependency the 1.3 baseline does not guarantee. On drivers where it is unsupported, some silently sample element 0 — every polygon gets CLAMP wrap.
- **`maxComputeWorkGroupInvocations` is insufficient on Metal.** `MVKPipeline.mm:2459` enforces a *per-pipeline* threadgroup limit from the actual `MTLComputePipelineState`, which can be lower than the device-reported `maxTotalThreadsPerThreadgroup`. A TileSize-32 rasterize pipeline can fail `vkCreateComputePipelines` on a device reporting 1024 — at the exact moment a user drags the resolution slider.

**Correction.** Drop the storage-buffer gate to 8 or delete it. Add `maxTexelBufferElements`, `maxPerStageDescriptorUniformBuffers`, `minUniformBufferOffsetAlignment` and `maxStorageBufferRange` (see finding 2). Add a required-features list enabled at `vkCreateDevice`: `shaderSampledImageArrayDynamicIndexing`, `independentBlend`. For the Metal limit, clamp on actual pipeline-creation success at each candidate TileSize rather than on the reported limit, and validate on real Metal hardware.

### 12. MAJOR · INCOMPLETE · The parent seam is undefined, and the capture-output textures have no owner until Phase 4

*Phase 2 bullets 1 and 4; Phase 4 bullet 4.*

Every concrete `Renderer3D` holds a *concrete-typed* parent reference, not a `Renderer&`: `GLRenderer& Parent` (`GPU3D_Compute.h:56`, `GPU3D_OpenGL.h:49`), `SoftRenderer& Parent` (`GPU3D_Soft.h:52`). `ComputeRenderer3D` uses it for parent-owned GL objects: `Parent.OutputTex3D = Framebuffer` (`:386`) and — critically — it binds `Parent.CaptureOutput128Tex` and `Parent.CaptureOutput256Tex` at `:1091-1095` **unconditionally** inside the `numYSpans > 0` block, on every frame with geometry, not only for capture-as-texture polygons. `SoftRenderer` has no `OutputTex3D` and no `CaptureOutput*` members at all, and constructs all three children itself at `GPU_Soft.cpp:36-38` — so an injected child must be constructed before the parent it needs a reference to.

Phase 2's only stated work is "a constructor variant accepting an injected `Renderer3D`", which answers none of: what parent reference does `VkRenderer3D` take, how is it built before `SoftRenderer` exists, and where do the two capture-output images live in Phase 2. Because the binding is unconditional, Phase 2 either binds a dummy image every frame or trips an unbound-descriptor validation error on essentially every 3D frame. It also poisons Phase 2's own oracle: golden-hash parity with the GL twin cannot hold for capture frames, and capture-torture titles are only listed in Phase 4's validation — so the divergence surfaces late and gets misattributed to barriers.

**Correction.** Define the parent seam as an explicit Phase-2 work item: either give `VkRenderer3D` no parent reference plus a small output-target interface implemented by both `SoftRenderer` (owns the readback ring) and `VkRenderer` (owns the images), or add `SetParent`/two-phase init to `Renderer3D`. State where the capture-output images live in Phase 2 — a renderer-owned dummy image plus descriptor is the cheapest answer. Extend the "copy verbatim from `:633-984`" instruction to include the consumers at `:1091-1096`.

### 13. MAJOR · WRONG · The Phase-4 2D port is not "mechanical, pass-for-pass"

*Phase 4 GPU2D_Vulkan bullet; premise at Part 1.4.*

`GPU2D_OpenGL.cpp:461-476` binds two color attachments onto two layers of `OBJLayerTex`, then `:1658-1683` mutates per-attachment write masks **between draws inside one pass**, with the two attachments differing: `glColorMaski(0,F,F,F,F)` alongside `glColorMaski(1,F,T,F,T)` at `:1663/:1669`, then `glColorMaski(1,F,F,T,F)` at `:1675`, then `glColorMaski(0,T,T,T,T)`/`glColorMaski(1,T,T,T,F)` at `:1682-1683` — five distinct mask states across three `RenderSprites` calls, plus depth-test and depth-mask toggles.

In Vulkan `colorWriteMask` is a member of `VkPipelineColorBlendAttachmentState`, i.e. **pipeline-baked**, and per-attachment divergence requires the `independentBlend` core feature. Making it dynamic instead requires `vkCmdSetColorWriteMaskEXT`, which exists only under `VK_EXT_extended_dynamic_state3` / `VK_EXT_shader_object` — in no core feature block, so not in Vulkan 1.3 or 1.4 — and **MoltenVK hard-disables it** (`extendedDynamicState3ColorWriteMask = false`).

So the port is not pass-for-pass: GL's per-draw state mutation must be decomposed into a static pipeline permutation matrix — precisely the redesign the plan says is unnecessary, and precisely the growth in the "handful of graphics pipelines" that the no-async-compilation decision rests on. On macOS the escape hatch does not exist at all.

**Correction.** Rewrite the Phase-4 GPU2D bullet as what the work is: enumerate GL draw-state mutations into a static pipeline permutation table, count the permutations up front from `GPU2D_OpenGL.cpp:746-751, :1574-1687, :1761-1790`, and fold that count into the "~35 compute + a handful of graphics pipelines" figure. Add `independentBlend` to Phase 1's feature list. Do not plan on `VK_EXT_extended_dynamic_state3`. Re-size Phase 4 afterwards.

### 14. MAJOR · WRONG · Phase 1 does not configure on Windows or macOS, and four other build paths are broken

*Phase 1 bullets 2, 4, 5, 6; Part 2 shader-toolchain row; Phase 3 source list.*

- **vcpkg.** The `glslang` port at the pinned baseline (`9b965a11`, `vcpkg.json:6-7`) puts `glslangValidator` behind an opt-in `tools` feature with no default-features. `"host glslang"` as written installs **no binary**, so `src/Vulkan_shaders/CMakeLists.txt` cannot find it. Windows and macOS both use vcpkg — Phase 1 either fails configure or silently self-disables, and the failure surfaces only after a full vcpkg dependency build.
- **`MakeEmbedBinary.cmake` is dead work.** `glslangValidator` already emits a C header of `uint32_t` via its own flag. Hand-assembling words in CMake is invented, slow, and is where SPIR-V word order and alignment get decided — neither of which the plan mentions.
- **Loader story is incoherent.** The plan lists both vendored volk (dlopen, "no hard link dep") *and* `vulkan-loader`/`libvulkan-dev` packages across four package lists. On the BSD path (`build-bsd.yml:28,49,56`) following it literally gets glslang but no `vulkan/vulkan_core.h`, so `volk.h` will not compile. Two VMA copies (vendored + vcpkg) is an include-order lottery.
- **`flake.nix`.** `optionals (!isDarwin)` gives Nix macOS no Vulkan (it needs `pkgs.moltenvk`); Nix Linux builds fine but `volkInitialize` returns `VK_ERROR_INITIALIZATION_FAILED` at runtime unless `vulkan-loader` joins the existing `qtWrapperArgs` prefix (`flake.nix:60-64`).
- **`src/frontend/qt_sdl/CMakeLists.txt` is never mentioned**, yet Phase 3 needs a gated block there, a per-platform surface-source matrix, and new Apple framework links — and on macOS with `USE_QT6=ON` (the default and the CI config) the ObjC++ shim will not link until the `NOT USE_QT6` guard around `COCOA_LIB` at `:122-125` is restructured.
- **The GL gate being mirrored has never had its OFF state exercised** and does not build today — unguarded enum entries at `EmuInstance.h:71-79` with unguarded uses at `EmuThread.cpp:869,872` and `VideoSettingsDialog.cpp:49-73`, plus an unconditional `#include "GPU_OpenGL.h"` at `EmuThread.cpp:52`. Copying its shape reproduces the bug — discovered when Phase 1's own "CI builds both flag states" runs.

**Correction.** `{"name": "glslang", "host": true, "features": ["tools"]}`. Delete `MakeEmbedBinary.cmake` in favour of glslangValidator's own header output. Pick one loader story — vendored volk dlopen and *no* loader package anywhere — and add `vulkan-headers` to every path including BSD; pick one VMA source. Fix `flake.nix` for both Darwin and the Linux library path. Add `src/frontend/qt_sdl/CMakeLists.txt` to Phase 3 with the `COCOA_LIB`/`USE_QT6` fix. Fix the existing `ENABLE_OGLRENDERER=OFF` build first so it is actually a template.

### 15. MAJOR · WRONG · Both external feasibility proofs and the upstream-risk basis are factually wrong

*Part 1.3 bullets 1–3; Part 1.4 bullet 1; Part 4 upstream-acceptance row; Part 5.*

- **There is no no-AI-code policy.** `CONTRIBUTING.md` is a style guide; zero AI/LLM references exist across `CONTRIBUTING.md`, `README.md` or `.github/`. The real constraint is a per-maintainer, per-PR quality judgement voiced in one thread (PR #2529).
- **PR #2700 was not killed by anyone.** Opened 17 Jul 2026, closed 18 Jul 2026 **by its own author**, with zero reviews and zero maintainer comments. Upstream has never expressed any opinion on a Vulkan backend.
- **That also guts the tractability argument.** An unreviewed, unmerged, self-withdrawn, self-assessed 29-commit branch is evidence that someone wrote code, not that the port is correct.
- **melonDualDS does not prove the compute port.** It (now `SapphireRhodonite/WatermelonDS`) lists its renderers as "Software, OpenGL, Vulkan **and** compute" — Vulkan and compute as two *distinct* selectable renderers — and its 0.7.0 notes describe presentation-profile and texture-upload work, not tile-binning compute. This also collides with the plan's own baseline row, which writes Android off as out of scope while Part 1.3 leans on an Android fork as proof.
- **Supporting errors.** Issue #807 is titled "macOS" and is not a Vulkan request (the earliest honest one is #984, Feb 2021). The Citra citation is unverifiable — the repository is deleted — and mischaracterizes the PICA200 as fixed-function when Citra decompiled its programmable vertex/geometry stages to host shaders. That bullet is separately load-bearing: it is the basis for the plan's whole phase shape (infrastructure as the dominant cost), and it happens to *invert* the plan's own estimates, which put XL on the rasterizer and only M on infrastructure.

**Correction.** Delete the no-AI-policy claim; replace with "a maintainer has voiced quality concerns about AI-assisted contributions in at least one PR thread (#2529); there is no codified rule." Rewrite #2700 as "opened and self-closed within ~24 hours with no maintainer review — it demonstrates an attempt, not a correct result." Either cite WatermelonDS source showing its Vulkan renderer derives from `GPU3D_Compute`, or downgrade to "whether it derives from the compute renderer is unverified" and drop "shipped exactly this". Rewrite the upstream risk row as "upstream review appetite for a Vulkan backend is entirely unmeasured." Drop #807. Replace or delete the Citra bullet — then re-derive the Phase 2 and Phase 4 estimates with no existence proof.

### 16. MAJOR · RISKY · The readback ring is under-specified in three ways

*Phase 2 "Output contract".*

- **No host-read barrier.** Waiting a timeline value is not sufficient to make writes host-visible; the spec's host-access rules require a memory dependency with `VK_PIPELINE_STAGE_HOST_BIT` / `VK_ACCESS_HOST_READ_BIT`. Without it, sync validation flags it immediately — and where validation is off you get torn or stale scanlines that look like rasterizer bugs.
- **Compute scattering straight into host-visible memory.** ~196608 u32s written directly into `HOST_VISIBLE` means write-combined/BAR traffic on discrete GPUs — the wrong shape against the plan's own deadline.
- **Scalar CPU reads out of device memory, invisible to the plan's own measurement.** `HOST_VISIBLE|HOST_COHERENT` is commonly write-combined and uncached for CPU reads, and the consumers do per-u32 scalar reads: `GPU2D_Soft.cpp:372-378` (`DrawBG_3D`, 256 reads/line × 192 lines) and `GPU_Soft.cpp:281-282, :312-321` (`DoCapture`, doubling it whenever capture source A is 3D — exactly the dual-screen-3D titles Phase 4 calls capture torture). The plan's measurement item times the *wait*, not the subsequent reads, so this cost would never appear in the numbers it collects.

**Correction.** Final pass writes to a *device-local* buffer → `vkCmdCopyBuffer` into the host-visible ring → barrier with `dstStageMask` HOST / `dstAccessMask` HOST_READ before the timeline signal the host waits on. Prefer `HOST_VISIBLE|HOST_CACHED` with explicit invalidate over COHERENT write-combined. Have `GetLine` memcpy the line into a cached scratch buffer once — reuse the existing `ScrolledLine` shape at `GPU3D_Soft.cpp:1816-1823` — so `DrawBG_3D` and `DoCapture` never touch device memory scalar-wise. Add "time spent inside `DrawBG_3D`/`DoCapture`" to the measurement item.

### 17. MAJOR · UNSUPPORTED · The emu-thread timing model is mis-stated in two ways

*Part 1.1 first bullet; Phase 2 "Timing honesty"; Phase 2 Risks; Part 2 WSI row.*

- **Rendering is not all on the emu thread today.** The software rasterizer runs on its own thread by default: `GPU3D_Soft.cpp:48-58` creates it, `:1774-1805` is where rasterization happens, and `Config.cpp:99` defaults `{"3D.Soft.Threaded", true}`. `RenderFrame()` is fire-and-forget — `:1756-1760` just posts a semaphore — and `GetLine` is the sync point. The plan's own lines 23, 25, 27 and 107 contradict its bullet. An implementer taking it at face value concludes `RenderFrame()` returning means the work is done; the asynchrony is exactly the shape a Vulkan backend must reproduce.
- **The ~3 ms deadline is wall-clock at 100% speed only.** There is no intra-frame pacing anywhere (`NDS.cpp:924-1010`); all pacing is post-frame in the frontend (`EmuThread.cpp:311, :323, :361-363, :385-405`, with `curFPS=1000.0` when unlimited). At 300–600% — routine on a machine that can also run a Vulkan compute rasterizer — the real deadline is ~0.5–1.0 ms, and under fast-forward it approaches zero.

So "Acceptable at 1× on real GPUs" is drawn from an inflated figure, and the risk row scopes the problem to *slow* GPUs when it bites hardest on *fast* hosts and during fast-forward — where the IMMEDIATE/MAILBOX mitigation does nothing, because the stall is a readback host-wait, not a present. In practice fast-forward regresses relative to the software renderer: a user-visible functional loss.

**Correction.** Rewrite the Part 1.1 bullet: "Rendering is dispatched from the emu thread; the software rasterizer runs on its own worker thread by default. `RenderFrame()` is fire-and-forget and `GetLine` is the sync point — a Vulkan backend must reproduce that asynchrony." Restate the deadline in emulated-time units (~3 ms at 100%, ~0.5 ms at 600%, ~0 under fast-forward). Re-scope the risk row from "slow GPUs" to "fast hosts and fast-forward". Promote chunked line-range readback from "prepared mitigation" to a Phase-2 requirement, or state that fast-forward is degraded and by how much.

### 18. MAJOR · RISKY · MoltenVK serializes the 256-dispatch loop and over-synchronizes

*Phase 2 Risks; Part 2 descriptor row.*

MoltenVK's `MVKCommandBuffer.mm:1073-1080` returns `MTLDispatchTypeSerial` for all non-occlusion uses and creates the encoder with it at `:1105`, so back-to-back compute dispatches are **strictly serialized** rather than overlapped. The loop is confirmed at `GPU3D_Compute.cpp:1129/:1187/:1191` with `MaxVariants=256` (`GPU3D_Compute.h:166`). Many variants cover only a handful of work tiles, so launch latency dominates — landing directly on the Phase-2 deadline (which finding 17 shows is tighter than stated) and on Phase 5's "expect parity or better".

The second-order effect is worse: `MVKDevice.mm:4655-4664` early-returns from `applyMemoryBarrier` unless the barrier involves HOST — MoltenVK over-synchronizes and ignores most explicit barriers. So a missing or wrong `vkCmdPipelineBarrier2` in the 9-stage chain will **silently work on macOS and break on desktop drivers**. macOS testing is not evidence of barrier correctness — and per finding 4, sync validation is not available there under the planned packaging either.

**Correction.** Add a Phase-2 risk item naming `MTLDispatchTypeSerial`, with a measurement task on real Metal hardware early in the phase (not Phase 5). State in the validation section that macOS runs do not validate barrier correctness and that desktop/lavapipe with sync validation is the authority. Name the fallbacks now (merging low-occupancy variants, or a single dispatch with a variant index from the work descriptor) rather than discovering them in Phase 5.

### 19. MINOR · INCOMPLETE · `GetLine` has a hard non-null contract the plan never states *(PLAUSIBLE — not independently refuted)*

*Phase 2 "Output contract".*

`GPU_Soft.cpp:115` assigns `Output3D` unconditionally from `Rend3D->GetLine(line)` for every line < 192, and consumers dereference with no guard: `GPU2D_Soft.cpp:373-375` does `u32 c = Parent.Output3D[i]` inside a 256-iteration loop; `GPU_Soft.cpp:282` does `srcA = Output3D` and reads through it. `SoftRenderer3D::GetLine` always returns a valid pointer including on the black-line abort branch. There is no nullptr check anywhere on this path.

The proposed ring has at least four no-valid-slot states: first frame after Init/Reset; the frame after `SetRenderSettings` reallocates the ring; an `AbortFrame`/`RestartFrame` frame (finding 9); and `RenderFrameIdentical` on frame 0. Each is a null dereference in the 2D compositor on the emu thread — a hard crash on boot or on a settings change.

**Correction.** State the invariant: "`GetLine` must return a valid 256-u32 pointer in every state, never nullptr." Give the ring a zero-initialized fallback slot that `Reset()` selects. Add "boot to first frame" and "scale/settings change mid-game" to the Phase-2 test list.

### 20. MINOR · RISKY · Hoisting `ScreenPanelGL`'s native-widget setup into the base breaks the native panel

*Part 2 WSI row; Phase 3 bullet 2.*

`ScreenPanelNative` also derives from `ScreenPanel` (`Screen.h:157-167`) and paints with `QPainter painter(this)` (`Screen.cpp:798-803`). The setup proposed for hoisting (`Screen.cpp:868-878`, `:1350-1353`) sets `WA_PaintOnScreen` and a null `paintEngine`, which is precisely what makes `QPainter`-on-widget impossible. Hoisting as written silently breaks the software/native display path for every user with `Screen.UseGL=false` — the exact panel Phase 2's own goal says must keep working.

**Correction.** Hoist only `getWindowInfo()`. Put the `WA_NativeWindow`/`WA_PaintOnScreen`/null-paintEngine setup into an intermediate class that `ScreenPanelGL` and `ScreenPanelVK` both derive from. Add "software renderer with `Screen.UseGL=false` still displays" to Phase-3 validation.

### 21. MINOR · INCOMPLETE · Phase 3 has no panel-swap protocol

*Phase 3 third bullet.*

The transitions that actually trigger a panel swap are in `Window.cpp:2263-2293` (`onUpdateVideoSettings`), gated by `VideoSettingsDialog`'s composite predicates at `:35, :142, :162, :174`, with the emu-thread side at `EmuThread.cpp:558, :680-683`. A three-way `createScreenPanel` plus a `Screen.UseVK` key does not make those paths recreate the panel when the *presenter kind* changes rather than the GL flag, nor define pause/teardown ordering, nor the fan-out across up to four child windows. Phase 2's own "hot-swap soak" validation exercises exactly the path that silently does nothing — leaving a GL panel bound to a Vulkan renderer, the failure mode Phase 2 claims to eliminate.

**Correction.** Define the panel-swap protocol as a named state machine: which config changes require a recreate, who pauses the emu thread, in what order surfaces and swapchains are destroyed and recreated, and how it fans out to child windows. Name `Window.cpp:2263-2293` and `VideoSettingsDialog.cpp:35/:142/:162/:174` as edited locations. Generalize `UsesGL` into a presenter-kind predicate in the same change as Phase 2's `usesOpenGL` decoupling.

### 22. MINOR · RISKY · Phase 4's hi-res benefit reaches a much narrower audience than stated

*Phase 4 goal; Phase 3 Wayland bullet; Phase 5 bullet 2.*

Composing the plan's own constraints: hi-res scaling requires the Phase-4 full compositor → requires an active Vulkan *display* → requires `ScreenPanelVK` → **gated off on Wayland at first ship**. So after the entire Phase 2+3+4 investment, every Wayland user (the GNOME/KDE default), every user who keeps the GL display, and every `ScreenPanelNative` user gets 1×-only Vulkan with a per-frame CPU readback. Meanwhile `VideoSettingsDialog.cpp:43-50/:98-100` happily offers 1×–16× and the setting is silently ignored. The plan never states this restriction. Combined with finding 5 (silent fallback, no diagnostics), users cannot tell which of the two Vulkan modes they are in, why scaling does nothing, or why performance differs 5× between two machines with the same setting.

**Correction.** State the audience explicitly in the Phase-4 goal. Grey out or clamp the ScaleFactor control when the active mode cannot honour it (`setEnabled` precedent already at `VideoSettingsDialog.cpp:43-50`). Surface the active mode somewhere user-visible. Reconsider whether promoting Wayland enablement is worth more than Phase 4's compositor.

### 23. MINOR · RISKY · The plan points implementers at a CC-BY-NC-ND file as the model for its largest new subsystem

*Part 5 external references; Part 2 WSI rationale.*

Current DuckStation sources carry an SPDX header of **CC-BY-NC-ND-4.0** (verified on master, `src/util/vulkan_device.cpp` lines 1–2). melonDS is GPLv3 (`LICENSE:1`). The already-vendored `src/frontend/duckstation/` files predate the relicence and carry no SPDX headers, which is why the in-tree precedent looks clean. The plan points an implementer at a No-Derivatives, Non-Commercial file as the model for Phase 1's device layer and again for Phase 3's WSI, while the surrounding narrative normalises copying from DuckStation.

MINOR only because the fix is a one-line caveat; the failure mode if ignored is not minor.

**Correction.** Add a licensing note: DuckStation's current Vulkan sources are CC-BY-NC-ND and are read-only prior art, not a copy source; the vendored files predate that relicence. Prefer Dolphin (GPLv2+) and parallel-rdp (MIT) as structural references. State that all Vulkan code is written from scratch.

### 24. MINOR · WRONG · Three mischaracterizations of the compute renderer that would misdirect work

*Phase 2 "Output contract"; Phase 2 texcache bullet; Part 1.2.*

- **(a)** `GPU3D_Compute_shaders.h:1665-1668` is a commented-out *alpha test* setting bit 30, **not a packing**. The real packing is at `:1655`, consumed at `:1673-1675`; `GPU2D_Soft.cpp:380` applies `0x40000000` consumer-side, and `SoftRenderer3D`'s packing (`GPU3D_Soft.cpp:562`) has no bit 30. An implementer following the "cf." bakes a consumer-side flag into `GetLine` output — harmless to `DrawBG_3D`/`DoCapture` (both only test `(c>>24) != 0`) but bit-for-bit different from the software renderer for every opaque pixel, defeating the phase's own pixel-diff check.
- **(b)** Nothing in the texcache owns wrap state. The 3×3 matrix is nine GL sampler objects created in `ComputeRenderer3D::Init` (`:216-224`), selected at `:789`, bound at `:1163`, stored on the renderer (`GPU3D_Compute.h:208`). The only per-draw wrap mutation in the tree is in the classic GL renderer (`GPU3D_OpenGL.cpp:806-807`), which is out of scope.
- **(c)** There are **shared-memory atomics** too: `shared uint mergedMaskShared` at `GPU3D_Compute_shaders.h:925`, with `atomicOr` plus `barrier()` at `:955-959`. (The `ballotARB` path at `:944/:953` is indeed `#if 0`.) Shared-memory atomics under a specialization-constant workgroup size are a distinct portability class from SSBO atomics — and this bullet is the stated basis for "exactly the subset that is solid on MoltenVK/Metal and mobile".

**Correction.** (a) Point the readback-format note at `:1655` and `:1673-1675`, and require bit-for-bit match with `GPU3D_Soft.cpp:562` with no bit-30 flag. (b) Move the sampler-matrix work item to the `VkRenderer3D` bullet and drop "replaces GL per-draw wrap mutation". (c) Amend to "SSBO atomics plus shared-memory atomics with `barrier()`; no image atomics, no subgroup ops, no 64-bit ints."

---

## Status of the Part 2 headline decisions

| Decision | Status | Note |
|---|---|---|
| Vulkan 1.3 baseline, no pre-1.3 fallback | **STANDS** | Correct and well argued; the legacy-class analysis holds. The dent is the promise attached to it — "the selection UI must say so explicitly" is unimplementable on the plumbing the plan calls free (F5). |
| MoltenVK half of the baseline row | **BROKEN** | `portability_enumeration` is a loader extension MoltenVK does not expose; Phase 5 ships only `libMoltenVK.dylib`; deployment-target conflict unresolved. Move packaging to Phase 1 or drop macOS from Phase 2. (F4) |
| `ComputeRenderer3D` as the only 3D port target | **STANDS** | The strongest decision in the document; nothing found here touches it. Two wording caveats (F1, F24). |
| Build-time GLSL→SPIR-V, no runtime glslang | **STANDS** | Right call. Two mechanical fixes: vcpkg `tools` feature; drop `MakeEmbedBinary.cmake`. (F14) |
| Spec constants "eliminate runtime compilation entirely" | **BROKEN** | Pipeline creation *is* runtime compilation. Keep the time-slicing machinery; move `VkPipelineCache` to Phase 1. (F3) |
| Vendored volk + vk-bootstrap + VMA | **STANDS** | Sensible and consistent with existing precedent. Clean up the loader-package incoherence and the double VMA. (F14) |
| "Allocation profile is tiny and static" | **BROKEN** | ~2.6 GiB at 16×, one buffer 6× over the `maxStorageBufferRange` floor, then multiplied again by frames-in-flight. This is the dominant engineering constraint, not a parenthetical. (F1, F2) |
| No `QVulkanWindow`, `ScreenPanelVK` + `WindowInfo` surfaces | **STANDS** | Correct; the `WindowInfo`-already-has-the-handles observation is real and valuable. Hoist only `getWindowInfo()`. (F20) |
| "All rendering and presentation on the emu thread; borrow hooks become no-ops" | **BROKEN** | Deleting the borrow park while the emu thread can block in acquire/present is an unrecoverable UI deadlock, and leaves surface lifetime unsynchronized. The premise is separately wrong — the software rasterizer has its own thread. (F7, F17) |
| Descriptor model (per-pass layouts, push constants, 9 immutable samplers) | **DENTED** | Shape is right. Needs `shaderSampledImageArrayDynamicIndexing` in the feature list, a slot for the absent `MetaUniform` UBO, and the sampler work item moved off the texcache. (F11, F1, F24) |
| No cross-API RHI; three permanent seams | **STANDS** | Correct, and the three named seams are exactly the right three. |
| Validation layers + sync validation from day one | **DENTED** | Right priority, but unavailable on macOS under the planned packaging, and MoltenVK over-synchronizes so macOS runs are not barrier evidence either way. (F4, F18) |
| `VkPipelineCache` as Phase-5 "politeness" | **BROKEN** | It is a required mitigation, worst on MoltenVK. Move to Phase 1. Also 16 concurrent writers under multi-instance. (F3, F6) |
| Part 4 "Upstream acceptance" risk row | **BROKEN** | No policy, no rejection, no equivalent precedent. Rewrite as "upstream review appetite is entirely unmeasured." (F15) |
| Two frames in flight on one timeline semaphore | **BROKEN** | Every buffer is single-instanced and unfenced. Pick one frame in flight, or price the versioning. The timeline-clock idea itself is fine. (F1) |

## Phase scoping

**Phase 1 (M → should be L, and should absorb work from Phase 5).** As written it does not achieve its own goal. Beyond the build fixes, four things belong here: the device/instance **ownership model** (it determines the `VulkanDevice` API everything else builds on); **loader + MoltenVK + ICD packaging**, moved from Phase 5; **`VkPipelineCache` persistence**, moved from Phase 5; and a corrected **feature and limit checklist**. Drop `MakeEmbedBinary.cmake`. `--vk-selftest` is a good idea with no CI job that can fail on it.

**NEW Phase 1.5 — Validation harness (missing; realistically M–L).** Phase 2's correctness argument, Phase 4's VRAM diff and Phase 5's cache fuzzing all depend on tooling priced at zero: deterministic polygon-RAM capture, a serializer, a replay driver decoupled from NDS timing, a frame-hash mode in both renderers with format normalization, distributable test content, and CI that actually runs a binary. Either build and size it, or downgrade every "golden-hash" claim to "manual comparison" and re-rate the barrier-bug risk upward.

**Phase 2 (XL — correctly sized but wrongly *composed*).** Both framing claims are false: it is not inert with the flag off, and it is not parallel with Phase 3. Missing work items that must be added before the sizing means anything: the frames-in-flight and buffer-versioning design (do this first — it changes everything downstream); the parent-seam definition and where capture-output images live; the capture-as-texture path; correct `RestartFrame`/`AbortFrame` semantics including the no-wait black-line abort window; `Init()` propagation plus a fallback-reporting path; a fully specified readback design; and a `GetLine`-never-null invariant. The macOS half of the goal must move to Phase 1 or come out of the goal. **The Phase-2 idea itself remains the best structural decision in the plan and should survive all of this.**

**Phase 3 (M → L).** Three unbudgeted design problems live here: the broken threading model, the absent panel-swap protocol, and multi-instance (which changes the Phase-1 API — discovering it here is a Phase-1 rewrite). Add the frontend `CMakeLists` work, never mentioned. Fix the native-widget hoist. Given Wayland is where most Linux users are and Phase 4's benefits are gated behind a Vulkan display, consider whether Wayland belongs here rather than Phase 5.

**Phase 4 (XL — correctly sized, two wrong premises).** The GPU2D port is not "mechanical, pass-for-pass" (F13), and the one-enum auto-degrade is not delivered by the plumbing it cites (F5). Recompute the ScaleFactor clamp from the corrected memory table × frames-in-flight and from the omitted limits, and state the audience restriction plainly.

**Phase 5 (M — roughly right once emptied and refilled).** Loses MoltenVK packaging and `VkPipelineCache` persistence to Phase 1. Keeps Wayland, the perf pass, docs, the CI default flip. Two additions: a device-loss policy and diagnostics (absent everywhere, and without them a silent fallback plus no mode indicator makes bug reports unactionable); and the MoltenVK dispatch-serialization measurement, since "expect parity or better" is contradicted by `MTLDispatchTypeSerial` on a 256-dispatch loop.

---

## Limitations of this review

- Findings are static analysis of the plan against the tree at `d3cd616`, the Vulkan spec, and public sources. Nothing was compiled or run; no Vulkan code exists to test.
- Finding 19 reached this report without an independent skeptic pass and is marked PLAUSIBLE. Findings 1 and 13 were in the same position and were verified by hand before publication.
- External claims (MoltenVK internals, vcpkg port state, upstream PR history) were checked against public sources at review time and can drift.
- 34 raised findings were discarded during refutation. Absence from this document is not proof a claim is correct — only that no reviewer in these ten lenses could break it.
