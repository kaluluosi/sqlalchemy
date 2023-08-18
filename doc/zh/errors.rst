:orphan:

.. _errors:

=============
错误消息
=============

本节列出 SQLAlchemy 引发的常见错误消息和警告的描述和背景。

SQLAlchemy 通常在 SQLAlchemy 特定的异常中引发错误。
有关这些类的详细信息，请参见   :ref:`core_exceptions_toplevel`  和   :ref:` orm_exceptions_toplevel` 。

SQLAlchemy 错误大致可分为两类：**编程时错误**和**运行时错误**。
编程时错误是由于使用不正确的参数调用函数或方法或来自无法解决的其他配置导向的方法而引发的。
编程时错误通常是即时的和确定性的。相反，运行时错误表示程序在相应地以任意方式运行时发生的错误，
例如数据库连接被耗尽或发生某些数据相关问题。运行时错误更可能在运行应用程序时的日志中看到，
因为程序在遇到这些状态时，会响应加载和遇到的数据。

由于运行时错误不容易重现，并且通常是作为响应某些任意条件发生的，因此对于调试来说比较困难，
并且也会影响已经投入生产的程序。

在本节中，目标是尝试提供有关一些常见的运行时错误及编程时错误的背景信息。



连接和事务
----------------------------

.. _error_3o7r:

队列池大小限制 <x> 溢出 <y>，连接超时，超时 <z>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这可能是最常见的运行时错误，
因为它直接涉及应用程序的工作负荷超过了配置的限制，这通常适用于几乎所有 SQLAlchemy 应用程序。

以下是总结此错误的要点，从大多数 SQLAlchemy 用户应该已经熟悉的最基本的要点开始。

* **SQLAlchemy 引擎对象默认使用连接池** - 这意味着当使用   :class:`_engine.Engine`  对象的 SQL 数据库连接资源时，
  然后  :term:`releases`  该资源，该数据库连接本身仍连接到数据库，并返回到内部队列，可以再次使用。尽管代码似乎正在
  结束与数据库的对话，在许多情况下，应用程序将仍保留一定数量的数据库连接，这些连接在应用程序结束或池显式释放之前一直存在。

* 由于池，当应用程序使用 SQL 数据库连接时，通常最常使用的是  :meth:`_engine.Engine.connect` 
  或使用 ORM 的查询   :class:`.Session` ，此操作未必在获取连接对象时立即与数据库建立新连接;
  它将查询连接池以获取一个连接，该池通常从中检索要重新使用的现有连接。如果没有可用的连接，池将创建一个
  新的数据库连接，但仅在池未超过配置的容量时。

* 在大多数情况下使用的默认池名为   :class:`.QueuePool` 。当您要求此池提供连接而没有可用连接时，
  它将创建一个新的连接，**如果播放的总连接数少于配置的值**，这个值等于**池大小加上最大溢出**。这意味着，
  如果您已将引擎配置为:: 

   engine = create_engine("mysql+mysqldb://u:p@host/db", pool_size=10, max_overflow=20)

  上面，   :class:`_engine.Engine`  将允许任意时间内最多有 **30 个连接**，这不包括已从引擎中分离或失效的连接。如果到达新连接的请求，
  并且已有 30 个连接正在应用程序的其他部分中使用，则连接池将在固定的时间内阻塞，然后超时并引发此错误消息。

  为了允许同时使用更多的连接，池可以使用  :paramref:`_sa.create_engine.pool_size` 
  和  :paramref:`_sa.create_engine.max_overflow`  参数调整，作为   :func:` _sa.create_engine` 
  函数传递。等待连接可用的超时是通过配置  :paramref:`_sa.create_engine.pool_timeout`  参数来配置的。

* 通过将  :paramref:`_sa.create_engine.max_overflow`  设置为-1，可以配置池具有无限溢出。在此设置下，
  池仍会维护一组固定的连接池，但如果没有可用连接，则不会阻塞对新连接的请求。但是，在此方式下运行时，
  如果应用程序存在耗尽所有可用连通性资源的问题，则最终会在数据库本身的可用连接数限制下出现错误。
  更严重的是，当应用程序耗尽数据库的连接时，它通常会导致使用了大量资源失败，并且可能会干扰其它应用程序和依赖于连接到数据库的数据库状态机制。

  考虑到以上情况，连接池可以被看作是连接使用的**安全阀门**，提供了对流氓应用程序防止整个数据库
  变为对所有其他应用程序不可用的重要保护层。在收到此错误消息时，最好修复使用太多连接和/或适当配置限制，
  而不是允许无限溢出，这实际上并没有解决根本问题。

是什么导致应用程序使用完所有可用的连接？

* **应用程序面临的并发请求数过多** - 这是最直接的原因。如果您有一个运行在允许 30
  个并发线程的线程池中的应用程序，并且每个线程使用一个连接，如果您的池未配置为允许同时
  有至少30个连接签出，则在您的应用程序接收足够的并发请求后，您将获得此错误。
  解决方法是提高池的限制或降低并行线程的数量。

* **应用程序未将连接还回池** - 这是第二个最常见的原因，即应用程序使用连接池，
   但程序未终止使用此连接资源，而是将其保持打开状态。连接池以及 ORM 的   :class:`.Session`  均具有逻辑，
  当回收 session 和/或 connection 对象时，会释放底层连接资源，但不能依赖此行为及时释放资源。

  发生这种情况的常见原因是应用程序使用 ORM 会话并且在完成涉及该会话的工作后未调用  :meth:`.Session.close` 。
  解决方法是，确保使用 ORM 会话（如果使用 ORM）或使用 Core 接口绑定的   :class:`_engine.Connection`  对象在完成工作后被显式关闭，
  即通过适当的 ``.close()`` 方法或使用其中一个可用的上下文管理器（例如 “with:” 语句）正确释放资源。

