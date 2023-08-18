.. highlight:: pycon+sql

.. |prev| replace::  :doc:`relationships` 
.. |next| replace::  :doc:`query` 

.. include:: queryguide_nav_include.rst


==================================
查询时使用的ORM API功能
==================================

ORM加载器选项
-------------------

加载器选项是当在  :meth:`_sql.Select.options`  方法中传递给 :class:` .Select`对象或类似的SQL构造时，影响列和关系导向属性的加载的对象。
大部分加载器选项从 :class:`_orm.Load` 继承层次结构中派生。
有关使用加载器选项的完整概述，请参见下面的链接部分。

.. seealso::

    *   :ref:`loading_columns`  - 关于如何加载列和SQL表达式映射属性的
      映射器和加载选项的详细信息。

    *   :ref:`loading_toplevel`  - 关于如何加载   :func:` _orm.relationship` 
      映射属性的关系和加载选项的详细信息。

.. _orm_queryguide_execution_options:

ORM执行选项
---------------------

ORM级别的执行选项是可以与语句执行关联的关键字选项，可以使用
  :meth:`_orm.Session.execute`   和  :meth:` _orm.Session.scalars`  ，也可以直接将其与要调用的语句相关联，使用  :meth:`_sql.Executable.execution_options`  方法，该方法接受它们作为任意关键字参数。

ORM级别的选项与在
  :meth:`_engine.Connection.execution_options`   中记录的核心级别执行选项不同。
重要的是要注意，以下ORM选项与核心级别方法
  :meth:`_engine.Connection.execution_options`   或  :meth:` _engine.Engine.execution_options`  **不** 兼容；
即使关联了使用的  :class:`_orm.Session`  的  :class:` .Engine`  或   :class:`.Connection` ，这些选项在该级别上也会被忽略。

在本节中，将使用  :meth:`_sql.Executable.execution_options`   方法样式来说明示例。

.. _orm_queryguide_populate_existing:

填充现有的实例
^^^^^^^^^^^^^^^^^^

``populate_existing`` 执行选项确保对于所有加载的行，  :class:`_orm.Session`  中对应的实例都将被完全刷新-擦除对象中任何现有数据（包括待定更改）并替换为从结果加载的数据。

示例用法如下::

    >>> stmt = select(User).execution_options(populate_existing=True)
    >>> result = session.execute(stmt)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    ...

通常，ORM对象只被加载一次，并且如果它们与后续结果行中的主键匹配，该行不会应用于对象。
这既是为了保留对象上待处理的未刷新更改，也是为了避免刷新已经存在的数据的开销和复杂性。
  :class:`_orm.Session`  假定高度隔离的事务的默认工作模型，在事务内部预期的数据随着本地更改之外的程度发生更改，应该使用诸如此方法的显式步骤来处理这些用例。

使用 ``populate_existing``，可以刷新匹配查询的任何对象集，并且它还允许控制关系加载器选项。
例如。在刷新实例的同时刷新相关对象集：

.. sourcecode:: python

    stmt = (
        select(User)
        .where(User.name.in_(names))
        .execution_options(populate_existing=True)
        .options(selectinload(User.addresses))
    )
    # 将刷新所有匹配的用户对象以及相关的Address对象
    users = session.execute(stmt).scalars().all()

``populate_existing`` 的另一个用例是支持各种特性属性加载，这些属性加载可以更改以每个查询为基础如何加载属性的方式。对此适用的选项包括：

*   :func:`_orm.with_expression`  选项。

*  :meth:`_orm.PropComparator.and_`  方法，可以修改加载器策略加载的内容

*   :func:`_orm.contains_eager`  选项

*   :func:`_orm.with_loader_criteria`  选项

``populate_existing`` 执行选项相当于 :term:`1.x style` ORM 查询中的  :meth:` _orm.Query.populate_existing`  方法。

.. seealso::

      :ref:`faq_session_identity`  - 在  :doc:` /faq/index`  中

      :ref:`session_expire`  - 在ORM   :class:` _orm.Session` 
    文档中

.. _orm_queryguide_autoflush:

自动刷新
^^^^^^^^^

