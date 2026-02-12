# Portrait Mode Fix - Debug Log

## Monitor Setup
- Physical panel: 1920×1080 (landscape)
- Rotation: 270° (`WL_OUTPUT_TRANSFORM_270`)
- Logical size after rotation: 1080×1920 (portrait)
- `pMonitor->m_size` = {1080/scale, 1920/scale}
- `pMonitor->m_pixelSize` = {1080, 1920}
- `pMonitor->m_transformedSize` = unknown (not yet inspected at runtime)

## Key Code Paths

The plugin captures workspace content into framebuffers in two places:
1. **`COverview::COverview()`** (constructor) — captures all 9 workspaces on overview open
2. **`COverview::redrawID()`** — re-captures a single workspace during live updates

Both follow the same pattern:
```cpp
monbox = {{0, 0}, pMonitor->m_pixelSize};           // {0, 0, 1080, 1920}
image.fb.alloc(monbox.w, monbox.h, ...);             // 1080×1920 framebuffer
g_pHyprRenderer->beginRender(PMONITOR, ..., &image.fb);
g_pHyprRenderer->renderWorkspace(PMONITOR, ..., monbox);
g_pHyprRenderer->endRender();
```

The captured texture is displayed in **`COverview::fullRender()`**:
```cpp
CBox texbox{...tile position and size...};
g_pHyprOpenGL->renderTextureInternal(images[id].fb.getTexture(), texbox, ...);
```

## Original Bug
Thumbnails appear in landscape mode and rotated 90°. The desktop content is sideways and in the wrong aspect ratio.

---

## Attempt 1: `pushMonitorTransformEnabled(false)`

### What I did
Added `g_pHyprOpenGL->pushMonitorTransformEnabled(false)` after `beginRender()` and `popMonitorTransformEnabled()` before `endRender()` in both capture paths.

### Rationale
This API controls whether individual render operations apply the monitor's transform. By disabling it, workspace content should render in the logical coordinate space without the rotation being applied.

### Result
No visible change — thumbnails still appeared landscape and rotated.

### Why it didn't work
`pushMonitorTransformEnabled` likely only controls per-surface transform application during individual `renderTexture` calls, not the projection matrix setup. The rotation is baked into the projection matrix during `beginRender()`, which this flag doesn't affect. The workspace content is still rendered through the rotated projection regardless of this flag.

---

## Attempt 2: `CBox::rot` counter-rotation on display

### What I did
In `fullRender()`, set `texbox.rot = M_PI_2` (for 270° monitors) on the tile CBox before calling `renderTextureInternal`. No changes to the capture path.

### Rationale
If the framebuffer content has the monitor's rotation baked in, we can counter-rotate when displaying the texture. For a 270° monitor, rotating the display quad by +90° should undo the rotation.

### Result
User reported: "the thumbnails have rotated now however, they are in the landscape aspect ratio and not aligned with the zoomed out overview grid."

### Why it didn't work
`renderTextureInternal` does NOT respect the `CBox::rot` field. The thumbnails showed no rotation effect. The content orientation appeared to change vs the original bug only because I was comparing against a different baseline. Looking at the actual screenshot, the thumbnails were landscape rectangles — meaning `rot` was completely ignored and the thumbnails just displayed with whatever dimensions the texbox had.

**Key learning: `renderTextureInternal` ignores `CBox::rot`.**

---

## Attempt 3: Framebuffer dimension swap + `CBox::rot` counter-rotation

### What I did
1. In capture: swapped monbox dimensions for rotated monitors: `monbox = {{0, 0}, {monbox.h, monbox.w}}` (1920×1080 instead of 1080×1920)
2. In display: swapped texbox w/h, re-centered, and set `texbox.rot`

### Rationale
The framebuffer was 1080×1920 (portrait) but the renderer's rotated projection produced landscape-oriented content, causing aspect ratio distortion. By making the framebuffer 1920×1080 (matching the physical panel), the renderer's rotated projection should produce correctly-proportioned content. Then counter-rotate at display time.

### Result
Landscape thumbnails with correctly-oriented content, but not rotated into portrait position. Thumbnails appeared as landscape rectangles centered in each grid cell with large black gaps above and below.

### Why it didn't work
Confirmed that `CBox::rot` is ignored by `renderTextureInternal`. The dimension swap produced a landscape framebuffer with correct content proportions, but since we can't rotate at display time, the thumbnails remain landscape.

**Key learning: We cannot rotate textures at display time with `renderTextureInternal`.**

---

## Attempt 4: Temporary `m_transform` override (no monbox swap)

### What I did
Before the capture loop:
```cpp
const auto savedTransform = pMonitor->m_transform;
if (isTransformRotated(savedTransform))
    pMonitor->m_transform = WL_OUTPUT_TRANSFORM_NORMAL;
```
After the capture loop, restored: `pMonitor->m_transform = savedTransform;`