* **应用程序尝试运行长时间的事务** - 数据库事务是一种非常昂贵的资源，
  不应该**保持空闲等待某个事件发生**。如果应用程序正在等待用户按按钮，或来自长时间运行的作业队列中传递
  的结果，或保持打开的持久连接到浏览器，则不要为整个时间**都保持数据库事务打开**。当应用程序需要使用数据库并与事件交互时，
  在此时开启一个短寿命的事务，然后关闭它。

* **应用程序死锁** - 这也是此错误的常见原因，更难以理解，如果应用程序不能完成其对连接的使用，
  无论是由于应用程序侧还是由于数据库侧死锁，应用程序可能会使用完所有可用连接，从而导致其他请求接收此错误。
  死锁原因包括：

  * 使用诸如 gevent 或 eventlet 等隐式异步系统而未正确 monkeypatch 所有套接字库和驱动程序，
  或者隐式异步系统在未在全面覆盖所有 monkeypatched 驱动程序方法方面存在 bug，
  或者非常不常见的情况是，如果异步系统用于 CPU 绑定工作负载，那么使用数据库资源的 greenlets 只需等太长时间才能对其进行处理。
  对于绝大多数关系型数据库操作，隐式和显式异步编程框架通常不是必要或适当的；如果应用程序必须在某些功能区域使用异步系统，
  最好是让面向数据库业务方法在传递消息到应用程序的异步部分中运行传统线程。

  * 数据库侧死锁，例如相互死锁的行

  * 线程错误，例如互锁的互斥锁，或在同一线程中调用已被锁定的互斥锁

请记住，除了使用池之外还有一种替代方法，即完全关闭池。
请参见   :ref:`pool_switching`  部分，了解有关此背景信息。但是，请注意，当出现此错误消息时，
它始终由应用程序本身中更大的问题引起；池只有帮助更早地暴露了该问题。





DBAPI 错误
------------

Python 数据库 API（DBAPI）是定位于数据库驱动程序
的规范，它位于 `Pep-249 <https://www.python.org/dev/peps/pep-0249/>`_ 中。
该 API 指定了适应数据库的完整故障模式所需要的一组异常类。

SQLAlchemy 并不直接生成这些异常。相反，它们被从数据库驱动程序中拦截并由由 SQLAlchemy 提供的异常   :class:`.DBAPIError` 包装，
但是该异常中的消息是**由驱动程序生成，而不是 SQLAlchemy 生成的**。

.. _error_rvf5:

InterfaceError
~~~~~~~~~~~~~~

与数据库本身而不是与数据库接口相关的错误引发的异常。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

“InterfaceError”有时由驱动程序在上下文中引发，
因为检测到了数据库连接是否断开，或者无法连接到数据库。有关如何处理此问题的提示，请参见   :ref:`pool_disconnects`  部分。





.. _error_4xp6:

DatabaseError
~~~~~~~~~~~~~

与数据库本身而不是与所传输数据或接口相关的错误引发的异常。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。





.. _error_9h9h:

DataError
~~~~~~~~~

由于处理数据时出现问题（例如被零整除、数字值超出范围等）而引发的异常。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。





.. _error_e3q8:

OperationalError
~~~~~~~~~~~~~~~~

与数据库的操作而不是程序员控制下的错误相关引发的异常，例如，出现意外断开连接，
数据源名称未找到，无法处理事务，执行期间发生内存分配错误等。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

“OperationalError”是驱动程序在上下文中使用的最常见（但并非唯一）错误类，
因为数据库连接被丢弃，或无法连接到数据库。有关如何处理此问题的提示，请参见   :ref:`pool_disconnects`  部分。






.. _error_gkpj:

IntegrityError
~~~~~~~~~~~~~~

在影响数据库的关系完整性时引发的异常，例如，外键检查失败。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。





.. _error_2j85:

InternalError
~~~~~~~~~~~~~

在数据库遇到内部错误时引发的异常，例如，光标不再有效，事务不同步等。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

“InternalError”有时由驱动程序在上下文中引发，
因为检测到了数据库连接是否断开，或者无法连接到数据库。有关如何处理此问题的提示，请参见   :ref:`pool_disconnects`  部分。





.. _error_f405:

ProgrammingError
~~~~~~~~~~~~~~~~

与编程错误相关的异常，例如，找不到或已存在表，SQL 语句中的语法错误，指定了错误数量的参数等。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

“ProgrammingError”有时由驱动程序在上下文中引发，
因为检测到了数据库连接是否断开，或者无法连接到数据库。有关如何处理此问题的提示，请参见   :ref:`pool_disconnects`  部分。





.. _error_tw8g:

NotSupportedError
~~~~~~~~~~~~~~~~~

在使用不支持的方法或数据库 API 的情况下引发的异常，例如，在不支持事务的连接上请求 .rollback()。

这是个   :ref:`DBAPI 错误 <error_dbapi>`  ，并起源于
数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。




SQL 表达式语言
-----------------------
.. _error_cprf:
.. _caching_caveats:

对象不会生成缓存键，性能影响
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 版本 1.4 包括一个   :ref:`SQL 编译缓存设施 <sql_caching>` ，其会使 Core 和 ORM SQL 构造缓存其字符串形式，
以及用于从语句获取结果的其他结构信息，允许相对昂贵的字符串编译过程在下次使用另一构造时跳过。该系统依赖于所有 SQL 构造
（例如   :class:`_schema.Column` ，  :func:` _sql.select`  和   :class:`_types.TypeEngine`  对象）实现用于生成完全表示它们的状态的**缓存键**，
到了这个程度影响 SQL 编译过程。

如果警告涉及广泛使用的对象，例如   :class:`_schema.Column`  对象，并且显示它们影响了大多数 SQL 构造（使用   :ref:` sql_caching_logging` 
中描述的估计技术），以便缓存通常对于应用程序是未启用的，这将对性能产生负面影响，并且在某些情况下，甚至可能有效地产生了**性能下降**，
与之前的 SQLAlchemy 版本相比。   :ref:`faq_new_caching`  在   :ref:` faq_toplevel`  部分中介绍了此问题的详细信息。

