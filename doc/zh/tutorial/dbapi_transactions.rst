.. |prev| replace:: :doc:`engine`
.. |next| replace:: :doc:`metadata`

.. include:: tutorial_nav_include.rst


.. _tutorial_working_with_transactions:

处理事务和DBAPI
========================================



有了:class:`_engine.Engine`对象，我们现在可以深入研究:class:`_engine.Engine`的基本操作以及其主要交互端点，:class:`_engine.Connection`和:class:`_engine.Result`。此外，我们还介绍了ORM的这些对象的外观，即:class:`_orm.Session`。

.. container:: orm-header

    **ORM读者注意**

    在使用ORM时，:class:`_engine.Engine`由另一个名为:class:`_orm.Session`的对象管理。现代SQLAlchemy中的:class:`_orm.Session`强调事务和SQL执行模式，这与本文中讨论的:class:`_engine.Connection`基本相同，因此，尽管本小节主要关注Core，但所有这里的概念对ORM使用都基本相关，建议所有ORM学习者都要学习此部分。:class:`_engine.Connection`使用的执行模式将在本节末尾与:class:`_orm.Session`进行对比。

既然我们还没有介绍SQLAlchemy表达语言（Expression Language），我们将在本软件包中使用一个名为:func:`_sql.text`的简单构造，它允许我们编写**文本SQL**语句。请放心，在日常SQLAlchemy使用中，文本SQL通常是例外而不是规则，尽管它始终完全可用。

.. rst-class:: core-header

.. _tutorial_getting_connection:

获取连接
---------------------

从用户角度来看，:class:`_engine.Engine`对象的唯一目的是提供名为:class:`_engine.Connection`的数据库连接单位。当直接使用Core时，:class:`_engine.Connection`对象是所有与数据库交互的交互方式。由于:class:`_engine.Connection`表示数据库上的打开资源，我们始终希望将此对象的使用范围限制在特定上下文中，并且最好的方法是使用Python上下文管理器形式，也称为“with语句”。下面我们用一个文本SQL语句说明“Hello World”，这里的文本SQL是使用一种称为:func:`_sql.text`的构造发出的，稍后将对其进行详细讨论：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import text

    >>> with engine.connect() as conn:
    ...     result = conn.execute(text("select 'hello world'"))
    ...     print(result.all())
    {execsql}BEGIN (implicit)
    select 'hello world'
    [...] ()
    {stop}[('hello world',)]
    {execsql}ROLLBACK{stop}

在上面的示例中，上下文管理器提供了一个数据库连接，并将操作框架置于事务内部。 Python DBAPI的默认行为包括始终处于进程中的事务。当连接的范围被释放时，将发出回滚以终止事务。事务不会**自动提交**；当我们要提交数据时，通常需要调用:meth:`_engine.Connection.commit`，我们将在下一节中看到。

.. tip:: 特殊情况下可用“autocommit”模式。:ref:`dbapi_autocommit`讨论了这一点。

SELECT的结果也是返回一个名为:class:`_engine.Result`的对象，稍后将对其进行讨论，但是，暂时我们将其添加为最好确保在“connect”块内消耗此对象，并且不要将它传递到连接范围之外的作用域。

.. rst-class:: core-header

.. _tutorial_committing_data:

提交更改
------------------

我们刚刚知道DBAPI连接不会自动提交。如果我们想提交一些数据怎么办呢？我们可以将上面的示例更改为创建表并插入一些数据，然后通过:meth:`_engine.Connection.commit`方法在获得:class:`_engine.Connection`对象的块**内**提交事务：

.. sourcecode:: pycon+sql

    # "在操作过程中进行提交"
    >>> with engine.connect() as conn:
    ...     conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    ...     conn.execute(
    ...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
    ...         [{"x": 1, "y": 1}, {"x": 2, "y": 4}],
    ...     )
    ...     conn.commit()
    {execsql}BEGIN (implicit)
    CREATE TABLE some_table (x int, y int)
    [...] ()
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    INSERT INTO some_table (x, y) VALUES (?, ?)
    [...] [(1, 1), (2, 4)]
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT

以上，我们发出了两个SQL语句，它们通常是事务的，"CREATE TABLE"语句[1]和参数化的“INSERT”语句（上面的参数化语法将在稍后的几节中讨论）。由于我们希望我们所做的工作在块内被提交，我们调用了:meth:`_engine.Connection.commit`方法，该方法提交了事务。在块内调用此方法后，我们可以继续运行更多的SQL语句，并且如果我们想，可以继续调用:meth:`_engine.Connection.commit`以进行后续语句的提交。 SQLAlchemy将这种样式称为**随时提交**。

