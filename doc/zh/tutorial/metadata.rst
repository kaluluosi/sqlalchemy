.. |prev| replace::  :doc:`dbapi_transactions` 
.. |next| replace::  :doc:`data` 

.. include:: tutorial_nav_include.rst

.. _tutorial_working_with_metadata:

使用数据库元数据
==============================

在了解 SQLAlchemy Core 和 ORM 的引擎和 SQL 执行之后，我们准备开始一些 Alchemy。
SQL 表达式语言是构建 SQL 查询流畅可组合的元素，数据库概念如表格和列在Python中的代表,我们也将其称为  :term:`数据库元数据` .
对于数据库元元素数据的基础对象是是   :class:`_schema.MetaData` ,   :class:` _schema.Table` , 和   :class:`_schema.Column` .
此文章将详细介绍这些对象是如何在使用面向核心的风格和面向ORM的风格中使用.

.. container:: orm-header

    **ORM readers, stay with us!**

    如其他章节所述，Core 用户可以跳过 ORM 部分，但 ORM 用户最好从两种角度熟悉这些对象。讨论的   :class:`.Table`  对象在使用 ORM 时以一种更间接的方式（也是完全 Python 类型）声明，但 ORM 的配置中仍有一个  :class:` .Table`  对象。

.. rst-class:: core-header, orm-dependency


.. _tutorial_core_metadata:

使用 Table 对象设置元数据
-----------------------------------

当与关系型数据库一起使用时，数据库中的基本数据持有数据的数据结构被称为**表**。
在 SQLAlchemy 中，数据库 "table" 最终由类似命名为的Python对象   :class:`_schema.Table`  表示.

为了开始使用 SQLAlchemy Expression Language, 我们需要构建   :class:`_schema.Table`  代表我们感兴趣的数据库所有表的对象.    :class:` _schema.Table`  对象通过程序进行构建，直接使用   :class:`_schema.Table`  构造函数，或者通过使用 ORM Mapped 类 (在   :ref:` tutorial_orm_table_metadata`  中介绍) 间接构造. 也有选项从现有数据库中加载所有或部分表信息，称为  :term:`reflection` 。

无论使用哪种方法，我们始终从一个集合开始，将我们的表放置在这里，这被称为   :class:`_schema.MetaData`  对象. 该对象本质上是一个Python字典的  :term:` facade` , 存储按其字符串名称键入的一系列   :class:`_schema.Table`  对象。虽然ORM提供了一些选项来获取此集合的位置，但我们始终可以直接创建一个集合，其如下所示::

    >>> from sqlalchemy import MetaData
    >>> metadata_obj = MetaData()

有了   :class:`_schema.MetaData`  对象后，我们可以声明一些   :class:` _schema.Table`  对象。此教程将从经典的SQLAlchemy教程模型开始，该模型具有名为 "user_account" 表，其中存储例如网站用户的行，以及一个相关表 "address"，该表存储与 "user_account" 表中的行相关联的电子邮件地址。当完全不使用 ORM Declarative 模型时，我们直接构造每个   :class:`_schema.Table`  对象，通常将每个对象分配给变量，“user_table” 就是我们在本文中所用的命名::

    >>> from sqlalchemy import Table, Column, Integer, String
    >>> user_table = Table(
    ...     "user_account",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30)),
    ...     Column("fullname", String),
    ... )

通过以上示例，当我们希望编写引用数据库中 "user_account" 表的代码时，我们将使用 "user_table" Python 变量引用该表。

.. topic:: 什么时候在程序中创建 "MetaData" 对象?

  一个恰当的   :class:`_schema.MetaData`  对象对于整个应用程序来说是最常见的情况，常表示为单个位置的模块级变量，通常在“模型”或“dbschema”类别的包中。通常，使用 ORM-centric 的   :class:` _orm.registry`  或   :ref:`Declarative Base <tutorial_orm_declarative_base>`  基础类访问   :class:` _schema.MetaData` ，以便 ORM 和 Core 声明的   :class:`_schema.Table`  对象共享此相同的  :class:` _schema.MetaData` 。

  同样也可以有多个   :class:`_schema.MetaData`  集合；   :class:` _schema.Table`  对象可以引用任何其他集合中的   :class:`_schema.Table`  对象。但是，对于群组我们建议使用单个   :class:` _schema.MetaData` 。  :class:`_schema.Table`  集合内从声明和 DDL（即 CREATE 和 DROP）语句的角度来看都更加简单和直接。

