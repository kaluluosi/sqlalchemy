.. _connections_toplevel:

====================================
使用Engine和Connection
====================================

.. module:: sqlalchemy.engine

本节详细介绍了如何使用:class:`_engine.Engine`、:class:`_engine.Connection`和相关对象。需要注意的是，在使用SQLAlchemy ORM时，通常不会直接访问这些对象；相反，使用:class:`.Session`对象作为数据库的接口。然而，对于那些在没有ORM较高级别管理服务介入的情况下直接使用文本SQL语句和/或SQL expression构造的应用程序，:class:`_engine.Engine`和:class:`_engine.Connection`是“国王”（和女王？）---继续阅读。

基本用法
----------

回想一下从 :doc:`/core/engines` 中了解到，通过 :func:`_sa.create_engine` 调用来创建
:class:`_engine.Engine` 如下：

    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")

通常使用 :func:`_sa.create_engine` 就是为了处理特定的数据库URL，在整个应用程序进程生命周期中保留全局。单个 :class:`_engine.Engine` 代表了处理多个单独的 :term:`DBAPI` 连接，旨在以并发方式调用。:class:`_engine.Engine` 不是 DBAPI `connect()` 的同义词，它仅代表一个连接资源 - 在应用程序的模块水平上创建一次 :class:`_engine.Engine` 需要最少次每个对象和每个函数调用都要创建。

.. sidebar:: 提示

    当使用一个有多个Python进程操作 :class:`_engine.Engine` 也就是使用 ``os.fork`` 或 Python ``multiprocessing`` 时，重要的是让在每个进程中初始化engine 。详见 :ref:`pooling_multiprocessing`。

:class:`_engine.Engine` 最基本的功能是提供访问
:class:`_engine.Connection`，它可以调用SQL语句。要将文本语句发送到数据库，如下所示：

    from sqlalchemy import text

    with engine.connect() as connection:
        result = connection.execute(text("select username from users"))
        for row in result:
            print("username:", row.username)

上面，:meth:`_engine.Engine.connect` 方法返回一个 :class:`_engine.Connection`
对象，通过在Python上下文管理器中使用它（例如使用 ``with`` 语句），自动调用 :meth:`_engine.Connection.close` 方法来结束块。 :class:`_engine.Connection` 是 *代理* 对象，代表一个实际的DBAPI连接。DBAPI连接在创建 :class:`_engine.Connection` 时从连接池中检索。

返回的对象称为 :class:`_engine.CursorResult`，引用DBAPI游标，并提供了获取与DBAPI游标类似的行的方法。
DBAPI光标会被 :class:`_engine.CursorResult` 关闭，当它的所有结果行（如果有的话）结束时。一个没有返回行的 :class:`_engine.CursorResult`，例如UPDATE语句（没有返回行），
会在创建后立即释放光标资源。

在 ``with：`` 块结束时关闭 :class:`_engine.Connection`，DBAPI连接将被释放到连接池中。从数据库的观点来看，连接池实际上不会“关闭”连接，只要连接池有空间来存储下一个使用连接。当连接返回连接池以便重新使用时，连接池机制会在DBAPI连接上发出 ``rollback()`` 调用，以删除任何事务状态或锁定（这称为 :ref:`pool_reset_on_return`） ，并且连接已准备好用于下一次使用。

我们上面的示例说明了执行文本SQL字符串，要使用 :func:`_expression.text` 构造表明要使用文本SQL。
当然 :meth:`_engine.Connection.execute` 方法可以承载更多内容。请参见 :ref:`tutorial_working_with_data`
在 :ref:`unified_tutorial` 中的教程。

使用事务
--------

.. note::

  本节描述了在直接使用 :class:`_engine.Engine` 和 :class:`_engine.Connection` 对象时如何使用事务。在使用SQLAlchemy ORM时，事务控制的公共API是通过
  使用 :class:`.Session` 对象，该对象在内部使用 :class:`.Transaction` 对象。
  有关详细信息，请参见 :ref:`unitofwork_transaction`。

走你所能到的地方
~~~~~~~~~~~~~~~~

:class:`~sqlalchemy.engine.Connection` 对象始终在事务块的上下文中发出SQL语句。第一次使用 :meth:`_engine.Connection.execute` 方法执行SQL语句时，这个事务将自动开始，使用的是叫做 **autobegin** 的行为。为在 :class:`_engine.Connection` 对象的范围内保持事务的状态，直到调用 :meth:`_engine.Connection.commit` 或 :meth:`_engine.Connection.rollback` 方法。事务结束后， :class:`_engine.Connection` 为等待调用 :meth:`_engine.Connection.execute` 方法而继续管理的块自动开始新的事务。

这种调用方式被称为 **commit as you go**，下面是一个示例：

    with engine.connect() as connection:
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

        connection.commit()  # commit the transaction

.. topic:: Python DBAPI 中的 autobegin

    与 :term:`DBAPI` 的设计意图“按需提交”相呼应，该后者是SQLAlchemy与之交互的基础数据库接口。
    在DBAPI中， ``connection`` 对象不假设对数据库进行的更改将自动提交，而需要在默认情况下调用 ``connection.commit()`` 方法，以便将更改提交到数据库中。应注意，DBAPI本身没有 begin() 方法。
    所有Python的DBAPI没有支持显式开启事务的方法，而是使用 autocommit 模式（例如：当第一次用任何的Executemany(),execute()命令等开始执行语句时，会自动调用BEGIN指令）。
    SQLAlchemy的API将此行为重新陈述为基于更高级别Python对象的行为。

在 "commit as you go" 的示例中，我们可以多次调用 :meth:`_engine.Connection.commit` 方法和 :meth:`_engine.Connection.rollback` 方法，在不断进行的命令序列中使用，每次结束事务，开始新的事务。如下：

    with engine.connect() as connection:
        connection.execute("<some statement>")
        connection.commit()  # commits "some statement"

        # new transaction starts
        connection.execute("<some other statement>")
        connection.rollback()  # rolls back "some other statement"

        # new transaction starts
        connection.execute("<a third statement>")
        connection.commit()  # commits "a third statement"

.. versionadded:: 2.0  "commit as you go" 在 SQLAlchemy 2.0 中是一个新功能。在使用“未来”式引擎时，该模式也适用于SQLAlchemy 1.4版本中的“过渡”模式。

首次进入
~~~~~~~~~~

:class:`_engine.Connection` 对象提供了一个更明确的事务管理风格，称为 **begin once**。与 "commit as you go" 相反，"begin once" 允许显式地指定事务的开始点，并允许将事务本身框架为上下文管理器块，以使事务的结束点成为隐含的。要使用 "begin once"，请使用 :meth:`_engine.Connection.begin` 方法，该方法返回一个 :class:`.Transaction` 对象，该对象表示DBAPI事务。该对象还支持显式管理，通过自己的 :meth:`_engine.Transaction.commit` 和 :meth:`_engine.Transaction.rollback` 方法，但是作为首选做法也支持上下文管理器接口，在块正常结束时对其进行提交并提供回滚，如果出现异常，则在向外传递异常之前会将其回滚。下面说明了 "begin once" 的形式：

    with engine.connect() as connection:
        with connection.begin():
            connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
            connection.execute(
                some_other_table.insert(), {"q": 8, "p": "this is some more data"}
            )

        # transaction is committed

从Engine连接并开始一次
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 :class:`_engine.Engine` 对象的级别上使用 :meth:`_engine.Engine.begin` 方法比执行两个单独的步骤 :meth:`_engine.Engine.connect` 和 :meth:`_engine.Connection.begin` 更方便； :meth:`_engine.Engine.begin` 方法返回一个特殊的上下文管理器，内部维护了 :class:`_engine.Connection` 的上下文管理器，正常情况下也就是 :class:`_engine.Transaction` 到时候会被 :meth:`_engine.Connection.begin` 方法正常返回。

    with engine.begin() as connection:
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

    # transaction is committed, and Connection is released to the connection
    # pool

.. tip::

    在 :meth:`_engine.Engine.begin` 块内，我们可以使用 :meth:`_engine.Connection.commit` 或 :meth:`_engine.Connection.rollback` 方法，它们将提前正常结束块之前的事务点。但是，如果我们这样做，则不能在 :class:`_engine.Connection` 上发出进一步的SQL操作，直到块结束：

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

混合模式
~~~~~~~~~~~~~

可以在单个 :meth:`_engine.Engine.connect` 块中自由地混用 "commit as you go" 和 "begin once" 样式，只要 :meth:`_engine.Connection.begin` 调用不与 "autobegin" 行为发生冲突。为了实现这一点， :meth:`_engine.Connection.begin` 只应在发出任何SQL语句之前或后直接调用 :meth:`_engine.Connection.commit` 或 :meth:`_engine.Connection.rollback`::

    with engine.connect() as connection:
        with connection.begin():
            # run statements in a "begin once" block
            connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})

        # transaction is committed

        # run a new statement outside of a block. The connection
        # autobegins
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

        # commit explicitly
        connection.commit()

        # can use a "begin once" block here
        with connection.begin():
            # run more statements
            connection.execute(...)

当编写使用 "begin once" 的代码时，如果已经"autobegun"了事务，库会引发 :class:`_exc.InvalidRequestError`。

.. _dbapi_autocommit:

设置交易隔离级别，包括DBAPI Autocommit
---------------------------------------------------------------

大多数DBAPI支持可配置的事务 :term:`isolation`级别的概念。这些通常是四个级别 "READ UNCOMMITTED"、"READ COMMITTED"、"REPEATABLE READ"和"SERIALIZABLE"。通常在DBAPI连接开始新的事务之前对其应用这些级别。

支持隔离级别的DBAPI也通常支持真正的“自动提交”，这意味着DBAPI连接本身将被放置在非事务性的自动提交模式中。这通常意味着DBAPI常规行为不再会自动发出"BEGIN"到数据库中，但它也可能包括其他指令。SQLAlchemy将“autocommit”概念视为另一种事务隔离级别，它是一种失去"read committed"和原子性的隔离级别。

.. tip::

  需要注意的是， "DBAPI level autocommit" 隔离级别 **与** :class:`_engine.Connection` 对象的"begin"和"commit"概念完全独立，它们仅是隔离级别的配置细节，就像任何其他隔离级别一样。

SQLAlchemy方言应支持这些隔离级别以及自动提交功能的使用。

对连接设置隔离级别或DBAPI自动提交
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于从 :meth:`.Engine.connect` 检出的单个 :class:`_engine.Connection` 对象，可以使用 :meth:`_engine.Connection.execution_options` 方法，为其持续时间设置隔离级别。该参数称为 :paramref:`_engine.Connection.execution_options.isolation_level`，值通常为以下名称的字符串的子集：

    # Connection.execution_options(isolation_level="<value>") 的可能值

    "AUTOCOMMIT"
    "READ COMMITTED"
    "READ UNCOMMITTED"
    "REPEATABLE READ"
    "SERIALIZABLE"

