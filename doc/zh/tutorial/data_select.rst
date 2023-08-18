.. highlight:: pycon+sql

.. |prev| replace:: :doc:`data_insert`
.. |next| replace:: :doc:`data_update`

.. include:: tutorial_nav_include.rst

.. _tutorial_selecting_data:

.. rst-class:: core-header, orm-dependency

使用 SELECT 语句
------------------

在 Core 和 ORM 中，:func:`_sql.select` 函数可以生成一个 :class:`_sql.Select` 构造，用于所有的 SELECT 查询。将 SELECT 语句传递给 Core 中的方法，例如 :meth:`_engine.Connection.execute` 或者 ORM 中的方法 :meth:`_orm.Session.execute`，一个 SELECT 语句将被发出并且执行的结果可通过返回的 :class:`_engine.Result` 对象访问结果行。

.. container:: orm-header

    **ORM 读者** - 这里的内容同样适用于 Core 和 ORM 的基本使用方法，但是也可以使用许多更多面向 ORM 的特性；这些内容在 :ref:`queryguide_toplevel` 标题下进行了详细的文档介绍。


select() SQL 表达式构造
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_sql.select` 构造函数像其它 SQL 语句构造函数一样，使用增量的方式生成一条查询语句。query_clause = select()相当于query_clause = select().select_from(None)
SQLAlchemy提供了 select() 的语法糖 select_from()，表示select的from部分,explain select if_exists(select_from(user_table).where(and_(col1==xxx),))
::func:`_sql.select` 接收的位置参数表示多个 class:`_schema.Column` 和 `table-like` 的表达式，也接受一些兼容对象，这些对象被解析为要从 ""SELECT" 这个列才可以) 通过展示is_in_t1这一列的输出结果，此时你的我只能获得表达式最终的布尔值; 如果要查看两张表某个列的相等与不等，需要将这一表达式放在where的一侧，其他表达式放在另一侧。
将列和类似表达式用于简单的情况，可以创建 :func:`_sql.select` 构造中的 FROM 子句；它被解释为通过查询中提供的列和类似表达式来构建。::

    >>> from sqlalchemy import select
    >>> stmt = select(user_table).where(user_table.c.name == "spongebob")
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account WHERE user_account.name = :name_1

与其它语句级 SQL 构造一样，要执行该语句，我们将它传递到执行方法中。由于 SELECT 语句返回行，因此我们始终可以迭代结果对象以返回名为 :class:`_engine.Row` 的对象。::

    >>> with engine.connect() as conn:
    ...     for row in conn.execute(stmt):
    ...         print(row)
    {execsql}BEGIN(implicitly)
           SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account
           WHERE user_account.name = ?
           [...] ('spongebob',){stop}
    (1, 'spongebob', 'Spongebob Squarepants')
    {execsql}ROLLBACK{stop}

使用 ORM 时，特别是使用 :func:`_sql.select` 构造针对 ORM 实体的情况下，我们将使用 :meth:`_orm.Session.execute` 方法在 :class:`_orm.Session`上执行该语句；使用此方法，我们继续从结果中获取 :class:`_engine.Row` 对象，但是这些行现在可以将完整的实体（例如“User”类的实例）作为每行中的单独元素来包括。::

    >>> stmt = select(User).where(User.name == "spongebob")
    >>> with Session(engine) as session:
    ...     for row in session.execute(stmt):
    ...         print(row)
    {execsql}BEGIN(implicitly)
           SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account
           WHERE user_account.name = ?
           [...] ('spongebob',){stop}
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
    {execsql}ROLLBACK{stop}

.. topic:: 从表中选择与从 ORM 类中选择

    虽然这些示例中生成的 SQL 看起来在索引了 ``select(user_table)`` 或 ``select(User)`` 之后是一样的，但在一般情况下，它们并不一定会生成相同的语句，因为 ORM 映射的类可能还会映射到表以外的其他“selectables”，同时转到一个 ORM 实体上的 select() 还表明可能会作为结果返回 ORM 映射的实例，当从 :class:`_schema.Table` 对象中选择时则不会返回此结果。

接下来的部分将更详细地讨论 SELECT 构造。


.. _tutorial_selecting_columns:

设置 COLUMNS 和 FROM 子句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_sql.select` 函数接受位置元素，表示任何数量的 :class:`_schema.Column` 和/或 `table-like` 表达式，以及许多兼容对象，它们被解析成要作为结果集中的列返回的 SQL 表达式列表。这些元素在较简单的情况下还用作创建 FROM 子句，该子句根据传递的列和表类似表达式进行推断：::

    >>> print(select(user_table))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account

使用 Core 方法从单个列中选择列时，可以直接从 :attr:`_schema.Table.c` 访问 :class:`_schema.Column` 对象，并且可以直接发送列。 FROM 子句将推断为表示它们的所有 :class:`_schema.Table` 和其他 :class:`_sql.FromClause` 对象的集合。::

    >>> print(select(user_table.c.name, user_table.c.fullname))
    {printsql}SELECT user_account.name, user_account.fullname
           FROM user_account

或者，当使用 :attr:`.FromClause.c` 集合时，可以通过使用字符串名称的元组选择 :func:`_sql.select` 中的多个列。::- 2.0 添加元组访问器功能到 :attr:`.FromClause.c` 集合

    >>> print(select(user_table.c["name", "fullname"]))
    {printsql}SELECT user_account.name, user_account.fullname
           FROM user_account


.. _tutorial_selecting_orm_entities:

选择 ORM 实体和列
~~~~~~~~~~~~~~~~~~~~~~~~~

ORM 实体，如我们的 ``User`` 类以及其中映射的列，例如 ``User.name``，还参与表示表和列的 SQL 表达式语言系统中。下面举例说明从 ``User`` 实体中 SELECT，最终呈现的方式与我们直接使用 ``user_table`` 是相同的：::

    >>> print(select(User))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account

当我们使用 ORM :meth:`_orm.Session.execute` 方法调用上述语句时，如果选择整个实体，例如 ``User``，则有一个重要区别，因为 **每行中实际上都返回实体本身**。也就是说，当我们从上述语句中提取行时，由于只有 ``User`` 实体在要提取的结果中，我们获取到的是仅有一个元素的 :class:`_engine.Row` 对象，其中包含 ``User`` 类的实例：::

    >>> row = session.execute(select(User)).first()
    {execsql}BEGIN(implicitly)
           SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account
           WHERE user_account.name = ?
           [...] ('spongebob',){stop}
    >>> row
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)

