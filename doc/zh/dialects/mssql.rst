.. _mssql_toplevel:

Microsoft SQL Server
====================

.. automodule:: sqlalchemy.dialects.mssql.base

SQL Server SQL Constructs
-------------------------

与所有SQLAlchemy方言一样，所有已知在SQL服务器中有效的大写类型都可以从顶级方言中导入，无论它们起源于 :mod:`sqlalchemy.types` 还是本地方言::

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

SQL Server数据类型
-------------------

特定于SQL Server或具有SQL Server特定的构造参数的类型如下：

.. 注释：当使用 :noindex: 时，表示在方言模块中未重新定义的类型，仅从 sqltypes 导入。这避免了sphinx构建中的警告。

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