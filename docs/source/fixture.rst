.. _fixture:
.. _fixtures:
.. _`fixture functions`:

Фикстуры ``pytest``: явные, модальные, расширяемые
========================================================

.. currentmodule:: _pytest.python


`Тестовые фикстуры <https://en.wikipedia.org/wiki/Test_fixture#Software>`_
инициализируют тестовые функции.
Они обеспечивают надежность тестов, согласованнность и повторямость их результатов.
При инициализации можно настраивать сервисы, состояния, переменные окружения.
Доступ к ним осуществляется через аргументы тестовых функций; для каждой фикстуры,
используемой тестовой функцией, в самой функции, как правило,  существует
соответствующий аргумент, имя которого совпадает с наименованием фикстуры.

Фикстуры ``pytest`` значительно удобнее классических
``setup/teardown``-функций ``xUnit``, поскольку:

* фикстуры имеют явные имена и активируются путем их объявления
  в тестовых функциях, модулях, классах и проектах.

* фикстуры реализованы модально: каждый вызов фикстуры инициализирует
  *функцию-фикстуру*, которая в свою очередь может использовать другие фикстуры.

* управление фикстурами расширяется от простого модуля до комплексного функционального тестирования,
  позволяя параметризовать фикстуры и тесты в соответствии с конфигурацией и опциями компонентов,
  или повторно использовать фикстуры внутри функции, класса, модуля или тестовой сесси в целом.

При этом ``pytest`` продолжает поддерживать :ref:`xunitsetup`. Можно смешивать оба стиля,
постепенно переходя от классического стиля к новому, если вам так нравится.
Можно начинать с существующего :ref:`стиля unittest.TestCase <unittest.TestCase>`
или с проектов :ref:`nose <nosestyle>`.


:ref:`Фикстуры <fixtures-api>` определяются с использованием декоратора
:ref:`@pytest.fixture <pytest.fixture-api>`, :ref:`описанного ниже <funcargs>`.
В ``pytest`` есть полезные встроенные фикстуры,
см. `список встроенных фикстур <https://docs.pytest.org/en/latest/fixture.html>`_.


.. _`funcargs`:
.. _`funcarg mechanism`:
.. _`fixture function`:
.. _`@pytest.fixture`:
.. _`pytest.fixture`:

Фикстуры как аргументы функций
-----------------------------------------

Тестовые функции принимают фикстуры как входящий аргумент с тем же именем.
Для каждого такого аргумента функция-фикстура предоставляет объект фикстуры.
Для того, чтобы зарегистировать функцию как фикстуру, нужно использовать
декоратор ``@pytest.fixture``. Давайте рассмотрим простой тестовый модуль,
содержащий фикстуру и использующую ее тестовую функцию:

.. code-block:: python

    # content of ./test_smtpsimple.py
    import pytest


    @pytest.fixture
    def smtp_connection():
        import smtplib

        return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)


    def test_ehlo(smtp_connection):
        response, msg = smtp_connection.ehlo()
        assert response == 250
        assert 0  # в демонстрационных целях

Здесь ``test_ehlo`` использует значение фикстуры ``smtp_connection``. При передаче
аргумента ``pytest`` найдет и вызовет маркированную функцию-фикстуру ``smtp_connection``.
Запустив тест, увидим следующее:

.. code-block:: pytest

    $ pytest test_smtpsimple.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item

    test_smtpsimple.py F                                                 [100%]

    ================================= FAILURES =================================
    ________________________________ test_ehlo _________________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_smtpsimple.py:14: AssertionError
    ============================ 1 failed in 0.12s =============================

В трейсбеке мы видим, что тестовая функция была вызвана с аргументом ``smtp_connection`` -
объектом ``smtplib.SMTP()``, который был создан фикстурой. Тестовая функция упала
на проверке ``assert 0``. В данном случае ``pytest`` использует следующий алгоритм
для вызова тестовой функции:

1. ``pytest`` :ref:`находит <test discovery>` функцию ``test_ehlo`` по ее префиксу
   ``test_``. Ей передается аргумент с именем ``smtp_connection``, поэтому ``pytest``
   ищет и находит функцию с именем ``smtp_connection``, помеченную как фикстура.

2. Фикстура ``smtp_connection()`` вызывается для создания объекта-функции.

3. Затем вызывается функция ``test_ehlo(<объект smtp_connection>)``, выполняется
   и падает на последней строчке.

Обратите внимание, что если вы неправильно напишите имя аргумента-функции
или попытаетесь использовать недоступную функцию, то получите ошибку
со списком доступных аргументов-функций.

.. note::

    Чтобы посмотреть список доступных фикстур, можно использовать опцию ``--fixtures``:

    .. code-block:: bash

        pytest --fixtures test_simplefactory.py

    При этом фикстуры с ведущим символом "_" будут выведены в список, только если вы используете опцию ``-v``.

Фикстуры: яркий пример внедрения зависимостей
---------------------------------------------------

Фикстуры позволяют тестовым функциям легко получать
предварительно инициализированные объекты и работать с ними,
не заботясь об импорте/установке/очистке.

Вот яркий пример `внедрения зависимостей <https://en.wikipedia.org/wiki/Dependency_injection>`_,
где фикстуры играют роль *внедренного объекта*, а тестовые функции
являются *потребителями* объектов-фикстур.

.. _`conftest.py`:
.. _`conftest`:

``conftest.py``: расширение фикстур
------------------------------------------

