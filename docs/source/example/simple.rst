Основные шаблоны и примеры
==========================================================

.. _request example:

Передача значений тестовой функции с помощью опций командной строки
----------------------------------------------------------------------------

Предположим, что мы хотим написать тест, который зависит от опции
командной строки. Вот как это делается:

.. code-block:: python

    # content of test_sample.py
    def test_answer(cmdopt):
        if cmdopt == "type1":
            print("first")
        elif cmdopt == "type2":
            print("second")
        assert 0  # чтобы увидеть вывод


Чтобы это работало, нам нужно добавить опцию командной строки и
представить ``cmdopt`` через :ref:`фикстуру <fixture function>`:

.. code-block:: python

    # content of conftest.py
    import pytest


    def pytest_addoption(parser):
        parser.addoption(
            "--cmdopt", action="store", default="type1", help="my option: type1 or type2"
        )


    @pytest.fixture
    def cmdopt(request):
        return request.config.getoption("--cmdopt")

Давайте запустим БЕЗ нашей новой опции:

.. code-block:: pytest

    $ pytest -q test_sample.py
    F                                                                    [100%]
    ================================= FAILURES =================================
    _______________________________ test_answer ________________________________

    cmdopt = 'type1'

        def test_answer(cmdopt):
            if cmdopt == "type1":
                print("first")
            elif cmdopt == "type2":
                print("second")
    >       assert 0  # to see what was printed
    E       assert 0

    test_sample.py:6: AssertionError
    --------------------------- Captured stdout call ---------------------------
    first
    1 failed in 0.12s

А теперь применим ``cmdopt``:

.. code-block:: pytest

    $ pytest -q --cmdopt=type2
    F                                                                    [100%]
    ================================= FAILURES =================================
    _______________________________ test_answer ________________________________

    cmdopt = 'type2'

        def test_answer(cmdopt):
            if cmdopt == "type1":
                print("first")
            elif cmdopt == "type2":
                print("second")
    >       assert 0  # to see what was printed
    E       assert 0

    test_sample.py:6: AssertionError
    --------------------------- Captured stdout call ---------------------------
    second
    1 failed in 0.12s

Как видите, значение опции появилось в нашем тесте. Это основной шаблон.
Однако скорее всего захочется обрабатывать опцию вне тестов, а так же
передавать ее различным или более сложным объектам.

Динамическое добавлений опций командной строки
--------------------------------------------------------------

С помощью :ref:`addopts ref` можно статически добавить опцию
командной строки для вашего проекта. Можно также динамически
модифицировать аргументы командной строки перед их обработкой:

.. code-block:: python

    # setuptools plugin
    import sys


    def pytest_load_initial_conftests(args):
        if "xdist" in sys.modules:  # pytest-xdist plugin
            import multiprocessing

            num = max(multiprocessing.cpu_count() / 2, 1)
            args[:] = ["-n", str(num)] + args

Если у вас установлен `xdist plugin <https://pypi.org/project/pytest-xdist/>`_,
то теперь вы будете всегда прогонять тесты с использованием числа
подпроцессов, близкого к параметрам вашего  процессора.
Запустим в пустой директории с нашим ``conftest.py``:

.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 0 items

    ========================== no tests ran in 0.12s ===========================

.. _`excontrolskip`:

Конролируем пропуск тестов с помощью опции командной строки
--------------------------------------------------------------

Добавим в файл ``conftest.py`` опцию ``--runslow``, чтобы
контролировать пропуск тестов с пометкой ``pytest.mark.slow``:

.. code-block:: python

    # content of conftest.py

    import pytest


    def pytest_addoption(parser):
        parser.addoption(
            "--runslow", action="store_true", default=False, help="run slow tests"
        )


    def pytest_configure(config):
        config.addinivalue_line("markers", "slow: mark test as slow to run")


    def pytest_collection_modifyitems(config, items):
        if config.getoption("--runslow"):
            #  опция --runslow запрошена в командной строке: медленные тесты не пропускаем
            return
        skip_slow = pytest.mark.skip(reason="need --runslow option to run")
        for item in items:
            if "slow" in item.keywords:
                item.add_marker(skip_slow)

Можно теперь написать тестовый модуль:

.. code-block:: python

    # content of test_module.py
    import pytest


    def test_func_fast():
        pass


    @pytest.mark.slow
    def test_func_slow():
        pass

