.. _`pythoncollection`:

Изменение стандартных правил поиска тестов Python
==================================================

Игнорирование путей при поиске тестов
----------------------------------------

Можно игнорировать отдельные директории и файлы при сборке тестов,
используя опцию командной строки ``--ignore=path``. ``pytest`` позволяет
указывать несколько опций ``--ignore``. Пример:

.. code-block:: text

    tests/
    |-- example
    |   |-- test_example_01.py
    |   |-- test_example_02.py
    |   '-- test_example_03.py
    |-- foobar
    |   |-- test_foobar_01.py
    |   |-- test_foobar_02.py
    |   '-- test_foobar_03.py
    '-- hello
        '-- world
            |-- test_world_01.py
            |-- test_world_02.py
            '-- test_world_03.py

Теперь, если вызвать ``pytest`` с опциями ``--ignore=tests/foobar/test_foobar_03.py --ignore=tests/hello/``,
то он соберет только те тестовые модули, которые не удовлетворяют указанному шаблону:

.. code-block:: pytest

    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    rootdir: $REGENDOC_TMPDIR, inifile:
    collected 5 items

    tests/example/test_example_01.py .                                   [ 20%]
    tests/example/test_example_02.py .                                   [ 40%]
    tests/example/test_example_03.py .                                   [ 60%]
    tests/foobar/test_foobar_01.py .                                     [ 80%]
    tests/foobar/test_foobar_02.py .                                     [100%]

    ========================= 5 passed in 0.02 seconds =========================

Опция ``--ignore-glob`` позволяет игнорировать пути  в виде шаблонов Unix. К примеру,
если вы хотите исключить тестовые модули, которые заканчиваются на ``_01.py``, можно запустить
``pytest`` с опцией ``--ignore-glob='*_01.py'``.

Отмена подбора теста
----------------------------------------

С помощью опции ``--deselect=item`` можно отменить подбор конкретного теста.
Допустим, ``tests/foobar/test_foobar_01.py`` содержит  ``test_a`` и ``test_b``,
и мы хотим выполнить все тесты из ``tests/`` *кроме* ``tests/foobar/test_foobar_01.py::test_a``.
Это можно сделать, запустив ``pytest`` с опцией ``--deselect tests/foobar/test_foobar_01.py::test_a``.
``pytest`` поддерживает и множественное применение опции.

Повторение путей в командной строке
----------------------------------------------------

По умолчанию ``pytest`` игнорирует задвоенные пути в командной строке:

.. code-block:: pytest

    pytest path_a path_a

    ...
    collected 1 item
    ...

Тесты будут собраны только один раз.

Чтобы собрать тесты дважды, используйте опцию ``--keep-duplicates``.
Пример:

.. code-block:: pytest

    pytest --keep-duplicates path_a path_a

    ...
    collected 2 items
    ...

Из-за особенностей поиска,  если вы 2 раза укажете один и тот же тестовый файл,
``pytest`` подберет его дважды, даже если опция ``--keep-duplicates`` не применялась.
Пример:

.. code-block:: pytest

    pytest test_a.py test_a.py

    ...
    collected 2 items
    ...

Изменение правил рекурсивного обхода
-----------------------------------------------------

Опцию `norecursedirs <https://docs.pytest.org/en/latest/reference.html#confval-norecursedirs>`_
можно задавать в ini-файле, например, в корневом ``pytest.ini`` вашего проекта:

.. code-block:: ini

    # content of pytest.ini
    [pytest]
    norecursedirs = .svn _build tmp*

Такая запись указывает ``pytest`` не проводить рекурсивный поиск тестов
в "build"-директориях "sphinx" и в директориях с префиксом ``tmp``

.. _`change naming conventions`:

Изменение соглашений по именам тестов для поиска
-----------------------------------------------------

Вы можете настроить свои правила для поиска тестов по имени, установив опции
`python_files, python_classes and python_functions <https://docs.pytest.org/en/latest/example/pythoncollection.html>`_.
Вот примеры:

.. code-block:: ini

    # content of pytest.ini
    # Пример 1: pytest  должен искать "check" вместо "test"
    # можно также определить в tox.ini или setup.cfg file, при этом секция
    # имен в setup.cfg files должна быть "tool:pytest"
    [pytest]
    python_files = check_*.py
    python_classes = Check
    python_functions = *_check

Такая настройка заставит ``pytest`` искать файлы по шаблону ``check_*.py``,
классы - по префиксу ``Check``, а функции и методы по шаблону
``*_check``.
Допустим, у нас есть:

.. code-block:: python

    # content of check_myapp.py
    class CheckMyApp:
        def simple_check(self):
            pass

        def complex_check(self):
            pass

Тогда собранные тесты будут выглядеть вот так:

