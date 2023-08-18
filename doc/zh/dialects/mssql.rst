.. _mssql_toplevel:

Microsoft SQL Server
====================

.. automodule:: sqlalchemy.dialects.mssql.base

SQL Server SQL Constructs
-------------------------

与所有SQLAlchemy方言一样，所有已知适用于SQL Server的大写类型都可以从顶级方言中导入，无论它们起源于 :mod：`sqlalchemy.types`或本地方言::

    from sqlalchemy.dialects.mssql import (
        BIGINT,
        BINARY,
        BIT,
        CHAR,
        DATE,
        DATETIME,
        DATETIME2,
        DATETIMEOFFSET,
        DECIMAL,
        DOUBLE_PRECISION,
        FLOAT,
        IMAGE,
        INTEGER,
        JSON,
        MONEY,
        NCHAR,
        NTEXT,
        NUMERIC,
        NVARCHAR,
        REAL,
        SMALLDATETIME,
        SMALLINT,
        SMALLMONEY,
        SQL_VARIANT,
        TEXT,
        TIME,
        TIMESTAMP,
        TINYINT,
        UNIQUEIDENTIFIER,
        VARBINARY,
        VARCHAR,
    )

特定于SQL Server,或具有特定于SQL Server的构造参数的类型如下：

.. 注意：使用 :noindex: 表示在首选模板中未重新定义类型，只是从sqltypes导入。这避免了sphinx构建中的警告

.. currentmodule:: sqlalchemy.dialects.mssql

.. autoclass:: BIT
   :members: __init__

.. autoclass:: CHAR
   :members: __init__
   :noindex:


.. autoclass:: DATETIME2
   :members: __init__


.. autoclass:: DATETIMEOFFSET
   :members: __init__

.. autoclass:: DOUBLE_PRECISION
   :members: __init__

.. autoclass:: IMAGE
   :members: __init__


.. autoclass:: JSON
   :members: __init__


.. autoclass:: MONEY
   :members: __init__


.. autoclass:: NCHAR
   :members: __init__
   :noindex:


.. autoclass:: NTEXT
   :members: __init__


.. autoclass:: NVARCHAR
   :members: __init__
   :noindex:

.. autoclass:: REAL
   :members: __init__

.. autoclass:: ROWVERSION
   :members: __init__

.. autoclass:: SMALLDATETIME
   :members: __init__


.. autoclass:: SMALLMONEY
   :members: __init__


.. autoclass:: SQL_VARIANT
   :members: __init__


.. autoclass:: TEXT
   :members: __init__
   :noindex:

.. autoclass:: TIME
   :members: __init__


.. autoclass:: TIMESTAMP
   :members: __init__

.. autoclass:: TINYINT
   :members: __init__


.. autoclass:: UNIQUEIDENTIFIER
   :members: __init__


.. autoclass:: VARBINARY
   :members: __init__
   :noindex:

.. autoclass:: VARCHAR
   :members: __init__
   :noindex:


.. autoclass:: XML
   :members: __init__


PyODBC
------
.. automodule:: sqlalchemy.dialects.mssql.pyodbc

pymssql
-------
.. automodule:: sqlalchemy.dialects.mssql.pymssql