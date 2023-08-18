.. _errors:

==============
错误消息
==============

这个部分列出了常见的SQLAlchemy抛出或发出的错误消息和警告的描述和背景。

SQLAlchemy通常会在SQLAlchemy特定的异常类上下文中引发错误。有关这些类的详细信息，请参见 :ref:`core_exceptions_toplevel` 和 :ref:`orm_exceptions_toplevel`。

SQLAlchemy错误可以大致分为两类，即 **编程时间错误** 和 **运行时错误**。编程时间错误是由于使用错误的参数调用函数或方法或来自无法解析的其他配置导向方法，如映射器配置而引发的。编程时间错误通常是即时的和确定性的。另一方面，运行时错误表示程序在响应发生在任意情况下的某些条件时发生的失败，例如数据库连接被用尽或发生某些与数据相关的问题。运行时错误更可能在正在运行的应用程序的日志中看到，因为程序在遇到这些状态时会响应加载和遇到的数据。

由于运行时错误不容易重现，并且经常发生在程序运行时，以响应某些任意条件，例如数据库连接被耗尽或出现某些数据相关问题，因此它们更难于调试，并且还会影响已经投入生产的程序。

在本节中，目标是尝试提供有关一些常见的运行时错误以及编程时间错误的背景。

连接和事务
----------------------------

.. _error_3o7r:

达到size <x>的QueuePool限制，连接已超时，超时时间 <z>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这可能是经验最丰富的常见的运行时错误，因为它直接涉及应用程序的工作负载，对几乎所有SQLAlchemy应用程序都适用的配置限制，这是一种直接应用的负载。

以下几点总结了此错误的含义，从大多数SQLAlchemy用户应已熟悉的最基本的点开始。

* **SQLAlchemy引擎对象默认使用连接池** - 这意味着当一个 :class:`_engine.Engine` 对象的SQL数据库连接资源被使用，并且释放了这个资源之后，数据库连接本身仍然连到数据库，并且返回到内部队列中，可以重新使用它。即使代码似乎正在与数据库结束对话，在许多情况下，应用程序仍将保留一定数量的数据库连接，这些连接会持续，直到应用程序结束或显示丢弃池。

* 由于连接池，当一个应用程序使用SQL数据库连接时，大多数情况下从 :meth:`_engine.Engine.connect` 中返回，或在使用ORM :class:`.Session` 时进行查询，这些操作并不一定在获取连接对象时立即建立新的数据库连接；相反，它会查询连接池以获取连接，它通常会从池中检索一个现有连接以重新使用。如果没有可用的连接，则连接池将创建一个新的数据库连接，但仅当池未超过配置容量时。

* 在大多数情况下使用的默认池称为 :class:`.QueuePool`。当要求此池提供连接但没有可用连接时，如果正在播放的**总连接数小于配置的值**，它将创建一个新连接。该值等于**池的大小加上max overflow大小**。这意味着如果您已经配置了引擎作为：

   engine = create_engine("mysql+mysqldb://u:p@host/db", pool_size = 10, max_overflow = 20)

   上述 :class:`_engine.Engine` 将允许在任何时间最多30个连接，不包括从引擎分离或失效的连接。如果到达新连接的请求并且已经有30个连接被其他应用程序的其他部分使用，则连接池将阻塞一段固定的时间，然后超时并引发此错误消息。

   为了允许同时使用更多的连接，可以使用 :paramref:`_sa.create_engine.pool_size` 和 :paramref:`_sa.create_engine.max_overflow` 传递给  :func:`_sa.create_engine` 函数的参数来调整池。等待连接可用的超时时间是使用 :paramref:`_sa.create_engine.pool_timeout` 参数配置的。

* 可以通过将 :paramref:`_sa.create_engine.max_overflow` 设置为值“-1”来配置池具有无限制的溢出。使用此设置时，池仍将维护固定数量的连接池，但如果没有可用连接，则不会无条件地阻止请求新连接。

  但是，以这种方式运行时，如果应用程序存在使用所有可用连接可用性资源的问题，则最终会达到数据库本身的可用连接限制，从而再次返回错误。更为严重的是，当应用程序耗尽数据库的连接时，通常会使用大量的资源，并且可能会干扰依赖于能够连接到数据库的其他应用程序和数据库状态机制。

  鉴于上述原因，连接池可以看作是连接使用的 **安全阀**，为防止恶意应用程序导致整个数据库对所有其他应用程序不可用，从而提供了关键的保护层。当收到此错误消息时，最好使用使用过多连接的问题进行修复和/或适当地配置限制，而不是允许无限制的溢出，因为这并不实际解决潜在的问题。

是什么导致应用程序使用完所有可用连接？

* **应用程序正在处理太多并发请求以基于池的配置做工作** - 这是最直接的原因。如果您有一个运行于允许30个并发线程的线程池中的应用程序，并且每个线程使用一个连接，在不允许同时检出至少30个连接的情况下，一旦您的应用程序接收到足够的并发请求，您将获得此错误。解决方案是提高池的限制或降低并发线程的数量。

* **应用程序没有将连接返回到池中** - 这是下一个最常见的原因，即应用程序正在使用连接池，但该程序正在未能 :term:`释放` 这些连接，而是将它们保持打开状态。无论ORM会话和/或连接对象何时被垃圾回收，它会导致底层连接资源被释放，但是在及时释放资源方面，无法依赖这种行为。

  这种情况可能会发生的一个常见原因是，应用程序使用ORM会话并在完成涉及该会话的工作之后没有调用 :meth:`.Session.close`。应该确保ORM会话（如果使用ORM）还是绑定到引擎的 :class:`_engine.Connection` 对象（如果使用Core），在完成工作的结束时被明确关闭，通过适当的 ``.close()`` 方法或使用可用上下文管理器（例如“with:”语句）之一来释放资源。

* **应用程序正试图运行长时间运行的事务** - 数据库事务是非常昂贵的资源，并且应该**永远不要闲置等待某些事件发生**。如果应用程序正在等待用户按按钮或等待长时间运行作业队列的结果，或者正在持久保持连接到浏览器，则**不要将数据库事务保持打开状态**。由于应用程序需要与数据库交互并与事件交互，因此在该点打开短暂的事务然后关闭它。

