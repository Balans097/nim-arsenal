# bigints — справочник по библиотеке

Версия библиотеки: **1.1.0** · Лицензия: **MIT** · Авторы: Dennis Felsing, narimiran · Требуется: **Nim ≥ 1.6.20**

`bigints` — чистая (без C-зависимостей, без GMP) реализация целых чисел произвольной точности на Nim. Библиотека даёт один тип `BigInt` и набор операций со стандартным синтаксисом Nim: арифметику, сравнения, битовые операции с семантикой дополнительного кода, преобразования из/в строки в системах счисления от 2 до 36, теорию чисел (НОД, обратный элемент, модульное возведение в степень) и итераторы. Отдельный модуль `bigints/random` генерирует случайные `BigInt` в заданном диапазоне.

Во всех примерах ниже используется префиксная запись вызовов: `initBigInt(5)`, `toString(a, 16)`, `pow(x, 3)`.

---

## Содержание

1. [Установка и подключение](#1-установка-и-подключение)
2. [Устройство типа BigInt](#2-устройство-типа-bigint)
3. [Создание значений](#3-создание-значений)
4. [Преобразование в строку и в целое](#4-преобразование-в-строку-и-в-целое)
5. [Сравнение](#5-сравнение)
6. [Арифметика](#6-арифметика)
7. [Битовые операции и сдвиги](#7-битовые-операции-и-сдвиги)
8. [Теория чисел](#8-теория-чисел)
9. [Итераторы и последовательности](#9-итераторы-и-последовательности)
10. [Случайные числа: модуль bigints/random](#10-случайные-числа-модуль-bigintsrandom)
11. [Исключения](#11-исключения)
12. [Производительность и ограничения](#12-производительность-и-ограничения)
13. [Рецепты](#13-рецепты)
14. [Примеры из репозитория](#14-примеры-из-репозитория)
15. [Тесты и сборка](#15-тесты-и-сборка)
16. [Сводный указатель API](#16-сводный-указатель-api)

---

## 1. Установка и подключение

Установка через nimble:

```
nimble install https://github.com/nim-lang/bigints
```

Либо зависимость в `.nimble`-файле проекта:

```nim
requires "bigints >= 1.1.0"
```

Подключение основного модуля и модуля случайных чисел:

```nim
import bigints          # BigInt и вся арифметика
import bigints/random   # rand(...) для BigInt (по желанию)
```

Если вы используете `std/options` (а он понадобится для `toInt`), импортируйте его отдельно: `import std/options`.

Пакет не рассчитан на 32-битные платформы. Внутри лимбы (машинные слова числа) имеют размер 32 бита, а промежуточные результаты считаются в `uint64`/`int64`.

---

## 2. Устройство типа BigInt

```nim
type
  BigInt* = object
    limbs: seq[uint32]   # little-endian: limbs[0] — младшие 32 бита
    isNegative: bool
```

Число хранится в форме «знак и модуль». Модуль — последовательность 32-битных «лимбов», младший идёт первым. Поля приватные, поэтому напрямую с ними работать нельзя: только через API. Инвариант библиотеки: у ненулевого числа старший лимб не равен нулю, а у нуля длина `limbs` не превышает единицы.

`BigInt` — обычный value-тип: присваивание копирует содержимое, объекты не разделяют память. Неинициализированная переменная `var a: BigInt` корректно ведёт себя как ноль (это закреплено регрессионным тестом), так что `a * b` для такой переменной даёт `0`, а не падает с `IndexDefect`.

Все публичные операции объявлены как `func`, то есть не имеют побочных эффектов; их можно вызывать из `func`, а также на этапе компиляции (`const`, `static`).

---

## 3. Создание значений

### `initBigInt` из целых чисел

```nim
func initBigInt*[T: int8|int16|int32](val: T): BigInt
func initBigInt*[T: uint8|uint16|uint32](val: T): BigInt
func initBigInt*(val: int64): BigInt
func initBigInt*(val: uint64): BigInt
template initBigInt*(val: int): BigInt      # int64 на 64-битных платформах
template initBigInt*(val: uint): BigInt     # uint64 на 64-битных платформах
func initBigInt*(val: BigInt): BigInt       # копия
```

Конструктор принимает значения любого стандартного целого типа. Отрицательные значения обрабатываются через ручное дополнение до двух, так что `initBigInt(low(int64))` работает без переполнения.

```nim
let a = initBigInt(42)
let b = initBigInt(-7'i8)
let c = initBigInt(high(uint64))       # 18446744073709551615
let d = initBigInt(a)                  # независимая копия
```

Обратите внимание: неявного преобразования из `int` в `BigInt` нет. Выражение `a + 1` не скомпилируется, нужно писать `a + initBigInt(1)`.

### `initBigInt` из строки

```nim
func initBigInt*(str: string, base: range[2..36] = 10): BigInt
```

Разбирает строку в заданной системе счисления. Допустим необязательный знак `+` или `-` в начале. Цифры выше девяти обозначаются буквами `a`–`z`, регистр не важен. Между цифрами можно ставить подчёркивания как разделители (`"1_000_000"`), но число не может начинаться или заканчиваться на `_`. Префиксы вроде `0x` здесь **не** поддерживаются: строка `"0xff"` при `base = 16` вызовет `ValueError`, потому что `x` не является шестнадцатеричной цифрой. Для префиксов есть литералы `'bi` (см. ниже).

Если в строке встретился символ, недопустимый в данной системе счисления, пустая строка или одинокий знак, выбрасывается `ValueError`.

```nim
let a = initBigInt("1234")                    # 1234
let b = initBigInt("1234", base = 8)          # 668
let c = initBigInt("-ff", 16)                 # -255
let d = initBigInt("1_000_000")               # 1000000
let e = initBigInt("z", 36)                   # 35

doAssertRaises(ValueError):
  discard initBigInt("2", 2)                  # цифра 2 недопустима в двоичной системе
```

Для оснований 2, 4, 8, 16 и 32 разбор идёт побитово и работает за линейное время. Для остальных оснований строка обрабатывается блоками максимальной длины, помещающейся в `uint32`, и время растёт квадратично с длиной входа.

### `initBigInt` из последовательности лимбов

```nim
func initBigInt*(vals: sink seq[uint32], isNegative = false): BigInt
```

Низкоуровневый конструктор. Первый элемент последовательности — младший лимб. Значение равно `vals[0] + vals[1]·2³² + vals[2]·2⁶⁴ + …`. Ведущие нулевые лимбы автоматически отбрасываются.

```nim
let x = initBigInt(@[10'u32, 2'u32])                 # 10 + 2·2^32 = 8589934602
let y = initBigInt(@[1'u32], isNegative = true)      # -1
```

### Литералы `'bi`

```nim
proc `'bi`*(s: string): BigInt
```

Доступны начиная с Nim 1.5. Пользовательский числовой суффикс позволяет записывать большие числа прямо в коде, включая шестнадцатеричные и двоичные формы:

```nim
let a = 123'bi
let b = 0xFF'bi                       # 255
let c = 0b1011'bi                     # 11
let d = -123456789012345678'bi        # унарный минус применяется к литералу
let e = 0X123456789ABCDEF'bi          # регистр префикса не важен

assert $b == "255"
```

Поддерживаются десятичная, шестнадцатеричная (`0x`/`0X`) и двоичная (`0b`/`0B`) формы. Восьмеричного префикса `0o` нет; для восьмеричных чисел используйте `initBigInt("17", 8)`.

Кроме того, `const` можно инициализировать и строкой:

```nim
const N = "44077431694086786329".initBigInt   # из примера pollard_rho.nim
```

---

## 4. Преобразование в строку и в целое

### `toString` и `$`

```nim
func toString*(a: BigInt, base: range[2..36] = 10): string
func `$`*(a: BigInt): string          # то же, что toString(a, 10)
```

`toString` формирует строковое представление в любой системе счисления от 2 до 36 без префиксов (`0x`, `0b` не добавляются). Цифры `a`–`z` выводятся строчными буквами. У отрицательных чисел перед цифрами стоит `-`. Ноль всегда даёт `"0"`. Как и при разборе, для степеней двойки перевод линейный, для остальных оснований квадратичный.

```nim
let a = initBigInt(55)
assert toString(a) == "55"
assert toString(a, 2) == "110111"
assert toString(a, 16) == "37"
assert toString(-a, 16) == "-37"
assert $pow(initBigInt(2), 100) == "1267650600228229401496703205376"
```

Оператор `$` делает `echo` и интерполяцию `fmt` естественными: `echo a` выведет число в десятичной записи.

### `toInt`

```nim
func toInt*[T: SomeInteger](x: BigInt): Option[T]
```

Безопасно переводит `BigInt` в стандартный целый тип. Если значение не помещается в `T`, возвращается `none(T)`, иначе `some(значение)`. Для беззнаковых типов любое отрицательное число даёт `none`. Тип `T` указывается в квадратных скобках.

```nim
import std/options

let a = initBigInt(44)
let b = initBigInt(130)

assert toInt[int8](a) == some(44'i8)
assert toInt[int8](b) == none(int8)        # 130 > 127
assert toInt[uint8](b) == some(130'u8)
assert toInt[int](b) == some(130)
assert toInt[uint8](initBigInt(-1)) == none(uint8)

let n = get(toInt[int](initBigInt(12345)))  # 12345; get бросит, если это none
```

Граничные значения обрабатываются точно: `low(int64)` и `high(int64)` возвращаются как `some`, а следующее за ними число даёт `none`.

Конвертации в `float` в библиотеке нет.

---

## 5. Сравнение

```nim
func `==`*(a, b: BigInt): bool
func `<`*(a, b: BigInt): bool
func `<=`*(a, b: BigInt): bool
```

Экспортируются три оператора. Остальные (`!=`, `>`, `>=`) Nim выводит автоматически, так что они тоже работают. Сравнение учитывает знак и корректно обрабатывает ноль с «отрицательным» флагом.

```nim
let a = initBigInt(5)
let b = initBigInt(3)
let c = initBigInt(2)

assert a == b + c
assert b != c
assert b < a and b > c
assert a >= b
```

Сравнения с обычными целыми (`a == 5`) не экспортируются. Литерал нужно предварительно обернуть: `a == initBigInt(5)`. Функция `hash` для `BigInt` не определена, поэтому напрямую использовать `BigInt` как ключ `Table` или элемент `HashSet` нельзя; в таких случаях ключом делают строку `$a`. Сортировка `seq[BigInt]` стандартным `sort` работает, поскольку опирается на `<` и `==`.

---

## 6. Арифметика

### Сложение, вычитание, умножение

```nim
func `+`*(a, b: BigInt): BigInt
func `-`*(a, b: BigInt): BigInt
func `-`*(a: BigInt): BigInt          # унарный минус
func `*`*(a, b: BigInt): BigInt
template `+=`*(a: var BigInt, b: BigInt)
template `-=`*(a: var BigInt, b: BigInt)
template `*=`*(a: var BigInt, b: BigInt)
```

Составные операторы присваивания реализованы как шаблоны `a = a + b`, поэтому оптимизации «на месте» не даёт: каждая операция создаёт новый объект. Умножение классическое, по школьному алгоритму, сложность O(n·m) по числу лимбов.

```nim
let a = initBigInt(421)
let b = initBigInt(200)
assert a * b == initBigInt(84200)
assert -a == initBigInt(-421)

var s = initBigInt(5)
s += initBigInt(2)                  # 7
s *= initBigInt(10)                 # 70
s -= initBigInt(1)                  # 69
```

### `abs`

```nim
func abs*(a: BigInt): BigInt
```

Возвращает модуль числа: `abs(initBigInt(-12)) == initBigInt(12)`.

### `pow`

```nim
func pow*(x: BigInt, y: Natural): BigInt
```

Возводит `x` в неотрицательную целую степень `y` двоичным возведением. Показатель — обычный `Natural`, а не `BigInt`. `pow(x, 0)` равен единице, в том числе для `x = 0`.

```nim
assert pow(initBigInt(2), 10) == initBigInt(1024)
assert pow(initBigInt(-3), 3) == initBigInt(-27)
echo pow(initBigInt(2), 100)        # 1267650600228229401496703205376
```

### Целочисленное деление: `div`, `mod`, `divmod`

```nim
func `div`*(a, b: BigInt): BigInt
func `mod`*(a, b: BigInt): BigInt
func divmod*(a, b: BigInt): tuple[q, r: BigInt]
```

Здесь важная деталь: деление **округляет вниз** (floor), как в Python, а не к нулю, как обычный `div` для `int` в Nim. Остаток `mod` получает знак делителя, а тождество `a == (a div b) * b + (a mod b)` выполняется всегда. При делении на ноль выбрасывается `DivByZeroDefect`. Если нужны и частное, и остаток, вызывайте `divmod`: алгоритм при этом выполняется один раз.

```nim
let a = initBigInt(17)
let b = initBigInt(5)

assert a div b == initBigInt(3)          and a mod b == initBigInt(2)
assert (-a) div b == initBigInt(-4)      and (-a) mod b == initBigInt(3)
assert a div (-b) == initBigInt(-4)      and a mod (-b) == initBigInt(-3)
assert (-a) div (-b) == initBigInt(3)    and (-a) mod (-b) == initBigInt(-2)

assert divmod(a, b) == (initBigInt(3), initBigInt(2))
```

Если нужен неотрицательный остаток независимо от знаков (например, в модульной арифметике с положительным модулем), при положительном делителе `mod` уже даёт значение из `[0, b-1]`. Для отрицательного делителя остаток будет неположительным.

Деление реализовано алгоритмом D Кнута (с особой веткой для однолимбового делителя).

### `inc`, `dec`, `succ`, `pred`

```nim
func inc*(a: var BigInt, b: int = 1)
func dec*(a: var BigInt, b: int = 1)
func succ*(a: BigInt, b: int = 1): BigInt
func pred*(a: BigInt, b: int = 1): BigInt
```

Прибавление и вычитание обычного `int` без необходимости оборачивать его в `BigInt`. Шаг может быть отрицательным. `succ` и `pred` не меняют аргумент, а возвращают новое значение.

```nim
var a = initBigInt(15)
inc(a)                     # 16
inc(a, 7)                  # 23
dec(a, 5)                  # 18
assert succ(a, 2) == initBigInt(20)
assert pred(a) == initBigInt(17)
```

---

## 7. Битовые операции и сдвиги

Битовые операции работают так, будто отрицательные числа записаны в дополнительном коде **бесконечной разрядности**. Поэтому `-1` — это «все единицы», и `-1 and x` равно `x`.

```nim
func `not`*(a: BigInt): BigInt
func `and`*(a, b: BigInt): BigInt
func `or`*(a, b: BigInt): BigInt
func `xor`*(a, b: BigInt): BigInt
func `shl`*(x: BigInt, y: Natural): BigInt
func `shr`*(x: BigInt, y: Natural): BigInt
```

Побитовое отрицание подчиняется тождеству `not a == -(a + 1)`.

```nim
assert (not initBigInt(5)) == initBigInt(-6)
assert (initBigInt(-1) and initBigInt(0xFF)) == initBigInt(255)
assert (initBigInt(-8) or initBigInt(3)) == initBigInt(-5)
assert (initBigInt(6) xor initBigInt(3)) == initBigInt(5)
```

Сдвиг влево `shl` эквивалентен умножению на `2^y` и сохраняет знак. Сдвиг вправо `shr` арифметический: для отрицательных чисел он эквивалентен делению на `2^y` с округлением вниз (floor). Если сдвиг превосходит длину числа, результат — ноль. Величина сдвига — обычный `Natural`.

```nim
let a = initBigInt(24)
assert a shl 2 == initBigInt(96)
assert a shr 2 == initBigInt(6)
assert initBigInt(-5) shr 1 == initBigInt(-3)      # floor(-2.5)
assert (initBigInt(1) shl 100) == pow(initBigInt(2), 100)
```

Отдельной функции для подсчёта битов нет, но ближайший аналог — `fastLog2` (см. ниже).

---

## 8. Теория чисел

### `gcd`

```nim
func gcd*(a, b: BigInt): BigInt
```

Наибольший общий делитель по бинарному алгоритму. Результат неотрицателен, знаки аргументов игнорируются. `gcd(0, x)` равен `abs(x)`.

```nim
assert gcd(initBigInt(54), initBigInt(24)) == initBigInt(6)
assert gcd(initBigInt(-54), initBigInt(24)) == initBigInt(6)
```

### `invmod`

```nim
func invmod*(a, modulus: BigInt): BigInt
```

Обратный элемент к `a` по модулю `modulus` (расширенный алгоритм Евклида). Результат лежит в интервале `[1, modulus-1]`. Модуль должен быть строго положительным. Отрицательное `a` допустимо: оно предварительно приводится по модулю. Нулевое `a` отвергается.

Если обратного элемента не существует (`gcd(a, modulus) ≠ 1`), выбрасывается `ValueError`. Нулевой модуль или нулевое `a` дают `DivByZeroDefect`, отрицательный модуль — `ValueError`.

```nim
assert invmod(initBigInt(3), initBigInt(7)) == initBigInt(5)      # 3·5 = 15 ≡ 1 (mod 7)

doAssertRaises(ValueError):
  discard invmod(initBigInt(2), initBigInt(4))                    # gcd = 2, обратного нет
```

### `powmod`

```nim
func powmod*(base, exponent, modulus: BigInt): BigInt
```

Модульное возведение в степень: `base^exponent mod modulus`. Результат лежит в `[0, modulus-1]`. Показатель степени — `BigInt`, причём допускается отрицательный: тогда сначала берётся обратный элемент `invmod(base, modulus)`. Вычисление идёт двоичным методом с приведением по модулю на каждом шаге, поэтому промежуточные значения не разрастаются. Модуль обязан быть строго положительным, иначе `ValueError` (для нуля — `DivByZeroDefect`). При `modulus == 1` результат равен нулю.

```nim
assert powmod(initBigInt(2), initBigInt(3), initBigInt(7)) == initBigInt(1)      # 8 mod 7

# отрицательный показатель: 3^(-1) mod 7 = 5
assert powmod(initBigInt(3), initBigInt(-1), initBigInt(7)) == initBigInt(5)
```

### `fastLog2`

```nim
func fastLog2*(a: BigInt): int
```

Целая часть двоичного логарифма модуля числа, то есть индекс старшего установленного бита. Для нуля возвращается `-1`. Работает за O(1), так как смотрит лишь на старший лимб. Удобно для оценки размера числа в битах: длина в битах равна `fastLog2(a) + 1` для `a ≠ 0`.

```nim
assert fastLog2(initBigInt(1000)) == 9       # 512 <= 1000 < 1024
assert fastLog2(initBigInt(-1000)) == 9      # знак игнорируется
assert fastLog2(initBigInt(0)) == -1
```

---

## 9. Итераторы и последовательности

```nim
iterator countup*(a, b: BigInt, step: int32 = 1): BigInt      # от a до b включительно
iterator countdown*(a, b: BigInt, step: int32 = 1): BigInt    # от a вниз до b включительно
iterator `..`*(a, b: BigInt): BigInt                          # a, a+1, …, b
iterator `..<`*(a, b: BigInt): BigInt                         # a, a+1, …, b-1
```

Итераторы работают в цикле `for` и позволяют перебирать диапазоны `BigInt` так же, как обычные целые. Шаг — небольшое целое типа `int32`.

```nim
for i in initBigInt(1) .. initBigInt(5):
  echo i                                        # 1 2 3 4 5

for i in initBigInt(1) ..< initBigInt(5):
  echo i                                        # 1 2 3 4

for i in countdown(initBigInt(10), initBigInt(0), 5):
  echo i                                        # 10 5 0

for i in countup(initBigInt(0), initBigInt(20), 10):
  echo i                                        # 0 10 20
```

Именно так решена задача о факториале 123 из регрессионных тестов: `for i in two .. n: result *= i`.

---

## 10. Случайные числа: модуль `bigints/random`

```nim
import bigints
import bigints/random
import std/random        # для Rand и initRand
```

```nim
func rand*(r: var Rand, x: Slice[BigInt]): BigInt
func rand*(r: var Rand, max: BigInt): BigInt
proc rand*(x: Slice[BigInt]): BigInt
proc rand*(max: BigInt): BigInt
```

Модуль расширяет `std/random` перегрузками для `BigInt`. Все границы **включительные**: `rand(a..b)` возвращает число из отрезка `[a, b]`, а `rand(max)` — из `[0, max]`. Диапазон должен быть корректным (`a <= b`), иначе срабатывает `assert`.

Первые две формы (`func`) принимают явное состояние генератора `Rand` и потому воспроизводимы и не имеют побочных эффектов на глобальное состояние. Две последние (`proc`) используют глобальный генератор. В версиях Nim, где нет `randState`, библиотека создаёт собственное глобальное состояние с фиксированным зерном `777`, то есть последовательность детерминирована. Если нужна недетерминированность, передавайте собственный `Rand`, инициализированный подходящим зерном.

Генерация равномерная: старшие два лимба выбираются осторожно, остальные — произвольно, а результат при выходе за диапазон отвергается и генерируется заново.

```nim
import bigints, bigints/random, std/random

var gen = initRand(42)                 # воспроизводимый генератор

let lo = pow(initBigInt(10), 90)
let hi = pow(initBigInt(10), 100)

let x = rand(gen, lo .. hi)            # 10^90 <= x <= 10^100
let y = rand(gen, initBigInt(1000))    # 0 <= y <= 1000
let z = rand(initBigInt(10) .. initBigInt(20))   # глобальное состояние
```

Обратите внимание: сгенерированные числа не предназначены для криптографии. Генератор `std/random` не является криптографически стойким.

---

## 11. Исключения

| Ситуация | Исключение |
| --- | --- |
| `div`, `mod`, `divmod` с нулевым делителем | `DivByZeroDefect` |
| `invmod` или `powmod` с нулевым модулем | `DivByZeroDefect` |
| `invmod` с нулевым `a` | `DivByZeroDefect` |
| `invmod` или `powmod` с отрицательным модулем | `ValueError` |
| `invmod`, когда обратного элемента нет | `ValueError` |
| `initBigInt(str, base)` — пустая строка, одинокий знак, символ вне алфавита основания, ведущий или замыкающий `_` | `ValueError` |
| `rand` с перевёрнутым диапазоном | срабатывание `assert` (в режиме `-d:danger`/`-d:release` проверка может быть отключена) |

`DivByZeroDefect` — наследник `Defect`, то есть отражает ошибку программиста, а не ожидаемую ситуацию. Для неверного пользовательского ввода перехватывайте `ValueError` при разборе строки, а нулевой делитель проверяйте заранее.

---

## 12. Производительность и ограничения

Библиотека написана с упором на простоту и корректность, а не на скорость. Умножение выполняется классическим алгоритмом за квадратичное время: Карацубы, Тоома–Кука и FFT нет. Перевод в десятичную строку и обратно тоже квадратичный. Быстрые (линейные) пути есть только для оснований, являющихся степенью двойки. Деление реализовано алгоритмом D Кнута. Для чисел в тысячи цифр производительности хватает, для сотен тысяч цифр и больше стоит присмотреться к обёрткам над GMP.

Составные операторы `+=`, `-=`, `*=` не работают на месте, каждый раз создавая новый объект. В горячих циклах это стоит учитывать: предпочитайте создавать меньше временных значений.

Не реализовано и потому отсутствует: `hash`, конвертация в `float`, извлечение квадратного корня, проверка простоты, факториал и биномиальные коэффициенты (их нужно писать самостоятельно, см. рецепты), операции с обычными `int` без явного `initBigInt` (кроме `inc`/`dec`/`succ`/`pred`), а также поддержка 32-битных платформ.

Библиотека тестируется на бэкендах `c` и `cpp` со сборщиками мусора `refc`, `arc` и `orc` при включённом `--experimental:strictFuncs`.

---

## 13. Рецепты

### Факториал

```nim
import bigints

func factorial(n: int): BigInt =
  result = initBigInt(1)
  for i in 2 .. n:
    result *= initBigInt(i)

echo factorial(30)     # 265252859812191058636308480000000
```

### Числа Фибоначчи

```nim
import bigints

func fib(n: int): BigInt =
  var (a, b) = (initBigInt(0), initBigInt(1))
  for _ in 1 .. n:
    (a, b) = (b, a + b)
  a

echo fib(100)          # 354224848179261915075
```

### Число цифр и сумма цифр

```nim
import bigints, std/strutils

let n = pow(initBigInt(2), 1000)
let s = $n
echo len(s)                                  # количество десятичных цифр
var total = 0
for ch in s:
  total += ord(ch) - ord('0')
echo total                                   # сумма цифр 2^1000
```

### Конвертация между системами счисления

```nim
import bigints

let x = initBigInt("deadbeefcafebabe", 16)
echo toString(x, 10)       # десятичная запись
echo toString(x, 2)        # двоичная
echo toString(x, 36)       # компактная запись в base36
```

### Учебный RSA

```nim
import bigints

let
  p = initBigInt(61)
  q = initBigInt(53)
  n = p * q                                  # 3233
  phi = (p - initBigInt(1)) * (q - initBigInt(1))   # 3120
  e = initBigInt(17)
  d = invmod(e, phi)                         # 2753

let message = initBigInt(65)
let cipher = powmod(message, e, n)           # 2790
let plain = powmod(cipher, d, n)
assert plain == message
```

Это лишь иллюстрация модульной арифметики. Для реальной криптографии эта библиотека не подходит: она не даёт защиты от атак по времени выполнения, а генератор случайных чисел не является криптографическим.

### Факторизация методом ро-Полларда (сокращённо)

Полная версия лежит в `examples/pollard_rho.nim`. Идея в том, что последовательность `x → x² + 1 (mod n)` вычисляется двумя «указателями», движущимися с разной скоростью, а НОД разности с `n` рано или поздно даёт нетривиальный делитель.

```nim
import bigints, std/options

func pollardRho(n: BigInt): Option[BigInt] =
  var
    turtle = initBigInt(2)
    hare = initBigInt(2)
    divisor = initBigInt(1)
  while divisor == initBigInt(1):
    turtle = (turtle * turtle + initBigInt(1)) mod n
    hare = (hare * hare + initBigInt(1)) mod n
    hare = (hare * hare + initBigInt(1)) mod n
    divisor = gcd(turtle - hare, n)
  if divisor != n: some(divisor) else: none(BigInt)

const N = "44077431694086786329".initBigInt
let f = pollardRho(N)
if isSome(f):
  echo get(f), " — делитель ", N
```

### Хранение `BigInt` в таблицах

Так как `hash` не определён, в качестве ключа берут строковое представление:

```nim
import bigints, std/tables

var cache: Table[string, BigInt]
let key = initBigInt(12345)
cache[$key] = key * key
```

---

## 14. Примеры из репозитория

В каталоге `examples/` лежат готовые программы. Файл `examples/nim.cfg` добавляет `../src` в путь поиска модулей, поэтому их можно запускать прямо из этого каталога.

| Файл | Что демонстрирует |
| --- | --- |
| `pidigits.nim` | Потоковый вывод цифр числа π (алгоритм-«spigot»); аргумент командной строки задаёт число цифр, без аргумента печатает бесконечно. |
| `pollard_rho.nim` | Факторизация методом ро-Полларда с использованием `gcd` и `mod`. |
| `pollard_p_minus_1.nim` | Метод `p − 1` Полларда. |
| `elliptic.nim` | Арифметика на эллиптических кривых (получение открытого ключа из закрытого), активно использует `invmod`. |
| `rc_pow.nim` | Задача Rosetta Code: вычислить 5^(4^(3^2)), вывести первые и последние 20 цифр и длину числа. |
| `rc_sum35.nim` | Сумма кратных 3 или 5 ниже 10^k до k = 30. |
| `rc_leftfactorials.nim` | Левые факториалы через итератор. |
| `rc_combperm.nim` | Размещения и сочетания. |
| `rc_godtheinteger.nim`, `rc_godtheinteger2.nim` | Число разбиений натурального числа двумя способами. |
| `rc_hammingnumbers.nim` | Числа Хэмминга. |
| `rc_paraffins.nim` | Подсчёт изомеров парафинов. |
| `rc_integersequence.nim` | Бесконечный счётчик (останавливается по Ctrl+C). |

Запуск, например:

```
cd examples
nim r pidigits.nim 100
nim r rc_pow.nim
```

---

## 15. Тесты и сборка

Проверка всей матрицы «бэкенд × сборщик мусора»:

```
nimble test
```

Задача перебирает бэкенды `c` и `cpp` и сборщики `refc`, `arc`, `orc` и для каждой комбинации запускает `tests/tbigints.nim`, `tests/tbugs.nim`, `tests/trandom.nim`, а также собирает документацию. Литералы `'bi` покрыты отдельным включаемым тестом `tests/tliterals.nim`. Проверка синтаксиса всех примеров:

```
nimble checkExamples
```

Генерация HTML-документации из исходников (в проекте она публикуется на GitHub Pages):

```
nimble doc --index:on --project src/bigints.nim
```

Опубликованная документация: <https://nim-lang.github.io/bigints>.

Структура пакета:

```
bigints-master/
├── bigints.nimble          # описание пакета
├── src/
│   ├── bigints.nim         # BigInt и вся арифметика
│   └── bigints/
│       ├── random.nim      # rand(...) для BigInt
│       └── private/
│           └── literals.nim  # литерал 'bi (include-файл, отдельно не импортировать)
├── tests/                  # tbigints, tbugs, tliterals, trandom
└── examples/               # примеры
```

---

## 16. Сводный указатель API

| Символ | Сигнатура | Кратко |
| --- | --- | --- |
| `BigInt` | `object` | Целое произвольной точности |
| `initBigInt` | `(val: SomeInteger)` | Из стандартного целого |
| `initBigInt` | `(val: BigInt)` | Копия |
| `initBigInt` | `(str: string, base: range[2..36] = 10)` | Из строки |
| `initBigInt` | `(vals: seq[uint32], isNegative = false)` | Из лимбов (младший первым) |
| `'bi` | `(s: string): BigInt` | Литерал `123'bi`, `0xFF'bi`, `0b101'bi` |
| `$` | `(a: BigInt): string` | Десятичная строка |
| `toString` | `(a: BigInt, base: range[2..36] = 10): string` | Строка в заданной системе |
| `toInt` | `[T: SomeInteger](x: BigInt): Option[T]` | В стандартное целое или `none` |
| `==`, `<`, `<=` | `(a, b: BigInt): bool` | Сравнение (`!=`, `>`, `>=` выводятся) |
| `+`, `-`, `*` | `(a, b: BigInt): BigInt` | Арифметика |
| `-` | `(a: BigInt): BigInt` | Унарный минус |
| `+=`, `-=`, `*=` | `(a: var BigInt, b: BigInt)` | Составное присваивание (шаблоны) |
| `abs` | `(a: BigInt): BigInt` | Модуль |
| `pow` | `(x: BigInt, y: Natural): BigInt` | Степень |
| `div`, `mod` | `(a, b: BigInt): BigInt` | Деление с округлением вниз и остаток |
| `divmod` | `(a, b: BigInt): tuple[q, r: BigInt]` | Оба сразу |
| `inc`, `dec` | `(a: var BigInt, b: int = 1)` | Изменение на месте |
| `succ`, `pred` | `(a: BigInt, b: int = 1): BigInt` | Следующее и предыдущее значение |
| `not`, `and`, `or`, `xor` | Побитовые | Как в дополнительном коде |
| `shl`, `shr` | `(x: BigInt, y: Natural): BigInt` | Сдвиги (`shr` арифметический) |
| `gcd` | `(a, b: BigInt): BigInt` | НОД |
| `invmod` | `(a, modulus: BigInt): BigInt` | Обратный по модулю |
| `powmod` | `(base, exponent, modulus: BigInt): BigInt` | Степень по модулю |
| `fastLog2` | `(a: BigInt): int` | Индекс старшего бита, `-1` для нуля |
| `countup`, `countdown` | `(a, b: BigInt, step: int32 = 1)` | Итераторы со ступенью |
| `..`, `..<` | `(a, b: BigInt)` | Итераторы диапазона |
| `rand` | `(r: var Rand, x: Slice[BigInt])` и др. | Случайное `BigInt` (модуль `bigints/random`) |
