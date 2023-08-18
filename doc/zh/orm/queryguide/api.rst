.. highlight:: pycon+sql

.. |prev| replace:: :doc:`relationships`
.. |next| replace:: :doc:`query`

.. include:: queryguide_nav_include.rst


=================================
用于查询的ORM API特性
=================================

ORM Loader选项
-------------------

当传递loader选项对象给类:func:`_sql.Select`或类似的SQL构造方法中的方法:meth:`_sql.Select.options`时，它会影响属性（包括列和关系）的加载。大多数loader选项从:class:`_orm.Load`层次结构中降下，关于使用loader选项的完整概述，请参阅下面链接的相关节。

.. seealso::

    * :ref:`loading_columns` - 明确列和SQL表达式映射属性加载的细节和mapper选项

    * :ref:`loading_toplevel` - 明确关系和映射属性（:func:`_orm.relationship`）的加载的细节和loader 选项

.. _orm_queryguide_execution_options:

ORM执行选项
---------------------

ORM级别的执行选项是关键字选项，可以使用 :paramref:`_orm.Session.execute.execution_options` 参数与 :class:`_orm.Session` 方法（如:meth:`_orm.Session.execute` 以及:meth:`_orm.Session.scalars`）进行关联，或直接将这些选项与待调用的语句相关联。

ORM级别的选项不同于文档中记录的核心级别执行选项方法:meth:`_engine.Connection.execution_options`。应注意，在这个级别，ORM选项被忽略，即使:class:`.Engine`或:class:`.Connection`与正在使用的:class:`_orm.Session`相关联，这些选项也会被忽略。

在这个部分中，:meth:`_sql.Executable.execution_options` 方法的样式将用于示例。

.. _orm_queryguide_populate_existing:

现有数据充填
^^^^^^^^^^^^^^^^^^

``populate_existing``执行选项将确保，对于加载的所有行，:class:`_orm.Session` 中对应的实例将完全刷新——消除任何对象中的现有数据（包括挂起的更改），并使用从结果加载的数据进行替换。

示例使用方法如下::

    >>> stmt = select(User).execution_options(populate_existing=True)
    >>> result = session.execute(stmt)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    ...

通常，ORM对象只会加载一次。如果它们在随后的结果行中与主键匹配，则该行不会被应用于对象。这既是为了保留对象上挂起的未刷新更改，又避免刷新已经存在的数据。

使用 ``populate_existing``，可以刷新与查询匹配的任何对象集，并且还可以控制关系加载器选项。例如，可以刷新一个实例，同时刷新一组相关对象:

.. sourcecode:: python

    stmt = (
        select(User)
        .where(User.name.in_(names))
        .execution_options(populate_existing=True)
        .options(selectinload(User.addresses))
    )
    # 将刷新所有匹配的User对象以及关联的Address对象
    users = session.execute(stmt).scalars().all()

另一个使用 ``populate_existing`` 的用例与各种属性加载特性的支持有关，可以更改属性在每个查询中加载的方式。适用于此类选项的选项包括:

* :func:`_orm.with_expression` Option

* :meth:`_orm.PropComparator.and_`方法可修改加载器策略所加载的内容

* :func:`_orm.contains_eager`选项

* :func:`_orm.with_loader_criteria`选项

``populate_existing``执行选项等效于:meth:`_orm.Query.populate_existing`,在:term:`1.x style` ORM查询中。

.. seealso::

    :ref:`faq_session_identity` - 在 :doc:`/faq/index`

    :ref:`session_expire` - 在 ORM :class:`_orm.Session` 文档中

.. _orm_queryguide_autoflush:

自动刷新
^^^^^^^^^^

当传递 ``False`` 作为 ``autoflush`` 执行选项时，:class:`_orm.Session` 将不会触发 "autoflush" 步骤。 这相当于使用 :attr:`_orm.Session.no_autoflush` 上下文管理器来禁用 autoflush::

    >>> stmt = select(User).execution_options(autoflush=False)
    >>> session.execute(stmt)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    ...

这个选项也适用于启用ORM 的:class:`_sql.Update` 和
:class:`_sql.Delete`。

``autoflush``选项等价于在 :term:`1.x style` ORM查询中使用 :meth:`_orm.Query.autoflush` 方法。

.. seealso::

    :ref:`session_flushing`

.. _orm_queryguide_yield_per:

使用Yield Per抓取大量结果集
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``yield_per``执行选项是一个整数值，它只会在缓冲器中包含一个有限的行和/或ORM对象，并在为客户端提供数据之前，逐一从每个子集中产生行的:class:`_engine.Result`。

通常，ORM会**立即**获取所有行，为每个行构建ORM对象，并将这些对
象组装到单个缓冲区中，然后将此缓冲区作为源返回给:class:`_engine.Result`，以便返回行。此行为的基本原理是允许所有对象在提取时保持一致。

``yield_per`` 选项的目的是为了更改该行为，以便ORM结果集是针对迭代非常大的结果集（例如> 10K行），其中用户确定以上模式不适用的情况进行优化。当使用 ``yield_per`` 时，ORM将ORM结果批处理为子集，并在:class:`_engine.Result`对象迭代时，逐一排放每个子集中的行，以便Python解释器不需要声明超大的内存区域，这既费时又会导致过度使用内存。该选项同时影响数据库游标的使用以及ORM结构行和对象的构造方式，以供:class:`_engine.Result`。

.. Tip::
    如上述内容所述，由于结果需要保持游标等仅在列表风格迭代器中传递且无法追溯或索引，而不是随机访问的方式，因此:class:`_engine.Result` 必须按可迭代的方式使用，如 `for row in result` 使用迭代或使用 :meth:`_engine.Result.fetchmany` 或 :meth:`_engine.Result.partitions` 使用部分行的方法。调用:meth:`_engine.Result.all` 将破坏使用 `yield_per` 的意图。

使用 ``yield_per`` 等价于使用 :paramref:`_engine.Connection.execution_options.stream_results` 执行选项，如果支持，则选择使用后端的服务器端游标，以及返回的 :class:`_engine.Result` 对象上的 :meth:`_engine.Result.yield_per` 方法，该方法建立要获取的行的固定大小和同时构建的 ORM对象数量的相应限制。

.. Tip::
    `yield_per` 在 Core 执行选项上也可用，详细信息请参见:ref:`engine_stream_results`。本节详细说明 `yield_per` 作为 ORM: class:`_orm.Session`执行选项的使用。 该选项的行为尽可能在两种情况下相似。

当与ORM配合使用时，可以专门在给定语句上使用 ``yield_per``
通过:meth:`.Executable.execution_options`方法，或将其传递到:meth:`_orm.Session.execute` 或其他类似:meth:`_orm.Session` 随后使用的方法（例如 :meth:`_orm.Session.scalars`）的 :paramref:`_orm.Session.execute.execution_options`参数。 示例：:    

    >>> stmt = select(User).execution_options(yield_per=10)
    >>> for user_obj in session.scalars(stmt):
    ... print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...]()
    {stop}User(id=1, name =' spongebob'，fullname ='海绵宝宝')
    User(id=2, name ='sandy'，fullname ='桑迪的脚')
    ...
    >>> # ... rows continue ...

以上代码等同于下面的示例，后者将使用 _engine.Result.yield_per和 _engine.Connection.execution_options.stream_results(Core级别 )方法结合使用：

    # 等效代码
    >>> stmt = select(User).execution_options(stream_results=True, max_row_buffer=10)
    >>> for user_obj in session.scalars(stmt).yield_per(10):
    ... print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...]()
    {stop}User(id=1, name ='spongebob', fullname ='Spongebob Squarepants')
    User(id=2, name ='sandy', fullname ='Sandy Cheeks')
    ...
    >>> # ... rows continue ...

 ``yield_per``通常与:meth:`_engine.Result.partitions`方法结合使用，该方法将以分组分区的方式迭代行。默认情况下，每个分区的大小与传递给 ``yield_per`` 的整数值相同，如下面的示例::

    >>> stmt = select(User).execution_options(yield_per=10)
    >>> for partition in session.scalars(stmt).partitions():
    ... for user_obj in partition:
    ... print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...]()
    {stop}User(id=1, name ='spongebob', fullname ='Spongebob Squarepants')
    User(id=2, name ='sandy', fullname ='Sandy Cheeks')
    ...
    >>> # ... rows continue ...

 ``yield_per``执行选项与使用集合时的:ref:`"subquery" eager loading <subquery_eager_loading>`加载或:ref:`"joined" eager loading <joined_eager_loading>`加载**不兼容**。如果数据库驱动程序支持多个独立光标，则它可能与 :ref:`"select in" eager loading <selectin_eager_loading>` 是兼容的。
