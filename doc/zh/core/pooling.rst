连接池
====

.. module:: sqlalchemy.pool

连接池是一种标准技术，可用于在内存中维护长时间运行的连接，使其可以有效地重复使用，并提供管理应用程序同时可能使用的连接总数的功能。

特别是对于服务器端Web应用程序，连接池是在内存中维护活动数据库连接的“池”并可在请求之间重用的标准方式。

SQLAlchemy包括多个连接池实现，这些实现集成到 :class:`_engine.Engine` 中。它们也可以直接用于希望为纯DBAPI方法添加连接池的应用程序。

连接池配置
------------

  :func:`~sqlalchemy.create_engine` 
中已经集成了一个  :class:`.QueuePool` ，并预先设置了合理的池默认值。如果您阅读本节仅了解如何启用连接池-恭喜！您已经做完了。

最常见的  :class:`.QueuePool` ~sqlalchemy.create_engine` ：``pool_size``、``max_overflow``、``pool_recycle``和
``pool_timeout``。例如::

    engine = create_engine(
        "postgresql+psycopg2://me@localhost/mydb", pool_size=20, max_overflow=0
    )

所有SQLAlchemy连接池实现都共同具有一个特点-它们都不会“预先创建”连接-在许多实现上，所有实现都会等待首次使用前等待创建连接。
此时，如果未提出更多的并发检出请求以获取更多连接，则不会创建其他连接。这就是为什么：func:`_sa.create_engine`默认使用大小为5的 :class:`.QueuePool` 非常适合的默认行为-无论应用程序是否真正需要队列起5个连接，池的大小只会增长到我们真正需要5个连接的情况下，如果应用程序同时使用了5个连接，那么使用小型池非常适合默认行为。

.. _pool_switching:

切换连接池实现
----------------

使用不同类型的连接池与  :func:`_sa.create_engine` ` poolclass``参数。此参数接受从``sqlalchemy.pool``库中导入的类，并处理了为您构建池的详细信息。

当连接池被禁用时，通常情况下使用 :class:`.NullPool` 实现即可实现这一点：
    

    from sqlalchemy.pool import NullPool

    engine = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test", poolclass=NullPool
    )

使用自定义连接函数
--------------------

请参阅部分   :ref:`custom_dbapi_args`  以了解各种自定义连接选项。


构造一个连接池
-------------------

