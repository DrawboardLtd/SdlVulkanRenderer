# Changelog

Release notes for SdlVulkan.Renderer, one entry per `Major.Minor`, newest first.

The version NUMBER is not here: it lives in `src/Directory.Build.props` (`VersionMajorMinor`), and the
build job reads that property back rather than restating it, so a package can never declare a version
this file disagrees with. Bump it there and add the entry here, in the same commit.

**7.6 and later are THIS repo's releases, and do not line up with the upstream repo's numbers.** The
two version lines diverged at 7.6 — upstream's 7.6 is a lockstep DIR.Lib rebuild, this one's is the
descriptor-pool chaining below — and they have run independently since, so the same `Major.Minor` names
a different release in each. 7.5 and earlier are the shared history from before the split. Do not read
an entry here against upstream's entry for the same number, and do not conclude from a version gap that
this repo is behind: it tracks DIR.Lib's number, upstream numbers its own way.

## 8.15

**A depth-tested offscreen scene target, for content whose visibility is geometry rather than draw
order.** Everything this renderer drew was painter's-order 2D — the back-to-front sequence of draws IS
the occlusion — so no pass had a depth attachment. A mesh cannot be drawn that way: its own triangles
occlude each other in an order that depends on the camera, and no CPU-side sort is correct for every
view.

`VulkanContext.SceneTarget` is a sampleable colour+depth target built to `CachedLayer`'s rules — one
target per frame in flight, fixed capacity, colour finalising to `ShaderReadOnlyOptimal` with its own
descriptor set — so the result composites through the existing `DrawTexture`. It is a separate pass
rather than depth on the main one because compatibility is per-attachment and the pre-baked pipelines
are shared: a depth attachment on the swapchain pass would force one onto the cached-layer, damage and
thumbnail passes too. `VkMeshPipeline` draws into it, with its own 96-byte push-constant layout, and
`VkRenderer` gains `EnsureSceneTargets` / `BeginScene` / `DrawMesh` / `EndScene`.

`FillSubpassDependencies` is widened with the depth stages rather than the scene pass carrying its own
list, since dependencies are part of render-pass compatibility and the count has to match everywhere.

