.. |prev| replace:: :doc:`dbapi_transactions`
.. |next| replace:: :doc:`data`

.. include:: tutorial_nav_include.rst

.. _tutorial_working_with_metadata:

使用数据库元数据
=================

现在我们已经具备了使用SQLAlchemy的SQL表达式语言所需的所有组件。SQLAlchemy的核心和ORM的中心是用于流畅、可组合地构造SQL查询的SQL Expression
Language。这些查询的基础是表示数据库概念（如表和列）的Python对象。这些对象成为数据库元数据。

SQLAlchemy中用于数据库元数据的最常见的基本对象是:class:`_schema.MetaData`, :class:`_schema.Table`和:class:`_schema.Column`.
下面的章节将示例说明这些对象是如何在Core导向风格和ORM导向风格中使用的。

.. container:: orm-header

    **ORM 读者，就跟我们走吧！**

    与其他章节一样，Core用户可以跳过ORM部分，但ORM用户最好从两个角度都熟悉这些对象。
    在使用ORM时，所讨论的:class:`_schema.Table`
    对象是用更间接的（也是完全Python类型化的）方式声明的，但是ORM配置中仍有一个
    :class:`.Table` 对象。

.. rst-class:: core-header, orm-dependency


.. _tutorial_core_metadata:

使用Table对象设置MetaData
--------------------------

当我们使用关系型数据库时，数据库中的基本数据保存结构
我们从中查询的是一个称为 **table** 的数据结构。 在SQLAlchemy中，
数据库 "table" 最终由一个Python对象来表示，这个对象名字也叫
:class:`_schema.Table`。

为了开始使用SQLAlchemy Expression Language，我们将需要建立
:class:`_schema.Table`对象，来代表我们想要处理的所有数据库表。:class:`_schema.Table`
可通过程序方式构建，直接使用
:class:`_schema.Table` 构造函数，也可以间接地使用ORM映射的类
（稍后在 :ref:`tutorial_orm_table_metadata` 中介绍）。还有一个选择是从现有数据库中加载一些或所有表信息，称为 :term:`reflection`。

无论使用哪种方法，我们都始终从一个集合开始
这是我们放置表的地方，被称为 :class:`_schema.MetaData`
对象。 这个对象本质上是一个 :term:`facade` ，用于存储一个
以字符串名为键的:class:`_schema.Table`对象系列。
虽然ORM为获取此集合提供了一些选项，但我们始终可以通过自己创建一个集合来实现，它看起来像这样：

    >>> from sqlalchemy import MetaData
    >>> metadata_obj = MetaData()

当我们拥有一个:class:`_schema.MetaData`对象时，我们可以声明一些
:class:`_schema.Table`对象。本教程将从经典的开始
SQLAlchemy教程模型，其中具有名为``user_account``的表，其中存储了例如
网站的用户，以及一个相关的表``address``，其中存储与
``user_account``表中的行关联的电子邮件地址。
当不使用ORM Declarative模型时，我们直接构造每个
:class:`_schema.Table`对象，通常将每个分配给一个变量，该变量将是我们在应用程序代码中引用表的方式::

    >>> from sqlalchemy import Table, Column, Integer, String
    >>> user_table = Table(
    ...     "user_account",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30)),
    ...     Column("fullname", String),
    ... )

在上面的示例中，当我们希望编写引用数据库中的
``user_account``表的代码时，我们将使用``user_table``
Python变量参考它。

.. topic:: 我什么时候在我的程序中创建一个 ``MetaData`` 对象？

  对于整个应用程序而言，具有单个 :class:`_schema.MetaData` 对象是最常见的情况，该对象在应用程序的单个位置（通常是在“models”或“dbschema”类型的包中的模块级变量）表示为模块级变量。使用ORM-和Core-declared
  :class:`_schema.Table`对象的一个公共 :class:`_schema.MetaData` 是通常的选择，
  这些表存储在应用程序中与Core和ORM相关的所有元数据中。

  也可以有多个 :class:`_schema.MetaData` 集合; :class:`_schema.Table` 对象
  可以引用其他集合中的:class:`_schema.Table`对象而不受任何限制。但是，
  对于相互关联的 :class:`_schema.Table` 对象组，从声明和DDL（例如 CREATE 和 DROP）语句发出的角度来看，它们都是在单个:class:`_schema.MetaData` 集合中设置的。

