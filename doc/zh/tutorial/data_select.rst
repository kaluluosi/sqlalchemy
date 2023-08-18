.. highlight:: pycon+sql

.. |prev| replace::  :doc:`data_insert` 
.. |next| replace::  :doc:`data_update` 

.. include:: tutorial_nav_include.rst

.. _tutorial_selecting_data:

.. rst-class:: core-header, orm-dependency

使用SELECT语句
-----------------------

在Core和ORM中，  :func:`_sql.select`  方法传递，在ORM中通过  :meth:` _orm.Session.execute`  方法传递, SELECT语句将发出，并通过返回的 :class:`_engine.Result` 对象获得结果行。

.. container:: orm-header

    **ORM读者** - 此处的内容同样适用于Core和ORM，并提及了基本的ORM变体用例。
    但是还有更多专门针对ORM的功能可用; 
    这些在   :ref:`queryguide_toplevel`  中有详细说明。

select() SQL表达式构造
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :func:`_sql.select`  相同的方式构建语句，使用一种生成式方法，
其中每个方法都将更多的状态构建到对象中。与其他SQL构造一样，它可以在原地转化为字符串：

    >>> from sqlalchemy import select
    >>> stmt = select(user_table).where(user_table.c.name == "spongebob")
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = :name_1

与所有其他级别的语句级SQL构造一样，要实际运行语句，我们需要将其传递给执行方法。
由于SELECT语句返回行，我们始终可以迭代结果对象以获取 :class:`_engine.Row ` 对象：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     for row in conn.execute(stmt):
    ...         print(row)
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('spongebob',){stop}
    (1, 'spongebob', 'Spongebob Squarepants')
    {execsql}ROLLBACK{stop}

在使用ORM时，特别是对临近ORM实体的 :func:`_sql.select` 构造进行操作时，
我们将希望使用  :class:`_orm.Session`  方法执行它；
使用这种方法，我们继续从结果中获取 :class:`_engine.Row` 对象，但是这些行现在有能力包含
完整的实体，例如 ``User`` 类的实例，作为每个行中的单个元素：

.. sourcecode:: pycon+sql

    >>> stmt = select(User).where(User.name == "spongebob")
    >>> with Session(engine) as session:
    ...     for row in session.execute(stmt):
    ...         print(row)
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('spongebob',){stop}
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
    {execsql}ROLLBACK{stop}

.. topic:: 从表格和ORM类中的select()问题

    尽管无论我们调用 ``select(user_table)`` 还是 ``select(User)`` 所生成的SQL 
    看起来都是相同的，但在更一般的情况下，它们不一定会渲染为相同的内容，因为ORM映射类
    可以映射到表格以外的其他类型的 "可选择"对象。针对ORM实体的 ``select()`` 也表明应该在结果中返回ORM映射实例，
    这不是从   :class:`_schema.Table`  对象中选择时的情况。

接下来的几节将更详细地讨论SELECT构造。

.. _tutorial_selecting_columns:

设置列和FROM语句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

 :func:`_sql.select` 函数接受位置元素，代表任意数量的 :class:`_schema.Column` 和 / 或  :class:`_schema.Table` 表达式，
以及广泛可兼容的对象，这些对象解析成要从中SELECT的SQL表达式列表，这些表达式将作为结果集中的列返回。
这些元素还在更简单的情况下用于创建FROM子句，
该子句是由传递的列和类似于表格的表达式自动推断出来的。

    >>> print(select(user_table))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account

在使用Core方法从单个列中进行SELECT时，可以直接访问  :attr:`_schema.Table.c`  访问器中的 :class:` _schema.Column`对象， 
FROM子句将从这些列所代表的所有 :class:`_schema.Table` 和其他 :class:`_sql.FromClause ` 对象推断出来：

    >>> print(select(user_table.c.name, user_table.c.fullname))
    {printsql}SELECT user_account.name, user_account.fullname
    FROM user_account

或者 使用任何   :class:`.FromClause`  的  :c:` .FromClause.c`   集合，例如  :class:`.Table` ，可以指定多个列：使用一个字符串名字元组来调用   :func:` _sql.select` ：

    >>> print(select(user_table.c["name", "fullname"]))
    {printsql}SELECT user_account.name, user_account.fullname
    FROM user_account

.. versionadded:: 2.0 向 :attr`.FromClause.c` 集合中添加了元组访问的能力。

选择ORM实体和列
~~~~~~~~~~~~~~~~~

ORM实体，例如我们的 ``User`` 类以及列映射属性，例如 ``User.name``，同样参与SQL表达式语言系统中的表和列。下面我们看一下从 ``User`` 实体中进行SELECT的示例，它最终呈现的方式与我们直接使用 ``user_table`` 相同::

    >>> print(select(User))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account

当我们使用 ORM 的  :meth:`_orm.Session.execute`  方法执行像上述语句这样的语句时，有一个重要的区别，即当我们从完整实体（比如 ` `User``）进行选择时，与 ``user_table`` 相比，**实体本身作为每一行的单个元素返回**。也就是说，当我们从上述语句获取行时，由于只有 ``User`` 实体在要获取的列表中，我们会得到包含 ``User`` 类实例的   :class:`_engine.Row`  对象，该对象只有一个元素::

    >>> row = session.execute(select(User)).first()
    {execsql}BEGIN...
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] (){stop}
    >>> row
    (User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)

以上的   :class:`_engine.Row`  只有一个元素，表示 ` `User`` 实体：

    >>> row[0]
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')

一个非常推荐的方便方法是使用  :meth:`_orm.Session.scalars`  方法直接执行相同的操作；该方法将返回一个   :class:` _result.ScalarResult`  对象，该对象会一次性返回每一行的第一个“列”，在本例中是 ``User`` 类的实例::

    >>> user = session.scalars(select(User)).first()
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] (){stop}
    >>> user
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')

或者，我们可以作为结果行中不同的元素选择ORM实体的单个列，通过使用类绑定属性；当传递给诸如   :func:`_sql.select`  这样的构造时，它们会解析为每个属性所代表的   :class:` _schema.Column`  或其他SQL表达式::

    >>> print(select(User.name, User.fullname))
    {printsql}SELECT user_account.name, user_account.fullname
    FROM user_account

当我们使用  :meth:`_orm.Session.execute`  调用*此*语句时，我们现在会收到每个值对应于单独列或其他SQL表达式的结果行的每个元素：

    >>> row = session.execute(select(User.name, User.fullname)).first()
    {execsql}SELECT user_account.name, user_account.fullname
    FROM user_account
    [...] (){stop}
    >>> row
    ('spongebob', 'Spongebob Squarepants')