``Table`` 的组件
^^^^^^^^^^^^^^^^^^^^^^^

我们可以观察到在 Python 中编写的   :class:`_schema.Table`  结构类似于 SQL CREATE TABLE 语句；从表的名称开始，然后列出每列，其中每列都有一个名称和一个数据类型。我们使用的对象如下：

*   :class:`_schema.Table`  - 表示一个数据库表，并将其分配给一个   :class:` _schema.MetaData`  集合。
*   :class:`_schema.Column`  - 表示数据库表中的一列，并将其分配给一个   :class:` _schema.Table`  对象。  :class:`_schema.Column`  通常包括一个字符串名称和一个类型对象。在父级   :class:` _schema.Table`  中，一组   :class:`_schema.Column`  对象通常通过位于  :attr:` _schema.Table.c`  的关联数组访问::

    >>> user_table.c.name
    Column('name', String(length=30), table=<user_account>)

    >>> user_table.c.keys()
    ['id', 'name', 'fullname']

*   :class:`_types.Integer` ，  :class:` _types.String`  - 这些类表示 SQL 数据类型，并可传递给   :class:`_schema.Column` ，无论是否需要实例化。在上面的示例中，我们想要为 "name" 列提供长度为 "30"，因此实例化了 ` `String(30)``。但是对于 "id" 和 "fullname"，我们没有指定这些，因此可以发送类本身。

.. seealso::

      :class:`_schema.MetaData` 、  :class:` _schema.Table`  和   :class:`_schema.Column`  的参考和 API 文档位于   :ref:` metadata_toplevel` 。数据类型的参考文档位于   :ref:`types_toplevel` 。

接下来，我们将说明   :class:`_schema.Table`  的一个基本功能，即在特定的数据库连接上生成  :term:` DDL` 。但首先我们将声明第二个   :class:`_schema.Table` 。

声明简单约束
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

上面示例中的第一个   :class:`_schema.Column`  包括  :paramref:` _schema.Column.primary_key`  参数，它是一种简写技术，用于指示此   :class:`_schema.Column`  应该是此表的主键的一部分。主键本身通常是隐式声明的，并由   :class:` _schema.PrimaryKeyConstraint`  结构表示，我们可以在   :class:`_schema.Table`  对象上的  :attr:` _schema.Table.primary_key`  属性中看到它::

    >>> user_table.primary_key
    PrimaryKeyConstraint(Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False))

通常显式声明的约束是对应于数据库  :term:`外键约束`  的   :class:` _schema.ForeignKeyConstraint`  对象。当我们声明相互关联的表时，SQLAlchemy 不仅使用这些外键约束声明，以便将它们发出到数据库中的 CREATE 语句中，而且对于构建 SQL 表达式也很有帮助。

涉及目标表中仅包含单个列的   :class:`_schema.ForeignKeyConstraint`  通常使用基于列的快捷方式符号   :class:` _schema.ForeignKey`  对象进行声明。下面声明了一个将具有引用 ``user`` 表的外键约束的第二个表 ``address``::

    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table(
    ...     "address",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("user_id", ForeignKey("user_account.id"), nullable=False),
    ...     Column("email_address", String, nullable=False),
    ... )

以上表还使用了第三种约束，即在 SQL 中为“NOT NULL”约束，上面使用  :paramref:`_schema.Column.nullable`  参数表示。

.. tip:: 在   :class:`_schema.Column`  定义中使用   :class:` _schema.ForeignKey`  对象时，我们可以省略该   :class:`_schema.Column`  的数据类型；它将从相关列的数据类型自动推断出来，在上面的示例中是 ` `user_account.id`` 列的   :class:`_types.Integer`  数据类型。

