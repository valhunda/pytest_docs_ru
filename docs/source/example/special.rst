Фикстура уровня сессии для поиска во всех собираемых тестах
----------------------------------------------------------------

Фикстура уровня сессии имеет доступ ко всем собираемым тестам.
Ниже - пример фикстуры, которая просматривает все собираемые тесты,
ищет тестовый класс, содержащий метод ``callme`` и вызывает его:

.. code-block:: python

    # content of conftest.py

    import pytest


    @pytest.fixture(scope="session", autouse=True)
    def callattr_ahead_of_alltests(request):
        print("callattr_ahead_of_alltests called")
        seen = {None}
        session = request.node
        for item in session.items:
            cls = item.getparent(pytest.Class)
            if cls not in seen:
                if hasattr(cls.obj, "callme"):
                    cls.obj.callme()
                seen.add(cls)

Теперь в тестовых классах можно определить метод ``callme``,
который будет вызываться перед запуском любых тестов:

.. code-block:: python

    # content of test_module.py


    class TestHello:
        @classmethod
        def callme(cls):
            print("callme called!")

        def test_method1(self):
            print("test_method1 called")

        def test_method2(self):
            print("test_method1 called")


    class TestOther:
        @classmethod
        def callme(cls):
            print("callme other called")

        def test_other(self):
            print("test other")


    # works with unittest as well ...
    import unittest


    class SomeTest(unittest.TestCase):
        @classmethod
        def callme(self):
            print("SomeTest callme called")

        def test_unit1(self):
            print("test_unit1 method called")

Заапустим этот код без перехвата вывода:

.. code-block:: pytest

    $ pytest -q -s test_module.py
    callattr_ahead_of_alltests called
    callme called!
    callme other called
    SomeTest callme called
    test_method1 called
    .test_method1 called
    .test other
    .test_unit1 method called
    .
    4 passed in 0.12s