Если вы планируете использовать фикстуру в нескольких тестах,
то можно объявить ее в файле ``conftest.py``. При этом
импортировать ее не нужно - ``pytest`` найдет ее автоматически.
Поиск фикстур начинается с тестовых классов, затем они иущется в
тестовых модулях и в файлах ``conftest.py``, и, в последнюю очередь,
во встроенных и сторонних плагинах.

В ``conftest.py`` также можно встраивать
:ref:`плагины для подкаталогов <conftest.py plugins>`.

Расширение тестовых данных
-----------------------------

Хороший способ для того, чтобы сделать тестовые данные из файлов доступными для ваших
тестов - загрузка этих данных в фикстуру.
При этом используются механизмы кэширования ``pytest``.

Еще один хороший подход заключается в добавлении файлов с данными в папку ``tests``.
Существуют также плагины, которые помогают управлять этим аспектом тестирования,
например, `pytest-datadir <https://pypi.org/project/pytest-datadir/>`__, или
`pytest-datafiles <https://pypi.org/project/pytest-datafiles/>`__.

.. _smtpshared:

Область действия (уровень) фикстуры: расширение фикстуры на все тесты класса, модуля, сессии
----------------------------------------------------------------------------------------------

.. regendoc:wipe

Фикстуры, требующие доступа к сети, зависят от подключения и обычно
требуют больших временных затрат на их создание. Расширяя предыдущий пример,
мы можем добавить параметр ``scope="module"`` в декоратор фикстуры
``@pytest.fixture`` , чтобы функция-фикстура ``smtp_connection``
вызывалась только один раз для тестового модуля (по умолчанию параметр
``scope`` установлен в значение ``function``). Таким образом,
каждая тестовая функция модуля получит тот же самый объект ``smtp_connection``,
что позволит сэкономить время на создание подключения. Возможными значениями
параметра ``scope`` являются ``function``, ``class``, ``module``, ``package`` или ``session``.

В следующем примере мы помещаем фикстуру в файл ``conftest.py``, чтобы
доступ к ней могли иметь разные тесты из разных тестовых модулей нашего каталога:

.. code-block:: python

    # content of conftest.py
    import pytest
    import smtplib


    @pytest.fixture(scope="module")
    def smtp_connection():
        return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

Наша фикстура по-прежнему называется ``smtp_connection``, и получить к ней доступ
из любой тестовой функции или другой фикстуры (в пределах директории, в которой
расположен наш  файл ``conftest.py`` и ее поддиректорий) можно,
передав параметр ``smtp_connection`` в объявлении нашей функции/фикстуры:


.. code-block:: python

    # content of test_module.py


    def test_ehlo(smtp_connection):
        response, msg = smtp_connection.ehlo()
        assert response == 250
        assert b"smtp.gmail.com" in msg
        assert 0  # в демонстрационных целях


    def test_noop(smtp_connection):
        response, msg = smtp_connection.noop()
        assert response == 250
        assert 0  # в демонстрационных целях

Здесь мы специально вставляем обреченные на неудачу операторы ``assert 0``,
чтобы посмотреть, что происходит при запуске тестов:

.. code-block:: pytest

    $ pytest test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items

    test_module.py FF                                                    [100%]

    ================================= FAILURES =================================
    ________________________________ test_ehlo _________________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
            assert b"smtp.gmail.com" in msg
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:7: AssertionError
    ________________________________ test_noop _________________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_noop(smtp_connection):
            response, msg = smtp_connection.noop()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:13: AssertionError
    ============================ 2 failed in 0.12s =============================

Мы видим, что оба оператора ``assert 0`` упали. И, поскольку ``pytest``
показывает значения входящих параметров, мы также можем увидеть,
что в обе тестовые функции был передан один и тот же объект
``smtp_connection`` (с уровнем модуля).
В итоге, мы выполнили только одно smtp-подключение для обеих
тестовых функций вместо двух.

Если вы предпочитаете создавать одно smtp-подключение на сессию,
можно просто задать параметру ``scope`` значение ``session``:

.. code-block:: python

    @pytest.fixture(scope="session")
    def smtp_connection():
        # the returned fixture value will be shared for
        # all tests needing it
        ...

Соответственно, применив область действия ``class``, получим один вызов фикстуры для класса.

.. note::

    ``pytest`` кэширует только один экземпляр фикстуры одновременно.
    Это значит, что при использовании параметризованных фикстур ``pytest``
    может вызывать фикстуру для заданного уровня больше одного раза.

Область действия ``package`` (экспериментальная возможность)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

В ``pytest 3.7`` была введена область действия ``package``. Фикстура с
такой областью действия работает, пока не будет выполнен последний тест
*пакета (package)*.

.. warning::
    Эта возможность рассматривается как **экспериментальная** и в последующих
    версиях может быть удалена, если при ее использовании будут обнаружены
    подводные камни или серьезные проблемы.
    Поэтому, пожалуйста,  используйте ее с осторожностью и не забывайте сообщать
    об обнаруженных проблемах.

.. _dynamic scope:

Динамическая область действия
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


В некоторых случаях можно изменять область действия фикстуры без изменения кода.
Чтобы этого добиться, сделайте фикстуру *вызываемым объектом* (callable).
Этот объект должен возвращать строку с допустимым значением области действия,
и выполнен он будет только один раз - во время определения фикстуры.
Такой объект вызывается с двумя ключевыми параметрами: строкой ``fixture_name``
и конфигурируемым объектом ``config``.