上述 :class:`_engine.Row` 有一个元素，表示 ``User`` 实体：::

    >>> row[0]
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')

一个高度推荐的方式是使用 :meth:`_orm.Session.scalars` 方法直接执行查询，这种方法将返回一个 :class:`_result.ScalarResult` 对象，可以一次性得到每行的第一个 "列"，在这种情况下，是 ``User`` 类的实例：::

    >>> user = session.scalars(select(User)).first()
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account
           WHERE user_account.name = ?
           [...] ('spongebob',){stop}
    >>> user
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')

或者，我们可以将 ORM 实体的单个列选择为结果行中的不同元素，方法是使用类绑定的属性；当它们被传递给诸如 :func:`_sql.select` 这样的结构时，它们被解析为每个属性所代表的 :class:`_schema.Column` 或其他 SQL 表达式：::

    >>> print(select(User.name, User.fullname))
    {printsql}SELECT user_account.name, user_account.fullname
           FROM user_account

当我们使用 :meth:`_orm.Session.execute` 调用 *此* 语句时，我们接收到指向一个元素的行，每个元素都对应于单个列或其他 SQL 表达式。如下所示，每个值都对应于单个列或其他 SQL 表达式。  :class:`_engine.Row`：：

    >>> row = session.execute(select(User.name, User.fullname)).first()
    {execsql}SELECT user_account.name, user_account.fullname
           FROM user_account
           WHERE user_account.name = ?
           [...] ('spongebob',){stop}
    >>> row
    ('spongebob', 'Spongebob Squarepants')

这些方法也可以混用，如下面的例子，我们将 ``User`` 实体的 ``name`` 属性作为行的第一个元素选择，然后与第二个元素中的完整的 ``Address`` 实体进行组合。::

    >>> session.execute(
    ...     select(User.name, Address).where(User.id == Address.user_id).order_by(Address.id)
    ... ).all()
    {execsql}SELECT user_account.name, address.id, address.email_address, address.user_id
           FROM user_account, address
           WHERE user_account.id = address.user_id ORDER BY address.id
           [...] (){stop}
    [('spongebob', Address(id=1, email_address='spongebob@sqlalchemy.org')),
    ('sandy', Address(id=2, email_address='sandy@sqlalchemy.org')),
    ('sandy', Address(id=3, email_address='sandy@squirrelpower.org'))]

在 :ref:`orm_queryguide_select_columns` 中可以进一步了解有关选择 ORM 实体和列以及将行转换为实体的常见方法。

.. seealso::

    :ref:`orm_queryguide_select_columns` - 在 :ref:`queryguide_toplevel` 中


.. _tutorial_selecting_arbitrary_text:

使用文本列表达式进行 Select
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当我们使用 :func:`_sql.select` 函数构造 :class:`_sql.Select` 对象时，通常会在查询语句中注入任意 SQL 代码块，例如常量字符串表达式或者直接通过字符串快速编写的其他 SQL 代码块。

:func:`_sql.text` 构造函数可以直接嵌入到 :class:`_sql.Select` 构造中，如下面的例子所示，在查询中制造硬编码字符串文字 "some phrase"：::

  >>> from sqlalchemy import text
  >>> stmt = select(
  ...     text("'some phrase'"), user_table.c.name).order_by(user_table.c.name)
  >>> with engine.connect() as conn:
  ...     print(conn.execute(stmt).all())
  {execsql}BEGIN(implicitly)
           SELECT 'some phrase', user_account.name
           FROM user_account ORDER BY user_account.name
           [...] (){stop}[('some phrase', 'patrick'), ('some phrase', 'sandy'), ('some phrase', 'spongebob')]
  {execsql}ROLLBACK{stop}

虽然 :func:`_sql.text` 构造函数可以在大多数地方注入文本 SQL 短语，但在此并非总是必要的。通常我们要处理的是表示单个“列”的文本单元，使用 :func:`_sql.literal_column` 可以获得更多的功能，该对象类似于 :func:`_sql.text`，不同的是它显式表示一个“单列”，然后可以被赋予标签，并且可以称为子查询和其他表达式中引用的任意名称::


  >>> from sqlalchemy import literal_column
  >>> stmt = select(literal_column("'some phrase'").label("p"), user_table.c.name).order_by(
  ...     user_table.c.name
  ... )
  >>> with engine.connect() as conn:
  ...     for row in conn.execute(stmt):
  ...         print(f"{row.p}, {row.name}")
  {execsql}BEGIN(implicitly)
           SELECT 'some phrase' AS p, user_account.name
           FROM user_account ORDER BY user_account.name
           [...] (){stop}
  some phrase, patrick
  some phrase, sandy
  some phrase, spongebob
  {execsql}ROLLBACK{stop}


请注意，在使用 :func:`_sql.text` 或 :func:`_sql.literal_column` 时，我们编写的是一种经过语法处理的 SQL 表达式，而不是字面值。我们还必须包含所需的任何引号或语法等，这是我们要查看的视图要呈现的 SQL。


.. _tutorial_select_where_clause:

WHERE子句
^^^^^^^^^^^^^^^^

使用标准 Python 操作符，结合 :class:`_schema.Column` 和类似的对象，SQLAlchemy 允许我们组合 SQL 表达式，例如 ``name = 'squidward'`` 或 ``user_id > 10``。对于布尔表达式，类似于 ``==`` 、``!=``、``<``、``>=`` 等大多数 Python 运算符在涉及到原表达式时都会生成新的 SQL 表达式对象，而不是生成纯布尔值 `True` 或 `False`：。

   >>> print(user_table.c.name == "squidward")
   user_account.name = :name_1

   >>> print(address_table.c.user_id > 10)
   address.user_id > :user_id_1


我们可以将这些表达式用于 WHERE 子句，方法是将结果对象传递到 :meth:`_sql.Select.where` 方法中：：

    >>> print(select(user_table).where(user_table.c.name == "squidward"))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account WHERE user_account.name = :name_1


