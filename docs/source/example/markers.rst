
.. _`mark examples`:

Работа с пользовательской маркировкой
=================================================

Вот несколько примеров использования механизма :ref:`маркировки <mark>`.

.. _`mark run`:

Маркировка тестовых функций и отбор маркированных тестов для запуска
----------------------------------------------------------------------

Тестовую функцию можно настроить с помощью маркировки:

.. code-block:: python

    # content of test_server.py

    import pytest


    @pytest.mark.webtest
    def test_send_http():
        pass  # perform some webtest test for your app


    def test_something_quick():
        pass


    def test_another():
        pass


    class TestClass:
        def test_method(self):
            pass



Теперь вы можете запускать тесты только с меткой ``webtest``:

.. code-block:: pytest

    $ pytest -v -m webtest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 4 items / 3 deselected / 1 selected

    test_server.py::test_send_http PASSED                                [100%]

    ===================== 1 passed, 3 deselected in 0.12s ======================

Можно и наоборот - запустить все тесты, кроме помеченных ``webtest``:

.. code-block:: pytest

    $ pytest -v -m "not webtest"
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 4 items / 1 deselected / 3 selected

    test_server.py::test_something_quick PASSED                          [ 33%]
    test_server.py::test_another PASSED                                  [ 66%]
    test_server.py::TestClass::test_method PASSED                        [100%]

    ===================== 3 passed, 1 deselected in 0.12s ======================

Отбор тестов по идентификатору узла (node ID)
------------------------------------------------

В качестве позиционных аргументов для отбора тестов можно передать
``pytest`` один или несколько идентификаторов узлов (см. ниже).
Это облегчает отбор тестов по именам модулей, классов, методов или функций:

.. code-block:: pytest

    $ pytest -v test_server.py::TestClass::test_method
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 1 item

    test_server.py::TestClass::test_method PASSED                        [100%]

    ============================ 1 passed in 0.12s =============================

Можно выбрать и сам класс:

.. code-block:: pytest

    $ pytest -v test_server.py::TestClass
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 1 item

    test_server.py::TestClass::test_method PASSED                        [100%]

    ============================ 1 passed in 0.12s =============================

Или отобрать сразу несколько узлов:

.. code-block:: pytest

    $ pytest -v test_server.py::TestClass test_server.py::test_send_http
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 2 items

    test_server.py::TestClass::test_method PASSED                        [ 50%]
    test_server.py::test_send_http PASSED                                [100%]

    ============================ 2 passed in 0.12s =============================


.. note::

    Идентификаторы узлов имеют формат ``module.py::class::method``
    или ``module.py::function``. Тесты собираются по идентификатрам узлов,
    так что при передаче ``module.py::class`` будут выбраны все тестовые методы класса.
    Для каждого параметра параметризованной фикстуры или функции
    также создаются узлы, так что идентификатор для отбора конкретного
    параметризованного теста должен включать значение параметра, например,
    ``module.py::function[param]``.

    Идентификаторы узлов упавшего теста отборажаются в сводном отчете,
    если ``pytest`` запущен с опцией ``-rf``. Идентификаторы узлов
    можно определять на основании списка собранных тестов, выводимого
    ``pytest --collectonly``.

Использование опции ``-k "выражение"`` для отбора тестов по именам
----------------------------------------------------------------------

Опцию ``-k`` командной строки можно использовать, чтобы указать подстроку,
которая должна присутствовать в именах тестов (при использовании опции ``-m``
проверяется точное совпадение). Это облегчает отбор тестов по именам.

При этом сопоставление строк производится без учета регистра.
Запустим с модулем из примера выше:

.. code-block:: pytest

    $ pytest -v -k http  # running with the above defined example module
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 4 items / 3 deselected / 1 selected

    test_server.py::test_send_http PASSED                                [100%]

    ===================== 1 passed, 3 deselected in 0.12s ======================

Можно также запустить все тесты, которые не содержат ключевого слова:

.. code-block:: pytest

    $ pytest -k "not send_http" -v
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 4 items / 1 deselected / 3 selected

    test_server.py::test_something_quick PASSED                          [ 33%]
    test_server.py::test_another PASSED                                  [ 66%]
    test_server.py::TestClass::test_method PASSED                        [100%]

    ===================== 3 passed, 1 deselected in 0.12s ======================