* **应用程序死锁** -这也是产生此错误的常见原因，而且更难以理解，如果应用程序无法完成对连接的使用，无论是由于应用程序端还是由于数据库端的死锁，应用程序可以使用所有可用连接，这然后导致额外的请求接收到此错误。死锁的原因包括：

  * 使用隐式异步系统（如gevent或eventlet）而没有正确的 monkeypatching 所有场景库和驱动程序，或者有错误，在不完全覆盖所有monkeypatched驱动程序方法方面，或者在使用异步系统进行对CPU绑定工作负载的情况下，使用数据库资源的greenlets简单地等待太长时间而无法对其进行处理。对于绝大多数关系型数据库操作，隐式或显式异步编程框架通常是不必要的或不适用的。如果应用程序必须在某些功能区域中使用异步系统，则最好运行数据库导向的业务方法在传统线程中运行，该线程向应用程序的异步部分传递消息。

  * 数据库死锁，例如行彼此死锁

  * 线程错误，例如互斥体在互斥死锁中，或在同一线程中调用已锁定的互斥体

请记住，使用连接池的替代方法是完全关闭池。有关此问题的背景，请参见 :ref:`pool_switching` 部分。但是，请注意，当此错误消息发生时，它始终是由应用程序本身中的更大问题造成的；池只有在早期阶段揭示了该问题。

.. seealso::

 :ref:`pooling_toplevel`

 :ref:`connections_toplevel`


.. _error_8s2b:

在继续之前, 请将无效事务完全回滚。请回滚()
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此错误情况指的是 :class:`_engine.Connection` 已失效，由于检测到数据库断开连接或由于显式调用了 :meth:`_engine.Connection.invalidate`，但仍然存在一个事务，该事务由 :meth:`_engine.Connection.begin` 方法明确或由于连接在发出任何SQL语句时自动开始，而处于“未完成”状态。连接无效专为不再使用的一次性使用而设计，并用于避免该语言之前的历史中可能存在的问题，它通常伴随有较多消耗的项目。

.. _error_dbapi:

DBAPI错误
------------

Python数据库API或DBAPI是数据库驱动程序的规范，可在 `Pep 249 <https://www.python.org/dev/peps/pep-0249/>`_ 上找到。此API指定了一组异常类，其能够适应数据库的所有故障模式。

SQLAlchemy不会直接生成这些异常。相反，它们被从数据库驱动程序拦截并由SQLAlchemy提供的异常 :class:`.DBAPIError` 进行包装，但是异常消息是 **由驱动程序生成的，而不是SQLAlchemy** 。

.. _error_rvf5:

InterfaceError
~~~~~~~~~~~~~~

与数据库本身相关，而非数据库接口的错误引发的异常。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

“InterfaceError”有时由驱动程序在上下文中引发，而数据库连接被放弃，或无法连接到数据库。有关如何处理此问题的提示，请参见 :ref:`pool_disconnects`。

.. _error_4xp6:

DatabaseError
~~~~~~~~~~~~~

与数据库本身相关而不是与传递的接口或数据相关的错误引发的异常。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

.. _error_9h9h:

DataError
~~~~~~~~~

与处理的数据存在问题（例如除以零，数字值超出范围等）相关的错误引发的异常。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

.. _error_e3q8:

OperationalError
~~~~~~~~~~~~~~~~

与数据库操作相关，而不一定受程序员控制的错误引发的异常，例如。出现意外断开连接，未找到数据源名称，无法处理事务，处理过程中发生内存分配错误等。。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

“OperationalError”是由许多驱动程序（但不是唯一的）在上下文中使用了丢失的数据库连接或无法连接到数据库时使用的错误类别。有关如何处理此问题的提示，请参见 :ref:`pool_disconnects`。

.. _error_gkpj:

IntegrityError
~~~~~~~~~~~~~~

与数据库的关系完整性受到影响，例如外键检查失败时引发的异常。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

.. _error_2j85:

InternalError
~~~~~~~~~~~~~

数据库遇到内部错误时引发的异常，例如，光标不再有效，事务不同步等。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

“InternalError”有时由驱动程序在上下文中引发，而数据库连接被放弃，或无法连接到数据库。有关如何处理此问题的提示，请参见 :ref:`pool_disconnects`。

.. _error_f405:

ProgrammingError
~~~~~~~~~~~~~~~~

与编程错误相关的异常，例如，未找到表或表已存在，SQL语句中的语法错误，指定的参数数量不正确等。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

“ProgrammingError”有时由驱动程序在上下文中引发，而数据库连接被放弃，或无法连接到数据库。有关如何处理此问题的提示，请参见 :ref:`pool_disconnects`。

.. _error_tw8g:

NotSupportedError
~~~~~~~~~~~~~~~~~

在使用不支持的方法或数据库API的情况下引发异常，例如，在不支持事务或关闭事务的连接上请求.rollback()。

此错误是一个 :ref:`DBAPI Error <error_dbapi>`，起源于数据库驱动程序（DBAPI），而非SQLAlchemy本身。

SQL表达式语言
-----------------------
.. _error_cprf:
.. _caching_caveats:

Object will not produce a cache key, Performance Implications
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

从版本1.4开始，SQLAlchemy包括一个 :ref:`SQL compilation caching facility <sql_caching>` ，可以使Core和ORM SQL结构在缓存其字符化形式时，包括用于从语句中检索结果的其他结构信息，允许在下一次使用完全相等结构的构造时跳过相对昂贵的字符串编译过程。该系统依赖于对所有SQL构造实现的功能，包括对象 :class:`_schema.Column`、:func:`_sql.select` 和 :class:`_types.TypeEngine`，以产生一个 **缓存密钥** 以完全表示其状态，因为它影响SQL编译过程。

如果这些警告涉及到诸如 :class:`_schema.Column` 之类的广泛使用对象，并且显示影响到发出的大多数SQL结构（使用 :ref:`sql_caching_logging` 中描述的估计技术），以至于在应用程序中通常不启用缓存，这会对性能产生负面影响，并且在某些情况下，实际上会与先前的SQLAlchemy版本相比产生 **性能降级**。 :ref:`faq_new_caching` 在附加详细信息中涵盖了此内容。

缓存会自行禁用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

缓存依赖于能够生成准确代表语句的完整结构的缓存密钥，这些密钥必须在 **一致** 的方式上符合该语句的结构。如果某个特定的SQL构造（或类型）没有必要的指令，允许它生成适当的缓存密钥，则无法安全地启用缓存。

