Ленивые диапазоны и стирание типов
==================================

Введение
--------

В публикации [Ленивые операции над множествами в C++](../lazy-range-operations/lazy-range-operations.md) я показал, как проектировать ленивые операции над несколькими диапазонами. Теперь я хочу подробнее рассказать о важном решении, делающем такие операции удобными в использовании.

Один из основных моментов в интерфейсе ленивых операций над диапазонами — это возможность следующей записи

```cpp
burst::merge(std::tie(range1, range2, ...));
```

То есть возможность работать с произвольным набором исходных диапазонов.

В коде это будет выглядеть как-то так:

```cpp
const auto odd = std::vector{1, 3, 5, 7};
const auto even = std::list{0, 2, 4, 6, 8};

const auto merged_range = burst::merge(std::tie(odd, even));

const auto expected = {0, 1, 2, 3, 4, 5, 6, 7, 8};
assert(merged_range == expected);
```

Почему же это так важно, и что стоит за этой записью?

Ответ на вопрос "почему это важно" понятен. Во-первых, это красиво. Кроме того, удобно.
А вот что за этим стоит — и есть суть данной публикации.

Содержание
----------

1.  [Проблематика](#проблематика)
2.  [Стирание типа диапазона](#стирание-типа-диапазона)
    1.  [Стирание типов на пальцах](#стирание-типов-на-пальцах)
3.  [Сделай. Пожалуйста](#сделай-пожалуйста)
4.  [Массив диапазонов](#массив-диапазонов)
5.  [Диапазон из контейнера](#диапазон-из-контейнера)
6.  [Собираем всё вместе](#собираем-всё-вместе)
7.  [std::tie](#stdtie)
8.  [Заключение](#заключение)
9.  [Ссылки](#ссылки)

[Проблематика](#содержание)
---------------------------

Ключевой момент дизайна ленивых итераторов над диапазонами (см. [Дизайн->Быстрое копирование](../lazy-range-operations/lazy-range-operations.md#быстрое-копирование)) — это их легковесность. Если коротко — в итераторе не лежит ничего лишнего, кроме пары других итераторов.

Это даёт гибкость, но и накладывает определённые ограничения.

Допустим, мы хотим слить несколько разнотипных диапазонов. Но под капотом у итератора слияния сидит алгоритм, который хранит сливаемые диапазоны в одном контейнере и может переупорядочивать их (см. [Алгоритм слияния](../lazy-range-operations/lazy-range-operations.md#алгоритм-слияния)).
Хранить в одном контейнере разнотипные объекты можно (например, использовать [std::variant](https://en.cppreference.com/w/cpp/utility/variant) или [динамический кортеж](../dynamic-tuple/dynamic-tuple.md)), но в нашем случае довольно накладно. Поэтому нужно привести их к одному типу.

>   — Но как, Холмс?
>
>   — Элементарно, мой дорогой Ватсон. Использовать стирание типов.

[Стирание типа диапазона](#содержание)
--------------------------------------

Канонический пример стирания типов — [`std::any`](https://en.cppreference.com/w/cpp/utility/any).
Наш "стёртый" диапазон — это `any`-диапазон. Собственно, в Бусте он так и называется — [`boost::any_range`](https://www.boost.org/doc/libs/1_67_0/libs/range/doc/html/range/reference/ranges/any_range.html).

Для приведения разных диапазонов к единому типу нужно:
1.  Найти минимальную категорию из категорий исходных диапазонов;
2.  Выделить тип итерируемых элементов;
3.  Создать `any`-диапазон, который знает только свою категорию и тип итерируемых элементов.

### [Стирание типов на пальцах](#содержание)

Если совсем по-простому, стирание типов — это механизм, который позволяет "потерять" тип объекта на этапе компиляции, но "запомнить" этот тип в обработчике, который будет вызван когда-нибудь потом.

```cpp
#include <iostream>
#include <string>

template <typename T>
void handle (const void * erased_object)
{
    std::cout << *static_cast<const T *>(erased_object) << std::endl;
}

struct erased_t
{
    // Обработчик принимает указатель на void.
    using handler_type = void (*) (const void *);

    const void * object;
    handler_type handler;
};

template <typename T>
erased_t erase (const T & object)
{
    // Запомнили указатель на объект и нужный ему обработчик.
    return {&object, &handle<T>};
}

int main ()
{
    auto s = std::string("123");

    // Стёрли тип.
    auto erased = erase(s);

    // ...
    // Много кода.
    // ...

    // Вызвали запомненный обработчик.
    erased.handler(erased.object);
}

```

[Сделай. Пожалуйста](#содержание)
---------------------------------

Итак, у нас есть возможность стирать типы диапазонов. Напишем функцию [`uniform_range_tuple_please`](https://github.com/izvolov/burst/blob/master/include/burst/iterator/detail/uniform_range_tuple_please.hpp#L61), которая из кортежа ссылок на различные диапазоны должна сделать кортеж ссылок на одинаковые диапазоны.

>   Тут я использую идиому, которую называю "please" (если у неё есть какое-то общераспространённое название, напишите в комментариях, пожалуйста). Заключается она в том, что некая функция делает что-то только в том случае, если это требуется. А если не требуется, то возвращает то же, что получила на входе.

```cpp
template <typename... Ranges>
auto uniform_range_tuple_please (std::tuple<Ranges &...> ranges)
{
    return uniform_range_tuple_please_impl(ranges, are_same<Ranges...>{});
}
```

Если изначальные кортежи неодинаковые, то мы их приводим к единому типу.

```cpp
template <typename... Ranges>
auto uniform_range_tuple_please_impl (std::tuple<Ranges &...> ranges, std::false_type)
{
    static_assert(not are_same_v<Ranges...>, "");
    using iterator_category = minimum_category_t<range_category_t<Ranges>...>;
    using boost_traversal =
        typename boost::iterator_category_to_traversal<iterator_category>::type;
    return by_all(to_any_range<boost_traversal>, ranges);
}
```

Если же они изначально одинаковые, то ничего делать не нужно. Возвращаем исходный объект.

```cpp
template <typename... Ranges>
auto uniform_range_tuple_please_impl (std::tuple<Ranges &...> ranges, std::true_type)
{
    static_assert(are_same_v<Ranges...>, "");
    return ranges;
}
```

[Массив диапазонов](#содержание)
--------------------------------

После того, как исходные диапазоны были приведены к единому типу, нужно их сложить в контейнер. Здесь просто складываем их в `std::vector`. Для этого я использую самописную утилиту [`burst::make_range_vector`](https://github.com/izvolov/burst/blob/master/include/burst/range/make_range_vector.hpp#L18). Всё.

[Диапазон из контейнера](#содержание)
-------------------------------------

Дальше требуется ещё одна хитрость.

Как уже было сказано, в итераторе слияния должна храниться только пара итераторов на контейнер, в котором лежат сливаемые диапазоны. А после предыдущего шага у нас на руках как раз-таки контейнер. И его нужно, с одной стороны, где-то сохранить (он должен жить, пока жив итератор слияния), а с другой — в итераторе слияния должны быть только итераторы на начало и конец нашего контейнера.

Для этого есть [владеющий итератор](https://github.com/izvolov/burst/blob/master/include/burst/iterator/owning_iterator.hpp#L20). Он хранит итерируемый контейнер (в умном указателе) и итератор этого контейнера. Время жизни итерируемого контейнера ограничено созданием первого и уничтожением последнего экземпляров владеющего итератора. Кроме того, владеющий итератор сохраняет все характеристики итератора хранимого контейнера. Все действия с ним — продвижение, разыменование, сравнение — производятся так, как будто бы с итератором хранимого контейнера.

[Собираем всё вместе](#содержание)
----------------------------------

А теперь осталось только взять всё, что у нас имеется, и написать нужную обвязку:

```cpp
template <typename ... Ranges>
auto make_merge_iterator (std::tuple<Ranges &...> ranges)
{
    auto common_ranges = detail::uniform_range_tuple_please(ranges);
    return make_merge_iterator(own_as_range(burst::apply(make_range_vector, common_ranges)));
}
```

[std::tie](#содержание)
-----------------------

Ленивый итератор над диапазонами оперирует только ссылками и итераторами на исходные диапазоны. Поэтому нужно, чтобы был внешний (по отношению к нашему ленивому итератору) владелец этих диапазонов.

Я долго думал над тем, какой именно интерфейс нужно предоставить для того, чтобы передавать произвольный набор исходных диапазонов: то ли создать специальную структуру, то ли использовать вариадик и `enable_if`-магию. Но потом понял, что в стандартной библиотеке уже есть то, что мне нужно: `std::tie`.

Во-первых, `std::tie` создаёт кортеж. Во-вторых, запись получается достаточно лаконичной. Ну и в-третьих, `std::tie` принимает только lvalue, то есть временные объекты он принимать откажется. Значит, тем, что мы передаём в `std::tie`, уже кто-то владеет, и объект не умрёт в процессе слияния. Никаких висячих ссылок.

[Заключение](#содержание)
-------------------------

Мы получили удобный интерфейс для ленивого слияния (или любой другой операции над множествами) и достаточно эффективную реализацию этого механизма.

При этом, если для нас недопустимы накладные расходы на, например, `std::shared_ptr` в недрах `burst::owning_iterator`, то мы по-прежнему можем [создать диапазон для слияния руками](https://github.com/izvolov/papers/blob/master/lazy-range-operations/lazy-range-operations.md#можно-создать-диапазон).

[Ссылки](#содержание)
---------------------

-   [`std::any`](https://en.cppreference.com/w/cpp/utility/any)
-   [`boost::any_range`](https://www.boost.org/doc/libs/1_67_0/libs/range/doc/html/range/reference/ranges/any_range.html)
-   [Ленивые операции над диапазонами в библиотеке Burst](https://github.com/izvolov/burst#ленивые-вычисления-над-диапазонами)