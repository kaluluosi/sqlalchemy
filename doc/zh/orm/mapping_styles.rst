.. _orm_mapping_classes_toplevel:

==========================
ORM映射类概述
==========================

ORM类映射配置概述。

对于不熟悉SQLAlchemy ORM和/或不熟悉Python的读者，建议浏览 :ref:`orm_quickstart`，
最好通过 :ref:`unified_tutorial` 学习，其中ORM配置首先在 :ref:`tutorial_orm_table_metadata`
中介绍。

.. _orm_mapping_styles:

ORM映射样式
==================

SQLAlchemy具有两种不同的映射配置样式，然后根据如何设置它们进一步分为子选项。
映射器样式的可变性在于满足不同的开发人员喜好，包括用户定义的类从被映射为关系模式的表和列中的程度抽象，
使用的类层次结构种类，包括是否存在自定义元类方案，最后如果同时使用Python DataClasses等其他类检测方法。

在现代SQLAlchemy中，这些样式之间的差异基本上是表面的；当使用特定的SQLAlchemy配置样式表达映射类意图时，
映射类的内部映射过程在很大程度上是相同的，最终结果始终是一个用户定义的类，该类具有配置的:class:`_orm.Mapper`，
该映射关联了一个可选单元，通常表示为 :class:`_schema.Table` 对象，并且该类本身已经被包含关系操作的行为链接在类和类实例级别。
由于这个过程在所有情况下基本上是相同的，从不同样式映射的类始终是完全互通的。

原始的映射API通常被称为“classical”样式，而更自动化的映射样式称为“declarative”样式。
SQLAlchemy现在将这两种映射样式称为**imperative mapping**和**declarative mapping**。

无论使用什么样式映射，从SQLAlchemy 1.4开始，所有ORM映射都起源于一个称为 :class:`_orm.registry` 的单个对象，
它是一组已映射的类的注册表。使用此注册表，可以作为一组完成一组映射器配置，并且特定注册表中的类可以在配置过程中按名称相互引用。

.. versionchanged:: 1.4 意味着声明式和经典映射现在被称为“声明式”和“命令式”映射，并且在内部统一，
   始终可以从表示相关映射的 :class:`_orm.registry` 构造中获得。

.. _orm_declarative_mapping:

声明式映射
-------------------

**声明式映射**是在现代SQLAlchemy中构建映射的典型方式。 最常见的模式是首先使用 :class:`_orm.DeclarativeBase` 超类构建基类。
生成的基类，在子类化时将应用相对于默认情况下本地 :class:`_orm.registry` 上的所有子类的声明性映射过程。
下面的示例说明了使用声明性基础构建的示例映射类::

    from sqlalchemy import Integer, String, ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    # 生命基础类
    class Base(DeclarativeBase):
        pass


    # 使用基类的示例映射
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        fullname: Mapped[str] = mapped_column(String(30))
        nickname: Mapped[Optional[str]]

上面的代码中使用了 :class:`_orm.DeclarativeBase` 类生成了一个新的基类(在SQLAlchemy的文档中通常称为 `Base`)，
从这个基类中，可以生成新的映射类 ``User``。

.. versionchanged:: 2.0 :class:`_orm.DeclarativeBase` 超类取代了使用 :func:`_orm.declarative_base` 函数
   和 :meth:`_orm.registry.generate_base` 方法; 超类方法集成了 :pep:`484` 工具，
   而不使用插件。有关办法详见 :ref:`whatsnew_20_orm_declarative_typing` 中的迁移说明。

基类引用一个 :class:`_orm.registry` 对象来维护一组相关映射类。以及一个 :class:`_schema.MetaData` 对象，该对象存储
:class:`_schema.Table` 对象集合，这些表将被映射到类。

主要的声明式映射样式在以下部分中进一步详细说明：

* :ref:`orm_declarative_generated_base_class` - 使用基类进行声明性映射。

* :ref:`orm_declarative_decorator` - 使用装饰器进行声明性映射，而不是基类。

在声明性映射类的范围内，还有两种声明 :class:`_schema.Table` 元数据的方式。包括:

* :ref:`orm_declarative_table` - 在映射类中使用 :func:`_orm.mapped_column` 指令直接声明
  表列（或使用 :class:`_schema.Column` 对象的遗留形式）。
  :func:`_orm.mapped_column` 指令也可以使用 :class:`_orm.Mapped` 类型的类型注释可选地与
  映射列组合使用，可以直接提供有关映射列的一些详细信息。列指令结合使用 ``__tablename__``
  和可选的 ``__table_args__`` 类级指令将允许声明性映射过程构建 :class:`_schema.Table` 对象进行映射。