Table 的组件
^^^^^^^^^^^^

我们可以观察到，Python中的:class:`_schema.Table`构造具有类似于SQL CREATE TABLE语句的外观；以表名称开始，然后列出每个列，其中每个列都有一个名称和数据类型。上面我们使用的对象是：

* :class:`_schema.Table` - 表示数据库表并将其分配给:class:`_schema.MetaData`集合。

* :class:`_schema.Column` - 表示数据库表中的列，并将其分配给:class:`_schema.Table`对象。:class:`_schema.Column`
  通常包括字符串名称和类型对象。在父类:class:`_schema.Table`的基础上用于保存 :class:`_schema.Column` 对象的集合通常可以通过位于 :attr:`_schema.Table.c` 的一个关联数组来访问 ::

    >>> user_table.c.name
    Column('name', String(length=30), table=<user_account>)

    >>> user_table.c.keys()
    ['id', 'name', 'fullname']

* :class:`_types.Integer`， :class:`_types.String` - 这些类表示SQL数据类型，可以将其与 :class:`_schema.Column` 一起使用，也可以不实例化地使用。在上面的示例中，我们想将长度为“30”的“name”列分配给“String (30)”，所以我们对其进行了实例化。但是对于"id"和"fullname"，我们没有指定它们，因此我们可以发送类本身。

.. seealso::

    :class:`_schema.MetaData`、
    :class:`_schema.Table`和:class:`_schema.Column`的参考和API文档在 :ref:`metadata_toplevel` 中。
    数据类型的参考文档在 :ref:`types_toplevel` 中。

在接下来的部分中，我们将说明
:class:`_schema.Table` 最基本的一个函数，它
在特定的数据库连接上生成 :term:`DDL` 。但是首先
我们将声明第二个:class:`_schema.Table`，以查看完成的结果。

声明简单的约束
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

示例“user_table”中的第一个 :class:`_schema.Column` 包括
:paramref:`_schema.Column.primary_key` 参数，这是一个简写技术
表明此 :class:`_schema.Column` 应为此表的主键的部分。一般来说，主键
本身通常是隐含地声明，由 :class:`_schema.PrimaryKeyConstraint` 构造表示，我们可以在:class:`_schema.Table.primary_key`attribut 上看到它::

    >>> user_table.primary_key
    PrimaryKeyConstraint(Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False))

明确声明的约束最常见的是表示数据库
:term:`foreign key constraint`的 :class:`_schema.ForeignKeyConstraint` 对象。当我们声明相互关联的表时，SQLAlchemy不仅会使用这些外键约束声明将其发出到
数据库，而且还帮助构建SQL表达式。

含有仅涉及目标表上的单个列的 :class:`_schema.ForeignKeyConstraint`
通常使用列级简写符号来声明 :class:`_schema.ForeignKey` 对象。通常，表二
称为``address``，将具有引用``user``表的外键约束，如下所示::

    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table(
    ...     "address",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("user_id", ForeignKey("user_account.id"), nullable=False),
    ...     Column("email_address", String, nullable=False),
    ... )

上面的表还采用了第三种约束，这在SQL中是“NOT NULL”约束，使用
:paramref:`_schema.Column.nullable`参数来指示。

.. tip:: 在 :class:`_schema.Column` 定义中使用 :class:`_schema.ForeignKey` 对象时，我们可以省略该 :class:`_schema.Column` 的数据类型；它将从相关列的类型自动推断出来，在上面的示例中是“user_account.id”的 :class:`_types.Integer` 数据类型。

在下一节中，我们将发出DDL的完整结果。


