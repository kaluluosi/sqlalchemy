.. |prev| replace::  :doc:`engine` 
.. |next| replace::  :doc:`metadata` 

.. include:: tutorial_nav_include.rst


.. _tutorial_working_with_transactions:

使用事务和DBAPI
========================================



现在我们已经准备好使用  :class:`_engine.Engine` 。此外，我们还将介绍ORM的这些对象的外观，称为  :class:` _orm.Session` 。

.. container:: orm-header

    **适用于ORM读者的注意事项**

    当使用ORM时， :class:`_engine.Engine` 由称为 :class:`_orm.Session` 的另一个对象管理。现代SQLAlchemy中的 :class:`_orm.Session` 强调了一种事务性和SQL执行模式，这在很大程度上与下文中讨论的 :class:`_engine.Connection` 相同，因此，虽然这个子部分是针对Core的，但是这里的所有概念基本上也与ORM使用相关，并建议所有ORM学习者使用。下面将对 :class:`_engine.Connection` 使用的执行模式与 :class:`_orm.Session` 进行对比。

由于我们还没有介绍SQLAlchemy表达式语言是SQLAlchemy的主要特性，因此我们将使用这个软件包中的一种简单构造，称为 :func:`_sql.text` 构造，该构造允许我们将SQL语句编写为文本SQL。请放心，在日常SQLAlchemy使用中，文本SQL往往是大多数任务的例外，尽管它始终完全可用。

.. rst-class:: core-header

.. _tutorial_getting_connection:

获取连接
---------------------

从用户界面的角度看， :class:`_engine.Engine` 对象的唯一目的是提供称为 :class:`_engine.Connection` 的数据库连接单位。当直接处理Core时， :class:`_engine.Connection` 对象是与数据库交互的全部操作。由于 :class:`_engine.Connection` 表示与数据库打开的资源，因此我们希望始终将此对象的使用范围限制为特定上下文，并且最好使用Python上下文管理器格式，也称为“with语句”。下面我们使用一个文本SQL语句说明“Hello World”。稍后将讨论名为 :func:`_sql.text` 的文本SQL构造的更多细节：

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

在上面的示例中，上下文管理器提供了一个数据库连接，并将操作放置在事务内。Python DBAPI的默认行为包括始终存在事务；当连接的范围为“released”时，将发出ROLLBACK来结束事务。这个事务**不会自动提交**。我们通常需要调用  :meth:`_engine.Connection.commit`  来提交数据，如下一节所示。

.. tip::  对于特殊情况，可以使用“autocommit”模式。本文后面将会讨论。

我们的SELECT的结果也返回一个称为 :class:`_engine.Result` 的对象，稍后我们将讨论它，但是暂时添加的最好是确保在“connect”块内使用此对象，并且不将其传递到连接范围之外。

.. rst-class:: core-header

.. _tutorial_committing_data:

提交更改
------------------

我们刚刚学习到，DBAPI连接不会自动提交。如果我们想要提交一些数据怎么办？可以更改我们上面的示例以创建一个表并插入一些数据，随后使用  :meth:`_engine.Connection.commit`  方法在我们获取 :class:` _engine.Connection`对象的块**内**提交事务：

.. sourcecode:: pycon+sql

    # "commit as you go"
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

在上面的代码中，我们发出了两个通常是事务性的 SQL 语句，一个是 "CREATE TABLE" 语句 [1]_，另一个是参数化的 "INSERT" 语句（参数化的语法在下面的   :ref:`tutorial_multiple_parameters`  节中会进一步讨论）。我们执行  :meth:` _engine.Connection.commit`  方法，将我们在该块内的操作提交，以便事务生效。当我们在代码块内调用此方法之后，可以继续运行更多 SQL 语句，并且如果我们选择了，可以在随后的语句中再次调用  :meth:`_engine.Connection.commit` 。SQLAlchemy 将这种风格称之为 **逐步提交**。

另一种提交数据的方式是，在事先声明我们的 "connect" 代码块是事务块的情况下。为了实现该操作模式，我们使用  :meth:`_engine.Engine.begin`  方法来获取连接，而不是使用  :meth:` _engine.Engine.connect`  的范围，并将其封装到一个事务中，以确保最终提交数据（如果该块成功）或回滚（如果抛出异常）。这种风格可以被称之为 **一次性开启事务**：

.. sourcecode:: pycon+sql

    # "一次性开启事务"
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

一次性开启事务的风格通常更简洁，并在一开始就指示整个块的意图。然而，在本教程中，我们通常使用 "逐步提交" 的方式，因为这种方式更灵活，便于演示。

.. topic:: 什么是 "BEGIN (implicit)"？

    你可能注意到日志中在事务块开头处出现了 "BEGIN (implicit)" 字样。这里的 "implicit" 表示 SQLAlchemy 实际上**没有向数据库发送任何指令**；它只是认为这是 DBAPI 隐式事务的开始。你可以注册   :ref:`事件回调函数 <core_sql_events>`  来拦截此事件，进行进一步处理。