并不是每个DBAPI支持每个值；如果在某个后端中使用了不受支持的值，则会引发错误。

例如，要强制执行REPEATABLE READ，然后开始一次事务：

    with engine.connect().execution_options(
        isolation_level="REPEATABLE READ"
    ) as connection:
        with connection.begin():
            connection.execute("<statement>")

.. tip:: :meth:`_engine.Connection.execution_options` 方法的返回值是调用该方法的相同 :class:`_engine.Connection`对象，这意味着它在本质上在原地修改了 :class:`_engine.Connection` 对象的状态。这是SQLAlchemy 2.0版本后的新行为。这种行为不适用于 :meth:`_engine.Engine.execution_options` 方法；该方法仍然返回 :class:`.Engine` 的副本，如下所述，该副本可以使用不同的 execution options 构建多个 :class:`.Engine` 对象，但仍共享同一方言和连接池。

.. note:: :paramref:`_engine.Connection.execution_options.isolation_level` 参数必然不适用于语句级别选项，比如 :meth:`_sql.Executable.execution_options`，如果在这个级别上设置了一个选项，则它将被拒绝。这是因为只能在DBAPI连接上在每个事务基础上设置该选项。

为一个Engine设置隔离级别或DBAPI Autocommit
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:paramref:`_engine.Connection.execution_options.isolation_level`选项可以作为下面的实例所示，在engine范围内设置，通常为最好。可以通过将 :paramref:`_sa.create_engine.isolation_level` 参数传递给 :func:`.sa.create_engine` 来实现此目的：

    from sqlalchemy import create_engine

    eng = create_engine(
        "postgresql://scott:tiger@localhost/test", isolation_level="REPEATABLE READ"
    )

有了上述设置，每次创建的新DBAPI连接在它创建时都将设置为使用“REPEATABLE READ”隔离级别来执行所有后续操作。

.. _dbapi_autocommit_multiple:

在单个Engine中保持多个隔离级别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 :paramref:`_sa.create_engine.execution_options` 参数和 :meth:`_engine.Engine.execution_options` 方法可以在引擎级别上设置隔离级别，并具有更大的灵活性；
以这种方式设置的隔离级别可能更好。这可以通过设置一个值为 :paramref:`_sa.create_engine.isolation_level` 参数来实现。

    from sqlalchemy import create_engine

    eng = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test",
        execution_options={"isolation_level": "REPEATABLE READ"},
    )

具有上述设置的DBAPI连接将在开始每个新事务时设置为“REPEATABLE READ”隔离级别；但是，当连接作为连接池被重置时，连接池将被重置为最初存在的原始隔离级别。在 :func:`~sqlalchemy.create_engine` 级别的最终效果与使用 :paramref:`_sa.create_engine.isolation_level` 参数没有区别。

但是，经常需要在不同的隔离级别中运行操作的应用程序可能希望针对主 :class:`_engine.Engine` 创建多个“子引擎”，每个子引擎都配置为不同的隔离级别。其中一个案例是一个具有"事务性"和"只读"操作的操作的应用程序，可以将一个单独的 :class:`_engine.Engine` 的另一个方临渴掘井人，在这个方中使用 `AUTOCOMMIT` 或默认的隔离级别。

    from sqlalchemy import create_engine

    eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

上面， :meth:`_engine.Engine.execution_options` 方法创建原始 :class:`_engine.Engine` 的浅层副本。 ``eng`` 和 ``autocommit_engine`` 共享相同的方言和连接池。但是，在从 ``autocommit_engine`` 获取连接时，将在获取创建连接时在其上设置 "AUTOCOMMIT" 模式。

无论采用哪种隔离级别，这些状态变量在连接返回到连接池时都会不受条件地恢复。

.. seealso::

      :ref:`SQLite Transaction Isolation <sqlite_isolation_level>`

      :ref:`PostgreSQL Transaction Isolation <postgresql_isolation_level>`

      :ref:`MySQL Transaction Isolation <mysql_isolation_level>`

      :ref:`SQL Server Transaction Isolation <mssql_isolation_level>`

      :ref:`Oracle Transaction Isolation <oracle_isolation_level>`

      :ref:`session_transaction_isolation` -针对ORM

      :ref:`faq_execute_retry_autocommit` -使用DBAPI自动提交来透明地重新连接到数据库的一种方法

.. _dbapi_autocommit_understanding:

了解DBAPI级别的自动提交隔离级别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在上一部分中，我们介绍了 :paramref:`_engine.Connection.execution_options.isolation_level` 参数如何用于设置数据库隔离级别，包括 SQLAlchemy 如何将其用作另一个事务隔离级别的 autocommit 模式。 在本节中，我们将尝试澄清这种方法的含义。

如果我们想检查 :class:`_engine.Connection` 对象并使用 "autocommit" 模式，我们将按如下方式进行：

    with engine.connect() as connection:
        connection.execution_options(isolation_level="AUTOCOMMIT")
        connection.execute("<statement>")
        connection.execute("<statement>")

上述示例显示了正常使用"DBAPI autocommit"模式。没有必要使用诸如 :meth:`_engine.Connection.begin`
或 :meth:`_engine.Connection.commit` 等方法，因为所有语句都会立即提交到数据库中。当块结束时候，:class:`_engine.Connection` 对象将撤消 "autocommit" 隔离级别，DBAPI 连接因此恢复为连接池中的连接，DBAPI ``connection.rollback()`` 方法通常被调用，但由于这些语句已经被提交，因此此回滚不会改变数据库状态。

需要注意的是， "autocommit"模式即使在调用 :meth:`_engine.Connection.begin` 方法时也会继续存在；DBAPI不会向数据库发出任何 "BEGIN"，也不会在调用 :meth:`_engine.Connection.commit` 时发出 COMMIT。这种使用方式也不是错误场景，因为可以预期 "关键性" 这一概念可能会在写出了假设事务上下文的代码之后应用；毕竟，“隔离级别”是事务本身的配置细节，就像任何其他隔离级别一样。

在下面的示例中，语句仍将自动提交，无论SQLAlchemy级别的事务块如何：

    with engine.connect() as connection:
        connection = connection.execution_options(isolation_level="AUTOCOMMIT")

        # this begin() does not affect the DBAPI connection, isolation stays at AUTOCOMMIT
        with connection.begin() as trans:
            connection.execute("<statement>")
            connection.execute("<statement>")

当我们像上面的日志记录一样运行代码块时，将尝试指示根据自动提交设置虽然调用了DBAPI级别的``.commit()``，但是由于存在 autocommit 模式，它可能实际上没有效果：

.. sourcecode:: text

    INFO sqlalchemy.engine.Engine BEGIN (implicit)
    ...
    INFO sqlalchemy.engine.Engine COMMIT using DBAPI connection.commit(), DBAPI should ignore due to autocommit mode

同时，即使使用 "DBAPI autocommit"，SQLAlchemy 的事务语义，即 :meth:`_engine.Connection.begin` 的 Python行为，和 "autobegin" ：对于 :class:`_engine.Connection` 来说，**仍然保持，即使这些不影响DBAPI连接本身**。为了说明，下面的代码将引发错误，因为在执行 autobegin 后调用了 :meth:`_engine.Connection.begin`：

    with engine.connect() as connection:
        connection = connection.execution_options(isolation_level="AUTOCOMMIT")

        # "transaction" is autobegin (但由于autocommit没有影响)
        connection.execute("<statement>")

        # this will raise; "transaction" already begun
        with connection.begin() as trans:
            connection.execute("<statement>")

上面的示例还说明了相同主题，即 "autocommit" 隔离级别是底层数据库事务的配置细节，独立于 :class:`_engine.Connection` 的 "begin" 和 "commit" 概念。 "autocommit" 模式不会以任何方式与 :meth:`_engine.Connection.begin` 交互，并且 :class:`_engine.Connection` 在执行自己的关于事务的状态更改时不会在任何情况下咨询此状态（以下 engine 日志除外，它暗示这些块实际上没有提交）。进行此设计的基本原理是保持 :class:`_engine.Connection` 完全一致的使用模式，其中可以独立地更改 DBAPI-autocommit 模式，而不表示其他地方存在任何代码更改。

在隔离级别之间进行更改
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. topic:: 小结

    建议为每个隔离级别使用单独的 :class:`_engine.Connection` 对象，而不是在单个 :class:`_engine.Connection` 上切换隔离级别。这样可以使代码更易于阅读，并且少出现错误。

隔离级别设置（包括 autocommit 模式）在将连接释放回连接池时会自动重置。因此，最好避免尝试在单个 :class:`_engine.Connection` 对象上切换隔离级别，因为这会导致冗余语言。

为说明如何在一个单独的 :class:`_engine.Connection` 检查中在一次性模式下使用 "autocommit"，必须使用 :paramref:`_engine.Connection.execution_options.isolation_level` 参数和之前的隔离级别重新应用该参数。

前面的部分说明了尝试在“autocommit”模式下调用 :meth:`_engine.Connection.begin` 以开始事务的过程；我们可以重写该示例来实际执行此操作。

    # 如果我们想在单个连接中打开和关闭autocommit模式

    with engine.connect() as connection:
        connection.execution_options(isolation_level="AUTOCOMMIT")

        # run statement(s) in autocommit mode
        connection.execute("<statement>")

        # "commit" 事务 autobegun
        connection.commit()

        # switch to default isolation level
        connection.execution_options(isolation_level=connection.default_isolation_level)

        #  使用一个begin块
        with connection.begin() as trans:
            connection.execute("<statement>")

上面，在进行手动撤销隔离级别之前，我们使用 :attr:`_engine.Connection.default_isolation_level` 来恢复默认的隔离级别（认为这是我们想要的）。但是，最好的方法是使用 :class:`_engine.Connection` 的架构，它已经自动处理了在检查中恢复隔离级别的操作。编写上述示例的 **推荐** 方法是使用两个块：

    # use an autocommit block
    with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as connection:
        # run statement in autocommit mode
        connection.execute("<statement>")

    # use a regular block
    with engine.begin() as connection:
        connection.execute("<statement>")

总之：

1. "DBAPI level autocommit" 隔离级别完全独立于 :class:`_engine.Connection` 对象的 "begin" 和 "commit" 概念。
2. 对于每一个隔离级别使用单个 :class:`_engine.Connection` 保持连接检查。避免在单个连接检查中试图在“ autocommit”之间更改；让engine进行恢复默认隔离级别的工作。