* 缓存密钥必须表示 **完整的结构**：如果使用两个独立实例的构造可能导致不同的SQL被呈现，则在使用缓存密钥时缓存第一个实例时，使用未能捕获第一个实例和第二个实例之间不同的差异之间的区别的缓存密钥将导致不正确的缓存SQL 字符串和对第二个实例呈现的缓存的使用。

* 缓存密钥必须是 **一致的**：例如，如果构造表示每个时间都会更改的状态，例如文字值，则对于该构造重复使用唯一的SQL，这样每个实例都会产生唯一的SQL， 对于相同的SQL结构，只有在实际上进行再次编译所需的数目可能很快填充语句缓存。

出于上述两个原因，SQLAlchemy的缓存系统在决定是否缓存与特定后端特定的构造相关的SQL时非常“谨慎”。

缓存引用属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

警告是根据以下标准发出的。有关每个标准的进一步详细信息，请参见章节 :ref:`faq_new_caching`。

* 本身的 :class:`.Dialect`（即我们传递给 :func:`_sa.create_engine` 的URL的第一部分指定的模块）必须表明它已审查并经过测试以正确支持缓存，这由 :attr:`.Dialect.supports_statement_cache` 属性设置为True表示。当使用第三方方言时，请咨询该方言的维护者，以便他们可以遵循 :ref:`steps to ensure caching may be enabled<engine_thirdparty_caching>` 中的步骤，并发布新版本。

* 继承自 :class:`.TypeDecorator` 或 :class:`.UserDefinedType` 的第三方或用户定义类型必须包括其定义中的 :attr:`.ExternalType.cache_ok` 属性，包括所有衍生子类，遵循 :attr:`.ExternalType.cache_ok` 的文档字符串所描述的准则。与之前一样，如果这些数据类型是从第三方库导入的，请咨询该库的维护者，以便他们可以提供其库的必要更改并发布新版本。

* 继承自 :class:`_expression.ClauseElement`、 :class:`_schema.Column`、 :class:`_dml.Insert` 等类的第三方或用户定义的SQL构造，包括简单子类以及设计用于与 :ref:`sqlalchemy.ext.compiler_toplevel` 一起使用的构造，应通常包括 :attr:`.HasCacheKey.inherit_cache` 属性， set to True 或 False，根据构造的设计，在 :ref:`compilerext_caching` 中描述的准则，应包括具有缓存密钥的原始实例的缓存密钥。

.. seealso::

    :ref:`sql_caching_logging` - 背景观察缓存行为和效率

    :ref:`faq_new_caching` - 在 :ref:`faq_toplevel` 部分中


.. _error_l7de:

Compiler StrSQLCompiler can't render element of type <element type>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在将 SQL 表达式构造串行化出现元素不在默认的编译中时，通常会出现此错误；在这种情况下，将出现对 :class:`.StrSQLCompiler` 类的错误.在不太常见的情况下，当错误类型的SQL表达式与特定类型的数据库后端一起使用时，也会出现该错误；在这些情况下，将命名其他SQL编译器类，例如 ``SQLCompiler`` 或 ``sqlalchemy.dialects.postgresql.PGCompiler``。以下是更特定于“字符化”用例但描述了一般背景的指导方针。

通常，Core SQL构造或ORM :class:`_query.Query`对象可以直接串行化，如使用 ``print（）`` 时：

.. sourcecode:: pycon+sql

  >>> from sqlalchemy import column
  >>> print(column("x") == 5)
  {printsql}x = :x_1

从上面的SQL表达式字符串是字符串化，使用的是一个 :class:`.StrSQLCompiler` 编译器类，它是一个特殊的语句编译器，当一个构造没有任何特定于方言的信息就被字符串化时被调用。

但是，有许多构造是特定于某个特定类型的数据库方言的，例如 PostgreSQL
“Insert on conflict” construct: ：

  >>> from sqlalchemy.dialects.postgresql import insert
  >>> from sqlalchemy import table, column
  >>> my_table = table("my_table", column("x"), column("y"))
  >>> insert_stmt = insert(my_table).values(x="foo")
  >>> insert_stmt = insert_stmt.on_conflict_do_nothing(index_elements=["y"])
  >>> print(insert_stmt)
  Traceback (most recent call last):

  ...

  sqlalchemy.exc.UnsupportedCompilationError:
  Compiler <sqlalchemy.sql.compiler.StrSQLCompiler object at 0x7f04fc17e320>
  can't render element of type
  <class 'sqlalchemy.dialects.postgresql.dml.OnConflictDoNothing'>

为了将特定于某个特定的后端的构造串行化，必须使用 :meth:`_expression.ClauseElement.compile` 方法，传递 :class:`_engine.Engine` 或 :class:`.Dialect` 对象以调用正确的编译器。在以下示例中，我们使用了PostgreSQL方言：

.. sourcecode:: pycon+sql

  >>> from sqlalchemy.dialects import postgresql
  >>> print(insert_stmt.compile(dialect=postgresql.dialect()))
  {printsql}INSERT INTO my_table (x) VALUES (%(x)s) ON CONFLICT (y) DO NOTHING

对于ORM :class:`_query.Query`对象，语句可以使用引用 :attr:`~.orm.query.Query.statement` 的方式获得：

    statement = query.statement
    print(statement.compile(dialect=postgresql.dialect()))

有关有关直接字符串化/编译SQL元素的额外详细信息，请参见FAQ链接。

.. seealso::

  :ref:`faq_sql_expression_string`


TypeError: <操作符>不支持在“ColumnProperty”实例和<something>之间的实例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通常情况下，当尝试在SQL表达式上下文中使用 :func:`.column_property` 或 :func:`.deferred` 对象时，通常在声明性中出现此错误，例如：

    class Bar(Base):
        __tablename__ = "bar"

        id = Column(Integer, primary_key=True)
        cprop = deferred(Column(Integer))

        __table_args__ = (CheckConstraint(cprop > 5),)

上面的“cprop”属性在映射之前在行内使用，但是该“cprop”属性不是 :class:`_schema.Column`，它是一个 :class:`.ColumnProperty`，即一个中间对象，因此没有 :class:`_schema.Column` 对象或 :class:`.InstrumentedAttribute` 对象的全部功能映射到完成时将映射到“Bar”类。虽然 :class:`.ColumnProperty` 确实有 ``__clause_element __()`` 方法，允许它在一些以列为导向的上下文中工作，但是在上面所示的开放式比较上下文中是无法工作的，因为它没有 Python 的 ``__eq __（）`` 方法，允许它将与数字“5”的比较解释为SQL表达式而不是正常的Python比较。

