.. highlight:: pycon+sql
.. |prev| replace:: :doc:`inheritance`
.. |next| replace:: :doc:`columns`

.. include:: queryguide_nav_include.rst

.. doctest-include _dml_setup.rst

.. _orm_expression_update_delete:

支持 ORM 的 INSERT、UPDATE 和 DELETE 语句
=================================================

.. admonition:: About this Document （关于本文件）

    本节使用 ORM 映射，可在 :ref:`unified_tutorial` 部分阐述的基础上处理 ORM-Enabled
    :class:`_sql.Insert`, :class:`_sql.Update` 和 :class:`_sql.Delete` 对象。
    本节同时使用了继承映射在 :ref:`inheritance_toplevel` 部分展示。

    请查看本页相关的 :doc:`ORM 设置 <_dml_setup>`。

在使用 :meth:`_orm.Session.execute` 方法处理 ORM-enabled :class:`_sql.Select` 对象的基础上，
该方法可以处理 ORM-enabled :class:`_sql.Insert`, :class:`_sql.Update` 和 :class:`_sql.Delete` 对象，从而以各种方式同时插入、更新或删除多个数据库行。
此外，对于 ORM-enabled “upserts”，即自动使用 UPDATE 为已存在的数据行插入的 INSERT 语句，不同的 SQL 方言提供了不同的支持。

下面的表总结了本文所讨论的各种调用形式：

=====================================================   ==========================================   ========================================================================     ========================================================= ============================================================================
ORM Use Case                                            DML Construct Used                           Data is passed using ...                                                     Supports RETURNING?                                       Supports Multi-Table Mappings?
=====================================================   ==========================================   ========================================================================     ========================================================= ============================================================================
:ref:`orm_queryguide_bulk_insert`                       :func:`_dml.insert`                          传递给 :paramref:`_orm.Session.execute.params` 的字典列表。               :ref:`yes <orm_queryguide_bulk_insert_returning>`         :ref:`yes <orm_queryguide_insert_joined_table_inheritance>`
:ref:`orm_queryguide_bulk_insert_w_sql`                 :func:`_dml.insert`                          带有 :meth:`_dml.Insert.values` 的 :paramref:`_orm.Session.execute.params`  :ref:`yes <orm_queryguide_bulk_insert_w_sql>`             :ref:`yes <orm_queryguide_insert_joined_table_inheritance>`
:ref:`orm_queryguide_insert_values`                     :func:`_dml.insert`                          传递给 :meth:`_dml.Insert.values` 的字典列表。                              :ref:`yes <orm_queryguide_insert_values>`                 no
:ref:`orm_queryguide_upsert`                            :func:`_dml.insert`                          传递给 :meth:`_dml.Insert.values` 的字典列表。                              :ref:`yes <orm_queryguide_upsert_returning>`              no
:ref:`orm_queryguide_bulk_update`                       :func:`_dml.update`                          传递给 :paramref:`_orm.Session.execute.params` 的字典列表。               no                                                        :ref:`yes <orm_queryguide_bulk_update_joined_inh>`
:ref:`orm_queryguide_update_delete_where`               :func:`_dml.update`, :func:`_dml.delete`     传递给 :meth:`_dml.Update.values` 的关键字。                                    :ref:`yes <orm_queryguide_update_delete_where_returning>` :ref:`partial, with manual steps <orm_queryguide_update_delete_joined_inh>`
=====================================================   ==========================================   ========================================================================     ========================================================= ============================================================================



.. _orm_queryguide_bulk_insert:

ORM 批量 INSERT 语句
--------------------------

可以通过 ORM 类以 :func:`_dml.insert` 构造，再将其传递给 :meth:`_orm.Session.execute` 方法来设置 ORM 批量 INSERT，通过在 :paramref:`_orm.Session.execute.params` 参数中传递一个参数字典列表来触发。
除了 :class:`_dml.Insert` 之外，还会优化此操作以尽可能的快速处理多行：

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

参数字典包含键/值对，这些对应于与映射的 :class:`._schema.Column` 或 :func:`_orm.mapped_column` 声明相匹配的 ORM 映射属性，以及与 :ref:`composite <mapper_composite>` 声明相一致的属性。如果这两个名称不同，
则键应与 ORM 映射属性名匹配。

.. versionchanged:: 2.0 
   将 :class:`_dml.Insert` 构造传递到 :meth:`_orm.Session.execute` 方法现在会调用“批量插入”功能，该功能使用的是与旧版 :meth:`_orm.Session.bulk_insert_mappings` 方法相同的功能。这是与 1.x 系列相比的行为更改，1.x 系列会将 :class:`_dml.Insert` 解释为基于 Core 的方式，使用列名称作为取值键；现在可以接受 ORM 属性键。可以为 :meth:`_orm.Session.execute` 的 :paramref:`_orm.Session.execution_options` 参数传递执行选项 `{"dml_strategy": "raw"}` 以提供 Core 级别的功能。