Если запустим его, то "медленный" тест будет пропущен:

.. code-block:: pytest

    $ pytest -rs    # "-rs" means report details on the little 's'
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items

    test_module.py .s                                                    [100%]

    ========================= short test summary info ==========================
    SKIPPED [1] test_module.py:8: need --runslow option to run
    ======================= 1 passed, 1 skipped in 0.12s =======================

А теперь запустим и медленные тесты, применив нашу опцию ``--runslow``:

.. code-block:: pytest

    $ pytest --runslow
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items

    test_module.py ..                                                    [100%]

    ============================ 2 passed in 0.12s =============================


Настройка ``__tracebackhide__``
--------------------------------------------------

Если у вас есть вспомогательная функция, которую вы используете в тесте,
то можно использовать маркер ``pytest.fail``, чтобы "уронить" тест
с определенным сообщением. Вспомогательная функция не будет отображаться
в трейсбэке, если вы примените опцию ``__tracebackhide__``
где-нибудь в теле этой функции.
Пример:

.. code-block:: python

    # content of test_checkconfig.py
    import pytest


    def checkconfig(x):
        __tracebackhide__ = True
        if not hasattr(x, "config"):
            pytest.fail("not configured: {}".format(x))


    def test_something():
        checkconfig(42)

Настройка ``__tracebackhide__`` влияет на то, ЧТО  ``pytest``
выводит в трейсбэке: функция ``checkconfig`` не будет показана,
пока в командной строке не будет применена опция ``--full-trace``.
Давайте запустим наш маленький тест:

.. code-block:: pytest

    $ pytest -q test_checkconfig.py
    F                                                                    [100%]
    ================================= FAILURES =================================
    ______________________________ test_something ______________________________

        def test_something():
    >       checkconfig(42)
    E       Failed: not configured: 42

    test_checkconfig.py:11: Failed
    1 failed in 0.12s

Если вы хотите скрыть только определенные исключения, можно сопоставить
``__tracebackhide__``  объект, который, в свою очередь, вернет
объект ``ExceptionInfo``. Это можно использовать, к примеру, для того
чтобы убедиться, что неожиданные исключения будут отображены:

.. code-block:: python

    import operator
    import pytest


    class ConfigException(Exception):
        pass


    def checkconfig(x):
        __tracebackhide__ = operator.methodcaller("errisinstance", ConfigException)
        if not hasattr(x, "config"):
            raise ConfigException("not configured: {}".format(x))


    def test_something():
        checkconfig(42)

Такое решение позволит избежать скрытия в трассировке неожиданных исключений.



Как определить, запущено ли приложение из ``pytest``
--------------------------------------------------------------

Вообще-то, заставлять приложение вести себя по-другому при тестировании
- плохая идея. Но если уж совершенно необходимо выяснить,
запускается ли приложение из теста или нет, можно сделать как-то так:

.. code-block:: python

    # content of your_module.py


    _called_from_test = False

.. code-block:: python

    # content of conftest.py


    def pytest_configure(config):
        your_module._called_from_test = True

И потом проверять флажок ``your_module._called_from_test``:

.. code-block:: python

    if your_module._called_from_test:
        # запущено для тестирования
        ...
    else:
        # запущено "нормально"
        ...

в самом приложении


Добавление информации к заголовку отчета
--------------------------------------------------------------

Добавить дополнительную информацию к запуску ``pytest`` легко:

.. code-block:: python

    # content of conftest.py


    def pytest_report_header(config):
        return "project deps: mylib-1.1"

При выводе эта строка отобразится в  заголовке:


.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    project deps: mylib-1.1
    rootdir: $REGENDOC_TMPDIR
    collected 0 items

    ========================== no tests ran in 0.12s ===========================


Можно возвращать список строк - для каждого элемента списка будет добавлена отдельная строка.
Можно также рассмотреть ``config.getoption('verbose')`` для получения подробной информации:

.. code-block:: python

    # content of conftest.py


    def pytest_report_header(config):
        if config.getoption("verbose") > 0:
            return ["info1: did you know that ...", "did you?"]

Эта строка будет добавлена только при использовании опции ``--v``:

.. code-block:: pytest

    $ pytest -v
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    info1: did you know that ...
    did you?
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 0 items

    ========================== no tests ran in 0.12s ===========================

