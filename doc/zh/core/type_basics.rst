类型层次结构
=====================

.. module:: sqlalchemy.types

SQLAlchemy提供了大多数常见数据库数据类型的抽象，以及几种用于自定义数据类型的技术。

数据库类型使用Python类表示，这些类最终都扩展自称为:class:`_types.TypeEngine`的基本类型类。有两种数据类型类别，每种类别在类型层次结构中以不同的方式表达。每个单独的数据类型类使用的类别可以根据使用两种不同命名约定来识别，即“CamelCase”和“UPPERCASE”。

.. seealso::

    :ref:`unified_tutorial`中的:ref:`tutorial_core_metadata` - 说明
    使用:class:`_types.TypeEngine`类型对象定义:class:`_schema.Table`元数据的最基本用法，并在教程形式中介绍了类型对象的概念。

“CamelCase”数据类型
-------------------------

最基本的类型具有“CamelCase”名称，例如:class:`_types.String`、:class:`_types.Numeric`、:class:`_types.Integer`和:class:`_types.DateTime`。所有:class:`_types.TypeEngine`的直接子类都是“CamelCase”类型。在最大程度上，“CamelCase”类型是**与数据库无关**的，这意味着它们可以在任何数据库后端上使用，其中它们将根据需要在适当的后端上以产生所需的行为的方式表现。

一个直接的“CamelCase”数据类型的例子是:class:`_types.String`。在大多数后端上，将该数据类型用于:ref:`metadata_describing`中的表规范将相应地使用“VARCHAR”数据库类型，在数据库和字符串值之间传递，如下面的示例所示：

    from sqlalchemy import MetaData
    from sqlalchemy import Table, Column, Integer, String

    metadata_obj = MetaData()

    user = Table(
        "user",
        metadata_obj,
        Column("user_name", String, primary_key=True),
        Column("email_address", String(60)),
    )

在:class:`_schema.Table`定义或任何SQL表达式中使用特定的:class:`_types.TypeEngine`类时，如果不需要参数，则可以将其作为类本身传递，即不使用“()”实例化它。如果需要参数，例如上面的“email_address”列中的长度参数60，则可以实例化该类型。

另一个表示更多后端特定行为的“CamelCase”数据类型是:class:`_types.Boolean`数据类型。与:class:`_types.String`不同，它代表所有数据库的字符串数据类型，并非每个后端都有原始的“boolean”数据类型。有些后端使用整数或位值0和1，有些有布尔文字字面值“true”和“false”，而其他后端则没有。对于这种数据类型，:class:`_types.Boolean`可能在例如PostgreSQL上渲染出“BOOLEAN”，在MySQL后端上渲染出“BIT”，在Oracle上渲染出“SMALLINT”。当使用此类型发送和接收来自数据库的数据时，根据使用的SQL方言，它可以解释Python的数字或布尔值。

典型的SQLAlchemy应用程序很可能希望在一般情况下主要使用“CamelCase”类型，因为它们通常提供最佳的基本行为，并可以自动移植到所有后端。

下面是用于一般“CamelCase”数据类型的整个参考:ref:`types_generic`。

“UPPERCASE”数据类型
-------------------------

与“CamelCase”类型相对应的是“UPPERCASE”数据类型。这些数据类型始终从特定的“CamelCase”数据类型继承，并始终表示**确切**数据类型。对于使用“UPPERCASE”数据类型，类型的名称总是精确地呈现，而不考虑当前后端是否支持它。因此，在SQLAlchemy应用程序中使用“UPPERCASE”类型表示需要特定数据类型，这意味着应用程序通常将受到限制，除非采取其他步骤，否则将限于使用恰好按给定方式使用类型的后端。包括:class:`_types.VARCHAR`、:class:`_types.NUMERIC`、:class:`_types.INTEGER`和:class:`_types.TIMESTAMP`在内的大写字母类型是直接从上述“CamelCase”类型继承的。

