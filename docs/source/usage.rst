.. _usage:

Использование и вызов
=========================


.. _cmdline:

Вызов ``pytest`` с помощью ``python -m pytest``
-----------------------------------------------------

Тестирование можно запустить из командной  строки интерпретатора ``Python``, используя команду:

.. code-block:: text

    python -m pytest [...]

В отличие от запуска напрямую командой ``pytest [...]``, запуск через ``Python``
добавит текущий каталог в  ``sys.path``.


Статусы завершения
-------------------------

Выполнение ``pytest`` может генерировать один из следующих статусов завершения:

:Exit code 0: Все тесты были собраны и успешно прошли
:Exit code 1: Тесты были собраны и запущены, но некоторые из них упали
:Exit code 2: Выполнение тестов было прервано пользователем
:Exit code 3: Во время выполнения тестов произошла внутренняя ошибка
:Exit code 4: Ошибка запуска ``pytest`` из командной строки
:Exit code 5: Не удалось собрать тесты (тесты не найдены)

Коллекция статусов представлена перечислением
`_pytest.config.ExitCode <https://docs.pytest.org/en/latest/reference.html#_pytest.config.ExitCode>`_.
Статусы завершения являются частью публичного API, их можно импортировать и использовать непосредственно:

.. code-block:: python

    from pytest import ExitCode

.. note::

    Для настройки  кода завершения сценария, особенно когда тесты не удалось собрать,
    можно использовать  плагин
    `pytest-custom_exit_code <https://github.com/yashtodi94/pytest-custom_exit_code>`__.

Получение помощи по версии, параметрам, переменным окружения
--------------------------------------------------------------

.. code-block:: bash

    pytest --version   # показывает версию и место, откуда импортирован ``pytest``
    pytest --fixtures  # показывает доступные встроенные функции
    pytest -h | --help # показывает помощь по командной строке и параметры конфигруационного файла


.. _maxfail:

Остановка после первых N падений
---------------------------------------------------

Чтобы остановить процесс тестирования после первых N падений, используются параметры:

.. code-block:: bash

    pytest -x           # остановка после первого упавшего теста
    pytest --maxfail=2  # остановка после первых двух упавших тестов

.. _select-tests:

Выбор выполняемых тестов
-------------------------------

``pytest`` поддерживает несколько способов выбора и запуска тестов из командной строки.

**Запуск тестов модуля**

.. code-block:: bash

    pytest test_mod.py

**Запуск тестов из директории**

.. code-block:: bash

    pytest testing/

**Запуск тестов, удовлетворяющих ключевому выражению**

.. code-block:: bash

    pytest -k "MyClass and not method"

Эта команда запустит тесты, имена которых удовлетворяют заданному строковому выражению
(без учета регистра). Строковые выражения могут включать операторы ``Python``, которые
используют имена файлов, классов и функций в качестве переменных. В приведенном выше
примере будет запущен тест ``MyClass.test_something``, но не  будет запущен тест
``TestMyClass.test_method_simple``.

.. _nodeids:

**Запуск тестов по идентификаторам узлов**

Каждому собранному тесту присваивается уникальный идентификатор ``nodeid``,
который состоит из имени файла модуля, за которым следуют спецификаторы,
такие как имена классов, имена функций и параметры из параметризации, разделенные символами ``::``:

Чтобы запустить конкретный тест из модуля, выполните:

.. code-block:: bash

    pytest test_mod.py::test_func

Еще один пример спецификации тестового метода в командной строке:

.. code-block:: bash

    pytest test_mod.py::TestClass::test_method

**Запуск маркированных тестов**

.. code-block:: bash

    pytest -m slow

Будут запущены тесты, помеченные декоратором ``@pytest.mark.slow``.

Подробнее см. :ref:`marks <mark>`.

**Запуск тестов из пакетов**

.. code-block:: bash

    pytest --pyargs pkg.testing

Будет импортирован пакет ``pkg.testing``, и его расположение в файловой системе
будет использовано для поиска и запуска тестов.

Изменение вывода сообщений трассировки
----------------------------------------------

Примеры вывода:

.. code-block:: bash

    pytest --showlocals # показывать локальные переменные в сообщениях
    pytest -l           # показывать локальные переменные в сообщениях (краткий вариант)
    pytest --tb=auto    # (по умолчанию) "расширенный" вывод для первого и
                        # последнего сообщений, и "короткий" для остальных
    pytest --tb=long    # исчерпывающий, подробный формат сообщений
    pytest --tb=short   # сокращенный формат сообщений
    pytest --tb=line    # только одна строка на падение
    pytest --tb=native  # стандартный формат библиотеки Python
    pytest --tb=no      # никаких сообщений

Использование ``--full-trace`` приводит к тому, что при ошибке печатаются очень длинные
трассировки (длиннее, чем при ``--tb=long``). Параметр также гарантирует, что сообщения
трассировки будут напечатаны при **прерывании выполнения c клавиатуры** с помощью Ctrl+C.
Это очень полезно, если тесты занимают слишком много времени, и вы прерываете их
с клавиатуры с помощью Ctrl+C, чтобы узнать, где они зависли. По умолчанию при прерывании
вывод не будет показан (поскольку исключение ``KeyboardInterrupt`` будет поймано ``pytest``).
Используя этот параметр, вы можете быть уверены, что увидите трассировку.

.. _`pytest.detailed_failed_tests_usage`:

Детализация сводного отчета
------------------------------

Флаг ``-r`` можно использовать для отображения "краткой сводной информации по тестированию"
в конце тестового сеанса, что упрощает получение четкой картины всех сбоев, пропусков, xfails и т. д.

По умолчанию для списка сбоев и ошибок используется добавочная комбинация ``fE``.

Пример:

.. code-block:: python

    # content of test_example.py
    import pytest


    @pytest.fixture
    def error_fixture():
        assert 0


    def test_ok():
        print("ok")


    def test_fail():
        assert 0


    def test_error(error_fixture):
        pass


    def test_skip():
        pytest.skip("skipping this test")


    def test_xfail():
        pytest.xfail("xfailing this test")


    @pytest.mark.xfail(reason="always xfail")
    def test_xpass():
        pass


.. code-block:: pytest

    $ pytest -ra
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 6 items

    test_example.py .FEsxX                                               [100%]

    ================================== ERRORS ==================================
    _______________________ ERROR at setup of test_error _______________________

        @pytest.fixture
        def error_fixture():
    >       assert 0
    E       assert 0

    test_example.py:6: AssertionError
    ================================= FAILURES =================================
    ________________________________ test_fail _________________________________

        def test_fail():
    >       assert 0
    E       assert 0

    test_example.py:14: AssertionError
    ========================= short test summary info ==========================
    SKIPPED [1] $REGENDOC_TMPDIR/test_example.py:22: skipping this test
    XFAIL test_example.py::test_xfail
      reason: xfailing this test
    XPASS test_example.py::test_xpass always xfail
    ERROR test_example.py::test_error - assert 0
    FAILED test_example.py::test_fail - assert 0
    == 1 failed, 1 passed, 1 skipped, 1 xfailed, 1 xpassed, 1 error in 0.12s ===

Параметр ``-r`` принимает ряд символов после себя. Использованный выше символ ``а`` означает
“все, кроме успешных".

Вот полный список доступных символов, которые можно использовать:

 - ``f`` - упавшие (добавляет раздел FAILED)
 - ``E`` - ошибки (добавляет раздел ERROR)
 - ``s`` - пропущенные (добавляет раздел SKIPPED)
 - ``x`` - тесты XFAIL (добавляет раздел XFAIL)
 - ``X`` - тесты XPASS (добавляет раздел XPASS)
 - ``p`` - успешные (passed)
 - ``P`` - успешные (passed) с выводом

Есть и специальные символы для пропуска отдельных групп:

 - ``a`` - выводить все, кроме ``pP``
 - ``A`` - выводить все
 - ``N`` - ничего не выводить (может быть полезным, поскольку по умолчанию используется комбинация ``fE``).

Можно использовать более одного символа. Например, для того, чтобы увидеть только
упавшие и пропущенные тесты, можно выполнить:

.. code-block:: pytest

    $ pytest -rfs
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 6 items

    test_example.py .FEsxX                                               [100%]

    ================================== ERRORS ==================================
    _______________________ ERROR at setup of test_error _______________________

        @pytest.fixture
        def error_fixture():
    >       assert 0
    E       assert 0

    test_example.py:6: AssertionError
    ================================= FAILURES =================================
    ________________________________ test_fail _________________________________

        def test_fail():
    >       assert 0
    E       assert 0

    test_example.py:14: AssertionError
    ========================= short test summary info ==========================
    FAILED test_example.py::test_fail - assert 0
    SKIPPED [1] $REGENDOC_TMPDIR/test_example.py:22: skipping this test
    == 1 failed, 1 passed, 1 skipped, 1 xfailed, 1 xpassed, 1 error in 0.12s ===

