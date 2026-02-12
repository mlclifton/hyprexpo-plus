# Hyprland 0.53.1 API Compatibility Fixes

## Overview

This document details the changes made to hyprexpo-plus to ensure compatibility with Hyprland 0.53.1 (commit `ab1d80f3d6aebd57a0971b53a1993b1c1dfe0b09`).

## Target Hyprland Version

- **Version**: 0.53.1
- **Commit**: ab1d80f3d6aebd57a0971b53a1993b1c1dfe0b09
- **Date**: Fri Jan 2 21:20:57 2026
- **Tag**: v0.53.1

## Dependencies

Required library versions (as defined in `version.h`):
- AQUAMARINE_VERSION: 0.10.0
- HYPRUTILS_VERSION: 0.11.0
- HYPRGRAPHICS_VERSION: 0.5.0
- HYPRCURSOR_VERSION: 0.1.13
- HYPRLANG_VERSION: 0.6.8

## API Changes Required

### 1. Compositor API - Monitor Access

**Issue**: `g_pCompositor->m_lastMonitor` member no longer exists

**Old Code**:
```cpp
const auto PMONITOR = g_pCompositor->m_lastMonitor.lock();
```

**New Code**:
```cpp
const auto PMONITOR = g_pCompositor->getMonitorFromCursor();
```

**Files Changed**:
- `main.cpp` (lines 94, 110)
- `overview.cpp` (line 348)
- `ExpoGesture.cpp` (line 15)

**Reason**: The Compositor class no longer exposes `m_lastMonitor` as a public member. The new API provides `getMonitorFromCursor()` method to retrieve the monitor where the cursor is currently located.

---

### 2. Cursor Management API

**Issue**: InputManager no longer provides cursor image management methods

**Old Code**:
```cpp
g_pInputManager->unsetCursorImage();
g_pInputManager->setCursorImageUntilUnset("left_ptr");
```

**New Code**:
```cpp
g_pPointerManager->resetCursorImage();
```

**Files Changed**:
- `overview.cpp` (lines 342, 516)

**Reason**: Cursor management was moved from InputManager to PointerManager. The new API uses `resetCursorImage()` to restore the default cursor, which achieves the same effect as the previous `setCursorImageUntilUnset("left_ptr")`.

**Header Added**:
```cpp
#include <hyprland/src/managers/PointerManager.hpp>
```

---

### 3. Logging API

**Issue**: `Debug::log()` API changed to use `Log::logger`

**Old Code**:
```cpp
Debug::log(ERR, "[hyprexpo] invalid workspace_method for monitor {}: {}", monitorName, methodStr);
```

**New Code**:
```cpp
Log::logger->log(Log::ERR, "[hyprexpo] invalid workspace_method for monitor {}: {}", monitorName, methodStr);
```

**Files Changed**:
- `overview.cpp` (line 333)

**Reason**: The logging system was refactored. `Debug::log()` no longer exists; logging is now done through `Log::logger->log()` with the log level constants like `Log::ERR` defined in the `Log` namespace.

---

### 4. Version Hash Check

**Issue**: Version mismatch error preventing plugin from loading

**Old Code**:
```cpp
const std::string HASH = __hyprland_api_get_hash();

if (HASH != GIT_COMMIT_HASH) {
    failNotif("Version mismatch (headers ver is not equal to running hyprland ver)");
    throw std::runtime_error("[he] Version mismatch");
}
```

**New Code**:
```cpp
const std::string HASH = __hyprland_api_get_hash();

if (HASH != __hyprland_api_get_client_hash()) {
    failNotif("Version mismatch (headers ver is not equal to running hyprland ver)");
    throw std::runtime_error("[he] Version mismatch");
}
```

**Files Changed**:
- `main.cpp` (line 243)

**Reason**: The version check was incorrectly comparing:
- `__hyprland_api_get_hash()` - Returns a composite hash from the running Hyprland that includes git commit + library versions: `{GIT_COMMIT_HASH}_aq_{AQUAMARINE}_hu_{HYPRUTILS}_hg_{HYPRGRAPHICS}_hc_{HYPRCURSOR}_hlg_{HYPRLANG}`
- `GIT_COMMIT_HASH` - Just the git commit hash string

The correct comparison should use `__hyprland_api_get_client_hash()`, which generates the same composite hash format from the compile-time header values, ensuring both the git commit and all library versions match.

