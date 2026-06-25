# SDL3 Reference for Nim

Based on `sdl3.nim` — bindings for SDL 3.1.8.  
All functions can be called via UFCS or directly; `proc foo*(x: T)` means you can write either `foo(x)` or `x.foo()`.

**Error handling:** most SDL3 procedures return `bool` (`true` on success). On `false`, the reason is available via `getError()`. Examples often use `discard` when the error is non-critical; in production code it is better to check the return value explicitly.

---

## Table of Contents

1. [Initialization and Shutdown](#1-initialization-and-shutdown)
2. [Windows and Displays](#2-windows-and-displays)
3. [Renderer and Textures](#3-renderer-and-textures)
4. [Events](#4-events)
5. [Keyboard and Text Input](#5-keyboard-and-text-input)
6. [Mouse and Cursor](#6-mouse-and-cursor)
7. [Surfaces](#7-surfaces)
8. [Pixel Formats and Colors](#8-pixel-formats-and-colors)
9. [Audio](#9-audio)
10. [Timers](#10-timers)
11. [Filesystem and I/O](#11-filesystem-and-io)
12. [Threads, Mutexes, Semaphores](#12-threads-mutexes-semaphores)
13. [Joysticks and Gamepads](#13-joysticks-and-gamepads)
14. [Sensors and Haptic Feedback](#14-sensors-and-haptic-feedback)
15. [GPU API](#15-gpu-api)
16. [Clipboard](#16-clipboard)
17. [Properties](#17-properties)
18. [Logging](#18-logging)
19. [Miscellaneous: CPU, System Info, Dialogs](#19-miscellaneous-cpu-system-info-dialogs)
20. [Complete Minimal Example](#20-complete-minimal-example)

---

## 1. Initialization and Shutdown

### Subsystem Flags

```nim
const INIT_AUDIO*:    uint32 = 0x00000010  # Audio (automatically enables EVENTS)
const INIT_VIDEO*:    uint32 = 0x00000020  # Video + EVENTS, must run on main thread
const INIT_JOYSTICK*: uint32 = 0x00000200  # Joystick + EVENTS
const INIT_HAPTIC*:   uint32 = 0x00001000  # Haptic/Rumble
const INIT_GAMEPAD*:  uint32 = 0x00002000  # Gamepad + JOYSTICK
const INIT_EVENTS*:   uint32 = 0x00004000  # Event queue only
const INIT_SENSOR*:   uint32 = 0x00008000  # Sensors + EVENTS
const INIT_CAMERA*:   uint32 = 0x00010000  # Camera + EVENTS
```

### Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `init(flags: InitFlags): bool` | `true` on success | Initializes the specified subsystems |
| `initSubSystem(flags): bool` | `true` on success | Adds subsystems after startup |
| `quitSubSystem(flags)` | — | Shuts down the specified subsystems |
| `wasInit(flags): InitFlags` | flag bitmask | Returns the set of already-initialized subsystems |
| `quit()` | — | Stops all SDL subsystems |
| `getVersion(): cint` | version number | Numeric SDL version (e.g. `3001008`) |
| `getRevision(): cstring` | string | Build revision string |

**`wasInit`** accepts 0 to get a bitmask of **all** active subsystems: `wasInit(0)` returns the bitmask. To check a specific subsystem, pass its flag: `(wasInit(INIT_VIDEO) and INIT_VIDEO) != 0`.

**`initSubSystem` / `quitSubSystem`** are useful for dynamically enabling/disabling audio without restarting all of SDL. Calls are reference-counted — each `initSubSystem(INIT_AUDIO)` requires a matching `quitSubSystem(INIT_AUDIO)`.

### Application Metadata

```nim
proc setAppMetadata*(appname, appversion, appidentifier: cstring): bool
proc setAppMetadataProperty*(name, value: cstring): bool
proc getAppMetadataProperty*(name: cstring): cstring
```

Standard property names:
- `PROP_APP_METADATA_NAME_STRING` = `"SDL.app.metadata.name"`
- `PROP_APP_METADATA_VERSION_STRING`
- `PROP_APP_METADATA_IDENTIFIER_STRING`
- `PROP_APP_METADATA_CREATOR_STRING`

### Main Thread

```nim
proc isMainThread*(): bool
proc runOnMainThread*(callback: MainThreadCallback, userdata: pointer, wait_complete: bool): bool
```

**`INIT_VIDEO` requires the main thread.** All window and renderer operations must be performed from the same thread where `init(INIT_VIDEO)` was called. If you need to create a window from another thread, use `runOnMainThread` with `wait_complete = true`.

**`isMainThread`** is useful for debugging: if it returns `false` and you are trying to create a window, expect problems on macOS and Windows.

### Example

```nim
import sdl3

if not init(INIT_VIDEO or INIT_AUDIO):
  echo "SDL init failed: ", getError()
  quit(1)

defer: quit()
```

---

## 2. Windows and Displays

### Types

```nim
type
  Window* = ptr WindowObj
  DisplayID* = uint32
  WindowID* = uint32
  WindowFlags* = uint64    # bitfield flags
  DisplayMode* {.bycopy.} = object
    displayID*: DisplayID
    format*: PixelFormat
    w*, h*: cint
    pixel_density*: cfloat
    refresh_rate*: cfloat
    refresh_rate_numerator*: cint
    refresh_rate_denominator*: cint
```

### Window Flags (WindowFlags)

In `sdl3.nim` the `WindowFlags` constants are exported under names such as `SDL_WINDOW_FULLSCREEN`, `SDL_WINDOW_RESIZABLE`, etc. (or `WINDOW_FULLSCREEN` without the prefix — depends on the binding version). Numeric values are provided below for reference; named constants are preferred in code.

| Constant | Description |
|----------|-------------|
| `0x0000000000000001` | FULLSCREEN |
| `0x0000000000000002` | OPENGL |
| `0x0000000000000004` | OCCLUDED |
| `0x0000000000000008` | HIDDEN |
| `0x0000000000000020` | BORDERLESS |
| `0x0000000000000040` | RESIZABLE |
| `0x0000000000000080` | MINIMIZED |
| `0x0000000000000100` | MAXIMIZED |
| `0x0000000000000200` | MOUSE_GRABBED |
| `0x0000000000000400` | INPUT_FOCUS |
| `0x0000000000000800` | MOUSE_FOCUS |
| `0x0000000000002000` | EXTERNAL |
| `0x0000000000004000` | MODAL |
| `0x0000000000008000` | HIGH_PIXEL_DENSITY |
| `0x0000000000010000` | MOUSE_CAPTURE |
| `0x0000000000040000` | ALWAYS_ON_TOP |
| `0x0000000000080000` | UTILITY |
| `0x0000000000100000` | TOOLTIP |
| `0x0000000000200000` | POPUP_MENU |
| `0x0000000000400000` | KEYBOARD_GRABBED |
| `0x0000000010000000` | VULKAN |
| `0x0000000020000000` | METAL |
| `0x0000000040000000` | TRANSPARENT |
| `0x0000000080000000` | NOT_FOCUSABLE |

### Creation and Destruction

```nim
proc createWindow*(title: cstring, w, h: cint, flags: WindowFlags): Window
proc createPopupWindow*(parent: Window, offset_x, offset_y, w, h: cint, flags: WindowFlags): Window
proc createWindowWithProperties*(props: PropertiesID): Window
proc createWindowAndRenderer*(title: cstring, width, height: cint,
                              window_flags: WindowFlags,
                              window: var Window, renderer: var Renderer): bool
proc destroyWindow*(window: Window)
```

**Example:**
```nim
let win = createWindow("My Window", 800, 600, 0)
if win == nil:
  echo getError()
defer: destroyWindow(win)
```

### Size and Position

```nim
proc setWindowSize*(window: Window, w, h: cint): bool
proc getWindowSize*(window: Window, w, h: var cint): bool
proc setWindowPosition*(window: Window, x, y: cint): bool
proc getWindowPosition*(window: Window, x, y: var cint): bool
proc getWindowSizeInPixels*(window: Window, w, h: var cint): bool  # DPI-aware
proc setWindowMinimumSize*(window: Window, min_w, min_h: cint): bool
proc setWindowMaximumSize*(window: Window, max_w, max_h: cint): bool
proc getWindowMinimumSize*(window: Window, w, h: var cint): bool
proc getWindowMaximumSize*(window: Window, w, h: var cint): bool
proc setWindowAspectRatio*(window: Window, min_aspect, max_aspect: cfloat): bool
proc getWindowAspectRatio*(window: Window, min_aspect, max_aspect: var cfloat): bool
```

**`getWindowSize` vs `getWindowSizeInPixels`:** on high-DPI displays (Retina, HiDPI) `getWindowSize` returns the logical size in "screen coordinates", while `getWindowSizeInPixels` returns the actual pixel count (may be 2× larger). Use `getWindowSizeInPixels` for pixel-coordinate calculations (e.g. drawing); use `getWindowSize` for UI layout.

**Example — reading window size:**
```nim
var w, h: cint
discard getWindowSize(win, w, h)
echo "Logical size: ", w, "×", h

var pw, ph: cint
discard getWindowSizeInPixels(win, pw, ph)
echo "Pixel size: ", pw, "×", ph
```

### Title, Icon, Mode

```nim
proc setWindowTitle*(window: Window, title: cstring): bool
proc getWindowTitle*(window: Window): cstring
proc setWindowIcon*(window: Window, icon: ptr Surface): bool
proc setWindowFullscreen*(window: Window, fullscreen: bool): bool
proc setWindowFullscreenMode*(window: Window, mode: ptr DisplayMode): bool
proc getWindowFullscreenMode*(window: Window): ptr DisplayMode
proc setWindowBordered*(window: Window, bordered: bool): bool
proc setWindowResizable*(window: Window, resizable: bool): bool
proc setWindowAlwaysOnTop*(window: Window, on_top: bool): bool
proc setWindowOpacity*(window: Window, opacity: cfloat): bool
proc getWindowOpacity*(window: Window): cfloat
```

### Window State

```nim
proc showWindow*(window: Window): bool
proc hideWindow*(window: Window): bool
proc raiseWindow*(window: Window): bool
proc maximizeWindow*(window: Window): bool
proc minimizeWindow*(window: Window): bool
proc restoreWindow*(window: Window): bool
proc syncWindow*(window: Window): bool       # waits for async changes to apply
proc getWindowFlags*(window: Window): WindowFlags
proc getWindowID*(window: Window): WindowID
proc getWindowFromID*(id: WindowID): Window
proc getWindowParent*(window: Window): Window
proc flashWindow*(window: Window, operation: FlashOperation): bool
```

**`syncWindow`** is needed after operations that are applied asynchronously (e.g. `setWindowFullscreen`, `maximizeWindow`). Without it, the next `getWindowSize` may return stale values. Call it after a group of changes, before reading the new state.

**`flashWindow`** draws the user's attention (e.g. taskbar icon blinking). `FlashOperation` values: `FLASH_BRIEFLY` — single flash, `FLASH_UNTIL_FOCUSED` — flash until focused, `FLASH_CANCEL` — stop flashing.

### Mouse Capture

```nim
proc setWindowKeyboardGrab*(window: Window, grabbed: bool): bool
proc setWindowMouseGrab*(window: Window, grabbed: bool): bool
proc getWindowKeyboardGrab*(window: Window): bool
proc getWindowMouseGrab*(window: Window): bool
proc getGrabbedWindow*(): Window
proc setWindowMouseRect*(window: Window, rect: ptr Rect): bool
proc getWindowMouseRect*(window: Window): ptr Rect
```

### Displays

```nim
proc getDisplays*(count: var cint): ptr UncheckedArray[DisplayID]
proc getPrimaryDisplay*(): DisplayID
proc getDisplayName*(displayID: DisplayID): cstring
proc getDisplayBounds*(displayID: DisplayID, rect: var Rect): bool
proc getDisplayUsableBounds*(displayID: DisplayID, rect: var Rect): bool
proc getDisplayContentScale*(displayID: DisplayID): cfloat
proc getDisplayForWindow*(window: Window): DisplayID
proc getWindowDisplayScale*(window: Window): cfloat
proc getWindowPixelDensity*(window: Window): cfloat
proc getFullscreenDisplayModes*(displayID: DisplayID, count: var cint): ptr UncheckedArray[ptr DisplayMode]
proc getDesktopDisplayMode*(displayID: DisplayID): ptr DisplayMode
proc getCurrentDisplayMode*(displayID: DisplayID): ptr DisplayMode
```

**Memory management:** functions like `getDisplays` and `getFullscreenDisplayModes` return a pointer to an SDL-allocated array. It must be freed via `sdlFree(cast[pointer](result))` after use. Example:
```nim
var count: cint
let displays = getDisplays(count)
defer: sdlFree(cast[pointer](displays))
for i in 0 ..< count:
  echo getDisplayName(displays[i])
```

**`getDisplayUsableBounds`** returns the rectangle excluding the taskbar and system panels (taskbar, dock). Use it when calculating the initial window position.

### Window Surface (without Renderer)

**Important:** `getWindowSurface` and `Renderer` are **incompatible** for the same window. The choice is made at creation time: either `createRenderer` (GPU-accelerated) or `getWindowSurface` (CPU rendering directly into the window pixels). They cannot be mixed.

```nim
proc getWindowSurface*(window: Window): ptr Surface
proc updateWindowSurface*(window: Window): bool
proc updateWindowSurfaceRects*(window: Window, rects: openArray[Rect]): bool
proc destroyWindowSurface*(window: Window): bool
proc setWindowSurfaceVSync*(window: Window, vsync: cint): bool
```

### OpenGL / EGL

```nim
proc gL_LoadLibrary*(path: cstring): bool
proc gL_GetProcAddress*(procname: cstring): ProcPointer
proc gL_CreateContext*(window: Window): GLContext
proc gL_MakeCurrent*(window: Window, context: GLContext): bool
proc gL_SwapWindow*(window: Window): bool
proc gL_SetSwapInterval*(interval: cint): bool
proc gL_GetSwapInterval*(interval: var cint): bool
proc gL_SetAttribute*(attr: GLAttr, value: cint): bool
proc gL_GetAttribute*(attr: GLAttr, value: var cint): bool
proc gL_DestroyContext*(context: GLContext): bool
```

**Example — creating an OpenGL window:**
```nim
discard gL_SetAttribute(GL_CONTEXT_MAJOR_VERSION, 3)
discard gL_SetAttribute(GL_CONTEXT_MINOR_VERSION, 3)
let
  win = createWindow("OpenGL", 800, 600, 0x00000002u64) # OPENGL flag
  ctx = gL_CreateContext(win)
discard gL_MakeCurrent(win, ctx)
```

**`gL_DestroyContext`** — the only GL context deletion function in SDL3; returns `bool`. `SDL_GL_DeleteContext` from SDL2 is absent from the bindings.

**`gL_SetSwapInterval(1)`** enables VSync (waits for the vertical blanking interval). `0` — no waiting, `-1` — adaptive VSync (if supported; if not, it returns an error, then fall back to `1`).

**Attributes must be set BEFORE `createWindow`** — after window creation `gL_SetAttribute` has no effect.

---

## 3. Renderer and Textures

The renderer is SDL3's hardware-accelerated 2D engine. It works on top of Metal, Direct3D, Vulkan, or OpenGL depending on the platform.

### Types

```nim
type
  Renderer* = ptr RendererObj
  Texture* = ptr TextureObj
  TextureAccess* = enum
    TEXTUREACCESS_STATIC,    # loaded once, GPU reads
    TEXTUREACCESS_STREAMING, # CPU updates every frame
    TEXTUREACCESS_TARGET     # can render into the texture
```

> **Choosing `TextureAccess`:**
> - `STATIC` — for sprites and backgrounds that don't change. Most efficient.
> - `STREAMING` — for video, procedural textures updated every frame. Use `lockTexture` / `unlockTexture` or `updateTexture`.
> - `TARGET` — for render-to-texture (RTT). Required for post-processing, shadows, minimaps.

```nim
  RendererLogicalPresentation* = enum
    LOGICAL_PRESENTATION_DISABLED,
    LOGICAL_PRESENTATION_STRETCH,
    LOGICAL_PRESENTATION_LETTERBOX,
    LOGICAL_PRESENTATION_OVERSCAN,
    LOGICAL_PRESENTATION_INTEGER_SCALE

  Vertex* {.bycopy.} = object
    position*: FPoint
    color*: FColor
    tex_coord*: FPoint
```

### Creating a Renderer

```nim
proc createRenderer*(window: Window, name: cstring): Renderer
  # name = nil → auto-select; "software" → software renderer
proc createRendererWithProperties*(props: PropertiesID): Renderer
proc createSoftwareRenderer*(surface: ptr Surface): Renderer
proc destroyRenderer*(renderer: Renderer)
proc getRenderer*(window: Window): Renderer
proc getRenderWindow*(renderer: Renderer): Window
proc getRendererName*(renderer: Renderer): cstring
proc getNumRenderDrivers*(): cint
proc getRenderDriver*(index: cint): cstring
```

**Available backends** can be enumerated via `getNumRenderDrivers` / `getRenderDriver`. Typical values on Linux: `"opengl"`, `"opengles2"`, `"software"`. On Windows, `"direct3d11"` and `"direct3d12"` are added. On macOS — `"metal"`. Passing `nil` automatically selects the best available backend.

**`createSoftwareRenderer`** creates a CPU-based renderer on top of an arbitrary `Surface`. Useful for off-screen rendering into memory without a GPU context.

**Example:**
```nim
let renderer = createRenderer(win, nil)
if renderer == nil:
  echo "Renderer error: ", getError()
defer: destroyRenderer(renderer)
```

### Color, Blend Mode, Scale

```nim
proc setRenderDrawColor*(renderer: Renderer, r, g, b, a: uint8): bool
proc setRenderDrawColorFloat*(renderer: Renderer, r, g, b, a: cfloat): bool
proc getRenderDrawColor*(renderer: Renderer, r, g, b, a: var uint8): bool
proc setRenderDrawBlendMode*(renderer: Renderer, blendMode: BlendMode): bool
proc getRenderDrawBlendMode*(renderer: Renderer, blendMode: var BlendMode): bool
proc setRenderScale*(renderer: Renderer, scaleX, scaleY: cfloat): bool
proc getRenderScale*(renderer: Renderer, scaleX, scaleY: var cfloat): bool
proc setRenderColorScale*(renderer: Renderer, scale: cfloat): bool
proc getRenderColorScale*(renderer: Renderer, scale: var cfloat): bool
```

### Viewport, Clip, Logical Size

```nim
proc setRenderViewport*(renderer: Renderer, rect: ptr Rect): bool
proc getRenderViewport*(renderer: Renderer, rect: var Rect): bool
proc renderViewportSet*(renderer: Renderer): bool
proc setRenderClipRect*(renderer: Renderer, rect: ptr Rect): bool
proc getRenderClipRect*(renderer: Renderer, rect: var Rect): bool
proc renderClipEnabled*(renderer: Renderer): bool
proc setRenderLogicalPresentation*(renderer: Renderer, w, h: cint,
                                   mode: RendererLogicalPresentation): bool
proc getRenderLogicalPresentation*(renderer: Renderer, w, h: var cint,
                                   mode: var RendererLogicalPresentation): bool
proc getRenderLogicalPresentationRect*(renderer: Renderer, rect: var FRect): bool
proc getRenderOutputSize*(renderer: Renderer, w, h: var cint): bool
proc getCurrentRenderOutputSize*(renderer: Renderer, w, h: var cint): bool
```

### Coordinate Transforms

```nim
proc renderCoordinatesFromWindow*(renderer: Renderer, window_x, window_y: cfloat,
                                  x, y: var cfloat): bool
proc renderCoordinatesToWindow*(renderer: Renderer, x, y: cfloat,
                                window_x, window_y: var cfloat): bool
proc convertEventToRenderCoordinates*(renderer: Renderer, event: var Event): bool
```

### Drawing Primitives

```nim
proc renderClear*(renderer: Renderer): bool          # fill with current color

proc renderPoint*(renderer: Renderer, x, y: cfloat): bool
proc renderPoints*(renderer: Renderer, points: openArray[FPoint]): bool

proc renderLine*(renderer: Renderer, x1, y1, x2, y2: cfloat): bool
proc renderLines*(renderer: Renderer, points: openArray[FPoint]): bool  # polyline

proc renderRect*(renderer: Renderer, rect: ptr FRect): bool  # outline
proc renderRect*(renderer: Renderer, rect: FRect): bool
proc renderRects*(renderer: Renderer, rects: openArray[FRect]): bool

proc renderFillRect*(renderer: Renderer, rect: ptr FRect): bool  # filled
proc renderFillRect*(renderer: Renderer, rect: FRect): bool
proc renderFillRects*(renderer: Renderer, rects: openArray[FRect]): bool
```

**`renderLines`** draws a polyline: connects points sequentially `p[0]→p[1]→p[2]→...`. To draw a closed contour, duplicate the first point at the end of the array.

**`renderRect`** (outline) draws only the border; **`renderFillRect`** fills the rectangle. Both accept `ptr FRect` or `FRect` by value (overloaded). `nil` instead of `ptr FRect` means the entire viewport.

**`FRect` vs `Rect`:** SDL3 separates integer (`Rect`, `Point`) and floating-point (`FRect`, `FPoint`) geometry types. The renderer uses `FRect`/`FPoint`; Surface pixel operations use `Rect`/`Point`.

**Example draw loop:**
```nim
discard setRenderDrawColor(renderer, 30, 30, 30, 255)
discard renderClear(renderer)
discard setRenderDrawColor(renderer, 255, 80, 0, 255)
discard renderFillRect(renderer, FRect(x: 100, y: 100, w: 200, h: 150))
discard renderPresent(renderer)
```

### Geometry (Arbitrary Triangles)

**`renderGeometry`** is the foundation for 2D sprites with rotation, custom shapes, and particle systems. Each `Vertex` contains a position, color (`FColor`, values 0.0–1.0), and UV texture coordinates. If `texture = nil`, colored triangles are drawn with no texture. The `indices` array enables vertex reuse (indexed drawing).

**Example — colored triangle:**
```nim
let verts = [
  Vertex(position: FPoint(x: 400, y: 100), color: FColor(r: 1, g: 0, b: 0, a: 1), tex_coord: FPoint(x: 0, y: 0)),
  Vertex(position: FPoint(x: 200, y: 500), color: FColor(r: 0, g: 1, b: 0, a: 1), tex_coord: FPoint(x: 0, y: 0)),
  Vertex(position: FPoint(x: 600, y: 500), color: FColor(r: 0, g: 0, b: 1, a: 1), tex_coord: FPoint(x: 0, y: 0)),
]
discard renderGeometry(renderer, nil, verts, [])
```

```nim
proc renderGeometry*(renderer: Renderer, texture: Texture,
                     vertices: ptr[Vertex], num_vertices: cint,
                     indices: ptr[cint], num_indices: cint): bool
proc renderGeometry*(renderer: Renderer, texture: Texture,
                     vertices: openArray[Vertex], indices: openArray[cint]): bool
proc renderGeometryRaw*(renderer: Renderer, texture: Texture,
                        xy: ptr cfloat, xy_stride: cint,
                        color: ptr FColor, color_stride: cint,
                        uv: ptr cfloat, uv_stride: cint,
                        num_vertices: cint, indices: pointer,
                        num_indices, size_indices: cint): bool
```

### Reading Pixels

```nim
proc renderReadPixels*(renderer: Renderer, rect: ptr Rect): ptr Surface
# returns a Surface with pixels — destroySurface must be called after use
```

### Presentation and VSync

```nim
proc renderPresent*(renderer: Renderer): bool
proc flushRenderer*(renderer: Renderer): bool
proc setRenderVSync*(renderer: Renderer, vsync: cint): bool
proc getRenderVSync*(renderer: Renderer, vsync: var cint): bool

const RENDERER_VSYNC_DISABLED* = 0
const RENDERER_VSYNC_ADAPTIVE* = -1
```

**`renderPresent`** displays the current frame on screen. With VSync enabled (`setRenderVSync(renderer, 1)`) the call blocks until the vertical blanking interval, capping FPS to the monitor refresh rate.

**`flushRenderer`** forcibly submits accumulated commands to the GPU without swapping the frame buffer. Needed before reading GPU state or when integrating with external APIs (OpenGL, Vulkan).

### Debug Text

```nim
proc renderDebugText*(renderer: Renderer, x, y: cfloat, str: cstring): bool
proc renderDebugTextFormat*(renderer: Renderer, x, y: cfloat, fmt: cstring): bool
const DEBUG_TEXT_FONT_CHARACTER_SIZE* = 8  # 8×8 pixels per character
```

### Render to Texture (RTT)

```nim
proc setRenderTarget*(renderer: Renderer, texture: Texture): bool
  # texture = nil → return to screen
proc getRenderTarget*(renderer: Renderer): Texture
```

**RTT example:**
```nim
let target = createTexture(renderer, PIXELFORMAT_RGBA8888,
                           TEXTUREACCESS_TARGET, 512, 512)
discard setRenderTarget(renderer, target)
# draw into target ...
discard setRenderTarget(renderer, nil)  # back to screen
discard renderTexture(renderer, target, nil, nil)
```

### Textures

```nim
proc createTexture*(renderer: Renderer, format: PixelFormat,
                    access: TextureAccess, w, h: cint): Texture
proc createTextureFromSurface*(renderer: Renderer, surface: ptr Surface): Texture
proc createTextureWithProperties*(renderer: Renderer, props: PropertiesID): Texture
proc destroyTexture*(texture: Texture)

proc getTextureSize*(texture: Texture, w, h: var cfloat): bool
proc setTextureColorMod*(texture: Texture, r, g, b: uint8): bool
proc setTextureColorModFloat*(texture: Texture, r, g, b: cfloat): bool
proc getTextureColorMod*(texture: Texture, r, g, b: uint8): bool  # ⚠ see below
proc setTextureAlphaMod*(texture: Texture, alpha: uint8): bool
proc setTextureAlphaModFloat*(texture: Texture, alpha: cfloat): bool
proc getTextureAlphaMod*(texture: Texture, alpha: var uint8): bool
proc setTextureBlendMode*(texture: Texture, blendMode: BlendMode): bool
proc getTextureBlendMode*(texture: Texture, blendMode: var BlendMode): bool
proc setTextureScaleMode*(texture: Texture, scaleMode: ScaleMode): bool
proc getTextureScaleMode*(texture: Texture, scaleMode: var ScaleMode): bool

proc updateTexture*(texture: Texture, rect: ptr[Rect],
                    pixels: ptr[uint8], pitch: cint): bool
proc lockTexture*(texture: Texture, rect: ptr Rect,
                  pixels: var pointer, pitch: var cint): bool
proc lockTextureToSurface*(texture: Texture, rect: ptr Rect,
                            surface: var ptr Surface): bool
proc unlockTexture*(texture: Texture)
```

**⚠ `getTextureColorMod`:** in the bindings the `r`, `g`, `b` parameters are declared **without** `var` — passed by value. This is a bindings bug: the function cannot return values through these parameters. Use `getTextureProperties` + reading numeric properties as a workaround until it is fixed.

**`updateTexture` vs `lockTexture`:** `updateTexture` copies data from an arbitrary buffer into the texture — simpler to use but involves an extra copy. `lockTexture` returns a direct `pixels` pointer to the texture's internal buffer — faster for frequent updates (video player, procedural generation). The `pitch` parameter = row width in **bytes**, not pixels. After `lockTexture` the texture is locked and will not be rendered until `unlockTexture` is called.

**Example — streaming texture:**
```nim
var
  pixels: pointer
  pitch: cint
discard lockTexture(tex, nil, pixels, pitch)
let buf = cast[ptr UncheckedArray[uint32]](pixels)
for y in 0 ..< 256:
  for x in 0 ..< 256:
    buf[y * (pitch div 4) + x] = 0xFF0000FF'u32  # red, RGBA8888
unlockTexture(tex)
```

### Rendering Textures

```nim
proc renderTexture*(renderer: Renderer, texture: Texture,
                    srcrect, dstrect: ptr FRect): bool
  # srcrect = nil → entire texture; dstrect = nil → entire screen

proc renderTextureRotated*(renderer: Renderer, texture: Texture,
                           srcrect, dstrect: ptr FRect, angle: cdouble,
                           center: ptr FPoint, flip: FlipMode): bool

proc renderTextureAffine*(renderer: Renderer, texture: Texture,
                          srcrect: ptr FRect,
                          origin, right, down: ptr FPoint): bool

proc renderTextureTiled*(renderer: Renderer, texture: Texture,
                         srcrect: ptr FRect, scale: cfloat,
                         dstrect: ptr FRect): bool

proc renderTexture9Grid*(renderer: Renderer, texture: Texture,
                         srcrect: ptr FRect,
                         left_width, right_width, top_height,
                         bottom_height, scale: cfloat,
                         dstrect: ptr FRect): bool
```

**`renderTextureRotated`:** angle is in **degrees** (not radians), clockwise. `center` — rotation point relative to `dstrect`; `nil` = texture center. `FlipMode`: `FLIP_NONE`, `FLIP_HORIZONTAL`, `FLIP_VERTICAL`.

**`renderTextureAffine`:** arbitrary affine transform without the constraint of a rotation axis. `origin` — top-left corner, `right` — top-right, `down` — bottom-left. Allows arbitrary translation, scale, skew, and rotation in a single call.

**`renderTexture9Grid`:** 9-segment scaling (9-patch/9-slice). The corners retain their original size (multiplied by `scale`), the edges stretch only along their axis, and the center stretches in both directions. The standard approach for buttons, panels, and dialogs.

**`renderTextureTiled`:** fills `dstrect` with repeating copies of `srcrect`. The `scale` parameter scales each copy. Convenient for ground, water, or background pattern textures.

### BlendMode

```nim
type BlendMode* = uint32

# Predefined values:
const BLENDMODE_NONE*  = BlendMode(0x00000000)
const BLENDMODE_BLEND* = BlendMode(0x00000001)  # alpha blending
const BLENDMODE_ADD*   = BlendMode(0x00000002)  # additive blending
const BLENDMODE_MOD*   = BlendMode(0x00000004)  # color modulation
const BLENDMODE_MUL*   = BlendMode(0x00000008)  # multiply

proc composeCustomBlendMode*(srcColorFactor, dstColorFactor: BlendFactor,
                             colorOperation: BlendOperation,
                             srcAlphaFactor, dstAlphaFactor: BlendFactor,
                             alphaOperation: BlendOperation): BlendMode
```

**When to use each:**
- `BLENDMODE_NONE` — no transparency, maximum speed.
- `BLENDMODE_BLEND` — standard alpha blending: `result = src_color * src_alpha + dst_color * (1 - src_alpha)`.
- `BLENDMODE_ADD` — additive blending: `result = src_color * src_alpha + dst_color`. Used for fire, glow, particles.
- `BLENDMODE_MOD` — multiplies the destination color by the source color. Used for shadows and tinting.
- `BLENDMODE_MUL` — like `BLENDMODE_MOD` but takes source alpha into account.
- `composeCustomBlendMode` — for non-standard effects (premultiplied alpha, screen mode, etc.).

---

## 4. Events

### Core Types

**`Event`** is a C union (in Nim: an object with variant fields). The common part: `event.type` and `event.timestamp`. Specific data is accessible via sub-fields by event type: `event.key` (keyboard), `event.button` / `event.motion` / `event.wheel` (mouse), `event.window` (window), `event.text` (text input), `event.drop` (drag & drop), etc. The size of `Event` is fixed (128 bytes) regardless of the event type.

```nim
type
  EventType* = enum
    EVENT_FIRST = 0,
    EVENT_QUIT = 0x100,           # user closed the window
    EVENT_TERMINATING,
    # ... application system events ...
    EVENT_WINDOW_SHOWN = 0x202,
    EVENT_WINDOW_HIDDEN,
    EVENT_WINDOW_EXPOSED,
    EVENT_WINDOW_MOVED,
    EVENT_WINDOW_RESIZED,
    EVENT_WINDOW_PIXEL_SIZE_CHANGED,
    EVENT_WINDOW_METAL_VIEW_RESIZED,
    EVENT_WINDOW_MINIMIZED,
    EVENT_WINDOW_MAXIMIZED,
    EVENT_WINDOW_RESTORED,
    EVENT_WINDOW_MOUSE_ENTER,
    EVENT_WINDOW_MOUSE_LEAVE,
    EVENT_WINDOW_FOCUS_GAINED,
    EVENT_WINDOW_FOCUS_LOST,
    EVENT_WINDOW_CLOSE_REQUESTED,
    EVENT_WINDOW_HIT_TEST,
    EVENT_WINDOW_ICCPROF_CHANGED,
    EVENT_WINDOW_DISPLAY_CHANGED,
    EVENT_WINDOW_DISPLAY_SCALE_CHANGED,
    EVENT_WINDOW_SAFE_AREA_CHANGED,
    EVENT_WINDOW_OCCLUDED,
    EVENT_WINDOW_ENTER_FULLSCREEN,
    EVENT_WINDOW_LEAVE_FULLSCREEN,
    EVENT_WINDOW_DESTROYED,
    EVENT_WINDOW_HDR_STATE_CHANGED,
    EVENT_KEY_DOWN = 0x300,
    EVENT_KEY_UP,
    EVENT_TEXT_EDITING,
    EVENT_TEXT_INPUT,
    EVENT_MOUSE_MOTION = 0x400,
    EVENT_MOUSE_BUTTON_DOWN,
    EVENT_MOUSE_BUTTON_UP,
    EVENT_MOUSE_WHEEL,
    EVENT_JOYSTICK_AXIS_MOTION = 0x600,
    EVENT_JOYSTICK_BUTTON_DOWN,
    EVENT_JOYSTICK_BUTTON_UP,
    EVENT_JOYSTICK_ADDED,
    EVENT_JOYSTICK_REMOVED,
    EVENT_GAMEPAD_AXIS_MOTION = 0x650,
    EVENT_GAMEPAD_BUTTON_DOWN,
    EVENT_GAMEPAD_BUTTON_UP,
    EVENT_GAMEPAD_ADDED,
    EVENT_GAMEPAD_REMOVED,
    EVENT_FINGER_DOWN = 0x700,
    EVENT_FINGER_UP,
    EVENT_FINGER_MOTION,
    EVENT_CLIPBOARD_UPDATE = 0x900,
    EVENT_DROP_FILE = 0x1000,
    EVENT_AUDIO_DEVICE_ADDED = 0x1100,
    EVENT_SENSOR_UPDATE = 0x1200,
    EVENT_USER = 0x8000,
    # ...

  KeyboardEvent* {.bycopy.} = object
    `type`*: EventType
    reserved*: uint32
    timestamp*: uint64
    windowID*: WindowID
    which*: KeyboardID
    scancode*: Scancode
    key*: Keycode
    mod*: Keymod
    raw*: uint16
    down*: bool
    repeat*: bool

  MouseButtonEvent* {.bycopy.} = object
    `type`*: EventType
    reserved*: uint32
    timestamp*: uint64
    windowID*: WindowID
    which*: MouseID
    button*: uint8       # 1=left, 2=middle, 3=right
    down*: bool
    clicks*: uint8       # 1=single, 2=double
    padding*: uint8
    x*, y*: cfloat
```

### Polling Events

```nim
proc pollEvent*(event: var Event): bool
  # returns true if there is an event; non-blocking

proc waitEvent*(event: var Event): bool
  # waits indefinitely

proc waitEventTimeout*(event: var Event, timeoutMS: int32): bool
  # waits at most timeoutMS milliseconds

proc pumpEvents*()
  # updates the queue without retrieving events

proc pushEvent*(event: var Event): bool
  # manually pushes an event into the queue

# EventAction:
#   AddEventAction   — add to queue
#   PeekEventAction  — peek without removing
#   GetEventAction   — remove from queue
proc peepEvents*(events: ptr[Event], numevents: cint, action: EventAction,
                 minType, maxType: uint32): cint
proc peepEvents*(events: openArray[Event], action: EventAction,
                 minType, maxType: uint32): cint
proc hasEvent*(kind: uint32): bool
proc hasEvents*(minType, maxType: uint32): bool
proc flushEvent*(kind: uint32)
proc flushEvents*(minType, maxType: uint32)
```

**`pollEvent` vs `waitEvent`:** game loops use `pollEvent` (non-blocking, allows frame rendering). `waitEvent` / `waitEventTimeout` are suited for editors and tools that don't need to do anything without events (saves CPU).

**`pumpEvents`** is called automatically inside `pollEvent` / `waitEvent`. Explicitly needed only if you read keyboard/mouse state without processing events (`getKeyboardState`, `getMouseState`).

**`pushEvent`** is used for inter-thread communication (from a worker thread to the main thread) and for custom events (`EVENT_USER`).

**`peepEvents`** with `action = PeekEventAction` lets you inspect the queue **without removing** events; with `GetEventAction` — extract only events of a certain type range. The first overload (with `ptr[Event]` and `numevents`) is a direct C API mapping; the second accepts `openArray[Event]` and determines `numevents` automatically.

### Event Filters

```nim
type EventFilter* = proc (userdata: pointer; event: ptr Event): bool {.cdecl.}

proc setEventFilter*(filter: EventFilter, userdata: pointer)
proc getEventFilter*(filter: var EventFilter, userdata: var pointer): bool
proc addEventWatch*(filter: EventFilter, userdata: pointer): bool
proc removeEventWatch*(filter: EventFilter, userdata: pointer)
proc filterEvents*(filter: EventFilter, userdata: pointer)
proc setEventEnabled*(kind: uint32, enabled: bool)
proc eventEnabled*(kind: uint32): bool
proc registerEvents*(numevents: cint): uint32
proc getWindowFromEvent*(event: ptr Event): Window
```

### Standard Game Loop

```nim
var
  running = true
  event: Event

while running:
  while pollEvent(event):
    case event.type
    of EVENT_QUIT:
      running = false
    of EVENT_KEY_DOWN:
      if event.key.scancode == SDL_SCANCODE_ESCAPE:
        running = false
    else: discard

  # update & render ...
  discard renderClear(renderer)
  discard renderPresent(renderer)
```

---

## 5. Keyboard and Text Input

### Keyboard State

```nim
proc hasKeyboard*(): bool
proc getKeyboards*(count: var cint): ptr UncheckedArray[KeyboardID]
proc getKeyboardNameForID*(instance_id: KeyboardID): cstring
proc getKeyboardFocus*(): Window

proc getKeyboardState*(numkeys: var cint): ptr UncheckedArray[bool]
  # returns a snapshot; indexed by Scancode
  # e.g.: state[SDL_SCANCODE_W.ord]

proc resetKeyboard*()
proc getModState*(): Keymod
proc setModState*(modstate: Keymod)
```

**`getKeyboardState`** returns a pointer to SDL's internal array — **do not copy it**. The snapshot is valid until the next `pumpEvents` / `pollEvent`. Indexing: `state[SCANCODE_W.ord]` (cast `Scancode` to `int`).

**`Scancode` vs `Keycode`:** `Scancode` — the physical key position on the keyboard (layout-independent). `Keycode` — the character the key generates in the current layout. Use `Scancode` for game actions (WASD); use `Keycode` or `EVENT_TEXT_INPUT` for text fields.

**Example — continuous movement:**
```nim
var numkeys: cint
let state = getKeyboardState(numkeys)
if state[SCANCODE_W.ord]: playerY -= speed
if state[SCANCODE_S.ord]: playerY += speed
if state[SCANCODE_A.ord]: playerX -= speed
if state[SCANCODE_D.ord]: playerX += speed
```

### Key Codes

```nim
proc getKeyFromScancode*(scancode: Scancode, modstate: Keymod,
                         key_event: bool): Keycode
proc getScancodeFromKey*(key: Keycode, modstate: var Keymod): Scancode
proc getScancodeName*(scancode: Scancode): cstring
proc getScancodeFromName*(name: cstring): Scancode
proc getKeyName*(key: Keycode): cstring
proc getKeyFromName*(name: cstring): Keycode
proc setScancodeName*(scancode: Scancode, name: cstring): bool
```

### Text Input (IME)

```nim
proc startTextInput*(window: Window): bool
proc startTextInputWithProperties*(window: Window, props: PropertiesID): bool
proc textInputActive*(window: Window): bool
proc stopTextInput*(window: Window): bool
proc clearComposition*(window: Window): bool
proc setTextInputArea*(window: Window, rect: ptr Rect, cursor: cint): bool
proc getTextInputArea*(window: Window, rect: var Rect, cursor: var cint): bool
proc hasScreenKeyboardSupport*(): bool
proc screenKeyboardShown*(window: Window): bool
```

**Example — receiving text:**
```nim
discard startTextInput(win)
# then in the event loop:
of EVENT_TEXT_INPUT:
  echo "Typed: ", event.text.text  # UTF-8 string
```

---

## 6. Mouse and Cursor

### Mouse State

```nim
proc hasMouse*(): bool
proc getMice*(count: var cint): ptr UncheckedArray[MouseID]
proc getMouseNameForID*(instance_id: MouseID): cstring
proc getMouseFocus*(): Window

proc getMouseState*(x, y: var cfloat): uint32
  # returns a button bitmask; x,y — position in the window

proc getGlobalMouseState*(x, y: var cfloat): uint32
proc getRelativeMouseState*(x, y: var cfloat): uint32
  # dx, dy since the last call
```

**Decoding the button bitmask:** the returned `uint32` is a set of bits, one per button. Button N mask = `(1'u32 shl (N - 1))`. Buttons: 1=left, 2=middle, 3=right, 4=X1, 5=X2.
```nim
let
  buttons = getMouseState(mx, my)
  leftDown  = (buttons and (1'u32 shl 0)) != 0
  rightDown = (buttons and (1'u32 shl 2)) != 0
```

**`getRelativeMouseState`** returns the movement delta since the previous call, not absolute coordinates. Used for first-person camera control. For correct behavior, enable relative mode: `setWindowRelativeMouseMode(win, true)` — the cursor is hidden and no longer hits the window edge.

### Mouse Control

```nim
proc warpMouseInWindow*(window: Window, x, y: cfloat)
proc warpMouseGlobal*(x, y: cfloat): bool
proc setWindowRelativeMouseMode*(window: Window, enabled: bool): bool
proc getWindowRelativeMouseMode*(window: Window): bool
proc captureMouse*(enabled: bool): bool  # capture outside the window
```

### Cursor

```nim
type SystemCursor* = enum
  SYSTEM_CURSOR_DEFAULT,
  SYSTEM_CURSOR_TEXT,
  SYSTEM_CURSOR_WAIT,
  SYSTEM_CURSOR_CROSSHAIR,
  SYSTEM_CURSOR_PROGRESS,
  SYSTEM_CURSOR_NWSE_RESIZE,
  SYSTEM_CURSOR_NESW_RESIZE,
  SYSTEM_CURSOR_EW_RESIZE,
  SYSTEM_CURSOR_NS_RESIZE,
  SYSTEM_CURSOR_MOVE,
  SYSTEM_CURSOR_NOT_ALLOWED,
  SYSTEM_CURSOR_POINTER,
  SYSTEM_CURSOR_NW_RESIZE, ...

proc createSystemCursor*(id: SystemCursor): Cursor
proc createColorCursor*(surface: ptr Surface, hot_x, hot_y: cint): Cursor
proc createCursor*(data, mask: ptr uint8, w, h: cint,
                   hot_x, hot_y: cint): Cursor   # B&W cursor
proc setCursor*(cursor: Cursor): bool
proc getCursor*(): Cursor
proc getDefaultCursor*(): Cursor
proc destroyCursor*(cursor: Cursor)
proc showCursor*(): bool
proc hideCursor*(): bool
proc cursorVisible*(): bool
```

**Example — changing the cursor:**
```nim
let cursor = createSystemCursor(SYSTEM_CURSOR_CROSSHAIR)
discard setCursor(cursor)
# ... later:
destroyCursor(cursor)
```

---

## 7. Surfaces

`Surface` — a pixel buffer in CPU memory. Used for loading images, CPU rendering, and creating textures.

### Structure

```nim
type
  Surface* {.bycopy.} = object
    flags*: SurfaceFlags
    format*: PixelFormat
    w*, h*: cint
    pitch*: cint                        # bytes per row
    pixels*: ptr UncheckedArray[uint8]  # pointer to pixels (available after Lock)
    refcount*: cint
    reserved*: pointer
```

**`pitch`** (row stride) — the number of **bytes** per pixel row, **not** the number of pixels. Due to memory alignment, `pitch` may be larger than `w * bytesPerPixel`. Always use `pitch` to advance to the next row:
```nim
let bpp = 4  # for RGBA8888
let pixel = cast[ptr uint32](addr surf.pixels[y * surf.pitch + x * bpp])
```

**`refcount`** is used internally by SDL for safe surface passing between functions. Do not modify it manually.

### Creation and Destruction

```nim
proc createSurface*(width, height: cint, format: PixelFormat): ptr Surface
proc createSurfaceFrom*(width, height: cint, format: PixelFormat,
                        pixels: pointer, pitch: cint): ptr Surface
proc destroySurface*(surface: ptr Surface)
proc duplicateSurface*(surface: ptr Surface): ptr Surface
proc scaleSurface*(surface: ptr Surface, width, height: cint,
                   scaleMode: ScaleMode): ptr Surface
```

### Loading and Saving BMP

```nim
proc loadBMP*(file: cstring): ptr Surface
proc loadBMP_IO*(src: IOStream, closeio: bool): ptr Surface
proc saveBMP*(surface: ptr Surface, file: cstring): bool
proc saveBMP_IO*(surface: ptr Surface, dst: IOStream, closeio: bool): bool
```

**Example — creating a texture from a file:**
```nim
let surf = loadBMP("image.bmp")
if surf == nil: echo getError()
let tex = createTextureFromSurface(renderer, surf)
destroySurface(surf)
```

### Pixel Access

```nim
proc lockSurface*(surface: ptr Surface): bool
proc unlockSurface*(surface: ptr Surface)
func mUSTLOCK*(s: ptr Surface): bool  # is Lock required before access?

proc readSurfacePixel*(surface: ptr Surface, x, y: cint,
                       r, g, b, a: var uint8): bool
proc writeSurfacePixel*(surface: ptr Surface, x, y: cint,
                        r, g, b, a: uint8): bool
proc readSurfacePixelFloat*(surface: ptr Surface, x, y: cint,
                             r, g, b, a: var cfloat): bool
proc writeSurfacePixelFloat*(surface: ptr Surface, x, y: cint,
                              r, g, b, a: cfloat): bool
```

**`mustLock`:** not all surfaces require locking before accessing `pixels`. Check via `mUSTLOCK(surf)` and lock only if needed. Hardware surfaces (SDL_Surface with RLE flag or in video memory) always require locking.

**`readSurfacePixel` / `writeSurfacePixel`** work without an explicit Lock and without knowing the format — SDL decodes the pixel itself. Slower than direct access via `pixels`, but convenient for infrequent operations.

### Surface Operations

```nim
proc blitSurface*(src: ptr Surface, srcrect: ptr Rect,
                  dst: ptr Surface, dstrect: ptr Rect): bool
proc blitSurfaceScaled*(src: ptr Surface, srcrect: ptr Rect,
                        dst: ptr Surface, dstrect: ptr Rect,
                        scaleMode: ScaleMode): bool
proc blitSurfaceTiled*(src, dst: ptr Surface, ...): bool
proc blitSurface9Grid*(src: ptr Surface, ...): bool  # 9-segment scaling

proc fillSurfaceRect*(dst: ptr Surface, rect: ptr Rect, color: uint32): bool
proc fillSurfaceRects*(dst: ptr Surface, rect: openArray[Rect], color: uint32): bool
proc clearSurface*(surface: ptr Surface, r, g, b, a: cfloat): bool

proc convertSurface*(surface: ptr Surface, format: PixelFormat): ptr Surface
proc convertSurfaceAndColorspace*(surface: ptr Surface, format: PixelFormat,
                                  palette: ptr Palette, colorspace: Colorspace,
                                  props: PropertiesID): ptr Surface

proc flipSurface*(surface: ptr Surface, flip: FlipMode): bool
```

### Surface Properties

```nim
proc setSurfaceColorMod*(surface: ptr Surface, r, g, b: uint8): bool
proc getSurfaceColorMod*(surface: ptr Surface, r, g, b: var uint8): bool
proc setSurfaceAlphaMod*(surface: ptr Surface, alpha: uint8): bool
proc getSurfaceAlphaMod*(surface: ptr Surface, alpha: var uint8): bool
proc setSurfaceBlendMode*(surface: ptr Surface, blendMode: BlendMode): bool
proc getSurfaceBlendMode*(surface: ptr Surface, blendMode: var BlendMode): bool
proc setSurfaceClipRect*(surface: ptr Surface, rect: ptr Rect): bool
proc getSurfaceClipRect*(surface: ptr Surface, rect: var Rect): bool
proc setSurfaceColorKey*(surface: ptr Surface, enabled: bool, key: uint32): bool
proc getSurfaceColorKey*(surface: ptr Surface, key: var uint32): bool
proc setSurfaceRLE*(surface: ptr Surface, enabled: bool): bool
proc surfaceHasRLE*(surface: ptr Surface): bool
```

---

## 8. Pixel Formats and Colors

### Types

```nim
type
  Color* {.bycopy.} = object
    r*, g*, b*, a*: uint8

  FColor* {.bycopy.} = object
    r*, g*, b*, a*: cfloat

  Palette* {.bycopy.} = object
    ncolors*: cint
    colors*: ptr UncheckedArray[Color]
    version*: uint32
    refcount*: cint

  PixelFormatDetails* {.bycopy.} = object
    format*: PixelFormat
    bits_per_pixel*: uint8
    bytes_per_pixel*: uint8
    Rmask*, Gmask*, Bmask*, Amask*: uint32
    Rbits*, Gbits*, Bbits*, Abits*: uint8
    Rshift*, Gshift*, Bshift*, Ashift*: uint8
```

### Common PixelFormat Constants

| Constant | Description |
|----------|-------------|
| `PIXELFORMAT_RGBA8888` | 32-bit RGBA |
| `PIXELFORMAT_ARGB8888` | 32-bit ARGB |
| `PIXELFORMAT_RGB24` | 24-bit RGB |
| `PIXELFORMAT_BGR24` | 24-bit BGR |
| `PIXELFORMAT_RGBA32` | alias, little-endian |
| `PIXELFORMAT_ARGB32` | alias, little-endian |
| `PIXELFORMAT_RGB565` | 16-bit |
| `PIXELFORMAT_INDEX8` | 8-bit, palette |
| `PIXELFORMAT_YV12`, `PIXELFORMAT_NV12`, ... | YUV formats |

### Format Operations

```nim
proc getPixelFormatName*(format: PixelFormat): cstring
proc getPixelFormatDetails*(format: PixelFormat): ptr PixelFormatDetails
proc getMasksForPixelFormat*(format: PixelFormat, bpp: var cint,
                              Rmask, Gmask, Bmask, Amask: var uint32): bool
proc getPixelFormatForMasks*(bpp: cint, Rmask, Gmask, Bmask, Amask: uint32): PixelFormat

proc mapRGB*(format: ptr PixelFormatDetails, palette: ptr Palette,
             r, g, b: uint8): uint32
proc mapRGBA*(format: ptr PixelFormatDetails, palette: ptr Palette,
              r, g, b, a: uint8): uint32
proc getRGB*(pixel: uint32, format: ptr PixelFormatDetails,
             palette: ptr Palette, r, g, b: var uint8)
proc getRGBA*(pixel: uint32, format: ptr PixelFormatDetails,
              palette: ptr Palette, r, g, b, a: var uint8)
```

### Palette

```nim
proc createPalette*(ncolors: cint): ptr Palette
proc setPaletteColors*(palette: ptr Palette, colors: ptr[Color],
                       firstcolor, ncolors: cint): bool
proc destroyPalette*(palette: ptr Palette)
```

### Pixel Conversion

```nim
proc convertPixels*(width, height: cint, src_format: PixelFormat,
                    src: pointer, src_pitch: cint,
                    dst_format: PixelFormat, dst: pointer, dst_pitch: cint): bool
proc premultiplyAlpha*(width, height: cint, src_format: PixelFormat,
                       src: pointer, src_pitch: cint,
                       dst_format: PixelFormat, dst: pointer, dst_pitch: cint,
                       linear: bool): bool
proc premultiplySurfaceAlpha*(surface: ptr Surface, linear: bool): bool
```

---

## 9. Audio

### Types

```nim
type
  AudioFormat* = enum
    AUDIO_UNKNOWN = 0x0000,
    AUDIO_U8      = 0x0008,
    AUDIO_S8      = 0x8008,
    AUDIO_S16LE   = 0x8010,
    AUDIO_S16BE   = 0x9010,
    AUDIO_S32LE   = 0x8020,
    AUDIO_S32BE   = 0x9020,
    AUDIO_F32LE   = 0x8120,
    AUDIO_F32BE   = 0x9120

  AudioDeviceID* = uint32
  AudioSpec* {.bycopy.} = object
    format*: AudioFormat
    channels*: cint
    freq*: cint              # Hz, e.g. 44100 or 48000
```

> **Typical `AudioSpec` values:**
> - `freq`: 44100 (CD quality) or 48000 (professional audio/HDMI).
> - `channels`: 1 (mono), 2 (stereo), 6 (5.1).
> - `format`: `AUDIO_F32LE` — most universal for DSP; `AUDIO_S16LE` — compatible with most WAV files.
>
> `AudioStream` automatically converts between formats when calling `putAudioStreamData` / `getAudioStreamData`, so src_spec and dst_spec may differ.

```nim
  AudioStream* = ptr object
  AudioStreamCallback* = proc (userdata: pointer; stream: AudioStream;
                               additional_amount, total_amount: cint) {.cdecl.}
  AudioPostmixCallback* = proc (userdata: pointer; spec: ptr AudioSpec;
                                buffer: ptr cfloat; buflen: cint) {.cdecl.}
```

### Playback Devices

```nim
proc getNumAudioDrivers*(): cint
proc getAudioDriver*(index: cint): cstring
proc getCurrentAudioDriver*(): cstring
proc getAudioPlaybackDevices*(count: var cint): var UncheckedArray[AudioDeviceID]
proc getAudioRecordingDevices*(count: var cint): var UncheckedArray[AudioDeviceID]
proc getAudioDeviceName*(devid: AudioDeviceID): cstring
proc getAudioDeviceFormat*(devid: AudioDeviceID, spec: ptr AudioSpec,
                           sample_frames: ptr cint): bool
proc openAudioDevice*(devid: AudioDeviceID, spec: ptr AudioSpec): AudioDeviceID
proc closeAudioDevice*(devid: AudioDeviceID)
proc pauseAudioDevice*(dev: AudioDeviceID): bool
proc resumeAudioDevice*(dev: AudioDeviceID): bool
proc audioDevicePaused*(dev: AudioDeviceID): bool
proc getAudioDeviceGain*(devid: AudioDeviceID): cfloat
proc setAudioDeviceGain*(devid: AudioDeviceID, gain: cfloat): bool
```

**Default device:** pass `0xFFFFFFFF'u32` (`SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK`) as `devid` in `openAudioDevice` for automatic selection of the system playback device. For recording: `0xFFFFFFFE'u32` (`SDL_AUDIO_DEVICE_DEFAULT_RECORDING`).

**`spec = nil`** in `openAudioDevice` — SDL opens the device with its native format. After opening, the actual format can be queried via `getAudioDeviceFormat`.

### AudioStream — Primary Playback API

`AudioStream` — a flexible buffer with on-the-fly format conversion.

```nim
proc createAudioStream*(src_spec, dst_spec: ptr AudioSpec): AudioStream
proc destroyAudioStream*(stream: AudioStream)
proc getAudioStreamFormat*(stream: AudioStream,
                           src_spec, dst_spec: ptr AudioSpec): bool
proc setAudioStreamFormat*(stream: AudioStream,
                           src_spec, dst_spec: ptr AudioSpec): bool

proc putAudioStreamData*(stream: AudioStream, buf: pointer, len: cint): bool
proc getAudioStreamData*(stream: AudioStream, buf: pointer, len: cint): cint
proc getAudioStreamAvailable*(stream: AudioStream): cint  # bytes ready to read
proc getAudioStreamQueued*(stream: AudioStream): cint      # bytes queued
proc flushAudioStream*(stream: AudioStream): bool  # finalize the data stream
proc clearAudioStream*(stream: AudioStream): bool  # reset the buffer

proc getAudioStreamGain*(stream: AudioStream): cfloat
proc setAudioStreamGain*(stream: AudioStream, gain: cfloat): bool
proc getAudioStreamFrequencyRatio*(stream: AudioStream): cfloat
proc setAudioStreamFrequencyRatio*(stream: AudioStream, ratio: cfloat): bool
```

### Binding a Stream to a Device

```nim
proc bindAudioStream*(devid: AudioDeviceID, stream: AudioStream): bool
proc unbindAudioStream*(stream: AudioStream)
proc openAudioDeviceStream*(devid: AudioDeviceID, spec: ptr AudioSpec,
                            callback: AudioStreamCallback,
                            userdata: pointer): AudioStream
```

### Stream Callbacks

```nim
proc setAudioStreamGetCallback*(stream: AudioStream,
                                callback: AudioStreamCallback,
                                userdata: pointer): bool
proc setAudioStreamPutCallback*(stream: AudioStream,
                                callback: AudioStreamCallback,
                                userdata: pointer): bool
proc setAudioPostmixCallback*(devid: AudioDeviceID,
                              callback: AudioPostmixCallback,
                              userdata: pointer): bool
proc lockAudioStream*(stream: AudioStream): bool
proc unlockAudioStream*(stream: AudioStream): bool
```

### Loading WAV

```nim
proc loadWAV*(path: cstring, spec: ptr AudioSpec,
              audio_buf: var ptr[uint8], audio_len: var uint32): bool
proc loadWAV_IO*(src: IOStream, closeio: bool, spec: ptr AudioSpec,
                 audio_buf: var UncheckedArray[uint8],
                 audio_len: var uint32): bool
```

### Conversion and Mixing

```nim
proc convertAudioSamples*(src_spec: ptr AudioSpec, src_data: ptr[uint8],
                          src_len: cint, dst_spec: ptr AudioSpec,
                          dst_data: var ptr[uint8], dst_len: var cint): bool
proc mixAudio*(dst, src: ptr [uint8], format: AudioFormat,
               len: uint32, volume: cfloat): bool
proc getAudioFormatName*(format: AudioFormat): cstring
proc getSilenceValueForFormat*(format: AudioFormat): cint
```

**Full WAV playback example:**
```nim
var
  spec: AudioSpec
  audioBuf: ptr[uint8]
  audioLen: uint32

if not loadWAV("sound.wav", spec.addr, audioBuf, audioLen):
  echo getError()
  
let
  devid = openAudioDevice(0xFFFFFFFF'u32, nil) # SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK
  stream = openAudioDeviceStream(devid, spec.addr, nil, nil)
discard putAudioStreamData(stream, audioBuf, audioLen.cint)
discard resumeAudioStreamDevice(stream)
```

---

## 10. Timers

### Time

```nim
proc getTicks*(): uint64      # milliseconds since SDL init
proc getTicksNS*(): uint64    # nanoseconds
proc getPerformanceCounter*(): uint64   # high-resolution counter
proc getPerformanceFrequency*(): uint64 # ticks per second

# Delays
proc delay*(ms: uint32)          # blocks the thread for ms milliseconds
proc delayNS*(ns: uint64)        # delay in nanoseconds
proc delayPrecise*(ns: uint64)   # precise delay (spin-loop)
```

**Delta time (frame time)** — standard calculation:
```nim
var lastCounter = getPerformanceCounter()
let freq = getPerformanceFrequency().float

# at the start of each frame:
let
  now = getPerformanceCounter()
  dt = float(now - lastCounter) / freq  # seconds
lastCounter = now
```

**`delay` vs `delayPrecise`:** `delay(ms)` uses the OS sleep — accuracy ±1–2 ms on most OSes. `delayPrecise(ns)` refines the delay via spin-wait after sleep — more accurate but consumes CPU. Use `delayPrecise` only for delays < 2 ms.

**`getTicks()`** will overflow in ~584 million years (uint64). For measuring short intervals, `getPerformanceCounter()` is preferable.

### Inline Conversion Helpers

```nim
proc secondsToNs*(S: uint64): uint64
proc nsToSeconds*(NS: uint64): uint64
proc msToNs*(MS: uint64): uint64
proc nsToMs*(NS: uint64): uint64
proc usToNs*(US: uint64): uint64
proc nsToUs*(NS: uint64): uint64
```

### Callback Timers

```nim
type
  TimerID* = uint32
  TimerCallback* = proc (userdata: pointer; timerID: TimerID;
                         interval: uint32): uint32 {.cdecl.}
  NSTimerCallback* = proc (userdata: pointer; timerID: TimerID;
                           interval: uint64): uint64 {.cdecl.}

proc addTimer*(interval: uint32, callback: TimerCallback,
               userdata: pointer): TimerID
proc addTimerNS*(interval: uint64, callback: NSTimerCallback,
                 userdata: pointer): TimerID
proc removeTimer*(id: TimerID): bool
```

**Example — callback every 500 ms:**
```nim
proc onTimer(userdata: pointer, id: TimerID, interval: uint32): uint32 {.cdecl.} =
  echo "Tick!"
  return interval  # return the same interval → repeat

let tid = addTimer(500, onTimer, nil)
# ... later:
discard removeTimer(tid)
```

**Timer callback contract:** if the callback returns `0` — the timer stops; any non-zero value — the next interval in ms. This allows adaptive timers with a variable interval.

**Timers run in a separate thread.** Access to shared data requires synchronization (mutex or atomic operations). Use `pushEvent` to send results to the main thread.

### Date and Time

```nim
type
  DateTime* {.bycopy.} = object
    year*, month*, day*: cint
    hour*, minute*, second*: cint
    nanosecond*: cint
    day_of_week*: cint  # 0=Sunday
    utc_offset*: cint   # seconds

proc getCurrentTime*(ticks: var Time): bool
proc timeToDateTime*(ticks: Time, dt: var DateTime, localTime: bool): bool
proc dateTimeToTime*(dt: ptr DateTime, ticks: var Time): bool
proc getDaysInMonth*(year, month: cint): cint
proc getDayOfYear*(year, month, day: cint): cint
proc getDayOfWeek*(year, month, day: cint): cint
```

---

## 11. Filesystem and I/O

### IOStream — Universal Stream

```nim
type
  IOStream* = ptr object
  IOStatus* = enum
    IO_STATUS_READY, IO_STATUS_ERROR, IO_STATUS_EOF,
    IO_STATUS_NOT_READY, IO_STATUS_READONLY, IO_STATUS_WRITEONLY
  IOWhence* = enum
    IO_SEEK_SET, IO_SEEK_CUR, IO_SEEK_END

proc ioFromFile*(file, mode: cstring): IOStream   # "r", "w", "rb", "wb" ...
proc ioFromMem*(mem: pointer, size: csize_t): IOStream
proc ioFromConstMem*(mem: pointer, size: csize_t): IOStream
proc ioFromDynamicMem*(): IOStream   # auto-expanding buffer
proc openIO*(iface: ptr IOStreamInterface, userdata: pointer): IOStream
proc closeIO*(context: IOStream): bool

proc readIO*(context: IOStream, p: pointer, size: csize_t): csize_t
proc writeIO*(context: IOStream, p: pointer, size: csize_t): csize_t
proc seekIO*(context: IOStream, offset: int64, whence: IOWhence): int64
proc tellIO*(context: IOStream): int64
proc getIOSize*(context: IOStream): int64
proc flushIO*(context: IOStream): bool
proc getIOStatus*(context: IOStream): IOStatus
proc ioPrintf*(context: IOStream, fmt: cstring): csize_t {.varargs.}
```

### Reading/Writing Typed Data

```nim
proc readU8*(src: IOStream, value: var uint8): bool
proc readS16LE*(src: IOStream, value: var int16): bool
proc readU32LE*(src: IOStream, value: var uint32): bool
proc readU64LE*(src: IOStream, value: var uint64): bool
# ... similar for S8, U16BE, S32BE, S64LE, S64BE, etc.

proc writeU8*(dst: IOStream, value: uint8): bool
proc writeU32LE*(dst: IOStream, value: uint32): bool
# ... similar
```

### Files and Directories

```nim
proc loadFile*(file: cstring, datasize: var csize_t): pointer
proc saveFile*(file: cstring, data: pointer, datasize: csize_t): bool
proc loadFileIO*(src: IOStream, datasize: var csize_t, closeio: bool): pointer
proc saveFileIO*(src: IOStream, data: pointer, datasize: csize_t, closeio: bool): bool

proc getBasePath*(): cstring  # path to the executable
proc getPrefPath*(org, app: cstring): cstring  # path for saves/config
proc getUserFolder*(folder: Folder): cstring
proc getCurrentDirectory*(): cstring

proc createDirectory*(path: cstring): bool
proc removePath*(path: cstring): bool
proc renamePath*(oldpath, newpath: cstring): bool
proc copyFile*(oldpath, newpath: cstring): bool
proc getPathInfo*(path: cstring, info: var PathInfo): bool
proc enumerateDirectory*(path: cstring,
                         callback: EnumerateDirectoryCallback,
                         userdata: pointer): bool
proc globDirectory*(path, pattern: cstring, flags: GlobFlags,
                    count: var cint): ptr UncheckedArray[cstring]
```

> **`getBasePath()`** returns the directory of the executable (with trailing slash). Used for loading resources shipped with the game. On macOS inside a `.app` bundle, it points to `Contents/MacOS/`.

> **`getPrefPath(org, app)`** returns the platform-specific path for user data (saves, config):
> - Windows: `%APPDATA%\org\app\`
> - Linux: `~/.local/share/org/app/` (XDG)
> - macOS: `~/Library/Application Support/org/app/`
>
> The directory is created automatically. The returned string must be freed via `sdlFree` after use.

> **`globDirectory`** returns an array of paths matching the pattern (e.g. `"*.png"`). Free via `sdlFree(cast[pointer](result))`.

### Asynchronous I/O

```nim
type
  AsyncIO* = ptr object
  AsyncIOQueue* = ptr object
  AsyncIOOutcome* {.bycopy.} = object
    asyncio*: ptr AsyncIO
    `type`*: AsyncIOTaskType
    result*: AsyncIOResult
    buffer*: pointer
    offset*, bytesRequested*, bytesTransferred*: uint64
    userdata*: pointer

proc asyncIOFromFile*(file, mode: cstring): AsyncIO
proc getAsyncIOSize*(asyncio: AsyncIO): int64
proc readAsyncIO*(asyncio: AsyncIO, p: pointer, offset, size: uint64,
                  queue: AsyncIOQueue, userdata: pointer): bool
proc writeAsyncIO*(asyncio: AsyncIO, p: pointer, offset, size: uint64,
                   queue: AsyncIOQueue, userdata: pointer): bool
proc closeAsyncIO*(asyncio: AsyncIO, flush: bool,
                   queue: AsyncIOQueue, userdata: pointer): bool
proc createAsyncIOQueue*(): AsyncIOQueue
proc destroyAsyncIOQueue*(queue: AsyncIOQueue)
proc getAsyncIOResult*(queue: AsyncIOQueue, outcome: var AsyncIOOutcome): bool
proc waitAsyncIOResult*(queue: AsyncIOQueue, outcome: var AsyncIOOutcome,
                        timeoutMS: int32): bool
proc signalAsyncIOQueue*(queue: AsyncIOQueue)
proc loadFileAsync*(file: cstring, queue: AsyncIOQueue, userdata: pointer): bool
```

### Storage API (Platform Stores)

```nim
proc openTitleStorage*(override: cstring, props: PropertiesID): Storage
proc openUserStorage*(org, app: cstring, props: PropertiesID): Storage
proc openFileStorage*(path: cstring): Storage
proc closeStorage*(storage: Storage): bool
proc storageReady*(storage: Storage): bool
proc getStorageFileSize*(storage: Storage, path: cstring, length: var uint64): bool
proc readStorageFile*(storage: Storage, path: cstring,
                      destination: pointer, length: uint64): bool
proc writeStorageFile*(storage: Storage, path: cstring,
                       source: pointer, length: uint64): bool
proc getStorageSpaceRemaining*(storage: Storage): uint64
```

---

## 12. Threads, Mutexes, Semaphores

### Threads

```nim
type
  SdlThread* = ptr object
  ThreadID* = uint64
  ThreadPriority* = enum
    THREAD_PRIORITY_LOW, THREAD_PRIORITY_NORMAL,
    THREAD_PRIORITY_HIGH, THREAD_PRIORITY_TIME_CRITICAL
  SdlThreadFunction* = proc (data: pointer): cint {.cdecl.}

proc getThreadName*(thread: SdlThread): cstring
proc getCurrentThreadID*(): ThreadID
proc getThreadID*(thread: SdlThread): ThreadID
proc setCurrentThreadPriority*(priority: ThreadPriority): bool
proc waitThread*(thread: SdlThread, status: var cint)
proc getThreadState*(thread: SdlThread): ThreadState
proc detachThread*(thread: SdlThread)
proc isMainThread*(): bool
```

> **Creating a thread:** use `createThread` (a wrapper requiring a name for correct behavior on all platforms):
> ```nim
> proc workerFunc(data: pointer): cint {.cdecl.} =
>   # background work
>   return 0
>
> let thread = createThread(workerFunc, "WorkerThread", nil)
> var status: cint
> waitThread(thread, status)
> ```

> **`waitThread` vs `detachThread`:** `waitThread` blocks the calling thread until the child thread finishes and releases resources. `detachThread` lets the thread run independently — resources are released when it finishes, but `waitThread` cannot be called on it afterward.

> **SDL and Nim threads:** Nim has its own threading system (`std/threads`). When mixing with SDL threads, ensure GC invariants are preserved; all renderer and window operations remain on the main thread.

### Mutex

```nim
type SdlMutex* = ptr object

proc createMutex*(): SdlMutex
proc lockMutex*(mutex: SdlMutex)
proc tryLockMutex*(mutex: SdlMutex): bool
proc unlockMutex*(mutex: SdlMutex)
proc destroyMutex*(mutex: SdlMutex)
```

> **Safe mutex pattern in Nim:**
> ```nim
> let mtx = createMutex()
> defer: destroyMutex(mtx)
>
> lockMutex(mtx)
> try:
>   # critical section
> finally:
>   unlockMutex(mtx)
> ```
> `tryLockMutex` returns `true` if the lock was acquired immediately, `false` if the mutex is already held. Does not block.

### RW Locks

```nim
type SdlRWLock* = ptr object

proc createRWLock*(): SdlRWLock
proc lockRWLockForReading*(rwlock: SdlRWLock)
proc lockRWLockForWriting*(rwlock: SdlRWLock)
proc tryLockRWLockForReading*(rwlock: SdlRWLock): bool
proc tryLockRWLockForWriting*(rwlock: SdlRWLock): bool
proc unlockRWLock*(rwlock: SdlRWLock)
proc destroyRWLock*(rwlock: SdlRWLock)
```

### Semaphores

```nim
type SdlSemaphore* = ptr object

proc createSemaphore*(initial_value: uint32): SdlSemaphore
proc destroySemaphore*(sem: SdlSemaphore)
proc waitSemaphore*(sem: SdlSemaphore)           # blocks
proc tryWaitSemaphore*(sem: SdlSemaphore): bool  # non-blocking
proc waitSemaphoreTimeout*(sem: SdlSemaphore, timeoutMS: int32): bool
proc signalSemaphore*(sem: SdlSemaphore)
proc getSemaphoreValue*(sem: SdlSemaphore): uint32
```

### Condition Variables

```nim
type SdlCondition* = ptr object

proc createCondition*(): SdlCondition
proc destroyCondition*(cond: SdlCondition)
proc signalCondition*(cond: SdlCondition)
proc broadcastCondition*(cond: SdlCondition)
proc waitCondition*(cond: SdlCondition, mutex: SdlMutex)
proc waitConditionTimeout*(cond: SdlCondition, mutex: SdlMutex,
                           timeoutMS: int32): bool
```

### Spinlocks and Memory Barriers

```nim
type SpinLock* = cint

proc tryLockSpinlock*(lock: var SpinLock): bool
proc lockSpinlock*(lock: var SpinLock)
proc unlockSpinlock*(lock: var SpinLock)
proc memoryBarrierReleaseFunction*()
proc memoryBarrierAcquireFunction*()
```

---

## 13. Joysticks and Gamepads

### Joysticks (Low-Level API)

```nim
type
  Joystick* = ptr object
  JoystickID* = uint32
  JoystickType* = enum
    JOYSTICK_TYPE_UNKNOWN, JOYSTICK_TYPE_GAMEPAD, JOYSTICK_TYPE_WHEEL,
    JOYSTICK_TYPE_ARCADE_STICK, JOYSTICK_TYPE_FLIGHT_STICK,
    JOYSTICK_TYPE_DANCE_PAD, JOYSTICK_TYPE_GUITAR,
    JOYSTICK_TYPE_DRUM_KIT, JOYSTICK_TYPE_ARCADE_PAD, JOYSTICK_TYPE_THROTTLE

proc lockJoysticks*()
proc unlockJoysticks*()
proc hasJoystick*(): bool
proc getJoysticks*(count: var cint): ptr UncheckedArray[JoystickID]
proc getJoystickNameForID*(instance_id: JoystickID): cstring
proc openJoystick*(instance_id: JoystickID): Joystick
proc closeJoystick*(joystick: Joystick)
proc getJoystickFromID*(instance_id: JoystickID): Joystick

proc getJoystickName*(joystick: Joystick): cstring
proc getJoystickID*(joystick: Joystick): JoystickID
proc getNumJoystickAxes*(joystick: Joystick): cint
proc getNumJoystickButtons*(joystick: Joystick): cint
proc getNumJoystickHats*(joystick: Joystick): cint
proc getJoystickAxis*(joystick: Joystick, axis: cint): int16  # -32768..32767
proc getJoystickButton*(joystick: Joystick, button: cint): bool
proc getJoystickHat*(joystick: Joystick, hat: cint): uint8

proc rumbleJoystick*(joystick: Joystick, low_frequency_rumble,
                     high_frequency_rumble: uint16, duration_ms: uint32): bool
proc setJoystickLED*(joystick: Joystick, red, green, blue: uint8): bool
proc updateJoysticks*()
proc joystickEventsEnabled*(): bool
proc setJoystickEventsEnabled*(enabled: bool)
```

### Gamepads (High-Level API)

```nim
type
  Gamepad* = ptr object
  GamepadAxis* = enum
    GAMEPAD_AXIS_INVALID = -1,
    GAMEPAD_AXIS_LEFTX, GAMEPAD_AXIS_LEFTY,
    GAMEPAD_AXIS_RIGHTX, GAMEPAD_AXIS_RIGHTY,
    GAMEPAD_AXIS_LEFT_TRIGGER, GAMEPAD_AXIS_RIGHT_TRIGGER
  GamepadButton* = enum
    GAMEPAD_BUTTON_INVALID = -1,
    GAMEPAD_BUTTON_SOUTH, GAMEPAD_BUTTON_EAST,
    GAMEPAD_BUTTON_WEST, GAMEPAD_BUTTON_NORTH,
    GAMEPAD_BUTTON_BACK, GAMEPAD_BUTTON_GUIDE, GAMEPAD_BUTTON_START,
    GAMEPAD_BUTTON_LEFT_STICK, GAMEPAD_BUTTON_RIGHT_STICK,
    GAMEPAD_BUTTON_LEFT_SHOULDER, GAMEPAD_BUTTON_RIGHT_SHOULDER,
    GAMEPAD_BUTTON_DPAD_UP, GAMEPAD_BUTTON_DPAD_DOWN,
    GAMEPAD_BUTTON_DPAD_LEFT, GAMEPAD_BUTTON_DPAD_RIGHT,
    GAMEPAD_BUTTON_MISC1, ...

proc hasGamepad*(): bool
proc getGamepads*(count: var cint): ptr UncheckedArray[JoystickID]
proc isGamepad*(instance_id: JoystickID): bool
proc openGamepad*(instance_id: JoystickID): Gamepad
proc closeGamepad*(gamepad: Gamepad)

proc getGamepadName*(gamepad: Gamepad): cstring
proc getGamepadAxis*(gamepad: Gamepad, axis: GamepadAxis): int16
proc getGamepadButton*(gamepad: Gamepad, button: GamepadButton): bool

proc addGamepadMapping*(mapping: cstring): cint
proc addGamepadMappingsFromFile*(file: cstring): cint
proc getGamepadMapping*(gamepad: Gamepad): cstring
proc setGamepadMapping*(instance_id: JoystickID, mapping: cstring): bool

proc rumbleGamepad*(gamepad: Gamepad, low_frequency_rumble,
                    high_frequency_rumble: uint16, duration_ms: uint32): bool
proc setGamepadLED*(gamepad: Gamepad, red, green, blue: uint8): bool
```

**Gamepad example:**
```nim
let pads = getGamepads(count)
if count > 0:
  let
    pad = openGamepad(pads[0])
    x = getGamepadAxis(pad, GAMEPAD_AXIS_LEFTX)
    fire = getGamepadButton(pad, GAMEPAD_BUTTON_SOUTH)
  closeGamepad(pad)
```

---

## 14. Sensors and Haptic Feedback

### Sensors

```nim
type
  Sensor* = ptr object
  SensorID* = uint32
  SensorType* = enum
    SENSOR_INVALID = -1, SENSOR_UNKNOWN,
    SENSOR_ACCEL,   # acceleration, m/s²
    SENSOR_GYRO,    # angular velocity, rad/s
    SENSOR_ACCEL_L, SENSOR_GYRO_L,  # left Joy-Con
    SENSOR_ACCEL_R, SENSOR_GYRO_R   # right Joy-Con

proc getSensors*(count: var cint): ptr UncheckedArray[SensorID]
proc openSensor*(instance_id: SensorID): Sensor
proc closeSensor*(sensor: Sensor)
proc getSensorName*(sensor: Sensor): cstring
proc getSensorType*(sensor: Sensor): SensorType
proc getSensorData*(sensor: Sensor, data: openArray[cfloat]): bool
proc updateSensors*()
```

### Haptic Feedback

```nim
type
  Haptic* = ptr object
  HapticID* = uint32

proc getHaptics*(count: var cint): ptr UncheckedArray[HapticID]
proc openHaptic*(instance_id: HapticID): Haptic
proc openHapticFromMouse*(): Haptic
proc openHapticFromJoystick*(joystick: Joystick): Haptic
proc closeHaptic*(haptic: Haptic)

proc getMaxHapticEffects*(haptic: Haptic): cint
proc hapticRumbleSupported*(haptic: Haptic): bool
proc initHapticRumble*(haptic: Haptic): bool
proc playHapticRumble*(haptic: Haptic, strength: cfloat, length: uint32): bool
proc stopHapticRumble*(haptic: Haptic): bool

proc createHapticEffect*(haptic: Haptic, effect: ptr HapticEffect): cint
proc runHapticEffect*(haptic: Haptic, effect: cint, iterations: uint32): bool
proc stopHapticEffect*(haptic: Haptic, effect: cint): bool
proc destroyHapticEffect*(haptic: Haptic, effect: cint)
proc setHapticGain*(haptic: Haptic, gain: cint): bool
proc pauseHaptic*(haptic: Haptic): bool
proc resumeHaptic*(haptic: Haptic): bool
proc stopHapticEffects*(haptic: Haptic): bool
```

---

## 15. GPU API

SDL3 includes a low-level GPU API (analogous to WebGPU/Metal/Vulkan/D3D12) intended for advanced rendering.

> **When to use the GPU API instead of Renderer:**
> - You need shaders (GLSL/HLSL/MSL) and full pipeline control.
> - You require Compute (compute shaders, GPGPU).
> - You need instanced rendering, MRT (multiple render targets), stencil.
> - You need maximum performance with minimum overhead.
>
> For simple 2D graphics, the SDL Renderer is sufficient and much easier to use.

> **Supported shader formats (`GPUShaderFormat`):** depend on the platform.
> - Windows: `GPU_SHADERFORMAT_DXBC` (HLSL → DXC), `GPU_SHADERFORMAT_SPIRV`
> - macOS/iOS: `GPU_SHADERFORMAT_MSL`, `GPU_SHADERFORMAT_METALLIB`
> - Linux/Android: `GPU_SHADERFORMAT_SPIRV`
>
> For cross-platform code, use SPIR-V and transpile via `spirv-cross`.

### Core Objects

```nim
type
  GPUDevice*           = ptr object
  GPUCommandBuffer*    = ptr object
  GPURenderPass*       = ptr object
  GPUComputePass*      = ptr object
  GPUCopyPass*         = ptr object
  GPUBuffer*           = ptr object
  GPUTexture*          = ptr object
  GPUSampler*          = object
  GPUShader*           = ptr object
  GPUGraphicsPipeline* = ptr object
  GPUComputePipeline*  = ptr object
  GPUTransferBuffer*   = ptr object
  GPUFence*            = ptr object

  GPUShaderFormat* = uint32   # bitmask of supported shader formats
```

### Device

```nim
proc gPUSupportsShaderFormats*(format_flags: GPUShaderFormat, name: cstring): bool
proc createGPUDevice*(format_flags: GPUShaderFormat, debug_mode: bool,
                      name: cstring): GPUDevice
proc createGPUDeviceWithProperties*(props: PropertiesID): GPUDevice
proc destroyGPUDevice*(device: GPUDevice)
proc getGPUDeviceDriver*(device: GPUDevice): cstring
proc getGPUShaderFormats*(device: GPUDevice): GPUShaderFormat
proc getNumGPUDrivers*(): cint
proc getGPUDriver*(index: cint): cstring
proc waitForGPUIdle*(device: GPUDevice): bool
```

### Resources

```nim
proc createGPUShader*(device: GPUDevice, createinfo: ptr GPUShaderCreateInfo): GPUShader
proc createGPUTexture*(device: GPUDevice, createinfo: ptr GPUTextureCreateInfo): GPUTexture
proc createGPUBuffer*(device: GPUDevice, createinfo: ptr GPUBufferCreateInfo): GPUBuffer
proc createGPUSampler*(device: GPUDevice, createinfo: ptr GPUSamplerCreateInfo): GPUSampler
proc createGPUTransferBuffer*(device: GPUDevice,
                              createinfo: ptr GPUTransferBufferCreateInfo): GPUTransferBuffer
proc createGPUGraphicsPipeline*(device: GPUDevice,
                                createinfo: ptr GPUGraphicsPipelineCreateInfo): GPUGraphicsPipeline
proc createGPUComputePipeline*(device: GPUDevice,
                               createinfo: ptr GPUComputePipelineCreateInfo): GPUComputePipeline

# Release (not destroyXxx, but releaseXxx):
proc releaseGPUShader*(device: GPUDevice, shader: GPUShader)
proc releaseGPUTexture*(device: GPUDevice, texture: GPUTexture)
proc releaseGPUBuffer*(device: GPUDevice, buffer: GPUBuffer)
proc releaseGPUSampler*(device: GPUDevice, sampler: GPUSampler)
proc releaseGPUTransferBuffer*(device: GPUDevice, transfer_buffer: GPUTransferBuffer)
proc releaseGPUGraphicsPipeline*(device: GPUDevice, graphics_pipeline: GPUGraphicsPipeline)
proc releaseGPUComputePipeline*(device: GPUDevice, compute_pipeline: GPUComputePipeline)
proc releaseGPUFence*(device: GPUDevice, fence: GPUFence)
```

> **`release` vs `destroy` convention:** GPU resources use `releaseXxx` rather than `destroyXxx`. The release may not happen immediately — SDL waits until the GPU is done using the resource (analogous to Vulkan's `vkDestroyXxx` after `vkDeviceWaitIdle`). Never release a resource that may still be in use on the GPU.

### Command Buffer and Passes

> **GPU command lifecycle:**
> ```
> acquireGPUCommandBuffer
>   → beginGPURenderPass / beginGPUCopyPass / beginGPUComputePass
>       → bindXxx / drawXxx / dispatchXxx
>     → endGPURenderPass / endGPUCopyPass / endGPUComputePass
>   → submitGPUCommandBuffer  (or cancelGPUCommandBuffer)
> ```
> After `submitGPUCommandBuffer` the command buffer is invalid — do not access it. You cannot have two active passes at the same time.

```nim
proc acquireGPUCommandBuffer*(device: GPUDevice): GPUCommandBuffer
proc submitGPUCommandBuffer*(command_buffer: GPUCommandBuffer): bool
proc submitGPUCommandBufferAndAcquireFence*(command_buffer: GPUCommandBuffer): GPUFence
proc cancelGPUCommandBuffer*(command_buffer: GPUCommandBuffer): bool

proc beginGPURenderPass*(command_buffer: GPUCommandBuffer,
                         color_target_infos: ptr GPUColorTargetInfo,
                         num_color_targets: uint32,
                         depth_stencil_target_info: ptr GPUDepthStencilTargetInfo): GPURenderPass
proc bindGPUGraphicsPipeline*(render_pass: GPURenderPass,
                              graphics_pipeline: GPUGraphicsPipeline)
proc setGPUViewport*(render_pass: GPURenderPass, viewport: ptr GPUViewport)
proc setGPUScissor*(render_pass: GPURenderPass, scissor: ptr Rect)
proc setGPUBlendConstants*(render_pass: GPURenderPass, blend_constants: FColor)
proc bindGPUVertexBuffers*(render_pass: GPURenderPass, first_slot: uint32,
                           bindings: ptr GPUBufferBinding, num_bindings: uint32)
proc bindGPUIndexBuffer*(render_pass: GPURenderPass,
                         binding: ptr GPUBufferBinding,
                         index_element_size: GPUIndexElementSize)
proc drawGPUPrimitives*(render_pass: GPURenderPass,
                        num_vertices, num_instances,
                        first_vertex, first_instance: uint32)
proc drawGPUIndexedPrimitives*(render_pass: GPURenderPass,
                               num_indices, num_instances: uint32,
                               first_index: uint32, vertex_offset: int32,
                               first_instance: uint32)
proc endGPURenderPass*(render_pass: GPURenderPass)

proc beginGPUCopyPass*(command_buffer: GPUCommandBuffer): GPUCopyPass
proc uploadToGPUBuffer*(copy_pass: GPUCopyPass,
                        source: ptr GPUTransferBufferLocation,
                        destination: ptr GPUBufferRegion, cycle: bool)
proc uploadToGPUTexture*(copy_pass: GPUCopyPass,
                         source: ptr GPUTextureTransferInfo,
                         destination: ptr GPUTextureRegion, cycle: bool)
proc endGPUCopyPass*(copy_pass: GPUCopyPass)

proc mapGPUTransferBuffer*(device: GPUDevice, transfer_buffer: GPUTransferBuffer,
                           cycle: bool): pointer
proc unmapGPUTransferBuffer*(device: GPUDevice, transfer_buffer: GPUTransferBuffer)
```

> **`cycle`** in `mapGPUTransferBuffer`: if `true` and the buffer is still in use by the GPU, SDL creates a new buffer instead of waiting — a "carousel" (cycled) data upload scheme without blocking. If `false` — waits for release. For uploading data every frame (uniform buffers, dynamic vertices) use `cycle = true`.

> **Pattern for uploading vertex data to GPU:**
> ```nim
> # 1. Create a transfer buffer
> let tb = createGPUTransferBuffer(device, addr GPUTransferBufferCreateInfo(
>   usage: GPU_TRANSFERBUFFERUSAGE_UPLOAD,
>   size: uint32(sizeof(Vertex) * numVerts)
> ))
> # 2. Write data
> let mapped = cast[ptr UncheckedArray[Vertex]](mapGPUTransferBuffer(device, tb, false))
> for i in 0 ..< numVerts:
>   mapped[i] = vertices[i]
> unmapGPUTransferBuffer(device, tb)
> # 3. Copy to GPU buffer via copy pass
> let copyPass = beginGPUCopyPass(cmdBuf)
> uploadToGPUBuffer(copyPass, addr GPUTransferBufferLocation(transfer_buffer: tb, offset: 0),
>                  addr GPUBufferRegion(buffer: vertBuf, offset: 0, size: ...),
>                  cycle = false)
> endGPUCopyPass(copyPass)
> ```

### Swapchain and Synchronization

```nim
proc claimWindowForGPUDevice*(device: GPUDevice, window: Window): bool
proc releaseWindowFromGPUDevice*(device: GPUDevice, window: Window)
proc setGPUSwapchainParameters*(device: GPUDevice, window: Window,
                                swapchain_composition: GPUSwapchainComposition,
                                present_mode: GPUPresentMode): bool
proc acquireGPUSwapchainTexture*(command_buffer: GPUCommandBuffer,
                                 window: Window,
                                 swapchain_texture: GPUTexture,        # not var
                                 swapchain_texture_width: var uint32,
                                 swapchain_texture_height: var uint32): bool
proc waitAndAcquireGPUSwapchainTexture*(command_buffer: GPUCommandBuffer,
                                        window: Window,
                                        swapchain_texture: var GPUTexture,  # var
                                        swapchain_texture_width: var uint32,
                                        swapchain_texture_height: var uint32): bool
proc waitForGPUFences*(device: GPUDevice, wait_all: bool,
                       fences: openArray[GPUFence]): bool
proc queryGPUFence*(device: GPUDevice, fence: GPUFence): bool
proc setGPUAllowedFramesInFlight*(device: GPUDevice,
                                  allowed_frames_in_flight: uint32): bool
proc blitGPUTexture*(command_buffer: GPUCommandBuffer, info: ptr GPUBlitInfo)
proc generateMipmapsForGPUTexture*(command_buffer: GPUCommandBuffer, texture: GPUTexture)
```

### Texture Formats (Examples)

| Constant | Description |
|----------|-------------|
| `GPU_TEXTUREFORMAT_R8G8B8A8_UNORM` | 8-bit RGBA, normalized |
| `GPU_TEXTUREFORMAT_B8G8R8A8_UNORM` | 8-bit BGRA |
| `GPU_TEXTUREFORMAT_R16G16B16A16_FLOAT` | 16-bit float RGBA |
| `GPU_TEXTUREFORMAT_D16_UNORM` | 16-bit depth |
| `GPU_TEXTUREFORMAT_D32_FLOAT` | 32-bit float depth |
| `GPU_TEXTUREFORMAT_BC1_RGBA_UNORM` | DXT1 compression |
| `GPU_TEXTUREFORMAT_BC3_RGBA_UNORM` | DXT5 compression |

### Debug

```nim
proc setGPUBufferName*(device: GPUDevice, buffer: GPUBuffer, text: cstring)
proc setGPUTextureName*(device: GPUDevice, texture: GPUTexture, text: cstring)
proc insertGPUDebugLabel*(command_buffer: GPUCommandBuffer, text: cstring)
proc pushGPUDebugGroup*(command_buffer: GPUCommandBuffer, name: cstring)
proc popGPUDebugGroup*(command_buffer: GPUCommandBuffer)
```

---

## 16. Clipboard

```nim
proc setClipboardText*(text: cstring): bool
proc getClipboardText*(): cstring
proc hasClipboardText*(): bool

# X11 primary selection:
proc setPrimarySelectionText*(text: cstring): bool
proc getPrimarySelectionText*(): cstring
proc hasPrimarySelectionText*(): bool

# Extended: arbitrary MIME types:
type
  ClipboardDataCallback* = proc (userdata: pointer; mime_type: cstring;
                                 size: var csize_t): pointer {.cdecl.}
  ClipboardCleanupCallback* = proc (userdata: pointer) {.cdecl.}

proc setClipboardData*(callback: ClipboardDataCallback,
                       cleanup: ClipboardCleanupCallback,
                       userdata: pointer,
                       mime_types: openArray[cstring]): bool
proc clearClipboardData*(): bool
proc getClipboardData*(mime_type: cstring, size: var csize_t): pointer
proc hasClipboardData*(mime_type: cstring): bool
proc getClipboardMimeTypes*(num_mime_types: var csize_t): var UncheckedArray[cstring]
```

---

## 17. Properties

A universal key-value system for passing parameters to the SDL3 API and storing custom data attached to objects.

### Value Types

```nim
type PropertyType* = enum
  PropertyTypeInvalid, PropertyTypePointer,
  PropertyTypeString, PropertyTypeNumber,
  PropertyTypeFloat, PropertyTypeBoolean
```

### Operations

```nim
proc getGlobalProperties*(): PropertiesID
proc createProperties*(): PropertiesID
proc copyProperties*(src, dst: PropertiesID): bool
proc destroyProperties*(props: PropertiesID)
proc lockProperties*(props: PropertiesID): bool
proc unlockProperties*(props: PropertiesID)

proc setPointerProperty*(props: PropertiesID, name: cstring, value: pointer): bool
proc setStringProperty*(props: PropertiesID, name, value: cstring): bool
proc setNumberProperty*(props: PropertiesID, name: cstring, value: int64): bool
proc setFloatProperty*(props: PropertiesID, name: cstring, value: cfloat): bool
proc setBooleanProperty*(props: PropertiesID, name: cstring, value: bool): bool

proc getPointerProperty*(props: PropertiesID, name: cstring,
                         default_value: pointer): pointer
proc getStringProperty*(props: PropertiesID, name: cstring,
                        default_value: cstring): cstring
proc getNumberProperty*(props: PropertiesID, name: cstring,
                        default_value: int64): int64
proc getFloatProperty*(props: PropertiesID, name: cstring,
                       default_value: cfloat): cfloat
proc getBooleanProperty*(props: PropertiesID, name: cstring,
                         default_value: bool): bool

proc hasProperty*(props: PropertiesID, name: cstring): bool
proc getPropertyType*(props: PropertiesID, name: cstring): PropertyType
proc clearProperty*(props: PropertiesID, name: cstring): bool
proc enumerateProperties*(props: PropertiesID,
                          callback: EnumeratePropertiesCallback,
                          userdata: pointer): bool
proc setPointerPropertyWithCleanup*(props: PropertiesID, name: cstring,
                                    value: pointer,
                                    cleanup: CleanupPropertyCallback,
                                    userdata: pointer): bool
```

**Example — creating a Renderer via Properties:**
```nim
let props = createProperties()
discard setStringProperty(props, PROP_RENDERER_CREATE_NAME_STRING, "vulkan")
discard setNumberProperty(props, PROP_RENDERER_CREATE_PRESENT_VSYNC_NUMBER, 1)
let renderer = createRendererWithProperties(props)
destroyProperties(props)
```

---

## 18. Logging

```nim
type LogPriority* = enum
  LOG_PRIORITY_INVALID, LOG_PRIORITY_TRACE, LOG_PRIORITY_VERBOSE,
  LOG_PRIORITY_DEBUG, LOG_PRIORITY_INFO, LOG_PRIORITY_WARN,
  LOG_PRIORITY_ERROR, LOG_PRIORITY_CRITICAL, LOG_PRIORITY_SILENT

proc setLogPriorities*(priority: LogPriority)
proc setLogPriority*(category: cint, priority: LogPriority)
proc getLogPriority*(category: cint): LogPriority
proc resetLogPriorities*()
proc setLogPriorityPrefix*(priority: LogPriority, prefix: cstring): bool

proc log*(fmt: cstring) {.varargs.}              # LOG_CATEGORY_APPLICATION, INFO
proc logDebug*(category: cint, fmt: cstring) {.varargs.}
proc logInfo*(category: cint, fmt: cstring) {.varargs.}
proc logWarn*(category: cint, fmt: cstring) {.varargs.}
proc logError*(category: cint, fmt: cstring) {.varargs.}
proc logCritical*(category: cint, fmt: cstring) {.varargs.}
proc logTrace*(category: cint, fmt: cstring) {.varargs.}
proc logMessage*(category: cint, priority: LogPriority, fmt: cstring) {.varargs.}

type LogOutputFunction* = proc (userdata: pointer; category: cint;
                                priority: LogPriority; message: cstring) {.cdecl.}

proc getLogOutputFunction*(callback: var LogOutputFunction, userdata: var pointer)
proc setLogOutputFunction*(callback: LogOutputFunction, userdata: pointer)
proc getDefaultLogOutputFunction*(): LogOutputFunction
```

> **By default** SDL outputs the log to stderr (Linux/macOS) or via `OutputDebugString` (Windows). Default level: `DEBUG` and above for `LOG_CATEGORY_APPLICATION`, `WARN` and above for other categories.

> **`setLogOutputFunction`** allows redirecting output, e.g. to a file:
> ```nim
> proc myLogger(userdata: pointer, category: cint,
>               priority: LogPriority, message: cstring) {.cdecl.} =
>   let f = cast[File](userdata)
>   writeLine(f, $priority & ": " & $message)
>
> let logFile = open("game.log", fmWrite)
> setLogOutputFunction(myLogger, cast[pointer](logFile))
> ```

> **Formatting via `{.varargs.}`:** logging functions accept a printf-compatible format. In Nim you can pass `cstring`, `cint`, `cfloat`, etc. For Nim strings use `.cstring`: `logInfo(0, "Player: %s", playerName.cstring)`.

### Standard Categories

| Constant | Value |
|----------|-------|
| `LOG_CATEGORY_APPLICATION` | 0 |
| `LOG_CATEGORY_ERROR` | 1 |
| `LOG_CATEGORY_ASSERT` | 2 |
| `LOG_CATEGORY_SYSTEM` | 3 |
| `LOG_CATEGORY_AUDIO` | 4 |
| `LOG_CATEGORY_VIDEO` | 5 |
| `LOG_CATEGORY_RENDER` | 6 |
| `LOG_CATEGORY_INPUT` | 7 |
| `LOG_CATEGORY_TEST` | 8 |
| `LOG_CATEGORY_GPU` | 9 |
| `LOG_CATEGORY_CUSTOM` | 19+ |

---

## 19. Miscellaneous: CPU, System Info, Dialogs

### CPU and SIMD

```nim
proc getNumLogicalCPUCores*(): cint
proc getCPUCacheLineSize*(): cint
proc getSystemRAM*(): cint       # MB
proc getSIMDAlignment*(): csize_t

proc hasMMX*(): bool
proc hasSSE*(): bool
proc hasSSE2*(): bool
proc hasSSE3*(): bool
proc hasSSE41*(): bool
proc hasSSE42*(): bool
proc hasAVX*(): bool
proc hasAVX2*(): bool
proc hasAVX512F*(): bool
proc hasARMSIMD*(): bool
proc hasNEON*(): bool
proc hasAltiVec*(): bool
proc hasLSX*(): bool
proc hasLASX*(): bool
```

### System Information

```nim
proc getPlatform*(): cstring      # "Windows", "Mac OS X", "Linux", ...
proc isTablet*(): bool
proc isTV*(): bool
proc getSystemTheme*(): SystemTheme  # SYSTEM_THEME_LIGHT, _DARK, _UNKNOWN
proc getSandbox*(): Sandbox
```

### Errors

```nim
proc setError*(fmt: cstring): bool {.varargs.}
proc getError*(): cstring
proc clearError*(): bool
proc outOfMemory*(): bool
```

### File Dialogs

```nim
type
  DialogFileCallback* = proc (userdata: pointer; filelist: ptr UncheckedArray[cstring];
                              filter: cint) {.cdecl.}
  DialogFileFilter* {.bycopy.} = object
    name*: cstring
    pattern*: cstring  # "*.png;*.jpg"

proc showOpenFileDialog*(callback: DialogFileCallback, userdata: pointer,
                         window: Window, filters: openArray[DialogFileFilter],
                         default_location: cstring, allow_many: bool)
proc showSaveFileDialog*(callback: DialogFileCallback, userdata: pointer,
                         window: Window, filters: openArray[DialogFileFilter],
                         default_location: cstring)
proc showOpenFolderDialog*(callback: DialogFileCallback, userdata: pointer,
                           window: Window, default_location: cstring,
                           allow_many: bool)
```

> **Dialogs are asynchronous.** Functions return immediately; the result arrives in `callback` later, on the main thread. If the user cancelled the dialog, `filelist = nil`.

> **`filter`** in the callback — the index of the selected filter from the `filters` array (or `-1` if not applied / user cancelled).

**Example — opening a file:**
```nim
proc onFileChosen(userdata: pointer, filelist: ptr UncheckedArray[cstring],
                  filter: cint) {.cdecl.} =
  if filelist == nil:
    echo "Cancelled"
    return
  var i = 0
  while filelist[i] != nil:
    echo "Selected file: ", filelist[i]
    inc i

let filters = [
  DialogFileFilter(name: "PNG images", pattern: "*.png"),
  DialogFileFilter(name: "All files", pattern: "*"),
]
showOpenFileDialog(onFileChosen, nil, win, filters, nil, false)
```

### System Tray

```nim
type
  Tray* = ptr object
  TrayMenu* = ptr object
  TrayEntry* = ptr object

proc createTray*(icon: Surface, tooltip: cstring): Tray
proc setTrayIcon*(tray: Tray, icon: ptr Surface)
proc setTrayTooltip*(tray: Tray, tooltip: cstring)
proc createTrayMenu*(tray: Tray): TrayMenu
proc insertTrayEntryAt*(menu: TrayMenu, pos: cint, label: cstring,
                        flags: TrayEntryFlags): TrayEntry
proc setTrayEntryCallback*(entry: TrayEntry, callback: TrayCallback,
                           userdata: pointer)
proc destroyTray*(tray: Tray)
proc updateTrays*()
```

### Message Boxes

```nim
proc showSimpleMessageBox*(flags: MessageBoxFlags, title, message: cstring,
                           window: Window): bool
proc showMessageBox*(messageboxdata: ptr MessageBoxData,
                     buttonid: var cint): bool
```

### Memory

```nim
proc sdlMalloc*(size: csize_t): pointer
proc sdlCalloc*(nmemb, size: csize_t): pointer
proc sdlRealloc*(mem: pointer, size: csize_t): pointer
proc sdlFree*(mem: pointer)
proc sdlAlignedAlloc*(alignment, size: csize_t): pointer
proc sdlAlignedFree*(mem: pointer)
proc getNumAllocations*(): cint
proc setMemoryFunctions*(mallocProc: MallocProc, callocProc: CallocProc,
                         reallocProc: ReallocProc, freeProc: FreeProc): bool
```

### Hints

```nim
type HintPriority* = enum
  HINT_DEFAULT, HINT_NORMAL, HINT_OVERRIDE

proc setHint*(name, value: cstring): bool
proc setHintWithPriority*(name, value: cstring, priority: HintPriority): bool
proc getHint*(name: cstring): cstring
proc getHintBoolean*(name: cstring, default_value: bool): bool
proc resetHint*(name: cstring): bool
proc resetHints*()
proc addHintCallback*(name: cstring, callback: HintCallback, userdata: pointer): bool
proc removeHintCallback*(name: cstring, callback: HintCallback, userdata: pointer)
```

> **Hints** — SDL string configuration variables, set **before** calling the related function. Most commonly used:
>
> | Hint | Value | Description |
> |------|-------|-------------|
> | `"SDL_RENDER_VSYNC"` | `"1"` | VSync for Renderer (alternative to `setRenderVSync`) |
> | `"SDL_FRAMEBUFFER_ACCELERATION"` | `"1"` / `"0"` | Hardware framebuffer acceleration |
> | `"SDL_MOUSE_RELATIVE_MODE_WARP"` | `"1"` | Warping instead of raw input for relative mouse mode |
> | `"SDL_JOYSTICK_ALLOW_BACKGROUND_EVENTS"` | `"1"` | Receive joystick events in the background |
> | `"SDL_IME_SHOW_UI"` | `"1"` | Show IME selection UI |
> | `"SDL_WINDOWS_DPI_SCALING"` | `"1"` | DPI awareness on Windows |
>
> Set hints **before** `init()` or the corresponding `createXxx` call — they may have no effect if set later.

### Processes

```nim
proc createProcess*(args: ptr[cstring], pipe_stdio: bool): Process
proc createProcessWithProperties*(props: PropertiesID): Process
proc readProcess*(process: Process, datasize: csize_t, exitcode: var cint): pointer
proc getProcessInput*(process: Process): IOStream
proc getProcessOutput*(process: Process): IOStream
proc killProcess*(process: Process, force: bool): bool
proc waitProcess*(process: Process, blocking: bool, exitcode: var cint): bool
proc destroyProcess*(process: Process)
```

### Other Utilities

```nim
proc openURL*(url: cstring): bool      # open URL in the browser
proc getPreferredLocales*(count: var cint): ptr UncheckedArray[ptr Locale]
proc loadObject*(sofile: cstring): SharedObject  # dynamic library
proc loadFunction*(handle: SharedObject, name: cstring): ProcPointer
proc unloadObject*(handle: SharedObject)
```

> **`openURL`** opens a URL in the default browser (or associated application). On Wayland/Linux requires an XDG-compliant desktop environment. For `file://` — opens the file manager.

> **`loadObject` / `loadFunction`** — analogous to `dlopen` / `dlsym` (Linux) / `LoadLibrary` / `GetProcAddress` (Windows). Used for loading plugins and optional dependencies (e.g. checking for a library before using it).

**Example — dynamic function loading:**
```nim
let lib = loadObject("libmyPlugin.so")
if lib != nil:
  let fn = cast[proc() {.cdecl.}](loadFunction(lib, "pluginInit"))
  if fn != nil:
    fn()
  unloadObject(lib)
```

---

## 20. Complete Minimal Example

This example creates an 800×600 window, draws a rectangle, and handles window close and Escape key events.

```nim
import sdl3

proc main() =
  # Initialization
  if not init(INIT_VIDEO):
    echo "SDL_Init error: ", getError()
    return
  defer: sdl3.quit()

  # Window and renderer
  var
    window: Window
    renderer: Renderer
  if not createWindowAndRenderer("SDL3 Demo", 800, 600, 0, window, renderer):
    echo "Window/Renderer error: ", getError()
    return
  defer:
    destroyRenderer(renderer)
    destroyWindow(window)

  # Set metadata
  discard setAppMetadata("SDL3 Demo", "1.0", "com.example.demo")

  # FPS timer
  var
    lastTime = getTicks()
    frames = 0u64

  var
    running = true
    event: Event

  while running:
    # --- Event processing ---
    while pollEvent(event):
      case event.`type`
      of EVENT_QUIT:
        running = false
      of EVENT_KEY_DOWN:
        if event.key.scancode == SCANCODE_ESCAPE:
          running = false
        elif event.key.scancode == SCANCODE_F11:
          discard setWindowFullscreen(window,
            (getWindowFlags(window) and 0x00000001u64) == 0)
      of EVENT_WINDOW_RESIZED:
        discard setRenderLogicalPresentation(renderer,
          800, 600, LOGICAL_PRESENTATION_LETTERBOX)
      else: discard

    # --- Rendering ---
    discard setRenderDrawColor(renderer, 20, 20, 40, 255)
    discard renderClear(renderer)

    # Blue rectangle
    discard setRenderDrawColor(renderer, 0, 120, 215, 255)
    discard renderFillRect(renderer, FRect(x: 100, y: 100, w: 200, h: 150))

    # White outline
    discard setRenderDrawColor(renderer, 255, 255, 255, 200)
    discard renderRect(renderer, FRect(x: 100, y: 100, w: 200, h: 150))

    # Diagonal line
    discard setRenderDrawColor(renderer, 255, 80, 0, 255)
    discard renderLine(renderer, 0, 0, 800, 600)

    # Debug text (built-in 8×8 font)
    discard setRenderDrawColor(renderer, 255, 255, 0, 255)
    discard renderDebugTextFormat(renderer, 10, 10, "FPS: %llu", frames)

    discard renderPresent(renderer)

    inc frames
    # Cap FPS ~60
    let
      now = getTicks()
      elapsed = now - lastTime
    if elapsed < 16:
      delay((16 - elapsed).uint32)
    lastTime = getTicks()

main()
```

---

## Appendix: SDL C → Nim Mapping

| SDL C | Nim |
|-------|-----|
| `SDL_Init` | `init` |
| `SDL_Quit` | `quit` |
| `SDL_CreateWindow` | `createWindow` |
| `SDL_DestroyWindow` | `destroyWindow` |
| `SDL_CreateRenderer` | `createRenderer` |
| `SDL_DestroyRenderer` | `destroyRenderer` |
| `SDL_RenderClear` | `renderClear` |
| `SDL_RenderPresent` | `renderPresent` |
| `SDL_RenderFillRect` | `renderFillRect` |
| `SDL_RenderDrawRect` | `renderRect` |
| `SDL_RenderDrawLine` | `renderLine` |
| `SDL_SetRenderDrawColor` | `setRenderDrawColor` |
| `SDL_PollEvent` | `pollEvent` |
| `SDL_WaitEvent` | `waitEvent` |
| `SDL_GetError` | `getError` |
| `SDL_ClearError` | `clearError` |
| `SDL_GetTicks` | `getTicks` |
| `SDL_Delay` | `delay` |
| `SDL_LoadBMP` | `loadBMP` |
| `SDL_CreateTextureFromSurface` | `createTextureFromSurface` |
| `SDL_RenderTexture` | `renderTexture` |
| `SDL_RenderTextureRotated` | `renderTextureRotated` |
| `SDL_DestroyTexture` | `destroyTexture` |
| `SDL_DestroySurface` | `destroySurface` |
| `SDL_BlitSurface` | `blitSurface` |
| `SDL_ConvertSurface` | `convertSurface` |
| `SDL_GetKeyboardState` | `getKeyboardState` |
| `SDL_GetModState` | `getModState` |
| `SDL_GetMouseState` | `getMouseState` |
| `SDL_WarpMouseInWindow` | `warpMouseInWindow` |
| `SDL_OpenAudioDevice` | `openAudioDevice` |
| `SDL_CloseAudioDevice` | `closeAudioDevice` |
| `SDL_LoadWAV` | `loadWAV` |
| `SDL_CreateAudioStream` | `createAudioStream` |
| `SDL_PutAudioStreamData` | `putAudioStreamData` |
| `SDL_CreateMutex` | `createMutex` |
| `SDL_LockMutex` | `lockMutex` |
| `SDL_UnlockMutex` | `unlockMutex` |
| `SDL_CreateSemaphore` | `createSemaphore` |
| `SDL_WaitSemaphore` | `waitSemaphore` |
| `SDL_SignalSemaphore` | `signalSemaphore` |
| `SDL_CreateCondition` | `createCondition` |
| `SDL_WaitCondition` | `waitCondition` |
| `SDL_SignalCondition` | `signalCondition` |
| `SDL_GetBasePath` | `getBasePath` |
| `SDL_GetPrefPath` | `getPrefPath` |
| `SDL_IOFromFile` | `ioFromFile` |
| `SDL_CreateGPUDevice` | `createGPUDevice` |
| `SDL_AcquireGPUCommandBuffer` | `acquireGPUCommandBuffer` |
| `SDL_SubmitGPUCommandBuffer` | `submitGPUCommandBuffer` |
| `SDL_BeginGPURenderPass` | `beginGPURenderPass` |
| `SDL_EndGPURenderPass` | `endGPURenderPass` |
| `SDL_ShowSimpleMessageBox` | `showSimpleMessageBox` |
| `SDL_ShowOpenFileDialog` | `showOpenFileDialog` |
| `SDL_SetClipboardText` | `setClipboardText` |
| `SDL_GetClipboardText` | `getClipboardText` |
| `SDL_SetHint` | `setHint` |
| `SDL_GetHint` | `getHint` |
| `SDL_Log` | `log` |
| `SDL_LogError` | `logError` |
| `SDL_OpenURL` | `openURL` |

> **Naming rule:** the `SDL_` prefix is dropped, the first letter is lowercased. `SDL_GetError` → `getError`. Exceptions: OpenGL functions (`gL_*`), EGL (`eGL_*`), GPU (`gPU*`), HID (`hid_*`).

> **Key changes SDL2 → SDL3 relevant to Nim users:**
> - `SDL_RenderDrawRect` → `renderRect` (new name, reflecting the removal of the "Draw" infix throughout the API).
> - `SDL_RenderCopy` / `SDL_RenderCopyEx` → `renderTexture` / `renderTextureRotated` (arguments are now `FRect`, not `Rect`).
> - `SDL_OpenAudioDevice` now takes an `AudioDeviceID` instead of a device name string.
> - `SDL_LockSurface` / `SDL_UnlockSurface` → `lockSurface` / `unlockSurface` (return `bool`).
> - All renderer geometry types are now float (`FRect`, `FPoint`) instead of int.
> - Audio initialization: `SDL_OpenAudioDevice` + `SDL_QueueAudio` (SDL2) are replaced by the `AudioStream` API.

---

## Appendix: Common Nim/SDL3 Patterns

### Nim `defer` for SDL Resources

In Nim, `defer` is convenient for guaranteed cleanup of SDL resources:

```nim
let win = createWindow("App", 800, 600, 0)
if win == nil: quit(1)
defer: destroyWindow(win)

let rend = createRenderer(win, nil)
if rend == nil: quit(1)
defer: destroyRenderer(rend)
# defer runs in reverse order: destroyRenderer first, then destroyWindow
```

### Loading PNG/JPG via SDL_image (separate library)

SDL3 itself only loads BMP. For PNG/JPG/etc., use `sdl3_image`:
```nim
import sdl3_image   # separate package: nimble install sdl3_image

discard imgInit(IMG_INIT_PNG or IMG_INIT_JPG)
defer: imgQuit()

let
  surf = imgLoad("sprite.png")
  tex = createTextureFromSurface(rend, surf)
destroySurface(surf)
```

### Loading TTF Fonts via SDL_ttf

```nim
import sdl3_ttf  # nimble install sdl3_ttf

discard ttfInit()
defer: ttfQuit()

let
  font = openFont("DejaVuSans.ttf", 24)
  surf = renderTextBlended(font, "Hello!", Color(r: 255, g: 255, b: 255, a: 255))
  tex = createTextureFromSurface(rend, surf)
destroySurface(surf)# Render tex as a normal texture
closeFont(font)
```

### Project Structure with nimble

```
mygame/
  mygame.nimble
  src/
    main.nim
    game.nim
  assets/
    sprites/
    sounds/
```

`mygame.nimble`:
```nim
requires "nim >= 2.0.0"
requires "sdl3 >= 0.1.0"
```

### Working with `cstring` and Nim Strings

SDL3 functions accept and return `cstring`. Conversions:
```nim
# Nim string → cstring (safe within the scope)
let title = "My Game"
discard createWindow(title.cstring, 800, 600, 0)

# cstring → Nim string (copy)
let errMsg = $getError()  # the $ operator converts cstring → string

# nil check for cstring
let name = getRendererName(rend)
if name != nil:
  echo "Renderer: ", $name
```

---

*Reference compiled from `sdl3.nim` (SDL 3.1.8). Platforms: Windows (`SDL3.dll`), macOS (`libSDL3.dylib`), Linux (`libSDL3.so`), Emscripten (`libSDL3.so`).*
