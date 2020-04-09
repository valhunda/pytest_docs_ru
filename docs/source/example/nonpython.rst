
.. _`non-python tests`:

Работа с тестами не на ``python``
====================================================

.. _`yaml plugin`:

Простой пример поиска тестов в файлах "yaml"
--------------------------------------------------------------

.. _`pytest-yamlwsgi`: http://bitbucket.org/aafshar/pytest-yamlwsgi/src/tip/pytest_yamlwsgi.py
.. _`PyYAML`: https://pypi.org/project/PyYAML/

Вот пример файла ``conftest.py`` (полученного из плагина `pytest-yamlwsgi`_).
Этот  ``conftest.py`` собирает файлы вида ``test*.yaml``
и выполняет их содержимое в виде тестов:


.. include:: nonpython/conftest.py
    :literal:

Можно создать простой тестовый файл:

.. include:: nonpython/test_simple.yaml
    :literal:

И если вы установите `PyYAML`_ или совместимый YAML-парсер, то сможете
запустить тесты:

.. code-block:: pytest

    nonpython $ pytest test_simple.yaml
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR/nonpython
    collected 2 items

    test_simple.yaml F.                                                  [100%]

    ================================= FAILURES =================================
    ______________________________ usecase: hello ______________________________
    usecase execution failed
       spec failed: 'some': 'other'
       no further details known at this point.
    ======================= 1 failed, 1 passed in 0.12s ========================


У нас получился один прошедший тест на проверку ``sub1: sub1`` и один упавший.
Очевидно, вы можете захотеть реализовать в ``conftest.py``
более интересную интерпретацию значений. Так можно легко
написать свой собственный "язык тестирования".

.. note::

    ``repr_failure(excinfo)`` вызывается для представления упавших тестов.
    Если вы сами будете генерировать узлы, то сможете возвращать
    любое строковое представление ошибки по вашему выбору. В отчете
    оно будет выделено красным.

``reportinfo()``  используется для представления расположения теста, а также
в режиме ``verbose``:

.. code-block:: pytest

    nonpython $ pytest -v
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR/nonpython
    collecting ... collected 2 items

    test_simple.yaml::hello FAILED                                       [ 50%]
    test_simple.yaml::ok PASSED                                          [100%]

    ================================= FAILURES =================================
    ______________________________ usecase: hello ______________________________
    usecase execution failed
       spec failed: 'some': 'other'
       no further details known at this point.
    ======================= 1 failed, 1 passed in 0.12s ========================


При разработке собственного поиска и выполнения тестов интересно
будет взглянуть на дерево собранных тестов:

.. code-block:: pytest

    nonpython $ pytest --collect-only
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR/nonpython
    collected 2 items
    <Package $REGENDOC_TMPDIR/nonpython>
      <YamlFile test_simple.yaml>
        <YamlItem hello>
        <YamlItem ok>

    ========================== no tests ran in 0.12s ===========================
