.. _connections_toplevel:

==============================
使用 Engine 和 Connections
==============================

.. module:: sqlalchemy.engine

本节详细介绍   :class:`_engine.Engine` ， :class:` _engine.Connection`和相关对象的直接使用。需要注意的是，
在使用SQLAlchemy ORM时，通常不会直接访问这些对象；相反，通常使用   :class:`.Session`  对象作为与数据库交互的接口。 
然而，对于那些围绕着直接使用文本SQL语句和/或SQL表达式构造物而建立的应用程序，在没有ORM的高级管理服务的情况下，
  :class:`_engine.Engine`  才是王（和女王？）——继续阅读。

基本用法
----------
 
回想一下 第  :doc:`/core/engines`  调用创建   :class:` _engine.Engine`  对象：

    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")

通常使用   :func:`_sa.create_engine`  一次创建一个特定的数据库URL，并保存为单个应用程序进程生命周期的全局变量。单个
  :class:`_engine.Engine`  管理代表进程的许多单独的  :term:` DBAPI`  连接，旨在以并发方式调用。   :class:`_engine.Engine`  
**不**等同于 DBAPI 的 ``connect()`` 函数，后者只表示一个连接资源 - 只创建一个   :class:`_engine.Engine`  对象
才是最有效的，而不是每个对象或每个函数调用。

.. sidebar:: 贴士

    在使用包含内容多个Python进程的   :class:`_engine.Engine` （例如使用“os.fork”或Python ` `multiprocessing``）时，
    重要的是要为每个进程初始化引擎。有关详情，请参阅   :ref:`pooling_multiprocessing` 。

  :class:`_engine.Engine`  最基本的功能是提供对   :class:` _engine.Connection`  的访问，然后可以调用 SQL 语句。如何发出对
数据库的文本语句看起来像这样::

    from sqlalchemy import text

    with engine.connect() as connection:
        result = connection.execute(text("select username from users"))
        for row in result:
            print("username:", row.username)

上面的  :meth:`_engine.Engine.connect`  方法返回一个   :class:` _engine.Connection`  对象，并通过在 Python 上下文管理器中 
（例如 ``with:`` 语句）使用它，  :meth:`_engine.Connection.close`  方法 在块结束时自动调用。   :class:` _engine.Connection`  
是实际 DBAPI 连接的**代理**对象。 DBAPI 连接从连接池中检索到在创建   :class:`_engine.Connection`  时。

返回的对象称为   :class:`_engine.CursorResult` ，引用了 DBAPI 游标，并提供类似于 DBAPI 游标的方法来检索行。当所有
结果行（如果有）都用完时，DBAPI 游标将被   :class:`_engine.CursorResult`  关闭。   :class:` _engine.CursorResult`  对象
不返回任何行，例如 UPDATE 语句（没有返回任何行），则这些游标资源在构建时立即释放。

在 ``with:`` block 结束时，将关闭   :class:`_engine.Connection` ，引用的 DBAPI 连接将 :term:` 释放`到连接池中。从数据库
的角度来看，假设池有空间将此连接存储以供下次使用，则连接池实际上不会“关闭”连接。当将连接返回池以供重新使用时，
连接池机制会在 DBAPI 连接上发出 ``rollback()`` 调用，以便删除任何事务状态或锁（这被称为   :ref:`pool_reset_on_return` )，
并准备好进行下一次使用。

我们上面的示例说明了如何执行文本 SQL 字符串，发出此类语句应使用   :func:`_expression.text`  构造对象，以表明我们希望
使用文本 SQL。当然，  :meth:`_engine.Connection.execute`  方法可以容纳更多内容；有关教程，请参见   :ref:` tutorial_working_with_data` 。

使用事务
----------

.. note::

  本节描述了在直接使用   :class:`_engine.Engine`  和   :class:` _engine.Connection`  对象时如何使用事务控制的公共 API。在使用 
  SQLAlchemy ORM 时，事务控制的公共 API 是通过   :class:`.Session`  对象实现的，该对象在内部使用   :class:` .Transaction`  对象。有关
  更多信息，请参见   :ref:`unitofwork_transaction` 。

按需提交
~~~~~~~~~

  :class:`~sqlalchemy.engine.Connection`  
方法以执行 SQL 语句时，此事务会自动启动，并使用称为 **autobegin** 的行为。事务在   :class:`_engine.Connection`  对象范围内保持不变，直
到调用  :meth:`_engine.Connection.commit`  或  :meth:` _engine.Connection.rollback`  方法。在结束事务后，   :class:`_engine.Connection`  对象等待 
调用  :meth:`_engine.Connection.execute`  方法，此时它将再次自动开始。

此调用样式称为 **按需提交**，下面的示例说明了此样式 ::

    with engine.connect() as connection:
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

        connection.commit()  # 提交事务

.. topic::  Python DBAPI 就是发生了 autopbegin
        
    “按需提交”的设计目的是与  :term:`DBAPI`  的设计相辅相成，后者是 SQLAlchemy 与之交互的底层数据库接口。在 DBAPI 中，
    ``connection`` 对象不假设更改数据库将自动提交，而是要求在默认情况下调用 ``connection.commit()`` 方法以便提交更改到数据库。 
    CN: DBAPI 的 ``connection`` 对象不假设更改数据库将自动提交，而是要求在默认情况下调用 ``connection.commit()`` 方法以便将更改提交到数据库，从而实现提交。
    必须注意的是，DBAPI 本身**根本没有begin()方法**。 所有Python DBAPI都实现了“autobegin”作为管理事务的基本方法，并在发出类似BEGIN 的语句时处理，
    当首次发出SQL语句时。 SQLAlchemy的API实际上是用更高级的Python对象重新陈述这种行为。
    
在“按需提交”的样式中，我们可以在  :meth:`_engine.Connection.execute`  的一系列其他语句中自由调用  :meth:` _engine.Connection.commit`  和 
  :meth:`_engine.Connection.rollback`   方法；每次事务结束并发出新语句时，都会隐式地开始一个新事务。

    with engine.connect() as connection:
        connection.execute("<some statement>")
        connection.commit()  # 提交“some statement”

        # 新的事务开始
        connection.execute("<some other statement>")
        connection.rollback()  # 回退“some other statement”
         
        # 新的事务开始
        connection.execute("<a third statement>")
        connection.commit()  # 提交“a third statement”

.. versionadded:: 2.0 "按需提交"样式是 SQLAlchemy 2.0的新功能。也可在使用“未来”样式引擎时
  （通过SQLAlchemy 1.4的“过渡”模式）使用它。

一旦提交，数据就不能再被回退。如果需要撤销部分提交，必须再次提交撤销 SQL 语句以及处理完成任务的 SQL 语句。

一次开始
~~~~~~~~~~~

  :class:`_engine.Connection`  对象提供了更明确的事务管理样式，称为 **一次开始**。与“按需提交”不同，
“一次开始”允许显式说明事务的开始点，并且允许将事务本身框架为上下文管理器块，以便事务结束变得隐式。使用“一次开始”时，
使用  :meth:`_engine.Connection.begin`  方法，它返回一个代表 DBAPI 事务的   :class:` .Transaction`  对象。此对象还支持通过 
  :meth:`_engine.Transaction.commit`   和  :meth:` _engine.Transaction.rollback`  方法进行显式管理，但作为一种首选的惯例，它还支持上下文管理器接口， 
在块结束时，它将正常自动提交并在引发异常时发送回滚以将异常传播出去。以下说明了“一次开始”块的形式：

    with engine.connect() as connection:
        with connection.begin():
            connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
            connection.execute(
                some_other_table.insert(), {"q": 8, "p": "this is some more data"}
            )

        # 事务已提交

连接和一次开始从 Engine 中开始
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一个便捷的简写形式是在发起调用时对  :meth:`_engine.Engine.begin`  进行调用，而不是执行两个单独的步骤  :meth:` _engine.Engine.connect`  
和  :meth:`_engine.Connection.begin`  ；:meth:` _engine.Engine.begin
` 方法返回一个特殊上下文管理器，内部维护了   :class:`_engine.Connection`  的上下文管理器和正常情况下由 
  :meth:`_engine.Connection.begin`   方法返回的-context manager class:` _engine.Transaction` 的上下文管理器::


    with engine.begin() as connection:
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

    # 事务已提交，并且 Connection 已释放到连接池中

.. tip::
    在  :meth:`_engine.Engine.begin`  块内，我们可以调用 
     :meth:`_engine.Connection.commit`  或  :meth:` _engine.Connection.rollback`  等方法，这将在块之前正常结束事务。
   但是，如果我们这样做了，则无法在   :class:`_engine.Connection`  上发出其他 SQL 操作，直到块结束::

        >>> from sqlalchemy import create_engine
        >>> e = create_engine("sqlite://", echo=True)
        >>> with e.begin() as conn:
        ...     conn.commit()
        ...     conn.begin()
        2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine BEGIN (implicit)
        2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine COMMIT
        Traceback (most recent call last):
        ...
        sqlalchemy.exc.InvalidRequestError: Can't operate on closed transaction inside
        context manager.  Please complete the context manager before emitting
        further commands.

样式混合
~~~~~~~~~~~

“按需提交”和“一次开始”样式可以自由混合在单个  :meth:`_engine.Engine.connect`  块内，只要调用  :meth:` _engine.Connection.begin`  时不会与 
“autobegin”行为发生冲突即可。 为了实现这一点，只有在未发出任何 SQL 语句之前或直接在前一个调用 
  :meth:`_engine.Connection.commit`   或  :meth:` _engine.Connection.rollback`  之后才应调用  :meth:`_engine.Connection.begin` 。

    with engine.connect() as connection:
        with connection.begin():
            # 在“一次开始”块中运行 SQL 语句
            connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})

        # 事务已提交

        # 在不使用块的情况下运行一个新语句。连接将进行“autobegin”
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

        # 显式提交
        connection.commit()

        # 可在此处使用“一次开始”块
        with connection.begin():
            # 运行更多 SQL 语句
            connection.execute(...)

当编写使用“一次开始”的代码时，库会在已经“自动开始”事务时引发   :class:`_exc.InvalidRequestError`  。

.. _dbapi_autocommit:

设置事务隔离级别，包括 DBAPI 自动提交
-------------------------------------------

大多数 DBAPI 支持可配置的事务  :term:`isolation`  级别的概念。这些通常是四个级别“READ UNCOMMITTED”、“READ COMMITTED”、
“REPEATABLE READ”和“SERIALIZABLE”。这些通常在 DBAPI 连接开始新事务之前应用。请注意，大多数 DBAPI 还支持真正的“自动提交”，
这意味着 DBAPI 连接本身将进入非事务性自动提交模式。这通常意味着 DBAPI 发送到数据库的“BEGIN”不再自动发出，但它还可能包括其他指令。
SQLAlchemy 将“自动提交”概念视为另一种事务隔离级别；因为它是一个失去了原子性的隔离级别。