这些方法也可以混合使用，在下面的示例中，我们将 ``User`` 实体的 ``name`` 属性作为行的第一个元素来选择，并将其与完整的 ``Address`` 实体合并为第二个元素：

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

选择ORM实体和列以及将行转换为Python对象的通用方法进一步讨论在   :ref:`orm_queryguide_select_columns`  中。

.. seealso::

      :ref:`orm_queryguide_select_columns`  - 在   :ref:` queryguide_toplevel`  中

从带标签的SQL表达式中进行选择
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :meth:`_sql.ColumnElement.label`   方法以及在ORM属性上也有同名方法提供了列或表达式的SQL标签，允许它在结果集中有一个特定的名称。当通过名称引用结果行中的任意SQL表达式时，这可能是有用的：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func, cast
    >>> stmt = select(
    ...     ("Username: " + user_table.c.name).label("username"),
    ... ).order_by(user_table.c.name)
    >>> with engine.connect() as conn:
    ...     for row in conn.execute(stmt):
    ...         print(f"{row.username}")
    {execsql}BEGIN (implicit)
    SELECT ? || user_account.name AS username
    FROM user_account ORDER BY user_account.name[...] ('用户名：'，){stop}
    用户名：帕特里克
    用户名：桑迪
    用户名：海绵宝宝
    {execsql}ROLLBACK{stop}

.. seealso::

      :ref:`tutorial_order_by_label`  - 我们创建的标签名称也可以在   :class:` _sql.Select`  的 ORDER BY 或 GROUP BY 子句中引用。

.. _tutorial_select_arbitrary_text:

使用文本列表达式选择
~~~~~~~~~~~~~~~~~~~~~~

当我们使用   :func:`_sql.select`  函数构造   :class:` _sql.Select`  对象时，
通常传递给它的是使用   :ref:`表元数据 <tutorial_working_with_metadata>`  定义的一系列   :class:` _schema.Table`  和   :class:`_schema.Column`  对象，或者在使用 ORM 时，我们可能会发送 ORM 映射的表示表列的属性。但是，有时还需要在语句中制造任意的 SQL 块，例如常量字符串表达式，或者仅仅是一些更易于直接编写的任意 SQL。

引入的   :func:`_sql.text`  构造，在   :ref:` tutorial_working_with_transactions`  中可以直接嵌入到   :class:`_sql.Select`  构造中，例如下面我们制造了一个硬编码的字符串字面量 ` `'some phrase'``，并将其嵌入到 SELECT 语句中:

  >>> from sqlalchemy import text
  >>> stmt = select(text("'some phrase'"), user_table.c.name).order_by(user_table.c.name)
  >>> with engine.connect() as conn:
  ...     print(conn.execute(stmt).all())
  {execsql}BEGIN (implicit)
  SELECT 'some phrase', user_account.name
  FROM user_account ORDER BY user_account.name
  [generated in ...] ()
  {stop}[('some phrase', 'patrick'), ('some phrase', 'sandy'), ('some phrase', 'spongebob')]
  {execsql}ROLLBACK{stop}

虽然   :func:`_sql.text`  构造可以在大多数地方用于插入字面 SQL 短语，但我们实际上更多地处理每个表示单个列表达式的文字单位。在这种常见情况下，我们可以使用   :func:` _sql.literal_column`  构造函数从我们的文本片段中获取更多的功能。此对象类似于   :func:`_sql.text` ，但它代表一个明确的单“列”，可以在子查询和其他表达式中带标签并引用。

  >>> from sqlalchemy import literal_column
  >>> stmt = select(literal_column("'some phrase'").label("p"), user_table.c.name).order_by(
  ...     user_table.c.name
  ... )
  >>> with engine.connect() as conn:
  ...     for row in conn.execute(stmt):
  ...         print(f"{row.p}, {row.name}")
  {execsql}BEGIN (implicit)
  SELECT 'some phrase' AS p, user_account.name
  FROM user_account ORDER BY user_account.name
  [generated in ...] ()
  {stop}some phrase, patrick
  some phrase, sandy
  some phrase, spongebob
  {execsql}ROLLBACK{stop}


请注意，在使用   :func:`_sql.text`  或   :func:` _sql.literal_column`  时，我们编写的是一个可用语法分析的表达式，而不是一个文字值。因此，我们必须包含所需的任何引号或语法，以出现我们想要呈现的 SQL。

.. _tutorial_select_where_clause:

WHERE 子句
^^^^^^^^^^^^^^^^^^^^

SQLAlchemy 允许我们通过结合   :class:`_schema.Column`  等对象和标准 Python 运算符来构成 SQL 表达式，例如 ` `name = 'squidward'`` 或 ``user_id > 10``。对于布尔表达式，大多数 Python 运算符（如 ``==``，``！=``，``<``，``>=`` 等）会生成新的 SQL 表达式对象，而不是普通的布尔 ``True/False`` 值：

    >>> print(user_table.c.name == "squidward")
    user_account.name = :name_1

    >>> print(address_table.c.user_id > 10)
    address.user_id > :user_id_1

我们可以将这样的表达式传递给  :meth:`_sql.Select.where`  方法，通过生成 WHERE 子句：

    >>> print(select(user_table).where(user_table.c.name == "squidward"))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = :name_1

要生成由 AND 连接的多个表达式，可多次调用  :meth:`_sql.Select.where`  方法：

    >>> print(
    ...     select(address_table.c.email_address)
    ...     .where(user_table.c.name == "squidward")
    ...     .where(address_table.c.user_id == user_table.c.id)
    ... )
    {printsql}SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

单调用  :meth:`_sql.Select.where`  方法也接受多个表达式：

    >>> print(
    ...     select(address_table.c.email_address).where(
    ...         user_table.c.name == "squidward",
    ...         address_table.c.user_id == user_table.c.id,
    ...     )
    ... )
    {printsql}SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

"AND" 和 "OR" 连接都可以直接使用   :func:`_sql.and_`  和   :func:` _sql.or_`  函数进行连接，如下所示：ORM实体查询::

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

如果只是对单个实体进行“相等”比较，还有一种流行的方法，称为  :meth:`_sql.Select.filter_by`  ，它接受与列键或ORM属性名称匹配的关键字参数。它将筛选左侧FROM子句或最后加入的实体::

    >>> print(select(User).filter_by(name="spongebob", fullname="Spongebob Squarepants"))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = :name_1 AND user_account.fullname = :fullname_1


.. seealso::


     :doc:`/core/operators`  - SQL表达式运算符函数的描述


.. _tutorial_select_join:

显式FROM子句和JOIN
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如前所述，FROM子句通常是**推断出来的**基于设置在列子句中的表达式以及  :class:`_sql.Select` 的其他元素。