解决方案是直接使用属性 :attr:`.ColumnProperty.expression` 访问 :class:`_schema.Column`  ：

    class Bar(Base):
        __tablename__ = "bar"

        id = Column(Integer, primary_key=True)
        cprop = deferred(Column(Integer))

        __table_args__ = (CheckConstraint(cprop.expression > 5),)

.. _error_cd3x:

在继续之前，必须为绑定参数 <x> 在参数组 <y> 中提供值
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在执行语句时，如果对 :func:`.bindparam` 利用隐式或显式并没有提供值，则会出现此错误::

    stmt = select(table.c.column).where(table.c.id == bindparam("my_param"))

    result = conn.execute(stmt)

以上示例中，为“my_param”提供了参数值。因此，正确的方法是：

    result = conn.execute(stmt, my_param=12)

当消息采用“在参数组<paramgrp>中的绑定参数<parammsg>需要值”的形式时，消息指的是执行“executemany”格式时。在这种情况下，语句通常是 INSERT、UPDATE 或 DELETE，并且传递了参数列表。在这种格式下，语句可以根据第一个参数集来动态生成，以包括参数列表中的所有参数位置，其中将使用第一组参数来确定这些应该是什么。

例如，下面的语句是基于第一个参数集计算的，要求参数“a”、“b”和“c”-这些名称确定了最终的字符串格式，该字符串格式将用于列表中的每个参数集。由于第二个实体未包含“b”，因此会生成此错误：

    m = MetaData()
    t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))

    e.execute(
        t.insert(),
        [
            {"a": 1, "b": 2, "c": 3},
            {"a": 2, "c": 4},
            {"a": 3, "b": 4, "c": 5},
        ],
    ).. code-block::

 sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError)
 参数分组1中的bind参数 'b' 需要值
 [SQL: u'INSERT INTO t (a, b, c) VALUES (?, ?, ?)']
 [parameters: [{'a': 1, 'c': 3, 'b': 2}, {'a': 2, 'c': 4}, {'a': 3, 'c': 5, 'b': 4}]]

由于"b"参数是必需的，因此将其传递为 ``None`` 可以让 INSERT 继续执行::

    e.execute(
        t.insert(),
        [
            {"a": 1, "b": 2, "c": 3},
            {"a": 2, "b": None, "c": 4},
            {"a": 3, "b": 4, "c": 5},
        ],
    )

.. seealso::

  :ref:`tutorial_sending_parameters`

.. _error_89ve:

预期 FROM 子句，得到 Select。要创建 FROM 子句，请使用 .subquery() 方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这是SQLAlchemy 1.4中引入的一项更改，其中由诸如 :func:`_expression.select` 生成的 SELECT 语句，以及包括联合和文本 SELECT 表达式等内容在内的其他内容 不再被认为是 :class:`_expression.FromClause` 对象，并且不能直接放置在另一个 SELECT 语句的 FROM 子句中，必须首先将其包装在 :class:`.Subquery` 中。这是核心中的一个关键概念变化，详细的解释可以在 :ref:`change_4617` 中找到。

如下面的示例：

    m = MetaData()
    t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))
    stmt = select(t)

在上面的代码中， stmt 表示一个 SELECT 语句。当我们想要直接使用 stmt 在另一个 SELECT 语句的 FROM 子句中时，例如在以下代码中：

    new_stmt_1 = select(stmt)

或者，如果我们想要在 FROM 子句中使用它，例如在 JOIN 中：

    new_stmt_2 = select(some_table).select_from(some_table.join(stmt))

则会产生上面的错误。在之前的 SQLAlchemy 版本中，在另一个 SELECT 内部使用 SELECT 将生成无名称子查询，用括号括起来。在大多数情况下，这种形式的 SQL 不是很有用，因为诸如 MySQL 和 PostgreSQL 之类的数据库要求 FROM 子句中的子查询具有命名别名，这意味着使用 :meth:`_expression.SelectBase.alias` 方法或如 1.4 版本中所使用的 :meth:`_expression.SelectBase.subquery` 方法来生成这一别名。在其他数据库上，使子查询具有名称以消除子查询内对列名称的未来引用带来了明显的优势。

除了上面的实际原因之外，这种变化还有许多其他针对 SQLAlchemy 的原因。因此，以上两条语句的正确形式需要使用 :meth:`_expression.SelectBase.subquery` ：

    subq = stmt.subquery()

    new_stmt_1 = select(subq)

    new_stmt_2 = select(some_table).select_from(some_table.join(subq))

.. seealso::

  :ref:`change_4617`

.. _error_xaj1:

自动为原始 clauseelement 生成别名
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 1.4.26

此警告绑定到使用 :meth:`_orm.Query.join` 方法的旧风格或 :term:`2.0 风格`的 :meth:`_sql.Select.join` 方法以及使用连接表继承的映射时解析。问题在于，在两个连接到共享基表的连接继承模型之间进行连接时，如果没有应用到其中一个操作对象的别名，将无法形成适当的 SQL JOIN；SQLAlchemy 在这种情况下对右侧加别名。例如，考虑以下连接继承映射：

    class Employee(Base):
        __tablename__ = "employee"
        id = Column(Integer, primary_key=True)
        manager_id = Column(ForeignKey("manager.id"))
        name = Column(String(50))
        type = Column(String(50))

        reports_to = relationship("Manager", foreign_keys=manager_id)

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": type,
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = Column(Integer, ForeignKey("employee.id"), primary_key=True)

        __mapper_args__ = {
            "polymorphic_identity": "manager",
            "inherit_condition": id == Employee.id,
        }

上面的映射包括 ``Employee`` 和 ``Manager`` 之间的关系。由于这两个类都使用 "employee" 数据库表，因此从 SQL 的角度来看这是一个 :ref:`自引用关系 <self_referential>` 。如果我们想要使用连接从 ``Employee`` 和 ``Manager`` 模型中查询，并在 SQL 中表示上述操作，不得不如下面的 ORM 这样写：

    s = Session()
    stmt = select(Employee, Manager).join(Employee.reports_to)

上面的 SQL 语句选择了 "employee" 表作为查询的起点，表示查询对象为 ``Employee`` ，接着连接到了一个右嵌套连接，其详细描述为 ``employee AS employee_1 JOIN manager AS manager_1``，这个表明 rawclause 的右侧是一个未命名的别名 access 语法为employee_1。这就是上述SQLAlchemy 的警告消息提示automotatically generated alias。

