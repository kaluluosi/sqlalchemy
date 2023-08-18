.. highlight:: pycon+sql

.. |prev| replace::  :doc:`data` 
.. |next| replace::  :doc:`data_select` 

.. include:: tutorial_nav_include.rst

.. rst-class:: core-header, orm-addin

.. _tutorial_core_insert:

使用 INSERT 语句
-----------------------

在通常使用 ORM 进行批量操作时，使用 Core 也需要直接生成 SQL INSERT 语句，这个过程使用 `SQLAlchemy` 的   :func:`_sql.insert`  方法生成一个新的：class:` _sql.Insert` 类的实例，该实例表示包含添加到表中的新数据的SQL语句。

.. container:: orm-header

    **ORM Readers** -

    本节将详细介绍生成单个SQL INSERT语句的Core方法，以便向表中添加新行。当使用ORM时，通常使用另一个名为  :term:`unit of work`  的工具，在这基础上可以自动化生产多个 INSERT 语句。即使ORM在运行时为我们处理SQL语句，了解Core如何处理数据的创建和操作也非常有用。此外，ORM 还支持使用称为   :ref:` tutorial_orm_bulk`  的功能直接使用 INSERT。

    要跳过如何使用常规工作单元模式使用ORM插入行，请参见   :ref:`tutorial_inserting_orm` 。