.. tip::

  重要的是要注意，在下面的   :ref:`dbapi_autocommit_understanding`  部分中，“autocommit”隔离级别就像另一个事务隔离级别 
  一样，对   :class:`_engine.Connection`  对象的“事务性”行为没有影响，  :meth:` _engine.Connection.begin`  方法仍会调用 
  DBAPI ``.commit()`` 和 ``.rollback()`` 方法（在自动提交下，它们没有任何作用），对于 ``.begin()`` 方法，``DBAPI`` 
  会自动进行事务性开始（这意味着 SQLAlchemy 的“begin”**不会更改自动提交模式）。

SQLAlchemy 方言应尽可能支持这些隔离级别以及自动提交。

为一个连接设置 Isolation Level 或 DBAPI Autocommit
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以使用  :meth:`_engine.Connection.execution_options`  方法为单个   :class:` _engine.Connection`  
对象设置隔离级别的持续时间。该参数称为  :paramref:`_engine.Connection.execution_options.isolation_level`  列表，
是字符串，通常是以下名称的子集：

    # 可能的 Connection.execution_options(isolation_level="<value>") 的取值：

    "AUTOCOMMIT"
    "READ COMMITTED"
    "READ UNCOMMITTED"
    "REPEATABLE READ"
    "SERIALIZABLE"

不是每个 DBAPI 都支持每个值; 如果在某个后端使用了不支持的值，则会引发错误。

例如，要强制使用特定连接的 REPEATABLE READ，然后开始事务：

    with engine.connect().execution_options(
        isolation_level="REPEATABLE READ"
    ) as connection:
        with connection.begin():
            connection.execute("<statement>")

.. tip::
     :meth:`_engine.Connection.execution_options`  方法的返回值是调用该方法的   :class:` _engine.Connection`  对象本身，
    这意味着它会直接修改   :class:`_engine.Connection`  对象的状态，这是 SQLAlchemy 2.0 的新行为。该行为不适用于 
     :meth:`_engine.Engine.execution_options`  方法; 该方法仍将返回一个类似副本的   :class:` .Engine`  对象，就像下面所述，可以使用此方法
    为具有不同执行选项的多个   :class:`.Engine`  对象构建引擎，但它们共享相同的方言和连接池。

.. note::  :paramref:`_engine.Connection.execution_options.isolation_level`  参数不一定适用于语句级选项，例如 
    :meth:`_sql.Executable.execution_options` ，如果在该级别上设置，将被拒绝。这是因为该选项必须在 DBAPI 连接上基于每个事务设置。

为引擎设置隔离级别或 DBAPI 自动提交
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用  :meth:`_sa.create_engine.execution_options`  参数或 在  :meth:` _engine.Engine.execution_options`  方法中
可以对引擎宽度设置隔离级别，这通常是首选的。可以使用参数  :paramref:`_sa.create_engine.isolation_level`  传递给 
  :func:`.sa.create_engine`  来实现：

    from sqlalchemy import create_engine

    eng = create_engine(
        "postgresql://scott:tiger@localhost/test", isolation_level="REPEATABLE READ"
    )

使用上面的设置，每当创建新的 DBAPI 连接时，它会被设置为使用“REPEATABLE READ”隔离级别设置以进行所有后续操作。

.. _dbapi_autocommit_multiple:

对于单个引擎维护多个隔离级别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

隔离级别也可以针对引擎进行设置，这具有更大程度的灵活性，可以使用  :paramref:`_sa.create_engine.execution_options`  
参数或  :meth:`_engine.Engine.execution_options`  方法。后者将创建原始引擎的副本，这些副本具有相同的方言和连接池，但具有自己 
独立的每个连接的隔离级别设置::

    from sqlalchemy import create_engine

    eng = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test",
        execution_options={"isolation_level": "REPEATABLE READ"},
    )

使用上面的设置，每当开始新事务时，DBAPI 连接都将被设置为使用“REPEATABLE READ”隔离级别设置；但是连接作为池将重置为
首次出现时存在的原始隔离级别。 在   :func:`_sa.create_engine`  的级别上，最终效果与使用 
  :paramref:`_sa.create_engine.isolation_level`   参数没有任何区别。

然而，经常选择在不同隔离级别下运行操作的应用程序可能希望创建超级   :class:`_engine.Engine`  的多个 "子引擎"，每个 "子引擎" 
将配置为不同的隔离级别。一个这样的用例是具有 "事务性" 和 "只读" 操作的操作的应用程序，可以从主引擎中分离出一个额外的 
  :class:`_engine.Engine` ，它使用 ` `"AUTOCOMMIT"`` 可以单独执行。

    from sqlalchemy import create_engine

    eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

上面，  :meth:`_engine.Engine.execution_options`   方法创建原始   :class:` _engine.Engine`  的浅层副本。 ``eng`` 和 
``autocommit_engine`` 共享相同的方言和连接池。但是，当从 autocommit_engine 获取连接时，将为它们设置“AUTOCOMMIT”模式。

无论使用哪种方法，不管隔离级别如何，当一个连接返回到连接池时，隔离级别都会被无条件地恢复。

.. seealso::

        :ref:`SQLite Transaction Isolation <sqlite_isolation_level>` 

        :ref:`PostgreSQL Transaction Isolation <postgresql_isolation_level>` 

        :ref:`MySQL Transaction Isolation <mysql_isolation_level>` 

        :ref:`SQL Server Transaction Isolation <mssql_isolation_level>` 

        :ref:`Oracle Transaction Isolation <oracle_isolation_level>` 

        :ref:`session_transaction_isolation`  -  ORM

        :ref:`faq_execute_retry_autocommit`  -  一种使用 DBAPI 自动提交连接的配方，以便对只读操作进行透明重新连接

.. _dbapi_autocommit_understanding:

理解 DBAPI 级别的自动提交隔离级别行为
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在上述部分中，我们介绍了  :paramref:`_engine.Connection.execution_options.isolation_level`  参数的概念，以及如何使用它来设置数据库
隔离级别，包括 SQLAlchemy 级别的事务隔离级别、包括 DBAPI 级别的“自动提交”隔离级别。
在本节中，我们将尝试澄清此方法的影响。

如果我们想要检出   :class:`_engine.Connection`  对象并在“autocommit”模式下使用它，我们会按如下方式进行：

    with engine.connect() as connection:
        connection.execution_options(isolation_level="AUTOCOMMIT")
        connection.execute("<statement>")
        connection.execute("<statement>")

上面说明了“DBAPI 自动提交”模式的正常用法。没有必要使用  :meth:`_engine.Connection.begin`  或
  :meth:`_engine.Connection.commit`   方法，因为所有语句都会立即提交到数据库中。当块结束时，   :class:` _engine.Connection`  
对象将恢复“自动提交”隔离级别，并将 DBAPI 连接释放到连接池中，DBAPI 的 ``connection.rollback()`` 方法通常将被调用，
但由于上述语句已提交，因此此回滚对数据库状态没有影响。

重要的是要注意，“自动提交”模式即使在调用  :meth:`_engine.Connection.begin`  时也会持续存在；
DBAPI 不会向数据库发出任何 BEGIN，也不会在调用  :meth:`_engine.Connection.commit`  时发出 COMMIT。这种使用也不是错误场景，
因为毕竟“隔离级别”是事务本身的配置细节，就像任何其他隔离级别一样。

在下面的示例中，语句保持自动提交，无论 SqlAlchemy 级别的事务块是否存在：

    with engine.connect() as connection:
        connection = connection.execution_options(isolation_level="AUTOCOMMIT")

        # “transaction”被自动开始（但由于自动提交，无效果）
        connection.execute("<statement>")

        # 这将引起错误，因为“transaction”已经开始了
        with connection.begin() as trans:
            connection.execute("<statement>")

上面的示例还演示了与前面提到的相同主题，即“自动提交”隔离级别是底层数据库事务的一项配置细节，并且独立于 SQLAlchemy 
的  :meth:`_engine.Connection.begin`  和 |meth:` _engine.Connection.commit` 方法的行为。这两种方式对于 SQLAlchemy 
的   :class:`_engine.Connection`  的取用方法来说没有任何影响；该“autocommit”模式可以独立地在其他位置更改，而无需表明其他不同。

更改隔离级别
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. topic:: TL;DR;

    更喜欢使用单个隔离级别的单个   :class:`_engine.Connection`  对象，而不是在单个   :class:` _engine.Connection`  上切换隔离级别。代码会更易读，更少出现错误。

隔离级别设置，包括自动提交模式，会在将连接释放回到连接池时自动重置。因此，最好不要尝试在单个   :class:`_engine.Connection`  
对象上切换隔离级别，因为这会导致过多的冗余 verbosity。

为说明如何在单个   :class:`_engine.Connection`  检出的范围内使用即兴的“自动提交”模式，必须重新应用 
  :paramref:`_engine.Connection.execution_options.isolation_level`   参数以前隔离级别。上一个部分说明了一个尝试调用 
  :meth:`_engine.Connection.begin`   来开始事务，而在自动提交正在进行的情况下，我们可以重写该示例来实现：

    # 如果我们想在单个连接上打开自动提交和关闭自动提交，我们应该这样做...
    with engine.connect() as connection:
        connection.execution_options(isolation_level="AUTOCOMMIT")

        # 在自动提交模式下运行语句
        connection.execute("<statement>")

        # “提交”自动启动 “transaction。”
        connection.commit()

        # 切换到默认隔离级别
        connection.execution_options(isolation_level=connection.default_isolation_level)

        # 使用begin block
        with connection.begin() as trans:
            connection.execute("<statement>")

上面，在手动恢复隔离级别时，我们使用了  :attr:`_engine.Connection.default_isolation_level`  以恢复默认隔离级别
（假设这就是我们在这里想要的）。但是，这可能更好地使用   :class:`_engine.Connection`  的体系结构， 利用它在检入时自动处理隔离级别重置的现有工作。
写入上述代码的**首选**方法是使用两个块：

    # 使用自动提交块
    with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as connection:
        # 在自动提交模式下运行语句
        connection.execute("<statement>")

    # 使用常规块
    with engine.begin() as connection:
        connection.execute("<statement>")

总结：

1.  “DBAPI 级别的自动提交”隔离级别完全独立于   :class:`_engine.Connection`  对象的“开始”和“提交”概念。
2.  使用单个   :class:`_engine.Connection`  检出每个隔离级别。 避免尝试在单个连接检出之间来回切换“自动提交”； 让引擎做恢复默认隔离级别的工作。


.. _engine_stream_results:

使用服务器端光标（又称流式结果）
--------------------------------------------一些后端支持“服务器端游标”和“客户机端游标”的概念。客户端游标在这里意味着在结果集返回之前，数据库驱动程序将所有行完全提取到内存中。像 PostgreSQL 和 MySQL/MariaDB 这样的驱动程序通常默认使用客户端游标。相比之下，“服务器端游标”表示结果行在客户端消耗时仍保留在数据库服务器的状态中。例如，Oracle 的驱动程序通常使用“服务器端”模型，而 SQLite 方言虽然不使用真正的“客户端/服务器”架构，但仍使用未缓冲的结果提取方法，在消耗结果行之前，将其保留在进程内存之外。

.. topic:: 其实我们的意思是“缓存”与“不缓存”结果

  服务器端游标还暗示了与关系数据库相关的更广泛的功能，例如滚动光标向前和向后的能力。SQLAlchemy 不包括任何针对这些行为的明确支持；在 SQLAlchemy 本身中，通用术语“服务器端游标”应被视为“未缓冲的结果”，而“客户端游标”表示“在返回第一行之前，结果行被缓冲到内存中”。要使用特定于某个 DBAPI 驱动程序的更丰富的“服务器端游标”特性集，请参见   :ref:`dbapi_connections_cursor`  部分。

从这个基本架构出发，可以得出“服务器端游标”在提取非常大的结果集时更加内存高效，同时可能引入更复杂的客户端/服务器通信过程，并且对小结果集 (通常少于 10000 行) 的效率不高。

对于那些有条件支持缓冲或未缓冲结果的方言，通常有使用“未缓冲”或服务器端游标模式的警告。例如，使用 psycopg2 方言时，如果使用服务器端游标对任何类型的 DML 或 DDL 语句进行了设置，就会引发错误。使用服务器端游标的 MySQL 驱动程序处于更为脆弱的状态，它不会像缓存式游标那样优雅地从错误条件中恢复，也不允许在游标完全关闭之前执行回滚。

因此，SQLAlchemy 的方言始终默认使用更少出错的游标，因此对于 PostgreSQL 和 MySQL 方言，它默认使用缓存、“客户端”游标，在从游标调用任何获取方法之前，将完整的结果集拉入内存。这种操作模式适用于 **绝大多数** 情况；未缓冲的游标通常只在应用程序以很大的块获取大量行的不寻常情况下才有用，这些行的处理在获取更多行之前就可以完成。

对于提供客户端和服务器端游标选项的数据库驱动程序，  :paramref:`_engine.Connection.execution_options.stream_results`   和  :paramref:` _engine.Connection.execution_options.yield_per`  执行选项在基于   :class:`_engine.Connection`  或语句的基础上提供“服务器端游标”的访问。使用 ORM   :class:` _orm.Session`  时也存在类似的选项。

通过 yield_per 使用固定大小的缓冲进行流处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于每个完全未缓存的服务器端游标的单个行提取操作通常比一次提取多行的批处理更昂贵，因此  :paramref:`_engine.Connection.execution_options.yield_per`  执行选项将为使用服务器端游标的   :class:` _engine.Connection`  或语句配置一定的以行批为单位的固定大小缓冲，以便在消耗它们时从服务器中检索这些行。可以使用  :meth:`_engine.Connection.execution_options`  方法在   :class:` _engine.Connection`  上或在语句上使用  :meth:`.Executable.execution_options`  方法将此参数配置为正整数值。

.. versionadded:: 1.4.40  :paramref:`_engine.Connection.execution_options.yield_per`  属于 Core 级别的选项，是 SQLAlchemy 1.4.40 新增的；对于之前的 1.4 版本，要结合  :paramref:` _engine.Connection.execution_options.stream_results`  和   :meth:`_engine.Result.yield_per`  使用。

使用此选项等效于手动设置  :paramref:`_engine.Connection.execution_options.stream_results`  选项（下文将描述为“stream”选项），然后调用  :meth:` _engine.Result.yield_per`  方法并为给定的整数值。在这两种情况下，此组合的效果包括：

* 选择了给定后端的服务器端游标模式（如果可用且不是该后端的默认行为）
* 由于获取结果行时，它们将被分批缓冲，因此在调用  :paramref:`_engine.Connection.execution_options.yield_per`  选项或  :meth:` _engine.Result.yield_per`  方法之前，每个批的大小将等于传递给它们的整数参数；然后最后一批与剩下的行的大小相同或更少
* 如果使用那个函数，则  :meth:`_engine.Result.partitions`  方法使用的默认分区大小，将与该整数大小相同。

下面示例说明这三种行为：

.. sourcecode:: pycon

    with engine.connect() as conn:
        with conn.execution_options(yield_per=100).execute(
            text("select * from table")
        ) as result:
            for partition in result.partitions():
                # partition is an iterable that will be at most 100 items
                for row in partition:
                    print(f"{row}")

上面的示例演示了组合使用 ``yield_per=100`` 以及使用  :meth:`_engine.Result.partitions`  方法在与从服务器检索的大小匹配的批处理中运行处理。使用  :meth:` _engine.Result.partitions`  是可选的，如果直接迭代   :class:`_engine.Result` ，则会为每 100 行获取一批新行。不应使用  :meth:` _engine.Result.all`  等方法，因为这样会立即获取所有剩余行，从而破坏使用 ``yield_per`` 的目的。

.. tip::

    可以像上面示例中那样使用   :class:`.Result`  对象作为上下文管理器。使用服务器端游标迭代时，这是确保   :class:` .Result`  对象被关闭的最佳方法，即使在迭代过程中引发异常。

  :paramref:`_engine.Connection.execution_options.yield_per`   选项也可移植到 ORM 中，由   :class:` _engine.Engine`  使用，然后将该引擎传递给   :class:`_orm.Session` 。在使用   :class:` _orm.Session`  获取 ORM 对象时也会使用此选项，该选项还限制了一次生成的 ORM 对象的数量。有关如何在 ORM 中使用  :paramref:`_engine.Connection.execution_options.yield_per`  的背景信息，请参见   :ref:` orm_queryguide_yield_per`  部分，其位于   :ref:`queryguide_toplevel`  中。

.. versionadded:: 1.4.40 新增了  :paramref:`_engine.Connection.execution_options.yield_per`  作为 Core 级别的执行选项，以方便设置流式结果、缓冲区大小和分区大小，与 ORM 的类似用例相对应。

.. _engine_stream_results_sr:

使用 stream_results 动态增长的缓冲区进行流处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

要使用 stream_results 选项启用服务器端游标而不指定分区大小，可以使用  :paramref:`_engine.Connection.execution_options.stream_results`  选项，在   :class:` _engine.Connection`  对象或语句对象上调用该选项。

当使用  :paramref:`_engine.Connection.execution_options.stream_results`  选项交付的   :class:` _engine.Result`  对象直接进行迭代时，将使用一种默认缓冲方案内部获取行，该方案首先缓冲少量行，然后在每次获取时缓冲越来越多的行，直到达到预配置的限制 1000 行。使用  :paramref:`_engine.Connection.execution_options.max_row_buffer`  执行选项可以影响此缓冲区的最大大小::

    with engine.connect() as conn:
        with conn.execution_options(stream_results=True, max_row_buffer=100).execute(
            text("select * from table")
        ) as result:
            for row in result:
                print(f"{row}")

虽然  :paramref:`_engine.Connection.execution_options.stream_results`  选项可以与使用  :meth:` _engine.Result.partitions`  方法结合使用，但应为  :meth:`_engine.Result.partitions`  方法提供特定的分区大小，以便不获取整个结果。使用  :paramref:` _engine.Connection.execution_options.yield_per`  选项通常更直接。

.. seealso::

      :ref:`orm_queryguide_yield_per`  - 位于   :ref:` queryguide_toplevel` 

     :meth:`_engine.Result.partitions` 

     :meth:`_engine.Result.yield_per` 

.. _schema_translating:

翻译模式名称
---------------------------

为支持将常见的表分发到多个架构的多租户应用程序，可以使用  :paramref:`.Connection.execution_options.schema_translate_map`  执行选项，将一组   :class:` _schema.Table`  对象重新用于不同的模式名称，而不需要进行任何更改。

给定一个表::

    user_table = Table(
        "user",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
    )

“模式”由  :paramref:`_schema.Table.schema`  属性定义，例如   :class:` _schema.Table`  这个没有。  :paramref:`.Connection.execution_options.schema_translate_map`   可以指定，在未指定模式的情况下，所有   :class:` _schema.Table`  对象都将被呈现为不同于 ``user_schema_one`` 架构的模式::

    connection = engine.connect().execution_options(
        schema_translate_map={None: "user_schema_one"}
    )

    result = connection.execute(user_table.select())

上面的代码将在数据库上执行类似如下的 SQL：

.. sourcecode:: sql

    SELECT user_schema_one.user.id, user_schema_one.user.name FROM
    user_schema_one.user

也就是说，方案名称已经被替换为我们的翻译名称。映射可以指定任意数量的目标->目标模式：

    connection = engine.connect().execution_options(
        schema_translate_map={
            None: "user_schema_one",  # no schema name -> "user_schema_one"
            "special": "special_schema",  # schema="special" becomes "special_schema"
            "public": None,  # Table objects with schema="public" will render with no schema
        }
    )

  :paramref:`.Connection.execution_options.schema_translate_map`   参数影响从 SQL 表达式语言生成的所有 DDL 和 SQL constructs，这些 constructs 是从   :class:` _schema.Table`  或   :class:`.Sequence`  表格派生的。它不会影响通过   :func:` _expression.text`  构造的文字字符串 SQL，也不会影响传递了纯字符串到  :meth:`_engine.Connection.execute`  方法的操作。

此功能仅在那些直接从   :class:`_schema.Table`  或   :class:` .Sequence`  中派生模式名称的情况下发生作用，并且不会影响   :class:`_reflection.Inspector`  对象的操作，因为模式名称是显式传递给这些方法的。 

.. tip::

    要在 ORM   :class:`_orm.Session`  上使用翻译方案特性，请在   :class:` _engine.Engine`  级别设置该选项，然后将该引擎传递给   :class:`_orm.Session` 。  :class:` _orm.Session`  为每个事务使用一个新   :class:`_engine.Connection` 。

    .. warning::

      默认情况下，使用 ORM   :class:`_orm.Session` ，仅支持 **每个会话的单个模式转换映射**，如果在每个语句的基础上给出了不同的模式转换映射，则它将 **无效**，因为 ORM   :class:` _orm.Session`  不考虑当前模式转换值。要使用具有多个 ``schema_translate_map`` 配置的单个   :class:`_orm.Session` ，可以使用   :ref:` horizontal_sharding_toplevel`  扩展。有关示例，请参见   :ref:`examples_sharding` 。

.. _sql_caching:

SQL 编译缓存
-----------------------