.. _engine_stream_results:

使用服务器端游标（又称流式结果）
一些后端支持显式支持“服务端游标”与“客户端游标”的概念。 这里的客户端游标意味着在语句执行返回之前，数据库驱动程序完全将结果集中的所有行全部提取到内存中。 PostgreSQL和MySQL/MariaDB之类的驱动程序通常默认使用客户端游标。相比之下，服务端游标表示在客户端使用结果行时，结果行仍在数据库服务器状态中保留。Oracle的驱动程序通常使用“服务端”模型，例如，虽然SQLite dialect不使用实际的“客户端/服务器”架构，但仍使用未缓冲的结果提取方法，这将使结果行在使用之前在进程内存之外。

.. 主题::我们实际上的意思是“缓冲”与“非缓冲”结果

    服务端游标还意味着在关系数据库中具有更广泛的一组功能，例如向前和向后“滚动”游标的能力。 SQLAlchemy本身不包含任何显式支持这些行为的支持；在SQLAlchemy内部，一般术语“服务端游标”应被认为是“非缓冲结果”，而“客户端游标”则意味着“结果行在返回第一行之前缓存在内存中”。如要处理特定于某个DBAPI驱动程序的较丰富的“服务器端游标”功能集，参见 :ref:`dbapi_connections_cursor` 部分。

基于此基本架构，可以推断出，“服务器端游标”在获取非常大的结果集时更具有内存效率，同时在处理小结果集（通常少于10000行）时可能会引入更复杂的客户端/服务器通信过程并且不如客户端游标的效率高。

对于那些条件性支持缓冲或非缓冲结果的方言，通常对于使用“非缓冲”或服务器端游标模式存在注意事项。例如，使用psycopg2 dialect时，如果使用服务器端游标与任何类型的DML或DDL语句，则会引发错误。使用服务器端游标的MySQL驱动程序，DBAPI连接处于更脆弱的状态，不能从错误条件中恢复得太好，并且在完全关闭游标之前不会允许回滚继续进行。

因此，SQLAlchemy的方言将始终默认为较不易出错的游标，这意味着对于PostgreSQL和MySQL方言，它会默认为缓冲的“客户端”游标，即在调用游标的任何提取方法之前，完全将结果集中的所有结果行提取到内存中。在绝大多数情况下，这种操作模式是合适的。除了不常见的情况，即应用程序按块获取大量行，并且在提取更多行之前可以完成这些行的处理，否则非缓冲游标通常不是很有用。

对于提供客户端和服务器端游标选项的数据库驱动程序，:paramref:`_engine.Connection.execution_options.stream_results`和:paramref:`_engine.Connection.execution_options.yield_per`执行选项分别提供了每个:class:`_engine.Connection`或每个语句的“服务器端游标”访问。（当使用ORM的:class:`_orm.Session`时也存在类似的选项。）


通过yield_per使用固定缓冲流
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于使用完全未缓冲的服务器端游标的单个获取行操作通常比一次获取大批行更昂贵，:paramref:`_engine.Connection.execution_options.yield_per`执行选项可将:class:`_engine.Connection`或语句配置为利用服务器端游标，同时配置一行将检索行从服务器中一次按批次消耗。该参数可以使用:meth:`_engine.Connection.execution_options`方法或:meth:`.Executable.execution_options`方法设置为正整数值。

.. versionadded:: 1.4.40 :paramref:`_engine.Connection.execution_options.yield_per`无需其他模块即可启用SQLAlchemy 1.4.40中引入；对于先前的1.4版本，请使用:paramref:`_engine.Connection.execution_options.stream_results`直接与:meth:`_engine.Result.yield_per`组合。

使用此选项等效于手动设置为:paramref:`_engine.Connection.execution_options.stream_results`选项在下一部分:ref:`dbapi_connections_cursor` 中说明된用于特定于某个DBAPI驱动程序的丰富的“服务器端游标”功能集的方法，然后使用给定的整数值调用:meth:`_engine.Result.yield_per`方法。在这两种情况下，该组合的效果包括:

*选择适用于给定后端的服务器端游标模式（如果可用且尚不是该后端的默认行为）
*随着结果行的提取，它们将按批次缓冲，其中每个批次直到:paramref:`_engine.Connection.execution_options.yield_per`选项或:meth:`_engine.Result.yield_per`方法传递的整数参数的最后一个批次，然后小于这个大小的剩余行将被制作。
*使用:meth:`_engine.Result.partitions`方法（如果使用）使用的默认分区大小也将相应地设置为此整数大小。

以下示例说明了这三种行为：

.. code-block:: python

    with engine.connect() as conn:
        with conn.execution_options(yield_per=100).execute(
            text("select * from table")
        ) as result:
            for partition in result.partitions():
                # partition is an iterable that will be at most 100 items
                for row in partition:
                    print(f"{row}")

上面的示例说明了如何使用``yield_per = 100``以及使用:meth:`_engine.Result.partitions`方法按与从服务器检索的大小相匹配的批次对行进行处理。:meth:`_engine.Result.partitions`的使用是可选的，如果直接迭代:meth:`_engine.Result`，则每100行获取一批新的行。不应使用诸如:meth:`_engine.Result.all`之类的方法，因为这将立即获取所有剩余的行并破坏使用``yield_per``的目的。

.. tip::

    :class:`.Result`对象可以像上面那样用作上下文管理器。对于使用具有服务器端游标的迭代，这是确保:class:`.Result`对象关闭的最佳方法，即使在迭代过程中引发异常。

:paramref:`_engine.Connection.execution_options.yield_per`选项也可移植到ORM，用于使用:class:`_orm.Session`获取ORM对象，它还限制在一次生成的ORM对象数量。

动态增长缓冲区使用stream_results流式传输
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了启用不带特定分区大小的服务器端游标，可以使用:paramref:`_engine.Connection.execution_options.stream_results`选项，该选项与:paramref:`_engine.Connection.execution_options.yield_per`选项类似，可以在:class:`_engine.Connection`对象或语句对象上调用。

当使用:paramref:`_engine.Connection.execution_options.stream_results`选项提供的:class:`_engine.Result` 对象直接迭代时，内部将使用默认的缓冲方案进行获取，该方案在依次获取的每个获取上都会缓冲越来越多的行，直到预配置为小于1000行的情况。该缓冲区的最大大小可以通过:paramref:`_engine.Connection.execution_options.max_row_buffer`执行选项来影响：

.. code-block:: python

    with engine.connect() as conn:
        with conn.execution_options(stream_results=True, max_row_buffer=100).execute(
            text("select * from table")
        ) as result:
            for row in result:
                print(f"{row}")

虽然:paramref:`_engine.Connection.execution_options.stream_results`选项可以与:meth:`_engine.Result.partitions`方法结合使用，但是应将特定的分区大小传递给:meth:`_engine.Result.partitions`，使整个结果集不被获取。将使用更简单的方法在调用:meth:`_engine.Result.partitions`时使用:paramref:`_engine.Connection.execution_options.yield_per`选项。

.. seealso::

    :ref:`orm_queryguide_yield_per` - 在 :ref:`queryguide_toplevel` 中

    :meth:`_engine.Result.partitions`

    :meth:`_engine.Result.yield_per`


.. _schema_translating:

模式名称的翻译
---------------------------

为了支持将常见的表集分发到多个架构中的多租户应用程序，可以使用:paramref:`.Connection.execution_options.schema_translate_map`执行选项对一组:class:`_schema.Table`对象进行重构，使其在没有更改的情况下呈现为不同的模式名称。

给定表格::

    user_table = Table(
        "user",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
    )

此:class:`_schema.Table`的“schema”，如:paramref:`_schema.Table.schema`属性所定义，为``None``。 :paramref:`.Connection.execution_options.schema_translate_map`可以指定具有模式为``None``的所有:class:`_schema.Table`对象应该按``user_schema_one``呈现模式而不进行任何更改：

    connection = engine.connect().execution_options(
        schema_translate_map={None: "user_schema_one"}
    )

    result = connection.execute(user_table.select())

上面的代码将在数据库中调用以下SQL：

.. sourcecode:: sql

    SELECT user_schema_one.user.id, user_schema_one.user.name FROM
    user_schema_one.user

也就是说，替换模式名称。映射可以指定任意数量的目标->目标模式：

    connection = engine.connect().execution_options(
        schema_translate_map={
            None: "user_schema_one",  # no schema name -> "user_schema_one"
            "special": "special_schema",  # schema="special" becomes "special_schema"
            "public": None,  # Table objects with schema="public" will render with no schema
        }
    )

推断:paramref:`.Connection.execution_options.schema_translate_map`参数影响从SQL表达式语言生成的所有DDL和SQL构造，这一点可以根据:class:`_schema.Table`或:class:`.Sequence`对象中派生出来。它不影响直接传递字符串模式名称的方法。按照此模式，它在执行:meth:`_schema.MetaData.create_all`或:meth:`_schema.MetaData.drop_all`等方法执行的“可以创建”/“可以删除”检查中发挥作用，并且在使用基于表反射给定:class:`_schema.Table`对象时发挥作用。但是，它不影响:class:`_reflection.Inspector`对象上存在的操作，因为此时模式名称会显式传递给这些方法。

.. tip::

  要在ORM:class:`_orm.Session`中使用模式翻译特性，请在级别的 :class:`_engine.Engine` 设置此选项，然后将该引擎传递给 :class:`_orm.Session`。 :class:`_orm.Session`为每个事务使用一个新的:class:`_engine.Connection`：

      schema_engine = engine.execution_options(schema_translate_map={...})

      session = Session(schema_engine)

      ...

  .. warning::

    如果没有扩展方式使用ORM :class:`_orm.Session`，则此模式转换功能仅支持每个会话的**单个架构转换映射**。如果在每个语句的基础上给出了不同的模式转换图，它将**无法正常运行**，因为ORM :class:`_orm.Session`不会考虑当前模式转换的值用于单个对象。

    要使用单个 :class:`_orm.Session` 使用多个 ``schema_translate_map`` 配置，则可以使用:ref:`horizontal_sharding_toplevel` 扩展。请参见:ref:`examples_sharding` 上的示例。

.. _sql_caching:

SQL编译缓存
-----------------------

.. versionadded:: 1.4 SQLAlchemy现在具有透明的查询缓存系统，可以极大地降低跨Core和ORM将SQL语句构造转换为SQL字符串的Python计算开销。请参见更改:ref:`change_4639`的简介。