如果对于某个特殊的后端，具体作用域的建议，例如在 PostgreSQL 中的 `"insert on conflict" <postgresql_insert_on_conflict>`_ 构造。
建议使用合理的方法生成缓存键，因此针对特定后端的构造可以缓存它们的 SQL 表示形式。

缓存会在存在任何疑问的情况下禁用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

缓存依赖于能够生成完整表示语句中 **完整的结构** 以使其安全，但保持 **一致** 的“缓存键”。如果某个特定的 SQL 构造（
或类型）没有适当的指令，该构造将无法生成正确的缓存键，因此不能安全地启用缓存：

* 缓存键必须表示 **完整的结构** ：如果使用两个单独的实例的使用可能会导致字符串不同的两个实例，
  则使用第一个元素将字符串化时缓存该 SQL 时将使用与第二元素的区别捕获的不同的参数，并且在“第二场”使用该元素时，
  会产生错误的 SQL。

* 缓存键必须是 **一致的** ：如果构造表示每次都会发生更改的状态，例如文字值，则无法安全地启用缓存，
  因为重复使用该构造将会快速填满独特的 SQL 字符串缓存，而这些 SQL 字符串通常不会再次使用，无用地消耗缓存区，从而破坏缓存的目的。

基于这两个原因，SQLAlchemy 的缓存系统对于决定将 SQL 缓存到对象中变得**极度保守**。



用于缓存的断言属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

警告基于以下标准发出。有关更多详细信息，请参见   :ref:`faq_new_caching` 。

*   :class:`.Dialect` （即我们传递给   :func:` _sa.create_engine`  的 URL 的第一部分指定的模块，例如 `postgresql+psycopg2://`）必须表明它已经得到验证并测试，以支持安全缓存。这由  :attr:`.Dialect.supports_statement_cache`  属性设置为 ` `True`` 时指示。当使用第三方方言时，请与方言的维护人员咨询，以便他们可以遵循   :ref:`步骤，确保可以启用缓存 <engine_thirdparty_caching>`  在其方言中，并发布新版本。

* 这些类型是从   :class:`.TypeDecorator`  或   :class:` .UserDefinedType`  继承的第三方或用户定义类型必须在其定义中包括  :attr:`.ExternalType.cache_ok` 
  属性，其中包括对所有派生子类都简要描述的该属性定义。如前所述，如果这些数据类型从第三方库导入，则请与该库的维护者商议，以便他们可以提供所需更改以及
  发布新版本。

* 第三方或用户定义的 SQL 构造，这些构造为某个特定的后端构造专用，例如   :ref:`pragma_foreign_keys` ，均需要使用某些 SQL 编译器类。
  不适用于默认的   :class:`_sql.compiler.StrSQLCompiler` ，也包括复杂的子类，以及为   :ref:` sqlalchemy.ext.compiler_toplevel`  进行设计的对象，应按照需要包括
   :attr:`.HasCacheKey.inherit_cache`  属性，并根据构造设计将其设置为 ` `True`` 或 ``False``，按照   :ref:`compilerext_caching`  中的指南描述。

.. seealso::

      :ref:`sql_caching_logging`  - 可观察缓存行为和效率的背景

      :ref:`faq_new_caching`  - 在   :ref:` faq_toplevel`  部分中





.. _error_cd3x:

Compiler StrSQLCompiler 无法呈现类型为 <x> 的元素
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通常在尝试对包含非默认编译的元素的 SQL 表达式构造进行字符串转换时发生此错误；
在这种情况下，将命名   :class:`.StrSQLCompiler`  类。
在较少见的情况下，它也可能会发生在某个特定类型的数据库后端上使用错误类型的 SQL 表达式时；在这些情况下，将命名其他类型的 SQL 编译器类，
例如``SQLCompiler`` 或 ``sqlalchemy.dialects.postgresql.PGCompiler``。以下是针对字符串化用例更具体的指导方针，但也描述了一般背景。

通常，可以直接对 Core SQL 构造或 ORM   :class:`_query.Query`  对象进行字符串化，例如使用 ` `print（）``：

.. sourcecode:: pycon+sql

  >>> from sqlalchemy import column
  >>> print(column("x") == 5)
  {printsql}x = :x_1

在上面的 SQL 表达式被字符串化时，将使用   :class:`.StrSQLCompiler`  编译器类，这是一个特殊的语句
编译器，当在没有任何特定于方言的信息的情况下对构造进行字符串化时，将使用该编译器类。

然而，有许多构造是特定于某个特定种类的数据库方言的，而   :class:`.StrSQLCompiler`  并不知道如何将它们转换成字符串，
例如 PostgreSQL 中的 "insert on conflict" 构造：

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

为了字符串化特定于特定后端的构造，必须使用  :meth:`_expression.ClauseElement.compile`  方法，传递   :class:` _engine.Engine`  或   :class:`.Dialect`  对象，
这将调用正确的编译器。 下面我们使用一个 PostgreSQL 方言：

.. sourcecode:: pycon+sql

  >>> from sqlalchemy.dialects import postgresql
  >>> print(insert_stmt.compile(dialect=postgresql.dialect()))
  {printsql}INSERT INTO my_table (x) VALUES (%(x)s) ON CONFLICT (y) DO NOTHING

对于 ORM   :class:`_query.Query`  对象，可以使用  :attr:` ~.orm.query.Query.statement`  访问语句::

    statement = query.statement
    print(statement.compile(dialect=postgresql.dialect()))

有关 SQL 元素直接字符串化/编译的额外详细信息，请参见其 FAQ 链接。

.. seealso::

    :ref:`faq_sql_expression_string` 


TypeError：“<operator>”不支持实例之间的'ColumnProperty'和 <something>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在上下文中使用一个   :func:`.column_property`  或   :func:` .deferred`  对象尝试使用 SQL 表达式时，
通常在声明中发生，如下所示：

    class Bar(Base):
        __tablename__ = "bar"

        id = Column(Integer, primary_key=True)
        cprop = deferred(Column(Integer))

        __table_args__ = (CheckConstraint(cprop > 5),)