.. [1] DDL 是指 SQL 子集，该子集指示数据库创建、修改或删除表等模式层面的结构。DDL（例如 "CREATE TABLE" ）建议在事务块内运行并以 COMMIT 结束，因为许多数据库使用事务 DDL 使得模式更改在事务提交之前不会生效。但是，正如稍后将要看到的，我们通常让 SQLAlchemy 在更高级别操作中运行 DDL 序列，我们通常不需要担心 COMMIT。


.. rst-class:: core-header

.. _tutorial_statement_execution:

语句执行的基本知识
----------------------------------------

我们已经看到了一些示例，演示了如何使用称为  :meth:`_engine.Connection.execute`  的方法远程运行 SQL 语句，与称为   :func:` _sql.text`  的其他对象一起使用，并返回名为   :class:`_engine.Result`  的对象。在本节中，我们将更加详细地说明这些组件的机制和相互作用。

.. container:: orm-header

  当使用  :meth:`_orm.Session.execute`  方法时，大部分此节的内容同样适用于现代 ORM 的使用方式，该方法与  :meth:` _engine.Connection.execute`  的工作方式非常相似，包括 ORM 结果行是使用同样的   :class:`_engine.Result`  接口，被 Core 使用。

.. rst-class:: orm-addin

.. _tutorial_fetching_rows:

提取行数据
^^^^^^^^^^^^^^^^

我们首先将通过运行文本 SELECT语句，从我们的刚刚创建的表中选取数据，来具体演示   :class:`_engine.Result`  对象。

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

在上述代码中，我们执行的 "SELECT" 语句是选取表中的所有行。返回的对象称为  :class:`_engine.Result` ，代表结果行的可迭代对象。

  :class:`_engine.Result`  方法，返回所有 :class:` _engine.Row`对象的列表。它还实现了Python迭代器接口，使我们可以直接遍历 :class:`_engine.Row` 对象的集合。

 :class:`_engine.Row` 对象本身旨在像Python`命名元组<https://docs.python.org/zh-cn/3/library/collections.html#collections.namedtuple>`_一样。下面我们举例说明了各种访问行的方法。

* **元组赋值** - 这是最Python化的风格，即根据接收顺序将变量分配给每个行位置：

  ::

      result = conn.execute(text("select x, y from some_table"))

      for x, y in result:
          ...

* **整数索引** - 元组是Python序列，因此也可以进行常规的整数访问：

  ::

      result = conn.execute(text("select x, y from some_table"))

      for row in result:
          x = row[0]

* **属性名称** - 因为这些是Python命名元组，元组具有与每个列名称匹配的动态属性名称。这些名称通常是SQL语句为每行分配到的列名。虽然它们通常是可以预测的，并且还可以通过标签进行控制，但在更少定义的情况下，它们可能受特定于数据库的行为的影响：

      result = conn.execute(text("select x, y from some_table"))

      for row in result:
          y = row.y

          # 说明如何使用Python的f-strings
          print(f"Row: {row.x} {y}")

  ..

* **映射访问** - 要将行接收为Python**映射**对象，这实际上是Python的接口的只读版本与常见的“dict”对象，  :class:`_engine.结果`  修饰符将变换成一个 :class:` _engine.MappingResult`对象；这是一个结果对象，它生成类似于字典的 :class:`_engine.RowMapping` 对象而不是 :class:`_engine.Row` 对象::

      result = conn.execute(text("select x, y from some_table"))

      for dict_row in result.mappings():
          x = dict_row["x"]
          y = dict_row["y"]

  ..

.. rst-class:: orm-addin

.. _tutorial_sending_parameters:

发送参数
^^^^^^^^^^^^^^^^^^