在下一节中，我们将发出“user”和“address”表的完整 DDL，以查看完成的结果。

.. _tutorial_emitting_ddl:

将 DDL 发出到数据库
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们构建了一个表示数据库中两个表的对象结构，从根   :class:`_schema.MetaData`  对象开始，然后进入两个   :class:` _schema.Table`  对象，每个对象都持有一组   :class:`_schema.Column`  和   :class:` _schema.Constraint`  对象。这个对象结构将是大多数操作的中心。我们将继续使用Core和ORM。

我们可以使用这个结构的第一个有用的操作是，将 CREATE TABLE 语句或 DDL 发送到我们的 SQLite 数据库，以便我们可以向其中插入和查询数据。我们已经拥有了执行此操作所需的所有工具，只需在我们的 _schema.MetaData 上调用 _schema.MetaData.create_all 方法，将引用目标数据库的 _engine.Engine 发送给它：

.. sourcecode:: pycon+sql

    >>> metadata_obj.create_all(engine)
    {execsql}BEGIN (implicit)
    PRAGMA main.table_...info("user_account")
    ...
    PRAGMA main.table_...info("address")
    ...
    CREATE TABLE user_account (
        id INTEGER NOT NULL,
        name VARCHAR(30),
        fullname VARCHAR,
        PRIMARY KEY (id)
    )
    ...
    CREATE TABLE address (
        id INTEGER NOT NULL,
        user_id INTEGER NOT NULL,
        email_address VARCHAR NOT NULL,
        PRIMARY KEY (id),
        FOREIGN KEY(user_id) REFERENCES user_account (id)
    )
    ...
    COMMIT

上述 DDL 创建过程包括一些特定于 SQLite 的 PRAGMA 语句，用于测试每个表在发出 CREATE 语句之前是否存在。完整的步骤也包括在 BEGIN/COMMIT 对中，以适应事务 DDL。

创建过程还会注意以正确的顺序发出 CREATE 语句；在上面的示例中，外键约束取决于存在 "user" 表，因此 "address" 表在第二次创建。在更复杂的依赖关系场景中，FOREIGN KEY 约束也可以在事后使用 ALTER 应用于表。

_schema.MetaData 对象还带有一个 _schema.MetaData.drop_all 方法，它会以与发出 CREATE 相反的顺序发出 DROP 语句以删除模式元素。

.. topic:: 迁移工具通常是恰当的

    总体而言，_schema.MetaData 的 CREATE/DROP 功能对测试套件、小型和/或新应用程序以及使用短期数据库的应用程序非常有用。然而，对于长期管理应用程序数据库模式的应用程序而言，一个基于 SQLAlchemy 的模式管理工具，例如 `Alembic <https://alembic.SQLAlchemy.org>`_，更具优势，因为它可以管理和协调随着应用程序设计的更改而逐步更改固定数据库模式的过程。

.. rst-class:: orm-header

.. _tutorial_orm_table_metadata:

使用 ORM 声明形式定义表元数据
------------------------------------

.. topic:: 另一种创建 Table 对象的方式？

  上面的例子演示了   :class:`_schema.Table`  直接使用的过程，它是 SQLAlchemy 在构建 SQL 表达式时最终引用到数据库表的基础。正如前面提到的，SQLAlchemy ORM 提供了称为 **声明形式 Table** 的一种绕路，它的声明过程指向构建   :class:` _schema.Table`  对象，但在此过程中还提供了一个称为 ORM 映射类或者是映射类的东西。映射类是使用 ORM 时最常见的基础 SQL 单元，在现代 SQLAlchemy 中也可以与面向 Core 的使用相当有效地结合使用。

  使用声明形式 Table 的一些好处包括：

  * 更简洁、更 Pythonic 的设置列定义的方式，其中可以使用 Python 类型来表示要在数据库中使用的 SQL 类型

  * 结果映射类可以用于形成 SQL 表达式，在许多情况下保持  :pep:`484`  类型信息，这些信息被静态分析工具如 Mypy 和 IDE 类型检查器拾取

  * 允许声明表元数据和 ORM 映射类在持久性/对象加载操作中同时使用。

  本节将演示使用声明形式 Table 构建与前面部分相同的   :class:`_schema.Table`  元数据。