Synced from upstream (SharpAstro/SdlVulkan.Renderer#89), where it landed first as generic renderer
capability. The drawboard consumer is the PDF viewer's `/3D` annotation support.

## 8.14

**The stroke pipeline is instanced: 16 bytes a segment, not 144.** A stroked line segment is drawn
as a quad — two triangles, six vertices — and the vertex buffer used to hold all six, each carrying
the segment's two endpoints (`aP0`, `aP1`) plus a per-vertex `(side, end)` selector: 6 floats x 6
vertices = 144 bytes to describe one line. The endpoints were identical across all six.

Now one INSTANCE per segment carries just the two endpoints (`vec2` + `vec2` = 16 bytes), and the
six quad vertices are expanded from `gl_VertexIndex` in `stroke.vert` against a constant corner
table. `DrawPersistentStrokes` / `DrawStrokeSegments` take a SEGMENT count and issue
`vkCmdDraw(6, segmentCount, ...)`; the binding is `VkVertexInputRate.Instance` at a 4-float stride.
The rasterised result is unchanged — the corner table reproduces the old six vertices in the same
winding — so this is a memory/bandwidth change with no visual one.

It is a ~9x cut in stroke vertex data, which on a dense CAD sheet (millions of hatch segments) is
the dominant cost in three places at once: GPU vertex memory, the managed heap that holds the
buffer before upload, and the on-disk geometry cache. One sheet of a 47-page architectural set went
from ~1.2GB of resident stroke vertices to a small fraction of that.

**Vertex-format break for stroke producers.** A consumer feeding `DrawPersistentStrokes` /
`DrawStrokeSegments` must now emit 4 floats per segment (`P0.xy, P1.xy`) rather than 36, and pass a
segment count rather than a vertex count. Nothing else in the draw API changes.

## 8.13

**BEHAVIOUR CHANGE: text sits on the FACE's baseline, not on its own ink.** Following DIR.Lib 8.13,
`DrawText` stops centring the measured bounds of the run it was handed. Centring the ink made the
baseline a function of the text: "a" landed at one height, "b" lower because its ascender inflated the
box, "g" higher because its descender did. One label never looks wrong; a ROW of independently centred
labels cannot agree, which is where it shows (a board's file letters step at b, d and g; a toolbar steps
wherever one caption carries a descender). Every vertically centred run moves slightly, and runs that
used to disagree now line up.

The formula is no longer restated here. It lives in `DIR.Lib.TextBaseline` (`LineHeightFactor`,
`LineHeight`, `WithinLine`), because it had four copies, this renderer among them, one of them inverted.
Face metrics come from `SdfFontAtlas.Rasterizer.GetVerticalMetrics`; a face that declares no hhea falls
back to the run's ink exactly as before.

**Deferred destruction** (`VulkanContext.DeferDestroy` / `VkRenderer.DeferDestroy`,
`PendingDeferredDestroys`). Hand an image view, image, memory, buffer or shared-pool descriptor set to
the context instead of destroying it, and it is destroyed once every frame that could reference it has
retired: the frame being recorded and every frame in flight. Retirement is read off the fence waits the
frame loop already performs, so it costs no drain. `VkTexture.Dispose` goes through it, so disposing a
texture in the frame that drew it is now legal.

The drains this library offered retire PREVIOUS frames and cannot retire the one being recorded, so
destroying a resource mid-frame was correct only if nothing earlier in the same frame had bound it, a
property of call order across hooks a consumer usually does not own. Getting it wrong reaches the GPU as
a dangling view: validation reads it as `vkCmdBindDescriptorSets(): ... VkImageView was destroyed`, the
NVIDIA driver as `nvlddmkm 153`, Windows as a `LiveKernelEvent 141` watchdog with the process gone.
Adoption notes, including the per-frame descriptor-set pattern that goes with it, are in
`docs/deferred-destroy-adoption.md`.

**`TryWaitAllFramesIdle`**, the between-frames drain a consumer needs: bounded, and it reports failure
rather than throwing, so a wedged GPU cannot hang the caller.

**Swapchain teardown flushes the present queue.** `RecreateSwapchain` / `PrepareForSurfaceLoss` /
`RecoverFromGpuError` follow their bounded fence drain with a `vkQueueWaitIdle` before
`CleanupSwapchain` destroys the swapchain and its per-image render-finished semaphores, but only when
the drain SUCCEEDED. The fence drain waits on graphics submits; present is a separate queue operation
gated by no fence, so a fence-only drain left the images and present semaphores still in use by
`vkQueuePresentKHR` when they were destroyed. Validation flagged it on every window resize
(VUID-vkDestroySwapchainKHR-swapchain-01282, VUID-vkDestroySemaphore-semaphore-05149): harmless on
desktop NVIDIA, which serialised, but this is the destroy-while-in-use pattern that surfaces on Adreno
as a rejected `vkQueueSubmit`. Gating on drain success keeps the no-hang property: a wedged GPU still
forces the teardown, exactly as before.

**The damage pass's `loadOp LOAD` is ordered after its own layout transition.** The shared external
dependency admitted `COLOR_ATTACHMENT_WRITE` alone, and a LOAD reads, so synchronization validation
reported a READ_AFTER_WRITE hazard once per swapchain image on every partial frame. The read is
admitted for every pass, not just the LOAD one: dependencies are not among the things render-pass
compatibility exempts, so widening one pass alone made it incompatible with the framebuffers and
pipelines built against the other (VUID 00904 / 02684). Only consumers of `AddFrameDamage` ran it.

**Inspector:** `move` is declared as an MCP tool. The verb has been on the wire since 8.9, added
precisely because press-based verbs cannot drive hover (`click`, `drag` and `press_hold` all arrive
with a button DOWN), but it was never declared, so every MCP-driven session had a hole exactly where
hover behaviour lives. A missing argument now says WHICH argument was missing.

CI runs on current action majors (checkout v7, setup-dotnet v6, upload-artifact v7).

## 8.12

Follows DIR.Lib to 8.12: `Node.Anchored` places one child at its own measured size inside a rect rather
than filling it, so a floating panel over a canvas states WHERE it floats instead of computing pixels;
`IconKind.Pan` and `IconKind.IBeam` join the icon family; and DIR.Lib's icon drawings move to their own
partial file.

Nothing here changes. `Anchored` is arranged by the engine and reaches `PixelWidgetBase`'s painter as
one more node in the pre-order list, and the two marks arrive at `DrawLayoutIcon` as two more kinds.
The inert `DIR.Lib` `PackageVersion` moves with the pin, as ever.

## 8.11

Follows DIR.Lib to 8.11, which adds `IconKind.Search` -- a lens with a handle -- and
`Content.TextInput.LeadingIcon`, a mark drawn inside a field at its leading edge with the text starting
after it.

Nothing here changes for either. The mark reaches `PixelWidgetBase`'s existing `DrawLayoutIcon` as one
more kind, and the field's leading room is stated in `TextInputRenderer` where the measure pass and the
paint both already read their shared inset from. What it buys a consumer is that a search bar stops
being a box with a placeholder in it: the affordance survives the first keystroke, which is when a
reader glancing back at a bar full of results needs to know which box was the query.

The inert `DIR.Lib` `PackageVersion` moves with the pin, as ever -- it decides nothing here, and left
behind it rots in silence.

## 8.10

Follows DIR.Lib to 8.10, where an icon takes its size from the text it sits beside:
`Layout.Builder.Icon(kind, color: c)` with no size puts the mark at `Content.Icon.TextSizeRatio` of
the font size of the run in the same container. Nothing in this repo has to change for it -- the
resolution happens in DIR.Lib's container factories, so `Content.Icon.Size` still arrives at
`PixelWidgetBase`'s painter as one concrete number -- and a consumer's chip stops restating its own
label's font size to size its caret.

The inert `DIR.Lib` `PackageVersion` moves with it. It decides nothing here (the csproj reaches
DIR.Lib through the submodule `ProjectReference`), but left behind it rots silently, which is what
`fix(deps)` had to correct once already after it had drifted to 7.16 against a submodule at 8.3.
## 8.9

Damage-based repaint: a frame can preserve the previous one and paint only the region that changed.
`BeginFrameRenderPass` picks a `loadOp = Load` variant of the swapchain pass — identical in
attachments, samples, subpass refs and dependency pair, so the pre-baked pipelines stay compatible —
and confines the frame to the accumulated damage. Render area and scissor are the region while the
VIEWPORT stays the full surface: an app submits geometry in surface coordinates, so shrinking the
viewport would squash the frame into the region rather than crop it to it. `AddFrameDamage` /
`MarkFullFrameDamage` declare it, and every clip is intersected with it, since DIR.Lib has already
intersected a clip with its parents but knows nothing about damage.

Damage is tracked PER SWAPCHAIN IMAGE, which is the only hard part. With 2-3 images in rotation the
image acquired this frame holds the frame from 2-3 frames ago, so what must be repainted into it is
the union of every frame's damage since THAT image was last painted. Using the current frame's damage
leaves stale pixels that appear only at particular frame counts and only in the images that missed an
update — an intermittent glitch with no visible connection to bookkeeping. `SwapchainDamage` is a
separate type so that algorithm is testable with no device, and nine tests do exercise it.

MSAA takes the clearing path unconditionally: the multisample attachment is transient and cannot be
reloaded from the resolved image, so `CreateLoadRenderPass` returns Null, which is correct rather than
merely safe.

`VulkanContext.CachedLayer` renders expensive, rarely-changing content into a sampleable secondary
target on the live device, so a frame that changed nothing but its chrome blits it instead of
re-shading it. It is a sibling of `ThumbnailCapture`, not of `CreateOffscreen` — the pass is recorded
into the frame's OWN command buffer from `OnPreRenderPass`, so there is no extra submit, no extra
fence and no queue-stalling wait. ONE TARGET PER FRAME IN FLIGHT is correctness, not tuning: the
frame fence retires frame N-2, never N-1, so a single shared target would be rewritten while the
previously submitted frame was still sampling it — the hazard `VkFontAtlas.Grow` guards with a drain,
and the one an Adreno X1-85 answers by failing the next submit.

`SdlVulkanWindow` implements `SharpAstro.AppShell.IActivatableWindow`, so a single-instance hand-off
can bring a window forward with the correct restore behaviour. The three members it needed already
existed; what was missing was the RULE, and two applications wrote it independently and both got it
wrong the same way — restore, then raise. Restoring un-maximises, so opening a second file knocked a
maximised window back to its floating size; raising without restoring leaves a minimised window
off-screen at -21333,-21333 while holding input focus. Restore only when actually minimised.

That adds a dependency on SharpAstro.AppShell, one small managed assembly whose own only dependency
(`Microsoft.Extensions.Logging.Abstractions`) this package already had. It is taken from nuget.org
through upstream's conditional-sibling shape kept verbatim, unlike the DIR.Lib submodule reference:
AppShell is not forked, so the PackageReference the probe falls through to is the real package rather
than a different codebase.

The inspector gains a `move` verb. Both existing pointer verbs press a button, and a press means
something — in a viewer it starts a pan — so hover-driven behaviour was undrivable: highlights,
tooltips, the cursor shape, and any repaint decided by where the pointer is.

`SdlWindowView.OnBeforeFrame` runs once a frame is committed to, before the pass opens, because damage
has to be declared there — by the time `OnRender` runs the pass is already begun. Deliberately
distinct from `CheckNeedsRedraw`, which is a predicate deciding WHETHER to draw; giving that side
effects would mean a declined frame reconfigures the next one.

`TryWaitPriorFramesIdle` is public, so a consumer destroying its own sampled texture can use the
bounded drain instead of an unbounded `vkDeviceWaitIdle`.

`ResizeOffscreen` now drains with `TryDrainDevice` instead of `vkDeviceWaitIdle` — it was the last
unbounded, unguarded device wait in the class, where every sibling is either bounded or
`IsGpuStuck`-guarded and `VkFontAtlas` engineered its per-Flush wait away entirely. On a wedged GPU the
raw call blocks its caller forever, and the offscreen path is reached from export, so an export would
hang without ever saying why. `TryDrainDevice` rather than `TryWaitPriorFramesIdle`, because the latter
excludes the CURRENT frame's fence by design — right for a mid-record atlas grow, wrong here, since
`ResizeOffscreen` runs between frames where that index can still hold the pending submit most likely to
be reading the old target. Same trade the swapchain-recreate and surface-loss paths already take: on
timeout it proceeds, because the target is being destroyed and recreated regardless.

Follows DIR.Lib to 8.9 for `LayoutDamage` and its unconditional layout capture.

CI: every job is now bounded by `timeout-minutes` (GitHub's default is six hours, which is how an
unreachable apt archive parked a job for an afternoon), and the WebKitGTK install goes through a
caching composite action, so the common path reaches no archive at all.

## 8.3

Follows DIR.Lib to 8.3, which is additive. Nothing in this repo's own source changes, but the
package re-exports DIR.Lib, so a consumer gains it by taking this version. TabStripTree.Build
describes a tab strip as one Layout.Node tree, and TabBar paints through it, so the strip exists
once rather than as a pixel copy and a cell copy that can drift. The + stays imperative: it belongs
to a tab BAR rather than a tab strip, and a nav rail has none. TabStripSide { Top, Bottom, Left,
Right } derives orientation from the side rather than taking it alongside, so a nav rail is the same
widget and a vertical strip on the top edge cannot be stated; TabSizing { Content, Uniform } comes
with it, because sizing a vertical tab by content sets its height from the WIDTH of its label. The
new Render overloads take the strip's whole RectF32, which Bottom and Right need to learn where the
far edge is. TabItem<T> hands a press back as TabClick<T> -- the value the tab selects -- instead of
an index the host maps through a switch nothing checks, and carries Icon, IsEnabled and Tooltip.
CompositeWidget<TSurface> is the base for a widget painting OTHER widgets into one surface: a
child's regions live on the child, so a host asking only the composite misses every control the
children registered -- the pixels stay right and the controls stop answering. It declares its
children once, in paint order, and every aggregate query derives from that list. Also IconKind.Plus
/ Minus, a stepper's pair sharing one bar thickness and centre line.

## 8.1

Follows DIR.Lib to 8.1, which is additive. Nothing in this repo's own source changes, but the
package re-exports DIR.Lib, so a consumer gains it by taking this version. PixelWidgetBase.Pointer
tells a widget where the pointer is; Layout.Node.BgHover(colour) paints a second fill while it is
inside a node's rect, and RenderDropdownMenu highlights the row under it (that one previously
tracked the keyboard alone, so a dropdown stayed dead under the mouse). The fill resolves against
the rect the engine arranged, so what lights up and what the pointer is over cannot drift apart --
which is what a consumer computing the rect a second time could never guarantee. Inert unless a host
sets Pointer, and a host that does must repaint on motion.