当以 ``False`` 作为参数传递时，此选项将导致  :class:`_orm.Session`  不调用 "自动刷新" 步骤，
相当于使用  :attr:`_orm.Session.no_autoflush`  上下文管理器禁用自动刷新::

    >>> stmt = select(User).execution_options(autoflush=False)
    >>> session.execute(stmt)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    ...

此选项也适用于支持ORM的   :class:`_sql.Update`  和   :class:` _sql.Delete`  查询。

``autoflush`` 执行选项相当于 :term:`1.x style` ORM 查询中的  :meth:` _orm.Query.autoflush`  方法。

.. seealso::

      :ref:`session_flushing` 

.. _orm_queryguide_yield_per:

使用Yield Per获取大结果集``yield_per`` 执行选项是一个整数值，它将会在数据提供给客户端之前，仅将一定数量的行和/或 ORM 对象缓冲在内存中。

通常，ORM 会立即获取**所有**行，为每个行构造 ORM 对象并将这些对象组装到一个单一的缓冲区中，再将此缓冲区作为行的源传递到   :class:`_engine.Result`  对象供返回。这种行为的基本原理是，允许正确地使用特性，如连接贪婪加载，结果惟一化和依赖于标识映射在提取结果集中的每个对象维护一致状态的常规结果处理逻辑。

使用 ``yield_per`` 的目的是改变这种行为，以使 ORM 结果集针对遍历非常大的结果集（例如，> 10K 行）进行优化，其中用户已确定上述模式不适用。当使用 ``yield_per`` 时，ORM 将 ORM 结果批处理成子集，并在迭代   :class:`_engine.Result`  对象时逐个产生每个子集中的行，因此 Python 解释器无需声明非常大的内存区域，这既耗时又会导致过度的内存使用。该选项影响数据库游标的使用方式以及 ORM 构建行和对象以供传递到   :class:` _engine.Result`  的方式。

.. tip::

    由上可知，  :class:`_engine.Result`  必须以可迭代的方式被消耗，即使用迭代（例如，` `for row in result``）或使用部分行方法，例如  :meth:`_engine.Result.fetchmany`  或  :meth:` _engine.Result.partitions` 。调用  :meth:`_engine.Result.all`  将会破坏使用 ` `yield_per`` 的目的。

使用 ``yield_per`` 相当于同时使用  :paramref:`_engine.Connection.execution_options.stream_results`  执行选项（如果得到支持，选择让后端使用服务器端游标）和返回的   :class:` _engine.Result`  对象上的  :meth:`_engine.Result.yield_per`  方法，它建立了被批次获取行的固定大小和同时构造 ORM 对象的相应限制。

.. tip::

    ``yield_per`` 现在还可以作为 Core 执行选项使用，详细信息请参见   :ref:`engine_stream_results` 。本节详细介绍了在 ORM   :class:` _orm.Session`  上使用 ``yield_per`` 作为执行选项。该选项在两种情况下的行为尽可能相似。

当与 ORM 一起使用时，``yield_per`` 必须是通过给定语句上的  :meth:`.Executable.execution_options`  方法或通过将其传递给  :meth:` _orm.Session.execute`  或其他类似的   :class:`_orm.Session`  方法（例如  :meth:` _orm.Session.scalars` ）的  :paramref:`_orm.Session.execute.execution_options`  参数来建立的。获取 ORM 对象的典型用法如下所示：

    >>> stmt = select(User).execution_options(yield_per=10)
    >>> for user_obj in session.scalars(stmt):
    ...     print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] ()
    {stop}User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    ...
    >>> # ... rows continue ...

上面的代码等同于下面的示例，该示例使用  :paramref:`_engine.Connection.execution_options.stream_results`  和  :paramref:` _engine.Connection.execution_options.max_row_buffer`  Core 级执行选项，以及   :class:`_engine.Result`  的  :meth:` _engine.Result.yield_per`  方法：

    # equivalent code
    >>> stmt = select(User).execution_options(stream_results=True, max_row_buffer=10)
    >>> for user_obj in session.scalars(stmt).yield_per(10):
    ...     print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] ()
    {stop}User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    ...
    >>> # ... rows continue ...