当使用 ORM 时，我们声明   :class:`_schema.Table`  元数据的过程通常与声明  :term:` 映射`  类的过程结合在一起。映射类是我们想要创建的任何 Python 类，然后它将在其中具有链接到数据库表中的列的属性。虽然有几种实现方式，但最常见的风格称为   :ref:`声明性 <orm_declarative_mapper_config_toplevel>` ，它允许我们同时声明用户定义类和   :class:` _schema.Table`  元数据。

.. _tutorial_orm_declarative_base:

建立一个声明形式 Base
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用 ORM 时，_schema.MetaData 集合仍然存在，但它本身与一个仅用于 ORM 的构造物关联起来，通常称为 **声明形式 Base**。获取新的声明形式 Base 的最简单方法是创建一个子类，该子类是 SQLAlchemy   :class:`_orm.DeclarativeBase`  类：

    >>> from sqlalchemy.orm import DeclarativeBase
    >>> class Base(DeclarativeBase):
    ...     pass

在上面的示例中，"Base" 类是我们将要引用的声明形式 Base。当我们创建新的类作为 ``Base`` 的子类时，并结合适当的类级指令，它们将在类创建时建立为新的**ORM映射类**，每个映射通常（但不总是）与特定的 :class:`_schema.Table` 对象相关联。

Declarative Base指的是一个  :class:`_schema.MetaData` .MetaData` 集合可以通过  :class:`_orm.DeclarativeBase.metadata` .MetaData` 集合中的一个  :class:`.Table` ：

    >>> Base.metadata
    MetaData()

Declarative Base也指一个名为   :class:`_orm.registry`  的集合， 这是SQLAlchemy ORM中的中心` 映射器配置`单元。虽然很少直接访问这个对象，但是这个对象对映射器配置过程非常重要，因为一组ORM映射类将通过这个注册表相互协调。就像  :class:`.MetaData` （可以通过传递自己的 :class:` _orm.registry`进行选项），可以通过 :class:`_orm.DeclarativeBase.registry` 类变量进行访问：

    >>> Base.registry
    <sqlalchemy.orm.decl_api.registry object at 0x...>

.. topic:: 使用``registry``映射的其他方式

    :class:`_orm.DeclarativeBase`  中都有相关的参考文档。

.. _tutorial_declaring_mapped_classes:

声明映射类
^^^^^^^^^^^^^^^^^^^^^^^^

有了 ``Base`` 类之后，我们现在可以为 ``user_account`` 和 ``address`` 表定义ORM映射类，这使用了新的类 ``User`` 和 ``Address``。下面我们展示最现代的声明式形式，它是通过使用特殊类型  :class:`.Mapped`  类型注释进行驱动，表示要映射为特定类型的属性：

    >>> from typing import List
    >>> from typing import Optional
    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import mapped_column
    >>> from sqlalchemy.orm import relationship

    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str] = mapped_column(String(30))
    ...     fullname: Mapped[Optional[str]]
    ...
    ...     addresses: Mapped[List["Address"]] = relationship(back_populates="user")
    ...
    ...     def __repr__(self) -> str:
    ...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

    >>> class Address(Base):
    ...     __tablename__ = "address"
    ...
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     email_address: Mapped[str]
    ...     user_id = mapped_column(ForeignKey("user_account.id"))
    ...
    ...     user: Mapped[User] = relationship(back_populates="addresses")
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"

上面的两个类，``User`` 和 ``Address``，现在被称为**ORM映射类**，并可用于ORM持久性和查询操作，这将在后面描述。关于这些类的详细信息包括：

* 每个类引用了作为声明映射过程的一部分生成的  :class:`_schema.Table`  属性来实现的。一旦创建类，此生成的  :class:` _schema.Table`  属性中获得。