No monbox swap — kept monbox at `pMonitor->m_pixelSize` (1080×1920).

### Rationale
By setting `m_transform = NORMAL` before `beginRender()`, the renderer should set up a projection matrix without rotation. The framebuffer (1080×1920) matches the logical workspace size, so `renderWorkspace` should render the portrait content directly into the portrait buffer without any rotation.

### Result
User reported (overview_2.jpg): "The thumbnail aspect ratio looks better but it doesn't seem to cover the whole desktop." Thumbnails showed portrait-shaped content but only a portion (roughly the top half) of the workspace was visible.

### Why it didn't work (partial)
The aspect ratio improved — the override DID affect the projection matrix. But the viewport/content mapping was wrong. The framebuffer is 1080×1920 pixels, but `beginRender` may be setting up GL viewport dimensions based on other monitor properties that still reflect the rotated state. The monbox geometry passed to `renderWorkspace` may not correctly map to the full workspace extent when the transform is overridden.

Possible explanations:
- `beginRender` reads `m_pixelSize` or `m_transformedSize` for viewport setup, but these still reflect the rotated geometry
- `renderWorkspace` uses the monbox to determine what region of the workspace to render, and the coordinate mapping is off when the transform doesn't match the monitor's other dimension fields
- The GL viewport is set to the framebuffer dimensions (1080×1920) but the projection matrix (now without rotation) maps a different coordinate range than expected

---

## Attempt 5: Temporary `m_transform` override WITH monbox swap

### What I did
Same as attempt 4, but also swapped monbox dimensions:
```cpp
if (isTransformRotated(savedTransform)) {
    pMonitor->m_transform = WL_OUTPUT_TRANSFORM_NORMAL;
    monbox = {{0, 0}, {monbox.h, monbox.w}};  // 1920×1080
}
```
Framebuffer allocated at 1920×1080.

### Rationale
Since attempt 4 showed the content was clipped (not covering the full workspace), perhaps the renderer with `NORMAL` transform expects physical panel dimensions (1920×1080) for the viewport/monbox. The workspace content would then render into a 1920×1080 buffer at full extent.

### Result
User reported (overview_3.jpg): "still not right" — screenshot appears identical to attempt 4. Thumbnails still show only a portion of the desktop.

### Why it didn't work
The screenshot looks the same as attempt 4, suggesting the monbox swap may not have taken effect, or the issue is deeper. The framebuffer was now 1920×1080 (landscape) but the grid tiles are portrait-shaped. The texture would be stretched/squeezed to fit the portrait tiles. However, the screenshot shows portrait-shaped thumbnails that are clipped, not stretched — which suggests the monbox swap either didn't change the behavior or the framebuffer content is being sampled incorrectly.

More likely: even with `m_transform = NORMAL`, other monitor fields like `m_size`, `m_pixelSize`, and `m_transformedSize` still reflect the rotated state. The renderer may use these fields (not just `m_transform`) when setting up the projection and coordinate mapping. Overriding `m_transform` alone creates an inconsistent state where the transform says NORMAL but the dimensions say rotated.

---

## Summary of Learnings

1. **`pushMonitorTransformEnabled(false)`** does not affect the projection matrix — it only controls per-surface transform handling
2. **`CBox::rot`** is ignored by `renderTextureInternal` — we cannot rotate textures at display time through this mechanism
3. **Overriding `m_transform` alone** partially works (fixes aspect ratio) but creates inconsistent state with the monitor's dimension fields, leading to incorrect viewport/content mapping
4. **The projection matrix** is the core issue — it's set up during `beginRender()` and bakes in the rotation based on the monitor's properties

## Current State of the Code

### Modifications currently in `overview.cpp`:

1. **Helper function added** (line ~20, after includes):
```cpp
static bool isTransformRotated(wl_output_transform t) {
    return t == WL_OUTPUT_TRANSFORM_90 || t == WL_OUTPUT_TRANSFORM_270 ||
           t == WL_OUTPUT_TRANSFORM_FLIPPED_90 || t == WL_OUTPUT_TRANSFORM_FLIPPED_270;
}
```

2. **`COverview::COverview()`** — temporary m_transform override (attempt 4, NO monbox swap):
   - Before capture loop: saves `pMonitor->m_transform`, sets to `WL_OUTPUT_TRANSFORM_NORMAL`
   - After capture loop: restores original transform

3. **`COverview::redrawID()`** — same pattern as constructor

4. **`fullRender()`** — NO display-side changes (all rotation/swap code was removed)

### Backup
- Original working plugin: `~/.config/hypr/plugins/hyprexpo.so.bak`
- Currently deployed: `~/.config/hypr/plugins/hyprexpo.so` (attempt 4 code)