SQLAlchemy包括全面的SQL编译器和其ORM变体的缓存系统。该缓存系统在:class:`.Engine`内部是透明的，并提供了，针对一个Core或ORM SQL语句，以及相关的计算，这些计算为该语句对象仅执行一次，并且在此之后可用于该语句对象和所有具有相同结构的其他语句，期间该特定结构保留在引擎的“编译缓存”中。按照“语句对象具有相同结构”，这通常对应于在函数中构造的SQL语句，并在函数每次运行时生成：

    def run_my_statement(connection, parameter):
        stmt = select(table)
        stmt = stmt.where(table.c.col == parameter)
        stmt = stmt.order_by(table.c.id)
        return connection.execute(stmt)

上述语句将生成类似于``SELECT id，col FROM table WHERE col =：col ORDER BY id``的SQL，注意尽管``parameter``的值是诸如字符串或整数之类的普通Python对象，但是该字符串SQL形式不包括该值，因为它使用绑定参数。对于上述``run_my_statement()``函数的后续调用将在``connection.execute()``调用的作用域内使用缓存的编译构造来提高性能。

..注意::需要注意的是，SQL编译缓存仅缓存传递给数据库的SQL字符串，并且**不缓存**查询返回的数据。它绝不是数据缓存，并且不影响特定SQL语句返回的结果，也不意味着与获取结果行相关的任何内存使用。

虽然从早期的1.x系列开始SQLAlchemy已经具有基本的语句缓存，并且此外还具有ORM的“Baked Query”扩展，但是这两个系统都需要高度的特殊API使用才能使缓存生效。自1.4版本以来的新缓存方式是完全自动的，无需更改编程风格即可生效。

该缓存在不需要任何配置更改的情况下自动使用，无需任何特殊步骤即可生效。以下各节详细说明了缓存的配置和高级使用模式。


配置
~~~~~~~~~~~~~

缓存本身是类似于字典的对象，称为“LRUCache”，它是SQLAlchemy的字典子类，可跟踪特定键的使用情况并具有周期性的“修剪”步骤，该步骤在缓存大小达到一定阈值时删除最近未使用的项目。该缓存的大小默认为500，可以使用:paramref:`_sa.create_engine.query_cache_size`参数进行配置：

    engine = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test", query_cache_size=1200
    )

该缓存的大小可以增长为所给大小的150％因子，直到将其修剪回目标大小为止。上面的缓存大小为1200，因此可以增长为最多1800个元素，然后缩小为1200。

缓存大小的调整基于每个唯一的呈现SQL语句使用一个条目，每个引擎使用相同的SQL语句从Core和ORM生成。DDL语句通常不会被缓存。要确定缓存正在执行的操作，引擎日志将包括有关缓存行为的详细信息，下一节将描述此信息。


.. _sql_caching_logging:

使用日志记录估计缓存性能
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

上面的缓存大小为1200实际上相当大。对于小应用程序，大小在100左右的容量就足够了。要估算缓存的最佳大小，假设目标主机上有足够的内存，则缓存的大小应基于针对使用中的目标引擎的唯一SQL字符串的数量。最快捷的方式是使用SQL echo，它可以使用 :paramref:`_sa.create_engine.echo` 标志或使用Python记录;请参见 :ref:`dbengine_logging` 部分，了解有关日志配置的背景知识。

例如，我们将检查以下程序产生的日志：

.. code-block:: python

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


运行时，每个日志记录的SQL语句都将在其参数左侧包括一个带方括号的缓存统计徽章。下面总结了我们可能会看到的四种消息类型：

*``[raw sql]`` - 驱动程序或最终用户使用 :meth:`.Connection.exec_driver_sql` 直接发出了原始SQL - 不适用缓存
*``[no key]`` - 语句对象是DDL语句，不被缓存，或者语句对象包含不可缓存的元素，例如自定义用户定义构造或任意大的VALUES子句。
*``[generated in Xs]`` - 语句是**缓存未命中**，必须先编译，然后存储在缓存中。它需要X秒才能产生编译的结构。数字X将在小的小数秒内。
*``[cached since Xs ago]`` - 语句是一个**缓存命中**，不需要重新编译。从X秒前起，语句已存储在缓存中。数字X将与此应用程序运行的时间以及语句已经存在于缓存中的时间成比例，例如24小时的时间为86400。

下面对每个徽章进行更详细的描述。

我们首先看到的是用于SQLite dialect检查“a”和“b”表是否存在的语句：

.. sourcecode:: text

  INFO sqlalchemy.engine.Engine PRAGMA temp.table_info("a")
  INFO sqlalchemy.engine.Engine [raw sql] ()
  INFO sqlalchemy.engine.Engine PRAGMA main.table_info("b")
  INFO sqlalchemy.engine.Engine [raw sql] ()

对于上述两个SQLite PRAGMA语句，徽章读取``[raw sql]``，这表示驱动程序正在使用 :meth:`.Connection.exec_driver_sql` 方法直接向数据库发送Python字符串。在这种语句中不适用缓存，因为它们已经以字符串形式存在，而且关于将返回哪些结果行的信息已经不存在，因为SQLAlchemy没有预先解析SQL字符串。

接下来看到的语句是创建表的语句：

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

对于每个语句，徽章读取``[no key 0.00006s]``。这表示对于这两个特定语句，不进行缓存，因为以DDL为导向的 :class:`_schema.CreateTable` 构造不会生成缓存密钥。DDL构造通常不参与缓存，因为它们通常不可能重复第二次，并且DDL也是一个数据库配置步骤，在那里性能并不重要。

``[no key]``徽章有另一个重要含义，即它可以用于可缓存，但一些特定子构造不可缓存的SQL语句。这些示例包括不定义缓存参数的自定义用户定义SQL元素，以及某些会生成任意长且不可重复的SQL字符串的构造，例如:class:`.Values`构造以及使用:meth:`.Insert.values`方法的“多值插入”。

到目前为止，我们的缓存仍然是空的。接下来的语句将被缓存，但它们的模式看起来像：

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

以上，我们实际上看到了两个唯一的SQL字符串；``"INSERT INTO a (data) VALUES (?)"``和``"INSERT INTO b (a_id, data) VALUES (?, ?)"``。由于SQLAlchemy为所有文本值使用绑定参数，即使这些语句不同对象重复多次，由于参数是分离的，因此实际的SQL字符串保持不变。

.. 注意:: 上面的两个语句是由ORM工作单元过程生成的，并且实际上将在一个与映射器本地不同的缓存中缓存。但是机制和术语是相同的。下面的 :ref:`engine_compiled_cache` 部分将描述用户面向的代码如何使用可在每个语句基础上使用的备用缓存容器。

每个这些两个语句的第一次出现时我们看到的缓存徽章是“[generated in 0.00011s]”。这表示该语句是**不在缓存中的，编译为一个字符串需要0.00011秒，然后被缓存**。当我们看到“[generated]”徽章时，我们知道这意味着存在**缓存未命中**。一个特定的语句的第一次出现如此不可避免地是这样。但是，如果观察到许多新的“[generated]”徽章以及一个长时间运行的应用程序通常在一遍又一遍地使用相同的一系列SQL语句，则这可能意味着 :paramref:`_sa.create_engine.query_cache_size` 参数太小。当一条已被缓存的语句由于LRU缓存修剪较少使用的项而被驱逐时，它将在下一次使用时显示“[generated]”徽章。

我们示例程序随后执行一些SELECT语句，可以看到与“generated”然后“cached”相同的模式，用于选择“a”表以及之后对“b”表进行懒加载：

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
  INFO sqlalchemy.engine.Engine [cached since 0.0005922s ago] (2,)INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id

通过上面的程序，完整的运行显示了一共有四个不同的 SQL 字符串缓存。这表明缓存大小为 **四** 就足够了。这显然是一个极小的规模，使用默认大小 500 即可。

缓存使用了多少内存？
~~~~~~~~~~~~~~~~~~~~~~~~~

前面的章节详细介绍了如何检查是否需要增加 :paramref:`_sa.create_engine.query_cache_size`。那么如何知道缓存不要太大呢？我们可能想要将 :paramref:`_sa.create_engine.query_cache_size` 设置为不高于某个特定数字，因为我们的应用程序可能会利用大量不同的语句（例如，动态地从搜索 UX 构建查询），如果在过去 24 小时内运行了一万个不同的查询并且它们都被缓存，我们不希望主机因为内存不足而宕机。

测量 Python 数据结构占用的内存极其困难，不过可以通过使用进程通过 ``top`` 测量内存增长来测量连续添加 250 个新语句的系列的内存需求，这表明一个中等规模的 Core 语句占用约为 12K，而小型 ORM 语句占用约为 20K，包括为 ORM 提取结果结构，这将远大于 Core。

.. _engine_compiled_cache:

禁用或使用其他字典缓存某些（或全部）语句
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用的内部缓存被称为“LRUCache”，但这基本上只是一个字典。任何字典都可以用作任何一系列语句的缓存，只需使用 :paramref:`.Connection.execution_options.compiled_cache` 选项作为执行选项即可。执行选项可在语句、:class:`_engine.Engine` 或 :class:`_engine.Connection` 上设置，以及在使用 SQLAlchemy-2.0 样式调用 ORM :meth:`_orm.Session.execute` 方法时设置。例如，要运行一系列 SQL 语句并将它们缓存在特定的字典中：

::

    my_cache = {}
    with engine.connect().execution_options(compiled_cache=my_cache) as conn:
        conn.execute(table.select())

SQLAlchemy ORM 使用上述技术在工作单元“刷新”过程中保留每个映射器的缓存，这些缓存与配置在 :class:`_engine.Engine` 上的默认缓存不同，以及用于某些关系加载器查询。

此参数还可以通过发送 ``None`` 值来禁用缓存：

::

    # 禁用此连接的缓存
    with engine.connect().execution_options(compiled_cache=None) as conn:
        conn.execute(table.select())

.. _engine_thirdparty_caching:

第三方方言的缓存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

缓存功能要求方言的编译器产生对于特定缓存键而言是安全的，可重用的 SQL 字符串，这是基于该键来排序的。这意味着语句中的任何文字值，例如 SELECT 的 LIMIT/OFFSET 值，不能在方言的编译方案中硬编码，因为编译后的字符串将无法重用。SQLAlchemy 支持使用 :meth:`_sql.BindParameter.render_literal_execute` 方法渲染绑定参数的渲染绑定参数，该方法可以应用于现有的 ``Select._limit_clause`` 和 ``Select._offset_clause`` 属性，自定义编译器会在本节中进行说明。

