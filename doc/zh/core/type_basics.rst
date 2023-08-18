类型层次结构
=====================

.. module:: sqlalchemy.types

SQLAlchemy提供了大多数常见数据库数据类型的抽象，以及几种自定义数据类型的技术。

数据库类型使用Python类表示，所有这些类最终都扩展自称为  :type:`_types.TypeEngine`  的基础类型类。有两种常见的数据类型，每个数据类型类在类型层次结构中表现出不同的方式。可以基于使用两种不同命名约定，即“CamelCase”和“UPPERCASE”来确定单个数据类型类使用的类别。

.. seealso::

      :ref:`unified_tutorial`  - 演示了
    最基本的使用 :class:`_types.TypeEngine` 类型对象定义 :class:`_schema.Table` 的元数据
    和介绍类型对象的概念。

“CamelCase” 数据类型
-------------------------

基本类型拥有像   :class:`_types.String` ，  :class:` _types.Numeric` ，  :class:`_types.Integer`  和   :class:` _types.DateTime`  这样的“CamelCase”名称。所有   :class:`_types.TypeEngine`  的直接子类都是“CamelCase”类型。尽可能地，"CamelCase" 类型在**数据库无关**上使用，这意味着它们可以在任何数据库后端上使用，它们将以适当的方式表现出后端以产生所需行为。