.. versionadded:: 1.4 SQLAlchemy 现在具有透明的查询缓存系统，在 Core 和 ORM 之间都能显著降低 Python 计算开销。有关详细信息，请参见   :ref:`change_4639` 。

SQLAlchemy 包括用于 SQL 编译器及其 ORM 变体的全面缓存系统。该缓存系统在   :class:`.Engine`  内部是透明的，它提供了一个特定的 Core 或 ORM SQL 语句的 SQL 编译过程，以及与该语句相关的计算结果，在该语句对象和具有相同结构的所有语句对象中只会发生一次，在引擎的“编译缓存”中。按照“具有相同结构的语句对象”，这通常对应于在函数中构造的 SQL 语句，并在每次运行该函数时构建::

    def run_my_statement(connection, parameter):
        stmt = select(table)
        stmt = stmt.where(table.c.col == parameter)
        stmt = stmt.order_by(table.c.id)
        return connection.execute(stmt)

上面的语句将生成类似于“SELECT id, col FROM table WHERE col = :col ORDER BY id”的 SQL，尽管“parameter” 的值是纯 Python 对象，例如字符串或整数，但字符串 SQL 形式不包括这个值，它使用绑定的参数。随后的对于上面的“run_my_statement()”函数的调用将使用长期有效的编译结构在“connection.execute()”调用的范围内，以提高性能。

.. note:: 重要的一点是，SQL 编译缓存仅缓存传递给数据库的**SQL字符串而已**，并且**不缓存**查询返回的数据。它不以任何方式是一个数据缓存，并且不会与获取结果行相关的任何内存使用。

虽然 SQLAlchemy 早期 1.x 系列自带了简单的语句缓存，但另外还提供了“Baked Query”扩展来提高缓存的效率，但是这些系统都需要高度特殊的 API 以使缓存生效。 2020 年 7 月开始，1.4 版本引入了新的缓存系统，这个缓存系统完全自动且不需要任何特殊步骤才能生效。

这个缓存系统会透明地在   :class:`._Engine`  内部使用类似的机制（由底层编译缓存机制驱动）。那么一个给定的 Core 或 ORM SQL 声明语句的 SQL 编译过程，以及为该语句集装配检索结果机制的相关计算，将仅针对该构造对象的某个具体结构执行一次，并保留在特定结构在引擎的“编译缓存”中的持续时间内。所有 SQLAlchemy 支持的 struct‌　类型都被缓存，including   :class:` .Select` .Insert`构造、  :class:`.Update` .Delete` 构造等。

配置
~~~~~~~~~~~~~

缓存本身是类似字典的对象，称为“LRUCache”。这是一个内部 SQLAlchemy 字典子类，可跟踪特定键的使用情况，并在缓存大小达到一定阈值时执行定期“修剪”步骤，以删除最近未使用的项目。缓存的大小默认为 500 ，可以使用  :paramref:`_sa.create_engine.query_cache_size`  参数进行配置：

    engine = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test", query_cache_size=1200
    )

缓存的大小可以增长到给定大小的 150%，然后被修剪回目标大小。 一个缓存大小为 1200 的缓存因此可以增长到 1800 个元素大小，然后将被剪回到 1200 个大小。

缓存的大小基于每个唯一的渲染 SQL 语句，按引擎分配一个条目。 从 Core 和 ORM 生成的 SQL 语句被视为相同。 DDL 语句通常不会被缓存。 为了确定缓存正在执行什么操作，引擎记录会包括有关缓存行为的详细信息，在下一节中描述。

.. _sql_caching_logging:

使用日志记录估计缓存性能
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

上面的 1200 缓存大小相当大。对于小型应用程序，大小 100 可能已足够。为了估计缓存的最佳大小，假设目标主机上存在足够的内存，则缓存的大小应基于可以为目标引擎呈现的唯一 SQL 字符串的数量。最直接的启用 SQL 回显的方法可能是使用  :paramref:`_sa.create_engine.echo`  标志，或使用 Python 日志；有关日志配置的背景，请参见   :ref:` dbengine_logging`  部分。

例如，我们将检查以下程序生成的日志：

.. sourcecode:: pycon

    from sqlalchemy import Column
    from sqlalchemy import create_engine
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import select
    from sqlalchemy import String
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy.orm import relationship
    from sqlalchemy.orm import Session

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        data = Column(String)
        bs = relationship("B")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))
        data = Column(String)


    e = create_engine("sqlite://", echo=True)
    Base.metadata.create_all(e)

    s = Session(e)

    s.add_all([A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()])])
    s.commit()

    for a_rec in s.scalars(select(A)):
        print(a_rec.bs)

当运行时，每个日志语句都会在参数左侧包含括号括起来的缓存统计。可以看到，我们可能会看到以下四种类型的消息：

* “[raw sql]”-驱动程序或最终用户使用  :meth:`.Connection.exec_driver_sql`  直接向数据库发送原始 SQL-缓存不适用
* “[no key]”-语句对象是DDL语句，因此不会缓存；或者语句对象包含不可缓存元素，例如用户定义构造或任意的大操作数 VALUES 子句。
* “[generated in Xs]”- 该语句是一个**缓存未命中**语句，并且必须被编译，然后存储在缓存中。编译构造的时间为 X 秒。数字 X 将是非常小的小数秒数。
* “[cached since Xs ago]”-该语句是一个**缓存命中**语句，并且无需重新编译即可获得。该语句自 X 秒前开始存储在缓存中。数字 X 与应用程序运行时间和语句驻留缓存中的时间长相应关，例如可以为 24 小时的期间达到 86400。

下面更详细地描述了每个徽章。

我们会看到的前两个语句是 SQLite 方言检查 “a” 和 “b” 表的存在：

.. sourcecode:: text

  INFO sqlalchemy.engine.Engine PRAGMA temp.table_info("a")
  INFO sqlalchemy.engine.Engine [raw sql] ()
  INFO sqlalchemy.engine.Engine PRAGMA main.table_info("b")
  INFO sqlalchemy.engine.Engine [raw sql] ()

对于上述两个 SQLite PRAGMA 语句，徽章为“[raw sql]”，这表示驱动程序正在使用  :meth:`.Connection.exec_driver_sql`  直接将 Python 字符串发送到数据库中。不会对此类语句执行缓存，因为它们已以字符串形式存在，而且不知道将返回哪些类型的结果行，因为 SQLAlchemy 不会提前解析 SQL 字符串。

接下来我们看到了 CREATE TABLE 语句：

.. sourcecode:: sql

  INFO sqlalchemy.engine.Engine
  CREATE TABLE a (
    id INTEGER NOT NULL,
    data VARCHAR,
    PRIMARY KEY (id)
  )

  INFO sqlalchemy.engine.Engine [no key 0.00007s] ()
  INFO sqlalchemy.engine.Engine
  CREATE TABLE b (
    id INTEGER NOT NULL,
    a_id INTEGER,
    data VARCHAR,
    PRIMARY KEY (id),
    FOREIGN KEY(a_id) REFERENCES a (id)
  )

  INFO sqlalchemy.engine.Engine [no key 0.00006s] ()

对于这两个语句，徽章为“[no key 0.00006s]”。这表示这两个特殊语句均未缓存，原因是 DDL 相关的   :class:`_schema.CreateTable`  构造没有生成缓存键。DDL 构造通常不参与缓存，因为它们通常不会被重复使用，而且 DDL 还是数据库配置步骤，性能不是那么重要。

“[no key]” 徽章也非常重要，因为它可能会为可缓存的 SQL 语句生成，但某些子构造当前不能缓存而产生。此类示例包括没有定义缓存参数的自定义用户定义 SQL 元素，以及生成任意长且不可复制的 SQL 字符串的某些构造形式，主要是   :class:`.Values`  构造和使用  :meth:` .Insert.values`  方法的“多属性插入”。

到目前为止，我们的缓存仍为空。然后缓存的语句将被缓存，一个段落看起来像：

.. sourcecode:: sql

  INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (?)
  INFO sqlalchemy.engine.Engine [generated in 0.00011s] (None,)
  INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (?)
  INFO sqlalchemy.engine.Engine [cached since 0.0003533s ago] (None,)
  INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (?)
  INFO sqlalchemy.engine.Engine [cached since 0.0005326s ago] (None,)
  INFO sqlalchemy.engine.Engine INSERT INTO b (a_id, data) VALUES (?, ?)
  INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1, None)
  INFO sqlalchemy.engine.Engine INSERT INTO b (a_id, data) VALUES (?, ?)
  INFO sqlalchemy.engine.Engine [cached since 0.0003232s ago] (1, None)
  INFO sqlalchemy.engine.Engine INSERT INTO b (a_id, data) VALUES (?, ?)
  INFO sqlalchemy.engine.Engine [cached since 0.0004887s ago] (1, None)

上面，我们真正看到的是两个唯一的 SQL 字符串：“INSERT INTO a (data) VALUES (?)”以及“INSERT INTO b (a_id, data) VALUES (?, ?)”在参数是完全不同的情况下被重复使用，由于 SQLAlchemy 对所有行值使用绑定参数，因此即使这些语句多次重复使用，SQL 语句的实际字符串保持不变。

.. note:: 上面的两个语句是由 ORM 工作单元过程生成的，并且实际上将在每个映射器上缓存使用本地缓存。但是机制和术语是一样的。  :ref:`engine_compiled_cache`  将描述用户面向代码如何使用可在线程间共享其缓存容器。

对于上面两个唯一的 SQL 字符串，我们可以看到第一次出现每个语句时的缓存徽标是“[generated in 0.00011s]”。这表示该语句是**不在缓存中的，它被编译为一个字符串，需要 0.00011 秒，并且然后被缓存**。当我们使用“[generated]”徽章时，我们知道这意味着缓存未命中。这可以预期为特定语句的首次出现。然而，如果在长时间运行的应用程序中看到了许多新的“[generated]”标记，而该应用程序通常一遍又一遍地使用同一系列 SQL 语句，则表示  :paramref:`_sa.create_engine.query_cache_size`  参数可能太小。当从缓存中删除时，如果该语句被剪掉，则显示“[generated]”标记，则下次使用它时，它将再次导致缓存未命中。

我们示例程序中的程序执行一些查询，我们可以看到相同的“生成”然后是“缓存”，就如“a”表的 select 以及之后的“b”表的后续的懒惰加载：

.. sourcecode:: text

  INFO sqlalchemy.engine.Engine SELECT a.id AS a_id, a.data AS a_data
  FROM a
  INFO sqlalchemy.engine.Engine [generated in 0.00009s] ()
  INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
  FROM b
  WHERE ? = b.a_id
  INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1,)
  INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
  FROM b
  WHERE ? = b.a_id
  INFO sqlalchemy.engine.Engine [cached since 0.0005922s ago] (2,)

INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id

从上面的程序可以看出，总共缓存了4个不同的SQL语句。这表明缓存大小为4就足够了。显然，这个大小非常小，通常可以将默认大小500保持不变。

缓存使用了多少内存？
~~~~~~~~~~~~~~~~

上一节详细介绍了一些技术，用于检查:paramref: _sa.create_engine.query_cache_size是否需要更大。我们如何知道缓存不太大？我们可能希望将:paramref: _sa.create_engine.query_cache_size设置为不高于某个特定数字，因为我们有一个可能会使用大量不同语句的应用程序，例如从搜索UX动态构建查询，并且我们不希望如果过去24小时运行了十万个不同的查询并且它们都被缓存，主机就会耗尽内存。

测量Python数据结构占用的内存极其困难，不过使用一个进程来度量通过连续一系列添加到缓存中的250个新语句的内存增长，暗示一个中等核心语句占用约12K的内存，而ORM中的小语句占用约20 K，包括结构获取的结构，其中ORM的这些结构将更大。

.. _engine_compiled_cache:

禁用或使用其他字典缓存一些（或全部）语句
~~~~~~~~~~~~~~~~~~~~~~~~~

内部使用的缓存称为“LRUCache”，但这主要只是一个字典。任何字典都可以作为任何一系列语句的缓存来使用，方法是使用:paramref: .Connection.execution_options.compiled_cache选项作为执行选项。执行选项可以在语句、  :clas:`_engine.Engine`  或  :clas:` _engine.Connection`  上设置，以及在使用ORM的  :meth:`_orm.Session.execute`  方法时进行设置，用于SQLAlchemy-2.0样式的调用。例如，要运行一系列SQL语句并将它们缓存在特定的字典中：

my_cache = {}
with engine.connect().execution_options(compiled_cache=my_cache) as conn:
    conn.execute(table.select())

SQLAlchemy ORM使用上述技术来在工作单元“flush”过程中保留独立于 :class:`_engine.Engine` 上配置的默认缓存的每个映射器缓存，以及某些关系加载器查询。

也可以使用此参数禁用缓存，方法是发送值为`None`：

# disable caching for this connection
with engine.connect().execution_options(compiled_cache=None) as conn:
    conn.execute(table.select())

.. _engine_thirdparty_caching:

第三方方言的缓存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

缓存功能要求方言的编译器生成安全重用的SQL字符串，以给定针对该SQL字符串键入的特定缓存键。这意味着语句中的任何文字值，例如SELECT的LIMIT / OFFSET值，都不能在方言的编译方案中硬编码，因为编译后的字符串将无法重新使用。SQLAlchemy支持使用:meth:`_sql.BindParameter.render_literal_execute` 方法呈现绑定参数的渲染限制/偏移子句，该方法可以通过自定义编译器应用于现有的`Select._limit_clause`和`Select._offset_clause`属性，如本节稍后的演示。

由于有许多第三方方言，并且其中许多可能会从SQL语句中生成文字值，而没有使用新的“字面执行”功能的好处，SQLAlchemy从版本1.4.5开始添加了称为  :attr:`_engine.Dialect.supports_statement_cache`   的方言属性。此属性在运行时直接在特定方言类上检查其是否存在，即使在超类上已经存在，因此即使在继承现有可缓存的SQLAlchemy方言（例如` ` sqlalchemy.dialects.postgresql.PGDialect``）的第三方方言也必须显式包括此属性才能启用缓存。一旦方言已被修改为需要并经过可重用编译的SQL语句进行测试，就应该启用该属性。对于不支持此属性的所有第三方方言，日志记录将指示“方言不支持缓存”。

测试完成方言的缓存后，特别是SQL编译器已更新为不直接呈现任何文本LIMIT / OFFSET，编译器作者可以如下应用该属性：

from sqlalchemy.engine.default import DefaultDialect

class MyDialect(DefaultDialect):
    supports_statement_cache = True

flag还需要应用于方言的所有子类，例如：

class MyDBAPIForMyDialect(MyDialect):
    supports_statement_cache = True

.. versionadded:: 1.4.5

    添加了  :attr:`.Dialect.supports_statement_cache`  属性。

通常的方言修改情况如下。

示例：使用后编译参数呈现LIMIT / OFFSET
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

例如，假设方言重写了  :meth:`SQLCompiler.limit_clause`  方法，该方法为SQL语句生成“LIMIT / OFFSET”子句，如下所示：

# pre 1.4 style code
def limit_clause(self, select, **kw):
    text = ""
    if select._limit is not None:
        text += " \n LIMIT %d" % (select._limit,)
    if select._offset is not None:
        text += " \n OFFSET %d" % (select._offset,)
    return text

上面的例程将：attr:`.Select._limit` 和  :attr:`.Select._offset`   整数值呈现为嵌入在SQL语句中的文字整数。这是对于不支持SELECT语句的LIMIT / OFFSET子句中使用绑定参数的数据库的常见要求。但是，在初始编译阶段呈现整数值本身直接与缓存不兼容：  :class:` .Select` .Select`语句无法正确生成可重复使用的值。

上述代码的纠正是将文字整数移动到SQLAlchemy的  :ref:`post-compile <change_4808>`  工具中，该工具将文字整数在初始编译阶段之外呈现，而是在发送到DBAPI的语句执行之前就执行了。在编译阶段中，可以使用  :meth:` _sql.BindParameter.render_literal_execute`  方法，结合  :attr:`.Select._limit_clause`   和  :attr:` .Select._offset_clause`   属性来表示LIMIT / OFFSET，这些属性表示将LIMIT / OFFSET显示为完整的SQL表达式，如下所示：

# 1.4 cache-compatible code
def limit_clause(self, select, **kw):
    text = ""

    limit_clause = select._limit_clause
    offset_clause = select._offset_clause

    if select._simple_int_clause(limit_clause):
        text += " \n LIMIT %s" % (
            self.process(limit_clause.render_literal_execute(), **kw)
        )
    elif limit_clause is not None:
        # assuming the DB doesn't support SQL expressions for LIMIT.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for LIMIT"
        )
    if select._simple_int_clause(offset_clause):
        text += " \n OFFSET %s" % (
            self.process(offset_clause.render_literal_execute(), **kw)
        )
    elif offset_clause is not None:
        # assuming the DB doesn't support SQL expressions for OFFSET.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for OFFSET"
        )

    return text

上述方法将生成一个编译过的SELECT语句，其样貌如下：

.. sourcecode:: sql

    SELECT x FROM y
    LIMIT __[POSTCOMPILE_param_1]
    OFFSET __[POSTCOMPILE_param_2]

上面，“__[POSTCOMPILE_param_1]”和“__[POSTCOMPILE_param_2]”指示符将在语句执行时间之前用其相应的整数值填充从缓存中检索到的SQL字符串。

完成上述更改后，  :attr:`.Dialect.supports_statement_cache`  标志应设置为“ True” 。强烈建议第三方方言使用  :ref:` examples_performance`中的 `dialect第三方测试套件<https://github.com/sqlalchemy/sqlalchemy/blob/main/README.dialects.rst>`_，以确保正确呈现和缓存诸如SELECT with LIMIT / OFFSET之类的操作。

.. 参见:: ：

    faq_new_caching-在 :ref:`faq_toplevel` 部分

.. _engine_lambda_caching:

使用Lambda在语句生成中添加显着的速度增益
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



.. deepalchemy:: 这种技术在非常性能密集的场景下通常不是必需的，并且适用于经验丰富的Python程序员。虽然相当简单直接，不过它涉及到不适合初学者Python程序员的元编程概念。Lambda方法可以稍后应用于现有代码，而无需进行任何努力。

通常，Python函数，通常表示为lambda，可以用于生成基于lambda函数本身以及lambda内的闭包变量的SQL表达式，这可以基于应用程序运行中的Python代码位置和闭包变量来缓存不仅是SQL字符串编译形式，正如SQLAlchemy在不使用lambda系统时的正常行为，还缓存SQL表达式结构本身的维护。

注：重要的是注意，即使不使用lambda系统，SQLAlchemy中已有SQL缓存。 lambda系统只通过将构建SQL结构本身的缓存和使用更简单的缓存密钥一个附加的操作量来减少每个SQL语句作为缓存渐近性调用的工作量。

简要指南对于Lambda
^^^^^^^^^^^^^^^^^^^^^^^^^^^

关键是，重点在于lambda SQL系统确保生成lambda的缓存键与它将生成的SQL字符串永远不会不匹配。  :class:`_sql.LambdaElement` 和相关对象将在每次运行时运行并分析给定的lambda，以计算如何在每次运行时缓存它，试图检测任何潜在问题。基本准则包括：

* **支持任何类型的声明符** - 虽然有望  :meth:`_sql.select`  构造函数是  :func:` _sql.lambda_stmt`  和  :meth:`_sql.update`  也可用：：

    def upd(id_, newname):
        stmt = lambda_stmt(lambda: users.update())
        stmt += lambda s: s.values(name=newname)
        stmt += lambda s: s.where(users.c.id == id_)
        return stmt


    with engine.begin() as conn:
        conn.execute(upd(7, "foo"))

* **ORM用例直接支持** -   :func:`_sql.lambda_stmt`  直接使用：：

    def select_user(session, name):
        stmt = lambda_stmt(lambda: select(User))
        stmt += lambda s: s.where(User.name == name)

        row = session.execute(stmt).first()
        return row

