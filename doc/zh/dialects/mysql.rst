.. _mysql_toplevel:

MySQL 和 MariaDB
=================

.. automodule:: sqlalchemy.dialects.mysql.base

MySQL SQL 构造
--------------------

.. currentmodule:: sqlalchemy.dialects.mysql

.. autoclass:: match
    :members:

MySQL 数据类型
----------------

和所有 SQLAlchemy 方言一样，所有已经在 MySQL 中被识别为有效类型的大写类型都可以从顶级方言进行导入：

::

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

特定于 MySQL 的类型或具有 MySQL 特定构造参数的类型如下：

.. note:: 当使用 :noindex: 时，表示该类型未在方言模块中重新定义，只是从 sqltypes 导入。这避免了 sphinx 构建中的警告。

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

MySQL 数据操纵语言构造
-------------------------

.. autofunction:: sqlalchemy.dialects.mysql.insert

.. autoclass:: sqlalchemy.dialects.mysql.Insert
  :members:



mysqlclient (MySQL-Python 的分支)
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

pyodbc
------

.. automodule:: sqlalchemy.dialects.mysql.pyodbc