从ORM映射将DDL发出到数据库
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^由于我们的 ORM 映射类与 `~sqlalchemy.schema.MetaData` 中包含的 `~sqlalchemy.schema.Table` 对象相关联，因此使用 Declarative Base 发出 DDL 的过程与 :ref:`tutorial_emitting_ddl` 中描述的过程相同。在我们的情况下，我们已经在 SQLite 数据库中生成了``user``和 ``address``表。如果我们还没有这样做，我们可以自由地使用与 ORM Declarative Base 类相关联的 `~sqlalchemy.schema.MetaData`，通过访问 :attr:`~sqlalchemy.ext.declarative.api.DeclarativeBase.metadata` 属性中的集合，然后像之前一样使用 `~sqlalchemy.schema.MetaData.create_all` 来生成表。在这种情况下，运行 PRAGMA 语句，但不会生成新表，因为它们已经存在：

.. sourcecode:: pycon+sql

    >>> Base.metadata.create_all(engine)
    {execsql}BEGIN (implicit)
    PRAGMA main.table_...info("user_account")
    ...
    PRAGMA main.table_...info("address")
    ...
    COMMIT


.. rst-class:: core-header, orm-addin

.. _tutorial_table_reflection:

表反射
-------------------------------

.. topic:: 可选章节

    本节仅是有关表反射的简要介绍，或如何从现有数据库自动生成 `~ sqlalchemy.schema.Table` 对象。想要继续编写查询的教程读者可以跳过本节。

为了完成与表元数据一起工作的部分，我们将演示另一种在本节开始时提到的操作，即 **表反射**。表反射是指通过读取数据库的当前状态来生成 `~ sqlalchemy.schema.Table` 和相关对象的过程。在前面的部分中，我们已经在 Python 中声明了 `~ sqlalchemy.schema.Table` 对象，然后我们可以选择向数据库发出 DDL 以生成此类模式。反射过程将这两个步骤颠倒过来，从现有数据库开始并生成用于表示该数据库中的模式的 In-Python 数据结构。

.. tip::  不必使用反射才能将 SQLAlchemy 用于现有数据库。通常情况下，SQLAlchemy 应用程序在 Python 中明确声明所有元数据，使其结构与现有数据库的结构相对应。元数据结构也不必包括不需要本地应用程序运行的现有数据库中的表、列或其他约束和结构。

例如反射，我们将创建一个新的 `~ sqlalchemy.schema.Table` 对象，该对象表示我们在本文档的前面部分手动创建的 ``some_table`` 对象。此处有一些不同的方法，但是最基本的方法是构建一个 `~sqlalchemy.schema.Table` 对象，给出表的名称和一个 `~sqlalchemy.schema.MetaData` 集合，然后通过使用 :paramref:`_schema.Table.autoload_with` 参数将目标 `~sqlalchemy.engine.Engine` 传递给它：   

.. sourcecode:: pycon+sql

    >>> some_table = Table("some_table", metadata_obj, autoload_with=engine)
    {execsql}BEGIN (implicit)
    PRAGMA main.table_...info("some_table")
    [raw sql] ()
    SELECT sql FROM  (SELECT * FROM sqlite_master UNION ALL   SELECT * FROM sqlite_temp_master) WHERE name = ? AND type in ('table', 'view')
    [raw sql] ('some_table',)
    PRAGMA main.foreign_key_list("some_table")
    ...
    PRAGMA main.index_list("some_table")
    ...
    ROLLBACK{stop}

此过程结束时，``some_table`` 对象现在包含表中存在的 `~ sqlalchemy.schema.Column` 对象的信息，该对象可以与我们明确声明的 `~ sqlalchemy.schema.Table` 以完全相同的方式使用：

    >>> some_table
    Table('some_table', MetaData(),
        Column('x', INTEGER(), table=<some_table>),
        Column('y', INTEGER(), table=<some_table>),
        schema=None)

.. seealso::

    关于表和模式反射的更多信息请参见 :ref:`metadata_reflection_toplevel`。

    对于 ORM 相关的表反射变体，本节 :ref:`orm_declarative_reflected` 包括可以使用的可用选项的概述。

下一步
----------

我们现在准备好使用两个表与 Core 和 ORM 与表有关的构造与 :class:`_engine.Connection` 和/或 ORM :class:`_orm.Session` 进行交互。 在接下来的几个部分中，我们将说明如何使用这些结构创建、操作和选择数据。