SQL语句通常由要与语句本身一起传递的数据附件，如我们之前在INSERT示例中看到的一样。因此  :meth:`_engine.连接`  方法也接受参数，这些参数被称为  :term:` bound parameters`  。一个初步的示例可能是，如果我们只想将SELECT语句限制为仅符合某些条件的行，例如仅当“y”值大于传递给函数的某个值时，我们添加名为“y”的新参数的WHERE条件到我们的语句中；  :func:`_sql.text` ` :y``”接受这些条件。然后将“``:y``”的实际值以字典的形式作为  :meth:`_engine.Connection.execute`  的第二个参数传递：

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

在已记录SQL输出中，可以看到绑定参数`:y`在发送到SQLite数据库时被转换为问号。这是因为SQLite数据库驱动程序使用了称为"问号参数样式"的格式，这是DBAPI规范允许的六种不同格式之一。SQLAlchemy将这些格式抽象为仅使用冒号的“命名”格式。

.. topic:: 始终使用绑定参数

    如本节开头所述，文本SQL不是我们使用SQLAlchemy的常规方式。然而，当使用文本SQL时，Python字面值，甚至是非字符串，如整数或日期，应**永远不会直接将其字符串化为SQL字符串**；应该**始终使用参数**。这最为人所知的是避免SQL注入攻击的方法，当数据不受信任时。但是，它也允许SQLAlchemy方言和/或DBAPI正确地处理后端传入的输入。除了纯文本SQL用例外，SQLAlchemy的核心表达式API还确保Python字面值在适当的位置以绑定参数的形式传递。

.. _tutorial_multiple_parameters:

发送多个参数
^^^^^^^^^^^^^^^^^^^^^^^^^^^

在   :ref:`tutorial_committing_data`  中的例子中，我们执行了一个INSERT语句，其中看起来我们能够一次向数据库插入多个行。对于 "INSERT"、"UPDATE" 和 "DELETE" 等 DML 语句，我们可以通过传递一个字典列表而不是单个字典来向  :meth:` _engine.Connection.execute`  方法发送**多个参数集**，这表明单个SQL语句应该被调用多次，每次针对一个参数集。这种执行风格称为  :term:`executemany` ：

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

上述操作相当于针对每个参数集运行给定的INSERT语句一次，但该操作将针对许多行进行优化以提高性能。

"execute" 和 "executemany" 之间的一个关键行为差异在于，后者不支持返回结果行，即使语句包含 RETURNING 子句。这个规则有一个例外情况，就是当使用 Core 的   :func:`_sql.insert`  构造函数时，也就是后面将在本教程的   :ref:` tutorial_core_insert`  中介绍时，它还使用了  :meth:`_sql.Insert.returning`  方法来表示 RETURNING。在这种情况下，SQLAlchemy 使用特殊逻辑重新组织 INSERT 语句，以便针对多行调用 INSERT 语句同时仍支持 RETURNING。

.. seealso::

    :term:`executemany`  - 在  :doc:` Glossary </glossary>`  中，描述了用于大多数 "executemany" 执行的 DBAPI 级
   `cursor.executemany() <https://peps.python.org/pep-0249/#executemany>`_ 方法。

     :ref:`engine_insertmanyvalues`  - 在   :ref:` connections_toplevel`  中，描述了  :meth:`_sql.Insert.returning`  所使用的专门逻辑，以便在使用 "executemany" 执行时提供结果集。

.. rst-class:: orm-header

.. _tutorial_executing_orm_session:

使用 ORM Session 执行
-----------------------------

正如先前提到的那样，上面大部分的模式和例子也适用于使用 ORM，所以在这里我们将介绍这种用法，以便在本教程继续时，我们可以在 Core 和 ORM 中使用每种模式和技术。

在使用 ORM 时，基本的事务/数据库交互对象称为   :class:`_orm.Session` 。在现代 SQLAlchemy 中，这个对象被用在与   :class:` _engine.Connection`  非常相似的方式中。事实上，当   :class:`_orm.Session`  被使用时，它在内部引用了一个   :class:` _engine.Connection` ，它用于发出 SQL。

当使用   :class:`_orm.Session`  与非 ORM 构造函数一起使用时，它会传递我们给它的 SQL 语句，并且一般不会做与   :class:` _engine.Connection`  直接相比有太大的不同，因此我们可以在这里使用简单的文本 SQL 操作来进行说明。

  :class:`_orm.Session`  有几种不同的创建模式，但这里我们将介绍最基本的一种模式，该模式与使用   :class:` _engine.Connection`  的方式完全相同，即在上下文管理器中构造它：

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

上面的例子可以与前面   :ref:`tutorial_sending_parameters`  中的例子进行比较 - 我们直接将对 ` `with engine.connect() as conn`` 的调用替换为 ``with Session(engine) as session``，然后像使用  :meth:`_engine.Connection.execute`  方法一样使用  :meth:` _orm.Session.execute`  方法。另外，与  :class:`_engine.Connection`  方法实现“即时提交”的行为，如下例所示，使用文本更新语句改变了一些数据：

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

在上面的示例中，我们使用了绑定参数“executemany”风格的方法调用了一个UPDATE语句，结束整个块时使用了“即时提交”的方法。

.. tip:: 事实上， :class:`_orm.Session` 在事务结束后并不会保持 :class:`_engine.Connection` 对象。
   需要在下一次使用SQL查询数据库时，它会从 :class:`_engine.Engine` 重新获得一个 :class:`_engine.Connection` 对象。

  :class:`_orm.Session`  方法与  :meth:` _engine.Connection.execute`  方法的使用方式，即可开始后续的示例。

.. seealso::

      :ref:`session_basics`  - 介绍了 :class:` _orm.Session`对象的基本创建和使用模式。