# Portrait Mode Scaling Fix - Final Solution

## Problem

On monitors with portrait orientation (90°/270° rotation via `wl_output_transform`), the hyprexpo-plus overview was rendering workspace thumbnails incorrectly with only partial content visible.

## Root Cause

For a portrait monitor (270° rotation):
- Physical panel: 1920×1080 (landscape)
- Logical workspace: 1080×1920 (portrait)
- `m_pixelSize` field contains **physical panel dimensions** (1920×1080), not logical dimensions
- `beginRender` uses these dimensions to set up the GL viewport and projection matrix

When capturing workspace thumbnails:
1. Framebuffer was allocated using `m_pixelSize` dimensions (1920×1080 landscape)
2. Monitor fields retained rotation state, causing incorrect viewport setup
3. Result: only partial workspace content rendered into the framebuffer

## Solution

Override **three** monitor fields during framebuffer capture to create consistent non-rotated state:

```cpp
if (isTransformRotated(savedTransform)) {
    // Swap monbox dimensions from physical (landscape) to logical (portrait)
    monbox = {{0, 0}, {monbox.h, monbox.h}};

    // Override monitor state: disable rotation and set all size fields
    pMonitor->m_transform       = WL_OUTPUT_TRANSFORM_NORMAL;
    pMonitor->m_pixelSize       = {monbox.w, monbox.h};  // Portrait dimensions
    pMonitor->m_transformedSize = {monbox.w, monbox.h};  // Portrait dimensions
}
```

After capture, restore original values:
```cpp
pMonitor->m_transform       = savedTransform;
pMonitor->m_pixelSize       = savedPixelSize;
pMonitor->m_transformedSize = savedTransformedSize;
```

## Why This Works

1. **Dimension swap**: Converts physical landscape dimensions (1920×1080) to logical portrait (1080×1920)
2. **m_transform = NORMAL**: Tells renderer to use identity rotation (no transform)
3. **m_pixelSize override**: Sets viewport dimensions to portrait
4. **m_transformedSize override**: Ensures projection matrix is computed for portrait orientation

All three overrides are necessary because `beginRender` reads multiple fields to set up the GL state. Overriding only some fields creates inconsistent state.

## Files Modified

- `overview.cpp` lines 447-464 (COverview constructor)
- `overview.cpp` lines 498-502 (restore in constructor)
- `overview.cpp` lines 778-789 (COverview::redrawID)
- `overview.cpp` lines 828-832 (restore in redrawID)

## Testing

Verified on:
- Monitor: 1920×1080 physical, rotated 270° to 1080×1920 logical
- Hyprland 0.53.1
- Scale: 1.0

Results:
- ✅ Full workspace content visible in thumbnails
- ✅ Correct portrait orientation
- ✅ Proper aspect ratio (no stretching/squeezing)
- ✅ No regressions on landscape monitors

## Related Issues

This fix also applies to upstream hyprexpo plugin, which has the same bug.