要生成由 AND 连接起来的多个表达式，请多次调用 :meth:`_sql.Select.where` 方法:

   >>> print(
   ...     select(address_table.c.email_address).where(
   ...         user_table.c.name == "squidward",
   ...         address_table.c.user_id == user_table.c.id,
   ...     )
   ... )
   {printsql}SELECT address.email_address
       FROM user_account, address
       WHERE user_account.name = :name_1 AND address.user_id = user_account.id

一次调用 :meth:`_sql.Select.where` 方法还可以直接接受多个表达式，效果相同：

    >>> print(
    ...     select(address_table.c.email_address).where(
    ...         user_table.c.name == "squidward",
    ...         address_table.c.user_id == user_table.c.id,
    ...     )
    ... )
    {printsql}SELECT address.email_address
           FROM user_account, address
           WHERE user_account.name = :name_1 AND address.user_id = user_account.id


使用 :func:`_sql.and_` 和 :func:`_sql.or_` 函数，可以在 FROM 子句或最后一个实体后面直接使用 "AND" 和 "OR" 连接：：

    >>> from sqlalchemy import and_, or_
    >>> print(
    ...     select(Address.email_address).where(
    ...         and_(
    ...             or_(User.name == "squidward", User.name == "sandy"),
    ...             Address.user_id == User.id,
    ...         )
    ...     )
    ... )
    {printsql}SELECT address.email_address
           FROM address, user_account
           WHERE (user_account.name = :name_1 OR user_account.name = :name_2)
           AND address.user_id = user_account.id

对于左侧和右侧没有外键约束，或存在多个外键约束的左右目标的情况，如果左侧和右侧目标没有这样的约束，则 SELECT 构造不会默认为 WHERE 子句生成 ON 子句，此时我们需要直接指定 ON 子句。:meth:`_sql.Select.join` 和 :meth:`_sql.Select.join_from` 方法都接受额外的 ON 子句参数，使用的方法与 :ref:`tutorial_selecting_columns` 部分介绍的 SQL 表达式机制相同。::

    >>> print(
    ...     select(address_table.c.email_address)
    ...     .select_from(user_table)
    ...     .join(address_table, user_table.c.id == address_table.c.user_id)
    ... )
    {printsql}SELECT address.email_address FROM user_account JOIN address ON user_account.id = address.user_id


.. sidebar:: ON 子句是被自动生成的

    当使用 :meth:`_sql.Select.join_from` 或 :meth:`_sql.Select.join` 时，我们会发现，在简单的外键情况下，自动为我们生成了 JOIN 的 ON 子句。下一部分将进一步介绍。

OUTER 和 FULL JOIN
~~~~~~~~~~~~~~~~~~~~

:meth:`_sql.Select.join` 和 :meth:`_sql.Select.join_from` 方法都接受关键字参数 :paramref:`_sql.Select.join.isouter` 和 :paramref:`_sql.Select.join.full`，从而将生成 LEFT OUTER JOIN 和 FULL OUTER JOIN，如下面所示:：

    >>> print(select(user_table).join(address_table, isouter=True))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account LEFT OUTER JOIN address ON user_account.id = address.user_id{stop}

    >>> print(select(user_table).join(address_table, full=True))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account FULL OUTER JOIN address ON user_account.id = address.user_id{stop}

还有一个方法 :meth:`_sql.Select.outerjoin` 等效于 "使用``.join(..., isouter=True) ``。


.. tip::

    SQL 还有一个“RIGHT OUTER JOIN”，SQLAlchemy 不直接生成此项；取而代之的，我们需要对表进行倒转并使用“LEFT OUTER JOIN”进行操作。


.. _tutorial_order_by_group_by_having:

ORDER BY、GROUP BY 和 HAVING
==============================

SELECT SQL 语句包括一个节点叫 ORDER BY，用于按给出顺序返回选择的行。

GROUP BY 子句类似于 ORDER BY 子句，在构造上类似，其目的是将所选行分成特定的组，从而可以对每个组分别应用聚合功能，如计数、计算平均值，以及查找一组值中的最大值或最小值。

HAVING 子句通常与 GROUP BY 子句一起使用，该子句类似于 WHERE 子句，除了它是应用于聚合函数，而不是应用于直接包含在行中的某个内容。

.. _tutorial_order_by:

ORDER BY
~~~~~~~~~

ORDER BY 子句基于 :class:`_schema.Column` 或类似对象构造。:meth:`_sql.Select.order_by` 方法接受一个或多个这些表达式表示的位置参数:：

    >>> print(select(user_table).order_by(user_table.c.name))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account ORDER BY user_account.name

升序/降序可以使用 :meth:`_sql.ColumnElement.asc` 和 :meth:`_sql.ColumnElement.desc` 修改符号来修改，ORM 相关属性中也存在这些元素：：

    >>> print(select(User).order_by(User.fullname.desc()))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
           FROM user_account ORDER BY user_account.fullname DESC

上述语句将生成按 ``user_account.fullname`` 列进行排序的行。


.. _tutorial_group_by_w_aggregates:

使用 GROUP BY / HAVING 进行聚合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 SQL 中，聚合函数允许将跨多行的列表达式聚合在一起，从而生成单个结果。示例包括计数、计算平均值，以及查找一组值中的最大值或最小值。

SQLAlchemy 使用一个名为 :data:`_sql.func` 的命名空间提供 SQL 函数，这是一个特殊的构造函数对象，当给出特定的 SQL 函数名称时，它会创建 :class:`_functions.Function` 的新实例，该实例可以有任何名称，以及零个或多个要传递给函数的参数，这些参数与所有其他情况一样，都是 SQL 表达式构造。例如，要根据 ``user_account.id`` 列渲染 SQL COUNT() 函数，我们调用 ``count()`` 名称：::

    >>> from sqlalchemy import func
    >>> count_fn = func.count(user_table.c.id)
    >>> print(count_fn)
    {printsql}count(user_account.id)

SQL 函数在本教程晚些时候的 :ref:`tutorial_functions` 一节将详细介绍。

在使用 SQL 聚合函数时，GROUP BY 子句非常重要，因为它允许行分成组，聚合函数将单独应用于每个组。 在选择SELECT语句的COLUMNS子句中请求非聚合列时，SQL 要求这些列全部都要受到 GROUP BY 子句的影响，直接或间接地基于主键关联。

