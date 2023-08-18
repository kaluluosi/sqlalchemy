=============================
SQLAlchemy 0.8有哪些新特性？
=============================

.. admonition:: 关于本文档

    本文档介绍了SQLAlchemy 0.7版本更新至2012年10月起的其他维护版本和即将于2013年初发布的SQLAlchemy 0.8版本之间的差异及其影响应用迁移。
    
    文档日期：2012年10月25日
    更新日期：2013年3月9日

介绍
============

本指南介绍了SQLAlchemy 0.8版本的新功能，并记录了影响将应用程序从SQLAlchemy 0.7系列迁移到0.8的用户的更改。

SQLAlchemy版本正在逼近1.0版本，自从0.5版本以来，每个新版本都具有更少的重要使用更改。大多数采用现代0.7模式的应用程序应该能够移动到0.8版本，无需更改。使用0.6甚至0.5模式的应用程序也应该直接可以迁移到0.8版本，虽然较大的应用程序可能需要测试每个中间版本。

平台支持
================

针对Python 2.5及以上版本
-------------------------------

SQLAlchemy 0.8将针对Python 2.5及以上版本进行开发，将取消针对Python 2.4的兼容性支持。

内部将能够使用Python三元运算法（即“x if y else z”），这样可以改善与使用“y and x or z”引起的错误，同时支持上下文管理器（即“with:”），在某些情况下还可以支持try:/except:/else:块，这将有助于代码的可读性。

SQLAlchemy最终将取消对2.5版本的支持-当2.6版本成为基线时，SQLAlchemy将转移到使用2.6 / 3.3本地兼容性，并删除“2to3”工具的使用，并维护源代码，在Python 2和3同时运行。

新的ORM功能
================

.. _feature_relationship_08:

重写的 :func:`_orm.relationship` 机制
----------------------------------------------

0.8版具有更强大和更可行的系统，用于确定 :func:`_orm.relationship` 在两个实体之间如何连接。新系统包括以下功能：

* 当使用多个外键路径连接到目标时，在构建 :func:`_orm.relationship` 时，不再需要 ``primaryjoin`` 参数。仅需要 ``foreign_keys`` 参数来指定应包括哪些列：

  ::


        class Parent(Base):
            __tablename__ = "parent"
            id = Column(Integer, primary_key=True)
            child_id_one = Column(Integer, ForeignKey("child.id"))
            child_id_two = Column(Integer, ForeignKey("child.id"))

            child_one = relationship("Child", foreign_keys=child_id_one)
            child_two = relationship("Child", foreign_keys=child_id_two)


        class Child(Base):
            __tablename__ = "child"
            id = Column(Integer, primary_key=True)

* 现在支持自引用，复合外部键的关系，其中**一列指向本身**。典型案例如下所示：

  ::

        class Folder(Base):
            __tablename__ = "folder"
            __table_args__ = (
                ForeignKeyConstraint(
                    ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
                ),
            )

            account_id = Column(Integer, primary_key=True)
            folder_id = Column(Integer, primary_key=True)
            parent_id = Column(Integer)
            name = Column(String)

            parent_folder = relationship(
                "Folder", backref="child_folders", remote_side=[account_id, folder_id]
            )

  上面的 ``Folder`` 引用了其父级 ``Folder``，从 ``account_id`` 到自己，以及从 ``parent_id`` 到 ``folder_id`` 进行连接。当SQLAlchemy构造“自动连接”时，不能再假定“远程”端上的所有列都已被别名化，并且所有“本地”端的列都没有别名- ``account_id`` 列存在于两侧。因此，内部关系机制完全地重写以支持一个完全不同的系统，其中两个包含不同*注释*的 ``account_id`` 副本被生成，每个副本包含不同的内容以确定在语句中的角色。有关基本急切加载中的连接条件：

  .. sourcecode:: sql

        SELECT
            folder.account_id AS folder_account_id,
            folder.folder_id AS folder_folder_id,
            folder.parent_id AS folder_parent_id,
            folder.name AS folder_name,
            folder_1.account_id AS folder_1_account_id,
            folder_1.folder_id AS folder_1_folder_id,
            folder_1.parent_id AS folder_1_parent_id,
            folder_1.name AS folder_1_name
        FROM folder
        LEFT OUTER JOIN folder AS folder_1
            ON folder_1.account_id = folder.account_id
            AND folder.folder_id = folder_1.parent_id
        WHERE folder.folder_id = ? AND folder.account_id = ?

* 以前很难实现的自定义连接条件，例如涉及函数和/或类型转换的条件，现在在大多数情况下可正常使用：

    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() 使用显式外键和 remote_side 
        parent_host = relationship(
            "HostEntry",
            primaryjoin=ip_address == cast(content, INET),
            foreign_keys=content,
            remote_side=ip_address,
        )

  新的 :func:`_orm.relationship` 机制使用 SQLAlchemy 概念称为 :term:`注释（annotations）` 。这些注释也可通过 :func:`.foreign` 和 :func:`.remote` 函数明确提供给应用程序代码，作为提高高级配置的可读性或直接注入精确配置的手段，从而绕过通常的连接检查启发式算法：

    from sqlalchemy.orm import foreign, remote


    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() 使用显式 foreign() 和 remote() 注释 
        # 而不是分开的参数
        parent_host = relationship(
            "HostEntry",
            primaryjoin=remote(ip_address) == cast(foreign(content), INET),
        )