.. _orm_queryguide_bulk_insert_returning:

使用 RETURNING 获取新对象
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  Setup code, not for display

  >>> session.rollback()
  ROLLBACK...
  >>> session.connection()
  BEGIN (implicit)...

对于支持 INSERT..RETURNING 语法的选定后端，ORM 批量插入功能支持还可对 SELECT 子句中返回的 :class:`.Result` 对象执行相同操作，并返回每个列以及完全构建的 ORM 对象。该操作要求所使用的后端支持 SQL RETURNING 语法以及带有 RETURNING 的 :term:`executemany` 支持；除了 MySQL（MariaDB 包括）之外的所有包括在内的 :ref:`SQLAlchemy-included <included_dialects>` 后端都支持此功能。

例如，可以运行与之前相同的语句，使用 :meth:`.UpdateBase.returning` 方法（将完整的“User”实体传递为返回值）：

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

在上面的示例中，生成的 SQL 使用了 :ref:`insertmanyvalues <engine_insertmanyvalues>` 选项使用了将单个参数字典内嵌成单个 INSERT 语句的形式，以便可以使用 RETURNING。

.. versionchanged:: 2.0
   ORM :class:`.Session` 现在在 ORM 上下文中解释 :class:`_dml.Insert`, :class:`_dml.Update` 甚至 :class:`_dml.Delete` 构造中的 RETURNING 子句，在这些构造中可以混合列表达式和 ORM 映射实体，这些将通过 :meth:`_dml.Insert.returning` 方法传递返回，并包括使用 ORM 结果传递时产生的 ORM mapView 映射的实体。此外还提供了对 ORM loader 选项（例如 :func:`_orm.load_only` 和 :func:`_orm.selectinload`）的有限支持。

.. _orm_queryguide_bulk_insert_returning_ordered:

将返回记录与输入数据顺序相关联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用带有 RETURNING 的批量 INSERT 时，重要的是要注意，大多数数据库后端并没有正式保证按返回的顺序返回记录，包括没有保证其顺序与输入记录相对应的保证。对于需要确保 RETURNING 记录与输入数据相关的应用程序，可使用附加参数 :paramref:`_dml.Insert.returning.sort_by_parameter_order`（根据所使用的后端可能使用保持令牌的特殊 INSERT 表单（该令牌用于正确重新排序返回的行），或在某些情况下，例如在下面使用 SQLite 后端的示例中，操作将一次插入一行：

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
    [insertmanyvalues 3/3 (ordered; batch not supported)] ('gary', 'Gary')
    {stop}>>> for user_id, input_record in zip(user_ids, data):
    ...     input_record["id"] = user_id
    >>> print(data)
    [{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
    {'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
    {'name': 'gary', 'fullname': 'Gary', 'id': 8}]

.. versionadded:: 2.0.10
   添加了 :paramref:`_dml.Insert.returning.sort_by_parameter_order`，该参数在 :term:`insertmanyvalues` 架构中实现。

.. seealso::

    :ref:`engine_insertmanyvalues_returning_order` - 这里介绍了在不显著降低性能的情况下保证输入数据与结果行对应的方法的背景知识

.. _orm_queryguide_insert_heterogeneous_params:

使用异构参数字典
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  Setup code, not for display

  >>> session.rollback()
  ROLLBACK...
  >>> session.connection()
  BEGIN (implicit)...

ORM 批量插入功能支持异构的参数字典列表，"异构" 基本上意味着"每个字典中的键可能不同"。当检测到此条件时，
ORM 将会根据每组键将这些参数字典分为一组并进行批处理：

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



在上面的示例中，传递的五个参数字典分别对应着三个 INSERT 语句，因为每个字典都有特定的键集，因此可以分组，单独处理。

.. _orm_queryguide_insert_joined_table_inheritance:

Joined Table Inheritance 批量 INSERT
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  Setup code, not for display

    >>> session.rollback()
    ROLLBACK
    >>> session.connection()
    BEGIN...

ORM 批量插入是在内部单元工作系统中加以实现的。该系统用于生成 INSERT 语句，因此对于多个映射到多个表格的 ORM 实体（通常是使用 :ref:`joined table inheritance <joined_inheritance>` 进行映射的实体），批量 INSERT 操作将为由映射表示的每个表格发出一个 INSERT 语句，并将服务器生成的主键值正确传输到依赖于它们的表格行中。在这里还支持 RETURNING，即 ORM 将接收每条 INSERT 语句执行的 :class:`.Result` 对象，然后“水平拼合”它们，以便返回的行包括所有插入的值：

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
    [insertmanyvalues 2/2 (ordered; batch not supported)] ('ehkrabs', 'manager')
    INSERT INTO manager (id, manager_name) VALUES (?, ?), (?, ?) RETURNING id, manager_name, id AS id__1
    [... (insertmanyvalues) 1/1 (ordered)] (1, 'Sandy Cheeks', 2, 'Eugene H. Krabs')

.. tip:: 批量插入：加入继承映射需要 ORM 在内部使用 :paramref:`_dml.Insert.returning.sort_by_parameter_order` 来解析它们，以便可以将来自 RETURNING 的主键值对应到用于插入到“子”表中的参数集中，这就是为什么上面使用 SQLite 后端透明地降级为使用非批处理语句的原因。
  
.. _orm_queryguide_bulk_insert_w_sql:

使用 SQL 表达式进行 ORM 批量插入
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ORM 批量插入支持添加一组固定的参数，该组参数可能包括用于每个目标行的 SQL 表达式。为此，请将 :meth:`_dml.Insert.values` 方法与通常的批量调用形式组合使用，通过在 :meth:`_orm.Session.execute` 中包括一个具有单独行参数字典的参数字典列表来传递。

例如，假设 ORM 映射包括“timestamp”列：

.. sourcecode:: python

    import datetime


    class LogRecord(Base):
        __tablename__ = "log_record"
        id: Mapped[int] = mapped_column(primary_key=True)
        message: Mapped[str]
        code: Mapped[str]
        timestamp: Mapped[datetime.datetime]

如果我们想要插入一系列带有唯一“message”字段的“LogRecord”元素，但是我们希望在所有行中都应用 SQL 函数 "now()"，我们可以通过将 “timestamp” 存储在 :meth:`_dml.Insert.values` 中来实现，然后使用“批量”模式传递额外的记录。::

    >>> from sqlalchemy import func
    >>> log_record_result = session.scalars(
    ...     insert(LogRecord).values(code="SQLA", timestamp=func.now()).returning(LogRecord),
    ...     [
    ...         {"message": "log message #1"},
    ...         {"message": "log message #2"},
    ...         {"message": "log message #3"},
    ...         {"message": "log message #4"},
    ...     ],
    ... )
    {execsql}INSERT INTO log_record (message, code, timestamp)
    VALUES (?, ?, CURRENT_TIMESTAMP), (?, ?, CURRENT_TIMESTAMP),
    (?, ?, CURRENT_TIMESTAMP), (?, ?, CURRENT_TIMESTAMP)
    RETURNING id, message, code, timestamp
    [... (insertmanyvalues) 1/1 (unordered)] ('log message #1', 'SQLA', 'log message #2',
    'SQLA', 'log message #3', 'SQLA', 'log message #4', 'SQLA')


    {stop}>>> print(log_record_result.all())
    [LogRecord('log message #1', 'SQLA', datetime.datetime(...)),
     LogRecord('log message #2', 'SQLA', datetime.datetime(...)),
     LogRecord('log message #3', 'SQLA', datetime.datetime(...)),
     LogRecord('log message #4', 'SQLA', datetime.datetime(...))]
    
由于使用了 :meth:`_dml.Insert.values`，所以不会使用批量 ORM 插入模式。

以下特征将不可用：

* 不支持 :ref:`Joined table inheritance <orm_queryguide_insert_joined_table_inheritance>` 或其他多表映射，因为这需要多个 INSERT 语句。

* 不支持 :ref:`Heterogenous parameter sets <orm_queryguide_insert_heterogeneous_params>`。每个 VALUES 集元素必须具有相同的列。

* 不可用 Core 级别的规模优化，例如 :ref:`insertmanyvalues <engine_insertmanyvalues>` 的批处理; 语句需要确保参数总数不超过备份数据库强制规定的限制。

出于这些原因，通常不建议在 ORM INSERT 语句中使用 :meth:`_dml.Insert.values` 的多个参数集，除非存在明确理由，即正在使用“upsert”或需要在每个参数集中嵌入每行的 SQL 表达式。

.. seealso::

    :ref:`orm_queryguide_upsert`

.. _orm_queryguide_legacy_bulk_insert:

旧版 Session 批量插入方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`_orm.Session` 包括处理“批量” INSERT 和 UPDATE 语句的旧版方法。这些方法使用与描述在 :ref:`orm_queryguide_bulk_insert` 和 :ref:`orm_queryguide_bulk_update` 中的 SQLAlchemy 2.0 版本具有相同实现，但不支持很多功能，即没有与回收支持，也不支持使用 RETURNING。

例如，使用 :meth:`.Session.bulk_insert_mappings` 的代码可以移植如下：

    session.bulk_insert_mappings(User, [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])

上述代码可以用新 API 如下语句表示：

    from sqlalchemy import insert

    session.execute(insert(User), [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])

.. seealso::

    :ref:`orm_queryguide_legacy_bulk_update`


.. _orm_queryguide_upsert:

ORM "upsert" 语句
~~~~~~~~~~~~~~~~~~~~~~~

使用 SQLAlchemy 的标准 SQL 方言，可能包括有特殊支持 upsert的 :class:`_dml.Insert` 构造，它还具有将现有的参数集的行转换为近似 UPDATE 语句的能力。由“现有行”指的是行，这些行具有相同的主键值，或者是在行中被认为是唯一的索引列；这取决于所使用的后端的能力。

包括具有特定于 SQL 的“upsert” API 功能的 SQLAlchemy 随附方言是：

* SQLite - 使用 :class:`_sqlite.Insert`，在 :ref:`sqlite_on_conflict_insert` 中记录
* PostgreSQL - 使用 :class:`_postgresql.Insert`，在 :ref:`postgresql_insert_on_conflict` 中记录
* MySQL/MariaDB - 使用 :class:`_mysql.Insert`，在 :ref:`mysql_insert_on_duplicate_key_update` 中记录

用户应查看上述各章节了解适当构建这些对象的背景知识；特别是“upsert”方法通常需要参考原始语句，因此通常需要分两个步骤构造声明。

第三方后端（例如位于 :ref:`external_toplevel` 中提到的后端）可能也提供类似的构造方法。

虽然 SQLAlchemy 还没有通用的 upsert 构造，但上述的 :class:`_dml.Insert` 变体在 ORM 兼容方面仍然可用，因为它们可以用与 :class:`_dml.Insert` 构造本身相同的方式使用，如在 :ref:`orm_queryguide_insert_values` 中所述，即通过将期望插入的行嵌入到 :meth:`_dml.Insert.values` 方法中的方式。在下面的示例中，使用 SQLite 的 :func:`_sqlite.insert` 函数生成一个 :class:`_sqlite.Insert` 构造，其中包括 "ON CONFLICT DO UPDATE" 选项。然后将语句传递给 :meth:`_orm.Session.execute`，其中还附带了在 :meth:`_dml.Insert.values` 中嵌入的所需行参数字典；这些将被解释为 ORM 映射属性键，而不是列名称：

..  Setup code, not for display

    >>> session.rollback()
    ROLLBACK
    >>> session.execute(
    ...     insert(User).values(
    ...         [
    ...             {"name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...             {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...         ],
    ...         on_conflict_do_update={
    ...             "index_elements": [User.name],
    ...             "set_": {"fullname": "My New Name"},
    ...         },
    ...     ),
    ... )
    BEGIN...

.. seealso::

    :ref:`orm_queryguide_upsert`.. _orm_queryguide_update_delete_where:

ORM自定义更新和删除条件
----------------------

:class:`_dml.Update`和:class:`_dml.Delete`构造，当构造使用自定义WHERE条件（即使用:meth:`_dml.Update.where`和:meth:`_dml.Delete.where`函数)时，可以在ORM上下文中通过将它们传递给:meth:`_orm.Session.execute`方法来调用，而不使用:param:`_orm.Session.execute.params`参数。对于:class:`_dml.Update`，要更新的值应该使用:meth:`_dml.Update.values`传递。

这种使用方式不同于在 :ref:`orm_queryguide_bulk_update` 中描述的特性，后者使用自动为每个记录生成主键的确切WHERE条件执行UPDATE或DELETE语句，并使用:term:`executemany`对每个参数集运行。 

例如，下面的更新语句会影响“fullname”字段的多个行：

..

    >>> from sqlalchemy import update
    >>> stmt = (
    ...     update(User)
    ...     .where(User.name.in_(["squidward", "sandy"]))
    ...     .values(fullname="Name starts with S")
    ... )
    >>> session.execute(stmt)
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.name IN (?, ?)
    [...] ('Name starts with S', 'squidward', 'sandy')
    {stop}<...>

对于DELETE，以下是基于条件删除行的示例：

..
    >>> from sqlalchemy import delete
    >>> stmt = delete(User).where(User.name.in_(["squidward", "sandy"]))
    >>> session.execute(stmt)
    {execsql}DELETE FROM user_account WHERE user_account.name IN (?, ?)
    [...] ('squidward', 'sandy')
    {stop}<...>

在ORM启用的UPDATE和DELETE WHERE标准中，与返回的支持完全兼容，支持完全。可以指定完整ORM对象和/或列来进行返回：


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



.. seealso::

    :meth:`_orm.Query.update`

    :meth:`_orm.Query.delete`