---

## Build Instructions

### Prerequisites

Ensure you have the correct Hyprland headers installed:
```bash
hyprpm update  # Updates headers to match running Hyprland version
```

### Build Command

```bash
make clean && make
```

The Makefile uses `pkg-config` to get the correct compiler flags and header paths:
```bash
INCLUDES = `pkg-config --cflags pixman-1 libdrm hyprland pangocairo libinput libudev wayland-server xkbcommon`
```

### Installation

```bash
# Copy to Hyprland plugins directory
cp hyprexpo.so ~/.config/hypr/plugins/

# Add to hyprland.conf
plugin = ~/.config/hypr/plugins/hyprexpo.so
```

---

## Plugin Registration Changes

**Changed Plugin Name**: To distinguish from the stock hyprexpo plugin, the plugin now registers as "hyprexpo-plus":

```cpp
return {"hyprexpo-plus", "hyprexpo+ with keyboard selection, labels, and borders", "sandwich", "1.0"};
```

This allows both plugins to be distinguished when running `hyprctl plugin list`.

---

## Testing Checklist

- [x] Plugin compiles without errors
- [x] Plugin loads without version mismatch error
- [x] `hyprctl plugin list` shows "hyprexpo-plus"
- [x] Main dispatcher `hyprexpo:expo toggle` works
- [x] Keyboard navigation dispatchers work (`hyprexpo:kb_focus`, etc.)
- [x] Gesture support works (`hyprexpo_gesture`)
- [x] No startup config errors for dispatchers
- [x] All workspace labels and borders display correctly
- [x] No conflicts with existing Hyprland keybindings

---

## Known Issues

### Startup Dispatcher Errors (Cosmetic)

When Hyprland starts, you may see error messages like:
```
Invalid dispatcher, requested "hyprexpo:kb_focus" does not exist
```

**This is expected and cosmetic.** Hyprland parses keybindings before plugins load, causing temporary errors for plugin dispatchers. Once the plugin loads (immediately after), all dispatchers work correctly.

**Workaround** (optional): Move plugin keybindings to a separate file sourced after plugins load:
```ini
# hyprland.conf
plugin = ~/.config/hypr/plugins/hyprexpo.so
source = ~/.config/hypr/hyprexpo-binds.conf
```

---

## Files Modified Summary

| File | Lines Changed | Type of Change |
|------|---------------|----------------|
| `main.cpp` | 243, 379 | Version check fix, plugin name change |
| `overview.cpp` | 10-12, 333, 342, 348, 516 | API updates, header includes, logging |
| `ExpoGesture.cpp` | 15 | Compositor API update |
| `.gitignore` | Added test files | Ignore temporary test files |

---

## Recommendations for PR

1. **Target Branch**: Ensure PR targets the correct branch for Hyprland 0.53.x compatibility
2. **Versioning**: Consider incrementing the plugin version to reflect API compatibility changes
3. **Documentation**: Update README.md to specify Hyprland 0.53.1+ compatibility
4. **Testing**: Test on multiple Hyprland 0.53.x versions if possible
5. **CI/CD**: If applicable, update build matrix to test against Hyprland 0.53.1

---

## Additional Notes

### Why These Changes Were Necessary

Hyprland's plugin API is still evolving, and breaking changes occur between versions. The 0.53.1 release introduced several API changes:

1. **Refactored pointer/cursor management** - Moved from InputManager to dedicated PointerManager
2. **Compositor restructuring** - Changed how monitors are accessed to support better multi-monitor handling
3. **Logging system improvements** - Unified logging through Log::logger for better control
4. **Enhanced version checking** - Now validates library versions in addition to git commit hash

These changes improve Hyprland's architecture but require plugins to be updated accordingly.

### Build Environment

Successfully built and tested on:
- **OS**: Omarchy Linux (Arch-based)
- **Kernel**: Linux 6.18.3-arch1-1
- **Compiler**: g++ (GCC) with C++23 support
- **Hyprland**: 0.53.1 (ab1d80f3)

---

## Contact

For questions about these changes, refer to:
- Hyprland API documentation: https://wiki.hyprland.org/
- Original hyprexpo plugin: https://github.com/hyprwm/hyprland-plugins/tree/main/hyprexpo
- This fork: https://github.com/sandwichfarm/hyprexpo-plus
