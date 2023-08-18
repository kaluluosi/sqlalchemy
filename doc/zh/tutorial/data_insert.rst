.. highlight:: pycon+sql

.. |prev| replace:: :doc:`data`
.. |next| replace:: :doc:`data_select`

.. include:: tutorial_nav_include.rst

.. rst-class:: core-header, orm-addin

.. _tutorial_core_insert:

使用 INSERT 语句
------------------

在使用 Core 同时使用 ORM 进行批量操作时，可以直接使用 :func:`_sql.insert` 函数生成 SQL INSERT 语句。这个函数会生成一个 :class:`_sql.Insert` 的实例，表示在 SQL 中将新数据添加到表中的 INSERT 语句。

.. container:: orm-header

    **ORM读者注意** -

    本节详细介绍了 Core 生成单个 SQL INSERT 语句的方法，以添加新行到表中。当使用 ORM 时，我们通常使用顶在其上的另一个工具，称为 :term:`unit of work`，它会自动化地生成许多 INSERT 语句。

    但即使 ORM 为我们运行它，了解 Core 如何处理数据创建和操作也非常有用。此外，ORM 支持使用称为 :ref:`tutorial_orm_bulk` 的功能直接使用 INSERT。

    要直接跳转到使用通常的单位操作模式使用 ORM 插入行，请参见 :ref:`tutorial_inserting_orm`。

insert() SQL 表达式构造器
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:class:`_sql.Insert` 的简单示例展示了目标表和 VALUES 子句::

    >>> from sqlalchemy import insert
    >>> stmt = insert(user_table).values(name="spongebob", fullname="Spongebob Squarepants")

上述 ``stmt`` 变量是 :class:`_sql.Insert` 的一个实例。大多数 SQL 表达式可以在其地方字符串化，以查看正在生成的对象的一般形式::

    >>> print(stmt)
    {printsql}INSERT INTO user_account (name, fullname) VALUES (:name, :fullname)

字符串化形式是通过生成该对象的 :class:`_engine.Compiled` 形式创建的，该形式包括语句的特定于数据库的字符串 SQL 表示形式；我们可以使用 :meth:`_sql.ClauseElement.compile` 方法直接获取该对象::

    >>> compiled = stmt.compile()

我们的 :class:`_sql.Insert` 构造是带有参数的构造，如 :ref:`tutorial_sending_parameters` 中所示；绑定的 ``name`` 和 ``fullname`` 参数也可以在 :class:`_engine.Compiled` 构造中使用::

    >>> compiled.params
    {'name': 'spongebob', 'fullname': 'Spongebob Squarepants'}


执行语句
^^^^^^^^^^^^^

我们可以使用执行语句将行插入到 ``user_table``。INSERT SQL 和绑定的参数可以在 SQL 日志中看到：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     conn.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] ('spongebob', 'Spongebob Squarepants')
    COMMIT

在上面的简单形式中，INSERT 语句不返回任何行，如果仅插入一行，则通常会包含返回有关插入该行时生成的列级默认值（最常见的是整数主键值）的信息。在上面的情况下，SQLite 数据库中的第一行通常为第一个整数主键值返回 ``1``，我们可以使用 :attr:`_engine.CursorResult.inserted_primary_key` 访问器获取：

.. sourcecode:: pycon+sql

    >>> result.inserted_primary_key
    (1,)

.. tip:: :attr:`_engine.CursorResult.inserted_primary_key` 返回元组，因为主键可能包含多列。这称为 :term:`composite primary key`。:attr:`_engine.CursorResult.inserted_primary_key` 有意包含刚刚插入的记录的完整主键，而不仅是 “cursor.lastrowid” 这样的值，也旨在在使用了 “autoincrement” 的情况下填充，因此表示完整主键的元组。

.. versionchanged:: 1.4.8， :attr:`_engine.CursorResult.inserted_primary_key` 返回的元组现在是由以特定名称返回 :class:`_result.Row` 对象履行的命名元组。