上面的“ cprop”属性用于未映射之前，但该“ cprop”属性不是   :class:`_schema.Column` ,
它是   :class:`.ColumnProperty` ，这是一个中间对象，因此不具有
 :class:`_schema.Column` 对象或   :class:` .InstrumentedAttribute`  对象的全部功能，一旦完成声明过程，后者将映射到“ Bar”类。

虽然   :class:`.ColumnProperty`  有一个 ` `__clause_element__()`` 方法，它允许它在某些面向列的上下文中工作，
但在上面的示例中，它不能在开放式比较上下文中工作，例如将比较与数字“ 5”作为 SQL 表达式而不是常规 Python 比较。

解决方案是直接访问   :class:`_schema.Column` ，使用  :attr:` .ColumnProperty.expression` ：

    class Bar(Base):
        __tablename__ = "bar"

        id = Column(Integer, primary_key=True)
        cprop = deferred(Column(Integer))

        __table_args__ = (CheckConstraint(cprop.expression > 5),)

.. _error_cd3x:



.. _error_8s2b:

无法在撤消无效事务之前重新连接。请在继续之前完全回滚（）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此错误条件是指   :class:`_engine.Connection`  被撤销的情况，
无论是因为检测到数据库断开连接，还是因为调用  :meth:`_engine.Connection.invalidate` ，但仍存在一个事务，
该事务由  :meth:`_engine.Connection.begin`  方法显式发起，或由连接自动开始在 2.x 系列中发生，当发出任何 SQL 语句时自动开始
到不合法的状态。当连接无效时，任何   :class:`_engine.Transaction`  正在进行的状态现在都是无效的，
必须显式回滚才能从   :class:`_engine.Connection`  中删除它。

.. _error_9mi2:

A ForeignKey’’’ expected named arguments.  If 'name', ‘’’name, ‘constraint', or ‘’’constraintname’ in kwargs, use the corresponding, explicit argument.  If providing the constraint name inline, use name=<name>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在创建外键约束时，发生此错误，这是因为 SQLAlchemy 中的关键字参数更改了。
以如下这种格式传递外键约束：

    column = Column(Integer, ForeignKey(OtherTable.id))

你需要以以下 format 传外键约束：

    column = Column(Integer, ForeignKey(column=OtherTable.id))、


.. code-block::

 sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError)
 绑定参数组 1 中需要值 'b'。
 [SQL: u'INSERT INTO t (a, b, c) VALUES (?, ?, ?)']
 [parameters: [{'a': 1, 'c': 3, 'b': 2}, {'a': 2, 'c': 4}, {'a': 3, 'c': 5, 'b': 4}]]

由于“b”是必需的，因此将其传递为“None”，以使INSERT可以继续：

::

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

期望的是FROM子句，实际却是Select。创建FROM子句时，请使用.subquery()方法。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这涉及到SQLAlchemy 1.4新增的一个更改，其中由函数生成的SELECT语句（例如：func:_expression.select）以及包括联合和文本SELECT表达式等内容在内的SELECT语句不再被视为 _expression.FromClause 对象，并且不能直接放置在另一个SELECT语句的FROM子句中，除非它们首先被包装在一个.Subquery对象中。 这是核心中的一个重要概念变化，完整的理由在：ref:`change_4617`中进行了讨论。

示例如下：

    m = MetaData()
    t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))
    stmt = select(t)

在以上代码中，stmt 代表一个 SELECT 语句。当我们想要将 stmt 直接用作另一个 SELECT 语句中的FROM子句时，例如我们试图从中查询时，就会出现该错误：

    new_stmt_1 = select(stmt)

或者如果我们想将其用于 FROM 子句中，例如在 JOIN 中：

    new_stmt_2 = select(some_table).select_from(some_table.join(stmt))

在SQLAlchemy的早期版本中，使用SELECT语句嵌套另一个SELECT语句将会产生一个带有括号的未命名子查询。在许多情况下，MySQL 和 PostgreSQL 等数据库要求 FROM 子句中的子查询具有命名别名，这意味着需要使用 _expression.SelectBase.alias 方法或对于 1.4 版本以降使用 _expression.SelectBase.subquery 方法来进行。在其他数据库中，仍然更清楚地使用别名来解决子查询内列名称的所有歧义。

在实践中，此更改受到多个内部因素的启示，因此正确编写下述两条语句需要使用  :meth:`_expression.SelectBase.subquery`  ：

    subq = stmt.subquery()

    new_stmt_1 = select(subq)

    new_stmt_2 = select(some_table).select_from(some_table.join(subq))

.. seealso::

    :ref:`change_4617` 

.. _error_xaj1:

自动为裸clauseelement生成别名
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:：1.4.26

此警告是针对  :meth:`_orm.Query.join`  方法的常见原因，也适用于
  :meth:`_sql.Select.join`   这种"2.0形式"的方法，其中加入可以是
基于   :func:`_orm.relationship`  创建的，但目标是映射
到该类或   :func:`_orm.aliased`  构造的   :class:` _schema.Table` 或其他
核心选择。

例如：

    a1 = Address.__table__

    q = (
        s.query(User)
        .join(a1, User.addresses)
        .filter(Address.email_address == "ed@foo.com")
        .all()
    )

上面的模式还允许选择可选的可选项，例如核心   :class:`_sql.Join`  或   :class:` _sql.Alias`  对象，
但在这种情况下没有自动适应此元素的适配器，这意味着访问核心元素时需要直接引用。

    a1 = Address.__table__.alias()

    q = (
        s.query(User)
        .join(a1, User.addresses)
        .filter(a1.c.email_address == "ed@foo.com")
        .all()
    )