使用  :class:`_pool.Pool` ` creator``函数是唯一需要的参数，并首先被传递，后跟任何附加选项：

    import sqlalchemy.pool as pool
    import psycopg2

    def getconn():
        c = psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")
        return c

    mypool = pool.QueuePool(getconn, max_overflow=10, pool_size=5)

可以使用  :meth:`_pool.Pool.connect`  函数从池中获取DBAPI连接。其返回值是包含在透明代理中并由DBAPI 组成的DBAPI连接：

    # 获取连接

    conn = mypool.connect()

    # 使用它
    cursor_obj = conn.cursor()
    cursor_obj.execute("select foo")

透明代理的作用是要截取 ``close ()`` 调用，以便不关闭DBAPI连接，而是将其返回到连接池中：

    # “关闭”连接。返回它到池中。
    conn.close()

在使用 asyncio DBAPI驱动程序时，此代理也将包含DBAPI连接的内部返回到池中，但不推荐使用此用法。

.. _pool_reset_on_return:

返回后重置表单
---------------

池包括“返回时重置”的行为，将在将连接返回到池时调用``rollback()``方法，并从连接中将任何现有的事务状态删除，这不仅包括未提交的数据，还包括表和行锁定。对于大多数DBAPI，对``rollback()``方法的调用是不昂贵的，如果DBAPI已经完成了事务，则它应该是一个无操作。

对于特定情况而言，可能不需要执行``rollback()``操作，例如在使用配置为“autocommit”的连接或使用没有ACID能力的数据库（例如MySQL的MyISAM引擎）时。通常出于性能原因而禁用重置次数。可以通过使用_pool.Pool的  :paramref:`_pool.Pool.reset_on_return`  将值传递为` `None``。

以下示例演示了如何在``AUTOCOMMIT``的情况下，使用  :paramref:`.create_engine.isolation_level`  参数设置和  :paramref:` _pool.Pool.reset_on_return`  参数禁用重置表单：


    non_acid_engine = create_engine(
        "mysql://scott:tiger@host/db",
        pool_reset_on_return=None,
        isolation_level="AUTOCOMMIT",
    )

上述引擎实际上在将连接返回到池时没有执行ROLLBACK；由于启用了AUTOCOMMIT，因此驱动程序也不会执行任何BEGIN操作。

自定义返回方案
----------------

“返回时重置”仅由一个``rollback()``操作组成可能不足以满足一些用例；尤其是，使用临时表的应用程序可能希望在检入连接时自动删除这些表。

对于其他服务器资源，例如准备好的语句句柄和服务器端语句缓存，可能会超过检入过程。这些具体取决于特定情况，可能会导致需要或不需要重置。

再次强调，某些后端可能提供重置的一些手段。已知有这种重置方案的两种包含在SQLAlchemy中的方言包括:微软SQL Server，其中一个名为“sp_reset_connection”的未记录但广为人知的存储过程通常用作重置方案，以及PostgreSQL，其中包含一系列明确记录的命令，包括“DISCARD”“RESET”、“DEALLOCATE”，和“UNLISTEN”。

.. note:: 下面的段落 + 示例应与 mssql/base.py 示例相匹配

以下示例说明如何使用 Microsoft SQL Server ``sp_reset_connection``存储过程替换返回时的重置行为，使用  :meth:`.PoolEvents.reset`  事件挂钩。 对于  :paramref:` _sa.create_engine.pool_reset_on_return`   我们将设置``None``值以便可以完全替换默认行为。自定义钩子的实现在所有情况下都会调用``.rollback()``，因为DBAPI的自身的跟踪提交/Rollback将与事务状态保持一致是非常重要的：

    from sqlalchemy import create_engine
    from sqlalchemy import event

    mssql_engine = create_engine(
        "mssql+pyodbc://scott:tiger^5HHH@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
        # 停用默认的“返回时重置”方案
        pool_reset_on_return=None,
    )


    @event.listens_for(mssql_engine, "reset")
    def _reset_mssql(dbapi_connection, connection_record, reset_state):
        if not reset_state.terminate_only:
            dbapi_connection.execute("{call sys.sp_reset_connection}")
        # 所以DBAPI本身知道连接已重置
        dbapi_connection.rollback()

.. versionchanged:: 2.0.0b3 添加了  :meth:`.PoolEvents.reset`  事件的附加状态参数，并确保该事件被触发了所有“重置”事件，因此它适用于自定义“重置”处理程序。以前使用  :meth:` .PoolEvents.checkin`  处理程序的方案仍然可用。

.. seealso::

    *   :ref:`mssql_reset_on_return`  - 在   :ref:` mssql_toplevel`  文档中
    *   :ref:`postgresql_reset_on_return`  - 在   :ref:` postgresql_toplevel`  文档中


记录返回事件的重置表格
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以通过将``logging.DEBUG``日志级别与``sqlalchemy.pool``记录器一起设置，或在使用  :func:`_sa.create_engine`  设置为` `"debug"``来记录池事件，包括返回时的重置：

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("postgresql://scott:tiger@localhost/test", echo_pool="debug")

上面的池将显示包括返回时的重置在内的详细日志记录：

    >>> c1 = engine.connect()
    DEBUG sqlalchemy.pool.impl.QueuePool Created new connection <connection object ...>
    DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> checked out from pool
    >>> c1.close()
    DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> being returned to pool
    DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> rollback-on-return


池活动
-------

连接池支持事件接口，允许在首次连接、每个新连接、检出连接和检入连接时执行钩子。有关详细信息，请参见  :class:`_events.PoolEvents` 。


.. _pool_disconnects:

处理断开连接
------------------------

连接池有刷新单个连接以及其所有连接集的能力，将先前的流入的连接设置为“无效”。当数据库服务器已被重新启动，并且所有先前建立的连接都不再工作时，这是一种常见用例。有两种方法可以实现这一点。

.. _pool_disconnects_pessimistic:

断开处理-悲观
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

悲观方法是在每次连接池检出时，在SQL连接上发出测试语句，以测试数据库连接是否仍然可用。该实现取决于方言特定，并使用DBAPI特定的ping方法，或者通过使用简单的SQL语句，如“SELECT 1”，来测试该语句。

该方法会在连接检出过程中增加一些负担，但是在完全消除由于过时的连接导致的数据库错误方面是最简单且最可靠的方法。调用应用程序无需关心如何组织操作以便能够从连接池中恢复过时的连接。当支持pool pre ping时，在  :func:`_sa.create_engine`   标志可以实现对悲观检查的需求：

    engine = create_engine("mysql+pymysql://user:pw@host/db", pool_pre_ping=True)

“预先ping”的功能基于每个方言，通过调用DBAPI特定的“ping”方法运行，或者如果不可用，则通过发出与等效于“SELECT 1”而抓取任何错误并将错误检测为“disconnect”情况。如果ping/错误检查确定连接无法使用，则连接将立即回收，而且所有早于当前时间的池化连接都无效，因此，由于这种情况下发生的断开检测，它们在下次使用之前将被替换为新连接。

如果当“pre ping”运行时数据库仍不可用，则初始连接会失败，并且将正常传播连接失败的错误。对于不常见的情况，在尝试了三次但仍然不可用时，"pre_ping"将放弃，并传播最后收到的数据库错误。

需要注意的是，预处理方法不能适应在事务或其他SQL操作中断开的连接。如果事务正在进行时，数据库变得不可用，事务将丢失，并引发数据库错误。虽然  :class:`_engine.Connection`  。

对于使用“SELECT 1”并通过捕获错误来检测断开连接的方言，可以通过  :meth:`_events.DialectEvents.handle_error`  钩子向新的后端特定错误消息添加支持。

.. _pool_disconnects_pessimistic_custom:

自定义/遗留悲观ping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在添加  :paramref:`_sa.create_engine.pool_pre_ping`  之前，历史上使用  :meth:` _events.ConnectionEvents.engine_connect`  事件手动执行“pre-ping”检查。对于引擎，传统方法的最常见用法已在下面给出。

这是为了参考目的，以防应用程序已经在使用这样的方法，或者需要特殊的行为：

    from sqlalchemy import exc
    from sqlalchemy import event
    from sqlalchemy import select

    some_engine = create_engine(...)


    @event.listens_for(some_engine, "engine_connect")
    def ping_connection(connection, branch):
        if branch:
            # 该参数始终为False，由于SQLAlchemy 2.0，但仍可由事件挂钩接受。在1.x版本中，应跳过“分支”连接。
            return
        try:
            # 运行 SELECT 1。使用核心select() 以便无表地选择标量值。
            connection.scalar(select(1))
        except exc.DBAPIError as err:
            # 转到SQLAlchemy的DBAPIError，它是包装DBAPI的例外。它包括一个.connection_invalidated属性，指定此连接是否为“disconnect”条件，它基于方言中对原始异常的检查。
            if err.connection_invalidated:
                # 再次运行相同的SELECT——连接将重新验证它自己并建立一个新的连接。在此处进行的断开连接检测还导致无效化整个连接池，以便丢弃所有过时的连接。
                connection.scalar(select(1))
            else:
                raise

上面的配方的优点是我们利用了SQLAlchemy的设施来检测已知可表明数据库连接已“断开”的所有DBAPI异常以及 :class:`_engine.Engine` 的能力来正确使当前连接池无效化此情况发生时，允许当前 :class:`_engine.Connection` 重新验证到新的DBAPI连接。


断开处理-乐观
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当不使用悲观处理时，以及在引擎使用的连接在事务的使用期内在数据库关闭/重新启动时，处理过时/关闭连接的另一种方法是让SQLAlchemy在发生时处理断开连接，此时，在连接池中的所有连接都无效化，这意味着它们被认为是过时的，并在下次检出时被替换为新的连接。该行为假定  :class:`_pool.Pool`  尝试使用DBAPI连接时，如果引发与“断开”事件相对应的异常，该连接将无效。  :class:` _engine.Connection`  然后调用  :meth:`_pool.Pool.recreate`  方法，相当于无效化所有当前没有被检出的连接，以便在下次检出时使用新的连接池中的连接。在下面的代码示例中说明了该流：：

    from sqlalchemy import create_engine, exc

    e = create_engine(...)
    c = e.connect()

    try:
        # 假设数据库已重新启动。
        c.execute(text("SELECT * FROM table"))
        c.close()
    except exc.DBAPIError as e:
        # 引发异常，连接无效。
        if e.connection_invalidated:
            print("Connection was invalidated!")

    # invalidate 事件之后，一个新的连接启动一个新的Pool。
    c = e.connect()
    c.execute(text("SELECT * FROM table"))

上面的示例说明，不需要特殊处理即可刷新池，在检测到断开连接事件后继续正常运行。但是，将引发一个数据库异常，每个使用过时连接的连接将被抛出，最终会导致套个请求失败并给出500错误。因此，该方法在频繁地退出数据库时是一种“乐观”的方法。

.. _pool_setting_recycle:

设置池回收站
~~~~~~~~~~~~~~~~~~~~

可以增加设置``pool recycle``参数来增强“optimistic”处理。该参数将防止使用已过时特定连接的池，对于自动关闭特定时间后已过时的连接的数据库backend，如MySQL等，是适合的。

    from sqlalchemy import create_engine

    e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", pool_recycle=3600)

以上，在使用了一个小时以上的任何DBAPI连接将被无效化并替换和下一次检出的新连接。请注意，无效化仅在检出时适用于 :class:`_pool.Pool` 本身，不受是否使用 :class:`_engine.Engine` 的影响。


.. _pool_connection_invalidation:

关于无效化的更多信息
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

 :class:`_pool.Pool` 提供了“连接无效化”服务，允许显式无效化连接以及基于被判断为不可用而自动无效化的连接条件。

“无效化”意味着从特定的DBAPI连接中删除，并将其废弃。如果未清楚连接本身可能尚未关闭，则会调用``.close()``方法，但如果此方法失败，则将记录异常，但操作仍将继续。

当使用  :class:`_engine.Engine`  方法是显式无效化的常规入口。一个DBAPI异常，例如  :class:` .OperationalError` ，当调用诸如 ``connection.execute()`` 这样的方法时被检测为指示其为断开连接事件时被检测为无效情况。由于Python DBAPI没有用于确定异常本质的标准系统，所有SQLAlchemy方言都包含一种名为 ``is_disconnect()`` 的系统，它将检查包括字符串消息和任何潜在错误代码在内的异常对象的内容，以确定异常是否表示连接不再可用。如果是这样，则将调用  :meth:`._ConnectionFairy.invalidate`   方法，然后丢弃DBAPI连接。

当连接返回到池并调用``connection.rollback()``或``connection.commit()``方法时，根据池的“return when reset”行为引发异常时，将调用create_garbage_collection_hack（）方法。将尝试最终关闭连接，并将其丢弃。

当实现  :meth:`_events.PoolEvents.checkout` ~sqlalchemy.exc.DisconnectionError` 异常时，也会使DBAPI连接失效，指示需要进行新连接尝试。