Использование ``p`` добавляет в сводный отчет успешные тесты, а ``P`` добавляет
дополнительный раздел "пройдены” (PASSED) для тестов, которые прошли, но перехватили вывод:

.. code-block:: pytest

    $ pytest -rpP
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 6 items

    test_example.py .FEsxX                                               [100%]

    ================================== ERRORS ==================================
    _______________________ ERROR at setup of test_error _______________________

        @pytest.fixture
        def error_fixture():
    >       assert 0
    E       assert 0

    test_example.py:6: AssertionError
    ================================= FAILURES =================================
    ________________________________ test_fail _________________________________

        def test_fail():
    >       assert 0
    E       assert 0

    test_example.py:14: AssertionError
    ================================== PASSES ==================================
    _________________________________ test_ok __________________________________
    --------------------------- Captured stdout call ---------------------------
    ok
    ========================= short test summary info ==========================
    PASSED test_example.py::test_ok
    == 1 failed, 1 passed, 1 skipped, 1 xfailed, 1 xpassed, 1 error in 0.12s ===

.. _pdb-option:

Запуск отладчика PDB_ (Python Debugger) при падении тестов
--------------------------------------------------------------

.. _PDB: http://docs.python.org/library/pdb.html

``python`` содержит встроенный отладчик PDB_ (Python Debugger). ``pytest`` позволяет
запустить отладчик с помощью параметра командной строки:

.. code-block:: bash

    pytest --pdb

Использование параметра позволяет запускать отладчик при каждом падении теста
(или прерывании его с клавиатуры). Часто хочется сделать это для первого же упавшего теста,
чтобы понять причину его падения:

.. code-block:: bash

    pytest -x --pdb   # вызывает отладчик при первом падении и завершает тестовую сессию
    pytest --pdb --maxfail=3  # вызывает отладчик для первых трех падений

Обратите внимание, что при любом падении информация об исключении сохраняется в
``sys.last_value``, ``sys.last_type`` и ``sys.last_traceback``. При интерактивном использовании
это позволяет перейти к отладке после падения с помощью любого инструмента отладки.
Можно также вручную получить доступ к информации об исключениях, например:

    >>> import sys
    >>> sys.last_traceback.tb_lineno
    42
    >>> sys.last_value
    AssertionError('assert result == "ok"',)

.. _trace-option:

Запуск отладчика PDB_ (Python Debugger) в начале теста
----------------------------------------------------------

``pytest`` позволяет запустить отладчик сразу же при старте каждого теста.
Для этого нужно передать следующий параметр:

.. code-block:: bash

    pytest --trace

В этом случае отладчик будет вызываться при запуске каждого теста.

.. _breakpoints:

Установка точек останова
-------------------------

.. versionadded: 2.4.0

Чтобы установить точку останова, вызовите в коде ``import pdb;pdb.set_trace()``, и
``pytest`` автоматически отключит перехват вывода для этого теста, при этом:

* на перехват вывода в других тестах это не повлияет;
* весь перехваченный ранее вывод будет обработан как есть;
* перехват вывода возобновится после завершения отладочной сессии (с помощью команды ``continue``).

.. _`breakpoint-builtin`:

Использование встроенной функции breakpoint
----------------------------------------------

``Python 3.7`` содержит встроенную функцию ``breakpoint()``.
``pytest`` поддерживает использование ``breakpoint()`` следующим образом:

 - если вызывается ``breakpoint()``, и при этом переменная ``PYTHONBREAKPOINT`` установлена в
   значение по умолчанию, ``pytest`` использует расширяемый отладчик PDB_ вместо
   системного;
 - когда тестирование будет завершено, система снова будет использовать отладчик ``Pdb`` по умолчанию;
 - если ``pytest`` вызывается с опцией ``--pdb`` то расширяемый отладчик PDB_ используется
   как для функции ``breakpoint()``, так и для упавших тестов/необработанных исключений;
 - для определения пользовательского класса отладчика можно использовать ``--pdbcls``.

.. _durations:

Профилирование продолжительности выполнения теста
-------------------------------------------------------