В ситуациях, когда требуется значительное время для инициализации фикстуры
(например, при создании процессов docker-контейнеров), такая возможность может быть очень полезна.
Например, можно использовать опцию командной строки для определения
области действия процессов docker-контейнеров в разных виртуальных средах:

.. code-block:: python

    def determine_scope(fixture_name, config):
        if config.getoption("--keep-containers", None):
            return "session"
        return "function"


    @pytest.fixture(scope=determine_scope)
    def docker_container():
        yield spawn_container()



Порядок создания фикстур
----------------------------------------------------

При запросе фикстуры функцией сначала инициализиурются фикстуры с самой широкой
областью действия  - ``session`` и ``module``, а затем - фикстуры более низкого уровня с
областями ``class`` или ``function``. В рамках одной тестовой функции порядок создания
фикстур с одинаковой областью действия зависит от очередности вызова
этих фикстур и установленных между ними зависимостей. При этом фикстуры с параметром
``autouse = True`` инициализируются прежде явно объявленных фикстур того же уровня.

Рассмотрим следующий код:

.. literalinclude:: example/fixtures/test_fixtures_order.py

Фикстуры, запрошенные функцией ``test_order``, будут инициализированы в следующем порядке:

1. ``s1``: фикстура с самой широкой областью действия (``session``).
2. ``m1``: фикстура второго уровня (``module``).
3. ``a1``: фикстура с областью действия ``function`` (``function-scoped`` fixture) и параметром ``autouse = True``:
   экземпляр этой фикстуры будет создан до создания остальных ``function-scoped`` фикстур.
4. ``f3``: ``function-scoped`` фикстура, которую запрашивает функция ``f1``:
   ее нужно создать в момент запроса
5. ``f1``: первая ``function-scoped`` фикстура в списке аргументов функции ``test_order``.
6. ``f2``: последняя ``function-scoped`` фикстура в списке аргументов функции ``test_order``.

.. _`finalization`:

Финализаторы в фикстуре / выполнение завершающего кода
-------------------------------------------------------------

``pytest`` поддерживает выполнение фикстурами специфического завершающего кода
при выходе из области действия. Если вы используете оператор ``yield`` вместо ``return``,
то весь код после ``yield`` выполняет роль "уборщика":

.. code-block:: python

    # content of conftest.py

    import smtplib
    import pytest


    @pytest.fixture(scope="module")
    def smtp_connection():
        smtp_connection = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)
        yield smtp_connection  # возвращает значение фикстуры
        print("teardown smtp")
        smtp_connection.close()

Операторы ``print`` и ``smtp.close()`` будут выполнены после завершения последнего теста модуля
независимо от того, было ли вызвано исключение или нет.

Давайте запустим:

.. code-block:: pytest

    $ pytest -s -q --tb=no
    FFteardown smtp

    ========================= short test summary info ==========================
    FAILED test_module.py::test_ehlo - assert 0
    FAILED test_module.py::test_noop - assert 0
    2 failed in 0.12s

Мы видим, что подключение ``smtp_connection`` было закрыто после выполнения
двух тестов. Однако если вы зададите для фикстуры
область действия ``function``, то установка и разрыв соединения
будут производиться для каждого запущенного теста.
В любом случае, нет нужды обрабатывать инициализацию и демонтаж
соединения в самом модуле.

Обратите внимание, что можно использовать синтаксис ``yield`` с оператором  ``with``:

.. code-block:: python

    # content of test_yield2.py

    import smtplib
    import pytest


    @pytest.fixture(scope="module")
    def smtp_connection():
        with smtplib.SMTP("smtp.gmail.com", 587, timeout=5) as smtp_connection:
            yield smtp_connection  # возвращает значение фикстуры


После выполнения теста соединение ``smtp_connection`` будет разорвано, поскольку
объект ``smtp_connection`` автоматически закрывается после завершения
выполнения оператора ``with``.

Использование финализатора менеджера контекста ``contextlib.ExitStack()`` гарантирует
корректное закрытие соединений, вне зависимости от того, вызвала ли *установочная*
часть кода фикстуры исключение. Это удобно, поскольку позволяет корректно очищать все
ресурсы, созданные фикстурой, даже если один из них не удастся создать или получить:

.. code-block:: python

    # content of test_yield3.py

    import contextlib

    import pytest


    @contextlib.contextmanager
    def connect(port):
        ...  # устанавливаем соединение
        yield
        ...  # разрываем соединение


    @pytest.fixture
    def equipments():
        with contextlib.ExitStack() as stack:
            yield [stack.enter_context(connect(port)) for port in ("C1", "C3", "C28")]

Если в приведенном примере попытка установить соединение ``"C28"`` будет неудачной,
``"C1"`` и ``"C3"`` все равно будут корректно разорваны.

Обратите внимание: если исключение было вызвано во время выполнения *установочной* части
(до оператора ``yield``), *завершающий* код (после ``yield``) выполнен не будет.

Альтернативным способом добиться выполнения *завершающего* кода является
использование метода ``addfinalizer`` объекта `request-context`_ для
регистрации финализатора.

Вот пример использования ``addfinalizer`` для разрыва соединения в фикстуре ``smtp_connection``:

.. code-block:: python

    # content of conftest.py
    import smtplib
    import pytest


    @pytest.fixture(scope="module")
    def smtp_connection(request):
        smtp_connection = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

        def fin():
            print("teardown smtp_connection")
            smtp_connection.close()

        request.addfinalizer(fin)
        return smtp_connection  # возвращает значение фикстуры