.. _tutorial_core_insert_values_clause:

INSERT 通常会自动生成 "values" 子句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

上面的示例使用了 :meth:`_sql.Insert.values` 方法来显式创建 SQL INSERT 语句的 VALUES 子句。如果我们没有使用 :meth:`_sql.Insert.values` 仅打印出 “空” 语句，我们将获得一个将表中每列插入一个 INSERT::

    >>> print(insert(user_table))
    {printsql}INSERT INTO user_account (id, name, fullname) VALUES (:id, :name, :fullname)

如果获取没有调用 :meth:`_sql.Insert.values` 的 :class:`_sql.Insert` 构造，并执行它而不打印它，则语句将基于传递给 :meth:`_engine.Connection.execute` 方法的参数编译为字符串，并仅包括与传递的参数相关的列。这实际上是插入行的一般方式，而无需输入明确的 VALUES 子句。下面的示例展示了使用列表参数同时执行两列 INSERT 语句的示例：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         insert(user_table),
    ...         [
    ...             {"name": "sandy", "fullname": "Sandy Cheeks"},
    ...             {"name": "patrick", "fullname": "Patrick Star"},
    ...         ],
    ...     )
    ...     conn.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] [('sandy', 'Sandy Cheeks'), ('patrick', 'Patrick Star')]
    COMMIT{stop}

以上执行特征在 :ref:`tutorial_multiple_parameters` 中首次展示了“executemany”形式，但与使用 :func:`_sql.text` 构造不同，我们不必拼写任何 SQL。通过将字典或字典列表与 :class:`_sql.Insert` 构造一起传递给 :meth:`_engine.Connection.execute` 方法，:class:`_engine.Connection` 确保传递的列名将自动在 :class:`_sql.Insert` 构造的 VALUES 子句中表示。

.. deepalchemy::

    欢迎来到 **Deep Alchemy** 的第一版。左边的人被称为 **炼金术士**，你会注意到他们 **不是** 巫师，因为尖头的帽子没有竖起来。炼金术士来描述通常**更高级和/或棘手**的事情，此外**通常不需要**，但由于某种原因，他们觉得您应该了解SQLAlchemy可以做的这一点。 

    在此版本中，为了在 ``address_table`` 中有一些有趣的数据，下面是一个更高级的示例，说明如何同时显式使用 :meth:`_sql.Insert.values` 方法，同时从参数生成额外的 `VALUES`。构建了一个标量子查询，使用下一个章节中介绍的 :func:`_sql.select` 构造，使用 :func:`_sql.bindparam` 构建了一个明确的绑定参数名称来设置子查询中使用的参数。

    这是一个有点**深入**的炼金术，以便我们可以在将相关行添加到应用程序时，无需从 ``user_table`` 操作中提取主键标识符。大多数炼金术师将简单地使用 ORM，ORM 为此类事情提供了帮助。

    .. sourcecode:: pycon+sql

        >>> from sqlalchemy import select, bindparam
        >>> scalar_subq = (
        ...     select(user_table.c.id)
        ...     .where(user_table.c.name == bindparam("username"))
        ...     .scalar_subquery()
        ... )

        >>> with engine.connect() as conn:
        ...     result = conn.execute(
        ...         insert(address_table).values(user_id=scalar_subq),
        ...         [
        ...             {
        ...                 "username": "spongebob",
        ...                 "email_address": "spongebob@sqlalchemy.org",
        ...             },
        ...             {"username": "sandy", "email_address": "sandy@sqlalchemy.org"},
        ...             {"username": "sandy", "email_address": "sandy@squirrelpower.org"},
        ...         ],
        ...     )
        ...     conn.commit()
        {execsql}BEGIN (implicit)
        INSERT INTO address (user_id, email_address) VALUES ((SELECT user_account.id
        FROM user_account
        WHERE user_account.name = ?), ?)
        [...] [('spongebob', 'spongebob@sqlalchemy.org'), ('sandy', 'sandy@sqlalchemy.org'),
        ('sandy', 'sandy@squirrelpower.org')]
        COMMIT{stop}

    经过这样的处理，我们在表中有了一些更有趣的数据，我们将在下一节中使用它们。