还有一种提交数据的样式，即我们可以事先声明我们的“connect”块是一个事务块。这种操作模式下，我们使用:meth:`_engine.Engine.begin`方法获取连接，而不是:meth:`_engine.Engine.connect`方法。此方法既管理:class:`_engine.Connection`的范围，还将包含内部事务中的所有内容，在成功的块中以COMMIT结尾，或在异常引发时以ROLLBACK结尾。这种样式可以称为**始终开始**：

.. sourcecode:: pycon+sql

    # "始终开始"
    >>> with engine.begin() as conn:
    ...     conn.execute(
    ...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
    ...         [{"x": 6, "y": 8}, {"x": 9, "y": 10}],
    ...     )
    {execsql}BEGIN (implicit)
    INSERT INTO some_table (x, y) VALUES (?, ?)
    [...] [(6, 8), (9, 10)]
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT

“始终开始”样式经常被认为更为简洁，并且在一开始就指示了整个块的意图。然而，在本教程中，我们通常使用“随时提交”的样式，因为它更灵活，适用于演示目的。

.. 主题:: 什么是“BEGIN(implicit)”？

    您可能已经注意到了事务块的日志行“BEGIN(implicit)”。这里的“implicit”表示SQLAlchemy实际上**没有发送任何命令**到数据库；它只认为这是DBAPI的隐式事务的开头。您可以注册:ref:`event hooks <core_sql_events>`以截取此事件，例如。

.. [1] :term:`DDL`是指SQL的子集，此SQL指示数据库创建、修改或删除架构级别的结构，例如表。DDL，例如"CREATE TABLE"，应该在以COMMIT结束的事务块内，因为许多数据库使用事务DDL，以使架构更改在提交事务之前不会生效。但是，就像我们后面要看到的那样，我们通常让SQLAlchemy在高级别操作中运行DDL序列，其中我们通常不需要担心提交。


.. rst-class:: core-header

.. _tutorial_statement_execution:

语句执行的基础
-----------------------------

我们已经看到了一些在数据库上运行SQL语句的示例，使用:meth:`_engine.Connection.execute`方法，结合称为:func:`_sql.text`的对象，并返回名为:class:`_engine.Result`的对象。在本节中，我们将更密切地说明这些组件的机制和交互。

.. container:: orm-header

  本节的大多数内容同样适用于现代ORM的使用，当使用:meth:`_orm.Session.execute`方法时，工作方式非常类似于:meth:`_engine.Connection.execute`，包括ORM结果行使用与Core使用时相同的:class:`_engine.Result`接口。

.. rst-class:: orm-addin

.. _tutorial_fetching_rows:

获取行
^^^^^^^^^^^^^

我们首先通过使用我们先前插入的行并在我们创建的表上运行文本SELECT语句，更详细地说明了:class:`_engine.Result`对象：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(text("SELECT x, y FROM some_table"))
    ...     for row in result:
    ...         print(f"x: {row.x}  y: {row.y}")
    {execsql}BEGIN (implicit)
    SELECT x, y FROM some_table
    [...] ()
    {stop}x: 1  y: 1
    x: 2  y: 4
    x: 6  y: 8
    x: 9  y: 10
    {execsql}ROLLBACK{stop}

在上面的示例中，我们发出了一个选择语句，以选择我们表中的所有行。返回的对象称为:class:`_engine.Result`，表示结果行的可迭代对象。

:class:`_engine.Result`有许多用于获取和转换行的方法，例如之前演示过的:meth:`_engine.Result.all`方法，它返回所有:class:`_engine.Row`对象的列表。它还实现了Python迭代器接口，以便我们可以直接迭代:class:`_engine.Row`对象的集合。

:class:`_engine.Row`对象本身旨在像Python "命名元组"<https://docs.python.org/3/library/collections.html#collections.namedtuple>`_。下面我们通过多种方式访问行。

* **元组分配运算符**----这是最符合Python风格的样式，即按位置将变量分配到每一行：

  ::

      result = conn.execute(text("select x, y from some_table"))

      for x, y in result:
          ...

  ..

