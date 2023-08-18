.. _oracle_toplevel:

Oracle
======

.. automodule:: sqlalchemy.dialects.oracle.base

Oracle 数据类型
-----------------

和所有的 SQLAlchemy 方言一样，所有已知可用于 Oracle 的大写类型都可以从 dialect 的顶层导入，无论是来自  :mod:`sqlalchemy.types`  还是来自本地 dialect。

::

    from sqlalchemy.dialects.oracle import (
        BFILE,
        BLOB,
        CHAR,
        CLOB,
        DATE,
        DOUBLE_PRECISION,
        FLOAT,
        INTERVAL,
        LONG,
        NCLOB,
        NCHAR,
        NUMBER,
        NVARCHAR,
        NVARCHAR2,
        RAW,
        TIMESTAMP,
        VARCHAR,
        VARCHAR2,
    )

.. versionadded:: 1.2.19 将  :class:`_types.NCHAR`  添加到 Oracle 数据类型的列表中。

以下是特定于 Oracle 或具有 Oracle 特定构造参数的类型：

.. currentmodule:: sqlalchemy.dialects.oracle

.. autoclass:: BFILE
  :members: __init__

.. autoclass:: BINARY_DOUBLE
  :members: __init__

.. autoclass:: BINARY_FLOAT
  :members: __init__

.. autoclass:: DATE
   :members: __init__

.. autoclass:: FLOAT
   :members: __init__

.. autoclass:: INTERVAL
  :members: __init__

.. autoclass:: NCLOB
  :members: __init__

.. autoclass:: NVARCHAR2
   :members: __init__

.. autoclass:: NUMBER
   :members: __init__

.. autoclass:: LONG
  :members: __init__

.. autoclass:: RAW
  :members: __init__

.. autoclass:: ROWID
  :members: __init__

.. autoclass:: TIMESTAMP
  :members: __init__

.. _cx_oracle:

cx_Oracle
---------

.. automodule:: sqlalchemy.dialects.oracle.cx_oracle

.. _oracledb:

Python-Oracledb
---------------

.. automodule:: sqlalchemy.dialects.oracle.oracledb