Чтобы получить список 10 самых медленных тестов, выполните:

.. code-block:: bash

    pytest --durations=10

По умолчанию, ``pytest`` не покажет тесты со слишком маленькой (менее одной сотой секунды)
длительностью выполнения, если в командной строке не будет передан параметр ``-vv``.


.. _faulthandler:

Модуль ``faulthandler``
-----------------------

Стандартный модуль `faulthandler <https://docs.python.org/3/library/faulthandler.html>`__
можно использовать для сброса трассировок ``Python`` при ошибке или по истечении времени ожидания.

При запуске ``pytest`` модуль автоматически подключается, если только в командной строке
не используется опция ``-p no:faulthandler``.

Кроме того, для сброса трассировок всех потоков в случае, когда тест длится более
``X`` секунд, можно использовать опцию
`faulthandler_timeout=X <https://docs.pytest.org/en/latest/reference.html#confval-faulthandler_timeout>`_
(для Windows неприменима).

.. note::

    Эта функциональность была интегрирована из внешнего плагина
    `pytest-faulthandler <https://github.com/pytest-dev/pytest-faulthandler>`__
    с двумя небольшими изменениями:

    * чтобы ее отключить, используйте ``-p no:faulthandler`` вместо ``--no-faulthandler``;

    * опция командной строки ``--faulthandler-timeout`` превратилась в конфигурационную опцию
      `faulthandler_timeout <https://docs.pytest.org/en/latest/reference.html#confval-faulthandler_timeout>`_.
      Ее по-прежнему можно настроить из команндной строки, используя ``-o faulthandler_timeout=X``.

Создание файлов формата JUnit
----------------------------------------------------

Чтобы создать результирующие файлы в формате, понятном  Jenkins_
или другому серверу непрерывной интеграции, используйте вызов:

.. code-block:: bash

    pytest --junitxml=path

Команда создает xml-файл по указанному пути.

Чтобы задать имя корневого xml-элемента для набора тестов, можно настроить параметр
``junit_suite_name`` в конфигурационном файле:

.. code-block:: ini

    [pytest]
    junit_suite_name = my_suite


Спецификация JUnit XML, по-видимому, указывает, что атрибут ``"time"`` должен сообщать
об общем времени выполнения теста, включая выполнение setup- и teardown- методов
(`1 <http://windyroad.com.au/dl/Open%20Source/JUnit.xsd>`_, `2
<https://www.ibm.com/support/knowledgecenter/en/SSQ2R2_14.1.0/com.ibm.rsar.analysis.codereview.cobol.doc/topics/cac_useresults_junit.html>`_).
Это поведение ``pytest`` по умолчанию. Чтобы вместо этого сообщать только о длительности вызовов,
настройте параметр ``junit_duration_report`` следующим образом:

.. code-block:: ini

    [pytest]
    junit_duration_report = call

.. _record_property example:

record_property
^^^^^^^^^^^^^^^

Чтобы записать дополнительную информацию для теста, используйте фикстуру ``record_property``:

.. code-block:: python

    def test_function(record_property):
        record_property("example_key", 1)
        assert True

Такая запись добавит дополнительное свойство ``example_key="1"`` к сгенерированному тегу ``testcase``:

.. code-block:: xml

    <testcase classname="test_function" file="test_function.py" line="0" name="test_function" time="0.0009">
      <properties>
        <property name="example_key" value="1" />
      </properties>
    </testcase>

Эту функциональность также можно использовать совместно с пользовательскими маркерами:

.. code-block:: python

    # content of conftest.py


    def pytest_collection_modifyitems(session, config, items):
        for item in items:
            for marker in item.iter_markers(name="test_id"):
                test_id = marker.args[0]
                item.user_properties.append(("test_id", test_id))

И в тесте:

.. code-block:: python

    # content of test_function.py
    import pytest


    @pytest.mark.test_id(1501)
    def test_function():
        assert True

В файле получим:

.. code-block:: xml

    <testcase classname="test_function" file="test_function.py" line="0" name="test_function" time="0.0009">
      <properties>
        <property name="test_id" value="1501" />
      </properties>
    </testcase>

.. warning::

    Пожалуйста, обратите внимание, что использование этой возможности приведет к записи
    некорректного с точки зрения  JUnitXML-схем последних версий файла и может вызывать
    проблемы при работе с некоторыми серверами непрерывной интеграции.

