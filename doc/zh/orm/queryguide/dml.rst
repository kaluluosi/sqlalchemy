.. highlight:: pycon+sql
.. |prev| replace::  :doc:`继承` 
.. |next| replace::  :doc:`列` 

.. include:: queryguide_nav_include.rst

.. doctest-include _dml_setup.rst

.. _orm_expression_update_delete:

ORM-启用的INSERT、UPDATE和DELETE语句
=================================================

.. admonition:: 关于本文档

    本节使用ORM映射，这些映射首先在   :ref:`统一教程`  中进行了说明，
    在   :ref:`教程_声明映射类`  中展示，以及顶级继承中展示的继承映射。

     :doc:`查看此页的ORM设置<_dml_setup>` 。

  :meth:`_orm.Session.execute`  方法不仅处理ORM启用的 :class:` _sql.Select`对象，
还可以支持ORM启用的   :class:`_sql.Insert` ,   :class:` _sql.Update`  和   :class:`_sql.Delete`  对象，
以多种不同的方式处理，用于一次性插入、更新或删除多行数据库。
还有特定于方言的支持ORM启用的"upsert"，它是INSERT语句，可以自动使用UPDATE来处理已经存在的行。

以下表格总结了本文档中讨论的调用形式：

=====================================================   ==========================================   ========================================================================     ========================================================= ============================================================================
ORM用例                                                  DML构造使用                             数据使用...                                                       支持返回吗？                                                是否支持多表映射？
=====================================================   ==========================================   ========================================================================     ========================================================= ============================================================================
  :ref:`orm_queryguide_bulk_insert`                             :func:` _dml.insert`                    列表字典传递给paramref:`_orm.Session.execute.params`的   :ref:`是<orm_queryguide_bulk_insert_returning>`  ：ref:` 是<orm_queryguide_insert_joined_table_inheritance>`
  :ref:`orm_queryguide_bulk_insert_w_sql`                       :func:` _dml.insert`                     :paramref:`_orm.Session.execute.params`  与  :meth:` _dml.Insert.values`      :ref:`yes <orm_queryguide_bulk_insert_w_sql>`                :ref:` yes <orm_queryguide_insert_joined_table_inheritance>` 
  :ref:`orm_queryguide_insert_values`                           :func:` _dml.insert`                    列表字典传递给  :meth:`_dml.Insert.values <_sql.Insert.values>`                  :ref:` yes <orm_queryguide_insert_values>`                  no
  :ref:`orm_queryguide_upsert`                                  :func:` _dml.insert`                    列表字典传递给  :meth:`_dml.Insert.values <_sql.Insert.values>`                  :ref:` yes <orm_queryguide_upsert_returning>`               no
  :ref:`orm_queryguide_bulk_update`                             :func:` _dml.update`                    列表字典传递给 ：paramref:`_orm.Session.execute.params`                no                                                          :ref:`是 <orm_queryguide_bulk_update_joined_inh>` 
  :ref:`orm_queryguide_update_delete_where`                     :func:` _dml.update` ,   :func:`_dml.delete`      对  :meth:` _dml.Update.values`  的关键字       :ref:`yes <orm_queryguide_update_delete_where_returning>`     :ref:` partial, with manual steps <orm_queryguide_update_delete_joined_inh>` 
=====================================================   ==========================================   ========================================================================     ========================================================= ============================================================================



.. _orm_queryguide_bulk_insert:

ORM大容量INSERT语句
--------------------------

一个    :func:`_dml.insert`  构造可以采用ORM类进行构造，并传递给   :meth:` _orm.Session.execute`  方法。
发送到   :paramref:`_orm.Session.execute.params`  参数的参数字典列表，与   :class:` _dml.Insert`  对象本身分开，
将调用 **大容量INSERT模式**，这实际上意味着该操作将尽可能地针对多行进行优化::

    >>> from sqlalchemy import insert
    >>> session.execute(
    ...     insert(User),
    ...     [
    ...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...         {"name": "patrick", "fullname": "Patrick Star"},
    ...         {"name": "squidward", "fullname": "Squidward Tentacles"},
    ...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ...     ],
    ... )
    {execsql}INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] [('spongebob', 'Spongebob Squarepants'), ('sandy', 'Sandy Cheeks'), ('patrick', 'Patrick Star'),
    ('squidward', 'Squidward Tentacles'), ('ehkrabs', 'Eugene H. Krabs')]
    {stop}<...>

参数字典包含键值对，这些键值对可能对应于映射到   :class:`._schema.Column`  或   :func:` _orm.mapped_column`  声明的映射ORM属性，
以及   :ref:`复合 <mapper_composite>`  声明的映射属性。如果这两个名称恰好不同，键应与**ORM映射属性名称**匹配，而不是实际的数据库列名称。

.. versionchanged:: 2.0  将   :class:`_dml.Insert`  构造传递到  :meth:` _orm.Session.execute`  方法中现在会调用“bulk insert”，
   使用与传统的  :meth:`_orm.Session.bulk_insert_mappings`  方法相同的功能。这是与 1.x系列相比的行为更改，其中基于列名的Core-centric方式对   :class:` _dml.Insert`  进行解释；现在接受ORM属性键。Core样式功能可通过将执行选项 ``{"dml_strategy": "raw"}`` 传递给  :meth:`_orm.Session.execute`  的  :paramref:` _orm.Session.execution_options`  参数来使用。使用 RETURNING 获取新对象
~~~~~~~~~~~~~~~~~~~~~~~~

