.. highlight:: pycon+sql

.. |prev| replace::  :doc:`data_select` 
.. |next| replace::  :doc:`orm_data_manipulation` 

.. include:: tutorial_nav_include.rst


.. rst-class:: core-header, orm-addin

.. _tutorial_core_update_delete:

使用UPDATE和DELETE语句
-------------------------------------

到目前为止，我们已经涵盖了   :class:`_sql.Insert`  ，以使我们能够将一些数据传输到我们的数据库中，并花费了大量时间在   :class:` _sql.Select`  上，它处理了用于从数据库检索数据的广泛使用模式。本节将介绍 `  :class:`_sql.Update`  and   :class:` _sql.Delete`  ，这些结构用于修改现有行以及删除现有行。本节将从核心中心的角度介绍这些结构。

.. container:: orm-header

    **ORM阅读器** - 正如在   :ref:`tutorial_core_insert`  中提到的那样，当使用ORM时，   :class:` _sql.Update`  操作通常作为   :class:`_orm.Session`  对象的一部分在  :term:` unit of work`  过程中内部调用。

    但是，与   :class:`_sql.Insert`  不同，   :class:` _sql.Update`  和   :class:`_sql.Delete`  结构也可以直接与ORM一起使用，使用称为“ORM启用的更新和删除”的模式；因此，熟悉这些结构对ORM使用很有用。两种使用样式都在   :ref:` tutorial_orm_updating`  和   :ref:`tutorial_orm_deleting`  中讨论。

.. _tutorial_core_update:

update() SQL 表达式结构
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :func:`_sql.update`  函数生成   :class:` _sql.Update`  的一个新实例，表示SQL中的UPDATE语句，这将更新表中的现有数据。

与   :func:`_sql.insert`  结构一样，有一个“传统”形式的   :func:` _sql.update` ，每次只对单个表发出UPDATE，不返回任何行。但是，一些后端支持可能同时修改多个表的UPDATE语句，并且UPDATE语句还支持RETURNING，以便返回匹配行中包含的列在结果集中。

一个基本的UPDATE语句如下所示::

    >>> from sqlalchemy import update
    >>> stmt = (
    ...     update(user_table)
    ...     .where(user_table.c.name == "patrick")
    ...     .values(fullname="派翠克·星")
    ... )
    >>> print(stmt)
    {printsql}UPDATE user_account SET fullname=:fullname WHERE user_account.name = :name_1

  :meth:`_sql.Update.values`   方法控制UPDATE语句的 SET 元素的内容。这是   :class:` _sql.Insert`  构造的相同方法。通常可以使用列名称作为关键字参数传递参数。

UPDATE支持UPDATE的所有主要SQL格式，包括对表达式的更新，我们可以使用   :class:`_schema.Column`  表达式。

::

    >>> stmt = update(user_table).values(fullname="用户名：" + user_table.c.name)
    >>> print(stmt)
    {printsql}UPDATE user_account SET fullname=(:name_1 || user_account.name)

为了支持在“executemany”上下文中使用update，其中许多参数集将对同一语句进行调用，可以使用   :func:`_sql.bindparam`  结构设置绑定参数；这些替换了通常会放置字面值的地方:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import bindparam
    >>> stmt = (
    ...     update(user_table)
    ...     .where(user_table.c.name == bindparam("oldname"))
    ...     .values(name=bindparam("newname"))
    ... )
    >>> with engine.begin() as conn:
    ...     conn.execute(
    ...         stmt,
    ...         [
    ...             {"oldname": "jack", "newname": "ed"},
    ...             {"oldname": "wendy", "newname": "mary"},
    ...             {"oldname": "jim", "newname": "jake"},
    ...         ],
    ...     )
    {execsql}BEGIN (implicit)
    UPDATE user_account SET name=? WHERE user_account.name = ?
    [...] [('ed', 'jack'), ('mary', 'wendy'), ('jake', 'jim')]
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT{stop}

可以应用于UPDATE的其他技术包括：

.. _tutorial_correlated_updates:

关联更新
~~~~~~~~

UPDATE语句可以使用其他表中的行，方法是使用   :ref:`关联子查询 <tutorial_scalar_subquery>` 。子查询可以用于任何可以放置列表达式的地方::

  >>> scalar_subq = (
  ...     select(address_table.c.email_address)
  ...     .where(address_table.c.user_id == user_table.c.id)
  ...     .order_by(address_table.c.id)
  ...     .limit(1)
  ...     .scalar_subquery()
  ... )
  >>> update_stmt = update(user_table).values(fullname=scalar_subq)
  >>> print(update_stmt)
  {printsql}UPDATE user_account SET fullname=(SELECT address.email_address
  FROM address
  WHERE address.user_id = user_account.id ORDER BY address.id
  LIMIT :param_1)


.. _tutorial_update_from:UPDATE..FROM
~~~~~~~~~~~~~

一些数据库如PostgreSQL和MySQL支持以下语法："UPDATE FROM"，直接在FROM子句中陈述其他表格。当语句的WHERE子句中有其他表时，会隐式生成此语法：

```python
update_stmt = (
        update(user_table)
        .where(user_table.c.id == address_table.c.user_id)
        .where(address_table.c.email_address == "patrick@aol.com")
        .values(fullname="Pat")
    )
print(update_stmt)

# 打印结果:
# UPDATE user_account SET fullname=:fullname FROM address
# WHERE user_account.id = address.user_id AND address.email_address = :email_address_1
```

还有一种MySQL的特定语法，可以更新多个表格。这要求在VALUES子句中引用 :class:`_schema.Table` 对象以引用其他表格：

```python
update_stmt = (
        update(user_table)
        .where(user_table.c.id == address_table.c.user_id)
        .where(address_table.c.email_address == "patrick@aol.com")
        .values(
            {
                user_table.c.fullname: "Pat",
                address_table.c.email_address: "pat@aol.com",
            }
        )
    )