规定一个链接目标的正确方法是使用映射类本身或   :class:`_orm.aliased`  对象，对于后一种情况，则使用  :meth:` _orm.PropComparator.of_type`  修改器设置一个别名：

    # normal join to relationship entity
    q = s.query(User).join(User.addresses).filter(Address.email_address == "ed@foo.com")

    # name Address target explicitly, not necessary but legal
    q = (
        s.query(User)
        .join(Address, User.addresses)
        .filter(Address.email_address == "ed@foo.com")
    )

Join to an alias::

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

由于重叠的表，自动为别名生成别名
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:：1.4.26

使用  :meth:`_sql.Select.join`  方法或遗留的  :meth:` _orm.Query.join`  方法进行查询时，
生成此警告通常是关于使用多态继承的表，导致。问题在于，当两个继承模型中的其中之一更改时，两个表的修改可能同时发生。

例如，考虑下面的多态继承映射：

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

上述映射包括 Employee 和 Manager 类之间的关系。由于这两个类都使用 “employee” 数据库表，从 SQL 的角度来看，这是一个自联关系。如果我们想要使用 JOIN 从 Employee 和 Manager 模型查询数据，那么在 SQL 层面上，“employee”表需要两次出现在查询中，这意味着它必须被赋予别名。当我们使用 SQLAlchemy ORM 创建这样的 join 时，我们得到的 SQL 如下：

    >>> stmt = select(Employee, Manager).join(Employee.reports_to)
    >>> print(stmt)
    SELECT employee.id, employee.manager_id, employee.name,
    employee.type, manager_1.id AS id_1, employee_1.id AS id_2,
    employee_1.manager_id AS manager_id_1, employee_1.name AS name_1,
    employee_1.type AS type_1
    FROM employee JOIN
    (employee AS employee_1 JOIN manager AS manager_1 ON manager_1.id = employee_1.id)
    ON manager_1.id = employee.manager_id

上面的 SQL 使用"employee"表作为“Employee”实体中查询的 FROM，然后连接到右嵌套 JOIN“employee AS employee_1 JOIN manager AS manager_1”，其中“employee”表再次被声明，但作为匿名别名“employee_1”。这是警告信息所指的“自动生成别名”。

当SQLAlchemy从每个ORM行中适配一个Employee和Manager对象时，ORM必须将基于employee_1和manager_1的行调整为未别名的Manager类的行。此处理在内部较为复杂，不符合所有API功能，特别是尝试使用包含_eager等负载时，这些API功能在比这里更深层次的查询中。由于此模式不可靠且涉及难以预见和遵循的隐式决策制定，因此会发出警告，该模式可被视为遗留功能。因此，应显式使用   :func:`_orm.aliased`  构造。在继承并使用其他基于 join 的映射的情况下，通常最好增加使用  :paramref:` _orm.aliased.flat`  参数的设置，该参数允许将两个或更多表的 JOIN 通过将表单独用作别名而而不是内嵌 JOIN 来进行别名：

    from sqlalchemy.orm import aliased
    manager_alias = aliased(Manager, flat=True)
    stmt = select(Employee, manager_alias).join(Employee.reports_to.of_type(manager_alias))
    print(stmt)

要使用 EAGER LOAD 填充“reports_to”属性，必须引用别名：

    stmt = (
        select(Employee)
        .join(Employee.reports_to.of_type(manager_alias))
        .options(contains_eager(Employee.reports_to.of_type(manager_alias)))
    )

在非错误场景中，Session.rollback方法无条件到期会话中的所有内容，并且也应避免在没有错误的情况下进行使用。

.. seealso::

      :ref:`unitofwork_cascades` 

      :ref:`cascade_delete_orphan` 

      :ref:`error_bbf1` 



.. _error_bhk3:

一个事务已经回滚了，原因是在 flush 过程中遇到了上一个异常
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`Session` 内的 flush 过程在遇到错误时将回滚数据库事务以维护内部一致性。但是，一旦发生这种情况，该会话的事务现在处于“非活动”状态，并且必须由调用应用程序显式回滚以及它会提交某些修改。

这是ORM使用中经常遇到的错误，通常适用于在ORM的   :class:`.Session`  操作中尚未正确地安排其“框架”的应用程序。更多细节在   :ref:` faq_session_rollback`  中进行了描述。

.. _error_bbf0:

在关系<relationship>中“delete-orphan”级联通常仅在一对多关系的“一”侧上配置，而不是多对一或多对多关系的“多”侧。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当在多对一或多对多关系上设置“delete-orphan”级联时，将会出现此错误。例如：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)

        bs = relationship("B", back_populates="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

        # 当使用 mapper 配置步骤时，此将发出错误消息
        a = relationship("A", back_populates="bs", cascade="all, delete-orphan")


    configure_mappers()

以上，“B.a”上的“delete-orphan”设置表明，当每个引用特定“A”的每个“B”对象被删除时，该“A”应该被删除。也就是说，它表明正在表达“孤儿”将被删除的意图，这些孤儿是指一个“A”对象，在删除与之关联的所有“B”对象后，它将会成为一个孤儿。

“delete-orphan”级联模型不支持此功能。该“孤儿”考虑仅基于单个对象的删除，这将随后将所有引用其的零个或多个对象可视为“孤儿”，从而导致这些对象也被删除。换句话说，它仅设计为跟踪项的创建，这些项基于删除一个而仅有一个“父”对象完全成为孤儿的情况，这是在一对多关系中自然情况，其中在“many”侧上删除对象会导致“one”侧上相关项的后续删除。

此种 mapping 的正确应用方式是在一对多方面上放置级联设置，例如：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)

        bs = relationship("B", back_populates="a", cascade="all, delete-orphan")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

        a = relationship("A", back_populates="bs")

其中的意图是表明当删除“A”时，它所引用的所有“B”对象也将被删除。