### Key Files
- **All changes are in**: `/home/mike/src/hyprexpo-plus/overview.cpp`
- **Header**: `/home/mike/src/hyprexpo-plus/overview.hpp` (unmodified)
- **Reference screenshots**: `~/Downloads/LocalSend/portrait_desktop.jpg`, `overview.jpg`, `overview_2.jpg`, `overview_3.jpg`
- **Build**: `make clean && make` produces `hyprexpo.so` in the source dir, copy to `~/.config/hypr/plugins/hyprexpo.so`
- **User must restart Hyprland to load new plugin** (hot-reload via hyprctl caused segfault earlier)

---

## Hyprland API Details Discovered

### Monitor Fields (`/usr/include/hyprland/src/helpers/Monitor.hpp`)
```
Line 100: Vector2D m_size             — logical layout size (post-transform, post-scale)
Line 101: Vector2D m_pixelSize        — m_size * m_scale (pixel dimensions)
Line 102: Vector2D m_transformedSize  — NOT YET INSPECTED AT RUNTIME
Line 125: wl_output_transform m_transform — the rotation enum
Line 108: float m_scale               — display scale factor
```

### `wl_output_transform` enum values:
```
WL_OUTPUT_TRANSFORM_NORMAL      = 0
WL_OUTPUT_TRANSFORM_90          = 1
WL_OUTPUT_TRANSFORM_180         = 2
WL_OUTPUT_TRANSFORM_270         = 3
WL_OUTPUT_TRANSFORM_FLIPPED     = 4
WL_OUTPUT_TRANSFORM_FLIPPED_90  = 5
WL_OUTPUT_TRANSFORM_FLIPPED_180 = 6
WL_OUTPUT_TRANSFORM_FLIPPED_270 = 7
```

### Rendering APIs
- `renderTextureInternal` — private/friend, skips monitor transform, **ignores CBox::rot**
- `renderTexture` — public version, may handle transforms differently (NOT YET TESTED)
- `pushMonitorTransformEnabled(bool)` / `popMonitorTransformEnabled()` — stack-based flag, does NOT affect projection matrix
- `beginRender(PMONITOR, damage, RENDER_MODE_FULL_FAKE, nullptr, &fb)` — sets up GL viewport + projection matrix, the projection includes the monitor's rotation
- `CTexture::m_transform` (eTransform) — exists on texture objects, NOT YET TESTED whether renderTextureInternal uses it
- `Math::wlTransformToHyprutils(wl_output_transform t)` — converts between wl and hyprutils transform enums
- `Math::invertTransform(wl_output_transform tr)` — returns the inverse transform

### `#define private public` hack
`overview.cpp` line 3 uses `#define private public` around Hyprland includes. This gives access to private members like `m_renderData`, `renderTextureInternal`, etc.

---

## What Hasn't Been Tried

### High Priority (most likely to work)

1. **Override multiple monitor fields consistently** — Set `m_transform`, `m_pixelSize`, and `m_transformedSize` together to create a fully consistent un-rotated state. This is the natural extension of attempt 4 which showed the approach partially works. Need to inspect `m_transformedSize` at runtime first to understand what it contains.

2. **Inspect actual runtime values** — Add temporary logging (e.g., `Log::logger->log(...)`) to dump `m_size`, `m_pixelSize`, `m_transformedSize`, and `m_transform` during capture. This will reveal exactly what the renderer is working with and what needs to change. Could also dump the monbox values.

3. **Read the Hyprland source for `beginRender`** — The Hyprland source is NOT installed locally, but it can be fetched from GitHub (`hyprwm/Hyprland`). Understanding exactly which fields `beginRender` reads when setting up the projection matrix would tell us precisely what to override. This is the most important research task.

### Medium Priority

4. **Two-pass rendering** — Capture normally (with rotation baked in) into a landscape framebuffer, then render that texture into a SECOND portrait framebuffer using a manual GL blit/render with correct UV mapping. Reliable but doubles memory and adds complexity.

5. **CTexture::m_transform** — After capturing to framebuffer, set `images[id].fb.getTexture()->m_transform` to the inverse of the monitor transform. If `renderTextureInternal` reads the texture's transform when setting up UV coordinates, this could correct the display without any capture-side changes.

6. **Use `renderTexture` (public) instead of `renderTextureInternal`** — The public version may apply monitor transforms and handle texture transforms differently. Could solve the display-side rotation problem.

### Lower Priority

7. **GL matrix manipulation** — Directly manipulate the GL projection/modelview matrix between `beginRender` and `renderWorkspace` using `g_pHyprOpenGL->saveMatrix()` / `setMatrixScaleTranslate()` / `restoreMatrix()`.

8. **Look at how screencopy/screenshot handles rotated monitors** — Hyprland's screencopy protocol must solve the same problem. Find that code for reference.