在“sqlalchemy.types”中的“UPPERCASE”数据类型是常见的SQL类型，这些类型通常期望在至少两个或多个后端可用。

下面是用于一般“UPPERCASE”数据类型的整个参考:ref:`types_sqlstandard`。

.. _types_vendor:

特定于后端的“UPPERCASE”数据类型
--------------------------------------

大多数数据库还具有它们自己的数据类型，
这些类型只针对这些数据库，或者添加了特定于这些数据库的其他参数。
对于这些数据类型，特定的SQLAlchemy方言提供了**特定于后端**的“UPPERCASE”数据类型，用于没有模拟在其他后端上的SQL类型。其中后端特定字母大写数据类型的示例包括PostgreSQL的:class:`_postgresql.JSONB`、SQL Server的:class:`_mssql.IMAGE`和MySQL的:class:`_mysql.TINYTEXT`。

特定于后端的大写字母数据类型可能还包括扩展来自与`:class:`_mysql.VARCHAR`相同的大写字母数据类型的参数的后端特定参数，例如创建MySQL字符串数据类型时，可能要指定MySQL特定参数，例如“charset”或“national”，它们在MySQL版本中可以从:class:`_mysql.VARCHAR`中使用作为MySQL-only类型参数：请参见 :paramref:`_mysql.VARCHAR.charset`和 :paramref:`_mysql.VARCHAR.national`。

特定于后端类型的API文档在特定于方言的文档中列出，在:ref:`dialect_toplevel`下

.. _types_with_variant:

对于多个后端使用“UPPERCASE”和特定后端的数据类型
------------------------------------------------------------------

回顾“UPPERCASE”和“CamelCase”类型的存在，自然的用例是如何利用后端特定数据类型的“UPPERCASE”数据类型进行后端特定的选项，但仅在该后端处于使用状态时。为了将数据库不可知的“CamelCase”与后端特定的“UPPERCASE”系统联系起来，可以使用:meth:`_types.TypeEngine.with_variant`方法将类型**组合**在一起，以便针对特定后端上的特定行为进行工作。

例如，要使用:class:`_types.String`数据类型，但在运行MySQL时，要在创建表时使用:class:`_mysql.VARCHAR`的:paramref:`_mysql.VARCHAR.charset`参数，
可以使用以下方法：:meth:`_types.TypeEngine.with_variant`：

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

在上面的表定义中，““bio””列将在所有后端上具有字符串行为。在大多数后端上，它将DD使用“VARCHAR”来渲染。然而，在MySQL和MariaDB（由以“mysql”或“mariadb”开头的数据库URL指示）上，它将呈现为“VARCHAR(255) CHARACTER SET utf8”。

.. seealso::

    :meth:`_types.TypeEngine.with_variant` - 其他用法示例和注释

.. _types_generic:

一般的“CamelCase”类型
-------------------------

常规类型指定可以读取、写入和存储特定类型的Python数据的列。当发出“CREATE TABLE”语句时，SQLAlchemy将在目标数据库上选择最佳的数据库列类型。要完全控制在“CREATE TABLE”中发射的列类型，例如“VARCHAR”，请参见:ref:`types_sqlstandard`和本章的其他部分。

.. autoclass:: BigInteger
   :members:

.. autoclass:: Boolean
   :members:

.. autoclass:: Date
   :members:

.. autoclass:: DateTime
   :members:

.. autoclass:: Enum
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

SQL标准和多供应商“UPPERCASE”类型
--------------------------------------------------

此类型类别是指作为SQL标准的一部分或可能在数据库后端子集中找到的类型。与通用类型不同，SQL标准/多供应商类型**没有**保证在所有后端上工作，并且仅会在明确支持它们的后端上工作名称。也就是说，该类型将始终在DDL中发出其确切的名称，即当发出与“CREATE TABLE”对应的DDL时。

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