* 如前所述，这种形式被称为：ref:`orm_declarative_table_configuration`。几种备选声明样式之一将直接构建  :class:`_schema.Table`  来构建该对象。这种风格被称为：ref:` Declarative with Imperative Table <orm_imperative_table_configuration>`。

* 要指示 :class:`_schema.Table` 中的列，我们使用 :func:`_orm.mapped_column` 结构，与基于 :class:`_orm.Mapped` 类型的类型注释结合使用。 此对象将生成应用于构建 :class:`_schema.Table` 的 :class:`_schema.Column` 对象。

* 对于具有简单数据类型且没有其他选项的列，我们可以仅使用  :class:`_orm.Mapped` ` int``和``str``等简单Python类型表示  :class:`.Integer` .String` 。如何在声明式映射过程中自定义 Python 类型的解释是非常开放的。详见   :ref:`orm_declarative_mapped_column`  和   :ref:` orm_declarative_mapped_column_type_map`  章节。

* 一个列（column）可以依据是否有 ``Optional[<typ>]`` 类型注释（或其等效形式 ``<typ> | None`` or ``Union[<typ>, None]``）来声明为可空或非空。也可以使用  :paramref:`_orm.mapped_column.nullable`  参数来显式指定（而且该指定不一定要与注释的可选属性一致）。

* 显式类型注释的使用是**完全可选的**。我们也可以在没有注释时使用   :func:`_orm.mapped_column` 。这种情况下，我们将在每一个   :func:` _orm.mapped_column`  构造中使用更明确的类型对象，比如   :class:`.Integer`  和   :class:` .String` ，以及根据需要的 ``nullable=False``。

* 两个附加属性， ``User.addresses`` 和 ``Address.user``，定义了一种不同类型的属性，称为   :func:`_orm.relationship` ，其中的注释感知配置样式与前述的属性类似。关于   :func:` _orm.relationship`  构造，更详细的内容在   :ref:`tutorial_orm_related_objects`  章节中讨论。

* 如果我们没有声明自己的 ``__init__()`` 方法，那么该类会自动被分配一个。这个方法的默认形式接受所有的属性名作为可选的关键字参数：

    >>> sandy = User(name="sandy", fullname="Sandy Cheeks")

   要自动生成一个完整的 ``__init__()`` 方法，既能提供位置参数，也能提供具有默认关键字值的参数，可以使用在   :ref:`orm_declarative_native_dataclasses`  中介绍的 dataclasses 功能。当然，使用显式的 ` `__init__()`` 方法也是一种选择。

* 添加 ``__repr__()`` 方法是为了让我们得到可读的字符串输出；这些方法不必存在的要求。与 ``__init__()`` 相同，使用   :ref:`dataclasses <orm_declarative_native_dataclasses>`  特性可以自动地生成一个 ` `__repr__()`` 方法。

.. topic:: 旧的 Declarative 去哪儿了？

   使用 SQLAlchemy 1.4 或更早版本的用户会注意到，上述映射的形式与以往截然不同；它不仅在 Declarative 映射中使用   :func:`_orm.mapped_column` ，而且还使用 Python 类型注释来派生列信息。

   为了提供“旧”方式的使用者的背景，仍然可以使用   :class:`.Column`  对象进行 Declarative 映射（以及使用   :func:` _orm.declarative_base`  函数创建基类），这些形式将继续得到支持，没有删除的计划。这两种设施被新构造所取代的原因首先是为了与  :pep:`484`  工具（包括诸如 VSCode、Mypy 和 Pyright 的类型检查器）平滑集成，而无需插件。其次，从类型注释派生声明是 SQLAlchemy 与 Python dataclasses 集成的一部分，现在可以从映射中   :ref:` 原生生成 <orm_declarative_native_dataclasses>` 。

   对于喜欢“旧”方式的用户，但仍希望 IDE 不会错误地报告其 Declarative 映射的类型错误的情况，  :func:`_orm.mapped_column`  构造是 ORM Declarative 映射中   :class:` .Column`  的一种可替换方案（注意，  :func:`_orm.mapped_column`  仅用于 ORM Declarative 映射；它不能在   :class:` .Table`  构造中使用），类型注释是可选的。我们上面的映射可以这样写，不带注释：

        class User(Base):
            __tablename__ = "user_account"

            id = mapped_column(Integer, primary_key=True)
            name = mapped_column(String(30), nullable=False)
            fullname = mapped_column(String)

            addresses = relationship("Address", back_populates="user")

            # ... definition continues

   上面的类比直接使用   :class:`.Column`  有一个优势，即 ` `User`` 类以及 ``User`` 的实例将向类型工具指示正确的类型信息，而无需使用插件。   :func:`_orm.mapped_column`  还允许使用其他 ORM 特定参数来配置行为，例如延迟列加载，这之前需要使用   :class:` _schema.Column`  和   :func:`_orm.deferred`  两个单独的函数。

   还有一个将旧式 Declarative 类转换为新式的例子，可以参见   :ref:`whatsnew_20_orm_declarative_typing`  章节中的   :ref:` whatsnew_20_toplevel`  指南。