А вот пример фикстуры ``equipments`` с использованием ``addfinalizer``:

.. code-block:: python

    # content of test_yield3.py

    import contextlib
    import functools

    import pytest


    @contextlib.contextmanager
    def connect(port):
        ...  # устанавливаем соединение
        yield
        ...  # разрываем соединение


    @pytest.fixture
    def equipments(request):
        r = []
        for port in ("C1", "C3", "C28"):
            cm = connect(port)
            equip = cm.__enter__()
            request.addfinalizer(functools.partial(cm.__exit__, None, None, None))
            r.append(equip)
        return r


Оба метода -  ``yield`` и ``addfinalizer`` - работают похоже, выполняя свою часть кода по завершении тестов.
Конечно, если исключение будет вызвано **до** инициализации финализатора, ее код
выполняться не будет.


.. _`request-context`:

Фикстуры могут анализировать запрашивающий контекст
-------------------------------------------------------------

Фикстура может принимать объект ``request`` для анализа контекста
запрашивающей тестовой функции, класса или модуля.
В продолжением предыдущего примера, давайте прочтем URL сервера
из тестового модуля, который использует нашу фикстуру.

.. code-block:: python

    # content of conftest.py
    import pytest
    import smtplib


    @pytest.fixture(scope="module")
    def smtp_connection(request):
        server = getattr(request.module, "smtpserver", "smtp.gmail.com")
        smtp_connection = smtplib.SMTP(server, 587, timeout=5)
        yield smtp_connection
        print("finalizing {} ({})".format(smtp_connection, server))
        smtp_connection.close()

Здесь мы используем параметр ``request.module`` чтобы получить
переменную ``smtpserver`` из модуля. Если мы просто запустим
``pytest``, ничего особо не изменится:

.. code-block:: pytest

    $ pytest -s -q --tb=no
    FFfinalizing <smtplib.SMTP object at 0xdeadbeef> (smtp.gmail.com)

    ========================= short test summary info ==========================
    FAILED test_module.py::test_ehlo - assert 0
    FAILED test_module.py::test_noop - assert 0
    2 failed in 0.12s

Давайте быстренько создадим еще один тестовый модуль,
который задает URL сервера в простарнстве имен модуля:

.. code-block:: python

    # content of test_anothersmtp.py

    smtpserver = "mail.python.org"  # будет прочитан фикстурой smtp_connection


    def test_showhelo(smtp_connection):
        assert 0, smtp_connection.helo()

Запустим:

.. code-block:: pytest

    $ pytest -qq --tb=short test_anothersmtp.py
    F                                                                    [100%]
    ================================= FAILURES =================================
    ______________________________ test_showhelo _______________________________
    test_anothersmtp.py:6: in test_showhelo
        assert 0, smtp_connection.helo()
    E   AssertionError: (250, b'mail.python.org')
    E   assert 0
    ------------------------- Captured stdout teardown -------------------------
    finalizing <smtplib.SMTP object at 0xdeadbeef> (mail.python.org)

Вуаля! Фикстура ``smtp_connection`` взяла имя нашего почтового сервера
из пространства имен использующего ее модуля.

.. _`fixture-factory`:

Фикстура как фабрика данных
--------------------------------------

Шаблон "фабрика-фикстура" может помочь в ситуациях, когда результат,
возвращаемый фикстурой используется много раз в отдельном тесте.
Суть в том, что вместо того, чтобы напрямую возвращать данные,
фикстура возвращает функцию, которая генерирует данные.
И затем эта функция может быть неоднократно вызвана в тесте.

Если нужно, фабрики-фикстуры могут принимать параметры:

.. code-block:: python

    @pytest.fixture
    def make_customer_record():
        def _make_customer_record(name):
            return {"name": name, "orders": []}

        return _make_customer_record


    def test_customer_records(make_customer_record):
        customer_1 = make_customer_record("Lisa")
        customer_2 = make_customer_record("Mike")
        customer_3 = make_customer_record("Meredith")

Если нужно управлять данными, созданными фабриками, фикстура позаботится и об этом
(в нашем случае, будут очищены созданные записи):

.. code-block:: python

    @pytest.fixture
    def make_customer_record():

        created_records = []

        def _make_customer_record(name):
            record = models.Customer(name=name, orders=[])
            created_records.append(record)
            return record

        yield _make_customer_record

        for record in created_records:
            record.destroy()


    def test_customer_records(make_customer_record):
        customer_1 = make_customer_record("Lisa")
        customer_2 = make_customer_record("Mike")
        customer_3 = make_customer_record("Meredith")


.. _`fixture-parametrize`:

Параметризация фикстур
-----------------------------------------------------------------

Фикстуры могут быть параметризованы, если их нужно вызывать неоднократно,
выполняя несколько одинаковых, использующих эти фикстуры, тестов.
Обычно повторно запускаемые тестовые функции не зависят друг от друга.
И в этом случае параметризация фикстур помогает писать исчерпывающие
функциональные тесты для компонентов, которые сами по себе
могут быть сконфигурированы разными способами.

Расширяя предыдущий пример, мы можем указать фикстуре, что нужно
создавать два объекта ``smtp_connection``, тем самым заставив все тесты,
ее использующие, выполняться дважды:

