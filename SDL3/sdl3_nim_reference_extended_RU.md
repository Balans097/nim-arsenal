# Справочник SDL3 для Nim

Основано на `sdl3.nim` — привязках к SDL 3.1.8.  
Все функции вызываются через UFCS или напрямую; `proc foo*(x: T)` означает, что можно писать как `foo(x)`, так и `x.foo()`.

**Обработка ошибок:** большинство процедур SDL3 возвращает `bool` (`true`). При `false` причина доступна через `getError()`. В примерах часто пишется `discard`, когда ошибка некритична; в продакшн-коде лучше проверять возвращаемое значение явно.

---

## Содержание

1. [Инициализация и завершение](#1-инициализация-и-завершение)
2. [Окна и дисплеи](#2-окна-и-дисплеи)
3. [Рендерер и текстуры](#3-рендерер-и-текстуры)
4. [События](#4-события)
5. [Клавиатура и текстовый ввод](#5-клавиатура-и-текстовый-ввод)
6. [Мышь и курсор](#6-мышь-и-курсор)
7. [Поверхности (Surface)](#7-поверхности-surface)
8. [Пиксельные форматы и цвета](#8-пиксельные-форматы-и-цвета)
9. [Аудио](#9-аудио)
10. [Таймеры](#10-таймеры)
11. [Файловая система и I/O](#11-файловая-система-и-io)
12. [Потоки, мьютексы, семафоры](#12-потоки-мьютексы-семафоры)
13. [Джойстики и геймпады](#13-джойстики-и-геймпады)
14. [Сенсоры и вибрация (Haptic)](#14-сенсоры-и-вибрация-haptic)
15. [GPU API](#15-gpu-api)
16. [Буфер обмена](#16-буфер-обмена)
17. [Свойства (Properties)](#17-свойства-properties)
18. [Логирование](#18-логирование)
19. [Прочее: CPU, системная информация, диалоги](#19-прочее-cpu-системная-информация-диалоги)
20. [Полный минимальный пример](#20-полный-минимальный-пример)

---

## 1. Инициализация и завершение

### Флаги подсистем

```nim
const INIT_AUDIO*:    uint32 = 0x00000010  # Audio (автоматически включает EVENTS)
const INIT_VIDEO*:    uint32 = 0x00000020  # Video + EVENTS, нужен в главном потоке
const INIT_JOYSTICK*: uint32 = 0x00000200  # Джойстик + EVENTS
const INIT_HAPTIC*:   uint32 = 0x00001000  # Вибрация
const INIT_GAMEPAD*:  uint32 = 0x00002000  # Геймпад + JOYSTICK
const INIT_EVENTS*:   uint32 = 0x00004000  # Только очередь событий
const INIT_SENSOR*:   uint32 = 0x00008000  # Сенсоры + EVENTS
const INIT_CAMERA*:   uint32 = 0x00010000  # Камера + EVENTS
```

### Функции

| Функция | Возвращает | Описание |
|---------|-----------|----------|
| `init(flags: InitFlags): bool` | `true` при успехе | Инициализирует указанные подсистемы |
| `initSubSystem(flags): bool` | `true` при успехе | Добавляет подсистемы после старта |
| `quitSubSystem(flags)` | — | Отключает указанные подсистемы |
| `wasInit(flags): InitFlags` | маска флагов | Возвращает набор уже инициализированных подсистем |
| `quit()` | — | Останавливает все подсистемы SDL |
| `getVersion(): cint` | версия | Числовая версия SDL (например, `3001008`) |
| `getRevision(): cstring` | строка | Строка ревизии сборки |

**`wasInit`** принимает 0 для получения маски **всех** активных подсистем: `wasInit(0)` возвращает битовую маску. Если нужно проверить конкретную подсистему, передайте её флаг: `(wasInit(INIT_VIDEO) and INIT_VIDEO) != 0`.

**`initSubSystem` / `quitSubSystem`** удобны для динамического включения/отключения аудио без перезапуска всего SDL. Вызовы подсчитываются — для каждого `initSubSystem(INIT_AUDIO)` нужен парный `quitSubSystem(INIT_AUDIO)`.

### Метаданные приложения

```nim
proc setAppMetadata*(appname, appversion, appidentifier: cstring): bool
proc setAppMetadataProperty*(name, value: cstring): bool
proc getAppMetadataProperty*(name: cstring): cstring
```

Стандартные имена свойств:
- `PROP_APP_METADATA_NAME_STRING` = `"SDL.app.metadata.name"`
- `PROP_APP_METADATA_VERSION_STRING`
- `PROP_APP_METADATA_IDENTIFIER_STRING`
- `PROP_APP_METADATA_CREATOR_STRING`

### Главный поток

```nim
proc isMainThread*(): bool
proc runOnMainThread*(callback: MainThreadCallback, userdata: pointer, wait_complete: bool): bool
```

**`INIT_VIDEO` требует главного потока.** Все операции с окнами и рендерером должны выполняться из того же потока, где был вызван `init(INIT_VIDEO)`. Если нужно создать окно из другого потока, используйте `runOnMainThread` с `wait_complete = true`.

**`isMainThread`** полезен в отладке: если вернул `false`, а вы пытаетесь создать окно — ждите проблем на macOS и Windows.

### Пример

```nim
import sdl3

if not init(INIT_VIDEO or INIT_AUDIO):
  echo "SDL init failed: ", getError()
  quit(1)

defer: quit()
```

---

## 2. Окна и дисплеи

### Типы

```nim
type
  Window* = ptr WindowObj
  DisplayID* = uint32
  WindowID* = uint32
  WindowFlags* = uint64    # битовые флаги
  DisplayMode* {.bycopy.} = object
    displayID*: DisplayID
    format*: PixelFormat
    w*, h*: cint
    pixel_density*: cfloat
    refresh_rate*: cfloat
    refresh_rate_numerator*: cint
    refresh_rate_denominator*: cint
```

### Флаги окна (WindowFlags)

В `sdl3.nim` константы `WindowFlags` экспортируются под именами вида `SDL_WINDOW_FULLSCREEN`, `SDL_WINDOW_RESIZABLE` и т.д. (или `WINDOW_FULLSCREEN` без префикса — зависит от версии привязок). Ниже приведены числовые значения для справки; в коде предпочтительно использовать именованные константы.

| Константа | Описание |
|-----------|----------|
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

### Создание и уничтожение

```nim
proc createWindow*(title: cstring, w, h: cint, flags: WindowFlags): Window
proc createPopupWindow*(parent: Window, offset_x, offset_y, w, h: cint, flags: WindowFlags): Window
proc createWindowWithProperties*(props: PropertiesID): Window
proc createWindowAndRenderer*(title: cstring, width, height: cint,
                              window_flags: WindowFlags,
                              window: var Window, renderer: var Renderer): bool
proc destroyWindow*(window: Window)
```

**Пример:**
```nim
let win = createWindow("Моё окно", 800, 600, 0)
if win == nil:
  echo getError()
defer: destroyWindow(win)
```

### Размер и позиция

```nim
proc setWindowSize*(window: Window, w, h: cint): bool
proc getWindowSize*(window: Window, w, h: var cint): bool
proc setWindowPosition*(window: Window, x, y: cint): bool
proc getWindowPosition*(window: Window, x, y: var cint): bool
proc getWindowSizeInPixels*(window: Window, w, h: var cint): bool  # учитывает DPI
proc setWindowMinimumSize*(window: Window, min_w, min_h: cint): bool
proc setWindowMaximumSize*(window: Window, max_w, max_h: cint): bool
proc getWindowMinimumSize*(window: Window, w, h: var cint): bool
proc getWindowMaximumSize*(window: Window, w, h: var cint): bool
proc setWindowAspectRatio*(window: Window, min_aspect, max_aspect: cfloat): bool
proc getWindowAspectRatio*(window: Window, min_aspect, max_aspect: var cfloat): bool
```

**`getWindowSize` vs `getWindowSizeInPixels`:** на дисплеях с высоким DPI (Retina, HiDPI) `getWindowSize` возвращает логический размер в «экранных координатах», а `getWindowSizeInPixels` — реальное количество пикселей (может быть в 2× больше). Для расчёта пиксельных координат (например, при рисовании) используйте `getWindowSizeInPixels`; для позиционирования интерфейса — `getWindowSize`.

**Пример чтения размера:**
```nim
var w, h: cint
discard getWindowSize(win, w, h)
echo "Логический размер: ", w, "×", h

var pw, ph: cint
discard getWindowSizeInPixels(win, pw, ph)
echo "Пиксельный размер: ", pw, "×", ph
```

### Заголовок, иконка, режим

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

### Состояние окна

```nim
proc showWindow*(window: Window): bool
proc hideWindow*(window: Window): bool
proc raiseWindow*(window: Window): bool
proc maximizeWindow*(window: Window): bool
proc minimizeWindow*(window: Window): bool
proc restoreWindow*(window: Window): bool
proc syncWindow*(window: Window): bool       # ждёт применения async-изменений
proc getWindowFlags*(window: Window): WindowFlags
proc getWindowID*(window: Window): WindowID
proc getWindowFromID*(id: WindowID): Window
proc getWindowParent*(window: Window): Window
proc flashWindow*(window: Window, operation: FlashOperation): bool
```

**`syncWindow`** необходим после операций, которые применяются асинхронно (например, `setWindowFullscreen`, `maximizeWindow`). Без него следующий `getWindowSize` может вернуть устаревшие значения. Вызывайте после группы изменений, перед чтением нового состояния.

**`flashWindow`** используется для привлечения внимания пользователя (например, мигание иконки в taskbar). Значения `FlashOperation`: `FLASH_BRIEFLY` — однократная вспышка, `FLASH_UNTIL_FOCUSED` — мигать до получения фокуса, `FLASH_CANCEL` — остановить.

### Захват мыши

```nim
proc setWindowKeyboardGrab*(window: Window, grabbed: bool): bool
proc setWindowMouseGrab*(window: Window, grabbed: bool): bool
proc getWindowKeyboardGrab*(window: Window): bool
proc getWindowMouseGrab*(window: Window): bool
proc getGrabbedWindow*(): Window
proc setWindowMouseRect*(window: Window, rect: ptr Rect): bool
proc getWindowMouseRect*(window: Window): ptr Rect
```

### Дисплеи

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

**Управление памятью:** функции вида `getDisplays`, `getFullscreenDisplayModes` возвращают указатель на массив, выделенный SDL. Его нужно освобождать через `sdlFree(cast[pointer](result))` после использования. Пример:
```nim
var count: cint
let displays = getDisplays(count)
defer: sdlFree(cast[pointer](displays))
for i in 0 ..< count:
  echo getDisplayName(displays[i])
```

**`getDisplayUsableBounds`** возвращает прямоугольник за вычетом панели задач и системных панелей (taskbar, dock). Используйте его при расчёте начального положения окна.

### Поверхность окна (без Renderer)

**Важно:** `getWindowSurface` и `Renderer` **несовместимы** для одного окна. Выбор делается при создании: либо `createRenderer` (GPU-ускорение), либо `getWindowSurface` (CPU-рисование напрямую в пиксели окна). Смешивать нельзя.

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

**Пример создания окна OpenGL:**
```nim
discard gL_SetAttribute(GL_CONTEXT_MAJOR_VERSION, 3)
discard gL_SetAttribute(GL_CONTEXT_MINOR_VERSION, 3)
let
  win = createWindow("OpenGL", 800, 600, 0x00000002u64) # OPENGL flag
  ctx = gL_CreateContext(win)
discard gL_MakeCurrent(win, ctx)
```

**`gL_DestroyContext`** — единственная функция удаления GL-контекста в SDL3; возвращает `bool`. `SDL_GL_DeleteContext` из SDL2 в обёртке отсутствует.

**`gL_SetSwapInterval(1)`** включает VSync (ждёт вертикального гасящего импульса). `0` — без ожидания, `-1` — адаптивный VSync (если поддерживается; при отсутствии поддержки возвращает ошибку, тогда откатитесь к `1`).

**Атрибуты нужно задавать ДО `createWindow`** — после создания окна `gL_SetAttribute` не имеет эффекта.

---

## 3. Рендерер и текстуры

Рендерер — аппаратно-ускоренный 2D-движок SDL3. Работает поверх Metal, Direct3D, Vulkan или OpenGL в зависимости от платформы.

### Типы

```nim
type
  Renderer* = ptr RendererObj
  Texture* = ptr TextureObj
  TextureAccess* = enum
    TEXTUREACCESS_STATIC,    # загружается один раз, GPU читает
    TEXTUREACCESS_STREAMING, # CPU обновляет каждый кадр
    TEXTUREACCESS_TARGET     # можно рисовать в текстуру

> **Выбор `TextureAccess`:**
> - `STATIC` — для спрайтов и фонов, которые не меняются. Наиболее эффективен.
> - `STREAMING` — для видео, процедурных текстур, обновляемых каждый кадр. Используйте `lockTexture` / `unlockTexture` или `updateTexture`.
> - `TARGET` — для рендера в текстуру (RTT: render to texture). Обязателен для пост-процессинга, теней, мини-карт.
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

### Создание рендерера

```nim
proc createRenderer*(window: Window, name: cstring): Renderer
  # name = nil → выбор автоматически; "software" → программный
proc createRendererWithProperties*(props: PropertiesID): Renderer
proc createSoftwareRenderer*(surface: ptr Surface): Renderer
proc destroyRenderer*(renderer: Renderer)
proc getRenderer*(window: Window): Renderer
proc getRenderWindow*(renderer: Renderer): Window
proc getRendererName*(renderer: Renderer): cstring
proc getNumRenderDrivers*(): cint
proc getRenderDriver*(index: cint): cstring
```

**Доступные бэкенды** можно перечислить через `getNumRenderDrivers` / `getRenderDriver`. Типичные значения на Linux: `"opengl"`, `"opengles2"`, `"software"`. На Windows добавляются `"direct3d11"`, `"direct3d12"`. На macOS — `"metal"`. Выбор `nil` автоматически выбирает лучший доступный.

**`createSoftwareRenderer`** создаёт рендерер на базе CPU поверх произвольной `Surface`. Полезно для рендеринга в память (offscreen rendering) без GPU-контекста.

**Пример:**
```nim
let renderer = createRenderer(win, nil)
if renderer == nil:
  echo "Renderer error: ", getError()
defer: destroyRenderer(renderer)
```

### Цвет, blend-режим, масштаб

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

### Viewport, clip, logical size

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

### Координатные преобразования

```nim
proc renderCoordinatesFromWindow*(renderer: Renderer, window_x, window_y: cfloat,
                                  x, y: var cfloat): bool
proc renderCoordinatesToWindow*(renderer: Renderer, x, y: cfloat,
                                window_x, window_y: var cfloat): bool
proc convertEventToRenderCoordinates*(renderer: Renderer, event: var Event): bool
```

### Примитивы рисования

```nim
proc renderClear*(renderer: Renderer): bool          # заливка текущим цветом

proc renderPoint*(renderer: Renderer, x, y: cfloat): bool
proc renderPoints*(renderer: Renderer, points: openArray[FPoint]): bool

proc renderLine*(renderer: Renderer, x1, y1, x2, y2: cfloat): bool
proc renderLines*(renderer: Renderer, points: openArray[FPoint]): bool  # ломаная

proc renderRect*(renderer: Renderer, rect: ptr FRect): bool  # контур
proc renderRect*(renderer: Renderer, rect: FRect): bool
proc renderRects*(renderer: Renderer, rects: openArray[FRect]): bool

proc renderFillRect*(renderer: Renderer, rect: ptr FRect): bool  # закрашенный
proc renderFillRect*(renderer: Renderer, rect: FRect): bool
proc renderFillRects*(renderer: Renderer, rects: openArray[FRect]): bool
```

**`renderLines`** рисует ломаную (polyline): соединяет точки последовательно `p[0]→p[1]→p[2]→...`. Для замкнутого контура продублируйте первую точку в конце массива.

**`renderRect`** (контур) рисует только рамку; **`renderFillRect`** заполняет прямоугольник. Обе принимают `ptr FRect` или `FRect` по значению (перегрузка). `nil` вместо `ptr FRect` означает весь viewport.

**`FRect` vs `Rect`:** SDL3 разделяет целочисленные (`Rect`, `Point`) и вещественные (`FRect`, `FPoint`) геометрические типы. Рендерер работает с `FRect`/`FPoint`; операции с пикселями Surface — с `Rect`/`Point`.

**Пример цикла рисования:**
```nim
discard setRenderDrawColor(renderer, 30, 30, 30, 255)
discard renderClear(renderer)
discard setRenderDrawColor(renderer, 255, 80, 0, 255)
discard renderFillRect(renderer, FRect(x: 100, y: 100, w: 200, h: 150))
discard renderPresent(renderer)
```

### Геометрия (произвольные треугольники)

**`renderGeometry`** — основа для 2D-спрайтов с вращением, нестандартных форм и particle-систем. Каждый `Vertex` содержит позицию, цвет (`FColor`, значения 0.0–1.0) и UV-координаты текстуры. Если `texture = nil`, рисуются цветные треугольники без текстуры. Массив `indices` позволяет переиспользовать вершины (indexed drawing).

**Пример: цветной треугольник:**
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

### Чтение пикселей

```nim
proc renderReadPixels*(renderer: Renderer, rect: ptr Rect): ptr Surface
# возвращает Surface с пикселями — необходимо destroySurface после использования
```

### Отображение и VSync

```nim
proc renderPresent*(renderer: Renderer): bool
proc flushRenderer*(renderer: Renderer): bool
proc setRenderVSync*(renderer: Renderer, vsync: cint): bool
proc getRenderVSync*(renderer: Renderer, vsync: var cint): bool

const RENDERER_VSYNC_DISABLED* = 0
const RENDERER_VSYNC_ADAPTIVE* = -1
```

**`renderPresent`** отображает текущий кадр на экран. При включённом VSync (`setRenderVSync(renderer, 1)`) вызов блокируется до вертикального гасящего импульса, ограничивая FPS частотой монитора.

**`flushRenderer`** принудительно отправляет накопленные команды в GPU, не меняя кадрового буфера. Нужен перед чтением состояния GPU или при интеграции с внешними API (OpenGL, Vulkan).

### Отладочный текст

```nim
proc renderDebugText*(renderer: Renderer, x, y: cfloat, str: cstring): bool
proc renderDebugTextFormat*(renderer: Renderer, x, y: cfloat, fmt: cstring): bool
const DEBUG_TEXT_FONT_CHARACTER_SIZE* = 8  # 8×8 пикселей на символ
```

### Рендер в текстуру (RTT)

```nim
proc setRenderTarget*(renderer: Renderer, texture: Texture): bool
  # texture = nil → вернуться на экран
proc getRenderTarget*(renderer: Renderer): Texture
```

**Пример RTT:**
```nim
let target = createTexture(renderer, PIXELFORMAT_RGBA8888,
                           TEXTUREACCESS_TARGET, 512, 512)
discard setRenderTarget(renderer, target)
# рисуем в target ...
discard setRenderTarget(renderer, nil)  # обратно на экран
discard renderTexture(renderer, target, nil, nil)
```

### Текстуры

```nim
proc createTexture*(renderer: Renderer, format: PixelFormat,
                    access: TextureAccess, w, h: cint): Texture
proc createTextureFromSurface*(renderer: Renderer, surface: ptr Surface): Texture
proc createTextureWithProperties*(renderer: Renderer, props: PropertiesID): Texture
proc destroyTexture*(texture: Texture)

proc getTextureSize*(texture: Texture, w, h: var cfloat): bool
proc setTextureColorMod*(texture: Texture, r, g, b: uint8): bool
proc setTextureColorModFloat*(texture: Texture, r, g, b: cfloat): bool
proc getTextureColorMod*(texture: Texture, r, g, b: uint8): bool  # ⚠ см. ниже
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

**⚠ `getTextureColorMod`:** в обёртке параметры `r`, `g`, `b` объявлены **без** `var` — передача по значению. Это баг привязок: функция не сможет вернуть значения через эти параметры. Используйте `getTextureProperties` + чтение числовых свойств как обходной путь до исправления.

**`updateTexture` vs `lockTexture`:** `updateTexture` копирует данные из произвольного буфера в текстуру — проще в использовании, но создаёт лишнее копирование. `lockTexture` возвращает прямой указатель `pixels` на внутренний буфер текстуры — быстрее для частых обновлений (видеопроигрыватель, процедурная генерация). Параметр `pitch` = ширина строки в байтах, **не** в пикселях. После `lockTexture` текстура заблокирована и не участвует в рендеринге до вызова `unlockTexture`.

**Пример streaming-текстуры:**
```nim
var
  pixels: pointer
  pitch: cint
discard lockTexture(tex, nil, pixels, pitch)
let buf = cast[ptr UncheckedArray[uint32]](pixels)
for y in 0 ..< 256:
  for x in 0 ..< 256:
    buf[y * (pitch div 4) + x] = 0xFF0000FF'u32  # красный, RGBA8888
unlockTexture(tex)
```

### Рендеринг текстур

```nim
proc renderTexture*(renderer: Renderer, texture: Texture,
                    srcrect, dstrect: ptr FRect): bool
  # srcrect = nil → вся текстура; dstrect = nil → весь экран

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

**`renderTextureRotated`:** угол задаётся в **градусах** (не радианах), по часовой стрелке. `center` — точка вращения относительно `dstrect`; `nil` = центр текстуры. `FlipMode`: `FLIP_NONE`, `FLIP_HORIZONTAL`, `FLIP_VERTICAL`.

**`renderTextureAffine`:** произвольное аффинное преобразование без ограничения на ось вращения. `origin` — левый верхний угол, `right` — правый верхний, `down` — левый нижний. Позволяет произвольный сдвиг, масштаб, скос и вращение за один вызов.

**`renderTexture9Grid`:** 9-сегментное масштабирование (9-patch/9-slice). Углы текстуры сохраняют исходный размер (умноженный на `scale`), края растягиваются только вдоль своей оси, центр растягивается в обоих направлениях. Стандартный подход для кнопок, панелей и диалогов.

**`renderTextureTiled`:** заполняет `dstrect` повторяющимися копиями `srcrect`. Параметр `scale` масштабирует каждую копию. Удобно для текстур земли, воды, фоновых паттернов.

### BlendMode

```nim
type BlendMode* = uint32

# Предопределённые значения:
const BLENDMODE_NONE*  = BlendMode(0x00000000)
const BLENDMODE_BLEND* = BlendMode(0x00000001)  # alpha-blending
const BLENDMODE_ADD*   = BlendMode(0x00000002)  # аддитивное смешение
const BLENDMODE_MOD*   = BlendMode(0x00000004)  # модуляция цвета
const BLENDMODE_MUL*   = BlendMode(0x00000008)  # умножение

proc composeCustomBlendMode*(srcColorFactor, dstColorFactor: BlendFactor,
                             colorOperation: BlendOperation,
                             srcAlphaFactor, dstAlphaFactor: BlendFactor,
                             alphaOperation: BlendOperation): BlendMode
```

**Когда что использовать:**
- `BLENDMODE_NONE` — без прозрачности, максимально быстро.
- `BLENDMODE_BLEND` — стандартный alpha-blending: `result = src_color * src_alpha + dst_color * (1 - src_alpha)`.
- `BLENDMODE_ADD` — аддитивное смешение: `result = src_color * src_alpha + dst_color`. Используется для огня, свечения, частиц.
- `BLENDMODE_MOD` — умножает цвет назначения на цвет источника. Используется для теней и тонирования.
- `BLENDMODE_MUL` — учитывает alpha источника при умножении. Аналог `BLENDMODE_MOD` с поддержкой прозрачности.
- `composeCustomBlendMode` — для нестандартных эффектов (premultiplied alpha, экранный режим и т.д.).

---

## 4. События

### Основные типы

**`Event`** — это C-union (в Nim: объект с вариантными полями). Общая часть: `event.type` и `event.timestamp`. Конкретные данные доступны через подполя по типу события: `event.key` (клавиатура), `event.button` / `event.motion` / `event.wheel` (мышь), `event.window` (окно), `event.text` (ввод текста), `event.drop` (drag & drop) и т.д. Размер `Event` фиксирован (128 байт) вне зависимости от типа события.

```nim
type
  EventType* = enum
    EVENT_FIRST = 0,
    EVENT_QUIT = 0x100,           # пользователь закрыл окно
    EVENT_TERMINATING,
    # ... системные события приложения ...
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
    button*: uint8       # 1=левая, 2=средняя, 3=правая
    down*: bool
    clicks*: uint8       # 1=одиночный, 2=двойной
    padding*: uint8
    x*, y*: cfloat
```

### Опрос событий

```nim
proc pollEvent*(event: var Event): bool
  # возвращает true, если есть событие; не блокирует

proc waitEvent*(event: var Event): bool
  # ждёт бесконечно

proc waitEventTimeout*(event: var Event, timeoutMS: int32): bool
  # ждёт не более timeoutMS миллисекунд

proc pumpEvents*()
  # обновляет очередь без получения событий

proc pushEvent*(event: var Event): bool
  # помещает событие в очередь вручную

# EventAction:
#   AddEventAction   — добавить в очередь
#   PeekEventAction  — просмотреть без извлечения
#   GetEventAction   — извлечь из очереди
proc peepEvents*(events: ptr[Event], numevents: cint, action: EventAction,
                 minType, maxType: uint32): cint
proc peepEvents*(events: openArray[Event], action: EventAction,
                 minType, maxType: uint32): cint
proc hasEvent*(kind: uint32): bool
proc hasEvents*(minType, maxType: uint32): bool
proc flushEvent*(kind: uint32)
proc flushEvents*(minType, maxType: uint32)
```

**`pollEvent` vs `waitEvent`:** игровые циклы используют `pollEvent` (не блокирует, позволяет рендерить кадры). `waitEvent` / `waitEventTimeout` подходят для редакторов и инструментов, которым не нужно ничего делать без событий (экономит CPU).

**`pumpEvents`** вызывается автоматически внутри `pollEvent` / `waitEvent`. Нужен явно только если вы читаете состояние клавиатуры/мыши без обработки событий (`getKeyboardState`, `getMouseState`).

**`pushEvent`** используется для межпоточной коммуникации (из рабочего потока в главный) и для кастомных событий (`EVENT_USER`).

**`peepEvents`** с `action = PeekEventAction` позволяет посмотреть очередь **без извлечения** событий; с `GetEventAction` — извлечь только события определённого диапазона типов. Первая перегрузка (с `ptr[Event]` и `numevents`) — прямое соответствие C API; вторая принимает `openArray[Event]` и определяет `numevents` автоматически.

### Фильтры событий

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

### Стандартный игровой цикл

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

## 5. Клавиатура и текстовый ввод

### Состояние клавиатуры

```nim
proc hasKeyboard*(): bool
proc getKeyboards*(count: var cint): ptr UncheckedArray[KeyboardID]
proc getKeyboardNameForID*(instance_id: KeyboardID): cstring
proc getKeyboardFocus*(): Window

proc getKeyboardState*(numkeys: var cint): ptr UncheckedArray[bool]
  # возвращает снимок: индексируется по Scancode
  # например: state[SDL_SCANCODE_W.ord]

proc resetKeyboard*()
proc getModState*(): Keymod
proc setModState*(modstate: Keymod)
```

**`getKeyboardState`** возвращает указатель на внутренний массив SDL — **не копировать**. Снимок актуален до следующего `pumpEvents` / `pollEvent`. Индексирование: `state[SCANCODE_W.ord]` (приводим `Scancode` к `int`).

**`Scancode` vs `Keycode`:** `Scancode` — физическая позиция клавиши на клавиатуре (не зависит от раскладки). `Keycode` — символ, который генерирует клавиша в текущей раскладке. Для игровых действий (WASD) используйте `Scancode`; для текстовых полей — `Keycode` или `EVENT_TEXT_INPUT`.

**Пример непрерывного движения:**
```nim
var numkeys: cint
let state = getKeyboardState(numkeys)
if state[SCANCODE_W.ord]: playerY -= speed
if state[SCANCODE_S.ord]: playerY += speed
if state[SCANCODE_A.ord]: playerX -= speed
if state[SCANCODE_D.ord]: playerX += speed
```

### Коды клавиш

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

### Текстовый ввод (IME)

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

**Пример получения текста:**
```nim
discard startTextInput(win)
# затем в цикле событий:
of EVENT_TEXT_INPUT:
  echo "Введено: ", event.text.text  # UTF-8 строка
```

---

## 6. Мышь и курсор

### Состояние мыши

```nim
proc hasMouse*(): bool
proc getMice*(count: var cint): ptr UncheckedArray[MouseID]
proc getMouseNameForID*(instance_id: MouseID): cstring
proc getMouseFocus*(): Window

proc getMouseState*(x, y: var cfloat): uint32
  # возвращает битовую маску кнопок; x,y — позиция в окне

proc getGlobalMouseState*(x, y: var cfloat): uint32
proc getRelativeMouseState*(x, y: var cfloat): uint32
  # dx, dy с последнего вызова
```

**Декодирование битовой маски кнопок:** возвращаемое `uint32` — набор битов, по одному на кнопку. Маска кнопки N = `(1'u32 shl (N - 1))`. Кнопки: 1=левая, 2=средняя, 3=правая, 4=X1, 5=X2.
```nim
let
  buttons = getMouseState(mx, my)
  leftDown  = (buttons and (1'u32 shl 0)) != 0
  rightDown = (buttons and (1'u32 shl 2)) != 0
```

**`getRelativeMouseState`** возвращает дельту движения с момента предыдущего вызова, а не абсолютные координаты. Используется для управления камерой от первого лица. Для корректной работы включайте относительный режим: `setWindowRelativeMouseMode(win, true)` — курсор скрывается и перестаёт упираться в границу окна.

### Управление мышью

```nim
proc warpMouseInWindow*(window: Window, x, y: cfloat)
proc warpMouseGlobal*(x, y: cfloat): bool
proc setWindowRelativeMouseMode*(window: Window, enabled: bool): bool
proc getWindowRelativeMouseMode*(window: Window): bool
proc captureMouse*(enabled: bool): bool  # захват вне окна
```

### Курсор

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
                   hot_x, hot_y: cint): Cursor   # ч/б курсор
proc setCursor*(cursor: Cursor): bool
proc getCursor*(): Cursor
proc getDefaultCursor*(): Cursor
proc destroyCursor*(cursor: Cursor)
proc showCursor*(): bool
proc hideCursor*(): bool
proc cursorVisible*(): bool
```

**Пример смены курсора:**
```nim
let cursor = createSystemCursor(SYSTEM_CURSOR_CROSSHAIR)
discard setCursor(cursor)
# ... позже:
destroyCursor(cursor)
```

---

## 7. Поверхности (Surface)

`Surface` — пиксельный буфер в памяти CPU. Используется для загрузки изображений, CPU-рендеринга и создания текстур.

### Структура

```nim
type
  Surface* {.bycopy.} = object
    flags*: SurfaceFlags
    format*: PixelFormat
    w*, h*: cint
    pitch*: cint                        # байт на строку
    pixels*: ptr UncheckedArray[uint8]  # указатель на пиксели (доступен после Lock)
    refcount*: cint
    reserved*: pointer
```

**`pitch`** (шаг строки) — количество **байт** на одну строку пикселей, **не** пикселей. Из-за выравнивания памяти `pitch` может быть больше `w * bytesPerPixel`. Всегда используйте `pitch` для перехода на следующую строку:
```nim
let bpp = 4  # для RGBA8888
let pixel = cast[ptr uint32](addr surf.pixels[y * surf.pitch + x * bpp])
```

**`refcount`** используется внутри SDL для безопасной передачи поверхностей между функциями. Не изменяйте вручную.

### Создание и уничтожение

```nim
proc createSurface*(width, height: cint, format: PixelFormat): ptr Surface
proc createSurfaceFrom*(width, height: cint, format: PixelFormat,
                        pixels: pointer, pitch: cint): ptr Surface
proc destroySurface*(surface: ptr Surface)
proc duplicateSurface*(surface: ptr Surface): ptr Surface
proc scaleSurface*(surface: ptr Surface, width, height: cint,
                   scaleMode: ScaleMode): ptr Surface
```

### Загрузка и сохранение BMP

```nim
proc loadBMP*(file: cstring): ptr Surface
proc loadBMP_IO*(src: IOStream, closeio: bool): ptr Surface
proc saveBMP*(surface: ptr Surface, file: cstring): bool
proc saveBMP_IO*(surface: ptr Surface, dst: IOStream, closeio: bool): bool
```

**Пример создания текстуры из файла:**
```nim
let surf = loadBMP("image.bmp")
if surf == nil: echo getError()
let tex = createTextureFromSurface(renderer, surf)
destroySurface(surf)
```

### Доступ к пикселям

```nim
proc lockSurface*(surface: ptr Surface): bool
proc unlockSurface*(surface: ptr Surface)
func mUSTLOCK*(s: ptr Surface): bool  # нужно ли Lock перед доступом?

proc readSurfacePixel*(surface: ptr Surface, x, y: cint,
                       r, g, b, a: var uint8): bool
proc writeSurfacePixel*(surface: ptr Surface, x, y: cint,
                        r, g, b, a: uint8): bool
proc readSurfacePixelFloat*(surface: ptr Surface, x, y: cint,
                             r, g, b, a: var cfloat): bool
proc writeSurfacePixelFloat*(surface: ptr Surface, x, y: cint,
                              r, g, b, a: cfloat): bool
```

**`mustLock`:** не все поверхности требуют блокировки перед доступом к `pixels`. Проверяйте через `mUSTLOCK(surf)` и блокируйте только если необходимо. Аппаратные поверхности (SDL_Surface с флагом RLE или видеопамяти) обязательно требуют Lock.

**`readSurfacePixel` / `writeSurfacePixel`** работают без явного Lock и без знания формата — SDL сам декодирует пиксель. Медленнее прямого доступа через `pixels`, но удобны для редких операций.

### Операции с поверхностями

```nim
proc blitSurface*(src: ptr Surface, srcrect: ptr Rect,
                  dst: ptr Surface, dstrect: ptr Rect): bool
proc blitSurfaceScaled*(src: ptr Surface, srcrect: ptr Rect,
                        dst: ptr Surface, dstrect: ptr Rect,
                        scaleMode: ScaleMode): bool
proc blitSurfaceTiled*(src, dst: ptr Surface, ...): bool
proc blitSurface9Grid*(src: ptr Surface, ...): bool  # 9-сегментный масштаб

proc fillSurfaceRect*(dst: ptr Surface, rect: ptr Rect, color: uint32): bool
proc fillSurfaceRects*(dst: ptr Surface, rect: openArray[Rect], color: uint32): bool
proc clearSurface*(surface: ptr Surface, r, g, b, a: cfloat): bool

proc convertSurface*(surface: ptr Surface, format: PixelFormat): ptr Surface
proc convertSurfaceAndColorspace*(surface: ptr Surface, format: PixelFormat,
                                  palette: ptr Palette, colorspace: Colorspace,
                                  props: PropertiesID): ptr Surface

proc flipSurface*(surface: ptr Surface, flip: FlipMode): bool
```

### Свойства поверхности

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

## 8. Пиксельные форматы и цвета

### Типы

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

### Основные форматы PixelFormat (перечень)

| Константа | Описание |
|-----------|----------|
| `PIXELFORMAT_RGBA8888` | 32 бита, RGBA |
| `PIXELFORMAT_ARGB8888` | 32 бита, ARGB |
| `PIXELFORMAT_RGB24` | 24 бита, RGB |
| `PIXELFORMAT_BGR24` | 24 бита, BGR |
| `PIXELFORMAT_RGBA32` | псевдоним, little-endian |
| `PIXELFORMAT_ARGB32` | псевдоним, little-endian |
| `PIXELFORMAT_RGB565` | 16 бит |
| `PIXELFORMAT_INDEX8` | 8 бит, палитра |
| `PIXELFORMAT_YV12`, `PIXELFORMAT_NV12`, ... | YUV форматы |

### Операции с форматами

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

### Палитра

```nim
proc createPalette*(ncolors: cint): ptr Palette
proc setPaletteColors*(palette: ptr Palette, colors: ptr[Color],
                       firstcolor, ncolors: cint): bool
proc destroyPalette*(palette: ptr Palette)
```

### Преобразование пикселей

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

## 9. Аудио

### Типы

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
    freq*: cint              # Гц, например 44100 или 48000

> **Типичные значения `AudioSpec`:**
> - `freq`: 44100 (CD-качество) или 48000 (профессиональное аудио/HDMI).
> - `channels`: 1 (моно), 2 (стерео), 6 (5.1).
> - `format`: `AUDIO_F32LE` — наиболее универсальный для DSP; `AUDIO_S16LE` — совместимость с большинством WAV-файлов.
>
> `AudioStream` автоматически конвертирует между форматами при вызове `putAudioStreamData` / `getAudioStreamData`, поэтому src_spec и dst_spec могут различаться.
  AudioStream* = ptr object
  AudioStreamCallback* = proc (userdata: pointer; stream: AudioStream;
                               additional_amount, total_amount: cint) {.cdecl.}
  AudioPostmixCallback* = proc (userdata: pointer; spec: ptr AudioSpec;
                                buffer: ptr cfloat; buflen: cint) {.cdecl.}
```

### Устройства воспроизведения

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

**Устройство по умолчанию:** передавайте `0xFFFFFFFF'u32` (`SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK`) как `devid` в `openAudioDevice` для автоматического выбора системного устройства воспроизведения. Для записи: `0xFFFFFFFE'u32` (`SDL_AUDIO_DEVICE_DEFAULT_RECORDING`).

**`spec = nil`** в `openAudioDevice` — SDL откроет устройство с нативным форматом. После открытия узнать реальный формат можно через `getAudioDeviceFormat`.

### AudioStream — основной API воспроизведения

`AudioStream` — гибкий буфер с конвертацией формата на лету.

```nim
proc createAudioStream*(src_spec, dst_spec: ptr AudioSpec): AudioStream
proc destroyAudioStream*(stream: AudioStream)
proc getAudioStreamFormat*(stream: AudioStream,
                           src_spec, dst_spec: ptr AudioSpec): bool
proc setAudioStreamFormat*(stream: AudioStream,
                           src_spec, dst_spec: ptr AudioSpec): bool

proc putAudioStreamData*(stream: AudioStream, buf: pointer, len: cint): bool
proc getAudioStreamData*(stream: AudioStream, buf: pointer, len: cint): cint
proc getAudioStreamAvailable*(stream: AudioStream): cint  # байт готово к чтению
proc getAudioStreamQueued*(stream: AudioStream): cint      # байт в очереди
proc flushAudioStream*(stream: AudioStream): bool  # завершить поток данных
proc clearAudioStream*(stream: AudioStream): bool  # сбросить буфер

proc getAudioStreamGain*(stream: AudioStream): cfloat
proc setAudioStreamGain*(stream: AudioStream, gain: cfloat): bool
proc getAudioStreamFrequencyRatio*(stream: AudioStream): cfloat
proc setAudioStreamFrequencyRatio*(stream: AudioStream, ratio: cfloat): bool
```

### Привязка потока к устройству

```nim
proc bindAudioStream*(devid: AudioDeviceID, stream: AudioStream): bool
proc unbindAudioStream*(stream: AudioStream)
proc openAudioDeviceStream*(devid: AudioDeviceID, spec: ptr AudioSpec,
                            callback: AudioStreamCallback,
                            userdata: pointer): AudioStream
```

### Обратные вызовы потока

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

### Загрузка WAV

```nim
proc loadWAV*(path: cstring, spec: ptr AudioSpec,
              audio_buf: var ptr[uint8], audio_len: var uint32): bool
proc loadWAV_IO*(src: IOStream, closeio: bool, spec: ptr AudioSpec,
                 audio_buf: var UncheckedArray[uint8],
                 audio_len: var uint32): bool
```

### Конвертация и микширование

```nim
proc convertAudioSamples*(src_spec: ptr AudioSpec, src_data: ptr[uint8],
                          src_len: cint, dst_spec: ptr AudioSpec,
                          dst_data: var ptr[uint8], dst_len: var cint): bool
proc mixAudio*(dst, src: ptr [uint8], format: AudioFormat,
               len: uint32, volume: cfloat): bool
proc getAudioFormatName*(format: AudioFormat): cstring
proc getSilenceValueForFormat*(format: AudioFormat): cint
```

**Полный пример воспроизведения WAV:**
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

## 10. Таймеры

### Время

```nim
proc getTicks*(): uint64      # миллисекунды с инициализации SDL
proc getTicksNS*(): uint64    # наносекунды
proc getPerformanceCounter*(): uint64   # высокоточный счётчик
proc getPerformanceFrequency*(): uint64 # тики в секунду

# Задержки
proc delay*(ms: uint32)          # блокирует поток на ms мс
proc delayNS*(ns: uint64)        # задержка в наносекундах
proc delayPrecise*(ns: uint64)   # точная задержка (spin-loop)
```

**Delta time (время кадра)** — стандартный способ расчёта:
```nim
var lastCounter = getPerformanceCounter()
let freq = getPerformanceFrequency().float

# в начале каждого кадра:
let
  now = getPerformanceCounter()
  dt = float(now - lastCounter) / freq  # секунды
lastCounter = now
```

**`delay` vs `delayPrecise`:** `delay(ms)` использует системный sleep — точность ±1–2 мс на большинстве ОС. `delayPrecise(ns)` уточняет задержку через spin-wait после sleep — точнее, но потребляет CPU. Используйте `delayPrecise` только для задержек < 2 мс.

**`getTicks()`** переполнится через ~584 млн лет (uint64). Для измерения коротких интервалов предпочтительнее `getPerformanceCounter()`.

### Вспомогательные inline-конверторы

```nim
proc secondsToNs*(S: uint64): uint64
proc nsToSeconds*(NS: uint64): uint64
proc msToNs*(MS: uint64): uint64
proc nsToMs*(NS: uint64): uint64
proc usToNs*(US: uint64): uint64
proc nsToUs*(NS: uint64): uint64
```

### Таймеры с обратным вызовом

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

**Пример: обратный вызов каждые 500 мс:**
```nim
proc onTimer(userdata: pointer, id: TimerID, interval: uint32): uint32 {.cdecl.} =
  echo "Тик!"
  return interval  # возвращаем тот же интервал → повтор

let tid = addTimer(500, onTimer, nil)
# ... позже:
discard removeTimer(tid)
```

**Контракт обратного вызова таймера:** если callback возвращает `0` — таймер останавливается; любое ненулевое значение — следующий интервал в мс. Это позволяет делать адаптивные таймеры с изменяемым интервалом.

**Таймеры выполняются в отдельном потоке.** Доступ к разделяемым данным требует синхронизации (мьютекс или атомарные операции). Для отправки результатов в главный поток используйте `pushEvent`.

### Дата и время

```nim
type
  DateTime* {.bycopy.} = object
    year*, month*, day*: cint
    hour*, minute*, second*: cint
    nanosecond*: cint
    day_of_week*: cint  # 0=воскресенье
    utc_offset*: cint   # секунды

proc getCurrentTime*(ticks: var Time): bool
proc timeToDateTime*(ticks: Time, dt: var DateTime, localTime: bool): bool
proc dateTimeToTime*(dt: ptr DateTime, ticks: var Time): bool
proc getDaysInMonth*(year, month: cint): cint
proc getDayOfYear*(year, month, day: cint): cint
proc getDayOfWeek*(year, month, day: cint): cint
```

---

## 11. Файловая система и I/O

### IOStream — универсальный поток

```nim
type
  IOStream* = ptr object
  IOStatus* = enum
    IO_STATUS_READY, IO_STATUS_ERROR, IO_STATUS_EOF,
    IO_STATUS_NOT_READY, IO_STATUS_READONLY, IO_STATUS_WRITEONLY
  IOWhence* = enum
    IO_SEEK_SET, IO_SEEK_CUR, IO_SEEK_END

```nim
proc ioFromFile*(file, mode: cstring): IOStream   # "r", "w", "rb", "wb" ...
proc ioFromMem*(mem: pointer, size: csize_t): IOStream
proc ioFromConstMem*(mem: pointer, size: csize_t): IOStream
proc ioFromDynamicMem*(): IOStream   # авторасширяемый буфер
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

### Чтение/запись типизированных данных

```nim
proc readU8*(src: IOStream, value: var uint8): bool
proc readS16LE*(src: IOStream, value: var int16): bool
proc readU32LE*(src: IOStream, value: var uint32): bool
proc readU64LE*(src: IOStream, value: var uint64): bool
# ... аналогичные для S8, U16BE, S32BE, S64LE, S64BE и т.д.

proc writeU8*(dst: IOStream, value: uint8): bool
proc writeU32LE*(dst: IOStream, value: uint32): bool
# ... аналогичные
```

### Работа с файлами и директориями

```nim
proc loadFile*(file: cstring, datasize: var csize_t): pointer
proc saveFile*(file: cstring, data: pointer, datasize: csize_t): bool
proc loadFileIO*(src: IOStream, datasize: var csize_t, closeio: bool): pointer
proc saveFileIO*(src: IOStream, data: pointer, datasize: csize_t, closeio: bool): bool

proc getBasePath*(): cstring  # путь к исполняемому файлу
proc getPrefPath*(org, app: cstring): cstring  # путь для сохранений
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

> **`getBasePath()`** возвращает директорию исполняемого файла (с завершающим слешем). Используется для загрузки ресурсов, поставляемых вместе с игрой. На macOS внутри `.app` bundle указывает на `Contents/MacOS/`.

> **`getPrefPath(org, app)`** возвращает платформенный путь для пользовательских данных (сохранения, конфиги):
> - Windows: `%APPDATA%\org\app\`
> - Linux: `~/.local/share/org/app/` (XDG)
> - macOS: `~/Library/Application Support/org/app/`
>
> Директория создаётся автоматически. Возвращённую строку нужно освобождать через `sdlFree` после использования.

> **`globDirectory`** возвращает массив путей, соответствующих паттерну (например `"*.png"`). Освобождать через `sdlFree(cast[pointer](result))`.

### Асинхронный I/O

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

### Storage API (платформенные хранилища)

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

## 12. Потоки, мьютексы, семафоры

### Потоки

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

> **Создание потока:** используется `createThread` (обёртка, требующая имени для корректной работы на всех платформах):
> ```nim
> proc workerFunc(data: pointer): cint {.cdecl.} =
>   # работа в фоне
>   return 0
>
> let thread = createThread(workerFunc, "WorkerThread", nil)
> var status: cint
> waitThread(thread, status)
> ```

> **`waitThread` vs `detachThread`:** `waitThread` блокирует вызывающий поток до завершения дочернего и освобождает ресурсы. `detachThread` позволяет потоку работать независимо — ресурсы освобождаются при его завершении, но `waitThread` после этого нельзя вызывать.

> **SDL и потоки Nim:** Nim имеет собственную систему потоков (`std/threads`). При смешивании с SDL потоками убедитесь, что GC-инварианты соблюдены; работа с рендерером и окнами остаётся только в главном потоке.

### Мьютекс

```nim
type SdlMutex* = ptr object

proc createMutex*(): SdlMutex
proc lockMutex*(mutex: SdlMutex)
proc tryLockMutex*(mutex: SdlMutex): bool
proc unlockMutex*(mutex: SdlMutex)
proc destroyMutex*(mutex: SdlMutex)
```

> **Паттерн безопасного использования мьютекса в Nim:**
> ```nim
> let mtx = createMutex()
> defer: destroyMutex(mtx)
>
> lockMutex(mtx)
> try:
>   # критическая секция
> finally:
>   unlockMutex(mtx)
> ```
> `tryLockMutex` возвращает `true` если захват удался немедленно, `false` — если мьютекс уже занят. Не блокирует.

### RW-локи

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

### Семафоры

```nim
type SdlSemaphore* = ptr object

proc createSemaphore*(initial_value: uint32): SdlSemaphore
proc destroySemaphore*(sem: SdlSemaphore)
proc waitSemaphore*(sem: SdlSemaphore)           # блокирует
proc tryWaitSemaphore*(sem: SdlSemaphore): bool  # не блокирует
proc waitSemaphoreTimeout*(sem: SdlSemaphore, timeoutMS: int32): bool
proc signalSemaphore*(sem: SdlSemaphore)
proc getSemaphoreValue*(sem: SdlSemaphore): uint32
```

### Условные переменные

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

### Spinlock и барьеры памяти

```nim
type SpinLock* = cint

proc tryLockSpinlock*(lock: var SpinLock): bool
proc lockSpinlock*(lock: var SpinLock)
proc unlockSpinlock*(lock: var SpinLock)
proc memoryBarrierReleaseFunction*()
proc memoryBarrierAcquireFunction*()
```

---

## 13. Джойстики и геймпады

### Джойстики (низкоуровневый API)

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

### Геймпады (высокоуровневый API)

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

**Пример обработки геймпада:**
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

## 14. Сенсоры и вибрация (Haptic)

### Сенсоры

```nim
type
  Sensor* = ptr object
  SensorID* = uint32
  SensorType* = enum
    SENSOR_INVALID = -1, SENSOR_UNKNOWN,
    SENSOR_ACCEL,   # ускорение, м/с²
    SENSOR_GYRO,    # угловая скорость, рад/с
    SENSOR_ACCEL_L, SENSOR_GYRO_L,  # левый джойкон
    SENSOR_ACCEL_R, SENSOR_GYRO_R   # правый джойкон

proc getSensors*(count: var cint): ptr UncheckedArray[SensorID]
proc openSensor*(instance_id: SensorID): Sensor
proc closeSensor*(sensor: Sensor)
proc getSensorName*(sensor: Sensor): cstring
proc getSensorType*(sensor: Sensor): SensorType
proc getSensorData*(sensor: Sensor, data: openArray[cfloat]): bool
proc updateSensors*()
```

### Вибрация (Haptic)

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

SDL3 содержит низкоуровневый GPU API (аналог WebGPU/Metal/Vulkan/D3D12), предназначенный для продвинутого рендеринга.

> **Когда использовать GPU API вместо Renderer:**
> - Нужны шейдеры (GLSL/HLSL/MSL) и полный контроль над пайплайном.
> - Требуется Compute (вычислительные шейдеры, GPGPU).
> - Нужны инстансированный рендеринг, MRT (множественные render targets), стенсил.
> - Требуется максимальная производительность с минимальным overhead.
>
> Для простой 2D-графики SDL Renderer достаточен и значительно проще в использовании.

> **Поддерживаемые форматы шейдеров (`GPUShaderFormat`):** зависят от платформы.
> - Windows: `GPU_SHADERFORMAT_DXBC` (HLSL → DXC), `GPU_SHADERFORMAT_SPIRV`
> - macOS/iOS: `GPU_SHADERFORMAT_MSL`, `GPU_SHADERFORMAT_METALLIB`
> - Linux/Android: `GPU_SHADERFORMAT_SPIRV`
>
> Для кросс-платформенного кода используйте SPIR-V и транспилируйте через `spirv-cross`.

### Основные объекты

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

  GPUShaderFormat* = uint32   # битовая маска поддерживаемых форматов шейдеров
```

### Устройство

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

### Ресурсы

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

# Освобождение (не destroyXxx, а releaseXxx):
proc releaseGPUShader*(device: GPUDevice, shader: GPUShader)
proc releaseGPUTexture*(device: GPUDevice, texture: GPUTexture)
proc releaseGPUBuffer*(device: GPUDevice, buffer: GPUBuffer)
proc releaseGPUSampler*(device: GPUDevice, sampler: GPUSampler)
proc releaseGPUTransferBuffer*(device: GPUDevice, transfer_buffer: GPUTransferBuffer)
proc releaseGPUGraphicsPipeline*(device: GPUDevice, graphics_pipeline: GPUGraphicsPipeline)
proc releaseGPUComputePipeline*(device: GPUDevice, compute_pipeline: GPUComputePipeline)
proc releaseGPUFence*(device: GPUDevice, fence: GPUFence)
```

> **Соглашение `release` вместо `destroy`:** GPU-ресурсы используют `releaseXxx`, а не `destroyXxx`. Освобождение может произойти не немедленно — SDL дождётся, пока GPU закончит использование ресурса (аналог Vulkan `vkDestroyXxx` после `vkDeviceWaitIdle`). Никогда не освобождайте ресурс, который может быть ещё в использовании на GPU.

### Command buffer и passes

> **Жизненный цикл команд GPU:**
> ```
> acquireGPUCommandBuffer
>   → beginGPURenderPass / beginGPUCopyPass / beginGPUComputePass
>       → bindXxx / drawXxx / dispatchXxx
>     → endGPURenderPass / endGPUCopyPass / endGPUComputePass
>   → submitGPUCommandBuffer  (или cancelGPUCommandBuffer)
> ```
> После `submitGPUCommandBuffer` командный буфер недействителен — нельзя к нему обращаться. Нельзя иметь два активных pass одновременно.

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

> **`cycle`** в `mapGPUTransferBuffer`: если `true` и буфер ещё используется GPU, SDL создаст новый буфер вместо ожидания — «карусельная» (cycled) схема загрузки данных без блокировки. Если `false` — ждёт освобождения. Для загрузки данных каждый кадр (uniform buffers, динамические вершины) используйте `cycle = true`.

> **Паттерн загрузки вершинных данных на GPU:**
> ```nim
> # 1. Создаём transfer buffer
> let tb = createGPUTransferBuffer(device, addr GPUTransferBufferCreateInfo(
>   usage: GPU_TRANSFERBUFFERUSAGE_UPLOAD,
>   size: uint32(sizeof(Vertex) * numVerts)
> ))
> # 2. Записываем данные
> let mapped = cast[ptr UncheckedArray[Vertex]](mapGPUTransferBuffer(device, tb, false))
> for i in 0 ..< numVerts:
>   mapped[i] = vertices[i]
> unmapGPUTransferBuffer(device, tb)
> # 3. Копируем в GPU buffer через copy pass
> let copyPass = beginGPUCopyPass(cmdBuf)
> uploadToGPUBuffer(copyPass, addr GPUTransferBufferLocation(transfer_buffer: tb, offset: 0),
>                  addr GPUBufferRegion(buffer: vertBuf, offset: 0, size: ...),
>                  cycle = false)
> endGPUCopyPass(copyPass)
> ```

### Swapchain и синхронизация

```nim
proc claimWindowForGPUDevice*(device: GPUDevice, window: Window): bool
proc releaseWindowFromGPUDevice*(device: GPUDevice, window: Window)
proc setGPUSwapchainParameters*(device: GPUDevice, window: Window,
                                swapchain_composition: GPUSwapchainComposition,
                                present_mode: GPUPresentMode): bool
proc acquireGPUSwapchainTexture*(command_buffer: GPUCommandBuffer,
                                 window: Window,
                                 swapchain_texture: GPUTexture,        # не var
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

### Форматы текстур (примеры)

| Константа | Описание |
|-----------|----------|
| `GPU_TEXTUREFORMAT_R8G8B8A8_UNORM` | 8 бит RGBA, нормализованный |
| `GPU_TEXTUREFORMAT_B8G8R8A8_UNORM` | 8 бит BGRA |
| `GPU_TEXTUREFORMAT_R16G16B16A16_FLOAT` | 16 бит float RGBA |
| `GPU_TEXTUREFORMAT_D16_UNORM` | 16-бит глубина |
| `GPU_TEXTUREFORMAT_D32_FLOAT` | 32-бит глубина float |
| `GPU_TEXTUREFORMAT_BC1_RGBA_UNORM` | DXT1 сжатие |
| `GPU_TEXTUREFORMAT_BC3_RGBA_UNORM` | DXT5 сжатие |

### Отладка

```nim
proc setGPUBufferName*(device: GPUDevice, buffer: GPUBuffer, text: cstring)
proc setGPUTextureName*(device: GPUDevice, texture: GPUTexture, text: cstring)
proc insertGPUDebugLabel*(command_buffer: GPUCommandBuffer, text: cstring)
proc pushGPUDebugGroup*(command_buffer: GPUCommandBuffer, name: cstring)
proc popGPUDebugGroup*(command_buffer: GPUCommandBuffer)
```

---

## 16. Буфер обмена

```nim
proc setClipboardText*(text: cstring): bool
proc getClipboardText*(): cstring
proc hasClipboardText*(): bool

# X11 primary selection:
proc setPrimarySelectionText*(text: cstring): bool
proc getPrimarySelectionText*(): cstring
proc hasPrimarySelectionText*(): bool

# Расширенный: произвольные MIME-типы:
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

## 17. Свойства (Properties)

Универсальная система ключ-значение для передачи параметров в SDL3 API и хранения пользовательских данных, прикреплённых к объектам.

### Типы значений

```nim
type PropertyType* = enum
  PropertyTypeInvalid, PropertyTypePointer,
  PropertyTypeString, PropertyTypeNumber,
  PropertyTypeFloat, PropertyTypeBoolean
```

### Операции

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

**Пример создания Renderer через Properties:**
```nim
let props = createProperties()
discard setStringProperty(props, PROP_RENDERER_CREATE_NAME_STRING, "vulkan")
discard setNumberProperty(props, PROP_RENDERER_CREATE_PRESENT_VSYNC_NUMBER, 1)
let renderer = createRendererWithProperties(props)
destroyProperties(props)
```

---

## 18. Логирование

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

> **По умолчанию** SDL выводит лог в stderr (Linux/macOS) или через `OutputDebugString` (Windows). Уровень по умолчанию: `DEBUG` и выше для `LOG_CATEGORY_APPLICATION`, `WARN` и выше для остальных категорий.

> **`setLogOutputFunction`** позволяет перенаправить вывод, например в файл:
> ```nim
> proc myLogger(userdata: pointer, category: cint,
>               priority: LogPriority, message: cstring) {.cdecl.} =
>   let f = cast[File](userdata)
>   writeLine(f, $priority & ": " & $message)
>
> let logFile = open("game.log", fmWrite)
> setLogOutputFunction(myLogger, cast[pointer](logFile))
> ```

> **Форматирование через `{.varargs.}`:** функции логирования принимают printf-совместимый формат. В Nim можно передавать `cstring`, `cint`, `cfloat` и т.д. Для Nim-строк используйте `.cstring`: `logInfo(0, "Игрок: %s", playerName.cstring)`.

### Стандартные категории

| Константа | Значение |
|-----------|---------|
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

## 19. Прочее: CPU, системная информация, диалоги

### CPU и SIMD

```nim
proc getNumLogicalCPUCores*(): cint
proc getCPUCacheLineSize*(): cint
proc getSystemRAM*(): cint       # МБ
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

### Системная информация

```nim
proc getPlatform*(): cstring      # "Windows", "Mac OS X", "Linux", ...
proc isTablet*(): bool
proc isTV*(): bool
proc getSystemTheme*(): SystemTheme  # SYSTEM_THEME_LIGHT, _DARK, _UNKNOWN
proc getSandbox*(): Sandbox
```

### Ошибки

```nim
proc setError*(fmt: cstring): bool {.varargs.}
proc getError*(): cstring
proc clearError*(): bool
proc outOfMemory*(): bool
```

### Диалоги файлов

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

> **Диалоги асинхронны.** Функции возвращаются немедленно; результат приходит в `callback` позднее, в главном потоке. Если пользователь отменил диалог, `filelist = nil`.

> **`filter`** в callback — индекс выбранного фильтра из массива `filters` (или `-1` если не применялся / пользователь отменил).

**Пример открытия файла:**
```nim
proc onFileChosen(userdata: pointer, filelist: ptr UncheckedArray[cstring],
                  filter: cint) {.cdecl.} =
  if filelist == nil:
    echo "Отменено"
    return
  var i = 0
  while filelist[i] != nil:
    echo "Выбран файл: ", filelist[i]
    inc i

let filters = [
  DialogFileFilter(name: "PNG изображения", pattern: "*.png"),
  DialogFileFilter(name: "Все файлы", pattern: "*"),
]
showOpenFileDialog(onFileChosen, nil, win, filters, nil, false)
```

### Системный трей

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

### Сообщения

```nim
proc showSimpleMessageBox*(flags: MessageBoxFlags, title, message: cstring,
                           window: Window): bool
proc showMessageBox*(messageboxdata: ptr MessageBoxData,
                     buttonid: var cint): bool
```

### Память

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

### Подсказки (Hints)

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

> **Hints** — строковые переменные конфигурации SDL, устанавливаемые **до** вызова связанной функции. Наиболее употребительные:
>
> | Hint | Значение | Описание |
> |------|----------|----------|
> | `"SDL_RENDER_VSYNC"` | `"1"` | VSync для Renderer (альтернатива `setRenderVSync`) |
> | `"SDL_FRAMEBUFFER_ACCELERATION"` | `"1"` / `"0"` | Аппаратное ускорение framebuffer |
> | `"SDL_MOUSE_RELATIVE_MODE_WARP"` | `"1"` | Warping вместо raw input для relative mouse mode |
> | `"SDL_JOYSTICK_ALLOW_BACKGROUND_EVENTS"` | `"1"` | Принимать события джойстика в фоне |
> | `"SDL_IME_SHOW_UI"` | `"1"` | Показывать UI выбора IME |
> | `"SDL_WINDOWS_DPI_SCALING"` | `"1"` | DPI-осознанность на Windows |
>
> Hints устанавливайте **до** `init()` или соответствующего `createXxx` — позже они могут не иметь эффекта.

### Процессы

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

### Прочие утилиты

```nim
proc openURL*(url: cstring): bool      # открыть URL в браузере
proc getPreferredLocales*(count: var cint): ptr UncheckedArray[ptr Locale]
proc loadObject*(sofile: cstring): SharedObject  # динамическая библиотека
proc loadFunction*(handle: SharedObject, name: cstring): ProcPointer
proc unloadObject*(handle: SharedObject)
```

> **`openURL`** открывает URL в браузере по умолчанию (или ассоциированном приложении). На Wayland/Linux требует XDG-compliant рабочего окружения. Для `file://` — открывает файловый менеджер.

> **`loadObject` / `loadFunction`** — аналог `dlopen` / `dlsym` (Linux) / `LoadLibrary` / `GetProcAddress` (Windows). Используется для загрузки плагинов и факультативных зависимостей (например, проверить наличие библиотеки перед использованием).

**Пример динамической загрузки функции:**
```nim
let lib = loadObject("libmyPlugin.so")
if lib != nil:
  let fn = cast[proc() {.cdecl.}](loadFunction(lib, "pluginInit"))
  if fn != nil:
    fn()
  unloadObject(lib)
```

---

## 20. Полный минимальный пример

Пример создаёт окно 800×600, рисует прямоугольник и обрабатывает закрытие окна и нажатие Escape.

```nim
import sdl3

proc main() =
  # Инициализация
  if not init(INIT_VIDEO):
    echo "SDL_Init error: ", getError()
    return
  defer: sdl3.quit()

  # Окно и рендерер
  var
    window: Window
    renderer: Renderer
  if not createWindowAndRenderer("SDL3 Demo", 800, 600, 0, window, renderer):
    echo "Window/Renderer error: ", getError()
    return
  defer:
    destroyRenderer(renderer)
    destroyWindow(window)

  # Устанавливаем метаданные
  discard setAppMetadata("SDL3 Demo", "1.0", "com.example.demo")

  # Таймер для FPS
  var
    lastTime = getTicks()
    frames = 0u64

  var
    running = true
    event: Event

  while running:
    # --- Обработка событий ---
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

    # --- Рендеринг ---
    discard setRenderDrawColor(renderer, 20, 20, 40, 255)
    discard renderClear(renderer)

    # Синий прямоугольник
    discard setRenderDrawColor(renderer, 0, 120, 215, 255)
    discard renderFillRect(renderer, FRect(x: 100, y: 100, w: 200, h: 150))

    # Белая рамка
    discard setRenderDrawColor(renderer, 255, 255, 255, 200)
    discard renderRect(renderer, FRect(x: 100, y: 100, w: 200, h: 150))

    # Диагональ
    discard setRenderDrawColor(renderer, 255, 80, 0, 255)
    discard renderLine(renderer, 0, 0, 800, 600)

    # Отладочный текст (встроенный 8×8 шрифт)
    discard setRenderDrawColor(renderer, 255, 255, 0, 255)
    discard renderDebugTextFormat(renderer, 10, 10, "FPS: %llu", frames)

    discard renderPresent(renderer)

    inc frames
    # Ограничение FPS ~60
    let
      now = getTicks()
      elapsed = now - lastTime
    if elapsed < 16:
      delay((16 - elapsed).uint32)
    lastTime = getTicks()

main()
```

---

## Приложение: соответствие SDL C → Nim

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

> **Правило именования:** префикс `SDL_` убирается, первая буква становится строчной. `SDL_GetError` → `getError`. Исключения: функции OpenGL (`gL_*`), EGL (`eGL_*`), GPU (`gPU*`), HID (`hid_*`).

> **Изменения SDL2 → SDL3, важные для Nim-пользователей:**
> - `SDL_RenderDrawRect` → `renderRect` (новое имя, отражает отсутствие «Draw»-инфикса во всём API).
> - `SDL_RenderCopy` / `SDL_RenderCopyEx` → `renderTexture` / `renderTextureRotated` (аргументы теперь `FRect`, не `Rect`).
> - `SDL_OpenAudioDevice` теперь принимает `AudioDeviceID` вместо строки имени.
> - `SDL_LockSurface` / `SDL_UnlockSurface` → `lockSurface` / `unlockSurface` (возвращают `bool`).
> - Все типы геометрии рендерера стали float (`FRect`, `FPoint`) вместо int.
> - Инициализация аудио: `SDL_OpenAudioDevice` + `SDL_QueueAudio` (SDL2) заменены на `AudioStream`-API.

---

## Приложение: типичные шаблоны Nim/SDL3

### Nim `defer` для ресурсов SDL

В Nim удобно использовать `defer` для гарантированного освобождения SDL-ресурсов:

```nim
let win = createWindow("App", 800, 600, 0)
if win == nil: quit(1)
defer: destroyWindow(win)

let rend = createRenderer(win, nil)
if rend == nil: quit(1)
defer: destroyRenderer(rend)
# defer выполняется в обратном порядке: сначала destroyRenderer, затем destroyWindow
```

### Загрузка PNG/JPG через SDL_image (отдельная библиотека)

SDL3 сам загружает только BMP. Для PNG/JPG/и т.д. подключают `sdl3_image`:
```nim
import sdl3_image   # отдельный пакет: nimble install sdl3_image

discard imgInit(IMG_INIT_PNG or IMG_INIT_JPG)
defer: imgQuit()

let
  surf = imgLoad("sprite.png")
  tex = createTextureFromSurface(rend, surf)
destroySurface(surf)
```

### Загрузка TTF-шрифтов через SDL_ttf

```nim
import sdl3_ttf  # nimble install sdl3_ttf

discard ttfInit()
defer: ttfQuit()

let
  font = openFont("DejaVuSans.ttf", 24)
  surf = renderTextBlended(font, "Привет!", Color(r: 255, g: 255, b: 255, a: 255))
  tex = createTextureFromSurface(rend, surf)
destroySurface(surf)
# Рендерим tex как обычную текстуру
closeFont(font)
```

### Структура проекта с nimble

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

### Работа с `cstring` и Nim-строками

SDL3-функции принимают и возвращают `cstring`. Преобразования:
```nim
# Nim string → cstring (безопасно в пределах scope)
let title = "Моя игра"
discard createWindow(title.cstring, 800, 600, 0)

# cstring → Nim string (копирование)
let errMsg = $getError()  # оператор $ конвертирует cstring → string

# nil-проверка cstring
let name = getRendererName(rend)
if name != nil:
  echo "Рендерер: ", $name
```

---

*Справочник составлен на основе `sdl3.nim` (SDL 3.1.8). Платформы: Windows (`SDL3.dll`), macOS (`libSDL3.dylib`), Linux (`libSDL3.so`), Emscripten (`libSDL3.so`).*