* **绑定参数会自动适应** - 与SQLAlchemy以前的“烤好的查询”系统不同，lambda SQL系统自动适应由Python操作符产生的文字值，这些文字值会自动成为SQL绑定参数。这意味着即使一个给定的lambda只运行一次，生成的闭包变量中的纯量值在每次运行时也是从lambda中提取出来的：

  .. sourcecode:: pycon+sql

        >>> def my_stmt(x, y):
        ...     stmt = lambda_stmt(lambda: select(column("q")))
        ...     stmt += lambda s: s.where(column("q") > x)
        ...     stmt += lambda s: s.where(column("q") == y)
        ...     return stmt
        >>> engine = create_engine("sqlite://", echo=True)
        >>> with engine.connect() as conn:
        ...     print(conn.scalar(my_stmt(5, 10)))
        ...     print(conn.scalar(my_stmt(12, 8)))
        SELECT q 
        FROM anon_1 
        WHERE q > ? AND q = ?
        [generated in 0.00099s] (5, 10){stop}
        10
        SELECT q 
        FROM anon_1 
        WHERE q > ? AND q = ?
        [generated in 0.00029s] (12, 8){stop}
        12

  上面的例子表明  :class:`_sql.StatementLambdaElement` ` x``和``y``的值，因此它们会在lambda调用之前的缓存构造中替换为相应的值。

* **lambda理想上应该在所有情况下生成一致的SQL结构** - 避免在lambda中使用有条件或自定义可调用程序，因为这可能会使其根据输入生成不同的SQL; 如果一个函数可能有条件地使用两个不同的SQL片段，那么使用两个不同的lambda：

        # **不要** 这样做：
        

        def my_stmt(parameter, thing=False):
            stmt = lambda_stmt(lambda: select(table))
            stmt += (
                lambda s: s.where(table.c.x > parameter)
                if thing
                else s.where(table.c.y == parameter)
            )
            return stmt


        # **要** 这样做：
        

        def my_stmt(parameter, thing=False):
            stmt = lambda_stmt(lambda: select(table))
            if thing:
                stmt += lambda s: s.where(table.c.x > parameter)
            else:
                stmt += lambda s: s.where(table.c.y == parameter)
            return stmt

  如果Lambda不能生成一致的SQL结构，可能会发生各种故障，其中一些可能无法轻松检测。

* **不要在Lambda内部引用Lambda用于生成绑定值的函数** - 绑定值跟踪方法要求在lambda的闭包中存在要在SQL语句中用于绑定的实际值。如果值从其他函数生成，则不可能这样做，而且  :class:`_sql.LambdaElement` 通常抛出错误如果尝试这样做

    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     stmt = lambda_stmt(lambda: select(func.max(get_x(), get_y())))
    ...     return stmt
    >>> engine = create_engine("sqlite://", echo=True)
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    Traceback (most recent call last):
      # ...
    sqlalchemy.exc.InvalidRequestError: Can't invoke Python callable get_x()
    inside of lambda expression argument at
    <code object <lambda> at 0x7fed15f35450, file
    "<stdin>", line 6>;
    lambda SQL constructs should not invoke functions from closure variables
    to produce literal values since the lambda SQL system normally extracts
    bound values without actually invoking the lambda or any functions within it.

  上述错误表示  :class:`_sql.LambdaElement` ` Foo``对象将始终保持相同的行为并且默认情况下不会将其用作缓存键。如果使用``Foo``对象作为缓存键，如果有许多不同的``Foo``对象，则会填充缓存具有重复信息，并且还将为所有这些对象保持长时间参考。解决上述情况的最佳方法是不要在lambda中引用``foo``，而是在lambda之外引用它：

    >>> def my_stmt(foo):
    ...     x_param, y_param = foo.x, foo.y
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt

  在某些情况下，如果保证lambda基于输入永远不会更改其SQL结构，则可以通过传递``track_closure_variables=False``来禁用任何跟踪除用于绑定参数的变量外其他闭包变量的默认情况：

    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(func.max(foo.x, foo.y)), track_closure_variables=False
    ...     )
    ...     return stmt

  还有一个选项可以将对象添加到元素中，以显式形式构成缓存键，使用``track_on`` 参数; 使用此参数允许指定的值用作缓存键，并且还将防止考虑其他闭包变量。这对于来自某些上下文对象的一部分SQL构造，这些对象可能具有许多不同的值的情况很有用。在下面的示例中，SELECT语句的第一个段将禁用变量``foo``的跟踪，而第二个段将显式跟踪“self”作为缓存键的一部分::

    >>> def my_stmt(self, foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(*self.column_expressions), track_closure_variables=False
    ...     )
    ...     stmt = stmt.add_criteria(lambda: self.where_criteria, track_on=[self])
    ...     return stmt

  使用``track_on``意味着在lambda的内部缓存中长期存储给定对象，并且在缓存不清除这些对象（默认情况下使用1000个条目的LRU方案）的时间内具有较强的引用。

  ..

缓存密钥生成
^^^^^^^^^^^^^^^^^

为了了解有关lambda SQL构造的某些选项和行为，有助于了解缓存系统是如何工作的。SQLAlchemy的缓存系统通常通过从给定SQL表达式构造表示构造缓存密钥来生成缓存密钥，该表示构成为所有可能发生变化的对象表示构成：

    >>> from sqlalchemy import select, column
    >>> stmt = select(column("q"))
    >>> cache_key = stmt._generate_cache_key()
    >>> print(cache_key)  # somewhat paraphrased
    CacheKey(key=(
      '0',
      <class 'sqlalchemy.sql.selectable.Select'>,
      '_raw_columns',
      (
        (
          '1',
          <class 'sqlalchemy.sql.elements.ColumnClause'>,
          'name',
          'q',
          'type',
          (
            <class 'sqlalchemy.sql.sqltypes.NullType'>,
          ),
        ),
      ),
      # a few more elements are here, and many more for a more
      # complicated SELECT statement
    ),)


以上密钥存储在缓存中，该缓存实际上是一个字典，而值是某种结构，其中包含SQL语句的字符串形式，即短语“SELECT q”。我们可以观察到，即使对于非常短的查询，缓存密钥也非常详细，因为它必须表示关于正在呈现和可能执行的一切的所有州。

相比之下，lambda构造系统创建了一种不同类型的缓存密钥：

    >>> from sqlalchemy import lambda_stmt
    >>> stmt = lambda_stmt(lambda: select(column("q")))
    >>> cache_key = stmt._generate_cache_key()
    >>> print(cache_key)
    CacheKey(key=(
      <code object <lambda> at 0x7fed1617c710, file "<stdin>", line 1>,
      <class 'sqlalchemy.sql.lambdas.StatementLambdaElement'>,
    ),)

上面，我们看到一个比非lambda语句短得多的缓存密钥，另外，甚至不需要生成“select(column("q"))”构造， Python lambda本身带有一个称为“__code__”的属性，该属性引用在应用程序的运行时是不可变且永久的Python代码对象。

当lambda还包括闭包变量时，通常情况下，如果这些变量引用SQL构造，例如列对象，则它们成为缓存密钥的一部分，或者，如果它们引用将绑定的文字值，则它们放置在缓存密钥的另一个元素中。 ：

    >>> def my_stmt(parameter):
    ...     col = column("q")
    ...     stmt = lambda_stmt(lambda: select(col))
    ...     stmt += lambda s: s.where(col == parameter)
    ...     return stmt

 :class:`_sql.StatementLambdaElement` 具有两个lambda，两个lambda都引用“col”闭包变量，因此缓存密钥将表示这两个段以及“column()”对象：

    >>> stmt = my_stmt(5)
    >>> key = stmt._generate_cache_key()
    >>> print(key)
    CacheKey(key=(
      <code object <lambda> at 0x7f07323c50e0, file "<stdin>", line 3>,
      (
        '0',
        <class 'sqlalchemy.sql.elements.ColumnClause'>,
        'name',
        'q',
        'type',
        (
          <class 'sqlalchemy.sql.sqltypes.NullType'>,
        ),
      ),
      <code object <lambda> at 0x7f07323c5190, file "<stdin>", line 4>,
      <class 'sqlalchemy.sql.lambdas.LinkedLambdaElement'>,
      (
        '0',
        <class 'sqlalchemy.sql.elements.ColumnClause'>,
        'name',
        'q',
        'type',
        (
          <class 'sqlalchemy.sql.sqltypes.NullType'>,
        ),
      ),
      (
        '0',
        <class 'sqlalchemy.sql.elements.ColumnClause'>,
        'name',
        'q',
        'type',
        (
          <class 'sqlalchemy.sql.sqltypes.NullType'>,
        ),
      ),
    ),)


缓存密钥的第二个部分检索将在语句调用时使用的绑定参数：

    >>> key.bindparams
    [BindParameter('%(139668884281280 parameter)s', 5, type_=Integer())]


如果想要了解具有性能比较的“lambda”缓存的一系列示例，请参见  :ref:`examples_performance` 中的“short_selects”测试套件。


.. _engine_insertmanyvalues:

INSERT语句的“插入多个值”行为
------------------------------------------------- ------------------

.. versionadded:: 2.0 有关更改的背景信息，请参见  :ref:`change_6047` ，其中包括示例性能测试

.. tip ::  :term:`insertmanyvalues`  。

