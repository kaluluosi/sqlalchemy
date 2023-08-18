使用ORM映射类编写SELECT语句
============================

.. admonition:: 关于本文档

    本节使用ORM映射示例首次在   :ref:`unified_tutorial`  中进行说明，
    显示在   :ref:`tutorial_declaring_mapped_classes`  部分。

     :doc:`此页面的ORM设置 </_plain_setup>` 。


SELECT语句由   :func:`_sql.select`  函数生成, 该函数返回一个   :class:` _sql.Select`  对象。
要返回的实体和/或SQL表达式 (即 "columns" 子句) 以位置方式传递给该函数。
从那里，使用其他方法生成完整语句，例如下面说明的  :meth:`_sql.Select.where`  方法：

    >>> from sqlalchemy import select
    >>> stmt = select(User).where(User.name == "spongebob")

给定一个完整的   :class:`_sql.Select`  对象，为了在ORM中执行它以获取行，该对象通过
传递给  :meth:`_orm.Session.execute` ，这时会返回一个   :class:` .Result`  对象来执行::

    >>> result = session.execute(stmt)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('spongebob',){stop}
    >>> for user_obj in result.scalars():
    ...     print(f"{user_obj.name} {user_obj.fullname}")
    spongebob Spongebob Squarepants


.. _orm_queryguide_select_columns:

选择ORM实体和属性
----------------------

  :func:`_sql.select`  构造接受ORM实体，包括映射类以及表示映射列的类级别属性，
这些在构建时将被转换为  :term:`ORM-annotated`    :class:` _sql.FromClause`  和 
  :class:`_sql.ColumnElement`  元素。

包含ORM注释实体的   :class:`_sql.Select`  对象通常使用   :class:` _orm.Session`  对象
执行，而不是   :class:`_engine.Connection`  对象，以便ORM相关特征生效，其中包括返回
ORM映射对象的实例。当直接使用   :class:`_engine.Connection`  时，结果行只包含列级数据。

.. _orm_queryguide_select_orm_entities:

选择ORM实体
^^^^^^^^^^^^^^^^^^^^^^

以下我们从 ``User`` 实体中选取，生成一个从映射到 ``User`` 的   :class:`_schema.Table` 
中选取的   :class:`_sql.Select`  对象::

    >>> result = session.execute(select(User).order_by(User.id))
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] ()

从ORM实体中选择时，实体本身作为只有一个元素的行返回，而不是一组单独的列，例如上述查询中，
  :class:`_engine.Result`  返回一行只有一个元素，该元素持有一个 ` `User`` 对象::

    >>> result.all()
    [(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),),
     (User(id=2, name='sandy', fullname='Sandy Cheeks'),),
     (User(id=3, name='patrick', fullname='Patrick Star'),),
     (User(id=4, name='squidward', fullname='Squidward Tentacles'),),
     (User(id=5, name='ehkrabs', fullname='Eugene H. Krabs'),)]


当选择包含ORM实体的单元素行列表时，通常跳过生成   :class:`_engine.Row`  对象，直接接收ORM实体本身。
最简单的方法是使用  :meth:`_orm.Session.scalars`  方法执行，而不是使用  :meth:` _orm.Session.execute` 
方法，因此返回一个   :class:`.ScalarResult`  对象，该对象产生单个元素而不是行::

    >>> session.scalars(select(User).order_by(User.id)).all()
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] ()
    {stop}[User(id=1, name='spongebob', fullname='Spongebob Squarepants'),
     User(id=2, name='sandy', fullname='Sandy Cheeks'),
     User(id=3, name='patrick', fullname='Patrick Star'),
     User(id=4, name='squidward', fullname='Squidward Tentacles'),
     User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]

调用  :meth:`_orm.Session.scalars`  方法等同于调用  :meth:` _orm.Session.execute`  来接收一个   :class:`_engine.Result` 
对象，然后调用  :meth:`_engine.Result.scalars`  来接收一个   :class:` _engine.ScalarResult`  对象。


.. _orm_queryguide_select_multiple_entities:

同时选择多个ORM实体
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