如果我们从特定的 :class:`_schema.Table` 在COLUMNS子句中设置了一个单列，它也将该  :class:`_schema.Table` 放入FROM子句中::

    >>> print(select(user_table.c.name))
    {printsql}SELECT user_account.name
    FROM user_account

如果我们从两个表中取列，则得到以逗号分隔的FROM子句::

    >>> print(select(user_table.c.name, address_table.c.email_address))
    {printsql}SELECT user_account.name, address.email_address
    FROM user_account, address

为了将这两个表JOIN起来，我们通常使用  :class:`_sql.Select`  方法，它允许我们显式地指示JOIN的左侧和右侧::

    >>> print(
    ...     select(user_table.c.name, address_table.c.email_address).join_from(
    ...         user_table, address_table
    ...     )
    ... )
    {printsql}SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id


另一个是  :meth:`_sql.Select.join`  方法，它只指示JOIN的右侧，左侧被推断出来::

    >>> print(select(user_table.c.name, address_table.c.email_address).join(address_table))
    {printsql}SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

.. sidebar:: ON子句是推断出来的

    当使用  :meth:`_sql.Select.join_from`  或  :meth:` _sql.Select.join`  时，我们可能会发现在简单的外键情况下，JOIN的ON子句也为我们推断出来了。下一节中会详细介绍。

我们还有另一种选择，可以显式地向FROM子句添加元素，如果它没有以我们想要的方式从columns子句中进行推断，我们使用  :meth:`_sql.Select.select_from`  方法来实现这一点，如下所示，我们将` `user_table``作为FROM子句中的第一个元素，并使用  :meth:`_sql.Select.join`  来建立` `address_table``作为第二个元素::

    >>> print(select(address_table.c.email_address).select_from(user_table).join(address_table))
    {printsql}SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

另一个例子是，如果我们的列子句不足以提供FROM子句的信息，我们可能想使用常见的SQL表达式``count(*)``进行SELECT，我们使用SQLAlchemy元素  :attr:`_sql.func`  来生成SQL` `count()``函数::

    >>> from sqlalchemy import func
    >>> print(select(func.count("*")).select_from(user_table))
    {printsql}SELECT count(:count_2) AS count_1
    FROM user_account

.. seealso::

      :ref:`orm_queryguide_select_from`  - 在   :ref:` queryguide_toplevel`  - 中包含有关  :meth:`_sql.Select.select_from`  和  :meth:` _sql.Select.join`  交互的其他示例和注意事项。

.. _tutorial_select_join_onclause:

设置ON子句
~~~~~~~~~~~~~~~~~~~~~

JOIN的先前的示例说明类似于  :class:`_sql.Select` ` user_table``和``address_table``的 :class:`_sql.Table` 对象包括一个单一的 :class:`_schema.ForeignKeyConstraint` 定义，该定义用于形成此ON子句。

如果JOIN的左侧和右侧目标没有这样的约束条件，或存在多个约束条件，则需要直接指定ON子句。  :meth:`_sql.Select.join`  和  :meth:` _sql.Select.join_from`  都接受一个额外的ON子句参数，该参数使用与我们在 :ref:`tutorial_select_where_clause` 中看到的相同的SQL表达式机制来表示：

    >>> print(
    ...     select(address_table.c.email_address)
    ...     .select_from(user_table)
    ...     .join(address_table, user_table.c.id == address_table.c.user_id)
    ... )
    {printsql}SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id.. container:: orm-header

    **ORM Tip** - 使用ORM实体时，还有一种生成ON子句的方法，
    这种方法需要使用   :func:`_orm.relationship`  构造函数，比如上一节中
    声明映射类的部分。这是另一个主题，您可以在   :ref:`tutorial_joining_relationships` 
    中详细了解。

OUTER 和 FULL  join
~~~~~~~~~~~~~~~~~~~

  :meth:`_sql.Select.join`   和  :meth:` _sql.Select.join_from`  方法都接受
  :paramref:`_sql.Select.join.isouter`   和  :paramref:` _sql.Select.join.full`  关键字参数，
分别表示 LEFT OUTER JOIN 和 FULL OUTER JOIN::

    >>> print(select(user_table).join(address_table, isouter=True))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account LEFT OUTER JOIN address ON user_account.id = address.user_id{stop}

    >>> print(select(user_table).join(address_table, full=True))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account FULL OUTER JOIN address ON user_account.id = address.user_id{stop}

还有一个方法  :meth:`_sql.Select.outerjoin`  等价于使用 ` `.join(..., isouter=True)``。

.. tip::

    SQL 还有一个 "RIGHT OUTER JOIN"。SQLAlchemy 并不直接生成该语句，
    而是调换表的顺序，使用 "LEFT OUTER JOIN" 代替。

.. _tutorial_order_by_group_by_having:

ORDER BY, GROUP BY, HAVING
^^^^^^^^^^^^^^^^^^^^^^^^^^^

SELECT SQL 语句包含的一个子句叫做 ORDER BY，它用于按照指定的顺序返回所选行。

GROUP BY 子句的构造方式与 ORDER BY 子句类似，并且其目的是将所选行分成特定的组，
这些组上可以调用聚合函数。HAVING 子句通常与 GROUP BY 一起使用，
其形式与 WHERE 子句类似，但是其作用于组内使用的聚合函数。

.. _tutorial_order_by:

ORDER BY
~~~~~~~~

ORDER BY 子句构造为基于 SQL 表达式构造的   :class:`_schema.Column`  或类似对象。
  :meth:`_sql.Select.order_by`   方法按位置接受一个或多个表达式::

    >>> print(select(user_table).order_by(user_table.c.name))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.name

升序/降序可以从  :meth:`_sql.ColumnElement.asc`  和  :meth:` _sql.ColumnElement.desc` 
修饰符中获得，从 ORM 绑定的属性中也有这些方法:：

    >>> print(select(User).order_by(User.fullname.desc()))
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.fullname DESC

上述语句将按照 ``user_account.fullname`` 列以降序排序返回行。

.. _tutorial_group_by_w_aggregates:

使用 GROUP BY / HAVING 的聚合函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 SQL 中，聚合函数允许将跨多行的列表达式聚合在一起，以生成单个结果。
例如，计数、计算平均值以及在一组值中定位最大或最小值。

SQLAlchemy 使用称为  :data:`_sql.func`  的命名空间以开放的方式提供 SQL 函数。
这是一个特殊的构造函数对象，当给定特定 SQL 函数的名称以及零个或多个要传递给函数的参数时，
它将创建一个新的   :class:`_functions.Function`  实例，参数就像在所有其他情况下一样，
是 SQL 表达式构造。例如，要对 ``user_account.id`` 列渲染 SQL COUNT() 函数，
我们调用 ``count()`` 函数名称::

    >>> from sqlalchemy import func
    >>> count_fn = func.count(user_table.c.id)
    >>> print(count_fn)
    {printsql}count(user_account.id)

本教程稍后在   :ref:`tutorial_functions`  中详细介绍了 SQL 函数。

在 SQL 中使用聚合函数时，GROUP BY 子句是必不可少的，因为它允许将行分区为组，
而聚合函数将分别应用于每个组。在在 SELECT 语句的 COLUMNS 子句中请求非聚合列时，
SQL 要求这些列都受到 GROUP BY 子句的约束，直接或者间接基于主键关联。
然后，HAVING 子句类似于 WHERE 子句，但是它是基于聚合值而不是直接行内容筛选行。

SQLAlchemy 使用  :meth:`_sql.Select.group_by`  和  :meth:` _sql.Select.having`  方法支持这两个子句。
下面我们示例说明选择用户名称字段以及地址计数，对于那些拥有多个地址的用户：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         select(User.name, func.count(Address.id).label("count"))
    ...         .join(Address)
    ...         .group_by(User.name)
    ...         .having(func.count(Address.id) > 1)
    ...     )
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    SELECT user_account.name, count(address.id) AS count
    FROM user_account JOIN address ON user_account.id = address.user_id GROUP BY user_account.name
    HAVING count(address.id) > ?
    [...] (1,){stop}    [('sandy', 2)]
    {execsql}ROLLBACK{stop}

.. _tutorial_order_by_label:

按照标签排序或分组
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在一些数据库后端中，一种非常重要的技术是能够对已经在列数据中声明的表达式进行 ORDER BY 或 GROUP BY ，而不必在 ORDER BY 或 GROUP BY 子句中重新声明该表达式并使用来自 COLUMNS 子句的该列的名称或标记名称。通过把该名称的字符串文本传递到  :meth:`_sql.Select.order_by`   或  :meth:` _sql.Select.group_by`  方法中即可使用这种形式。 传递的文本不会直接呈现，而是以应用程序中声明表达式名称的形式呈现，如果找不到匹配项，则会引发错误。 也可以在这种形式中使用单元修饰符  :func:`.asc`  和   :func:` .desc`  ：

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

现在，我们正在从多个表中进行选择并使用连接，我们很快就会遇到需要在语句的 FROM 子句中多次引用同一个表的情况。我们使用 SQL 别名来实现这一点，它是一种为表或子查询提供替代名称的语法，从而可以在语句中引用。

在 SQLAlchemy 表达式语言中，这些“名称”代替由于  :class:`_sql.Alias`  方法在核心中构造的。  :class:` _sql.Alias`   集合内具有 :class:`_schema.Column` 对象的名称空间。 例如，以下 SELECT 返回所有唯一的用户名对：

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
~~~~~~~~~~~~~~~~~~

ORM的类似于  :meth:`_sql.FromClause.alias`  方法的函数是ORM   :func:` _orm.aliased` `User``实体中选择包含两个特定电子邮件地址的所有对象：

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

    正如在   :ref:`tutorial_select_join_onclause`  中提到的，ORM 提供了使用   :func:` _orm.relationship`  构造的另一种连接方式。上面使用别名的示例在   :ref:`tutorial_joining_relationships_aliased 中 `  来演示。

