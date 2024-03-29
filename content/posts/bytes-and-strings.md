---
title: "А как же строки?"
date: 2023-05-20
draft: false
---

Вне всякого сомнения самое сложное в программировании это
именование переменных и _что-то там про кеш_.
Самое сложное в написании статьи это написать введение. Нельзя же начитать
с того, что бросить в читателя кусок случайного кода.

Это экспериментальная заметка. Я буду признателен за любой фидбек.

Все правки, очепятки и откровенные ошибки отправляйте на электронный
адрес rokkerruslan@protonmail.com, я исправлю это. Так же можно внести правки
самостоятельно через [Pull Request](https://github.com/rokkerruslan/gobuch/pulls).
Буду рад. Стараюсь как могу не сильно глубоко уходить в детали и не
распыляться по многим темам одновременно, чтобы сохранить некоторую целостность
в повествовании и темп рассказа. Сноски содержат более подробное описание
блока, его пояснение (или ещё больше запутывают). По ним можно проходить и
понять что имелось в виду в том или ином пункте, если этот пункт в статье
показался неочевидным. Так же сноски содержат ссылки на дополнительные источники
информации[^7].

Так же укажу кто есть аудитория статьи. Это программисты на Go желающие лучше
понять механизм работы встроенных типов данных, а именно — строк и слайсов байт
(далее просто слайсов).

Я рекомендую экспериментировать с примерами самостоятельно до момента полного
его понимания.

Вполне вероятно что примеры, результат выполнения той или иной команды будет
зависеть от версии компилятора и операционной системы, я запускаю на:

```text
go env GOVERSION GOARCH GOOS
devel go1.19
amd64
linux
```

С введением всё.

## Чсть 1. Поствведение

Преобразование одних данных в другие это одна из самых распространённых задач в
программировании. Пример
с которого мы начнём не будет отличаться оригинальностью или вычурностью.
Он вполне банален, вы скорее всего не раз делали это самостоятельно
Есть список байт, мы получили его из некого источика, нужно преобразовать массив в число.
Последовательности 
байт будут содержать закодированные [utf-8](https://en.wikipedia.org/wiki/UTF-8),
целые, неотрицательные, 64-битные числа в десятичной системе счисления.

```go
package main

func main() {
	input := [][]byte{
		{0x31, 0x32, 0x33}, // 1 2 3
		{0x34, 0x35, 0x36}, // 4 5 6
	}
	for i := range inputs {
		println(string(input[i])) // Нам нужно преобразовать в число значение inputs[i].
	}
}

// go run main.go
// 123
// 456
```

Структура данных представляющая последовательность байт в языке Go это
[Array](https://go.dev/ref/spec#Array_types) и реализованный на его базе тип
[Slice](https://go.dev/ref/spec#Slice_types), slice может изменять размер в
процессе работы программы. Некоторые
примеры требуют изменения размера, поэтому мы
используем именно его.
С типами входных данных определились, едём дальше. 

Алгоритм. На каждой итерации цикла копируем из источника данные в заранее выделенное место,
обрабатываем их, преобразуя в _число_, далее проводит вычисления с этим числом.
В первом приближении решение будет выглядеть так[^8]:

```go
// allocate buf
// loop
	n, err := source.ReadFrom(buf[:])
	// ...
	nn, err := strconv.ParseUint(buf[:n], 10, 64)
	// use nn
```

Всё хорошо, только этот код не компилируется -- [(playground)](https://go.dev/play/p/EAMjZgNOB6J).
Всё дело в несовпадении типа аргумента `buf[:n]` и типа параметра функции
[ParseUint](https://pkg.go.dev/strconv#ParseUint), _byte-slice_ и
[string](https://go.dev/ref/spec#String_types) соответственно. Функции
пакета `strconv` предназначены для обработки строк, но не со слайсами,
даже название намекает, `strconv` расшифруем как _строка, конвертируй_.

Для того чтобы понять что делать дальше, нужно погуглить [ref/spec](https://go.dev/ref/spec)
по ключевым словам `slice`, `string`, `convertion`.
И в Go есть механизм, позволяющий _изменить_ один тип данных на другой, называется он —
[Type Conversions](https://go.dev/ref/spec#Conversions). Синтаксис одинаков
для всех пар типов, имя типа, выражение в скобках — `Type(Expression)`. Правило,
по которому происходит конверсия,
отличается от пары к паре (если оно вообще реализовано).
Необходимое нам правило есть в языке:

> Converting a slice of bytes to a string type yields a string whose successive bytes are the elements of the slice.

Конверсия слайса в строку _выращивает_ строку байты которой есть элементы слайса.
Даём байты получаем строку, то что нужно, воспользуемся конверсией:

```diff
- v, err := strconv.ParseUint(buf[:n], 10, 64)
+ v, err := strconv.ParseUint(string(buf[:n]), 10, 64)
```

Да, теперь компилируется. Это корректный и быстрый код. А быстрый ли? С точки зрения
производственной среды — да, он ведь не светится как светодиодные игрушки на
новогодней ёлке в отчете
профилировщика. И я намеренно не привожу бенчмарки сферических примеров в
вакууме, потому что оптимизацией следует заниматься не на основе результатов
бенчмарков, и уж точно не потому что вы знаете что эта конструкция может работать неэффективно,
а только на основе отчётов инструмента `pprof` снятых с работающего приложения.

Но вопрос остаётся. Если вы всё же увидели этот блок кода в профилировщике?
Насколько быстрый это код? Делает ли он лишние вычисления?
Что он вообще делает?

Вернёмся и прочитаем ещё раз:

> Converting a slice of bytes to a string type yields a string whose successive bytes are the elements of the slice.

_Создаёт строку_, _последовательность байт совпадает с элементами слайса_. Это
похоже на определение, определение — не алгоритм, из него совершенно
не ясно что конкретно делает Go. Правило не говорит о реализации,
оставляя пространство для оптимизации в будущем, что правильно,
но мы вольны изучать реализацию.

Сейчас пока мы не можем предполагать насколько сложная будет реализация. Какую часть на
себя берёт компилятор, а какую часть — runtime[^14]. Если вспомнить что слайс
изменяемая структура данных, а строка — нет, то
возникает вопрос — как превратить изменяемую в неизменяемую?

Если зажать клавишу _Ctrl_ и кликнуть на `ParseUint` мы попадём[^12] на
реализацию функции `ParseUint`. Но точно такой-же
клик на слово `string` приводит нас в файл `builtin.go` документация которого
говорит что:

> The items documented here are not actually in package builtin

Если тут нет искать нужно в другом месте. Другой способ это посмотреть
на готовую программу и её мышиный код. Или ещё лучше попросить
компилятор предоставить нам промежуточное ассемблерное представление, а
не исполняемый файл. Прелесть современных инструментов программиста.

Один из вариантов для Go это запуск команды `GOOS=linux GOARCH=amd64 go build -gcflags=-S`[^11].
Выводит много информации, которая сейчас нам не интересна, нас интересует только момент
вызова функции `strconv.ParseUint` и _вычисление её аргументов_, так как строка это
один из аргументов функции.

Подготовка аргументов в вызов функции:

```asm
	// strconv.ParseUint(string(buf[:n]), 10, 64)

	CALL	runtime.slicebytetostring ; После вызова регистры AX/BX будут содержать строку.
	MOVL	$10, CX ; Положить число 10 в регистр C
	MOVL	$64, DI ; Положить число 64 в регистр D
	CALL	strconv.ParseUint
```

Обратить внимание нужно только на вызов функции `runtime.slicebytetostring`. Именно
эта функция конвертирует байт слайс в строку[^15].

Это уже то что можно найти в исходном коде. Без некоторых лишних деталей её реализация выглядит так[^10]:

```go
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}
```

Сначала мы проверяем выдали ли нам буфер куда мы поместим содержание строки,
если нет или длина буфера недостаточна, runtime выделит в куче пространство для
строки. Что же мы видим?

- Потенциальное выделенеи памяти — `mallocgc`.
- Копирование участка памяти — `memmove`.

Наша задача не предполагает использование старого значения слайса
(число уже распаршено и обработано) после завершения очередной итерации цикла, и, так же не предполагает
дальнейшее использование строки. Слайс байт мы можем переиспользовать явно, а что со строкой?

Мы всё равно её конструируем, выделяем память (в некоторых случаях),
копируем содержимое слайса вместо выделенное под строку (безальтернативно).
И так на каждой итерации. Нагружаем runtime бесполезной работой,
на работу аллокатора требуется CPU, на отслеживание выделенных объектов и их
дальнейшее освобождение сборщиком мусора так же тратит ресурсы CPU.

## Чсть 2. Что такое строки и что такое байты

> И ассемблер и рантайм. Энто сложно. Прекращай!
> А меж тем сказке — далеко до развязки!..

Мы удовлетворяем требование типа параметра функции `strconv.ParseUint` и более нам
не нужно значение типа `string`. Отсюда вытекает наше предположение, а можем
ли мы попробовать не вызывать функцию `slicebytetostring`? Не использовать
синтаксическую конструкцию _string(X)_. Как ещё можно преобразовать один тип
в другой?

Мы уже знаем что строки это просто последовательности байт, нет необходимости
проводить сложные вычисления при создании строки. Как например убедиться что
последовательность байт это валидная строка в _utf-8_. Это, как минимум в
теории, делает возможным преобразование значений с минимальными потерями в
производительности.

Можно ли заставить компилятор интерпретировать некоторый участок
памяти выделенный под слайс как строку? Для этого нужно исследовать
их внутреннее представление более тщательно.

Для начала посмотрим их представление
в коде языка. Объявление типов находятся в пакете _runtime_:

```go
// runtime/slice.go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

// runtime/string.go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

Слайс это структура с тремя полями — `unsafe.Pointer`, `int`, `int`. Тип `unsafe.Pointer`
в свою очередь это указатель на произвольный тип, что такое этот произвольный тип
тут не важно, нам достаточно что это указатель:

```go
type Pointer *ArbitraryType // type ArbitraryType int
````

Указатель на 64-битной системе равен 8-ми байтам. Два int-а это ещё 16 байт. Итого 24
байта на один слайс. Строка это два поля, так же указатель на данные — `unsafe.Pointer`
и длина строки int всего 16 байт, ёмкости у строк нет.

Предположим мы обманули компилятор и заставили его работать со слайсом как со
строкой. Сработает ли это? Для наглядности попробуем поработать не с абстрактным
описанием структур в тексте программы, а с реальными значениями.

Наш полигон:

```go
func foobar () {
	a := "foobar"
	b := []byte(string("foobar"))

	// ...
}
```

Ниже представлена открытая сессия [delve](https://github.com/go-delve/delve)
отладчика. В локальном окружении функции есть две переменные `a` и `b`, `а` строка,
`b` — это слайс (uint8 это подтип типа byte). Для просмотра локальных
переменных есть команда _locals_:

```text
(dlv) locals
  a = "foobar"
  b = []uint8 len: 6, cap: 6, [...]
```

Значение строки нам показывают, значение слайса мы не видим, указывается
только длина и ёмкость. Не очень понятное представление. Не ясно что это лежит в памяти.
Локальное оружение функции, а точнее сказать _кадр стека_ тоже объект,
выполнив команду `frame 0`, мы увидим его визуализизацию:

```text
(dlv) frame 0
> main.foobar() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x45fa02)
Frame 0: ./main.go:9 (PC: 45fa02)
     4:	func foobar() {
     5:		a := "foobar"
     6:		b := []byte(string("foobar"))
     7:	
     8:		// ...
=>   9:		println(a, b)
    10:	}
    11:	
    12:	func main() {
    13:		foobar()
    14:	}
```

Никакой нужной нам нформации нет. Можно вычислить адрес кадра стека
и после вычленить оттуда значения a и b, но намного проще получить
адреса переменых в памяти и инспектировать уже её.

Адреса переменных можно получить так:

```text
(dlv) print unsafe.Pointer(&a)
  (unsafe.Pointer)(0xc000052720)

(dlv) print unsafe.Pointer(&b)
  (unsafe.Pointer)(0xc000052730)
```

Так, как мы уже видели выше строка это указатель и _int_. И указатель и `int` на 64-битной
системе равны восьми байтам. Восемь плюс восемь будет 16.

Значит, чтобы вычитать значение строки, нужно прочитать 16 байт начиная с адреса
переменной `a` с помощью команды _examinemem_:

```text
(dlv) examinemem -fmt hex -size 1 -count 16 0xc000052720
  0xc000052720:   0x68   0xf6   0x46   0x00   0x00   0x00   0x00   0x00   
  0xc000052728:   0x06   0x00   0x00   0x00   0x00   0x00   0x00   0x00
```

Моя машина — little-endian[^13] машина. По младшему адресу `0xc000052720` хранится младший байт
указателя на строку это  _0x68_, по адресу старшему `0xc000052727` старший байт указателя — _0x00_.
По адресу `0xc000052728`
находится младший байт значения длины — _0x06_, по адресу `0xc00005272f` — старший
байт поля `len` — `0x00`. Чтобы отобразить их в человеко-понятном виде нужно выписать
значения задом наперёд, без пробелов и каких либо разделительных знаков (так как я привык ставить старший разряд
левее младшего, а не наоборот). Или попросить `delve` отобразить значение с учётом размера,
размер при этом нужно указать самостоятельно, так как размеры одинаковы, там и там по 8 байт,
попросим 2 раза по 8 байт:

```text
(dlv) examinemem -fmt hex -size 8 -count 2 0xc000052720
  0xc000052720:   0x000000000046f668   0x0000000000000006
```

Первые 8 байт это указатель на строку, по нему тоже можно пройтись и
посмотреть что там лежит:

```text
(dlv) examinemem -fmt hex -size 1 -count 6 0x000000000046f668
  0x46f668:   0x66   0x6f   0x6f   0x62   0x61   0x72 // f o o b a r -> foobar
```

Судя по документации delve не поддерживает формат char или что-то вроде того,
так что декодируйте в уме.

Так хорошо, со строками разобрались, а что из себя представляет байт слайс?

```text
(dlv) examinemem -fmt hex -size 1 -count 24 0xc000052730
  0xc000052730:   0x12   0x27   0x05   0x00   0xc0   0x00   0x00   0x00   
  0xc000052738:   0x06   0x00   0x00   0x00   0x00   0x00   0x00   0x00   
  0xc000052740:   0x06   0x00   0x00   0x00   0x00   0x00   0x00   0x00 
```

Так, первые 16 байт это тоже адрес и длина. По адресу `0xc000052740` хранится
ёмкость слайса.

А если с этого адреса прочитать только два байта? Вы можете ответить на вопрос, это строка
лежит или слайс байт?

```text
(dlv) examinemem -fmt hex -size 8 -count 2 0xc000052730
  0xc000052730:   0x000000c000052712   0x0000000000000006
```

Всё это наводит на мысль, что мы можем представить слайс как строку. Если
попытаться обмануть компилятор, сказать - _ты сейчас обращаешься к строке_
подсунув при этом эму участок памяти где лежит слайс. Он точно ничего не заметит.
Сломать систему типов может быть не просто. Начитать, как и всегда, стоит с
документации. В этот раз искать по ключевым словам `type system` и `violate`.

В самом конце спецификации (эт видимо чтобы подольше не находили) читаем:

```text
The built-in package unsafe, known to the compiler and accessible through the
import path "unsafe", provides facilities for low-level programming including
operations that violate the type system.
```

О, да. Кажется слова _нарушение системы типов_ и есть те заветные слова, что мы ищем.
Следующие параграфы проясняют _как именно_ можно её (систему типов) поломать:

```text
A Pointer is a pointer type but a Pointer value may not be dereferenced. Any
pointer or value of underlying type uintptr can be converted to a type of
underlying type Pointer and vice versa.
```

Так, то есть это предложение, в том числе, говорит нам о том, что любой
указатель может быть преобразован в тип, _underlying type_ которого есть тип Pointer,
_underlying type_ типа Pointer тоже Pointer. Наш алгоритм преобразования будет такой:

- Взять слайс.
- Взять указатель на это значение путём операции взятия адреса - `&`, теперь у нас
есть указатель на слайс.
- Конвертировать указатель на слайс в `unsafe.Pointer`.
- Конвертировать `unsafe.Pointer` в указатель на строку.
- Разыменовать указатель на строку и получить строку[^16].

По шагам в коде:

```go
buf := []byte{'H', 'e', 'l', 'l'}
pointerToByteSlice := unsafe.Pointer(&buf)
pointerToString := (*string)pointer
str := *pointerToString
fmt.Printf("val=%q type=%T\n", str, str) // val="Hell" type=string
```

Когда runtime пойдёт брать длину строки, так же отсчитает 8 байт от начала и
прочитает 8 байт где найдёт длину строки. А когда возьмёт 8 байт начиная с 0, он
обнаружит адрес, по которому лежат байты нашей строки. Как мы видели в реализации
функции `slicebytetostring` содержимое слайса байт не проходит никакой обработки
и не может завершиться неудачей[^18].

Однострочная версия:

```go
buf := []byte{'1', '2', '3', '4'}
v, err := strconv.ParseUint(*(*string)(unsafe.Pointer(&buf)), 10, 64)
check(err)

fmt.Printf("type=%T val=%v", v, v) // type=uint64 val=1234
```

А без unsafe можно? Нет.
Правила конверсии указателя на слайс в указатель на строку не существует. Как-то так:

```text
(*string)(&buf)                 // Cannot convert an expression of the type '*[]byte' to the type '*string'
(*string)(unsafe.Pointer(&buf)) // This is fine 🔥
```

Что мы получаем в итоге. Вызов функции `slicebytetostring`, потенциальную аллокацию и копирование
заменено взятием адреса и разыменованием:

```asm
  MOVQ  32(SP),  AX     ; Взять адрес и положить в AX
  MOVQ  AX,      24(SP) ; Сохранить адрес в SP по смещению 24
```

Этого точно не будет в отчёте `pprof`.

## Чсть 3. А как же строки?

> _— А как же строки, строки? Руслан._
> _— Какие ещё строки?_
> _— Ты перечислял неизменяемые типы, почему не упомянул строки?_
> _— А это из другой сказки._

Все эти операции немного расшатывают наше представление о том что строки
являются неизменяемыми структурами данных. Мы не только ссылаемся на один
и тот же массив, но и переиспользуем заголовок слайса байт как заголовок строки.
Это может означать только одно — они изменяемы. Документация же явно говорит что нет:

> Strings are immutable: once created, it is impossible to change the contents of a string.

Может быть те строки что мы строим из слайса _не настойщие_? Попытаемся пойти обратной
дорогой и начать вот с этих самых _не изменяемых_ строк. Например, что на счёт
[литерала](https://go.dev/ref/spec#String_literals) строки?

```text
here := "Hello, Go!"
here[0] = 1 // Compile time error: Cannot assign to here[0]
```

И правда. Менять содержимое строки не даёт ещё даже компилятор.

> _Ты когда нибудь Segmentation Fault на Go видел? И я не видел, а он — есть[^2]_:

Мы уже научились конвертировать слайсы в строки, что если действовать по аналогии?
Создадим слайс совместно использующий нижележащий массив вместе со
строкой. Компилятор же не будет против изменения слайса?

```go
package main

import "unsafe"

func main() {
	s := "Hello, Go!"
	b := *(*[]byte)(unsafe.Pointer(&s))
	b[0] = 'X' // unexpected fault address 0x100ebfe77
}
```

Компилятор откомпилировал, но ошибка (фатальная) уже
при исполнении. Это вполне способно разбудить вас ночью,
но программисты не спят — идём дальше:

```go
package main

import (
	"unsafe"
	"fmt"
)

func main() {
	s := string([]byte("Hello, Go!")) // Кто в Text Segment не спрятался, я не виноват.
	b := *(*[]byte)(unsafe.Pointer(&s))
	b[0] = 'X'

	fmt.Println(s) // Xello, Go!
}
```

Это работает, Гарольд. Получается что не все строки в программе такие уж и
неизменяемые, некоторое подмножество строк живущих в программе изменить можно,
[Playground](https://go.dev/play/p/YeZAO4Mhmdy). Изменяемы те, кто живёт в куче,
не изменяемы те, что живет в коде программы[^5].

Конструкция `[]byte(<str>)` заставляет компилятор вставить процедуру аналогичную `slicebytetostring`
и в изменяемой памяти процесса появится массив равный длине строки, куда будут скопированы
элементы строки (из сегмента неизменяемой памяти). Преобразование результата в строку так же
происходит уже во время выполнения программы и runtime-у нужно сконструировать строку
из слайса, поэтому она уже не может быть помещена в сегмент неизменяемой памяти.

Версия для [собеседований](https://go.dev/play/p/MNwXW4k9wxw). Запомните или запишите:

```go
package main

import (
	"unsafe"
	"fmt"
	"strings"
)

func main() {
	buf := strings.Builder{}
	buf.WriteString("Go strings are immutable")

	out := buf.String()

	doit(out) // Очевидно ничего плохого со строкой не сделает.

	fmt.Println(out) // Go strings are mutable
}

func doit(s string) {
	b := *(*[]byte)(unsafe.Pointer(&s))

	for i := 0; i < 7; i++ {
		b[15+i] = b[17+i]
	}

	b[len(b)-2] = ' '
	b[len(b)-1] = ' '
}
```

Шутяки заканчивается когда мы вспоминаем, что байт слайсы изменяемый
тип данных. Например, мы можем растянуть байт слайс до его емкости. Так стоп, а
какая вообще ёмкость у получившегося слайса?

```go
package main

import (
	"unsafe"
	"fmt"
)

func main() {
	here := "Hello "
	here = string([]byte(here))

	out := *(*[]byte)(unsafe.Pointer(&here))

	fmt.Printf("%d %d\n", len(out), cap(out)) // 6 1374390628136
}
```

Длина равна шести. Ожидаемо. Емкость равна... Один миллион... не, один миллиард... — _очень много_. Шести не равно.
Ответы 0, 6 и 42 пояснения не требуют, но 1374390628136?

Мы не создаём `[]byte` явно. Говоря что по адресу, к примеру `0xff0000`, лежит байт слайс
runtime может попытаться считать значение ёмкости по смещению `0xff0000` плюс `0x10`. Но этот
участок памяти ему уже не принадлежит. Если заголовок строки находился на стеке функции,
то после заголовка может лежать другая переменная. Если заголовок был выделен в куче, то
участок после заголовка строки может принадлежать совершенно другой переменной выделенной ранее где-то
в программе.

```text
// Заголовок строки в памяти.
0xff0000 0x00000001006a8fd0
0xff0008 0x0000000000000006
// Выделенная память аллокатором для переменной X=1374390628136 в другом участе програмыы.
0xff0010 0x000001400010af28
```

Мы не контролируем значение ёмкости получившегося слайса. Значение может быть совершенно
случайным. В зависимости от того что уже успел навыделять аллокатор.
Если вы запустите код на своей машине, вывод может быть другим, но точно
сказать, чему будет он равен — нельзя.

Раз ёмкость не равна длине, а может быть даже больше длины. Значит длину слайса
можно растянуть, например, до сотни:

```go
package main

import (
	"unsafe"
	"fmt"
)

func main() {
	here := "Hello "
	here = string([]byte(here))

	out := *(*[]byte)(unsafe.Pointer(&here))
	out = out[:100]
	fmt.Printf("%d %d\n", len(out), cap(out)) // "Hello  @�!@HelloO@"
}
```

Ха, участок памяти, где лежит строка "Hello " тоже не изолированный. За ним следующие ячейки
памяти, в которых так же хранятся значения других переменных программы.

Представим что клиент определяет сколько он может ещё получить данных. Распространённая
практика[^17]. Ограничение всё равно есть, чтобы не перегрузить сервер. Мы пишем надёжное ПО?

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	here := "Hello "
	here = string([]byte(here))

	func() {
		secretparol := string([]byte("lolkekcheburek")) // Секретное преобразование для секретного пароля
		secretparol += ""
	}()

	out := *(*[]byte)(unsafe.Pointer(&here))

	fmt.Printf("%d %d\n", len(out), cap(out))

	fmt.Println(string(out))
	out = out[:1000000]      // Я могу получить ещё 1000000. Присылай!
	fmt.Println(string(out))
}
```

Полный вывод:

```text
6 1374389815144
Hello
Hello �!@("@(�  @Hello Ў        @Ў      @�zp@�sP:�s
@$@@�!@`)��     @��     @�      @�      @`      @�      @ !     @@��!@�&@@@�@�@��
          ����r �����AQ�"���hdA�I8��hd�r        ��hd�r  ���cr�9�Z�@����r       �����AQ�"���hdA�I8��hd�r        ��hd�r  ���cr�9�
@p
@1374389815144.lolkekcheburekhG@6 Hello // ТУТ КАЖЕТСЯ ЕСТЬ ПАРОЛЬ
89815144
Hello ��px`
@f�f��0�`@0`@�@ �@
���l�r@@``@ �@�`@�`@x�@ H�@ a@
u*���(*�@�@�@�@0�@
@�@               @

  xY�\�@�`@`�P@/Users/rokkerruslan/w/secret.maxfilesperprocIp // Так, а это что такое? Мак?
@
```

В воздухе витает слово уязвимость и уже давно. Не смотрите что в выводе много лишней информации,
пароли и другие секретный последовательности обычно отличаются от строк в
программе, их легко определить. А некоторые строки вообще по структуре можно
искать, примерно так выглядят все AccessKeyID от AWS — `AKETOJOPAGANDAMSTYLE`.

Если вам кажется что это нереалистичный сценарий, то всё-таки кажется. У этого
типа ошибок своё название есть - [Buffer Overflow](https://en.wikipedia.org/wiki/Buffer_overflow).

Подумаем. Программа может:
- Упасть с ошибкой `Segmentation Fault`, если так выйдёт что при чтении
из слайса мы выйдем из границы адресного пространства выделенного нам ОС.
Программа завершается. И это самое безобидное из всех возможных ситуаций.

- Как показано выше, атакующий может вычитать область памяти, в которой в свою очередь
может присутствовать чувствительная информация, явки/пароли из стека горутины и
стека других горутин в программе. И всё содержимое памяти процесса.

- Атакующий может изменить чужую область памяти на своё значение. Например, если он знает по
какому смещению живёт оригинал пароля, то может его подменить его своим.

## Чсть 4. "Правильный" способ

Одним из самых лучших способов поднимать свой уровень знаний это учиться у
старших.

Компилятор это не только инструмент для сборки программ на Go, но и
большое множество примеров по написания кода. Если вы
хотите изучить некий алгоритм, начать поиски с исходного
кода компилятора — неплохая точка входа. Нам же, даже искать не надо,
мы уже видели нужный нам код.

Помним что есть функция `slicebytetostring`. Она выделяет место в памяти для
нижележащего массива, а потом копирует туда элементы исходного массива. Но, это
только часть, а что с самой структурой данных заголовка строки?

Посмотрим на реализацию ещё раз:

```go
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}
```

Заголовок строки создаётся внутри функции `slicebytetostring`. Поля `str` и `len`
заполняются явно, новыми значениями. Значение поля `str` это указатель на ново-выделенный
участок памяти, значение поля `len` это длина участка памяти.

Слово "правильный" звучит так себе, лучше скажем _более безопасный_, так вот это
более безопасный способ преобразовать строку в слайс и обратно, без аллокации и копирования
нижележащего массива (но будте внимательны, спать нельзя даже тут, потому что и
в его использовании можно накосячить[^4]).

Идея заключается в том, что мы конструируем заголовки (и строки и слайса)
самостоятельно и копируем поля из структуры из которой мы преобразовываем в
только что созданную структуру.

Используем подход `slicebytetostring`, структуры слайса и строки можно найти в
пакете `reflect`:

```go
// Конверсия из слайса в строку.
func btos(in []byte) (out string) {
	slice := (*reflect.SliceHeader)(unsafe.Pointer(&in))
	str := (*reflect.StringHeader)(unsafe.Pointer(&out))
	str.Data = slice.Data
	str.Len = slice.Len

	return s
}

// Конверсия из строки в слайс байт.
func stob(in string) (out []byte) {
	slice := (*reflect.SliceHeader)(unsafe.Pointer(&out))
	str := (*reflect.StringHeader)(unsafe.Pointer(&in))
	slice.Data = str.Data
	slice.Len = str.Len
	slice.Cap = str.Len

	return s
}
```

Подход не отличается концептуально, использование unsafe остаётся, формально осталось и
нарушение системы типов, ведь `reflect.SliceHeader` это не `runtime.slice`. Если их
представление в памяти не будет совпадать, то будет плохо, хотя это маловероятно и
говорит о баге в коде компилятора/runtime-а.

Но мы больше не интерпретируем блок памяти от заголовка слайса как блок памяти
заголовка строки. Мы создаём новый заголовок для слайса байт, а далее заполняем
его поля руками. Более ли этот подход устойчивее к ошибке, да, он более устойчив
(нужно упомянуть, вы создаёте новый объект, вы не можете указать компилятору не
выделять объект в куче)[^19].

Сложность реального процесса и изначальное представление строк, как неизменяемых типов, приводит
к тому что функциональности конверсии (без выделения памяти и копирования) нет
в стандартной библиотеке Go, по крайней мере, это так со
[слов разработчиков](https://github.com/golang/go/issues/25484#issuecomment-390846454).

Особняком стоит проблема, что язык Go, а именно компилятор языка, не
подталкивает вас к правильному решению[^3]. Для компилятора все способы одинаковые
пока это компилируется, а как видели мы выше, компилируется не значит работает.
Даже работает значит _корректно работает_. Корректную же работу должен
обеспечить программист. Не надейтесь, что кот будет работать просто так.

## Заключение

Заключения нет. Делитесь и распространяйте статью. С вас лайк, подписка и, конечно
же, не забудьте нажать на _колокольчик_ дабы не пропустить новые...

[^7]: Кроме первой. Но я больше так не буду, das verspreche ich. Нажмите на
обратную стрелку расположенную сразу за текстом, чтобы вернуться на место
сноски в тексте. На мобильных устройствах это должно работать ещё лучше,
попробуйте сделать жест которым вы обычно говорите браузеру — _назад!_.

[^2]: На самом деле, это довольно распространённая ошибка. Чаще всего
её встретить можно при разыменование `nil` указателя, когда забыли
инициализировать поле структуры или что-то в этом роде.

[^3]: Код конверсии не выдаёт ошибок на проверке go vet.

[^5]: Виды памяти в программе. Для нас
достаточно что для процесса (в котором исполняется наша программа) существует
неизменяемая область памяти и изменяемая. В неизменяемой памяти хранится, например,
сама программа, литералы строк. В изменяемой памяти хранятся переменные, стеки горутин,
любые значение выделенные в куче.
Это сложная тема. Детали сильно зависят от операционной системы. Я
не думаю что есть один материал описывающий всё досконально. Стартовая точка для
linux-based систем — [Anatomy of a Program in Memory](https://manybutfinite.com/post/anatomy-of-a-program-in-memory).
Более глубокое погружение —
[Understanding the Memory Layout of Linux Executables](https://gist.github.com/CMCDragonkai/10ab53654b2aa6ce55c11cfc5b2432a4).

[^4]: Нельзя явно создавать экземпляры этих структур, только конвертировать из
строк/байт-слайс. Более подробно в комментарии к методу
[unsafe.Pointer](https://pkg.go.dev/unsafe#Pointer), пункт номер 6.

[^8]: Разбиение потока на отдельные сообщения или по другому - 
[message framing](https://blog.stephencleary.com/2009/04/message-framing.html),
мы оставим за бортом. Предположим что одна операция чтения (вызов ReadFrom) возвращает одно сообщение,
которое в свою очередь содержит одно число и оно точно поместится в отведённый буфер.

[^10]: В версии 1.20 код немного изменён, но не принципиально. Мы не будем разбирать новую
версию, я уверен, после прочтения статьи вы без труда сами проанализируете изменение.

[^11]: То что мы видим в результате вызова команды `build -gcflags=-S` это промежуточное
представление называется Go-ассемблер. Мы не будем подробно останавливаться. Его знание
не играет роли для понимания статьи, `MOVL $1, AX` — это _копирование_ значения _один_ в
регистр `AX`, и этого достаточно. Если же вы хотите более подробно изучить его, начать
стоит с [A Quick Guide to Go's Assembler](https://go.dev/doc/asm).

[^12]: Если вы, как и я из поколения IDE. А если нет, то и сами знаете как найти.

[^13]: Порядок байт. Тема не большая и не маленькая. И полностью не покрыть в сноске
и ссылку оставлять не хочется. Есть числа, которые занимают больше
чем 1 байт, а значит больше чем 1 ячейку в памяти. Например, `int16` это два
байта, то есть оно занимает две ячейки в памяти. Возьмём значение 1 типа int16 — `0x0001`. Старший байт
равен 0, младший — 1. У каждой ячейки есть адрес. Например, по адресу `0xffff00a7` будем хранить
байт `0x00`, а в следующем адресе `0xffff00a8` будем хранить `0x01`. Так, или наоборот?
Наоборот будет красивше, да, давайте в `0xffff00a7` хранить младший байт — `0x01`, а в
старшем адресе старший байт — `0x00`. Чтобы определить чётность или не чётность числа,
достаточно прочитать только один первый байт из памяти, удобно-ж, не правда-ли?
Если без шуток, разницы принципиальной нет. Продолжение чтения —
[Endian Comparision](http://bear.ces.cwru.edu/eecs_314/endian_comparison.html)

[^14]: Для некоторых конверсий реализация будет простой. Если конвертировать число
`int64` в `uint64` то для runtime не будет никакой работы. Компилятору достаточно
генерировать ассемблерные инструкции для работы со _знаковыми числами_ до конвертации и
_инструкции для работы с беззнаковыми_ числами после этой конвертации. Например, оператор
`>>` должен генерировать инструкцию арифметического сдвига
[SAR — shift arithmetic right](https://www.felixcloutier.com/x86/sal:sar:shl:shr.html),
то есть с расширением знака, при работе
со знаковым числом. А при работе с беззнаковым, сдвиг должен быть логическим
`SHR — shift right`. Поэтому для сдвига переменной типа `int64` компилятор использует `SAR`.
После конверсии значения в `uint64` использует `SHR`. Содержимое памяти в процессе конверсии 
не изменяется, поэтому рантайм языка Go в этой процедуре принимать участие не будет и
можно сказать что _этот процесс не имеет накладных расходов_ во время работы программы.

[^15]: По первым отзывам на статью я понял что углубляться в ассемблер — плохо. Читаемость
падает. В результате я сильно сократил ассемблерные вставки и их объяснение. В этом случае
не нужно знать как именно строка становится аргументом. Но в сноске поговорим подробнее.
В Go недавно изменили способ передачи аргументов в функции. Как видите
сейчас на `amd64` они передаются через регистры (блоки памяти внутри микропроцессора),
. Первый аргумент (слева направо в сигнатуре функции) через регистр `AX`, второй через `BX`,
третий и четвёртый `CX` и `DI` соответственно. C возвращаемыми аргументами ситуация
аналогична.
`slicebytetostring` возвращает только один параметр типа string. Но строка в Go это
два поля — указатель на последовательность байт и значение длины строки. Поэтому, чтобы
передать _одну_ строку, нужно ровно две ячейки памяти.
Итак, вызов функции приведёт к тому что в регистре `AX` будет указатель на последовательность
байт новой строки, а в регистре `BX` её длина. Функция `strconv.ParseUint` требует
ещё два аргумента, базу системы счисления и размер в числа в битах. Строка уже находится
в регистрах `AX/BX` (как удобно), а вот `CX` и `DI` нужно заполнить.
Это не полное описание процесса. Подробнее в статье
[Calling Conversion](https://tip.golang.org/src/cmd/compile/abi-internal).

[^16]: Сылки на слайс более не существует, но есть ссылка на заголовок строки. В заголовке
слайса есть ещё ёмкость. Вопрос на самостоятельное изучение, что станет с теми восьмью байтами,
в которых хранился емкость, после того как заголовок строки (если он находился в куче) будет
собран сборщиком мусора.

[^17]: Например, в протоколе TCP есть функциональность
[окна](https://en.wikipedia.org/wiki/TCP_window_scale_option)
приёма/передачи когда одна сторона уведомляет вторую о том, сколько
ещё байт она может принять.

[^18]: Текст. Текст или строки, обычно, чутка более сложные сущности чем просто
последовательности байт. Текст это закодированные символы, и, очевидно,
не всё множество последовательностей байт это валидный текст. Поэтому
некоторые языки, при преобразовании массива байт в строку могут производить
более сложные операции. Так же этот процесс не всегда успешен, например, если мы
не смогли распознать последовательность байт как символы в той или иной
кодировке. Язык Go поступает проще — строки могут содержать произвольный набор
байт. Поэтому вся эта мишура с валидацией ложится на плечи программистов, но
только если им это нужно. То есть те строки что есть в Go это не совсем те строки что вы можете
видеть, например, в Rust. Нет строк  — нет проблем!

[^19]: За вас это решение принимает компилятор. Так как Go является языком со сборщиком мусора
для программиста нет принципиальной разницы где хранить то или иное значение. Посмотрим на С,
вызывая функцию malloc вы явно просите runtime (да, даже у C он есть) выделить вам память в куче.
Такой подход может быть нужен для программ жёстко контролирующих используемую память, встраиваемые
устройства — там памяти просто мало, ядра операционных систем — те гарантии, что ОС предоставляет
для пользовательских процессов по части памяти, она не может обеспечить для самой себя. 
Но в Go вы не можете предполагать, где компилятор разместит значение.
