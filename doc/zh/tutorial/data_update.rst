.. highlight:: pycon+sql

.. |prev| replace:: :doc:`data_select`
.. |next| replace:: :doc:`orm_data_manipulation`

.. include:: tutorial_nav_include.rst


.. rst-class:: core-header, orm-addin

.. _tutorial_core_update_delete:

使用UPDATE和DELETE语句
-------------------------------------

到目前为止，我们已经介绍了:class:`_sql.Insert`，可以将一些数据添加到我们的数据库中，然后在:class:`_sql.Select`
中花费大量时间处理用于从数据库检索数据的各种用法。 在这一节中，我们将介绍:class:`_sql.Update`和
:class:`_sql.Delete`构造函数，它们用于修改现有行以及删除现有行。 本节将从Core-centric的角度介绍这些构造函数。


.. container:: orm-header

    **ORM读者** - 如在：ref:`tutorial_core_insert`中所述的情况一样，
    当与ORM一起使用时，:class:` _sql.Update`和:class:` _sql.Delete`操作通常是作为
    :class:`_orm.Session`的一部分在 :term:`unit of work` 过程中内部调用的。

    但是，不像:class:`_sql.Insert`，当ORM使用"ORM-enabled update and delete"模式时，
    :class:`_sql.Update`和:class:`_sql.Delete`构造函数也可以直接使用；因此，熟悉这些构造函数对于ORM的使用非常有用。
    两种用法在:ref:`tutorial_orm_updating`和:ref:`tutorial_orm_deleting`中讨论。

.. _tutorial_core_update:

update() SQL表达式构造函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func: `_sql.update`函数生成一个 :
class:`_sql.Update` 的新实例，该实例表示SQL中的
UPDATE语句，将更新表中的现有数据。

与:func:`_sql.insert`构造函数一样，有一种“传统”的:func:`_sql.update`形式，
它一次只发出对单个表的UPDATE并且不返回任何行。
但是，某些后端支持可以同时修改多个表的UPDATE语句，UPDATE语句还支持RETURNING，
以便可以在结果集中返回匹配行中包含的列。

基本的UPDATE看起来像::

    >>> from sqlalchemy import update
    >>> stmt = (
    ...     update(user_table)
    ...     .where(user_table.c.name == "patrick")
    ...     .values(fullname="Patrick the Star")
    ... )
    >>> print(stmt)
    {printsql}UPDATE user_account SET fullname=:fullname WHERE user_account.name = :name_1

:meth:`_sql.Update.values` 方法控制UPDATE语句的SET元素的内容。这是与:class:`_sql.Insert`
构造函数共享的同一方法。通常可以使用列名称作为关键字参数传递参数。

UPDATE支持所有常见的UPDATE形式，包括针对表达式的更新，
在这些更新中，我们可以使用:class:`_schema.Column`表达式::

    >>> stmt = update(user_table).values(fullname="Username: " + user_table.c.name)
    >>> print(stmt)
    {printsql}UPDATE user_account SET fullname=(:name_1 || user_account.name)


为了支持"executemany"上下文中的UPDATE，其中将对同一语句调用多个参数集，
可以使用:func:`_sql.bindparam`构造函数设置绑定参数；这些替换了通常选择文本
值的位置：


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


可以应用于UPDATE的其他技术，包括：

.. _tutorial_correlated_updates:

相关更新
~~~~~~~~~~~~~~~~~~

使用相关的子查询，UPDATE语句可以使用其他表中的行。子查询可以用作
可以放置列表达式的任何位置::

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


.. _tutorial_update_from:

UPDATE..FROM
~~~~~~~~~~~~~

某些数据库（如PostgreSQL和MySQL）支持“UPDATE FROM”语法，
其中其他表可以在特殊FROM子句中直接指定。 当使用WHERE子句中存在其他表时，
此语法将自动隐式生成变量::

  >>> update_stmt = (
  ...     update(user_table)
  ...     .where(user_table.c.id == address_table.c.user_id)
  ...     .where(address_table.c.email_address == "patrick@aol.com")
  ...     .values(fullname="Pat")
  ... )
  >>> print(update_stmt)
  {printsql}UPDATE user_account SET fullname=:fullname FROM address
  WHERE user_account.id = address.user_id AND address.email_address = :email_address_1


MySQL还有一种特殊语法，可以更新多个表。这要求在VALUES子句中引用:class:`_schema.Table`对象，
以引用其他表::

  >>> update_stmt = (
  ...     update(user_table)
  ...     .where(user_table.c.id == address_table.c.user_id)
  ...     .where(address_table.c.email_address == "patrick@aol.com")
  ...     .values(
  ...         {
  ...             user_table.c.fullname: "Pat",
  ...             address_table.c.email_address: "pat@aol.com",
  ...         }
  ...     )
  ... )
  >>> from sqlalchemy.dialects import mysql
  >>> print(update_stmt.compile(dialect=mysql.dialect()))
  {printsql}UPDATE user_account, address
  SET address.email_address=%s, user_account.fullname=%s
  WHERE user_account.id = address.user_id AND address.email_address = %s

.. _tutorial_parameter_ordered_updates:

参数有序的更新
~~~~~~~~~~~~~~~~~~~~~~~~~~