.. 参见::

    :ref:`relationship_configure_joins` - 新修订了 :func:`_orm.relationship` 的一部分，详细介绍了定制相关属性和集合访问的最新技术。

:ticket:`1401` :ticket:`610`

.. _feature_orminspection_08:

新的类/对象检查系统
----------------------------------------

许多SQLAlchemy用户正在编写需要检查映射类的属性的系统，包括能够获取主键列、对象关系、普通属性等等，通常用于构建数据封送系统，如JSON/XML转换方案和当然还有表单库等等。

最初， :class:`_schema.Table` 和 :class:`_schema.Column` 模型是原始检查点，具有良好记录的系统。虽然ORM模型也是完全可检查的，但这从来不是完全稳定和受支持的特性，用户倾向于不知道如何获取此信息。

0.8现在为此提供了一致，稳定且完全记录的API，包括一组可用于映射类别，实例，属性和其他核心和ORM结构的检查系统。这个系统的入口点是核心级 :func:`_sa.inspect` 函数。在大多数情况下，正在检查的对象是SQLAlchemy系统的一部分，例如 :class:`_orm.Mapper`，:class:`.InstanceState`，:class:`_reflection.Inspector`。在某些情况下，已添加具有提供检查API的作业的新对象，在特定上下文中，例如 :class:`.AliasedInsp` 和 :class:`.AttributeState`。

以下是一些关键功能的说明：

.. sourcecode:: pycon+sql

    >>> class User(Base):
    ...     __tablename__ = "user"
    ...     id = Column(Integer, primary_key=True)
    ...     name = Column(String)
    ...     name_syn = synonym(name)
    ...     addresses = relationship("Address")

    >>> # 通用入口点是inspect()
    >>> b = inspect(User)

    >>> # b在这种情况下是Mapper
    >>> b
    <Mapper at 0x101521950; User>

    >>> # Column namespace
    >>> b.columns.id
    Column('id', Integer(), table=<user>, primary_key=True, nullable=False)

    >>> # Mapper从.attrs中获取的属性
    >>> b.attrs.keys()
    ['name_syn', 'addresses', 'id', 'name']

    >>> # .column_attrs，.relationships等过滤此集合
    >>> b.column_attrs.keys()
    ['id', 'name']

    >>> list(b.relationships)
    [<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>]

    >>> # 他们也是命名空间
    >>> b.column_attrs.id
    <sqlalchemy.orm.properties.ColumnProperty object at 0x101525090>

    >>> b.relationships.addresses
    <sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>

    >>> # 将inspect()指向映射后的类级属性，返回属性本身
    >>> b = inspect(User.addresses)
    >>> b
    <sqlalchemy.orm.attributes.InstrumentedAttribute object at 0x101521fd0>

    >>> # 从这里可以获取mapper：
    >>> b.mapper
    <Mapper at 0x101525810; Address>

    >>> # 父检查器，在这种情况下是mapper
    >>> b.parent
    <Mapper at 0x101521950; User>

    >>> # 表达式
    >>> print(b.expression)
    {printsql}"user".id = address.user_id{stop}

    >>> # inspect()适用于实例
    >>> u1 = User(id=3, name="x")
    >>> b = inspect(u1)

    >>> # 它返回InstanceState
    >>> b
    <sqlalchemy.orm.state.InstanceState object at 0x10152bed0>

    >>> # 类似的 attrs，指的是状态对象
    >>> b.attrs.keys()
    ['id', 'name_syn', 'addresses', 'name']

    >>> # 属性接口-从 attrs 中，您可以获得一个状态对象
    >>> b.attrs.id
    <sqlalchemy.orm.state.AttributeState object at 0x10152bf90>

    >>> # 此对象可以提供当前值...
    >>> b.attrs.id.value
    3

    >>> # ...目前的历史
    >>> b.attrs.id.history
    History(added=[3], unchanged=(), deleted=())

    >>> # InstanceState 还可以提供会话状态信息
    >>> # 假设对象是持久的
    >>> s = Session()
    >>> s.add(u1)
    >>> s.commit()

    >>> # 现在我们可以始终获取主键身份
    >>> # 总是在 query.get() 中运行
    >>> b.identity
    (3,)

    >>> # 映射器层面的密钥
    >>> b.identity_key
    (<class '__main__.User'>, (3,))

    >>> # 会话中的状态
    >>> b.persistent, b.transient, b.deleted, b.detached
    (True, False, False, False)

    >>> # 拥有的会话
    >>> b.session
    <sqlalchemy.orm.session.Session object at 0x101701150>

.. 参见::

    :ref:`core_inspection_toplevel`

:ticket:`2208`

可以将ORM类别用于核心构造
-------------------------------------------

虽然 :meth:`_query.Query.filter` 中使用的SQL表达式，如 ``User.id == 5``，对于 :func:`_expression.select` 等核心构造始终是兼容的，但映射的类本身在传递给 :func:`_expression.select`，:meth:`_expression.Select.select_from` 或 :meth:`_expression.Select.correlate` 时将无法识别。现在，新的SQL注册系统允许将映射类作为CORE中的FROM子句：

    from sqlalchemy import select

    stmt = select([User]).where(User.id == 5)

