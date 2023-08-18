.. _sqlite_toplevel:

SQLite
======

.. automodule:: sqlalchemy.dialects.sqlite.base

SQLite 数据类型
-----------------

与所有SQLAlchemy方言一样，所有大写类型，如果已知在SQLite中有效，则可以从顶级方言导入，无论它们是来自：mod：`sqlalchemy.types` 还是本地方言：

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

SQLite DML 语法
-------------------------

.. autofunction:: sqlalchemy.dialects.sqlite.insert

.. autoclass:: sqlalchemy.dialects.sqlite.Insert
  :members:

.. _pysqlite:

Pysqlite
--------

.. automodule:: sqlalchemy.dialects.sqlite.pysqlite

.. _aiosqlite:

Aiosqlite
---------

.. automodule:: sqlalchemy.dialects.sqlite.aiosqlite


.. _pysqlcipher:

Pysqlcipher
-----------

.. automodule:: sqlalchemy.dialects.sqlite.pysqlcipher