from sqlalchemy.dialects import mysql
print(update_stmt.compile(dialect=mysql.dialect()))

# 打印结果:
# UPDATE user_account, address
# SET address.email_address=%s, user_account.fullname=%s
# WHERE user_account.id = address.user_id AND address.email_address = %s
```

.. _tutorial_parameter_ordered_updates:

参数和顺序更新
~~~~~~~~~~~~~~~~~~~~~~

另一个仅MySQL的行为是，UPDATE的SET子句中的参数顺序实际上影响了每个表达式的计算。对于这种用例，  :meth:`_sql.Update.ordered_values`  方法接受一个元组序列，以便可以控制此顺序 [2]_：

```python
update_stmt = update(some_table).ordered_values(
        (some_table.c.y, 20), (some_table.c.x, some_table.c.y + 10)
    )
print(update_stmt)

# 打印结果：
# UPDATE some_table SET y=:y, x=(some_table.y + :y_1)
```

.. [2] 虽然Python3.7之后的版本已经保证了字典是按插入顺序排序的（见<https://mail.python.org/pipermail/python-dev/2017-December/151283.html>），但是使用  :meth:`_sql.Update.ordered_values`  方法仍提供了一种额外的清晰表达的措施，当必须明确指定MySQL UPDATE语句的SET子句的顺序时。

.. _tutorial_deletes:

delete() SQL表达式构建
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

 :func:`_sql.delete` 函数生成一个新的 :class:`_sql.Delete` 实例，该实例表示DELETE SQL语句，可以从表格中删除行。

从API的角度来看， :func:`_sql.delete` 语句与 :func:`_sql.update` 语句非常相似，传统上不返回任何行，但允许在某些数据库后端上使用RETURNING变体。

```python
from sqlalchemy import delete
stmt = delete(user_table).where(user_table.c.name == "patrick")
print(stmt)

# 打印结果
# DELETE FROM user_account WHERE user_account.name = :name_1
```

.. _tutorial_multi_table_deletes:

多表删除
~~~~~~~~~~~~~~~~~~~~~~

与 :class:`_sql.Update` 一样， :class:`_sql.Delete` 也支持在WHERE子句中使用相关子查询以及后端特定的多表语法，如MySQL中的“DELETE FROM..USING”。

```python
delete_stmt = (
        delete(user_table)
        .where(user_table.c.id == address_table.c.user_id)
        .where(address_table.c.email_address == "patrick@aol.com")
    )
from sqlalchemy.dialects import mysql
print(delete_stmt.compile(dialect=mysql.dialect()))

# 打印结果:
# DELETE FROM user_account USING user_account, address
# WHERE user_account.id = address.user_id AND address.email_address = %s
```

.. _tutorial_update_delete_rowcount:

从UPDATE、DELETE中获取受影响的行数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_sql.Update`  。根据下面提到的注意点，此值可从  :attr:` _engine.CursorResult.rowcount`  属性中获取：

```python
with engine.begin() as conn:
    result = conn.execute(
        update(user_table)
        .values(fullname="Patrick McStar")
        .where(user_table.c.name == "patrick")
    )
    print(result.rowcount)

# 打印结果：
# 1
```

.. tip::

      :class:`_engine.CursorResult`  方法调用语句时，会返回此子类的实例。使用ORM时，  :meth:` _orm.Session.execute`  方法为所有INSERT、UPDATE和DELETE语句返回此类型的对象。

  :attr:`_engine.CursorResult.rowcount`  的事实：

* 返回的值是WHERE子句匹配的行数。无论行是否被实际修改都无关紧要。

* 不能保证需要UPDATE的  :attr:`_engine.CursorResult.rowcount`  可用。使用 RETURNING 进行 UPDATE, DELETE
-------------------------------------

和   :class:`_sql.Insert`  一样,   :class:` _sql.Update`  和 
  :class:`_sql.Delete`  也支持返回结果集的 RETURNING 子句。
可以使用  :meth:`_sql.Update.returning`  和  :meth:` _sql.Delete.returning`  方法添加 RETURNING 子句。
当在一个支持 RETURNING 的后端中使用这些方法时，匹配 WHERE 条件的 
所有列将被返回到   :class:`_engine.Result`  对象中，并可以作为可以迭代的行返回::

    >>> update_stmt = (
    ...     update(user_table)
    ...     .where(user_table.c.name == "patrick")
    ...     .values(fullname="Patrick the Star")
    ...     .returning(user_table.c.id, user_table.c.name)
    ... )
    >>> print(update_stmt)
    {printsql}UPDATE user_account SET fullname=:fullname
    WHERE user_account.name = :name_1
    RETURNING user_account.id, user_account.name{stop}

    >>> delete_stmt = (
    ...     delete(user_table)
    ...     .where(user_table.c.name == "patrick")
    ...     .returning(user_table.c.id, user_table.c.name)
    ... )
    >>> print(delete_stmt)
    {printsql}DELETE FROM user_account
    WHERE user_account.name = :name_1
    RETURNING user_account.id, user_account.name{stop}


更新，删除的更多信息
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. 参见::

    UPDATE / DELETE 的 API 文档:

    *   :class:`_sql.Update` 

    *   :class:`_sql.Delete` 

    启用了 ORM 的 UPDATE 和 DELETE:

      :ref:`orm_expression_update_delete`  - 在   :ref:` queryguide_toplevel`  中.