当 SQLAlchemy 加载 ORM 行时，每个 ORM 行包含一个 ``Employee`` 和一个 ``Manager``，ORM 必须调整从 ``employee_1`` 和 ``manager_1`` 表别名游标中的行，使它们适配到未用别名的 ``Manager`` 类中。这个过程是内部复杂的，并且不提供所有 API 特性，尤其是在尝试使用比这里所示的更深层的查询时，如 :func:`_orm.contains_eager` 呈现的预加载特性。由于这种设计不可靠且涉及难以预测并且难以遵循的隐式决策，因此会发出警告，此模式可视为遗留特性。编写此查询的更好方法是使用与任何其他自引用关系相同的模式，即使用 :func:`_orm.aliased` 构造，对于连接继承和其他以连接为导向的映射，则通常最好添加使用 :paramref:`_orm.aliased.flat` 参数，以允许通过在连接内的各个表上应用别名来为两个或更多表的关联指定别名，而不是将连接嵌入到新的子查询中：

    from sqlalchemy.orm import aliased

    a1 = aliased(Address)

    # of_type() form; recommended
    q = (
        s.query(User)
        .join(User.addresses.of_type(a1))
        .filter(a1.email_address == "ed@foo.com")
    )

    # target, onclause form
    q = s.query(User).join(a1, User.addresses).filter(a1.email_address == "ed@foo.com")

.. _error_xaj2:

由于重叠的表而自动生成别名
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 1.4.26

该警告通常在使用 :meth:`_sql.Select.join` 方法或遗留的 :meth:`_orm.Query.join` 方法查询映射时生成，其中涉及加入表的继承。当在两个连接继承模型之间进行joining时，因为存在共同的基表，所以不能形成 SQL JOIN 来连接这两个实体，而不使用实现别名操作；在这种情况下，SQLAlchemy 将别名加右边的对齐。例如，给定如下连接继承映射：

    class Employee(Base):
        __tablename__ = "employee"
        id = Column(Integer, primary_key=True)
        manager_id = Column(ForeignKey("manager.id"))
        name = Column(String(50))
        type = Column(String(50))

        reports_to = relationship("Manager", foreign_keys=manager_id)

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": type,
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = Column(Integer, ForeignKey("employee.id"), primary_key=True)

        __mapper_args__ = {
            "polymorphic_identity": "manager",
            "inherit_condition": id == Employee.id,
        }

上述映射包括 ``Employee`` 和 ``Manager`` 之间的关系。由于两个类都使用 "employee" 数据库表，因此从 SQL 的角度来看这是一个 :ref:`自引用关系 <self_referential>` 。如果我们想要使用连接从 ``Employee`` 和 ``Manager`` 模型中查询，并在 SQL 中表示上述操作，我们得到的 SQL 将类似于以下 SQL：

    SELECT employee.id, employee.manager_id, employee.name,
    employee.type, manager_1.id AS id_1, employee_1.id AS id_2,
    employee_1.manager_id AS manager_id_1, employee_1.name AS name_1,
    employee_1.type AS type_1
    FROM employee JOIN
    (employee AS employee_1 JOIN manager AS manager_1 ON manager_1.id = employee_1.id)
    ON manager_1.id = employee.manager_id

上述 SQL 语句从 "employee" 表开始查询，表示查询对象为 ``Employee`` ，接着连接到一个右嵌套连接，其详细描述为 ``employee AS employee_1 JOIN manager AS manager_1``，这个表明 leftclause 的左侧是呈现为无名称子查询的左嵌套 join 语句。这就是上述SQLAlchemy 的警告消息提示automatically generated alias。

当 SQLAlchemy 加载 ORM 行时，每个 ORM 行包含一个 ``Employee`` 和一个 ``Manager``，ORM 必须调整 ``employee_1`` 和 ``manager_1`` 表别名游标中的行，使它们适配到没有别名的 ``Manager`` 类中。这个过程是内部复杂的，并且不提供所有 API 特性，特别是在尝试使用比此处所示更深度的查询时，仍会发生错误。由于这种设计不可靠且包含不易预测和不易跟踪的隐式决策，因此发出警告，并认为此模式可能是遗留特性。编写此查询的更好方法是使用与任何其他自引用关系相同的模式，即使用 :func:`_orm.aliased` 构造，对于连接继承和其他以连接为导向的映射，则通常最好添加使用 :paramref:`_orm.aliased.flat` 参数，以允许通过在连接内的各个表上应用别名来为两个或更多表的关联指定别名，而不是将连接嵌入到新的子查询中：

    from sqlalchemy.orm import aliased

    a1 = aliased(Address)

    # of_type() form; recommended
    q = (
        s.query(User)
        .join(User.addresses.of_type(a1))
        .filter(a1.email_address == "ed@foo.com")
    )

    # target, onclause form
    q = s.query(User).join(a1, User.addresses).filter(a1.email_address == "ed@foo.com")

.. _error_qzyx:

关系X将把列Q复制到列P，这与关系相冲突：'Y'
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此警告是指当两个或多个关系将数据写入刷新操作的相同的列时，但 ORM 没有任何手段来协调这些关系时。根据具体情况，解决方法可能是需要两个关系之间相互引用 :paramref:`_orm.relationship.back_populates`，或者一个或多个，该警告是不可避免的 SQLAlchemy捕获到可能导致数据丢失或不正确的连接关系。

relationships 应该配置为:paramref:`_orm.relationship.viewonly` ，以避免冲突写入，或者有时配置完全是有意的，并且应该将:paramref:`_orm.relationship.overlaps` 配置为静音每个警告。

对于缺少 :paramref:`_orm.relationship.back_populates` 的典型示例，给出以下映射::

    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        children = relationship("Child")


    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
        parent_id = Column(ForeignKey("parent.id"))
        parent = relationship("Parent")

上述映射将会生成警告：

.. sourcecode:: text

  SAWarning: relationship 'Child.parent' will copy column parent.id to column child.parent_id,
  which conflicts with relationship(s): 'Parent.children' (copies parent.id to child.parent_id).

关系 ``Child.parent`` 和 ``Parent.children`` 看起来处于冲突状态。解决方案是应用 :paramref:`_orm.relationship.back_populates`  ::

    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        children = relationship("Child", back_populates="parent")


    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
        parent_id = Column(ForeignKey("parent.id"))
        parent = relationship("Parent", back_populates="children")