## 8.0

Follows DIR.Lib to 8.0, which is BREAKING (see its MIGRATION.md). Nothing in this repo's own source
changes, and nothing here uses the type -- but the package re-exports DIR.Lib to its consumers, so a
consumer that draws a tab strip has to port. TabBar becomes TabBar<TSurface> :
PixelWidgetBase<TSurface>: it takes its Renderer at construction and reads the window's font,
fallback chain and DPI from the shared WindowUiSettings, so TabBar.Scale and the font/fallback
constructor arguments are gone and Render drops its renderer parameter. The bar now registers each
tab, each close button and the + as it paints them, so a press resolves against the strip that is on
screen -- and, through 7.25's frame stamp, against no strip at all on a frame the host did not draw
one. Additive in the same release: ClickableRegionTracker.Regions /
PixelWidgetBase.RegisteredRegions (read back your own regions without the per-call copy) and
PixelWidgetBase.DrewThisFrame, for the input a host resolves by geometry rather than by region and
which therefore cannot decline a stale frame by itself.

## 7.33

Follows DIR.Lib to 7.24: TabBar.Font / .Pad / .Border become public, joining .Height, for a host
that has to draw a tab somewhere the bar does not. A LATER BUILD of the same 7.33 also moved the
submodule to DIR.Lib 7.25 (frame-scoped regions + wrapping result lists) without bumping
Major.Minor, so two 7.33.<run> builds carry different DIR.Lib content. Bump the version with the
pin, not only when this repo's own source changes: the pin IS what this package ships.

## 7.32

Follows DIR.Lib to 7.23: which field in the window has the keyboard, and where its caret was drawn,
move onto the shared WindowUiSettings. Held per widget, a host had to know WHICH widget painted the
focused field in order to ask the right one -- a question with no stable answer, since the field
holding the keyboard moves between them.

## 7.31

Follows DIR.Lib to 7.22, and surfaces the input-method events SDL was already delivering.
SdlWindowView.OnTextEditing carries the in-progress composition (its text, its caret and its
selection length); this backend had a TextInput case and no TextEditing case at all, so the preedit
was read off the wire and dropped. With a CJK input method every keystroke before the commit arrives
on that event and nowhere else, which is why an app handling only TextInput can accept Latin and
nothing else and shows nothing on screen while the user types. An empty text is the normal
end-of-composition signal and is passed through rather than skipped, since the app has to be told
the preedit is gone. SdlVulkanWindow.SetTextInputArea wraps SDL_SetTextInputArea. SDL does not track
the app's caret, so this is the only way the platform can learn where text is being typed; without
it an input method has nothing to anchor to and puts its candidate window over the text.
DebugInspector's layout dump now names a text field: what it holds, whether it has the keyboard, and
its placeholder -- the last so an EMPTY field is still identifiable rather than an anonymous rect.
"Which box is focused" is the question every text-input bug starts from. DIR.Lib 7.22 makes
TextInputRenderer.Render binary-breaking (it returns the caret rect and takes a fallback resolver),
so this is a required rebuild rather than bookkeeping.