然后，错误消息会继续建议使用  :paramref:`_orm.relationship.single_parent`  标志。此标志可用于强制对可以允许多个对象引用特定对象的关系的明确处理，因为在每个表之间存在外键关系而实现的关系，并且实际上仅有一个对象实际上都引用给定目标对象。在使用了此标志的遗留模式或其他较少理想的数据库模式中，较少的实际对象实际上会在特定目标对象阵列中引用，而在其他方面，该目标对象看起来可以具有许多“孩子”对象。这种不寻常的情况可以如下所示：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)

        bs = relationship("B", back_populates="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

        a = relationship(
            "A",
            back_populates="bs",
            single_parent=True,
            cascade="all, delete-orphan",
        )

上面的配置将安装验证器，该验证器强制将仅一个 B 与特定 A 相关联，范围在 B.a 关系内。

注意，该验证器的范围有限，并且不会防止从其他方向创建多个“父”。
因此，对于不是一对多关系的关系，更合适的方法是在一个或
多个关系中使用：paramref:`_orm.relationship.back_populates`。
也就是说，如果在这些关系中出现冲突，则应通过更改建模来解决。

总体而言，“delete-orphan”级联通常仅适用于一对多关系的“one”侧，以便在“many”侧上删除对象，而不是反过来。

.. versionchanged::1.3.18
    当在多对一或多对多关系上使用“delete-orphans”级联设置时，“delete-orphan”错误消息的文本已更新为更加具有描述性。


.. seealso::

      :ref:`unitofwork_cascades` 

      :ref:`cascade_delete_orphan` 

      :ref:`error_bbf0` 

.. _error_bbf1:

实例< instance > 已经通过其<attribute>属性与实例<instance>相关联，只允许一个父项。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当使用  :paramref:`_orm.relationship.single_parent`  标志时，如果多个对象同时作为对象的“parent”，则会发出此错误。

例如，考虑以下映射：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

        a = relationship(
            "A",
            single_parent=True,
            cascade="all, delete-orphan",
        )

其中表明不应多于一个“B”对象引用特定的“A”对象。

以下是安排多个对象作为对象的“parent”时会出现意外错误时，通常导致的错误类型是对   :ref:`error_bbf0`  消息的误解。请参阅该消息了解详细信息。

.. seealso::

      :ref:`error_bbf0` 

.. _error_qzyx:

关系 X 将 Q 列复制到 P 列，这与关系(s)：“Y”冲突。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此警告涉及到当两个或多个关系都将写入到相同的列上时，在ORM没有手段协调这些关系。
具体情况而定，解决方案可能是两个关系需要使用参数  :paramref:`_orm.relationship.back_populates`  相互连接，
或者其中一个或多个  :term:`single-table`  继承映射没有正确进行配置，可以使用继承条件（inherit condition）级联来解决此问题。等等...应该使用  :paramref:` _orm.relationship.viewonly`  来配置关系，以防止冲突写入，或者有时意图完全并且应该配置  :paramref:`_orm.relationship.overlaps`  来消除每个警告的影响。

对于通常缺少  :paramref:`_orm.relationship.back_populates`  的示例，请考虑下面的映射类::

    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        children = relationship("Child")


    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
        parent_id = Column(ForeignKey("parent.id"))
        parent = relationship("Parent")

上面的映射会产生警告：

.. sourcecode:: text

  SAWarning: relationship 'Child.parent' will copy column parent.id to column child.parent_id,
  which conflicts with relationship(s): 'Parent.children' (copies parent.id to child.parent_id).

关系 ``Child.parent`` 和 ``Parent.children`` 显然是相互冲突的。解决方案是应用  :paramref:`_orm.relationship.back_populates`  ：

    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        children = relationship("Child", back_populates="parent")


    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
        parent_id = Column(ForeignKey("parent.id"))
        parent = relationship("Parent", back_populates="children")

对于需要更自定义的关系，其中“overlap”情况可能是有意的，并且不能被解决，则可以通过  :paramref:`_orm.relationship.overlaps`  参数指定名称，其中警告不应发生作用。这通常发生在针对相同基础表的两个或多个关系的情况下，这些表包括限制每种情况下的相关项的自定义  :paramref:` _orm.relationship.primaryjoin`  条件的情况::

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

在上述代码中，ORM 将知道 ``Parent.c1``、``Parent.c2`` 和 ``Child.parent`` 之间的重叠是有意的。

.. _error_lkrp:

由于   :class:`_orm.Session`  已关闭或以其他方式调用了  :meth:` _orm.Session.expunge_all`  方法，因此无法将对象转换为“persistent”状态，因此该标识映射不再有效。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 1.4.26

此消息是为了适应在关闭或以其他方式调用其  :meth:`_orm.Session.expunge_all`  方法的   :class:` _orm.Session`  之后仍会迭代会生成 ORM 对象的   :class:`_result.Result`  对象的情况而添加的。当   :class:` _orm.Session`  一次性删除所有对象时，该  :term:`identity map`  内部使用的  :term:` identity map`  将被更换为新的，原始的部分会被丢弃。未使用的和未缓冲的   :class:`_result.Result`  对象将在内部保留对该现已丢弃的标识映射的引用。因此，当消耗   :class:` _result.Result`  对象时，将产生对象，这些对象不能与该   :class:`_orm.Session`  相关联。这是设计的排列，因为通常不建议在创建它的事务上下文之外迭代未缓冲的   :class:` _result.Result`  对象::

    # 上下文管理器创建新的 Session
    with Session(engine) as session_obj:
        result = sess.execute(select(User).where(User.id == 7))

    # 上下文管理器已关闭，因此 session_obj 在此处关闭，identity
    # 映射被替换

    # 迭代结果对象无法将对象与
    # Session 相关联，因此会引发此错误。
    user = result.first()

在上述情况下，通常不会发生使用 ``asyncio`` ORM 扩展，因为当   :class:`.AsyncSession`  返回类似于同步样式的   :class:` _result.Result`  时，结果已在执行该语句时预先缓冲。这是为了允许在不需要额外的 ``await`` 调用的情况下调用二次贪婪加载器。

要在使用常规的   :class:`_orm.Session`  中使用上面所述的结果预取方式模拟情况，可以使用 ` `prebuffer_rows`` 执行选项，如下所示::

    # 上下文管理器创建新的 Session
    with Session(engine) as session_obj:
        # 外部结果预取所有对象
        result = sess.execute(
            select(User).where(User.id == 7), execution_options={"prebuffer_rows": True}
        )

    # 上下文管理器已关闭，因此 session_obj 在此处关闭，identity
    # 映射被替换

    # 返回预缓冲对象
    user = result.first()

    # 但它们被分离，与已关闭的会话有关
    assert inspect(user).detached
    assert inspect(user).session is None

在上述代码中，所选的 ORM 对象完全在 ``session_obj`` 块内生成，与 ``session_obj`` 相关联，并在   :class:`_result.Result`  对象中缓冲以进行迭代。在块外部，` `session_obj`` 被关闭并且摆脱了这些 ORM 对象。迭代   :class:`_result.Result`  对象将提供这些 ORM 对象，但是由于它们的起始   :class:` _orm.Session`  已将它们删除，因此它们将以  :term:`分离`  状态传递。

.. 注意:: 以上有关“预缓冲”vs.“未缓冲”的   :class:`_result.Result`  对象的引用是指 ORM 如何从  :term:` DBAPI`  中转换传入的原始数据库行为 ORM 对象。它并不意味着底层“游标”对象本身是否已缓冲，因为这实际上是缓冲的较低层。有关 ``游标`` 结果自身的缓冲的背景信息，请参见   :ref:`engine_stream_results`  部分。

.. _error_zlpr:

引用声明性表格表单的类型注释不能解释
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0 引入了一种新的   :ref:`Annotated Declarative Table <orm_declarative_mapped_column>`  声明性系统，该系统通过在类定义中派生使用  :pep:` 484`  注释来注释 ORM 映射属性信息。此形式的要求是所有 ORM 注释必须使用名为   :class:`_orm.Mapped`  的通用容器才能被正确注释。使用显式  :pep:` 484`  类型注释的旧版 SQLAlchemy 映射，例如使用   :ref:`legacy Mypy 扩展 <mypy_toplevel>`  为类型支持的那些映射，可能包括不包括使用不包括此通用程序的   :func:` _orm.relationship`  的指令。

要解决此问题，可以在类被完全迁移到 2.0 语法之前，将这些类标记为具有 ``__allow_unmapped__`` 布尔属性。有关示例，请参见   :ref:`migration_20_step_six`  中的迁移说明.


.. 参见::

      :ref:`migration_20_step_six`  - 在   :ref:` migration_20_toplevel`  文档中

.. _error_dcmx:

将 <cls> 转换为数据类时发生 Python 数据类错误
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此警告是在   :ref:`orm_declarative_native_dataclasses`  中描述的 SQLAlchemy ORM Mapped Dataclasses 功能中添加的，结合非本身声明为数据类的 mixin 类或抽象基类使用时。例如，下面是一个包含非数据类 mixin 类的映射声明及其生成的警告信息：

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

如上所述，由于 ``Mixin`` 本身没有从   :class:`_orm.MappedAsDataclass`  派生，因此会生成以下警告：

.. sourcecode:: none

    SADeprecationWarning: When transforming <class '__main__.User'> to a
    dataclass, attribute(s) "create_user", "update_user" originates from
    superclass <class
    '__main__.Mixin'>, which is not a dataclass. This usage is deprecated and
    will raise an error in SQLAlchemy 2.1. When declaring SQLAlchemy
    Declarative Dataclasses, ensure that all mixin classes and other
    superclasses which include attributes are also a subclass of
    MappedAsDataclass.

解决方法是同样在 ``Mixin`` 签名中添加   :class:`_orm.MappedAsDataclass` ，如下所示::

    class Mixin(MappedAsDataclass):
        create_user: Mapped[int] = mapped_column()
        update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

Python 的  :pep:`681`  规范不包括在不是数据类的数据类超类上声明的属性。根据 Python 数据类的行为，类似地，这些字段将被忽略，例如，如以下示例所示::

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

上述 ``User`` 类将不在其构造函数中包含 ``create_user``，也不会尝试将 ``update_user`` 解释为数据类属性。这是因为 ``Mixin`` 不是数据类。

SQLAlchemy 2.0 系列中的数据类特性不正确解决此行为；反过来，数据类混合和具有 SQLAlchemy 映射属性的超类被视为最终的数据类配置的一部分。但是，Pyright 和 Mypy 等类型检查器不会将这些字段视为数据类构造函数的一部分，因为它们根据  :pep:`681`  要求应该被忽略。因此，由于其存在是模棱两可的，从 SQLAlchemy 2.1 开始，使用数据类继承 SQLAlchemy 映射属性时，必须将 mixin 类本身成为数据类。

.. _error_bupq:

使用按主键的 ORM 按行批量更新需要记录包含主键值的记录
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在使用类似以下用法的   :ref:`orm_queryguide_bulk_update`  功能时，如果记录中不提供主键值，则会发生此错误：

    >>> session.execute(
    ...     update(User).where(User.name == bindparam("u_name")),
    ...     [
    ...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"u_name": "patrick", "fullname": "Patrick Star"},
    ...     ],
    ... )