可使用 :meth:`_sql.Select.group_by` 和 :meth:`_sql.Select.having` 方法提供这两个子句。 下面我们说明如何将用户名称字段与地址计数一起选择，以获得拥有多个地址的用户。:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         select(User.name, func.count(Address.id).label("count"))
    ...         .join(Address)
    ...         .group_by(User.name)
    ...         .having(func.count(Address.id) > 1)
    ...     )
    ...     print(result.all())
    {execsql}BEGIN IMPLICIT
           SELECT user_account.name, count(address.id) AS count
           FROM user_account JOIN address ON user_account.id = address.user_id GROUP BY user_account.name
           HAVING count(address.id) > ?
           [...] (1,){stop}
    [('sandy', 2)]
    {execsql}ROLLBACK{stop}

.. _tutorial_order_by_label:


按标签排序或分组
~~~~~~~~~~~~~~~~~

在某些数据库后端上，特别是在ORDER BY或GROUP BY一个已在列子句中已经存在的表达式的能力是一种重要的技术，而不必在ORDER BY或GROUP BY子句中重新声明该表达式，而是可以使用COLUMNS子句中的列名称或标签名称。可以通过将名称的字符串文本传递给:meth:`_sql.Select.order_by`或:meth:`_sql.Select.group_by`方法来使用此形式。传递的文本**不会直接呈现**；相反，在列子句中给出一个表达式的名称，并在上下文中呈现为该表达式名称，如果找不到匹配项，则引发错误。还可以使用一元修饰符:func:`.asc`和:func:`.desc`使用此形式：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func, desc
    >>> stmt = (
    ...     select(Address.user_id, func.count(Address.id).label("num_addresses"))
    ...     .group_by("user_id")
    ...     .order_by("user_id", desc("num_addresses"))
    ... )
    >>> print(stmt)
    {printsql}SELECT address.user_id, count(address.id) AS num_addresses
    FROM address GROUP BY address.user_id ORDER BY address.user_id, num_addresses DESC

.. _tutorial_using_aliases:

使用别名
^^^^^^^^^^^^^

既然我们正在从多个表中进行选择并使用连接，那么很快就会遇到需要在语句的FROM子句中多次引用同一张表格的情况。我们使用SQL别名来完成这一点，这是一种语法，为表或子查询提供另一种名称，以便可以在语句中引用它。

在SQLAlchemy表达式语言中，这些“名称”由称为:class:`_sql.Alias`的:class:`_sql.FromClause`对象表示，该对象是使用:meth:`_sql.FromClause.alias`方法在Core中构造的。:class:`_sql.Alias`构造类似于:class:`_sql.Table`构造，因为它也具有:class:`_schema.Column`对象的命名空间，该命名空间在:attr:`_sql.Alias.c`中。例如，下面的SELECT语句返回所有唯一的用户名对:: 

    >>> user_alias_1 = user_table.alias()
    >>> user_alias_2 = user_table.alias()
    >>> print(
    ...     select(user_alias_1.c.name, user_alias_2.c.name).join_from(
    ...         user_alias_1, user_alias_2, user_alias_1.c.id > user_alias_2.c.id
    ...     )
    ... )
    {printsql}SELECT user_account_1.name, user_account_2.name AS name_1
    FROM user_account AS user_account_1
    JOIN user_account AS user_account_2 ON user_account_1.id > user_account_2.id

.. _tutorial_orm_entity_aliases:

ORM实体别名
~~~~~~~~~~~~~~

:meth:`_sql.FromClause.alias`方法的ORM等效方法是ORM:func:`_orm.aliased`函数，它可以应用于实体，例如“User”和“Address”。这将在针对原始映射的:class:`_schema.Table`对象的:class:`_sql.Alias`对象内部产生一个对象，同时保持ORM功能。下面的SELECT从“User”实体中选择所有包含两个特定电子邮件地址的对象：

    >>> from sqlalchemy.orm import aliased
    >>> address_alias_1 = aliased(Address)
    >>> address_alias_2 = aliased(Address)
    >>> print(
    ...     select(User)
    ...     .join_from(User, address_alias_1)
    ...     .where(address_alias_1.email_address == "patrick@aol.com")
    ...     .join_from(User, address_alias_2)
    ...     .where(address_alias_2.email_address == "patrick@gmail.com")
    ... )
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN address AS address_1 ON user_account.id = address_1.user_id
    JOIN address AS address_2 ON user_account.id = address_2.user_id
    WHERE address_1.email_address = :email_address_1
    AND address_2.email_address = :email_address_2

.. tip::

    如前所述，在:ref:`tutorial_select_join_onclause`中，ORM提供了使用:func:`_orm.relationship`构造另一种连接方式。使用别名的上述示例在:ref:`tutorial_joining_relationships_aliased`中进行了演示。


.. _tutorial_subqueries_ctes:

子查询和CTE
~~~~~~~~~~~~~~

在SQL中，子查询是在括号内呈现并放置在包含语句（通常是SELECT语句，但不一定）的上下文中的SELECT语句。

本节将介绍所谓的“非标量”子查询，通常位于封闭的SELECT的FROM子句中。我们还将介绍Common Table Expression或CTE，其用法类似于子查询，但包括其他功能。

SQLAlchemy使用:class:`_sql.Subquery`对象表示子查询和:class:`_sql.CTE`表示CTE，通常使用:meth:`_sql.Select.subquery`和:meth:`_sql.Select.cte`方法获得这些对象。可以将任何对象用作FROM元素。更大的:func:`_sql.select`结构。

我们可以构造一个:class:`_sql.Subquery`，该子查询将从“address”表中选择一行的聚合计数（聚合函数和GROUP BY在:ref:`tutorial_group_by_w_aggregates`中介绍过）：

    >>> subq = (
    ...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
    ...     .group_by(address_table.c.user_id)
    ...     .subquery()
    ... )

如果自行将子查询的字符串化，则其默认字符串形式没有任何封闭括号，只会呈现为一个普通的SELECT语句::

    >>> print(subq)
    {printsql}SELECT count(address.id) AS count, address.user_id
    FROM address GROUP BY address.user_id


:class:`_sql.Subquery` 对象的行为类似于其他FROM对象，例如:class:`_schema.Table`，其中值得注意的是，它包括一组:attr:`_sql.Subquery.c`列用于选择。我们可以使用此命名空间引用“user_id”列以及自定义的标记为“count”的表达式::

    >>> print(select(subq.c.user_id, subq.c.count))
    {printsql}SELECT anon_1.user_id, anon_1.count
    FROM (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id) AS anon_1