.. tip:: 如果要将“默认”仅插入到表中而不包括任何明确值的“真正”空 INSERT 语句，可以指示不带参数调用 :meth:`_sql.Insert.values`。并不是每个后端数据库都支持这样做，但这是 SQLite 产生的：

    >>> print(insert(user_table).values().compile(engine))
    {printsql}INSERT INTO user_account DEFAULT VALUES


.. _tutorial_insert_returning:

INSERT...RETURNING
^^^^^^^^^^^^^^^^^^^^^

对于支持的后端，RETURNING 子句自动用于检索最后插入的主键值以及服务器默认值。但是，RETURNING 子句也可以使用 :meth:`_sql.Insert.returning` 方法显式指定；在这种情况下，执行语句时返回的 :class:`_engine.Result` 对象有可获取的行：

    >>> insert_stmt = insert(address_table).returning(
    ...     address_table.c.id, address_table.c.email_address
    ... )
    >>> print(insert_stmt)
    {printsql}INSERT INTO address (id, user_id, email_address)
    VALUES (:id, :user_id, :email_address)
    RETURNING address.id, address.email_address

它也可以与 :meth:`_sql.Insert.from_select` 配合使用，在以下示例中构建自 :ref:`tutorial_insert_from_select` 开始的示例：

    >>> select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
    >>> insert_stmt = insert(address_table).from_select(
    ...     ["user_id", "email_address"], select_stmt
    ... )
    >>> print(insert_stmt.returning(address_table.c.id, address_table.c.email_address))
    {printsql}INSERT INTO address (user_id, email_address)
    SELECT user_account.id, user_account.name || :name_1 AS anon_1
    FROM user_account RETURNING address.id, address.email_address

.. tip::

    RETURNING 功能也支持 UPDATE 和 DELETE 语句，稍后将在本教程中介绍。

    对于 INSERT 语句，RETURNING 功能既可以用于插入单行语句，也可以用于同时插入多行的语句。RETURNING 支持多行 INSERT 的适用于特定的方言，但对于 SQLAlchemy 中支持 RETURNING 的所有方言都是支持的。有关此功能的背景，请参见 :ref:`engine_insertmanyvalues` 部分。

.. seealso::

    ORM 也支持带有或不带有 RETURNING 的批量 INSERT。请参见 :ref:`orm_queryguide_bulk_insert` 获取参考文档。



.. _tutorial_insert_from_select:

INSERT...FROM SELECT
^^^^^^^^^^^^^^^^^^^^^

:class:`_sql.Insert` 构造器的一个不常用的功能，但作为完整性而存在，是 :class:`_sql.Insert` 构造器可以使用 :meth:`_sql.Insert.from_select` 方法直接从 SELECT 中获取行，该方法接受 :func:`_sql.select` 构造器，该构造器将在下一节中讨论，以及在实际 INSERT 中将纳入目标的列名列表。下面的示例向 ``address`` 表添加行，这些行是从 ``user_account`` 表中派生的行，为每个用户提供免费的@aol.com 邮箱地址：

    >>> select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
    >>> insert_stmt = insert(address_table).from_select(
    ...     ["user_id", "email_address"], select_stmt
    ... )
    >>> print(insert_stmt)
    {printsql}INSERT INTO address (user_id, email_address)
    SELECT user_account.id, user_account.name || :name_1 AS anon_1
    FROM user_account

当希望直接将数据从数据库的其他部分复制到新的行集中时，而无需从客户端获取和重新发送数据时，可以使用此构造。


.. seealso::

    :class:`_sql.Insert` - 在 SQL 表达式 api 文档中。