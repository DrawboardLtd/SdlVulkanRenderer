# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Fork relationship

This repo is a **private fork** of an upstream repo of the same name (`SdlVulkan.Renderer`). It is for internal drawboard use and must **not** publish to nuget.org — only the upstream repo does.

The fork now mirrors upstream's **full source** — the Android host (`Android/`, the `net10.0-android` TFM), the native WebView subsystem (`SdlVulkan.Renderer.WebView` / `.WebView.Native` / `tools/WebViewSmoke`), and the test project — so a sync is a wholesale `git checkout upstream/main -- .` plus re-applying these fork-only divergences, which must be preserved each round:
- **skip the nuget-publish CI job** — only the upstream repo publishes to nuget.org;
- don't overwrite `LICENSE`, `README.md`, `CLAUDE.md`, or `RepositoryUrl` in `SdlVulkan.Renderer.csproj`;
- keep the submodule-based DIR.Lib `ProjectReference` (see below) rather than upstream's `UseLocalDirLib` sibling/nuget conditional;
- keep `submodules: recursive` on every CI checkout — the submodule DIR.Lib `ProjectReference` needs the nested checkout to exist;
- then bump the submodule pin + `VersionPrefix` / `VERSION_PREFIX`.

## Build commands

```bash
dotnet build -c Debug                   # Debug build
dotnet build -c Release                 # Release build
dotnet pack -c Release                  # Build + produce .nupkg
```

A test project (`src/SdlVulkan.Renderer.Tests`, offscreen-Vulkan render regressions) is mirrored from upstream and runs in CI on Mesa lavapipe. No linter is configured.

## Versioning

Package version is `Major.Minor.RunNumber` where `RunNumber` is the CI build number. Two places must stay in sync when bumping:
- `VersionPrefix` in `src/SdlVulkan.Renderer/SdlVulkan.Renderer.csproj` — used for local builds (`Major.Minor.0`)
- `VERSION_PREFIX` in `.github/workflows/dotnet.yml` — used for CI builds (`Major.Minor.${{ github.run_number }}`)

Central package versioning via `src/SdlVulkan.Renderer/Directory.Packages.props` — update there, not in `.csproj`.

## DIR.Lib dependency (submodule)

DIR.Lib lives as a **git submodule** at `./DeviceIndependentRenderingLibrary/`, pointing at the drawboard DIR.Lib fork. The csproj references it as a `ProjectReference` to `../../DeviceIndependentRenderingLibrary/src/DIR.Lib/DIR.Lib.csproj` — this fork does **not** use the upstream `UseLocalDirLib` conditional or consume DIR.Lib from nuget.org, because the drawboard DIR.Lib fork is not published there.

When DIR.Lib changes land:
1. Sync upstream changes into the drawboard DIR.Lib fork repo, commit, push.
2. In this repo: `cd DeviceIndependentRenderingLibrary && git pull && cd .. && git add DeviceIndependentRenderingLibrary && git commit -m "Update DIR.Lib submodule"`.

After pulling this repo fresh, run `git submodule update --init --recursive` before building.

## Architecture

**Rendering pipeline flow:**
`SdlVulkanWindow` (SDL3 window + Vulkan instance/surface) → `VulkanContext` (device, swapchain, command buffers, per-frame sync with `MaxFramesInFlight = 2`, `CurrentFrame` exposed for side-cars) → `VkRenderer` (2D draw API: rectangles, ellipses, lines, text, textures) → `VkPipelineSet` (pipelines compiled from GLSL 450 at runtime — flat, textured, ellipse, page, stroke, SDF, blend variants)

**Key design patterns:**
- **Push-constant-only uniforms** — no UBOs; all per-draw data (projection matrix, color, extra params) goes through an 84-byte push constant block.
- **Single descriptor set layout** — one combined-image-sampler layout shared by all pipelines; font atlas gets a fixed set, each `VkTexture` allocates its own (pool max 512).
- **Per-frame vertex ring buffer** — two host-visible/coherent buffers (one per in-flight frame), written linearly and reset each `BeginFrame`.
- **Deferred texture upload** — `VkTexture.CreateDeferred` + `RecordUpload` records GPU uploads into the frame command buffer before `BeginRenderPass`, avoiding `vkQueueWaitIdle` stalls. `VkTexture.Dispose` resets `IsUploaded=false` to prevent use-after-free.
- **Font atlas lifecycle** — `VkFontAtlas` manages a growable glyph atlas (up to 4096x4096) with dirty-region staging upload; eviction is deferred one frame to prevent stale UV sampling; `skipUnflushed` guards draw loops from sampling unuploaded glyphs.
- **SDF side-car for text** — `VkSdfFontAtlas` uses R8_Unorm single-channel textures for resolution-independent text. Emoji go through the regular RGBA atlas.
- **Idle-suppressing event loop** — `SdlEventLoop` uses `WaitEventTimeout` when idle, throttles mouse-motion redraws to ~30 fps. Touch: pinch/pinch-end gesture events.

**Side-car (custom) pipeline pattern:**
Consumer projects can create their own Vulkan pipelines that render within the same render pass. To create a side-car pipeline:
1. Create your own `VkDescriptorSetLayout` + `VkPipelineLayout` (with your UBO/push constants).
2. Create `VkPipeline` using `ctx.RenderPass` and `ctx.MsaaSamples` (must match).
3. Compile GLSL 450 → SPIR-V at runtime using `Vortice.ShaderCompiler.Compiler`.
4. Record draw commands via `renderer.CurrentCommandBuffer` between `BeginFrame`/`EndFrame`.
5. Use `ctx.WriteVertices()` for per-frame geometry or `ctx.CreatePersistentVertexBuffer()` for static geometry. Instancing supported — `vkCmdDraw(vertexCount, instanceCount, ...)`.

The 84-byte push-constant block is only a constraint if you use `ctx.PipelineLayout`. Side-cars with their own layout can define any push-constant shape.

**Key files:**
- `VkRenderer.cs` — high-level draw API, extends `Renderer<VulkanContext>` from DIR.Lib; GPU-optimized DrawLine/DrawEllipse.
- `VulkanContext.cs` — Vulkan device/swapchain/sync lifecycle.
- `VkFontAtlas.cs` — glyph rasterization cache + GPU texture management (drives DIR.Lib's `ManagedFontRasterizer`).
- `VkSdfFontAtlas.cs` — SDF glyph atlas for resolution-independent text.
- `VkPipelineSet.cs` — GLSL→SPIR-V compilation + pipeline creation.
- `VkTexture.cs` — per-image Vulkan texture with blocking and deferred upload modes.
- `SdlEventLoop.cs` — event-driven render loop with resize handling, touch gestures.
- `VkMenuWidget.cs` — self-contained menu UI widget implementing `IWidget`.
- `SdlInputMapping.cs` — SDL3 scancode/keymod → DIR.Lib `InputKey`/`InputModifier` mapping.