拥有contained rows的子查询的选择，我们可以将对象应用于更大的:class:`_sql.Select`，该类将数据连接到“user_account”表：

    >>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
    ...     user_table, subq
    ... )

    >>> print(stmt)
    {printsql}SELECT user_account.name, user_account.fullname, anon_1.count
    FROM user_account JOIN (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id) AS anon_1 ON user_account.id = anon_1.user_id

为了从“user_account”连接到“address”，我们使用:meth:`_sql.Select.join_from`方法。正如先前示例所示，此连接的ON子句再次被**推断**，这是基于外键约束的。即使SQL子查询本身没有任何约束，SQLAlchemy也可以通过确定“subq.c.user_id”列**派生**自“address_table.c.user_id”列来作用于列上的约束，该列实际上表示外键关系返回到“user_table .*id”列，然后用于生成ON子句。

通用表达式（CTE）
~~~~~~~~~~~~~~~~~

使用:class:`_sql.CTE`构造在SQLAlchemy中的使用方式几乎与使用:class:`_sql.Subquery`构造相同。通过将调用:meth:`_sql.Select.subquery`方法更改为使用:meth:`_sql.Select.cte`，我们可以使用所得到的对象作为相同的方法中的FROM元素。但是，呈现的SQL是非常不同的常用表达式语法::

    >>> subq = (
    ...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
    ...     .group_by(address_table.c.user_id)
    ...     .cte()
    ... )

    >>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
    ...     user_table, subq
    ... )

    >>> print(stmt)
    {printsql}WITH anon_1 AS
    (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id)
     SELECT user_account.name, user_account.fullname, anon_1.count
    FROM user_account JOIN anon_1 ON user_account.id = anon_1.user_id

:class:`_sql.CTE`结构还提供了使用“递归”样式的能力，并且在更复杂的情况下可能由INSERT、UPDATE或DELETE语句的RETURNING子句组成。:class:`_sql.CTE`的 docstring中包含有关这些其他模式的详细信息。

在两种情况下，子查询和CTE都是用匿名名称在SQL级别命名的。在Python代码中，我们根本不需要提供这些名称。当呈现时，:class:`_sql.Subquery`或:class:`_sql.CTE`实例的对象标识作为对象的语法标识。可以通过将其作为:meth:`_sql.Select.subquery`或:meth:`_sql.Select.cte`方法的第一个参数传递来提供编译为SQL的名称。


.. seealso::

    :meth:`_sql.Select.subquery` - 有关子查询的更多详细信息

    :meth:`_sql.Select.cte` - 包括如何使用CTE的示例，以及如何使用RECURSIVE以及DML导向的CTE

.. _tutorial_subqueries_orm_aliased:

ORM实体子查询/CTE
~~~~~~~~~~~~~~~~~~~~

在ORM中，:func:`_orm.aliased`构造函数可用于将ORM实体，例如我们的“User”或“Address”类，与任何表示行源的:class:`_sql.FromClause`概念相关联。前面的一节:ref:`tutorial_orm_entity_aliases`说明了如何使用:func:`_orm.aliased`将映射类与其映射的:class:`_schema.Table`的:class:`_sql.Alias`相关联。在这里，我们将演示如何使用:func:`_orm.aliased`将其应用于对:class:`_sql.Subquery`的生成以及对:class:`_sql.CTE`使用的相同概念。

以下是将:func:`_orm.aliased`应用于:class:`_sql.Subquery`构造的示例，以便我们可以从其行中提取ORM实体。结果显示了一系列“User”和“Address”对象，其中每个“Address”对象的数据最终源自“address”表的子查询而不是该表本身：

