Oracle
======

.. automodule:: sqlalchemy.dialects.oracle.base

Oracle 数据类型
------------

与所有SQLAlchemy方言一样，所有已知在Oracle中有效的大写数据类型可以从顶层方言导入，
无论它们来自 :mod:`sqlalchemy.types` 还是本地方言::

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

.. versionadded:: 1.2.19 添加了:class:`_types.NCHAR` 数据类型到Oracle方言的导出列表中。

以下数据类型是特定于Oracle或具有Oracle特定构造参数的：

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

cx_Oracle
---------

.. automodule:: sqlalchemy.dialects.oracle.cx_oracle

python-oracledb
---------------

.. automodule:: sqlalchemy.dialects.oracle.oracledb