随着SQLite和MariaDB增加对RETURNING的支持，SQLAlchemy不再需要依赖于大多数后端提供的仅限单行的`cursor.lastrowid <https://peps.python.org/pep-0249/#lastrowid>`_属性； RETURNING现在可以用于除MySQL以外的所有  :ref:`SQLAlchemy-included <included_dialects>` ` executemany()``并改为重组单个INSERT语句来解决，每个语句都可容纳大量的行在一次语句中，该语句使用``cursor.execute()``调用。这种方法源自于“psycopg2快速执行帮手”<https://www.psycopg.org/docs/extras.html#fast-execution-helpers>功能，SQLAlchemy在最近的发布系列中逐步增加了对此功能的支持。

当前支持
~~~~~~~~~~

该功能适用于SQLAlchemy中包含支持RETURNING的后端，但Oracle除外，因为cx_Oracle和OracleDB驱动程序都提供自己的等效特性。使用  :paramref:`_engine.Connection.execute.parameters`  参数将字典列表传递给  :meth:` _engine.Connection.execute`  或  :meth:`_orm.Session.execute`  方法（以及  :ref:` asyncio <asyncio_toplevel>`  ）并使用  :term:`executemany`  执行时，通常会发生特征发生。当使用  :meth:` _orm.Session.add`  和  :meth:`_orm.Session.add_all`  添加行时，它也将在ORM  :term:` unit of work`  过程中发生。

对于SQLAlchemy的包含方言，当前支持或等效支持如下：

* SQLite-受SQLite 3.35及以上版本支持
* PostgreSQL-所有受支持的Postgresql版本（9及以上）
* SQL Server-受支持的所有SQL Server版本[#]_
* MariaDB-受MariaDB版本10.5及以上支持
* MySQL-不支持，没有RETURNING特性
* Oracle-使用本机cx_Oracle / OracleDB的executemany支持RETURNINGAPI，在所有支持的Oracle版本9及以上使用多行OUT参数。这不是"executemanyvalues"的相同实现，但具有相同的使用模式和等效的性能。

.. versionchanged :: 2.0.10

   .. [#] "insertmanyvalues"支持Microsoft SQL Server的临时禁用在版本2.0.9中被恢复。

禁用功能
~~~~~~~~~~

要为某个后端禁用“insertmanyvalues”功能，请在  :meth:`_sa.create_engine`  中将  :paramref:` _sa.create_engine.use_insertmanyvalues`  参数设置为``False``：

    engine = create_engine(
        "mariadb+mariadbconnector://scott:tiger@host/db", use_insertmanyvalues=False
    )

也可以通过传递  :paramref:`_schema.Table.implicit_returning`  参数为` `False``来禁用这一特性，从而不会在一个特定的 :class:`._schema.Table` 对象中使用隐式的返回特性。：

      t = Table(
          "t",
          metadata,
          Column("id", Integer, primary_key=True),
          Column("x", Integer),
          implicit_returning=False,
      )

禁用返回特性的原因是解决特定于后端的限制。

批操作模式
~~~~~~~~~~~

该功能有两种操作模式，可以在每个方言和 :class:`_schema.Table` 基础上选中。 一个是“批处理模式”，它通过重新编写INSERT语句的形式来减少数据库往返次数：

.. sourcecode:: sql

    INSERT INTO a (data, x, y) VALUES (%(data)s, %(x)s, %(y)s) RETURNING a.id

将其转换为类似于以下形式的“分批”形式：

.. sourcecode:: sql

    INSERT INTO a (data, x, y) VALUES
        (%(data_0)s, %(x_0)s, %(y_0)s),
        (%(data_1)s, %(x_1)s, %(y_1)s),
        (%(data_2)s, %(x_2)s, %(y_2)s),
        ...
        (%(data_78)s, %(x_78)s, %(y_78)s)
    RETURNING a.id

在上面，语句是针对输入数据的子集（即“批”）进行组织的，其大小由数据库后端以及每批参数对应于语句大小/参数数的已知限制来确定。该特征然后对每个输入数据的批执行一次INSERT语句，直到消耗所有记录，并将每个批的RETURNING结果连接到一个单个大行集合中，该集合可以从单个  :class:`_result.Result`  对象中获得。

这种“分批”形式允许使用较少的数据库往返次数插入许多行，并已经被证明对大多数后端都可以实现显着的性能提升。

.. _engine_insertmanyvalues_returning_order:

将RETURNING行与参数集相关联
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded :: 2.0.10

在前面一节中说明的“分批”模式查询不保证返回的记录的顺序与输入数据的顺序相对应。当被SQLAlchemy ORM  :term:`unit of work`  处理或对于将返回的服务器生成的值与输入数据相关联的应用程序时，  :meth:` _dml.Insert.returning`  和  :meth:`_dml.UpdateBase.return_defaults`  方法包括一个选项:paramref :` _dml.Insert.returning.sort_by_parameter_order`，表示 "insertmanyvalues" 模式应保证此对应关系。这与记录实际被数据库后端插入的顺序无关，其在任何情况下都不被假定；只是应该假定在接收到返回的记录时，应按照传递原始输入数据的顺序组织这些记录。

当出现  :paramref:`_dml.Insert.returning.sort_by_parameter_order`  参数时，对于使用诸如“IDENTITY”、PostgreSQL“SERIAL”、MariaDB“AUTO_INCREMENT”或SQLite的“ROWID”方案等服务器生成的整数主键值的表，可能会代替“分批”模式，使用更复杂的INSERT..RETURNING形式，结合基于返回值的行的后期排序，如果不存在这样的形式，则“insertmanyvalues”特性可能会以一种优雅的方式降级为“非分批”模式，该方式为每个参数集运行单个INSERT语句。

例如，在SQL Server上，当自动增加的“IDENTITY”列用作主键时，将使用以下SQL形式：

.. sourcecode:: sql

    INSERT INTO a (data, x, y)
    OUTPUT inserted.id, inserted.id AS id__1
    SELECT p0, p1, p2 FROM (VALUES
        (?, ?, ?, 0), (?, ?, ?, 1), (?, ?, ?, 2),
        ...
        (?, ?, ?, 77)
    ) AS imp_sen(p0, p1, p2, sen_counter) ORDER BY sen_counter

当PostgreSQL的主键列使用SERIAL或IDENTITY时，将使用类似的形式。上述形式**不能**保证插入行的顺序。但是，它保证IDENTITY或SERIAL值随每个参数集按顺序创建。然后，“insertmanyvalues”功能根据递增整数标识符对返回的行进行排序。

对于SQLite数据库，没有适当的INSERT表单可以将新的ROWID值与传递的参数集的顺序相关联。因此，在使用后端生成的主键值时，SQLite后端将降级为“非分批”模式，当请求有序RETURNING时将运行单个INSERT语句。

对于MariaDB，insertmanyvalues使用的默认INSERT表单是足够的，因为这个数据库后端在使用InnoDB时将自动将：AUTO_INCREMENT的顺序与输入数据的顺序对齐[#]_。

对于具有其他类型的服务器生成的主键值的 :class:`_schema.Table` 配置，以及不能提供确定性地对多个参数集对齐的服务器已生成值的后端，当要求保证RETURNING排序时，"insertmanyvalues"模式将使用**非分批**模式。

.. seealso::

    .. [#]

    * Microsoft SQL Server论证

      “使用SELECT和ORDER BY填充行的INSERT查询保证如何计算标识值，但不保证插入行的顺序。”
      https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions

    * PostgreSQL batched INSERT讨论

      2018年原始描述https://www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us

      2023年后续 - https://www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com

.. _engine_insertmanyvalues_non_batch:

非批次模式操作
~~~~~~~~~~~~~~~~~~~~~~~~~~

对于不具有客户端主键值的  :class:`_schema.Table`  的需求的 :class:` _dml.Insert`语句进行有保证的RETURNING排序时，当不支持此类值时，"insertmanyvalues"特征可能会减少分批模式，而使用**非批次**模式。

在这种模式下，保留INSERT语句的原始SQL形式，“insertmanyvalues”特性将为每个参数集单独运行该语句，将返回的行组织为完整的结果集。与先前的SQLAlchemy版本不同，它确实以最小化Python开销为基础进行循环。在某些情况下，例如在SQLite上，“非分批”模式的性能与“分批”模式完全相同。

语句执行模型
~~~~~~~~~~~~~~~~~~~~~~~~~

对于“分批”和“非分批”两种模式，该特征将必然使用DBAPI``cursor.execute（）``方法在Core级别的  :meth:`_engine.Connection.execute`  方法的范围内调用**多个INSERT语句**，每个语句最多包含一定数量的参数集。该限制可以根据下文的  :ref:` engine_insertmanyvalues_page_size` `cursor.execute()``调用被单独记录并传递给事件侦听器，例如  :meth:`.ConnectionEvents.before_cursor_execute`  （请参见  :ref:` engine_insertmanyvalues_events` ）。

...


.. _engine_insertmanyvalues_page_size:

控制批量大小
~~~~~~~~~~~~~~~~~~~~~~~~~~

“insertmanyvalues”的一个重要特征是，INSERT语句的大小限制为固定的最大“值”子句数，以及每次表示的绑定参数的固定总数，在一次执行中不能表示大于此数量的参数字典。当给定的参数字典的数量超过固定限制，或者当要呈现在单个INSERT语句中的绑定参数总数超过固定限制时（两个固定限制是单独的），将在单个  :meth:`_engine.Connection.execute`  调用的范围内调用多个INSERT语句，每个INSERT语句都适用于一个参数字典的一部分，称为“批”。每个“批”的参数字典数然后被称为“批量”。例如，批量大小为500意味着每个INSERT语句最多插入500行。

重要的是能够调整批量大小，因为较大的批量大小可能对输入本身相对较小的插入效果更高，并且较小的批量大小可能对使用非常大的值集的插入更合适，其中使用最小化后端行为和内存约束的固定大小显然是必要的。因此，批量大小可以在每个 :class:`.Engine` 以及每个语句的基础上进行配置。限制参数另一方面是基于正在使用的数据库的已知特征的固定参数。

对于大多数后端，默认情况下，批量大小为1000，具有额外的每方言“最大参数数”限制因素，该限制因素可以根据需要在每个语句的基础上进一步降低。最大参数数因方言和服务器版本而异；最大大小为32700（选择为健康的距离PostgreSQL 32767和SQLite现代限制32766，同时留有余地以便于语句中的附加参数以及DBAPI古怪行），较旧的SQLite版本（3.32.0以前）将此值设置为999。MariaDB没有确定的限制，但32700仍然是SQL消息大小的限制因素。

批量大小的值可以通过   :class:`._engine.Engine`  参数设置在各后端上。例如，要使INSERT语句包含100个参数集：

    e = create_engine("sqlite://", insertmanyvalues_page_size=100)

批量大小也可以使用  :paramref:`_engine.Connection.execution_options.insertmanyvalues_page_size`  执行选项每个语句地受影响，如下所示::

    with e.begin() as conn:
        result = conn.execute(
            table.insert().returning(table.c.id),
            parameterlist,
            execution_options={"insertmanyvalues_page_size": 100},
        )

或者在语句本身上进行配置::

    stmt = (
        table.insert()
        .returning(table.c.id)
        .execution_options(insertmanyvalues_page_size=100)
    )
    with e.begin() as conn:
        result = conn.execute(stmt, parameterlist)

.. _engine_insertmanyvalues_events:

日志和事件
~~~~~~~~~~~~~~~~~~

