.. _paramexamples:

Параметризация тестов
=================================================

.. currentmodule:: _pytest.python

``pytest`` позволяет леко параметризовать тестовые функции.
Основы параметризации см. :ref:`parametrize-basics`.

Дальше приводятся некоторые примеры использования встроенного механизма.

Генарация комбинаций параметров, зависящих от опции командной строки
----------------------------------------------------------------------------

Давайте представим, что мы хотим проводить тесты с различными
вычисляемыми параметрами и диапазон параметров должен определяться
параметром командной строки. Для начала напишем ничего не делающий
вычислительный тест:

.. code-block:: python

    # content of test_compute.py


    def test_compute(param1):
        assert param1 < 4

Дальше добавим конфигурирование:

.. code-block:: python

    # content of conftest.py


    def pytest_addoption(parser):
        parser.addoption("--all", action="store_true", help="run all combinations")


    def pytest_generate_tests(metafunc):
        if "param1" in metafunc.fixturenames:
            if metafunc.config.getoption("all"):
                end = 5
            else:
                end = 2
            metafunc.parametrize("param1", range(end))

Это означает, что если мы не используем ``--all``, будет запускаться
2 теста:

.. code-block:: pytest

    $ pytest -q test_compute.py
    ..                                                                   [100%]
    2 passed in 0.12s

Мы произвели всего пару вычислений, поэтому видим 2 точки.
Давайте воспользуемся нашей опцией:

.. code-block:: pytest

    $ pytest -q --all
    ....F                                                                [100%]
    ================================= FAILURES =================================
    _____________________________ test_compute[4] ______________________________

    param1 = 4

        def test_compute(param1):
    >       assert param1 < 4
    E       assert 4 < 4

    test_compute.py:4: AssertionError
    1 failed, 4 passed in 0.12s

Как и ожидалось, при выполнении тестов по всему диапазону значений
``param1``, мы получили ошибку на последнем тесте.

Различные способы определения ID тестов
-----------------------------------------

``pytest`` конструирует строку, которая является идентификатором (ID) теста
для каждого множества значений параметризованного теста. Эти индентификаторы
можно использовать с опцией ``-k``, чтобы отборать для выполнения определенные
тесты, и они же идентифицируют конкретный упавший тест. Запустив
``pytest --collect-only`` , можно посмотреть сгенерированные ID.

У чисел, строк, логических значений и значения None есть свои строковые
представления, которые используются в ID тестов. Для остальных объектов
``pytest`` генерирует ID на основании имен аргументов:

.. code-block:: python

    # content of test_time.py

    import pytest

    from datetime import datetime, timedelta

    testdata = [
        (datetime(2001, 12, 12), datetime(2001, 12, 11), timedelta(1)),
        (datetime(2001, 12, 11), datetime(2001, 12, 12), timedelta(-1)),
    ]


    @pytest.mark.parametrize("a,b,expected", testdata)
    def test_timedistance_v0(a, b, expected):
        diff = a - b
        assert diff == expected


    @pytest.mark.parametrize("a,b,expected", testdata, ids=["forward", "backward"])
    def test_timedistance_v1(a, b, expected):
        diff = a - b
        assert diff == expected


    def idfn(val):
        if isinstance(val, (datetime,)):
            # note this wouldn't show any hours/minutes/seconds
            return val.strftime("%Y%m%d")


    @pytest.mark.parametrize("a,b,expected", testdata, ids=idfn)
    def test_timedistance_v2(a, b, expected):
        diff = a - b
        assert diff == expected


    @pytest.mark.parametrize(
        "a,b,expected",
        [
            pytest.param(
                datetime(2001, 12, 12), datetime(2001, 12, 11), timedelta(1), id="forward"
            ),
            pytest.param(
                datetime(2001, 12, 11), datetime(2001, 12, 12), timedelta(-1), id="backward"
            ),
        ],
    )
    def test_timedistance_v3(a, b, expected):
        diff = a - b
        assert diff == expected

В ``test_timedistance_v0`` мы позволяем ``pytest`` самому генерировать ID.