无效化时发生的所有无效化事件都将调用  :meth:`_events.PoolEvents.invalidate`  事件。

.. _pool_new_disconnect_codes:

为断开连接方案提供新的数据库错误代码支持
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

每个SQLAlchemy方言都包含一个名为``is_disconnect()``的程序，每当DBAPI异常被捕获时就会调用该程序。由于用户使用的方言的启发式将决定接收到的错误代码是否表示数据库连接已“断开”或处于其他不可用状态，导致其应转为可回收状态，可以使用``_events.DialectEvents.handle_error`` 事件钩子自定义启发式。通常，钩子通过拥有其上的所有错误传递上下文对象，称为   :class:`.ExceptionContext`  。自定义事件钩子可以控制特定错误是否应被视为“断开”情况，以及此断开是否应导致整个连接池被使无效化。

例如，要添加支持将Oracle错误代码``DPY-1001``和``DPY-4011``视为断开连接代码，请在创建引擎后添加事件处理程序：

    import re

    from sqlalchemy import create_engine

    engine = create_engine("oracle://scott:tiger@dnsname")


    @event.listens_for(engine, "handle_error")
    def handle_exception(context: ExceptionContext) -> None:
        if not context.is_disconnect and re.match(
            r"^(?:DPI-1001|DPI-4011)", str(context.original_exception)
        ):
            context.is_disconnect = True

        return None