由于有许多第三方方言，其中许多可能会从 SQL 语句中生成文本值而不使用新的“文字执行”功能，因此 SQLAlchemy 从版本 1.4.5 开始添加了称为 :attr:`_engine.Dialect.supports_statement_cache` 的方言属性。这个属性会在运行时直接在特定方言的类上检查其存在，即使它已经存在于超类中，因此即使第三方方言子类化一个现有的可缓存 SQLAlchemy 方言，例如 ``sqlalchemy.dialects.postgresql.PGDialect``，也必须明确包含此属性才能启用缓存。一旦方言已经经过测试可重用具有不同参数的编译的 SQL 语句，就应该 **只** 启用该属性。

对于所有不支持此属性的第三方方言，该方言的日志将指示“方言不支持缓存”。

当某个方言经过缓存测试时认为 SQL 编译器已经更新，不再在 SQL 字符串中直接呈现任何字面 LIMIT/OFFSET，并且可以像以下示例中所示将属性应用于方言：

::

    from sqlalchemy.engine.default import DefaultDialect


    class MyDialect(DefaultDialect):
        supports_statement_cache = True

需要将该标志应用于所有方言的子类：

::

    class MyDBAPIForMyDialect(MyDialect):
        supports_statement_cache = True

.. versionadded:: 1.4.5

    增加了 :attr:`.Dialect.supports_statement_cache` 属性。

典型的方言修改情况如下所示。

示例：使用后编译参数渲染 LIMIT / OFFSET
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

例如，假设方言重写了 :meth:`.SQLCompiler.limit_clause` 方法，该方法为 SQL 语句生成“LIMIT/ OFFSET”子句，类似于这样：

::

    # 1.4 版本前的代码
    def limit_clause(self, select, **kw):
        text = ""
        if select._limit is not None:
            text += " \n LIMIT %d" % (select._limit,)
        if select._offset is not None:
            text += " \n OFFSET %d" % (select._offset,)
        return text

上述例程在 SQL 语句中嵌入了 :attr:`.Select._limit` 和 :attr:`.Select._offset` 整数值作为字面整数。在不支持使用 SELECT 语句的 LIMIT/OFFSET 子句中使用整数值是一种常见要求。但是，在编译阶段将整数值呈现为初始值对缓存不利，因为 :class:`.Select` 对象的限制和偏移量整数值不是缓存键的一部分，因此许多 :class:`.Select` 语句的不同限制/偏移量值将不会呈现正确的值。

针对上面的代码所做的纠正是将字面整数移动到 SQLAlchemy 的 :ref:`后编译 <change_4808>` 设施中，它将在执行之前在执行阶段之外呈现字面整数。使用 :meth:`_sql.BindParameter.render_literal_execute` 方法，在编译阶段中使用 :attr:`.Select._limit_clause` 和 :attr:`.Select._offset_clause` 属性，二者都将完全表示为 SQL 表达式。如下所示：

::

    # 1.4 可缓存代码
    def limit_clause(self, select, **kw):
        text = ""

        limit_clause = select._limit_clause
        offset_clause = select._offset_clause

        if select._simple_int_clause(limit_clause):
            text += " \n LIMIT %s" % (
                self.process(limit_clause.render_literal_execute(), **kw)
            )
        elif limit_clause is not None:
            # 假设 DB 不支持 LIMIT 的 SQL 表达式。否则此处正常呈现
            raise exc.CompileError(
                "dialect 'mydialect' can only render simple integers for LIMIT"
            )
        if select._simple_int_clause(offset_clause):
            text += " \n OFFSET %s" % (
                self.process(offset_clause.render_literal_execute(), **kw)
            )
        elif offset_clause is not None:
            # 假设 DB 不支持 OFFSET 的 SQL 表达式。否则此处正常呈现
            raise exc.CompileError(
                "dialect 'mydialect' can only render simple integers for OFFSET"
            )

        return text

上述方法将生成一个编译 SELECT 语句，它如下所示：

.. sourcecode:: sql

    SELECT x FROM y
    LIMIT __[POSTCOMPILE_param_1]
    OFFSET __[POSTCOMPILE_param_2]

在上面的语句中，“__[POSTCOMPILE_param_1]”和“__[POSTCOMPILE_param_2]”指标将在执行语句时填充其相应的整数值，而在 SQL 字符串从缓存中检索到之后。

进行必要的更改后， :attr:`.Dialect.supports_statement_cache` 标志应设置为 ``True``。强烈建议第三方方言使用
`方言第三方测试套件 <https://github.com/sqlalchemy/sqlalchemy/blob/main/README.dialects.rst>`_ 来验证诸如使用带 LIMIT/OFFSET 的 SELECT 正确呈现并缓存的操作。

.. seealso::

    :ref:`faq_new_caching` - 在 :ref:`faq_toplevel` 部分

.. _engine_lambda_caching:

使用 Lambda 将语句生成速度显著提高
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. deepalchemy:: 通常情况下，除了在非常高级的性能方案中，这种技术并不是必需的，面向有经验的 Python 程序员。虽然相当直截了当，但它涉及到元编程概念，这并不适合初学者的 Python 开发人员。Lambda 方法可以在之后的时间应用于现有代码，而且付出的代价也很小。


Python 函数，通常表示为 Lambda，可以用于生成基于 Python 代码位置的 SQL 表达式并缓存在界面上，以及在 Lambda 闭包变量内部引用。其理念是允许缓存 SQL 表达式结构的编译形式以及 Python 中构建 SQL 表达式结构本身，后者也具有某些 Python 开销的程度。

Lambda SQL 表达式功能可用作性能增强功能，同时在 :func:`_orm.with_loader_criteria` ORM 选项中为提供通用 SQL 片段。

概要
^^^^^^^^

Lambda 语句是使用 :func:`_sql.lambda_stmt` 函数构造的，它返回 :class:`_sql.StatementLambdaElement` 的实例，它本身是可执行的语句构造。可以使用 Python 加法运算符“+”或 :meth:`_sql.StatementLambdaElement.add_criteria` 方法将其他修饰符和标准添加到对象中。

假定 :func:`_sql.lambda_stmt` 构造是在包含函数或方法中调用的，它期望在应用程序中多次使用，因此除了第一次调用之外，在后续调用中，可以利用已经编译生成的 SQL 语句缓存。在闭包中且是 Python 文本值的假定值将成为缓存键并在每次运行时从闭包中提取。

如下所示：

::

    from sqlalchemy import lambda_stmt


    def run_my_statement(connection, parameter):
        stmt = lambda_stmt(lambda: select(table))
        stmt += lambda s: s.where(table.c.col == parameter)
        stmt += lambda s: s.order_by(table.c.id)

        return connection.execute(stmt)


    with engine.connect() as conn:
        result = run_my_statement(some_connection, "some parameter")

在上面的示例中，用于定义 SELECT 语句结构的三个 ``lambda`` 可调用仅被调用一次，并且生成的 SQL 字符串缓存在引擎的编译缓存中。从那时起，“run_my_statement()”函数可以任何次调用，且 ``lambda`` 调用在其中不会被调用，只使用它们作为缓存键检索已编译的 SQL。

.. note::
    在没有使用 Lambda 系统的情况下，SQLAlchemy 中已经使用了 SQL 缓存系统。Lambda 系统只会在每次调用时额外减少一层工作量，通过缓存 SQL 构造本身以及使用更简单的缓存键。

Lambda 的快速指南
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

总体而言，在 Lambda SQL 系统中的重点是确保 Lambda 的缓存键生成和它所生成的 SQL 字符串永远不会存在不匹配。:class:`_sql.LambdaElement` 和相关对象将运行 Lambda 并分析给定 Lambda 的语句中的所有内容，以计算如何在每次运行中对其进行缓存，尝试检测任何潜在问题。基本准则包括：

* **支持任何类型的语句** - 虽然期望 :func:`_sql.select` 构造体是 :func:`_sql.lambda_stmt` 的主要用例，但 DML 语句诸如 :func:`_sql.insert` 和 :func:`_sql.update` 同样可用：

::

    def upd(id_, newname):
        stmt = lambda_stmt(lambda: users.update())
        stmt += lambda s: s.values(name=newname)
        stmt += lambda s: s.where(users.c.id == id_)
        return stmt


    with engine.begin() as conn:
        conn.execute(upd(7, "foo"))

  ..

* **ORM 用例直接支持** - :func:`_sql.lambda_stmt`
  完全适应 ORM 功能，并可直接与 :meth:`_orm.Session.execute` 结合使用。

::

    def select_user(session, name):
        stmt = lambda_stmt(lambda: select(User))
        stmt += lambda s: s.where(User.name == name)

        row = session.execute(stmt).first()
        return row

  ..

* **绑定参数会自动适应** - 与 SQLAlchemy 以前的“烤查询”系统不同，Lambda SQL 系统会自动适应将成为 SQL 绑定参数的 Python 文本值。这意味着即使某个 Lambda 仅执行一次，也会从 Lambda 的闭包中获取绑定值用于每次运行：

  .. sourcecode:: pycon+sql

        >>> def my_stmt(x, y):
        ...     stmt = lambda_stmt(lambda: select(func.max(x, y)))
        ...     return stmt
        >>> engine = create_engine("sqlite://", echo=True)
        >>> with engine.connect() as conn:
        ...     print(conn.scalar(my_stmt(5, 10)))
        ...     print(conn.scalar(my_stmt(12, 8)))
        {execsql}SELECT max(?, ?) AS max_1
        [generated in 0.00057s] (5, 10){stop}
        10
        {execsql}SELECT max(?, ?) AS max_1
        [cached since 0.002059s ago] (12, 8){stop}
        12

  上述代码显示了 :class:`_sql.StatementLambdaElement` 如何从 Lambda 闭包中提取 ``x`` 和 ``y`` 的值，这些值在每次运行时将用作缓绑定参数，而 Lambda 或其中任何函数都不会在其中调用。


* **Lambda 应该始终生成相同的 SQL 结构** - 避免在 Lambda 中使用有条件的结构或自定义可调用项，这可能会基于输入产生不同的 SQL，如果一个函数可能有两个不同的 SQL 片段，则使用两个单独的 Lambda：

        # **不要** 这样做:

        def my_stmt(parameter, thing=False):
            stmt = lambda_stmt(lambda: select(table))
            stmt += (
                lambda s: s.where(table.c.x > parameter)
                if thing
                else s.where(table.c.y == parameter)
            )
            return stmt


        # **要** 这样做:


        def my_stmt(parameter, thing=False):
            stmt = lambda_stmt(lambda: select(table))
            if thing:
                stmt += lambda s: s.where(table.c.x > parameter)
            else:
                stmt += lambda s: s.where(table.c.y == parameter)
            return stmt

  如果 Lambda 不生成一致的 SQL 构造，则可能会发生多种失败，其中一些目前无法轻松检测。