另一个仅适用于MySQL的行为是，在UPDATE的SET子句中参数的顺序实际上会影响
每个表达式的评估结果。。对于此用例，:meth:`_sql.Update.ordered_values` 方法接受元组序列，
因此可以控制这种顺序[2] _ ::


  >>> update_stmt = update(some_table).ordered_values(
  ...     (some_table.c.y, 20), (some_table.c.x, some_table.c.y + 10)
  ... )
  >>> print(update_stmt)
  {printsql}UPDATE some_table SET y=:y, x=(some_table.y + :y_1)


.. [2] 虽然从Python 3.7开始，Python字典被保证是插入有序的，
    但是：meth:`_sql.Update.ordered_values` 方法仍然在必须特定情况下提供额外的意图清晰度，
    即MySQL UPDATE语句的SET子句必须按特定方式进行处理。

.. _tutorial_deletes:

delete() SQL表达式构造函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:` _sql.delete`函数生成一个:class:` _sql.Delete`的新实例，
表示SQL中的一个DELETE语句，该语句将从表中删除行。

从API的角度来看，:func:`_sql.delete`语句类似于 :func:` _sql.update`. 函数，
传统上不返回任何行，但允许一些数据库后端的RETURNING变体。

::

    >>> from sqlalchemy import delete
    >>> stmt = delete(user_table).where(user_table.c.name == "patrick")
    >>> print(stmt)
    {printsql}DELETE FROM user_account WHERE user_account.name = :name_1


.. _tutorial_multi_table_deletes:

多表删除
~~~~~~~~~~~~~~~~~~~~~~

像:class:` _sql.Update`一样，:class:`_sql.Delete`支持在WHERE子句中使用相关子查询，以及后端特定的多个表语法，例如在MySQL上的“DELETE FROM ... USING”语法::

  >>> delete_stmt = (
  ...     delete(user_table)
  ...     .where(user_table.c.id == address_table.c.user_id)
  ...     .where(address_table.c.email_address == "patrick@aol.com")
  ... )
  >>> from sqlalchemy.dialects import mysql
  >>> print(delete_stmt.compile(dialect=mysql.dialect()))
  {printsql}DELETE FROM user_account USING user_account, address
  WHERE user_account.id = address.user_id AND address.email_address = %s

.. _tutorial_update_delete_rowcount:

从UPDATE，DELETE获取影响行数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:class:` _sql.Update`和:class:` _sql.Delete`都支持能够返回语句执行后匹配到的行数，
这些语句是使用Core :class:`_engine.Connection`触发的，即 :meth:`_engine.Connection.execute`. 
如下所述的注意事项，可以从 :attr:`_engine.CursorResult.rowcount`获取此值属性：

.. sourcecode:: pycon+sql

    >>> with engine.begin() as conn:
    ...     result = conn.execute(
    ...         update(user_table)
    ...         .values(fullname="Patrick McStar")
    ...         .where(user_table.c.name == "patrick")
    ...     )
    ...     print(result.rowcount)
    {execsql}BEGIN (implicit)
    UPDATE user_account SET fullname=? WHERE user_account.name = ?
    [...] ('Patrick McStar', 'patrick'){stop}
    1
    {execsql}COMMIT{stop}

.. tip::

    :class:` _engine.CursorResult`类是:class:` _engine.Result`的子类，
    包含特定于DBAPI“一游标”对象的其他属性。 
    当通过:meth:`_engine.Connection.execute` 方法触发语句时，将返回此子类的对象。
    在使用ORM时，:meth:`_orm.Session.execute` 方法为所有INSERT，UPDATE和DELETE语句返回此类型的对象。

关于 :attr:`_engine.CursorResult.rowcount`的事实：

* 返回的值是WHERE子句所匹配的行数。无论行实际上是否被修改都无所谓。

* :attr:`_engine.CursorResult.rowcount`不一定适用于使用RETURNING的UPDATE或DELETE语句。

* 对于:ref:`executemany <tutorial_multiple_parameters>`执行，:attr:`_engine.CursorResult.rowcount`
    可能也无法使用，这高度依赖于使用的DBAPI模块以及配置的选项。 属性:
    :attr:`_engine.CursorResult.supports_sane_multi_rowcount`指示当前所使用的后端是否将为此值提供可用性。

* 一些驱动程序，特别是非关系性数据库的第三方方言，可能根本不支持:attr:` _engine.CursorResult.rowcount`。
    属性:attr:`_engine.CursorResult.supports_sane_rowcount` 指示了这一点。

* “rowcount”在ORM :term: `unit of work`过程中用于验证UPDATE或DELETE语句匹配的预期行数，
    对于文档中的ORM版本化功能至关重要，详见 :ref:`mapper_version_counter`。

使用RETURNING更新，删除
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

与:class:` _sql.Insert`构造函数一样，:class:` _sql.Update`和:class:` _sql.Delete`也支持RETURNING子句，
可以通过使用:meth:` _sql.Update.returning`和:meth:` _sql.Delete.returning`方法添加该子句。
在可支持RETURNING的后端上，所选择行的某些列将返回:meth:` _engine.Result`对象作为可以迭代的行：

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

进一步阅读UPDATE，DELETE
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. seealso::

    UPDATE / DELETE的API文档：

    * :class:`_sql.Update`

    * :class:`_sql.Delete`

    ORM-enabled UPDATE and DELETE:

    :ref:`orm_expression_update_delete` -请参见：ref:`queryguide_toplevel`