.. code-block:: python

    # content of conftest.py
    import pytest
    import smtplib


    @pytest.fixture(scope="module", params=["smtp.gmail.com", "mail.python.org"])
    def smtp_connection(request):
        smtp_connection = smtplib.SMTP(request.param, 587, timeout=5)
        yield smtp_connection
        print("finalizing {}".format(smtp_connection))
        smtp_connection.close()

Главным внесенным изменением является объявление списка параметров
``params`` в декораторе ``@pytest.fixture <_pytest.python.fixture>``,
для каждого из которых фикстура будет выполняться и получать значение
``request.param``. Менять код в тестовой функции не нужно.
Давайте запустим на ш тестовый модуль "test_module.py":

.. code-block:: pytest

    $ pytest -q test_module.py
    FFFF                                                                 [100%]
    ================================= FAILURES =================================
    ________________________ test_ehlo[smtp.gmail.com] _________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
            assert b"smtp.gmail.com" in msg
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:7: AssertionError
    ________________________ test_noop[smtp.gmail.com] _________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_noop(smtp_connection):
            response, msg = smtp_connection.noop()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:13: AssertionError
    ________________________ test_ehlo[mail.python.org] ________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
    >       assert b"smtp.gmail.com" in msg
    E       AssertionError: assert b'smtp.gmail.com' in b'mail.python.org\nPIPELINING\nSIZE 51200000\nETRN\nSTARTTLS\nAUTH DIGEST-MD5 NTLM CRAM-MD5\nENHANCEDSTATUSCODES\n8BITMIME\nDSN\nSMTPUTF8\nCHUNKING'

    test_module.py:6: AssertionError
    -------------------------- Captured stdout setup ---------------------------
    finalizing <smtplib.SMTP object at 0xdeadbeef>
    ________________________ test_noop[mail.python.org] ________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_noop(smtp_connection):
            response, msg = smtp_connection.noop()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:13: AssertionError
    ------------------------- Captured stdout teardown -------------------------
    finalizing <smtplib.SMTP object at 0xdeadbeef>
    4 failed in 0.12s

Мы видим, что каждая из пары наших тестовых функций была выполнена дважды,
сначала с одним, потом с другим объектом ``smtp_connection``.
Обратите внимание, что с соединением ``mail.python.org``
второй тест упал на функции ``test_ehlo``, поскольку мы ожидали
найти в сообщении строку с другим названием сервера.

Для каждого значения параметра параметризованной фикстуры ``pytest``
сгенерирует ID - строку, которая его идентифицирует (например, строки
``test_ehlo[smtp.gmail.com]`` и ``test_ehlo[mail.python.org]``
для нашего кода). Можно использовать эти ID вместе с опцией ``-k``
для выбора отдельных вариантов теста для запуска. По этим же ID
можно понять, какой именно параметр использовался в упавшем тесте.
Если вы запустите ``pytest`` с опцией ``--collect-only``, то сможете
увидеть все сгенерированные ID.

Числа, строки, логические значения и значение None имеют свои строковые
представления, которые используются в ID теста. Для остальных объектов
``pytest`` создает строку, основываясь на имени аргумента.
С помощью ключевого слова ``ids`` можно самостоятельно определить строку,
которая будет использоваться в ID теста для определенного значения фикстуры:

.. code-block:: python

   # content of test_ids.py
   import pytest


   @pytest.fixture(params=[0, 1], ids=["spam", "ham"])
   def a(request):
       return request.param


   def test_a(a):
       pass


   def idfn(fixture_value):
       if fixture_value == 0:
           return "eggs"
       else:
           return None


   @pytest.fixture(params=[0, 1], ids=idfn)
   def b(request):
       return request.param


   def test_b(b):
       pass

Пример выше показывает, что ``ids`` можно определять как списком строк,
так и функцией, которая будет вызвана со значением фикстуры и
вернет строку. В последнем случае, если функция вернет ``None``,
то ``pytest`` сгенерирует ID автоматически.

При запуске тестов из примеров выше будут сгенерированы следующие ID:

.. code-block:: pytest

   $ pytest --collect-only
   =========================== test session starts ============================
   platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
   cachedir: $PYTHON_PREFIX/.pytest_cache
   rootdir: $REGENDOC_TMPDIR
   collected 10 items
   <Module test_anothersmtp.py>
     <Function test_showhelo[smtp.gmail.com]>
     <Function test_showhelo[mail.python.org]>
   <Module test_ids.py>
     <Function test_a[spam]>
     <Function test_a[ham]>
     <Function test_b[eggs]>
     <Function test_b[1]>
   <Module test_module.py>
     <Function test_ehlo[smtp.gmail.com]>
     <Function test_noop[smtp.gmail.com]>
     <Function test_ehlo[mail.python.org]>
     <Function test_noop[mail.python.org]>

   ========================== no tests ran in 0.12s ===========================

.. _`fixture-parametrize-marks`:

Использование маркировки с параметризованными фикстурами
-----------------------------------------------------------

`pytest.param <https://docs.pytest.org/en/latest/_modules/_pytest/mark.html#param>`_
можно использовать для маркировки значений параметров параметризованных фикстур,
точно так же, как и с :ref:`@pytest.mark.parametrize <@pytest.mark.parametrize>`.

Пример:

.. code-block:: python

    # content of test_fixture_marks.py
    import pytest


    @pytest.fixture(params=[0, 1, pytest.param(2, marks=pytest.mark.skip)])
    def data_set(request):
        return request.param


    def test_data(data_set):
        pass

При выполнении этого теста вызов ``data_set`` со значением ``2`` будет пропущен
(*skipped*).

