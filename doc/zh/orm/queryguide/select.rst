用ORM Mapper类编写SELECT语句
=============================================

.. admonition:: 关于本文档

    本节利用了 :ref:`unified_tutorial` 中首次介绍的ORM映射，展示在
    :ref:`tutorial_declaring_mapped_classes` 部分。

    :doc:`_plain_setup` 查看该页面的ORM设置。


SELECT语句的生成由 :func:`_sql.select` 函数完成，该函数返回一个 :class:`_sql.Select` 对象。作为函数的位置参数传递实体、SQL表达式等所需返回的内容（即“列”子句）。从这里开始，可以使用其他方法来生成完整语句，例如下面演示的 :meth:`_sql.Select.where` 方法：

    >>> from sqlalchemy import select
    >>> stmt = select(User).where(User.name == "spongebob")

已经完成 :class:`_sql.Select` 对象，为了在ORM中执行它以获取行，必须将对象传递给 :meth:`_orm.Session.execute`，然后返回一个 :class:`.Result` 对象。如下所示：

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
--------------------------------------

:func:`_sql.select` 结构接受ORM实体，包括映射类和表示映射列的类级属性，这些属性在构造时会转换为 :term:`ORM-annotated` :class:`_sql.FromClause` 和 :class:`_sql.ColumnElement` 元素。

普遍使用的很多例子中，同时从多个ORM实体中选择，这是使用以下方法之一构成的：func:`_sql.union_all`、:func:`_sql.union` 和 :meth:`_sql.CompoundSelect`。请参阅 :ref:`orm_queryguide_unions` 以获取有关此主题的详细信息。

当选择ORM实体时，结果以单个元素的行形式返回实体本身，而不是一系列个体列。如果选择多个实体，可以跳过 :class:`_engine.Row` 对象的生成，而直接使用 :meth:`_orm.Session.scalars` 方法来执行，而不是使用 :meth:`_orm.Session.execute` 方法，这样返回的是一个:class:`.ScalarResult` 对象，它只包含一个元素而不是多行。例如：

    >>> session.scalars(select(User).order_by(User.id)).all()
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] ()
    {stop}[User(id=1, name='spongebob', fullname='Spongebob Squarepants'),
     User(id=2, name='sandy', fullname='Sandy Cheeks'),
     User(id=3, name='patrick', fullname='Patrick Star'),
     User(id=4, name='squidward', fullname='Squidward Tentacles'),
     User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]

.. _bundles:

总结选定的属性（Bundles）
--------------------------

:class:`_orm.Bundle` 结构是一种可扩展的只在ORM中使用的结构，可将列表达式分组在分组中。函数 :func:`_sql.select` 可以选择将ORM实体包括在其列子句中，这样就可以将ORM实体与普通列一起返回到同一行。有关 :class:`_orm.Bundle` 的具体细节，请参见 :meth:`_orm.Bundle.create_row_processor`。


.. _orm_queryguide_orm_aliases:

ORM别名
-----------

如 :ref:`tutorial_using_aliases` 中所述，使用 :func:`_orm.aliased` 构建ORM实体的别名。例如：

    >>> from sqlalchemy.orm import aliased
    >>> u1 = aliased(User)
    >>> print(select(u1).order_by(u1.id))
    {printsql}SELECT user_account_1.id, user_account_1.name, user_account_1.fullname
    FROM user_account AS user_account_1 ORDER BY user_account_1.id

使用 :func:`_orm.aliased` 从行中指定实体的名称：

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

    :meth:`_orm.aliased`

.. _orm_queryguide_join_subqueries:

在子查询中进行ORM联接
--------------------------

我们可以通过两种方式选择从子查询字段中加载ORM实体：

1.在 :func:`_sql.Select.from_stmt <sqlalchemy.sql.selectable.Select.from_statement>` 中传递一个含有ORM加载列绑定信息的 :func:`_sql.text` 实例，使用相同语句创建 :class:`_sql.TextualSelect` 实例后可以通过 :func:`_orm.aliased` 与ORM实体关联：

