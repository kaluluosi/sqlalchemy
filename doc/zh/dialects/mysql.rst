MySQL 和 MariaDB
===============

.. automodule:: sqlalchemy.dialects.mysql.base

MySQL SQL 构造
--------------------

和所有 SQLAlchemy 方言一样，所有 MySQL 可用的大写类型都可以从方言的顶层导入：

```Python
from sqlalchemy.dialects.mysql import (
    BIGINT,
    BINARY,
    BIT,
    BLOB,
    BOOLEAN,
    CHAR,
    DATE,
    DATETIME,
    DECIMAL,
    DOUBLE,
    ENUM,
    FLOAT,
    INTEGER,
    LONGBLOB,
    LONGTEXT,
    MEDIUMBLOB,
    MEDIUMINT,
    MEDIUMTEXT,
    NCHAR,
    NUMERIC,
    NVARCHAR,
    REAL,
    SET,
    SMALLINT,
    TEXT,
    TIME,
    TIMESTAMP,
    TINYBLOB,
    TINYINT,
    TINYTEXT,
    VARBINARY,
    VARCHAR,
    YEAR,
)
```

以下类型是特定于 MySQL 的，或具有 MySQL 特定的构造参数：

.. 笔记：使用 :noindex: 的情况表示不在方言模块中重新定义的类型，只从 sqltypes 导入。这避免了 sphinx 构建中的警告。

.. currentmodule:: sqlalchemy.dialects.mysql

.. autoclass:: BIGINT
    :members: __init__


.. autoclass:: BINARY
    :noindex:
    :members: __init__


.. autoclass:: BIT
    :members: __init__


.. autoclass:: BLOB
    :members: __init__
    :noindex:


.. autoclass:: BOOLEAN
    :members: __init__
    :noindex:


.. autoclass:: CHAR
    :members: __init__


.. autoclass:: DATE
    :members: __init__
    :noindex:


.. autoclass:: DATETIME
    :members: __init__


.. autoclass:: DECIMAL
    :members: __init__


.. autoclass:: DOUBLE
    :members: __init__
    :noindex:

.. autoclass:: ENUM
    :members: __init__


.. autoclass:: FLOAT
    :members: __init__


.. autoclass:: INTEGER
    :members: __init__

.. autoclass:: JSON
    :members:

.. autoclass:: LONGBLOB
    :members: __init__


.. autoclass:: LONGTEXT
    :members: __init__


.. autoclass:: MEDIUMBLOB
    :members: __init__


.. autoclass:: MEDIUMINT
    :members: __init__


.. autoclass:: MEDIUMTEXT
    :members: __init__


.. autoclass:: NCHAR
    :members: __init__


.. autoclass:: NUMERIC
    :members: __init__


.. autoclass:: NVARCHAR
    :members: __init__


.. autoclass:: REAL
    :members: __init__


.. autoclass:: SET
    :members: __init__


.. autoclass:: SMALLINT
    :members: __init__


.. autoclass:: TEXT
    :members: __init__
    :noindex:


.. autoclass:: TIME
    :members: __init__


.. autoclass:: TIMESTAMP
    :members: __init__


.. autoclass:: TINYBLOB
    :members: __init__


.. autoclass:: TINYINT
    :members: __init__


.. autoclass:: TINYTEXT
    :members: __init__


.. autoclass:: VARBINARY
    :members: __init__
    :noindex:


.. autoclass:: VARCHAR
    :members: __init__


.. autoclass:: YEAR
    :members: __init__

MySQL DML Constructs
-------------------------

.. autofunction:: sqlalchemy.dialects.mysql.insert

.. autoclass:: sqlalchemy.dialects.mysql.Insert
  :members:


mysqlclient (MySQL-Python 的 fork)
----------------------------------

.. automodule:: sqlalchemy.dialects.mysql.mysqldb

PyMySQL
-------

.. automodule:: sqlalchemy.dialects.mysql.pymysql

MariaDB-Connector
------------------

.. automodule:: sqlalchemy.dialects.mysql.mariadbconnector

MySQL-Connector
---------------

.. automodule:: sqlalchemy.dialects.mysql.mysqlconnector

.. _asyncmy:

asyncmy
-------

.. automodule:: sqlalchemy.dialects.mysql.asyncmy


.. _aiomysql:

aiomysql
--------

.. automodule:: sqlalchemy.dialects.mysql.aiomysql

cymysql
-------

.. automodule:: sqlalchemy.dialects.mysql.cymysql