А без этой опции вывод не изменится:

.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 0 items

    ========================== no tests ran in 0.12s ===========================

Определение продолжительности выполнения тестов
--------------------------------------------------

Если у вас есть медленно выполняющийся огромный набор тестов, то
может возникнуть желание выяснить, какие тесты самые медленные.
Давайте создадим искусственный тестовый набор:

.. code-block:: python

    # content of test_some_are_slow.py
    import time


    def test_funcfast():
        time.sleep(0.1)


    def test_funcslow1():
        time.sleep(0.2)


    def test_funcslow2():
        time.sleep(0.3)

Теперь мы можем выяснить продолжительность трех самых медленных тестов с помощью ``--durations=3``:

.. code-block:: pytest

    $ pytest --durations=3
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 3 items

    test_some_are_slow.py ...                                            [100%]

    ========================= slowest 3 test durations =========================
    0.30s call     test_some_are_slow.py::test_funcslow2
    0.20s call     test_some_are_slow.py::test_funcslow1
    0.11s call     test_some_are_slow.py::test_funcfast
    ============================ 3 passed in 0.12s =============================

Тестирование по шагам (incremental testing)
---------------------------------------------------

Иногда тесты могут состоять из нескольких серий, и выполнять их надо по шагам.
Если на каком-то шаге тест упал, нет смысла выполнять следующие шаги этой серии,
поскольку они в любом случае должны упасть и трейсбэк не пополнится
никакой полезной информацией. Ниже - пример файла ``conftest.py``,
который вводдит маркер ``incremental`` для использования с классами:

.. code-block:: python

    # content of conftest.py

    # сохраняем историю падений в разрезе имен классов и индексов в параметризации (если она используется)
    _test_failed_incremental: Dict[str, Dict[Tuple[int, ...], str]] = {}


    def pytest_runtest_makereport(item, call):
        if "incremental" in item.keywords:
            # используется маркер incremental
            if call.excinfo is not None:
                # тест упал
                # извлекаем из теста имя класса
                cls_name = str(item.cls)
                # извлекаем индексы теста (если вместе с  incremental используется параметризация)
                parametrize_index = (
                    tuple(item.callspec.indices.values())
                    if hasattr(item, "callspec")
                    else ()
                )
                # извлекаем имя тестовой функции
                test_name = item.originalname or item.name
                # сохраняем в _test_failed_incremental оригинальное имя упавшего теста
                _test_failed_incremental.setdefault(cls_name, {}).setdefault(
                    parametrize_index, test_name
                )


    def pytest_runtest_setup(item):
        if "incremental" in item.keywords:
            # извлекаем из теста имя класса
            cls_name = str(item.cls)
            # проверяем, падал ли предыдущий тест на этом классе
            if cls_name in _test_failed_incremental:
                # извлекаем индексы теста (если вместе с  incremental используется параметризация)
                parametrize_index = (
                    tuple(item.callspec.indices.values())
                    if hasattr(item, "callspec")
                    else ()
                )
                # извлекаем имя первой тестовой функции, которая должна упасть для этого имени класса и индекса
                test_name = _test_failed_incremental[cls_name].get(parametrize_index, None)
                # если нашли такое имя, значит, тест падал для такой комбинации класса & фукнкции
                if test_name is not None:
                    pytest.xfail("previous test failed ({})".format(test_name))


Эти два хука совместно работают на прерывание маркированных ``incremental``
тестов в классе. Вот пример тестового модуля:

.. code-block:: python

    # content of test_step.py

    import pytest


    @pytest.mark.incremental
    class TestUserHandling:
        def test_login(self):
            pass

        def test_modification(self):
            assert 0

        def test_deletion(self):
            pass


    def test_normal():
        pass

Запустим:

.. code-block:: pytest

    $ pytest -rx
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 4 items

    test_step.py .Fx.                                                    [100%]

    ================================= FAILURES =================================
    ____________________ TestUserHandling.test_modification ____________________

    self = <test_step.TestUserHandling object at 0xdeadbeef>

        def test_modification(self):
    >       assert 0
    E       assert 0

    test_step.py:11: AssertionError
    ========================= short test summary info ==========================
    XFAIL test_step.py::TestUserHandling::test_deletion
      reason: previous test failed (test_modification)
    ================== 1 failed, 2 passed, 1 xfailed in 0.12s ==================