``yield_per`` 还经常与  :meth:`_engine.Result.partitions`  方法一起使用，该方法将按分组分区迭代行。每个分区的大小默认为传递给 ` `yield_per`` 的整数值，如下例所示：

    >>> stmt = select(User).execution_options(yield_per=10)
    >>> for partition in session.scalars(stmt).partitions():
    ...     for user_obj in partition:
    ...         print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] ()
    {stop}User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    ...
    >>> # ... rows continue ...

``yield_per`` 执行选项与使用集合时的   :ref:`"子查询" eager loading <subquery_eager_loading>`  或   :ref:` "连接" lazy loading <joined_eager_loading>`  **不兼容**。如果数据库驱动程序支持多个独立的游标，则   :ref:`"select in" eager loading <selectin_eager_loading>`  可能兼容，并且每个游标能够处理特定数量的结果。

此外，``yield_per`` 执行选项与  :meth:`_engine.Result.unique`  方法不兼容；由于该方法依赖于存储所有行的完整标识符集，这将必然破坏使用 ` `yield_per`` 处理任意大量的结果集的目的。

.. versionchanged:: 1.4.6 从一个   :class:`_engine.Result`  对象中获取 ORM 行时，如果与同时使用 ` `yield_per`` 执行选项，则会触发异常  :meth:`_engine.Result.unique`  过滤器。

使用旧版   :class:`_orm.Query`  对象，配合  :term:` 1.x style`  的 ORM 使用时，  :meth:`_orm.Query.yield_per`   方法的结果与 ` `yield_per`` 执行选项相同。


.. seealso::

      :ref:`engine_stream_results` 

.. _queryguide_identity_token:

标识符令牌
^^^^^^^^^^^^^^

.. doctest-disable:

.. deepalchemy::   该选项是高级选项，大多数情况下用于与   :ref:`horizontal_sharding_toplevel`  扩展一同使用。
   对于从不同的“片”或分区中加载具有相同主键的对象的典型情况，请首先考虑分别使用每个   :class:`_orm.Session`  对象。


“标识符令牌”是一个任意值，它可以关联到新加载对象的  :term:`identity key`  中。 首先和主要是为了支持每行“分片”的扩展，其中对象可以从一个表的任意副本中加载，这些副本与重叠的主键值。 “标识符令牌”的主要用户是   :ref:` horizontal_sharding_toplevel`  扩展，它提供了一个通用框架，用于在特定数据库表的多个“片”之间持久化对象。

``identity_token`` 执行选项可以在每个查询的基础上使用以直接影响此令牌。 使用它可以直接在   :class:`_orm.Session`  中添加具有相同主键和来源表的多个对象，但具有不同的“标识符”。

一个例子是使用   :ref:`schema_translating`  功能从不同模式的同名表中填充   :class:` _orm.Session` 。 给定以下映射：

.. sourcecode:: python

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped, mapped_column


    class Base(DeclarativeBase):
        pass


    class MyTable(Base):
        __tablename__ = "my_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]

上面类的默认“schema”名称为``None``，表示 SQL 语句不会写入模式资格认证。 然而，如果使用  :paramref:`_engine.Connection.execution_options.schema_translate_map` ，将` `None``映射到替代模式，我们可以将 ``MyTable`` 的实例放入两个不同的模式中：

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

上述两个块每次都会创建一个链接到不同的模式翻译映射的   :class:`_orm.Session`  对象，并将 ` `MyTable`` 的实例持续保留在 ``test_schema.my_table`` 和 ``test_schema_2.my_table`` 中。

上面的   :class:`_orm.Session`  对象时相互独立的。 如果我们想在一个事务中同时保存这两个对象，则需要使用   :ref:` horizontal_sharding_toplevel`  扩展来完成此操作。

但是，我们可以通过以下方法说明在一个会话中查询这些对象：

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