例如：

  >>> from sqlalchemy import text
  >>> textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")
  >>> textual_sql = textual_sql.columns(User.id, User.name, User.fullname)
  >>> orm_sql = select(User).from_statement(textual_sql)
  >>> for user_obj in session.execute(orm_sql).scalars():
  ...     print(user_obj)
  {execsql}SELECT id, name, fullname FROM user_account ORDER BY id
  [...] (){stop}
  User(id=1, name='spongebob', fullname='Spongebob Squarepants')

2.使用 :meth:`_sql.Selectable.subquery <sqlalchemy.sql.selectable.Selectable.subquery>` 方法创建 :class:`_sql.Subquery` 实例，并使用 :func:`_orm.aliased` 与ORM实体关联：

例如：

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
  [...] (){stop}
  User(id=1, name='spongebob', fullname='Spongebob Squarepants')


.. seealso::

    :ref:`tutorial_subqueries_orm_aliased` - from the :ref:`unified_tutorial`

    :ref:`orm_queryguide_joins`


.. _orm_queryguide_joining_relationships_aliased:

用别名进行关系联接
-----------------------------------

利用 :func:`_orm.aliased` 我们可以更好地编写涉及关系的SQL，并且可以为它们命名。例如，假设“User”和“Address”是ORM的两个实体，而“User.addresses”是“User”实体中的一个关系，它表示与每个“User”相关联的“Address”对象的集合。我们可以使用以下方法构建代表联接“User.addresses”的SQL：

  >>> stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)
  >>> for row in session.execute(stmt):
  ...     print(f"{row.User.name} {row.Address.email_address}")
  {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
  address.id AS id_1, address.user_id, address.email_address
  FROM user_account JOIN address ON user_account.id = address.user_id
  ORDER BY user_account.id, address.id
  [...] (){stop}
  spongebob spongebob@sqlalchemy.org


因为 :func:`_sql.Select.join` 需要有一个表作为其参数，可以使用代替 .addresses （或其它类型的关系属性 - 关于这一点可以阅读 第8章）的 :class:`~sqlalchemy.orm.aliased` 对象来指定 “Address” 表，就像 doing  .aliases(Address, name="x") 一样，这是一种有助于将表的表达从关系的表达中区分出来的实用方法。例如，以下是修改后的代码：

    >>> from sqlalchemy.orm import aliased
    >>> address_entity = aliased(Address, name="address")
    >>> stmt = select(User, address_entity).join(User.addresses)
    >>> for row in session.execute(stmt):
    ...     print(f"{row.User.name} {row.address.name}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    address_1.id, address_1.user_id, address_1.email_address
    FROM user_account JOIN address AS address_1 ON user_account.id = address_1.user_id
    [...] (){stop}
    spongebob spongebob@sqlalchemy.org

.. _orm_queryguide_join_onclause:

指定ON从句的目标实体
---------------------------------

在另一种 :func:`_sql.Select.join` 形式中，我们在不在关系上指定 “onclause”的情况下传递了 Entity。这种方法通常是使用映射的ForeignKeyConstraint（有人也可能刻意不这样做）的唯一方法。例如：

    >>> stmt = select(User).join(Address)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account JOIN address ON user_account.id = address.user_id

使用 :meth:`_sql.Select.join_from` 创建自定义情况的联接 详见 :ref:`sqlalchemy:tutorial_select_join` 的最后一点，它们也可以和上面介绍的那样 :class:`_orm.aliased`。

.. note:: 使用 :func:`_sql.Select.join` 或者 :func:`_sql.Select.join_from` 没有在联接上指定“onclause”，ORM 配置的 :func:`_orm.relationship` 构造 **不参与** 计算。只有通过映射的 :class:`_schema.Table` 对象配置的 :class:`_schema.ForeignKeyConstraint` 关系才会在尝试推导“onclause”时被参考。


.. _tutorial_joining_relationships_explicit_on:

用显式ON从句关联实体和文本时的问题
-----------------------------------------------

映射类对于其子句具有特殊关系（如在 :ref:`tutorial_metadata_relationships` 中描述）。因此，如果您显式地编写了一个含有ORM实体以及SQL表达式的查询，您可能需要连接两个类型的元素以进行联接，即如何使用 :meth:`_sql.Select.join` 使用显式ON从句进行联接。当正在进行JOIN时，为了从一个结果返回多个实体，您需要 :func:`_sql.select` 函数，类似 :func:`_sql.select()`，对所有实体进行选择，这将导致相同的实体出现在多个位置上。例如：

    >>> stmt = select(
    ...     User.id,
    ...     User.name.label("user_name"),
    ...     User.fullname.label("user_fullname"),
    ...     Address.id.label("address_id"),
    ...     Address.email_address.label("address_email"),
    ... ).join(Address, User.id == Address.user_id)
    {printsql}SELECT user_account.id, user_account.name AS user_name, user_account.fullname AS user_fullname, address.id AS address_id, address.email_address AS address_email
    FROM user_account JOIN address ON user_account.id = address.user_id

在 :func:`_sql.select` 的情况下，重复实体意味着您需要对其进行别名：

    >>> stmt = select(
    ...     User.id.label("user_id"),
    ...     User.name.label("user_name"),
    ...     User.fullname.label("user_fullname"),
    ...     Address.id.label("address_id"),
    ...     Address.email_address.label("address_email"),
    ... ).join(Address, User.id == Address.user_id)
    {printsql}SELECT user_account.id AS user_id, user_account.name AS user_name, user_account.fullname AS user_fullname, address.id AS address_id, address.email_address AS address_email
    FROM user_account JOIN address ON user_account.id = address.user_id

参见
------

:func:`_orm.aliased`

:func:`_sql.Select.join`

:func:`_sql.Select.join_from`要显示传递ON子句等，必须在join时使用ON操作符。下面是一个使用SQL表达式作为ON子句的示例：

```python
stmt = select(User).join(Address, User.id == Address.user_id)
print(stmt)
```

查询可视化:

```sql
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id
```

同样，基于表达式的ON子句也可以是与 :func:`_orm.relationship`绑定属性，可以参考：:ref:`orm_queryguide_simple_relationship_join`。

```python
stmt = select(User).join(Address, User.addresses)
print(stmt)
```

查询可视化:

```sql
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id
```

使用别名实体相关联的例子请参见 :ref:`orm_queryguide_joining_relationships_aliased` 一章。

:meth:`_orm.PropComparator.and_` 方法接受一系列 SQL 表达式作为位置，将以 AND 连接到 JOIN 的 ON 子句上，可用于快速限制与关系路径相关的特定JOIN的范围或配置加载策略。

```python
stmt = select(User.fullname).join(
    User.addresses.and_(Address.email_address == "squirrel@squirrelpower.org")
)
```

查询可视化:

```sql
SELECT user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id AND address.email_address = 'squirrel@squirrelpower.org'
```

关于 :meth:`_orm.PropComparator.and_` 方法的更多细节可以参考 :ref:`loader_option_criteria` 一章。

:meth:`_orm.relationship`绑定属性的另一个常见用例是生成SQL查询中的WHERE子句，使用 :meth:`.Select.where` 方法。

对于像 `User.addresses` 这样的一对多关系，可以使用 :meth:`_orm.PropComparator.any` 创建一个针对 `address` 表的 EXISTS 关键字，关联回到 `user_account` 表。

```python
stmt = select(User.fullname).where(User.addresses.any(Address.email_address == "squirrel@squirrelpower.org"))
```

查询可视化:

```sql
SELECT user_account.fullname
FROM user_account
WHERE EXISTS (
  SELECT 1
  FROM address
  WHERE user_account.id = address.user_id AND address.email_address = 'squirrel@squirrelpower.org')
```

与 NEGATIVE EXISTS 结合使用以在没有相关实体存在的情况下查找实体的查询，这可以使用 ``~User.addresses.any()`` 实现。

```python
stmt = select(User.fullname).where(~User.addresses.any())
```

查询可视化:

```sql
SELECT user_account.fullname
FROM user_account
WHERE NOT (EXISTS (
  SELECT 1
  FROM address
  WHERE user_account.id = address.user_id)) 
```

在 :func:`_orm.relationship`-bound 属性中，还有一些 SQL 构建实现，这些实现是为了通过指定与目标实例相关的属性值来过滤该属性，具体可参考 :ref:`orm_queryguide_relationship_common_operators`。