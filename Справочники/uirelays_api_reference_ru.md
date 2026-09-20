# uirelays — полный справочник API

Версия: **0.11.0**. Справочник построен по исходному коду `src/uirelays/*.nim`
и охватывает **всё**, что помечено `*` (экспортировано) в каждом модуле:
типы, поля типов, константы, глобальные переменные-релеи и процедуры.
Внутренние (неэкспортированные) детали реализации отмечены отдельно, там,
где это важно для понимания поведения публичного API.

## Оглавление

- [1. `uirelays/coords`](#1-uirelayscoords)
- [2. `uirelays/screen`](#2-uirelaysscreen)
- [3. `uirelays/input`](#3-uirelaysinput)
- [4. `uirelays/backend`](#4-uirelaysbackend)
- [5. `uirelays/layout`](#5-uirelayslayout)
- [6. `uirelays/tinynif`](#6-uirelaystinynif)
- [7. `uirelays` (корневой модуль)](#7-uirelays-корневой-модуль)
- [Алфавитный указатель процедур](#алфавитный-указатель-процедур)

---

## 1. `uirelays/coords`

Геометрия окна, независимая от платформы и от драйвера. Ни от чего другого
в библиотеке не зависит.

### 1.1. Типы

#### `Rect`

```nim
Rect* = object
  x*, y*, w*, h*: int
```

Прямоугольник: левый верхний угол `(x, y)`, ширина `w`, высота `h`. Единицы
измерения — «логические» пиксели драйвера (см. `ScreenLayout.uiScale` в
разделе 2 о том, как это соотносится с физическими пикселями экрана).

```nim
let panel = Rect(x: 10, y: 10, w: 200, h: 100)
```

#### `Point`

```nim
Point* = object
  x*, y*: int
```

Точка на плоскости. Используется в основном как аргумент `contains` и как
результат разбора координат мыши.

#### `GlobalPos`

```nim
GlobalPos* = object
  x*, y*, z*: int
  t*: int
```

Позиция с дополнительными полями `z` (глубина/слой) и `t` (метка времени/такт).
В самой библиотеке используется только как часть `CellHit` из модуля
`layout` — там `pos` хранит координаты клика **относительно** найденной
ячейки. Поля `z` и `t` — задел на будущее (мультитач, история), в текущем
коде библиотеки не заполняются.

### 1.2. Процедуры

#### `rect`

```nim
proc rect*(x, y, w, h: int): Rect
```

Конструктор `Rect` через позиционные аргументы — короче, чем писать
`Rect(x: .., y: .., w: .., h: ..)` целиком.

```nim
let r = rect(0, 0, 640, 480)
fillRect(r, color(30, 30, 46))
```

#### `point`

```nim
proc point*(x, y: int): Point
```

Конструктор `Point`.

```nim
let cursor = point(mouseX, mouseY)
```

#### `contains`

```nim
proc contains*(r: Rect; p: Point): bool
```

`true`, если точка `p` лежит внутри `r`. Интервал **полуоткрытый**:
`x ∈ [r.x, r.x + r.w)`, `y ∈ [r.y, r.y + r.h)` — то есть правая и нижняя
границы прямоугольника точке уже не принадлежат. Это стандартное соглашение
для хит-тестинга: два соседних прямоугольника без зазора никогда не
«делят» одну и ту же граничную точку между собой.

```nim
if buttonRect.contains(point(e.x, e.y)):
  onButtonClicked()
```

---

## 2. `uirelays/screen`

Оконные, шрифтовые и рисующие релеи — самая большая часть публичного API.
Зависит только от `coords`.

### 2.1. Типы значений (value types)

#### `Color`

```nim
Color* = object
  r*, g*, b*, a*: uint8
```

Цвет в формате RGBA, каждый канал — `uint8` (0–255). Создаётся конструктором
`color` (см. ниже), напрямую через литерал `Color(...)` обычно не собирают.

#### `Font`

```nim
Font* = distinct int    ## opaque handle; 0 = invalid
```

Непрозрачный дескриптор открытого шрифта. `0` — недействительный/неудачно
открытый шрифт. Сравнение через `==` определено (`{.borrow.}`), больше
никаких операций над самим значением не предполагается — все действия идут
через процедуры модуля (`openFont`, `drawText`, …).

```nim
var fm: FontMetrics
let f: Font = openFont("", 16, fm)
if f == Font(0):
  echo "не удалось открыть шрифт"
```

#### `Image`

```nim
Image* = distinct int   ## opaque handle; 0 = invalid
```

Непрозрачный дескриптор загруженного изображения. Семантика идентична
`Font`: `0` — «недействительный/не загружен».

#### `FontStyle` / `FontStyles`

```nim
FontStyle* {.pure.} = enum
  bold, italics
FontStyles* = set[FontStyle]
```

Что от начертания шрифта хочет вызывающий код сверх обычного прямого
(regular) начертания. `{}` — обычный прямой шрифт, который умеет открыть
любой драйвер. Остальное — **пожелание**: если у семейства нет жирного или
курсивного начертания (или драйвер не умеет его запросить), библиотека
рисует обычным начертанием — худший случай, стилизованный текст выглядит
как обычный, но никогда не пропадает вовсе.

```nim
let bold = styledFont(font, {FontStyle.bold})
let boldItalic = styledFont(font, {FontStyle.bold, FontStyle.italics})
```

#### `TextExtent`

```nim
TextExtent* = object
  w*, h*: int
```

Размеры прямоугольника, который займёт текст при отрисовке данным шрифтом
(результат `measureText`, а также то, что возвращает `drawText`).

#### `FontMetrics`

```nim
FontMetrics* = object
  ascent*, descent*, lineHeight*: int
```

* `ascent` — высота от базовой линии до верха самых высоких глифов;
* `descent` — глубина от базовой линии до низа самых нижних глифов (у
  символов вроде `g`, `y`);
* `lineHeight` — рекомендованное межстрочное расстояние (то, на сколько
  нужно сдвигать `y` при переходе к следующей строке текста). Это же
  значение отдаёт `fontLineSkip`, и это же значение обычно передают как
  `lineHeight` в `layout.resolve` для расчёта `(lines N)`-ячеек.

```nim
var fm: FontMetrics
let font = openFont("", 16, fm)
echo "Высота строки: ", fm.lineHeight
```

#### `ScreenLayout`

```nim
ScreenLayout* = object
  width*, height*: int
  pitch*: int
  scaleX*, scaleY*: int
  uiScale*: int
  fullScreen*: bool
```

Полное состояние окна на текущий момент, возвращается `createWindow` и
`getWindowLayout`.

| Поле | Значение |
|---|---|
| `width`, `height` | фактический размер окна в единицах, в которых рисует драйвер (может отличаться от запрошенного — см. раздел про HiDPI ниже) |
| `pitch` | служебное поле раскладки памяти буфера кадра (заполняется не всеми драйверами; для типового прикладного кода не нужно) |
| `scaleX`, `scaleY` | сколько физических пикселей устройства драйвер уже кладёт на одну единицу координат. **Чисто информационно** — приложение не должно домножать на это своё собственное рисование |
| `uiScale` | процент, на который приложению следует увеличить размеры шрифтов и «зашитых» пиксельных величин, чтобы они физически выглядели одинаково на любом экране. `100` = плотность уже учтена драйвером/ОС; `200` = приложение должно рисовать вдвое крупнее |
| `fullScreen` | true, если окно открыто в полноэкранном режиме |

```nim
let layout = createWindow(800, 600)
echo "Реальный размер: ", layout.width, "x", layout.height
echo "uiScale = ", layout.uiScale, "%"
```

#### `CursorKind`

```nim
CursorKind* = enum
  curDefault, curArrow, curIbeam, curWait,
  curCrosshair, curHand, curSizeNS, curSizeWE
```

Форма курсора мыши, передаётся в `setCursor`.

| Значение | Типичное применение |
|---|---|
| `curDefault` | системный курсор по умолчанию (передать управление ОС/драйверу) |
| `curArrow` | обычная стрелка |
| `curIbeam` | текстовый ввод («I»-балка) |
| `curWait` | ожидание/загрузка |
| `curCrosshair` | прицельный крест (инструменты рисования/выделения) |
| `curHand` | «рука» — над кликабельным элементом |
| `curSizeNS` | изменение размера по вертикали (↕) |
| `curSizeWE` | изменение размера по горизонтали (↔) |

```nim
setCursor(if hoveringSplitter: curSizeWE else: curArrow)
```

### 2.2. Типы релеев (объекты указателей на процедуры)

Каждый релей — обычный `object`, где каждое поле — `proc {.nimcall.}`.
Драйвер платформы присваивает **весь объект целиком** при инициализации;
прикладной код с этими объектами напрямую не работает — только через
обёртки, перечисленные в разделе 2.4.

#### `WindowRelays`

```nim
WindowRelays* = object
  createWindow*: proc (layout: var ScreenLayout;
                       icon: pointer; iconLen: int) {.nimcall.}
  getWindowLayout*: proc (): ScreenLayout {.nimcall.}
  refresh*: proc () {.nimcall.}
  saveState*: proc () {.nimcall.}
  restoreState*: proc () {.nimcall.}
  setClipRect*: proc (r: Rect) {.nimcall.}
  setCursor*: proc (c: CursorKind) {.nimcall.}
  setWindowTitle*: proc (title: string) {.nimcall.}
```

| Поле | Пояснение |
|---|---|
| `createWindow` | Создаёт и показывает окно. `layout` передаётся по ссылке — драйвер читает запрошенные `width`/`height` и пишет обратно фактические, плюс `scaleX`/`scaleY`/`uiScale`. `icon` — упакованный `_NET_WM_ICON`-подобный набор: `iconLen` значений `uint32` в порядке «ширина, высота, затем пиксели ARGB», либо `nil`. Всё, что окну даётся один раз на всю жизнь (размер, картинка и имя, под которым его знает рабочий стол), даётся именно здесь — имя не параметр, потому что драйвер читает его прямо из исполняемого файла |
| `getWindowLayout` | Текущий размер и масштаб |
| `refresh` | Показать текущий кадр (для двойной буферизации — скопировать задний буфер в окно) |
| `saveState` | Сохранить графическое состояние (сейчас — clip-rect) в стек |
| `restoreState` | Восстановить состояние из стека |
| `setClipRect` | Ограничить область отрисовки прямоугольником |
| `setCursor` | Сменить форму курсора |
| `setWindowTitle` | То, что окно *показывает* (например, имя документа) — меняется часто. Не то же самое, что `WM_CLASS`, который выставляет `createWindow` и который больше ничто не меняет |

#### `FontRelays`

```nim
FontRelays* = object
  openFont*: proc (path: string; size: int; style: FontStyles;
                   metrics: var FontMetrics): Font {.nimcall.}
  closeFont*: proc (f: Font) {.nimcall.}
  getFontMetrics*: proc (f: Font): FontMetrics {.nimcall.}
  measureText*: proc (f: Font; text: string): TextExtent {.nimcall.}
  drawText*: proc (f: Font; x, y: int; text: string;
                   fg, bg: Color): TextExtent {.nimcall.}
  drawMeasuredText*: proc (f: Font; x, y: int; text: string;
                           fg, bg: Color; size: TextExtent) {.nimcall.}
```

| Поле | Пояснение |
|---|---|
| `openFont` | Загрузить шрифт, вернуть дескриптор (`0` при неудаче), заполнить `metrics` |
| `closeFont` | Освободить дескриптор |
| `getFontMetrics` | Вернуть уже посчитанные метрики открытого шрифта |
| `measureText` | Измерить размер строки без отрисовки |
| `drawText` | Отрисовать строку, вернуть её фактический размер |
| `drawMeasuredText` | Необязательное поле-ускоритель: то же, что `drawText`, но для вызывающего кода, который уже сам измерил текст через `measureText` — драйверу не нужно ходить по глиф ам второй раз, чтобы понять, каким по ширине залить фон. Опционально: драйвер, оставивший поле `nil`, будет вызван через обычный `drawText` и нарисует ровно то же, что и раньше — это ускоряющий путь, а не второй способ рисовать |

#### `DrawRelays`

```nim
DrawRelays* = object
  fillRect*: proc (r: Rect; color: Color) {.nimcall.}
  drawLine*: proc (x1, y1, x2, y2: int; color: Color) {.nimcall.}
  drawPoint*: proc (x, y: int; color: Color) {.nimcall.}
  loadImage*: proc (path: string): Image {.nimcall.}
  freeImage*: proc (img: Image) {.nimcall.}
  drawImage*: proc (img: Image; src, dst: Rect) {.nimcall.}
  imageSize*: proc (img: Image): tuple[w, h: int] {.nimcall.}
  blitRGBA*: proc (pixels: ptr UncheckedArray[uint32]; w, h: int;
                   dst: Rect): bool {.nimcall.}
```

| Поле | Пояснение |
|---|---|
| `fillRect` | Залить прямоугольник сплошным цветом |
| `drawLine` | Линия между двумя точками |
| `drawPoint` | Одна точка (пиксель) |
| `loadImage` | Загрузить изображение из файла, `0` при неудаче |
| `freeImage` | Освободить дескриптор изображения |
| `drawImage` | Нарисовать область `src` изображения в прямоугольник назначения `dst` (с масштабированием, если размеры отличаются) |
| `imageSize` | Необязательное поле: собственный размер изображения в его собственных пикселях. Без него нельзя сказать «нарисуй *всё* изображение» — драйвер без этого поля отдаёт `(0, 0)` |
| `blitRGBA` | Необязательное поле: «люк» для готовых RGBA-пикселей (декодер картинок, график, страница PDF), которые нужно просто вывести на поверхность. Без масштабирования, без альфа-блендинга — только клиппинг по `dst`. Возвращает `false`, если поверхность не может принять такие пиксели вовсе |

### 2.3. Глобальные переменные-релеи

```nim
var windowRelays*: WindowRelays
var fontRelays*: FontRelays
var drawRelays*: DrawRelays
```

По умолчанию инициализированы «пустыми» реализациями (не делают ничего /
возвращают нулевые значения). Драйвер платформы **переприсваивает** эти
переменные целиком в своей `initXxxDriver()`. Прикладной код с ними
напрямую не работает.

### 2.4. Константы

```nim
const
  MaxWindowWidth* = -1
  MaxWindowHeight* = -1
```

Передаются вместо конкретного числа в `createWindow`, означают «всё, что
рабочий стол готов дать окну по этому измерению» (для ширины и высоты
соответственно) — экран за вычетом панели задач, Dock, меню и т.п. Это
**не** то же самое, что `fullScreen = true`: окно сохраняет заголовок и
своё место среди других окон, на macOS не переходит в отдельный Space.
Любое из двух измерений можно задать отдельно — окно может занять всю
ширину экрана при фиксированной высоте. Сентинел никогда не «просачивается»
наружу: `ScreenLayout`, возвращённый `createWindow`, всегда содержит
реальный размер в пикселях.

```nim
let layout = createWindow(MaxWindowWidth, 480)  # во всю ширину экрана, высота 480
```

### 2.5. Процедуры-обёртки

#### `createWindow`

```nim
proc createWindow*(requestedW, requestedH: int; fullScreen = false;
                    icon: openArray[uint32] = []): ScreenLayout
```

Создаёт окно и возвращает актуальный `ScreenLayout`. `icon` — набор
изображений в формате «ширина, высота, затем `width*height` пикселей
`0xAARRGGBB`» — иконка панели задач/заголовка окна.

```nim
let layout = createWindow(1024, 768)
let full = createWindow(MaxWindowWidth, MaxWindowHeight, fullScreen = true)
```

#### `getWindowLayout`

```nim
proc getWindowLayout*(): ScreenLayout
```

Текущий размер и масштаб. Стоит перечитывать после того, как окно могло
переместиться на другой монитор — плотность экрана могла измениться.

```nim
let now = getWindowLayout()
if now.uiScale != lastKnownScale:
  reopenFontsForNewScale(now)
```

#### `scaled`

```nim
proc scaled*(layout: ScreenLayout; value: int): int
```

Увеличивает «зашитый» размер шрифта или пиксельную величину под текущий
экран: `value * layout.uiScale div 100`. Целочисленная арифметика на всём
пути — поэтому дисплеи 125% и 150% дают точные значения без накопления
ошибки округления.

```nim
let fontSizePx = layout.scaled(18)   # 18 логических пунктов → физический размер
let padding = layout.scaled(8)
```

#### `refresh`, `saveState`, `restoreState`

```nim
proc refresh*()
proc saveState*()
proc restoreState*()
```

`refresh()` — показать нарисованный кадр. `saveState`/`restoreState` —
парные вызовы вокруг временного ограничения области рисования.

```nim
saveState()
setClipRect(listArea)
drawLongList()
restoreState()
refresh()
```

#### `setClipRect`

```nim
proc setClipRect*(r: Rect)
```

Ограничивает последующую отрисовку прямоугольником `r` — всё, что рисуется
за его пределами, отсекается.

#### `setCursor`

```nim
proc setCursor*(c: CursorKind)
```

Меняет форму курсора мыши. Вызывается каждый кадр по текущему состоянию
наведения — дешёвая операция, специально следить за «изменилось ли»
не требуется.

```nim
setCursor(if overButton: curHand else: curArrow)
```

#### `setWindowTitle`

```nim
proc setWindowTitle*(title: string)
```

```nim
setWindowTitle("документ.txt — Редактор")
```

#### `openFont`

```nim
proc openFont*(path: string; size: int; metrics: var FontMetrics;
               style: FontStyles = {}): Font
```

`path` — путь к файлу шрифта, `""` — моноширинный шрифт платформы по
умолчанию. `size` обычно берётся уже через `scaled(...)`. Заполняет
`metrics`. Стилизованный шрифт, запрошенный сразу здесь (не через
`styledFont`), открывается «напрямую», как отдельный шрифт.

```nim
var fm: FontMetrics
let mono = openFont("", layout.scaled(16), fm)
let boldMono = openFont("", layout.scaled(16), fm, {FontStyle.bold})
```

#### `styledFont`

```nim
proc styledFont*(f: Font; style: FontStyles): Font
```

Тот же шрифт `f` в другом начертании — открывается лениво, при первом
запросе, и закрывается автоматически вместе с `f`. Работает только для
шрифтов, изначально открытых без явного стиля (`style: {}`). Возвращает сам
`f`, если: стиль пустой, `f` не открывался этим модулем, либо драйвер не
смог получить нужное начертание — так что худший исход — прямой текст, а
не отсутствие текста.

```nim
let font = openFont("", 16, fm)
...
let boldVariant = styledFont(font, {FontStyle.bold})
discard drawText(boldVariant, x, y, "Важно!", fg, bg)
```

#### `closeFont`

```nim
proc closeFont*(f: Font)
```

Закрывает шрифт `f` **и все его производные**, полученные через
`styledFont`. Отдельно закрывать варианты не нужно.

```nim
closeFont(font)   # закроет также styledFont(font, {bold}) и т.п., если открывались
```

#### `getFontMetrics`, `fontLineSkip`

```nim
proc getFontMetrics*(f: Font): FontMetrics
proc fontLineSkip*(f: Font): int
```

`fontLineSkip` — короткая форма `getFontMetrics(f).lineHeight`, самая
частая величина, нужная при вёрстке текста построчно.

```nim
let y2 = y1 + fontLineSkip(font)
```

#### `measureText`

```nim
proc measureText*(f: Font; text: string): TextExtent
```

```nim
let extent = measureText(font, "Привет, мир")
let centeredX = (width - extent.w) div 2
```

#### `drawText`

```nim
proc drawText*(f: Font; x, y: int; text: string; fg, bg: Color;
               known = TextExtent()): TextExtent
```

`y` — верх текста, не базовая линия. Возвращает фактический занятый
размер. `known` — необязательный уже посчитанный `measureText` для этой же
строки и шрифта: если передан и драйвер поддерживает быстрый путь, текст
рисуется без повторного измерения.

```nim
let ext = measureText(font, label)
discard drawText(font, x, y, label, fg, bg, known = ext)
```

#### `fillRect`

```nim
proc fillRect*(r: Rect; color: Color)
```

```nim
fillRect(rect(0, 0, width, height), color(30, 30, 46))
```

#### `drawFrame`

```nim
proc drawFrame*(r: Rect; color: Color; width = 1)
```

Рамка толщиной `width`, нарисованная **внутри** `r` (четырьмя `fillRect`).
Ничего не рисует, если `r.w <= 0`, `r.h <= 0` или `width <= 0`; фактическая
толщина ограничивается меньшей стороной прямоугольника.

```nim
drawFrame(panelRect, color(88, 91, 112), width = 2)
```

#### `drawLine`

```nim
proc drawLine*(x1, y1, x2, y2: int; color: Color)
```

```nim
drawLine(0, 40, width, 40, color(60, 60, 80))   # горизонтальная линия-разделитель
```

#### `drawPoint`

```nim
proc drawPoint*(x, y: int; color: Color)
```

#### `loadImage`, `freeImage`

```nim
proc loadImage*(path: string): Image
proc freeImage*(img: Image)
```

```nim
let icon = loadImage("assets/icon.png")
defer: freeImage(icon)
```

#### `drawImage`

```nim
proc drawImage*(img: Image; src, dst: Rect)
```

`src` — область изображения-источника (в его собственных пикселях), `dst`
— куда её нарисовать на экране (с масштабированием при необходимости).

```nim
let (iw, ih) = imageSize(icon)
drawImage(icon, rect(0, 0, iw, ih), rect(10, 10, 32, 32))
```

#### `imageSize`

```nim
proc imageSize*(img: Image): tuple[w, h: int]
```

`(0, 0)`, если драйвер не реализовал соответствующий релей.

#### `blitRGBA`

```nim
proc blitRGBA*(pixels: ptr UncheckedArray[uint32]; w, h: int; dst: Rect): bool
```

Прямой вывод `w × h` уже готовых пикселей (`0x00RRGGBB`, построчно, порядок
байт хоста) в прямоугольник `dst`, без масштабирования — только клиппинг.
`false`, если драйвер не поддерживает это (или поддерживает, но данная
поверхность не может принять пиксели). Предназначено для кода, который сам
умеет генерировать пиксели (декодер изображений, рендерер графиков, вывод
PDF-страницы) и которому от драйвера нужно только «куда положить».

```nim
var buf = newSeq[uint32](w * h)
# ... заполнение buf своим рендерером ...
if not blitRGBA(cast[ptr UncheckedArray[uint32]](addr buf[0]), w, h,
                rect(20, 20, w, h)):
  # fallback: например, залить серым прямоугольником
  fillRect(rect(20, 20, w, h), color(128, 128, 128))
```

#### `color`

```nim
proc color*(r, g, b: uint8; a: uint8 = 255): Color
```

```nim
let accent = color(137, 180, 250)          # непрозрачный
let translucent = color(137, 180, 250, 128) # 50% альфа
```

### 2.6. Операторы

```nim
proc `==`*(a, b: Font): bool {.borrow.}
proc `==`*(a, b: Image): bool {.borrow.}
```

Сравнение дескрипторов на равенство (заимствовано у базового `int`).

```nim
if font == Font(0): echo "шрифт не открылся"
```

---

## 3. `uirelays/input`

События ввода, тайминг и буфер обмена. Не зависит от `screen` — только от
встроенных типов Nim.

### 3.1. Типы

#### `KeyCode`

```nim
KeyCode* = enum
  KeyNone,
  KeyA, KeyB, KeyC, KeyD, KeyE, KeyF, KeyG, KeyH, KeyI, KeyJ,
  KeyK, KeyL, KeyM, KeyN, KeyO, KeyP, KeyQ, KeyR, KeyS, KeyT,
  KeyU, KeyV, KeyW, KeyX, KeyY, KeyZ,
  Key0, Key1, Key2, Key3, Key4, Key5, Key6, Key7, Key8, Key9,
  KeyF1, KeyF2, KeyF3, KeyF4, KeyF5, KeyF6,
  KeyF7, KeyF8, KeyF9, KeyF10, KeyF11, KeyF12,
  KeyEnter, KeySpace, KeyEsc, KeyTab,
  KeyBackspace, KeyDelete, KeyInsert,
  KeyLeft, KeyRight, KeyUp, KeyDown,
  KeyPageUp, KeyPageDown, KeyHome, KeyEnd,
  KeyCapslock, KeyComma, KeyPeriod, KeySlash,
  KeyMinus, KeyEqual, KeyPlus
```

Физические клавиши, независимые от раскладки клавиатуры и от того, что
реально печатается (для печатаемого текста — `TextInputEvent`, не
`KeyCode`). `KeyNone` — «клавиша не определена/не важна». Буквы `KeyA..KeyZ`
соответствуют **позиции** клавиши на клавиатуре QWERTY, а не букве текущей
раскладки — так же, как `e.key` в большинстве низкоуровневых API.

Диапазоны для проверки принадлежности группе (как в `paint.nim`):

```nim
if e.key >= Key1 and e.key <= Key6:
  brushColor = palette[e.key.ord - Key1.ord]
```

#### `EventKind`

```nim
EventKind* = enum
  NoEvent,
  KeyDownEvent, KeyUpEvent, TextInputEvent,
  MouseDownEvent, MouseUpEvent, MouseMoveEvent, MouseWheelEvent,
  WindowResizeEvent, WindowMetricsEvent, WindowCloseEvent,
  WindowFocusGainedEvent, WindowFocusLostEvent,
  QuitEvent
```

| Значение | Когда возникает | Актуальные поля `Event` |
|---|---|---|
| `NoEvent` | «нет события» / значение по умолчанию | — |
| `KeyDownEvent` | клавиша нажата | `key`, `mods` |
| `KeyUpEvent` | клавиша отпущена | `key`, `mods` |
| `TextInputEvent` | введён печатаемый символ (после раскладки/IME) | `text` |
| `MouseDownEvent` | нажата кнопка мыши | `x`, `y`, `button`, `clicks` |
| `MouseUpEvent` | кнопка мыши отпущена | `x`, `y`, `button` |
| `MouseMoveEvent` | движение мыши | `x`, `y` |
| `MouseWheelEvent` | прокрутка колеса | `x`, `y` (направление/дельта) |
| `WindowResizeEvent` | *устаревшее*: только размер изменился | `x`, `y` |
| `WindowMetricsEvent` | размер и/или плотность экрана изменились | `x`, `y`, `scaleX`, `scaleY`, `uiScale` |
| `WindowCloseEvent` | пользователь нажал кнопку закрытия окна | — |
| `WindowFocusGainedEvent` | окно получило фокус | — |
| `WindowFocusLostEvent` | окно потеряло фокус | — |
| `QuitEvent` | системный сигнал завершения (например, выход из сессии) | — |

> Ни один драйвер из этого репозитория больше не генерирует
> `WindowResizeEvent` — он оставлен в перечислении ради обратной
> совместимости с драйверами, которые ещё не обновлены. Новый код должен
> обрабатывать `WindowMetricsEvent` (и не помешает заодно перечислить
> `WindowResizeEvent` в той же ветке `case`, как это делают примеры).

#### `Modifier`

```nim
Modifier* = enum
  ShiftPressed, CtrlPressed, AltPressed, GuiPressed
```

`GuiPressed` — клавиша «Windows»/«Command»/«Super» в зависимости от
платформы.

```nim
if e.key == KeyQ and CtrlPressed in e.mods:
  running = false
```

#### `MouseButton`

```nim
MouseButton* = enum
  LeftButton, RightButton, MiddleButton
```

#### `InputFlag`

```nim
InputFlag* = enum
  WantTextInput   ## показать экранную клавиатуру / включить IME
```

Передаётся набором (`set[InputFlag]`) в `pollEvent`/`waitEvent`. Сейчас
единственный флаг — сообщает драйверу, что сейчас есть текстовое поле в
фокусе (актуально для мобильных/тач-платформ и для IME; десктопные
драйверы вправе его игнорировать).

```nim
let flags = if textFieldFocused: {WantTextInput} else: {}
while pollEvent(e, flags): ...
```

#### `Event`

```nim
Event* = object
  kind*: EventKind
  key*: KeyCode
  mods*: set[Modifier]
  text*: array[4, char]
  x*, y*: int
  scaleX*, scaleY*: int
  uiScale*: int
  button*: MouseButton
  clicks*: int
```

Одна универсальная структура на все виды событий; какие поля значимы —
зависит от `kind` (см. таблицу `EventKind` выше).

| Поле | Смысл |
|---|---|
| `kind` | тип события |
| `key` | код физической клавиши (`KeyDownEvent`/`KeyUpEvent`) |
| `mods` | набор зажатых модификаторов |
| `text` | один UTF-8 codepoint без аллокации (`TextInputEvent`); до 4 байт, остаток — `'\0'` |
| `x`, `y` | позиция мыши, дельта прокрутки, либо новый размер окна — в зависимости от `kind` |
| `scaleX`, `scaleY`, `uiScale` | новые значения плотности экрана (`WindowMetricsEvent`), формат идентичен полям `ScreenLayout` |
| `button` | какая кнопка мыши (`MouseDownEvent`/`MouseUpEvent`) |
| `clicks` | число последовательных кликов подряд (двойной клик = `2`, тройной = `3`); считает сам драйвер |

Извлечение текста из `text: array[4, char]` (как в `todo.nim`):

```nim
proc eventText(text: array[4, char]): string =
  result = ""
  for ch in text:
    if ch == '\0': break
    result.add ch
```

#### `ClipboardRelays`

```nim
ClipboardRelays* = object
  getText*: proc (): string {.nimcall.}
  putText*: proc (text: string) {.nimcall.}
```

#### `InputRelays`

```nim
InputRelays* = object
  pollEvent*: proc (e: var Event; flags: set[InputFlag]): bool {.nimcall.}
  waitEvent*: proc (e: var Event; timeoutMs: int;
                    flags: set[InputFlag]): bool {.nimcall.}
  getTicks*: proc (): int {.nimcall.}
  sleep*: proc (ms: int) {.nimcall.}
  shutdown*: proc () {.nimcall.}
```

| Поле | Пояснение |
|---|---|
| `pollEvent` | неблокирующее чтение следующего события; `false` — очередь пуста |
| `waitEvent` | блокирует до события или истечения `timeoutMs` (`< 0` — бесконечно); обязан продолжать прокачивать очередь ОС во время ожидания |
| `getTicks` | монотонный счётчик миллисекунд |
| `sleep` | сон на `ms` миллисекунд, тоже обязан прокачивать очередь сообщений |
| `shutdown` | закрыть окно, освободить платформенные ресурсы |

### 3.2. Глобальные переменные-релеи

```nim
var clipboardRelays*: ClipboardRelays
var inputRelays*: InputRelays
```

Как и в `screen`, по умолчанию — заглушки; драйвер переприсваивает их
целиком.

### 3.3. Процедуры-обёртки

#### `pollEvent`

```nim
proc pollEvent*(e: var Event; flags: set[InputFlag] = {}): bool
```

Неблокирующий опрос очереди событий. Типичное использование — «слить»
всю очередь за кадр:

```nim
var e = Event()
while pollEvent(e):
  case e.kind
  of QuitEvent, WindowCloseEvent: running = false
  of MouseMoveEvent: mouseX = e.x; mouseY = e.y
  else: discard
```

#### `waitEvent`

```nim
proc waitEvent*(e: var Event; timeoutMs: int = -1;
                flags: set[InputFlag] = {}): bool
```

Блокирующее ожидание — экономит CPU для приложений без постоянной
анимации (например, текстовый редактор, ждущий следующего нажатия).

```nim
var e = Event()
while waitEvent(e, timeoutMs = 1000):   # проснуться и без событий раз в секунду
  handle(e)
  if not waitEvent(e): break
```

#### `getClipboardText`, `putClipboardText`

```nim
proc getClipboardText*(): string
proc putClipboardText*(text: string)
```

```nim
if e.key == KeyV and CtrlPressed in e.mods:
  inputText.add getClipboardText()
if e.key == KeyC and CtrlPressed in e.mods:
  putClipboardText(selectedText)
```

#### `getTicks`

```nim
proc getTicks*(): int
```

```nim
let blinkOn = (getTicks() div 500) mod 2 == 0   # мигание с периодом 1с
```

#### `sleep`

```nim
proc sleep*(ms: int)
```

```nim
sleep(16)   # ~60 кадров в секунду
```

#### `shutdown`

```nim
proc shutdown*()
```

Вызывается один раз в конце `main`, после выхода из главного цикла.

```nim
closeFont(font)
shutdown()
```

---

## 4. `uirelays/backend`

Автоматический выбор платформенного драйвера. Не объявляет собственных
типов — только одну условно определяемую процедуру.

#### `initBackend`

```nim
proc initBackend*()
```

Инициализирует релеи выбранным драйвером. Логика выбора (проверяется в
этом порядке, побеждает первое совпадение):

| Условие компиляции | Драйвер |
|---|---|
| `--define:"features.uirelays.figDrawWindy"` или `-d:figDrawWindy` | FigDraw + Windy |
| `--define:"features.uirelays.figDrawSiwin"` или `-d:figDrawSiwin` | FigDraw + siwin |
| (оба флага FigDraw одновременно) | **ошибка компиляции** — взаимоисключающие |
| `-d:sdl3` | SDL3 |
| `-d:sdl2` | SDL2 |
| `-d:gtk4` | GTK4 |
| `defined(macosx)` (по умолчанию) | Cocoa |
| `defined(windows)` (по умолчанию) | WinAPI |
| `defined(linux)`/`freebsd`/`openbsd`/`netbsd` (по умолчанию) | X11 |
| иначе | SDL3 (универсальный fallback) |

```nim
import uirelays/[coords, screen, input, backend]
initBackend()
let layout = createWindow(800, 600)
```

При `import uirelays` (без подмодулей) `initBackend()` вызывается
автоматически последней строкой корневого модуля — вручную звать не нужно.

---

## 5. `uirelays/layout`

Необязательная надстройка: текстовое NIF-описание разметки окна → таблица
именованных `Rect`. Зависит от `coords` и `tinynif`, не зависит от
`screen`/`input`/`backend` — можно использовать отдельно, даже с другим
рендерером.

### 5.1. Публичные типы

#### `Layout`

```nim
Layout* = object
  error*: string
```

Результат разбора NIF-текста. `error` пусто при успехе; иначе — строка
вида `"3:5: причина"` (позиция и человекочитаемое описание проблемы),
готовая для показа в статус-баре. Внутреннее дерево узлов (`root: Node`)
не экспортируется — доступ к разметке только через процедуры модуля.

```nim
let parsed = parseLayout(spec)
if parsed.error.len > 0:
  echo "Ошибка разметки: ", parsed.error
```

#### `CellHit`

```nim
CellHit* = object
  name*: string
  pos*: GlobalPos
```

Результат `hitTest`: имя ячейки, в которую попала точка, и позиция
**относительно начала координат этой ячейки** (`pos.x`, `pos.y`). Пустая
строка `name`, если ни одна ячейка не задета.

```nim
let hit = cells.hitTest(mouseX, mouseY)
if hit.name == "editor":
  handleEditorClick(hit.pos.x, hit.pos.y)
```

#### `LayoutMetrics`

```nim
LayoutMetrics* = object
  screenW*, screenH*: int
  lineHeight*: int
  padding*: int
  gap*: int
  uiScale*: int
```

Единая «пачка» чисел, от которой зависит перевод разметки в прямоугольники
— передаётся одинаковой в `resolve`, `splitterAt` и `dragTo`, чтобы все три
не могли разойтись во мнениях о том, где что находится (граница, найденная
с одним набором чисел, и подвинутая с другим — сдвинулась бы не туда).

| Поле | Значение |
|---|---|
| `screenW`, `screenH` | размер окна, которое заполняет разметка |
| `lineHeight` | во что переводится `(lines N)` |
| `padding` | добавляется сверху и снизу `(lines N)`-ячейки |
| `gap` | зазор в пикселях между соседними ячейками — граница, через которую просвечивает фон, и одновременно «ручка» для сплиттера |
| `uiScale` | процент; увеличивает `(px N)`-размеры |

```nim
let metrics = LayoutMetrics(screenW: width, screenH: height,
                            lineHeight: fm.lineHeight, padding: 6,
                            gap: 4, uiScale: layout.uiScale)
let cells = parsed.resolve(metrics)
```

#### `BoxPath`

```nim
BoxPath* = seq[int]
```

Путь до узла дерева разметки в виде индексов детей от корня: `@[0, 1]` —
второй ребёнок первого ребёнка корня. По имени такой узел не адресовать —
двигаемые границы лежат между безымянными `rows`/`cols`-контейнерами, а не
между именованными ячейками.

#### `Splitter`

```nim
Splitter* = object
  found*: bool
  parent*: BoxPath
  before*: int
  vertical*: bool
  grab*: int
```

Граница между двумя соседними ячейками — и всего, что нужно, чтобы потом
подвинуть её ещё раз.

| Поле | Значение |
|---|---|
| `found` | `false` — «указатель не на границе» |
| `parent` | путь к контейнеру (`rows`/`cols`), между детьми которого лежит эта граница |
| `before` | индекс ребёнка **перед** границей (слева от неё / выше неё) |
| `vertical` | `true` — граница `rows` (горизонтальная линия, двигается вверх/вниз); `false` — граница `cols` |
| `grab` | на сколько пикселей вглубь границы указатель «взялся» за неё при захвате — нужно, чтобы перетаскивание двигало границу *на столько же*, на сколько сдвинулся указатель, а не «прыгало» к нему в момент касания (ручка шириной в несколько пикселей) |

### 5.2. Внутренняя (неэкспортируемая) константа `MinBox`

```nim
const MinBox = 12   # не экспортируется — упомянуто для понимания поведения dragTo
```

Минимальный размер ячейки в логических пикселях, ниже которого `dragTo` её
не сожмёт. Не часть публичного API, но важна для понимания: перетащить
границу так, чтобы панель схлопнулась до нуля, нельзя — вернуть панель,
пропавшую совсем, можно было бы только правкой файла руками, а так —
никогда не меньше `12` пикселей.

### 5.3. Процедуры

#### `parseLayout` (из строки)

```nim
proc parseLayout*(s: string): Layout
```

Разбирает весь текст `s` как одну NIF-разметку. **Никогда не бросает
исключений** — проверяйте `result.error`.

```nim
const Spec = """
(layout
  (header (lines 2))
  (body)
  (footer (lines 1)))
"""
let parsed = parseLayout(Spec)
doAssert parsed.error.len == 0, parsed.error
```

#### `parseLayout` (из лексера/токена)

```nim
proc parseLayout*(lex: var Lexer; tok: var Token): Layout
```

Вариант для вызывающего кода, который сам владеет лексером `tinynif` —
например, когда файл содержит не только разметку, а более крупную
конфигурацию (`(config (layout ...) (other ...))`). Разбирает
`(layout ...)`, начиная с текущего `tok`, и оставляет `tok`/`lex` сразу
**после** него, чтобы вызывающий код мог продолжить разбор остального
файла тем же лексером.

```nim
var lex = initLexer(configText)
var tok = next(lex)
# ... разбор до (layout ...) ...
let layoutPart = parseLayout(lex, tok)
# ... tok сейчас указывает на то, что идёт сразу после (layout ...) ...
```

#### `resolve` (по `LayoutMetrics`)

```nim
proc resolve*(layout: Layout; m: LayoutMetrics): Table[string, Rect]
```

Главная процедура модуля: превращает разметку в таблицу «имя ячейки →
прямоугольник». Разметка с непустым `error` резолвится в пустую таблицу.

```nim
let cells = parsed.resolve(metrics)
for name, r in cells:
  fillRect(r, panelColor)
```

#### `resolve` (короткая форма с именованными параметрами)

```nim
proc resolve*(layout: Layout; screenW, screenH: int;
              lineHeight: int = 20; padding: int = 6;
              gap: int = 0; uiScale: int = 100): Table[string, Rect]
```

То же самое, без явного создания `LayoutMetrics` — удобно для быстрого
кода и небольших демонстраций.

```nim
let cells = parsed.resolve(width, height, fm.lineHeight, uiScale = win.uiScale)
```

#### `cell`

```nim
proc cell*(layout: Layout; name: string): bool
```

Есть ли в разметке ячейка с именем `name` — не обращаясь к результату
`resolve`.

```nim
if not parsed.cell("sidebar"):
  echo "В текущей разметке нет боковой панели"
```

#### `cellNames`

```nim
proc cellNames*(layout: Layout): seq[string]
```

Все имена ячеек в порядке, в котором они записаны в тексте — по этому
списку приложение обычно строит перечень нужных ему виджетов.

```nim
for name in parsed.cellNames:
  ensureWidgetExists(name)
```

#### `hitTest`

```nim
proc hitTest*(cells: Table[string, Rect]; x, y: int): CellHit
```

```nim
let hit = cells.hitTest(mouseX, mouseY)
setCursor(if hit.name == "divider": curSizeWE else: curArrow)
```

#### `splitterAt`

```nim
proc splitterAt*(layout: Layout; m: LayoutMetrics; x, y: int;
                  slack = 2): Splitter
```

Какую границу «держит» точка `(x, y)`, с запасом `slack` логических
пикселей по обе стороны от зазора (полезно при `gap = 0`, где иначе не за
что «ухватиться» указателем).

```nim
# в обработчике MouseDownEvent:
let s = parsed.splitterAt(metrics, e.x, e.y)
if s.found:
  dragging = s
```

#### `splitterRect`

```nim
proc splitterRect*(layout: Layout; m: LayoutMetrics; s: Splitter): Rect
```

Прямоугольник самого зазора границы `s` — например, чтобы подсветить его
при наведении. Пустой `Rect`, если `s.found = false`.

```nim
if dragging.found:
  fillRect(parsed.splitterRect(metrics, dragging), accentColor)
```

#### `dragTo`

```nim
proc dragTo*(layout: var Layout; m: LayoutMetrics; s: Splitter;
             x, y: int): bool
```

Двигает границу `s` к точке `(x, y)`, возвращает, изменилось ли что-то.
Размеры пишутся обратно в **той же единице**, в которой были написаны
изначально (`px` остаётся `px`, `lines` округляется к целому числу строк,
доли `stretch` продолжают делить остаток).

```nim
# в обработчике MouseMoveEvent, пока кнопка мыши зажата:
if dragging.found:
  discard parsed.dragTo(metrics, dragging, e.x, e.y)
```

#### `splitCell`

```nim
proc splitCell*(layout: var Layout; name, newName: string;
                 asColumn: bool): bool
```

Добавляет ячейку `newName` рядом с существующей `name`: справа
(`asColumn = true`) или снизу. Место берётся только у `name`, поделённое
пополам. `false` и без изменений — если `name` не найдена, `newName` уже
есть, или разметка не разобралась.

```nim
if parsed.splitCell("editor", "preview", asColumn = true):
  echo parsed   # новый текст разметки с добавленной панелью
```

#### `removeCell`

```nim
proc removeCell*(layout: var Layout; name: string): bool
```

Убирает ячейку `name`. `false` — если её нет, либо она последняя
оставшаяся в окне.

```nim
if parsed.removeCell("preview"):
  saveLayoutToFile($parsed)
```

#### `` `$` `` (сериализация)

```nim
proc `$`*(layout: Layout): string
```

Разметка как текст, который её породил бы: `(layout ...)`, два пробела на
уровень вложенности, завершающий перевод строки. Пустая строка — если
разметка не разобралась.

```nim
writeFile("layout.nif", $parsed)
```

---

## 6. `uirelays/tinynif`

Самодостаточный лексер формата NIF (паренthesized-формат: `(тег
дочерний-узел*)`). Используется модулем `layout`, но пригоден и для любого
собственного формата на основе того же скобочного синтаксиса — семантику
тегов лексер сознательно не знает.

### 6.1. Типы

#### `TokenKind`

```nim
TokenKind* = enum
  tkEof, tkError, tkParLe, tkParRi, tkDot,
  tkIdent, tkSymbol, tkSymbolDef,
  tkIntLit, tkCharLit, tkStringLit
```

| Значение | Что это |
|---|---|
| `tkEof` | конец ввода |
| `tkError` | некорректный ввод; причина — в `text` |
| `tkParLe` | `(` — тег указывается сразу в `text` |
| `tkParRi` | `)` |
| `tkDot` | одиночная `.` — «пустой узел» NIF, незаполненный слот |
| `tkIdent` | простое имя, напр. `cell` |
| `tkSymbol` | имя с точкой внутри, напр. `foo.3.mymod` |
| `tkSymbolDef` | то же, но введено через `:` — так NIF помечает место *определения* символа |
| `tkIntLit` | целочисленный литерал (значение в `intVal`) |
| `tkCharLit` | символьный литерал (байт в `intVal`) |
| `tkStringLit` | строковый литерал (значение в `text`, escape уже раскрыты) |

#### `Token`

```nim
Token* = object
  kind*: TokenKind
  text*: string
  intVal*: int64
  line*, col*: int
```

`line`/`col` — позиция начала токена, обе величины 1-based.

#### `Lexer`

```nim
Lexer* = object
```

Тип экспортирован, но его внутренние поля (`input`, `pos`, `line`, `col`)
**не экспортированы** — работать с ним можно только через `initLexer` и
`next`.

### 6.2. Процедуры

#### `initLexer`

```nim
proc initLexer*(input: string; line = 1; col = 1): Lexer
```

`line`/`col` — с какой позиции считать ввод начинающимся; полезно, когда
лексер разбирает не весь файл целиком, а вырезанный из него фрагмент, и
нужно, чтобы номера строк в сообщениях об ошибках совпадали с исходным
файлом.

```nim
var lex = initLexer(fileContents)
```

#### `next`

```nim
proc next*(lex: var Lexer): Token
```

Следующий токен. После конца ввода продолжает возвращать `tkEof`.

```nim
var tok = next(lex)
while tok.kind != tkParRi:
  echo tok
  tok = next(lex)
```

#### `position`

```nim
proc position*(tok: Token): string
```

`"строка:столбец"` — готовая приставка для сообщения об ошибке.

```nim
if tok.kind == tkError:
  echo tok.position, ": ", tok.text
```

#### `` `$` `` (представление токена)

```nim
proc `$`*(tok: Token): string
```

Токен в том виде, в каком он был бы написан — для сообщений вида
«ожидалось …, но встречено …».

```nim
echo "ожидалось ')', но встречено ", $tok
```

### 6.3. Особенности формата (не отдельные процедуры, но часть контракта)

* Ничего не бросает исключений — единственный сигнал ошибки — `tkError`.
* Плавающая точка не поддерживается (`1.5` → ошибка, не усечение).
* Целые литералы: необязательный суффикс `u` в конце принимается и
  игнорируется; поддерживаются десятичные и `0x`/`0X`-шестнадцатеричные.
* Строки/символы: единственный escape — `\` + ровно два hex-символа
  (`\0A` = перевод строки, `\5C` = обратный слэш).
* `#` начинает комментарий до конца строки — добавление сверх «чистого»
  NIF, специально для рукописных файлов.

---

## 7. `uirelays` (корневой модуль)

```nim
import uirelays / [coords, screen, input, backend]
export coords, screen, input

initBackend()
```

Реэкспортирует `coords`, `screen` и `input` (но **не** `backend` — этот
модуль только используется для одноразового вызова `initBackend()` при
импорте) и автоматически инициализирует нативный бэкенд для текущей
платформы. Одна строка `import uirelays` покрывает подавляющее большинство
случаев использования; модуль `layout` в реэкспорт не входит и **всегда**
импортируется отдельно:

```nim
import uirelays
import uirelays/layout
```

---

## Алфавитный указатель процедур

| Процедура | Модуль |
|---|---|
| `blitRGBA` | `screen` |
| `cell` | `layout` |
| `cellNames` | `layout` |
| `closeFont` | `screen` |
| `color` | `screen` |
| `contains` | `coords` |
| `createWindow` | `screen` |
| `dragTo` | `layout` |
| `drawFrame` | `screen` |
| `drawImage` | `screen` |
| `drawLine` | `screen` |
| `drawPoint` | `screen` |
| `drawText` | `screen` |
| `fillRect` | `screen` |
| `fontLineSkip` | `screen` |
| `freeImage` | `screen` |
| `getClipboardText` | `input` |
| `getFontMetrics` | `screen` |
| `getTicks` | `input` |
| `getWindowLayout` | `screen` |
| `hitTest` | `layout` |
| `imageSize` | `screen` |
| `initBackend` | `backend` |
| `initLexer` | `tinynif` |
| `loadImage` | `screen` |
| `measureText` | `screen` |
| `next` | `tinynif` |
| `openFont` | `screen` |
| `parseLayout` (×2) | `layout` |
| `point` | `coords` |
| `pollEvent` | `input` |
| `position` | `tinynif` |
| `putClipboardText` | `input` |
| `rect` | `coords` |
| `refresh` | `screen` |
| `removeCell` | `layout` |
| `resolve` (×2) | `layout` |
| `restoreState` | `screen` |
| `saveState` | `screen` |
| `scaled` | `screen` |
| `setClipRect` | `screen` |
| `setCursor` | `screen` |
| `setWindowTitle` | `screen` |
| `shutdown` | `input` |
| `sleep` | `input` |
| `splitCell` | `layout` |
| `splitterAt` | `layout` |
| `splitterRect` | `layout` |
| `styledFont` | `screen` |
| `waitEvent` | `input` |
| `` `$` `` (Layout) | `layout` |
| `` `$` `` (Token) | `tinynif` |
| `` `==` `` (Font, Image) | `screen` |

---

*Справочник составлен по исходному коду `uirelays` версии 0.11.0 (автор —
Araq, лицензия MIT): `src/uirelays.nim`, `src/uirelays/coords.nim`,
`src/uirelays/screen.nim`, `src/uirelays/input.nim`,
`src/uirelays/backend.nim`, `src/uirelays/layout.nim`,
`src/uirelays/tinynif.nim`.*