## 7.30

Follows DIR.Lib to 7.21: Layout.Builder.TextInput(state, fontSize) declares an editable field, and
the painter registers its TextInputHit + I-beam, so click-to-focus, blur-on-outside and Tab order
follow from a registration a consumer cannot forget to make. Adds TextInputFocus and
TextInputInteraction. Additive -- no API this backend implements changed.

## 7.29

Follows DIR.Lib to 7.20, where clips nest: a push inside a push draws in the intersection and a pop
restores the enclosing clip. The base owns the stack now, so VkRenderer implements
ApplyClip/ClearClip -- one absolute, already-intersected region, which is what vkCmdSetScissor takes
anyway -- instead of overriding PushClip/PopClip. BeginFrame and BeginOffscreenFrame drop the stack:
a scissor lives on the command buffer, so a fresh one has already discarded the region, and a widget
that threw between its push and its pop must not leave every later frame clipped to a rect nobody
can name.

## 7.28

Follows DIR.Lib to 7.19: Renderer.DrawTriangles, a triangle list with a scanline default written in
terms of FillRectangle, so every backend has one. VkRenderer overrides it, so a caller holding this
as a Renderer<TSurface> keeps the FlatPipeline: one draw call for the whole list, against a
row-at-a-time fill that here costs a quad and a push-constant update per scanline. That was the last
primitive the viewer's shared chrome had to reach past the abstraction for.

## 7.27

Follows DIR.Lib to 7.18: RgbaImageRenderer honours PushClip/PopClip, and PixelWidgetBase gains
PushClip(x, y, w, h) / PopClip so a widget can state a clip in the same x/y/w/h terms as everything
else it draws. No renderer change -- VkRenderer already overrode the pair onto the Vulkan scissor --
but it is what lets a widget stop calling SetScissor by name, which is the last renderer method the
shared chrome could not express through the base class.

## 7.26

Four rounds, three of which shipped under 7.25 without saying so; this entry covers them. The
inspector synthesizes double and triple clicks (a `clicks` count on click / clickLabel, emitted as
the full run 1..N, the way SDL delivers one). CursorKind.ToSystemCursor maps DIR.Lib's cursor
vocabulary to SDL's, beside the existing scancode and keymod mappings — so a widget states a cursor
on the region it drew and the host does not keep its own table. The inspector's `scroll` now takes
modifiers too, like click and drag: a wheel tick usually means something else with one held (Ctrl
zooms, Shift scrolls sideways), and an app reading that off the global keyboard state cannot be
driven there by any synthesized input. And DIR.Lib to 7.24: cursor as a property of a clickable
region, TextInputColors as a per-call parameter, caret icons, a width sample for a readout that must
not jitter, and padding stated per axis. All additive.

## 7.25

Additive, two things. SdlVulkanWindow.SetIcon(RgbaImage) sets the window's icon: title bar, taskbar
or dock button, alt-tab entry (synced from upstream SdlVulkan.Renderer 7.12). Not a Windows
convenience, since a Win32 icon resource in the exe covers Windows and nothing else: X11 and Wayland
read the icon off the window. SDL copies the pixels, so the caller's array is free on return. And
DIR.Lib 7.17, whose TabBar takes a Pointer position and hovers itself from it: the tab under the
pointer lifts, and its close mark plates. Both additive; a consumer that calls neither is unchanged.
The viewer consumes both.

## 7.24

Rebuild on DIR.Lib 7.16, whose TabBar can draw a "+" after the last tab and report its click
(ShowNewTabButton / NewTabActive / NewTabHovered / HitNewTabButton). Additive there and here:
nothing in this repo consumes it, and a consumer that sets nothing is unchanged. Bumped so the
viewer can pin a renderer whose DIR.Lib carries the button, rather than reaching around the chain
for it.

## 7.23