.. code-block:: pytest

    $ pytest test_fixture_marks.py -v
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 3 items

    test_fixture_marks.py::test_data[0] PASSED                           [ 33%]
    test_fixture_marks.py::test_data[1] PASSED                           [ 66%]
    test_fixture_marks.py::test_data[2] SKIPPED                          [100%]

    ======================= 2 passed, 1 skipped in 0.12s =======================

.. _`interdependent fixtures`:

Модальность: использование фикстур фикстурами
----------------------------------------------------------

Фикстуры могут использоваться не только тестовыми функциями, но и другими фикстурами.
Это помогает делать ваши тесты модальными и дает возможность повторного
использования фреймворк-зависимых фикстур во множестве проектов.
Чтобы продемонстрировать это, давайте расширим предыдущий пример
и инициализируем объект ``app``, в котором будем использовать
уже объявленный ресурс ``smtp_connection``:


.. code-block:: python

    # content of test_appsetup.py

    import pytest


    class App:
        def __init__(self, smtp_connection):
            self.smtp_connection = smtp_connection


    @pytest.fixture(scope="module")
    def app(smtp_connection):
        return App(smtp_connection)


    def test_smtp_connection_exists(app):
        assert app.smtp_connection

Здесь мы объявляем фикстуру ``app``, которая принимает ранее объявленную
фикстуру ``smtp_connection`` и создает с ее помощью объект ``App``:

.. code-block:: pytest

    $ pytest -v test_appsetup.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 2 items

    test_appsetup.py::test_smtp_connection_exists[smtp.gmail.com] PASSED [ 50%]
    test_appsetup.py::test_smtp_connection_exists[mail.python.org] PASSED [100%]

    ============================ 2 passed in 0.12s =============================

Поскольку фикстура ``smtp_connection`` параметризована, тест запустится дважды
с разными экземплярами приложения ``App`` и соответствующими им серверами smtp.
Нам не нужно параметризовать ``smtp_connection`` в фикстуре ``app``,
так как ``pytest`` самостоятельно анализирует граф зависимостей.

Обратите внимание, что фикстура ``app`` имеет уровень модуля и
использует фикстуру ``smtp_connection`` того же уровня. Пример
будет работать и в том случае, если ``smtp_connection`` будет кэшироваться
на уровне сессии: для фикстур нормально использовать другие фикстуры
с более обширной областью действия, но не наоборот - фикстура
уровня сессии не может полноценно использовать фикстуру уровня модуля.

.. _`automatic per-resource grouping`:

Автоматическая группировка тестов экземплярами фикстур
----------------------------------------------------------

.. regendoc: wipe

``pytest`` минимизирует число активных фикстур во время выполнения теста.
Если у вас есть параметризованная фикстура, то каждый экземпляр теста,
ее использующий, сначала запускается с очередным параметром, а затем
вызывает финализатор прежде, чем слудующий объект фикстуры будет
инициализирован. Это, как и другие предоставляемые возможности,
облегчает тестирование приложений, которые создают и используют
глобальные состояния.

Следующий пример использует две параметризованные фикстуры,
одна из которых имеет уровень модуля, и обе функции вызывают
``print``, чтобы продемонстрировать поток инициализации/завершения.

.. code-block:: python

    # content of test_module.py
    import pytest


    @pytest.fixture(scope="module", params=["mod1", "mod2"])
    def modarg(request):
        param = request.param
        print("  SETUP modarg", param)
        yield param
        print("  TEARDOWN modarg", param)


    @pytest.fixture(scope="function", params=[1, 2])
    def otherarg(request):
        param = request.param
        print("  SETUP otherarg", param)
        yield param
        print("  TEARDOWN otherarg", param)


    def test_0(otherarg):
        print("  RUN test0 with otherarg", otherarg)


    def test_1(modarg):
        print("  RUN test1 with modarg", modarg)


    def test_2(otherarg, modarg):
        print("  RUN test2 with otherarg {} and modarg {}".format(otherarg, modarg))


Давайте запустим код в режиме подробных сообщений (с опцией ``-v``) и посмотрим на вывод:

.. code-block:: pytest

    $ pytest -v -s test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 8 items

    test_module.py::test_0[1]   SETUP otherarg 1
      RUN test0 with otherarg 1
    PASSED  TEARDOWN otherarg 1

    test_module.py::test_0[2]   SETUP otherarg 2
      RUN test0 with otherarg 2
    PASSED  TEARDOWN otherarg 2

    test_module.py::test_1[mod1]   SETUP modarg mod1
      RUN test1 with modarg mod1
    PASSED
    test_module.py::test_2[mod1-1]   SETUP otherarg 1
      RUN test2 with otherarg 1 and modarg mod1
    PASSED  TEARDOWN otherarg 1

    test_module.py::test_2[mod1-2]   SETUP otherarg 2
      RUN test2 with otherarg 2 and modarg mod1
    PASSED  TEARDOWN otherarg 2
    test_module.py::test_1[mod2]   TEARDOWN modarg mod1
      SETUP modarg mod2
      RUN test1 with modarg mod2
    PASSED

    test_module.py::test_2[mod2-1]   SETUP otherarg 1
      RUN test2 with otherarg 1 and modarg mod2
    PASSED  TEARDOWN otherarg 1

    test_module.py::test_2[mod2-2]   SETUP otherarg 2
      RUN test2 with otherarg 2 and modarg mod2
    PASSED  TEARDOWN otherarg 2
      TEARDOWN modarg mod2


    ============================ 8 passed in 0.12s =============================