在上面的示例中，映射的 ``User`` 类将会被扩展为与其映射的 :class:`_schema.Table`。

:ticket:`2245`

.. _change_orm_2365:

Query.update() 支持 UPDATE..FROM
-----------------------------------

现在支持在 query.update() 中在更新 ``SomeEntity`` 时添加 FROM 子句（或等效的方式依赖于后端）添加至 ``SomeOtherEntity``：

    query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
        SomeOtherEntity.foo == "bar"
    ).update({"data": "x"})

特别地，如果更新的目标本地于在筛选的表上，或者如果父级和子级表是混合的，它们在查询中显式地连接，则形成联接继承实体的更新将受到支持。在下面的示例中，假设 ``Engineer`` 作为 `Person ``的连续子类：

::

    query(Engineer).filter(Person.id == Engineer.id).filter(
        Person.name == "dilbert"
    ).update({"engineer_data": "java"})

将会产生：

.. sourcecode:: sql

    UPDATE engineer SET engineer_data='java' FROM person
    WHERE person.id=engineer.id AND person.name='dilbert'

:ticket:`2365`

rollback() 仅回滚begin_nested（）中的“dirty”对象
-----------------------------------------------------

针对使用 ``Session.begin_nested()`` 的SAVEPOINT的用户，将在 ``rollback()`` 时仅过期那些自上次刷新后变脏的对象，而会话的其余部分仍然完好无损。这是因为ROLLBACK到SAVEPOINT并不会终止包含事务的隔离，因此除了那些没有在当前事务中刷新的更改之外，不需要过期。这将提高工作效率。

:ticket:`2452`

缓存示例现在使用dogpile.cache
-----------------------------------

缓存示例现在使用 `dogpile.cache <https://dogpilecache.readthedocs.io/>`_。Dogpile.cache是Beaker缓存部分的重写，具有更简单，更快的操作以及分布式锁定支持。

请注意，Dogpile示例以及之前的Beaker示例所使用的SQLAlchemy API已略有改变，特别是在Beaker示例中需要进行以下更改：

