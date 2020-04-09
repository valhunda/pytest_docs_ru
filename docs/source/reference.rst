.. _`reference`:
.. _`fixtures-api`:
.. _`pytest.fixture-api`:

Справка по API
================

Раздел находится в разработке.

См. `API Reference <https://docs.pytest.org/en/latest/reference.html>`_

.. _`ini options ref`:
.. _`addopts ref`:

Опции конфигурации
---------------------

Раздел находится в разработке.

См. `API Reference <https://docs.pytest.org/en/latest/reference.html>`_


 Add the specified ``OPTS`` to the set of command line arguments as if they
   had been specified by the user. Example: if you have this ini file content:

   .. code-block:: ini

        # content of pytest.ini
        [pytest]
        addopts = --maxfail=2 -rf  # exit after 2 failures, report fail info

   issuing ``pytest test_hello.py`` actually means:

   .. code-block:: bash

        pytest --maxfail=2 -rf test_hello.py

   Default is to add no options.