* **整数索引**----元组是Python序列，因此也可以正常使用整数访问：

  ::

      result = conn.execute(text("select x, y from some_table"))

      for row in result:
          x = row[0]

* **属性名称**----由于这些是Python命名元组，因此元组具有与每个列中的列名匹配的动态属性名称。这些名称通常是SQL语句分配给每行列的名称。虽然它们通常是相当可预测的，并且还可以通过标签进行控制，但在定义不太明确的情况下，它们可能受到数据库特定的行为影响::

      result = conn.execute(text("select x, y from some_table"))

      for row in result:
          y = row.y

          #采用Python格式化字符串使用
          print(f"Row: {row.x} {y}")

  ..

* **映射访问**----为了将行作为Python的**映射**对象接收，这本质上是Python与常见``dict``对象接口的只读版本，可以使用:meth:`_engine.Result.mappings`修改器将:class:`_engine.Result`**转换**为:class:`_engine.MappingResult`对象。对象返回类似于字典的:class:`_engine.RowMapping`对象而不是:class:`_engine.Row`对象:

      result = conn.execute(text("select x, y from some_table"))

      for dict_row in result.mappings():
          x = dict_row["x"]
          y = dict_row["y"]

  ..

.. rst-class:: orm-addin

.. _tutorial_sending_parameters:

发送参数
^^^^^^^^^^^^^^^^^^

SQL语句通常伴随着要用SQL语句传递的数据，就像我们上面在INSERT示例中看到的那样。因此:meth:`_engine.Connection.execute`方法也接受参数，称为:term:`bound parameters`。一个简单的例子可能是，如果我们想限制SELECT语句仅选择特定条件下的行，比如y值大于某个传递到函数中的值。

为了实现这一点，使SQL语句保持固定，并且驱动程序可以正确地对该值进行消毒，我们在语句中添加了一个名为"y"的WHERE条件，该条件命名了一个名为":y"的新参数; :func:`_sql.text`构造使用冒号格式":y"接受这些参数。实际的"：y"值以一个字典的形式传递给:meth:`_engine.Connection.execute`的第二个参数：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     result = conn.execute(text("SELECT x, y FROM some_table WHERE y > :y"), {"y": 2})
    ...     for row in result:
    ...         print(f"x: {row.x}  y: {row.y}")
    {execsql}BEGIN (implicit)
    SELECT x, y FROM some_table WHERE y > ?
    [...] (2,)
    {stop}x: 2  y: 4
    x: 6  y: 8
    x: 9  y: 10
    {execsql}ROLLBACK{stop}


在日志记录的SQL输出中，我们可以看到绑定参数“：y”在发送到SQLite数据库时已被转换为问号。这是因为SQLite数据库驱动程序使用名为“qmark parameter style”的格式，这是DBAPI规范允许的六种不同格式之一。SQLAlchemy将这些格式抽象为使用冒号的“named”格式。

.. 主题:: 始终使用绑定参数

    如本节开头所述，文本SQL通常不是我们使用SQLAlchemy的常规方式。但是，如果使用文本SQL，Python文本值，即使是整数或日期之类的非字符串，也**不应直接作为SQL字符串**进行字符串化;应该使用参数。这是最著名的避免SQL注入攻击的方法，但是它还允许SQLAlchemy方言和/或DBAPI正确处理后端输入。在平面文本SQL使用情况之外，SQLAlchemy的Core表达式API使用适当的方式将Python文本值传递为绑定参数。

.. _tutorial_multiple_parameters:

发送多个参数
^^^^^^^^^^^^^^^^^^^^^^^^^^^

在:ref:`tutorial_committing_data`中的示例中，我们执行了一个INSERT语句，似乎能够一次将多个行插入到数据库中。对于“DML”语句，例如“INSERT”，“ UPDATE”和“ DELETE”，我们可以通过传递字典的列表而不是单个字典来向:meth:`_engine.Connection.execute`方法发送**多个参数集**，这表明单个SQL语句应该针对每个参数集执行一次。这种执行方式称为:term:`executemany`：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     conn.execute(
    ...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
    ...         [{"x": 11, "y": 12}, {"x": 13, "y": 14}],
    ...     )
    ...     conn.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO some_table (x, y) VALUES (?, ?)
    [...] [(11, 12), (13, 14)]
    <sqlalchemy.engine.cursor.CursorResult object at 0x...>
    COMMIT

上面的操作等效于为每个参数集运行给定的一次INSERT语句，除了该操作将针对许多行进行优化以获得更好的性能。

