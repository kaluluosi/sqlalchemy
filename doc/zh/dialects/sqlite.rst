.. _sqlite_toplevel:

SQLite
======

.. automodule:: sqlalchemy.dialects.sqlite.base

SQLite 数据类型
-----------------

与所有SQLAlchemy语言方言一样，所有已知可用于SQLite的大写类型都可以从顶级方言导入，无论它们是源自 :mod:`sqlalchemy.types` 还是本地方言::

    from sqlalchemy.dialects.sqlite import (
        BLOB,
        BOOLEAN,
        CHAR,
        DATE,
        DATETIME,
        DECIMAL,
        FLOAT,
        INTEGER,
        NUMERIC,
        JSON,
        SMALLINT,
        TEXT,
        TIME,
        TIMESTAMP,
        VARCHAR,
    )

.. module:: sqlalchemy.dialects.sqlite

.. autoclass:: DATETIME
.. autoclass:: DATE
.. autoclass:: JSON
.. autoclass:: TIME

SQLite DML构造
-------------------------

.. autofunction:: sqlalchemy.dialects.sqlite.insert

.. autoclass:: sqlalchemy.dialects.sqlite.Insert
  :members:


Pysqlite
--------

.. automodule:: sqlalchemy.dialects.sqlite.pysqlite

Aiosqlite
---------

.. automodule:: sqlalchemy.dialects.sqlite.aiosqlite

Pysqlcipher
-----------

.. automodule:: sqlalchemy.dialects.sqlite.pysqlcipher