Вы можете увидеть, что параметризация фикстуры ``modarg`` на уровне модуля
привела к выполнению тестов в порядке, позволяющем минимизировать "активные"
ресурсы. Финализатор фикстуры с параметром  ``mod1`` был вызван до
инициализация фикстуры с параметром ``mod2``.

Заметьте, что ``test_0`` полностью независим и поэтому был завершен первым.
``test_1`` был выполнен с параметром ``mod1``, потом с тем же параметром
был запущен ``test_2``, после этого - ``test_1`` с параметром ``mod2``,
последним был запущен ``test_2``с параметром ``mod2``.

При этом параметризованная фикстура ``otherarg`` с уровнем функции
инициализировалась и демонтировалась для каждого теста, который ее использовал.


.. _`usefixtures`:

Использование фикстур в классах, модулях и проектах
----------------------------------------------------------------------

.. regendoc:wipe

Иногда тестовым функциям не нужно напрямую обращаться к объекту фикстуры.
Например, для тестов может потребоваться пустой рабочий каталог,
но нам не важно, какой именно каталог это будет.
Вот здесь можно посмотреть, как для этого использовать встроенную
фикстуру  ``pytest`` `tempfile <http://docs.python.org/library/tempfile.html>`_.
Мы же опишем создание такой фикстуры в файле ``conftest.py``:

.. code-block:: python

    # content of conftest.py

    import os
    import shutil
    import tempfile

    import pytest


    @pytest.fixture
    def cleandir():
        old_cwd = os.getcwd()
        newpath = tempfile.mkdtemp()
        os.chdir(newpath)
        yield
        os.chdir(old_cwd)
        shutil.rmtree(newpath)

Затем объявим ее использование в тестовом модуле с помощью декортатора ``usefixtures``:

.. code-block:: python

    # content of test_setenv.py
    import os
    import pytest


    @pytest.mark.usefixtures("cleandir")
    class TestDirectoryInit:
        def test_cwd_starts_empty(self):
            assert os.listdir(os.getcwd()) == []
            with open("myfile", "w") as f:
                f.write("hello")

        def test_cwd_again_starts_empty(self):
            assert os.listdir(os.getcwd()) == []


Фикстура ``cleandir`` будет инициализироваться
для выполнения каждого тестового метода.
Давайте запустим код и убедимся, что наша фикстура
инициализируется и тесты проходят:

.. code-block:: pytest

    $ pytest -q
    ..                                                                   [100%]
    2 passed in 0.12s

Также можно "прицепить" несколько фикстур сразу

.. code-block:: python

    @pytest.mark.usefixtures("cleandir", "anotherfixture")
    def test():
        ...

и определять использование фикстуры на уровне модуля, используя
возможности механизма маркировки:

.. code-block:: python

    pytestmark = pytest.mark.usefixtures("cleandir")

Обратите внимание, что переменная **должна** называться именно ``pytestmark``;
если вы назовете ее, например, ``foomark``, ваша фикстура инициализироваться не будет.

Можно также затребовать вашу фикстуру для всех тестов проекта, указав
в "ini"-файле:

.. code-block:: ini

    # content of pytest.ini
    [pytest]
    usefixtures = cleandir


.. warning::

    Внимание! Такая маркировка неэффективна для **функций-фикстур**!
    Нижеприведенный код **не будет работать так, как должен**:

    .. code-block:: python

        @pytest.mark.usefixtures("my_other_fixture")
        @pytest.fixture
        def my_fixture_that_sadly_wont_use_my_other_fixture():
            ...

    На данный момент подобный код не генерирует ошибок или предупреждений,
    но это планируется исправить, см. `#3664 <https://github.com/pytest-dev/pytest/issues/3664>`_.

.. _`autouse`:
.. _`autouse fixtures`:


Фикстуры ``autouse`` (автоматическое использование фикстур)
----------------------------------------------------------------------

.. regendoc:wipe

Иногда хочется, чтобы фикстуры вызывались автоматически,
без явного указания их в качестве аргумента и без использования
декоратора `usefixtures`_ . Предположим, у нас есть фикстура,
имитирующая базу данных с архитектурой "begin/rollback/commit" и мы хотим
автоматически обернуть каждый тестовый метод транзакцией и откатом
к начальному состоянию. Вот макет реализации этой идеи:

.. code-block:: python

    # content of test_db_transact.py

    import pytest


    class DB:
        def __init__(self):
            self.intransaction = []

        def begin(self, name):
            self.intransaction.append(name)

        def rollback(self):
            self.intransaction.pop()


    @pytest.fixture(scope="module")
    def db():
        return DB()


    class TestClass:
        @pytest.fixture(autouse=True)
        def transact(self, request, db):
            db.begin(request.function.__name__)
            yield
            db.rollback()

        def test_method1(self, db):
            assert db.intransaction == ["test_method1"]

        def test_method2(self, db):
            assert db.intransaction == ["test_method2"]

Фикстура ``transact`` уровня класса промаркирована *autouse=true*,
и это означает, что все тестовые методы класса будут использовать
эту фикстуру без необходимости указывать ее в сигнатуре тестовой функции
или использовать на уровне класса декоратор ``usefixtures``.

Запустив, получим два успешно пройденных теста:

.. code-block:: pytest

    $ pytest -q
    ..                                                                   [100%]
    2 passed in 0.12s

Вот как работают фикстуры на разных уровнях:

- "autouse"-фикстуры соблюдают область действия, определенную с помощью
  параметра ``scope=``: если для фикстуры установлен уровень
  ``scope='session'`` - она будет инициализирована только один раз,
  при этом неважно, где она определена. ``scope='class'`` означает
  инициализацию один раз для класса и т. д.

- если "autouse"-фикстура определена в тестовом модуле,
  то ее будут автоматически использовать все тесты модуля.

- если "autouse"-фикстура определена в файле ``conftest.py``,
  то вызывать фикстуру будут все тесты во всех тестовых модулях
  соответствующей директории.

- и, наконец (**пожалуйста, используйте эту возможность с осторожностью**):
  если вы определяете "autouse"-фикстуру в плагине, она будет вызываться
  для всех тестов во всех проектах, где установлен этот плагин.
  Это может быть полезно, если фикстура работает только при
  определенных настройках (указанных, например, в "ini"-файлах).
  Такая глобальная фикстура всегда  должна быстро определять,
  нужно ли ей что-либо делать, чтобы избежать дорогостоящего импорта и вычислений.

Что касается приведенной выше фикстуры ``transact``, вы можете захотеть,
чтобы она была доступна в вашем проекте, не будучи при этом активной.
Классический способ сделать это - поместить ее в файл ``conftest.py``,
**не применяяя "autouse"**:

.. code-block:: python

    # content of conftest.py
    @pytest.fixture
    def transact(request, db):
        db.begin()
        yield
        db.rollback()


И затем, если понадобится, создать тестовый класс, объявив ее использование:

.. code-block:: python

    @pytest.mark.usefixtures("transact")
    class TestClass:
        def test_method1(self):
            ...

В этом случае  фикстуру ``transact`` будут использовать все тестовые методы класса ``TestClass``;
остальные тесты не будут к ней обращаться, пока вы так же явно не укажете необходимость ее использования.

Переопределение фикстур разного уровня
-----------------------------------------

В больших тестовых наборах вам может понадобиться переопределять глобальные (``global``)
или корневые (``root``) фкстуры локальными (``locally``), сохраняя тестовый код
читабельным и поддерживаемым.

Переопределение фикстур на уровне каталога (conftest.py)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Допустим, наш проект имеет такую файловую структуру:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture
            def username():
                return 'username'

        test_something.py
            # content of tests/test_something.py
            def test_username(username):
                assert username == 'username'

        subfolder/
            __init__.py

            conftest.py
                # content of tests/subfolder/conftest.py
                import pytest

                @pytest.fixture
                def username(username):
                    return 'overridden-' + username

            test_something.py
                # content of tests/subfolder/test_something.py
                def test_username(username):
                    assert username == 'overridden-username'

Как видите, фикстура с одним и тем же именем может быть переопределена на уровне
конкретного подкаталога. При этом "базовая" фикстура доступна в переопределенной
(см. пример выше).

Переопределение фикстуры на уровне тестового модуля
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Рассмотрим еще одну файловую структуру:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture
            def username():
                return 'username'

        test_something.py
            # content of tests/test_something.py
            import pytest

            @pytest.fixture
            def username(username):
                return 'overridden-' + username

            def test_username(username):
                assert username == 'overridden-username'

        test_something_else.py
            # content of tests/test_something_else.py
            import pytest

            @pytest.fixture
            def username(username):
                return 'overridden-else-' + username

            def test_username(username):
                assert username == 'overridden-else-username'

Пример показывает, как можно переопределить фикстуру с одним и тем же именем
в конкретном тестовом модуле.

Переопределению фикстуры с помощью параметризации
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Возьмем следующую структуру тестов:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture
            def username():
                return 'username'

            @pytest.fixture
            def other_username(username):
                return 'other-' + username

        test_something.py
            # content of tests/test_something.py
            import pytest

            @pytest.mark.parametrize('username', ['directly-overridden-username'])
            def test_username(username):
                assert username == 'directly-overridden-username'

            @pytest.mark.parametrize('username', ['directly-overridden-username-other'])
            def test_username_other(other_username):
                assert other_username == 'other-directly-overridden-username-other'

В этом варианте значение фикстуры переопределяется значением параметра.
Обратите внимание, что значение фикстуры может быть переопределено таким способом,
деже если тесты не используют фикстуру напрямую (т. е. она не упоминается в самих функциях).

Замена параметризованной фикстуры непараметризованной и наоборот
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Расммотрим такую структуру:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture(params=['one', 'two', 'three'])
            def parametrized_username(request):
                return request.param

            @pytest.fixture
            def non_parametrized_username(request):
                return 'username'

        test_something.py
            # content of tests/test_something.py
            import pytest

            @pytest.fixture
            def parametrized_username():
                return 'overridden-username'

            @pytest.fixture(params=['one', 'two', 'three'])
            def non_parametrized_username(request):
                return request.param

            def test_username(parametrized_username):
                assert parametrized_username == 'overridden-username'

            def test_parametrized_username(non_parametrized_username):
                assert non_parametrized_username in ['one', 'two', 'three']

        test_something_else.py
            # content of tests/test_something_else.py
            def test_username(parametrized_username):
                assert parametrized_username in ['one', 'two', 'three']

            def test_username(non_parametrized_username):
                assert non_parametrized_username == 'username'

Здесь параметризованная фикстура заменяется непараметризованной и наоборот
в рамках конкретного тестового модуля. То же самое можно проделывать
и для тестовых каталогов/подкаталогов.

