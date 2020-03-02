Помоги компилятору помочь тебе
==============================

Предисловие
-----------

Современные компиляторы обладают огромным количеством диагностик. И удивительно, что очень малая их часть включена по умолчанию.

Огромное количество претензий, которые предъявляют к языку C++ в этих ваших интернетах, — про сложность, небезопасность, стрельбу по ногам и т.п., — относятся как раз к тем случаям, когда люди просто не знают о том, что можно решить эти проблемы лёгким движением пальцев по клавиатуре.

Давайте же исправим эту вопиющую несправедливость, и прольём свет истины на возможности компилятора по предотвращению ошибок.

Содержание
----------

1.  [Ода компилятору](#ода-компилятору)
2.  [Игнорировать нельзя исправить](#игнорировать-нельзя-исправить)
3.  [-Wall](#-wall)
4.  [-Wextra](#-wextra)
5.  [-Wpedantic](#-wpedantic)
6.  [Отдельные предупреждения](#отдельные-предупреждения)
7.  [-Werror](#-werror)
8.  [Заключение](#заключение)
9.  [Ссылки](#ссылки)

[Ода компилятору](#содержание)
------------------------------

Компилятор – лучший друг плюсовика. Компилятор — это не просто транслятор формального человекочитаемого языка в машинные коды. Компилятор — лучший помощник в написании программ.

Важная (и не единственная) помощь, которую оказывает компилятор — поиск ошибок. И я говорю не об опечатках, несовпадении типов и прочих синтаксических ошибках. Я говорю об огромном наборе ошибок, которые можно выловить с помощью механизма предупреждений.

Часто встречаю мнение о том, что предупреждений слишком много, они дают ложноположительные результаты, мешают работать, замыливают глаз, отвлекают от "настоящих" ошибок и т.п. Такое действительно бывает, но это большая редкость.

[Игнорировать нельзя исправить](#содержание)
--------------------------------------------

Большинство предупреждений — это не "бзик" компилятора, который можно просто проигнорировать. Предупреждение — это потенциальная ошибка. Предупреждение — это сигнал от компилятора о том, что написано одно, а требуется, возможно, что-то совершенно иное.

Поэтому программист должен помочь компилятору понять, как трактовать спорную ситуацию. То есть либо _поправить свою ошибку_, либо сообщить компилятору явно о том, что нужно верить программисту и _делать именно то, что написано_. Причём это поможет не только компилятору, но и человеку, который будет читать код. Лишний `static_cast` или пара скобок будут явно свидетельствовать о том, что имелось в виду именно то, что написано.

Далее я расскажу о наиболее важных на мой взгляд предупреждениях и покажу, какие ошибки можно отловить с их помощью.

Надеюсь, что данное не слишком занимательное чтиво поможет правильно поставить запятую в заголовке этого раздела.

> Сразу хочу оговориться, что далее речь пойдёт исключительно о языке C++ и компиляторе GCC (впрочем, подавляющая часть информации актуальна и для компилятора Clang). Информацию о других компиляторах и языках придётся искать в соответствующих справочниках.

[-Wall](#содержание)
--------------------

`-Wall` — это "агрегатор" базовых предупреждений. В языке C++ он включает в себя длинный перечень предупреждений, каждое из которых будет рассмотрено отдельно (на самом деле, рассмотрены будут не все, а только те, которые непосредственно помогают выявлять ошибки).

> __Важно:__
>
> Несмотря на название, этот флаг включает далеко не все предупреждения, которые умеет обнаруживать компилятор.

### -Waddress

Предупреждает о странной работе с адресами. Например, об использование адреса функции в условном выражении. Такое может произойти, если забыть поставить скобки после имени функции:

```c++
bool func () {return false;}

int main ()
{
    if (func)
    {
        return 1;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:9: warning: the address of 'bool func()' will never be NULL [-Waddress]
    5 |     if (func)
      |         ^~~~
```

Также этот флаг может спасти от типичной ошибки новичка — сравнения строкового литерала с адресом. Очевидно, программист хотел сравнить строки, но в результате сравнил два указателя:

```c++
int main ()
{
    const char * a = "abc";
    if (a == "abc")
    {
        return 0;
    }
}
```

Компилятор бдит:

```
prog.cc: In function 'int main()':
prog.cc:4:11: warning: comparison with string literal results in unspecified behavior [-Waddress]
    4 |     if (a == "abc")
      |         ~~^~~~~~~~
```

### -Warray-bounds=1

Предупреждает о выходе за пределы массивов. Используется только вместе с `-O2`.

```c++
int main()
{
    int a[5] = {1, 2, 3, 4, 5};
    return a[5];
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:15: warning: array subscript 5 is above array bounds of 'int [5]' [-Warray-bounds]
    4 |     return a[5];
      |            ~~~^
prog.cc:3:9: note: while referencing 'a'
    3 |     int a[5] = {1, 2, 3, 4, 5};
      |         ^
```

### -Wbool-compare

Предупреждает о сравнении булева выражения с целым числом, которое нельзя трактовать как булево:

```c++
int main ()
{
    int n = 5;

    if ((n > 1) == 2)
    {
        return 0;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:17: warning: comparison of constant '2' with boolean expression is always false [-Wbool-compare]
    5 |     if ((n > 1) == 2)
      |         ~~~~~~~~^~~~
```

### -Wbool-operation

Предупреждает о подозрительных операциях с булевыми выражениями. Например, о побитовом отрицании:

```c++
int main ()
{
    bool b = true;
    auto c = ~b;
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:15: warning: '~' on an expression of type 'bool' [-Wbool-operation]
    4 |     auto c = ~b;
      |               ^
prog.cc:4:15: note: did you mean to use logical not ('!')?
```

Что касается инкрементов и декрементов булевых переменных, то в C++17 это просто ошибки, безо всяких предупреждений.

### -Wcatch-value

Предупреждает о обработчиках исключений, которые принимают полиморфные объекты по значению:

```c++
struct A
{
    virtual ~A () {};
};

struct B: A{};

int main ()
{
    try {}
    catch (A) {}
}
```

```
prog.cc: In function 'int main()':
prog.cc:11:12: warning: catching polymorphic type 'struct A' by value [-Wcatch-value=]
   11 |     catch (A) {}
      |            ^
```

Есть и более сильные версии предупреждения: `-Wcatch-value=n` (см. справку к компилятору).

### -Wchar-subscripts

Предупреждает об обращении к массиву по индексу, тип которого `char`. А ведь `char` является знаковым на многих машинах:

```c++
int main ()
{
    int a[] = {1, 2, 3, 4};
    char index = 'a' - 'b';
    return a[index];
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:14: warning: array subscript has type 'char' [-Wchar-subscripts]
    5 |     return a[index];
      |              ^~~~~
```

### -Wcomment

Предупреждает о наличии последовательности, начинающей новый комментарий (`/*`), внутри многострочного комментария, либо о разрыве строки при помощи обратного слеша внутри однострочного комментария.

```c++
int main ()
{
    /* asd /* fgh */
    //qwe\
    rty
}
```

```
prog.cc:3:8: warning: "/*" within comment [-Wcomment]
    3 |     /* /* */
      |
prog.cc:4:5: warning: multi-line comment [-Wcomment]
    4 |     //ssd\
      |     ^
```

### -Wint-in-bool-context

Предупреждает о подозрительном использовании целых чисел там, где ожидаются булевы выражения, например, в условных выражениях:

```c++
int main ()
{
    int a = 5;
    if (a <= 4 ? 2 : 3)
    {
        return 0;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:16: warning: ?: using integer constants in boolean context, the expression will always evaluate to 'true' [-Wint-in-bool-context]
    4 |     if (a <= 4 ? 2 : 3)
      |         ~~~~~~~^~~~~~~
```

Другой пример — операции побитового сдвига в булевом контексте. Вполне вероятно, что здесь произошла опечатка, и имелся в виду не сдвиг, а сравнение:

```c++
int main ()
{
    for (auto a = 0; 1 << a; a++);
}
```

```
prog.cc: In function 'int main()':
prog.cc:3:24: warning: '<<' in boolean context, did you mean '<' ? [-Wint-in-bool-context]
    3 |     for (auto a = 0; 1 << a; a++);
      |                      ~~^~~~
```

А также сообщает о любых видах умножения в булевом контексте.

```c++
int main ()
{
    int a = 5;
    if (a * 5);
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:11: warning: '*' in boolean context, suggest '&&' instead [-Wint-in-bool-context]
    4 |     if (a * 5);
      |         ~~^~~
```

### -Winit-self

Предупреждает об самоинициализации переменных. Используется только вместе с флагом [`-Wuninitialized`](#-wuninitialized).

```c++
int main ()
{
    int i = i;
    return i;
}
```

```
prog.cc: In function 'int main()':
prog.cc:3:9: warning: 'i' is used uninitialized in this function [-Wuninitialized]
    3 |     int i = i;
      |         ^
```

### -Wlogical-not-parentheses

Предупреждает об использовании логического отрицания в левой части сравнения. При этом если правая часть сравнения является сама по себе булевым выражением, то предупреждения не будет.

Используется для того, чтобы отлавливать подозрительные конструкции вроде следующей:

```c++
int main ()
{
    int a = 9;
    if (!a > 1);
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:12: warning: logical not is only applied to the left hand side of comparison [-Wlogical-not-parentheses]
    5 |     if (!a > 1);
      |            ^
prog.cc:5:9: note: add parentheses around left hand side expression to silence this warning
    5 |     if (!a > 1);
      |         ^~
      |         ( )
```

Традиционный способ сообщить компилятору, что так и было задумано — поставить скобки, о чём и сообщает компилятор.

### -Wmaybe-uninitialized

Предупреждает о том, что существует возможность использования непроинициализированной переменной.

```c++
int main (int argc, const char * argv[])
{
    int x;
    switch (argc)
    {
        case 1: x = 1;
            break;
        case 2: x = 4;
            break;
        case 3: x = 5;
    }
    return x;
}
```

```
prog.cc: In function 'int main(int, const char**)':
prog.cc:3:9: warning: 'x' may be used uninitialized in this function [-Wmaybe-uninitialized]
    3 |     int x;
      |         ^
```

В данном конкретном случае решается с помощью конструкции `default`:

```c++
int main (int argc, const char * argv[])
{
    int x;
    switch (argc)
    {
        case 1: x = 1;
            break;
        case 2: x = 4;
            break;
        case 3: x = 5;
        default:
            x = 17;
    }
    return x;
}
```

### -Wmemset-elt-size

Предупреждает о подозрительных вызовах функции `memset`, когда первый аргумент — это массив, а третий аргумент — количество элементов в массиве, но не количество байт, занимаемой этим массивом в памяти.

```c++
#include <cstring>

int main ()
{
    constexpr auto size = 20ul;
    int a[size];
    std::memset(a, 0, size);
}
```

```
prog.cc: In function 'int main()':
prog.cc:7:27: warning: 'memset' used with length equal to number of elements without multiplication by element size [-Wmemset-elt-size]
    7 |     std::memset(a, 0, size);
      |                           ^
```

### -Wmemset-transposed-args

Предупреждает о том, что пользователь, вероятно, перепутал порядок аргументов в функции `memset`:

```c++
#include <cstring>

int main ()
{
    constexpr auto size = 20ul;
    int a[size];
    std::memset(a, size, 0);
}
```

```
prog.cc: In function 'int main()':
prog.cc:7:27: warning: 'memset' used with constant zero length parameter; this could be due to transposed parameters [-Wmemset-transposed-args]
    7 |     std::memset(a, size, 0);
      |                           ^
```

### -Wmisleading-indentation

Предупреждает о том, что отступы в коде не отражают структуру этого кода. Особенно это актуально для конструкций `if`, `else`, `while` и `for`. Пример:

```c++
int main ()
{
    int x;
    if (true)
        x = 3;
        return x;
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:5: warning: this 'if' clause does not guard... [-Wmisleading-indentation]
    4 |     if (true)
      |     ^~
prog.cc:6:9: note: ...this statement, but the latter is misleadingly indented as if it were guarded by the 'if'
    6 |         return x;
      |         ^~~~~~
```

См. также [`-Wempty-body`](#-wempty-body).

### -Wmissing-attributes

Предупреждает о ситуации, когда специализация шаблона объявлена не с тем же списком атрибутов, что и оригинальный шаблон.

```c++
template <class T>
T* __attribute__ ((malloc, alloc_size (1)))
foo (long);

template <>
void* __attribute__ ((malloc))
foo<void> (long);

int main ()
{
}
```

```
prog.cc:7:1: warning: explicit specialization 'T* foo(long int) [with T = void]' may be missing attributes [-Wmissing-attributes]
    7 | foo<void> (long);
      | ^~~~~~~~~
prog.cc:3:1: note: missing primary template attribute 'alloc_size'
    3 | foo (long);
      | ^~~
```

### -Wmultistatement-macros

Предупреждает о макросах, состоящих из нескольких инструкций, и используемых в выражениях `if`, `else`, `while` и `for`. В такой ситуации под действие выражений попадает только первая инструкция макроса, и это, вероятно, ошибка:

```c++
#define INCREMENT x++; y++

int main ()
{
    int x = 0;
    int y = 0;
    if (true)
        INCREMENT;
}
```

```
prog.cc: In function 'int main()':
prog.cc:1:19: warning: macro expands to multiple statements [-Wmultistatement-macros]
    1 | #define INCREMENT x++; y++
      |                   ^
prog.cc:8:9: note: in expansion of macro 'INCREMENT'
    8 |         INCREMENT;
      |         ^~~~~~~~~
prog.cc:7:5: note: some parts of macro expansion are not guarded by this 'if' clause
    7 |     if (true)
      |     ^~
```

См. также [`-Wmisleading-indentation`](#-wmisleading-indentation).

### -Wnonnull

Предупреждает о передаче нулевого указателя в функцию, аргументы которой помечены атрибутом `nunnull`.

```c++
void f (int * ptr) __attribute__((nonnull));

int main ()
{
    f(nullptr);
}

void f (int *) {}
```

```
prog.cc: In function 'int main()':
prog.cc:5:14: warning: null argument where non-null required (argument 1) [-Wnonnull]
    5 |     f(nullptr);
      |              ^
```

### -Wnonnull-compare

Предупреждает о сравнении с нулём аргумента функции, помеченного атрибутом `nonnull`.

```c++
bool f (int * ptr) __attribute__((nonnull));

int main ()
{
    f(nullptr);
}

bool f (int * p)
{
    return p == nullptr;
}
```

```
prog.cc: In function 'bool f(int*)':
prog.cc:10:17: warning: nonnull argument 'p' compared to NULL [-Wnonnull-compare]
   10 |     return p == nullptr;
      |                 ^~~~~~~
```

### -Wparentheses

Типичный случай — опечатались, и вместо равенства написали присвоение:

```c++
int main ()
{
    int x = 5;
    if (x = 4)
    {
        x = 3;
    }
}
```

Компилятор, естественно, сомневается:

```
prog.cc: In function 'int main()':
prog.cc:4:11: warning: suggest parentheses around assignment used as truth value [-Wparentheses]
    4 |     if (x = 4)
      |         ~~^~~
```

Либо исправляем код, либо убеждаем компилятор в том, что мы хотели именно присвоение:

```c++
int main ()
{
    int x = 5;
    if ((x = 4))
    {
        x = 3;
    }
}
```

### -Wpessimizing-move

Иногда явная попытка переноса может ухудшить производительность. Пример:

```c++
#include <utility>

struct A {};

A f ()
{
    A a;
    return std::move(a);
}

int main ()
{
    f();
}
```

```
prog.cc: In function 'A f()':
prog.cc:8:21: warning: moving a local object in a return statement prevents copy elision [-Wpessimizing-move]
    8 |     return std::move(a);
      |            ~~~~~~~~~^~~
prog.cc:8:21: note: remove 'std::move' call
```

### -Wreorder

Предупреждает о том, что порядок инициализации членов класса не соответствует порядку их объявления. Поскольку компилятор может переупорядочить инициализацию этих членов, результат может быть неочевидным.

```c++
struct A
{
    int i;
    int j;

    A(): j(0), i(1)
    {
    }
};

int main () {}
```

```
prog.cc: In constructor 'A::A()':
prog.cc:4:9: warning: 'A::j' will be initialized after [-Wreorder]
    4 |     int j;
      |         ^
prog.cc:3:9: warning:   'int A::i' [-Wreorder]
    3 |     int i;
      |         ^
prog.cc:6:5: warning:   when initialized here [-Wreorder]
    6 |     A(): j(0), i(1)
      |     ^
```

### -Wreturn-type

Предупреждает о том, что из функции не вернули заявленный результат:

```c++
int f ()
{
}

int main () {}
```

```
prog.cc: In function 'int f()':
prog.cc:3:1: warning: no return statement in function returning non-void [-Wreturn-type]
    3 | }
      | ^
```

### -Wsequence-point

Сообщает о подозрительных операциях относительно точек следования. Любимый пример (никогда так не делайте):

```c++
int main ()
{
    int a = 6;
    ++a = a++;
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:12: warning: operation on 'a' may be undefined [-Wsequence-point]
    4 |     ++a = a++;
      |           ~^~
```

### -Wsign-compare

Одно из важнейших предупреждений. Сообщает о сравнении знаковых и беззнаковых чисел, которое может произвести некорректный результат из-за неявных преобразований. К примеру, отрицательное знаковое число неявно приводится к беззнаковому и внезапно становится положительным:

```c++
int main ()
{
    short a = -5;
    unsigned b = 4;
    if (a < b)
    {
        return 0;
    }

    return 1;
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:11: warning: comparison of integer expressions of different signedness: 'short int' and 'unsigned int' [-Wsign-compare]
    5 |     if (a < b)
      |         ~~^~~
```

### -Wsizeof-pointer-div

Предупреждает о подозрительном делении друг на друга двух результатов выражения `sizeof`, когда размер указателя делиться на размер объекта. Обычно это бывает, когда пытаются вычислить размер массива, но вместо массива по ошибке берут указатель:

```c++
int main ()
{
    const char * a = "12345";
    auto size = sizeof (a) / sizeof (a[0]);
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:28: warning: division 'sizeof (const char*) / sizeof (const char)' does not compute the number of array elements [-Wsizeof-pointer-div]
    4 |     auto size = sizeof (a) / sizeof (a[0]);
      |                 ~~~~~~~~~~~^~~~~~~~~~~~~~~
prog.cc:3:18: note: first 'sizeof' operand was declared here
    3 |     const char * a = "12345";
      |                  ^
```

### -Wsizeof-pointer-memaccess

Предупреждает о подозрительных параметрах, передаваемых в строковые функции и функции для работы с памятью (`str...`, `mem...` и т.п.), и использующих оператор `sizeof`. Например:

```c++
#include <cstring>

int main ()
{
    char a[10];
    const char * s = "qwerty";
    std::memcpy (a, s, sizeof(s));
}
```

```
prog.cc: In function 'int main()':
prog.cc:7:24: warning: argument to 'sizeof' in 'void* memcpy(void*, const void*, size_t)' call is the same expression as the source; did you mean to provide an explicit length? [-Wsizeof-pointer-memaccess]
    7 |     std::memcpy (a, s, sizeof(s));
      |                        ^~~~~~~~~
```

### -Wstrict-aliasing

Каламбур типизации (strict aliasing) — это отдельная большая тема для разговора. Предлагаю читателю найти литературу по этой теме самостоятельно.

В общем, это тоже крайне полезное предупреждение.

### -Wswitch

Предупреждает о том, что не все элементы перечисления задействованы в конструкции `switch`:

```c++
enum struct enum_t
{
    a, b, c
};

int main ()
{
    enum_t e = enum_t::a;
    switch (e)
    {
        case enum_t::a:
        case enum_t::b:
            return 0;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:9:12: warning: enumeration value 'c' not handled in switch [-Wswitch]
    9 |     switch (e)
      |            ^
```

### -Wtautological-compare

Предупреждает о бессмысленном сравнении переменной с самой собой:

```c++
int main ()
{
    int i = 1;
    if (i > i);
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:11: warning: self-comparison always evaluates to false [-Wtautological-compare]
    4 |     if (i > i);
      |         ~ ^ ~
```

Кроме того, сообщает о сравнениях при участии битовых операций, которые имеют всегда один и тот же результат (всегда истинно или всегда ложно):

```c++
int main ()
{
    int i = 1;
    if ((i & 16) == 10);
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:18: warning: bitwise comparison always evaluates to false [-Wtautological-compare]
    4 |     if ((i & 16) == 10);
      |         ~~~~~~~~ ^~ ~~
```

### -Wtrigraphs

Предупреждает о наличии [триграфов](https://ru.wikipedia.org/wiki/Триграф_(языки_Си)), которые могут изменить смысл программы. Не сообщается о триграфах в теле комментария, за исключением случаев, когда триграф трактуется как перевод строки.

См. также [`-Wcomment`](#-wcomment).

### -Wuninitialized

Предупреждает об использовании переменных и членов класса, которые не были проинициализированы:

```c++
int main ()
{
    int x;
    return x;
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:12: warning: 'x' is used uninitialized in this function [-Wuninitialized]
    4 |     return x;
      |            ^
```

### -Wunused-function

Предупреждает о том, что статическая функция объявлена, но не определена, либо о том, что статическая функция, не помеченная как `inline`, не используется.

### -Wunused-variable

Предупреждает о том, что переменная не используется.

```c++
int main ()
{
    int x = 0;
}
```

```
prog.cc: In function 'int main()':
prog.cc:3:9: warning: unused variable 'x' [-Wunused-variable]
    3 |     int x = 0;
      |         ^
```

Для того, чтобы помочь компилятору понять, что так и задумывалось, можно использовать конструкцию `static_cast<void>(...)`:

```c++
int main ()
{
    int x = 0;
    static_cast<void>(x);
}
```

[-Wextra](#содержание)
----------------------

"Агрегатор" дополнительных предупреждений. Включает много интересного, чего нет в `-Wall` (как и в случае с `-Wall`, рассмотрены будут не все возможности).

### -Wempty-body

Предупреждает о пустом теле условных выражений или цикла `do-while`. Чаще всего это говорит об опечатке, меняющей логику программы:

```c++
int main ()
{
    if (true);
    {
        return 1;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:3:14: warning: suggest braces around empty body in an 'if' statement [-Wempty-body]
    3 |     if (true);
      |              ^
```

См. также [`-Wmisleading-indentation`](#-wmisleading-indentation).

### -Wimplicit-fallthrough

Предупреждает о "проваливании" в операторе `switch`:

```c++
int main ()
{
    int x = 7;
    int a;
    switch (x)
    {
        case 1:
            a = 1;
            break;
        case 2:
            a = 2;
        case 3:
            a = 3;
            break;
        default:
            a = 0;
    }
    return a;
}
```

Компилятор предполагает, что программист забыл `break`, и `case 2` не должен проваливаться:

```
prog.cc: In function 'int main()':
prog.cc:11:15: warning: this statement may fall through [-Wimplicit-fallthrough=]
   11 |             a = 2;
      |             ~~^~~
prog.cc:13:9: note: here
   13 |         case 3:
      |         ^~~~
```

В C++17 для обозначения явного намерения появился специальный атрибут — `fallthrough`:

```c++
int main ()
{
    int x = 7;
    int a;
    switch (x)
    {
        case 1:
            a = 1;
            break;
        case 2:
            a = 2;
            [[fallthrough]];
        case 3:
            a = 3;
            break;
        default:
            a = 0;
    }
    return a;
}
```

### -Wmissing-field-initializers

Предупреждает о том, что отдельные члены структуры не были проинициализированы. Скорее всего это просто забыли сделать:

```c++
struct S
{
    int f;
    int g;
    int h;
};

int main ()
{
    S s{3, 4};
}
```

```
prog.cc: In function 'int main()':
prog.cc:10:13: warning: missing initializer for member 'S::h' [-Wmissing-field-initializers]
   10 |     S s{3, 4};
      |             ^
```

### -Wredundant-move

Предупреждает о ненужном вызове `std::move` в случаях, когда компилятор сам сделает всё, что нужно:

```c++
#include <utility>

struct S {};

S f (S s)
{
    return std::move(s);
}

int main ()
{
    auto s = f(S{});
}
```

```
prog.cc: In function 'S f(S)':
prog.cc:7:21: warning: redundant move in return statement [-Wredundant-move]
    7 |     return std::move(s);
      |            ~~~~~~~~~^~~
prog.cc:7:21: note: remove 'std::move' call
```

### -Wtype-limits

Предупреждает о сравнениях, которые всегда имеют один и тот же результат. Например, когда беззнаковое число проверяется на неотрицательность. Если программист делает такую проверку, то, видимо, предполагает, что число в теории может быть отрицательным, однако, это не так. Видимо, он где-то ошибся:

```c++
int main ()
{
    unsigned x = 17;
    if (x >= 0)
    {
        return 1;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:11: warning: comparison of unsigned expression >= 0 is always true [-Wtype-limits]
    4 |     if (x >= 0)
      |         ~~^~~~
```

### -Wshift-negative-value

Предупреждает об операциях сдвига для отрицательных значений. Отрицательными могут быть только знаковые числа, а для них это некорректно:

```c++
int main ()
{
    const int x = -7;
    return x << 4;
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:17: warning: left shift of negative value [-Wshift-negative-value]
    4 |     return x << 4;
      |                 ^
```

### -Wunused-parameter

Предупреждает о неиспользуемом параметре функции. Возможно, про него просто забыли, и в этом случае функция может работать некорректно.

```c++
void f (int x) {}

int main ()
{
}
```

```
prog.cc: In function 'void f(int)':
prog.cc:1:13: warning: unused parameter 'x' [-Wunused-parameter]
    1 | void f (int x) {}
      |         ~~~~^
```

В C++17 для явного выражения намерения существует атрибут `maybe_unused`:

```c++
void f ([[maybe_unused]] int x) {}

int main ()
{
}
```

### -Wunused-but-set-parameter

Предупреждает о том, что в параметр функции было записано значение, но после этого он ни разу не использовался. Возможно, про него снова забыли:

```c++
void f (int x)
{
    x = 7;
}

int main ()
{
}
```

```
prog.cc: In function 'void f(int)':
prog.cc:1:13: warning: parameter 'x' set but not used [-Wunused-but-set-parameter]
    1 | void f (int x)
      |         ~~~~^
```

[-Wpedantic](#содержание)
-------------------------

`-Wall` и `-Wextra` — это не всё, на что способен компилятор.

В дополнение к ним существует флаг `-Wpedantic` (он же `-pedantic`), который проверяет соответствие кода стандарту ISO C++, сообщает об использовании запрещённых расширений, о наличии лишних точек с запятой, нехватке переноса строки в конце файла и прочих полезных штуках.

[Отдельные предупреждения](#содержание)
---------------------------------------

Но и это ещё не всё. Есть несколько флагов, которые почему-то не входят ни в один из "аргегаторов", но крайне важны и полезны.

### -Wctor-dtor-privacy

Предупреждает о том, что класс выглядит неиспользуемым, потому что конструкторы и деструкторы закрыты, а друзей и открытых статических функций членов у него нет.

```c++
class base
{
    base () {};
    ~base() {};
};

int main ()
{
}
```

```
prog.cc:1:7: warning: 'class base' only defines a private destructor and has no friends [-Wctor-dtor-privacy]
    1 | class base
      |       ^~~~
```

Аналогично, сообщает, что у класса есть закрытые функции-члены, а открытых нет ни одной.

### -Wnon-virtual-dtor

Предупреждает о том, что у класса есть виртуальные функции-члены, но деструктор при этом не виртуальный. Очень сложно представить себе такой класс. Вероятнее всего, это ошибка.

```c++
struct base
{
    virtual void f (int) {}
    ~base() {};
};

int main ()
{
}
```

```
prog.cc:1:8: warning: 'struct base' has virtual functions and accessible non-virtual destructor [-Wnon-virtual-dtor]
    1 | struct base
      |        ^~~~
```

### -Wold-style-cast

Предупреждает о приведении типов в стиле языка C. В плюсах есть прекрасные и ужасные `static_cast`, `dynamic_cast`, `reinterpret_cast` и `const_cast`, которые более локальны и более описательны. Сишный способ слишком сильный и — о, ужас, — небезопасный. Лучше его не использовать вообще.

### -Woverloaded-virtual

Предупреждает о попытке в классе-наследнике переопределить виртуальную функцию базового класса:

```c++
struct base
{
    virtual void f (int) {}
};

struct derived: base
{
    void f () {};
};

int main ()
{
}
```

```
prog.cc:3:18: warning: 'virtual void base::f(int)' was hidden [-Woverloaded-virtual]
    3 |     virtual void f (int) {}
      |                  ^
prog.cc:8:10: warning:   by 'void derived::f()' [-Woverloaded-virtual]
    8 |     void f () {};
      |          ^
```

### -Wsign-promo

Крайне полезный флаг. Предупреждает о неочевидном выборе перегруженной функции:

```c++
void f (int) {}
void f (unsigned) {}

int main ()
{
    unsigned short x = 7;
    f(x);
}
```

```
prog.cc: In function 'int main()':
prog.cc:7:8: warning: passing 'short unsigned int' chooses 'int' over 'unsigned int' [-Wsign-promo]
    7 |     f(x);
      |        ^
prog.cc:7:8: warning:   in call to 'void f(int)' [-Wsign-promo]
```

Вероятнее всего, хотели-таки позвать вторую перегрузку, а не первую. А если всё-таки первую, то будьте любезны сказать об этом явно.

### -Wduplicated-branches

Предупреждает о том, что ветви `if` и `else` одинаковы:

```c++
int main ()
{
    if (true)
        return 0;
    else
        return 0;
}
```

```
prog.cc: In function 'int main()':
prog.cc:3:5: warning: this condition has identical branches [-Wduplicated-branches]
    3 |     if (true)
      |     ^~
```

Условный оператор `?:` также под прицелом:

```c++
int main ()
{
    auto x = true ? 4 : 4;
}
```

```
prog.cc: In function 'int main()':
prog.cc:3:19: warning: this condition has identical branches [-Wduplicated-branches]
    3 |     auto x = true ? 4 : 4;
      |              ~~~~~^~~~~~~
```

Для меня абсолютная загадка, почему этот флаг не включён не то, что в `-Wall`, а вообще по умолчанию.

См. также [`-Wduplicated-cond`](#-wduplicated-cond).

### -Wduplicated-cond

Предупреждает об одинаковых условиях в цепочках `if-else-if`:

```c++
int main ()
{
    auto x = 6;
    if (x > 7) {return 0;}
    else if (x > 7) {return 1;}
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:10: warning: duplicated 'if' condition [-Wduplicated-cond]
    5 |     else if (x > 7) {return 1;}
      |          ^~
prog.cc:4:5: note: previously used here
    4 |     if (x > 7) {return 0;}
      |     ^~
```

См. также [`-Wduplicated-branches`](#-wduplicated-branches).

### -Wfloat-equal

Предупреждает о проверке на равенство между двумя числами с плавающей точкой. Скорее всего, это ошибка, и сравнение нужно проводить с заданной точностью.

Если же требуется именно сравнить на равенство (такое редко, но бывает), то можно использовать [`std::equal_to`](https://en.cppreference.com/w/cpp/utility/functional/equal_to), который под предупреждение не попадает.

### -Wshadow=compatible-local

Полезная опция, которая не даёт перекрыть локальную переменную другой локальной переменной при условии, что они имеют совместимые типы.

### -Wcast-qual

Предупреждает о преобразовании указателя, при котором сбрасываются квалификаторы. Например, чтобы случайно не потерять `const`.

### -Wconversion

Очень, очень, очень важный флаг. Он предупреждает об огромном количестве неявных сужающих (то есть потенциально приводящих к потере информации) преобразований, которые могут быть следствием ошибки программиста. Например:

```c++
int main ()
{
    double x = 4.5;
    int y = x;
}
```

```
prog.cc: In function 'int main()':
prog.cc:12:13: warning: conversion from 'double' to 'int' may change value [-Wfloat-conversion]
   12 |     int y = x;
      |             ^
```

Если вы раньше никогда не включали этот флаг, то будет будет интересно.

### -Wzero-as-null-pointer-constant

Предупреждает об использовании целочисленного нуля вместо `nullptr`.

### -Wextra-semi

Флаг для педантов. Сообщает о лишней точке с запятой после определения функции-члена.

### -Wsign-conversion

Как и `-Wconversion` помогает предотвратить большое количество неявных преобразований, которые запросто могут быть ошибками:

```c++
int main ()
{
    signed si = -8;
    unsigned ui;
    ui = si;
}
```

```
prog.cc: In function 'int main()':
prog.cc:5:10: warning: conversion to 'unsigned int' from 'int' may change the sign of the result [-Wsign-conversion]
    5 |     ui = si;
      |          ^~
```

### -Wlogical-op

Предупреждает о подозрительных логических выражениях. Например, когда вместо побитового "И" поставили логическое "И", или логическое выражение имеет одинаковые операнды:

```c++
int main ()
{
    int a = 8;
    if (a < 0 && a < 0)
    {
        return 1;
    }
}
```

```
prog.cc: In function 'int main()':
prog.cc:4:15: warning: logical 'and' of equal expressions [-Wlogical-op]
     if (a < 0 && a < 0)
         ~~~~~~^~~~~~~~
```

[-Werror](#содержание)
----------------------

С этого, вообще говоря, стоило бы начать. Данный флаг делает все предупреждения ошибками. Код не скомпилируется при наличии хотя бы одного предупреждения.

Без этого флага всё остальное имеет мало смысла. Но если понять и принять мысль о том, что предупреждение — это что-то подозрительное, и их быть не должно, то именно этот флаг и позволит поддерживать код в чистоте.

> В дополнение к `-Werror` существует флаг `-pedantic-errors`, который не эквивалентен комбинации `-Wpedantic -Werror`.
>
> Да, всё непросто.

[Заключение](#содержание)
-------------------------

Резюмируя, для компилятора GCC (Clang кое-что из этого не умеет, к сожалению) я рекомендую включать следующий минимальный набор флагов, по необходимости дополняя его более сложными диагностиками.

```
-Werror
-pedantic-errors

-Wall
-Wextra
-Wpedantic

-Wcast-align
-Wcast-qual
-Wconversion
-Wctor-dtor-privacy
-Wduplicated-branches
-Wduplicated-cond
-Wextra-semi
-Wfloat-equal
-Wlogical-op
-Wnon-virtual-dtor
-Wold-style-cast
-Woverloaded-virtual
-Wredundant-decls
-Wsign-conversion
-Wsign-promo
```

Да, такой список флагов может породить большое количество ошибок, которые поначалу могут показаться излишними. Но явное лучше неявного. Если знаешь, что делаешь — делай. Но делай это так, чтобы всем было понятно, что именно так ты и хотел. Поработав таким образом хотя бы неделю, вы поймёте, насколько это прекрасно, и уже не сможете вернуться обратно.

Любите ваш компилятор и помогайте ему помогать вам писать программы.

[Ссылки](#содержание)
---------------------

1. [Документация к компилятору GCC](https://gcc.gnu.org/onlinedocs/gcc-9.2.0/gcc/)
2. [Документация к компилятору Clang](https://clang.llvm.org/docs/UsersManual.html)