The fallout of one device-loss investigation, plus a rebuild against DIR.Lib 7.15. Five renderer
changes had landed on main since 7.22 WITHOUT a version, so they were publishing as 7.22
build-counter republishes; this bump is what gives them one. Four of the five are things the
validation layer had been unable to tell us, because on a host with no Khronos layer installed it
was never actually running (see below). VK_ERROR_DEVICE_LOST is now TERMINAL rather than entering
mid-frame recovery. Recovery rebuilds sync and the swapchain and continues, which cannot work once
the device and every object it owns are gone: each later call returns DEVICE_LOST again, so the
rebuild re-failed on the next submit. Three attempts burned inside 34ms, and the right outcome (hand
the session to a successor) was reached only because the recover-streak detector happened to trip.
That detector logs a "recovery storm" and asks the app to shed load, which reads as a workload
problem and points diagnosis away from a dead device. Event 110 "Recovering swapchain" is now logged
only for errors that path can actually recover; new event 115 names device loss for what it is. Each
swapchain image gets its OWN present-wait semaphore. render-finished was allocated per frame in
flight and indexed by frame slot, but vkQueuePresentKHR's wait completes only when the presentation
engine is done with the IMAGE, and image count is not bounded by MaxFramesInFlight, so with more
images than frames a new submit could re-signal a semaphore an earlier present was still waiting on
(VUID-vkQueueSubmit-pSignalSemaphores-00067). Its lifetime moves onto the swapchain so a resize that
changes the image count rebuilds it. Latent while presentation is regular, which is why it surfaced
over Remote Desktop. ONE shared subpass-dependency pair, in VulkanDevice.FillSubpassDependencies,
used by all six creation sites. Every pass had written its own: render-pass compatibility covers
dependencies, so pre-baked pipelines drawing in a pass with a different dependency count were
rejected (VUID-vkCmdDraw-renderPass-02684, "2 != 1"). And every external dependency used
srcAccessMask = 0, which orders execution but creates no memory dependency against the previous
frame's storeOp write, so the layout transition raced it, a WRITE_AFTER_WRITE hazard on every
alternating pair of frames. srcAccessMask is now ColorAttachmentWrite. Diagnostics that were lying:
validationReport now reports layerAvailable and active, not just the DEBUG + SDLVK_VALIDATION gate.
On a host with no Khronos layer installed it read enabled:true with zero messages and zero hazards,
indistinguishable from a clean run, and was read as one during the very investigation above. A zero
message count is evidence only when active is true. The sync-hazard counter matched only the retired
"SYNC-HAZARD" token and so reported zero while the ring buffer plainly held WRITE_AFTER_WRITE
messages. And a DiscreteGpu PREFERENCE (never a requirement, so integrated-only hosts are
unaffected): both pickers took the first device meeting their requirements, so a machine with both
cards ran on whatever the loader enumerated first, possibly integrated graphics on shared memory.
New event 501 records device name, type, driver version, API version, queue family and how many
devices were enumerated. Nothing had logged the selection, so no GPU report could be attributed to
hardware from our own logs. Then the rebuild: DIR.Lib 7.15 makes TextFit.ShrinkToWidth return a size
it actually measured, so a run fitted with TextTrim.Shrink stops drawing a fraction of a pixel past
its rect. Shrink is opt-in and the default is End, so a consumer that never asks for it renders
byte-identically. That release also records, late, a behaviour change that shipped unannounced in
DIR.Lib 7.12 (this repo's 7.20): a whitespace advance comes from the font instead of being borrowed
from the 'n' glyph, and in DejaVu every measured space had been 1.99x too wide. Text laid out with
space padding is therefore narrower since 7.20; U+2007 FIGURE SPACE is the pad that holds a column,
being digit-width by definition.

## 7.22

Rebuild against DIR.Lib 7.14: a glyph the SDF atlas can never rasterize is given up on after three
attempts and recorded blank, instead of being retried forever. No renderer code change. What changes
for a consumer is the failure MODE of a broken font: previously the atlas re-claimed such a glyph
every frame and IsDirty never cleared, so VkRenderer.FontAtlasDirty stayed true for the session — an
event loop redrew forever and an offscreen capture waited out its whole frame budget, reporting a
pixel difference with nothing about fonts anywhere in it. Now the glyph draws as nothing, the atlas
reports clean, and one line per font names the glyph, the attempts, and whether the font was
registered when we gave up. Also brings ManagedFontRasterizer.IsFontRegistered.

## 7.21

Rebuild against DIR.Lib 7.13, which follows SharpAstro.Fonts 1.11. No renderer code change, no API
change — but GLYPH SELECTION changes, so text can draw differently. An embedded PDF subset whose
only cmap subtable is Mac Roman (1,0) now selects through it rather than falling back to treating
the char code as a glyph index. macOS Quartz writes exactly that shape, and the guess failed two
ways at once: inside a large subset it picked an unrelated glyph (Korean body text drew as
correctly-shaped nonsense in codepoint-sorted order), and in a 2-20 glyph Latin subset it indexed
off the end and drew nothing (commas, list markers and page numbers vanished). Also carried in from
the skipped 1.10 pin: three compounding TrueType hint-interpreter fixes and RTL mark attachment.
Consumers rasterizing text through an unhinted path (e.g. MTSDF atlases) see no baseline movement
from the hinting half.

## 7.20

Rebuild against DIR.Lib 7.12: TextInputRenderer takes a palette, the same shape TabBar got in 7.10 —
a TextInputColors record whose defaults are what it has always drawn, a FromPalette factory, and a
settable TextInputRenderer.Colors. Additive: a consumer that sets nothing is unchanged. No renderer
code change; the pin moves so consumers building through this submodule see the new API.

## 7.19

The Inspector follows SharpAstro.Png 3.4 -> 3.8, and the hashed shader sources are pinned `-text` in
a new .gitattributes. The bake check compares source hashes against Shaders/spirv/sources.sha256,
which makes those files' exact bytes part of the build contract — with no .gitattributes,
core.autocrlf could rewrite them on checkout and SVR0001 would fire on a tree nobody had edited.
Re-baking is the WRONG response to that: the recorded hashes are right and the SPIR-V is
byte-identical, so committing hashes taken from CRLF bytes only moves the warning to Linux.
Submodule bump for DIR.Lib's matching refresh.

## 7.18

Dependency refresh, no renderer code change. Vortice.Vulkan 3.2.1 -> 3.2.3 is the only one a
consumer sees, and it moves in TWO places: the CPM file here and an inline pin in
SdlVulkan.Renderer.Tests, which sits outside src/SdlVulkan.Renderer/ and so is out of that
Directory.Packages.props' scope. Bumping one alone fails restore on NU1605. Also
Microsoft.Extensions.Logging.Abstractions 10.0.0 -> 10.0.10, SourceLink 10.0.300 -> 10.0.301, and a
DIR.Lib submodule bump carrying the same kind of refresh.

## 7.17

A space moves the pen. An ink-free glyph keeps its own hmtx advance now (DIR.Lib 7.11 submodule
bump), and VkFontAtlas caches that blank entry rather than discarding it, so a glyph with no pixels
stops re-entering rasterization on every draw. Text laid out through the bitmap atlas gets the
font's real space width where it used to get the 'n' glyph's — visibly narrower, and correct. Text
laid out by glyph id (any shaped run) gets a space at all, where it previously ran every word
together.

## 7.16

Rebuild against DIR.Lib 7.11: BREAKING for anyone constructing a UiPalette. It gains eight roles and
becomes a sealed record with required members, so default(UiPalette) — every role transparent black,
painted silently — is now a compile error instead. Five of the new roles derive from the role they
extend. Note HeaderText moved from required to derived-from-Accent: a palette that omits it gets the
accent, so a near-white header colour has to stay stated rather than falling out of the port. No
renderer code change; the pin moves so consumers building through this submodule see the new API.
See DIR.Lib's MIGRATION.md.

## 7.15

Rebuild against DIR.Lib 7.10: TabBar's colours are a palette rather than private static fields, so a
host can theme the tab strip, and TabBarColors.FromPalette derives that palette from the UiPalette
chrome roles a host may already hold. No renderer code change; the pin moves so consumers building
through this submodule see the new API.

## 7.14

BeginThumbnailCapture takes the clear colour instead of assuming white. The capture target was
cleared to a hard-coded (1,1,1,1) "white page background", which is fine while every page is white
and wrong the moment a consumer renders a dark one: the ink inverts, the sheet does not, so the
thumbnail comes back as light text on white paper and reads as a bug in the ink. Now explicit,
matching BeginOffscreenFrame, which has always taken its clear colour. Breaking for direct callers
of BeginThumbnailCapture / BeginThumbnailCapturePass -- pass white to keep the old behaviour.

## 7.13

The inspector says what a refused request was missing, and a batch step and a direct call accept the
same parameters. Brings the inspector request parsing in line with upstream, which had moved on:
readers now NAME the parameter they wanted instead of surfacing GetProperty's "The given key was not
present in the dictionary", which reached the caller as the entire explanation of a refused step and
said nothing about which step, which parameter, or what would have worked. `text` accepts either the
batch contract's spelling or the `s` the direct verb has always sent, and postSignal reads args as
an object OR argsJson as a string -- a batch step passing argsJson previously reported success and
then did nothing, because every field fell back to its default, and for a set-view signal whose
fields all mean "leave unchanged" that is indistinguishable from working. DEBUG-only surface; 12
tests come with it.

## 7.12

Offscreen rendering survives a rejected queue submit instead of hanging on it. The dropped frame
handling added in 7.9 covered the swapchain path only, so on hardware that returns
VK_ERROR_INITIALIZATION_FAILED from vkQueueSubmit an offscreen frame threw, and threw before
advancing the frame index — leaving that fence reset with nothing behind it to signal it, so the
next offscreen frame waited on it forever with no timeout. A TIFF export or thumbnail capture of a
heavy page could therefore hang outright. Offscreen submits now retry a rejection (the frame IS the
output, so it is retried rather than dropped) and a submit that never took is recorded as such, so
no wait can park on an unsignalable fence. Readback does the same and fails loudly rather than
returning an unwritten buffer as pixels. Also fixes the swapchain path leaving a stale pending mark
when a rejection followed an earlier successful submit on the same frame index. No API change.

## 7.11

Rebuild against DIR.Lib 7.9: a PDF font's own /Encoding now decides which glyph a char code selects,
so embedded name-keyed CFF subsets stop drawing unrelated glyphs. Carries SharpAstro.Fonts 1.8,
which loads PDF subset fonts whose cmap a subsetter malformed (previously the whole font was
rejected and every glyph fell back to a system face). No renderer API change.

## 7.10

Lockstep rebuild against DIR.Lib 7.8 (submodule 7.7 -> 7.8). No renderer code change; the CPM pin
already tracked 7.8.*, so CI was restoring 7.8 while a local build used the 7.7 submodule — this
closes that split. BEHAVIOUR CHANGE flowing through from DIR.Lib, not additive: the PIXEL painter
now fits every text run to the rect the layout engine arranged for it, so a run left on the default
TextTrim.End ellipsizes where it used to start at the rect edge and overhang its neighbour.
TextTrim.Shrink (scale down, keep every character) and TextTrim.None (the previous overhang) are
available for callers that want the old shape. DIR.Lib 7.8 also derives AssemblyVersion from its
version property, fixing the same ships-the-wrong-identity bug this repo had: DIR.Lib and
DIR.Lib.Shaping had published 6.4.0.0 since 6.5.

## 7.9

GPU-WEDGE ROOT CAUSE, plus the forensics that found it. Four parts, one release. (1) wedge
forensics: the breadcrumb logged when a fence sticks now carries device-object churn
(buffer/image/memory create+free since the frame that hung, counted on VulkanDevice and noted by
VkTexture) and the idle gap since the last clean frame — a field wedge arrived with the atlas half
of the breadcrumb reading all zeros and the actual churn had to be dug out of host logs. The "fence
late" line carries the gap too, so recovered episodes are data points as well. NEW:
VulkanDevice.ChurnCounters/DeviceChurn, VkRenderer.DeviceChurnBreadcrumb. ADDITIVE. (2) a frame's
fence is now reset immediately before the submit that signals it, instead of in BeginFrame.
Everything in between — atlas flush/grow, texture uploads, the consumer's whole render callback —
used to run with a fence only EndFrame could signal, so any exit that skipped EndFrame orphaned it
permanently and the next BeginFrame waited on a fence nothing would ever signal: indistinguishable
from a hung GPU, but with the GPU idle. Same fix on the offscreen path, where the wait is unbounded
and the hang therefore permanent. Added VulkanContext.AbortFrame (resolve a frame abandoned mid-way,
so its acquire semaphore and image do not leak into the next), VK_ERROR_DEVICE_LOST now reported
distinctly instead of as a generic VkException, and a submission ledger on the wedge breadcrumb that
states whether the fence being waited on ever had work submitted under it. Queue/command-pool
external synchronization is asserted (single-owner submission) rather than locked. NEW:
VulkanContext.AbortFrame/SubmissionLedger/DeviceLost, VkRenderer.AbortFrame/SubmissionLedger.
ADDITIVE. (3) the actual root cause of the field wedges, caught live by the ledger above: Adreno's
spec-illegal VK_ERROR_INITIALIZATION_FAILED from vkQueueSubmit is a REAL rejection (the work never
executes; the old tolerance believed it did), so a rejected submit is now a dropped frame: fence
left unmarked and never waited on (a fence with no submission behind it cannot signal — waiting on
it WAS the wedge), present skipped, acquire semaphore replaced, pending thumbnail capture cancelled,
frame index advanced. The stuck-fence recovery drain now runs even in stuck mode (it is on the
sacrificial task, and destroying sync objects a slow frame still references is what poisoned the
driver into rejecting), and all drains skip fences with nothing pending. (4) the renderer's
unconditional diagnostics (wedge forensics, recovery decisions, driver anomalies) now log through an
injectable ILogger: SdlVulkanLog.Logger (defaults to a stderr sink so unwired hosts keep today's
output — the breadcrumbs are the black box and must not be droppable by omission), with every
message a [LoggerMessage] source-generated event (compile-time templates, IsEnabled-gated,
allocation-free; ids 1xx event loop, 2xx context, 3xx readback, 4xx validation). Templates reproduce
the exact former stderr text, so log-grepping tooling is unaffected. RenderDiag stays
[Conditional("DEBUG")] raw stderr. NEW dependency: Microsoft.Extensions.Logging.Abstractions
(interface-only). NEW: SdlVulkanLog. ADDITIVE. Later in 7.9, no X.Y bump: AssemblyVersion is now
DERIVED from VersionMajorMinor for every project instead of being restated as a literal per csproj,
and the DIR.Lib pin moves 7.5.* -> 7.8.* to match upstream. Both come down from upstream, which had
already corrected them. The AssemblyVersion half matters most: CI stamps -p:Version and
-p:FileVersion but NOT -p:AssemblyVersion, so where a stale VersionPrefix only spoiled a local pack,
a stale AssemblyVersion SHIPPED — SdlVulkan.Renderer published 6.11.0.0 and both WebView packages
6.0.0.0 right through 7.9, each against an informational version a major or more ahead. All three
are now 7.9.0.0. Deliberately not a minor bump: the value only moves UP, and the runtime rejects a
loaded assembly LOWER than the compiled reference, never higher, so anything already built against
6.11.0.0 or 6.0.0.0 keeps loading.

## 7.7

Lockstep rebuild against DIR.Lib 7.7. No renderer code change. A layout tree can now state a
hyperlink, and a text run which end it trims (new DIR.Lib TextTrim); ten dangling <see cref> targets
in DIR.Lib's own docs now resolve. DIR.Lib's SharpAstro.LALR.CC pin also moves 3.1.* -> 4.7.*, which
only ever bound CI — a local build swaps it for the sibling ProjectReference, which is how it sat a
major behind unnoticed. ADDITIVE.

## 7.6

Descriptor sets and samplers stop being a fixed budget. The descriptor pool was created once with
maxSets 512 and treated as a hard ceiling, so a page carrying a few thousand small images ran the
device out — and since the glyph atlas draws from the same pool, the allocation refused could be
TEXT as easily as an image. Pools cannot be resized, so growth is by chaining: VulkanDevice adds a
pool when the current one is spent. Freed sets are recycled rather than returned to the driver
(every set shares one combined-image-sampler layout, so a returned set is re-pointed at another
image), which also keeps handles valid instead of invalidating one an in-flight command buffer may
still name, and keeps the pool count tracking PEAK live sets rather than growing with churn.
VkTexture no longer creates a sampler each — identical settings every time, no per-image state, and
thousands of them walked into maxSamplerAllocationCount as a second ceiling behind the first; one
shared VulkanDevice.LinearClampSampler replaces them. NEW: DrawTexturedQuadRegion draws an arbitrary
quad from a UV sub-rect, so many small images can share one atlas texture. FIX: a failed page
allocation in the SDF backend left its image and memory behind. Lockstep with DIR.Lib 7.6. ADDITIVE.

## 7.5

Lockstep rebuild against DIR.Lib 7.5. No renderer code change. DIR.Lib now resolves installed fonts
by the family each face DECLARES rather than by a guessed file name, so a face whose file isn't
named after it resolves (Segoe UI Symbol lives in seguisym.ttf) and so does every face of a .ttc
past the first — both previously unreachable. Those are named by FontFaceId (path, or path#index for
a collection), which ManagedFontRasterizer honours; since the id is already every glyph/atlas/shaper
cache key, faces separate without further plumbing. FontFallbackResolver gains
TryResolveFont/CanRender and role-based construction, and PixelWidgetBase.FontFallback carries
per-run fallback into the declarative painter. Additive.

## 7.4

Lockstep rebuild against DIR.Lib 7.4. No renderer code change. DIR.Lib gains per-axis
PixelMeasureContext scales plus a CellAuthored factory, so a cell-authored tree arranges on a pixel
surface, and PixelWidgetBase Arrange/Paint/RenderLayout gain overloads taking the measure context
itself — the painter reads FontPath/FontScale/radius from the same object the measure pass used,
rather than a dpiScale threaded into both by hand. Additive: the scalar overloads delegate with an
isotropic context, so this renderer paints byte-identically.

## 7.3

DebugInspector folds onto DIR.Lib's DebugInspectorCore (needs DIR.Lib 7.3): 940 -> ~600 lines. Gone
are the private TCP accept loop, the newline framing, the discovery responder and descriptor, the
command queue, the whole InspectorCommand record hierarchy with its BuildCommand parser, and the
batch/wait state machine -- all of it duplicated what the core does. What stays is what is actually
SDL: the verbs, ResolveInputKey/ResolveModifier, and pressHold, now expressed as a background
IDebugInspectorOperation (pressed in Begin, released in Advance, deliberately NOT exclusive so a
screenshot can be taken mid-hold). The prior split had been measured -- only ~48 lines touched SDL --
but that measured SDL-COUPLING, not shareability: a frame-stepped batch calls no SDL function yet
presumes a loop with frames, which the core could not express until now.
ONE discovery protocol: `dir-inspect` on 239.255.77.91:47892, was `sdlvk-inspect` on
239.255.77.90:47891. A sidecar now tells surfaces apart by the `kind` field ("sdl") rather than by
which port answered, and drops replies it cannot drive. The sidecar moves in lockstep.
SECURITY: DebugInspectorOptions.BindAddress defaulted to IPAddress.Any, so this command server --
which injects input, captures the framebuffer and reads app state -- accepted connections from the
whole LAN. The core binds LOOPBACK with no opt-out.
BREAKING, but only in a DEBUG build: DebugInspectorOptions loses BindAddress, Port, DiscoveryGroup
and DiscoveryPort. The core owns addressing, so they could only be accepted-and-ignored, and a
`Port = 5000` that silently does nothing is worse than one that fails to compile. The whole type is
#if DEBUG, so it is ABSENT from this published Release package and no package consumer can hit
this; only a local Debug build against the sibling can, which is exactly who needs telling.
EnableDiscovery survives -- not announcing yourself is still a real choice.
ping now answers with the core's {"ok":true,"protocol":N,"app":"..."} rather than the bare string
"pong". The sidecar accepts BOTH, because it ships separately from the app it drives: its old
`GetString() ?? "pong"` would have THROWN on an object, and its liveness probe would have reported a
healthy app as dead. It also now connects to 127.0.0.1 instead of the discovery reply's source
address, which a Hyper-V or WSL bridge makes the one address guaranteed to refuse a loopback-bound
server. Corrected a sidecar comment claiming the pump drains from OnPostFrame: it is
OnLoopIteration, which is why a minimized window still answers and is correctly reported alive.
4 host-contract tests (suite 43 passed / 1 skipped).

## 7.2

The inspector REFUSES an unrecognised `mods` string instead of resolving it to None. Behaviour
change, deliberate: resolving the unknown to None delivered a BARE key or click, and a bare key is
frequently a different valid binding rather than a no-op -- so a typo ("ctlr") or an unsupported
modifier ("Cmd") surfaced as the app IGNORING a correct chord rather than as a bad request. The
worked example is chess, which flips its board on Ctrl+F while bare `f` selects file f. This also
made ResolveModifier the one resolver in DebugInspector that did not reject what it could not
understand -- ResolveInputKey has always thrown, listing the valid names -- so this is a
consistency fix as much as a safety one. A partial match still resolves ("ctrl+cmd" is Ctrl), so a
chord that CAN be delivered is never blocked; only text with nothing recognisable is refused. The
four sidecar tools carrying `mods` (click, key, drag, pressHold) all default to "None" and say so,
so no existing driver changes. Console.Lib 4.7 made the same call for the terminal inspector's
`key` verb; the two now agree. ResolveModifier/ResolveInputKey are internal (not private) with
InternalsVisibleTo the test project, since what an injected key MEANS is otherwise only observable
against a live SDL window -- 20 tests.

## 6.33

VkRenderer.FillRoundedRectangle: a real GPU override of DIR.Lib 6.20's scanline
fallback. One rounded-box SDF quad per rect instead of one FillRectangle per row, with
antialiased corners, and single-coverage so a translucent fill blends exactly once.
New roundrect.vert/.frag + RoundRectPipeline. The box parameters (half extents, radius)
ride on VERTEX ATTRIBUTES, not push constants, so the shared 84-byte push block stays
byte-identical across every pipeline -- growing it for one pipeline is the per-stage
mismatch that ellipse.vert documents as an llvmpipe shader-compiler SEGV. A zero radius
delegates to FillRectangle, so the square path is untouched.
Re-pins DIR.Lib 6.19.* -> 6.21.* (FillRoundedRectangle lands in 6.20, Layout.Node.Radius
in 6.21; pinning straight to 6.21 keeps this a single re-pin).

## 6.30

DeviceTransform GPU compose (needs DIR.Lib 6.17). VkRenderer.DeviceTransform folds the
content->device affine into the projection in UpdateProjection — the compose stays a
Matrix3x2 (2D affine, no wasted lanes) and only widens to the mat4 push-constant at upload,
so the whole frame (text included) rotates/scales as one. Identity transform is byte-
identical to the previous screen-space projection. Verified via offscreen render + readback
(180° flip moves a top-left fill to the bottom-right).

## 6.19

Opt-in Vulkan validation diagnostics (VulkanValidation). The Khronos validation layer's
output was previously enabled in DEBUG but dropped to the loader's default sink; now a
debug-utils messenger routes it to a prefixed stderr line + a bounded ring buffer. Adds an
opt-in SYNCHRONIZATION validation switch (the GPU memory-hazard checker behind the wedge
class) and a validation_report inspector/MCP tool. Gated: DEBUG or SDLVK_VALIDATION=1
(+ SDLVK_SYNC_VALIDATION=1 for sync); zero cost + no layer in a normal Release build.

## 6.18

Idle the render loop while a window is minimized. A minimized window reports a non-zero
pixel size on Windows, so the old size guard never caught it and the loop busy-spun through
swapchain recreation (~270ms/frame) for invisible frames. Gate redraw on the SDL minimized
flag (SdlVulkanWindow.IsMinimized): ~0% CPU while minimized, instant repaint on restore.
DEBUG-only: inspector minimize/maximize/restore commands + a per-iteration command pump
(SdlEventLoop.OnLoopIteration) so commands drain on a minimized window.

## 6.17

GPU-wedge resilience. Stuck-fence recovery now runs on a sacrificial background task the
render thread only polls (on a truly hung GPU the driver can block INSIDE vkFreeMemory /
teardown — observed on Adreno: the old synchronous recovery froze the render thread
permanently); deadline blown or repeated stuck escalations → OnGpuWedged (new host
callback) + clean loop stop. SDF atlas: per-frame upload BYTE budget alongside the glyph
count cap (MTSDF quadrupled bytes/glyph vs R8), one-frame quarantine for glyphs on a
just-appended page (first transition and first sample no longer share a submission), and
a FrameStats wedge breadcrumb logged at fence-stuck escalation.

## 6.16

Re-pin DIR.Lib 6.6 -> 6.8. DIR.Lib 6.8's DIR.Lib.Shaping satellite is rebuilt against
SharpAstro.Fonts.Shaping 1.5.551 (Fonts.Lib F6 zero-alloc bidi + F7 HarfBuzz-style
coverage-digest lookup skipping, ~3-4x faster shaping). Renderer core is unchanged; apps
that plug DIR.Lib.Shaping's ShapingTextShaper into renderer.TextShaper get the speedup.

## 6.15

Shaped-text GID-direct atlas fetch. DrawText/MeasureText now honor ShapedGlyph.Glyph (the
substituted glyph id from an ITextShaper -- GSUB ligatures, Arabic joined forms, ...),
fetching the SDF/bitmap atlas by glyph id instead of the source codepoint. Adds
VkFontAtlas/VkSdfFontAtlas.GetGlyphByGid + VkRenderer.PreWarmSdfGlyphByGid. Opt-in via
renderer.TextShaper (e.g. DIR.Lib.Shaping's ShapingTextShaper); the default AdvanceShaper
per-rune path is byte-identical to before. Bumps DIR.Lib pin 6.5 -> 6.6.

## 6.9

Inspector describe_layout MCP tool -- serializes the FULL arranged DIR.Lib.Layout tree (depth +
kind + rect + content/bg/hit chrome), not just the clickable subset describe_ui shows. Needs
DIR.Lib's ArrangedNode.Depth + PixelWidgetBase.GetCapturedLayout + LayoutInspection (DIR.Lib 6.0.x).

## 6.8

Inspector render-thread watchdog -- render_liveness MCP tool + ProbeRenderAsync (ALIVE/BLOCKED/DEAD
via a short-budget ping that round-trips ON the render thread; detects a wedged render loop).

## 6.7

Lockstep rebuild against DIR.Lib 6.0 (layout namespace + Layout.Builder DSL).

## 6.5

Live-device thumbnail capture (VulkanContext.ThumbnailCapture + VkRenderer
BeginThumbnailCapture/EndThumbnailCapture/TryGetThumbnailCapture) — re-issues already-
tessellated geometry into an offscreen target at thumbnail scale, non-blocking readback.
Plus SDF atlas per-page LRU eviction (replaces EvictAll thrash) + bounded disk-load drain.

## 6.4

Rebuilt against DIR.Lib 5.0 (layout engine + PixelMenuWidget); removes VkMenuWidget
(superseded by DIR.Lib's surface-agnostic PixelMenuWidget).

## 6.0

Multi-window — one VulkanDevice shared across windows (SdlVulkanApp); VulkanContext split into
device-level (VulkanDevice) + per-window state; multi-window SdlEventLoop. Breaking: standalone
consumers move to SdlVulkanApp. Adds opt-in SDF glyph disk cache + window placement/morph API.

## 5.1

VkRenderer overrides DIR.Lib's PushClip/PopClip → Vulkan scissor (needs DIR.Lib >= 4.4).

## 5.0 (breaking)

Multi-page SDF glyph atlas. The atlas is now a list of fixed-size
page textures (default 2048²); a full page appends a new page instead of reallocating, so
glyph-atlas growth no longer does a vkDeviceWaitIdle + image realloc + re-upload (the visible
frame stall). Internal change to VkSdfFontAtlas + the SDF draw path; the public VkRenderer API
is source-compatible (the optional sdfInitialAtlasDim param now sizes a page).