.. _tutorial_subqueries_ctes:

子查询和公共表达式CTEs
^^^^^^^^^^^^^^^^^^^^

在SQL中，子查询是在括号内呈现在上下文中放置在封闭语句内的SELECT语句。

本节将介绍所谓的“非标量”子查询，通常放置在较大的 SELECT 语句的 FROM 子句中。我们还将介绍公共表达式 (Common Table Expression) 或 CTE ，它与子查询用法相似但还包括其他功能。

SQLAlchemy 使用   :class:`_sql.Subquery`  对象表示子查询和   :class:` _sql.CTE`  来表示公共表达式，通常从  :meth:`_sql.Select.subquery`  和  :meth:` _sql.Select.cte`  方法中获取。任何一个对象都可以用作较大的   :func:`_sql.select`  构造中的FROM元素。

我们可以构建一个   :class:`_sql.Subquery` ，该   :class:` _sql.Subquery`  将从'address'表中选择行的聚合计数(聚合功能和 GROUPBY 已在  :ref:`tutorial_group_by_w_aggregates ` 中介绍）：

    >>> subq = (
    ...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)子查询
~~~~~~~~~

对于基本的子查询，我们可以使用   :class:`_sql.Subquery`  类创建一个SELECT子查询，即：

.. sourcecode:: python

    subq = (
        select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
        .group_by(address_table.c.user_id)
        .subquery()
    )

如果您把打印输出的命令行COPY到Python代码块中，您会得到以下结果：

.. sourcecode:: python

    >>> from sqlalchemy import select, func
    >>> subq = (
    ...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
    ...     .group_by(address_table.c.user_id)
    ...     .subquery()
    ... )
    >>> print(subq)
    SELECT count(address.id) AS count, address.user_id
    FROM address GROUP BY address.user_id

将子查询转换为普通的SELECT语句可以直接调用以返回只含SELECT查询和不含括号的SELECT语句：

.. sourcecode:: python

    >>> print(subq.select())
    SELECT count(address.id) AS count, address.user_id
    FROM address GROUP BY address.user_id

与框架其他FROM元素相同，   :class:`_sql.Subquery`  对象具有  :attr:` _sql.Subquery.c`  命名空间。我们可以使用这个命名空间引用 ``user_id`` 列，以及用我们定制的标签 ``count`` 表达式:

.. sourcecode:: python

    >>> print(select(subq.c.user_id, subq.c.count))
    SELECT anon_1.user_id, anon_1.count
    FROM (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id) AS anon_1

在子查询对象中选择行后，可以将该对象应用于较大的   :class:`_sql.Select`  ，以将数据连接到 ` `user_account`` 表上：

.. sourcecode:: python

    >>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
    ...     user_table, subq
    ... )

    >>> print(stmt)
    SELECT user_account.name, user_account.fullname, anon_1.count
    FROM user_account JOIN (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id) AS anon_1 ON user_account.id = anon_1.user_id

为了从 ``user_account`` 表连接到 ``address`` ，我们使用了  :meth:`_sql.Select.join_from`  方法。尽管 SQL 子查询本身并没有任何约束，但 SQLAlchemy 可以在表示该列的约束时作用于列上的约束，通过确定 ` `subq.c.user_id`` 值由 ``address_table.c.user_id`` 列派生而来，派生的列扩展了该外键约束关系，然后生成了 ON 子句。