Или отборать все тесты, в именах которых есть подстрока "http" или "quick":

.. code-block:: pytest

    $ pytest -k "http or quick" -v
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 4 items / 2 deselected / 2 selected

    test_server.py::test_send_http PASSED                                [ 50%]
    test_server.py::test_something_quick PASSED                          [100%]

    ===================== 2 passed, 2 deselected in 0.12s ======================

.. note::

    Если вы используете выражение вида ``"X and Y"``, то и ``X``, и ``Y``
    должны быть простыми именами без ключевых слов. К примеру, запуск с
    переданными ``"pass"`` или ``"from"`` приведет к синтаксической ошибке,
    поскольку ``"-k"`` вычисляет выражения с помощью
    ``Python``-функции `eval`_ .

.. _`eval`: https://docs.python.org/3.6/library/functions.html#eval

    Однако, если аргумент ``-k`` является просто строкой, то таких ограничений нет,
    так же как и в случае с ``"-k 'не строка'"``. Можно указывать и числа, например,
    ``"-k 1.3"``, если ваши тесты параметризованы действительным числом ``"1.3"``.

Регистрация маркеров
-------------------------------------

.. ini-syntax for custom markers:

Зарегистрировать свои маркеры несложно:

.. code-block:: ini

    # content of pytest.ini
    [pytest]
    markers =
        webtest: mark a test as a webtest.

Можно запросить список маркеров для тестового набора и увидеть, что в списке появился только что
зарегистрированный маркер ``webtest``:

.. code-block:: pytest

    $ pytest --markers
    @pytest.mark.webtest: mark a test as a webtest.

    @pytest.mark.filterwarnings(warning): add a warning filter to the given test. see https://docs.pytest.org/en/latest/warnings.html#pytest-mark-filterwarnings

    @pytest.mark.skip(reason=None): skip the given test function with an optional reason. Example: skip(reason="no way of currently testing this") skips the test.

    @pytest.mark.skipif(condition): skip the given test function if eval(condition) results in a True value.  Evaluation happens within the module global context. Example: skipif('sys.platform == "win32"') skips the test if we are on the win32 platform. see https://docs.pytest.org/en/latest/skipping.html

    @pytest.mark.xfail(condition, reason=None, run=True, raises=None, strict=False): mark the test function as an expected failure if eval(condition) has a True value. Optionally specify a reason for better reporting and run=False if you don't even want to execute the test function. If only specific exception(s) are expected, you can list them in raises, and if the test fails in other ways, it will be reported as a true failure. See https://docs.pytest.org/en/latest/skipping.html

    @pytest.mark.parametrize(argnames, argvalues): call a test function multiple times passing in different arguments in turn. argvalues generally needs to be a list of values if argnames specifies only one name or a list of tuples of values if argnames specifies multiple names. Example: @parametrize('arg1', [1,2]) would lead to two calls of the decorated test function, one with arg1=1 and another with arg1=2.see https://docs.pytest.org/en/latest/parametrize.html for more info and examples.

    @pytest.mark.usefixtures(fixturename1, fixturename2, ...): mark tests as needing all of the specified fixtures. see https://docs.pytest.org/en/latest/fixture.html#usefixtures

    @pytest.mark.tryfirst: mark a hook implementation function such that the plugin machinery will try to call it first/as early as possible.

    @pytest.mark.trylast: mark a hook implementation function such that the plugin machinery will try to call it last/as late as possible.


Пример добавления маркеров из плагина и работы с ними
см. ниже :ref:`adding a custom marker from a plugin`.

.. note::

    Мы рекомендуем явно регистрировать маркеры так, чтобы:

    * Ваши маркеры определялись только в одном месте тестового набора

    * Получение списка маркеров с помощью ``pytest --markers`` давало правильный результат

    * Опечатки в маркерах рассматривались как ошибка при использовании опции ``--strict-markers``.


.. _`scoped-marking`:

Маркировка классов и модулей
----------------------------------------------------

Декоратор ``pytest.mark`` можно применять для классов, чтобы пометить все его тестовые методы:

.. code-block:: python

    # content of test_mark_classlevel.py
    import pytest


    @pytest.mark.webtest
    class TestClass:
        def test_startup(self):
            pass

        def test_startup_and_more(self):
            pass

Такая запись равносильна применению декоратора к обеим тестовым функциям.

