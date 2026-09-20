# uirelays — руководство пользователя

Версия библиотеки: **0.11.0** · Язык: **Nim ≥ 2.0.0** · Лицензия: **MIT**

## Оглавление

1. [Что такое uirelays](#1-что-такое-uirelays)
2. [Установка](#2-установка)
3. [Быстрый старт](#3-быстрый-старт)
4. [Архитектура: концепция «релеев»](#4-архитектура-концепция-релеев)
5. [Базовые типы (`coords`)](#5-базовые-типы-coords)
6. [Окно, шрифты и отрисовка (`screen`)](#6-окно-шрифты-и-отрисовка-screen)
7. [Ввод и буфер обмена (`input`)](#7-ввод-и-буфер-обмена-input)
8. [Выбор бэкенда (`backend`)](#8-выбор-бэкенда-backend)
9. [Менеджер компоновки (`layout`) и формат NIF](#9-менеджер-компоновки-layout-и-формат-nif)
10. [Формат NIF и лексер `tinynif`](#10-формат-nif-и-лексер-tinynif)
11. [Разбор примеров из репозитория](#11-разбор-примеров-из-репозитория)
12. [HiDPI и масштабирование](#12-hidpi-и-масштабирование)
13. [Написание собственного драйвера](#13-написание-собственного-драйвера)
14. [Справочник по всем процедурам](#14-справочник-по-всем-процедурам)
15. [Частые вопросы и советы](#15-частые-вопросы-и-советы)

---

## 1. Что такое uirelays

**uirelays** — нативная UI-библиотека для Nim, построенная на идее **«релеев»**
(relays) — простого варианта dependency injection через глобальные
callback-объекты, состоящие из голых указателей на процедуры (`proc
{.nimcall.}`). Никакой виртуализации, никакого наследования, никаких
аллокаций в куче на каждый вызов — просто набор полей-указателей, которые
драйвер платформы заполняет один раз при старте, а дальше приложение вызывает
обычные обёртки (`fillRect`, `drawText`, `waitEvent`, …), которые
диспетчеризуют вызов через соответствующий релей.

Библиотека поддерживает бэкенды:

| Драйвер | Платформа | Зависимости |
|---|---|---|
| `winapi_driver` | Windows | нет (GDI) |
| `cocoa_driver` | macOS | нет (AppKit) |
| `x11_driver` | Linux/BSD | libX11, libXft |
| `gtk4_driver` | Linux/BSD | GTK4, Cairo, Pango |
| `sdl3_driver` | кроссплатформенный | SDL3, SDL3_ttf |
| `sdl2_driver` | кроссплатформенный | SDL2, SDL2_ttf |
| `figdraw_windy_driver` | кроссплатформенный | FigDraw, Windy |
| `figdraw_siwin_driver` | кроссплатформенный | FigDraw, siwin |

На этой библиотеке написан [focim](https://github.com/Araq/focim) — редактор
кода с интегрированным терминалом (сам редактор в репозиторий uirelays не
входит: виджеты вроде SynEdit — это часть редактора, а не UI-библиотеки).

---

## 2. Установка

### 2.1. Через Nimble

```sh
nimble install
```

или, если пакет ещё не опубликован в официальном реестре, — прямой git-путь /
локальная директория.

### 2.2. Системные зависимости

По умолчанию на **Linux/BSD** используется бэкенд **X11**, на **Windows** —
**WinAPI**, на **macOS** — **Cocoa**. Оба нативных бэкенда для Windows и macOS
не требуют внешних библиотек. Для Linux нужно поставить заголовки X11:

**Fedora** (актуально для вашей системы):

```sh
sudo dnf install libX11-devel libXft-devel
```

**Ubuntu/Debian**:

```sh
sudo apt install libx11-dev libxft-dev
```

#### Альтернативный бэкенд GTK4

```sh
# Fedora
sudo dnf install gtk4-devel pango-devel cairo-devel fontconfig-devel glib2-devel pkgconf-pkg-config

# Ubuntu
sudo apt install libgtk-4-dev libpango1.0-dev libcairo2-dev libfontconfig1-dev libglib2.0-dev pkg-config
```

Сборка с этим бэкендом:

```sh
nim c -d:gtk4 examples/hello.nim
```

#### SDL3 / SDL2

Нужны и системные библиотеки, и Nim-обёртки:

```sh
# Fedora
sudo dnf install SDL3-devel SDL3_ttf-devel
nimble install https://github.com/nim-lang/sdl3

# или SDL2
sudo dnf install SDL2-devel SDL2_ttf-devel
nimble install https://github.com/nim-lang/sdl2
```

Сборка:

```sh
nim c -d:sdl3 examples/hello.nim
nim c -d:sdl2 examples/hello.nim
```

#### FigDraw (Windy / siwin)

Два взаимоисключающих feature-флага Nimble; ставится только выбранная
оконная зависимость:

```sh
atlas install --feature:uirelays.figDrawWindy
nim c --define:"features.uirelays.figDrawWindy" examples/hello.nim

atlas install --feature:uirelays.figDrawSiwin
nim c --define:"features.uirelays.figDrawSiwin" examples/hello.nim
```

Короткие алиасы `-d:figDrawWindy` / `-d:figDrawSiwin` тоже работают, если
соответствующие зависимости уже доступны компилятору.

> Одновременное указание обоих feature-флагов FigDraw — ошибка компиляции
> (`figDrawWindy and figDrawSiwin are mutually exclusive`).

---

## 3. Быстрый старт

Минимальный способ подключить библиотеку:

```nim
import uirelays
```

Этот единственный импорт реэкспортирует модули `coords`, `screen` и `input`
и **автоматически** инициализирует нативный бэкенд для текущей платформы
(вызывается `initBackend()` из модуля `backend`).

Если нужен более тонкий контроль (например, отложенная инициализация или
явный выбор бэкенда в рантайме), импортируйте подмодули напрямую и вызовите
`initBackend()` сами:

```nim
import uirelays/[coords, screen, input, backend]
initBackend()
```

### 3.1. Минимальное приложение

```nim
import uirelays

proc main =
  let layout = createWindow(640, 480)
  var width = layout.width
  var height = layout.height

  var fm = FontMetrics()
  # scaled() учитывает плотность экрана; путь "" — моноширинный шрифт платформы
  let font = openFont("", layout.scaled(18), fm)
  setWindowTitle("Hello uirelays")

  var running = true
  while running:
    var e = Event()
    while pollEvent(e):
      case e.kind
      of QuitEvent, WindowCloseEvent:
        running = false
      of WindowResizeEvent, WindowMetricsEvent:
        width = e.x
        height = e.y
      of KeyDownEvent:
        if e.key == KeyEsc: running = false
      else: discard

    fillRect(rect(0, 0, width, height), color(30, 30, 46))
    discard drawText(font, 20, 20, "Привет, uirelays!",
                      color(205, 214, 244), color(30, 30, 46))

    refresh()
    sleep(16)  # ~60 fps

  closeFont(font)
  shutdown()

main()
```

Компиляция:

```sh
nim c examples/hello.nim
nim c -d:gtk4 examples/hello.nim   # принудительно другой бэкенд
```

Структура каждого приложения на uirelays одна и та же:

1. `createWindow` — открыть окно, получить `ScreenLayout`.
2. `openFont` — открыть шрифт(ы).
3. Главный цикл: `pollEvent`/`waitEvent` → обработка событий → отрисовка →
   `refresh()` → `sleep()`.
4. `closeFont` + `shutdown()` — освободить ресурсы при выходе.

---

## 4. Архитектура: концепция «релеев»

Библиотека делится на **пять групп релеев** — обычных `object`-типов с
полями-`proc`:

| Модуль | Релей | Назначение |
|---|---|---|
| `screen` | `WindowRelays` (`windowRelays`) | Жизненный цикл окна, курсор, clip-rect |
| `screen` | `FontRelays` (`fontRelays`) | Загрузка шрифтов, измерение и отрисовка текста |
| `screen` | `DrawRelays` (`drawRelays`) | Прямоугольники, линии, точки, изображения |
| `input` | `InputRelays` (`inputRelays`) | События, тайминг, завершение работы |
| `input` | `ClipboardRelays` (`clipboardRelays`) | Копирование/вставка |

По умолчанию каждый релей — набор «заглушек», которые ничего не делают (или
возвращают нулевые значения). Драйвер платформы (например,
`x11_driver.nim`) при инициализации **полностью переопределяет** объект
релея своей реализацией:

```nim
proc initX11Driver*() =
  windowRelays = WindowRelays(createWindow: ..., getWindowLayout: ..., ...)
  fontRelays   = FontRelays(openFont: ..., ...)
  drawRelays   = DrawRelays(fillRect: ..., ...)
  inputRelays  = InputRelays(pollEvent: ..., ...)
  clipboardRelays = ClipboardRelays(getText: ..., putText: ...)
```

Прикладной код никогда не обращается к релеям напрямую — он вызывает
удобные обёртки верхнего уровня (`fillRect(...)`, `drawText(...)`,
`waitEvent(...)` и т.д.), которые внутри себя делают
`drawRelays.fillRect(...)`. Благодаря этому:

* приложение не знает, какой драйвер сейчас активен;
* драйвер можно подменить одним `-d:` флагом на этапе компиляции без правки
  прикладного кода;
* не заполненное драйвером поле релея — это не ошибка компиляции, а
  безобидный no-op: приложение просто не увидит эффекта от этой конкретной
  возможности.

---

## 5. Базовые типы (`coords`)

Модуль `uirelays/coords` не имеет зависимостей от платформы — это чистая
геометрия.

```nim
type
  Rect* = object
    x*, y*, w*, h*: int

  Point* = object
    x*, y*: int

  GlobalPos* = object
    x*, y*, z*: int
    t*: int
```

Конструкторы и утилиты:

```nim
proc rect*(x, y, w, h: int): Rect
proc point*(x, y: int): Point
proc contains*(r: Rect; p: Point): bool
```

`contains` проверяет попадание точки в прямоугольник **полуоткрытым**
интервалом (`x >= r.x and x < r.x + r.w`), это стандартное поведение для
хит-тестинга.

---

## 6. Окно, шрифты и отрисовка (`screen`)

### 6.1. Типы

```nim
type
  Color* = object
    r*, g*, b*, a*: uint8

  Font* = distinct int     # opaque handle; 0 = недействителен
  Image* = distinct int    # opaque handle; 0 = недействителен

  FontStyle* {.pure.} = enum
    bold, italics
  FontStyles* = set[FontStyle]

  TextExtent* = object
    w*, h*: int

  FontMetrics* = object
    ascent*, descent*, lineHeight*: int

  ScreenLayout* = object
    width*, height*: int
    pitch*: int
    scaleX*, scaleY*: int
    uiScale*: int
    fullScreen*: bool

  CursorKind* = enum
    curDefault, curArrow, curIbeam, curWait,
    curCrosshair, curHand, curSizeNS, curSizeWE
```

`ScreenLayout` — всё, что нужно знать о текущем состоянии окна:

* `width`, `height` — фактический размер в единицах, в которых рисует
  драйвер;
* `scaleX`, `scaleY` — сколько физических пикселей устройства драйвер уже
  кладёт на одну единицу координат (чисто информационно — приложение **не
  должно** само домножать на это своё рисование);
* `uiScale` — на сколько процентов приложению следует увеличить размеры
  шрифтов и «зашитых» пиксельных величин, чтобы они физически выглядели
  одинаково на любом экране (100 = плотность уже учтена драйвером/ОС; 200 =
  приложение должно рисовать вдвое крупнее). Подробнее — раздел
  [«HiDPI и масштабирование»](#12-hidpi-и-масштабирование).

### 6.2. Окно

```nim
proc createWindow*(requestedW, requestedH: int; fullScreen = false;
                    icon: openArray[uint32] = []): ScreenLayout
proc getWindowLayout*(): ScreenLayout
proc scaled*(layout: ScreenLayout; value: int): int
proc refresh*()
proc saveState*()
proc restoreState*()
proc setClipRect*(r: Rect)
proc setCursor*(c: CursorKind)
proc setWindowTitle*(title: string)
```

Особенности:

* `MaxWindowWidth` / `MaxWindowHeight` (константы, равные `-1`) можно
  передать в `createWindow` вместо конкретного числа, чтобы запросить всё
  доступное пространство рабочего стола по этому измерению (не путать с
  `fullScreen` — окно при этом сохраняет заголовок и место среди других
  окон). Реальный размер всегда возвращается в `ScreenLayout` — сентинел
  наружу не просачивается.
* `icon` — растровая иконка в формате, который ожидает X11
  (`_NET_WM_ICON`): последовательность записей «ширина, высота, затем
  `width*height` пикселей `0xAARRGGBB`». Передаётся в момент создания окна,
  а не отдельным вызовом, потому что нужна ещё *до* показа окна — иначе
  какое-то время будет видна иконка-заглушка рабочего стола. Игнорируется
  бэкендами, которые берут иконку из ресурса PE, из бандла приложения или
  из темы иконок.
* Кем «владеет» окно (для панели задач) определяется не параметром, а
  именем исполняемого файла, которое драйвер сам читает и кладёт в
  `WM_CLASS`; именно это поле сверяет `StartupWMClass` `.desktop`-файла.
* `getWindowLayout()` стоит перечитывать после события перемещения окна на
  другой монитор — плотность экрана могла измениться.
* `setWindowTitle` — это то, что окно *показывает* (например, имя открытого
  документа) и может меняться часто; это не то же самое, что `WM_CLASS`,
  который выставляется один раз в `createWindow` и не меняется.
* `saveState`/`restoreState` — стек графического состояния (сейчас — только
  clip-rect), удобно оборачивать временное ограничение области отрисовки:

```nim
saveState()
setClipRect(listInner)
# ... рисование списка, которое не должно вылезать за границы ...
restoreState()
```

### 6.3. Шрифты

```nim
proc openFont*(path: string; size: int; metrics: var FontMetrics;
               style: FontStyles = {}): Font
proc styledFont*(f: Font; style: FontStyles): Font
proc closeFont*(f: Font)
proc getFontMetrics*(f: Font): FontMetrics
proc fontLineSkip*(f: Font): int
proc measureText*(f: Font; text: string): TextExtent
proc drawText*(f: Font; x, y: int; text: string; fg, bg: Color;
               known = TextExtent()): TextExtent
```

* `path` — путь к файлу шрифта, либо `""` для моноширинного шрифта
  платформы по умолчанию.
* `styledFont(f, {bold})` возвращает **тот же самый** шрифт `f`, открытый
  ранее через `openFont`, но в жирном/курсивном начертании. Такая
  «производная» версия открывается **один раз при первом обращении** и
  закрывается автоматически вместе с базовым шрифтом при вызове
  `closeFont(f)` — отдельно закрывать её не нужно. Если у семейства шрифтов
  нет нужного начертания (или его не умеет запросить драйвер),
  `styledFont` тихо возвращает исходный `f`: в худшем случае текст будет
  прямым, но не исчезнет вовсе.
* `styledFont` работает только для шрифтов, открытых именно через этот
  модуль без явного стиля (`style: {}`) — для шрифта, изначально открытого
  сразу с `{bold}`, функция вернёт его же без изменений.
* `drawText` принимает необязательный параметр `known` — уже посчитанный
  `measureText` для этого же текста и шрифта. Если он передан **и** драйвер
  поддерживает `drawMeasuredText`, отрисовка идёт без повторного обхода
  глифов (не нужно второй раз мерить текст, чтобы понять, какой ширины
  заливать фон). Это именно ускоряющая опция, а не альтернативный способ
  рисования: драйвер без `drawMeasuredText` рисует как обычно, и
  неправильный `known` в этом случае ни на что не повлияет; там же, где
  ускорение есть, неверный `known` даст неверный фон. Передавайте либо то,
  что вернул `measureText` для этой же строки, либо не передавайте вовсе.
* `getFontMetrics(f).lineHeight` (или короче — `fontLineSkip(f)`) — высота
  строки, нужна почти в каждом расчёте компоновки текста.

### 6.4. Отрисовка

```nim
proc fillRect*(r: Rect; color: Color)
proc drawFrame*(r: Rect; color: Color; width = 1)
proc drawLine*(x1, y1, x2, y2: int; color: Color)
proc drawPoint*(x, y: int; color: Color)
proc loadImage*(path: string): Image
proc freeImage*(img: Image)
proc drawImage*(img: Image; src, dst: Rect)
proc imageSize*(img: Image): tuple[w, h: int]
proc blitRGBA*(pixels: ptr UncheckedArray[uint32]; w, h: int; dst: Rect): bool
proc color*(r, g, b: uint8; a: uint8 = 255): Color
```

* `drawFrame` — рамка толщиной `width` пикселей, нарисованная **внутри**
  `r` четырьмя вызовами `fillRect`; ничего сверх того, что уже умеет
  `fillRect`, драйверу не требуется.
* `imageSize` возвращает собственный размер картинки в её собственных
  пикселях — без этого нельзя нарисовать «всё изображение целиком»: прямая
  попытка (`rect(0, 0, dst.w, dst.h)` в качестве `src`) на самом деле
  вырежет только левый верхний угол в форме дырки назначения. Драйвер, не
  реализовавший это поле, отдаёт `(0, 0)` — приложению, которому нужно
  соотношение сторон, придётся узнавать его другим способом.
* `blitRGBA` — «аварийный люк» для всего, для чего у драйвера нет
  отдельного релея: декодер картинок, график, страница PDF — что угодно,
  способное выдать готовые пиксели. Передаётся `w * h` уже готовых
  непрозрачных пикселей построчно, каждый в формате `0x00RRGGBB` в
  порядке байт хоста (композитинг прозрачности — забота вызывающего кода,
  поверхность ничего не блендит и верхний байт игнорирует). Масштабирования
  нет: `w`/`h` — то, что ляжет на экран как есть, а `dst.w`/`dst.h` только
  обрезают лишнее; если нужен другой размер — ресайзить нужно заранее,
  вызывающий код лучше знает, миниатюра это или печатный лист. Возвращает
  `false`, если поверхность вообще не может принять такие пиксели (странный
  формат либо окна ещё нет) — это не «ошибка, которую можно исправить», а
  сигнал «нарисуй что-то другое», и для данной поверхности ответ всегда
  один и тот же.

---

## 7. Ввод и буфер обмена (`input`)

### 7.1. Типы событий

```nim
type
  KeyCode* = enum
    KeyNone, KeyA..KeyZ, Key0..Key9, KeyF1..KeyF12,
    KeyEnter, KeySpace, KeyEsc, KeyTab,
    KeyBackspace, KeyDelete, KeyInsert,
    KeyLeft, KeyRight, KeyUp, KeyDown,
    KeyPageUp, KeyPageDown, KeyHome, KeyEnd,
    KeyCapslock, KeyComma, KeyPeriod, KeySlash,
    KeyMinus, KeyEqual, KeyPlus

  EventKind* = enum
    NoEvent,
    KeyDownEvent, KeyUpEvent, TextInputEvent,
    MouseDownEvent, MouseUpEvent, MouseMoveEvent, MouseWheelEvent,
    WindowResizeEvent, WindowMetricsEvent, WindowCloseEvent,
    WindowFocusGainedEvent, WindowFocusLostEvent,
    QuitEvent

  Modifier* = enum
    ShiftPressed, CtrlPressed, AltPressed, GuiPressed

  MouseButton* = enum
    LeftButton, RightButton, MiddleButton

  InputFlag* = enum
    WantTextInput   # показать экранную клавиатуру / включить IME

  Event* = object
    kind*: EventKind
    key*: KeyCode
    mods*: set[Modifier]
    text*: array[4, char]   # TextInputEvent: один UTF-8 codepoint, без alloc
    x*, y*: int             # позиция мыши, дельта прокрутки или новый размер окна
    scaleX*, scaleY*: int   # WindowMetricsEvent
    uiScale*: int           # WindowMetricsEvent
    button*: MouseButton
    clicks*: int            # число последовательных кликов (двойной клик = 2)
```

Важные нюансы:

* `WindowMetricsEvent` — актуальное, «живое» событие: несёт и новый размер
  (`e.x`, `e.y`), и текущий масштаб (`e.scaleX`, `e.scaleY`, `e.uiScale`).
  Генерируется при изменении **либо** размера, **либо** плотности экрана —
  так что приложению, слушающему только это событие, достаточно одной
  ветки `case`. `WindowResizeEvent` — старое, «размер-только» событие,
  оставленное в перечислении для необновлённых драйверов; ни один драйвер
  из этого репозитория его больше не генерирует, но обрабатывать стоит оба
  (см. примеры выше — `WindowResizeEvent, WindowMetricsEvent` в одной
  ветке).
* `TextInputEvent` — **отдельное** от `KeyDownEvent` событие: одно нажатие
  клавиши может породить и то, и другое. Для полей ввода текста
  подписывайтесь именно на `TextInputEvent` и передавайте флаг
  `WantTextInput` в `pollEvent`/`waitEvent`, когда поле ввода в фокусе (это
  включает экранную клавиатуру/IME там, где она есть):

```nim
let flags = if focus == ComposerFocus: {WantTextInput} else: {}
while pollEvent(e, flags):
  case e.kind
  of TextInputEvent:
    if focus == ComposerFocus:
      inputText.add eventText(e.text)   # e.text — 4 байта UTF-8, обрежьте по '\0'
  ...
```

* `MouseWheelEvent` использует `e.y` для направления прокрутки (`+1` —
  вверх, `-1` — вниз).
* Двойные/тройные клики (`e.clicks`) отслеживаются самим драйвером.

### 7.2. Релеи и обёртки

```nim
proc pollEvent*(e: var Event; flags: set[InputFlag] = {}): bool
proc waitEvent*(e: var Event; timeoutMs: int = -1;
                flags: set[InputFlag] = {}): bool
proc getTicks*(): int
proc sleep*(ms: int)
proc shutdown*()
proc getClipboardText*(): string
proc putClipboardText*(text: string)
```

* `pollEvent` — неблокирующий, забирает следующее событие из очереди,
  `false` — событий больше нет. Обычно вызывается в цикле `while pollEvent(e):
  ...` внутри каждого кадра.
* `waitEvent` — блокирует поток до прихода события или истечения
  `timeoutMs` (`-1` — ждать бесконечно). Полезен для приложений без
  анимации, которым не нужно рисовать 60 кадров в секунду вхолостую —
  экономит CPU/батарею.
* `getTicks()` — монотонный счётчик миллисекунд, удобен для миганий (курсор
  ввода, как в `todo.nim`: `(getTicks() div 500) mod 2 == 0`).
* `sleep(ms)` — по контракту драйвер обязан продолжать «прокачивать»
  очередь событий платформы во время сна, чтобы ОС не посчитала приложение
  зависшим.
* `shutdown()` — закрывает окно и освобождает все ресурсы платформы; должен
  вызываться в конце `main`.

---

## 8. Выбор бэкенда (`backend`)

Модуль `uirelays/backend` автоматически выбирает и инициализирует нужный
драйвер (`initBackend()`), опираясь на переданные флаги компиляции. Порядок
проверки:

1. `--define:"features.uirelays.figDrawWindy"` или `-d:figDrawWindy`
2. `--define:"features.uirelays.figDrawSiwin"` или `-d:figDrawSiwin`
   (флаги `figDrawWindy`/`figDrawSiwin` взаимоисключающие — сочетание даёт
   ошибку компиляции)
3. `-d:sdl3`
4. `-d:sdl2`
5. `-d:gtk4`
6. иначе — платформа по умолчанию: `macosx` → Cocoa, `windows` → WinAPI,
   `linux`/`freebsd`/`openbsd`/`netbsd` → X11, всё остальное → SDL3 как
   «универсальный» fallback.

Если вы импортируете `uirelays` целиком, `initBackend()` вызывается за вас
автоматически (последняя строка `src/uirelays.nim`). Если работаете с
подмодулями напрямую (как `paint.nim` в примерах) — вызовите его сами:

```nim
import uirelays/[coords, screen, input, backend]
initBackend()
```

---

## 9. Менеджер компоновки (`layout`) и формат NIF

Модуль `uirelays/layout` не является обязательным — это отдельная,
опциональная надстройка поверх `coords`, которая превращает текстовое
описание разметки окна в набор именованных `Rect`.

### 9.1. Синтаксис

Разметка описывается деревом вложенных скобочных выражений (диалект
формата **NIF**, см. [раздел 10](#10-формат-nif-и-лексер-tinynif)):

```
(layout
  (toolbar (lines 2))
  (cols
    (sidebar (px 250))
    (editor))
  (status (lines 1)))
```

Правила:

* `(layout ...)` — корень, всегда единственный внешний тег; его дети
  укладываются **сверху вниз**, как если бы это был `rows`.
* `(rows ...)` — располагает детей сверху вниз.
* `(cols ...)` — располагает детей слева направо.
* `rows` и `cols` можно вкладывать друг в друга произвольно — это заменяет
  все частные случаи компоновки.
* Любой другой тег — **лист**, то есть виджет: `resolve` кладёт под этим
  именем один `Rect`. Анонимных ячеек не бывает — ячейка, в которой ничего
  не рисуют, не стоит того, чтобы её называть.
* Слова `layout`, `rows`, `cols`, `px`, `lines`, `stretch` зарезервированы и
  не могут быть именами виджетов.

Ячейка может указать свой размер вдоль оси, по которой её делит родитель
(высоту — внутри `rows`, ширину — внутри `cols`):

| Запись | Значение |
|---|---|
| `(px 250)` | 250 **логических** пикселей; `resolve` домножает их на `uiScale`, так что одна и та же разметка описывает одно и то же окно и на 4K-ноутбучной панели, и на мониторе 96 dpi |
| `(lines 5)` | `5 * lineHeight` плюс отступ (`padding`) сверху и снизу |
| `(stretch 2)` | две доли того, что осталось после вычета фиксированных размеров |

Если размер не указан — по умолчанию `(stretch 1)`.

Комментарии начинаются с `#` и идут до конца строки — это нестандартное для
NIF, но удобное для рукописных файлов дополнение.

### 9.2. Разбор и получение прямоугольников

```nim
type
  Layout* = object
    error*: string    # пусто, если разбор успешен; иначе "строка:столбец: причина"

  LayoutMetrics* = object
    screenW*, screenH*: int
    lineHeight*: int
    padding*: int
    gap*: int
    uiScale*: int

proc parseLayout*(s: string): Layout
proc resolve*(layout: Layout; m: LayoutMetrics): Table[string, Rect]
proc resolve*(layout: Layout; screenW, screenH: int;
              lineHeight: int = 20; padding: int = 6;
              gap: int = 0; uiScale: int = 100): Table[string, Rect]
proc cell*(layout: Layout; name: string): bool
proc cellNames*(layout: Layout): seq[string]
```

`parseLayout` **никогда не бросает исключений** — единственный сигнал об
ошибке разбора это непустое `layout.error`, короткая однострочная строка
вида `"3:5: two cells are called 'editor'"`, которую удобно выводить прямо в
строку статуса приложения. Разбор, упавший на первой ошибке, дальше
результатов не даёт: `resolve` для такой раскладки вернёт пустую таблицу.

```nim
const LayoutSpec = """
(layout
  (toolbar (px 30))
  (cols
    (sidebar (px 200))
    (divider (px 4))
    (editor))
  (status (lines 1)))
"""

let parsed = parseLayout(LayoutSpec)
doAssert parsed.error.len == 0, parsed.error

let cells = parsed.resolve(width, height, fm.lineHeight, uiScale = win.uiScale)
for name, r in cells:
  fillRect(r, panelBg)
```

`uiScale` в `resolve` — то же самое поле, что и `ScreenLayout.uiScale`: оно
домножает `(px N)`-размеры в тексте разметки (единственные величины здесь, с
которыми вызывающий код сам ничего сделать не может). Всё остальное, что
передаётся в `resolve` (высота строки, отступы, зазор), уже в собственной
единице драйвера — если вы масштабируете свой шрифт, масштабируйте вместе с
ним и `padding`, и `gap`.

### 9.3. Хит-тестинг

```nim
type
  CellHit* = object
    name*: string
    pos*: GlobalPos

proc hitTest*(cells: Table[string, Rect]; x, y: int): CellHit
```

Возвращает, в какую именованную ячейку попала точка `(x, y)`, и позицию
относительно её начала координат (`pos.x`, `pos.y` — уже локальные для
ячейки).

### 9.4. Сплиттеры: перетаскивание границ мышью

Идея: зазор (`gap`) между соседними ячейками, который `resolve` и так
оставляет, — это уже готовая «ручка» для перетаскивания. Ничего
дополнительно рисовать не нужно (хотя можно закрасить зазор явным цветом,
как в `layout_demo.nim`).

```nim
type
  BoxPath* = seq[int]
  Splitter* = object
    found*: bool
    parent*: BoxPath
    before*: int
    vertical*: bool
    grab*: int

proc splitterAt*(layout: Layout; m: LayoutMetrics; x, y: int;
                  slack = 2): Splitter
proc splitterRect*(layout: Layout; m: LayoutMetrics; s: Splitter): Rect
proc dragTo*(layout: var Layout; m: LayoutMetrics; s: Splitter;
             x, y: int): bool
```

* `splitterAt` — какую границу «держит» указатель в точке `(x, y)`, с
  запасом `slack` логических пикселей по обе стороны от зазора (полезно,
  когда `gap = 0`, но границу всё равно нужно за что-то «поймать»).
  `s.found = false`, если под указателем нет ни одной границы.
* `splitterRect` — прямоугольник самого зазора (например, чтобы подсветить
  его при наведении).
* `dragTo` — двигает найденную границу к текущей позиции указателя и
  сообщает, изменилось ли что-то. Размеры при этом пишутся обратно **в той
  же единице**, в которой были написаны изначально: ячейка в `(px N)`
  остаётся в пикселях, ячейка в `(lines N)` округляется к целому числу
  строк, а «резиновые» (`stretch`) ячейки продолжают делить оставшееся
  место в тех же долях. Минимальный размер ячейки — `12` логических
  пикселей (константа `MinBox`), меньше перетащить нельзя — иначе панель
  пропадёт совсем и вернуть её можно будет только вручную правкой файла.

Типичный цикл обработки перетаскивания:

```nim
var dragging = Splitter(found: false)

# в MouseDownEvent:
dragging = parsed.splitterAt(metrics, e.x, e.y)

# в MouseMoveEvent, пока кнопка мыши зажата:
if dragging.found:
  discard parsed.dragTo(metrics, dragging, e.x, e.y)

# в MouseUpEvent:
dragging = Splitter(found: false)
```

### 9.5. Сериализация обратно в текст

```nim
proc `$`*(layout: Layout): string
```

Выводит разметку в точности в том виде, в каком её написал бы человек: один
`(layout ...)`, отступ 2 пробела на уровень вложенности, завершающий
перевод строки. Раскладка, которая не разобралась (`error.len > 0`), не
выводит ничего — записывать нечего.

Благодаря этому приложение, которое хранит свою разметку в файле, получает
мышью-перетаскиваемые панели, **не** держа второй копии размеров нигде
больше: что подвинул указатель — то и написано в файле.

### 9.6. Рост и сжатие дерева: добавление и удаление панелей

```nim
proc splitCell*(layout: var Layout; name, newName: string;
                 asColumn: bool): bool
proc removeCell*(layout: var Layout; name: string): bool
```

* `splitCell` кладёт новую ячейку `newName` рядом с существующей `name` —
  справа от неё, если `asColumn = true`, снизу — иначе. Место берётся
  **только** у `name` (пополам, в той единице, в которой она была
  написана), поэтому больше ничто в окне не двигается. Возвращает `false` и
  ничего не меняет, если `name` не найдена, `newName` уже существует, или
  разметка не разобралась.
* `removeCell` убирает ячейку `name` из дерева; если контейнер после этого
  остаётся с одним-единственным ребёнком, он «складывается» — выживший
  ребёнок занимает место и размер родителя. Возвращает `false`, если ячейка
  не найдена или она последняя оставшаяся в окне (окно без единой ячейки
  некуда было бы вписать обратно в файл).
* `cellNames(layout)` — список всех имён ячеек в порядке, в котором они
  записаны; по нему приложение узнаёт, какие виджеты ему нужно создать.

Поскольку разбиение/удаление — это правка дерева, а дерево — это файл,
приложение, порождающее и закрывающее панели, ничего не хранит о них само:
что лежит в окне — то и написано в разметке, а ещё одна вручную вписанная
ячейка — это ещё одна панель.

---

## 10. Формат NIF и лексер `tinynif`

`uirelays/tinynif` — самодостаточный (без внешних зависимостей) лексер
формата **NIF**: дерево — это `(тег дочерний-узел*)`, всё остальное —
атомарный токен. Модуль сознательно **не знает семантики тегов** — теги
остаются строками, и разбор их значения (как это делает `layout.nim` для
`rows`/`cols`/`px`/…) — забота вызывающего кода.

```nim
type
  TokenKind* = enum
    tkEof, tkError, tkParLe, tkParRi, tkDot,
    tkIdent, tkSymbol, tkSymbolDef,
    tkIntLit, tkCharLit, tkStringLit

  Token* = object
    kind*: TokenKind
    text*: string
    intVal*: int64
    line*, col*: int

proc initLexer*(input: string; line = 1; col = 1): Lexer
proc next*(lex: var Lexer): Token
proc position*(tok: Token): string   # "line:col"
```

Пример ручного разбора:

```nim
var lex = initLexer(src)
var tok = next(lex)
while tok.kind != tkParRi:
  ...
  tok = next(lex)
```

Особенности формата, о которых стоит помнить:

* Токенизация никогда не «бросает» исключений — некорректный ввод даёт
  `tkError` с человекочитаемой причиной в `text` и позицией в `line`/`col`.
* Плавающая точка **не поддерживается** — `1.5` даёт ошибку, а не тихое
  усечение.
* У целых литералов нет типовых суффиксов кроме необязательной `u` в конце
  — она принимается, но никак не влияет на значение.
* Строковый escape — только `\` и ровно две шестнадцатеричные цифры
  (`\0A` — перевод строки, `\5C` — сам обратный слэш).
* `#` начинает комментарий до конца строки — это добавление uirelays поверх
  «чистого» NIF, специально для файлов, которые правит человек.
* `.` сам по себе — специальный токен `tkDot`, «пустой узел» NIF, означающий
  незаполненный слот.

Этот модуль напрямую в прикладном коде обычно не нужен — им пользуется
`layout.nim`, а собственный формат конфигурации имеет смысл строить поверх
`tinynif`, только если ваш формат — это тоже скобочное NIF-дерево с
собственным набором тегов.

---

## 11. Разбор примеров из репозитория

В каталоге `examples/` — четыре законченных приложения, от простого к
сложному.

### 11.1. `hello.nim` — минимальное окно

Открывает окно, рисует текст и цветные прямоугольники, обрабатывает клавиши
и клик мышью. Хорошая отправная точка, чтобы увидеть главный цикл целиком.

```sh
nim c examples/hello.nim
```

### 11.2. `paint.nim` — простая рисовалка

Демонстрирует **явный** импорт подмодулей вместо единого `import uirelays`
и, соответственно, ручной вызов `initBackend()`:

```nim
import uirelays/[coords, screen, input, backend]
...
initBackend()
```

Функциональность: клик и перетаскивание — рисование, правый клик — очистка
холста, колесо мыши — изменение размера кисти, клавиши `1`–`6` — выбор
цвета из палитры. Хранит все мазки в `seq[Stroke]` и просто перерисовывает
их каждый кадр — простейшая, но рабочая модель немедленного режима (immediate
mode).

### 11.3. `layout_demo.nim` — демонстрация менеджера компоновки

Показывает связку `uirelays` + `uirelays/layout`: разбирает NIF-описание,
резолвит его в прямоугольники на каждый кадр (`parsed.resolve(width,
height, fm.lineHeight, uiScale = win.uiScale)`), подсвечивает ячейку под
курсором через `hitTest`, перерисовывается при изменении размера окна.

```sh
nim c examples/layout_demo.nim
```

### 11.4. `todo.nim` — список задач

Самый развёрнутый пример: список дел с чекбоксами, удалением, переключением
фокуса по Tab между полем ввода и списком, прокруткой колесом мыши,
клавиатурной навигацией (стрелки, Enter, Space, Backspace/Delete),
обрезкой длинного текста многоточием (`truncateText`, с бинарным поиском по
рунам, а не по байтам — важно для корректной работы с UTF-8), миганием
каретки через `getTicks()`, вставкой из буфера обмена (`Ctrl+V`).

Обратите внимание на паттерн повторного `resolve` до и после обработки
событий в одном кадре — второй вызов нужен, потому что событие (например,
изменение размера окна) могло случиться *внутри* обработки, и следующий
рендер должен опираться на уже актуальные прямоугольники:

```nim
var cells = parsedLayout.resolve(width, height, fm.lineHeight)
# ... обработка событий, которая может изменить width/height ...
cells = parsedLayout.resolve(width, height, fm.lineHeight)
# ... отрисовка по актуальным cells ...
```

Компиляция под HiDPI явно через SDL3:

```sh
nim c --path:../src -d:sdl3 -o:todo-highdpi todo.nim
```

---

## 12. HiDPI и масштабирование

Один и тот же коэффициент масштаба означает **разное** на разных платформах
— и это единственная причина, по которой в `ScreenLayout` их два, а не один:

| Поле | Смысл |
|---|---|
| `scaleX`, `scaleY` | сколько физических пикселей устройства драйвер **уже** кладёт на единицу координат — чисто информационно, приложение не должно домножать на это своё рисование |
| `uiScale` | на сколько процентов приложению нужно **самому** увеличить шрифты и «зашитые» пиксельные размеры |

Пример разницы: Cocoa отдаёт координаты в *points* и рендерит в 2×
back-store, поэтому 16pt шрифт уже физически правильного размера —
`scaleX = 2`, `uiScale = 100`. X11 отдаёт «сырые» пиксели устройства и
ничего сам не масштабирует — тот же самый `16` на панели 192 dpi будет
вдвое меньше нужного: `scaleX = 1`, `uiScale = 200`.

Как считают `uiScale` разные драйверы:

| Драйвер | Единица координат | Источник плотности | Что репортит |
|---|---|---|---|
| `x11_driver` | пиксели устройства | ресурс `Xft.dpi` | `scaleX = 1`, `uiScale = dpi * 100 / 96` |
| `winapi_driver` | пиксели устройства | `GetDpiForWindow` | `scaleX = 1`, `uiScale = dpi * 100 / 96` |
| `cocoa_driver` | points | `backingScaleFactor` | `scaleX = 1` или `2`, `uiScale = 100` |
| `gtk4_driver` | логические пиксели | `gtk_widget_get_scale_factor` | `scaleX = множитель`, `uiScale = 100` |
| `sdl3_driver` | пиксели устройства | отношение физического размера к логическому | `scaleX = 1`, `uiScale` = это отношение |
| `sdl2_driver` | пиксели устройства | нет (не умеет определять) | `scaleX = 1`, `uiScale = 100` |
| `figdraw_*_driver` | логические пиксели | `contentScale` | `scaleX` = округлённый масштаб, `uiScale = 100` |

**Практическое правило**: приложение, которое домножает свои размеры
шрифтов через `layout.scaled(size)` (то есть на `uiScale`), корректно
работает на любом из этих драйверов и получает чёткие, не «размазанные»
глифы — потому что шрифт растеризуется сразу нужного физического размера, а
не берётся меньшим и растягивается.

```nim
let layout = createWindow(640, 480)
let font = openFont("", layout.scaled(18), fm)   # 18 "логических" пунктов
```

То же самое верно и для `(px N)` внутри разметки `layout`-модуля — `resolve`
принимает `uiScale` отдельным параметром именно затем, чтобы масштабировать
эти зашитые числа так же, как масштабируется шрифт.

Размер, который вы просите у `createWindow`, — тоже в единице конкретного
драйвера, и он может вернуть не совсем то, что вы просили: на дисплее 200%
`sdl3_driver` превращает запрошенные `1100` в окно `2200` физических
пикселей (SDL принимает запрос в логических единицах), а `x11_driver` даст
ровно запрошенные `1100` пикселей устройства. Поэтому **всегда** читайте
фактический размер из `layout.width`/`layout.height`, а не полагайтесь на
то, что запрос был выполнен буквально.

`sdl2_driver` — единственный драйвер, который в принципе не может узнать
плотность корректно (`SDL_GetDisplayDPI` берёт данные из того же
ненадёжного «физического размера», а на Windows SDL2 вообще не
DPI-aware) — для HiDPI-экранов используйте SDL3.

---

## 13. Написание собственного драйвера

Если целевая платформа не покрыта готовыми драйверами, можно написать
свой — модуль, который заполняет все пять глобальных объектов релеев и
экспортирует единственную процедуру `initMyDriver*()`.

### 13.1. Скелет

```nim
# mydriver.nim
import uirelays/[coords, screen, input]

# ... процедуры реализации ...

proc initMyDriver*() =
  windowRelays = WindowRelays(...)
  fontRelays   = FontRelays(...)
  drawRelays   = DrawRelays(...)
  inputRelays  = InputRelays(...)
  clipboardRelays = ClipboardRelays(...)
```

Каждое поле релея нужно заполнить явно. Незаполненные поля остаются
безобидными заглушками из стандартных значений — драйвер соберётся и
запустится, просто не сделает ничего для нереализованных частей.

### 13.2. О чём важно помнить при реализации

* **Двойная буферизация.** Все процедуры отрисовки пишут в закадровый
  буфер (pixmap/bitmap/texture); `refresh()` копирует его на видимую
  поверхность окна; при изменении размера буфер пересоздаётся под новые
  размеры. Это убирает мерцание и упрощает модель рендеринга.
* **Трансляция событий.** `KeyDownEvent`/`KeyUpEvent` — платформенные коды
  клавиш в `KeyCode`, модификаторы — в `e.mods`. `TextInputEvent` —
  отдельное событие с UTF-8 codepoint в `e.text[0..3]` (одно нажатие клавиши
  может дать и `KeyDownEvent`, и `TextInputEvent`). Мышь — координаты
  клиентской области в `e.x`/`e.y`, для `MouseDownEvent` — `e.button` и
  `e.clicks` (двойные/тройные клики отслеживаются самим драйвером).
  Прокрутка — `MouseWheelEvent` с `e.y` как направлением (+1 вверх, −1
  вниз). `WindowMetricsEvent` — генерировать при изменении размера **или**
  плотности (не только размера!), с новыми `e.x`/`e.y` и `e.scaleX`/
  `e.scaleY`/`e.uiScale`. `WindowCloseEvent` — по клику на кнопку закрытия
  (**не** уничтожать окно самому — решение остаётся за приложением).
  `QuitEvent` — по системным сигналам завершения.
* **Плотность экрана.** См. таблицу в разделе 12 — не репортите физическую
  плотность в `uiScale`, если уже масштабируете рисование сами (иначе текст
  станет вдвое крупнее, чем просили). Плотность, зашитую X-сервером,
  использовать нельзя напрямую — Xwayland всегда отдаёт плоские 96 dpi
  независимо от реального монитора, поэтому `x11_driver` читает ресурс
  `Xft.dpi`, который пишет само окружение рабочего стола.
* **Путь к шрифту → имя гарнитуры.** `openFont` получает путь к файлу;
  многим нативным API нужно имя гарнитуры, а не путь: GDI — через
  `AddFontResourceExW` + `CreateFontW`; Xft/X11 — через строку-паттерн
  fontconfig (`"DejaVu Sans Mono:pixelsize=15"`); SDL_ttf принимает путь
  напрямую, преобразований не требует.
* **Регистрация в `backend.nim`.** Чтобы драйвер выбирался автоматически:

```nim
elif defined(myplatform):
  import drivers/my_driver
  proc initBackend*() = initMyDriver()
```

  либо пользователь просто импортирует ваш драйвер и вызывает
  `initMyDriver()` напрямую, минуя `backend.nim` вовсе. Опциональные
  Nimble-бэкенды стоит закрывать сгенерированным feature-флагом (как
  `defined(features.uirelays.figDrawWindy)`), чтобы их зависимости не
  тянулись в обычную сборку под конкретную платформу.

### 13.3. Чек-лист перед тем как считать драйвер готовым

- [ ] Реализованы все 5 групп релеев (либо неиспользуемые осознанно
      оставлены заглушками)
- [ ] Всё рисование идёт через двойной буфер, показ — на `refresh()`
- [ ] Платформенные события транслируются в `Event`
- [ ] Закрытие окна не уничтожает его само по себе
- [ ] `sleep()` и `waitEvent()` продолжают прокачивать очередь сообщений
- [ ] Двойные/тройные клики отслеживаются в `MouseDownEvent`
- [ ] Пути к файлам шрифтов конвертируются в имена гарнитур платформы
- [ ] Проверено: окно появляется, текст рисуется, клики мыши регистрируются,
      клавиатурный ввод работает, буфер обмена работает, ресайз работает

---

## 14. Справочник по всем процедурам

### `uirelays/coords`

| Процедура | Сигнатура |
|---|---|
| `rect` | `(x, y, w, h: int): Rect` |
| `point` | `(x, y: int): Point` |
| `contains` | `(r: Rect; p: Point): bool` |

### `uirelays/screen`

| Процедура | Сигнатура |
|---|---|
| `createWindow` | `(requestedW, requestedH: int; fullScreen = false; icon: openArray[uint32] = []): ScreenLayout` |
| `getWindowLayout` | `(): ScreenLayout` |
| `scaled` | `(layout: ScreenLayout; value: int): int` |
| `refresh` / `saveState` / `restoreState` | `()` |
| `setClipRect` | `(r: Rect)` |
| `setCursor` | `(c: CursorKind)` |
| `setWindowTitle` | `(title: string)` |
| `openFont` | `(path: string; size: int; metrics: var FontMetrics; style: FontStyles = {}): Font` |
| `styledFont` | `(f: Font; style: FontStyles): Font` |
| `closeFont` | `(f: Font)` |
| `getFontMetrics` | `(f: Font): FontMetrics` |
| `fontLineSkip` | `(f: Font): int` |
| `measureText` | `(f: Font; text: string): TextExtent` |
| `drawText` | `(f: Font; x, y: int; text: string; fg, bg: Color; known = TextExtent()): TextExtent` |
| `fillRect` | `(r: Rect; color: Color)` |
| `drawFrame` | `(r: Rect; color: Color; width = 1)` |
| `drawLine` | `(x1, y1, x2, y2: int; color: Color)` |
| `drawPoint` | `(x, y: int; color: Color)` |
| `loadImage` / `freeImage` | `(path: string): Image` / `(img: Image)` |
| `drawImage` | `(img: Image; src, dst: Rect)` |
| `imageSize` | `(img: Image): tuple[w, h: int]` |
| `blitRGBA` | `(pixels: ptr UncheckedArray[uint32]; w, h: int; dst: Rect): bool` |
| `color` | `(r, g, b: uint8; a: uint8 = 255): Color` |

### `uirelays/input`

| Процедура | Сигнатура |
|---|---|
| `pollEvent` | `(e: var Event; flags: set[InputFlag] = {}): bool` |
| `waitEvent` | `(e: var Event; timeoutMs: int = -1; flags: set[InputFlag] = {}): bool` |
| `getClipboardText` / `putClipboardText` | `(): string` / `(text: string)` |
| `getTicks` | `(): int` |
| `sleep` | `(ms: int)` |
| `shutdown` | `()` |

### `uirelays/backend`

| Процедура | Сигнатура |
|---|---|
| `initBackend` | `()` — выбор и инициализация драйвера по флагам компиляции |

### `uirelays/layout`

| Процедура | Сигнатура |
|---|---|
| `parseLayout` | `(s: string): Layout` |
| `resolve` | `(layout: Layout; m: LayoutMetrics): Table[string, Rect]` (и удобный overload с именованными параметрами) |
| `cell` | `(layout: Layout; name: string): bool` |
| `cellNames` | `(layout: Layout): seq[string]` |
| `hitTest` | `(cells: Table[string, Rect]; x, y: int): CellHit` |
| `splitterAt` | `(layout: Layout; m: LayoutMetrics; x, y: int; slack = 2): Splitter` |
| `splitterRect` | `(layout: Layout; m: LayoutMetrics; s: Splitter): Rect` |
| `dragTo` | `(layout: var Layout; m: LayoutMetrics; s: Splitter; x, y: int): bool` |
| `splitCell` | `(layout: var Layout; name, newName: string; asColumn: bool): bool` |
| `removeCell` | `(layout: var Layout; name: string): bool` |
| `` `$` `` | `(layout: Layout): string` — сериализация обратно в NIF-текст |

### `uirelays/tinynif`

| Процедура | Сигнатура |
|---|---|
| `initLexer` | `(input: string; line = 1; col = 1): Lexer` |
| `next` | `(lex: var Lexer): Token` |
| `position` | `(tok: Token): string` |
| `` `$` `` | `(tok: Token): string` |

---

## 15. Частые вопросы и советы

**Как выбрать бэкенд под конкретную задачу?**
На Fedora/Linux по умолчанию используется X11 — он самый лёгкий по
зависимостям (`libX11-devel libXft-devel`) и достаточен для большинства
десктопных приложений. GTK4-бэкенд (`-d:gtk4`) имеет смысл, если приложению
важна нативная интеграция с темами GNOME/GTK (в т. ч. под Wayland через
XWayland-независимый путь GTK4). SDL3 стоит выбирать, если нужна
предсказуемая работа на HiDPI-дисплеях (см. раздел 12) или кроссплатформенная
сборка без правки кода под каждую ОС.

**Почему после `pollEvent` иногда не хватает одного кадра до применения
нового размера окна?**
Разметку (`layout.resolve`) и любые вычисления, зависящие от `width`/`height`,
нужно пересчитывать **после** цикла обработки событий, а не до него — иначе
в кадре, где пришёл `WindowMetricsEvent`, отрисовка будет опираться на
устаревшие размеры. См. пример из `todo.nim` в разделе 11.4.

**Нужно ли закрывать шрифты, полученные через `styledFont`?**
Нет — они закрываются автоматически вместе с базовым шрифтом при вызове
`closeFont` на `f`, из которого они были получены.

**Как ограничить область отрисовки (например, список с прокруткой)?**
`saveState()` → `setClipRect(rect)` → рисование → `restoreState()`. Не
забывайте закрывать `saveState`/`restoreState` парой — иначе стек
графического состояния драйвера рассинхронизируется с ожиданиями остального
кода.

**Что делать, если разметка (`layout`) не разбирается?**
Проверяйте `parsed.error` сразу после `parseLayout` — строка формата
`"строка:столбец: причина"` человекочитаема и годится для прямого показа
пользователю (например, в строке статуса редактора конфигурации).

**Можно ли использовать `uirelays/layout` без остальной библиотеки uirelays?**
Да — модуль `layout` зависит только от `coords` и `tinynif`, никаких
платформенных вызовов в нём нет. Его можно использовать и для расчёта
компоновки в собственном рендерере, не связанном с `screen`/`input`.

---

*Руководство составлено на основе исходного кода репозитория `uirelays`
(версия 0.11.0, автор — Araq, лицензия MIT) и документации из `README.md`
и `doc/drivers.md`.*