В ``test_timedistance_v1`` мы определяем идентификаторы, используя
список строк, которые будут использоваться в качестве ID. Они весьма лаконичны,
но могут быть сложны для поддержания.

В ``test_timedistance_v2`` мы определяем ``ids`` с помощью функции,
которая генерирует строковое представление, которое будет частью
идентификатора теста. Здесь обозначения наших ``datetime``-аргументов
генерируются функцией ``idfn``, но из-за того, что мы не можем
формировать представление для объектов ``timedelta``, для них
все еще используется стандартное представление ``pytest``:

.. code-block:: pytest

    $ pytest test_time.py --collect-only
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 8 items
    <Module test_time.py>
      <Function test_timedistance_v0[a0-b0-expected0]>
      <Function test_timedistance_v0[a1-b1-expected1]>
      <Function test_timedistance_v1[forward]>
      <Function test_timedistance_v1[backward]>
      <Function test_timedistance_v2[20011212-20011211-expected0]>
      <Function test_timedistance_v2[20011211-20011212-expected1]>
      <Function test_timedistance_v3[forward]>
      <Function test_timedistance_v3[backward]>

    ========================== no tests ran in 0.12s ===========================

В ``test_timedistance_v3`` мы используем ``pytest.param`` для указания
ID вместе с конкретными данными, вместо того, чтобы перечислять их по отдельности.

Быстрый запуск "testscenarios"
------------------------------------

.. _`test scenarios`: https://pypi.org/project/testscenarios/

Вот быстрый способ запуска тестов, сконфигурированных с помощью `test scenarios`_
(дополнения от Роберта Коллинза для стандартного фреймворка ``unittest``).
Тут придется немного потрудиться с созданием корректных аргументов
для `Metafunc.parametrize <https://docs.pytest.org/en/latest/reference.html#_pytest.python.Metafunc.parametrize>`_:

.. code-block:: python

    # content of test_scenarios.py


    def pytest_generate_tests(metafunc):
        idlist = []
        argvalues = []
        for scenario in metafunc.cls.scenarios:
            idlist.append(scenario[0])
            items = scenario[1].items()
            argnames = [x[0] for x in items]
            argvalues.append([x[1] for x in items])
        metafunc.parametrize(argnames, argvalues, ids=idlist, scope="class")


    scenario1 = ("basic", {"attribute": "value"})
    scenario2 = ("advanced", {"attribute": "value2"})


    class TestSampleWithScenarios:
        scenarios = [scenario1, scenario2]

        def test_demo1(self, attribute):
            assert isinstance(attribute, str)

        def test_demo2(self, attribute):
            assert isinstance(attribute, str)

Это полностью самодостаточный пример, который можно запустить:

.. code-block:: pytest

    $ pytest test_scenarios.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items

    test_scenarios.py ....                                               [100%]

    ============================ 4 passed in 0.12s =============================

Если вы просто соберете тесты, то увидите 'advanced' и 'basic' в качестве
переменных тестовой функции:

.. code-block:: pytest

    $ pytest --collect-only test_scenarios.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items
    <Module test_scenarios.py>
      <Class TestSampleWithScenarios>
          <Function test_demo1[basic]>
          <Function test_demo2[basic]>
          <Function test_demo1[advanced]>
          <Function test_demo2[advanced]>

    ========================== no tests ran in 0.12s ===========================


Отсрочка настройки параметризованных ресурсов
---------------------------------------------------

Параметризация тестовой функции происходит во время сборки тестов.
Хорошей идеей является настройка длительных по времени процессов,
вроде подключения к базе данных, только при запуске такого теста.
Вот простой пример, как можно этого добиться. Этот тест
использует объект фикстуры ``db``:


.. code-block:: python

    # content of test_backends.py

    import pytest


    def test_db_initialized(db):
        # a dummy test
        if db.__class__.__name__ == "DB2":
            pytest.fail("deliberately failing for demo purposes")

Теперь мы добавим конфигурацию тестов, которая генерирует два
вызова функции ``test_db_initialized`` и реализует фабрику,
которая создает объект базы данных для конкретного запуска теста:

.. code-block:: python

    # content of conftest.py
    import pytest


    def pytest_generate_tests(metafunc):
        if "db" in metafunc.fixturenames:
            metafunc.parametrize("db", ["d1", "d2"], indirect=True)


    class DB1:
        "one database object"


    class DB2:
        "alternative database object"


    @pytest.fixture
    def db(request):
        if request.param == "d1":
            return DB1()
        elif request.param == "d2":
            return DB2()
        else:
            raise ValueError("invalid internal test config")

Давайте сначала глянем, как все это выглядит во время сборки:

.. code-block:: pytest

    $ pytest test_backends.py --collect-only
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items
    <Module test_backends.py>
      <Function test_db_initialized[d1]>
      <Function test_db_initialized[d2]>

    ========================== no tests ran in 0.12s ===========================

Потом запустим тесты:

.. code-block:: pytest

    $ pytest -q test_backends.py
    .F                                                                   [100%]
    ================================= FAILURES =================================
    _________________________ test_db_initialized[d2] __________________________

    db = <conftest.DB2 object at 0xdeadbeef>

        def test_db_initialized(db):
            # a dummy test
            if db.__class__.__name__ == "DB2":
    >           pytest.fail("deliberately failing for demo purposes")
    E           Failed: deliberately failing for demo purposes

    test_backends.py:8: Failed
    1 failed, 1 passed in 0.12s

Первый вызов с ``db == "DB1"`` прошел, в то время как второй с
``db == "DB2"`` - упал. Наша фикстура ``db`` создавала каждую из баз данных
на этапе настройки, в то время как ``pytest_generate_tests``
генерировала два соответствующих вызова ``test_db_initialized``
на этапе сборки.

Применение ``indirect`` к отдельному аргументу
---------------------------------------------------

Очень часто при параметризации используется более одного аргумента.
К конкретному аргументу можно применять параметр ``indirect``.
Это можно сделать, передав список или кортеж имен аргументов параметру ``indirect``.
В нижеприведенном примере есть функция ``test_indirect``, которая
использует 2 фикстуры: ``x`` и ``y``.
Здесь мы передаем ``indirect`` список, который
содержит имя фикстуры ``x``, соответственно, он будет применен только к этому аргумента,
и значение ``a`` будет передано соответствующей фикстуре:


.. code-block:: python

    # content of test_indirect_list.py

    import pytest


    @pytest.fixture(scope="function")
    def x(request):
        return request.param * 3


    @pytest.fixture(scope="function")
    def y(request):
        return request.param * 2


    @pytest.mark.parametrize("x, y", [("a", "b")], indirect=["x"])
    def test_indirect(x, y):
        assert x == "aaa"
        assert y == "b"

Тест пройдет успешно:

.. code-block:: pytest

    $ pytest test_indirect_list.py --collect-only
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item
    <Module test_indirect_list.py>
      <Function test_indirect[a-b]>

    ========================== no tests ran in 0.12s ===========================


Обратите внимание, что каждый аргумент в списке ``parametrize`` должен
быть явно объявлен в соответствующей тестовой функции или с помощью ``indirect``.

Параметризация тестовых методов через конфигурацию класса
--------------------------------------------------------------

.. _`unittest parametrizer`: https://github.com/testing-cabal/unittest-ext/blob/master/params.py


Ниже - пример тестовой функции ``pytest_generate_tests``, реализующей
схему параметризации, похожую на `unittest parametrizer`_, но
с более коротким кодом:

.. code-block:: python

    # content of ./test_parametrize.py
    import pytest


    def pytest_generate_tests(metafunc):
        # вызывается один раз для каждой тестовой функции
        funcarglist = metafunc.cls.params[metafunc.function.__name__]
        argnames = sorted(funcarglist[0])
        metafunc.parametrize(
            argnames, [[funcargs[name] for name in argnames] for funcargs in funcarglist]
        )


    class TestClass:
        # карта, определяющая множество аргументов тестовой функции
        params = {
            "test_equals": [dict(a=1, b=2), dict(a=3, b=3)],
            "test_zerodivision": [dict(a=1, b=0)],
        }

        def test_equals(self, a, b):
            assert a == b

        def test_zerodivision(self, a, b):
            with pytest.raises(ZeroDivisionError):
                a / b