Можно также установить атрибут ``pytestmark`` тестовому классу ``TestClass`` таким образом:

.. code-block:: python

    import pytest


    class TestClass:
        pytestmark = pytest.mark.webtest

или назначить список маркеров:

.. code-block:: python

    import pytest


    class TestClass:
        pytestmark = [pytest.mark.webtest, pytest.mark.slowtest]

Можно также установить пометку на уровне модуля:

    import pytest
    pytestmark = pytest.mark.webtest

равно как и список маркеров:

    pytestmark = [pytest.mark.webtest, pytest.mark.slowtest]

В этом случае маркеры будут применяться (слева направо) ко всем функциям
и методам модуля.

.. _`marking individual tests when using parametrize`:

Маркировка отдельных тестов при использовании параметризации
----------------------------------------------------------------

Если тест параметризован, то маркировка такого теста
равносильна маркировке каждого экземпляра теста с конкретным параметром.
Тем не менее, можно пометить и отдельный экземпляр параметризованного теста:

.. code-block:: python

    import pytest


    @pytest.mark.foo
    @pytest.mark.parametrize(
        ("n", "expected"), [(1, 2), pytest.param(1, 3, marks=pytest.mark.bar), (2, 3)]
    )
    def test_increment(n, expected):
        assert n + 1 == expected

В приведенном выше примере маркером "foo" окажутся помечены
все три запускаемых теста, а вот маркер "bar" будет применен только ко второму.
Тем же способом можно пометить ``skip`` и ``xfail`` тесты,
см. :ref:`skip/xfail with parametrize`.

.. _`adding a custom marker from a plugin`:

Настраиваемые маркеры и опции командной строки для контроля запуска тестов
----------------------------------------------------------------------------

.. regendoc:wipe

Плагины могут предоставлять настраиваемые маркеры и реализовывать
определенное поведение на их основе. Вот полноценный пример
добавления опции командной строки и параметризованного маркера тестовой
функции для запуска тестов в определенных виртуальных средах:


.. code-block:: python

    # content of conftest.py

    import pytest


    def pytest_addoption(parser):
        parser.addoption(
            "-E",
            action="store",
            metavar="NAME",
            help="only run tests matching the environment NAME.",
        )


    def pytest_configure(config):
        # register an additional marker
        config.addinivalue_line(
            "markers", "env(name): mark test to run only on named environment"
        )


    def pytest_runtest_setup(item):
        envnames = [mark.args[0] for mark in item.iter_markers(name="env")]
        if envnames:
            if item.config.getoption("-E") not in envnames:
                pytest.skip("test requires env in {!r}".format(envnames))

Вот тестовый файл с использованием этого плагина:

.. code-block:: python

    # content of test_someenv.py

    import pytest


    @pytest.mark.env("stage1")
    def test_basic_db_operation():
        pass

и пример запуска теста в виртуальной среде, отличной от "stage1":

.. code-block:: pytest

    $ pytest -E stage2
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item

    test_someenv.py s                                                    [100%]

    ============================ 1 skipped in 0.12s ============================

А здесь мы запускаем тест в нужном виртуальном окружении:

.. code-block:: pytest

    $ pytest -E stage1
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item

    test_someenv.py .                                                    [100%]

    ============================ 1 passed in 0.12s =============================

Опцию ``--markers`` всегда можно использовать для получения актуального списка доступных маркеров:

.. code-block:: pytest

    $ pytest --markers
    @pytest.mark.env(name): mark test to run only on named environment

    @pytest.mark.filterwarnings(warning): add a warning filter to the given test. see https://docs.pytest.org/en/latest/warnings.html#pytest-mark-filterwarnings

    @pytest.mark.skip(reason=None): skip the given test function with an optional reason. Example: skip(reason="no way of currently testing this") skips the test.

    @pytest.mark.skipif(condition): skip the given test function if eval(condition) results in a True value.  Evaluation happens within the module global context. Example: skipif('sys.platform == "win32"') skips the test if we are on the win32 platform. see https://docs.pytest.org/en/latest/skipping.html

    @pytest.mark.xfail(condition, reason=None, run=True, raises=None, strict=False): mark the test function as an expected failure if eval(condition) has a True value. Optionally specify a reason for better reporting and run=False if you don't even want to execute the test function. If only specific exception(s) are expected, you can list them in raises, and if the test fails in other ways, it will be reported as a true failure. See https://docs.pytest.org/en/latest/skipping.html

    @pytest.mark.parametrize(argnames, argvalues): call a test function multiple times passing in different arguments in turn. argvalues generally needs to be a list of values if argnames specifies only one name or a list of tuples of values if argnames specifies multiple names. Example: @parametrize('arg1', [1,2]) would lead to two calls of the decorated test function, one with arg1=1 and another with arg1=2.see https://docs.pytest.org/en/latest/parametrize.html for more info and examples.

    @pytest.mark.usefixtures(fixturename1, fixturename2, ...): mark tests as needing all of the specified fixtures. see https://docs.pytest.org/en/latest/fixture.html#usefixtures

    @pytest.mark.tryfirst: mark a hook implementation function such that the plugin machinery will try to call it first/as early as possible.

    @pytest.mark.trylast: mark a hook implementation function such that the plugin machinery will try to call it last/as late as possible.


.. _`passing callables to custom markers`:

Передача callable-объекта настраиваемым маркерам
------------------------------------------------------------------

.. regendoc:wipe

Вот конфигурационный файл, который будет использоваться в последующих примерах:

.. code-block:: python

    # content of conftest.py
    import sys


    def pytest_runtest_setup(item):
        for marker in item.iter_markers(name="my_marker"):
            print(marker)
            sys.stdout.flush()

Настраиваемый маркер может иметь свое множество позиционных и именованных аргументов, т. е. свойств
``args`` и ``kwargs``, которые можно передать как с помощью вызова callable-объекта, так и спомощью
``pytest.mark.MARKER_NAME.with_args``.
В большинстве случаев оба метода работают одинаково.

Однако, если единственным позиционным аргументом является callable-объект
без именованных аргументов, использование ``pytest.mark.MARKER_NAME(c)`` не передаст
``"c"`` в качестве позиционного аргумента, а просто обернет ``"c"`` нашим маркером
(см. :ref:`маркировка тестов <mark>`).
К счастью, на помощь приходит ``pytest.mark.MARKER_NAME.with_args``

.. code-block:: python

    # content of test_custom_marker.py
    import pytest


    def hello_world(*args, **kwargs):
        return "Hello World"


    @pytest.mark.my_marker.with_args(hello_world)
    def test_with_args():
        pass

Результатом запуска будет:

.. code-block:: pytest

    $ pytest -q -s
    Mark(name='my_marker', args=(<function hello_world at 0xdeadbeef>,), kwargs={})
    .
    1 passed in 0.12s

Мы видим, что у нашего настраиваемого маркера есть свое множество аргументов,
одним из которых является функция ``hello_world``. В этом и заключается ключевое различие между
созданием маркера в качестве callable-объекта, который за кулисами
вызывает ``__call__``, и использованием ``with_args``.

Считывание маркера, который используется в разных местах
---------------------------------------------------------------

Если вы активно используете маркеры в своем тестовом наборе, то можете столкнуться с ситуацией,
когда маркер применяется к тестовой функции несколько раз. Вы можете посмотреть все эти случаи,
настроив плагин. К примеру, у нас есть модуль:

.. code-block:: python

    # content of test_mark_three_times.py
    import pytest

    pytestmark = pytest.mark.glob("module", x=1)


    @pytest.mark.glob("class", x=2)
    class TestClass:
        @pytest.mark.glob("function", x=3)
        def test_something(self):
            pass

Здесь у нас маркер "glob" применяется к одной и той же функции три раза.
Мы можем увидеть это, прописав в файле ``conftest.py``:

.. code-block:: python

    # content of conftest.py
    import sys


    def pytest_runtest_setup(item):
        for mark in item.iter_markers(name="glob"):
            print("glob args={} kwargs={}".format(mark.args, mark.kwargs))
            sys.stdout.flush()

Давайте запустим без перехвата вывода и посмотрим, что получится:

.. code-block:: pytest

    $ pytest -q -s
    glob args=('function',) kwargs={'x': 3}
    glob args=('class',) kwargs={'x': 2}
    glob args=('module',) kwargs={'x': 1}
    .
    1 passed in 0.12s

Маркировка зависящих от платформы тестов
--------------------------------------------------------------

.. regendoc:wipe