公共表达式语句 (CTEs)
~~~~~~~~~~~~~~~~~~~~~~

在SQLAlchemy中，使用  :class:`_sql.CTE`  方法的调用更改为使用  :meth:` _sql.Select.cte`  即可，然后可以使用生成的对象作为FROM元素来使用，但渲染的 SQL 则是完全不同的公共表达式语法。

.. sourcecode:: python

    subq = (
        select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
        .group_by(address_table.c.user_id)
        .cte()
    )

查询输出结果基本相同：

.. sourcecode:: python

    >>> print(subq)
    WITH anon_1 AS
    (SELECT count(address.id) AS count, address.user_id AS user_id
    FROM address GROUP BY address.user_id)
     SELECT anon_1.count, anon_1.user_id
    FROM anon_1

 :class:`_sql.CTE` 构造还提供了使用“递归”样式的能力。在更复杂的情况下，CTE可能由INSERT，UPDATE或DELETE语句的 RETURNING 子句组成。

在两种情况下，子查询和CTE在SQL级别上都被命名为“匿名”名称。在Python代码中，我们不需要提供这些名称。  :class:`_sql.Subquery`  或  :meth:` _sql.Select.cte`  方法的第一个参数传递来提供呈现的名称。

.. seealso::

     :meth:`_sql.Select.subquery`  - 关于子查询的更多细节

     :meth:`_sql.Select.cte`  - CTE的示例，包括如何使用 RECURSIVE 以及面向数据操作语言（DDL）的 CTEs

.. _tutorial_subqueries_orm_aliased:

ORM实体子查询/CTEs
~~~~~~~~~~~~~~~~~~~~