.. sourcecode:: diff

    --- examples/beaker_caching/caching_query.py
    +++ examples/beaker_caching/caching_query.py
    @@ -222,7 +222,8 @@

             """
             if query._current_path:
    -            mapper, key = query._current_path[-2:]
    +            mapper, prop = query._current_path[-2:]
    +            key = prop.key

                 for cls in mapper.class_.__mro__:
                     if (cls, key) in self._relationship_options:

.. 参见::

    :ref:`examples_caching`

:ticket:`2589`

新的CORE功能
================

Core中完全可扩展的类型级运算符支持
-----------------------------------------------------

到目前为止，Core从未具有为Column和其他表达式结构添加支持新SQL运算符的任何系统，除了 :meth:`.ColumnOperators.op` 方法“只是足够”可以使其正常工作。在Core中还从未有过任何可以允许覆盖现有运算符行为的系统。迄今为止，唯一可以灵活地重新定义运算符的方式是使用ORM层，使用 :func:`.column_property` 并给定 ``comparator_factory`` 参数。因此，第三方库如GeoAlchemy被强制为ORM-centric，并依赖于一系列hack来应用新操作，以及使它们正确传播。

Core中的新运算符系统添加了一直缺失的一个钩子，即将新的和覆盖的运算符与*类型*关联起来。毕竟，实际上*不是*一个列，CAST运算符或SQL函数真正驱动可用的操作类型的种类。实现细节很少-只添加了一些额外的方法到核心 :class:`_expression.ColumnElement` 类型，以便在核心中的 :class:`.TypeEngine` 对象中查看可选运算符集。新的或修订的操作可以与任何类型关联，无论是通过现有类型的子类化，使用 :class:`.TypeDecorator`，或通过将新的 :class:`.TypeEngine.Comparator` 对象附加到现有类型类“全局跨越。” 

例如，要将logarithm支持添加到 :class:`.Numeric` 类型：

::

    from sqlalchemy.types import Numeric
    from sqlalchemy.sql import func


    class CustomNumeric(Numeric):
        class comparator_factory(Numeric.Comparator):
            def log(self, other):
                return func.log(self.expr, other)

可以像任何其他类型一样使用新类型：

::


    data = Table(
        "data",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("x", CustomNumeric(10, 5)),
        Column("y", CustomNumeric(10, 5)),
    )

    stmt = select([data.c.x.log(data.c.y)]).where(data.c.x.log(2) < value)
    print(conn.execute(stmt).fetchall())

新功能立即提供的新特性包括对PostgreSQL's HSTORE类型的支持，以及与PostgreSQL's ARRAY类型相关的新操作。它还为现有类型铺平了道路，使其获得更多针对那些类型特定的更多的运算符，例如更多字符串，整数和日期运算符。

.. 参见::

    :ref:`types_operators`

    :class:`.HSTORE`

:ticket:`2547`

.. _feature_2623:

支持Insert的多重VALUES
--------------------------------------------------

:meth:`_expression.Insert.values` 方法现在支持字典列表，这将呈现出multi-VALUES语句，例如 ``VALUES (<row1>), (<row2>), ...``。这仅涉及支持这种语法的后端，包括PostgreSQL，SQLite和MySQL。它与通常的 ``executemany()`` 样式的INSERT不同，它保持不变：

    users.insert().values(
        [
            {"name": "some name"},
            {"name": "some other name"},
            {"name": "yet another name"},
        ]
    )

.. 参见::

    :meth:`_expression.Insert.values`

:ticket:`2623`

类型表达式
----------------

现在可以将SQL表达式与类型相对应。 历史上， :class:`.TypeEngine` 总是允许Python侧函数，其接收绑定参数以及结果行值，并在通过Python侧转换函数时将其传递到/从数据库。新功能允许在数据库方面进行类似功能，但是它要将SQL表达式与类型相关联：

    from sqlalchemy.types import String
    from sqlalchemy import func, Table, Column, MetaData


    class LowerString(String):
        def bind_expression(self, bindvalue):
            return func.lower(bindvalue)

        def column_expression(self, col):
            return func.lower(col)


    metadata = MetaData()
    test_table = Table("test_table", metadata, Column("data", LowerString))

上述 ``LowerString`` 类型定义了一个SQL表达式，每当在SELECT语句的列子句中呈现“test_table.c.data”列时，就会发出该表达式：

.. sourcecode:: pycon+sql

    >>> print(select([test_table]).where(test_table.c.data == "HI"))
    {printsql}SELECT lower(test_table.data) AS data
    FROM test_table
    WHERE test_table.data = lower(:data_1)

此功能也被新版本的GeoAlchemy广泛使用，以根据类型规则内联嵌入PostGIS表达式。

.. 参见::

    :ref:`types_sql_value_processing`

:ticket:`1534`

Core检查系统
------------------

在 :ref:`feature_orminspection_08` 中引入的 :func:`_sa.inspect` 函数，现在也适用于Core。应用于 :class:`_engine.Engine` 它会产生 :class:`_reflection.Inspector` 对象：

    from sqlalchemy import inspect
    from sqlalchemy import create_engine

    engine = create_engine("postgresql://scott:tiger@localhost/test")
    insp = inspect(engine)    print(insp.get_table_names())

它还可以应用于任何 :class:`_expression.ClauseElement`，它返回 :class:`_expression.ClauseElement` 本身，例如 :class:`_schema.Table`、:class:`_schema.Column`、:class:`_expression.Select` 等。这使得它可以在 Core 和 ORM 结构之间流畅地工作。


新的方法 :meth:`_expression.Select.correlate_except`
-------------------------------------------------------
现在，:func:`_expression.select` 有一个方法 :meth:`_expression.Select.correlate_except`，它指定“对除特定选择之外的所有 FROM 子句进行关联”。它可用于映射场景，其中一个相关子查询应该正常关联，除了特定的目标可选择的子句：

    class SnortEvent(Base):
        __tablename__ = "event"

        id = Column(Integer, primary_key=True)
        signature = Column(Integer, ForeignKey("signature.id"))

        signatures = relationship("Signature", lazy=False)


    class Signature(Base):
        __tablename__ = "signature"

        id = Column(Integer, primary_key=True)

        sig_count = column_property(
            select([func.count("*")])
            .where(SnortEvent.signature == id)
            .correlate_except(SnortEvent)
        )

.. 参见::

    :meth:`_expression.Select.correlate_except`

PostgreSQL HSTORE 类型
----------------------

现在可以使用 PostgreSQL 的 ``HSTORE`` 类型，例如 :class:`_postgresql.HSTORE`。该类型充分利用了新的运算符系统，为 HSTORE 类型提供了全面的运算符范围，包括索引访问、连接和包含方法，例如 :meth:`~.HSTORE.comparator_factory.has_key`、:meth:`~.HSTORE.comparator_factory.has_any` 和 :meth:`~.HSTORE.comparator_factory.matrix`：

    from sqlalchemy.dialects.postgresql import HSTORE

    data = Table(
        "data_table",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("hstore_data", HSTORE),
    )

    engine.execute(select([data.c.hstore_data["some_key"]])).scalar()

    engine.execute(select([data.c.hstore_data.matrix()])).scalar()

.. 参见::

    :class:`_postgresql.HSTORE`

    :class:`_postgresql.hstore`

:ticket:`2606`


增强的 PostgreSQL ARRAY 类型
-----------------------------

:class:`_postgresql.ARRAY` 类型将接受一个名为“维度”的可选参数，将其固定为一定数量的维度，并在检索结果时大大提高效率：

::

    # 老方法，因为 PG 支持每行 N 维度：
    Column("my_array", postgresql.ARRAY(Integer))

    # 新方法，将在 DDL 中呈现带有正确数量的 [] 的 ARRAY，
    # 由于我们不需要猜测要向哪里深入，因此将更有效地处理绑定和结果
    Column("my_array", postgresql.ARRAY(Integer, dimensions=2))

该类型还引入了新运算符，使用新的特定于类型的 operator 框架。新操作包括索引访问：

    result = conn.execute(select([mytable.c.arraycol[2]]))

在 SELECT 中的切片访问：

    result = conn.execute(select([mytable.c.arraycol[2:4]]))

在 UPDATE 中的切片更新：

    conn.execute(mytable.update().values({mytable.c.arraycol[2:3]: [7, 8]}))

自由数组字面量：

    >>> from sqlalchemy.dialects import postgresql
    >>> conn.scalar(select([postgresql.array([1, 2]) + postgresql.array([3, 4, 5])]))
    [1, 2, 3, 4, 5]

数组连接，其中下面的右侧 “[4, 5, 6]” 被强制转换为数组文字：

    select([mytable.c.arraycol + [4, 5, 6]])

.. 参见::

    :class:`_postgresql.ARRAY`

    :class:`_postgresql.array`

:ticket:`2441`

适用于 SQLite 的新可配置 DATE、TIME 类型
---------------------------------------------

SQLite 没有内置的 DATE、TIME 或 DATETIME 类型，而是提供了一些支持以字符串或整数格式存储日期和时间值的工具。在 0.8 中，SQLite 的日期和时间类型大大提高了配置性，包括“微秒”部分是可选的，以及几乎所有内容都可以配置。

::

    Column("sometimestamp", sqlite.DATETIME(truncate_microseconds=True))
    Column(
        "sometimestamp",
        sqlite.DATETIME(
            storage_format=(
                "%(year)04d%(month)02d%(day)02d"
                "%(hour)02d%(minute)02d%(second)02d%(microsecond)06d"
            ),
            regexp="(\d{4})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\d{6})",
        ),
    )
    Column(
        "somedate",
        sqlite.DATE(
            storage_format="%(month)02d/%(day)02d/%(year)04d",
            regexp="(?P<month>\d+)/(?P<day>\d+)/(?P<year>\d+)",
        ),
    )

感谢 Nate Dub 在 Pycon 2012 上的 sprint。

.. 参见::

    :class:`_sqlite.DATETIME`

    :class:`_sqlite.DATE`

    :class:`_sqlite.TIME`

:ticket:`2363`

在所有方言上支持“COLLATE”；特别是在 MySQL、PostgreSQL 和 SQLite 中
----------------------------------------------------------------------------

“collate”关键字在 MySQL 方言上被长期接受，现在已在所有 :class:`.String` 类型上及任何后端上呈现：

.. sourcecode:: pycon+sql

    >>> stmt = select([cast(sometable.c.somechar, String(20, collation="utf8"))])
    >>> print(stmt)
    {printsql}SELECT CAST(sometable.somechar AS VARCHAR(20) COLLATE "utf8") AS anon_1
    FROM sometable

.. 参见::

    :class:`.String`

:ticket:`2276`

为 :func:`_expression.update`、:func:`_expression.delete` 启用“前缀”
-----------------------------------------------------------------------
面向 MySQL，可以在任何这些结构中呈现前缀。“a=1”这样的比较就变成了“a IN (1,)”。
例如：

    stmt = table.delete().prefix_with("LOW_PRIORITY", dialect="mysql")


    stmt = table.update().prefix_with("LOW_PRIORITY", dialect="mysql")

该方法是新添加的，另外加入了一些已经存在的方法，
如 :func:`_expression.insert`、:func:`_expression.select` 和 :class:`_query.Query`

.. 参见::

    :meth:`_expression.Update.prefix_with`

    :meth:`_expression.Delete.prefix_with`

    :meth:`_expression.Insert.prefix_with`

    :meth:`_expression.Select.prefix_with`

    :meth:`_query.Query.prefix_with`

:ticket:`2431`

行为变化
==========
::


legacy_is_orphan_addition：

将“挂起”对象视为“孤立”对象的考虑更加积极
----------------------------------------------------

这是 0.8 系列的一个迟到的添加，但是希望新行为通常在更广泛的情况下更一致和直观。ORM 从至少版本 0.4 开始就包含行为，使得“挂起”的对象（即它与 :class:`.Session` 关联，但尚未插入到数据库中的对象）在成为“孤儿”时，在断开与引用它的父对象的关系（在配置的 :func:`_orm.relationship` 上使用“delete-orphan”级联时）时会自动从 :class:`.Session` 中删除，这种行为旨在大约类似于持久（即已插入）对象的行为，其中 ORM 将拦截分离事件并为基于分离事件的孤儿对象发出 DELETE。

该行为适用于从属于多个父级都指定“delete-orphan”级联的任何父级都指定“delete-orphan”级联并且规模为一对多或多对多的 :func:`_orm.relationship`；这是一种尴尬而又无法预知地使用用例（受到一定限制），但尽管如此，该行为仍然希望在对象被部分地关联到要求的父级时能够更一致地操作，而不是需要所有父级都必须置于一个已知的状态。

发生更改的行为适用于以下对象：

    class UserKeyword(Base):
        __tablename__ = "user_keyword"
        user_id = Column(Integer, ForeignKey("user.id"), primary_key=True)
        keyword_id = Column(Integer, ForeignKey("keyword.id"), primary_key=True)

        user = relationship(
            User, backref=backref("user_keywords", cascade="all, delete-orphan")
        )

        keyword = relationship(
            "Keyword", backref=backref("user_keywords", cascade="all, delete-orphan")
        )

在以前的行为中，在所有父对象上均未放置该挂起的对象时，该对象将被删除，并且我的结果不同，如果该挂起的对象已与任何父级关联，将重新关联 :class:`.Session` 。

无论如何，您仍然可以刷新未与所有必需的父项关联的对象（即，如果该对象在首次关联时未与那些父项关联，或者如果该对象被强制删除，但是它后来又通过后续附加事件重新关联了 :class:`.Session`，但仍未完全关联）在此情况下，预计数据库会发出完整性错误，因为可能存在未填充的 NOT NULL 外键列。ORM 做出的决定是让这些 INSERT 尝试发生，基于这样的审判，一个只与某些父级部分关联但已积极关联到其中某些父级的对象更经常是用户错误，而不是意向性遗漏，此时忽略 INSERT 操作是更常见的用户错误，而不是应该默默地跳过--在这里默默跳过 INSERT 将使这种类型的用户错误非常难以调试。

对于可能依赖这种行为的应用程序，可以通过 mapper 选项将旧行为重新启用“legacy_is_orphan”。

新行为使以下测试用例适用：

    from sqlalchemy import Column, Integer, String, ForeignKey
    from sqlalchemy.orm import relationship, backref
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()

    class User(Base):
        __tablename__ = "user"
        id = Column(Integer, primary_key=True)
        name = Column(String(64))

    class UserKeyword(Base):
        __tablename__ = "user_keyword"
        user_id = Column(Integer, ForeignKey("user.id"), primary_key=True)
        keyword_id = Column(Integer, ForeignKey("keyword.id"), primary_key=True)

        user = relationship(
            User, backref=backref("user_keywords", cascade="all, delete-orphan")
        )

        keyword = relationship(
            "Keyword", backref=backref("user_keywords", cascade="all, delete-orphan")
        )

        # 取消注释以启用旧行为
        # __mapper_args__ = {"legacy_is_orphan": True}


    class Keyword(Base):
        __tablename__ = "keyword"
        id = Column(Integer, primary_key=True)
        keyword = Column("keyword", String(64))

    from sqlalchemy import create_engine
    from sqlalchemy.orm import Session

    # note we're using PostgreSQL to ensure that referential integrity
    # is enforced, for demonstration purposes.
    e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)

    Base.metadata.drop_all(e)
    Base.metadata.create_all(e)

    session = Session(e)

    u1 = User(name="u1")
    k1 = Keyword(keyword="k1")

    session.add_all([u1, k1])

    uk1 = UserKeyword(keyword=k1, user=u1)

    # 在以前的行为中，如果在此处调用了 session.flush()，
    # 则此操作将成功，但是如果没有调用 session.flush()，
    # 操作将因完整性错误而失败。
    # session.flush()
    del u1.user_keywords[0]

    session.commit()

:ticket:`2655`

在对象关联到 Session 后发生 after_attach 事件，而不是在之前；添加 before_attach
-------------------------------------------------------------------------------------

使用 after_attach 的事件处理程序现在可以假定给定实例与给定的 Session 关联：

    @event.listens_for(Session, "after_attach")
    def after_attach(session, instance):
        assert instance in session

某些用例需要它按此方式工作。但是，另一些用例要求该项尚未成为会话的一部分，例如，当查询旨在加载某些用于实例的必要状态时，它会首先发出 autoflush，并且将在未执行 :class:`.Session` 的任何关联事件之前更改结果。这些用例应使用新的“before_attach”事件：

    @event.listens_for(Session, "before_attach")
    def before_attach(session, instance):
        instance.some_necessary_attribute = (
            session.query(Widget).filter_by(instance.widget_name).first()
        )

:ticket:`2464`

查询现在会像 select() 一样自动关联
--------------------------------------

以前，必须调用 :meth:`_query.Query.correlate` 才能使列子查询或 WHERE 子查询与父级关联 :

    subq = (
        session.query(Entity.value)
        .filter(Entity.id == Parent.entity_id)
        .correlate(Parent)
        .as_scalar()
    )
    session.query(Parent).filter(subq == "some value")

这与普通的“select()”构造相反行为，后者默认情况下将自动关联。在 0.8 中，如果选择语句处于此上下文中，则选择语句将仅在实际使用该上下文时从 FROM 子句中省略“相关(即保留)”目标。此外，无法自己指定一个在一个封闭的选择语句中被放置作为 FROM 的选择语句来“关联”（即，省略）FROM 子句。

此更改仅使呈现 SQL 更佳，这样即使在选择语句用法正确的情况下也不可能呈现非法 SQL，其中没有足够的 FROM 对象相对于所选择的项目：

    from sqlalchemy.sql import table, column, select

    t1 = table("t1", column("x"))
    t2 = table("t2", column("y"))
    s = select([t1, t2]).correlate(t1)

    print(s)

在此更改之前，上面的代码将返回:

.. sourcecode:: sql

    SELECT t1.x, t2.y FROM t2

由于 t1 在任何 FROM 子句中都没有被引用，这是非法的 SQL。

现在，在没有封闭的 SELECT 中，它将返回：

.. sourcecode:: sql

    SELECT t1.x, t2.y FROM t1, t2

在 SELECT 中，关联行为与期望相同：

    s2 = select([t1, t2]).where(t1.c.x == t2.c.y).where(t1.c.x == s)
    print(s2)

.. sourcecode:: sql

    SELECT t1.x, t2.y FROM t1, t2
    WHERE t1.x = t2.y AND t1.x =
        (SELECT t1.x, t2.y FROM t2)

此更改不应影响任何现有应用程序，因为关联行为对于正确构建的表达式保持相同。只有测试场景中依赖于关联 SELECT 的非法字符串输出的应用程序会看到任何更改。

:ticket:`2668`


correlation_context_specific:

关联始终是上下文特定的
--------------------------

为了允许更广泛的关联方案，:meth:`_expression.Select.correlate` 和 :meth:`_query.Query.correlate` 的行为略有更改，以便仅在语句在该上下文中实际使用时，才从选择语句的 FROM 子句中省略“关联”目标。此外，无法自己指定一个在封闭的 SELECT 中作为一个 FROM 语句被放置的选择语句来“关联”（即，省略）FROM 子句。

此更改仅使呈现 SQL 更佳，在正确构建的表达式中，关联行为仍完全相同，包括“名字”和“键”方面的所有行为相同，包括呈现 SQL 仍然使用：<tablename>_<colname>该重点在于防止 :attr:`_schema.Column.key` 内容呈现为“SELECT”语句，以使 :attr:`_schema.Column.key` 中使用的特殊/非 ASCII 字符没有任何问题。

:ticket:`2668`


metadata_create_drop_tables:
.:meth:`_schema.MetaData.create_all` 和 :meth:`_schema.MetaData.drop_all` 现在将尊重空列表作为这样的
----------------------------------------------------------------

方法 :meth:`_schema.MetaData.create_all` 和 :meth:`_schema.MetaData.drop_all` 现在将接受一个空的 :class:`_schema.Table` 对象列表，并且不会发出任何 CREATE 或 DROP 语句。以前，空列表与传递“None”集合相同，并且 CREATE/DROP 将无条件地发出。

这是一个错误修复，但某些应用程序可能已经依赖于以前的行为。

:ticket:`2664`

:func:`_orm.Session.is_modified` 的行为已修复
----------------------------------------------

:meth:`.Session.is_modified` 方法接受一个参数“被动”，基本上不应该必要的情况下，所有情况下都应该是 “True” 的值 - 当其保留在其默认值“False”时，它会显示数据库，通常会触发 autoflush，它本身会更改结果。在 0.8 中，“被动”参数将无效，并且未加载的属性将永远不会检查其记录，因为按定义，未加载的属性上不能有待处理的状态更改。

.. 参见::

    :meth:`.Session.is_modified`

:ticket:`2320`

:attr:`_schema.Column.key` 在 :meth:`_expression.Select.apply_labels` 中使用 :class:`_expression.Select.c` 属性时得到了尊重
----------------------------------------------------------------------------------------------------------------

表达式系统的用户知道，:meth:`_expression.Select.apply_labels` 将表名添加到每个列名之前，影响可从 :attr:`_expression.Select.c` 中使用的名称：

::

    s = select([table1]).apply_labels()
    s.c.table1_col1
    s.c.table1_col2

在 0.8 之前，如果 :class:`_schema.Column` 具有不同的 :attr:`_schema.Column.key`，则此键将被忽略，相对于未使用 :meth:`_expression.Select.apply_labels` 时的不一致：

    # 在 0.8 之前
    table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
    s = select([table1])
    s.c.column_one  # 可以像这样访问
    s.c.col1  # 将引发 AttributeError

    s = select([table1]).apply_labels()
    s.c.table1_column_one  # 将引发 AttributeError
    s.c.table1_col1  # 可以像这样访问

在 0.8 中，:attr:`_schema.Column.key` 在两种情况下都会被尊重：

    # 与 0.8 一起使用
    table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
    s = select([table1])
    s.c.column_one  # 有效
    s.c.col1  # AttributeError

    s = select([table1]).apply_labels()
    s.c.table1_column_one  # 有效
    s.c.table1_col1  # AttributeError

所有其他的“name”和“key”行为都是相同的，包括呈现的 SQL 仍然使用形式“< tablename> _ <colname>”--重点是防止 :attr:`_schema.Column.key` 内容被呈现为“SELECT”语句，从而避免了使用 :attr:`_schema.Column.key` 中的特殊/非 ASCII 字符可能会出现的问题。

:ticket:`2397`

单个父级警告现在为错误
------------------------

由于某种原因，如果 :func:`_orm.relationship` 是一对多或多对多关系，并且指定了“cascade='all, delete-orphan'”，这是一种尴尬但仍然支持的用例（具有限制条件），如果关系没有指定 ``single_parent=True`` 选项，则现在将引发错误而不是警告。
以前只会发出警告，但是几乎立即会在属性系统中因为警告而失败。

:ticket:`2405`

为“列反映”添加“审核器”参数
---------------------------------------------

在 0.7 中添加了一个名为“column_reflect”的新事件，以便可以在反射每个列时添加增量。我们将此事件搞错了，因为事件没有提供任何方式来获取正在使用的当前“Inspector”和“Connection”，在需要数据库中的其他信息时。在这个新事件不被广泛使用的情况下，我们将直接将“审核器”参数添加到其中：

    @event.listens_for(Table, "column_reflect")
    def listen_for_col(inspector, table, column_info):
        ...

:ticket:`2418`

禁用 MySQL 的自动检测排序方式，大小写方式
-----------------------------------------------------

MySQL 方言为了连接所有可能的排序方式花费了两次调用，特别是消耗大量的时间，第一次在 ``Engine`` 连接的第一次，获取所有可能的排序方式，以及大小写方式的信息。这些集合不用于 SQLAlchemy 的任何功能，因此将更改这些调用不再自动发出。如果需要，可能会依赖于“engin.dialect” 中存在这些集合的应用程序需要直接调用“_detect_collations()”和“_detect_casing()”。

:ticket:`2404`

“未使用的列名称”警告现在变成异常
----------------------------------

在“insert()”或“update()”构造中引用不存在的列将引发错误，而不是警告：

::

    t1 = table("t1", column("x"))
    t1.insert().values(x=5, z=5)  # 引发 "Unconsumed column names: z"

:ticket:`2415`

Inspector.get_primary_keys() 被弃用，请使用 Inspector.get_pk_constraint
--------------------------------------------------------------------------

这两种方法在“Inspector”上是冗余的，其中“get_primary_keys()”将返回与“get_pk_constraint()”相同（不包括约束的名称）的信息：

::

    >>> insp.get_primary_keys()
    ["a", "b"]

    >>> insp.get_pk_constraint()
    {"name":"pk_constraint", "constrained_columns":["a", "b"]}

:ticket:`2422`

在大多数情况下，将不再启用大小写不敏感的结果行名称
------------------------------------------------------

对于大多数数据库/方言，添加一个新的配置 - :paramref:`_engine.create_engine.case_sensitive` 来控制大小写是否保留为结果行名称。该选项指示对结果行名称进行适当大小写，或仅对结果行名称进行小写或大写，具体取决于方言的处理方式。但是，几乎所有版本的 SQLite 都不能保留大小写，而要么只使用大写名称，要么只是小写名称。因此，对于 SQLite，case_sensitive=False 现在是默认设置，并且对于其他方言是 True。标准的 INI 文件系列目前未引用此设置，但是可以通过传递“sqlalchemy.create_engine（*args， ** kwargs）”参数来使用此选择。

An error has occured parsing your request. Please try again. If this issue persists, contact support.一个非常老的行为，即``RowProxy``中的列名始终是不区分大小写比较：

::

    >>> row = result.fetchone()
    >>> row["foo"] == row["FOO"] == row["Foo"]
    True

这是为了一些早期需要这种行为的方言，如Oracle和Firebird，但现代使用中，我们有更精确的方法来处理这两个平台的不区分大小写行为。

从现在开始，只有通过传递参数```case_sensitive=False```到```create_engine()```以可选方式使用此行为，否则从该行请求的列名必须匹配大小写。

``InstrumentationManager``和替代类中间件是一个扩展
----------------------------------------------------------------------------------

类``sqlalchemy.orm.interfaces.InstrumentationManager``
已经转移到了``sqlalchemy.ext.instrumentation.InstrumentationManager``。
为了让少数需要使用现有或非常规类中间件系统的安装受益，
已构建“替代中间件”系统，通常很少使用。这个系统的复杂性已经导出到一个``ext.``模块中。直到一旦导入，通常是当第三方库导入``InstrumentationManager``时，它才会被注入回到``sqlalchemy.orm``中，通过``ExtendedInstrumentationRegistry``用替代默认的``InstrumentationFactory``。

删除
=======

SQLSoup
-------

SQLSoup是SQLAlchemy ORM上的一个备选接口。SQLSoup现已移到它自己的项目中，并单独进行文档/发布；见https://bitbucket.org/zzzeek/sqlsoup。

SQLSoup是一个非常简单的工具，也可以受益于那些对它的使用风格感兴趣的贡献者。

:ticket:`2262`

MutableType
-----------

在SQLAlchemy ORM内部已删除旧的“可变”系统。这指的是应用于类型如``PickleType``和有条件的``TypeDecorator``的``MutableType``接口，并且自从SQLAlchemy很早的版本以来一直提供了一种检测所谓的“可变”数据结构（如JSON结构和拾取对象）变化的方法。然而，实现从未合理，强迫单元工作以非常低效的方式进行，导致在刷新期间进行了昂贵的对象扫描。在0.7中，引入了`sqlalchemy.ext.mutable <https://docs.sqlalchemy.org/en/latest/orm/extensions/mutable.html>`_扩展，以便用户定义的数据类型可以在更改发生时适当地向工作单元发送事件。

今天，预计使用``MutableType``的人很少，因为多年来已经警告过其低效性。

:ticket:`2442`

sqlalchemy.exceptions（数年来已是sqlalchemy.exc）
---------------------------------------------------------

我们留下了一个别名``sqlalchemy.exceptions``，以试图使一些尚未升级到使用``sqlalchemy.exc``的非常老的库稍微容易一些。然而，一些用户仍然被困惑了，因此在0.8中完全删除它以消除任何混淆。

:ticket:`2433`