* :ref:`orm_imperative_table_configuration` - 与其将表名和属性分别指定，
  不如将一个明确定义的 :class:`_schema.Table` 对象与以声明方式映射的类相关联。
  这种映射样式是“声明式”和“命令式”映射的混合体，并适用于将类映射到 :term:`reflected`
  :class:`_schema.Table` 对象以及将类映射到现有的 Core 构造（例如连接和子查询）的技术。

对于 :ref:`declarative_config_toplevel` 的声明式映射的文档继续。

.. _classic_mapping:
.. _orm_imperative_mapping:

命令式映射
-------------------

**命令式**或**经典的**映射是指使用 :meth:`_orm.registry.map_imperatively` 方法配置映射类的
映射配置，其中目标类不包括任何声明性的类属性。

.. tip:: 命令式映射形式是一种很少使用的映射形式，起源于2006年的最初版本。 它基本上是一种绕过声明性系统提供的“骨架”映射的方法，
   并且不像现代的特性(如 :pep:`484`支持) 。因此，大多数文档示例使用声明式形式，建议新用户从 :ref:`declarative_table
   <orm_declarative_table_config_toplevel>` 配置开始。

.. versionchanged:: 2.0 现在使用 :meth:`_orm.registry.map_imperatively` 方法来创建经典映射， ``sqlalchemy.orm.mapper()`` 独立函数已经被删除。

以“classical”形式创建的表元数据将与 :class:`_schema.Table` 构造分开创建，然后通过 :meth:`_orm.registry.map_imperatively` 方法
与 ``User`` 类关联，在创建 :class:`_orm.registry` 实例后与其中的所有映射类共享单个实例::

    from sqlalchemy import Table, Column, Integer, String, ForeignKey
    from sqlalchemy.orm import registry

    mapper_registry = registry()

    user_table = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )


    class User:
        pass


    mapper_registry.map_imperatively(User, user_table)

提供有关映射属性（例如，与其他类的关系）的信息通过 ``properties`` 字典。 下面的示例说明了第二个 :class:`_schema.Table`
对象，映射到名为“Address”的类，然后通过 :func:`_orm.relationship` 与 ``User`` 类链接::

    address = Table(
        "address",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )

    mapper_registry.map_imperatively(
        User,
        user,
        properties={
            "addresses": relationship(Address, backref="user", order_by=address.c.id)
        },
    )

    mapper_registry.map_imperatively(Address, address)

注意，使用命令式方法映射的类与使用声明性方法映射的类是**完全可互换的**。 两个系统最终创建相同的配置，包括一个 :class:`_schema.Table`，
一个链接到该表的用户定义的类，以及一个 :class:`_orm.Mapper` 对象。 当我们谈论“ :class:`_orm.Mapper` 行为” 时，这也适用于使用
声明式系统，只是在幕后使用 - 它仍然在使用。

.. _orm_mapper_configuration_overview:

映射类的必要组件
==================================

使用 :class:`_orm.Mapper` 的构造参数，可以通过传递构造参数的方式来配置映射类的映射，
这些参数最终通过其构造函数成为 :class:`_orm.Mapper` 对象的一部分。 传递给 :class:`_orm.Mapper` 的参数
来源于给定的映射形式，包括对于 :ref:`orm_imperative_mapping` 的 :meth:`_orm.registry.map_imperatively` 传递的参数，
如果使用声明性系统的话，源自表列、SQL表达式和映射的关系以及例如 :ref:`__mapper_args__ <orm_declarative_mapper_options>`
等属性的组合。

:class:`_orm.Mapper` 类正在寻找的配置信息有四个一般类别：

要映射的类
----------------------

这是我们在应用程序中构造的类。一般情况下，这个类的结构没有限制。 [1]_ Python映射类时，每个类只有**一个** :class:`_orm.Mapper`
对象。[2]_

使用 :ref:`declarative <orm_declarative_mapping>` 映射样式时，要映射的类是声明性基类的子类，
或者由通过像 :meth:`_orm.registry.mapped` 这样的装饰器或函数处理。

使用 :ref:`imperative <orm_imperative_mapping>` 样式映射时，类直接作为
:paramref:`_orm.registry.map_imperatively.class_` 参数传递。

表或其他从句对象
--------------------------------------

在绝大多数常见情况下，这是 :class:`_schema.Table` 的实例。更高级的使用情况下，
也可以引用任何类型的 :class:`_sql.FromClause` 对象，最常见的替代对象是 :class:`_sql.Subquery`
和 :class:`_sql.Join` 对象。