Наш генератор тестов отслеживает определение множества аргументов
для каждой тестовой функции на уровне класса. Давайте выполним:


.. code-block:: pytest

    $ pytest -q
    F..                                                                  [100%]
    ================================= FAILURES =================================
    ________________________ TestClass.test_equals[1-2] ________________________

    self = <test_parametrize.TestClass object at 0xdeadbeef>, a = 1, b = 2

        def test_equals(self, a, b):
    >       assert a == b
    E       assert 1 == 2

    test_parametrize.py:21: AssertionError
    1 failed, 2 passed in 0.12s

Непрямая параметризация несколькими фикстурами
--------------------------------------------------------------

Вот реальный (урезанный) пример использования параметризации для
тестированмя кроссплатформенной сериализации объектов различными
интерпретаторами ``Python``. Мы определяем функцию ``test_basic_objects``,
которая должна запускаться с различными множествами трех своих аргументов:

* ``python1``: первый интерпретатор ``python``, запускается для сериализации объекта
* ``python2``: второй интерпретатор ``python``, запускается для десериализации объекта
* ``obj``: объект, который нужно записывать/считывать

.. literalinclude:: multipython.py

Если не все требуемые интерпретаторы установлены, то некоторые
множества аргументов будут пропущены; в противном случае запускаются
все комбинации аргументов (3 первых интерпретатора х
3 вторых интерпретатора х 3 объекта для сериализации/десериализации):

.. code-block:: pytest

   . $ pytest -rs -q multipython.py
   ssssssssssss...ssssssssssss                                          [100%]
   ========================= short test summary info ==========================
   SKIPPED [12] $REGENDOC_TMPDIR/CWD/multipython.py:29: 'python3.5' not found
   SKIPPED [12] $REGENDOC_TMPDIR/CWD/multipython.py:29: 'python3.7' not found
   3 passed, 24 skipped in 0.12s

Непрямая параметризация реализованных опций/импорта
--------------------------------------------------------------------

Если вы хотите сравнить несколько вариантов реализации одного и того же API,
то можете написать тестовые функции, которые получают уже импортированные опции
и пропускаются в случае, если опция недоступна или ее не удалось импортировать.
Пусть у нас есть "базовая" реализация, а остальные (возможно, оптимизированные)
должны обеспечивать такие же результаты:

.. code-block:: python

    # content of conftest.py

    import pytest


    @pytest.fixture(scope="session")
    def basemod(request):
        return pytest.importorskip("base")


    @pytest.fixture(scope="session", params=["opt1", "opt2"])
    def optmod(request):
        return pytest.importorskip(request.param)

Базовое воплощение нашей функции выглядит так:

.. code-block:: python

    # content of base.py
    def func1():
        return 1


А ее оптимизированная версия так:

.. code-block:: python

    # content of opt1.py
    def func1():
        return 1.0001

И наконец, небольшой тестовый модуль:

.. code-block:: python

    # content of test_module.py


    def test_func1(basemod, optmod):
        assert round(basemod.func1(), 3) == round(optmod.func1(), 3)


Если вы запустите его с опцией ``-rs``:

.. code-block:: pytest

    $ pytest -rs test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items

    test_module.py .s                                                    [100%]

    ========================= short test summary info ==========================
    SKIPPED [1] $REGENDOC_TMPDIR/conftest.py:12: could not import 'opt2': No module named 'opt2'
    ======================= 1 passed, 1 skipped in 0.12s =======================

Как видите, у нас нет модуля ``opt2``, и поэтому второй тест ``test_func1``
был пропущен. Несколько замечаний:

- фикстуры в ``conftest.py`` имеют уровень сессии, ибо нам не нужно импортировать
  модуль больше одного раза;