.. seealso::

     :ref:`orm_mapping_styles`  - ORM 配置样式的全面背景。

     :ref:`orm_declarative_mapping`  - Declarative 类映射的概述

     :ref:`orm_declarative_table`  - 有关如何使用   :func:` _orm.mapped_column`  和   :class:`_orm.Mapped`  定义要映射的   :class:` _schema.Table`  中的列时，使用 Declarative 的详细信息。从ORM映射向数据库发出DDL
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

由于ORM映射的类引用了包含在  :class:`_schema.MetaData`  属性访问集合，然后像以前一样使用  :meth:` _schema.MetaData.create_all`  。在这种情况下，运行PRAGMA语句，但由于它们已经存在，因此不会生成新表：

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

.. topic:: 可选部分

    这个部分只是关于相关主题“表反射”的一个简要介绍，或者如何从现有的数据库自动地生成 :class:`_schema.Table` 对象。想要写查询的教程读者可以自由地跳过本节。

为了完成有关处理表元数据的部分，我们将演示另一种在该部分开头提到的操作，即**表反射**。表反射指的是通过读取数据库的当前状态来生成 :class:`_schema.Table` 和相关对象的过程。在以前的部分中，我们在Python中声明了 :class:`_schema.Table` 对象，然后我们可以选择发出DDL到数据库以生成这样的架构，而反射过程则在此过程的两个步骤中以相反的顺序执行，从现有的数据库开始生成Python内的数据结构，以表示该数据库中的模式。

.. tip::  反射不必使用以前存在的数据库，在使用SQLAlchemy时强制使用。 SQLAlchemy应用程序完全典型，它在Python中明确声明了所有元数据，以便其结构对应于现有数据库的结构。元数据结构也不必包括在现有数据库中不需要的表、列或其他约束和构造。

作为反射的示例，我们将创建一个新的  :class:`_schema.Table`  传递目标: class:` _engine.Engine`：

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

在此过程结束时，“some_table”对象现在包含关于存在于表中的 :class:`_schema.Column` 对象的信息，并且对象可以按与我们明确声明的 :class:`_schema一样使用，` _schema.Table`以完全相同的方式。：

    >>> some_table
    Table('some_table', MetaData(),
        Column('x', INTEGER(), table=<some_table>),
        Column('y', INTEGER(), table=<some_table>),
        schema=None)

.. seealso::

    在   :ref:` metadata_reflection_toplevel`  中阅读有关表和模式反射的更多信息。

    对于与ORM相关的表反射变体，部分 :ref:`orm_declarative_reflected` 包括可用选项的概述。

下一步
----------

我们现在拥有一个SQLite数据库，其中包含两个表，并具有可以使用 :class:`_engine.Connection` 和/或ORM :class:`_orm.Session` 来与这些表进行交互的Core和ORM面向表的结构。在以下部分中，我们将说明如何使用这些结构来创建、操作和选择数据。