使用 :ref:`declarative <orm_declarative_mapping>` 映射样式时，主题表是基于 ``__tablename__`` 属性和
呈现的 :class:`_schema.Column` 对象生成的声明性系统，或者它通过 ``__table__`` 属性建立。这两种配置样式的详细信息在
 :ref:`orm_declarative_table` 和 :ref:`orm_imperative_table_configuration` 中提供。

使用 :ref:`imperative <orm_imperative_mapping>` 样式映射时，主题表作为
:paramref:`_orm.registry.map_imperatively.local_table` 参数位置传递。

与映射类“一对一映射”的要求相反， :class:`_schema.Table` 或其他 :class:`_sql.FromClause` 对象是映射的个数没有任何限制的。
:class:`_orm.Mapper` 直接应用于用户定义的类，但不对给定的 :class:`_schema.Table` 或其他 :class:`_sql.FromClause` 进行任何修改。

.. _orm_mapping_properties:

properties字典
-------------------------

这是将与映射类关联的所有属性的字典。默认情况下， :class:`_orm.Mapper` 会从给定 :class:`_schema.Table` 表格中生成
此字典的条目，形成 :class:`_orm.ColumnProperty` 对象，它们各自指的是映射表的特定 :class:`_schema.Column`。
properties字典还将包含所有其他种类的要在其中配置的 :class:`_orm.MapperProperty` 对象，最常见的是由 :func:`_orm.relationship`
构建的实例。


使用 :ref:`declarative <orm_declarative_mapping>` 映射样式时，属性字典由声明性系统生成，
通过扫描要映射的类寻找适当的属性。请参阅 :ref:`orm_declarative_properties` 部分以获取有关此过程的提示。

使用 :ref:`imperative <orm_imperative_mapping>` 样式映射时，属性字典作为
``properties`` 参数直接传递到 :meth:`_orm.registry.map_imperatively`，然后传递到 :paramref:`_orm.Mapper.properties` 参数中。

其他映射器配置参数
-------------------------------------

使用 :ref:`declarative <orm_declarative_mapping>` 映射样式映射时，其他映射器配置参数通过 ``__mapper_args__`` 类属性进行配置。
使用示例可以在 :ref:`orm_declarative_mapper_options` 中找到。

使用 :ref:`imperative <orm_imperative_mapping>` 样式映射时，关键字参数直接传递到 :meth:`_orm.registry.map_imperatively`
的 :class:`_orm.Mapper` 类中。

可以接受的完整参数范围在 :class:`_orm.Mapper` 中记录。


.. _orm_mapped_class_behavior:


映射类的行为
=====================

在使用 :class:`_orm.registry` 的所有映射样式中，以下行为是常见的：

.. _mapped_class_default_constructor:

默认的构造函数
-------------------

:class:`_orm.registry` 会向所有没有明确自己的 ``__init__`` 方法的映射类应用默认构造函数，即 ``__init__`` 方法。
这个方法的行为是为所有命名属性提供一个方便的关键字构造函数。例如::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        fullname: Mapped[str]

上面的代码将创建一个允许创建 ``User`` 对象的关键字构造函数的相应对象::

    u1 = User(name="some name", fullname="some fullname")

.. tip::

    :ref:`orm_declarative_native_dataclasses` 特性提供了一种有别的，使用Python dataclasses 可以生成默认 ``__init__()`` 方法的方式，
    并且允许高度配置的构造函数表单。

包含明确的 ``__init__()`` 方法的类将保持该方法，而不会应用默认构造函数。

更改默认构造函数的使用方式，用户定义Python可调用对象可以提供给 :paramref:`_orm.registry.constructor` 参数，该参数将用作默认构造函数。

在经典映射以及依据 :meth:`_orm.registry.map_imperatively` 映射的情况下，映射的类也会应用默认的构造函数。

.. versionadded:: 1.4 经典映射现在在映射于 :meth:`_orm.registry.map_imperatively` 方法的情况下支持标准的构造函数形式。

.. _orm_mapper_inspection:

映射器对象的运行时自省
---------------------------------------------------------------

使用 :class:`_orm.registry` 所映射的类还将具有一些对所有映射的类常见的属性：

* ``__mapper__`` 属性将引用与类相关联的 :class:`_orm.Mapper`::

    mapper = User.__mapper__

  当使用 :func:`_sa.inspect` 函数访问映射类时，也会返回相应的 :class:`_orm.Mapper` 对象::

    from sqlalchemy import inspect

    mapper = inspect(User)

  ..