..  设置代码，不用显示

  >>> session.rollback()
  ROLLBACK...
  >>> session.connection()
  BEGIN (implicit)...

批量 ORM 插入功能支持选定后端的 INSERT..RETURNING，并且能够返回   :class:`.Result`  对象，可以返回独立的列以及与新生成的记录相对应的完整的 ORM 对象。 INSERT..RETURNING 需要使用支持 SQL RETURNING 语法以及支持 RETURNING 和  :term:` executemany`  的后端。此功能在所有   :ref:`SQLAlchemy-included <included_dialects>`  后端中都可用，不包括 MySQL（MariaDB 包括在内）。

例如，我们可以运行与以前相同的语句，添加使用  :meth:`.UpdateBase.returning`  方法，传递完整的“User”实体作为我们想要返回的内容。  :meth:` _orm.Session.scalars`  用于允许迭代“User”对象

::

    >>> users = session.scalars(
    ...     insert(User).returning(User),
    ...     [
    ...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...         {"name": "patrick", "fullname": "Patrick Star"},
    ...         {"name": "squidward", "fullname": "Squidward Tentacles"},
    ...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ...     ],
    ... )
    {execsql}INSERT INTO user_account (name, fullname)
    VALUES (?, ?), (?, ?), (?, ?), (?, ?), (?, ?)
    RETURNING id, name, fullname, species
    [...] ('spongebob', 'Spongebob Squarepants', 'sandy', 'Sandy Cheeks',
    'patrick', 'Patrick Star', 'squidward', 'Squidward Tentacles',
    'ehkrabs', 'Eugene H. Krabs')
    {stop}>>> print(users.all())
    [User(name='spongebob', fullname='Spongebob Squarepants'),
     User(name='sandy', fullname='Sandy Cheeks'),
     User(name='patrick', fullname='Patrick Star'),
     User(name='squidward', fullname='Squidward Tentacles'),
     User(name='ehkrabs', fullname='Eugene H. Krabs')]

在上面的例子中，渲染的 SQL 采用了 SQLite 后端要求的形式，即将单个参数字典内联到单个 INSERT 语句中，以便可以使用 RETURNING。

.. versionchanged:: 2.0  ORM   :class:`.Session`  现在在 ORM 上下文中解释   :class:` _dml.Insert` 、  :class:`_dml.Update`  甚至包括   :class:` _dml.Delete`  结构中的 RETURNING 子句，这意味着可以将列表达式和 ORM 映射实体混合传递给  :meth:`_dml.Insert.returning`  方法，然后可以按照类似于   :class:` _sql.Select`  的方式传递结果，包括映射实体作为 ORM 映射对象的传递方式。 还有 ORM 加载器选项的有限支持，例如   :func:`_orm.load_only`  和   :func:` _orm.selectinload` 。

.. _orm_queryguide_bulk_insert_returning_ordered:

将 RETURNING 记录与输入数据顺序相关联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用带有 RETURNING 的批量 INSERT 时，重要的是要注意，大多数数据库后端不提供保证 RETURNING 返回记录的顺序，包括它们的顺序将与输入记录的顺序对应的保证。 对于需要确保 RETURNING 记录与输入数据相关联的应用程序，可以指定附加参数  :paramref:`_dml.Insert.returning.sort_by_parameter_order` ，该参数可能使用特殊的 INSERT 表单，以保持有序并且可正确排序返回的行中使用的令牌，或者在某些情况下，例如在下面的示例中使用 SQLite 后端时，该操作将一次插入一行：