.. sourcecode:: pycon+sql

    >>> subq = select(Address).where(~Address.email_address.like("%@aol.com")).subquery()
    >>> address_subq = aliased(Address, subq)
    >>> stmt = (
    ...     select(User, address_subq)
    ...     .join_from(User, address_subq)
    ...     .order_by(User.id, address_subq.id)
    ... )
    >>> with Session(engine) as session:
    ...     for user, address in session.execute(stmt):
    ...         print(f"{user} {address}")
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname,
    anon_1.id AS id_1, anon_1.email_address, anon_1.user_id
    FROM user_account JOIN
    (SELECT address.id AS id, address.email_address AS email_address, address.user_id AS user_id
    FROM address
    WHERE address.email_address NOT LIKE ?) AS anon_1 ON user_account.id = anon_1.user_id
    ORDER BY user_account.id, anon_1.id
    [...] ('%@aol.com',){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
    {execsql}ROLLBACK{stop}

接下来的另一个示例完全相同，只是使用:meth:`_sql.CTE`构造代替了:class:`_sql.Subquery`构造：

.. sourcecode:: pycon+sql

    >>> cte_obj = select(Address).where(~Address.email_address.like("%@aol.com")).cte()
    >>> address_cte = aliased(Address, cte_obj)
    >>> stmt = (
    ...     select(User, address_cte)
    ...     .join_from(User, address_cte)
    ...     .order_by(User.id, address_cte.id)
    ... )
    >>> with Session(engine) as session:
    ...     for user, address in session.execute(stmt):
    ...         print(f"{user} {address}")
    {execsql}BEGIN (implicit)
    WITH anon_1 AS
    (SELECT address.id AS id, address.email_address AS email_address, address.user_id AS user_id
    FROM address
    WHERE address.email_address NOT LIKE ?)
    SELECT user_account.id, user_account.name, user_account.fullname,
    anon_1.id AS id_1, anon_1.email_address, anon_1.user_id
    FROM user_account JOIN anon_1 ON user_account.id = anon_1.user_id
    ORDER BY user_account.id, anon_1.id
    [...] ('%@aol.com',){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
    User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
    {execsql}ROLLBACK{stop}

.. seealso::

    :ref:`orm_queryguide_subqueries` - 在 :ref:`queryguide_toplevel` 中


.. _tutorial_scalar_subquery:

标量和相关子查询
~~~~~~~~~~~~~~~~~~~~

标量子查询是返回正好零行或一行的子查询，以及正好一列。然后，在封闭的SELECT语句的COLUMNS或WHERE子句中使用该子查询，这与常规子查询不同，后者用于FROM子句中。ScalarSelect构造表示标量子查询，该标量子查询是:类:`_sql.ColumnElement`表达式层次结构中的一部分，与表示常规子查询的:class:`_sql.Subquery`构造不同，在:class:`_sql.FromClause`层次结构中表示。

通常情况下，如果SELECT语句引用“table1 JOIN（SELECT…）AS subquery”在其FROM子句中，则右侧的子查询可能不引用左侧的“table1”表达式；相关可以仅引用完全包含此SELECT的另一个SELECT的一张表。LATERAL关键字允许我们将这种行为反转并允许来自右侧JOIN的相关性。

SQLAlchemy使用:meth:`_expression.Select.lateral`方法支持该特性，该方法创建一个称为:class:`.Lateral`的对象。:class:`.Lateral`与:class:`.Subquery`和:class:`.Alias`在同一系列中，但在添加到含有母语的SELECT的FROM子句中时还包括相关行为，与:ref:`tutorial_expressions_lateral`中的例子类似，关联电子邮件字符串。

.. seealso::

    :class:`.Lateral`

    :meth:`_expression.Select.lateral`

    :class:`_or` 子句，例如SELECT UNION

    :func:`_sql.union_all`

    :func:`_sql.intersect`

    :func:`_sql.except_`

.. _tutorial_orm_union:

从联合选择中选择ORM实体
~~~~~~~~~~~~~~~~~~~~~~~~~~~

先前的示例说明了如何使用两个:class:`_schema.Table`对象构造UNION，然后返回数据库行。如果我们想使用UNION或其他集合操作来选择我们然后作为ORM对象接收的行，则可以使用两种方法。在这两种情况下，我们首先构造一个表示我们要执行的SELECT / UNION /等语句的:func:`_sql.select`或:class:`_sql.CompoundSelect`对象。应根据目标ORM实体或其基本映射的:class:`_schema.Table`对象组成此语句::

    >>> stmt1 = select(User).where(User.name == "sandy")
    >>> stmt2 = select(User).where(User.name == "spongebob")
    >>> u = union_all(stmt1, stmt2)

对于不包含在子查询中的简单SELECT和UNION，这些功能通常可以使用:meth:`_sql.Select.from_statement`方法在ORM对象获取上下文中使用。使用此方法，UNION语句表示整个查询；在使用:meth:`_sql.Select.from_statement`之后，无法添加任何其他标准：

    >>> orm_stmt = select(User).from_statement(u)
    >>> with Session(engine) as session:
    ...     for obj in session.execute(orm_stmt).scalars():
    ...         print(obj)
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ? UNION ALL SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [generated in ...] ('sandy', 'spongebob')
    {stop}User(id=2, name='sandy', fullname='Sandy Cheeks')
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    {execsql}ROLLBACK{stop}

要将UNION或其他集合相关构造用作与实体相关的组件​​的更灵活的方式，可以使用:meth:`_sql.CompoundSelect.subquery`将:class:`_sql.CompoundSelect`构造组织成子查询，然后使用:func:`_orm.aliased`函数将其链接到ORM对象。这与在:ref:`tutorial_subqueries_orm_aliased`中介绍的方式相同，首先为所需的实体创建一个临时的“映射”，然后选择该映射的新实体，就好像它是任何其他映射类一样。在下面的示例中，我们可以添加额外的条件，例如ORDER BY，因为我们可以过滤或按导出的列排序。

    >>> user_alias = aliased(User, u.subquery())
    >>> orm_stmt = select(user_alias).order_by(user_alias.id)
    >>> with Session(engine) as session:
    ...     for obj in session.execute(orm_stmt).scalars():
    ...         print(obj)
    {execsql}BEGIN (implicit)
    SELECT anon_1.id, anon_1.name, anon_1.fullname
    FROM (SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
    FROM user_account
    WHERE user_account.name = ? UNION ALL SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
    FROM user_account
    WHERE user_account.name = ?)
    AS anon_1 ORDER BY anon_1.id
    [generated in ...] ('sandy', 'spongebob')
    {stop}User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    {execsql}ROLLBACK{stop}

WHERE user_account.name = ? UNION ALL SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
FROM user_account
WHERE user_account.name = ?) AS anon_1 ORDER BY anon_1.id
[生成于...] ('sandy', 'spongebob')
{stop}User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
{execsql}ROLLBACK{stop}

.. 另请参阅::

    :ref:`orm_queryguide_unions` - 在 :ref:`queryguide_toplevel`

.. _tutorial_exists:

存在子查询
^^^^^^^^^^^^^^^^^^

SQL中的EXISTS关键字是一种与 :ref:`scalar subqueries<tutorial_scalar_subquery>`
一起使用的运算符，根据SELECT语句是否返回行返回一个布尔值真或假。
SQLAlchemy包括一种:class:`_sql.ScalarSelect`对象的变体，
称为:class:`_sql.Exists`，它将生成一个EXISTS子查询，
并且最方便地使用:meth:`_sql.SelectBase.exists`方法生成。
下面我们生成一个EXISTS，以便我们可以返回在“address”中具有多个相关行的“user_account”行：

.. sourcecode:: pycon+sql

    >>> subq = (
    ...     select(func.count(address_table.c.id))
    ...     .where(user_table.c.id == address_table.c.user_id)
    ...     .group_by(address_table.c.user_id)
    ...     .having(func.count(address_table.c.id) > 1)
    ... ).exists()
    >>> with engine.connect() as conn:
    ...     result = conn.execute(select(user_table.c.name).where(subq))
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    SELECT user_account.name
    FROM user_account
    WHERE EXISTS (SELECT count(address.id) AS count_1
    FROM address
    WHERE user_account.id = address.user_id GROUP BY address.user_id
    HAVING count(address.id) > ?)
    [...] (1,){stop}
    [('sandy',)]
    {execsql}ROLLBACK{stop}

在上面的代码中，我们使用了:meth:`_sql.SelectBase.exists`方法，生成了一个EXISTS子查询，
并且最后返回了查询结果的所有行。

.. _tutorial_functions:

使用SQL函数
^^^^^^^^^^^^^^^^^^^^^^^^^^

在本章的前面部分:tutorial_group_by_w_aggregates中首次引入的 :data:`_sql.func` 对象用作用于函数的工厂，
当被用于诸如 :func:`_sql.select` 这样的表达式中时，
它会生成一个SQL函数显示，通常由一个名称、括号（虽然不总是）和可能的一些参数组成。常见的SQL函数示例包括：

* ``count()`` 函数，一个聚合函数，用于计算返回行数：

  .. sourcecode:: pycon+sql

      >>> print(select(func.count()).select_from(user_table))
      {printsql}SELECT count(*) AS count_1
      FROM user_account

  ..

* ``lower()`` 函数，一个字符串函数，将字符串转换为小写：

  .. sourcecode:: pycon+sql

      >>> print(select(func.lower("A String With Much UPPERCASE")))
      {printsql}SELECT lower(:lower_2) AS lower_1

  ..

* ``now()`` 函数，用于提供当前日期和时间；由于这是一个常用的函数，因此SQLAlchemy会根据后端的不同情况进行不同的显示，例如SQLite中使用``CURRENT_TIMESTAMP``函数：

  .. sourcecode:: pycon+sql

      >>> stmt = select(func.now())
      >>> with engine.connect() as conn:
      ...     result = conn.execute(stmt)
      ...     print(result.all())
      {execsql}BEGIN (implicit)
      SELECT CURRENT_TIMESTAMP AS now_1
      [...] ()
      [(datetime.datetime(...),)]
      ROLLBACK

  ..

由于大多数数据库用后百甚至数百个不同的SQL函数，因此 :data:`_sql.func` 尝试尽可能宽松地接受任何名称。
从此命名空间中访问的任何名称都会自动被视为将以一种通用方式呈现的SQL函数，并通过上下文中的数据来确定SQL语法。


并不是所有的SQL函数都被 SQLAlchemy 这个小型函数列表所知，例如“lower”和“upper”这样的方便操作并不是它所知道的函数，这是非常常见的状况。

.. _tutorial_window_functions:

使用窗口函数
######################

窗口函数是一种特殊的 SQL 聚合函数模式，它计算作为单个结果行被处理时，一组计算此聚合函数值的行的值。
虽然诸如“MAX()”等聚合函数会为您提供一组行中的最高值，但将相同的函数用作“窗口函数”将为您提供该行的最高值，即该行上的值。。
在SQL中，窗口函数允许指定应应用函数的行，一个“分区” 值，其中考虑不同的行子集的窗口以及一个“order by”表达式，
它特别重视是在哪个顺序将行应用于聚合函数。

在SQLAlchemy中， :data:`_sql.func` 命名空间生成的所有 SQL 函数都包括 :meth:`_functions.FunctionElement.over` 方法，该方法授予窗口函数或“over”语法；它生成的构造是“:class：_sql.Over”构造。

常用于窗口函数的函数是“row_number()”函数，它仅计算行数。我们可以将此行计数划分为用户名，以为单个用户编号的电子邮件地址：

.. sourcecode:: pycon+sql

    >>> stmt = (
    ...     select(
    ...         func.row_number().over(partition_by=user_table.c.name),
    ...         user_table.c.name,
    ...         address_table.c.email_address,
    ...     )
    ...     .select_from(user_table)
    ...     .join(address_table)
    ... )
    >>> with engine.connect() as conn:  # doctest:+SKIP
    ...     result = conn.execute(stmt)
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    SELECT row_number() OVER (PARTITION BY user_account.name) AS anon_1,
    user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    [...] ()
    {stop}[('1', 'sandy', 'sandy@sqlalchemy.org'), ('2', 'sandy', 'sandy@squirrelpower.org'), ('1', 'spongebob', 'spongebob@sqlalchemy.org')]
    {printsql}ROLLBACK{stop}

在上面的代码中，我们使用了 :meth:`_functions.FunctionElement.over` 方法，
生成了一个_OVER构造，并且最后返回了查询结果的所有行。

.. tip::

 值得注意的是， :meth:`_functions.FunctionElement.over` 方法仅适用于那些 SQL 聚合函数，
 如果函数本身不是 SQL 聚合函数，虽然 :class:`_sql.Over` 构造将很愉快地呈现。
 但如果函数本身不是 SQL 聚合函数，则数据库将拒绝表达。


.. _tutorial_functions_within_group:

特殊的WITHIN GROUP，FILTER修饰符
######################################

“WITHIN GROUP” SQL语法与“有序集”或“虚拟集”聚合函数一起使用，
并计算被返回的行中作为单个结果行处理的聚合值。
常见的“有序集”函数包括“percentile_cont ()”和“rank ()”。
SQLAlchemy 包括内置的实现 :class:`_functions.rank`， :class:`_functions.dense_rank`，
:class:`_functions.mode`， :class:`_functions.percentile_cont`和 :class:`_functions.percentile_disc`，
它们包括 :meth:`_functions.FunctionElement.within_group` 方法::

    >>> print(
    ...     func.unnest(
    ...         func.percentile_disc([0.25, 0.5, 0.75, 1]).within_group(user_table.c.name)
    ...     )
    ... )
    {printsql}unnest(percentile_disc(:percentile_disc_1) WITHIN GROUP (ORDER BY user_account.name))


“FILTER”由一些后端支持，以将聚合函数的范围限制为与返回的总行不同的特定子集，
可使用 :meth:`_functions.FunctionElement.filter` 方法来实现::

    >>> stmt = (
    ...     select(
    ...         func.count(address_table.c.email_address).filter(user_table.c.name == "sandy"),
    ...         func.count(address_table.c.email_address).filter(
    ...             user_table.c.name == "spongebob"
    ...         ),
    ...     )
    ...     .select_from(user_table)
    ...     .join(address_table)
    ... )
    >>> with engine.connect() as conn:  # doctest:+SKIP
    ...     result = conn.execute(stmt)
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    SELECT count(address.email_address) FILTER (WHERE user_account.name = ?) AS anon_1,
    count(address.email_address) FILTER (WHERE user_account.name = ?) AS anon_2
    FROM user_account JOIN address ON user_account.id = address.user_id
    [...] ('sandy', 'spongebob')
    {stop}[(2, 1)]
    {execsql}ROLLBACK

.. _tutorial_functions_table_valued:

表值函数
#######################

表值 SQL 函数支持包含命名子元素的标量表示。通常用于 JSON 和 ARRAY 导向的函数，
以及函数例如：generate_series（）等。表值函数在PostgreSQL数据库中很常见，
但某些表值函数形式也受到SQLite、Oracle和SQL Server的支持。

.. seealso::

    :ref:`postgresql_table_valued_overview` - in the :ref:`postgresql_toplevel` documentation.

SQLAlchemy 提供了 :meth:`_functions.FunctionElement.table_valued` 方法作为基本的“表值函数”构造，
该构造将 :data:`_sql.func` 对象转换为包含一系列命名列的 FROM 子句。这将返回一个 :class:`_sql.TableValuedAlias` 对象，
这是一个支持函数的 :class:`_sql.Alias` 构造，可像其他 FROM 子句一样使用，如：在 :ref:`tutorial_using_aliases` 中介绍的那样。
下面我们用JSON格式的“json_each（）”函数来说明它的应用，虽然它在PostgreSQL上很常见，但现在SQLite的现代版本也支持。

.. sourcecode:: pycon+sql

    >>> onetwothree = func.json_each('["one", "two", "three"]').table_valued("value")
    >>> stmt = select(onetwothree).where(onetwothree.c.value.in_(["two", "three"]))
    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     result.all()
    {execsql}BEGIN (implicit)
    SELECT anon_1.value
    FROM json_each(?) AS anon_1
    WHERE anon_1.value IN (?, ?)
    [...] ('["one", "two", "three"]', 'two', 'three')
    {stop}[('two',), ('three',)]
    {execsql}ROLLBACK{stop}

上面的代码中，我们使用“json_each（）”JSON函数，它在SQLite和PostgreSQL中都很常见，用于生成一个包含单个列
的表值表达式，该列被称为“value”。 然后我们选择了其三行中的两行。

.. seealso::

    :ref:`postgresql_table_valued` - in the :ref:`postgresql_toplevel` documentation -
    此节将详细介绍额外的PostgreSQL语法，例如特殊列派生和“WITH ORDINALITY”。
    它们众所周知，在 PostgreSQL 中会产生非常多的表值函数，并且一些情形下也很流行。

.. _tutorial_functions_column_valued:

列值函数 - 将表值函数视为标量列
##################################################################

由 PostgreSQL 和 Oracle 支持的一种特殊语法是，在 FROM 子句中引用函数，然后将其作为 SELECT 语句或其他列表达式的列提供。
PostgreSQL将这个句法用到了许多函数中，例如“json_array_elements（）”、“json_object_keys（）”、“json_each_text（）”、“json_each（）”等。

SQLAlchemy 将其称为“ 列值 函数”，可通过在 :class:`_functions.Function` 构造中应用 :meth:`_functions.FunctionElement.column_valued` 修改符来使用：

   >>> from sqlalchemy import select, func
   >>> stmt = select(func.json_array_elements('["one", "two"]').column_valued("x"))
   >>> print(stmt)
   {printsql}SELECT x
   FROM json_array_elements(:json_array_elements_1) AS x

这个“列值”的形式也支持Oracle方言，其中它可用于自定义SQL函数：

    >>> from sqlalchemy.dialects import oracle
    >>> stmt = select(func.scalar_strings(5).column_valued("s"))
    >>> print(stmt.compile(dialect=oracle.dialect()))
    {printsql}SELECT s.COLUMN_VALUE
    FROM TABLE (scalar_strings(:scalar_strings_1)) s


.. seealso::

    :ref:`postgresql_column_valued` - 在 :ref:`postgresql_toplevel` 文档中阅读

.. _tutorial_casts:

数据转换和类型转换
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 SQL 中，我们经常需要为表达式显式指定数据类型，要么是为了告诉数据库预期在其它情况下模棱两可的表达式中的类型是什么，
要么是为了在某些情况下将 SQL 表达式的隐式数据类型转换为其他类型。
CAST 是用于此任务的 SQL CAST 关键字，在 SQLAlchemy 中是由 :func:`.cast` 函数提供的。
这个函数接受一个列表达式和一个数据类型对象作为参数，
如下图所示，我们从 user_table.c.id 列对象生成了一个 SQL 表达式“CAST(user_account.id AS VARCHAR)”：

    >>> from sqlalchemy import cast
    >>> stmt = select(cast(user_table.c.id, String))
    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     result.all()
    {execsql}BEGIN (implicit)
    SELECT CAST(user_account.id AS VARCHAR) AS id
    FROM user_account
    [...] ()
    {stop}[('1',), ('2',), ('3',)]
    {execsql}ROLLBACK{stop}

:func:`.cast` 函数不仅渲染 SQL CAST 语法，而且还将生成 SQLAlchemy 列表达式，该表达式在 Python 方面也拥有给定的数据类型。例如，将一个被 :func:`.cast` 转换为 :class:`_sqltypes.JSON` 的字符串表达式涉及 JSON 子脚本和比较运算符：

    >>> from sqlalchemy import JSON
    >>> print(cast("{'a': 'b'}", JSON)["a"])
    {printsql}CAST(:param_1 AS JSON)[:param_2]


type_coerce() - Python 专用转换
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

有时需要告诉 SQLAlchemy 表达式的数据类型，出于上述所有原因，但不应在 SQL 方面呈现 CAST 表达式，
在 SQL 方面，这可能会干扰已经可以正常工作的 SQL 操作。对于这个很常见的用例，有另一个函数 :func:`.type_coerce`，
与 :func:`.cast` 密切相关，它会将特定的 SQL 数据库类型设置为 Python 表达式，但不会在数据库方面渲染 ``CAST`` 关键字或数据类型。
:func:`.type_coerce` 在处理 :class:`_types.JSON` 数据类型时特别重要，
因为它在不同平台上通常与面向字符串的数据类型具有复杂的关系甚至可能不是显式数据类型，例如在 SQLite 和 MariaDB 上。

.. sourcecode:: pycon+sql

    >>> import json
    >>> from sqlalchemy import JSON
    >>> from sqlalchemy import type_coerce
    >>> from sqlalchemy.dialects import mysql
    >>> s = select(type_coerce({"some_key": {"foo": "bar"}}, JSON)["some_key"])
    >>> print(s.compile(dialect=mysql.dialect()))
    {printsql}SELECT JSON_EXTRACT(%s, %s) AS anon_1

上面的代码中，我们使用 :func:`.type_coerce` 来指示 Python 字典应该被视为 :class:`_types.JSON`，这是特别重要的。
Python ``__getitem__`` 运算符，``['some_key']`` 在这种情况下获得了这种功能并允许呈现 “JSON_EXTRACT” 路径表达式。
