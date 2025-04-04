# Расширение и встраивание интерпретатора Python
Этот документ описывает, как писать модули на C или C++ для расширения интерпретатора Python новыми модулями. Эти модули могут не только определять новые функции, но и создавать новые типы объектов и их методы. Документ также объясняет, как встроить интерпретатор Python в другое приложение, чтобы использовать его в качестве языка расширения. Наконец, в нём описано, как компилировать и связывать модули расширения, чтобы они могли загружаться динамически (во время выполнения) в интерпретатор, если эта возможность поддерживается операционной системой.

Этот документ предполагает наличие базовых знаний языка Python. Для неформального знакомства с языком см. [<u>The Python Tutorial</u>](https://docs.python.org/3/tutorial/index.html#tutorial-index). [<u>The Python Language Reference</u>](https://docs.python.org/3/reference/index.html#reference-index) даёт более формальное определение языка. Стандартная библиотека Python [<u>The Python Standard Library</u>](https://docs.python.org/3/library/index.html#library-index) содержит документацию по существующим типам объектов, функциям и модулям (как встроенным, так и написанным на Python), которые делают язык универсальным.

Для детального описания всего API Python/C см. [<u>Python/C API Reference Manual</u>](https://docs.python.org/3/c-api/index.html#c-api-index).

## Рекомендуемые сторонние инструменты
Этот справочник описывает только основные инструменты для создания расширений, которые предоставляются в данной версии CPython. Сторонние инструменты, такие как [<u>Cython</u>](https://cython.org/), [<u>cffi</u>](https://cffi.readthedocs.io/en/stable/), [<u>SWIG</u>](https://www.swig.org/) и [<u>Numba</u>](https://numba.pydata.org/), предлагают как более простые, так и более сложные подходы к созданию расширений на C и C++ для Python.

## Создание расширений без сторонних инструментов
Этот раздел справочника посвящён созданию расширений на C и C++ без использования сторонних инструментов. Он предназначен в первую очередь для разработчиков таких инструментов, а не как рекомендуемый способ создания собственных расширений на C.

<span style="font-size: 21px; font-weight: 600;">Оглавление</span>
- [Расширение и встраивание интерпретатора Python](#расширение-и-встраивание-интерпретатора-python)
  - [Рекомендуемые сторонние инструменты](#рекомендуемые-сторонние-инструменты)
  - [Создание расширений без сторонних инструментов](#создание-расширений-без-сторонних-инструментов)
- [1. Расширение Python с помощью C или C++](#1-расширение-python-с-помощью-c-или-c)
  - [1.1 Простой пример](#11-простой-пример)
  - [1.2. Интермеццо: ошибки и исключения](#12-интермеццо-ошибки-и-исключения)
  - [1.3. Возвращение к примеру](#13-возвращение-к-примеру)
  - [1.4. Таблица методов модуля и функция инициализации](#14-таблица-методов-модуля-и-функция-инициализации)
  - [1.5. Компиляция и компоновка](#15-компиляция-и-компоновка)
  - [1.6. Вызов функций Python из C](#16-вызов-функций-python-из-c)
  - [1.7. Извлечение параметров в функциях расширения](#17-извлечение-параметров-в-функциях-расширения)
  - [1.8. Ключевые параметры для функций расширения](#18-ключевые-параметры-для-функций-расширения)
  - [1.9. Создание произвольных значений](#19-создание-произвольных-значений)
  - [1.10. Подсчёт ссылок (Reference Counts)](#110-подсчёт-ссылок-reference-counts)
    - [1.10.1 Подсчёт ссылок в Python](#1101-подсчёт-ссылок-в-python)
    - [1.10.2 Правила владения](#1102-правила-владения)
    - [1.10.3 Тонкий лёд](#1103-тонкий-лёд)
    - [1.10.4 Пустые указатели](#1104-пустые-указатели)
  - [1.11. Написание расширений на C++](#111-написание-расширений-на-c)
  - [1.12. Предоставление C API для модуля расширения](#112-предоставление-c-api-для-модуля-расширения)
- [2. Определение типов расширений: учебное пособие](#2-определение-типов-расширений-учебное-пособие)
  - [2.1. Основы](#21-основы)
  - [2.2. Добавление данных и методов к базовому примеру](#22-добавление-данных-и-методов-к-базовому-примеру)
  - [2.3. Более точный контроль над атрибутами данных](#23-более-точный-контроль-над-атрибутами-данных)
  - [2.4. Поддержка циклического сборщика мусора](#24-поддержка-циклического-сборщика-мусора)
  - [2.5. Наследование от других типов](#25-наследование-от-других-типов)
- [3. Определение типов расширений: различные темы](#3-определение-типов-расширений-различные-темы)
  - [3.1. Завершение и освобождение памяти](#31-завершение-и-освобождение-памяти)
  - [3.2. Представление объекта](#32-представление-объекта)
  - [3.3. Управление атрибутами](#33-управление-атрибутами)
    - [3.3.1 Управление атрибутами общего типа](#331-управление-атрибутами-общего-типа)
    - [3.3.2 Управление атрибутами для конкретного типа](#332-управление-атрибутами-для-конкретного-типа)
  - [3.4. Сравнение объектов](#34-сравнение-объектов)
  - [3.5. Поддержка абстрактных протоколов](#35-поддержка-абстрактных-протоколов)
  - [3.6. Поддержка слабых ссылок](#36-поддержка-слабых-ссылок)
  - [3.7. Дополнительные рекомендации](#37-дополнительные-рекомендации)
- [4. Сборка расширений на C и C++](#4-сборка-расширений-на-c-и-c)
  - [4.1. Сборка расширений на C и C++ с помощью setuptools](#41-сборка-расширений-на-c-и-c-с-помощью-setuptools)
- [5. Сборка расширений на C и C++ в Windows](#5-сборка-расширений-на-c-и-c-в-windows)
  - [5.1. Пошаговое руководство](#51-пошаговое-руководство)
  - [5.2. Различия между Unix и Windows](#52-различия-между-unix-и-windows)
  - [5.3. Использование DLL на практике](#53-использование-dll-на-практике)
  - [Встраивание среды выполнения CPython в более крупное приложение](#встраивание-среды-выполнения-cpython-в-более-крупное-приложение)
- [6. Встраивание Python в другое приложение](#6-встраивание-python-в-другое-приложение)
  - [6.1. Встраивание на очень высоком уровне](#61-встраивание-на-очень-высоком-уровне)
  - [6.2. За пределами встраивания на очень высоком уровне: обзор](#62-за-пределами-встраивания-на-очень-высоком-уровне-обзор)
  - [6.3. Чистое встраивание](#63-чистое-встраивание)
  - [6.4. Расширение встроенного Python](#64-расширение-встроенного-python)
  - [6.5. Встраивание Python в C++](#65-встраивание-python-в-c)
  - [6.6. Компиляция и компоновка в Unix-подобных системах](#66-компиляция-и-компоновка-в-unix-подобных-системах)

# 1. Расширение Python с помощью C или C++
Довольно легко можно добавить новые встроенные модули в Python, если вы умеете программировать на C. Такие *расширения* могут делать две вещи, которые невозможно реализовать напрямую в Python: создавать новые встроенные типы объектов и вызывать функции библиотек C и системные вызовы.

Для того, чтобы поддерживать расширения Python API (Application Programmers Interface) предоставляет набор функций, макросов и переменных, которые обеспечивают доступ к большинству аспектов среды выполнения Python. Python API подключается в исходном коде C с помощью заголовочного файла `"Python.h"`.

Компиляция модуля расширения зависит от его назначения, а также от конфигурации вашей системы; подробности об этом приведены в следующих главах.

> **Примечание**: Интерфейс расширений на C специфичен для CPython, и модули расширений не работают в других реализациях Python. В большинстве случаев можно обойтись без написания расширений на C и сохранить переносимость на другие реализации. Например, если ваша задача заключается в вызове функций из C-библиотек или системных вызовов, вам следует рассмотреть возможность использования модуля [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">ctypes</span>](https://docs.python.org/3/library/ctypes.html#module-ctypes) или библиотеки [<u>cffi</u>](https://cffi.readthedocs.io/en/stable/) вместо написания собственного кода на C. Эти модули позволяют писать Python-код для взаимодействия с C-кодом и обладают большей переносимостью между реализациями Python, чем написание и компиляция модуля расширения на C.

## 1.1 Простой пример
<a id="footnote-1-back"></a>Давайте создадим модуль расширения с именем `spam` и предположим, что мы хотим создать Python интерфейс для функции <span style="font-family: Consolas, sans-serif;">system()</span> [[1]](#footnote-1) из C-библиотеки. Эта функция принимает строку, заканчивающуюся нулевым символом, в качестве аргумента и возвращает целое число. Мы хотим, чтобы эту функцию можно было вызвать из Python следующим образом:

```python
import spam
status = spam.system("ls -l")
```

Начнём с создания файла `spammodule.c.` (Исторически, если модуль называется `spam`, файл C, содержащий его реализацию, называется `spammodule.c`; если имя модуля очень длинное, например `spammify`, то файл может быть просто `spammify.c.`)

Первые две строки нашего файла могут быть следующими:

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>
```

Эти строки подключают Python API (вы можете добавить комментарий, описывающий назначение модуля, и уведомление о авторских правах, если хотите).

> **Примечание**: Поскольку Python может определять некоторые макросы препроцессора, которые влияют на стандартные заголовочные файлы на некоторых системах, необходимо включать `Python.h` до любых стандартных заголовочных<br><br>`#define PY_SSIZE_T_CLEAN` был использован для указания, что в некоторых API должен использоваться тип `Py_ssize_t` вместо `int`. Это больше не обязательно с версии Python 3.13, но мы оставим это для обеспечения обратной совместимости. См. раздел [<u>Strings and buffers</u>](https://docs.python.org/3.13/c-api/arg.html#arg-parsing-string-and-buffers) для описания этого макроса.

Все символы, видимые пользователю, определённые `Python.h`, имеют префикс `Py` или `PY`, за исключением тех, что определены в стандартных заголовочных файлах. Для удобства `"Python.h"` включает несколько стандартных заголовочных файлов: `<stdio.h>`, `<string.h>`, `<errno.h>` и `<stdlib.h>`,поскольку они активно используются интерпретатором Python. Если последний заголовочный файл отсутствует в вашей системе, он напрямую объявляет функции <span style="font-family: Consolas, sans-serif;">malloc()</span>, <span style="font-family: Consolas, sans-serif;">free()</span> и <span style="font-family: Consolas, sans-serif;">realloc()</span>.

Следующее, что мы добавим в файл нашего модуля будет функция на C, которая будет вызываться, когда выражение на Python `spam.system(string)` будет вычисляться (мы вскоре увидим, как она будет вызываться):

```c
static PyObject *
spam_system(PyObject *self, PyObject *args)
{
    const char *command;
    int sts;

    if (!PyArg_ParseTuple(args, "s", &command))
        return NULL;
    sts = system(command);
    return PyLong_FromLong(sts);
}
```

Существует прямой перевод из списка аргументов в Python (например, единственного выражения `"ls -l"`) в аргументы, передаваемые в функцию C. Функция на С всегда имеет два аргумента, которые принято называть *self* и *args*.

Аргумент *self* указывает на объект модуля для функций на уровне модуля; для метода он будет указывать на экземпляр объекта.

Аргумент *args* будет указателем на объект кортежа Python, содержащий аргументы. Каждый элемент этого кортежа соответствует одному аргументу из списка аргументов вызова. Аргументы — это объекты Python, и чтобы работать с ними в нашей функции на языке С, необходимо преобразовать их в значения C. Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3.13/c-api/arg.html#c.PyArg_ParseTuple) из API Python проверяет типы аргументов и преобразует их в значения C. Она использует строку-шаблон для определения требуемых типов аргументов, а также типов C-переменных, в которые будут сохранены преобразованные значения. Подробнее об этом будет сказано позже.

[<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3.13/c-api/arg.html#c.PyArg_ParseTuple) возвращает **true** (не ноль), если все аргументы имеют правильный тип, и их компоненты были сохранены в переменных, адреса которых были переданы в функцию. Если передан некорректный список аргументов, функция возвращает **false** (ноль). В этом случае она также вызывает соответствующее исключение, чтобы вызывающая функция могла сразу вернуть `NULL` (как мы видели в примере).

## 1.2. Интермеццо: ошибки и исключения
Важная договорённость, которая применяется во всей интерпретаторе Python, заключается в следующем: когда функция завершает выполнение с ошибкой, она должна установить условие исключения и вернуть значение ошибки (обычно `-1` или указатель `NULL`). Информация об исключении сохраняется в трёх полях состояния потока интерпретатора. Эти поля равны `NULL`, если исключения нет. В противном случае они являются C-эквивалентами полей кортежа Python, возвращаемого функцией [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">sys.exc_info()</span>](https://docs.python.org/3/library/sys.html#sys.exc_info). Это тип исключения, экземпляр исключения и объект трассировки стека. Важно знать о них, чтобы понять, как ошибки передаются между различными частями программы.

Python API определяет ряд функций для установки различных типов исключений.

Самая распространённая функция — это [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_SetString()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_SetString). Её аргументами являются объект исключения и строка в C-строка. Объект исключения обычно является заранее определённым объектом, таким как <span style="font-family: Consolas, sans-serif;">PyExc_ZeroDivisionError</span>. C-Строка указывает на причину ошибки и преобразуется в объект строки Python, который сохраняется как «сопутствующее значение» исключения.

Ещё одна полезная функция — [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_SetFromErrno()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_SetFromErrno), которая принимает только аргумент исключения и строит сопутствующее значение, используя глобальную переменную <span style="font-family: Consolas, sans-serif;">errno</span>. Самая универсальная функция — это [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_SetObject()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_SetObject), которая принимает два объекта в качестве аргументов: исключение и его сопутствующее значение. Вам не нужно вызывать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_INCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF) для объектов, передаваемых в эти функции.

Вы можете безопасно проверить, было ли установлено исключение, с помощью [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_Occurred()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Occurred). Эта функция возвращает текущий объект исключения или `NULL`, если исключение не произошло. Обычно нет необходимости вызывать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_Occurred()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Occurred), чтобы проверить, произошло ли исключение в результате вызова функции, так как это можно понять по возвращаемому значению.

Когда функция *f*, вызывающая другую функцию *g*, обнаруживает, что последняя завершилась с ошибкой, *f* сама должна вернуть значение ошибки (обычно `NULL` или `-1`). Ей не следует вызывать одну из функций `PyErr_*` — одна из них уже была вызвана функцией *g*. Затем функция *f* должна вернуть индикатор ошибки своей функции, которая её вызвала, опять же без вызова `PyErr_*`, и так далее — наиболее подробная причина ошибки уже была сообщена функцией, которая её впервые обнаружила. Как только ошибка достигает главного цикла интерпретатора Python, выполнение текущего Python-кода прерывается и начинается поиск обработчика исключения, указанного программистом Python.

(Существуют ситуации, когда модуль может предоставить более детализированное сообщение об ошибке, вызвав другую функцию `PyErr_*`, и в таких случаях это вполне допустимо. Однако, как правило, в этом нет необходимости, так как это может привести к потере информации о причине ошибки: большинство операций могут завершиться с ошибкой по разным причинам.)

Для того, чтобы игнорировать исключение, установленное в результате вызова функции, которая завершилась с ошибкой, условие исключения должно быть явно очищено с помощью вызова [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_Clear()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Clear). C-код должен вызывать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_Clear()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Clear) только в том случае, если он хочет обработать ошибку самостоятельно (например, попробовав что-то другое или сделав вид, что ничего не случилось), а не передавать её интерпретатору.

Каждый неудачный вызов <span style="font-family: Consolas, sans-serif;">malloc()</span> должен превращаться в исключение — прямой вызывающий <span style="font-family: Consolas, sans-serif;">malloc()</span> (или <span style="font-family: Consolas, sans-serif;">realloc()</span>) должен вызвать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_NoMemory()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_NoMemory) и сам вернуть индикатор ошибки. Все функции создания объектов (например, [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyLong_FromLong()</span>](https://docs.python.org/3/c-api/long.html#c.PyLong_FromLong)) уже делают это, поэтому это замечание актуально только для тех, кто вызывает malloc() напрямую.

Также стоит отметить, что, за исключением таких функций, как [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple) и её аналогов, функции, возвращающие целочисленный статус, обычно возвращают положительное значение или ноль в случае успеха и `-1` в случае ошибки, как это происходит в системных вызовах Unix.

Будьте осторожны при очистке мусора (вызывайте [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XDECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XDECREF) или [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) для объектов, которые вы уже создали), когда возвращаете индикатор ошибки!

Выбор того, какое исключение вызвать, полностью зависит от вас. Существуют заранее определённые C-объекты, соответствующие всем встроенным исключениям Python, такие как <span style="font-family: Consolas, sans-serif;">PyExc_ZeroDivisionError</span>, которые вы можете использовать напрямую. Конечно, вам следует разумно выбирать исключения — не используйте <span style="font-family: Consolas, sans-serif;">PyExc_TypeError</span> для того, чтобы указать, что файл не удалось открыть (для этого лучше подойдёт <span style="font-family: Consolas, sans-serif;">PyExc_OSError</span>). Если что-то не так с аргументами, функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple) обычно вызывает <span style="font-family: Consolas, sans-serif;">PyExc_TypeError</span>. Если у вас есть аргумент, чье значение должно быть в определённом диапазоне или удовлетворять другим условиям, то <span style="font-family: Consolas, sans-serif;">PyExc_ValueError</span> будет подходящим выбором.

Вы также можете определить новое исключение, уникальное для вашего модуля. Для этого обычно объявляют статическую переменную объекта в начале файла:

```c
static PyObject *SpamError;
```

и инициализируют её в функции инициализации модуля (<span style="font-family: Consolas, sans-serif;">PyInit_spam()</span>) с объектом исключения:

```c
PyMODINIT_FUNC
PyInit_spam(void)
{
    PyObject *m;

    m = PyModule_Create(&spammodule);
    if (m == NULL)
        return NULL;

    SpamError = PyErr_NewException("spam.error", NULL, NULL);
    if (PyModule_AddObjectRef(m, "error", SpamError) < 0) {
        Py_CLEAR(SpamError);
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Обратите внимание, что имя этого исключения в Python — <span style="font-family: Consolas, sans-serif;">spam.error</span>. Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_NewException()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_NewException) может создать класс, унаследованный от класса [<u>Exception</u>](https://docs.python.org/3/library/exceptions.html#Exception) (если вместо `NULL` не передан другой класс), который описан в разделе [<u>Built-in Exceptions</u>](https://docs.python.org/3/library/exceptions.html#bltin-exceptions).

Также стоит отметить, что переменная <span style="font-family: Consolas, sans-serif;">SpamError</span> сохраняет ссылку на только что созданный класс исключения; это сделано намеренно! Поскольку исключение может быть удалено из модуля внешним кодом, необходима собственная ссылка на класс, чтобы гарантировать, что он не будет удалён, что может привести к тому, что <span style="font-family: Consolas, sans-serif;">SpamError</span> станет висячим указателем. Если это произойдёт, C-код, вызывающий это исключение, может вызвать аварийное завершение программы (core dump) или другие непредвиденные побочные эффекты.

Мы обсудим использование [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyMODINIT_FUNC</span>](https://docs.python.org/3/c-api/intro.html#c.PyMODINIT_FUNC) в качестве типа возвращаемого значения функции позже в этом примере.

Исключение <span style="font-family: Consolas, sans-serif;">spam.error</span> может быть вызвано в вашем расширении с помощью вызова [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_SetString()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_SetString), как показано ниже:

```c
static PyObject *
spam_system(PyObject *self, PyObject *args)
{
    const char *command;
    int sts;

    if (!PyArg_ParseTuple(args, "s", &command))
        return NULL;
    sts = system(command);
    if (sts < 0) {
        PyErr_SetString(SpamError, "System command failed");
        return NULL;
    }
    return PyLong_FromLong(sts);
}
```

## 1.3. Возвращение к примеру
Возвращаясь к нашему примеру функции, теперь вы должны понять следующую инструкцию:

```c
if (!PyArg_ParseTuple(args, "s", &command))
    return NULL;
```

Он вернёт `NULL` (индикатор ошибки для функций, возвращающих указатели на объекты), если в списке аргументов обнаружена ошибка, полагаясь на исключение, установленное функцией [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple). В противном случае строковое значение аргумента копируется в локальную переменную <span style="font-family: Consolas, sans-serif;">command</span>. Это присваивание указателя, и вам не следует изменять строку, на которую он указывает (поэтому в стандартном C переменная command должна быть правильно объявлена как `const char *command`).

Следующее выражение — это вызов функции Unix <span style="font-family: Consolas, sans-serif;">system()</span>, в который передаётся строка, полученная нами из [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple):

```c
sts = system(command);
```

Наша функция <span style="font-family: Consolas, sans-serif;">spam.system()</span> должна вернуть значение переменной <span style="font-family: Consolas, sans-serif;">sts</span> в виде объекта Python. Это выполняется с помощью функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyLong_FromLong()</span>](https://docs.python.org/3/c-api/long.html#c.PyLong_FromLong).

```c
return PyLong_FromLong(sts);
```

В этом случае она вернёт объект целого числа. (Да, даже целые числа являются объектами в куче в Python!)

Если у вас есть C-функция, которая не возвращает полезного значения (функция, возвращающая `void`), соответствующая Python-функция должна вернуть None. Для этого используется следующий идиоматический приём (который реализован через макрос [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_RETURN_NONE</span>](https://docs.python.org/3/c-api/none.html#c.Py_RETURN_NONE)):

```c
Py_INCREF(Py_None);
return Py_None;
```

[<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_None</span>](https://docs.python.org/3/c-api/none.html#c.Py_RETURN_NONE) — это C-имя для специального объекта Python `None`. Это настоящий объект Python, а не указатель `NULL`, который обычно означает «ошибку» в большинстве контекстов, как мы уже видели.

## 1.4. Таблица методов модуля и функция инициализации
Рассмотрим, как функция <span style="font-family: Consolas, sans-serif;">spam_system()</span> вызывается из Python-программ. Для начала нам нужно перечислить её имя и адрес в «таблице методов»:

```c
static PyMethodDef SpamMethods[] = {
    ...
    {"system",  spam_system, METH_VARARGS,
     "Execute a shell command."},
    ...
    {NULL, NULL, 0, NULL}        /* Sentinel */
};
```

Обратите внимание на третий элемент (`METH_VARARGS`). Это флаг, который сообщает интерпретатору, какую конвенцию вызова следует использовать для C-функции. Обычно это всегда должно быть `METH_VARARGS` или `METH_VARARGS | METH_KEYWORDS`; значение `0` означает, что используется устаревшая версия [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple).

При использовании только `METH_VARARGS` функция должна ожидать, что параметры на уровне Python будут переданы в виде кортежа, который можно обработать с помощью [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple); более подробную информацию об этой функции можно найти ниже.

Бит [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">METH_KEYWORDS</span>](https://docs.python.org/3/c-api/structures.html#c.METH_KEYWORDS) может быть установлен в третьем поле, если аргументы-ключевые слова должны быть переданы функции. В этом случае C-функция должна принимать третий параметр типа `PyObject *`, который будет представлять собой словарь ключевых слов. Для парсинга аргументов такой функции используйте [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">yArg_ParseTupleAndKeywords()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTupleAndKeywords).

Таблица методов должна быть указана в структуре определения модуля:

```c
static struct PyModuleDef spammodule = {
    PyModuleDef_HEAD_INIT,
    "spam",   /* имя модуля */
    spam_doc, /* документация модуля (может быть NULL) */
    -1,       /* размер состояния на интерпретатор модуля, или -1, если модуль хранит состояние в глобальных переменных. */
    SpamMethods
};
```

Эта структура, в свою очередь, должна быть передана интерпретатору в функции инициализации модуля. Функция инициализации должна называться <span style="font-family: Consolas, sans-serif;">PyInit_name()</span>, где *name* — это имя модуля, и должна быть единственным нестатическим элементом, определённым в файле модуля:

```c
PyMODINIT_FUNC
PyInit_spam(void)
{
    return PyModule_Create(&spammodule);
}
```

Обратите внимание, что [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyMODINIT_FUNC</span>](https://docs.python.org/3/c-api/intro.html#c.PyMODINIT_FUNC) объявляет функцию с типом возвращаемого значения `PyObject *`, указывает на любые специальные декларации связи, требуемые платформой, и для C++ объявляет функцию как `extern "C"`.

Когда программа Python импортирует модуль <span style="font-family: Consolas, sans-serif;">spam</span> в первый раз, вызывается функция <span style="font-family: Consolas, sans-serif;">PyInit_spam()</span>. (См. ниже комментарии о встраивании Python.) Эта функция вызывает [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyModule_Create()</span>](https://docs.python.org/3/c-api/module.html#c.PyModule_Create), которая возвращает объект модуля, и вставляет встроенные объекты функций в только что созданный модуль на основе таблицы (массива структур [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyMethodDef</span>](https://docs.python.org/3/c-api/structures.html#c.PyMethodDef)), найденной в определении модуля. [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyModule_Create()</span>](https://docs.python.org/3/c-api/module.html#c.PyModule_Create) возвращает указатель на объект модуля, который она создаёт. Она может завершиться с фатальной ошибкой в случае некоторых ошибок или вернуть `NULL`, если модуль не может быть инициализирован должным образом. Функция инициализации должна вернуть объект модуля своему вызывающему, чтобы он был вставлен в `sys.modules`.

При встраивании Python функция <span style="font-family: Consolas, sans-serif;">PyInit_spam()</span> не вызывается автоматически, если только нет записи в таблице <span style="font-family: Consolas, sans-serif;">PyImport_Inittab</span>. Чтобы добавить модуль в таблицу инициализации, используйте [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyImport_AppendInittab()</span>](https://docs.python.org/3/c-api/import.html#c.PyImport_AppendInittab), а затем, при необходимости, выполните импорт модуля:

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

int
main(int argc, char *argv[])
{
    PyStatus status;
    PyConfig config;
    PyConfig_InitPythonConfig(&config);

    /* Добавляем встроенный модуль до вызова Py_Initialize */
    if (PyImport_AppendInittab("spam", PyInit_spam) == -1) {
        fprintf(stderr, "Error: could not extend in-built modules table\n");
        exit(1);
    }

    /* Передаём argv[0] интерпретатору Python */
    status = PyConfig_SetBytesString(&config, &config.program_name, argv[0]);
    if (PyStatus_Exception(status)) {
        goto exception;
    }

    /* Инициализировать интерпретатор Python.  Обязательно.
       Если этот шаг не удастся - критическая ошибка. */
    status = Py_InitializeFromConfig(&config);
    if (PyStatus_Exception(status)) {
        goto exception;
    }
    PyConfig_Clear(&config);

    /* Дополнительно импортируем модуль; альтернативно,
       импорт может быть отложен до тех пор, пока встроенный скрипт не импортирует его. */
    PyObject *pmodule = PyImport_ImportModule("spam");
    if (!pmodule) {
        PyErr_Print();
        fprintf(stderr, "Error: could not import module 'spam'\n");
    }

    // ... используем Python C API ...

    return 0;

  exception:
     PyConfig_Clear(&config);
     Py_ExitStatusException(status);
}
```

> **Примечание**: Удаление записей из `sys.modules` или импорт компилированных модулей в несколько интерпретаторов внутри процесса (или после вызова `fork()`, если не было промежуточного вызова `exec()`) может вызвать проблемы для некоторых расширений. Авторы расширений должны проявлять осторожность при инициализации внутренних структур данных.

Более подробный пример модуля включён в исходный код Python в файле `Modules/xxmodule.c`. Этот файл можно использовать как шаблон или просто рассматривать в качестве примера.

> **Примечание**: В отличие от нашего примера с `spam`, `xxmodule` использует *многоэтапную инициализацию* (нововведение в Python 3.5), где структура PyModuleDef возвращается из `PyInit_spam`, а создание модуля остаётся на усмотрение механизма импорта. Для подробностей о многоэтапной инициализации см. [<u><b>PEP 489</b></u>](https://peps.python.org/pep-0489/).

## 1.5. Компиляция и компоновка
Перед тем как использовать ваше новое расширение, вам нужно выполнить еще два шага: скомпилировать и связать его с системой Python. Если вы используете динамическую загрузку, детали могут зависеть от типа динамической загрузки, используемой вашей системой. Подробнее об этом можно узнать в главах, посвященных созданию модулей расширений (глава [<u>Building C and C++ Extensions</u>](#4-сборка-расширений-на-c-и-c)), а также в разделе, описывающем особенности сборки на Windows (глава [<u>Building C and C++ Extensions on Windows</u>](#5-сборка-расширений-на-c-и-c-в-windows)).

Если вы не можете использовать динамическую загрузку или хотите сделать ваш модуль постоянной частью интерпретатора Python, вам потребуется изменить конфигурацию и пересобрать интерпретатор. К счастью, на Unix это делается очень просто: просто поместите ваш файл (например, `spammodule.c`) в каталог `Modules/` распакованного исходного кода Python и добавьте строку в файл `Modules/Setup.local`, описывающую ваш файл:

```sh
spam spammodule.o
```

и пересоберите интерпретатор, запустив **make** в корневом каталоге. Вы также можете запустить **make** в подкаталоге `Modules/`, но перед этим необходимо пересобрать `Makefile`, выполнив команду '**make** Makefile'. (Это требуется каждый раз после изменения файла `Setup`.)

Если вашему модулю необходимо подключить дополнительные библиотеки, их также можно указать в строке конфигурационного файла, например:

```sh
spam spammodule.o -lX11
```

## 1.6. Вызов функций Python из C
До этого момента мы сосредотачивались на том, как сделать функции на C вызываемыми из Python. Однако обратный процесс также полезен — вызов функций Python из C. Это особенно актуально для библиотек, поддерживающих так называемые функции «обратного вызова» (callback). Если интерфейс на C использует обратные вызовы, то эквивалент на Python часто должен предоставлять механизм обратного вызова для программиста. Реализация этого требует вызова функций Python из функций обратного вызова на C. Также возможны и другие способы использования.

К счастью, интерпретатор Python легко поддерживает рекурсивные вызовы, и существует стандартный интерфейс для вызова функций Python. (Я не буду подробно останавливаться на том, как вызвать парсер Python с определенной строкой в качестве входных данных — если вас это интересует, загляните в реализацию опции командной строки [<u>-c</u>](https://docs.python.org/3/using/cmdline.html#cmdoption-c) в файле `Modules/main.c` исходного кода Python.)

Вызов функции Python довольно прост. Сначала программа на Python должна каким-то образом передать вам объект функции Python. Для этого вам следует предоставить специальную функцию (или другой интерфейс). Когда эта функция вызывается, сохраните указатель на объект функции Python (не забудьте использовать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_INCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF), чтобы увеличить счетчик ссылок!) в глобальной переменной или в другом подходящем месте. 

Например, следующая функция может быть частью определения модуля:

```c
static PyObject *my_callback = NULL;

static PyObject *
my_set_callback(PyObject *dummy, PyObject *args)
{
    PyObject *result = NULL;
    PyObject *temp;

    if (PyArg_ParseTuple(args, "O:set_callback", &temp)) {
        if (!PyCallable_Check(temp)) {
            PyErr_SetString(PyExc_TypeError, "parameter must be callable");
            return NULL;
        }
        Py_XINCREF(temp);         /* Добавляем ссылку на callback */
        Py_XDECREF(my_callback);  /* Освобождаем предыдущий callback. */
        my_callback = temp;       /* Сохраняем новый оcallback */
        /* Шаблонный код для возврата "None" */
        Py_INCREF(Py_None);
        result = Py_None;
    }
    return result;
}
```

Эту функцию необходимо зарегистрировать в интерпретаторе с флагом [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">METH_VARARGS</span>](https://docs.python.org/3/c-api/structures.html#c.METH_VARARGS); подробнее об этом можно прочитать в разделе [<u>Таблица методов модуля и функция инициализации</u>](#14-таблица-методов-модуля-и-функция-инициализации). Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple) и её аргументы описаны в разделе [<u>Извлечение параметров в функциях расширения</u>](#17-извлечение-параметров-в-функциях-расширения).

Макросы [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XINCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XINCREF) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XDECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XDECREF) увеличивают/уменьшают счетчик ссылок объекта и безопасны при использовании с `NULL`-указателями (однако стоит отметить, что в данном контексте переменная <span style="font-family: Consolas, sans-serif;">temp</span> не будет `NULL`). Дополнительную информацию о них можно найти в разделе [<u>Подсчёт ссылок (Reference Counts)</u>](#110-подсчёт-ссылок-reference-counts).

Позже, когда придет время вызвать функцию, используйте C-функцию [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject). Эта функция принимает два аргумента, оба — указатели на произвольные объекты Python: саму функцию Python и список аргументов. Список аргументов должен всегда быть объектом типа кортеж, длина которого равна количеству аргументов. Чтобы вызвать функцию Python без аргументов, передайте `NULL` или пустой кортеж; чтобы вызвать её с одним аргументом, передайте кортеж с одним элементом. Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BuildValue()</span>](https://docs.python.org/3/c-api/arg.html#c.Py_BuildValue) возвращает кортеж, если её строка формата состоит из нуля или более кодов формата, заключенных в скобки. 

Например:

```c
int arg;
PyObject *arglist;
PyObject *result;
...
arg = 123;
...
/* Время вызвать callback */
arglist = Py_BuildValue("(i)", arg);
result = PyObject_CallObject(my_callback, arglist);
Py_DECREF(arglist);
```

[<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject) возвращает указатель на объект Python: это значение, которое возвращает сама функция Python. [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject) является «нейтральной» в отношении счетчиков ссылок по отношению к своим аргументам. В приведенном примере новый кортеж создается для использования в качестве списка аргументов, и он освобождается с помощью [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) сразу после вызова [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject).

Возвращаемое значение функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject) является «новым»: либо это совершенно новый объект, либо существующий объект, чей счетчик ссылок был увеличен. Поэтому, если вы не планируете сохранить его в глобальной переменной, вам следует как-то вызвать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) для этого результата, даже (и особенно!) если вы не заинтересованы в его значении.

Прежде чем это сделать, важно проверить, что возвращаемое значение не равно `NULL`. Если это так, значит, функция Python завершилась с ошибкой, вызвав исключение. Если C-код, который вызвал [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject), был вызван из Python, он должен вернуть информацию об ошибке своему Python-вызывающему коду, чтобы интерпретатор мог отобразить трассировку стека, или чтобы вызывающий Python-код мог обработать исключение. Если это невозможно или нежелательно, исключение следует очистить, вызвав [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyErr_Clear()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Clear). 

Например:

```c
if (result == NULL)
    return NULL; /* Передаём ошибку назад */
...use result...
Py_DECREF(result);
```

В зависимости от желаемого интерфейса для функции обратного вызова Python, возможно, вам также придется предоставить список аргументов для функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_CallObject()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_CallObject). В некоторых случаях список аргументов также передается программой на Python через тот же интерфейс, который был использован для указания функции обратного вызова. В таком случае его можно сохранить и использовать аналогично объекту функции. В других случаях вам нужно будет создать новый кортеж для передачи в качестве списка аргументов. Самый простой способ сделать это — использовать функцию [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BuildValue()</span>](https://docs.python.org/3/c-api/arg.html#c.Py_BuildValue). 

Например, если вы хотите передать целочисленный код события, можно использовать следующий код:

```c
PyObject *arglist;
...
arglist = Py_BuildValue("(l)", eventcode);
result = PyObject_CallObject(my_callback, arglist);
Py_DECREF(arglist);
if (result == NULL)
    return NULL; /* Передаём ошибку назад */
/* Здесь можно использовать результат */
Py_DECREF(result);
```

Обратите внимание на размещение `Py_DECREF(arglist)` сразу после вызова, до проверки на ошибку! Также стоит отметить, что строго говоря, этот код не является завершенным: функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BuildValue()</span>](https://docs.python.org/3/c-api/arg.html#c.Py_BuildValue) может не иметь достаточно памяти для выполнения, и это следует проверить.

Вы также можете вызвать функцию с именованными аргументами, используя [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_Call()</span>](https://docs.python.org/3/c-api/call.html#c.PyObject_Call), которая поддерживает как обычные, так и ключевые аргументы. Как и в примере выше используем [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BuildValue()</span>](https://docs.python.org/3/c-api/arg.html#c.Py_BuildValue) для создания словаря. 
 
```c
PyObject *dict;
...
dict = Py_BuildValue("{s:i}", "name", val);
result = PyObject_Call(my_callback, NULL, dict);
Py_DECREF(dict);
if (result == NULL)
    return NULL; /* Передаём ошибку назад */
/* Здесь можно использовать результат */
Py_DECREF(result);
```

## 1.7. Извлечение параметров в функциях расширения
Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple) объявлена следующим образом:

```c
int PyArg_ParseTuple(PyObject *arg, const char *format, ...);
```

Аргумент *arg* должен быть объектом типа кортеж, содержащим список аргументов, переданных из Python в C-функцию. Аргумент *format* должен быть строкой формата, синтаксис которой объясняется в разделе [<u>Парсинг аргументов и построение значений</u>](https://docs.python.org/3/c-api/arg.html#arg-parsing) в руководстве Python/C API. Оставшиеся аргументы должны быть адресами переменных, тип которых определяется строкой формата.

Обратите внимание, что хотя функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple) проверяет, что аргументы Python имеют необходимые типы, она не может проверять корректность адресов C-переменных, переданных в вызов: если вы допустите ошибку здесь, ваш код, скорее всего, завершится с ошибкой или, по крайней мере, перезапишет случайные участки памяти. Так что будьте осторожны!

Также обратите внимание, что все ссылки на объекты Python, передаваемые вызывающему коду, являются *заимствованными* ссылками; не уменьшайте их счетчик ссылок!

Вот несколько примеров вызовов:

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>
```

```c
int ok;
int i, j;
long k, l;
const char *s;
Py_ssize_t size;

ok = PyArg_ParseTuple(args, ""); /* Нет аргументов */
    /* Python вызов: f() */
```

```c
ok = PyArg_ParseTuple(args, "s", &s); /* Строка */
    /* Возможный Python вызов: f('whoops!') */
```

```c
ok = PyArg_ParseTuple(args, "lls", &k, &l, &s); /* Два больших (long) целочисленных числа и строка */
    /* Возможный Python вызов: f(1, 2, 'three') */
```

```c
ok = PyArg_ParseTuple(args, "(ii)s#", &i, &j, &s, &size);
    /* Пара целых чисел и строка, размер которой также возвращается */
    /* Возможный Python вызов: f((1, 2), 'three') */
```

```c
{
    const char *file;
    const char *mode = "r";
    int bufsize = 0;
    ok = PyArg_ParseTuple(args, "s|si", &file, &mode, &bufsize);
    /* Строка, а также, опционально, другая строка и целое число. */
    /* Возможные Python вызовы:
       f('spam')
       f('spam', 'w')
       f('spam', 'wb', 100000) */
}
```

```c
{
    int left, top, right, bottom, h, v;
    ok = PyArg_ParseTuple(args, "((ii)(ii))(ii)",
             &left, &top, &right, &bottom, &h, &v);
    /* Прямоугольник и точка */
    /* Возможный Python вызов:
       f(((0, 0), (400, 300)), (10, 10)) */
}
```

```c
{
    Py_complex c;
    ok = PyArg_ParseTuple(args, "D:myfunction", &c);
    /* Комплексное число с указанием имени функции для обработки ошибок. */
    /* Возможный Python вызов: myfunction(1+2j) */
}
```

## 1.8. Ключевые параметры для функций расширения
Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTupleAndKeywords()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTupleAndKeywords) объявлена следующим образом:

```c
int PyArg_ParseTupleAndKeywords(PyObject *arg, PyObject *kwdict,
                                const char *format, char * const *kwlist, ...);
```

Параметры *arg* и *format* идентичны параметрам функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple). Параметр *kwdict* — это словарь ключевых слов, получаемый как третий параметр от среды выполнения Python. Параметр *kwlist* представляет собой список строк, завершённый `NULL`, который идентифицирует параметры; имена сопоставляются с типовой информацией из *format* слева направо. В случае успеха, функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTupleAndKeywords()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTupleAndKeywords) возвращает `true`, в противном случае она возвращает `false` и возбуждает соответствующее исключение.

> **Примечание**: Вложенные кортежи не могут быть разобраны при использовании именованных аргументов! Если переданы именованные параметры, которых нет в списке kwlist, будет вызвано исключение [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">TypeError</span>](https://docs.python.org/3/library/exceptions.html#TypeError).

Вот пример модуля, который использует ключевые слова, основанный на примере Джеффа Филбрика (philbrick@hks.com):

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

static PyObject *
keywdarg_parrot(PyObject *self, PyObject *args, PyObject *keywds)
{
    int voltage;
    const char *state = "a stiff";
    const char *action = "voom";
    const char *type = "Norwegian Blue";

    static char *kwlist[] = {"voltage", "state", "action", "type", NULL};

    if (!PyArg_ParseTupleAndKeywords(args, keywds, "i|sss", kwlist,
                                     &voltage, &state, &action, &type))
        return NULL;

    printf("-- This parrot wouldn't %s if you put %i Volts through it.\n",
           action, voltage);
    printf("-- Lovely plumage, the %s -- It's %s!\n", type, state);

    Py_RETURN_NONE;
}

static PyMethodDef keywdarg_methods[] = {
    /* Приведение типа функции необходимо, поскольку значения
     * PyCFunction принимают только два параметра типа PyObject*, а
     * функция keywdarg_parrot() принимает три.
     */
    {"parrot", (PyCFunction)(void(*)(void))keywdarg_parrot, METH_VARARGS | METH_KEYWORDS,
     "Print a lovely skit to standard output."},
    {NULL, NULL, 0, NULL}   /* маркер */
};

static struct PyModuleDef keywdargmodule = {
    PyModuleDef_HEAD_INIT,
    "keywdarg",
    NULL,
    -1,
    keywdarg_methods
};

PyMODINIT_FUNC
PyInit_keywdarg(void)
{
    return PyModule_Create(&keywdargmodule);
}
```

## 1.9. Создание произвольных значений
Эта функция является аналогом функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple). Она объявлена следующим образом:

```c
PyObject *Py_BuildValue(const char *format, ...);
```

Она распознает набор форматов, аналогичных тем, что распознаются функцией [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple), но аргументы (которые передаются в функцию, а не выводятся) не должны быть указателями, а только значениями. Функция возвращает новый объект Python, подходящий для возвращения из C-функции, вызываемой из Python.

Отличие от [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple) состоит в том, что он требует, чтобы его первым аргументом был кортеж (так как списки аргументов в Python внутренне всегда представлены в виде кортежей), [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BuildValue()</span>](https://docs.python.org/3/c-api/arg.html#c.Py_BuildValue) не всегда создаёт кортеж. Он создаёт кортеж, только если строка формата содержит два или более форматных спецификатора. Если строка формата пуста, функция возвращает `None`. Если строка формата содержит ровно один форматный спецификатор, возвращается объект, описанный этим спецификатором. Чтобы заставить функцию вернуть кортеж размером 0 или 1, необходимо заключить строку формата в круглые скобки.

Примеры (слева вызов, справа – полученное значение в Python):

```c
Py_BuildValue("")                        None
Py_BuildValue("i", 123)                  123
Py_BuildValue("iii", 123, 456, 789)      (123, 456, 789)
Py_BuildValue("s", "hello")              'hello'
Py_BuildValue("y", "hello")              b'hello'
Py_BuildValue("ss", "hello", "world")    ('hello', 'world')
Py_BuildValue("s#", "hello", 4)          'hell'
Py_BuildValue("y#", "hello", 4)          b'hell'
Py_BuildValue("()")                      ()
Py_BuildValue("(i)", 123)                (123,)
Py_BuildValue("(ii)", 123, 456)          (123, 456)
Py_BuildValue("(i,i)", 123, 456)         (123, 456)
Py_BuildValue("[i,i]", 123, 456)         [123, 456]
Py_BuildValue("{s:i,s:i}",
              "abc", 123, "def", 456)    {'abc': 123, 'def': 456}
Py_BuildValue("((ii)(ii)) (ii)",
              1, 2, 3, 4, 5, 6)          (((1, 2), (3, 4)), (5, 6))
```

## 1.10. Подсчёт ссылок (Reference Counts)
В таких языках, как C или C++, программист отвечает за динамическое выделение и освобождение памяти в куче. В C это выполняется с помощью функций <span style="font-family: Consolas, sans-serif;">malloc()</span> и <span style="font-family: Consolas, sans-serif;">free()</span>. В C++ для этого используются операторы `new` и `delete`, имеющие по сути тот же смысл, однако в дальнейшем мы ограничимся рассмотрением случая для C.

Каждый блок памяти, выделенный с помощью <span style="font-family: Consolas, sans-serif;">malloc()</span>, должен в конечном итоге быть возвращён в пул доступной памяти ровно одним вызовом <span style="font-family: Consolas, sans-serif;">free()</span>. Важно вызывать <span style="font-family: Consolas, sans-serif;">free()</span> в правильный момент. Если адрес блока теряется, но <span style="font-family: Consolas, sans-serif;">free()</span> для него не вызывается, память, которую он занимает, не может быть повторно использована до завершения работы программы. Это называется *утечкой памяти (memory leak)*. С другой стороны, если программа вызывает <span style="font-family: Consolas, sans-serif;">free()</span> для блока, а затем продолжает его использовать, это создаёт конфликт с повторным использованием этого блока через другой вызов <span style="font-family: Consolas, sans-serif;">malloc()</span>. Это называется *использованием освобождённой памяти (using freed memory)*. Это может привести к тем же проблемам, что и обращение к неинициализированным данным — аварийному завершению программы, некорректным результатам, загадочным сбоям.

Частыми причинами утечек памяти являются необычные пути выполнения кода. Например, функция может выделить блок памяти, выполнить некоторые вычисления, а затем освободить этот блок. Однако, если в функцию вносятся изменения, например добавляется проверка, которая выявляет ошибочное состояние и приводит к преждевременному выходу, можно легко забыть освободить выделенный блок памяти перед выходом, особенно если этот выход был добавлен позже. Такие утечки, появившись однажды, часто остаются незамеченными в течение долгого времени: ошибка срабатывает лишь в небольшом числе вызовов, а современные машины обладают достаточным объёмом виртуальной памяти, поэтому утечка становится заметной только в долгоживущих процессах, которые часто вызывают проблемную функцию. Поэтому важно предотвращать утечки, придерживаясь определённых соглашений или стратегий кодирования, которые минимизируют вероятность подобных ошибок.

Поскольку Python активно использует <span style="font-family: Consolas, sans-serif;">malloc()</span> и <span style="font-family: Consolas, sans-serif;">free()</span>, ему необходима стратегия для предотвращения утечек памяти и использования освобождённой памяти. Выбранный метод называется *подсчётом ссылок (reference counting)*. Принцип работы прост: каждый объект содержит счётчик, который увеличивается, когда где-то сохраняется ссылка на объект, и уменьшается, когда ссылка удаляется. Когда счётчик достигает нуля, это означает, что последняя ссылка на объект была удалена, и объект освобождается.

Альтернативная стратегия называется *автоматическим сборщиком мусора (automatic garbage collection)*. (Иногда подсчёт ссылок тоже относят к методам сборки мусора, поэтому я использую слово «автоматический», чтобы различать эти два подхода.) Главное преимущество автоматического сборщика мусора в том, что пользователю не нужно явно вызывать <span style="font-family: Consolas, sans-serif;">free()</span>. (Другое заявленное преимущество — улучшение скорости работы или использования памяти, хотя это не является бесспорным фактом.) Недостаток заключается в том, что для C не существует по-настоящему переносимого автоматического сборщика мусора, в то время как подсчёт ссылок можно реализовать на любой платформе (при условии, что доступны функции <span style="font-family: Consolas, sans-serif;">malloc()</span> и <span style="font-family: Consolas, sans-serif;">free()</span>, что гарантируется стандартом C). Возможно, однажды появится достаточно переносимый автоматический сборщик мусора для C. До тех пор нам придётся обходиться подсчётом ссылок.

Хотя Python использует традиционную реализацию подсчёта ссылок, он также предлагает детектор циклов, который помогает обнаруживать циклические ссылки. Это позволяет приложениям не беспокоиться о создании прямых или косвенных циклических ссылок, которые являются слабым местом сборки мусора, основанной только на подсчёте ссылок. Циклы ссылок состоят из объектов, которые (возможно, косвенно) ссылаются сами на себя, так что у каждого объекта в цикле счётчик ссылок остаётся ненулевым. Обычные реализации подсчёта ссылок не способны освободить память, занимаемую объектами внутри такого цикла, или объектами, на которые они ссылаются, даже если больше нет внешних ссылок на сам цикл.

Детектор циклов способен обнаруживать циклы мусора и освобождать занимаемую ими память. Модуль [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">gc</span>](https://docs.python.org/3/library/gc.html#module-gc) предоставляет способ запустить детектор (функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">collect()</span>](https://docs.python.org/3/library/gc.html#gc.collect)), а также интерфейсы для настройки и возможность отключить детектор во время выполнения программы.

### 1.10.1 Подсчёт ссылок в Python
Существуют два макроса, `Py_INCREF(x)` и `Py_DECREF(x)`, которые отвечают за увеличение и уменьшение счётчика ссылок. [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) также освобождает объект, когда счётчик ссылок достигает нуля. Для гибкости он не вызывает <span style="font-family: Consolas, sans-serif;">free()</span> напрямую — вместо этого он вызывает функцию через указатель на функцию в *объекте типа* этого объекта (object’s *type object*). Для этой цели (и не только) каждый объект также содержит указатель на свой объект типа.

Остаётся большой вопрос: когда использовать `Py_INCREF(x)` и `Py_DECREF(x)`? Cначала введём несколько терминов. Никто не "владеет" объектом, однако можно *владеть ссылкой* на объект. Счётчик ссылок объекта теперь определяется как количество ссылок, которыми на него владеют. Владелец ссылки отвечает за вызов [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF), когда ссылка больше не нужна. Владение ссылкой может быть передано. Существует три способа распорядиться принадлежащей ссылкой: передать её, сохранить или вызвать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF). Если вы забудете распорядиться принадлежащей ссылкой — это приведёт к утечке памяти.

<a id="footnote-2-back"></a><a id="footnote-3-back"></a>Также возможно "занять" [[2]](#footnote-2) ссылку на объект. "Заёмщик" ссылки не должен вызывать [Py_DECREF()](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) для этого объекта. Заёмщик не должен держать объект дольше, чем его владелец, у которого была занята ссылка. Использование занятых ссылок после того, как владелец распорядился ими, может привести к использованию освобождённой памяти, и это следует полностью избегать [[3]](#footnote-3).

Преимущество заимствования ссылки по сравнению с владением ссылкой заключается в том, что вам не нужно заботиться о её освобождении на всех возможных путях выполнения кода — другими словами, с заимствованной ссылкой вы не рискуете столкнуться с утечкой памяти при преждевременном выходе. Недостаток заимствования по сравнению с владением заключается в том, что существуют некоторые тонкие ситуации, когда в казалось бы правильном коде заимствованная ссылка может быть использована после того, как её владелец, у которого она была заимствована, фактически распорядился ею.

Заимствованную ссылку можно преобразовать в принадлежащую, вызвав [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_INCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF). Это не влияет на статус владельца, у которого была заимствована ссылка — создаётся новая принадлежащая ссылка, и на неё возлагаются все обязанности владельца (новый владелец должен правильно распорядиться ссылкой, так же как и предыдущий владелец).

### 1.10.2 Правила владения
Каждый раз, когда ссылка на объект передаётся в функцию или из неё, часть спецификации интерфейса функции заключается в том, передаётся ли владение ссылкой или нет.

Большинство функций, которые возвращают ссылку на объект, передают владение этой ссылкой. В частности, все функции, создающие новый объект, такие как [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyLong_FromLong()</span>](https://docs.python.org/3/c-api/long.html#c.PyLong_FromLong) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BuildValue()</span>](https://docs.python.org/3/c-api/arg.html#c.Py_BuildValue), передают владение получателю. Даже если объект на самом деле не новый, вы всё равно получаете владение новой ссылкой на этот объект. Например, [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyLong_FromLong()</span>](https://docs.python.org/3/c-api/long.html#c.PyLong_FromLong) поддерживает кеш популярных значений и может вернуть ссылку на объект, который уже был сохранён в кеше.

Многие функции, которые извлекают объекты из других объектов, также передают владение ссылкой, например, [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_GetAttrString()</span>](https://docs.python.org/3/c-api/object.html#c.PyObject_GetAttrString). Однако ситуация менее однозначна в этом случае, так как есть несколько исключений: функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyTuple_GetItem()</span>](https://docs.python.org/3/c-api/tuple.html#c.PyTuple_GetItem), [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyList_GetItem()</span>](https://docs.python.org/3/c-api/list.html#c.PyList_GetItem), [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyDict_GetItem()</span>](https://docs.python.org/3/c-api/dict.html#c.PyDict_GetItem) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyDict_GetItemString()</span>](https://docs.python.org/3/c-api/dict.html#c.PyDict_GetItemString) возвращают ссылки, которые вы заимствуете из кортежа, списка или словаря.

Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyImport_AddModule()</span>](https://docs.python.org/3/c-api/import.html#c.PyImport_AddModule) также возвращает заимствованную ссылку, даже если она на самом деле может создавать объект, который возвращает. Это возможно, потому что владение ссылкой на объект хранится в `sys.modules`.

Когда вы передаёте ссылку на объект в другую функцию, обычно функция заимствует ссылку у вас — если она собирается сохранить её, она использует [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_INCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF), чтобы стать независимым владельцем. Существует ровно два важных исключения из этого правила: [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyTuple_SetItem()</span>](https://docs.python.org/3/c-api/tuple.html#c.PyTuple_SetItem) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyList_SetItem()</span>](https://docs.python.org/3/c-api/list.html#c.PyList_SetItem). Эти функции забирают на себя владение переданным объектом — даже если они завершаются с ошибкой! (Обратите внимание, что [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyDict_SetItem()</span>](https://docs.python.org/3/c-api/dict.html#c.PyDict_SetItem) и подобные функции не забирают владение — они являются "нормальными".)

Когда C-функция вызывается из Python, она заимствует ссылки на свои аргументы от вызывающей стороны. Вызывающая сторона владеет ссылкой на объект, поэтому срок жизни заимствованной ссылки гарантирован до тех пор, пока функция не вернёт управление. Только в случае, когда заимствованную ссылку нужно сохранить или передать дальше, её необходимо преобразовать в принадлежащую ссылку, вызвав [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_INCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF).

Ссылка на объект, возвращаемая из C-функции, вызываемой из Python, должна быть принадлежащей ссылкой — владение передаётся от функции к её вызывающему коду.

### 1.10.3 Тонкий лёд
Есть несколько ситуаций, когда, казалось бы, безобидное использование заимствованной ссылки может привести к проблемам. Все они связаны с неявными вызовами интерпретатора, которые могут привести к освобождению ссылки её владельцем.

Первый и самый важный случай, о котором нужно знать — это использование [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) на несвязанном объекте во время заимствования ссылки на элемент списка. Например

```c
void
bug(PyObject *list)
{
    PyObject *item = PyList_GetItem(list, 0);

    PyList_SetItem(list, 1, PyLong_FromLong(0L));
    PyObject_Print(item, stdout, 0); /* BUG! */
}
```

Эта функция сначала заимствует ссылку на `list[0]`, затем заменяет `list[1]` значением `0`, и наконец выводит заимствованную ссылку. Выглядит безобидно, не так ли? Но это не так!

Проследим логику выполнения внутри [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyList_SetItem()</span>](https://docs.python.org/3/c-api/list.html#c.PyList_SetItem). Список владеет ссылками на все свои элементы, поэтому при замене элемента 1 он должен освободить исходный элемент 1. Теперь предположим, что исходный элемент 1 был экземпляром пользовательского класса, и допустим, что этот класс определял метод <span style="font-family: Consolas, sans-serif;">\_\_del\_\_()</span>. Если счетчик ссылок этого экземпляра класса равен 1, его освобождение вызовет метод <span style="font-family: Consolas, sans-serif;">\_\_del\_\_()</span>.

Поскольку метод <span style="font-family: Consolas, sans-serif;">\_\_del\_\_()</span> написан на Python, он может выполнять произвольный Python-код. Может ли он сделать что-то, что аннулирует ссылку на `item` в функции <span style="font-family: Consolas, sans-serif;">bug()</span>? Ещё как может! Если предположить, что список, переданный в <span style="font-family: Consolas, sans-serif;">bug()</span>, доступен из метода <span style="font-family: Consolas, sans-serif;">\_\_del\_\_()</span>, он может выполнить операцию вида `del list[0]`, и при условии что это была последняя ссылка на данный объект - это освободит связанную с ним память, тем самым сделав `item` недействительным.

Когда вы поняли источник проблемы, решение оказывается простым: нужно временно увеличить счётчик ссылок. Правильная версия функции выглядит так:

```c
void
no_bug(PyObject *list)
{
    PyObject *item = PyList_GetItem(list, 0);

    Py_INCREF(item);
    PyList_SetItem(list, 1, PyLong_FromLong(0L));
    PyObject_Print(item, stdout, 0);
    Py_DECREF(item);
}
```

Другое дело! В одной из старых версий Python присутствовали подобные баги, и одному разработчику пришлось потратить значительное время в C-отладчике, чтобы понять, почему его методы <span style="font-family: Consolas, sans-serif;">\_\_del\_\_()</span> переставали работать...

Второй случай проблем с заимствованными ссылками связан с работой в многопоточной среде. Обычно потоки в интерпретаторе Python не мешают друг другу благодаря глобальной блокировке, защищающей всё объектное пространство. Однако эту блокировку можно временно снять с помощью макроса [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_BEGIN_ALLOW_THREADS</span>](https://docs.python.org/3/c-api/init.html#c.Py_BEGIN_ALLOW_THREADS) и восстановить через [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_END_ALLOW_THREADS</span>](https://docs.python.org/3/c-api/init.html#c.Py_END_ALLOW_THREADS). Это стандартная практика при выполнении блокирующих операций ввода-вывода, позволяющая другим потокам использовать процессор во время ожидания завершения ввода-вывода. Очевидно, что следующая функция имеет ту же проблему, что и предыдущая:

```c
void
bug(PyObject *list)
{
    PyObject *item = PyList_GetItem(list, 0);
    Py_BEGIN_ALLOW_THREADS
    ...some blocking I/O call...
    Py_END_ALLOW_THREADS
    PyObject_Print(item, stdout, 0); /* БАГ! */
}
```

### 1.10.4 Пустые указатели
Как правило, функции, принимающие ссылки на объекты в качестве аргументов, не предполагают передачу `NULL` указателей и могут вызвать аварийное завершение (или привести к последующим ошибкам сегментации) при таком вызове. Функции, возвращающие ссылки на объекты, обычно возвращают `NULL` только для индикации возникновения исключения. Причина отказа от проверки `NULL` аргументов заключается в том, что функции часто передают полученные объекты другим функциям — если бы каждая функция выполняла проверку на `NULL`, это привело бы к множеству избыточных проверок и замедлению выполнения кода.

Проверку на `NULL` лучше выполнять только в точке "источника" - там, где получен указатель, который потенциально может быть `NULL`, например, при вызове <span style="font-family: Consolas, sans-serif;">malloc()</span> или функции, которая может вызвать исключение.

Макросы [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_INCREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF) не проверяют указатели на `NULL` — в отличие от их вариантов [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XINCREF</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XINCREF) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XDECREF</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XDECREF), которые выполняют такую проверку.

Макросы для проверки типа объекта (`Pytype_Check()`) также не проверяют указатели на `NULL` — поскольку существует множество участков кода, где эти макросы вызываются последовательно для проверки объекта на соответствие различным ожидаемым типам, что привело бы к избыточным проверкам. Вариантов этих макросов с проверкой на `NULL` не существует.

<a id="footnote-4-back"></a>Механизм вызова C-функций гарантирует, что список аргументов, передаваемый в C-функции (`args` в примерах), никогда не будет `NULL` — более того, гарантируется, что это всегда будет кортеж [[4]](#footnote-4).

Допустить 'утечку' `NULL` указателя на уровень Python-пользователя является серьёзной ошибкой.

## 1.11. Написание расширений на C++
Существует возможность написания модулей расширения на C++. Однако здесь действуют некоторые ограничения. Если главная программа (интерпретатор Python) скомпилирована и ей связи настроены компилятором C, нельзя использовать глобальные или статические объекты с конструкторами. Это не является проблемой, если связи главной программы созданы компилятором C++. Функции, которые будут вызываться интерпретатором Python (в частности, функции инициализации модулей), должны быть объявлены с использованием `extern "C"`. Нет необходимости заключать заголовочные файлы Python в `extern "C" {...}` — они уже используют эту форму, если определён символ `__cplusplus` (все современные компиляторы C++ определяют этот символ).

## 1.12. Предоставление C API для модуля расширения
Многие модули расширения просто предоставляют новые функции и типы для использования из Python, но иногда код в модуле расширения может быть полезен для других модулей расширения. Например, модуль может реализовать тип "коллекция", который работает как неупорядоченные списки. Подобно тому, как стандартный тип list в Python имеет C API, позволяющий модулям расширения создавать и управлять списками, этот новый тип коллекции должен включать набор C-функций для прямого управления из других модулей расширения.

На первый взгляд это кажется простым: достаточно написать функции (не объявляя их `static`, конечно), предоставить соответствующий заголовочный файл и задокументировать C API. И действительно, это работало бы, если бы все модули расширения всегда статически связывались с интерпретатором Python. Однако при использовании модулей в качестве разделяемых библиотек, символы, определённые в одном модуле, могут быть не видны другому модулю. Детали видимости зависят от операционной системы; некоторые системы используют единое глобальное пространство имён для интерпретатора Python и всех модулей расширения (например, Windows), другие требуют явного указания импортируемых символов на этапе линковки модуля (как AIX) или предлагают выбор из нескольких стратегий (большинство Unix-систем). Более того, даже если символы глобально видимы, модуль, чьи функции требуется вызвать, может быть ещё не загружен!

Следовательно, для обеспечения переносимости нельзя делать никаких предположений о видимости символов. Это означает, что все символы в модулях расширения должны объявляться как `static`, за исключением функции инициализации модуля - во избежание конфликтов имён с другими модулями расширения (как обсуждалось в разделе [Таблица методов модуля и функция инициализации](#14-таблица-методов-модуля-и-функция-инициализации)). Это означает, что символы, которые *должны* быть доступны из других модулей расширения, необходимо экспортировать другим способом.

Python предоставляет специальный механизм для передачи информации на уровне C (указателей) между модулями расширения: капсулы (Capsules). Капсула - это тип данных Python, который хранит указатель (`void*`). Капсулы можно создавать и получать только через их C API, но передавать их можно как любой другой объект Python. В частности, их можно присвоить имени в пространстве имен модуля расширения. Другие модули расширения могут затем импортировать этот модуль, получить значение его имени и извлечь указатель из капсулы.

Существует множество способов использования капсул (Capsules) для экспорта C API модуля расширения. Каждая функция может получить собственную капсулу или все указатели C API могут храниться в массиве, адрес которого публикуется в капсуле. Различные задачи хранения и получения указателей могут по-разному распределяться между модулем, предоставляющим код и клиентскими модулями.

Независимо от выбранного метода, важно правильно именовать ваши капсулы (Capsules). Функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyCapsule_New()</span>](https://docs.python.org/3/c-api/capsule.html#c.PyCapsule_New) принимает параметр name (`const char*`); хотя допустимо передавать `NULL` в качестве имени, мы настоятельно рекомендуем указывать осмысленное имя. Правильно именованные капсулы обеспечивают определенный уровень типобезопасности во время выполнения - невозможно надежно различить безымянные капсулы между собой

В частности, для капсул, предоставляющих доступ к C API, следует использовать имена в соответствии со следующим соглашением:

```sh
modulename.attributename
```

Вспомогательная функция [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyCapsule_Import()</span>](https://docs.python.org/3/c-api/capsule.html#c.PyCapsule_Import) упрощает загрузку C API, предоставляемого через капсулу, но только если имя капсулы соответствует этому соглашению. Такое поведение даёт пользователям C API высокую степень уверенности, что загруженная капсула содержит корректный C API.

Следующий пример демонстрирует подход, при котором основная нагрузка ложится на разработчика экспортирующего модуля, что особенно подходит для часто используемых библиотечных модулей. В этом подходе все указатели C API (в примере всего один!) хранятся в массиве `void` указателей, который затем помещается в капсулу (Capsule). Соответствующий заголовочный файл модуля предоставляет макрос, который автоматизирует импорт модуля и получение его указателей C API; клиентским модулям достаточно вызвать этот макрос перед использованием C API.

Экспортирующий модуль представляет собой модифицированную версию модуля spam из раздела [Простой пример](#11-простой-пример). Функция <span style="font-family: Consolas, sans-serif;">spam.system()</span> не вызывает напрямую системную С-функцию <span style="font-family: Consolas, sans-serif;">system()</span>, а функцию <span style="font-family: Consolas, sans-serif;">PySpam_System()</span>, которая в реальности, конечно, выполняла бы более сложные операции (например, добавляла «spam» к каждой команде). Функция <span style="font-family: Consolas, sans-serif;">PySpam_System()</span> также экспортируется для использования другими модулями расширения.

Функция <span style="font-family: Consolas, sans-serif;">PySpam_System()</span> является обычной C-функцией, объявленной как `static`, как и всё остальное:

```c
static int
PySpam_System(const char *command)
{
    return system(command);
}
```

В функция <span style="font-family: Consolas, sans-serif;">spam_system()</span> внесены небольшие изменения:

```c
static PyObject *
spam_system(PyObject *self, PyObject *args)
{
    const char *command;
    int sts;

    if (!PyArg_ParseTuple(args, "s", &command))
        return NULL;
    sts = PySpam_System(command);
    return PyLong_FromLong(sts);
}
```

В начале модуля, сразу после строки

```c
#include <Python.h>
```

Необходимо добавить ещё две строки:

```c
#define SPAM_MODULE
#include "spammodule.h"
```

Директива `#define` используется, чтобы указать заголовочному файлу, что он включается в экспортирующий модуль, а не в клиентский. Наконец, функция инициализации модуля должна проинициализировать массив указателей C API:

```c
PyMODINIT_FUNC
PyInit_spam(void)
{
    PyObject *m;
    static void *PySpam_API[PySpam_API_pointers];
    PyObject *c_api_object;

    m = PyModule_Create(&spammodule);
    if (m == NULL)
        return NULL;

    /* Инициализируем масств указателей C API */
    PySpam_API[PySpam_System_NUM] = (void *)PySpam_System;

    /* Создаём капсулу, содержащую адрес массива указателей API */
    c_api_object = PyCapsule_New((void *)PySpam_API, "spam._C_API", NULL);

    if (PyModule_Add(m, "_C_API", c_api_object) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Обратите внимание, что `PySpam_API` объявлен как `static` — в противном случае массив указателей перестал бы существовать после завершения <span style="font-family: Consolas, sans-serif;">PyInit_spam()</span>!

Основная часть работы сосредоточена в заголовочном файле `spammodule.h`, который выглядит следующим образом:

```c
#ifndef Py_SPAMMODULE_H
#define Py_SPAMMODULE_H
#ifdef __cplusplus
extern "C" {
#endif

/* Заголовочный файл для spammodule */

/* Функции C API */
#define PySpam_System_NUM 0
#define PySpam_System_RETURN int
#define PySpam_System_PROTO (const char *command)

/* Общее количество указателей C API */
#define PySpam_API_pointers 1


#ifdef SPAM_MODULE
/* Этот раздел используется при компиляции spammodule.c */

static PySpam_System_RETURN PySpam_System PySpam_System_PROTO;

#else
/* Этот раздел используется в модулях, которые используют API spammodule */

static void **PySpam_API;

#define PySpam_System \
 (*(PySpam_System_RETURN (*)PySpam_System_PROTO) PySpam_API[PySpam_System_NUM])

/* Возвращает -1 при ошибке, 0 при успехе.
 * PyCapsule_Import установит исключение при ошибке.
 */
static int
import_spam(void)
{
    PySpam_API = (void **)PyCapsule_Import("spam._C_API", 0);
    return (PySpam_API != NULL) ? 0 : -1;
}

#endif

#ifdef __cplusplus
}
#endif

#endif /* !defined(Py_SPAMMODULE_H) */
```

Чтобы получить доступ к функции <span style="font-family: Consolas, sans-serif;">PySpam_System()</span>, клиентскому модулю достаточно вызвать функцию (точнее, макрос) <span style="font-family: Consolas, sans-serif;">import_spam()</span> в своей функции инициализации:

```c
PyMODINIT_FUNC
PyInit_client(void)
{
    PyObject *m;

    m = PyModule_Create(&clientmodule);
    if (m == NULL)
        return NULL;
    if (import_spam() < 0)
        return NULL;
    /* здесь может быть добавлена дополнительная инициализация */
    return m;
}
```

Основной недостаток этого подхода в том, что файл `spammodule.h` получается довольно сложным. Однако базовая структура одинакова для каждой экспортируемой функции, поэтому её нужно понять только один раз.

Наконец, следует упомянуть, что капсулы (Capsules) предоставляют дополнительную функциональность, особенно полезную для управления памятью при выделении и освобождении указателей, хранящихся в капсуле. Подробности описаны в Python/C API Reference Manual в разделе, посвящённом [<u>капсулам</u>](https://docs.python.org/3/c-api/capsule.html#capsules), а также в реализации капсул (файлы `Include/pycapsule.h` и `Objects/pycapsule.c` в исходном коде Python).

# 2. Определение типов расширений: учебное пособие
Python позволяет разработчику модулей расширений на C определять новые типы, которыми можно управлять из Python-кода, подобно встроенным типам [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">str</span>](https://docs.python.org/3/library/stdtypes.html#str) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">list</span>](https://docs.python.org/3/library/stdtypes.html#list). Код для всех типов расширений следует определённому шаблону, но перед началом работы необходимо разобраться в некоторых деталях. Данный документ представляет собой вводное руководство по этой теме.

## 2.1. Основы
Интерпретатор [<u>CPython</u>](https://docs.python.org/3/glossary.html#term-CPython) рассматривает все объекты Python как переменные типа [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject)*, который служит «базовым типом» для всех объектов Python. Сама структура [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject) содержит только [<u>счётчик ссылок</u>](https://docs.python.org/3/glossary.html#term-reference-count) ([reference count](https://docs.python.org/3/glossary.html#term-reference-count)) объекта и указатель на его «объект типа». Именно здесь происходит основное действие: объект типа определяет, какие (C) функции будут вызваны интерпретатором, например, при поиске атрибута объекта, вызове метода или умножении на другой объект. Эти C-функции называются «методами типа» (type methods).

Таким образом, для определения нового типа расширения необходимо создать новый объект типа.

Такую концепцию проще всего объяснить на примере. Ниже представлен минимальный, но полный модуль расширения на C под названием <span style="font-family: Consolas, sans-serif;">Custom</span>, который определяет новый тип <span style="font-family: Consolas, sans-serif;">custom</span>:

> **Примечание**: Здесь мы демонстрируем традиционный способ определения *статических* типов расширений. Этого должно быть достаточно для большинства случаев. C API также позволяет определять типы расширений с выделением памяти в куче с помощью функции [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyType_FromSpec()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_FromSpec), однако это не рассматривается в данном руководстве.


```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

typedef struct {
    PyObject_HEAD
    /* Type-specific fields go here. */
} CustomObject;

static PyTypeObject CustomType = {
    .ob_base = PyVarObject_HEAD_INIT(NULL, 0)
    .tp_name = "custom.Custom",
    .tp_doc = PyDoc_STR("Custom objects"),
    .tp_basicsize = sizeof(CustomObject),
    .tp_itemsize = 0,
    .tp_flags = Py_TPFLAGS_DEFAULT,
    .tp_new = PyType_GenericNew,
};

static PyModuleDef custommodule = {
    .m_base = PyModuleDef_HEAD_INIT,
    .m_name = "custom",
    .m_doc = "Example module that creates an extension type.",
    .m_size = -1,
};

PyMODINIT_FUNC
PyInit_custom(void)
{
    PyObject *m;
    if (PyType_Ready(&CustomType) < 0)
        return NULL;

    m = PyModule_Create(&custommodule);
    if (m == NULL)
        return NULL;

    if (PyModule_AddObjectRef(m, "Custom", (PyObject *) &CustomType) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Пока что информации довольно много для восприятия сразу, но, надеюсь, некоторые моменты покажутся знакомыми из предыдущей главы. Этот файл определяет три основные вещи:

1. Содержимое **объекта** <span style="font-family: Consolas, sans-serif;">Custom</span> определяется структурой `CustomObject`, которая создаётся для каждого экземпляра <span style="font-family: Consolas, sans-serif;">Custom</span>.
2. Поведение **типа** <span style="font-family: Consolas, sans-serif;">Custom</span> определяется структурой `CustomType`, содержащей флаги и указатели на функции, которые интерпретатор использует при выполнении операций с объектом.
3. Инициализация модуля <span style="font-family: Consolas, sans-serif;">custom</span> осуществляется функцией `PyInit_custom` и связанной с ней структурой `custommodule`.

Начнём с первого пункта:

```c
typedef struct {
    PyObject_HEAD
} CustomObject;
```

Вот что содержит объект Custom. `PyObject_HEAD` - обязательный элемент в начале каждой структуры объекта, который определяет поле `ob_base` типа [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject), содержащее указатель на объект типа, счётчик ссылок (доступ через макросы [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_TYPE</span>](https://docs.python.org/3/c-api/structures.html#c.Py_TYPE) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_REFCNT</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_REFCNT) соответственно). Этот макрос абстрагирует внутреннюю структуру и позволяет добавлять дополнительные поля в [<u>отладочных сборках</u>](https://docs.python.org/3/using/configure.html#debug-build).

> **Примечание**: После макроса [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyObject_HEAD</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject_HEAD) в примере отсутствует точка с запятой. Будьте внимательны и не добавляйте её случайно — некоторые компиляторы выдадут ошибку.

Разумеется, объекты обычно хранят дополнительные данные помимо стандартного шаблона `PyObject_HEAD`. Например, вот определение стандартного вещественного числа (float) в Python

```c
typedef struct {
    PyObject_HEAD
    double ob_fval;
} PyFloatObject;
```

Второй пункт - это определение объекта типа.

```c
static PyTypeObject CustomType = {
    .ob_base = PyVarObject_HEAD_INIT(NULL, 0)
    .tp_name = "custom.Custom",
    .tp_doc = PyDoc_STR("Custom objects"),
    .tp_basicsize = sizeof(CustomObject),
    .tp_itemsize = 0,
    .tp_flags = Py_TPFLAGS_DEFAULT,
    .tp_new = PyType_GenericNew,
};
```

> **Примечание**: Рекомендуется использовать инициализаторы в стиле C99 (как показано выше), чтобы избежать перечисления всех полей [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject), которые вам не нужны и не зависеть от порядка объявления полей.

Фактическое определение [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject) в `object.h` содержит значительно больше [<u>полей</u>](https://docs.python.org/3/c-api/typeobj.html#type-structs), чем приведённое выше. Оставшиеся поля будут автоматически заполнены нулями компилятором C, и общепринятой практикой является их явное указание только при необходимости.

Разберём эту структуру по полям:

```c
.ob_base = PyVarObject_HEAD_INIT(NULL, 0)
```

Эта строка является обязательным шаблонным кодом для инициализации поля `ob_base`, упомянутого выше.

```c
.tp_name = "custom.Custom",
```

Имя нашего типа. Оно будет использоваться в текстовом представлении объектов по умолчанию и в некоторых сообщениях об ошибках, например:

```python
>>> "" + custom.Custom()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can only concatenate str (not "custom.Custom") to str
```

Обратите внимание, что имя должно быть составным (dotted name) и содержать как имя модуля, так и имя типа внутри модуля. В нашем случае модуль называется <span style="font-family: Consolas, sans-serif;">custom</span>, а тип - <span style="font-family: Consolas, sans-serif;">Custom</span>, поэтому мы задаём имя типа как <span style="font-family: Consolas, sans-serif;">custom.Custom.</span> Использование полного составного пути важно для обеспечения совместимости вашего типа с модулями [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">pydoc</span>](https://docs.python.org/3/library/pydoc.html#module-pydoc) и [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">pickle</span>](https://docs.python.org/3/library/pickle.html#module-pickle).

```c
.tp_basicsize = sizeof(CustomObject),
.tp_itemsize = 0,
```

Это необходимо, чтобы Python знал, сколько памяти выделять при создании новых экземпляров <span style="font-family: Consolas, sans-serif;">Custom</span>. Поле [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_itemsize</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_itemsize) используется только для объектов переменного размера, в остальных случаях должно быть равно нулю.

> **Примечание**: Если вы хотите, чтобы ваш тип можно было наследовать в Python, и при этом он имеет тот же [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_basicsize</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_basicsize), что и базовый тип, могут возникнуть проблемы с множественным наследованием. Python-подкласс вашего типа должен будет указать ваш тип первым в [<span style="font-family: Consolas, sans-serif;">\_\_bases\_\_</span>](https://docs.python.org/3/reference/datamodel.html#type.__bases__), иначе он не сможет вызвать метод [<span style="font-family: Consolas, sans-serif;">\_\_new\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__new__) вашего типа без ошибки. Этой проблемы можно избежать, если убедиться, что значение [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_basicsize</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_basicsize) вашего типа больше, чем у базового типа. В большинстве случаев так и будет, поскольку базовым типом будет [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">object</span>](https://docs.python.org/3/library/functions.html#object) или вы будете добавлять новые поля данных к базовому типу, увеличивая его размер.

Устанавливаем флаги класса в [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_TPFLAGS_DEFAULT</span>](https://docs.python.org/3/c-api/typeobj.html#c.Py_TPFLAGS_DEFAULT).

```c
.tp_flags = Py_TPFLAGS_DEFAULT,
```

Все типы должны включать эту константу в свои флаги. Она активирует все члены, определённые как минимум до Python 3.3. Если требуются дополнительные члены, необходимо использовать оператор OR с соответствующими флагами.

Указываем строку документации для типа в [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_doc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_doc).

```c
.tp_doc = PyDoc_STR("Custom objects"),
```

Чтобы разрешить создание объектов, мы должны предоставить обработчик [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new). Это эквивалент Python-метода [<span style="font-family: Consolas, sans-serif;">\_\_new\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__new__), но требует явного указания. В данном случае мы можем просто использовать стандартную реализацию, предоставляемую API-функцией [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyType_GenericNew()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_GenericNew).

```c
.tp_new = PyType_GenericNew,
```

Всё остальное в файле должно быть вам знакомо, за исключением некоторого кода в <span style="font-family: Consolas, sans-serif;">PyInit_custom()</span>:

```c
if (PyType_Ready(&CustomType) < 0)
    return;
```

Он инициализирует тип <span style="font-family: Consolas, sans-serif;">Custom</span>, заполняя ряд полей значениями по умолчанию, включая поле [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">ob_type</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyObject.ob_type), которое мы изначально установили в `NULL`.

```c
if (PyModule_AddObjectRef(m, "Custom", (PyObject *) &CustomType) < 0) {
    Py_DECREF(m);
    return NULL;
}
```

Это добавит тип в словарь модуля. Теперь мы можем создавать экземпляры <span style="font-family: Consolas, sans-serif;">Custom</span>, вызывая класс <span style="font-family: Consolas, sans-serif;">Custom</span>:

```python
import custom
mycustom = custom.Custom()
```

Вот и всё! Осталось только собрать модуль; поместите приведённый выше код в файл с именем `custom.c`,

```toml
[build-system]
requires = ["setuptools"]
build-backend = "setuptools.build_meta"

[project]
name = "custom"
version = "1"
```

добавьте код выше в файл `pyproject.toml`

```py
from setuptools import Extension, setup
setup(ext_modules=[Extension("custom", ["custom.c"])])
```

и поместите предыдущий код в файл `setup.py`

```sh
python -m pip install .
```

введите команду выше в командной строке — это создаст файл `custom.so` в подкаталоге и установит его. Теперь запустите Python — вы сможете импортировать модуль `custom` и экспериментировать с объектами `Custom`.

Это было легко, не так ли?

Разумеется, текущая реализация типа `Custom` пока довольно бесполезна. Она не содержит данных и ничего не делает. Более того, её даже нельзя наследовать.

## 2.2. Добавление данных и методов к базовому примеру
Давайте расширим базовый пример, добавив данные и методы. Также сделаем тип пригодным для использования в качестве базового класса. Мы создадим новый модуль <span style="font-family: Consolas, sans-serif;">custom2</span> с этими возможностями:


```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>
#include <stddef.h> /* for offsetof() */

typedef struct {
    PyObject_HEAD
    PyObject *first; /* имя */
    PyObject *last;  /* фамилия */
    int number;
} CustomObject;

static void
Custom_dealloc(CustomObject *self)
{
    Py_XDECREF(self->first);
    Py_XDECREF(self->last);
    Py_TYPE(self)->tp_free((PyObject *) self);
}

static PyObject *
Custom_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    CustomObject *self;
    self = (CustomObject *) type->tp_alloc(type, 0);
    if (self != NULL) {
        self->first = PyUnicode_FromString("");
        if (self->first == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->last = PyUnicode_FromString("");
        if (self->last == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->number = 0;
    }
    return (PyObject *) self;
}

static int
Custom_init(CustomObject *self, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"first", "last", "number", NULL};
    PyObject *first = NULL, *last = NULL;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "|OOi", kwlist,
                                     &first, &last,
                                     &self->number))
        return -1;

    if (first) {
        Py_XSETREF(self->first, Py_NewRef(first));
    }
    if (last) {
        Py_XSETREF(self->last, Py_NewRef(last));
    }
    return 0;
}

static PyMemberDef Custom_members[] = {
    {"first", Py_T_OBJECT_EX, offsetof(CustomObject, first), 0,
     "first name"},
    {"last", Py_T_OBJECT_EX, offsetof(CustomObject, last), 0,
     "last name"},
    {"number", Py_T_INT, offsetof(CustomObject, number), 0,
     "custom number"},
    {NULL}  /* Sentinel */
};

static PyObject *
Custom_name(CustomObject *self, PyObject *Py_UNUSED(ignored))
{
    if (self->first == NULL) {
        PyErr_SetString(PyExc_AttributeError, "first");
        return NULL;
    }
    if (self->last == NULL) {
        PyErr_SetString(PyExc_AttributeError, "last");
        return NULL;
    }
    return PyUnicode_FromFormat("%S %S", self->first, self->last);
}

static PyMethodDef Custom_methods[] = {
    {"name", (PyCFunction) Custom_name, METH_NOARGS,
     "Return the name, combining the first and last name"
    },
    {NULL}  /* Sentinel */
};

static PyTypeObject CustomType = {
    .ob_base = PyVarObject_HEAD_INIT(NULL, 0)
    .tp_name = "custom2.Custom",
    .tp_doc = PyDoc_STR("Custom objects"),
    .tp_basicsize = sizeof(CustomObject),
    .tp_itemsize = 0,
    .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
    .tp_new = Custom_new,
    .tp_init = (initproc) Custom_init,
    .tp_dealloc = (destructor) Custom_dealloc,
    .tp_members = Custom_members,
    .tp_methods = Custom_methods,
};

static PyModuleDef custommodule = {
    .m_base =PyModuleDef_HEAD_INIT,
    .m_name = "custom2",
    .m_doc = "Example module that creates an extension type.",
    .m_size = -1,
};

PyMODINIT_FUNC
PyInit_custom2(void)
{
    PyObject *m;
    if (PyType_Ready(&CustomType) < 0)
        return NULL;

    m = PyModule_Create(&custommodule);
    if (m == NULL)
        return NULL;

    if (PyModule_AddObjectRef(m, "Custom", (PyObject *) &CustomType) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Эта версия модуля содержит ряд изменений.

Теперь тип <span style="font-family: Consolas, sans-serif;">Сustom</span> содержит три атрибута данных в своей C-структуре: *first*, *last* и *number*. Переменные *first* и las*t являются строками Python (содержащими имя и фамилию соответственно). Атрибут *number* представляет собой целое число в C.

Структура объекта была обновлена соответствующим образом:

```c
typedef struct {
    PyObject_HEAD
    PyObject *first; /* имя */
    PyObject *last;  /* фамилия */
    int number;
} CustomObject;
```

Поскольку теперь мы работаем с данными, нам нужно более внимательно относиться к выделению и освобождению памяти для объектов. Как минимум, нам потребуется метод для освобождения памяти:

```c
static void
Custom_dealloc(CustomObject *self)
{
    Py_XDECREF(self->first);
    Py_XDECREF(self->last);
    Py_TYPE(self)->tp_free((PyObject *) self);
}
```

который присваивается члену [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc):

```c
.tp_dealloc = (destructor) Custom_dealloc,
```

Этот метод сначала обнуляет счётчики ссылок двух Python-атрибутов. [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XDECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XDECREF) корректно обрабатывает случай, когда аргумент равен `NULL` (что может произойти, если `tp_new` завершился с ошибкой). Затем вызывается член [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_free</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_free) типа объекта (получаемый через `Py_TYPE(self)`) для освобождения памяти. Обратите внимание, что тип объекта может отличаться от <span style="font-family: Consolas, sans-serif;">CustomType</span>, так как объект может быть экземпляром подкласса.

> **Примечание**: Явное приведение к типу `destructor` необходимо, потому что мы определили `Custom_dealloc` как принимающую аргумент `CustomObject *`, тогда как указатель на функцию `tp_dealloc` ожидает `PyObject *`. Без этого приведения компилятор выдаст предупреждение. И это - полиморфизм в объектно-ориентированном стиле, реализованный на C!

Чтобы гарантировать инициализацию имени и фамилии пустыми строками, мы реализуем `tp_new`:

```c
static PyObject *
Custom_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    CustomObject *self;
    self = (CustomObject *) type->tp_alloc(type, 0);
    if (self != NULL) {
        self->first = PyUnicode_FromString("");
        if (self->first == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->last = PyUnicode_FromString("");
        if (self->last == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->number = 0;
    }
    return (PyObject *) self;
}
```

и присвоим его члену [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new):

```c
.tp_new = Custom_new,
```

Обработчик `tp_new` отвечает за создание (но не инициализацию) объектов данного типа. В Python он доступен как метод [<span style="font-family: Consolas, sans-serif;">\_\_new\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__new__). Определение члена `tp_new` не является обязательным - многие типы расширений просто используют [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyType_GenericNew()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_GenericNew), как в первой версии типа <span style="font-family: Consolas, sans-serif;">Custom</span> выше. В нашем случае мы используем обработчик `tp_new` для установки атрибутов `first` и `last` в не-`NULL` значения по умолчанию.

`tp_new` получает тип создаваемого экземпляра (не обязательно `CustomType`, если создаётся подкласс) и аргументы, переданные при вызове типа. Ожидается, что он вернёт созданный экземпляр. Обработчики `tp_new` всегда принимают позиционные и именованные аргументы, но часто их игнорируют, оставляя обработку аргументов методам инициализации (известным как `tp_init` в C или `__init__ `в Python).

> **Примечание**: `tp_new` не должен явно вызывать `tp_init`, так как интерпретатор сделает это автоматически.

Реализация `tp_new` вызывает слот [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_alloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_alloc) для выделения памяти:

```c
self = (CustomObject *) type->tp_alloc(type, 0);
```

Учитывая возможность неудачного выделения памяти, необходимо выполнить проверку результата [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_alloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_alloc) на значение `NULL` перед выполнением последующих операций.

> **Примечание**: Мы не заполняли слот [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_alloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_alloc) вручную. Вместо этого [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyType_Ready()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_Ready) автоматически заполнил его для нас, наследуя от нашего базового класса (по умолчанию - [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">object</span>](https://docs.python.org/3/library/functions.html#object)). Большинство типов используют стратегию выделения памяти по умолчанию.

> **Примечание**: При создании кооперативного [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new) (который вызывает [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new) или [<span style="font-family: Consolas, sans-serif;">\_\_new\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__new__) базового типа), нельзя определять вызываемый метод во время выполнения через порядок разрешения методов (MRO). Всегда явно указывайте тип, чей [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new) следует вызвать, и вызывайте его напрямую или через `type->tp_base->tp_new`. В противном случае Python-подклассы вашего типа, которые также наследуют от других Python-классов, могут работать некорректно (в частности, могут возникать [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">TypeError</span>](https://docs.python.org/3/library/exceptions.html#TypeError) при создании экземпляров таких подклассов).

Определим функцию инициализации, которая принимает аргументы для установки начальных значений нашего экземпляра:

```c
static int
Custom_init(CustomObject *self, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"first", "last", "number", NULL};
    PyObject *first = NULL, *last = NULL, *tmp;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "|OOi", kwlist,
                                     &first, &last,
                                     &self->number))
        return -1;

    if (first) {
        tmp = self->first;
        Py_INCREF(first);
        self->first = first;
        Py_XDECREF(tmp);
    }
    if (last) {
        tmp = self->last;
        Py_INCREF(last);
        self->last = last;
        Py_XDECREF(tmp);
    }
    return 0;
}
```

заполнив [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_init</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_init) слот.

```c
.tp_init = (initproc) Custom_init,
```

Слот [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_init</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_init) доступен в Python как метод [<span style="font-family: Consolas, sans-serif;">\_\_init\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__init__). Он используется для инициализации объекта после его создания. Инициализаторы всегда принимают позиционные и именованные аргументы и должны возвращать `0` при успехе или `-1` при ошибке.

В отличие от обработчика `tp_new`, нет гарантии, что `tp_init` вообще будет вызван (например, модуль [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">pickle</span>](https://docs.python.org/3/library/pickle.html#module-pickle) по умолчанию не вызывает [<span style="font-family: Consolas, sans-serif; ">\_\_init\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__init__) для десериализованных экземпляров). Более того, он может вызываться многократно. Любой код может вызвать метод <span style="font-family: Consolas, sans-serif;">\_\_init\_\_()</span> для наших объектов. Поэтому при присвоении новых значений атрибутам нужно быть особенно внимательными. Например, можно было бы попытаться присвоить значение `first` следующим образом:

```c
if (first) {
    Py_XDECREF(self->first);
    Py_INCREF(first);
    self->first = first;
}
```

Но это было бы рискованно. Наш тип не ограничивает тип атрибута `first`, поэтому им может быть любой объект. У этого объекта может быть деструктор, который выполняет код, пытающийся обратиться к атрибуту `first`. Или же этот деструктор может освободить [<u>Global Interpreter Lock</u>](https://docs.python.org/3/glossary.html#term-GIL) (GIL), позволив произвольному коду в других потоках обращаться к нашему объекту и изменять его.

<a id="footnote-5-back"></a><a id="footnote-6-back"></a>Для полной защиты от подобных ситуаций мы почти всегда должны сначала перезаписывать члены объекта, а уже затем уменьшать их счётчики ссылок. В каких случаях это не требуется?
- Когда мы точно знаем, что счётчик ссылок больше 1;
- Когда мы уверены, что освобождение объекта [<u>[5]</u>](#footnote-5) не приведёт к освобождению [<u>GIL</u>](https://docs.python.org/3/glossary.html#term-GIL) и не вызовет обратных вызовов в код нашего типа;
- При уменьшении счётчика ссылок в обработчике [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc) для типа, который не поддерживает циклический сборщик мусора [<u>[6]</u>](#footnote-6).

Мы хотим предоставить доступ к переменным экземпляра как к атрибутам. Существует несколько способов реализации этого. Наиболее простой подход - определение дескрипторов членов (member definitions):

```c
static PyMemberDef Custom_members[] = {
    {"first", Py_T_OBJECT_EX, offsetof(CustomObject, first), 0,
     "first name"},
    {"last", Py_T_OBJECT_EX, offsetof(CustomObject, last), 0,
     "last name"},
    {"number", Py_T_INT, offsetof(CustomObject, number), 0,
     "custom number"},
    {NULL}  /* Sentinel */
};
```

и поместим определения в слот [<u>tp_members</u>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_members):

```c
.tp_members = Custom_members,
```

Каждое определение члена (member) включает имя члена, тип данных, смещение (offset), флаги доступа и cтроку документации. Подробности см. в разделе [<u>Управление атрибутами общего типа</u>](#331-управление-атрибутами-общего-типа) ниже.

Недостаток этого подхода в том, что он не позволяет ограничить типы объектов, которые могут быть присвоены Python-атрибутам. Мы ожидаем, что first и last будут строками, но можно присвоить любые Python-объекты. Более того, атрибуты можно удалять, что приведёт к установке C-указателей в `NULL`. Хотя мы можем гарантировать инициализацию членов не-`NULL` значениями, они могут стать `NULL` при удалении атрибутов.

Определим метод <span style="font-family: Consolas, sans-serif;">Custom.name()</span>, который возвращает имя объекта в виде объединения (конкатенации) имени и фамилии.

```c
static PyObject *
Custom_name(CustomObject *self, PyObject *Py_UNUSED(ignored))
{
    if (self->first == NULL) {
        PyErr_SetString(PyExc_AttributeError, "first");
        return NULL;
    }
    if (self->last == NULL) {
        PyErr_SetString(PyExc_AttributeError, "last");
        return NULL;
    }
    return PyUnicode_FromFormat("%S %S", self->first, self->last);
}
```

Метод реализован в виде C-функции, которая принимает экземпляр класса <span style="font-family: Consolas, sans-serif;">Custom</span> (или его подкласса) в качестве первого аргумента. Методы всегда принимают экземпляр первым аргументом. Часто методы также принимают позиционные и именованные аргументы, но в данном случае они не требуются, поэтому нет необходимости обрабатывать кортеж позиционных аргументов или словарь именованных аргументов. Этот метод эквивалентен следующему Python-методу:

```python
def name(self):
    return "%s %s" % (self.first, self.last)
```

Обратите внимание, что нам необходимо проверить, что <span style="font-family: Consolas, sans-serif;">first</span> и <span style="font-family: Consolas, sans-serif;">last</span> равны `NULL`. Это необходимо, поскольку их могут удалить, и в таком случае они примут значение `NULL`. Оптимальным решением было бы запретить удаление этих полей и ограничить допустимые значения только строками. Как это реализовать, мы рассмотрим в следующем разделе.

Теперь, когда мы определили метод, нам нужно создать массив определений методов

```c
static PyMethodDef Custom_methods[] = {
    {"name", (PyCFunction) Custom_name, METH_NOARGS,
     "Return the name, combining the first and last name"
    },
    {NULL}  /* Sentinel */
};
```

(обратите внимание, что мы использовали флаг [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">METH_NOARGS</span>](https://docs.python.org/3/c-api/structures.html#c.METH_NOARGS), чтобы указать, что метод не принимает никаких аргументов, кроме *self*)

и присвоим его слоту [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_methods</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_methods):

```c
.tp_methods = Custom_methods,
```

Наконец, сделаем наш тип пригодным для наследования. До этого момента мы аккуратно писали наши методы, чтобы они не делали никаких предположений о типе создаваемого или используемого объекта. Поэтому всё, что нам остаётся сделать — это добавить флаг [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_TPFLAGS_BASETYPE</span>](https://docs.python.org/3/c-api/typeobj.html#c.Py_TPFLAGS_BASETYPE) в определение флагов нашего класса:

```c
.tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
```

Мы переименовываем <span style="font-family: Consolas, sans-serif">PyInit_custom()</span> в <span style="font-family: Consolas, sans-serif">PyInit_custom2()</span>, обновляем название модуля в структуре [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyModuleDef</span>](https://docs.python.org/3/c-api/module.html#c.PyModuleDef) и полное имя класса в структуре [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject).

Теперь обновим наш файл `setup.py`, чтобы включить в него новый модуль,

```c
from setuptools import Extension, setup
setup(ext_modules=[
    Extension("custom", ["custom.c"]),
    Extension("custom2", ["custom2.c"]),
])
```

затем переустановим пакет, чтобы получить возможность импортировать модуль `custom2`:

```sh
python -m pip install .
```

## 2.3. Более точный контроль над атрибутами данных
В этом разделе мы реализуем более точный контроль над установкой атрибутов <span style="font-family: Consolas, sans-serif;">first</span> и <span style="font-family: Consolas, sans-serif;">last</span> в классе Custom. В предыдущей версии нашего модуля переменные экземпляра <span style="font-family: Consolas, sans-serif;">first</span> и <span style="font-family: Consolas, sans-serif;">last</span> могли принимать значения не строкового типа или даже быть удалёнными. Нам необходимо гарантировать, что эти атрибуты всегда содержат строковые значения.

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>
#include <stddef.h> /* for offsetof() */

typedef struct {
    PyObject_HEAD
    PyObject *first; /* имя */
    PyObject *last;  /* фамилия */
    int number;
} CustomObject;

static void
Custom_dealloc(CustomObject *self)
{
    Py_XDECREF(self->first);
    Py_XDECREF(self->last);
    Py_TYPE(self)->tp_free((PyObject *) self);
}

static PyObject *
Custom_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    CustomObject *self;
    self = (CustomObject *) type->tp_alloc(type, 0);
    if (self != NULL) {
        self->first = PyUnicode_FromString("");
        if (self->first == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->last = PyUnicode_FromString("");
        if (self->last == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->number = 0;
    }
    return (PyObject *) self;
}

static int
Custom_init(CustomObject *self, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"first", "last", "number", NULL};
    PyObject *first = NULL, *last = NULL;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "|UUi", kwlist,
                                     &first, &last,
                                     &self->number))
        return -1;

    if (first) {
        Py_SETREF(self->first, Py_NewRef(first));
    }
    if (last) {
        Py_SETREF(self->last, Py_NewRef(last));
    }
    return 0;
}

static PyMemberDef Custom_members[] = {
    {"number", Py_T_INT, offsetof(CustomObject, number), 0,
     "custom number"},
    {NULL}  /* Sentinel */
};

static PyObject *
Custom_getfirst(CustomObject *self, void *closure)
{
    return Py_NewRef(self->first);
}

static int
Custom_setfirst(CustomObject *self, PyObject *value, void *closure)
{
    if (value == NULL) {
        PyErr_SetString(PyExc_TypeError, "Cannot delete the first attribute");
        return -1;
    }
    if (!PyUnicode_Check(value)) {
        PyErr_SetString(PyExc_TypeError,
                        "The first attribute value must be a string");
        return -1;
    }
    Py_SETREF(self->first, Py_NewRef(value));
    return 0;
}

static PyObject *
Custom_getlast(CustomObject *self, void *closure)
{
    return Py_NewRef(self->last);
}

static int
Custom_setlast(CustomObject *self, PyObject *value, void *closure)
{
    if (value == NULL) {
        PyErr_SetString(PyExc_TypeError, "Cannot delete the last attribute");
        return -1;
    }
    if (!PyUnicode_Check(value)) {
        PyErr_SetString(PyExc_TypeError,
                        "The last attribute value must be a string");
        return -1;
    }
    Py_SETREF(self->last, Py_NewRef(value));
    return 0;
}

static PyGetSetDef Custom_getsetters[] = {
    {"first", (getter) Custom_getfirst, (setter) Custom_setfirst,
     "first name", NULL},
    {"last", (getter) Custom_getlast, (setter) Custom_setlast,
     "last name", NULL},
    {NULL}  /* Sentinel */
};

static PyObject *
Custom_name(CustomObject *self, PyObject *Py_UNUSED(ignored))
{
    return PyUnicode_FromFormat("%S %S", self->first, self->last);
}

static PyMethodDef Custom_methods[] = {
    {"name", (PyCFunction) Custom_name, METH_NOARGS,
     "Return the name, combining the first and last name"
    },
    {NULL}  /* Sentinel */
};

static PyTypeObject CustomType = {
    .ob_base = PyVarObject_HEAD_INIT(NULL, 0)
    .tp_name = "custom3.Custom",
    .tp_doc = PyDoc_STR("Custom objects"),
    .tp_basicsize = sizeof(CustomObject),
    .tp_itemsize = 0,
    .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
    .tp_new = Custom_new,
    .tp_init = (initproc) Custom_init,
    .tp_dealloc = (destructor) Custom_dealloc,
    .tp_members = Custom_members,
    .tp_methods = Custom_methods,
    .tp_getset = Custom_getsetters,
};

static PyModuleDef custommodule = {
    .m_base = PyModuleDef_HEAD_INIT,
    .m_name = "custom3",
    .m_doc = "Example module that creates an extension type.",
    .m_size = -1,
};

PyMODINIT_FUNC
PyInit_custom3(void)
{
    PyObject *m;
    if (PyType_Ready(&CustomType) < 0)
        return NULL;

    m = PyModule_Create(&custommodule);
    if (m == NULL)
        return NULL;

    if (PyModule_AddObjectRef(m, "Custom", (PyObject *) &CustomType) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Для обеспечения более строгого контроля над атрибутами <span style="font-family: Consolas, sans-serif;">first</span> и <span style="font-family: Consolas, sans-serif;">last</span> мы реализуем специальные функции - геттеры и сеттеры. Ниже представлены функции для работы с атрибутом <span style="font-family: Consolas, sans-serif;">first</span>:

```c
static PyObject *
Custom_getfirst(CustomObject *self, void *closure)
{
    Py_INCREF(self->first);
    return self->first;
}

static int
Custom_setfirst(CustomObject *self, PyObject *value, void *closure)
{
    PyObject *tmp;
    if (value == NULL) {
        PyErr_SetString(PyExc_TypeError, "Cannot delete the first attribute");
        return -1;
    }
    if (!PyUnicode_Check(value)) {
        PyErr_SetString(PyExc_TypeError,
                        "The first attribute value must be a string");
        return -1;
    }
    tmp = self->first;
    Py_INCREF(value);
    self->first = value;
    Py_DECREF(tmp);
    return 0;
}
```

Функция-геттер принимает два параметра: объект типа <span style="font-family: Consolas, sans-serif;">Custom</span> и "closure" (замыкание) в виде указателя типа void. В нашем случае параметр closure не используется. (Пояснение: closure поддерживает сложные сценарии использования, когда данные определения передаются в геттер и сеттер. Например, это позволяет создать единый набор функций-геттеров и сеттеров, которые определяют, какой атрибут нужно получить или изменить, на основе данных из closure.)

Функция-сеттер принимает объект типа <span style="font-family: Consolas, sans-serif;">Custom</span>, новое значение (может быть `NULL`) и closure (указатель типа void, как в геттере). Если передано значение NULL, это означает запрос на удаление атрибута. В нашей реализации при попытке удаления атрибута или попытке установить значение не строкового типа вызывается ошибка.

Создадим массив структур [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyGetSetDef</span>](https://docs.python.org/3/c-api/structures.html#c.PyGetSetDef):

```c
static PyGetSetDef Custom_getsetters[] = {
    {"first", (getter) Custom_getfirst, (setter) Custom_setfirst,
     "first name", NULL},
    {"last", (getter) Custom_getlast, (setter) Custom_setlast,
     "last name", NULL},
    {NULL}  /* Sentinel */
};
```
и присвоим его слоту [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_getset</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_getset):

```c
.tp_getset = Custom_getsetters,
```

Последним элементом структуры [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">PyGetSetDef</span>](https://docs.python.org/3/c-api/structures.html#c.PyGetSetDef) является упомянутый выше closure. В нашем случае мы не используем closure, поэтому просто передаём `NULL`.

Теперь удалим определения этих атрибутов как членов структуры:

```c
static PyMemberDef Custom_members[] = {
    {"number", Py_T_INT, offsetof(CustomObject, number), 0,
     "custom number"},
    {NULL}  /* Sentinel */
};
```

<a id="footnote-7-back"></a>Также нам необходимо обновить обработчик [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_init</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_init), чтобы он разрешал передачу только строковых значений [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">[7]</span>](#footnote-7):

```c
static int
Custom_init(CustomObject *self, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"first", "last", "number", NULL};
    PyObject *first = NULL, *last = NULL, *tmp;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "|UUi", kwlist,
                                     &first, &last,
                                     &self->number))
        return -1;

    if (first) {
        tmp = self->first;
        Py_INCREF(first);
        self->first = first;
        Py_DECREF(tmp);
    }
    if (last) {
        tmp = self->last;
        Py_INCREF(last);
        self->last = last;
        Py_DECREF(tmp);
    }
    return 0;
}
```

С внесёнными изменениями мы можем гарантировать, что члены `first` и `last` никогда не будут `NULL`, поэтому почти во всех случаях можно удалить проверки на `NULL`. Это означает, что большинство вызовов [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_XDECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XDECREF) можно заменить на [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">Py_DECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF). Единственное место, где мы не можем изменить эти вызовы — реализация `tp_dealloc`, поскольку существует вероятность, что инициализация этих членов завершилась неудачно в `tp_new`.

Также мы переименовываем функцию инициализации модуля и его название в соответствующей функции (как мы делали ранее), а также добавляем новое определение в файл `setup.py`.

## 2.4. Поддержка циклического сборщика мусора
В Python используется циклический сборщик мусора [(<u>cyclic garbage collector (GC)</u>)](https://docs.python.org/3/glossary.html#term-garbage-collection), который способен обнаруживать ненужные объекты, даже если их счётчик ссылок не равен нулю. Это происходит, когда объекты образуют циклические ссылки. Например: 

```python
l = []
l.append(l)
del l
```

В этом примере мы создаём список, который содержит сам себя. Когда мы удаляем его, он всё ещё имеет ссылку на самого себя, поэтому его счётчик ссылок не обнуляется. Однако циклический сборщик мусора (GC) Python в итоге определит, что этот список больше не нужен, и освободит занимаемую списком память.

<a id="footnote-8-back"></a>Во второй версии примера с классом <span style="font-family: Consolas, sans-serif;">Custom</span> мы разрешали хранить объекты любого типа в атрибутах <span style="font-family: Consolas, sans-serif;">first</span> и <span style="font-family: Consolas, sans-serif;">last</span> [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">[8]</span>](#footnote-8). Кроме того, во второй и третьей версиях мы разрешили наследование от <span style="font-family: Consolas, sans-serif;">Custom</span>, и подклассы могут добавлять произвольные атрибуты. По любой из этих двух причин объекты <span style="font-family: Consolas, sans-serif;">Custom</span> могут участвовать в циклах:

```python
import custom3
class Derived(custom3.Custom): pass

n = Derived()
n.some_attribute = n
```
Чтобы объекты типа <span style="font-family: Consolas, sans-serif;">Custom</span>, участвующие в циклических ссылках, могли быть корректно обнаружены и обработаны циклическим сборщиком мусора (GC), необходимо заполнить два дополнительных слота в структуре типа и установить флаг, который активирует эти слоты:

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>
#include <stddef.h> /* for offsetof() */

typedef struct {
    PyObject_HEAD
    PyObject *first; /* first name */
    PyObject *last;  /* last name */
    int number;
} CustomObject;

static int
Custom_traverse(CustomObject *self, visitproc visit, void *arg)
{
    Py_VISIT(self->first);
    Py_VISIT(self->last);
    return 0;
}

static int
Custom_clear(CustomObject *self)
{
    Py_CLEAR(self->first);
    Py_CLEAR(self->last);
    return 0;
}

static void
Custom_dealloc(CustomObject *self)
{
    PyObject_GC_UnTrack(self);
    Custom_clear(self);
    Py_TYPE(self)->tp_free((PyObject *) self);
}

static PyObject *
Custom_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    CustomObject *self;
    self = (CustomObject *) type->tp_alloc(type, 0);
    if (self != NULL) {
        self->first = PyUnicode_FromString("");
        if (self->first == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->last = PyUnicode_FromString("");
        if (self->last == NULL) {
            Py_DECREF(self);
            return NULL;
        }
        self->number = 0;
    }
    return (PyObject *) self;
}

static int
Custom_init(CustomObject *self, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"first", "last", "number", NULL};
    PyObject *first = NULL, *last = NULL;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "|UUi", kwlist,
                                     &first, &last,
                                     &self->number))
        return -1;

    if (first) {
        Py_SETREF(self->first, Py_NewRef(first));
    }
    if (last) {
        Py_SETREF(self->last, Py_NewRef(last));
    }
    return 0;
}

static PyMemberDef Custom_members[] = {
    {"number", Py_T_INT, offsetof(CustomObject, number), 0,
     "custom number"},
    {NULL}  /* Sentinel */
};

static PyObject *
Custom_getfirst(CustomObject *self, void *closure)
{
    return Py_NewRef(self->first);
}

static int
Custom_setfirst(CustomObject *self, PyObject *value, void *closure)
{
    if (value == NULL) {
        PyErr_SetString(PyExc_TypeError, "Cannot delete the first attribute");
        return -1;
    }
    if (!PyUnicode_Check(value)) {
        PyErr_SetString(PyExc_TypeError,
                        "The first attribute value must be a string");
        return -1;
    }
    Py_XSETREF(self->first, Py_NewRef(value));
    return 0;
}

static PyObject *
Custom_getlast(CustomObject *self, void *closure)
{
    return Py_NewRef(self->last);
}

static int
Custom_setlast(CustomObject *self, PyObject *value, void *closure)
{
    if (value == NULL) {
        PyErr_SetString(PyExc_TypeError, "Cannot delete the last attribute");
        return -1;
    }
    if (!PyUnicode_Check(value)) {
        PyErr_SetString(PyExc_TypeError,
                        "The last attribute value must be a string");
        return -1;
    }
    Py_XSETREF(self->last, Py_NewRef(value));
    return 0;
}

static PyGetSetDef Custom_getsetters[] = {
    {"first", (getter) Custom_getfirst, (setter) Custom_setfirst,
     "first name", NULL},
    {"last", (getter) Custom_getlast, (setter) Custom_setlast,
     "last name", NULL},
    {NULL}  /* Sentinel */
};

static PyObject *
Custom_name(CustomObject *self, PyObject *Py_UNUSED(ignored))
{
    return PyUnicode_FromFormat("%S %S", self->first, self->last);
}

static PyMethodDef Custom_methods[] = {
    {"name", (PyCFunction) Custom_name, METH_NOARGS,
     "Return the name, combining the first and last name"
    },
    {NULL}  /* Sentinel */
};

static PyTypeObject CustomType = {
    .ob_base = PyVarObject_HEAD_INIT(NULL, 0)
    .tp_name = "custom4.Custom",
    .tp_doc = PyDoc_STR("Custom objects"),
    .tp_basicsize = sizeof(CustomObject),
    .tp_itemsize = 0,
    .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE | Py_TPFLAGS_HAVE_GC,
    .tp_new = Custom_new,
    .tp_init = (initproc) Custom_init,
    .tp_dealloc = (destructor) Custom_dealloc,
    .tp_traverse = (traverseproc) Custom_traverse,
    .tp_clear = (inquiry) Custom_clear,
    .tp_members = Custom_members,
    .tp_methods = Custom_methods,
    .tp_getset = Custom_getsetters,
};

static PyModuleDef custommodule = {
    .m_base = PyModuleDef_HEAD_INIT,
    .m_name = "custom4",
    .m_doc = "Example module that creates an extension type.",
    .m_size = -1,
};

PyMODINIT_FUNC
PyInit_custom4(void)
{
    PyObject *m;
    if (PyType_Ready(&CustomType) < 0)
        return NULL;

    m = PyModule_Create(&custommodule);
    if (m == NULL)
        return NULL;

    if (PyModule_AddObjectRef(m, "Custom", (PyObject *) &CustomType) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Во-первых, функция обхода сообщает циклическому сборщику мусора (GC) о внутренних объектах, которые могут участвовать в циклах:

```c
static int
Custom_traverse(CustomObject *self, visitproc visit, void *arg)
{
    int vret;
    if (self->first) {
        vret = visit(self->first, arg);
        if (vret != 0)
            return vret;
    }
    if (self->last) {
        vret = visit(self->last, arg);
        if (vret != 0)
            return vret;
    }
    return 0;
}
```

Для каждого подобъекта, который может участвовать в циклах, мы должны вызвать функцию <span style="font-family: Consolas, sans-serif;">visit()</span>, передаваемую в метод обхода. Функция <span style="font-family: Consolas, sans-serif;">visit()</span> принимает в качестве аргументов подобъект и дополнительный аргумент *arg*, переданный в метод обхода. Она возвращает целочисленное значение, которое должно быть возвращено, если оно не равно нулю.

В Python предусмотрен макрос [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_VISIT()</span>](https://docs.python.org/3/c-api/gcsupport.html#c.Py_VISIT), который автоматизирует вызов функций обхода. Использование [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_VISIT()</span>](https://docs.python.org/3/c-api/gcsupport.html#c.Py_VISIT) позволяет сократить шаблонный код в `Custom_traverse`:

```c
static int
Custom_traverse(CustomObject *self, visitproc visit, void *arg)
{
    Py_VISIT(self->first);
    Py_VISIT(self->last);
    return 0;
}
```

> **Примечание**: Реализация [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_traverse</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_traverse) должна использовать точные имена аргументов *visit* и *arg* для корректной работы [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_VISIT()</span>](https://docs.python.org/3/c-api/gcsupport.html#c.Py_VISIT).

Во-вторых, необходимо реализовать метод для очистки подобъектов, которые могут участвовать в циклах:

```c
static int
Custom_clear(CustomObject *self)
{
    Py_CLEAR(self->first);
    Py_CLEAR(self->last);
    return 0;
}
```

Обратите внимание на использование макроса [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_CLEAR()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_CLEAR). Это рекомендуемый и безопасный способ очистки атрибутов данных произвольных типов с одновременным уменьшением их счётчиков ссылок. Если вместо этого вызвать [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_XDECREF()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_XDECREF) для атрибута перед установкой значения `NULL`, существует вероятность, что деструктор атрибута вызовет код, который снова прочитает этот атрибут (особенно при наличии циклических ссылок)

> **Примечание**: Вы можете воспроизвести поведение [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_CLEAR()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_CLEAR), написав:
> ```c
> PyObject *tmp;
> tmp = self->first;
>self->first = NULL;
>Py_XDECREF(tmp);
>```
> Тем не менее, всегда используйте [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_CLEAR()</span>](https://docs.python.org/3/c-api/refcounting.html#c.Py_CLEAR) при удалении атрибутов — это проще и снижает риск ошибок. Не жертвуйте надёжностью ради микро-оптимизаций!

Деструктор `Custom_dealloc` может выполнять произвольный код при очистке атрибутов, что потенциально способно запустить циклический сборщик мусора (GC) внутри этой функции. Поскольку GC предполагает, что счётчик ссылок объекта не равен нулю, необходимо исключить объект из отслеживания GC с помощью вызова [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_GC_UnTrack()</span>](https://docs.python.org/3/c-api/gcsupport.html#c.PyObject_GC_UnTrack) до очистки его членов. Вот переработанная реализация деструктора с использованием [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_GC_UnTrack()</span>](https://docs.python.org/3/c-api/gcsupport.html#c.PyObject_GC_UnTrack) и `Custom_clear`:

```c
static void
Custom_dealloc(CustomObject *self)
{
    PyObject_GC_UnTrack(self);
    Custom_clear(self);
    Py_TYPE(self)->tp_free((PyObject *) self);
}
```

Наконец, добавим флаг [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_TPFLAGS_HAVE_GC</span>](https://docs.python.org/3/c-api/typeobj.html#c.Py_TPFLAGS_HAVE_GC) к флагам класса

```c
.tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE | Py_TPFLAGS_HAVE_GC,
```

Вот и всё. Если бы мы реализовали собственные обработчики [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_alloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_alloc) или [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_free</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_free), их потребовалось бы доработать для поддержки циклического сборщика мусора. Однако большинство расширений используют версии этих методов, предоставляемые Python по умолчанию.

## 2.5. Наследование от других типов
Можно создавать новые типы расширений, унаследованные от существующих типов. Проще всего наследовать встроенные типы, так как расширение может легко использовать необходимую структуру [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject). Однако совместное использование этих структур  [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject) между модулями расширений может быть затруднено.

В этом примере мы создадим тип <span style="font-family: Consolas, sans-serif;">SubList</span>, унаследованный от встроенного типа [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">list</span>](https://docs.python.org/3/library/stdtypes.html#list). Новый тип будет полностью совместим с обычными списками, но получит дополнительный метод <span style="font-family: Consolas, sans-serif;">increment()</span>, увеличивающий внутренний счётчик:

```python
import sublist
s = sublist.SubList(range(3))
s.extend(s)
print(len(s))

print(s.increment())

print(s.increment())
```

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

typedef struct {
    PyListObject list;
    int state;
} SubListObject;

static PyObject *
SubList_increment(SubListObject *self, PyObject *unused)
{
    self->state++;
    return PyLong_FromLong(self->state);
}

static PyMethodDef SubList_methods[] = {
    {"increment", (PyCFunction) SubList_increment, METH_NOARGS,
     PyDoc_STR("increment state counter")},
    {NULL},
};

static int
SubList_init(SubListObject *self, PyObject *args, PyObject *kwds)
{
    if (PyList_Type.tp_init((PyObject *) self, args, kwds) < 0)
        return -1;
    self->state = 0;
    return 0;
}

static PyTypeObject SubListType = {
    PyVarObject_HEAD_INIT(NULL, 0)
    .tp_name = "sublist.SubList",
    .tp_doc = PyDoc_STR("SubList objects"),
    .tp_basicsize = sizeof(SubListObject),
    .tp_itemsize = 0,
    .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
    .tp_init = (initproc) SubList_init,
    .tp_methods = SubList_methods,
};

static PyModuleDef sublistmodule = {
    PyModuleDef_HEAD_INIT,
    .m_name = "sublist",
    .m_doc = "Example module that creates an extension type.",
    .m_size = -1,
};

PyMODINIT_FUNC
PyInit_sublist(void)
{
    PyObject *m;
    SubListType.tp_base = &PyList_Type;
    if (PyType_Ready(&SubListType) < 0)
        return NULL;

    m = PyModule_Create(&sublistmodule);
    if (m == NULL)
        return NULL;

    if (PyModule_AddObjectRef(m, "SubList", (PyObject *) &SubListType) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Как мы видим, исходный код во многом напоминает примеры с классом <span style="font-family: Consolas, sans-serif;">Custom</span> из предыдущих разделов. Далее мы разберём ключевые отличия между этими реализациями.

```c
typedef struct {
    PyListObject list;
    int state;
} SubListObject;
```

Главное отличие для производных типов объектов заключается в том, что структура объекта базового типа должна быть первым элементом. Базовый тип уже включает макрос [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_HEAD()</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject_HEAD) в начале своей структуры.

Когда объект Python является экземпляром <span style="font-family: Consolas, sans-serif;">SubList</span>, указатель `PyObject *` может быть безопасно приведён как к `PyListObject *`, так и к `SubListObject *`:

```c
static int
SubList_init(SubListObject *self, PyObject *args, PyObject *kwds)
{
    if (PyList_Type.tp_init((PyObject *) self, args, kwds) < 0)
        return -1;
    self->state = 0;
    return 0;
}
```

В приведённом примере показано, как вызвать метод [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_init\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__init__) базового типа.

Этот подход особенно важен при создании типа с пользовательскими обработчиками [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new) и [tp_dealloc](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc). Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new) не должен самостоятельно выделять память для объекта через свой [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_alloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_alloc), а должен делегировать эту операцию базовому классу, вызывая его версию [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_new</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_new).

Структура [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject) включает поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_base</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_base) для указания конкретного базового класса. Однако из-за особенностей кросс-платформенной компиляции нельзя напрямую присвоить этому полю ссылку на [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyList_Type</span>](https://docs.python.org/3/c-api/list.html#c.PyList_Type); это следует сделать позже, в функции инициализации модуля:

```c
PyMODINIT_FUNC
PyInit_sublist(void)
{
    PyObject* m;
    SubListType.tp_base = &PyList_Type;
    if (PyType_Ready(&SubListType) < 0)
        return NULL;

    m = PyModule_Create(&sublistmodule);
    if (m == NULL)
        return NULL;

    if (PyModule_AddObjectRef(m, "SubList", (PyObject *) &SubListType) < 0) {
        Py_DECREF(m);
        return NULL;
    }

    return m;
}
```

Перед вызовом [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyType_Ready()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_Ready) необходимо заполнить слот [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_base</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_base) в структуре типа. При наследовании существующего типа нет необходимости заполнять слот [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_alloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_alloc) функцией [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyType_GenericNew()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_GenericNew) — функция аллокации будет унаследована от базового типа.

После этого вызов [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyType_Ready()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_Ready) и добавление объекта типа в модуль выполняются аналогично базовым примерам с <span style="font-family: Consolas, sans-serif;">Custom</span>.

# 3. Определение типов расширений: различные темы
Этот раздел призван дать краткий обзор различных методов типов, которые вы можете реализовать, и объяснить их назначение.

Вот определение [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyTypeObject</span>](https://docs.python.org/3/c-api/type.html#c.PyTypeObject), с опущенными некоторыми полями, которые используются только в [<u>debug-сборках</u>](https://docs.python.org/3/using/configure.html#debug-build):

```c
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* Для вывода в формате "<модуль>.<имя>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* Для выделения памяти */

    /* Методы для стандартных операций */

    destructor tp_dealloc;
    Py_ssize_t tp_vectorcall_offset;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* Ранее известен как tp_compare (Python 2)
                                    или tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Наборы методов для стандартных классов */

    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    /* Дополнительные стандартные операции (здесь для бинарной совместимости) */

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;

    /* Функции для доступа к объекту как к буферу ввода/вывода */
    PyBufferProcs *tp_as_buffer;

    /* Флаги, определяющие наличие дополнительных возможностей */
    unsigned long tp_flags;

    const char *tp_doc; /* Строка документации */

    /* Новое значение в версии 2.0 */
    /* Функция обхода для всех доступных объектов */
    traverseproc tp_traverse;

    /* Удаление ссылок на содержащиеся объекты */
    inquiry tp_clear;

    /* Новое значение в версии 2.1 */
    /* "Богатые" сравнения */
    richcmpfunc tp_richcompare;

    /* Смещение для слабых ссылок */
    Py_ssize_t tp_weaklistoffset;

    /* Итераторы */
    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    /* Дескрипторы атрибутов и механизмы наследования */
    struct PyMethodDef *tp_methods;
    struct PyMemberDef *tp_members;
    struct PyGetSetDef *tp_getset;
    // Сильная ссылка для типа в куче, заимствованная ссылка для статического типа
    struct _typeobject *tp_base;
    PyObject *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    Py_ssize_t tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free; /* Низкоуровневая функция освобождения памяти */
    inquiry tp_is_gc; /* Для PyObject_IS_GC */
    PyObject *tp_bases;
    PyObject *tp_mro; /* порядок разрешения методов */
    PyObject *tp_cache;
    PyObject *tp_subclasses;
    PyObject *tp_weaklist;
    destructor tp_del;

    /* Тег версии кеша атрибутов типа. Добавлено в версии 2.6 */
    unsigned int tp_version_tag;

    destructor tp_finalize;
    vectorcallfunc tp_vectorcall;

    /* Битовая маска наблюдателей за типом */
    unsigned char tp_watched;
} PyTypeObject;
```

Получилось довольно много методов. Но не переживайте — если вам нужно определить тип, скорее всего, вам понадобится реализовать лишь небольшую часть из них.

Как вы, вероятно, уже догадались, мы сейчас разберём это подробнее и предоставим дополнительную информацию о различных обработчиках. Мы не будем придерживаться порядка, в котором они определены в структуре, поскольку на их расположение повлияло множество исторически сложившихся особенностей. Чаще всего проще всего найти пример, содержащий нужные вам поля, а затем изменить их значения под ваш новый тип.


```c
const char *tp_name; /* Для отображения */
```

Название типа — как упоминалось в предыдущей главе, оно будет появляться в различных местах, почти исключительно в диагностических целях. Постарайтесь выбрать что-то, что будет полезным в таких ситуациях!

```c
Py_ssize_t tp_basicsize, tp_itemsize; /* Для выделения памяти */
```

Эти поля указывают среде выполнения, сколько памяти выделять при создании новых объектов данного типа. В Python есть встроенная поддержка структур переменной длины (например, строки, кортежи), для чего и предназначено поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_itemsize</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_itemsize). Мы рассмотрим это подробнее позже.

```c
const char *tp_doc;
```

Здесь вы можете указать строку (или её адрес), которая будет возвращаться при обращении к `obj.__doc__` для получения документационной строки.

Теперь перейдём к основным методам типа — тем, которые реализует большинство расширяемых типов.

## 3.1. Завершение и освобождение памяти
```
destructor tp_dealloc;
```

Эта функция вызывается, когда счётчик ссылок на экземпляр вашего типа достигает нуля и интерпретатор Python хочет его освободить. Если ваш тип требует освобождения памяти или других операций очистки, их следует реализовать здесь. Сам объект также должен быть освобождён в этой функции.Пример такой функции:

```c
static void
newdatatype_dealloc(newdatatypeobject *obj)
{
    free(obj->obj_UnderlyingDatatypePtr);
    Py_TYPE(obj)->tp_free((PyObject *)obj);
}
```

Если ваш тип поддерживает сборку мусора, деструктор должен вызвать [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_GC_UnTrack()</span>](https://docs.python.org/3/c-api/gcsupport.html#c.PyObject_GC_UnTrack) перед очисткой любых полей объекта:

```c
static void
newdatatype_dealloc(newdatatypeobject *obj)
{
    PyObject_GC_UnTrack(obj);
    Py_CLEAR(obj->other_obj);
    ...
    Py_TYPE(obj)->tp_free((PyObject *)obj);
}
```

Важное требование к функции деструктора — она не должна изменять существующие исключения. Это важно, поскольку деструкторы часто вызываются, когда интерпретатор разматывает стек Python. Если стек разматывается из-за исключения (а не обычного возврата), ничто не защищает деструкторы от обработки уже установленного исключения. Любые действия деструктора, которые могут привести к выполнению дополнительного кода Python, могут обнаружить установленное исключение. Это может вызвать misleading ошибки интерпретатора. Правильный способ защиты — сохранить активное исключение перед выполнением небезопасных действий и восстановить его после завершения, используя функции [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyErr_Fetch()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Fetch) и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyErr_Restore()</span>](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Restore).

```c
static void
my_dealloc(PyObject *obj)
{
    MyObject *self = (MyObject *) obj;
    PyObject *cbresult;

    if (self->my_callback != NULL) {
        PyObject *err_type, *err_value, *err_traceback;

        /* Сохраняем текущее состояние исключения */
        PyErr_Fetch(&err_type, &err_value, &err_traceback);

        cbresult = PyObject_CallNoArgs(self->my_callback);
        if (cbresult == NULL)
            PyErr_WriteUnraisable(self->my_callback);
        else
            Py_DECREF(cbresult);

        /* Восстанавливаем сохранённое состояние исключения */
        PyErr_Restore(err_type, err_value, err_traceback);

        Py_DECREF(self->my_callback);
    }
    Py_TYPE(obj)->tp_free((PyObject*)self);
}
```

> **Примечание**: Существуют ограничения на безопасные операции в функции деструктора. Во-первых, если ваш тип поддерживает сборку мусора (используя [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_traverse</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_traverse) и/или [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_clear</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_clear)), некоторые члены объекта могут быть уже очищены или завершены к моменту вызова [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc). Во-вторых, в [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc) объект находится в нестабильном состоянии: его счётчик ссылок равен нулю. Любой вызов нетривиального объекта или API (как в примере выше) может привести к повторному вызову [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc), что вызовет двойное освобождение памяти и аварию. 
> 
> Начиная с Python 3.4, рекомендуется не размещать сложный код завершения в [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc), а вместо этого использовать новый метод типа [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_finalize</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_finalize).

## 3.2. Представление объекта
В Python существует два способа получить текстовое представление объекта: функция [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">repr()</span>](https://docs.python.org/3/library/functions.html#repr) и функция [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">str()</span>](https://docs.python.org/3/library/stdtypes.html#str). (Функция [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">print()</span>](https://docs.python.org/3/library/functions.html#print) просто вызывает [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">str()</span>](https://docs.python.org/3/library/stdtypes.html#str).) Оба обработчика являются необязательными.

```c
reprfunc tp_repr;
reprfunc tp_str;
```

Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_repr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_repr) должен возвращать строковый объект с представлением экземпляра, для которого он вызывается. Простой пример:

```c
static PyObject *
newdatatype_repr(newdatatypeobject *obj)
{
    return PyUnicode_FromFormat("Repr-ified_newdatatype{{size:%d}}",
                                obj->obj_UnderlyingDatatypePtr->size);
}
```

Если обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_repr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_repr) не указан, интерпретатор предоставит представление, использующее [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_name</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_name) типа и уникальный идентификатор объекта.

Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_str</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_str) для [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">str()</span>](https://docs.python.org/3/library/stdtypes.html#str) выполняет ту же роль, что и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_repr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_repr) для [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">repr()</span>](https://docs.python.org/3/library/functions.html#repr) — он вызывается, когда код Python вызывает [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">str()</span>](https://docs.python.org/3/library/stdtypes.html#str) для экземпляра вашего объекта. Его реализация аналогична [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_repr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_repr), но возвращаемая строка предназначена для человека. Если [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_str</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_str)  не задан, вместо него будет использоваться обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_repr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_repr).

Простой пример:

```c
static PyObject *
newdatatype_str(newdatatypeobject *obj)
{
    return PyUnicode_FromFormat("Stringified_newdatatype{{size:%d}}",
                                obj->obj_UnderlyingDatatypePtr->size);
}
```

## 3.3. Управление атрибутами
Для любого объекта, поддерживающего атрибуты, соответствующий тип должен предоставлять функции, которые управляют способом разрешения атрибутов. Необходима функция для получения атрибутов (если они определены) и другая - для установки атрибутов (если их разрешено изменять). Удаление атрибута является особым случаем, при котором обработчику передается значение `NULL`.

Python поддерживает две пары обработчиков атрибутов; тип, поддерживающий атрибуты, должен реализовать функции только для одной из пар. Разница заключается в том, что одна пара принимает имя атрибута как `char*`, а другая — как [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject)*. Каждый тип может использовать ту пару, которая удобнее для реализации.

```c
getattrfunc  tp_getattr;        /* вариант c char* */
setattrfunc  tp_setattr;
/* ... */
getattrofunc tp_getattro;       /* вариант c PyObject* */
setattrofunc tp_setattro;
```

Если доступ к атрибутам объекта всегда является простой операцией (это будет объяснено далее), существуют универсальные реализации, которые можно использовать для предоставления версии управления атрибутами с [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject)*. Необходимость в специализированных обработчиках атрибутов практически полностью исчезла, начиная с Python 2.2, хотя многие примеры ещё не были обновлены для использования новых доступных универсальных механизмов.

### 3.3.1 Управление атрибутами общего типа
Большинство расширяемых типов используют только *простые* атрибуты. Но что делает атрибуты простыми? Есть всего несколько условий, которые должны быть выполнены:
1. Имена атрибутов должны быть известны при вызове [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyType_Ready()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_Ready)
2. Не требуется специальной обработки для отслеживания факта обращения или установки атрибута, а также не нужно выполнять действия на основе его значения.

Обратите внимание, что этот список не накладывает никаких ограничений на значения атрибутов, момент вычисления значений или способ хранения соответствующих данных.

При вызове [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyType_Ready()</span>](https://docs.python.org/3/c-api/type.html#c.PyType_Ready) используются три таблицы, на которые ссылается объект типа, для создания [<u>дескрипторов</u>](https://docs.python.org/3/glossary.html#term-descriptor), помещаемых в словарь объекта типа. Каждый дескриптор управляет доступом к одному атрибуту объекта-экземпляра. Все таблицы являются необязательными; если все они равны `NULL`, экземпляры типа будут иметь только атрибуты, унаследованные от базового типа, и должны оставлять поля [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_getattro</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_getattro) и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_setattro</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_setattro) равными `NULL`, позволяя базовому типу обрабатывать атрибуты.

Таблицы объявляются как три поля объекта типа:

```c
struct PyMethodDef *tp_methods;
struct PyMemberDef *tp_members;
struct PyGetSetDef *tp_getset;
```

Если [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_methods</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_methods) не равен `NULL`, он должен ссылаться на массив структур [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyMethodDef</span>](https://docs.python.org/3/c-api/structures.html#c.PyMethodDef). Каждая запись в таблице представляет собой экземпляр этой структуры:

```c
typedef struct PyMethodDef {
    const char  *ml_name;       /* имя метода */
    PyCFunction  ml_meth;       /* функция реализации */
    int          ml_flags;      /* флаги */
    const char  *ml_doc;        /* строку документации */
} PyMethodDef;
```

Для каждого метода, предоставляемого типом, должна быть определена одна запись; для методов, унаследованных от базового типа, записи не требуются. В конце массива необходима дополнительная запись-ограничитель, отмечающая конец массива. Поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">ml_name</span>](https://docs.python.org/3/c-api/structures.html#c.PyMethodDef.ml_name) ограничителя должно быть равно `NULL`.

```c
typedef struct PyMemberDef {
    const char *name;
    int         type;
    int         offset;
    int         flags;
    const char *doc;
} PyMemberDef;
```

Вторая таблица используется для определения атрибутов, которые напрямую соответствуют данным, хранящимся в экземпляре. Поддерживаются различные примитивные типы C, а доступ может быть только для чтения или для чтения-записи. Структуры в таблице определяются следующим образом:

Для каждой записи в таблице будет создан [<u>дескриптор</u>](https://docs.python.org/3/glossary.html#term-descriptor) и добавлен к типу, который сможет извлекать значение из структуры экземпляра. Поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">type</span>](https://docs.python.org/3/c-api/structures.html#c.PyMemberDef.type) должно содержать код типа, например [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_T_INT</span>](https://docs.python.org/3/c-api/structures.html#c.Py_T_INT) или [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_T_DOUBLE</span>](https://docs.python.org/3/c-api/structures.html#c.Py_T_DOUBLE); это значение будет использоваться для определения способа преобразования между значениями Python и C. Поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">flags</span>](https://docs.python.org/3/c-api/structures.html#c.PyMemberDef.flags) хранит флаги, управляющие доступом к атрибуту: вы можете установить его в [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_READONLY</span>](https://docs.python.org/3/c-api/structures.html#c.Py_READONLY), чтобы запретить изменение атрибута из кода Python.

Использование таблицы [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_members</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_members) для создания дескрипторов, используемых во время выполнения, даёт интересное преимущество: любой атрибут, определённый таким способом, может иметь связанную строку документации, просто указав текст в таблице. Приложение может использовать API интроспекции для получения дескриптора из объекта класса и извлечения строки документации через его атрибут [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_doc\_\_</span>](https://docs.python.org/3/reference/datamodel.html#type.__doc__).

Как и в таблице [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_methods</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_methods), требуется запись-ограничитель со значением [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">ml_name</span>](https://docs.python.org/3/c-api/structures.html#c.PyMethodDef.ml_name) равным `NULL`.

### 3.3.2 Управление атрибутами для конкретного типа
Для простоты здесь будет показана только версия с `char*`; единственное различие между версиями интерфейса с `char*` и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject</span>](https://docs.python.org/3/c-api/structures.html#c.PyObject)* заключается в типе параметра name. Этот пример делает по сути то же самое, что и обобщённый пример выше, но не использует обобщённую поддержку, добавленную в Python 2.2. Он объясняет, как вызываются функции-обработчики, чтобы при необходимости расширения их функциональности было понятно, что нужно сделать.

Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_getattr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_getattr) вызывается, когда объект требует поиска атрибута. Он вызывается в тех же ситуациях, когда вызывался бы метод [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_getattr\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__getattr__) класса.

Пример:

```c
static PyObject *
newdatatype_getattr(newdatatypeobject *obj, char *name)
{
    if (strcmp(name, "data") == 0)
    {
        return PyLong_FromLong(obj->data);
    }

    PyErr_Format(PyExc_AttributeError,
                 "'%.100s' object has no attribute '%.400s'",
                 Py_TYPE(obj)->tp_name, name);
    return NULL;
}
```

Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_setattr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_setattr) вызывается, когда вызывается метод [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_setattr\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__setattr__) или [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_delattr\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__delattr__) экземпляра класса. При удалении атрибута третий параметр будет иметь значение `NULL`. Вот пример, который просто вызывает исключение; если это всё, что вам нужно, обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_setattr</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_setattr) следует установить в `NULL`.

```c
static int
newdatatype_setattr(newdatatypeobject *obj, char *name, PyObject *v)
{
    PyErr_Format(PyExc_RuntimeError, "Read-only attribute: %s", name);
    return -1;
}
```

## 3.4. Сравнение объектов
```c
richcmpfunc tp_richcompare;
```

Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_richcompare</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_richcompare) вызывается при необходимости выполнения ["богатых" сравнений](https://docs.python.org/3/reference/datamodel.html#richcmpfuncs). Он аналогичен методам богатого сравнения, таким как <span style="font-family: Consolas, sans-serif;">\_\_lt\_\_()</span>, а также вызывается функциями [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_RichCompare()</span>](https://docs.python.org/3/c-api/object.html#c.PyObject_RichCompare) и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_RichCompareBool()</span>](https://docs.python.org/3/c-api/object.html#c.PyObject_RichCompareBool).

Функция вызывается с двумя объектами Python и оператором (один из `Py_EQ`, `Py_NE`, `Py_LE`, `Py_GE`, `Py_LT` или `Py_GT`). Она должна сравнить объекты согласно оператору и вернуть `Py_True`/`Py_False` при успешном сравнении, `Py_NotImplemented` если сравнение не реализовано (чтобы попробовать метод другого объекта), или `NULL` при возникновении исключения.

Вот пример реализации для типа данных, который считается равным, если размер внутреннего указателя совпадает:

```c
static PyObject *
newdatatype_richcmp(newdatatypeobject *obj1, newdatatypeobject *obj2, int op)
{
    PyObject *result;
    int c, size1, size2;

    /* код проверки, что оба аргумента имеют тип newdatatype был опущен */

    size1 = obj1->obj_UnderlyingDatatypePtr->size;
    size2 = obj2->obj_UnderlyingDatatypePtr->size;

    switch (op) {
    case Py_LT: c = size1 <  size2; break;
    case Py_LE: c = size1 <= size2; break;
    case Py_EQ: c = size1 == size2; break;
    case Py_NE: c = size1 != size2; break;
    case Py_GT: c = size1 >  size2; break;
    case Py_GE: c = size1 >= size2; break;
    }
    result = c ? Py_True : Py_False;
    Py_INCREF(result);
    return result;
 }
```

## 3.5. Поддержка абстрактных протоколов
Python поддерживает различные абстрактные "протоколы"; конкретные интерфейсы для работы с ними описаны в разделе [<u>Abstract Objects Layer</u>](https://docs.python.org/3/c-api/abstract.html#abstract).

Ряд этих абстрактных интерфейсов был определён на ранних этапах разработки Python. В частности, протоколы чисел, отображений и последовательностей существуют с самого начала. Другие протоколы добавлялись со временем. Для протоколов, зависящих от нескольких обработчиков в реализации типа, старые протоколы определены как опциональные блоки обработчиков, на которые ссылается объект типа. Для новых протоколов есть дополнительные слоты в основном объекте типа, с флагом, указывающим на их наличие (флаг не означает, что слоты не-`NULL` - флаг может быть установлен, но слоты оставаться незаполненными).

```c
PyNumberMethods   *tp_as_number;
PySequenceMethods *tp_as_sequence;
PyMappingMethods  *tp_as_mapping;
```

Если вы хотите, чтобы ваш объект мог вести себя как число, последовательность или отображение, вам нужно указать адрес структуры, реализующей соответствующий C-тип: [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyNumberMethods</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyNumberMethods), [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PySequenceMethods</span>](https://docs.python.org/3/c-api/typeobj.html#c.PySequenceMethods) или [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyMappingMethods</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyMappingMethods). Заполнение этих структур соответствующими значениями - ваша задача. Примеры использования каждой из них можно найти в директории `Objects` исходного кода Python.

```c
hashfunc tp_hash;
```

Эта функция (если вы решите её реализовать) должна возвращать хеш-значение для экземпляра вашего типа данных. Простой пример:

```c
static Py_hash_t
newdatatype_hash(newdatatypeobject *obj)
{
    Py_hash_t result;
    result = obj->some_size + 32767 * obj->some_number;
    if (result == -1)
       result = -2;
    return result;
}
```

[<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_hash_t</span>](https://docs.python.org/3/c-api/hash.html#c.Py_hash_t) — это знаковый целочисленный тип, размер которого зависит от платформы. Возврат значения `-1` из [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_hash</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_hash) указывает на ошибку, поэтому важно избегать его возврата при успешном вычислении хеша, как показано выше.

```c
ternaryfunc tp_call;
```

Эта функция принимает три аргумента:
1. *self* - экземпляр типа данных, для которого осуществляется вызов. Например, при вызове `obj1('hello')`, *self* будет `obj1`.
2. *args* - кортеж с аргументами вызова. Для извлечения аргументов используйте [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple).
3. *kwds* - словарь keyword-аргументов. Если он не `NULL` и вы поддерживаете keyword-аргументы, используйте [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyArg_ParseTupleAndKeywords()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTupleAndKeywords). Если именованные аргументы не поддерживаются, а *kwds* не `NULL`, вызовите [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">TypeError</span>](https://docs.python.org/3/library/exceptions.html#TypeError) с сообщением о неподдержке именованных аргументов.


```c
static PyObject *
newdatatype_call(newdatatypeobject *obj, PyObject *args, PyObject *kwds)
{
    PyObject *result;
    const char *arg1;
    const char *arg2;
    const char *arg3;

    if (!PyArg_ParseTuple(args, "sss:call", &arg1, &arg2, &arg3)) {
        return NULL;
    }
    result = PyUnicode_FromFormat(
        "Returning -- value: [%d] arg1: [%s] arg2: [%s] arg3: [%s]\n",
        obj->obj_UnderlyingDatatypePtr->size,
        arg1, arg2, arg3);
    return result;
}
```

```c
/* Итераторы */
getiterfunc tp_iter;
iternextfunc tp_iternext;
```

Эти функции обеспечивают поддержку протокола итератора. Оба обработчика принимают ровно один параметр - экземпляр, для которого они вызываются, и возвращают новую ссылку. В случае ошибки они должны установить исключение и вернуть `NULL`. [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iter</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iter) соответствует методу Python [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_iter\_\_()</span>](https://docs.python.org/3/reference/datamodel.html#object.__iter__), а [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iternext</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iternext) - методу Python [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">\_\_next\_\_()</span>](https://docs.python.org/3/library/stdtypes.html#iterator.__next__).

Любой [<u>итерируемый</u>](https://docs.python.org/3/glossary.html#term-iterable) объект должен реализовать обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iter</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iter), возвращающий объект-итератор. Здесь действуют те же правила, что и для классов Python:
- Для коллекций (списки, кортежи), поддерживающих несколько независимых итераторов, каждый вызов [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iter</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iter) должен создавать и возвращать новый итератор.
- Объекты с одноразовой итерацией (обычно из-за побочных эффектов, как у файловых объектов) могут реализовать [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iter</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iter), возвращая новую ссылку на себя, и тогда они также должны реализовывать обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iternext</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iternext).

Любой объект-итератор должен реализовывать и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iter</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iter), и [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iternext</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iternext). Обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iter</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iter) итератора должен возвращать новую ссылку на итератор. Его обработчик [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iternext</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iternext) должен возвращать новую ссылку на следующий объект в итерации, если таковой имеется. Если итерация достигла конца, [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iternext</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iternext) может вернуть `NULL` без установки исключения или может установить [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">StopIteration</span>](https://docs.python.org/3/library/exceptions.html#StopIteration) в дополнение к возврату `NULL`; избегание исключения может дать немного лучшую производительность. Если происходит фактическая ошибка, [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_iternext</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_iternext) должен всегда устанавливать исключение и возвращать `NULL`.

## 3.6. Поддержка слабых ссылок
Одной из целей реализации слабых ссылок в Python является возможность участия любых типов в механизме слабых ссылок без накладных расходов для критичных к производительности объектов (таких как числа).

Чтобы объект мог иметь слабые ссылки, тип расширения должен установить бит `Py_TPFLAGS_MANAGED_WEAKREF` в поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_flags</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_flags). Поле [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">tp_weaklistoffset</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_weaklistoffset) следует оставить равным нулю.

Вот как будет выглядеть статически объявленный объект типа

```c
static PyTypeObject TrivialType = {
    PyVarObject_HEAD_INIT(NULL, 0)
    /* ... остальные члены опущены для краткости ... */
    .tp_flags = Py_TPFLAGS_MANAGED_WEAKREF | ...,
};
```

Единственное дополнительное требование — `tp_dealloc` должен очищать все слабые ссылки (вызовом [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_ClearWeakRefs()</span>](https://docs.python.org/3/c-api/weakref.html#c.PyObject_ClearWeakRefs)):

```c
static void
Trivial_dealloc(TrivialObject *self)
{
    /* Сначала очищаем слабые ссылки перед вызовом деструкторов */
    PyObject_ClearWeakRefs((PyObject *) self);
    /* ... остальной код деструкции опущен для краткости ... */
    Py_TYPE(self)->tp_free((PyObject *) self);
}
```

## 3.7. Дополнительные рекомендации
Чтобы узнать, как реализовать конкретный метод для вашего нового типа данных, возьмите исходный код [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">CPython</span>](https://docs.python.org/3/glossary.html#term-CPython). Перейдите в директорию `Objects` и выполните поиск по C-файлам, используя `tp_` плюс название нужной функции (например, `tp_richcompare`). Вы найдёте примеры реализации нужной вам функции.

Когда вам нужно проверить, является ли объект конкретным экземпляром реализуемого вами типа, используйте функцию [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_TypeCheck()</span>](https://docs.python.org/3/c-api/object.html#c.PyObject_TypeCheck). Пример использования:

```c
if (!PyObject_TypeCheck(some_object, &MyType)) {
    PyErr_SetString(PyExc_TypeError, "arg #1 not a mything");
    return NULL;
}
```

# 4. Сборка расширений на C и C++
C-расширение для CPython — это динамически подключаемая библиотека (например, файл `.so` в Linux или `.pyd` в Windows), которая экспортирует *функцию инициализации*.

Чтобы модуль можно было импортировать, динамическая библиотека должна находиться в [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PYTHONPATH</span>](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONPATH) и иметь имя, соответствующее названию модуля, с соответствующим расширением. При использовании setuptools правильное имя файла генерируется автоматически.

Функция инициализации имеет следующую сигнатуру:

<span style="font-family: Consolas, sans-serif;">[PyObject](https://docs.python.org/3/c-api/structures.html#c.PyObject) <b>*PyInit_modulename</b>(void)</span>

Она возвращает либо полностью инициализированный модуль, либо экземпляр [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyModuleDef</span>](https://docs.python.org/3/c-api/module.html#c.PyModuleDef). Подробности см. в разделе [<u>Инициализация C-модулей</u>](https://docs.python.org/3/c-api/module.html#initializing-modules).

Для модулей с именами, содержащими только ASCII-символы, функция должна называться `PyInit_<modulename>`, где `<modulename>` заменяется на имя модуля.При использовании [<u>многофазовой инициализации</u>](https://docs.python.org/3/c-api/module.html#multi-phase-initialization) допустимы имена модулей с не-ASCII символами. В этом случае функция инициализации называется `PyInitU_<modulename>`, где `<modulename>` кодируется с использованием Punycode с заменой дефисов на подчёркивания.

```python
def initfunc_name(name):
    try:
        suffix = b'_' + name.encode('ascii')
    except UnicodeEncodeError:
        suffix = b'U_' + name.encode('punycode').replace(b'-', b'_')
    return b'PyInit' + suffix
```

Можно экспортировать несколько модулей из одной общей библиотеки, определив несколько функций инициализации. Однако их импорт требует использования символических ссылок или пользовательского загрузчика, так как по умолчанию обнаруживается только функция, соответствующая имени файла. Подробности см. в разделе «Несколько модулей в одной библиотеке» в [<b><u>PEP 489</u></b>](https://peps.python.org/pep-0489/).

## 4.1. Сборка расширений на C и C++ с помощью setuptools
Python 3.12 и новее больше не включают distutils. Рекомендуется обратиться к документации setuptools:
<u>https://setuptools.readthedocs.io/en/latest/setuptools.html</u>
для получения информации о сборке и распространении C/C++ расширений с помощью setuptools.

# 5. Сборка расширений на C и C++ в Windows
Эта глава кратко объясняет, как создать модуль расширения для Python под Windows с использованием Microsoft Visual C++, а затем приводит более подробную справочную информацию о принципах его работы. Материал будет полезен как разработчикам под Windows, изучающим создание расширений Python, так и Unix-программистам, заинтересованным в создании кроссплатформенного программного обеспечения, которое можно собрать как под Unix, так и под Windows.

Авторам модулей рекомендуется использовать подход distutils для сборки модулей расширений вместо метода, описанного в этом разделе. Однако вам всё равно потребуется компилятор C, которым был собран Python (обычно Microsoft Visual C++).

> **Примечание**: В этой главе упоминаются имена файлов, содержащих закодированную версию Python. В тексте версия обозначена как `XY`, где: `X` — номер мажорной версии, `Y` — номер минорной версии Python. Например, для Python 2.2.1 значение `XY` будет `22`. 

## 5.1. Пошаговое руководство
Существует два подхода к сборке модулей расширений на Windows, как и в Unix: использовать пакет `setuptools` для управления процессом сборки или делать всё вручную. Подход с setuptools хорошо подходит для большинства расширений; документация по использованию `setuptools` для сборки и упаковки модулей расширений доступна в разделе [<u>Building C and C++ Extensions with setuptools</u>](https://docs.python.org/3/extending/building.html#setuptools-index). Если вы обнаружите, что вам действительно нужно делать всё вручную, может быть полезно изучить файл проекта для модуля [<u>winsound</u>](https://github.com/python/cpython/blob/3.13/PCbuild/winsound.vcxproj) из стандартной библиотеки.

## 5.2. Различия между Unix и Windows
Unix и Windows используют совершенно разные парадигмы для загрузки кода во время выполнения. Прежде чем пытаться собрать модуль, который может быть загружен динамически, следует понять, как работает ваша система.

В Unix файл общей библиотеки (`.so`) содержит код для использования программой, а также имена функций и данных, которые он ожидает найти в программе. При подключении файла к программе все ссылки на эти функции и данные в коде файла изменяются так, чтобы указывать на фактические расположения в памяти, где находятся функции и данные программы. По сути, это операция "линковки".

В Windows динамически подключаемая библиотека (`.dll`) не содержит висячих ссылок. Доступ к функциям или данным осуществляется через таблицу поиска. Таким образом, код DLL не требует исправлений во время выполнения для обращения к памяти программы - вместо этого код использует таблицу поиска DLL, которая модифицируется во время выполнения для указания на функции и данные.

В Unix существует только один тип библиотечных файлов (`.a`), содержащий код из нескольких объектных файлов (`.o`). На этапе "линковки" при создании файла общей библиотеки (`.so`) линкер может обнаружить, что ему неизвестно, где определён идентификатор. Линкер ищет его в объектных файлах внутри библиотек; если находит, то включает весь код из этого объектного файла.

В Windows существуют два типа библиотек: статическая библиотека и библиотека импорта (обе называются `.lib`). Статическая библиотека похожа на Unix-файл `.a`; она содержит код, который включается по мере необходимости. Библиотека импорта в основном используется только для подтверждения линкеру, что определенный идентификатор является допустимым и будет присутствовать в программе при загрузке DLL. Таким образом, линкер использует информацию из библиотеки импорта для построения таблицы поиска идентификаторов, которые не включены в DLL. При связывании приложения или DLL может быть сгенерирована библиотека импорта, которую нужно будет использовать для всех будущих DLL, зависящих от символов в этом приложении или DLL.

Предположим, вы создаете два динамически загружаемых модуля B и C, которые должны использовать общий блок кода A. В Unix вы *не передаете* `A.a` линковщику для `B.so` и `C.so` - это привело бы к двойному включению, из-за чего B и C получили бы свои собственные копии. В Windows при сборке `A.dll` также создается `A.lib`. Вы *передаете* `A.lib` линковщику для B и C. `A.lib` не содержит код; он лишь содержит информацию, которая будет использоваться во время выполнения для доступа к коду A.

В Windows использование библиотеки импорта похоже на `import spam` — оно даёт доступ к именам из spam, но не создаёт отдельную копию. В Unix линковка с библиотекой больше напоминает `from spam import *` — при этом создаётся отдельная копия.

## 5.3. Использование DLL на практике
Python для Windows собирается с помощью Microsoft Visual C++; использование других компиляторов может работать, а может и нет. Остальная часть этого раздела относится исключительно к MSVC++.

При создании DLL в Windows необходимо передать `pythonXY.lib` линковщику. Для сборки двух DLL - spam и ni (которая использует функции из spam), можно использовать следующие команды:

```sh
cl /LD /I/python/include spam.c ../libs/pythonXY.lib
cl /LD /I/python/include ni.c spam.lib ../libs/pythonXY.lib
```

Первая команда создаёт три файла: `spam.obj`, `spam.dll` и `spam.lib`. Файл `spam.dll` не содержит Python-функций (таких как [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyArg_ParseTuple()</span>](https://docs.python.org/3/c-api/arg.html#c.PyArg_ParseTuple)), но он знает, как найти Python-код благодаря `pythonXY.lib`.

Вторая команда создает `ni.dll` (а также `.obj` и `.lib`), который знает, как найти необходимые функции из spam, а также из исполняемого файла Python.

## Встраивание среды выполнения CPython в более крупное приложение
Иногда вместо создания расширения, которое выполняется внутри интерпретатора Python как основное приложение, бывает полезно встроить среду выполнения CPython в более крупное приложение. В этом разделе рассматриваются основные аспекты успешной интеграции CPython в такие приложения.

Не каждый идентификатор экспортируется в таблицу поиска. Если вы хотите, чтобы другие модули (включая Python) могли видеть ваши идентификаторы, вы должны указать `_declspec(dllexport)`, как в `void _declspec(dllexport)`, `initspam(void)` или `PyObject _declspec(dllexport)` `*NiGetSpamData(void)`.

Developer Studio добавит множество библиотек импорта, которые вам на самом деле не нужны, увеличивая размер исполняемого файла примерно на 100 КБ. Чтобы избавиться от них, используйте диалоговое окно (Project Settings Dialog), вкладку Link, чтобы указать *игнорирование стандартных библиотек*. Добавьте правильную `msvcrtxx.lib` в список библиотек.

# 6. Встраивание Python в другое приложение
Предыдущие главы обсуждали, как расширять Python, то есть как добавлять новые возможности в Python, подключая библиотеки функций на языке C. Также можно поступить наоборот: расширить ваше приложение на C/C++, встроив в него Python. Встраивание позволяет реализовать часть функциональности вашего приложения на Python вместо C или C++. Это можно использовать для разных целей; например, пользователи смогут настраивать приложение под свои нужды, написав скрипты на Python. Вы также можете использовать этот подход, если какие-то функции проще реализовать на Python.

Встраивание Python похоже на его расширение, но не совсем. Разница в том, что при расширении Python главной программой приложения остается интерпретатор Python, тогда как при встраивании основная программа может вообще не быть связана с Python — вместо этого отдельные части приложения время от времени вызывают интерпретатор Python для выполнения кода.

Таким образом, при встраивании Python вы предоставляете собственную основную программу. Одна из задач, которую должна выполнять эта программа — инициализация интерпретатора Python. Как минимум, необходимо вызвать функцию [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_Initialize()</span>](https://docs.python.org/3/c-api/init.html#c.Py_Initialize). Также можно передать интерпретатору аргументы командной строки с помощью дополнительных вызовов. После этого вы сможете обращаться к интерпретатору из любой части приложения.

Существует несколько различных способов вызова интерпретатора: вы можете передать строку с инструкциями Python в функцию [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyRun_SimpleString()</span>](https://docs.python.org/3/c-api/veryhigh.html#c.PyRun_SimpleString) или передать файловый указатель stdio и имя файла (только для идентификации в сообщениях об ошибках) в функцию [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyRun_SimpleFile()</span>](https://docs.python.org/3/c-api/veryhigh.html#c.PyRun_SimpleFile). Также можно использовать низкоуровневые операции, описанные в предыдущих главах, для создания и работы с объектами Python.

## 6.1. Встраивание на очень высоком уровне
Самая простая форма встраивания Python — использование интерфейса высокого уровня. Этот интерфейс предназначен для выполнения скрипта Python без необходимости прямого взаимодействия с приложением. Например, его можно использовать для выполнения операций с файлом.

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

int
main(int argc, char *argv[])
{
    PyStatus status;
    PyConfig config;
    PyConfig_InitPythonConfig(&config);

    /* необязательно,  но рекомендуется */
    status = PyConfig_SetBytesString(&config, &config.program_name, argv[0]);
    if (PyStatus_Exception(status)) {
        goto exception;
    }

    status = Py_InitializeFromConfig(&config);
    if (PyStatus_Exception(status)) {
        goto exception;
    }
    PyConfig_Clear(&config);

    PyRun_SimpleString("from time import time,ctime\n"
                       "print('Today is', ctime(time()))\n");
    if (Py_FinalizeEx() < 0) {
        exit(120);
    }
    return 0;

  exception:
     PyConfig_Clear(&config);
     Py_ExitStatusException(status);
}
```

> **Примечание**: `#define PY_SSIZE_T_CLEAN` использовался для указания, что в некоторых API следует применять Py_ssize_t вместо `int`. Начиная с Python 3.13, это больше не требуется, но макрос оставлен для обратной совместимости. Описание этого макроса см. в разделе [<u>Strings and buffers</u>](https://docs.python.org/3/c-api/arg.html#arg-parsing-string-and-buffers).

Установка [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyConfig.program_name</span>](https://docs.python.org/3/c-api/init_config.html#c.PyConfig.program_name) должна выполняться до вызова [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_InitializeFromConfig()</span>](https://docs.python.org/3/c-api/init.html#c.Py_InitializeFromConfig), чтобы сообщить интерпретатору пути к библиотекам Python. Затем интерпретатор инициализируется с помощью [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_Initialize()</span>](https://docs.python.org/3/c-api/init.html#c.Py_Initialize), после чего выполняется встроенный Python-скрипт, выводящий дату и время. Далее вызов [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_FinalizeEx()</span>](https://docs.python.org/3/c-api/init.html#c.Py_FinalizeEx) завершает работу интерпретатора, и программа завершается.В реальной программе Python-скрипт может быть получен из другого источника, например, из текстового редактора, файла или базы данных. Загрузка кода Python из файла удобнее с использованием функции [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyRun_SimpleFile()</span>](https://docs.python.org/3/c-api/veryhigh.html#c.PyRun_SimpleFile), которая избавляет от необходимости выделять память и загружать содержимое файла вручную.

## 6.2. За пределами встраивания на очень высоком уровне: обзор
Интерфейс высокого уровня позволяет выполнять произвольные фрагменты Python-кода из приложения, но обмен данными между ними, мягко говоря, довольно неудобен. Если вам нужен такой обмен, следует использовать низкоуровневые вызовы. Ценой написания большего объёма C-кода можно добиться практически чего угодно.

Следует отметить, что расширение Python и встраивание Python — это, по сути, одно и то же действие, несмотря на разницу в целях. Большинство тем, рассмотренных в предыдущих главах, остаются актуальными. Чтобы проиллюстрировать это, рассмотрим, что на самом деле делает код расширения от Python к C:

1. Преобразует значения данных из Python в C,
2. Выполняет вызов функции в C-коде с использованием преобразованных значений,
3. Конвертирует результаты вызова из C обратно в Python.
   
При встраивании Python интерфейсный код выполняет:

1. Преобразование данных из C в Python,
2. Вызов функции Python-интерфейса с преобразованными значениями,
3. Конвертацию возвращаемых данных из Python обратно в C.
   
Как видите, шаги преобразования данных просто меняются местами, чтобы учесть разное направление передачи между языками. Единственное отличие заключается в вызываемой процедуре между двумя преобразованиями данных. При расширении вы вызываете функцию на C, а при встраивании - функцию Python.

В этой главе не рассматривается преобразование данных между Python и C. Также предполагается, что читатель уже знаком с корректным использованием ссылок и обработкой ошибок. Поскольку эти аспекты не отличаются от расширения интерпретатора, за дополнительной информацией можно обратиться к предыдущим главам.

## 6.3. Чистое встраивание
Первая программа предназначена для выполнения функции в Python-скрипте. Как и в разделе об интерфейсе очень высокого уровня, интерпретатор Python не взаимодействует напрямую с приложением (но это изменится в следующем разделе).

Код для выполнения функции, определённой в Python-скрипте:

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

int
main(int argc, char *argv[])
{
    PyObject *pName, *pModule, *pFunc;
    PyObject *pArgs, *pValue;
    int i;

    if (argc < 3) {
        fprintf(stderr,"Usage: call pythonfile funcname [args]\n");
        return 1;
    }

    Py_Initialize();
    pName = PyUnicode_DecodeFSDefault(argv[1]);
    /* Проверка ошибок pName опущена */

    pModule = PyImport_Import(pName);
    Py_DECREF(pName);

    if (pModule != NULL) {
        pFunc = PyObject_GetAttrString(pModule, argv[2]);
        /* pFunc является новой ссылкой */

        if (pFunc && PyCallable_Check(pFunc)) {
            pArgs = PyTuple_New(argc - 3);
            for (i = 0; i < argc - 3; ++i) {
                pValue = PyLong_FromLong(atoi(argv[i + 3]));
                if (!pValue) {
                    Py_DECREF(pArgs);
                    Py_DECREF(pModule);
                    fprintf(stderr, "Cannot convert argument\n");
                    return 1;
                }
                /* Ссылка pValue "украдена" здесь: */
                PyTuple_SetItem(pArgs, i, pValue);
            }
            pValue = PyObject_CallObject(pFunc, pArgs);
            Py_DECREF(pArgs);
            if (pValue != NULL) {
                printf("Result of call: %ld\n", PyLong_AsLong(pValue));
                Py_DECREF(pValue);
            }
            else {
                Py_DECREF(pFunc);
                Py_DECREF(pModule);
                PyErr_Print();
                fprintf(stderr,"Call failed\n");
                return 1;
            }
        }
        else {
            if (PyErr_Occurred())
                PyErr_Print();
            fprintf(stderr, "Cannot find function \"%s\"\n", argv[2]);
        }
        Py_XDECREF(pFunc);
        Py_DECREF(pModule);
    }
    else {
        PyErr_Print();
        fprintf(stderr, "Failed to load \"%s\"\n", argv[1]);
        return 1;
    }
    if (Py_FinalizeEx() < 0) {
        return 120;
    }
    return 0;
}
```

Этот код загружает Python-скрипт, используя `argv[1]`, и вызывает функцию, указанную в `argv[2]`. Целочисленные аргументы функции берутся из остальных значений массива `argv`. Если [скомпилировать и "слинковать"](https://docs.python.org/3/extending/embedding.html#compiling) эту программу (назовём полученный исполняемый файл, например, **call**), то с её помощью можно выполнять Python-скрипты. Например:

```python
def multiply(a,b):
    print("Will compute", a, "times", b)
    c = 0
    for i in range(0, a):
        c = c + b
    return c
```

тогда результат будет:

```sh
call multiply multiply 3 2
Will compute 3 times 2
Result of call: 6
```

Несмотря на относительно большой объём кода для выполняемой функциональности, бо́льшая его часть отвечает за преобразование данных между Python и C, а также за обработку ошибок. Наиболее интересная часть, касающаяся встраивания Python, начинается с

```c
Py_Initialize();
pName = PyUnicode_DecodeFSDefault(argv[1]);
/* Проверка ошибок pName опущена */
pModule = PyImport_Import(pName);
```

После инициализации интерпретатора скрипт загружается с помощью функции [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyImport_Import()</span>](https://docs.python.org/3/c-api/import.html#c.PyImport_Import). Для этого требуется строка Python в качестве аргумента, которая создаётся с использованием функции преобразования данных [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyUnicode_DecodeFSDefault()</span>](https://docs.python.org/3/c-api/unicode.html#c.PyUnicode_DecodeFSDefault).

```c
pFunc = PyObject_GetAttrString(pModule, argv[2]);
/* pFunc является новой ссылкой */

if (pFunc && PyCallable_Check(pFunc)) {
    ...
}
Py_XDECREF(pFunc);
```

После загрузки скрипта искомое имя можно получить с помощью [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">PyObject_GetAttrString()</span>](https://docs.python.org/3/c-api/object.html#c.PyObject_GetAttrString). Если имя существует и возвращённый объект является вызываемым, можно с уверенностью предположить, что это функция. Затем программа создаёт кортеж аргументов стандартным способом. Вызов Python-функции осуществляется следующим образом:

```c
pValue = PyObject_CallObject(pFunc, pArgs);
```

После выполнения функции, `pValue` будет содержать либо `NULL`, либо ссылку на возвращаемое значение. Важно не забыть освободить эту ссылку после проверки значения.

## 6.4. Расширение встроенного Python
До этого момента встроенный интерпретатор Python не имел доступа к функциональности самого приложения. Python API позволяет это исправить путем расширения встроенного интерпретатора - то есть добавления в него функций, предоставляемых приложением. Хотя это звучит сложно, на практике всё не так страшно. Просто временно забудьте, что приложение запускает интерпретатор Python. Вместо этого представьте, что приложение - это набор подпрограмм, и напишите связующий код, который предоставит Python доступ к этим подпрограммам, точно так же, как вы писали бы обычное расширение для Python. Например:

```c
static int numargs=0;

/* Возвращаем количество аргументов переданных через командную строку */
static PyObject*
emb_numargs(PyObject *self, PyObject *args)
{
    if(!PyArg_ParseTuple(args, ":numargs"))
        return NULL;
    return PyLong_FromLong(numargs);
}

static PyMethodDef EmbMethods[] = {
    {"numargs", emb_numargs, METH_VARARGS,
     "Return the number of arguments received by the process."},
    {NULL, NULL, 0, NULL}
};

static PyModuleDef EmbModule = {
    PyModuleDef_HEAD_INIT, "emb", NULL, -1, EmbMethods,
    NULL, NULL, NULL, NULL
};

static PyObject*
PyInit_emb(void)
{
    return PyModule_Create(&EmbModule);
}
```

Добавьте приведённый выше код непосредственно перед функцией <span style="font-family: Consolas, sans-serif;text-decoration: underline;">main()</span>. Также вставьте следующие две строки перед вызовом [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">Py_Initialize()</span>](https://docs.python.org/3/c-api/init.html#c.Py_Initialize):

```c
numargs = argc;
PyImport_AppendInittab("emb", &PyInit_emb);
```

Эти две строки инициализируют переменную `numargs` и делают функцию <span style="font-family: Consolas, sans-serif;text-decoration: underline;">emb.numargs()</span> доступной для встроенного интерпретатора Python. С такими расширениями Python-скрипт может выполнять такие операции, как:

```python
import emb
print("Number of arguments", emb.numargs())
```

В реальном приложении эти методы будут предоставлять API приложения для Python.

## 6.5. Встраивание Python в C++
Встраивание Python в программу на C++ также возможно. Точный способ реализации зависит от используемой C++ системы. В общем случае вам потребуется написать основную программу на C++ и использовать компилятор C++ для компиляции и "линковки" вашей программы. Не требуется перекомпилировать сам Python с использованием C++.

## 6.6. Компиляция и компоновка в Unix-подобных системах
Не всегда просто подобрать правильные флаги для компилятора (и линковщика), чтобы встроить интерпретатор Python в ваше приложение. Особенно это сложно, потому что Python требуется загружать библиотечные модули, реализованные как C-расширения (`.so` файлы), которые должны быть слинкованы с ним.

Чтобы определить необходимые флаги компилятора и линковщика, вы можете выполнить скрипт `pythonX.Y-config`, который создаётся в процессе установки (также может быть доступен скрипт `python3-config`). Этот скрипт имеет несколько опций, из которых вам непосредственно пригодятся следующие:

- `pythonX.Y-config --cflags` выведет рекомендуемые флаги для компиляции
   
```sh
/opt/bin/python3.11-config --cflags
-I/opt/include/python3.11 -I/opt/include/python3.11 -Wsign-compare  -DNDEBUG -g -fwrapv -O3 -Wall
```

- `pythonX.Y-config --ldflags --embed` выведет рекомендованные флаги для "линковки"

```sh
/opt/bin/python3.11-config --ldflags --embed
-L/opt/lib/python3.11/config-3.11-x86_64-linux-gnu -L/opt/lib -lpython3.11 -lpthread -ldl  -lutil -lm
```

> **Примечание**: Во избежание путаницы между разными установками Python (особенно между системной версией Python и вашей собственной скомпилированной версией), рекомендуется использовать абсолютный путь к `pythonX.Y-config`, как показано в примере выше.

Если эта процедура не сработала (она не гарантированно работает на всех Unix-подобных платформах, однако мы приветствуем [<u>сообщения об ошибках</u>](https://docs.python.org/3/bugs.html#reporting-bugs)), вам придётся изучить документацию вашей системы о динамической компоновке и/или изучить `Makefile` Python (используйте [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">sysconfig.get_makefile_filename()</span>](https://docs.python.org/3/library/sysconfig.html#sysconfig.get_makefile_filename) для определения его местоположения) и параметры компиляции. В этом случае модуль [<span style="font-family: Consolas, sans-serif;text-decoration: underline;">sysconfig</span>](https://docs.python.org/3/library/sysconfig.html#module-sysconfig) является полезным инструментом для программного извлечения конфигурационных значений, которые вы захотите объединить. Например:

```python
import sysconfig
sysconfig.get_config_var('LIBS')

sysconfig.get_config_var('LINKFORSHARED')
```

<span style="font-size: 21px; font-weight: 600;">Сноски</span>

<a id="footnote-1"></a>[[<u>1</u>](#footnote-1-back)] Интерфейс для этой функции уже существует в стандартном модуле [os](https://docs.python.org/3/library/os.html#module-os) — он был выбран в качестве простого и наглядного примера.

<a id="footnote-2"></a>[[<u>2</u>](#footnote-2-back)] Метафора «занять» ссылку не совсем точна: у владельца всё ещё остаётся копия ссылки.

<a id="footnote-3"></a>[[<u>3</u>](#footnote-3-back)] Проверка, что счётчик ссылок хотя бы равен 1, не работает — сам счётчик ссылок может находиться в освобождённой памяти и, следовательно, быть повторно использован для другого объекта!

<a id="footnote-4"></a>[<u>[4</u>](#footnote-4-back)] Эти гарантии не действуют при использовании «старого» стиля вызова функций — который до сих пор встречается во многих существующих кодовых базах.

<a id="footnote-5"></a>[<u>[5</u>](#footnote-5-back)] Это верно, когда мы знаем, что объект имеет базовый тип, например строку или число с плавающей точкой.

<a id="footnote-6"></a>[[<u>6</u>](#footnote-6-back)] Мы полагались на это в обработчике [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">tp_dealloc</span>](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_dealloc) в данном примере, поскольку наш тип не поддерживает сборку мусора.

<a id="footnote-7"></a>[[<u>7</u>](#footnote-7-back)] Теперь мы точно знаем, что члены <span style="font-family: Consolas, sans-serif;">first</span> и <span style="font-family: Consolas, sans-serif;">last</span> содержат строки, поэтому теоретически могли бы менее строго подходить к уменьшению их счетчиков ссылок. Однако мы также допускаем использование подклассов строк. Хотя освобождение обычных строк не приведет к обратным вызовам в наш объект, мы не можем гарантировать, что освобождение экземпляра подкласса строки не вызовет методов нашего класса.

<a id="footnote-8"></a>[[<u>8</u>](#footnote-8-back)] Кроме того, даже если мы ограничили атрибуты экземплярами [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">str</span>](https://docs.python.org/3/library/stdtypes.html#str), пользователь может передавать подклассы строк (произвольные наследники [<span style="font-family: Consolas, sans-serif; text-decoration: underline;">str</span>](https://docs.python.org/3/library/stdtypes.html#str)), что всё ещё позволяет создавать циклические ссылки.