在上面的示例中，由于带有参数字典的列表结合   :class:`_orm.Session`  使用 ORM 执行了启用了 ORM 按主键的按行批量更新，因此参数字典必须包括主键值，例如::

    >>> session.execute(
    ...     update(User),
    ...     [
    ...         {"id": 1, "fullname": "Spongebob Squarepants"},
    ...         {"id": 3, "fullname": "Patrick Star"},
    ...         {"id": 5, "fullname": "Eugene H. Krabs"},
    ...     ],
    ... )

要在没有提供每行主键值的情况下调用 UPDATE 语句，可以使用  :meth:`_orm.Session.connection`  方法，以获取当前   :class:` _engine.Connection` ，然后使用该方法执行::

    >>> session.connection().execute(
    ...     update(User).where(User.name == bindparam("u_name")),
    ...     [
    ...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
    ...         {"u_name": "patrick", "fullname": "Patrick Star"},
    ...     ],
    ... )


.. 参见::

          :ref:`orm_queryguide_bulk_update` 

          :ref:`orm_queryguide_bulk_update_disabling` 



AsyncIO 异常
------------------

.. _error_xd1r:

AwaitRequired
~~~~~~~~~~~~~

SQLAlchemy 异步模式需要使用异步驱动程序连接到 db。
通常在试图使用非兼容  :term:`DBAPI`  时使用异步 SQLAlchemy 版本会引发此错误。

