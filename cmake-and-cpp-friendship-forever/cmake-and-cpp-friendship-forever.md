CMake и C++ — братья навек
==========================

![CMake и C++ — братья навек](cmake-and-cpp-friendship-forever.png)

В процессе разработки я люблю менять компиляторы, режимы сборки, версии зависимостей, производить статический анализ, замерять производительность, собирать покрытие, генерировать документацию и т.д. И очень люблю CMake, потому что он позволяет мне делать всё то, что я хочу.

Многие ругают CMake, и часто заслуженно, но если разобраться, то не всё так плохо, а в последнее время __очень даже неплохо__, и направление развития вполне позитивное.

В данной заметке я хочу рассказать, как достаточно просто организовать заголовочную библиотеку на языке C++ в системе CMake, чтобы получить следующую функциональность:

1.  Сборка;
2.  Автозапуск тестов;
3.  Замер покрытия кода;
4.  Установка;
5.  Автодокументирования;
6.  Генерация онлайн-песочницы;
7.  Статический анализ.

> Кто и так разбирается в плюсах и си-мейке может просто [скачать шаблон проекта](https://github.com/izvolov/mylib) и начать им пользоваться.

Содержание
----------

1.  [Проект изнутри](#проект-изнутри)
    1.  [Структура проекта](#структура-проекта)
    2.  [Главный CMake-файл (./CMakeLists.txt)](#главный-cmake-файл-cmakeliststxt)
        1.  [Информация о проекте](#информация-о-проекте)
        2.  [Опции проекта](#опции-проекта)
        3.  [Опции компиляции](#опции-компиляции)
        4.  [Основная цель](#основная-цель)
        5.  [Установка](#установка)
        6.  [Тесты](#тесты)
        7.  [Документация](#документация)
        8.  [Онлайн-песочница](#онлайн-песочница)
    3.  [Скрипт для тестов (test/CMakeLists.txt)](#скрипт-для-тестов-testcmakeliststxt)
        1.  [Тестирование](#тестирование)
        2.  [Покрытие](#покрытие)
    4.  [Скрипт для документации (doc/CMakeLists.txt)](#скрипт-для-документации-doccmakeliststxt)
    5.  [Скрипт для онлайн-песочницы (online/CMakeLists.txt)](#скрипт-для-онлайн-песочницы-onlinecmakeliststxt)
2.  [Проект снаружи](#проект-снаружи)
    1.  [Сборка](#сборка)
        1.  [Генерация](#генерация)
        2.  [Сборка](#сборка-проекта)
    2.  [Опции](#опции)
        1.  [MYLIB_COVERAGE](#MYLIB_COVERAGE)
        2.  [MYLIB_TESTING](#MYLIB_TESTING)
        3.  [MYLIB_DOXYGEN_LANGUAGE](#MYLIB_DOXYGEN_LANGUAGE)
    3.  [Сборочные цели](#сборочные-цели)
        1.  [По умолчанию](#по-умолчанию)
        2.  [mylib-unit-tests](#mylib-unit-tests)
        3.  [check](#check)
        4.  [coverage](#coverage)
        5.  [doc](#doc)
        6.  [wandbox](#wandbox)
    4.  [Примеры](#примеры)
3.  [Инструменты](#инструменты)
4.  [Статический анализ](#статический-анализ)
5.  [Послесловие](#послесловие)

[Проект изнутри](#содержание)
-----------------------------

### [Структура проекта](#содержание)

```
.
├── CMakeLists.txt
├── README.en.md
├── README.md
├── doc
│   ├── CMakeLists.txt
│   └── Doxyfile.in
├── include
│   └── mylib
│       └── myfeature.hpp
├── online
│   ├── CMakeLists.txt
│   ├── mylib-example.cpp
│   └── wandbox.py
└── test
    ├── CMakeLists.txt
    ├── mylib
    │   └── myfeature.cpp
    └── test_main.cpp
```

Главным образом речь пойдёт о том, как организовать CMake-скрипты, поэтому они будут разобраны подробно. Остальные файлы каждый желающий может посмотреть непосредственно [на странице проекта-шаблона](https://github.com/izvolov/mylib).

### [Главный CMake-файл (./CMakeLists.txt)](#содержание)

#### [Информация о проекте](#содержание)

В первую очередь нужно затребовать нужную версию системы CMake. CMake развивается, меняются сигнатуры команд, поведение в разных условиях. Чтобы CMake сразу понимал, чего мы от него хотим, нужно сразу зафиксировать наши к нему требования.

```cmake
cmake_minimum_required(VERSION 3.13)
```

Затем обозначим наш проект, его название, версию, используемые языки и прочее (см. [команду `project`](https://cmake.org/cmake/help/v3.8/command/project.html)).

В данном случае указываем язык `CXX` (а это значит C++), чтобы CMake не напрягался и не искал компилятор языка C (по умолчанию в CMake включены два языка: C и C++).

```cmake
project(Mylib VERSION 1.0 LANGUAGES CXX)
```

Здесь же можно сразу проверить, включён ли наш проект в другой проект в качестве подпроекта. Это сильно поможет в дальнейшем.

```cmake
get_directory_property(IS_SUBPROJECT PARENT_DIRECTORY)
```

#### [Опции проекта](#содержание)

Предусмотрим две опции.

Первая опция — [`MYLIB_TESTING`](#MYLIB_TESTING) — для выключения модульных тестов. Это может понадобиться, если мы уверены, что с тестами всё в порядке, а мы хотим, например, только установить или запакетировать наш проект. Или наш проект включён в качестве подпроекта — в этом случае пользователю нашего проекта не интересно запускать наши тесты. Вы же не тестируете зависимости, которыми пользуетесь?

```cmake
option(MYLIB_TESTING "Включить модульное тестирование" ON)
```

Кроме того, мы сделаем отдельную опцию [`MYLIB_COVERAGE`](#MYLIB_COVERAGE) для замеров покрытия кода тестами, но она потребует дополнительных инструментов, поэтому включать её нужно будет явно.

```cmake
option(MYLIB_COVERAGE "Включить измерение покрытия кода тестами" OFF)
```

#### [Опции компиляции](#содержание)

Разумеется, мы крутые программисты-плюсовики, поэтому хотим от компилятора максимального уровня диагностик времени компиляции. Ни одна мышь не проскочит.

```cmake
add_compile_options(
    -Werror

    -Wall
    -Wextra
    -Wpedantic

    -Wcast-align
    -Wcast-qual
    -Wconversion
    -Wctor-dtor-privacy
    -Wenum-compare
    -Wfloat-equal
    -Wnon-virtual-dtor
    -Wold-style-cast
    -Woverloaded-virtual
    -Wredundant-decls
    -Wsign-conversion
    -Wsign-promo
)
```

Расширения тоже отключим, чтобы полностью соответствовать стандарту языка C++. По умолчанию в CMake они включены.

```cmake
if(NOT CMAKE_CXX_EXTENSIONS)
    set(CMAKE_CXX_EXTENSIONS OFF)
endif()
```

#### [Основная цель](#содержание)

Наша библиотека состоит только из заголовочных файлов, а значит, мы не располагаем каким-либо выхлопом в виде статических или динамических библиотек. С другой стороны, чтобы использовать нашу библиотеку снаружи, её нужно установить, нужно, чтобы её можно было обнаружить в системе и подключить к своему проекту, и при этом вместе с ней были привязаны эти самые заголовки, а также, возможно, какие-то дополнительные свойства.

Для этой цели создаём интерфейсную библиотеку.

```cmake
add_library(mylib INTERFACE)
```

Привязываем заголовки к нашей интерфейсной библиотеке.

Современное, модное, молодёжное использование CMake подразумевает, что заголовки, свойства и т.п. передаются через одну единственную цель. Таким образом, достаточно сказать [`target_link_libraries(target PRIVATE dependency)`](https://cmake.org/cmake/help/v3.8/command/target_link_libraries.html#libraries-for-a-target-and-or-its-dependents), и все заголовки, которые ассоциированы с целью `dependency`, будут доступны для исходников, принадлежащих цели `target`. И не требуется никаких `[target_]include_directories`. Это будет продемонстрировано ниже при разборе [CMake-скрипта для модульных тестов](#скрипт-для-тестов-testcmakeliststxt).

Также стоит обратить внимание на т.н. [выражения-генераторы: `$<...>`](https://cmake.org/cmake/help/v3.8/manual/cmake-generator-expressions.7.html).

Данная команда ассоциирует нужные нам заголовки с нашей интерфейсной библиотекой, причём, в случае, если наша библиотека будет подключена к какой-либо цели в рамках одной иерархии CMake, то с ней будут ассоциированы заголовки из директории `${CMAKE_CURRENT_SOURCE_DIR}/include`, а если наша библиотека установлена в систему и подключена в другой проект с помощью команды [`find_package`](https://cmake.org/cmake/help/v3.8/command/find_package.html), то с ней будут ассоциированы заголовки из директории `include` относительно директории установки.

```cmake
target_include_directories(mylib INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
```

Установим стандарт языка. Разумеется, самый последний. При этом не просто включаем стандарт, но и распространяем его на тех, кто будет использовать нашу библиотеку. Это достигается за счёт того, что установленное свойство имеет категорию `INTERFACE` (см. [команду target_compile_features](https://cmake.org/cmake/help/v3.8/command/target_compile_features.html)).

```cmake
target_compile_features(mylib INTERFACE cxx_std_17)
```

Заводим псевдоним для нашей библиотеки. Причём для красоты он будет в специальном "пространстве имён". Это будет полезно, когда в нашей библиотеке появятся разные модули, и мы заходим подключать их независимо друг от друга. [Как в Бусте, например](https://cmake.org/cmake/help/v3.8/module/FindBoost.html).

```cmake
add_library(Mylib::mylib ALIAS mylib)
```

#### [Установка](#содержание)

Установка наших заголовков в систему. Тут всё просто. Говорим, что папка со всеми заголовками должна попасть в директорию `include` относительно места установки.

```cmake
install(DIRECTORY include/mylib DESTINATION include)
```

Далее сообщаем системе сборки о том, что мы хотим иметь возможность в сторонних проектах звать команду `find_package(Mylib)` и получать цель `Mylib::mylib`.

```cmake
install(TARGETS mylib EXPORT MylibConfig)
install(EXPORT MylibConfig NAMESPACE Mylib:: DESTINATION share/Mylib/cmake)
```

Следующее заклинание нужно понимать так. Когда в стороннем проекте мы вызовем команду `find_package(Mylib 1.2.3 REQUIRED)`, и при этом реальная версия установленной библиотеки окажется несовместимой с версией `1.2.3`, CMake автоматически сгенерирует ошибку. То есть не нужно будет следить за версиями вручную.

```cmake
include(CMakePackageConfigHelpers)
write_basic_package_version_file("${PROJECT_BINARY_DIR}/MylibConfigVersion.cmake"
    VERSION
        ${PROJECT_VERSION}
    COMPATIBILITY
        AnyNewerVersion
)
install(FILES "${PROJECT_BINARY_DIR}/MylibConfigVersion.cmake" DESTINATION share/Mylib/cmake)
```

#### [Тесты](#содержание)

Если тесты выключены явно с помощью [соответствующей опции](#MYLIB_TESTING) или наш проект является подпроектом, то есть подключён в другой CMake-проект с помощью команды [`add_subdirectory`](https://cmake.org/cmake/help/v3.8/command/add_subdirectory.html), мы не переходим дальше по иерархии, и скрипт, в котором описаны команды для генерации и запуска тестов, просто не запускается.

```cmake
if(NOT MYLIB_TESTING)
    message(STATUS "Тестирование проекта Mylib выключено")
elseif(IS_SUBPROJECT)
    message(STATUS "Mylib не тестируется в режиме подмодуля")
else()
    add_subdirectory(test)
endif()
```

#### [Документация](#содержание)

Документация также не будет генерироваться в случае подпроекта.

```cmake
if(NOT IS_SUBPROJECT)
    add_subdirectory(doc)
endif()
```

#### [Онлайн-песочница](#содержание)

Аналогично, онлайн-песочницы у подпроекта тоже не будет.

```cmake
if(NOT IS_SUBPROJECT)
    add_subdirectory(online)
endif()
```

### [Скрипт для тестов (test/CMakeLists.txt)](#содержание)

#### [Тестирование](#содержание)

Первым делом находим пакет с нужным тестовым фреймворком (замените на свой любимый).

```cmake
find_package(doctest 2.3.3 REQUIRED)
```

Создаём наш исполняемый файл с тестами. Обычно непосредственно в исполняемый бинарник я добавляю только файл, в котором будет функция `main`.

```cmake
add_executable(mylib-unit-tests test_main.cpp)
```

А файлы, в которых описаны сами тесты, добавляю позже. Но так делать не обязательно.

```cmake
target_sources(mylib-unit-tests PRIVATE mylib/myfeature.cpp)
```

Подключаем зависимости. Обратите внимание, что к нашему бинарнику мы привязали только нужные нам CMake-цели, и не вызывали команду `target_include_directories`. Заголовки из тестового фреймворка и из нашей `Mylib::mylib`, а также параметры сборки (в нашем случае это стандарт языка C++) пролезли вместе с этими целями.

```cmake
target_link_libraries(mylib-unit-tests
    PRIVATE
        Mylib::mylib
        doctest::doctest
)
```

Наконец, создаём фиктивную цель, "сборка" которой эквивалентна запуску тестов, и добавляем эту цель в сборку по умолчанию (за это отвечает атрибут `ALL`). Это значит, что сборка по умолчанию инициирует запуск тестов, то есть мы никогда не забудем их запустить.

```cmake
add_custom_target(check ALL COMMAND mylib-unit-tests)
```

#### [Покрытие](#содержание)

Далее включаем замер покрытия кода, если задана соответствующая опция. В детали вдаваться не буду, потому что они относятся больше к инструменту для замеров покрытия, чем к CMake. Важно только отметить, что по результатам будет создана цель [`coverage`](#coverage), с помощью которой удобно запускать замер покрытия.

```cmake
find_program(GCOVR_EXECUTABLE gcovr)
if(MYLIB_COVERAGE AND GCOVR_EXECUTABLE)
    message(STATUS "Измерение покрытия кода тестами включено")

    target_compile_options(mylib-unit-tests PRIVATE --coverage)
    target_link_libraries(mylib-unit-tests PRIVATE gcov)

    add_custom_target(coverage
        COMMAND
            ${GCOVR_EXECUTABLE}
                --root=${PROJECT_SOURCE_DIR}/include/
                --object-directory=${CMAKE_CURRENT_BINARY_DIR}
        DEPENDS
            check
    )
elseif(MYLIB_COVERAGE AND NOT GCOVR_EXECUTABLE)
    set(MYLIB_COVERAGE OFF)
    message(WARNING "Для замеров покрытия кода тестами требуется программа gcovr")
endif()
```

### [Скрипт для документации (doc/CMakeLists.txt)](#содержание)

[Нашли Doxygen](https://cmake.org/cmake/help/v3.8/module/FindDoxygen.html).

```cmake
find_package(Doxygen)
```

Дальше проверяем, установлена ли пользователем переменная с языком. Если да, то не трогаем, если нет, то берём русский. Затем конфигурируем файлы системы Doxygen. Все нужные переменные, в том числе и язык попадают туда в процессе конфигурации (см. [команду `configure_file`](https://cmake.org/cmake/help/v3.8/command/configure_file.html)).

После чего создаём цель [`doc`](#doc), которая будет запускать генерирование документации. Поскольку генерирование документации — не самая большая необходимость в процессе разработки, то по умолчанию цель включена не будет, её придётся запускать явно.


```cmake
if (Doxygen_FOUND)
    if (NOT MYLIB_DOXYGEN_LANGUAGE)
        set(MYLIB_DOXYGEN_LANGUAGE Russian)
    endif()
    message(STATUS "Doxygen documentation will be generated in ${MYLIB_DOXYGEN_LANGUAGE}")
    configure_file(Doxyfile.in Doxyfile)
    add_custom_target(doc COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
endif ()
```

### [Скрипт для онлайн-песочницы (online/CMakeLists.txt)](#содержание)

Тут находим третий Питон и создаём цель [`wandbox`](#wandbox), которая генерирует запрос, соответствующий API сервиса [Wandbox](https://wandbox.org), и отсылает его. В ответ приходит ссылка на готовую песочницу.

```cmake
find_program(PYTHON3_EXECUTABLE python3)
if(PYTHON3_EXECUTABLE)
    set(WANDBOX_URL "https://wandbox.org/api/compile.json")

    add_custom_target(wandbox
        COMMAND
            ${PYTHON3_EXECUTABLE} wandbox.py mylib-example.cpp "${PROJECT_SOURCE_DIR}" include |
            curl -H "Content-type: application/json" -d @- ${WANDBOX_URL}
        WORKING_DIRECTORY
            ${CMAKE_CURRENT_SOURCE_DIR}
        DEPENDS
            mylib-unit-tests
    )
else()
    message(WARNING "Для создания онлайн-песочницы требуется интерпретатор ЯП python 3-й версии")
endif()
```

[Проект снаружи](#содержание)
-----------------------------

Теперь рассмотрим, как этим всем пользоваться.

### [Сборка](#содержание)

Сборка данного проекта, как и любого другого проекта на системе сборки CMake, состоит из двух этапов:

#### [Генерация](#содержание)

```shell
cmake -S путь/к/исходникам -B путь/к/сборочной/директории [опции ...]
```

[Подробнее про опции](#опции).

#### [Сборка проекта](#содержание)

```shell
cmake --build путь/к/сборочной/директории [--target target]
```

[Подробнее про сборочные цели](#сборочные-цели).

### [Опции](#содержание)

#### [MYLIB_COVERAGE](#содержание)

```shell
cmake -S ... -B ... -DMYLIB_COVERAGE=ON [прочие опции ...]
```

Включает цель [`coverage`](#coverage), с помощью которой можно запустить замер покрытия кода тестами.

#### [MYLIB_TESTING](#содержание)

```shell
cmake -S ... -B ... -DMYLIB_TESTING=OFF [прочие опции ...]
```

Предоставляет возможность выключить сборку модульных тестов и цель [`check`](#check). Как следствие, выключается замер покрытия кода тестами (см. [`MYLIB_COVERAGE`](#MYLIB_COVERAGE)).

Также тестирование автоматически отключается в случае, если проект подключается в другой проект качестве подпроекта с помощью команды [`add_subdirectory`](https://cmake.org/cmake/help/v3.8/command/add_subdirectory.html).

#### [MYLIB_DOXYGEN_LANGUAGE](#содержание)

```shell
cmake -S ... -B ... -DMYLIB_DOXYGEN_LANGUAGE=English [прочие опции ...]
```

Переключает язык документации, которую генерирует цель [`doc`](#doc) на заданный. Список доступных языков см. на [сайте системы Doxygen](http://www.doxygen.nl/manual/config.html#cfg_output_language).

По умолчанию включён русский.

### [Сборочные цели](#содержание)

#### [По умолчанию](#содержание)

```shell
cmake --build path/to/build/directory
cmake --build path/to/build/directory --target all
```

Если цель не указана (что эквивалентно цели `all`), собирает всё, что можно, а также вызывает цель [`check`](#check).

#### [mylib-unit-tests](#содержание)

```shell
cmake --build path/to/build/directory --target mylib-unit-tests
```

Компилирует модульные тесты. Включено по умолчанию.

#### [check](#содержание)

```shell
cmake --build путь/к/сборочной/директории --target check
```

Запускает собранные (собирает, если ещё не) модульные тесты. Включено по умолчанию.

См. также [`mylib-unit-tests`](#mylib-unit-tests).

#### [coverage](#содержание)

```shell
cmake --build путь/к/сборочной/директории --target coverage
```

Анализирует запущенные (запускает, если ещё не) модульные тесты на предмет покрытия кода тестами при помощи программы [gcovr](https://gcovr.com).

Выхлоп покрытия будет выглядеть примерно так:

```
------------------------------------------------------------------------------
                           GCC Code Coverage Report
Directory: /path/to/cmakecpptemplate/include/
------------------------------------------------------------------------------
File                                       Lines    Exec  Cover   Missing
------------------------------------------------------------------------------
mylib/myfeature.hpp                            2       2   100%   
------------------------------------------------------------------------------
TOTAL                                          2       2   100%
------------------------------------------------------------------------------
```

Цель доступна только при включённой опции [`MYLIB_COVERAGE`](#MYLIB_COVERAGE).

См. также [`check`](#check).

#### [doc](#содержание)

```shell
cmake --build путь/к/сборочной/директории --target doc
```

Запускает генерацию документации к коду при помощи системы [Doxygen](http://doxygen.nl).

#### [wandbox](#содержание)

```shell
cmake --build путь/к/сборочной/директории --target wandbox
```

Ответ от сервиса выглядит примерно так:

```json
{
    "permlink" :    "QElvxuMzHgL9fqci",
    "status" :  "0",
    "url" : "https://wandbox.org/permlink/QElvxuMzHgL9fqci"
}
```

Для этого используется сервис [Wandbox](https://wandbox.org). Не знаю, насколько у них резиновые сервера, но думаю, что злоупотреблять данной возможностью не стоит.

### [Примеры](#содержание)

#### Сборка проекта в отладочном режиме с замером покрытия

```shell
cmake -S путь/к/исходникам -B путь/к/сборочной/директории -DCMAKE_BUILD_TYPE=Debug -DMYLIB_COVERAGE=ON
cmake --build путь/к/сборочной/директории --target coverage --parallel 16
```

#### Установка проекта без предварительной сборки и тестирования

```shell
cmake -S путь/к/исходникам -B путь/к/сборочной/директории -DMYLIB_TESTING=OFF -DCMAKE_INSTALL_PREFIX=путь/к/установойной/директории
cmake --build путь/к/сборочной/директории --target install
```

#### Сборка в выпускном режиме заданным компилятором

```shell
cmake -S путь/к/исходникам -B путь/к/сборочной/директории -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_COMPILER=g++-8 -DCMAKE_PREFIX_PATH=путь/к/директории/куда/установлены/зависимости
cmake --build путь/к/сборочной/директории --parallel 4
```

#### Генерирование документации на английском

```shell
cmake -S путь/к/исходникам -B путь/к/сборочной/директории -DCMAKE_BUILD_TYPE=Release -DMYLIB_DOXYGEN_LANGUAGE=English
cmake --build путь/к/сборочной/директории --target doc
```

[Инструменты](#содержание)
--------------------------

1.  [CMake](https://cmake.org) 3.13

    На самом деле версия CMake 3.13 требуется только для запуска некоторых консольных команд, описанных в данной справке. С точки зрения синтаксиса CMake-скриптов достаточно версии 3.8, если генерацию вызывать другими способами.

2.  Библиотека тестирования [doctest](https://github.com/onqtam/doctest)

    Тестирование можно отключать (см. [опцию `MYLIB_TESTING`](#MYLIB_TESTING)).

3.  [Doxygen](http://doxygen.nl)

    Для переключения языка, на котором будет сгенерирована документация, предусмотрена опция [`MYLIB_DOXYGEN_LANGUAGE`](#MYLIB_DOXYGEN_LANGUAGE).

4.  Интерпретатор ЯП [Python 3](https://www.python.org)

    Для автоматической генерации [онлайн-песочницы](#wandbox).

[Статический анализ](#содержание)
---------------------------------

С помощью CMake и пары хороших инструментов можно обеспечить статический анализ с минимальными телодвижениями.

#### Cppcheck

В CMake встроена поддержка инструмента для статического анализа [Cppcheck](http://cppcheck.sourceforge.net).

Для этого нужно воспользоваться опцией [`CMAKE_CXX_CPPCHECK`](https://cmake.org/cmake/help/v3.10/variable/CMAKE_LANG_CPPCHECK.html#variable:CMAKE_<LANG>_CPPCHECK):

```shell
cmake -S путь/к/исходникам -B путь/к/сборочной/директории -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_CPPCHECK="cppcheck;--enable=all;-Iпуть/к/исходникам/include"
```

После этого статический анализ будет автоматически запускаться каждый раз во время компиляции и перекомпиляции исходников. Ничего дополнительного делать не нужно.

#### Clang

При помощи чудесного инструмента [`scan-build`](https://clang-analyzer.llvm.org/scan-build) тоже можно запускать статический анализ в два счёта:

```shell
scan-build cmake -S путь/к/исходникам -B путь/к/сборочной/директории -DCMAKE_BUILD_TYPE=Debug
scan-build cmake --build путь/к/сборочной/директории
```

Здесь, в отличие от случая с [Cppcheck](#cppcheck), требуется каждый раз запускать сборку через `scan-build`.

[Послесловие](#содержание)
--------------------------

CMake — очень мощная и гибкая система, позволяющая реализовывать функциональность на любой вкус и цвет. И, хотя, синтаксис порой оставляет желать лучшего, всё же не так страшен чёрт, как его малюют. Пользуйтесь системой сборки CMake на благо общества и с пользой для здоровья.