“执行”和“executemany”之间的关键行为差异在于后者不支持返回结果行，即使语句包括RETURNING子句也不支持。这个例外情况是当使用Core :func:`_sql.insert`构造时引入的，稍后在:ref:`tutorial_core_insert`中介绍，其中还使用:meth:`_sql.Insert.returning`方法指示RETURNING。在这种情况下，SQLAlchemy利用特殊逻辑重组INSERT语句，使其能够在支持RETURNING的情况下为多行调用。


.. seealso::

   :term:`executemany`----在:doc:`<Glossary>中描述了，它描述的是大部分“executemany”执行所使用的DBAPI级别的`cursor.executemany() <https://peps.python.org/pep-0249/#executemany>`_方法。

   :ref:`engine_insertmanyvalues`----在:ref:`connections_toplevel`中，描述了用于使用:meth:`_sql.Insert.returning`提供结果集的“executemany”执行的专门逻辑。


.. rst-class:: orm-header

.. _tutorial_executing_orm_session:

使用ORM会话执行
-----------------------------

如前所述，大多数模式和示例适用于与ORM一起使用，因此在此处介绍此使用，以便在随后的示例中可以用Core和ORM使用每个模式进行说明。

使用ORM时的基本事务性/数据库交互对象是称为:class:`_orm.Session`的对象。在现代SQLAlchemy中，此对象的使用方式非常类似于:class:`_engine.Connection`，实际上，在使用:class:`_orm.Session`时，它在内部引用:class:`_engine.Connection`，用于发出SQL。

当对非ORM构造使用:class:`_orm.Session`时，它通过我们给它传递的SQL语句并不会实际做什么，因此返回结果与使用:class:`_engine.Connection`直接 执行的结果基本相同，因此在这里我们将它们用简单的文本SQL操作演示。

:class:`_orm.Session`具有几种不同的创建模式，但是在此处我们将说明与使用:class:`_engine.Connection`的方式完全相同的最基本模式，即将其构建在上下文管理器内：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import Session

    >>> stmt = text("SELECT x, y FROM some_table WHERE y > :y ORDER BY x, y")
    >>> with Session(engine) as session:
    ...     result = session.execute(stmt, {"y": 6})
    ...     for row in result:
    ...         print(f"x: {row.x}  y: {row.y}")
    {execsql}BEGIN (implicit)
    SELECT x, y FROM some_table WHERE y > ? ORDER BY x, y
    [...] (6,){stop}
    x: 6  y: 8
    x: 9  y: 10
    x: 11  y: 12
    x: 13  y: 14
    {execsql}ROLLBACK{stop}

上面的示例可以与在:ref:`tutorial_sending_parameters`中的示例进行比较——我们直接将调用``with engine.connect() as conn``替换为``with Session(engine) as session``，然后使用:meth:`_orm.Session.execute`方法，就像使用:meth:`_engine.Connection.execute`方法一样。

另外，与:class:`_engine.Connection`一样，:class:`_orm.Session`还使用:meth:`_orm.Session.commit`方法实现了“随时提交”行为，下面我们使用一个文本UPDATE语句更改了一些数据：

.. sourcecode:: pycon+sql

    >>> with Session(engine) as session:
    ...     result = session.execute(
    ...         text("UPDATE some_table SET y=:y WHERE x=:x"),
    ...         [{"x": 9, "y": 11}, {"x": 13, "y": 15}],
    ...     )
    ...     session.commit()
    {execsql}BEGIN (implicit)
    UPDATE some_table SET y=? WHERE x=?
    [...] [(11, 9), (15, 13)]
    COMMIT{stop}

上面，我们使用:ref:`tutorial_multiple_parameters`方式执行了一个UPDATE语句，并以“随时提交”提交了块。

.. tip:: :class:`_orm.Session`实际上并未在结束事务后保留:class:`_engine.Connection`对象。在下一次需要执行SQL语句时，它将从:class:`_engine.Engine`中获得新的:class:`_engine.Connection`对象。

显然，:class:`_orm.Session`有很多技巧，但是了解它具有一个:meth:`_orm.Session.execute`方法，该方法的使用方式与:meth:`_engine.Connection.execute`方法相同，将为我们使用后面的示例获取良好的启动。


.. seealso::

    :ref:`session_basics`----呈现了与:class:`_orm.Session`对象的基本创建和用法方式。