- если у вас есть несколько тестов и пропущенный импорт, счетчик пропущенных
  тестов (``SKIPPED [1]``)  покажет большее число;

- для параметризации тестовых функций можно также
  использовать :ref:`@pytest.mark.parametrize <@pytest.mark.parametrize>`.

Установка маркера или ID для конкретного параметризованного теста
--------------------------------------------------------------------

Чтобы применить маркер или установить ID конкретному тесту, используйте
``pytest.param``. Например:


.. code-block:: python

    # content of test_pytest_param_example.py
    import pytest


    @pytest.mark.parametrize(
        "test_input,expected",
        [
            ("3+5", 8),
            pytest.param("1+7", 8, marks=pytest.mark.basic),
            pytest.param("2+4", 6, marks=pytest.mark.basic, id="basic_2+4"),
            pytest.param(
                "6*9", 42, marks=[pytest.mark.basic, pytest.mark.xfail], id="basic_6*9"
            ),
        ],
    )
    def test_eval(test_input, expected):
        assert eval(test_input) == expected

В примере у нас есть 4 параметризованных теста. Исключая первый тест, мы маркируем
все остальные собственным маркером ``basic``, а для четвертого используем
еще и встроеннй маркер ``xfail``, чтобы отметить, что этот тест должен упасть.
Некоторым тестам мы еще и устанавливаем ID для ясности.

Теперь давайте запустим маркированные ``basic`` тесты в режиме подробной трассировки:

.. code-block:: pytest

    $ pytest -v -m basic
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 17 items / 14 deselected / 3 selected

    test_pytest_param_example.py::test_eval[1+7-8] PASSED                [ 33%]
    test_pytest_param_example.py::test_eval[basic_2+4] PASSED            [ 66%]
    test_pytest_param_example.py::test_eval[basic_6*9] XFAIL             [100%]

    =============== 2 passed, 14 deselected, 1 xfailed in 0.12s ================

В результате:

- были собраны все четыре теста;
- один тест был отброшен, поскольку не имел пометки ``basic``;
- все три теста с маркером ``basic`` были выбраны;
- тест ``test_eval[1+7-8]`` успешно прошел, но его ID был сгенерирован автоматически и мало что объясняет;
- тест ``test_eval[basic_2+4]`` прошел;
- тест ``test_eval[basic_6*9]`` должен был упасть и упал.

.. _`parametrizing_conditional_raising`:

Параметризация с генерацией исключений
--------------------------------------------------------------------

Чтобы написать параметризованные тесты, часть из которых
могла бы генерировать исключения, используйте
`pytest.raises <https://docs.pytest.org/en/latest/reference.html#pytest.raises>`_
с декоратором
`pytest.mark.parametrize <https://docs.pytest.org/en/latest/reference.html#pytest-mark-parametrize-ref>`_.

В дополнение к ``raises`` будет полезно определить простейший
контекстный менеджер ``does_not_raise``, например, так:

.. code-block:: python

    from contextlib import contextmanager
    import pytest


    @contextmanager
    def does_not_raise():
        yield


    @pytest.mark.parametrize(
        "example_input,expectation",
        [
            (3, does_not_raise()),
            (2, does_not_raise()),
            (1, does_not_raise()),
            (0, pytest.raises(ZeroDivisionError)),
        ],
    )
    def test_division(example_input, expectation):
        """Test how much I know division."""
        with expectation:
            assert (6 / example_input) is not None

В вышеприведенном примере первые три тестовых случая должны
проходить в обычном режиме, а вот четвертый - генерировать ``ZeroDivisionError``.

Если планируется поддержка только ``Python 3.7`` и выше, для определения
``does_not_raise`` можно просто использовать ``nullcontext``:

.. code-block:: python

    from contextlib import nullcontext as does_not_raise

Или, если поддерживается ``Python 3.3`` и выше:

.. code-block:: python

    from contextlib import ExitStack as does_not_raise

Или, если хочется, можно запустить  ``pip install contextlib2``
и использовать:

.. code-block:: python

    from contextlib2 import nullcontext as does_not_raise