对于更加自定义化的关系，如果"overlap" 情况是有意的且无法解决，则:paramref:`_orm.relationship.overlaps` 参数可以指定不应发出警告的关系的名称。这通常发生在与包括自定义 :paramref:`_orm.relationship.primaryjoin` 条件的某个基础表的两个或多个关系相关的情况下，以限制每种情况下的相关项::

    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        c1 = relationship(
            "Child",
            primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 0)",
            backref="parent",
            overlaps="c2, parent",
        )
        c2 = relationship(
            "Child",
            primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 1)",
            overlaps="c1, parent",
        )


    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
        parent_id = Column(ForeignKey("parent.id"))

        flag = Column(Integer)

上述 ORM 将知道 ``Parent.c1`` 、 ``Parent.c2`` 和 ``Child.parent`` 之间的重叠是有意的。

.. _error_lkrp:

无法将对象转换为 "persistent" 状态，因为此标识图不再有效。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 1.4.26

此消息添加是为了处理在 :class:`_orm.Session` 已关闭或已调用其 :meth:`_orm.Session.expunge_all` 方法的之后，仍然迭代 :class:`_result.Result` 对象而导致的错误。当 :class:`_orm.Session` 一次性清除所有对象时，由该 :class:`_orm.Session` 使用的内部 :term:`identity map` 将被替换为新的，并且丢弃原始的 identity map。一个未使用且未缓冲的 :class:`_result.Result` 对象在内部将保留对该现在已弃用的 identity map 的引用。因此，当消耗 :class:`_result.Result` 时，将产生错误。

:ref:`2.0 AsyncIO 扩展 <asyncio_toplevel>` 的情况通常不会出现此类情况，因为当 :class:`.AsyncSession` 返回一个传统的 :class:`_result.Result` 时，当执行语句时，结果已预先缓冲。这是为了在无需额外的`` await`` 调用的情况下启用次要的急切装入程序。

要在使用普通的 :class:`_orm.Session` 的情况下以与 ``asyncio`` 扩展类似的方式预缓冲结果，可以使用 ``prebuffer_rows`` 执行选项，如下所示::

    # context manager creates new Session
    with Session(engine) as session_obj:
        # result internally pre-fetches all objects
        result = sess.execute(
            select(User).where(User.id == 7), execution_options={"prebuffer_rows": True}
        )

    # context manager is closed, so session_obj above is closed, identity
    # map is replaced

    # pre-buffered objects are returned
    user = result.first()

    # however they are detached from the session, which has been closed
    assert inspect(user).detached
    assert inspect(user).session is None

此处，所选的 ORM 对象完全是在 ``session_obj`` 块内生成的，与 ``session_obj`` 关联并在 :class:`_result.Result` 对象中缓冲。在块外，``session_obj`` 已关闭并清除了这些 ORM 对象。迭代 :class:`_result.Result` 对象将产生那些 ORM 对象，但是由于其中的 :class:`_orm.Session` 已将其清除，因此它们将在 :term:`分离`(detached) 状态下返回。

.. note:: 上述对 "预缓冲" vs. "未缓冲" :class:`_result.Result` 对象的提及是指 ORM 将从 :term:`DBAPI` 中的完整的原始值转换为 ORM 对象的过程。它并不意味着底层的 ``游标`` 对象本身，该对象表示来自 DBAPI 的预处理结果，是缓冲或未缓冲的，因为这本质上是缓冲的较低层。关于缓冲 ``游标`` 结果本身的背景，请参阅 :ref:`engine_stream_results` 节。

.. _error_zlpr:

无法解释 Annotated Declarative Table 形式的注释类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0 引入了一种新的 :ref:`Annotated Declarative Table <orm_declarative_mapped_column>` 声明系统，该系统在运行时从类定义中的 :pep:`484` 注释中派生 ORM 映射的属性信息。 此形式的要求是所有 ORM 注释必须使用名为 :class:`_orm.Mapped` 的通用容器来进行正确注释。 使用显式 :pep:`484` 类型注释的旧版 SQLAlchemy 映射（例如，那些使用 :ref:`legacy Mypy 扩展 <mypy_toplevel>` 提供类型支持的映射）可能会包括不包括该泛型的 :func:`_orm.relationship` 指令等指令。

要解决此问题，类可以标记为 ``__allow_unmapped__`` 布尔属性，直到它们可以完全迁移到 2.0 语法。请参见 :ref:`migration_20_step_six` 中的迁移说明的示例。

.. seealso::

    :ref:`migration_20_step_six` - 在 :ref:`migration_20_toplevel` 文档中

.. _error_dcmx:

将 <cls> 转换为数据类时出错，其中一个或多个属性来自于不是数据类的超类 <cls>。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此警告在使用 SQLAlchemy ORM Mapped Dataclasses 功能 (:ref:`orm_declarative_native_dataclasses`) 与任何混合类或抽象基本类一起使用时触发，这些混合类或抽象基本类本身没有声明为数据类。例如以下示例::

    from __future__ import annotations

    import inspect
    from typing import Optional
    from uuid import uuid4

    from sqlalchemy import String
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Mixin:
        create_user: Mapped[int] = mapped_column()
        update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)


    class Base(DeclarativeBase, MappedAsDataclass):
        pass


    class User(Base, Mixin):
        __tablename__ = "sys_user"

        uid: Mapped[str] = mapped_column(
            String(50), init=False, default_factory=uuid4, primary_key=True
        )
        username: Mapped[str] = mapped_column()
        email: Mapped[str] = mapped_column()

上述示例中，由于 ``Mixin`` 本身没有扩展 :class:`_orm.MappedAsDataclass`，因此将生成以下警告：

.. sourcecode:: none

    SADeprecationWarning: When transforming <class '__main__.User'> to a
    dataclass, attribute(s) "create_user", "update_user" originates from
    superclass <class
    '__main__.Mixin'>, which is not a dataclass. This usage is deprecated and
    will raise an error in SQLAlchemy 2.1. When declaring SQLAlchemy
    Declarative Dataclasses, ensure that all mixin classes and other
    superclasses which include attributes are also a subclass of
    MappedAsDataclass.

解决方法是将 :class:`_orm.MappedAsDataclass` 添加到 ``Mixin`` 签名中:

    class Mixin(MappedAsDataclass):
        create_user: Mapped[int] = mapped_column()
        update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