“insertmanyvalues”功能完全与SQLAlchemy的  :ref:`statement logging <dbengine_logging>` .ConnectionEvents.before_cursor_execute` ）集成。当参数列表被分成单独的批次时，“每个INSERT语句都会单独记录并传递给事件处理程序”**。这是与仅限于psycopg2的功能在SQLAlchemy 1.x系列中的工作方式的主要更改，其中多个INSERT语句的生成被隐藏在记录和事件之外。日志显示将截断参数的长列表以便于阅读，并且还将指示每个语句的特定批次。下面的示例说明了这种记录的摘录：

.. sourcecode:: text

  INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
  [generated in 0.00177s (insertmanyvalues) 1/10 (unordered)] ('d0', 0, 0, 'd1',  ...
  INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
  [insertmanyvalues 2/10 (unordered)] ('d100', 100, 1000, 'd101', ...

  ...

  INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
  [insertmanyvalues 10/10 (unordered)] ('d900', 900, 9000, 'd901', ...

当进行 :ref:`非批处理模式 <engine_insertmanyvalues_non_batch>` 时，日志将指示此操作与insertmanyvalues消息一起发生：

.. sourcecode:: text

  ...

  INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
  [insertmanyvalues 67/78 (ordered; batch not supported)] ('d66', 66, 66)
  INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
  [insertmanyvalues 68/78 (ordered; batch not supported)] ('d67', 67, 67)
  INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
  [insertmanyvalues 69/78 (ordered; batch not supported)] ('d68', 68, 68)
  INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
  [insertmanyvalues 70/78 (ordered; batch not supported)] ('d69', 69, 69)

  ...

.. seealso::

      :ref:`dbengine_logging` 

Upsert Support
~~~~~~~~~~~~~~

PostgreSQL、SQLite和MariaDB方言提供了特定于后端的“upsert”构造  :func:`_postgresql.insert` 、  :func:` _sqlite.insert` ，它们均为 :class:`_dml.Insert` 构造，具有另外的方法，如“on_conflict_do_update()”或“on_duplicate_key()”。当这些构造函数和RETURNING一起使用时，还支持“insertmanyvalues”行为，从而允许高效的upsert with RETURNING。

.. _engine_disposal:

引擎处理
---------------

  :class:`_engine.Engine` .QueuePool` 的正常默认池实现。

  :class:`_engine.Engine` ._engine.Engine` ；相反，它是一个注册表，维护连接池以及关于正在使用的数据库和DBAPI的配置信息，以及一些程度的内部缓存。每个数据库资源的缓存。

然而，有很多情况可以在全部关闭由  :class:`_engine.Engine`  方法显式处理  :class:` _engine.Engine` 。这会处理引擎的底层连接池，并用空的连接池替换它。只要此时放弃了 :class:`_engine.Engine` 并且不再使用它，则所有已检入的连接将完全关闭。

调用  :meth:`_engine.Engine.dispose`  的有效用例包括：

* 当程序想要释放连接池中仍存在的任何剩余检入连接，并且期望将来不再连接到该数据库以进行任何未来操作时。

* 当程序使用 multiprocessing 或“fork()”时，将  :class:`_engine.Engine`  ，以便引擎创建专门针对该fork的全新数据库连接。通常情况下，数据库连接不会穿越进程边界。在这种情况下，在参数  :paramref:` .Engine.dispose.close`  设置为False。请参见 :ref:`pooling_multiprocessing` 部分以获取有关此用例的更多背景知识。

* 在测试套件或多租户场景中，可能会创建许多临时的、短寿命的 :class:`_engine.Engine` 对象，并进行处置。

当引用的有  :class:`_engine.Engine`  或垃圾回收时被丢弃，因为这些连接仍然在应用程序的其他地方被强烈引用。但是，在调用  :meth:` _engine.Engine.dispose`  后，这些连接不再与该  :class:`_engine.Engine`  。


  :class:`_engine.Engine`  时不随之被释放，因为这些句柄仍然在应用程序的其他地方存在。如果您还有其他共享同一池的  :class:` _engine.Engine` ，请确保我们通过实际调用  :meth:`_pool.QueuePool.dispose`  或其它方法释放废弃池中的连接。完全兼容DBAPI连接池
------------------------

SQLAlchemy包含一个内置的连接池实现，它提供了对程序中使用的所有连接的管理。
这就避免了每次需要连接时都要重新创建连接的开销。默认情况下，连接池在连接
不再需要时并不会将其释放回DBAPI。相反，它们只是被返回给池中。这通常只会对
使用新连接产生一些轻微性能影响，并且意味着当连接签入时，它被完全关闭，
而不会在内存中保留。请参见  :ref:`pool_switching` 获取有关如何禁用池的指南。

.. seealso::

      :ref:`pooling_toplevel` 

      :ref:`pooling_multiprocessing` 

.. _dbapi_connections:

使用Driver SQL和 Raw DBAPI Connections 工作
-------------------------------------------------

  :meth:`_engine.Connection.execute`  方法的介绍使用了  :func:` _expression.text` 
构造以说明如何调用文本SQL语句。 使用SQLAlchemy时，文本SQL实际上更多的是
一个例外，而不是规则，因为Core表达式语言和ORM都抽象了SQL的文本表示。
然而， :func:`_expression.text` 构造本身也提供了对文本SQL的抽象，因为它规范化了
如何传递绑定参数，并支持参数和结果集行的数据类型行为。

直接向驱动程序调用SQL字符串
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于需要直接传递给底层驱动程序（称为  :term:`DBAPI`  ）的文本SQL的用例，
可以使用  :meth:`_engine.Connection.exec_driver_sql`  方法::

    with engine.connect() as conn:
        conn.exec_driver_sql("SET param='bar'")

.. versionadded:: 1.4  Added the  :meth:`_engine.Connection.exec_driver_sql`  method.

.. _dbapi_connections_cursor:

直接与DBAPI游标交互
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在某些情况下，SQLAlchemy无法提供一种通用的方式来访问一些  :term:`DBAPI`  函数，
比如调用存储过程以及处理多个结果集。在这些情况下，直接使用原始的
DBAPI连接方法同样方便。

访问原始DBAPI连接的最常见方式是直接从已经存在的 :class:`_engine.Connection` 对象
中获取它。它使用  :attr:`_engine.Connection.connection`  属性进行访问::

    connection = engine.connect()
    dbapi_conn = connection.connection

在这里，DBAPI连接实际上是“代理”的，因为它们起始于连接池，但这是一个能够忽略的
实现细节。由于该DBAPI连接仍然包含在拥有的 :class:`_engine.Connection` 对象的范围内，
因此最好使用 :class:`_engine.Connection` 对象来进行大多数特性使用，
如事务控制以及调用  :meth:`_engine.Connection.close`  方法；如果在DBAPI连接上直接执行
这些操作，则拥有的 :class:`_engine.Connection` 将不会知道这些状态的更改。

为了克服由拥有的 :class:`_engine.Connection` 维护的DBAPI连接所施加的限制，
还有一种DBAPI连接可用，而不需要获取
 :class:`_engine.Connection` 首先使用 :meth:`_engine.Engine.raw_connection` 方法
  :class:`_engine.Engine` ::

    dbapi_conn = engine.raw_connection()

在这里，DBAPI连接再次是“代理”形式，如之前所述。这种代理的目的现在显而易见，
当我们调用``close()``这个连接的方法时，DBAPI连接通常并没有真正关闭，而是将其
：term:`released`回引擎的连接池。

    dbapi_conn.close()

虽然在未来，SQLAlchemy可能会为更多的DBAPI用例添加内置模式，
但是这些用例往往很少使用，而且还高度依赖于使用的DBAPI类型，
因此在任何情况下，直接DBAPI调用模式总是存在于需要的情况下。

.. seealso::

      :ref:`faq_dbapi_connection`  - 包括有关如何访问DBAPI连接以及使用asyncio驱动程序时的“驱动程序”连接的其他详细信息。

DBAPI连接使用的一些示例见下文。

.. _stored_procedures:

调用存储过程和用户定义函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy支持以多种方式调用存储过程和用户定义函数。
请注意，所有的DBAPI都有不同的做法，因此必须查阅底层DBAPI的文档以获得有关特定用法的详细信息。
以下示例是假设的，可能无法在您的底层DBAPI中使用。

对于具有特殊语法或参数问题的存储过程或函数，
可以使用DBAPI级别的`callproc <https://legacy.python.org/dev/peps/pep-0249/#callproc>`_
来使用您的DBAPI。此模式的示例如下::

    connection = engine.raw_connection()
    try:
        cursor_obj = connection.cursor()
        cursor_obj.callproc("my_procedure", ["x", "y", "z"])
        results = list(cursor_obj.fetchall())
        cursor_obj.close()
        connection.commit()
    finally:
        connection.close()

.. note::

  并非所有DBAPI都使用`callproc`，总体使用详情会有所不同。上面的示例仅说明了如何使用特定的DBAPI函数。

您的DBAPI可能没有``callproc``要求，或者可能要求以另一种模式（例如使用普通的SQLAlchemy连接使用）来调用存储过程或用户定义函数。在编写本文档时，例如用psycopg2 DBAPI在PostgreSQL数据库中执行存储过程应该使用普通连接使用::

    connection.execute("CALL my_procedure();")

以上示例是假设性的。底层数据库不保证在这些情况下支持“CALL”或“SELECT”，关键字可能因存储过程或用户定义函数而异。在这些情况下，您应查阅底层DBAPI和数据库文档，以确定要使用的正确语法和模式。

多个结果集
~~~~~~~~~~~~~~~~~~~~~

从原始的DBAPI游标可以使用'nextset <https://legacy.python.org/dev/peps/pep-0249/#nextset>`_方法支持多个结果集::

    connection = engine.raw_connection()
    try:
        cursor_obj = connection.cursor()
        cursor_obj.execute("select * from table1; select * from table2")
        results_one = cursor_obj.fetchall()
        cursor_obj.nextset()
        results_two = cursor_obj.fetchall()
        cursor_obj.close()
    finally:
        connection.close()

注册新方言
------------------------

 :func:`_sa.create_engine` 函数调用会使用setuptools entrypoints查找给定方言。
这些条目可以在setup.py脚本中为第三方方言建立。
例如，要创建一个新的方言"foodialect：//"，步骤如下：

1. 创建一个名为“foodialect”的包。
2. 该包应具有包含方言类的模块，通常是 :class:`sqlalchemy.engine.default.DefaultDialect` 的子类。在这个例子中，我们假设它叫做“FooDialect”，并且它的模块是通过“foodialect.dialect”访问的。
3. 可以在“setup.cfg”中这样建立条目：

   .. sourcecode:: ini

          [options.entry_points]
          sqlalchemy.dialects =
              foodialect = foodialect.dialect:FooDialect

   如果方言为特定的DBAPI提供了对已支持的SQLAlchemy支持的数据库的支持，
   则可以给出名称，包括数据库资格。例如，如果“FooDialect”实际上是MySQL方言，则可以这样建立条目：

   .. sourcecode:: ini

      [options.entry_points]
      sqlalchemy.dialects
          mysql.foodialect = foodialect.dialect:FooDialect

   上述入口点将在使用 ``create_engine("mysql+foodialect://")`` 时被访问。

在进程中注册方言
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy还允许在当前进程中注册方言，绕过需要单独安装的要求。使用 ``register()`` 函数如下::

    from sqlalchemy.dialects import registry


    registry.register("mysql.foodialect", "myapp.dialect", "MyMySQLDialect")

上述内容将响应 ``create_engine("mysql+foodialect://")`` 并从 ``myapp.dialect`` 模块加载 ``MyMySQLDialect`` 类。

连接/引擎API
-----------------------

.. autoclass:: Connection
   :members:

.. autoclass:: CreateEnginePlugin
   :members:

.. autoclass:: Engine
   :members:

.. autoclass:: ExceptionContext
   :members:

.. autoclass:: NestedTransaction
    :members:
    :inherited-members:

.. autoclass:: RootTransaction
    :members:
    :inherited-members:

.. autoclass:: Transaction
    :members:

.. autoclass:: TwoPhaseTransaction
    :members:
    :inherited-members:


结果集API
---------------

.. autoclass:: ChunkedIteratorResult
    :members:

.. autoclass:: CursorResult
    :members:
    :inherited-members:

.. autoclass:: FilterResult
    :members:

.. autoclass:: FrozenResult
    :members:

.. autoclass:: IteratorResult
    :members:

.. autoclass:: MergedResult
    :members:

.. autoclass:: Result
    :members:
    :inherited-members:

.. autoclass:: ScalarResult
    :members:
    :inherited-members:

.. autoclass:: MappingResult
    :members:
    :inherited-members:

.. autoclass:: Row
    :members:
    :private-members: _asdict, _fields, _mapping, _t, _tuple

.. autoclass:: RowMapping
    :members:

.. autoclass:: TupleResult