* ``__table__`` 属性将引用为其映射的 :class:`_schema.Table`，或者
  更通用地，引用任何类型的 :class:`_sql.FromClause` 对象。

    table = User.__table__

  当使用 :class:`_orm.Mapper` 的 :attr:`_orm.Mapper.local_table` 属性时，也会返回
  相同的 ``.__table__`` 属性引用的对象::

    table = inspect(User).local_table

  对于一对多的继承映射，在类是一个不具有自己表的子类的情况下，``.__table__`` 属性以及
  :attr:`_orm.Mapper.local_table` 属性的返回值将为 ``None``。 要检索查询期间实际选择的 “selectable” 对象，可以通过 :attr:`_orm.Mapper.selectable` 属性获得::

    table = inspect(User).selectable

  ..

.. _orm_mapper_inspection_mapper:

映射器对象的自省
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

正如前一节所示，可以从任何映射类中获取 :class:`_orm.Mapper` 对象，而不管方法如何，使用 :ref:`core_inspection_toplevel` 系统。
使用 :func:`_sa.inspect` 函数，可以从映射类获得 :class:`_orm.Mapper` 对象::

    >>> from sqlalchemy import inspect
    >>> insp = inspect(User)

可以获得的详细信息包括 :attr:`_orm.Mapper.column`::

    >>> insp.columns
    <sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>

这是一个命名空间，可以通过列表格式或通过个体名称查看::

    >>> list(insp.columns)
    [Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
    >>> insp.columns.name
    Column('name', String(length=50), table=<user>)

其他命名空间包括 :attr:`_orm.Mapper.all_orm_descriptors`，其包括所有映射属性以及混合，关系代理::

    >>> insp.all_orm_descriptors
    <sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
    >>> insp.all_orm_descriptors.keys()
    ['fullname', 'nickname', 'name', 'id']

以及 :attr:`_orm.Mapper.column_attrs`::

    >>> list(insp.column_attrs)
    [<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
    >>> insp.column_attrs.name
    <ColumnProperty at 0x10403fce8; name>
    >>> insp.column_attrs.name.expression
    Column('name', String(length=50), table=<user>)

.. seealso::

    :class:`.Mapper`

.. _orm_mapper_inspection_instancestate:

映射实例的自省
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用 :func:`_sa.inspect` 函数还提供有关映射类的实例的信息。当应用于映射类的实例而不是类本身时，返回的对象称为
:class:`.InstanceState`，它将提供一个链接，不仅指向类使用的 :class:`.Mapper`，还提供一个详细的接口，该接口提供有关
实例内部属性状态的信息，包括其当前值以及它与其数据库加载值之间的关系。

给定从数据库中加载的 ``User`` 类的实例::

  >>> u1 = session.scalars(select(User)).first()

:func:`_sa.inspect` 函数将返回 :class:`.InstanceState` 对象::

  >>> insp = inspect(u1)
  >>> insp
  <sqlalchemy.orm.state.InstanceState object at 0x7f07e5fec2e0>

使用这个对象，我们可以看到元素，如:class:`.Mapper`::

  >>> insp.mapper
  <Mapper at 0x7f07e614ef50; User>

以及应用于对象的 :term:`attached` :class:`_orm.Session` (如果有的话)：

  >>> insp.session
  <sqlalchemy.orm.session.Session object at 0x7f07e614f160>

有关该对象当前 :ref:`persistence state <session_object_states>` 的信息::

  >>> insp.persistent
  True
  >>> insp.pending
  False

属于对象中未加载或 :term:`lazy loaded` 的属性的属性状态信息（假定 ``addresses`` 引用到映射类上的一个 :func:`_orm.relationship`）::

  >>> insp.unloaded
  {'addresses'}

有关属性在 Python 中的当前状态的信息，例如自上次刷新以来未被修改的属性::

  >>> insp.unmodified
  {'nickname', 'name', 'fullname', 'id'}

以及自上次刷新以来对属性的修改的详细历史记录::

  >>> insp.attrs.nickname.value
  'nickname'
  >>> u1.nickname = "new nickname"
  >>> insp.attrs.nickname.history
  History(added=['new nickname'], unchanged=(), deleted=['nickname'])

.. seealso::

    :class:`.InstanceState`

    :attr:`.InstanceState.attrs`

    :class:`.AttributeState`


.. _dataclasses: https://docs.python.org/3/library/dataclasses.html

.. [1] 在运行Python 2时，Python 2的“旧式”类是唯一不兼容的类。
       在Python 2上运行代码时，所有类都必须扩展Python ``object`` 类。

.. [2] 有一个称为“非主映射器”的遗留功能，可以将多个 :class:`_orm.Mapper` 对象与已经映射的类关联，
       但是它不提供对类的仪器化。此功能自SQLAlchemy 1.3起已弃用。