record_xml_attribute
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Чтобы добавить дополнительный атрибут в элемент ``testcase``, можно использовать
фикстуру ``record_xml_attribute``. Ее также можно использовать для переопределения
существующих значений:

.. code-block:: python

    def test_function(record_xml_attribute):
        record_xml_attribute("assertions", "REQ-1234")
        record_xml_attribute("classname", "custom_classname")
        print("hello world")
        assert True

В отличие от ``record_property``, дочерний элемент в данном случае не добавляется.
Вместо этого в элемент ``testcase`` будет добавлен атрибут ``assertions="REQ-1234"``,
а значение атрибута ``classname`` по умолчанию будет заменено на ``"classname=custom_classname"``:

.. code-block:: xml

    <testcase classname="custom_classname" file="test_function.py" line="0" name="test_function" time="0.003" assertions="REQ-1234">
        <system-out>
            hello world
        </system-out>
    </testcase>

.. warning::

    ``record_xml_attribute`` пока используется в режиме эксперимента, и в будущем может быть
    заменен чем-то более мощным и/или общим. Однако сама функциональность как таковая будет сохранена.

    Использование ``record_xml_attribute`` поверх ``record_xml_property`` может быть полезным при
    парсинге xml-отчетов средствами непрерывной интеграции. Однако некоторые парсеры допускают
    не любые элементы и атрибуты. Многие инструменты (как в примере ниже), используют xsd-схему
    для валидации входящих xml. Поэтому убедитесь, что имена атрибутов, которые вы используете,
    являются допустимыми для вашего парсера.

    Ниже представлена схема, которую использует ``Jenkins`` для валидации xml-отчетов:

    .. code-block:: xml

        <xs:element name="testcase">
            <xs:complexType>
                <xs:sequence>
                    <xs:element ref="skipped" minOccurs="0" maxOccurs="1"/>
                    <xs:element ref="error" minOccurs="0" maxOccurs="unbounded"/>
                    <xs:element ref="failure" minOccurs="0" maxOccurs="unbounded"/>
                    <xs:element ref="system-out" minOccurs="0" maxOccurs="unbounded"/>
                    <xs:element ref="system-err" minOccurs="0" maxOccurs="unbounded"/>
                </xs:sequence>
                <xs:attribute name="name" type="xs:string" use="required"/>
                <xs:attribute name="assertions" type="xs:string" use="optional"/>
                <xs:attribute name="time" type="xs:string" use="optional"/>
                <xs:attribute name="classname" type="xs:string" use="optional"/>
                <xs:attribute name="status" type="xs:string" use="optional"/>
            </xs:complexType>
        </xs:element>

.. warning::

    Пожалуйста, обратите внимание, что использование этой возможности приведет к записи
    некорректного с точки зрения  JUnitXML-схем последних версий файла и может вызывать
    проблемы при работе с некоторыми серверами непрерывной интеграции.

.. _record_testsuite_property example:

Чтобы добавить свойства каждому тесту из набора, можно использовать фикстуру
``record_testsuite_property`` с параметром ``scope="session"`` (в этом случае она будет
применяться ко всем тестам тестовой сессии).

.. code-block:: python

    import pytest


    @pytest.fixture(scope="session", autouse=True)
    def log_global_env_facts(record_testsuite_property):
        record_testsuite_property("ARCH", "PPC")
        record_testsuite_property("STORAGE_TYPE", "CEPH")


    class TestMe:
        def test_foo(self):
            assert True

Этой фикстуре передаются имя (``name``) и значение (``value``) тэга ``<property>``, который
добавляется на уровне тестового набора для генерируемого xml-файла:

.. code-block:: xml

    <testsuite errors="0" failures="0" name="pytest" skipped="0" tests="1" time="0.006">
      <properties>
        <property name="ARCH" value="PPC"/>
        <property name="STORAGE_TYPE" value="CEPH"/>
      </properties>
      <testcase classname="test_me.TestMe" file="test_me.py" line="16" name="test_foo" time="0.000243663787842"/>
    </testsuite>

``name`` должно быть строкой, а  ``value`` будет преобразовано в строку
и корректно экранировано.

В отличие от случаев использования `record_property`_ и `record_xml_attribute`_
созданный xml-файл будет совместим с последним стандартом ``xunit``.

Создание файлов в формате resultlog
----------------------------------------------------