此外，``yield_per`` 执行选项不兼容:meth:`_engine.Result.unique` 方法；因为此方法依赖于存储所有行的完整标识集，所以它一定会打败使用 ``yield_per`` 的目的，这是处理任意数量行的理想方式。

.. versionchanged:: 1.4.6 引发异常当用 ``yield_per``执行选项从使用 :class:`_engine.Result.unique` 过滤器的 :class:`_engine.Result` 对象中获取ORM行时，同时使用 ``yield_per`` 执行选项时。

当使用 :term:`1.x style` ORM使用旧的:class:`_orm.Query`对象时，:meth:`_orm.Query.yield_per`方法的结果与 ``yield_per``执行选项相同。


.. seealso::

    :ref:`engine_stream_results`

.. _queryguide_identity_token:

Identity Token
^^^^^^^^^^^^^^

.. doctest-disable:

.. deepalchemy:: 此选项是主要用于:ref:`horizontal_sharding_toplevel`扩展的高级用法，大多数情况下，用于从不同的"碎片"或分区加载具有相同主键的对象，首先考虑为每个分区使用单个:class:`_orm.Session`对象。

"标识令牌"是可以关联到新加载对象的:term:`identity key` 的任意值。首先提供这个元素以支持对每行进行"分片"的扩展，其中可能会从特定数据库表的任意副本加载对象，这些副本具有重叠的主键值。最主要的消费者是:ref:`horizontal_sharding_toplevel`扩展，它提供了全面框架，以在特定数据库表的多个"分片"之间持久化对象。

可以在每个查询上使用``identity_token``执行选项直接影响此令牌。直接使用它，可以在:class:`_orm.Session`中填充多个具有相同主键和来源表，但具有不同标识的对象。

这种情况之一是，使用可以影响查询范围内的schema选择的:ref:`schema_translating` 功能，将一个:class:`.Table`类下来到不同的schema中，即``None``极其所需的映射。。

.. sourcecode:: python

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class MyTable(Base):
        __tablename__ = "my_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]

上述类的默认"schema"名称为 ``None``，这意味着，不会将模式限定名称写入SQL语句。但是，如果使用 :paramref:`_engine.Connection.execution_options.schema_translate_map`，将None映射到替代模式，则可以在不同的模式中放置``MyTable``的实例：

.. sourcecode:: python

    engine = create_engine(
        "postgresql+psycopg://scott:tiger@localhost/test",
    )

    with Session(
        engine.execution_options(schema_translate_map={None: "test_schema"})
    ) as sess:
        sess.add(MyTable(name="this is schema one"))
        sess.commit()

    with Session(
        engine.execution_options(schema_translate_map={None: "test_schema_2"})
    ) as sess:
        sess.add(MyTable(name="this is schema two"))
        sess.commit()

上述两个块创建了连接到不同模式的 :class:`_orm.Session`对象。现在可以将 SELECT 查询查询为 :

.. sourcecode:: python

    with Session(engine) as sess:
        obj1 = sess.scalar(
            select(MyTable)
            .where(MyTable.id == 1)
            .execution_options(
                schema_translate_map={None: "test_schema"},
                identity_token="test_schema",
            )
        )
        obj2 = sess.scalar(
            select(MyTable)
            .where(MyTable.id == 1)
            .execution_options(
                schema_translate_map={None: "test_schema_2"},
                identity_token="test_schema_2",
            )
        )

``obj1``和``obj2``都是互不相关的。然而，它们都引用``MyTable``类的主键id 1，在数据库中具有相同名称的表，但具有不同的“标识”。这就是``identity_token``如何发挥作用的，可以在检查每个对象时看到这一点，查看 :attr:`_orm.InstanceState.key`属性，以查看两个不同的标识符令牌::

    >>> from sqlalchemy import inspect
    >>> inspect(obj1).key
    (<class '__main__.MyTable'>, (1,), 'test_schema')
    >>> inspect(obj2).key
    (<class '__main__.MyTable'>, (1,), 'test_schema_2')


上述逻辑会在使用:ref:`horizontal_sharding_toplevel`扩展时自动发生。

.. versionadded:: 2.0.0rc1 - 增加了 ``identity_token`` ORM级别执行选项。

.. seealso::

    :ref:`examples_sharding` - 在 :ref:`examples_toplevel`部分中。请参见脚本``separate_schema_translates.py``，以演示使用全面分片API的上述用例。


.. doctest-enable:

.. _queryguide_inspection:

检查来自启用ORM的SELECT和DML语句的实体和列
==========================================================================

方法:func:`_sql.select` 构造方法以及 :func:`_sql.insert`、:func:`_sql.update`、:func:`_sql.delete` 构造方法（这些后者的DML构建方式已从 SQLAlchemy 1.4.33开始支持）都支持检查这些语句将针对的实体以及返回的列和数据类型。

对于:class:`.Select`对象，它可以从 :attr:`.Select.column_descriptions` 属性获得此信息。 此属性与遗留的 :attr:`.Query.column_descriptions` 属性以相同的方式运作。返回的格式是字典列表：

    >>> from pprint import pprint
    >>> user_alias = aliased(User, name="user2")
    >>> stmt = select(User, User.id, user_alias)
    >>> pprint(stmt.column_descriptions)
    [{'aliased': False,
      'entity': <class 'User'>,
      'expr': <class 'User'>,
      'name': 'User',
      'type': <class 'User'>},
     {'aliased': False,
      'entity': <class 'User'>,
      'expr': <....InstrumentedAttribute object at ...>,
      'name': 'id',
      'type': Integer()},
     {'aliased': True,
      'entity': <AliasedClass ...; User>,
      'expr': <AliasedClass ...; User>,
      'name': 'user2',
      'type': <class 'User'>}]


当使用 :attr:`.Select.column_descriptions` 在此类:meth:`.Table` 或 :class:`.Column`对象之外的非ORM对象中使用时，条目将包含有关在所有情况下返回的单个列的基本信息：

    >>> stmt = select(user_table, address_table.c.id)
    >>> pprint(stmt.column_descriptions)
    [{'expr': Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
      'name': 'id',
      'type': Integer()},
     {'expr': Column('name', String(), table=<user_account>, nullable=False),
      'name': 'name',
      'type': String()},
     {'expr': Column('fullname', String(), table=<user_account>),
      'name': 'fullname',
      'type': String()},
     {'expr': Column('id', Integer(), table=<address>, primary_key=True, nullable=False),
      'name': 'id_1',
      'type': Integer()}]

.. versionchanged:: 1.4.33 当对 :class:`.Select` 使用前面的方法时，即使没有启用ORM， :attr:`.Select.column_descriptions`属性现在也会返回值，而不是引发 ``NotImplementedError``。

对于 :func:`_sql.insert`、:func:`.update`和 :func:`.delete` 构造方法，有两个不同的属性。一个是 :attr:`.UpdateBase.entity_description` 属性，它返回有关编写未启用ORM的数据库表和主要ORM实体的信息：

    >>> from sqlalchemy import update
    >>> stmt = update(User).values(name="somename").returning(User.id)
    >>> pprint(stmt.entity_description)
    {'entity': <class 'User'>,
     'expr': <class 'User'>,
     'name': 'User',
     'table': Table('user_account', ...),
     'type': <class 'User'>}

.. tip::  :attr:`.UpdateBase.entity_description `包含一个条目``"table"``，它实际上是该语句将要插入，更新或删除的**表**，这并不总是相同的SQL"selectable"，该类可能被映射到。例如，在连接的表继承场景中，``"table"``将引用给定实体的本地表。

另一个属性是 :attr:`.UpdateBase.returning_column_descriptions` ，它以与 :attr:`.Select.column_descriptions`大致相同的方式提供有关 :func:`_sql.select` 的返回集合中存在的列的信息：

    >>> pprint(stmt.returning_column_descriptions)
    [{'aliased': False,
      'entity': <class 'User'>,
      'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute ...>,
      'name': 'id',
      'type': Integer()}]

.. versionadded:: 1.4.33 添加了 :attr:`.UpdateBase.entity_description `和 :attr:`.UpdateBase.returning_column_descriptions` 属性。


.. _queryguide_additional:

其他ORM API结构
=============================


.. autofunction:: sqlalchemy.orm.aliased

.. autoclass:: sqlalchemy.orm.util.AliasedClass

.. autoclass:: sqlalchemy.orm.util.AliasedInsp

.. autoclass:: sqlalchemy.orm.Bundle
    :members:

.. autofunction:: sqlalchemy.orm.with_loader_criteria

.. autofunction:: sqlalchemy.orm.join

.. autofunction:: sqlalchemy.orm.outerjoin

.. autofunction:: sqlalchemy.orm.with_parent


.. Setup code, not for display。

    >>> session.close()
    >>> conn.close()
    ROLLBACK