Предпложим, что у нас есть тестовый набор, в котором мы используем
маркеры ``pytest.mark.darwin``, ``pytest.mark.win32`` и т. п.
для маркировки тестов, запускаемых на разных платформах.
При этом в набор такжде входят тесты, которые должны проводиться на всех платформах,
и они никак не помечены. Теперь, если вы хотите запустить
тесты для конкретной платформы, может пригодиться такой плагин:

.. code-block:: python

    # content of conftest.py
    #
    import sys
    import pytest

    ALL = set("darwin linux win32".split())


    def pytest_runtest_setup(item):
        supported_platforms = ALL.intersection(mark.name for mark in item.iter_markers())
        plat = sys.platform
        if supported_platforms and plat not in supported_platforms:
            pytest.skip("cannot run on platform {}".format(plat))

При его использовании тесты для остальных платформ будут пропущены.
Давайте напишем небольшой модуль, чтобы посмотреть, как это выглядит:

.. code-block:: python

    # content of test_plat.py

    import pytest


    @pytest.mark.darwin
    def test_if_apple_is_evil():
        pass


    @pytest.mark.linux
    def test_if_linux_works():
        pass


    @pytest.mark.win32
    def test_if_win32_crashes():
        pass


    def test_runs_everywhere():
        pass

Здесь два теста должны быть пропущены, а два выполнены:

.. code-block:: pytest

    $ pytest -rs # this option reports skip reasons
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items

    test_plat.py s.s.                                                    [100%]

    ========================= short test summary info ==========================
    SKIPPED [2] $REGENDOC_TMPDIR/conftest.py:12: cannot run on platform linux
    ======================= 2 passed, 2 skipped in 0.12s =======================

Обратите внимание, что если вы определяете платформу с помощью маркера и опции ``-m``,
как показано ниже,

.. code-block:: pytest

    $ pytest -m linux
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items / 3 deselected / 1 selected

    test_plat.py .                                                       [100%]

    ===================== 1 passed, 3 deselected in 0.12s ======================

то непомеченные тесты запускаться не будут. Таким образом, это способ ограничиться
выполнением конкретных тестов.

Автоматическое добавление маркеров на основе имен тестов
--------------------------------------------------------

Если в вашем тестовом наборе имена тестовых функций отражают
виды выполняемых тестов, вы можете реализовать hook, который автоматически
задает маркировку для использования с опцией ``-m``.
Взгляните на этот тестовый модуль:

.. code-block:: python

    # content of test_module.py


    def test_interface_simple():
        assert 0


    def test_interface_complex():
        assert 0


    def test_event_simple():
        assert 0


    def test_something_else():
        assert 0

Мы хотим динамически маркировать тесты и можем сделать это
в ``conftest.py``:

.. code-block:: python

    # content of conftest.py

    import pytest


    def pytest_collection_modifyitems(items):
        for item in items:
            if "interface" in item.nodeid:
                item.add_marker(pytest.mark.interface)
            elif "event" in item.nodeid:
                item.add_marker(pytest.mark.event)

Теперь можно использовать для отбора опцию ``-m``:

.. code-block:: pytest

    $ pytest -m interface --tb=short
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items / 2 deselected / 2 selected

    test_module.py FF                                                    [100%]

    ================================= FAILURES =================================
    __________________________ test_interface_simple ___________________________
    test_module.py:4: in test_interface_simple
        assert 0
    E   assert 0
    __________________________ test_interface_complex __________________________
    test_module.py:8: in test_interface_complex
        assert 0
    E   assert 0
    ===================== 2 failed, 2 deselected in 0.12s ======================

Или можно выполнить и "event", и "interface" тесты:

.. code-block:: pytest

    $ pytest -m "interface or event" --tb=short
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items / 1 deselected / 3 selected

    test_module.py FFF                                                   [100%]

    ================================= FAILURES =================================
    __________________________ test_interface_simple ___________________________
    test_module.py:4: in test_interface_simple
        assert 0
    E   assert 0
    __________________________ test_interface_complex __________________________
    test_module.py:8: in test_interface_complex
        assert 0
    E   assert 0
    ____________________________ test_event_simple _____________________________
    test_module.py:12: in test_event_simple
        assert 0
    E   assert 0
    ===================== 3 failed, 1 deselected in 0.12s ======================