一个简单“CamelCase”数据类型的示例是   :class:`_types.String` 。在大多数后端数据库中，将这种数据类型在  :ref:` table specification <metadata_describing>` `VARCHAR`` 数据库类型，从数据库传输和接收字符串值，如下面的示例所示::

    from sqlalchemy import MetaData
    from sqlalchemy import Table, Column, Integer, String

    metadata_obj = MetaData()

    user = Table(
        "user",
        metadata_obj,
        Column("user_name", String, primary_key=True),
        Column("email_address", String(60)),
    )

在   :class:`_schema.Table`  定义中使用特定   :class:` _types.TypeEngine`  类或者在任何 SQL 表达式中使用时，如果不需要参数，则可以将其作为类本身，例如，不使用 `()` 实例化类方法。如果需要参数，例如，上面 ``"email_address"`` 列中的长度参数 60，则可以实例化该类型。

另一个表示更多后端特定行为的“CamelCase”名称的数据类型是   :class:`_types.Boolean`  数据类型。与  :class:` _types.String` `true`` 和``false``，有些则没有。对于此数据类型，  :class:`_types.Boolean` ` BOOLEAN``，MySQL上呈现为 ``BIT``，Oracle上呈现为 ``SMALLINT``。在使用此类型将数据从数据库发送和接收时，根据使用的方言，它可以转换为Python数值或布尔值。

通常， SQLAlchemy 应用程序在一般情况下可能希望主要使用“CamelCase”类型，因为它们通常提供最佳的基本行为，并且可以自动地移植到所有后端。

通用“CamelCase”数据类型的参考在下面的   :ref:`types_generic` 。

“UPPERCASE” 数据类型
-------------------------

与 “CamelCase” 类型相反的是“ UPPERCASE” 数据类型。这些数据类型始终继承自特定的“ CamelCase” 数据类型，并且始终表示一种**精确**数据类型。在使用“UPPERCASE” 数据类型时，类型的名称始终以给定的方式呈现，而不管当前的后端是否支持它。因此，在 SQLAlchemy 应用程序中使用“UPPERCASE” 数据类型表示需要特定数据类型，这意味着该应用程序通常会限制为使用完全按照给定类型使用的后端，如果没有将采取其他措施。UPPERCASE 类型的示例包括   :class:`_types.VARCHAR` ，  :class:` _types.NUMERIC` ，  :class:`_types.INTEGER`  和   :class:` _types.TIMESTAMP` ，它们直接从前面提到的“ CamelCase” 类型继承，即   :class:`_types.String` ，：class:` _types.Numeric`，：class:`_types.Integer` 和   :class:`_types.DateTime` 。

的“UPPERCASE” 数据类型是 ``sqlalchemy.types`` 的一部分，是通常期望在至少两个后端上可用的常见 SQL 类型。

参考标准集“UPPERCASE”数据类型在下面的   :ref:`types_sqlstandard` 。

.. _types_vendor:

特定于后端的“UPPERCASE” 数据类型
--------------------------------------

大多数数据库还具有完全适用于该数据库的特定数据类型，或者添加了对该数据库特定的其他参数。对于这些数据类型，特定于 SQLAlchemy 方言提供了特定于后端的“ UPPERCASE” 数据类型，作为没有类比于其他后端的 SQL 类型。后端特定大写的数据类型示例包括 PostgreSQL 的   :class:`_postgresql.JSONB`  ，SQL Server 的
：class:`_mssql.IMAGE` 和 MySQL 的   :class:`_mysql.TINYTEXT` 。

特定的后端还可以包括扩展与可在 ``sqlalchemy.types`` 模块找到的相同“ UPPERCASE” 数据类型相同的参数的“ UPPERCASE” 数据类型。例如，创建 MySQL 字符串数据类型时，可能希望指定 MySQL 特定参数，如 ``charset`` 或 ``national`` ， 这些参数可从   :class:`_mysql.VARCHAR`  的 MySQL 版本的  :paramref:` _mysql.VARCHAR.charset`  和  :paramref:`_mysql.VARCHAR.national`  中使用。

特定于后端类型的 API 文档在特定于方言的文档中列出，列于：ref:`dialect_toplevel`。

.. _types_with_variant:

将“UPPERCASE”和特定于后端的类型应用于多个后端
------------------------------------------------------------------

查看“UPPERCASE”和“ CamelCase” 类型的存在自然引导了在要求特定于后端选项时如何利用“UPPERCASE” 数据类型的用例，但仅在正在使用该后端时。要将数据库无关的“ CamelCase” 和后端特定的“UPPERCASE” 系统绑定在一起，必须使用  :meth:`_types.TypeEngine.with_variant`  方法将类型组合在一起，以便在具体后端上使用特定行为的情况下使用特定的类型。

例如，在使用   :class:`_types.String`  数据类型时，但在运行 MySQL 时，若要在 MySQL 或 MariaDB 上创建表时使用  :paramref:` _mysql.VARCHAR.charset`  参数，请使用以下属性 ::

    from sqlalchemy import MetaData
    from sqlalchemy import Table, Column, Integer, String
    from sqlalchemy.dialects.mysql import VARCHAR

    metadata_obj = MetaData()

    user = Table(
        "user",
        metadata_obj,
        Column("user_name", String(100), primary_key=True),
        Column(
            "bio",
            String(255).with_variant(VARCHAR(255, charset="utf8"), "mysql", "mariadb"),
        ),
    )

在上面的表定义中，“bio”列将具有所有后端的字符串行为。在大多数后端上，它将呈现为 DDL ``VARCHAR``。但是，在MySQL和MariaDB上（通过数据库URL以 ``mysql`` 或 ``mariadb`` 开头表示），它将呈现为 ``VARCHAR（255） CHARACTER SET utf8``。

.. seealso::

     :meth:`_types.TypeEngine.with_variant`  - 更多的用法示例和注意事项

.. _types_generic:

通用“CamelCase” 类型
-------------------------

通用类型指定可以读取、写入和存储特定类型 Python 数据的列。当发出 `CREATE TABLE` 语句时，SQLAlchemy 将选择目标数据库中最佳的数据库列类型。要完全控制发出 `CREATE TABLE` 中使用的列类型，例如 ``VARCHAR`` 请参见   :ref:`types_sqlstandard`  和本章的其他部分。

.. autoclass:: BigInteger
   :members:

.. autoclass:: Boolean
   :members:

.. autoclass:: Date
   :members:

.. autoclass:: DateTime
   :members:

.. autoclass :: Enum
  :members: __init__, create, drop

.. autoclass:: Double
  :members:

.. autoclass:: Float
  :members:

.. autoclass:: Integer
   :members:

.. autoclass:: Interval
   :members:

.. autoclass:: LargeBinary
   :members:

.. autoclass:: MatchType
   :members:

.. autoclass:: Numeric
   :members:

.. autoclass:: PickleType
   :members:

.. autoclass:: SchemaType
   :members:
   :undoc-members:

.. autoclass:: SmallInteger
   :members:

.. autoclass:: String
   :members:

.. autoclass:: Text
   :members:

.. autoclass:: Time
   :members:

.. autoclass:: Unicode
   :members:

.. autoclass:: UnicodeText
   :members:

.. autoclass:: Uuid
   :members:

.. _types_sqlstandard:

SQL 标准和多个供应商的“UPPERCASE”数据类型
--------------------------------------------------

此类类型指的是既是 SQL 标准组成部分，又可能在一部分数据库后端中找到的类型。与“通用”类型不同，SQL 标准/多供应商类型**没有**保证在所有后端上工作，只能在显式支持它们的那些后端上工作。即，类型将始终在使用 `CREATE TABLE` 发出时使用其确切名称。

.. autoclass:: ARRAY
   :members:

.. autoclass:: BIGINT

.. autoclass:: BINARY

.. autoclass:: BLOB

.. autoclass:: BOOLEAN

.. autoclass:: CHAR

.. autoclass:: CLOB

.. autoclass:: DATE

.. autoclass:: DATETIME

.. autoclass:: DECIMAL

.. autoclass:: DOUBLE

.. autoclass:: DOUBLE_PRECISION

.. autoclass:: FLOAT

.. autoclass:: INT

.. autoclass:: JSON
    :members:

.. autoclass:: sqlalchemy.types.INTEGER

.. autoclass:: NCHAR

.. autoclass:: NVARCHAR

.. autoclass:: NUMERIC

.. autoclass:: REAL

.. autoclass:: SMALLINT

.. autoclass:: TEXT

.. autoclass:: TIME

.. autoclass:: TIMESTAMP
    :members:

.. autoclass:: UUID

.. autoclass:: VARBINARY

.. autoclass:: VARCHAR