上述错误处理函数将针对引发的所有Oracle错误进行调用，从而包括在使用后端所依赖的断开连接错误处理的后端上进行断开连接错误处理的后端（2.0中的新功能）。

.. seealso::

     :meth:`_events.DialectEvents.handle_error` 

.. _pool_use_lifo：

使用FIFO与LIFO
-------------------

  :class:`.QueuePool` .QueuePool.use_lifo` 的标志，这标志也可以从  :func:` _sa.create_engine`   访问。将此标志设置为True将导致池的“队列”行为变为“栈”，即返回到池的最后一个连接是用于下一次请求的第一个连接。与池的先进先出行为形成循环效应使用每个连接在池中顺序使用时不同，后进先出模式允许多余的连接保持空闲状态，从而允许服务器端超时方案关闭这些连接。这是先进先出（FIFO）和后进先出（LIFO）之间的区别，基本上取决于是否希望池在空闲期间保持完整的连接集：

    engine = create_engine("postgreql://", pool_use_lifo=True, pool_pre_ping=True)

以上，我们还使用了  :paramref:`_sa.create_engine.pool_pre_ping`  标志来让从服务器端关闭的连接通过连接池优雅地处理并替换为新连接。

请注意，该标志仅适用于使用 :class:`.QueuePool` 的使用。

.. versionadded:: 1.3

.. seealso::

      :ref:`pool_disconnects` 

.. _pooling_multiprocessing：

在使用多进程或os.fork()中使用连接池
-------------------------------------------------

当使用连接池时，关键是使用连接的池**不应与多个进程共享**。 TCP连接表示为文件描述符，在通常情况下可以跨进程边界工作，这意味着这将导致用于两个或多个完全独立的Python解释器状态的文件描述符的并发访问。

根据驱动程序和操作系统的具体规格，这里出现的问题范围从不工作的连接到由多个进程同时使用的套接字连接，导致消息中断（后一种情况通常是最常见的）。

SQLAlchemy   :class:`_engine.Engine`  方法，传递  :paramref:` .Engine.dispose.close`  参数值为 ``False``。这是以便新进程不会触摸父进程的任何连接，并将启动新连接。

这是建议的方法：

1.使用:class：`。NullPool`禁用连接池。这是最简单的，一次性的系统，可以防止 :class：`_engine.Engine`使用任何连接超过一次.

    from sqlalchemy.pool import NullPool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname", poolclass=NullPool)

2.在程序生成子进程期间，初始化程序阶段内，为给定的  :class:`_engine.Engine`  传递  :paramref:` .Engine.dispose.close`  参数值为 ``False``。 这是为了使新进程不会触 touch' the parent process 的任何连接，并将开始使用新的连接集。**这是建议的方法**：

    from multiprocessing import Pool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname")


    def run_in_process(some_data_record):
        with engine.connect() as conn:
            conn.execute(text("..."))


    def initializer():
        """ensure the parent proc's database connections are not touched
        in the new connection pool"""
        engine.dispose(close=False)


    with Pool(10, initializer=initializer) as p:
        p.map(run_in_process, data)

    ..versionadded:: 1.4.33 添加了  :paramref:`.Engine.dispose.close`  参数，以允许在子流程中替换连接池而不干扰父进程使用的连接。

3.在创建子进程之前首先  :meth:`.Engine.dispose`  直接调用  :meth:` .Engine.dispose` 。 这也将使子进程开始使用新的连接池，同时确保不传输父进程的连接::

        engine = create_engine("mysql://user:pass@host/dbname")


        def run_in_process():
            with engine.connect() as conn:
                conn.execute(text("..."))


        # before process starts, ensure engine.dispose() is called
        engine.dispose()
        p = Process(target=run_in_process)
        p.start()4.可以应用事件处理器到连接池来测试跨进程共享的连接，并使它们失效：

```
    from sqlalchemy import event
    from sqlalchemy import exc
    import os

    engine = create_engine("...")


    @event.listens_for(engine, "connect")
    def connect(dbapi_connection, connection_record):
        connection_record.info["pid"] = os.getpid()


    @event.listens_for(engine, "checkout")
    def checkout(dbapi_connection, connection_record, connection_proxy):
        pid = os.getpid()
        if connection_record.info["pid"] != pid:
            connection_record.dbapi_connection = connection_proxy.dbapi_connection = None
            raise exc.DisconnectionError(
                "连接记录属于pid %s，试图在pid %s中进行检出" % (connection_record.info["pid"], pid)
            )