函数   :func:`_sql.select`  一次接受任意数量的ORM类和/或列表达式，包括请求多个
ORM类的情况。在从多个ORM类进行SELECT时，它们根据其类名在每个结果行中命名。
在下面的示例中，对 ``User`` 和 ``Address`` 进行SELECT的结果行将如下命名：将它们命名为“User”和“Address”并进行引用：

    >>> stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)
    >>> for row in session.execute(stmt):
    ...     print(f"{row.User.name} {row.Address.email_address}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    address.id AS id_1, address.user_id, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    ORDER BY user_account.id, address.id
    [...] (){stop}
    spongebob spongebob@sqlalchemy.org
    sandy sandy@sqlalchemy.org
    sandy squirrel@squirrelpower.org
    patrick pat999@aol.com
    squidward stentcl@sqlalchemy.org

如果我们想在行中为这些实体分配不同的名称，则应使用   :func:`_orm.aliased`  构造使用  :paramref:` _orm.aliased.name`  参数来使用显式名称进行别名处理：

    >>> from sqlalchemy.orm import aliased
    >>> user_cls = aliased(User, name="user_cls")
    >>> email_cls = aliased(Address, name="email")
    >>> stmt = (
    ...     select(user_cls, email_cls)
    ...     .join(user_cls.addresses.of_type(email_cls))
    ...     .order_by(user_cls.id, email_cls.id)
    ... )
    >>> row = session.execute(stmt).first()
    {execsql}SELECT user_cls.id, user_cls.name, user_cls.fullname,
    email.id AS id_1, email.user_id, email.email_address
    FROM user_account AS user_cls JOIN address AS email
    ON user_cls.id = email.user_id ORDER BY user_cls.id, email.id
    [...] ()
    {stop}>>> print(f"{row.user_cls.name} {row.email.email_address}")
    spongebob spongebob@sqlalchemy.org

上述别名形式在   :ref:`orm_queryguide_joining_relationships_aliased`  中进行了进一步讨论。

现有的   :class:`_sql.Select`  构造也可以使用  :meth:` _sql.Select.add_columns`  方法将 ORM 类和/或列表达式添加到其列子句中。我们也可以使用此形式来生成与上面相同的语句:

    >>> stmt = (
    ...     select(User).join(User.addresses).add_columns(Address).order_by(User.id, Address.id)
    ... )
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname,
    address.id AS id_1, address.user_id, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    ORDER BY user_account.id, address.id

选择单个属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

映射类上的属性，例如 `User.name` 和 `Address.email_address`，可以像 `:_schema.Column` 或其他 SQL 表达式对象一样在传递给 `  :func:`_sql.select`  时使用。创建针对特定列的 `  :func:` _sql.select`  将返回 `  :class:`.Row`  对象，而不是像 ` User` 或 `Address` 对象这样的实体。每个 `  :class:`.Row`  将单独表示每个列:

    >>> result = session.execute(
    ...     select(User.name, Address.email_address)
    ...     .join(User.addresses)
    ...     .order_by(User.id, Address.id)
    ... )
    {execsql}SELECT user_account.name, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    ORDER BY user_account.id, address.id
    [...] (){stop}

上述语句返回具有 `name` 和 `email_address` 列的 `  :class:`.Row`  对象，如下运行时演示所示:

    >>> for row in result:
    ...     print(f"{row.name}  {row.email_address}")
    spongebob  spongebob@sqlalchemy.org
    sandy  sandy@sqlalchemy.org
    sandy  squirrel@squirrelpower.org
    patrick  pat999@aol.com
    squidward  stentcl@sqlalchemy.org

.. _bundles:

使用 Bundles 分组选择的属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_orm.Bundle`  构造是一个可扩展的仅限于 ORM 的构造，允许将列表达式集合分组在结果行中::

    >>> from sqlalchemy.orm import Bundle
    >>> stmt = select(
    ...     Bundle("user", User.name, User.fullname),
    ...     Bundle("email", Address.email_address),
    ... ).join_from(User, Address)
    >>> for row in session.execute(stmt):
    ...     print(f"{row.user.name} {row.user.fullname} {row.email.email_address}")
    {execsql}SELECT user_account.name, user_account.fullname, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    [...] (){stop}
    spongebob Spongebob Squarepants spongebob@sqlalchemy.org
    sandy Sandy Cheeks sandy@sqlalchemy.org
    sandy Sandy Cheeks squirrel@squirrelpower.org
    patrick Patrick Star pat999@aol.com
    squidward Squidward Tentacles stentcl@sqlalchemy.org

  :class:`_orm.Bundle`  可能用于创建轻量级视图和自定义列分组。   :class:` _orm.Bundle`  也可以派生以返回替代数据结构；请参见 :meth:`_orm.Bundle.create_row_processor` 的示例。

.. seealso::

      :class:`_orm.Bundle` 

     :meth:`_orm.Bundle.create_row_processor` 


.. _orm_queryguide_orm_aliases:

选择 ORM 别名
^^^^^^^^^^^^^^^^^^^^^

如在   :ref:`tutorial_using_aliases`  中讨论的那样，要创建 ORM 实体的 SQL 别名，需要使用   :func:` _orm.aliased`  构造对映射的类进行别名处理：

    >>> from sqlalchemy.orm import aliased
    >>> u1 = aliased(User)
    >>> print(select(u1).order_by(u1.id))
    {printsql}SELECT user_account_1.id, user_account_1.name, user_account_1.fullname
    FROM user_account AS user_account_1 ORDER BY user_account_1.id

与使用  :meth:`_schema.Table.alias`  时的情况一样，SQL 别名将在查询中表示为 ` tableName_1`，`tableName_2` 等。在 hibernate 方言中，orm 的 SQL 别名表示为 `tableName<sequenceNo>`。匿名命名。针对通过显式名称从行中选择实体的情况，还可以传递  :paramref:`_orm.aliased.name`  参数::

    >>> from sqlalchemy.orm import aliased
    >>> u1 = aliased(User, name="u1")
    >>> stmt = select(u1).order_by(u1.id)
    >>> row = session.execute(stmt).first()
    {execsql}SELECT u1.id, u1.name, u1.fullname
    FROM user_account AS u1 ORDER BY u1.id
    [...] (){stop}
    >>> print(f"{row.u1.name}")
    spongebob

.. seealso::
      :class:`_orm.aliased`  构造对于一些用例非常重要，包括：

    * 使用子查询与 ORM；章节   :ref:`orm_queryguide_subqueries`  和
        :ref:`orm_queryguide_join_subqueries`  进一步讨论了这一点。
    * 控制结果集中实体的名称；请参阅   :ref:`orm_queryguide_select_multiple_entities`  以获取示例
    * 多次连接同一 ORM 实体；请参阅   :ref:`orm_queryguide_joining_relationships_aliased`  以获取示例。

.. _orm_queryguide_selecting_text:

从文本语句获取 ORM 结果
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ORM 支持从来自其他资源的 SELECT 语句中加载实体。典型的用例是文本 SELECT 语句，
在 SQLAlchemy 中使用   :func:`_sql.text`  构造表示。可以使用   :func:` _sql.text`  构造
增加关于语句将加载的 ORM 映射列的信息；然后，可以将其与 ORM 实体本身相关联，以便
基于此语句加载 ORM 对象。

假设我们想从文本 SQL 语句中进行加载::

    >>> from sqlalchemy import text
    >>> textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")

我们可以使用  :meth:`_sql.TextClause.columns`  方法将列信息添加到该语句中；
当此方法被调用时，  :class:`_sql.TextClause`  对象将转换为
  :class:`_sql.TextualSelect`  对象，该对象的角色与   :class:` _sql.Select`  构造相似。
  :meth:`_sql.TextClause.columns`   方法通常传递   :class:` _schema.Column`  对象或等效对象，
在此情况下，我们可以直接使用 ``User`` 类上的 ORM 映射属性::

    >>> textual_sql = textual_sql.columns(User.id, User.name, User.fullname)

我们现在拥有了一个 ORM 配置的 SQL 构造，即可单独加载 “id”，“name” 和 “fullname”
列。为了使用此 SELECT 语句作为完整 ``User`` 实体的源，我们可以使用  :meth:`_sql.Select.from_statement` 
方法将这些列链接到一个常规的 ORM-enabled
  :class:`_sql.Select`  构造中::

    >>> orm_sql = select(User).from_statement(textual_sql)
    >>> for user_obj in session.execute(orm_sql).scalars():
    ...     print(user_obj)
    {execsql}SELECT id, name, fullname FROM user_account ORDER BY id
    [...] (){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    User(id=3, name='patrick', fullname='Patrick Star')
    User(id=4, name='squidward', fullname='Squidward Tentacles')
    User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')

相同的   :class:`_sql.TextualSelect`  对象也可以使用  :meth:` _sql.TextualSelect.subquery`  方法
转换为子查询，并以类似于   :ref:`orm_queryguide_subqueries`  下面讨论的方式使用   :func:` _orm.aliased` 
构造将其连接到 ``User`` 实体上::

    >>> orm_subquery = aliased(User, textual_sql.subquery())
    >>> stmt = select(orm_subquery)
    >>> for user_obj in session.execute(stmt).scalars():
    ...     print(user_obj)
    {execsql}SELECT anon_1.id, anon_1.name, anon_1.fullname
    FROM (SELECT id, name, fullname FROM user_account ORDER BY id) AS anon_1
    [...] (){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')
    User(id=3, name='patrick', fullname='Patrick Star')
    User(id=4, name='squidward', fullname='Squidward Tentacles')
    User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')

使用  :meth:`_sql.Select.from_statement`  直接使用   :class:` _sql.TextualSelect` 
与使用   :func:`_sql.aliased`  的区别在于，在前一种情况下，不会在结果 SQL 中产生子查询。
这在某些情况下从性能或复杂性的角度来看是有优势的。

.. _orm_queryguide_subqueries:

从子查询中选择实体
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在前一节中讨论的   :func:`_orm.aliased`  构造可以与来自诸如  :meth:` _sql.Select.subquery`  的
任何   :class:`_sql.Subuqery`  构造一起使用，以将 ORM 实体链接到该子查询返回的列；
必须存在一种 **列对应关系** 关系，这意味着子查询提供的列和映射到实体的列之间必须存在
对应关系，即，最终需要将子查询连接到 ORM 实体元素上，如下面在   :ref:`orm_queryguide_join_subqueries` 
中讨论的一样：从这些实体派生，例如以下示例::

    >>> inner_stmt = select(User).where(User.id < 7).order_by(User.id)
    >>> subq = inner_stmt.subquery()
    >>> aliased_user = aliased(User, subq)
    >>> stmt = select(aliased_user)
    >>> for user_obj in session.execute(stmt).scalars():
    ...     print(user_obj)
    {execsql} SELECT anon_1.id, anon_1.name, anon_1.fullname
    FROM (SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
    FROM user_account
    WHERE user_account.id < ? ORDER BY user_account.id) AS anon_1
    [generated in ...] (7,)
    {stop}User(id=1, name='海绵宝宝', fullname='Spongebob Squarepants')
    User(id=2, name='珊迪', fullname='Sandy Cheeks')
    User(id=3, name='派大星', fullname='Patrick Star')
    User(id=4, name='章鱼哥', fullname='Squidward Tentacles')
    User(id=5, name='蟹老板', fullname='Eugene H. Krabs')

.. seealso::

      :ref:`tutorial_subqueries_orm_aliased`  - 在   :ref:` unified_tutorial`  中

      :ref:`orm_queryguide_join_subqueries` 

.. _orm_queryguide_unions:

从UNIONs和其他集合操作中选择实体
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :func:`_sql.union`  和   :func:` _sql.union_all`  函数是最常见的集合操作，还有其他集合操作，如
  :func:`_sql.except_` 、   :func:` _sql.intersect`  等等，它们生成一个名为
  :class:`_sql.CompoundSelect`  的对象，由多个   :class:` _sql.Select`  构造体通过集合操作关键词连接。ORM实体可以通过
  :meth:`_sql.Select.from_statement`   方法从简单的复合选择中选择，该方法在   :ref:` orm_queryguide_selecting_text`  中已经介绍过。在这种方法中，UNION语句是将呈现的完整语句，不能在使用  :meth:`_sql.Select.from_statement`  之后添加额外的条件：

    >>> from sqlalchemy import union_all
    >>> u = union_all(
    ...     select(User).where(User.id < 2), select(User).where(User.id == 3)
    ... ).order_by(User.id)
    >>> stmt = select(User).from_statement(u)
    >>> for user_obj in session.execute(stmt).scalars():
    ...     print(user_obj)
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.id < ? UNION ALL SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.id = ? ORDER BY id
    [generated in ...] (2, 3)
    {stop}User(id=1, name='海绵宝宝', fullname='Spongebob Squarepants')
    User(id=3, name='派大星', fullname='Patrick Star')

  :class:`_sql.CompoundSelect`  构造体可以更灵活地在查询中使用，可以通过将其组织为子查询并使用   :func:` _orm.aliased`  将其链接到 ORM 实体来进一步修改。 正如在   :ref:`orm_queryguide_subqueries`  中所示，下面的示例首先使用  :meth:` _sql.CompoundSelect.subquery`  创建 UNION ALL 语句的子查询，然后将其打包到   :func:`_orm.aliased`  构造体中，其中可以像任何其他映射实体一样在   :func:` _sql.select`  构造体中使用，包括我们可以基于其导出列添加过滤和排序标准：

    >>> subq = union_all(
    ...     select(User).where(User.id < 2), select(User).where(User.id == 3)
    ... ).subquery()
    >>> user_alias = aliased(User, subq)
    >>> stmt = select(user_alias).order_by(user_alias.id)
    >>> for user_obj in session.execute(stmt).scalars():
    ...     print(user_obj)
    {execsql}SELECT anon_1.id, anon_1.name, anon_1.fullname
    FROM (SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
    FROM user_account
    WHERE user_account.id < ? UNION ALL SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
    FROM user_account
    WHERE user_account.id = ?) AS anon_1 ORDER BY anon_1.id
    [generated in ...] (2, 3)
    {stop}User(id=1, name='海绵宝宝', fullname='Spongebob Squarepants')
    User(id=3, name='派大星', fullname='Patrick Star')


.. seealso::

      :ref:`tutorial_orm_union`  - 在   :ref:` unified_tutorial`  中

.. _orm_queryguide_joins:

连接
-----

  :meth:`_sql.Select.join`   和  :meth:` _sql.Select.join_from`  方法用于构建针对 SELECT 语句的 SQL JOIN。

本节将详细介绍这些方法的 ORM 使用情况。有关其在 Core 中使用的概述，请参见   :ref:`unified_tutorial`  中的   :ref:` tutorial_select_join` 。

在  :term:`2.0 style`  查询中，在 ORM 上下文中使用  :meth:` _sql.Select.join`  的用法大部分相同（除了遗留用例），与  :term:`1.x style`  查询中使用  :meth:` _orm.Query.join`  方法相同。

.. _orm_queryguide_simple_relationship_join:

简单的关系联接
^^^^^^^^^^^^^^^^^^^^^^^^^^

考虑两个类 ``User`` 和 ``Address`` 的映射，其中关系 ``User.addresses`` 表示与每个 ``User`` 关联的 ``Address`` 对象的集合。  :meth:`_sql.Select.join`  的最常见用法是在此创建 JOIN使用` `User.addresses``属性作为指示符进行关联，参考以下代码：

    >>> stmt = select(User).join(User.addresses)

以上代码中，调用`_sql.Select.join`方法连接`User.addresses`将导致SQL大致等效于：

    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account JOIN address ON user_account.id = address.user_id

在上面的示例中，我们将`User.addresses`作为传递给`_sql.Select.join`方法的“on子句”来引用，即，它表示如何构建JOIN的“ON”部分。

.. tip::

   请注意，使用`_sql.Select.join`方法从一个实体连接到另一个实体会影响SELECT语句中的FROM子句，但不会影响列子句；例如在此示例中，SELECT语句将继续从仅`User`实体返回行。要同时选择来自``User``和``Address``的列/实体，必须在`_sql.select`函数中指定``Address``实体，或者在之后使用`_sql.Select.add_columns`方法将其添加到`_sql.Select`结构中。有关这两种形式的示例，请参见   :ref:`orm_queryguide_select_multiple_entities`  章节。

级联多个连接
^^^^^^^^^^^^^^^^^^^^^^^^

要构建连接链，可以使用多个`_sql.Select.join`调用。关系绑定属性同时确定连接的左侧和右侧。考虑另外两个实体`Order`和`Item`，其中`User.orders`关系指向`Order`实体，`Order.items`关系通过关联表`order_items`指向`Item`实体，两个`_sql.Select.join`调用将分别从`User`连接到`Order`，以及从`Order`连接到`Item`。然而，由于`Order.items`是一个   :ref:`多对多<relationships_many_to_many>`  关系，因此会得到两个单独的JOIN元素，从而导致具有三个JOIN元素的SQL结果：

    >>> stmt = select(User).join(User.orders).join(Order.items)
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN user_order ON user_account.id = user_order.user_id
    JOIN order_items AS order_items_1 ON user_order.id = order_items_1.order_id
    JOIN item ON item.id = order_items_1.item_id

每次调用`_sql.Select.join`方法的顺序只有左侧需要在FROM列表中出 现时是有意义的。例如，如果我们指定 `select(User).join(Order.items).join(User.orders)`，则`_sql.Select.join`方法则不知道如何正确连接，将引发错误。在正确的做法中，`_sql.Select.join`方法应该以我们希望在SQL中呈现JOIN子句的方式调用，而每个调用应表示从其前面的内容中清晰地引用。

我们从FROM子句中获取的所有元素仍然可以作为进一步加入到上面示例中的User实体的潜在连接点。例如，我们将"user.addresses"关系添加到我们的连接中：

    >>> stmt = select(User).join(User.orders).join(Order.items).join(User.addresses)
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN user_order ON user_account.id = user_order.user_id
    JOIN order_items AS order_items_1 ON user_order.id = order_items_1.order_id
    JOIN item ON item.id = order_items_1.item_id
    JOIN address ON user_account.id = address.user_id

连接到目标实体
^^^^^^^^^^^^^^^^^^^^^^^^

`_sql.Select.join`方法的第二种形式允许以任何映射实体或核心可选构造为目标。在此用法中，`_sql.Select.join`方法将尝试使用两个实体之间的自然外键关系来推断连接的ON子句：

    >>> stmt = select(User).join(Address)
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account JOIN address ON user_account.id = address.user_id

在以上调用形式中，`_sql.Select.join`方法被调用以自动推断“on子句”。如果两个映射的`_schema.Table` 构造之间没有`_schema.ForeignKeyConstraint`设置，或者如果存在多个`_schema.ForeignKeyConstraint`连接，使得使用合适的约束不明确，这种调用形式最终会引发错误。

.. note:: 在使用没有指示ON子句的`_sql.Select.join` 或`_sql.Select.join_from`时，ORM配置的`_orm.relationship`结构不会被考虑。仅考虑映射在`_schema.Table`对象级别上的配置的`_schema.ForeignKeyConstraint`关系。
当尝试为JOIN推断ON子句时。

.. _queryguide_join_onclause:

使用ON子句连接到目标
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

第三个调用形式允许同时传递目标实体和ON子句
作为显式参数。包括 SQL 表达式作为 ON 子句的示例如下：

    >>> stmt = select(User).join(Address, User.id == Address.user_id)
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account JOIN address ON user_account.id = address.user_id

基于表达式的 ON 子句也可以是   :func:`_orm.relationship`  绑定属性，就像在
  :ref:`orm_queryguide_simple_relationship_join`  中使用的方式一样：

    >>> stmt = select(User).join(Address, User.addresses)
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account JOIN address ON user_account.id = address.user_id

上面的示例似乎有些冗余，因为它以两种不同的方式指示“Address”的目标；然而，
当加入到别名实体时，这种形式的效用变得明显；请参阅章节
  :ref:`orm_queryguide_joining_relationships_aliased`  中的示例。

.. _orm_queryguide_join_relationship_onclause_and:

.. _orm_queryguide_join_on_augmented:

将 Relationship 与自定义 ON 条件组合
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

由   :func:`_orm.relationship`  构建的 ON 子句可以添加其他条件以增强其表达式，
这个功能对于快速地限制关系路径上特定连接的作用非常有用，也可以使用它来配置装载策略，
例如   :func:`_orm.joinedload`  和   :func:` _orm.selectinload` 。方法  :meth:`_orm.PropComparator.and_` 
按位置接受一系列 SQL 表达式，这些表达式将通过 AND 连接到 JOIN 的 ON 子句上。
例如，如果我们想从“User”到“Address”进行连接，但也限制 ON 条件仅适用于某些电子邮件地址：

.. sourcecode:: pycon+sql

    >>> stmt = select(User.fullname).join(
    ...     User.addresses.and_(Address.email_address == "squirrel@squirrelpower.org")
    ... )
    >>> session.execute(stmt).all()
    {execsql}SELECT user_account.fullname
    FROM user_account
    JOIN address ON user_account.id = address.user_id AND address.email_address = ?
    [...] ('squirrel@squirrelpower.org',){stop}
    [('Sandy Cheeks',)]

.. seealso::

     :meth:`_orm.PropComparator.and_`  方法也适用于装载策略，例如
      :func:`_orm.joinedload`  和   :func:` _orm.selectinload` 。请参阅章节   :ref:`loader_option_criteria` 。

.. _tutorial_joining_relationships_aliased:

.. _orm_queryguide_joining_relationships_aliased:

使用 Relationship 在别名目标之间进行连接
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用   :func:`_orm.relationship`  绑定属性构建连接时，可以将
  :func:`_orm.aliased`  构造扩展为使用二元语法，以使用 SQL 别名作为连接的目标，
同时仍然利用   :func:`_orm.relationship`  绑定属性来指示 ON 子句，例如下面的示例，
其中 “User” 实体两次加入到两个不同的   :func:`_orm.aliased`  构造中：

    >>> address_alias_1 = aliased(Address)
    >>> address_alias_2 = aliased(Address)
    >>> stmt = (
    ...     select(User)
    ...     .join(address_alias_1, User.addresses)
    ...     .where(address_alias_1.email_address == "patrick@aol.com")
    ...     .join(address_alias_2, User.addresses)
    ...     .where(address_alias_2.email_address == "patrick@gmail.com")
    ... )
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN address AS address_1 ON user_account.id = address_1.user_id
    JOIN address AS address_2 ON user_account.id = address_2.user_id
    WHERE address_1.email_address = :email_address_1
    AND address_2.email_address = :email_address_2

同样的模式可以使用修饰符  :meth:`_orm.PropComparator.of_type`  更简洁地表示，它可以应用于
  :func:`_orm.relationship`  绑定属性，并传递目标实体来指示一步中的目标。
下面的示例使用  :meth:`_orm.PropComparator.of_type`  生成与上面所示的 SQL 语句相同的 SQL 语句：

    >>> print(
    ...     select(User)
    ...     .join(User.addresses.of_type(address_alias_1))
    ...     .where(address_alias_1.email_address == "patrick@aol.com")
    ...     .join(User.addresses.of_type(address_alias_2))
    ...     .where(address_alias_2.email_address == "patrick@gmail.com")
    ... )
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN address AS address_1 ON user_account.id = address_1.user_id
    JOIN address AS address_2 ON user_account.id = address_2.user_id
    WHERE address_1.email_address = :email_address_1
    AND address_2.email_address = :email_address_2

要使用   :func:`_orm.relationship`  构造连接 **从** 别名实体构造，该属性可直接从
  :func:`_orm.aliased`  构造中获取：

    >>> user_alias_1 = aliased(User)
    >>> print(select(user_alias_1.name).join(user_alias_1.addresses))
    {printsql}SELECT user_account_1.name
    FROM user_account AS user_account_1
    JOIN address ON user_account_1.id = address.user_id



.. _orm_queryguide_join_subqueries:

与子查询连接
^^^^^^^^^^^^^^^^^^^^^

连接的目标可以是任何“可选择”的实体，包括子查询。在使用 ORM 时，通常是
使用   :func:`_orm.aliased`  取别名的实体作为连接的目标，如下所示：  :func:` _orm.aliased`  方法构造一个  :class:`_sql.Subquery`  方法的目标使用：

    >>> subq = select(Address).where(Address.email_address == "pat999@aol.com").subquery()
    >>> stmt = select(User).join(subq, User.id == subq.c.user_id)
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    JOIN (SELECT address.id AS id,
    address.user_id AS user_id, address.email_address AS email_address
    FROM address
    WHERE address.email_address = :email_address_1) AS anon_1
    ON user_account.id = anon_1.user_id{stop}

上述SELECT语句在通过  :meth:`_orm.Session.execute`  调用时，将返回包含` User`实体而不包含`Address`实体的行。为了将`Address`实体包含到将被返回在结果集中的实体集合中，我们针对`Address`实体和  :class:`.Subquery` "address"` ，以便我们可以在结果行中用名称引用它：

    >>> address_subq = aliased(Address, subq, name="address")
    >>> stmt = select(User, address_subq).join(address_subq)
    >>> for row in session.execute(stmt):
    ...     print(f"{row.User} {row.address}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    anon_1.id AS id_1, anon_1.user_id, anon_1.email_address
    FROM user_account
    JOIN (SELECT address.id AS id,
    address.user_id AS user_id, address.email_address AS email_address
    FROM address
    WHERE address.email_address = ?) AS anon_1 ON user_account.id = anon_1.user_id
    [...] ('pat999@aol.com',){stop}
    User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')

通过关系路径连接到子查询
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在上一节中所展示的子查询形式可以使用一种更加具体的  :func:`_orm.relationship`  中指示的形式之一。例如，要创建相同的联接，同时确保联接沿特定的  :func:` _orm.relationship`  方法，传递  :func:`_orm.aliased` .Subquery` 对象：

    >>> address_subq = aliased(Address, subq, name="address")
    >>> stmt = select(User, address_subq).join(User.addresses.of_type(address_subq))
    >>> for row in session.execute(stmt):
    ...     print(f"{row.User} {row.address}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    anon_1.id AS id_1, anon_1.user_id, anon_1.email_address
    FROM user_account
    JOIN (SELECT address.id AS id,
    address.user_id AS user_id, address.email_address AS email_address
    FROM address
    WHERE address.email_address = ?) AS anon_1 ON user_account.id = anon_1.user_id
    [...] ('pat999@aol.com',){stop}
    User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')

引用多个实体的子查询
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

包含跨越多个ORM实体的列的子查询可以同时应用于多个  :func:`_orm.aliased` .Select` 构造中分别针对每个实体使用。渲染的SQL仍将所有这些 :func:`_orm.aliased` 建构视为同一个子查询，然而在ORM / Python层面上，不同的返回值和对象属性可以使用适当的 :func:`_orm.aliased` 建构来引用。

例如，给定一个涉及到`User`和`Address`的子查询：

    >>> user_address_subq = (
    ...     select(User.id, User.name, User.fullname, Address.id, Address.email_address)
    ...     .join_from(User, Address)
    ...     .where(Address.email_address.in_(["pat999@aol.com", "squirrel@squirrelpower.org"]))
    ...     .subquery()
    ... )

我们可以创建针对`User`和`Address`的 :func:`_orm.aliased` 建构，它们各自都引用相同的对象：

    >>> user_alias = aliased(User, user_address_subq, name="user")
    >>> address_alias = aliased(Address, user_address_subq, name="address")

一个从这两个实体选择的 :class:`.Select` 构造将渲染一次子查询，但在结果行上下文中，可以同时返回`User`和`Address`类的对象：

    >>> stmt = select(user_alias, address_alias).where(user_alias.name == "sandy")
    >>> for row in session.execute(stmt):设置连接的最左FROM子句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在当前的  :class:`_sql.Select`  方法:

    >>> stmt = select(Address).join_from(User, User.addresses).where(User.name == "sandy")
    >>> print(stmt)
    {printsql}SELECT address.id, address.user_id, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    WHERE user_account.name = :name_1

  :meth:`_sql.Select.join_from`  方法接受两个或三个参数，可以表示为` `(<join from>, <onclause>)``或
``(<join from>, <join to>, [<onclause>])``::

    >>> stmt = select(Address).join_from(User, Address).where(User.name == "sandy")
    >>> print(stmt)
    {printsql}SELECT address.id, address.user_id, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    WHERE user_account.name = :name_1

为了设置SELECT的初始FROM子句，以便可以随后使用  :meth:`_sql.Select.join`  ，也可以使用  :meth:` _sql.Select.select_from`  方法：

    >>> stmt = select(Address).select_from(User).join(Address).where(User.name == "sandy")
    >>> print(stmt)
    {printsql}SELECT address.id, address.user_id, address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id
    WHERE user_account.name = :name_1

提示：

  :meth:`_sql.Select.select_from`  方法实际上并没有最终决定FROM子句中表的顺序。如果语句还涉及到一个引用
  :class:`_sql.Join`  
和  :meth:`_sql.Select.join_from`  这些方法时，这些方法最终会创建一个 :class:` _sql.Join`对象。因此，我们可以看到
在这种情况下，  :meth:`_sql.Select.select_from`  的内容被覆盖了：

    >>> stmt = select(Address).select_from(User).join(Address.user).where(User.name == "sandy")
    >>> print(stmt)
    {printsql}SELECT address.id, address.user_id, address.email_address
    FROM address JOIN user_account ON user_account.id = address.user_id
    WHERE user_account.name = :name_1

上面，我们可以看到FROM子句是``address JOIN user_account``，即使我们先声明了``select_from(User)``也是如此。由于
``.join(Address.user)``方法调用，语句最终等效于下面的内容：

    >>> from sqlalchemy.sql import join
    >>>
    >>> user_table = User.__table__
    >>> address_table = Address.__table__
    >>>
    >>> j = address_table.join(user_table, user_table.c.id == address_table.c.user_id)
    >>> stmt = (
    ...     select(address_table)
    ...     .select_from(user_table)
    ...     .select_from(j)
    ...     .where(user_table.c.name == "sandy")
    ... )
    >>> print(stmt)
    {printsql}SELECT address.id, address.user_id, address.email_address
    FROM address JOIN user_account ON user_account.id = address.user_id
    WHERE user_account.name = :name_1

上面的  :class:`_sql.Join`  列表中作为另一个条目添加，它取代了之前的条目。

关系WHERE运算符
----------------------------

除了在  :meth:`.Select.join`  和  :meth:` .Select.join_from`  方法中使用  :func:`_orm.relationship` 
还在帮助构造通常用于WHERE子句的SQL表达式方面发挥了作用，使用  :meth:`.Select.where`  方法。

存在形式：has()/any()
^^^^^^^^^^^^^^^^^^^^^^^^^^^

 :class:`_sql.Exists` 构造首先出现在 :ref:`unified_tutorial` 的 :ref:`tutorial_exists` 中，该对象用于在标量子查询中
与SQL EXISTS关键字产生连接。 :func:`_orm.relationship` 构造为构造一些常见的存在样式提供了一些助手方法，这些样式用关系进行，例如
对于one-to-many关系，如``User.addresses``，可以使用  :meth:`_orm.PropComparator.any`  对` `address``表执行与
相关联的``user_account``表的EXISTS。该方法接受一个可选的WHERE标准，以限制子查询匹配的行：.. sourcecode:: pycon+sql

    >>> stmt = select(User.fullname).where(
    ...     User.addresses.any(Address.email_address == "squirrel@squirrelpower.org")
    ... )
    >>> session.execute(stmt).all()
    {execsql}SELECT user_account.fullname
    FROM user_account
    WHERE EXISTS (SELECT 1
    FROM address
    WHERE user_account.id = address.user_id AND address.email_address = ?)
    [...] ('squirrel@squirrelpower.org',){stop}
    [('Sandy Cheeks',)]

由于EXISTS tends在负查找方面更有效率，一个常见的查询是定位没有相关实体的实体，这可以使用诸如``~User.addresses.any()``这样的短语来简洁地选择“User”实体，这些实体没有相关的“Address”行:

.. sourcecode:: pycon+sql

    >>> stmt = select(User.fullname).where(~User.addresses.any())
    >>> session.execute(stmt).all()
    {execsql}SELECT user_account.fullname
    FROM user_account
    WHERE NOT (EXISTS (SELECT 1
    FROM address
    WHERE user_account.id = address.user_id))
    [...] (){stop}
    [('Eugene H. Krabs',)]

  :meth:`_orm.PropComparator.has`   方法在大多数情况下与  :meth:` _orm.PropComparator.any`   相同，除了它用于一对多关系，例如，如果我们想定位“sandy”所属的所有“Address”对象：

.. sourcecode:: pycon+sql

    >>> stmt = select(Address.email_address).where(Address.user.has(User.name == "sandy"))
    >>> session.execute(stmt).all()
    {execsql}SELECT address.email_address
    FROM address
    WHERE EXISTS (SELECT 1
    FROM user_account
    WHERE user_account.id = address.user_id AND user_account.name = ?)
    [...] ('sandy',){stop}
    [('sandy@sqlalchemy.org',), ('squirrel@squirrelpower.org',)]

.. _orm_queryguide_relationship_common_operators:

关系实例比较操作符
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. comment

    >>> session.expunge_all()

  :func:`_orm.relationship` -bound属性还提供了一些特定于SQL构造的实现，这些实现针对于在特定实例的相关对象中过滤   :func:` _orm.relationship` -bound属性，它可以从给定的  :term:`persistent`   (或较不常见的  :term:` detached`  )对象实例中解压出适当的属性值，并构造关于目标   :func:`_orm.relationship`  的 WHERE 条件。

* **many to one等于比较** - 可以将特定对象实例与many-to-one关系进行比较，以选择外键与目标实体的主键值匹配的行::

      >>> user_obj = session.get(User, 1)
      {execsql}SELECT ...{stop}
      >>> print(select(Address).where(Address.user == user_obj))
      {printsql}SELECT address.id, address.user_id, address.email_address
      FROM address
      WHERE :param_1 = address.user_id

  ..

* **many to one不等于比较** - 也可以使用非等于操作符::

      >>> print(select(Address).where(Address.user != user_obj))
      {printsql}SELECT address.id, address.user_id, address.email_address
      FROM address
      WHERE address.user_id != :user_id_1 OR address.user_id IS NULL

  ..

* **对象包含于one-to-many的集合之中** - 这基本上是“One”-“many”版本的“equals”比较，选择主键等于相关对象的外键值的行::

      >>> address_obj = session.get(Address, 1)
      {execsql}SELECT ...{stop}
      >>> print(select(User).where(User.addresses.contains(address_obj)))
      {printsql}SELECT user_account.id, user_account.name, user_account.fullname
      FROM user_account
      WHERE user_account.id = :param_1

  ..

* **从one-to-many的角度来看，对象具有特定的父对象** -  :func:`_orm.with_parent` 函数产生一个比较，该比较返回由给定父对象引用的行，这实际上与使用“==”操作符与many-to-one方面相同::

      >>> from sqlalchemy.orm import with_parent
      >>> print(select(Address).where(with_parent(user_obj, User.addresses)))
      {printsql}SELECT address.id, address.user_id, address.email_address
      FROM address
      WHERE :param_1 = address.user_id