Посокольку ``test_modification`` упал, ``test_deletion`` не выполнялся
и попал в отчет как "ожидаемо падающий" (xfailed).

Фикстуры уровня пакета/каталога (setups)
-------------------------------------------------------

Если в вашем дереве тестов есть вложенные каталоги, можно каждый из них
рассматривать как область действия фикстур - для этого достаточно разместить
фикстуры в файле ``conftest.py`` соответствующего каталога. При этом можно
использовать все типы фикстур, включая фикстуры :ref:`autouse <autouse fixtures>`
- аналоги ``setup/teardown`` функций ``xUnit`` Однако имейте в виду,
что рекомендуется явно ссылаться на фикстуры в тестах и классах, вместо того,
чтобы полагаться на неявное выполнение ``setup/teardown`` функций,
особенно если они расположены далеко от использующих их тестов.

Вот пример, как сделать фикстуру ``db`` доступной в каталоге:

.. code-block:: python

    # content of a/conftest.py
    import pytest


    class DB:
        pass


    @pytest.fixture(scope="session")
    def db():
        return DB()

И тестовый модуль в этой же директории:

.. code-block:: python

    # content of a/test_db.py
    def test_a1(db):
        assert 0, db  # to show value

Еще один тестовый модуль:

.. code-block:: python

    # content of a/test_db2.py
    def test_a2(db):
        assert 0, db  # to show value

А этот модуль расположен в соседней (сестринской) директории, и там
фикстура ``db`` будет не видна:

.. code-block:: python

    # content of b/test_error.py
    def test_root(db):  # no db here, will error out
        pass

Теперь запустим:

.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 7 items

    test_step.py .Fx.                                                    [ 57%]
    a/test_db.py F                                                       [ 71%]
    a/test_db2.py F                                                      [ 85%]
    b/test_error.py E                                                    [100%]

    ================================== ERRORS ==================================
    _______________________ ERROR at setup of test_root ________________________
    file $REGENDOC_TMPDIR/b/test_error.py, line 1
      def test_root(db):  # no db here, will error out
    E       fixture 'db' not found
    >       available fixtures: cache, capfd, capfdbinary, caplog, capsys, capsysbinary, doctest_namespace, monkeypatch, pytestconfig, record_property, record_testsuite_property, record_xml_attribute, recwarn, tmp_path, tmp_path_factory, tmpdir, tmpdir_factory
    >       use 'pytest --fixtures [testpath]' for help on them.

    $REGENDOC_TMPDIR/b/test_error.py:1
    ================================= FAILURES =================================
    ____________________ TestUserHandling.test_modification ____________________

    self = <test_step.TestUserHandling object at 0xdeadbeef>

        def test_modification(self):
    >       assert 0
    E       assert 0

    test_step.py:11: AssertionError
    _________________________________ test_a1 __________________________________

    db = <conftest.DB object at 0xdeadbeef>

        def test_a1(db):
    >       assert 0, db  # to show value
    E       AssertionError: <conftest.DB object at 0xdeadbeef>
    E       assert 0

    a/test_db.py:2: AssertionError
    _________________________________ test_a2 __________________________________

    db = <conftest.DB object at 0xdeadbeef>

        def test_a2(db):
    >       assert 0, db  # to show value
    E       AssertionError: <conftest.DB object at 0xdeadbeef>
    E       assert 0

    a/test_db2.py:2: AssertionError
    ============= 3 failed, 2 passed, 1 xfailed, 1 error in 0.12s ==============

Оба тестовых модуля из каталога ``a`` видят одну и ту же фикстуру ``db``,
а вот модуль из каталога ``b`` ее не видит. Конечно, мы можем так же
определить фикстуру ``db`` в файле ``b/conftest.py``. Обратите внимание,
что каждая фикстура создается, только если требуется в тесте (кроме ``autouse``
фикстур - они всегда выполняются перед запуском тестов).

Обработка отчетов
---------------------------------------

Если нужно обрабатывать отчеты ``pytest`` или получать доступ
к исполняющему тесты окружению, можно реализовать хук, который будет вызываться
во время создания объекта "report". Ниже мы обрабатываем все упавшие тесты
и получаем доступ к фикстуре (если она используется в тестах), которую
хотим посмотреть во время обработки. Всю информацию мы запишем
в файл ``failures``:

.. code-block:: python

    # content of conftest.py

    import pytest
    import os.path


    @pytest.hookimpl(tryfirst=True, hookwrapper=True)
    def pytest_runtest_makereport(item, call):
        # выполняем все остальные хуки, чтобы получить report object
        outcome = yield
        rep = outcome.get_result()

        # мы ищем только вызовы упавших тестов, а не setup/teardown
        if rep.when == "call" and rep.failed:
            mode = "a" if os.path.exists("failures") else "w"
            with open("failures", mode) as f:
                # давайте ради прикола посмотрим на фикстуру
                if "tmpdir" in item.fixturenames:
                    extra = " ({})".format(item.funcargs["tmpdir"])
                else:
                    extra = ""

                f.write(rep.nodeid + extra + "\n")


Допустим, у нас есть тесты:

.. code-block:: python

    # content of test_module.py
    def test_fail1(tmpdir):
        assert 0


    def test_fail2():
        assert 0

Запустим их:

.. code-block:: pytest

    $ pytest test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items

    test_module.py FF                                                    [100%]

    ================================= FAILURES =================================
    ________________________________ test_fail1 ________________________________

    tmpdir = local('PYTEST_TMPDIR/test_fail10')

        def test_fail1(tmpdir):
    >       assert 0
    E       assert 0

    test_module.py:2: AssertionError
    ________________________________ test_fail2 ________________________________

        def test_fail2():
    >       assert 0
    E       assert 0

    test_module.py:6: AssertionError
    ============================ 2 failed in 0.12s =============================

Мы получили файл ``failures`` с идентификаторами упавших тестов:

.. code-block:: bash

    $ cat failures
    test_module.py::test_fail1 (PYTEST_TMPDIR/test_fail10)
    test_module.py::test_fail2


Как сделать информацию о результатах тестов доступной для фикстуры
--------------------------------------------------------------------

Если вы хотите, чтобы отчеты о результатах тестов были доступны в
финализирующей фикстуре, можно реализовать следующий небольшой плагин:

.. code-block:: python

    # content of conftest.py

    import pytest


    @pytest.hookimpl(tryfirst=True, hookwrapper=True)
    def pytest_runtest_makereport(item, call):
        # выполняем все остальные хуки до получения report object
        outcome = yield
        rep = outcome.get_result()

        # устанавливаем атрубут отчета на каждом этапе вызова:
        # "setup", "call", "teardown"

        setattr(item, "rep_" + rep.when, rep)


    @pytest.fixture
    def something(request):
        yield
        # "request.node" в данном случае "item", поскольку по мы используем уровень
        # по умолчанию - "function" scope
        if request.node.rep_setup.failed:
            print("setting up a test failed!", request.node.nodeid)
        elif request.node.rep_setup.passed:
            if request.node.rep_call.failed:
                print("executing test failed", request.node.nodeid)


Теперь, пусть у нас есть неудачные тесты:

.. code-block:: python

    # content of test_module.py

    import pytest


    @pytest.fixture
    def other():
        assert 0


    def test_setup_fails(something, other):
        pass


    def test_call_fails(something):
        assert 0


    def test_fail2():
        assert 0

Запустим их:

.. code-block:: pytest

    $ pytest -s test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 3 items

    test_module.py Esetting up a test failed! test_module.py::test_setup_fails
    Fexecuting test failed test_module.py::test_call_fails
    F

    ================================== ERRORS ==================================
    ____________________ ERROR at setup of test_setup_fails ____________________

        @pytest.fixture
        def other():
    >       assert 0
    E       assert 0

    test_module.py:7: AssertionError
    ================================= FAILURES =================================
    _____________________________ test_call_fails ______________________________

    something = None

        def test_call_fails(something):
    >       assert 0
    E       assert 0

    test_module.py:15: AssertionError
    ________________________________ test_fail2 ________________________________

        def test_fail2():
    >       assert 0
    E       assert 0

    test_module.py:19: AssertionError
    ======================== 2 failed, 1 error in 0.12s ========================

Как видите, финализаторы могут использовать информацию из отчета.

.. _pytest current test env:


Переменная окружения ``PYTEST_CURRENT_TEST``
--------------------------------------------

