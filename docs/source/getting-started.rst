Установка и запуск
===========================


**Pythons**: Python 3.5, 3.6, 3.7, PyPy3

**Платформы**: Linux и Windows

**Имя пакета PyPI**: `pytest <https://pypi.org/project/pytest/>`_

**Документация в PDF**: `download latest <https://media.readthedocs.org/pdf/pytest/latest/pytest.pdf>`_

``pytest`` - это фреймворк, который позволяет легко создавать как простые, так и расширяемые
тесты. Тесты выразительны и легко читаются — не нужно никаких шаблонов.
Начните работу в считанные минуты с небольшого модульного или сложного функционального
теста для вашего приложения или библиотеки.

.. _`getstarted`:
.. _`installation`:

Установка ``pytest``
----------------------------------------

1. Выполните в командной строке:

.. code-block:: bash

    pip install -U pytest

2. Убедитесь, что установили нужную версию:

.. code-block:: bash

    $ pytest --version
    This is pytest version 5.x.y, imported from $PYTHON_PREFIX/lib/python3.6/site-packages/pytest/__init__.py

.. _`simpletest`:

Создание первого теста
--------------------------

Создайте простой тест всего лишь из четырех строк кода:

.. code-block:: python

    # content of test_sample.py
    def func(x):
        return x + 1


    def test_answer():
        assert func(3) == 5

Готово. Теперь можно запустить тест:

.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item

    test_sample.py F                                                     [100%]

    ================================= FAILURES =================================
    _______________________________ test_answer ________________________________

        def test_answer():
    >       assert func(3) == 5
    E       assert 4 == 5
    E        +  where 4 = func(3)

    test_sample.py:6: AssertionError
    ============================ 1 failed in 0.12s =============================

Поскольку ``func(3)`` возвращает вовсе не ``5``, тест вернул отчет о падении.

.. note::

    Для проверки ожидаемого результата можно использовать простой оператор ``assert``.
    Встроенный в ``pytest`` `подробный анализ assert <http://docs.python.org/reference/simple_stmts.html#the-assert-statement>`_
    распишет вам промежуточные значения выражений ``assert``, так что вам не нужно
    использовать множество имен `унаследованных от JUnit методов <http://docs.python.org/library/unittest.html#test-cases>`_.


Запуск множества тестов
---------------------------

Команда ``pytest`` запускает все файлы с именами в формате ``test_*.py`` или ``\*_test.py``
в текущей папке и ее подпапках.  В целом, тесты ищутся по :ref:`стандартным правилам поиска тестов <test discovery>`.

Проверка брошенного исключения
--------------------------------------------------------------

Используйте :ref:`проверку ожидаемых исключений <assertraises>`, чтобы убедиться, что код
бросает ожидаемое искючение:

.. code-block:: python

    # content of test_sysexit.py
    import pytest


    def f():
        raise SystemExit(1)


    def test_mytest():
        with pytest.raises(SystemExit):
            f()

Запустите этот тест с опцией “quiet” (``-q`` - указывает, что нужно опустить имя файла в отчете):

.. code-block:: pytest

    $ pytest -q test_sysexit.py
    .                                                                    [100%]
    1 passed in 0.12s

Группировка тестовых функций в класс
------------------------------------------

Если тестов много, вы можете захотеть создать тестовый класс. ``pytest`` облегчает
создание классов с множеством тестов:

.. code-block:: python

    # content of test_class.py
    class TestClass:
        def test_one(self):
            x = "this"
            assert "h" in x

        def test_two(self):
            x = "hello"
            assert hasattr(x, "check")

``pytest`` ищет тесты для запуска по :ref:`правилам Python по поиску тестов <test discovery>`,
так что найдет обе функции с префиксом ``test_``. Никаких подклассов создавать не нужно,
просто убедитесь, что имя вашего класса начинается с ``Test``, иначе класс будет пропущен.
Мы можем запустить именно этот модуль, просто указав его имя:

.. code-block:: pytest

    $ pytest -q test_class.py
    .F                                                                   [100%]
    ================================= FAILURES =================================
    ____________________________ TestClass.test_two ____________________________

    self = <test_class.TestClass object at 0xdeadbeef>

        def test_two(self):
            x = "hello"
    >       assert hasattr(x, "check")
    E       AssertionError: assert False
    E        +  where False = hasattr('hello', 'check')

    test_class.py:8: AssertionError
    1 failed, 1 passed in 0.12s

Первый тест пройдет (``pass``), а второй упадет (``fail``). Поскольку ``pytest`` выводит
промежуточные значения ``assert``, вам будет проще понять причину сбоя.

Запрос временного каталога для функциональных тестов
--------------------------------------------------------------

``pytest`` представляет :ref:`встроенные фикстуры <builtin>`
для запроса произвольных ресурсов, например, для создания уникального каталога
временных файлов:

.. code-block:: python

    # content of test_tmpdir.py
    def test_needsfiles(tmpdir):
        print(tmpdir)
        assert 0

Укажите ``tmpdir`` в описании тестовой функции, и ``pytest`` найдет и запустит
соответствующую фикстуру для создания нужного ресурса до вызова самой функции.
В данном случае перед запуском теста ``pytest`` создаст уникальный (для каждого вызова
тестовой фунции) каталог временных файлов:

.. code-block:: pytest

    $ pytest -q test_tmpdir.py
    F                                                                    [100%]
    ================================= FAILURES =================================
    _____________________________ test_needsfiles ______________________________

    tmpdir = local('PYTEST_TMPDIR/test_needsfiles0')

        def test_needsfiles(tmpdir):
            print(tmpdir)
    >       assert 0
    E       assert 0

    test_tmpdir.py:3: AssertionError
    --------------------------- Captured stdout call ---------------------------
    PYTEST_TMPDIR/test_needsfiles0
    1 failed in 0.12s

Подробнее об обработке параметра ``tmpdir`` см.: :ref:`Временные каталоги и файлы <tmpdir handling>`.

Получить список встроенных в ``pytest`` :ref:`фикстур <fixtures>` можно, выполнив команду:

.. code-block:: bash

    pytest --fixtures   # выводит список встроенных и пользовательских фикстур

Обратите внимание, что если не добавить опцию ``-v`` , фикстуры с префиксом ``_``
не попадут в список.

Читайте дальше
-------------------------------------

Посмотрите дополнительные ресурсы по ``pytest``, которые помогут вам
настроить тесты для вашего уникального рабочего процесса:

* ":ref:`cmdline`" - примеры вызова из командной строки
* ":ref:`existingtestsuite`" - работа с уже существующими тестами
* ":ref:`mark`" - маркировка тестов (``pytest.mark``)
* ":ref:`fixtures`"  - обеспечение функциональной основы тестов
* ":ref:`plugins`" - написание плагинов и управление ими
* ":ref:`goodpractices`" - виртуальное окружение и оформление тестов

.. include:: links.inc