在ORM中，  :func:`_orm.aliased` ` User`` 或 ``Address`` 类)与代表行源的任何  :class:`_sql.FromClause`  的   :class:` _sql.Alias`  相关联。下面的示例说明如何使用   :func:`_orm.aliased` ，生成的   :class:` _sql.Select`  构造最终导出与同一映射   :class:`_schema.Table`  相关的行。Scalar and Correlated Subqueries
--------------------------------

标量(subquery)是一种返回恰好零行或一行以及一个列的子查询功能。子查询然后在封闭的SELECT语句的COLUMNS或WHERE子句中使用，与FROM子句中的常规子查询不同。相关(subquery)是指在封闭的SELECT语句中提到了一个表的标量(subquery)。

SQLAlchemy使用_column.ScalarSelect构造表示标量(subquery)，该构造是_column.ColumnElement表达式层次结构的一部分，与使用_subquery.Subquery构造表示的常规子查询不同，该构造位于column.FromClause层次结构中。

标量(subquery)通常与聚合函数一起使用，但不一定是这样，之前在:ref：`tutorial_group_by_w_aggregates`中介绍。标量(subquery)可以通过使用_selectable.Select.scalar_subquery方法进行显式指示。它的默认字符串形式在被它本身字符串化时呈现为一个普通的SELECT语句，该语句从两个表中进行选择：

.. sourcecode:: pycon+sql

    >>> subq = (
    ...     select(func.count(address_table.c.id))
    ...     .where(user_table.c.id == address_table.c.user_id)
    ...     .scalar_subquery()
    ... )
    >>> print(subq)
    SELECT count(address.id) AS count_1
    FROM address, user_account
    WHERE user_account.id = address.user_id

上述“subq”对象现在在_column.SQL表达式层次结构中，因为它可以像任何其他列表达式一样使用：

.. sourcecode:: pycon+sql

    >>> print(subq == 5)
    (SELECT count(address.id) AS count_1
    FROM address, user_account
    WHERE user_account.id = address.user_id) = :param_1

虽然标量(subquery)本身在字符串化时通过FROM子句呈现出"user_account"和"address"，但在将其嵌入到处理"user_account"表的封闭 :func: '_sql.select'结构中时，“user_account”表会自动与标量(subquery)相关联，这意味着它不会将其呈现为标量(subquery)的FROM子句中。：

.. sourcecode:: pycon+sql

    >>> stmt = select(user_table.c.name, subq.label("address_count"))
    >>> print(stmt)
    SELECT user_account.name, (SELECT count(address.id) AS s_1
    FROM address
    WHERE user_account.id = address.user_id) AS address_count
    FROM user_account

简单的关联(subquery)通常会按预期执行所需的操作。然而，在相关性不明确的情况下，SQLAlchemy 会让我们知道需要更多的明确性::
    
    >>> stmt = (
    ...     select(
    ...         user_table.c.name,
    ...         address_table.c.email_address,
    ...         subq.label("address_count"),
    ...     )
    ...     .join_from(user_table, address_table)
    ...     .order_by(user_table.c.id, address_table.c.id)
    ... )
    >>> print(stmt)
    Traceback (most recent call last):
    ...
    InvalidRequestError: Select statement '<... Select object at ...>' returned
    no FROM clauses due to auto-correlation; specify correlate(<tables>) to
    control correlation manually.
   
为了指明 "user_table" 是我们想要相关的表，我们使用  :meth:`_sql.ScalarSelect.correlate`  或  :meth:` _sql.ScalarSelect.correlate_except`  方法来指定::
    
    >>> subq = (
    ...     select(func.count(address_table.c.id))
    ...     .where(user_table.c.id == address_table.c.user_id)
    ...     .scalar_subquery()
    ...     .correlate(user_table)
    ... )
    
该语句类似于其他列一样返回给定列的数据:

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(
    ...         select(
    ...             user_table.c.name,
    ...             address_table.c.email_address,
    ...             subq.label("address_count"),
    ...         )
    ...         .join_from(user_table, address_table)
    ...         .order_by(user_table.c.id, address_table.c.id)
    ...     )
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    SELECT user_account.name, address.email_address, (SELECT count(address.id) AS count_1
    FROM address
    WHERE user_account.id = address.user_id) AS address_count
    FROM user_account JOIN address ON user_account.id = address.user_id ORDER BY user_account.id, address.id
    [...] (){stop}
    [('spongebob', 'spongebob@sqlalchemy.org', 1), ('sandy', 'sandy@sqlalchemy.org', 2),
     ('sandy', 'sandy@squirrelpower.org', 2)]
    {execsql}ROLLBACK{stop}


.. _tutorial_lateral_correlation:

横跨 LATERAL 相关性
~~~~~~~~~~~~~~~~~~~

横跨 LATERAL 相关性是 SQL 相关性的一种特殊子类，允许可选择的单元在单个 FROM 子句中引用另一个可选择的单元。这是一个极其特殊的用例，虽然是 SQL 标准的一部分，但仅已知适用于最近版本的 PostgreSQL。

通常，如果 SELECT 语句引用它的 FROM 子句中的 “table1 JOIN (SELECT…) AS subquery”，则右侧的子查询可能不能参照左侧的“table1”表达式; 相关性只能参照完全包含此 SELECT 的另一个 SELECT 的一部分的表。LATERAL 关键字允许我们将这个行为反过来，允许来自右侧 JOIN 的相关性。

SQLAlchemy 使用  :meth:`_expression.Select.lateral`  方法支持此功能，该方法创建一个称为   :class:` .Lateral`  的对象。   :class:`.Lateral`  与   :class:` .Subquery`  和   :class:`.Alias`  属于同一系列，但当将构造添加到 SELECT 的 FROM 子句时，还包括相关性行为。下面的示例说明了使用 LATERAL 的 SQL 查询，选择“用户帐户 / 电子邮件地址计数”数据，就像上一节中所讨论的那样::

    >>> subq = (
    ...     select(
    ...         func.count(address_table.c.id).label("address_count"),
    ...         address_table.c.email_address,
    ...         address_table.c.user_id,
    ...     )
    ...     .where(user_table.c.id == address_table.c.user_id)
    ...     .lateral()
    ... )
    >>> stmt = (
    ...     select(user_table.c.name, subq.c.address_count, subq.c.email_address)
    ...     .join_from(user_table, subq)
    ...     .order_by(user_table.c.id, subq.c.email_address)
    ... )
    >>> print(stmt)
    {printsql}SELECT user_account.name, anon_1.address_count, anon_1.email_address
    FROM user_account
    JOIN LATERAL (SELECT count(address.id) AS address_count,
    address.email_address AS email_address, address.user_id AS user_id
    FROM address
    WHERE user_account.id = address.user_id) AS anon_1
    ON user_account.id = anon_1.user_id
    ORDER BY user_account.id, anon_1.email_address

在上述示例中，JOIN 的右侧是相关到左侧的 "user_account" 表的子查询。

使用  :meth:`_expression.Select.lateral`  时，也会将  :meth:` _expression.Select.correlate`  和  :meth:`_expression.Select.correlate_except`  方法应用于   :class:` .Lateral`  结构。

参见:

      :class:`_expression.Lateral` 

     :meth:`_expression.Select.lateral` 



.. _tutorial_union:

UNION，UNION ALL 和其他集合操作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 SQL 中，可以使用 UNION 或 UNION ALL SQL 操作将 SELECT 语句合并在一起，这将生成由一个或多个语句一起产生的所有行的集合。也可以进行其他的集合操作，例如：INTERSECT [ALL] 和 EXCEPT [ALL]。

SQLAlchemy 的   :class:`_sql.Select`  构造支持此类组合，使用类似   :func:` _sql.union` ，   :func:`_sql.intersect`  和   :func:` _sql.except_` ,   :func:`_sql.intersect_all`  和   :func:` _sql.except_all` 。这些函数都接受任意数量的子可选择项，通常是   :class:`_sql.Select`  构造，但也可以是现有的组合。

这些函数生成的结构是   :class:`_sql.CompoundSelect` ，它的使用方式与   :class:` _sql.Select`  结构相同，除了

使用联合查询选择ORM实体
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

前面的示例演示了如何通过给定两个   :class:`_schema.Table`  对象构建 UNION ，然后返回数据库行。如果我们想要使用 UNION 或者其他集合操作来选择 ORM 实体中的行，有两种方法可以使用。在这两种方法中，我们首先构建一个表示我们要执行的 SELECT / UNION / etc 语句的   :func:` _sql.select`  或者   :class:`_sql.CompoundSelect`  对象；这个语句应该是针对目标 ORM 实体或者它们的底层映射的   :class:` _schema.Table`  对象来组成的::

    >>> stmt1 = select(User).where(User.name == "sandy")
    >>> stmt2 = select(User).where(User.name == "spongebob")
    >>> u = union_all(stmt1, stmt2)

对于不在子查询中的简单 SELECT with UNION ，可以使用  :meth:`_sql.Select.from_statement`  方法在 ORM 对象获取上下文中使用。使用这种方法，UNION 语句代表整个查询；在使用  :meth:` _sql.Select.from_statement`  后不能添加其他条件::

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

如果想要以更灵活的方式将 UNION 或其他相关联的构造作为实体相关组件，则可以使用   :class:`_sql.CompoundSelect`  构造函数通过  :meth:` _sql.CompoundSelect.subquery`  将其组织成子查询，然后使用   :func:`_orm.aliased`  函数将其链接到 ORM 对象。这与   :ref:` tutorial_subqueries_orm_aliased`  中介绍的方法相同，首先创建一个自定义“映射”，将我们想要的实体映射到子查询上，然后选择从那个新实体进行选择，就像它是任何其他映射类一样。在下面的例子中，我们可以添加其他条件，例如 ORDER BY ，以外部的 UNION 进行过滤或排序，因为我们可以按子查询导出的列进行过滤或排序::

    >>> user_alias = aliased(User, u.subquery())
    >>> orm_stmt = select(user_alias).order_by(user_alias.id)
    >>> with Session(engine) as session:
    ...     for obj in session.execute(orm_stmt).scalars():
    ...         print(obj)
    {execsql}BEGIN (implicit)
    SELECT anon_1.id, anon_1.name, anon_1.fullname.. _tutorial_exists:

存在子查询
^^^^^^^^^^^^^^^^^^

SQL中的EXISTS关键词是一个运算符，与 :ref:`标量子查询<教程_标量子查询>` 一起使用，
根据SELECT语句是否返回行来返回布尔值true或false。 SQLAlchemy包含一个称为
 :class:`_sql.Exists` 的  :class:`_sql.ScalarSelect` 对象变体，它将生成一个
EXISTS子查询，并且最方便地使用 :meth:`_sql.SelectBase.exists` 方法生成。
以下我们生成一个EXISTS，以便我们可以返回具有多个相关行的" user_account "行：

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

EXISTS构造通常更多地用作否定，例如NOT EXISTS，因为它提供了一种定位相关表没有行的行的SQL
高效的形式。 下面我们选择没有电子邮件地址的用户名称；请注意二进制否定运算符（``~``）
在第二个WHERE子句中使用：

.. sourcecode:: pycon+sql

    >>> subq = (
    ...     select(address_table.c.id).where(user_table.c.id == address_table.c.user_id)
    ... ).exists()
    >>> with engine.connect() as conn:
    ...     result = conn.execute(select(user_table.c.name).where(~subq))
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    SELECT user_account.name
    FROM user_account
    WHERE NOT (EXISTS (SELECT address.id
    FROM address
    WHERE user_account.id = address.user_id))
    [...] (){stop}
    [('patrick',)]
    {execsql}ROLLBACK{stop}


.. _tutorial_functions:

使用SQL功能
^^^^^^^^^^^^^^^^^^^^^^^^^^

在本文档的前面部分，最先引入了  :data:`_sql.func` 
对象，它作为用于创建新的   :class:`_functions.Function`  对象的工厂，
当用在像   :func:`_sql.select`  这样的结构中时，会产生一个 SQL 函数集合，
通常由一个名称、一些括号（虽然不总是如此）和可能一些参数组成。 常见SQL函数的示例包括：

* count() 函数，一个返回多少行数据的聚合函数：

  .. sourcecode:: pycon+sql

      >>> print(select(func.count()).select_from(user_table))
      {printsql}SELECT count(*) AS count_1
      FROM user_account

  ..

* lower() 函数，一个将字符串转换为小写的字符串函数：

  .. sourcecode:: pycon+sql

      >>> print(select(func.lower("A String With Much UPPERCASE")))
      {printsql}SELECT lower(:lower_2) AS lower_1

  ..

* now() 函数，提供当前日期和时间；作为一种常见函数，SQLAlchemy 知道如何为每个后端以不同的方式呈现
  这个函数，在SQLite的情况下使用CURRENT_TIMESTAMP函数：

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

由于大多数数据库后端都具有几十甚至几百个不同的SQL函数，
  :data:`_sql.func`   在接受任何可能的名称时尽可能宽容。
从此命名空间访问的任何名称都被自动视为SQL功能，其以通用方式呈现::

    >>> print(select(func.some_crazy_function(user_table.c.name, 17)))
    {printsql}SELECT some_crazy_function(user_account.name, :some_crazy_function_2) AS some_crazy_function_1
    FROM user_account

同时，一组相对较小的极其常见的SQL函数，例如   :class:`_functions.count` 、
  :class:`_functions.max` 、  :class:` _functions.concat`  提供了自己的预打包版本，
这些版本提供了一些正确的typing信息以及特定于后端的SQL。在某些情况下，尽管某个函数在SQL表达式中的功能相同，但其SQL生成情况可能不同。下面的示例对比了针对PostgreSQL方言和Oracle方言的   :class:`_functions.now`  函数时所产生的SQL生成：

    >>> from sqlalchemy.dialects import postgresql
    >>> print(select(func.now()).compile(dialect=postgresql.dialect()))
    {printsql}SELECT now() AS now_1{stop}
    >>> from sqlalchemy.dialects import oracle
    >>> print(select(func.now()).compile(dialect=oracle.dialect()))
    {printsql}SELECT CURRENT_TIMESTAMP AS now_1 FROM DUAL{stop}

函数有返回类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于函数是列表达式，因此它们也具有描述在生成的SQL表达式中该表达式的SQL   :ref:`数据类型 <types_toplevel>`  的SQL返回类型。我们将这些类型在这里称为"SQL返回类型"，指的是在数据库端SQL表达式的上下文中函数返回的SQL值的类型，而不是Python函数的"返回类型"。

可以通过引用  :attr:`_functions.Function.type`  属性来访问任何SQL函数的SQL返回类型，通常是为了调试目的：

    >>> func.now().type
    DateTime()

在使用表达式的上下文中，这些SQL返回类型非常重要。例如，在执行数学运算符时，如果表达式的数据类型类似于   :class:`_types.Integer`  或   :class:` _types.Numeric` ，则这些运算符将工作得更好；为了使JSON访问器能够工作，必须使用一种诸如   :class:`_types.JSON`  的类型。某些类别的函数返回整个行而不是列值，因此需要引用特定的列；这些函数称为   :ref:` 表值函数 <tutorial_functions_table_valued>` 。

对于需要在创建的函数中应用特定类型的情况，我们可以使用  :paramref:`_functions.Function.type_`  参数来传递它；类型参数可以是   :class:` _types.TypeEngine`  类或实例。在下面的示例中，我们将   :class:`_types.JSON`  类传递给生成PostgreSQL ` `json_object()`` 函数，注意SQL返回类型将为JSON类型：

    >>> from sqlalchemy import JSON
    >>> function_expr = func.json_object('{a, 1, b, "def", c, 3.5}', type_=JSON)

通过使用   :class:`_types.JSON`  数据类型创建我们的JSON函数，SQL表达式对象具有了JSON相关的功能，例如访问元素：

    >>> stmt = select(function_expr["def"])
    >>> print(stmt)
    {printsql}SELECT json_object(:json_object_1)[:json_object_2] AS anon_1

内置函数有预配置的返回类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

针对常见聚合函数，如   :class:`_functions.count` 、  :class:` _functions.max` 、  :class:`_functions.min` ，以及极少数日期函数，如   :class:` _functions.now`  和字符串函数，SQL返回类型会得到适当的设置，有时基于使用情况。例如，   :class:`_functions.max`  函数和类似的聚合过滤函数将根据给定的参数设置SQL返回类型：

    >>> m1 = func.max(Column("some_int", Integer))
    >>> m1.type
    Integer()

    >>> m2 = func.max(Column("some_str", String))
    >>> m2.type
    String()

日期和时间函数通常对应于由   :class:`_types.DateTime` 、  :class:` _types.Date`  或   :class:`_types.Time`  描述的SQL表达式：

    >>> func.now().type
    DateTime()
    >>> func.current_date().type
    Date()

已知的字符串函数，如   :class:`_functions.concat` ，将知道SQL表达式的类型是   :class:` _types.String` 。

    >>> func.concat("x", "y").type
    String()

但是，对于大多数SQL函数，SQLAlchemy并没有将其明确地包括在其非常小的已知函数列表中配置。例如，尽管通常使用SQL函数 ``func.lower()`` 和 ``func.upper()`` 来转换字符串的大小写，但SQLAlchemy实际上并不知道这些函数，因此它们具有"null" SQL返回类型：

    >>> func.upper("lowercase").type
    NullType()

对于简单的函数，如 ``upper`` 和 ``lower``，通常不太重要，因为可以在不需要SQLAlchemy在处理方面进行特殊类型处理的情况下从数据库接收字符串值，SQLAlchemy的类型强制转换规则通常也可以正确猜测意图；Python ``+`` 操作符也可以在大多数情况下使用，因为SQLAlchemy通常可以发现合适的类型。窗口函数
==========
窗口函数是一种特殊用途的SQL聚合函数，它计算了作为单个结果行进行处理的组中返回的行的聚合值。虽然像`MAX()`这样的函数会给你一个集合中最高的列值，但使用同一函数作为“窗口函数”将为您提供每行的最高值，就像是在该行之前*的一样。

在SQL中，窗口函数允许您指定应应用函数的行，这是"分区"值，它考虑了不同子集合的窗口，并且"order by"表达式非常重要，包括其行应用到的聚合函数的顺序。
在SQLAlchemy中，由  :data:`_sql.func`  命名空间生成的所有SQL函数都包括一个方法  :meth:` _functions.FunctionElement.over` ，它赋予了窗口函数或"OVER"语法；生成的构造是   :class:`_sql.Over` 。

与窗口函数一起常用的函数是`row_number()`函数，它仅计算行数。我们可以针对用户名将此行计数进行分区，以为单个用户的电子邮件地址编号：

```pycon+sql
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
...
[(1, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (1, 'spongebob', 'spongebob@sqlalchemy.org')]
```

在上述示例中，使用  :paramref:`_functions.FunctionElement.over.partition_by`  参数使得` PARTITION BY`子句在 OVER 子句中呈现出来。
我们也可以使用 :paramref:`_functions.FunctionElement.over.order_by` 子句：

```pycon+sql
>>> stmt = (
...     select(
...         func.count().over(order_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  # doctest:+SKIP
...     result = conn.execute(stmt)
...     print(result.all())
...
[(2, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (3, 'spongebob', 'spongebob@sqlalchemy.org')]
```

窗口函数的其他选项包括使用范围；有关更多示例，请参见   :func:`_expression.over` 。

.. tip::

   请注意，  :meth:`_functions.FunctionElement.over`  方法只适用于那些实际上是聚合函数的SQL函数；虽然   :class:` _sql.Over`  构造会很愉快地为给定的任何SQL函数呈现自己，但如果函数本身不是SQL聚合函数，则数据库将拒绝表达式。.. _tutorial_functions_within_group:

WITHIN GROUP, FILTER和特殊修改器
######################################

"Within Group" SQL语法与"有序集"或"假设集合"函数一起使用。
通常的"有序集"函数包括``percentile_cont()``和``rank()``。
SQLAlchemy包括内置实现``_functions.rank``、``_functions.dense_rank``、
``_functions.mode``、``_functions.percentile_cont``和``_functions.percentile_disc``,
它们包括一个  :meth:`_functions.FunctionElement.within_group`  方法::

    >>> print(
    ...     func.unnest(
    ...         func.percentile_disc([0.25, 0.5, 0.75, 1]).within_group(user_table.c.name)
    ...     )
    ... )
    {printsql}unnest(percentile_disc(:percentile_disc_1) WITHIN GROUP (ORDER BY user_account.name))

"Filter"由某些后端支持，以将聚合函数的范围限制为与返回的总行范围相比的特定子集，可以使用
  :meth:`_functions.FunctionElement.filter`  方法::

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

表值SQL函数支持包含命名子元素的标量表示。
常用于JSON和ARRAY导向函数及如`_functions.generate_series()`等函数，
表值函数在FROM子句中指定，然后被引用为表，有时甚至被引用为列。
这种形式的函数在PostgreSQL数据库中非常突出，
但某些形式的表值函数也被SQLite、Oracle和SQL Server支持。

.. seealso::

      :ref:`postgresql_table_valued_overview`  - 在   :ref:` postgresql_toplevel`  文档中。

    虽然很多数据库支持表值和其他特殊形式，但PostgreSQL通常是最需要这些功能的地方。
    查看此部分以获取PostgreSQL语法的其他示例以及其他功能。

SQLAlchemy提供了  :meth:`_functions.FunctionElement.table_valued`  方法作为基本的“表值函数”结构，
它将函数  :_data:``  _sql.func` `对象转换为包含一系列命名列的FROM子句，基于传递的字符串名称位置。
这会返回一个 :class:`_sql.TableValuedAlias` 对象，这是一个启用函数的 :class:`_sql.Alias` 构造，
它可以像其他在  :ref:`tutorial_using_aliases` 中介绍的FROM子句一样使用。下面我们示例了`_functions.json_each()`函数，
该函数虽然在PostgreSQL上很常见，但也被现代版本的SQLite支持::

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

上面，我们使用SQLite和PostgreSQL支持的`json_each()` JSON函数生成了一个单列的表值表达式，
该列被称为`value`，然后选择了其三行中的两行。

.. seealso::

      :ref:`postgresql_table_valued`  - 在   :ref:` postgresql_toplevel`  文档中 -
    此部分将详细介绍其他语法，例如特殊列派生和“WITH ORDINALITY”，这些语法已知有效地使用PostgreSQL。

.. _tutorial_functions_column_valued:

列值函数 - 表值函数作为标量列
##################################################################

PostgreSQL和Oracle支持的一种特殊语法是指向“表值函数作为标量列”的语法。数据类型转换与强制转换
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在SQL中，我们经常需要明确表示表达式的数据类型，无论是告诉数据库在另一种不明确的表达式中期望什么类型，还是在某些情况下想要将SQL表达式的隐含数据类型转换成其他类型。SQL CAST关键字用于此任务，在SQLAlchemy中由  :func:`.cast` ` user_table.c.id``列对象生成SQL表达式``CAST(user_account.id AS VARCHAR)``：

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

  :func:`.cast` .cast` 成为 :class:`_sqltypes.JSON` 的字符串表达式将获得JSON下标和比较运算符，例如：

    >>> from sqlalchemy import JSON
    >>> print(cast("{'a': 'b'}", JSON)["a"])
    {printsql}CAST(:param_1 AS JSON)[:param_2]

type_coerce() - 一个仅限于Python的“转换”
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

有时需要让SQLAlchemy知道表达式的数据类型，出于上述所有原因，但不要在SQL端呈现CAST表达式，因为这可能会影响已无需CAST而正常工作的SQL操作。为这个相当普遍的用例，有另一个与  :func:`.cast` .type_coerce` ，它设置了一个Python表达式作为特定的SQL数据库类型，但不会在数据库方面呈现“CAST”关键字或数据类型。  :func:`.type_coerce` .type_coerce` 将Python结构作为JSON字符串传递到MySQL的JSON函数之一：

.. sourcecode:: pycon+sql

    >>> import json
    >>> from sqlalchemy import JSON
    >>> from sqlalchemy import type_coerce
    >>> from sqlalchemy.dialects import mysql
    >>> s = select(type_coerce({"some_key": {"foo": "bar"}}, JSON)["some_key"])
    >>> print(s.compile(dialect=mysql.dialect()))

以上，因为我们使用  :func:`.type_coerce` ，所以调用了MySQL的` `JSON_EXTRACT`` SQL函数，此时Python的``__getitem__``运算符，在这种情况下为``['some_key']``，由此允许渲染``JSON_EXTRACT``路径表达式（未显示出来，但在这种情况下，它最终将是``'$."some_key"'``）。