Для создания машиночитаемых логов в формате ``plain-text`` можно выполнить

.. code-block:: bash

    pytest --resultlog=path

и просмотреть содержимое по указанному пути ``path``. Эти файлы также используются
ресурсом `PyPy-test`_ для отображения результатов тестов после ревизий.

.. warning::

    Поскольку эта возможность редко используется, она запланирована к удалению в ``pytest 6.0``.

    Если вы пользуетесь ею, рассмотрите возможность использования нового плагина
    `pytest-reportlog <https://github.com/pytest-dev/pytest-reportlog>`__ .

    Подробнее см.:
    `the deprecation docs <https://docs.pytest.org/en/latest/deprecations.html#result-log-result-log>`__.

.. _`PyPy-test`: http://buildbot.pypy.org/summary

Отправка отчетов на сервис pastebin
-----------------------------------------------------

**Создание ссылки для каждого упавшего теста**:

.. code-block:: bash

    pytest --pastebin=failed

Эта команда отправит информацию о прохождении теста на удаленный
сервис регистрации и сгенерирует ссылку для каждого падения. Тесты можно отбирать
как обычно, или, например, добавить ``-x``, если вы хотите отправить данные
по конкретному упавшему тесту.

**Создание ссылки для лога тестовой сессии**:

.. code-block:: bash

    pytest --pastebin=all

В настоящее время реализовна регистрация только в сервисе http://bpaste.net.

*Изменено в вресии 5.2*

Если по каким-то причинам не удалось создать ссылку, вместо падения всего тестового набора
генерируется предупреждение.

Подключение плагинов
-----------------------

В командной строке можно явно подгрузить какой-либо внутренний или внешний плагин, используя опцию ``-p``::

    pytest -p mypluginmodule

Опция принимает параметр ``name``, который может быть:

* Полным именем модуля, записанным через точку, например ``myproject.plugins``. Имя должно быть импортируемым.
* "Входным" именем плагина, которое передается в ``setuptools`` при регистрации плагина. К примеру, чтобы подгрузить
  `pytest-cov <https://pypi.org/project/pytest-cov/>`__ , нужно использовать::

    pytest -p pytest_cov


Отключение плагинов
---------------------

Чтобы отключить загрузку определенных плагинов во время вызова, используйте опцию ``-p`` с префиксом ``no:``.

Пример: чтобы отключить загрузку плагина ``doctest``, который отвечает за выполнение
тестов из строк "docstring", вызовите  ``pytest`` следующим образом:

.. code-block:: bash

    pytest -p no:doctest

.. _`pytest.main-usage`:

Вызов ``pytest`` из кода Python
-----------------------------------------

``pytest`` можно вызвать прямо в коде Python:

.. code-block:: python

    pytest.main()

Такой способ эквивалентен вызову "pytest" из командной строки.
В этом случае вместо исключения ``SystemExit`` возвращается статус завершения.
Можно также передавать параметры и опции:

.. code-block:: python

    pytest.main(["-x", "mytestdir"])

Дополнительные плагины можно указать в ``pytest.main``:

.. code-block:: python

    # content of myinvoke.py
    import pytest


    class MyPlugin:
        def pytest_sessionfinish(self):
            print("*** test run reporting finishing")


    pytest.main(["-qq"], plugins=[MyPlugin()])

Выполнив этот код, увидим, что ``MyPlugin`` был загружен и применен:

.. code-block:: pytest

    $ python myinvoke.py
    .FEsxX.                                                              [100%]*** test run reporting finishing

    ================================== ERRORS ==================================
    _______________________ ERROR at setup of test_error _______________________

        @pytest.fixture
        def error_fixture():
    >       assert 0
    E       assert 0

    test_example.py:6: AssertionError
    ================================= FAILURES =================================
    ________________________________ test_fail _________________________________

        def test_fail():
    >       assert 0
    E       assert 0

    test_example.py:14: AssertionError

.. note::

    Вызов ``pytest.main()`` приводит к тому, что импортируются не только тесты,
    но и все модули, которые они используют. Из-за механизма кэширования импорта
    ``Python`` последующие вызовы ``pytest.main()`` из того же процесса не будут учитывать
    изменения в файлах, внесенные между вызовами. Поэтому не рекомендуется многократное
    использование ``pytest.main()`` в одном и том же процессе (например, при перезапуске тестов).


.. include:: links.inc