SQL 表达式构造的 insert() 方法
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_sql.Insert`  的一个简单示例，同时演示了目标表和 VALUES 子句::

    >>> from sqlalchemy import insert
    >>> stmt = insert(user_table).values(name="spongebob", fullname="Spongebob Squarepants")

上述 ``stmt`` 变量是   :class:`_sql.Insert`  的实例。 大多数 SQL 表达式都可以在代码中直接打印出来，以查看将要生成的语法结构

    >>> print(stmt)
    {printsql}INSERT INTO user_account (name, fullname) VALUES (:name, :fullname)
    
上面的字符串形式是通过生成   :class:`_engine.Compiled`  对象来创建的，该对象包含 SQL 语句的具体数据库特定字符串 SQL 表示形式；我们可以直接使用  :meth:` _sql.ClauseElement.compile`  方法获取这个对象。

    >>> compiled = stmt.compile()

我们可以从   :class:`_engine.Compiled`  中获取 bound parameter（绑定参数）的 ` `name`` 和 ``fullname`` 属性。

    >>> compiled.params
    {'name': 'spongebob', 'fullname': 'Spongebob Squarepants'}
    
执行 SQL 语句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

调用语句时，可以将数据插入到 ``user_table`` 中。可以在 SQL 记录中查看 INSERT SQL 以及捆绑的参数。

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     conn.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?)
    [...] ('spongebob', 'Spongebob Squarepants')
    COMMIT


在上面的简单形式中，INSERT 语句不返回任何行，如果只插入了一行，它通常包含返回有关在 INSERT 行期间生成的列级默认值的信息，最常见的是整数主键值。在上面的情况下，SQLite 数据库中的第一行通常将为第一个整数主键值返回 ``1``。我们可以使用  :attr:`_engine.CursorResult.inserted_primary_key`  访问器获取它。

.. sourcecode:: pycon+sql

    >>> result.inserted_primary_key
    (1,)

.. tip::  :attr:`_engine.CursorResult.inserted_primary_key`  返回一个元组，因为主键可能包含多列。这称为  :term:` composite primary key` 。  :attr:`_engine.CursorResult.inserted_primary_key`  旨在始终包含刚插入的记录的完整主键，而不仅仅是 “cursor.lastrowid” 类型的值，并且旨在在使用“自动递增”与否时填充该值，因此为了表示完整的主键，它是一个元组。

.. versionchanged:: 1.4.8 由  :attr:`_engine.CursorResult.inserted_primary_key`  返回的元组现在是一个命名元组，通过将其作为  :class:` _result.Row`  对象返回来满足。

.. _tutorial_core_insert_values_clause:

INSERT 通常会自动生成“值”子句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

上面的示例使用  :meth:`_sql.Insert.values`  方法显式创建了SQL INSERT语句的 VALUES 子句，如果我们不使用  :meth:` _sql.Insert.values` ，而只是打印出一个“空”语句，我们将获得 INSERT，这个 INSERT 会插入表中每个列的值。

    >>> print(insert(user_table))
    {printsql}INSERT INTO user_account (id, name, fullname) VALUES (:id, :name, :fullname)

如果我们拿到了   :class:`_sql.Insert`  构造器，却没有调用  :meth:` _sql.Insert.values`  方法，并执行 INSERT 语句，生成的语句将被编译为基于表定义的一条 SQL。在  :meth:`_engine.Connection.execute`   方法中传递的参数上，只包括与传递的参数相关的列。实际上，这是使用   :class:` _sql.Insert`  插入行的通常方式，而不必键入明确的 VALUES 子句。下面的示例说明了使用列表一次性执行具有两列的 INSERT 语句：

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

上面的执行首次展示了 "executemany" 形式，但与使用   :func:`_sql.text`  构造函数时不同，我们不必拼写任何 SQL 。通过在  :meth:` _engine.Connection.execute`  方法中与   :class:`_sql.Insert`  构造函数一起传递字典或字典列表，  :class:` _engine.Connection`  确保传递的列名将自动在   :class:`_sql.Insert`  构造函数的 VALUES 子句中表达。

.. deepalchemy::

    你好，欢迎阅读 **Deep Alchemy** 的第一版。左边的人被称为 **炼金术士**，你会注意到他们**不是**巫师，因为尖顶帽没有朝上伸。炼金术士会描述一些通常**更高级和/或棘手**的事情，并且通常**不需要**，但由于某种原因，他们觉得你应该了解 SQLAlchemy 可以做的这个东西。

    为了让``address_table``中有一些有趣的数据，下面是一个更高级的示例，说明如何在显式使用  :meth:`_sql.Insert.values`  方法的同时，生成来自参数的额外 VALUES。构建了一个  :term:` 标量子查询` ，利用了下一节介绍的   :func:`_sql.select`  构造函数，子查询中使用的参数使用   :func:` _sql.bindparam`  构造函数建立了一个显式的绑定参数名。

    这是一些稍微 **深入** 的炼金术，只是为了在不将主键标识符从``user_table``操作获取到应用程序时添加相关的行。大多数炼金术师将简单地使用 ORM 为我们处理这样的事情。

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

    通过以上方式，我们在表中有一些更有趣的数据，我们将在后面的章节中利用它们。

.. tip:: 如果我们指示  :meth:`_sql.Insert.values`  不带参数，则会生成真正的仅插入表中 "默认值" 而不包括任何明确值的 INSERT；并非所有的数据库后端都支持这个功能，但以下是 SQLite 生成的结果：

    >>> print(insert(user_table).values().compile(engine))
    {printsql}INSERT INTO user_account DEFAULT VALUES


.. _tutorial_insert_returning:

INSERT...RETURNING
^^^^^^^^^^^^^^^^^^^^^

为了检索最后插入的主键值及服务器默认值的值，在支持的后端上自动使用 RETURNING 子句。但是 RETURNING 子句也可以使用  :meth:`_sql.Insert.returning`  方法明确指定；在这种情况下，执行该语句时返回的   :class:` _engine.Result`  对象具有可抓取的行：

    >>> insert_stmt = insert(address_table).returning(
    ...     address_table.c.id, address_table.c.email_address
    ... )
    >>> print(insert_stmt).. _tutorial_insert_returning:

使用 RETURNING 检查插入结果
------------------------

在许多情况下，添加单个行只需要像例子中的样子制作唯一   :class:`_sql.Insert`  对象，这个   :class:` _sql.Insert`  可以在执行它的时候带回新的 ID :

.. sourcecode:: python

    from sqlalchemy import insert

    stmt = insert(user_table).values(name='spongebob')
    result = conn.execute(stmt)

    print(result.inserted_primary_key)


当支持 RETURNING 功能的后端的话，SQLAlchemy 还支持直接在 INSERT 语句中获取已经插入的行的某些列的值，例如 psycopg2 支持 RETURNING , 那么你可以很容易就从一个 INSERT 中|fetched-column|  :term:`fetched-columns`  中获取它，以这个例子来说我们将列 id 和 email_address 从 address 表中返回：

.. sourcecode:: python

    from sqlalchemy import insert

    result = conn.execute(
        insert(address_table).
        values(id=1, user_id=1, email_address='foo@bar.com').
        returning(address_table.c.id, address_table.c.email_address)
    )

    new_id, email_address = result.fetchone()

返回值是一个   :class:`~sqlalchemy.engine.ResultProxy` , 包含了由返回操作返回的行的所有元素。 最后，你也可以检索从 INSERT 包含多行时返回的多个行，只需传入一个元素列表即可：

.. sourcecode:: python

    result = conn.execute(
        insert(address_table).
        values([
            {"user_id": 1, "email_address": "foo@bar.com"},
            {"user_id": 1, "email_address": "bar@foo.com"},
        ]).
        returning(address_table.c.id, address_table.c.email_address)
    )

    new_rows = result.fetchall()

如前所述，每个支持 RETURNING 的后端都有一些限制或在如何使用它方面的沟通问题。这个应用程序是 PostgreSQL-specifc的，因为它是最好支持 RETURNING 功能的数据库之一。

.. admonition:: tip

    UPDATE 和 DELETE 语句也支持 RETURNING 功能，它们将在教程后面介绍。

    对于单行和多行 INSERT 语句，都可以使用 RETURNING 功能。 目前支持 RETURNING 功能的 SQLAlchemy 的所有方言都支持多行 INSERT 。 有关此功能的背景，请参见    :ref:`engine_insertmanyvalues`  章节。

.. seealso::

    ORM 支持带或不带返回值的大量的 INSERT 操作。 请参阅   :ref:`orm_queryguide_bulk_insert`  以获取参考文档。


.. _tutorial_insert_from_select:

插入...从选择操作
----------------------------

  :class:`_sql.Insert`  构造函数的一个不太常用的特性，是可以使 INSERT 通过  :meth:` _sql.Insert.from_select`  方法直接从 SELECT 获取行。
该方法接受一个   :func:`_sql.select`  构造函数，该构造函数在下一节中将进行讨论，以及要在实际 INSERT 中进行定位的列名称列表。 在下面的示例中，从 ` `user_account`` 表中派生的行被添加到 ``address`` 表中，每个用户都在 ``aol.com`` 上得到了一个免费的电子邮件地址：

.. code-block:: python

    select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
    insert_stmt = insert(address_table).from_select(
        ["user_id", "email_address"], select_stmt
    )

    print(insert_stmt)

下面是此构造函数的另一个例子，使用  :meth:`_sql.Insert.from_select`  结合   :ref:` tutorial_select`  中的   :func:`_sql.select`  函数使用：

.. code-block:: python

    select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
    insert_stmt = insert(address_table).from_select(
        ["user_id", "email_address"], select_stmt
    )

    print(insert_stmt)

当您想要直接将数据库中的某些数据复制到新的行集时，而无需从客户端获取并重新发送数据时，请使用此构造函数。


.. seealso::

      :class:`_sql.Insert`  - SQL Expression API 文档。