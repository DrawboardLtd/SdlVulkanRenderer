# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Fork relationship

This repo is a **private fork** of an upstream repo of the same name (`SdlVulkan.Renderer`). It is for internal drawboard use and must **not** publish to nuget.org — only the upstream repo does.

**`git fetch upstream` before you measure anything, and again before you push.** Ancestry is meaningless between these two — the histories are disjoint, so "N commits behind" and `git cherry` both lie (see the viewer's CLAUDE.md); the only honest measure is a content diff of the two trees, `git diff origin/main upstream/main --stat` plus a `git ls-tree` file-set comparison. That has to run against a *just-fetched* `upstream/main`: a fetch from the top of the session is already stale, and porting a change that upstream landed while you worked produces a PR that conflicts with `main` while adding nothing. Re-fetch and re-diff immediately before opening or pushing a PR, not only when you start.

The fork now mirrors upstream's **full source** — the Android host (`Android/`, the `net10.0-android` TFM), the native WebView subsystem (`SdlVulkan.Renderer.WebView` / `.WebView.Native` / `tools/WebViewSmoke`), and the test project — so a sync is a wholesale `git checkout upstream/main -- .` plus re-applying these fork-only divergences, which must be preserved each round:
- **skip the nuget-publish CI job** — only the upstream repo publishes to nuget.org;
- don't overwrite `LICENSE`, `README.md`, `CLAUDE.md`, or `RepositoryUrl` in `SdlVulkan.Renderer.csproj`. The README's **Native WebView** section is a deliberate rewrite, not stale copy: upstream's version sells the subsystem as two nuget.org packages, and this fork publishes none, so ours documents the submodule `ProjectReference` and warns that `dotnet add package SdlVulkan.Renderer.WebView` fetches upstream's different codebase. Take upstream's *technical* README improvements; re-apply that framing over them;
- keep the submodule-based DIR.Lib `ProjectReference` (see below) rather than upstream's `UseLocalDirLib` sibling/nuget conditional;
- keep `submodules: recursive` on every CI checkout — the submodule DIR.Lib `ProjectReference` needs the nested checkout to exist;
- keep the **android host TFM opt-in** — upstream targets `net10.0;net10.0-android` unconditionally, but desktop consumers reference this project by source (the viewer's submodule `ProjectReference`) and `dotnet restore` evaluates every TFM, so an unconditional android TFM would force the android workload on every consumer/CI. The csproj defaults to `net10.0` and adds the android TFM only when `BuildAndroidHost=true`, which the fork CI's build job sets (it installs the workload); the test/webview jobs stay `net10.0`;
- keep `src/Directory.Build.props`, and keep every csproj FREE of `VersionPrefix` — upstream versions per-csproj, and a csproj `VersionPrefix` overrides the props file, so a sync round silently restores the drift it exists to prevent (see Versioning);
- don't overwrite `CHANGELOG.md`. The two version lines diverged at 7.6 and the file says so in its own header, so taking upstream's replaces this repo's release history with one numbered differently — and because the entries LOOK right, nothing about the result reads as wrong until someone matches a `Major.Minor` here against a package that never carried it;
- keep `src/SdlVulkan.Renderer/Directory.Packages.props`. Its `DIR.Lib` line is inert here (the csproj reaches DIR.Lib through the submodule `ProjectReference`) and its comments say why, which is exactly the knowledge a wholesale overwrite discards;
- then bump the submodule pin + `VersionMajorMinor` in `src/Directory.Build.props`.

Those last two were found the hard way, by diffing before overwriting: a sync round that trusts this
list alone would have taken upstream's changelog and dropped this fork's version line from 8.14 to
7.30. Diff first, every round — the list is a reminder, not an inventory.

A fix to anything **outside** that list belongs upstream first, then comes back down on the next round — putting it here first either loses it to the next `git checkout upstream/main -- .` or grows this list, and the list is what a sync round has to re-apply by hand every time.

## Build commands

```bash
dotnet build -c Debug                   # Debug build
dotnet build -c Release                 # Release build
dotnet pack -c Release                  # Build + produce .nupkg
```

A test project (`src/SdlVulkan.Renderer.Tests`, offscreen-Vulkan render regressions) is mirrored from upstream and runs in CI on Mesa lavapipe. No linter is configured.

## Versioning

Package version is `Major.Minor.RunNumber` where `RunNumber` is the CI build number. **One place to bump:** `VersionMajorMinor` in `src/Directory.Build.props`. Local builds get `Major.Minor.0`; the workflow reads that same property back (`dotnet msbuild src/Directory.Build.props -getProperty:VersionMajorMinor`) rather than restating the number, so CI cannot stamp a version the packages disagree with.

It covers **every** package here — renderer, Inspector, WebView, WebView.Native — because CI stamps a single `-p:Version` across all of them. No csproj declares its own `VersionPrefix`: a per-project one silently overrides the props file, which is how Inspector/WebView/WebView.Native sat at 6.0.0 while the renderer shipped 7.5.0.

Add the matching entry to [CHANGELOG.md](CHANGELOG.md) at the repo root, in the same commit — newest first, one `## Major.Minor` section each. (The notes used to live in a comment block in `.github/workflows/dotnet.yml`, justified by the double hyphen several of them contain, which XML forbids inside a comment — but that only ever ruled out putting them in a csproj, nothing ever read them there, and they had reached 431 of that file's 590 lines.)

Central package versioning via `src/SdlVulkan.Renderer/Directory.Packages.props` — update there, not in `.csproj`.

## DIR.Lib dependency (submodule)

DIR.Lib lives as a **git submodule** at `./DeviceIndependentRenderingLibrary/`, pointing at the drawboard DIR.Lib fork. The csproj references it as a `ProjectReference` to `../../DeviceIndependentRenderingLibrary/src/DIR.Lib/DIR.Lib.csproj` — this fork does **not** use the upstream `UseLocalDirLib` conditional or consume DIR.Lib from nuget.org, because the drawboard DIR.Lib fork is not published there.

When DIR.Lib changes land:
1. Sync upstream changes into the drawboard DIR.Lib fork repo, commit, push.
2. In this repo: `cd DeviceIndependentRenderingLibrary && git pull && cd .. && git add DeviceIndependentRenderingLibrary && git commit -m "Update DIR.Lib submodule"`.

After pulling this repo fresh, run `git submodule update --init --recursive` before building.

## Architecture

**Rendering pipeline flow:**
`SdlVulkanApp` (process-wide SDL lifecycle + shared `VkInstance` and `VulkanDevice` for a multi-window app) → `SdlVulkanWindow` (per-window SDL3 window + Vulkan surface) → `VulkanContext` (per-window swapchain, command buffers, per-frame sync with `MaxFramesInFlight = 2`, vertex ring, `CurrentFrame` exposed for side-cars; references a `VulkanDevice`) → `VkRenderer` (2D draw API: rectangles, ellipses, lines, text, textures) → `VkPipelineSet` (pipelines built from pre-baked SPIR-V — flat, textured, ellipse, page, stroke, SDF, round-rect, blend variants)

**Device vs. context split (6.0+):** device-level state (logical device, queue, command pool, render pass, descriptor pool/layout, pipeline layout, MSAA) lives in `VulkanDevice`; per-window state (swapchain, framebuffers, sync, vertex ring, command buffers) lives in `VulkanContext`. `VulkanContext` forwards the device-level members (`ctx.RenderPass`, `ctx.PipelineLayout`, …) so existing consumers keep working. Three construction paths:
- `SdlVulkanApp.CreateWindow` + `VulkanContext.CreateForSharedDevice` — multi-window; one `VulkanDevice` shared across windows, so GPU resources stay valid in all of them and a document tab can move between windows without re-uploading geometry.
- `VulkanContext.Create(instance, surface, …)` — single-window; the context creates and owns its own device.
- `VulkanContext.CreateOffscreen(instance, …)` — headless render-to-`VkImage`; no surface, swapchain or SDL window. Used by tests, thumbnail/raster workers, and CI without a display.

A context tears down the device only when it created it; shared-device windows leave teardown to the device owner (the `SdlVulkanApp`).

**Key design patterns:**
- **Push-constant-only uniforms** — no UBOs; all per-draw data (projection matrix, color, extra params) goes through an 84-byte push constant block.
- **Single descriptor set layout** — one combined-image-sampler layout shared by all pipelines; font atlas gets a fixed set, each `VkTexture` allocates its own. `DescriptorSetsPerPool = 512` is the capacity of ONE pool, **not** a ceiling: pools can't be resized, so `VulkanDevice.AllocateDescriptorSet` chains a new one when the current fills, and `FreeDescriptorSet` recycles into a free-list rather than returning the set to the driver (the handle stays valid for in-flight command buffers, and nothing has to track which pool issued it). A fixed pool used to cap how many textures a document could have — and since the glyph atlas draws from the same pool, *text* was what got refused.
- **One shared sampler** — every `VkTexture` binds `VulkanDevice.LinearClampSampler`. Samplers carry no per-image state, so the old per-texture sampler bought nothing and ran image-dense pages into `maxSamplerAllocationCount` (commonly 4096).
- **Per-frame vertex ring buffer** — two host-visible/coherent buffers (one per in-flight frame), written linearly and reset each `BeginFrame`.
- **Deferred texture upload** — `VkTexture.CreateDeferred` + `RecordUpload` records GPU uploads into the frame command buffer before `BeginRenderPass`, avoiding `vkQueueWaitIdle` stalls. `VkTexture.Dispose` resets `IsUploaded=false` to prevent use-after-free.
- **Font atlas lifecycle** — `VkFontAtlas` manages a growable glyph atlas (up to 4096x4096) with dirty-region staging upload; eviction is deferred one frame to prevent stale UV sampling; `skipUnflushed` guards draw loops from sampling unuploaded glyphs.
- **MTSDF side-car for text** — `VkSdfFontAtlas` is a list of fixed-size R8G8B8A8Unorm page textures for resolution-independent text: RGB carry pseudo-distance (the shader reconstructs via median, which is what preserves corners) and A the true distance. A full page appends a new page rather than reallocating — no `vkDeviceWaitIdle` + realloc + re-upload stall — with per-page LRU eviction. Emoji go through the regular RGBA atlas.
- **Idle-suppressing event loop** — `SdlEventLoop` uses `WaitEventTimeout` when idle, throttles mouse-motion redraws to ~30 fps, and drives every window of a multi-window app. Touch: pinch/pinch-end gesture events.
- **Live-device thumbnail capture** — `VkRenderer.BeginThumbnailCapture` / `EndThumbnailCapture` / `TryGetThumbnailCapture` re-issue already-tessellated geometry into an offscreen target at thumbnail scale, with non-blocking readback (`VulkanContext.ThumbnailCapture`).

**Side-car (custom) pipeline pattern:**
Consumer projects can create their own Vulkan pipelines that render within the same render pass. To create a side-car pipeline:
1. Create your own `VkDescriptorSetLayout` + `VkPipelineLayout` (with your UBO/push constants).
2. Create `VkPipeline` using `ctx.RenderPass` and `ctx.MsaaSamples` (must match).
3. Bring your own SPIR-V. The shipped package bakes its shaders at build time and does **not** reference `Vortice.ShaderCompiler`, so a side-car that wants runtime GLSL 450 → SPIR-V has to take that package itself.
4. Record draw commands via `renderer.CurrentCommandBuffer` between `BeginFrame`/`EndFrame`.
5. Use `ctx.WriteVertices()` for per-frame geometry or `ctx.CreatePersistentVertexBuffer()` for static geometry. Instancing supported — `vkCmdDraw(vertexCount, instanceCount, ...)`.

The 84-byte push-constant block is only a constraint if you use `ctx.PipelineLayout`. Side-cars with their own layout can define any push-constant shape.

**Key files:**
- `SdlVulkanApp.cs` — process-wide SDL lifecycle + the shared `VkInstance`/`VulkanDevice` a multi-window app hangs off.
- `SdlVulkanWindow.cs` — per-window SDL3 window + Vulkan surface. Implements AppShell's `IActivatableWindow`, so a single-instance hand-off can raise it.
- `VkRenderer.cs` — high-level draw API, extends `Renderer<VulkanContext>` from DIR.Lib; GPU-optimized DrawLine/DrawEllipse.
- `VulkanContext.cs` — per-window swapchain/sync/vertex-ring lifecycle (references a `VulkanDevice`). Partials carry the paths that need their own: `VulkanContext.Offscreen.cs` (headless render-to-image), `.SwapchainReadback.cs`, `.ThumbnailCapture.cs`, `.Damage.cs` (repaint only what changed), `.CachedLayer.cs` (so a chrome-only redraw does not re-shade the picture), `.DeferredDestroy.cs` (release a bound object without a full drain).
- `VkFontAtlas.cs` — glyph rasterization cache + GPU texture management (drives DIR.Lib's `ManagedFontRasterizer`).
- `VkSdfFontAtlas.cs` — MTSDF glyph atlas for resolution-independent text.
- `VkPipelineSet.cs` — pipeline creation from the pre-baked SPIR-V embedded in the assembly. Shaders are authored as GLSL 450 in `Shaders/*.vert|*.frag` and baked to `Shaders/spirv/*.spv` (committed) by `tools/BakeShaders`; re-run it from the repo root after editing one — `dotnet run --project tools/BakeShaders -c Release -- src/SdlVulkan.Renderer/Shaders`. Commit `Shaders/spirv/sources.sha256` with the `.spv`: it records the source hashes the bake was made from, and build warning SVR0001 compares the sources against it. That check is content-based, so it catches a forgotten re-bake in CI as well as locally — `Shaders/.gitattributes` pins the sources' line endings to keep the hashes reproducible from any checkout.
- `VulkanDevice.cs` — instance/device/queue/render-pass creation and the descriptor-pool chain; shared across windows, so a torn-out window reuses one device.
- `VkTexture.cs` — per-image Vulkan texture with blocking and deferred upload modes.
- `SdlEventLoop.cs` — event-driven, multi-window render loop with resize handling, touch gestures.
- `SdlInputMapping.cs` — SDL3 scancode/keymod → DIR.Lib `InputKey`/`InputModifier` mapping.
- `DebugInspector.cs` / `DebugInspectorOptions.cs` — the in-process debug inspector, DEBUG-only; its client lives in `SdlVulkan.Renderer.Inspector`.
- `VulkanValidation.cs` / `RenderDiag.cs` — opt-in validation-layer reporting and the wedge/device-churn breadcrumb.