.. code-block:: pytest

    $ pytest --collect-only
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR, inifile: pytest.ini
    collected 2 items
    <Module check_myapp.py>
      <Class CheckMyApp>
          <Function simple_check>
          <Function complex_check>

    ========================== no tests ran in 0.12s ===========================

Можно проверять на соответствие нескольким глобальным шаблонам - в этом случае
при описании шаблоны нужно разделить пробелом:

.. code-block:: ini

    # Пример 2: pytest должен искать файлы с "test" и "example"
    # вставляется в pytest.ini, tox.ini, или setup.cfg  (заменив "pytest"
    # на "tool:pytest" в setup.cfg)
    [pytest]
    python_files = test_*.py example_*.py

.. note::

   параметры ``python_functions`` и ``python_classes`` не оказывают никакого действия
   на поиск ``unittest.TestCase``, поскольку обнаружение таких тестов
   производится средствами ``unittest``.

Аргументы командной строки как имена пакетов
-----------------------------------------------------

Чтобы заставить ``pytest`` интерпретировать аргументы как
имена пакетов, получать их системные пути и запускать тесты,
можно использовать опцию  ``--pyargs``. Например, если
у вас установлен ``unittest2``, можно выполнить

.. code-block:: bash

    pytest --pyargs unittest2.test.test_skipping -q

для запуска соответствующего тестового модуля.
Как и со всеми остальными опциями, можно сделать
ее использование постоянным с помощью
`addopts <https://docs.pytest.org/en/latest/reference.html#confval-addopts>`_
в ini-файле:

.. code-block:: ini

    # content of pytest.ini
    [pytest]
    addopts = --pyargs

После этого простой вызов вида ``pytest NAME`` будет проверять
существование ``NAME`` В качестве пригодного для импорта модуля
и рассматривать его как путь в файловой системе.

Просмотр дерева найденных тестов
-----------------------------------------------

Всегда можно посмоетреть дерево собранных тестов без их запуска:

.. code-block:: pytest

    . $ pytest --collect-only pythoncollection.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR, inifile: pytest.ini
    collected 3 items
    <Module CWD/pythoncollection.py>
      <Function test_function>
      <Class TestClass>
          <Function test_method>
          <Function test_anothermethod>

    ========================== no tests ran in 0.12s ===========================

.. _customizing-test-collection:

Настройка поиска тестов
-------------------------------

Можно легко заставить ``pytest`` искать тесты в любом ``python``-файле:

.. code-block:: ini

    # content of pytest.ini
    [pytest]
    python_files = *.py

Однако, во многих проектах есть файл ``setup.py``, который не хотелось бы
импортировать. Более того, там могут присутствовать файлы, которые можно импортировать
только определенной версией ``Python``. В таких случаях можно динамически определить
игнорируемые файлы, перечислив их в ``conftest.py``:

.. code-block:: python

    # content of conftest.py
    import sys

    collect_ignore = ["setup.py"]
    if sys.version_info[0] > 2:
        collect_ignore.append("pkg/module_py2.py")

Пусть у вас есть такой модуль:

.. code-block:: python

    # content of pkg/module_py2.py
    def test_only_on_python2():
        try:
            assert 0
        except Exception, e:
            pass

и макет файла ``setup.py``:

.. code-block:: python

    # content of setup.py
    0 / 0  # если будет импортирован - вызовет исключение

Тогда при запуске ``pytest`` в интерпретаторе ```Python 2`` мы соберем 1 тест,
а файл``setup.py`` будет проигнорирован:

.. code-block:: pytest

    #$ pytest --collect-only
    ====== test session starts ======
    platform linux2 -- Python 2.7.10, pytest-2.9.1, py-1.4.31, pluggy-0.3.1
    rootdir: $REGENDOC_TMPDIR, inifile: pytest.ini
    collected 1 items
    <Module 'pkg/module_py2.py'>
      <Function 'test_only_on_python2'>

    ====== no tests ran in 0.04 seconds ======

А при запуске на ```Python 3`` будут проигнорированы все файлы:

.. code-block:: pytest

    $ pytest --collect-only
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR, inifile: pytest.ini
    collected 0 items

    ========================== no tests ran in 0.12s ===========================

Для определения файлов, которые должны быть пропущены,
можно также добавлять в  ``collect_ignore_glob`` шаблоны в стиле Unix

В следующем примере в ``conftest.py`` игнорируется файл ``setup.py``
и все файлы, которые оканчиваются на ``*_py2.py`` и запускаются
с помощью ``python`` версии 3 и выше:

.. code-block:: python

    # content of conftest.py
    import sys

    collect_ignore = ["setup.py"]
    if sys.version_info[0] > 2:
        collect_ignore_glob = ["*_py2.py"]