Python 的 :pep:`681` 规范不支持在数据类的超类中声明的属性，这些属性本身不是数据类; 根据 Python 数据类的行为，这些字段被忽略，如以下示例:

    from dataclasses import dataclass
    from dataclasses import field
    import inspect
    from typing import Optional
    from uuid import uuid4


    class Mixin:
        create_user: int
        update_user: Optional[int] = field(default=None)


    @dataclass
    class User(Mixin):
        uid: str = field(init=False, default_factory=lambda: str(uuid4()))
        username: str
        password: str
        email: str

以上， ``User`` 类将不包括 ``create_user`` 在其构造函数中，也不会尝试将 ``update_user`` 解释为数据类属性。这是因为 ``Mixin`` 不是一个数据类。

SQLAlchemy 2.0 中的 ORM 数据类功能在正确处理上述情况方面未能正确处理; 相反，非数据类混合和超类上的属性被视为数据类配置的一部分。但是，像 Pyright 和 Mypy 这样的类型检查器将不考虑这些字段作为数据类构造函数的一部分，因为根据 :pep:`681`，应该忽略这些字段。由于它们的存在具有歧义，因此 SQLAlchemy 2.1 将需要使数据类层次结构中包括 SQLAlchemy 映射属性的混合类自身也成为数据类。

.. _error_bupq:

对于每个主键，按行 ORM 群集更新需要记录包含主键值
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果在给定记录中未提供主键值，则此错误在使用 :ref:`orm_queryguide_bulk_update` 功能是会发生的，例如::


    >>> session.execute(
    ...     update(User).where(User.name == bindparam("u_name")),
    ...     [
    ...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"u_name": "patrick", "fullname": "Patrick Star"},
    ...     ],
    ... )

上述代码中，由于具有参数字典列表和使用 :class:`_orm.Session` 执行启用了 ORM 批量更新主键，因此必须在每个参数字典中包括主键值，即::

    >>> session.execute(
    ...     update(User),
    ...     [
    ...         {"id": 1, "fullname": "Spongebob Squarepants"},
    ...         {"id": 3, "fullname": "Patrick Star"},
    ...         {"id": 5, "fullname": "Eugene H. Krabs"},
    ...     ],
    ... )

通过调用 ``session.connection()`` 获得当前 :class:`_engine.Connection` , 然后用它来执行语句::

    >>> session.connection().execute(
    ...     update(User).where(User.name == bindparam("u_name")),
    ...     [
    ...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"u_name": "patrick", "fullname": "Patrick Star"},
    ...     ],
    ... )


.. seealso::

        :ref:`orm_queryguide_bulk_update`

        :ref:`orm_queryguide_bulk_update_disabling`


异步 IO 异常
------------

.. _error_xd1r:

AwaitRequired
~~~~~~~~~~~~~

要使用异步模式连接数据库，需要使用异步驱动程序。当尝试使用与 :term:`DBAPI` 不兼容的要使用异步版本的 SQLAlchemy 时，通常会引发此错误。

.. seealso::

    :ref:`asyncio_toplevel`

.. _error_xd2s:

MissingGreenlet
~~~~~~~~~~~~~~~

当在意料之外的位置启动 IO 尝试时，使用不直接提供 ``await`` 关键字的调用模式，就会触发此错误。当使用 ORM 时，这通常是由于使用 :term:`lazy loading` 引起的，因为其在 asyncio 下无法直接支持，而是需要进行其他步骤和/或更改加载程序的模式。

.. seealso::

    :ref:`asyncio_orm_avoid_lazyloads` - 涵盖了大多数 ORM 场景，以及如何减轻这种问题，包括用于延迟加载场景的特定模式。

.. _error_xd3s:

No Inspection Available
~~~~~~~~~~~~~~~~~~~~~~~

在 :class:`_asyxncio.AsyncConnection` 或 :class:`_asyncio.AsyncEngine` 对象上直接使用 :func:`_sa.inspect` 函数时，暂未提供可等待的 :class:`_reflection.Inspector` 对象。因此，应在使用 :func:`_sa.inspect` 时以方式引用 :class:`_asyncio.AsyncConnection.sync_connection` 属性，以便最初引用 :class:`_engine.Inspector` 对象；然后使用 :meth:`_asyncio.AsyncConnection.run_sync` 方法以及执行所需操作的自定义函数以使用 :class:`_engine.Inspector`，类似于以 "同步" 调用方式的用法::

    async def async_main():
        async with engine.connect() as conn:
            tables = await conn.run_sync(
                lambda sync_conn: inspect(sync_conn).get_table_names()
            )

.. seealso::

    :ref:`asyncio_inspector` - 有关在 asyncio 扩展中使用 :func:`_sa.inspect` 的其他示例。

核心异常类
----------

有关核心异常类，请参见 :ref:`core_exceptions_toplevel`。

ORM 异常类
----------

有关 ORM 异常类，请参见 :ref:`orm_exceptions_toplevel`。


历史遗留问题
------------

此部分中的异常不由当前 SQLAlchemy 版本生成，但是在此处提供以配合异常消息超链接。

.. _error_b8d9:

在 SQLAlchemy 2.0 中，<某些函数> 将不再 <做某事>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0 对于 Core 和 ORM 组件的许多重要 SQLAlchemy 使用模式进行了重大调整。 2.0 发布的目标是对 SQLAlchemy 自从起初开始以来的一些最基本的假设进行微调，并提供一个新的经过简化的使用模型，其目的是显着更加简约，Consistent between the Core and ORM components, as 兼容性更强。

在退化至 :ref:`migration_20_ttl` 中介绍的 DSL 合规性等问题上，SQLAlchemy 2.0 项目包括一个完整的将来的兼容性系统，该系统集成到 SQLAlchemy 1.4 系列中，以便应用程序可以具有明确的，明确的，并逐渐的向 2.0 兼容迁移应用程序的过程。  :class:`.exc.RemovedIn20Warning` 废弃警告是这个系统的基础，可提供有关需要修改的现有代码库中的行为的指导。可以在 :ref:`deprecation_20_mode` 上查看有关如何启用此警告的概述。

.. seealso::

    :ref:`migration_20_ttl` - 从 1.x 系列开始的升级过程概述，以及实现全面 2.0 兼容性的当前目标和进度。

    :ref:`deprecation_20_mode` - 有关如何在 SQLAlchemy 1.4 中使用 "2.0 废除模式" 的特定指南。


.. _error_s9r1:

对象正在通过 backref 级联合并到 Session 中
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此消息是指 SQLAlchemy 的 "backref cascade" 行为，在版本 2.0 中已删除。 这指的是将一个对象添加到 :class:`_orm.Session` 中，作为与它已经存在于该会话中的其他对象关联的结果。由于此行为已被证明比有用更具有混淆性，因此添加了 :paramref:`_orm.relationship.cascade_backrefs` 和 :paramref:`_orm.backref.cascade_backrefs` 参数，可将其设置为 ``False`` 以禁用该行为，在 SQLAlchemy 2.0 中，“backref cascade”行为已完全删除。

对于以前的 SQLAlchemy 版本，要在具有 :paramref:`_orm.relationship.backref` 字符串参数的 backref 上将 :paramref:`_orm.relationship.cascade_backrefs` 设置为 ``False``，必须首先使用 :func:`_orm.backref` 函数声明 backref，以便可以通过 :paramref:`_orm.backref.cascade_backrefs` 参数进行传递。

或者，可以使用 :class:`_orm.Session` 上的 "future" 模式完全关闭“backref cascade”行为，即通过 :paramref:`_orm.Session.future` 参数传递 ``True``。

.. seealso::

    :ref:`change_5150` - SQLAlchemy 2.0 中的变更背景。


.. _error_c9ae:

在 "legacy" 模式下创建的 select() 构造; 关键字参数等。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

截至SQLAlchemy 1.4，已更新 :func:`_expression.select` 构造，使其支持标准 SQLAlchemy 2.0 中的新调用样式。在 1.4 系列内向后兼容，该构造接受旧式“遗留”样式作为参数，以及新样式。

"新" 样式的特点是列和表达式仅以位置传递到 :func:`_expression.select` 构造，任何其他修改对象的修饰符都必须使用后续的方法链接传递::

    # 这是正式的样子
    stmt = select(table1.c.myid).where(table1.c.myid == table2.c.otherid)

相比之下，在 SQLAlchemy 的旧式形式中，在添加类似 :meth:`.Select.where` 等方法之前，语句将是::

    # 这是由原始的 SQLAlchemy 版本文档记录的方式
    stmt = select([table1.c.myid], whereclause=table1.c.myid == table2.c.otherid)

甚至可能是将 "whereclause" 作为位置参数传递::

    # 这也是由原始的 SQLAlchemy 版本文档记录的方式
    stmt = select([table1.c.myid], table1.c.myid == table2.c.otherid)

多年来，已删除大多数叙述文档中包括任何其他参数（例如 "whereclause"）的指导，导致传递作为列表的列参数，但没有其他参数::

    # 这是从 1.0 或左右版本以来的文档中记录的方式
    stmt = select([table1.c.myid]).where(table1.c.myid == table2.c.otherid)

在 :ref:`migration_20_5284` 文档中描述了此更改的信息，涉及到 DSL 合规性问题。

.. seealso::

    :ref:`migration_20_5284`

    :ref:`migration_20_ttl`

.. _error_c9bf:

找到使用的是 legacy 绑定元数据绑定，但由于在此会话中设置了 future=True，因此将忽略该绑定。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

“已绑定元数据” 的概念仅存在于 SQLAlchemy 1.x 版本中，从 SQLAlchemy 2.0 开始已删除。

该错误指的是 :class:`_schema.MetaData` 对象中的 :paramref:`_schema.MetaData.bind` 参数，该对象允许类似 ORM :class:`_orm.Session` 将特定映射类与 :class:`_orm.Engine` 关联。 在 SQLAlchemy 2.0 中，:class:`_orm.Session` 必须直接链接到每个 :class:`_orm.Engine`。 也就是说，必须将 :class:`_orm.Session` 或 :class:`_orm.sessionmaker` 实例化为关联 :class:`_engine.Engine`，而不是使用 :class:`_schema.MetaData`::

    engine = create_engine("sqlite://")
    Session = sessionmaker()
    metadata_obj = MetaData(bind=engine)
    Base = declarative_base(metadata=metadata_obj)


    class MyClass(Base):
        ...


    session = Session()
    session.add(MyClass())
    session.commit()

必须直接将 :class:`_engine.Engine` 与 :class:`_orm.sessionmaker` 或 :class:`_orm.Session` 关联。 :class:`_schema.MetaData` 对象不应再与任何引擎关联::

    engine = create_engine("sqlite://")
    Session = sessionmaker(engine)
    Base = declarative_base()


    class MyClass(Base):
        ...


    session = Session()
    session.add(MyClass())
    session.commit()

在 SQLAlchemy 1.4 中，可以在使用 :class:`_orm.sessionmaker` 或 :class:`_orm.Session` 时将 :paramref:`_orm.Session.future` 标志设置为 ``True``，以启用此 :term:`2.0 样式` 行为。


.. _error_2afi:

此编译对象未绑定到任何引擎或连接
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此错误是指 "已绑定元数据" 的概念，此概念仅存在于 < SQLAlchemy 1.4 版本中。当从未关联到 :class:`_engine.Engine` 的 Core 表达式对象中直接从 :meth:`.Executable.execute` 方法调用时，将触发该错误::

    metadata_obj = MetaData()
    table = Table("t", metadata_obj, Column("q", Integer))

    stmt = select(table)
    result = stmt.execute()  # <--- raises

逻辑逻辑期望的是 :class:`_schema.MetaData` 对象已被 **绑定** 到 :class:`_engine.Engine` 上，例如::

    engine = create_engine("mysql+pymysql://user:pass@host/db")
    metadata_obj = MetaData(bind=engine)

上述，任何源自 :class:`_schema.Table` 的语句，该 :class:`_schema.Table` 又派生自该 :class:`_schema.MetaData`，将隐含使用给定 :class:`_engine.Engine` 以调用语句。

请注意，“已绑定元数据”的概念在 SQLAlchemy 2.0 中已经 **不存在**。正确的方法是通过 :meth:`_engine.Connection.execute` 方法的 :class:`_engine.Connection` 来调用语句::

    with engine.connect() as conn:
        result = conn.execute(stmt)

在 ORM 中，通过 :class:`.Session` 也可以使用类似的设施::

    result = session.execute(stmt)

.. seealso::

    :ref:`tutorial_statement_execution`在 :ref:`faq_session_rollback` 的常见问题解答中描述了这个问题。

“子事务”模式在SQLAlchemy 2.0中被移除，因此这种特定的编程模式将不再可用，从而防止了此错误消息的出现。