.. 参见::

      :ref:`asyncio_toplevel` 

.. _error_xd2s:

MissingGreenlet
~~~~~~~~~~~~~~~

在未预期的位置启动异步  :term:`DBAPI`  时，会发生此错误，通常是在某个 I/O 中尝试时，使用不直接提供使用 ` `await`` 关键字的调用模式。在使用 ORM 时，这几乎始终是由于使用了  :term:`lazy loading`  所引起的，而在 asyncio 中，则需要额外的步骤和/或备选装入器模式才能使用成功。

.. 参见::

      :ref:`asyncio_orm_avoid_lazyloads`  - 包括大多数 ORM 场景下可能发生该问题以及如何减轻影响的细节，包括与惰性加载场景一起使用的特定模式。

.. _error_xd3s:

No Inspection Available
~~~~~~~~~~~~~~~~~~~~~~~

在   :class:`_asyncio.AsyncConnection`  或   :class:` _asyncio.AsyncEngine`  对象上直接使用   :func:`_sa.inspect`  函数目前不受支持，因为还没有名称为   :class:` _reflection.Inspector`  的可等待对象。因此，可以通过以下方式获得引用：

使用   :func:`_sa.inspect`  函数，以便它引用   :class:` _asyncio.AsyncConnection.sync_connection`  属性的底层对象；然后使用   :class:`_asyncio.AsyncConnection.run_sync`  方法，以及执行所需操作的自定义函数来使用   :class:` _engine.Inspector` ：

    async def async_main():
        async with engine.connect() as conn:
            tables = await conn.run_sync(
                lambda sync_conn: inspect(sync_conn).get_table_names()
            )

.. 参见::

      :ref:`asyncio_inspector`  - 有关在 asyncio 扩展中使用   :func:` _sa.inspect`  的其他示例。


核心异常类
----------------------

有关核心异常类，请参见   :ref:`core_exceptions_toplevel` 。


ORM 异常类
---------------------

有关 ORM 异常类，请参见   :ref:`orm_exceptions_toplevel` 。


历史遗留异常
-----------------

本节中的异常不适用于当前 SQLAlchemy 版本，但在此提供以适应异常消息的超链接。

.. _error_b8d9:

在 SQLAlchemy 2.0 中，<某些函数> 将不再<某些行为>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0 对 Core 和 ORM 组件的众多关键用法模式进行了重大变化。2.0 发布的目标是在 SQLAlchemy 自从早期开始的一些最基本的假设中进行轻微的调整，并提供一个新的简化使用模型，这个模型希望显著更加极简主义，并且在 Core 和 ORM 组件之间更加一致，也更加具有可操作性。

在   :ref:`migration_20_toplevel`  中引入的 SQLAlchemy 2.0 项目包括综合未来兼容性的系统，该系统集成到 SQLAlchemy 1.4 系列中，这样应用程序就有了一个明确、明显的增量升级路径，以使其成为完全 2.0 兼容。  :class:` .exc.RemovedIn20Warning`  弃用警告是该系统的基础，提供有关现有代码库中需要修改的行为的指导。在   :ref:`deprecation_20_mode`  中可以找到如何启用此警告的概述。

.. 参见::

      :ref:`migration_20_toplevel`   - 有关从 1.x 系列升级的概述，以及当前的目标和进展情况。


      :ref:`deprecation_20_mode`  - 有关如何在 SQLAlchemy 1.4 中使用“2.0 废弃模式”的具体指南。


.. _error_s9r1:

此连接处于非活动事务中。请在继续之前执行回滚（）。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此错误的条件是在 SQLAlchemy 1.4 之后添加的，并且不适用于 SQLAlchemy 2.0。该错误是指在使用  :meth:`_engine.Connection.begin`  等方法将   :class:` _engine.Connection`  放入事务中，然后在该范围内的事务中创建了另一个“标记”事务；然后使用  :meth:`.Transaction.rollback`  或  :meth:` .Transaction.close`  回滚或关闭了内部事务，但外部事务仍处于“非活动”状态，并且必须回滚。

该模式类似于：

    engine = create_engine(...)

    connection = engine.connect()
    transaction1 = connection.begin()

    # 这是一个“子”或“标记”事务，一个基于“真实”事务 transaction1 的逻辑嵌套结构
    transaction2 = connection.begin()
    transaction2.rollback()

    # transaction1 仍然存在，并且需要显式回滚，
    # 因此这将引发错误。
    connection.execute(text("select 1"))

在上面的代码中，``transaction2`` 是一个“标记”事务，它表示外部事务的逻辑嵌套；虽然内部事务可以通过 rollback() 方法回滚整个事务，但是其 commit() 方法没有除了关闭“标记”事务自身的作用。调用 ``transaction2.rollback()`` 的效果是 **停用** transaction1，这意味着它在数据库级别上被回滚，但是仍然存在，以便适应事务的一致嵌套结构。

正确的解决方法是确保也回滚外部事务::

    transaction1.rollback()

在 Core 中不常用。在 ORM 中，也可以出现类似的问题，这是由 ORM 的“逻辑”事务结构引起的；这种情况下，请遵循以下建议进行操作。

这个错误在FAQ条目   :ref:`faq_session_rollback`  中有所描述。

在SQLAlchemy 2.0版本中，"subtransaction" 模式已被移除，因此这种编程模式不再可用，从而避免了这个错误信息的出现。