两者的 ``obj1`` 和 ``obj2`` 是不同的。 然而，它们都是 ``MyTable`` 类的主键 id 1，但却是不同的。 这就是“identity_token”发挥作用的方式，我们可以在检查每个对象时查看  :attr:`_orm.InstanceState.key` ，以查看两个不同的标识符令牌：

    >>> from sqlalchemy import inspect
    >>> inspect(obj1).key
    (<class '__main__.MyTable'>, (1,), 'test_schema')
    >>> inspect(obj2).key
    (<class '__main__.MyTable'>, (1,), 'test_schema_2')


当使用   :ref:`horizontal_sharding_toplevel`  扩展时，上述逻辑会自动执行。.. versionadded::2.0.0rc1 - 添加` `identity_token`` ORM 级别执行选项。

.. seealso::

      :ref:`examples_sharding`  - 见于   :ref:` examples_toplevel`  部分。查看脚本``separate_schema_translates.py``了解如何使用完整的分片 API 的演示案例。

.. doctest-enable:

.. _queryguide_inspection:

检查从启用 ORM 的 SELECT 和 DML 语句中获取的实体和列
=======================================================

  :class:`_sql.select`  构造，以及   :func:` _sql.insert` ，  :func:`_sql.update` ，和   :func:` _sql.delete`  构件（对于后者的 DML 构件，从 SQLAlchemy 1.4.33 开始），都支持检查这些语句所涉及的实体以及组成结果集的列和数据类型。

对于   :class:`.Select`  对象，可以从  :attr:` .Select.column_descriptions`  属性中获取这些信息。这个属性的操作方式与传统的  :attr:`.Query.column_descriptions`  属性相同。返回的格式为字典列表::

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


当使用  :attr:`.Select.column_descriptions`  与非 ORM 对象配合使用，例如普通的   :class:` .Table`  或   :class:`.Column`  对象时，所有情况下返回的条目将包含关于各个列的基本信息::

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

.. versionchanged:: 1.4.33 当应用于无 ORM 的   :class:`.Select`  时，  :attr:` .Select.column_descriptions`   属性现在会返回一个值。之前，这将会引发 ``NotImplementedError`` 异常。

对于   :func:`_sql.insert` ，  :func:` .update`  和   :func:`.delete`  构件，存在两个单独的属性。 其中一个是  :attr:` .UpdateBase.entity_description` ，返回的是与 DML 构件相关的主 ORM 实体和数据库表的信息::

    >>> from sqlalchemy import update
    >>> stmt = update(User).values(name="somename").returning(User.id)
    >>> pprint(stmt.entity_description)
    {'entity': <class 'User'>,
     'expr': <class 'User'>,
     'name': 'User',
     'table': Table('user_account', ...),
     'type': <class 'User'>}

.. tip::   :attr:`.UpdateBase.entity_description`  包括一个项 ` `"table"``，实际上是语句将要插入、更新或删除的表格，而这通常并不与映射到该类的 SQL "selectable" 相同。例如，在继承加入表行中，``"table"`` 将引用给定实体的本地表格。

另一个是  :attr:`.UpdateBase.returning_column_descriptions` ，以与  :attr:` .Select.column_descriptions`  类似的方式提供关于返回集合中存在的列的信息::

    >>> pprint(stmt.returning_column_descriptions)
    [{'aliased': False,
      'entity': <class 'User'>,
      'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute ...>,
      'name': 'id',
      'type': Integer()}]

.. versionadded:: 1.4.33 添加了  :attr:`.UpdateBase.entity_description`  和  :attr:` .UpdateBase.returning_column_descriptions`  属性。

.. _queryguide_additional:

其他 ORM API 构件
=================

.. autofunction:: sqlalchemy.orm.aliased

.. autoclass:: sqlalchemy.orm.util.AliasedClass

.. autoclass:: sqlalchemy.orm.util.AliasedInsp

.. autoclass:: sqlalchemy.orm.Bundle
    :members:

.. autofunction:: sqlalchemy.orm.with_loader_criteria

.. autofunction:: sqlalchemy.orm.join

.. autofunction:: sqlalchemy.orm.outerjoin

.. autofunction:: sqlalchemy.orm.with_parent..  不要显示设置代码

    >>> session.close()
    >>> conn.close()
    ROLLBACK