Иногда тестовая сессия может зависнуть, и бывает непросто выяснить,
на каком именно тесте она "застряла" (например, ``pytest`` запущен
в "тихом" (``-q``) режиме или нет доступа к консольному выводу).
Это особенно неприятно, когда проблема возникает нерегулярно -
получаем так называемые "мерцающие" (*flaky*) тесты.

При запуске тестов ``pytest`` задает переменную окружения ``PYTEST_CURRENT_TEST``,
которую можно проверять с помощью утилит мониторинга процессов или библиотек
вроде `psutil <https://pypi.org/project/psutil/>`_ для того, чтобы выяснить,
какой именно тест "застрял":


.. code-block:: python

    import psutil

    for pid in psutil.pids():
        environ = psutil.Process(pid).environ()
        if "PYTEST_CURRENT_TEST" in environ:
            print(f'pytest process {pid} running: {environ["PYTEST_CURRENT_TEST"]}')

Во время тестовой сессии ``pytest`` будет присваивать ``PYTEST_CURRENT_TEST``
текущий идентификатор узла (:ref:`nodeid <nodeids>`) и текущее состояние:
``setup``, ``call`` или ``teardown``.

К примеру, если мы запустим одну тестовую функцию ``test_foo``
из модуля  ``foo_module.py``, ``PYTEST_CURRENT_TEST`` будет принимать следующие значения:

#. ``foo_module.py::test_foo (setup)``
#. ``foo_module.py::test_foo (call)``
#. ``foo_module.py::test_foo (teardown)``

Именно в таком порядке.

.. note::

    Поскольку содержимое ``PYTEST_CURRENT_TEST`` должно быть читабельно,
    текущий формат от релиза к релизу может меняться (даже при фиксации багов),
    поэтому не стоит полагаться именно на такой вид при написании сценариев
    и автоматизации.


.. _freezing-pytest:

"Заморозка" ``pytest``
-------------------------


Если вы "замораживаете" приложение с помощью инструмента вроде
`PyInstaller <https://pyinstaller.readthedocs.io>`_, чтобы распространить
его среди конечных пользователей, хорошей идеей будет упаковать и ваш
``pytest`` и запускать тесты с "замороженным" приложением.
Благодаря такому способу некоторые ошибки (например, отсутствие в исполняемом
файле нужных зависимостей) могут быть обнаружены на раннем этапе;
кроме того, это позволяет вам отправлять тестовые файлы пользователям,
чтобы они сами могли запустить тесты на своих машинах, что может
быть полезно  для получения дополнительной информации о
трудновоспроизводимой ошибке.

К счастью, в последних релизах ``PyInstaller`` уже есть хук
для ``pytest``, но если вы используете для "заморозки" другие инструменты,
такие как ``cx_freeze`` или ``py2exe``, можно использовать
``pytest.freeze_includes()`` для получения полного списка используемых
``pytest`` модулей. Однако конфигурирование инструмента для
поиска внутренних модулей зависит от используемого инструмента.

Вместо того, чтобы "заморозить" ``pytest`` как отдельный исполняемый файл,
можно заставить "замороженную" программу воспринимать ``pytest`` как некий
хитрый аргумент, к которому она обращается во время запуска.
Это позволит вам иметь один исполняемый файл - обычно так удобнее.
Обратите внимание, что механизм поиска плагинов, используемый ``pytest``,
не работает с "замороженными" исполняемыми файлами, поэтому
``pytest`` не сможет найти сторонний плагин автоматически.
Чтобы подключить стороние плагины вроде ``pytest-timeout``, их нужно
явно импортировать и передать в ``pytest.main``.

.. code-block:: python

    # contents of app_main.py
    import sys
    import pytest_timeout  # сторонний плагин

    if len(sys.argv) > 1 and sys.argv[1] == "--pytest":
        import pytest

        sys.exit(pytest.main(sys.argv[2:], plugins=[pytest_timeout]))
    else:
        # нормальное выполнение приложения: здесь можно проанализировать argv
        # как обычно
        ...


Такой шаблон позволит вам запускать тесты на "замороженном"
приложении со стандартными опциями командной строки ``pytest``:

.. code-block:: bash

    ./app_main --pytest --verbose --tb=long --junitxml=results.xml test-suite/