```
上面，我们使用类似于``pool_disconnects_pessimistic``中所述的方法来将另一个父进程中产生的DBAPI连接视为无效连接，强制池回收连接记录以建立新的连接。

上述策略将适用于在进程间共享的``_engine.Engine``的情况。上述步骤本身并不足以处理跨进程边界共享特定``_engine.Connection``的情况；最好将特定``_engine.Connection``的作用域保持在单个进程（和线程）中。也没有支持直接在进程之间共享任何一种正在进行的事务状态，例如已经开始事务并引用``_orm.Connection``实例的ORM ``_orm.Session``对象；同样最好在新进程中创建新的``_orm.Session``对象。

直接使用池实例
-----------------

可以直接使用池实现而不使用引擎。这可用于只想使用池行为而不使用所有其他SQLAlchemy特性的应用程序。在下面的示例中，使用``_sa.create_pool_from_url``获取``MySQLdb``语言方言的默认池：

```
    from sqlalchemy import create_pool_from_url

    my_pool = create_pool_from_url(
        "mysql+mysqldb://", max_overflow=5, pool_size=5, pre_ping=True
    )

    con = my_pool.connect()
    # use the connection
    ...
    # then close it
    con.close()
```

如果没有指定要创建的池的类型，将使用该方言的默认池。可以使用``poolclass``参数来直接指定池，如以下示例所示：

```
    from sqlalchemy import create_pool_from_url
    from sqlalchemy import NullPool

    my_pool = create_pool_from_url("mysql+mysqldb://", poolclass=NullPool)
```

API文档 - 可用池实现
------------------------

.. autoclass:: sqlalchemy.pool.Pool
    :members:

.. autoclass:: sqlalchemy.pool.QueuePool
    :members:

.. autoclass:: SingletonThreadPool
    :members:

.. autoclass:: AssertionPool
    :members:

.. autoclass:: NullPool
    :members:

.. autoclass:: StaticPool
    :members:

.. autoclass:: ManagesConnection
    :members:

.. autoclass:: ConnectionPoolEntry
    :members:
    :inherited-members:

.. autoclass:: PoolProxiedConnection
    :members:
    :inherited-members:

.. autoclass:: _ConnectionFairy

.. autoclass:: _ConnectionRecord