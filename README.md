Список публикаций
=================

## [Поразрядная сортировка с человеческим лицом](radix-sort-with-human-face/radix-sort-with-human-face.md)

О том, каким должен быть правильный интерфейс поразрядной сортировки в языке C++, и почему именно таким.

## [Динамический неоднородный плотно упакованный контейнер](dynamic-tuple/dynamic-tuple.md)

О контейнере, ключевые особенности которого следующие:

1.  Складываемые в контейнер типы не обязаны быть членами одной иерархии классов — это могут быть совершенно разные типы. Например, можно поместить одновременно `bool`, `double`, `std::string`, `std::vector<int>` и т.п.

2.  Объекты плотно расположены в памяти. То есть они лежат в едином буфере подряд, друг за другом. Это снимает лишний уровень косвенности при обращении к ним по сравнению с тем, если бы это был, например, массив указателей на базовый класс, как это обычно делается для хранения в массиве классов из одной иерархии.

## [Концепты для отчаявшихся](concepts-for-despaired/concepts-for-despaired.md)

О технике, позволяющей создавать легковесные и удобочитаемые "концепты" со, впрочем, достаточно ограниченной функциональностью.

## [Элементы функционального программирования в C++: частичное применение](eofp-partial-application/eofp-partial-application.md)

О проектировании и реализации механизмов частичного применения в языке C++.

## [Элементы функционального программирования в C++: композиции отображений](eofp-compositions/eofp-compositions.md)

Об инструментах, делающих компоновку функций лёгкой и приятной.

## [Манифест жёсткого программиста](solid-manifesto/solid-manifesto.md)

О методологиях ведения разработки ПО и рабочих отношениях.

## [CMake и C++ — братья навек](cmake-and-cpp-friendship-forever/cmake-and-cpp-friendship-forever.md)

О том, как подружить систему сборки CMake и язык C++ в рамках проекта заголовочной библиотеки.

## [C++ и CMake — братья навек, часть II](cmake-and-cpp-friendship-forever/cmake-and-cpp-friendship-forever-part-ii.md).

О подготовке независимых идинообразных модулей и связывании их между собой.