* **不要在 Lambda 中使用函数生成绑定值** - 绑定值跟踪方法需要在 Lambda 的闭包中有要在 SQL 中使用的实际值。 如果值是从其他函数生成的，则不可能， :class:`_sql.LambdaElement` 通常会引发错误：

    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     stmt = lambda_stmt(lambda: select(func.max(get_x(), get_y())))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    Traceback (most recent call last):
      # ...
    sqlalchemy.exc.InvalidRequestError: Can't invoke Python callable get_x()
    inside of lambda expression argument at
    <code object <lambda> at 0x7fed15f35450, file
    "<stdin>", line 6>; lambda SQL constructs should not invoke functions from
    closure variables to produce literal values since the lambda SQL system
    normally extracts bound values without actually invoking the lambda or
    any functions within it.

  上述错误表明 :class:`_sql.LambdaElement` 系统不会假定传递的 ``Foo`` 对象始终保持不变。它还不会假定可以默认使用 ``Foo`` 作为缓存键。如果重复使用许多不同的 ``Foo`` 对象，这将填充具有重复信息的缓存，并且还将引用到这些对象的长时间参考。

  解决上述问题的最佳方法是不在 Lambda 中引用 ``foo``，而是在 **外部** 引用它：

    >>> def my_stmt(foo):
    ...     x_param, y_param = foo.x, foo.y
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt

  在某些情况下，如果 Lambda 的 SQL 构造保证基于输入永远不会发生更改，则可以将 ``track_closure_variables=False`` 传递给它，这将禁用除用于绑定参数的 closure 之外的任何 closure 变量跟踪：

    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(func.max(foo.x, foo.y)), track_closure_variables=False
    ...     )
    ...     return stmt


* **Lambda 内不要引用非 SQL 构造** - 这个问题是关于 :class:`_sql.LambdaElement` 如何从语句的闭包变量中创建一个缓存键的通知问题。为了提供最好的一致缓存键的保证， :class:`_sql.LambdaElement` 认为闭包中的所有对象都是重要的，并且默认情况下不会使用任何对象链作为缓存键。因此，如果 :class:`_sql.LambdaElement` 将 ``Foo`` 对象作为缓存键使用，如果有许多不同的 ``Foo`` 对象，这将填充缓存键，并且还将引用到所有这些对象的持久参考。

  为了解决上述情况，最好的方法是不在 Lambda 中引用 ``foo``，而是在 **外部** 引用它：

::

    >>> class Foo:
    ...     def __init__(self, x, y):
    ...         self.x = x
    ...         self.y = y
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(lambda: select(func.max(foo.x, foo.y)))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(Foo(5, 10))))

  在某些情况下，如果 Lambda 的 SQL 构造保证基于输入永远不会发生更改，则可以启用 ``track_closure_variables=False`` 避免单个使用的变量被作为缓存键使用。

  使用 ``track_on`` 参数也有一个选项可以将对象添加到元素中以显式形式形成缓存键。使用此参数允许特定值用作缓存键，并将阻止考虑其他 closure 变量。这对于从上下文对象等地方产生的 SQL 的情况非常有用，该 SQL 可能具有许多不同的值。在下面的示例中，SELECT 语句的第一段将禁用 ``foo`` 变量的跟踪，而第二段删除 ``self`` 以作为缓存键明确跟踪：

::

    >>> def my_stmt(self, foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(*self.column_expressions), track_closure_variables=False
    ...     )
    ...     stmt = stmt.add_criteria(lambda: self.where_criteria, track_on=[self])
    ...     return stmt


缓存键的生成
^^^^^^^^^^^^^^^^^^^^

为了理解 Lambda SQL 构造发生了哪些选项和行为，需要了解缓存系统的工作原理。正常情况下，SQLAlchemy 的缓存系统通过从构造中生成表示所有状态的结构来生成缓存键::

    >>> from sqlalchemy import select, column
    >>> stmt = select(column("q"))
    >>> cache_key = stmt._generate_cache_key()
    >>> print(cache_key)  # 稍作修改以简化说明
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
      # 更多元素在此处，更复杂的 SELECT 语句有更多元素。
    ),)


上面的缓存键存储在缓存中，该缓存基本上是一个字典，其值是一个构造体，其中包含在 SQL 语句字符串中存储的字符串形式，例如短语“SELECT q”。即使对于极短的查询，我们也可以看到缓存键非常详细，因为它必须表示关于所呈现和潜在执行的每个变化因素都不相同的一切？？？

Lambda 文本值系统与之相比生成了不同类型的缓存键：

::

    >>> from sqlalchemy import lambda_stmt
    >>> stmt = lambda_stmt(lambda: select(column("q")))
    >>> cache_key = stmt._generate_cache_key()
    >>> print(cache_key)
    CacheKey(key=(
      <code object <lambda> at 0x7fed1617c710, file "<stdin>", line 1>,
      <class 'sqlalchemy.sql.lambdas.StatementLambdaElement'>,
    ),)

上面是一个远远比未使用 Lambda 系统的非 Lambda 语句短得多的缓存键。而且，在构造前，它们甚至不必生产``select(column("q"))``。此外，Python Lambda 本身包含一个名为 “__code__” 的属性，该属性指向在应用程序运行时是不可变且永久的 Python 代码对象。

当 Lambda 还包括 closure 变量时，在通常情况下，如果这些变量引用 SQL 构造，它们将成为缓存键的一部分，或者如果它们引用将成为绑定参数的文字值，它们将被放置在缓存键的一个单独元素中：

::

    >>> def my_stmt(parameter):
    ...     col = column("q")
    ...     stmt = lambda_stmt(lambda: select(col))
    ...     stmt += lambda s: s.where(col == parameter)
    ...     return stmt

上面的 :class:`_sql.StatementLambdaElement` 包括两个Lambda，两者都引用了闭包变量 ``col``，因此缓存键将表示这两段以及 ``column()`` 对象：

::

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


缓存键的第二部分已检索 the bound parameters，它们将在语句被调用时使用：

::

    >>> key.bindparams
    [BindParameter('%(139668884281280 parameter)s', 5, type_=Integer())]


关于插入语句的“Insert Many Values”行为
---------------------------------------------------

.. versionadded:: 2.0 请参见 :ref:`change_6047` 以了解更多有关更改的背景信息，包括样本性能测试

.. tip:: :term:`insertmanyvalues` 功能是一种**透明可用**的性能功能，对于此功能执行并不需要用户干预或可以自由选择。本节会描述该功能的架构以及如何测量其性能并调整其行为以优化大容量 INSERT 语句的速度，特别是 ORM 的使用。ORM中的性能问题取决于能够检索服务器生成的主键值以正确填充:term:`identity map`。

随着对SQLite和MariaDB的RETURNING支持的最新支持，SQLAlchemy几乎不再依赖于大多数后端的DBAPI提供的单行-only`cursor.lastrowid <https://peps.python.org/pep-0249/#lastrowid>`_属性;现在除了MySQL外，所有：ref：`` SQLALchemy-included``后端都可以使用RETURNING。存在的剩余性能限制是``cursor.executemany() <https://peps.python.org/pep-0249/#executemany>`` DBAPI方法不允许抓取行，对于大多数后端可以通过放弃使用``executemany()``并重新构建单个INSERT语句来解决，以便每个语句都适应单个命令。使用``cursor.execute()``调用的大量行。此方法起源于``psycopg2`` DBAPI的“快速执行帮助器”功能，SQLAlchemy在最近的发布系列中逐步增加了对其的支持。

当前支持
~~~~~~~~~~~~~~~

该功能适用于所有支持RETURNING的SQLAlchemy中包含的后端，除了已由cx_Oracle和OracleDB驱动程序提供其自己的等效特性之外的Oracle。通过使用：meth：`_dml.Insert.returning`方法和：ref：`executemany`执行将一个字典列表传递给：paramref：`_engine.Connection.execute.parameters`参数的：meth：`_engine.Connection.execute`或：meth：`_orm.Session.execute`方法（以及：ref：`asyncio <asyncio_toplevel>`和缩写方法，如：meth：`_orm.Session.scalars`），更改将在ORM：term：`单元操作`进程中进行添加的方法，如：meth：`_orm.Session.add`和：meth：`_orm.Session.add_all`。 SQLAlchemy包含的方言的支持或等效支持目前如下：

