# Fix Portrait Monitor Orientation in Overview

## Context

On monitors with a portrait orientation (90°/270° rotation via `wl_output_transform`), the hyprexpo-plus zoomed-out overview renders workspace thumbnails incorrectly — they appear in landscape mode and rotated 90°. The thumbnails should match the monitor's actual aspect ratio and orientation.

This is also a bug in the upstream hyprexpo plugin (no transform handling exists there either).

The root cause: when `renderWorkspace` captures workspace content into a framebuffer, the monitor's rotation transform is baked into the rendering pipeline's projection matrix. This produces content rotated to match the physical panel orientation (landscape) inside a framebuffer allocated at the logical size (portrait). When displayed as tiles, the content appears landscape and rotated.

## Key Monitor Fields (`/usr/include/hyprland/src/helpers/Monitor.hpp:100-125`)

- `m_size` — logical layout size, post-transform (portrait for 90° rotation)
- `m_pixelSize` — `m_size * m_scale` (portrait pixel dimensions)
- `m_transformedSize` — physical/native output dimensions (landscape for 90° rotation)
- `m_transform` — `wl_output_transform` enum value (line 125)

## Files to Modify

- **`overview.cpp`** — all changes are in this file

## Implementation

### 1. Add rotation detection helper (top of overview.cpp, near line 20)

Add a helper function:
```cpp
static bool isTransformRotated(wl_output_transform t) {
    return t == WL_OUTPUT_TRANSFORM_90 || t == WL_OUTPUT_TRANSFORM_270 ||
           t == WL_OUTPUT_TRANSFORM_FLIPPED_90 || t == WL_OUTPUT_TRANSFORM_FLIPPED_270;
}
```

### 2. Fix framebuffer capture in `COverview::COverview()` (lines 431-488)

After `beginRender` and before `renderWorkspace`, disable the monitor transform so workspace content renders in the logical (portrait) orientation rather than the physical panel orientation:

```cpp
g_pHyprRenderer->beginRender(PMONITOR, fakeDamage, RENDER_MODE_FULL_FAKE, nullptr, &image.fb);
g_pHyprOpenGL->pushMonitorTransformEnabled(false);  // ADD: prevent rotation in capture
g_pHyprOpenGL->clear(CHyprColor{0, 0, 0, 1.0});
// ... renderWorkspace call ...
g_pHyprOpenGL->m_renderData.blockScreenShader = true;
g_pHyprOpenGL->popMonitorTransformEnabled();  // ADD: restore
g_pHyprRenderer->endRender();
```

### 3. Fix framebuffer capture in `COverview::redrawID()` (lines 743-812)

Same pattern — disable monitor transform during the render-to-framebuffer:

```cpp
g_pHyprRenderer->beginRender(pMonitor.lock(), fakeDamage, RENDER_MODE_FULL_FAKE, nullptr, &image.fb);
g_pHyprOpenGL->pushMonitorTransformEnabled(false);  // ADD
g_pHyprOpenGL->clear(CHyprColor{0, 0, 0, 1.0});
// ... renderWorkspace call ...
g_pHyprOpenGL->m_renderData.blockScreenShader = true;
g_pHyprOpenGL->popMonitorTransformEnabled();  // ADD
g_pHyprRenderer->endRender();
```

### 4. If step 2-3 don't fully resolve the issue (fallback approach)

If `pushMonitorTransformEnabled(false)` doesn't prevent rotation in the capture pipeline, the alternative is to apply a counter-rotation when displaying the captured textures.

In `fullRender()` (line 958-977), for rotated monitors, set the `rot` field on the texbox:

```cpp
CBox texbox{...};
texbox.scale(pMonitor->m_scale).translate(pos->value());
texbox.round();

// Counter-rotate for portrait monitors
if (isTransformRotated(pMonitor->m_transform)) {
    if (pMonitor->m_transform == WL_OUTPUT_TRANSFORM_90 ||
        pMonitor->m_transform == WL_OUTPUT_TRANSFORM_FLIPPED_90)
        texbox.rot = -M_PI_2;
    else  // 270
        texbox.rot = M_PI_2;
}
```

This may also require swapping the texbox width/height for rotated content to display with the correct aspect ratio.

## Build & Test Strategy

Since the user is actively running this plugin, we must NOT disrupt the working environment:

1. **Build**: `make clean && make` — produces `hyprexpo.so`
2. **Test**: Copy the built `.so` to a temporary location, then hot-reload with `hyprctl plugin unload` / `hyprctl plugin load` pointing to the new path
3. **Verify on portrait monitor**: Toggle the overview with `hyprexpo:expo toggle` and confirm:
   - Workspace thumbnails have the correct portrait aspect ratio
   - Content orientation matches the actual desktop
   - Mouse hover/click targets are correct (hit detection matches visual tiles)
   - Labels and borders render correctly
   - Zoom in/out animations work correctly
4. **Verify on landscape monitor**: Confirm no regression — overview works identically to before
5. **Rollback**: If anything is broken, reload the original `.so` file

## Risk Notes

- `pushMonitorTransformEnabled` may not fully control the transform during `RENDER_MODE_FULL_FAKE` rendering — this needs testing
- The `CBox::rot` field is supported by `renderTextureInternal` but the exact behavior with rotated textures needs verification
- Animation calculations (lines 500-503, 861-862) use `pMonitor->m_size` and `tileSize` which should already be in the correct logical coordinate space — no changes expected there
- Mouse hit-detection (lines 522-536) uses `pMonitor->m_size` which is in logical coordinates — should work correctly without changes