::

    >>> data = [
    ...     {"name": "pearl", "fullname": "Pearl Krabs"},
    ...     {"name": "plankton", "fullname": "Plankton"},
    ...     {"name": "gary", "fullname": "Gary"},
    ... ]
    >>> user_ids = session.scalars(
    ...     insert(User).returning(User.id, sort_by_parameter_order=True), data
    ... )
    {execsql}INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [... (insertmanyvalues) 1/3 (ordered; batch not supported)] ('pearl', 'Pearl Krabs')
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [insertmanyvalues 2/3 (ordered; batch not supported)] ('plankton', 'Plankton')
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [insertmanyvalues 3/3 (ordered; batch not supported)] ('gary', 'Gary')    {stop}>>> for user_id, input_record in zip(user_ids, data):
    ...     input_record["id"] = user_id
    >>> print(data)
    [{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
    {'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
    {'name': 'gary', 'fullname': 'Gary', 'id': 8}]

.. versionadded:: 2.0.10 添加了  :paramref:`_dml.Insert.returning.sort_by_parameter_order`  功能，
   该功能已实现在  :term:`insertmanyvalues`  架构中。

.. seealso::

      :ref:`engine_insertmanyvalues_returning_order`  - 确保输入数据和结果行一一对应
    的背景及取得良好性能的方法。

.. _orm_queryguide_insert_heterogeneous_params:

使用不同类型的参数字典
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  运行代码，不要显示

  >>> session.rollback()
  ROLLBACK...
  >>> session.connection()
  BEGIN (implicit)...

ORM批量插入功能支持"heterogeneous" 的参数字典，基本上意思是“不同的
字典可以有不同的键”。当检测到此条件时，ORM将把参数字典分成与每个键对应的组，并
相应地批量处理成单独的INSERT语句::

    >>> users = session.scalars(
    ...     insert(User).returning(User),
    ...     [
    ...         {
    ...             "name": "spongebob",
    ...             "fullname": "Spongebob Squarepants",
    ...             "species": "Sea Sponge",
    ...         },
    ...         {"name": "sandy", "fullname": "Sandy Cheeks", "species": "Squirrel"},
    ...         {"name": "patrick", "species": "Starfish"},
    ...         {
    ...             "name": "squidward",
    ...             "fullname": "Squidward Tentacles",
    ...             "species": "Squid",
    ...         },
    ...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs", "species": "Crab"},
    ...     ],
    ... )
    {execsql}INSERT INTO user_account (name, fullname, species)
    VALUES (?, ?, ?), (?, ?, ?) RETURNING id, name, fullname, species
    [... (insertmanyvalues) 1/1 (unordered)] ('spongebob', 'Spongebob Squarepants', 'Sea Sponge',
    'sandy', 'Sandy Cheeks', 'Squirrel')
    INSERT INTO user_account (name, species)
    VALUES (?, ?) RETURNING id, name, fullname, species
    [...] ('patrick', 'Starfish')
    INSERT INTO user_account (name, fullname, species)
    VALUES (?, ?, ?), (?, ?, ?) RETURNING id, name, fullname, species
    [... (insertmanyvalues) 1/1 (unordered)] ('squidward', 'Squidward Tentacles',
    'Squid', 'ehkrabs', 'Eugene H. Krabs', 'Crab')



在上面的示例中，传递的五个参数字典转换成了三个INSERT语句，按每个字典中
特定的键分组，同时仍然保持行顺序，即 "name", "fullname", "species",
"name", "species" 和 "name","fullname", "species"。

.. _orm_queryguide_insert_joined_table_inheritance:

使用连接表继承的批量INSERT
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  运行代码，不要显示

    >>> session.rollback()
    ROLLBACK
    >>> session.connection()
    BEGIN...

ORM批量插入建立在通常用于发送INSERT语句的传统单元工作体系的基础上。这意味着，
对于一个映射到多个表的ORM实体，通常使用   :ref:`joined table inheritance <joined_inheritance>` 
进行映射，批量插入操作将为映射表示的每个表发出一个INSERT语句，
将服务器生成的主键值正确传递到依赖于它们的表行。
同时也支持RETURNING功能，ORM将收到每个执行的INSERT语句的   :class:`.Result`  对象，
然后将它们 "横向拼接"在一起，使返回的行包括插入的所有列的值::

    >>> managers = session.scalars(
    ...     insert(Manager).returning(Manager),
    ...     [
    ...         {"name": "sandy", "manager_name": "Sandy Cheeks"},
    ...         {"name": "ehkrabs", "manager_name": "Eugene H. Krabs"},
    ...     ],
    ... )
    {execsql}INSERT INTO employee (name, type) VALUES (?, ?) RETURNING id, name, type
    [... (insertmanyvalues) 1/2 (ordered; batch not supported)] ('sandy', 'manager')
    INSERT INTO employee (name, type) VALUES (?, ?) RETURNING id, name, type
    [insertmanyvalues 2/2 (ordered; batch not supported)] ('ehkrabs', 'manager')    INSERT INTO manager (id, manager_name) VALUES (?, ?), (?, ?) RETURNING id, manager_name, id AS id__1
    [... (insertmanyvalues) 1/1 (ordered)] (1, '杉迪·奇克丝', 2, '尤金 H. 克拉布斯')

.. tip:: 使用ORM进行join继承映射的批量INSERT时，需要ORM在内部使用  :paramref:`_dml.Insert.returning.sort_by_parameter_order`  参数，以便将来自基表的RETURNING行中的主键值与用于INSERT到“子”表中的参数集相关联，这就是为什么上面介绍的SQLite后端透明地退化到使用非批量语句的原因。 关于此功能的背景在   :ref:` engine_insertmanyvalues_returning_order`  。


.. _orm_queryguide_bulk_insert_w_sql:

使用SQL表达式的ORM批量插入
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ORM的批量插入功能支持添加一组固定参数，其中可以包括要应用于每个目标行的SQL表达式。为了实现这一点，结合使用  :meth:`_dml.Insert.values`  方法，传递一个字典参数将应用于所有行，以及通常的批量调用形式，通过在调用  :meth:` _orm.Session.execute`  时包含包含单个行值的参数字典列表。

例如，假设存在一个包括“timestamp”列的ORM映射：

.. sourcecode:: python

    import datetime


    class LogRecord(Base):
        __tablename__ = "log_record"
        id: Mapped[int] = mapped_column(primary_key=True)
        message: Mapped[str]
        code: Mapped[str]
        timestamp: Mapped[datetime.datetime]

如果我们想要INSERT一系列具有唯一“message”字段的``LogRecord``元素，但我们想要将SQL函数``now()``应用于所有行，则可以使用  :meth:`_dml.Insert.values`  将` `timestamp``传递，然后使用“bulk”模式传递其他记录：

.. sourcecode:: python

    >>> from sqlalchemy import func
    >>> log_record_result = session.scalars(
    ...     insert(LogRecord).values(code="SQLA", timestamp=func.now()).returning(LogRecord),
    ...     [
    ...         {"message": "日志消息 #1"},
    ...         {"message": "日志消息 #2"},
    ...         {"message": "日志消息 #3"},
    ...         {"message": "日志消息 #4"},
    ...     ],
    ... )
    {execsql}INSERT INTO log_record (message, code, timestamp)
    VALUES (?, ?, CURRENT_TIMESTAMP), (?, ?, CURRENT_TIMESTAMP),
    (?, ?, CURRENT_TIMESTAMP), (?, ?, CURRENT_TIMESTAMP)
    RETURNING id, message, code, timestamp
    [... (insertmanyvalues) 1/1 (unordered)] ('日志消息 #1', 'SQLA', '日志消息 #2',
    'SQLA', '日志消息 #3', 'SQLA', '日志消息 #4', 'SQLA')


    {stop}>>> print(log_record_result.all())
    [LogRecord('日志消息 #1', 'SQLA', datetime.datetime(...)),
     LogRecord('日志消息 #2', 'SQLA', datetime.datetime(...)),
     LogRecord('日志消息 #3', 'SQLA', datetime.datetime(...)),
     LogRecord('日志消息 #4', 'SQLA', datetime.datetime(...))]


.. _orm_queryguide_insert_values:

使用每行SQL表达式的ORM批量插入
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..  设置代码，不作显示

    >>> session.rollback()
    ROLLBACK
    >>> session.execute(
    ...     insert(User),
    ...     [
    ...         {
    ...             "name": "海绵宝宝",
    ...             "fullname": "海绵宝宝",
    ...             "species": "海绵",
    ...         },
    ...         {"name": "杉迪", "fullname": "杉迪·奇克丝", "species": "松鼠"},
    ...         {"name": "帕特里克", "species": "海星"},
    ...         {
    ...             "name": "章鱼哥",
    ...             "fullname": "章鱼哥",
    ...             "species": "章鱼",
    ...         },
    ...         {"name": "E·克拉布斯", "fullname": "尤金 H. 克拉布斯", "species": "蟹"},
    ...     ],
    ... )
    BEGIN...

  :meth:`_dml.Insert.values`   方法本身可以直接容纳参数字典列表。当没有向  :paramref:` _orm.Session.execute.params`  参数传递任何参数字典列表使用同样的方式使用   :class:`_dml.Insert`  构造时，不使用批量ORM插入模式，而是将INSERT语句按原样呈现并正好调用一次。此操作模式对于每行传递SQL表达式的情况可能很有用，并且在使用ORM时使用“upsert”语句时也使用，本章稍后会在   :ref:` orm_queryguide_upsert`  说明。

下面是一个虚构的例子，显示了一个嵌入每行SQL表达式的INSERT，并且还以该形式演示了  :meth:`_dml.Insert.returning`   ：

  >>> from sqlalchemy import select
  >>> address_result = session.scalars(
  ...     insert(Address)
  ...     .values(
  ...         [由于上面没有使用批量ORM插入模式，因此以下功能不可用：

* 不支持   :ref:`Joined table inheritance <orm_queryguide_insert_joined_table_inheritance>`  或其他多表映射，因为那将需要多个INSERT语句。

* 不支持   :ref:`Heterogenous parameter sets <orm_queryguide_insert_heterogeneous_params>`  - VALUES参数集中的每个元素必须具有相同的列。

* 不能使用核心级别的规模优化，例如提供的批处理   :ref:`insertmanyvalues <engine_insertmanyvalues>` ；语句将需要确保参数总数不超过由后备数据库强加的限制。

由于以上原因，通常不建议使用ORM INSERT语句中的多个参数集  :meth:`_dml.Insert.values` ，除非有明确的理由，即正在使用“upsert”，或者需要在每个参数集中嵌入每行SQL表达式。

.. seealso::

      :ref:`orm_queryguide_upsert` 


.. _orm_queryguide_legacy_bulk_insert:

Legacy Session 批量INSERT方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`_orm.Session`  包含了执行“批量”INSERT和UPDATE语句的旧版方法。这些方法共享实现
在描述了这些特性的 SQLAlchemy 2.0版本中，
在这里说明为：  :ref:`orm_queryguide_bulk_insert`  和   :ref:` orm_queryguide_bulk_update` ,
但缺少许多功能，即缺少 RETURNING 支持和 session-synchronization 支持。

使用  :meth:`.Session.bulk_insert_mappings`  的代码示例例如，可以将代码移植为以下方式，从此映射示例开始：

    session.bulk_insert_mappings(User, [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])

以上使用新的API表示如下：

    from sqlalchemy import insert

    session.execute(insert(User), [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])

.. seealso::

      :ref:`orm_queryguide_legacy_bulk_update` 


.. _orm_queryguide_upsert:

ORM "upsert" 语句 （UPSERT语句）
~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 具有选定的后端可能包括针对方言的专用   :class:`_dml.Insert`  构造，此外还具有将参数集中的现有行变为类似于 UPDATE 语句的插入的能力。通过 "现有行"，这可能意味着共享相同主键值的行。
或者可能是涉及到在行中将被认为是唯一的其他索引列；这取决于使用的后端的能力。

包括方言特定的 "upsert" API 功能的 SQLAlchemy 包括：

* SQLite - 使用   :class:`_sqlite.Insert` ，文档参见   :ref:` sqlite_on_conflict_insert` 
* PostgreSQL - 使用   :class:`_postgresql.Insert` ，文档参见   :ref:` postgresql_insert_on_conflict` 
* MySQL/MariaDB - 使用   :class:`_mysql.Insert` ，文档参见   :ref:` mysql_insert_on_duplicate_key_update` 

用户应查看上述章节以了解这些对象的正确构造背景；特别是，“upsert”方法通常需要引用原始语句，因此通常将语句构造为两个单独的步骤。

第三方后端，如   :ref:`external_toplevel`  中提到的后端，也可能具有类似的结构。

虽然 SQLAlchemy 还没有一个后端无关的 upsert 构造，但上述   :class:`_dml.Insert`  变量仍然是 ORM 兼容的，因为它们可以使用与   :class:` _dml.Insert`  构造本身相同的方式使用，如   :ref:`orm_queryguide_insert_values`  中所述，
也就是将所需的行嵌入到  :meth:`_dml.Insert.values`  方法中。在下面的示例中，使用 SQLite   :func:` _sqlite.insert`  函数生成包括 "ON CONFLICT DO UPDATE" 支持的   :class:`_sqlite.Insert` ，其中继续进行正常操作，另外一个特征是

传递给  :meth:`_dml.Insert.values`  的参数字典被解释为ORM映射属性键，而不是列名称：

..  设置代码，不要显示

    >>> session.rollback()
    ROLLBACK
    >>> session.execute(
    ...     insert(User).values(
    ...         [
    ...             dict(name="sandy"),
    ...             dict(name="spongebob", fullname="Spongebob Squarepants"),
    ...         ]
    ...     )
    ... )
    BEGIN...

::

    >>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
    >>> stmt = sqlite_upsert(User).values(
    ...     [
    ...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...         {"name": "patrick", "fullname": "Patrick Star"},
    ...         {"name": "squidward", "fullname": "Squidward Tentacles"},
    ...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ...     ]
    ... )
    >>> stmt = stmt.on_conflict_do_update(
    ...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
    ... )
    >>> session.execute(stmt)
    {execsql}INSERT INTO user_account (name, fullname)
    VALUES (?, ?), (?, ?), (?, ?), (?, ?), (?, ?)
    ON CONFLICT (name) DO UPDATE SET fullname = excluded.fullname
    [...] ('spongebob', 'Spongebob Squarepants', 'sandy', 'Sandy Cheeks',
    'patrick', 'Patrick Star', 'squidward', 'Squidward Tentacles',
    'ehkrabs', 'Eugene H. Krabs')
    {stop}<...>

.. _orm_queryguide_upsert_returning:

在upsert语句中使用RETURNING
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

从SQLAlchemy ORM的角度来看，upsert语句看起来像常规的   :class:`_dml.Insert`  构造，其中包括  :meth:` _dml.Insert.returning`  与在   :ref:`orm_queryguide_insert_values`  中演示的方式一样与upsert语句一起使用，因此可以传递任何列表达式或相关的ORM实体类。从上一节的示例中继续执行：

    >>> result = session.scalars(
    ...     stmt.returning(User), execution_options={"populate_existing": True}
    ... )
    {execsql}INSERT INTO user_account (name, fullname)
    VALUES (?, ?), (?, ?), (?, ?), (?, ?), (?, ?)
    ON CONFLICT (name) DO UPDATE SET fullname = excluded.fullname
    RETURNING id, name, fullname, species
    [...] ('spongebob', 'Spongebob Squarepants', 'sandy', 'Sandy Cheeks',
    'patrick', 'Patrick Star', 'squidward', 'Squidward Tentacles',
    'ehkrabs', 'Eugene H. Krabs')
    {stop}>>> print(result.all())
    [User(name='spongebob', fullname='Spongebob Squarepants'),
      User(name='sandy', fullname='Sandy Cheeks'),
      User(name='patrick', fullname='Patrick Star'),
      User(name='squidward', fullname='Squidward Tentacles'),
      User(name='ehkrabs', fullname='Eugene H. Krabs')]

上面的示例使用 RETURNING 返回每一行插入或 upsert 的 ORM 对象。示例还添加了使用   :ref:`orm_queryguide_populate_existing`  执行选项。此选项表示对于在   :class:` _orm.Session`  中已存在的行，应该使用新行的数据来 **刷新** 已经存在的“User”对象。对于纯   :class:`_dml.Insert`  语句，此选项不重要，因为生成的每一行都是全新的主键标识符。但是，当   :class:` _dml.Insert`  也包含"upsert"选项时，它也可能产生来自已经存在的行的结果，因此可能已经在   :class:`_orm.Session`  对象的  :term:` identity map`  中表示了一个主键标识符。

.. 参见::

      :ref:`orm_queryguide_populate_existing` 


.. _orm_queryguide_bulk_update:

ORM基于主键的批量UPDATE
------------------------------

..  设置代码，不要显示

    >>> session.rollback()
    ROLLBACK
    >>> session.execute(
    ...     insert(User),
    ...     [
    ...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...         {"name": "patrick", "fullname": "Patrick Star"},
    ...         {"name": "squidward", "fullname": "Squidward Tentacles"},
    ...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ...     ],
    ... )
    BEGIN ...
    >>> session.commit()
    COMMIT...
    >>> session.connection()
    BEGIN ...

  :class:`_dml.Update`  建构可以与类似于   :class:` _dml.Insert`  语句在   :ref:`orm_queryguide_bulk_insert`  中描述的类似方式与  :meth:` _orm.Session.execute`  一起使用，传递一个将更改应用到本地对象上的主键和字典的列表。参数字典的列表中，每个字典表示一个与单个主键值对应的行。不要将此用法与ORM中更常见的使用   :class:`_dml.Update`  语句的方式混淆，其中使用显式 WHERE 子句，有关文档请参见   :ref:` orm_queryguide_update_delete_where` 。

对于“批量”版本的 UPDATE，通过一个ORM类形成一个   :func:`_dml.update`  构造体传递给  :meth:` _orm.Session.execute`  方法；生成的   :class:`_dml.Update`  对象**不应该有任何值和通常没有WHERE条件**，也就是说，不使用  :meth:` _dml.Update.values`  方法，而  :meth:`_dml.Update.where`  方法通常**不会**使用，但在 unusual 的情况下可能需要添加其他过滤条件。

将一个由多个参数字典组成的列表和   :class:`_dml.Update`  构造体一起传递给  :meth:` _orm.Session.execute`  方法将会调用 **bulk update by primary key mode** 以进行update操作，为语句生成适当的WHERE条件以匹配每行的主键，并使用  :term:`executemany`  对 UPDATE 语句运行每个参数集::

    >>> from sqlalchemy import update
    >>> session.execute(
    ...     update(User),
    ...     [
    ...         {"id": 1, "fullname": "Spongebob Squarepants"},
    ...         {"id": 3, "fullname": "Patrick Star"},
    ...         {"id": 5, "fullname": "Eugene H. Krabs"},
    ...     ],
    ... )
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.id = ?
    [...] [('Spongebob Squarepants', 1), ('Patrick Star', 3), ('Eugene H. Krabs', 5)]
    {stop}<...>


请注意，每个参数字典 **必须** 包括每个记录的完整主键，否则会引发错误。

与批量INSERT特性一样，异构参数列表也被支持，它们的参数将被分组为更新运行的子批次。

.. versionchanged:: 2.0.11 通过使用 `  :meth:`_dml.Update.where`   方法可以将更多的 WHERE 条件与  :ref: orm_queryguide_bulk_update 一起使用以添加附加过滤条件。 但是，此条件始终是在已经存在的 WHERE 条件之后添加的，其中包括主键值。

使用“bulk update by primary key”功能时，没有可用的RETURNING功能；使用多个参数字典的列表时，DBAPI  :term:`executemany`  通常不支持结果行。


.. versionchanged:: 2.0 与单个参数独立值不同，在  :meth:`_orm.Session.execute`  方法中传递   :class:` _dml.Update`  构造体并搭配一个参数字典的列表现在会调用“bulk update”，该特性使用与旧版  :meth:`_orm.Session.bulk_update_mappings`  方法相同的功能。这是与 1.x 系列相比的行为变化，因为在 1.x 系列中，   :class:` _dml.Update`  仅支持带有显式 WHERE 条件和内部值。

.. _orm_queryguide_bulk_update_disabling:

禁用使用批量ORM更新的多参数集的UPDATE语句的主键更新
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ORM的“Bulk Update by Primary Key” 功能，对每个包含每个主键值的WHERE条件的记录运行更新语句，当满足以下条件时自动启用：

1. 给定的 UPDATE 语句针对 ORM 实体。
2.   :class:`_orm.Session`  用于执行语句，而不是核心   :class:` _engine.Connection` 。
3. 传递的参数是**字典列表**。

为了调用不使用“ORM Bulk Update by Primary Key”的UPDATE语句，直接针对   :class:`_engine.Connection`  执行语句，使用  :meth:` _orm.Session.connection`  方法获取当前一   :class:`_engine.Connection` ::


    >>> from sqlalchemy import bindparam
    >>> session.connection().execute(
    ...     update(User).where(User.name == bindparam("u_name")),
    ...     [
    ...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"u_name": "patrick", "fullname": "Patrick Star"},
    ...     ],
    ... )
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.name = ?
    [...] [('Spongebob Squarepants', 'spongebob'), ('Patrick Star', 'patrick')]
    {stop}<...>

.. seealso::

      :ref:`error_bupq` 

.. _orm_queryguide_bulk_update_joined_inh:

使用联接表继承的主键批量更新
~~~~~~~~~~~~~~~~~~~~~~~~~~

..  设置代码，不用显示

    >>> session.execute(
    ...     insert(Manager).returning(Manager),
    ...     [
    ...         {"name": "sandy", "manager_name": "Sandy Cheeks"},
    ...         {"name": "ehkrabs", "manager_name": "Eugene H. Krabs"},
    ...     ],
    ... )
    INSERT...
    >>> session.commit()
    COMMIT...
    >>> session.connection()
    BEGIN (implicit)...ORM批量更新与使用映射的ORM批量插入相似，该映射具有连接表继承；如  :ref:`orm_queryguide_insert_joined_table_inheritance` 所述，批量UPDATE操作将为映射中表示的每个表发出一条UPDATE语句，其给定参数包括要更新的值（不受影响的表将被跳过）。

例如：

```
>>> session.execute(
...     update(Manager),
...     [
...         {
...             "id": 1,
...             "name": "scheeks",
...             "manager_name": "Sandy Cheeks, President",
...         },
...         {
...             "id": 2,
...             "name": "eugene",
...             "manager_name": "Eugene H. Krabs, VP Marketing",
...         },
...     ],
... )
{execsql}UPDATE employee SET name=? WHERE employee.id = ?
[...] [('scheeks', 1), ('eugene', 2)]
UPDATE manager SET manager_name=? WHERE manager.id = ?
[...] [('Sandy Cheeks, President', 1), ('Eugene H. Krabs, VP Marketing', 2)]
{stop}<...>
```

.. _orm_queryguide_legacy_bulk_update:

传统会话批量更新方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如   :ref:`orm_queryguide_legacy_bulk_insert`  的
  :meth:`_orm.Session.bulk_update_mappings`  方法是批量更新的传统形式，ORM在内部解释给定主键参数的  :func:` _sql.update`语句时使用此方法；但是，当使用遗留版本时，不包括同步会话支持等功能。

以下是示例：

```
session.bulk_update_mappings(
        User,
        [
            {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
            {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
        ],
    )
```

新API的表达方式为：

```
from sqlalchemy import update

session.execute(
        update(User),
        [
            {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
            {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
        ],
    )
```

.. seealso::

      :ref:`orm_queryguide_legacy_bulk_insert` 



.. _orm_queryguide_update_delete_where:

带有自定义WHERE条件的ORM UPDATE and DELETE
------------------------------------------------

..  设置代码，不显示

  :class:`_dml.Update`  和  :meth:` _dml.Delete.where`  方法）构造时，
可以通过在ORM上下文中将它们传递给  :meth:`_orm.Session.execute`  而不使用：paramref:` _orm.Session.execute.params`参数进行调用。
对于`_dml.Update`，应使用  :meth:`_dml.Update.values`  传递要更新的值。

使用这种模式的不同之处在于与  :ref:`orm_queryguide_bulk_update` 中先前介绍的特性不同，ORM使用给定的WHERE子句，而不是将WHERE子句固定为主键。
这意味着单个UPDATE或DELETE语句可以一次性影响许多行。

例如，下面发出一个update会影响多行的fullname字段的语句：

```
from sqlalchemy import update
stmt = (
        update(User)
        .where(User.name.in_(["squidward", "sandy"]))
        .values(fullname="Name starts with S")
    )
session.execute(stmt)
```

对于DELETE，以下是基于条件删除行的示例：

```
from sqlalchemy import delete
stmt = delete(User).where(User.name.in_(["squidward", "sandy"]))
session.execute(stmt)
```

..  设置代码，不显示

.. _orm_queryguide_update_delete_sync:

选择同步策略
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在使用ORM启用  :meth:`_orm.Session.execute`  进行执行时，通过   :func:` _dml.update`  通过“同步”，我们指的是将UPDATE的属性刷新为包含在ORM会话的  :term:`identity map`  中的对象属性。新值，或至少是  :term:` 已过期` ，以便它们在下次访问时重新填充其新值，而 DELETE 的对象将被移动到  :term:`已删除`  状态。

这种同步可以作为“同步策略”进行控制，该策略作为字符串 ORM 执行选项传递，通常使用  :paramref:`_orm.Session.execute.execution_options`  字典实现：

    >>> from sqlalchemy import update
    >>> stmt = (
    ...     update(User).where(User.name == "squidward").values(fullname="Squidward Tentacles")
    ... )
    >>> session.execute(stmt, execution_options={"synchronize_session": False})
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.name = ?
    [...] ('Squidward Tentacles', 'squidward')
    {stop}<...>

执行选项也可以与语句本身捆绑在一起，使用  :meth:`_sql.Executable.execution_options`  方法实现：

    >>> from sqlalchemy import update
    >>> stmt = (
    ...     update(User)
    ...     .where(User.name == "squidward")
    ...     .values(fullname="Squidward Tentacles")
    ...     .execution_options(synchronize_session=False)
    ... )
    >>> session.execute(stmt)
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.name = ?
    [...] ('Squidward Tentacles', 'squidward')
    {stop}<...>

支持以下的 ``synchronize_session`` 值：

- ``'auto'`` - 这是默认值。在支持 RETURNING 的后端上将使用 ``'fetch'`` 策略，其中包括所有 SQLAlchemy 本机驱动程序，除 MySQL 外。如果不支持 RETURNING，则改为使用 ``'evaluate'`` 策略。

- ``'fetch'`` - 通过在 UPDATE 或 DELETE 之前执行 SELECT，或者使用 RETURNING 来检索受影响行的主键标识，以便可以使用新值（更新）刷新受操作影响的内存中的对象，或将其从   :class:`_orm.Session` （删除）。即使使用  :meth:` _dml.UpdateBase.returning`  或  :meth:`_dml.delete`  指定了实体或列，也可以使用此同步策略。

  .. versionchanged:: 2.0 当使用具有 WHERE 准则的 ORM 启用的 UPDATE 和 DELETE 时，可以将显式  :meth:`_dml.UpdateBase.returning`  与 ` `'fetch'`` 同步策略相结合。实际语句将包含 ``'fetch'`` 策略要求和请求的列之间的交集。

- ``'evaluate'`` - 这表示在 Python 中评估 UPDATE 或 DELETE 语句中给出的 WHERE 准则，以在   :class:`_orm.Session`  中查找匹配的对象。此方法不会为操作添加任何 SQL 往返，并且在缺少 RETURNING 支持时可能更高效。对于具有复杂准则的 UPDATE 或 DELETE 语句，` `'evaluate'`` 策略可能无法在 Python 中计算表达式，并会引发错误。如果出现这种情况，应改为使用操作的 ``'fetch'`` 策略。

  .. tip::

    如果 SQL 表达式使用  :meth:`_sql.Operators.op`  或   :class:` _sql.custom_op`  特性使用自定义操作符，则可以使用  :paramref:`_sql.Operators.op.python_impl`  参数来指示将在 ` `"evaluate"`` 同步策略中使用的 Python 函数。

    .. versionadded:: 2.0

  .. warning::

    如果 UPDATE 操作将在已过期的   :class:`_orm.Session`  上运行，则应避免使用 ` `"evaluate"`` 同步策略，因为它必须刷新对象以根据给定的 WHERE 准则测试它们，这将为每个对象发出 SELECT。在这种情况下，特别是如果后端支持 RETURNING，则应首选 ``"fetch"`` 策略。

* ``False`` - 不同步会话。在不支持 RETURNING 的后端中，“evaluate”策略无法使用。在这种情况下，   :class:`_orm.Session`  中对象的状态保持不变，不会自动对应于发出的 UPDATE 或 DELETE 语句，如果存在与通常对应于匹配行的对象，则不会对应。

.. _orm_queryguide_update_delete_where_returning:

使用 RETURNING 与 UPDATE/DELETE 和自定义 WHERE 准则
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :meth:`.UpdateBase.returning`   方法与启用 ORM 的 WHERE 准则 UPDATE 和 DELETE 完全兼容。 可以为 RETURNING 指示完整的 ORM 对象和/或列：

    >>> from sqlalchemy import update
    >>> stmt = (
    ...     update(User)
    ...     .where(User.name == "squidward")
    ...     .values(fullname="Squidward Tentacles")
    ...     .returning(User)
    ... )
    >>> result = session.scalars(stmt)
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.name = ?
    RETURNING id, name, fullname, species
    [...] ('Squidward Tentacles', 'squidward')
    {stop}>>> print(result.all())
    [User(name='squidward', fullname='Squidward Tentacles')]

针对 RETURNING 的支持还与 ``fetch`` 同步兼容。使用RETURNING的UPDATE/DELETE
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ORM 将根据需要对 RETURNING 中的列进行排序，以使同步进行得更好，且返回的   :class:`.Result`  将按请求的实体和 SQL 列以其请求的顺序进行。

.. versionadded:: 2.0   :meth:`.UpdateBase.returning`  可用于启用 ORM 的 UPDATE 和 DELETE，同时仍然保留了与 ` `fetch`` 同步策略的完全兼容性。

.. _orm_queryguide_update_delete_joined_inh:

带有自定义 WHERE Criteria 的 JOIN 表继承的 UPDATE/DELETE
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  不显示设置代码

    >>> session.rollback()
    ROLLBACK...
    >>> session.connection()
    BEGIN (implicit)...

与   :ref:`orm_queryguide_bulk_update`  不同，带有 WHERE criteria 的 UPDATE/DELETE 功能仅对每个  :meth:` _orm.Session.execute`  或   :func:`_dml.delete`  语句时，该语句必须符合后端当前的功能，其中包括后端不支持引用多个表的 UPDATE 或 DELETE 语句或仅对此提供有限支持。这意味着针对类似连接继承子类的映射，UPDATE/DELETE with WHERE criteria 功能的 ORM 版本只能在具体情况下有限或完全无法使用。

发出一个 JOIN 表子类的多行 UPDATE 语句的最简单方法是仅引用子类表中的子表。这意味着   :func:`_dml.Update`  构造只应引用本地于子类表的属性，如下面的示例所示::

    >>> stmt = (
    ...     update(Manager)
    ...     .where(Manager.id == 1)
    ...     .values(manager_name="Sandy Cheeks, President")
    ... )
    >>> session.execute(stmt)
    {execsql}UPDATE manager SET manager_name=? WHERE manager.id = ?
    [...] ('Sandy Cheeks, President', 1)
    <...>

通过上面的形式，为了在任何 SQL 后端上工作，引用基础表以定位行的原始方法是使用子查询::

    >>> stmt = (
    ...     update(Manager)
    ...     .where(
    ...         Manager.id
    ...         == select(Employee.id).where(Employee.name == "sandy").scalar_subquery()
    ...     )
    ...     .values(manager_name="Sandy Cheeks, President")
    ... )
    >>> session.execute(stmt)
    {execsql}UPDATE manager SET manager_name=? WHERE manager.id = (SELECT employee.id
    FROM employee
    WHERE employee.name = ?) RETURNING id
    [...] ('Sandy Cheeks, President', 'sandy')
    {stop}<...>

对于支持 UPDATE...FROM 的后端，子查询可以以额外的明文 WHERE 条件表示，然后必须以某种明确的方式显式地表达两个表之间的条件::

    >>> stmt = (
    ...     update(Manager)
    ...     .where(Manager.id == Employee.id, Employee.name == "sandy")
    ...     .values(manager_name="Sandy Cheeks, President")
    ... )
    >>> session.execute(stmt)
    {execsql}UPDATE manager SET manager_name=? FROM employee
    WHERE manager.id = employee.id AND employee.name = ?
    [...] ('Sandy Cheeks, President', 'sandy')
    {stop}<...>

对于 DELETE，预期将同时删除基表和子表中的行。为了 **不使用级联外键** 删除连接继承对象的多行，请为每个表单独发出一个 DELETE 语句::

    >>> from sqlalchemy import delete
    >>> session.execute(delete(Manager).where(Manager.id == 1))
    {execsql}DELETE FROM manager WHERE manager.id = ?
    [...] (1,)
    {stop}<...>
    >>> session.execute(delete(Employee).where(Employee.id == 1))
    {execsql}DELETE FROM employee WHERE employee.id = ?
    [...] (1,)
    {stop}<...>

总体而言，对于连接继承和其他多表映射，请优先使用正常的  :term:`工作单元`  进程更新和删除行，除非使用自定义 WHERE criteria 有性能方面的原理。

旧的查询方法
~~~~~~~~~~~~~~

ORM 启用的带有 WHERE 的 UPDATE/DELETE 最初是   :class:`.Query`  对象的一部分，位于  :meth:` _orm.Query.update`  和  :meth:`_orm.Query.delete`  方法中。这些方法仍然可用，并提供与   :ref:` orm_queryguide_update_delete_where`  中所述的相同功能的子集。主要区别是旧的方法不提供明确的 RETURNING 支持。

.. seealso::

     :meth:`_orm.Query.update` 

     :meth:`_orm.Query.delete` 

..  不显示设置代码

    >>> session.close()
    ROLLBACK...
    >>> conn.close()