* SQLite - 支持SQLite 3.35及以上版本的SQLite
* PostgreSQL - 所有支持的Postgresql版本（9及以上）
* SQL Server - 所有支持的SQL Server版本[#]_
* MariaDB - 支持MariaDB 10.5及以上版本
* MySQL - 不支持，不存在RETURNING功能
* Oracle - 支持RETURNING使用本机cx_Oracle / OracleDB使用多行OUT数据库API参数的executemany，对所有支持的Oracle版本9及以上，这不是与“executemanyvalues”的实现相同，但具有相同的使用模式和等效的性能优势。

.. versionchanged :: 2.0.10

   .. [#] Microsoft SQL Server的“insertmanyvalues”支持
      在2.0.9版本后被暂时禁用后恢复。

禁用功能
~~~~~~~~~~~~~~~~~~~~~

要为给定的后端禁用“insertmanyvalues”功能，可以将:paramref：`_sa.create_engine.use_insertmanyvalues`参数设置为“False”传递给：func：`_sa.create_engine`::

    engine = create_engine(
        "mariadb+mariadbconnector://scott:tiger@host/db", use_insertmanyvalues=False
    )

该功能还可以通过传递:paramref：`_schema.Table.implicit_returning`参数为“False”的方式禁用特定：class：`_schema.Table`对象中使用的隐式返回：

      t = Table(
          "t",
          metadata,
          Column("id", Integer, primary_key=True),
          Column("x", Integer),
          implicit_returning=False,
      )

禁用返回具体表格是因为工作回避后端特定的限制。

批处理模式操作
~~~~~~~~~~~~~~~~~~~~~~

该功能有两种操作模式，可以在每个后端，每个:class：`_schema.Table`上选择。一个是**批处理模式**，它通过重写如下所示的INSERT语句来减少数据库往返次数：

.. sourcecode :: sql

    INSERT INTO a (data, x, y) VALUES (%(data)s, %(x)s, %(y)s) RETURNING a.id

成为像这样的“批处理”形式：

.. sourcecode :: sql

    INSERT INTO a (data, x, y) VALUES
        (%(data_0)s, %(x_0)s, %(y_0)s),
        (%(data_1)s, %(x_1)s, %(y_1)s),
        (%(data_2)s, %(x_2)s, %(y_2)s),
        ...
        (%(data_78)s, %(x_78)s, %(y_78)s)
    RETURNING a.id

在上面，该语句根据输入数据的子集（“批处理”）进行组织，其大小由数据库后端以及批处理的参数数量确定，以对应于语句大小/参数数量的已知限制。该功能然后为每个输入数据的批次执行一次INSERT语句，直到消耗所有记录，将每个批次的RETURNING结果连接到单个大的结果集中，该结果集可以从单个:class：`_result.Result`对象中获得。

这种“批处理”形式允许使用插入很少的数据库回合即可插入多行，并已经显示在大多数后端中可以显着提高性能。

.. _engine_insertmanyvalues_returning_order:

将返回行与参数集相关联
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded :: 2.0.10

在前面的小节中描绘的“批量”模式查询不保证返回的记录顺序与输入数据的顺序相对应。当由SQLAlchemy ORM进行的：term：`单元操作`过程使用时，以及对于将返回的服务器生成的值与输入数据相关联的应用程序，：meth：`_dml.Insert.returning`和：meth：`_dml.UpdateBase.return_defaults`方法包括一个选项：paramref：`_dml.Insert.returning.sort_by_parameter_order`，指示“ insertmanyvalues”模式应保证此对应关系。这与记录实际上插入的顺序无关，该顺序在任何情况下都不受到假设; 只有当接收到来自服务器时返回的记录应该根据传递的原始输入数据的顺序进行组织。

当存在:paramref：`_dml.Insert.returning.sort_by_parameter_order`参数时，对于使用服务器生成的整数主键值（例如``IDENTITY``，PostgreSQL``SERIAL``，MariaDB``AUTO_INCREMENT``或SQLite的``ROWID``方案等）的表，批处理模式可以选择使用更复杂的INSERT..RETURNING形式，以及基于返回的值进行的行排序，或者如果不可用，则“insertmanyvalues”功能可以平滑地降级为“非批量”模式，该模式可为每个参数设置运行单个INSERT语句。

例如，在SQL Server上，当自动增加的IDENTITY列用作主键时，使用以下SQL格式：

.. sourcecode:: sql

    INSERT INTO a (data, x, y)
    OUTPUT inserted.id, inserted.id AS id__1
    SELECT p0, p1, p2 FROM (VALUES
        (?, ?, ?, 0), (?, ?, ?, 1), (?, ?, ?, 2),
        ...
        (?, ?, ?, 77)
    ) AS imp_sen(p0, p1, p2, sen_counter) ORDER BY sen_counter

当主键列使用SERIAL或IDENTITY时，PostgreSQL也使用类似的格式。以上形式**不保证**插入行的顺序。但是，它确保IDENTITY或SERIAL值会随每个参数集被创建而按顺序创建[#]_。然后，“insertmanyvalues”特征按递增整数标识符对返回的行进行排序。

对于SQLite数据库，不存在适当的INSERT形式可以将新的ROWID值与传递的参数集的顺序相关联。因此，在使用服务器生成的主键值时，当请求有序的RETURNING时，SQLite后端将降级为“非批量”模式。
对于MariaDB，默认的INSERT表格“insertmanyvalues”形式已足够，因为此数据库后端将一行与InnoDB使用的输入数据的顺序相匹配[#]_。

对于具有客户端设备生成的主键的情况，例如使用Python``uuid.uuid4()``函数生成:new:的:class：`.Uuid`列的新值，特征*“insertmanyvalues”*会透明地将此列包含在RETURNING记录中，并将其值与给定的输入记录相关联，从而维护输入记录和结果行之间的对应关系。由此可见，所有后端都允许在使用客户端生成的主键值时进行批处理，参数相关的RETURNING顺序。

“insertmanyvalues”“批”模式确定要在输入参数和RETURNING行之间使用哪一列或列成为:term：`插入哨兵`，这是一列或列的特定样式的:term：，用于跟踪此类价值。插入哨兵通常会自动选择，但是在极端的特殊情况下也可以由用户配置;关于最后一节：ref:`engine_insertmanyvalues_sentinel_columns`描述。

对于无法以确定性或可排序的方式相对于多个参数集引用的现有的服务器生成的主键值（或没有主键）的:class：`_schema.Table`配置，*“insertmanyvalues”*当为：paramref：`_dml.insert`语句要求保证RETURNING排序时，特性可以选择使用：ref：`non-batched`模式。

在此模式下，将维护INSERT的原始SQL格式，并且当为:class：`_dml.Insert`语句所需的：paramref：`_dml.Insert.returning.sort_by_parameter_order`提供保证RETURNING排序时，“insertmanyvalues”特性将为每个参数集的语句运行一次该语句，将返回的行组织成完整的结果集。与SQLAlchemy早期版本不同，它确实可以通过紧密循环进行，以最小化Python开销。在某些情况下，例如在SQLite上，“非批量”模式与“批量”模式的表现完全相同。

语句执行模型
~~~~~~~~~~~~~~~~~~~~~~~~~

对于“批量”模式和“非批量”模式，功能将必然调用**多个INSERT语句**，使用DBAPI``cursor.execute()``方法，在** Core-level **范围内的方法:meth:`_engine.Connection.execute`，每个语句包含多达一个已知语句大小/ 参数数量的参数字典的固定限制。当参数字典的数量超过固定限制，或者总要绑定的参数数量在单个INSERT语句中超过固定限制（两个固定限制是分开的）时，将在单个：meth：`_engine.Connection.execute`调用的范围内调用多个INSERT语句，每个语句都适合于一个参数的一部分字典，称为“批次”。每个“批处理”中表示的参数字典的数量然后称为“批量”。例如，批大小为500表示每个插入语句将至多插入500行。

可能很重要的是能够调整“批量大小”，因为较大的批量大小可能对将互连的值集相对较小的插入更有性能，而较小的批量大小可能更适合使用非常大的值集的插入，其中渲染的SQL的大小以及单个statement中传递的总数据大小可能受益于基于后端行为和内存约束的大小限制。因此，批次大小可以在:class：`。Engine`上进行配置，也可以以每条语句为基础进行配置。参数限制另一方面基于正在使用的数据库的已知特性而固定。

大多数后端的批处理大小默认为1000，还存在每种SQL的max参数号码的dialect-specific限制因素，这些限制因素可能在每个语句的基础上进一步减少批处理大小。参数号码的最大数量因dialect和服务器版本而异;最大的大小为32700（以健康的距离远离PostgreSQL的32767和SQLite的现代32766界限，同时留出语句中的其他参数和DBAPI的怪癖的余地）。SQLite之前（在3.32.0之前）的旧版本将此值设置为999。MariaDB没有建立具体限制，但32700仍然是SQL消息大小的限制因素。

可以通过：class：`_engine.Engine`在全局影响批量大小：paramref：`_sa.create_engine.insertmanyvalues_page_size`参数。例如，要在每个语句中包括最多100个参数集：

    e = create_engine("sqlite://", insertmanyvalues_page_size=100)

可以使用：paramref：`_engine.Connection.execution_options.insertmanyvalues_page_size`一次性地在每个执行中以每个语句为基础影响批处理大小::

    with e.begin() as conn:
        result = conn.execute(
            table.insert().returning(table.c.id),
            parameterlist,
            execution_options={"insertmanyvalues_page_size": 100},
        )

也可以在语句本身上进行配置：

    stmt = (
        table.insert()
        .returning(table.c.id)
        .execution_options(insertmanyvalues_page_size=100)
    )
    with e.begin() as conn:
        result = conn.execute(stmt, parameterlist)

.. _engine_insertmanyvalues_events:

记录和事件
~~~~~~~~~~~~~~~~~~

“insertmanyvalues”特性与SQLAlchemy的:ref：`语句记录<dbengine_logging>`以及游标事件（例如：meth：`.ConnectionEvents.before_cursor_execute`）完全集成 。当参数列表被分成单独的批次时，**每个INSERT语句都会单独记录并传递到事件处理程序中** 。这是与SQLAlchemy 1.x系列中仅适用于psycopg2的功能在生产多个INSERT语句时隐藏了记录和事件的情况的主要更改。为了便于阅读，日志显示将截断长列表以及指示每个语句的特定批次。下面的示例显示了这种记录的摘录：

.. sourcecode:: text

  INSERT INTO a (data, x, y) VALUES (?, ?, ?)，... 795个字符被截短 ... (?, ?, ?),(?, ?, ?) RETURNING id 
  [在0.00177s（insertmanyvalues）1/10（unordered）中生成]（'d0'，0，0，'d1'，... 
  INSERT INTO a (data, x, y) VALUES (?, ?, ?)，... 795个字符被截短 ...，（？，？，？），...，（？，？，？））RETURNING id 
  [insertmanyvalues 2/10（unordered）]（'d100'，100，1000，'d101'，...

  ...

  INSERT INTO a (data, x, y) VALUES (?, ?, ?)，... 795个字符被截短 ..  （？，？，？），（？，？，？），（？，？，？）），... RETURNING id 
  [insertmanyvalues 10/10（unordered）]（'d900'，900，9000，'d901'，...

当:ref:` non-batch模式 <engine_insertmanyvalues_non_batch>`进行时，记录将指示此操作以及只执行插入的消息：“insertmanyvalues”。

.. seealso：

    :ref:`dbengine_logging`

Upsert支持
~~~~~~~~~~~~~~

PostgreSQL，SQLite和MariaDB方言提供了针对后端特定的“upsert”构造:func：`_postgresql.insert`，:func：`_sqlite.insert`
和：func：`_mysql.insert`，它们是:class：`_dml.Insert`构造，具有额外的方法（如``on_conflict_do_update()``或``on_duplicate_key()``）这些构造也支持在使用RETURNING的情况下进行“insertmanyvalues”行为，以允许高效的使用RETURNING对程序间插入进行操作。

.. _engine_disposal:

引擎处置
---------------

:class：`_engine.Engine`是一个连接池引用，这意味着在正常情况下，在内存中仍存在:class：`_engine.Engine`对象时，存在开放式数据库连接。当一个:class：`_engine.Engine`
被垃圾收集时，它的连接池不再由该:class：`_engine.Engine`引用，并且假设它的所有连接都没有被检查出，该池及其连接也将被垃圾收集，这将关闭实际的数据库连接。但是，否则，
:class：`_engine.Engine`将保留开放式数据库连接，假设它使用通常的:class：`.QueuePool`池实现。

:class：`_engine.Engine`通常是在应用程序的整个生命周期内建立并维护的永久设施。它不**旨在**按连接的方式创建和丢弃;相反，它是一个注册表，它维护连接池以及有关正在使用的数据库和DBAPI的配置信息以及一些内部缓存的特定于每个数据库类型的资源。

然而，有很多情况下，希望完全关闭:class：`_engine.Engine`引用的所有连接资源。通常不建议依赖Python垃圾回收来进行这种情况的控制;相反，可以使用
:meth:`_engine.Engine.dispose`方法显式处置引擎。这将处置引擎的底层连接池并用新池替换它，新池为空。只要在此时丢弃:class：`_engine.Engine`并再也不使用它，所有**检入**连接将关闭。

调用:meth：`_engine.Engine.dispose`的有效用例包括：

* 当程序希望释放连接池中剩余的所有已检入连接并希望将来不再连接到该数据库时使用。

* 当程序使用多进程或“fork（）”，并且将
  :class：`_engine.Engine`对象复制到子进程中，:meth:`_engine.Engine.dispose`应该被调用，以便引擎创建。
  
  .... 
 
需要明确说明的是，已记录执行穿越时间较长的事务（例如事务控件管理或使用大量条目的批处理操作）可能会异常地关闭，这取决于系统限制。对于这种情况，可能需要将事务事务拆分为可设置的较小单元操作。

连接**已检出**的连接在dispose或垃圾收集:class：`_engine.Engine`时不会被丢弃，因为这些连接在应用程序的其他地方仍然有强引用。但是，在:meth：`_engine.Engine.dispose`被调用后，这些连接将再次与该:class相关联：' _engine.Engine'；当它们关闭时，它们将返回到其现在孤立的连接池，而连接池最终将被垃圾回收，只要所有引用到它的连接也不再被引用您可以查看 pool.pprint_all() 并pool.pprint_checkout() 查看连接池的状态，并查看哪些连接当前被查出和哪些连接处于空闲状态。
因为这个过程不容易控制，所以强烈建议只有在**检查**所有被检查的连接被归还或以其他方式被解关该连接池时才调用：meth：`_engine.Engine.dispose`。
对于受:class：`_engine.Engine`对象的使用连接池的应用程序的负面影响的应用程序，另一个选择是禁用池(pooling)完整的连接池使用和关闭策略
--------------------------------------------

SQLAlchemy通过使用类似于生命周期的连接池模型来管理数据库连接。这意味着当您在使用连接并且再也不需要它了时，您应该将其关闭以返回到连接池，以便在以后的时间继续使用之前的连接而不是重新连接。这意味着一个连接被检查关闭时，它被完全关闭并不在内存中，这通常只会对新连接的使用产生适度的性能影响。有关如何禁用连接池的指南，请参阅 :ref: `pool_switching`。

.. 参见::

    :ref:`pooling_toplevel`

    :ref:`pooling_multiprocessing`

.. _dbapi_connections:

使用驱动程序 SQL 和原始 DBAPI 连接
-------------------------------------------------

在使用 :meth:`_engine.Connection.execute` 时，介绍了如何使用 :func:`_expression.text` 构造来说明如何调用文本 SQL 语句。使用 SQLAlchemy 时，文本 SQL 其实更多是例外而不是正常情况，因为核心表达式语言和 ORM 都将 SQL 的文本表示抽象掉了。但是，:func:`_expression.text`构造本身也提供了一些文本 SQL 的抽象，因为它规范了如何传递绑定的参数，以及它支持参数和结果集行的数据类型行为。

直接调用驱动程序的 SQL 字符串
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于希望直接传递给底层驱动程序（称为 DBAPI）的文本 SQL 的用例，可以使用 :meth:`_engine.Connection.exec_driver_sql` 方法::

    with engine.connect() as conn:
        conn.exec_driver_sql("SET param='bar'")

.. versionadded:: 1.4  添加了 :meth:`_engine.Connection.exec_driver_sql` 方法。

.. _dbapi_connections_cursor:

直接使用 DBAPI 光标
~~~~~~~~~~~~~~~

有些情况下，SQLAlchemy 并没有为访问某些 DBAPI 函数提供泛化的方式，例如调用存储过程以及处理多个结果集。在这些情况下，通过 DBAPI 连接直接操作是同样方便的。

访问原始的 DBAPI 连接所采用的最常见方式是直接从已有的 :class:`_engine.Connection` 对象中获取它。它可以使用 :attr:`_engine.Connection.connection` 属性进行访问::

    connection = engine.connect()
    dbapi_conn = connection.connection

然而，DBAPI 连接在这里实际上是被代理的，因为它由起始连接池处理，但这是可以忽略的实现细节。由于这个 DBAPI 连接仍然包含在拥有 :class:`_engine.Connection` 对象的范围内，因此最好使用 :class:`_engine.Connection` 对象来进行诸如事务控制以及调用 :meth:`_engine.Connection.close` 方法等大部分功能；如果这些操作直接在 DBAPI 连接上执行，则拥有的 :class:`_engine.Connection` 不会意识到状态的这些变化。

为了克服由拥有 :class:`_engine.Connection` 维护的 DBAPI 连接所施加的限制，还可以在不需要使用 :class:`_engine.Connection` 的情况下获取 DBAPI 连接，使用 :class:`_engine.Engine` 的 :meth:`_engine.Engine.raw_connection` 方法::

    dbapi_conn = engine.raw_connection()

这个 DBAPI 连接再次是一个 “被代理的” 形式，就像之前的情况一样。现在这种代理的目的是显然的，因为当我们调用 ``.close()`` 该连接的方法时，DBAPI 连接通常并没有被实际关闭，而是被 “释放” 回到引擎的连接池中::

    dbapi_conn.close()

虽然 SQLAlchemy 可能将来会为更多的 DBAPI 使用情况添加内置模式，但是这些情况往往很少出现，并且它们也高度依赖于使用的 DBAPI 类型，因此在任何情况下，直接 DBAPI 调用模式总是有可能需要的。

.. 参见::

    :ref:`faq_dbapi_connection` - 包括有关如何访问 DBAPI 连接以及在使用 asyncio 驱动程序时“驱动程序”连接的其他详细信息。

以下是一些关于 DBAPI 连接使用的配方。

.. _stored_procedures:

调用存储过程和用户定义函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 支持调用存储过程和用户定义函数的多种方法。请注意，所有 DBAPI 都有不同的做法，因此您必须查看您的基础 DBAPI 文档，以了解与您的特定用法相关的详细信息。以下示例是假设的，可能无法与您的基础 DBAPI 一起使用。

对于具有特殊语法或参数问题的存储过程或函数，DBAPI 级别的 `callproc <https://legacy.python.org/dev/peps/pep-0249/#callproc>`_ 可能可以用于您的 DBAPI。这种模式的示例是::

    connection = engine.raw_connection()
    try:
        cursor_obj = connection.cursor()
        cursor_obj.callproc("my_procedure", ["x", "y", "z"])
        results = list(cursor_obj.fetchall())
        cursor_obj.close()
        connection.commit()
    finally:
        connection.close()

..注意::

  并非所有 DBAPI 都使用 `callproc`，并且总体使用细节会有所不同。上面的示例仅说明如何使用特定的 DBAPI 函数。

您的 DBAPI 可能不需要 ``callproc`` 或者可能需要使用另一种模式调用存储过程或用户定义的函数，例如正常的 SQLAlchemy 连接用法。在撰写本文档时，使用 psycopg2 DBAPI 在 PostgreSQL 数据库中执行存储过程的一种方法是使用正常的连接用法::

    connection.execute("CALL my_procedure();")

上面的示例是假设的。不能保证底层数据库支持在这些情况下使用 “CALL” 或 “SELECT” keyword，而这些关键字可能会因函数是存储过程还是用户定义函数而有所不同。在这些情况下，应该查询基础 DBAPI 和数据库文档以确定要使用的正确语法和模式。


多结果集
~~~~~~~~~~~~~~~~~~~~

多个结果集支持可通过从原始 DBAPI 游标使用 `nextset <https://legacy.python.org/dev/peps/pep-0249/#nextset>`_ 方法进行访问::

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

:func:`_sa.create_engine` 函数调用使用 setuptools 入口点查找给定方言。可以在 setup.py 脚本中为第三方方言建立这些入口点。例如，要创建一个新的方言 “foodialect://”，步骤如下:

1. 创建一个名为 ``foodialect`` 的包。
2. 该包应当包含一个方言类的模块，该类通常是 :class:`sqlalchemy.engine.default.DefaultDialect` 的子类。在此示例中，我们假设它称为 ``FooDialect``，并且可以通过 ``foodialect.dialect`` 访问它的模块。
3. 可以在 ``setup.cfg`` 中的如下位置建立入口点：

   .. sourcecode:: ini

          [options.entry_points]
          sqlalchemy.dialects =
              foodialect = foodialect.dialect:FooDialect

如果方言在现有的 SQLAlchemy 支持的数据库上提供对某个特定 DBAPI 的支持，则可以给出名称，其中包括数据库限定。例如，如果 ``FooDialect`` 实际上是一个 MySQL 方言，则可以像这样建立入口点：

.. sourcecode:: ini

      [options.entry_points]
      sqlalchemy.dialects
          mysql.foodialect = foodialect.dialect:FooDialect

以上入口点将像这样访问：“create_engine('mysql+foodialect://')”。


注册进程中的方言
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 还允许在当前进程中注册方言，绕过了分别安装的需求。使用 ``register()`` 函数如下::

    from sqlalchemy.dialects import registry


    registry.register("mysql.foodialect", "myapp.dialect", "MyMySQLDialect")

以上将会响应 ``create_engine("mysql+foodialect://")`` 并从 ``myapp.dialect`` 模块中加载 ``MyMySQLDialect`` 类。


连接 / 引擎 API
-----------------------

.. 自动类:: Connection
   :members:

.. 自动类:: CreateEnginePlugin
   :members:

.. 自动类:: Engine
   :members:

.. 自动类:: ExceptionContext
   :members:

.. 自动类:: NestedTransaction
    :members:
    :inherited-members:

.. 自动类:: RootTransaction
    :members:
    :inherited-members:

.. 自动类:: Transaction
    :members:

.. 自动类:: TwoPhaseTransaction
    :members:
    :inherited-members:


结果集 API
---------------

.. 自动类:: ChunkedIteratorResult
    :members:

.. 自动类:: CursorResult
    :members:
    :inherited-members:

.. 自动类:: FilterResult
    :members:

.. 自动类:: FrozenResult
    :members:

.. 自动类:: IteratorResult
    :members:

.. 自动类:: MergedResult
    :members:

.. 自动类:: Result
    :members:
    :inherited-members:

.. 自动类:: ScalarResult
    :members:
    :inherited-members:

.. 自动类:: MappingResult
    :members:
    :inherited-members:

.. 自动类:: Row
    :members:
    :private-members: _asdict, _fields, _mapping, _t, _tuple

.. 自动类:: RowMapping
    :members:

.. 自动类:: TupleResult