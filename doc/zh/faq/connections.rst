连接/引擎
==========

.. contents::
    :local:
     :class: faq
    :backlinks: none

如何配置日志?
-------------

参见   :ref:`dbengine_logging` .

如何池化数据库连接?  连接池在哪里维护？
-----------------------------------------------

在大多数情况下，SQLAlchemy会自动执行应用程序级别的连接池操作。对于所有包含的方言（SQLite使用“memory”数据库除外），  :class:`_engine.Engine` .QueuePool` 作为连接的池源。

要获取更多详细信息，请参见   :ref:`engines_toplevel`  和   :ref:` pooling_toplevel` .

如何传递自定义的 connect arguments 给我的数据库API ?
------------------------------------------------------------

在   :func:`_sa.create_engine`  中通过 "connect_args" 关键字函数调用接受额外的参数::

    e = create_engine(
        "mysql+mysqldb://scott:tiger@localhost/test", connect_args={"encoding": "utf8"}
    )

或者对于基础字符串和整数参数，可以在Url查询字符串中指定::

    e = create_engine("mysql+mysqldb://scott:tiger@localhost/test?encoding=utf8")

..参见::

      :ref:`custom_dbapi_args` 

"MySQL Server has gone away" 该怎么办？
------------------------------------------

该错误的主要原因是MySQL连接已超时并已被服务器关闭。 MySQL服务器会默认关闭一段时间内处于空闲状态的连接，该时间段默认为8小时。
要解决这个问题，可以通过启用  :paramref:`_sa.create_engine.pool_recycle`  参数快速设置的方法，确保超过一定秒数的连接在下次被检出时将被丢弃并被新连接代替。

为了更一般地解决服务器重启和由于网络问题而导致的临时连接丢失，连接池中的连接可能会根据更广泛的断开检测技术被重复使用并发生回收。 部分   :ref:`pool_disconnects`  提供针对“悲观”（如预检）和“乐观”（如优雅恢复）技术的后台，而现代SQLAlchemy更倾向于“悲观”方法。

..参见::

      :ref:`pool_disconnects` 

.. _mysql_sync_errors:

"Commands out of sync; you can't run this command now" / "This result object does not return rows. It has been closed automatically"
-----------------------------------------------------------------------------------------------------------------------------------

MySQL驱动程序存在一类故障模式，其中与服务器的连接状态处于无效状态。通常，当再次使用连接时，两个错误消息中的一个错误消息将会发生。原因在于服务器状态已更改为客户端库不期望的状态，因此当客户端库在连接上发出新的语句时，服务器不会如预期地做出响应。

因为 SQLAlchemy 中数据库连接是池化的，所以在连接上的消息不同步的问题变得更加重要，因为如果操作失败，如果连接本身处于不可用状态并且返回池，它将在再次检出时失效。为了缓解此问题，当出现此类故障模式时，连接将无效，从而丢弃到MySQL的底层数据库连接。这种无效化对于许多已知的故障模式是自动发生的，并且也可以通过  :meth:`_engine.Connection.invalidate`  方法显式调用。

此类别的第二类失败模式是一个上下文管理器，例如 ``with session.begin_nested():``，它想在出现错误时向后“回滚”事务；但是，在某些连接的故障模式中，回滚本身（也可以是 RELEASE SAVEPOINT 操作）也会失败，从而导致迷惑的调用堆栈

最初，这个错误的原因很简单，它意味着多线程程序从多个线程调用单个连接上的命令。这适用于最初的“MySQLdb”本机C驱动程序，基本上它是唯一在使用的驱动程序。然而，随着纯Python驱动程序（如PyMySQL和MySQL-connector-Python）以及对gevent/eventlet，multiprocessing的增加（通常与Celery一起使用），存在一系列因素已知可能导致此问题，其中的一些因素已经随着SQLAlchemy版本的变化而得到了改善，但是其他因素是不可避免的 :

* **在线程之间共享连接** - 这是这些类型的错误最初发生的原因。程序同时在两个或多个线程中使用相同的连接，这意味着多组消息在该连接上混合，将服务器端会话放入客户端不再知道如何解释的状态中。但是，在今天，通常可能存在其他原因。

* **在进程之间共享连接的文件句柄** - 这通常发生在程序使用 ``os.fork()`` 生成一个新进程时，并且父进程中存在的TCP连接现在被共享到一个或多个子进程中。因为现在多个进程正向本质上相同的文件句柄发出消息，所以服务器收到交错的消息并且断开连接状态。

  当使用Python的“multiprocessing”模块并使用父进程中创建的   :class:`_engine.Engine`  时，此场景可能会发生。很常见的是，在使用类似Celery的工具时会使用“multiprocessing”。正确的方法应该是当子进程首次启动时，只需要产生一个新的   :class:` _engine.Engine` ，丢弃任何来自父进程的   :class:`_engine.Engine` ；或者从父进程继承的   :class:` _engine.Engine`  可以通过调用  :meth:`_engine.Engine.dispose`  方法来处理其内部的连接池资源。

* **Greenlet Monkeypatching w/ Exits** - 当使用像gevent或eventlet这样的库对Python网络API进行monkeypatches时，像PyMySQL这样的库现在在一种异步操作模式下工作，即使它们未明确针对该模型进行开发。常见的问题是其中一个协程被中断，通常是由于应用程序中的超时逻辑。这会导致 ``GreenletExit`` 异常引发，并且Pure-Python MySQL驱动程序从其工作中间中断，其可能是接收来自服务器的响应或准备重置连接状态。当异常短暂中断了全部工作时，客户端与服务器的对话现在不同步，因此后续连接的使用可能会失败。从1.1.0版本开始，SQLAlchemy知道如何防范这种情况，因为如果数据库操作被所谓的“exit异常”中断(包括GreenletExit和任何其他不是在Exception下且不是在BaseException下的Python子类)，则连接将无效。

* **回滚/SAVEPOINT释放失败** - 一些类别的错误导致连接在事务的上下文中不可用，以及在“SAVEPOINT”块中发生时也不可用。在这些情况下，连接上的故障已经使任何保存点都不再存在，但是当SQLAlchemy或应用程序尝试回滚此保存点时（该保存点也可以是RELEASE SAVEPOINT操作），失败，通常以类似“保存点不存在”的消息失败。在这种情况下，在Python 3下都会输出一系列异常，其中“cause”的最终原因也将被显示。在Python 2下，没有“涉及”异常，但是最近版本的SQLAlchemy将尝试发出警告说明原始故障原因，同时仍然抛出立即的错误，即ROLLBACK的失败。

.. _faq_execute_retry:

如何自动“重试”语句执行？
-----------------------------------

文档   :ref:`pool_disconnects`  阐述了对已自上次检查特定连接以来断开连接的连接进行重试的可用策略。这方面最现代的特性是  :paramref:` _sa.create_engine.pre_ping`  参数，该参数允许在从池中检索连接时对数据库连接执行“ping”，并在当前连接已断开连接时重新连接。

重要的是要注意，此“ping”仅在实际使用连接执行操作之前**发出**。一旦连接交付给用户调用方，根据Python的：term：`DBAPI` 规范，现在该连接处于**autobegin**操作状态，这意味着它将自动在第一次使用时开始新事务，并保持对随后语句的影响，直到调用DBAPI级别的``connection.commit()``或``connection.rollback()``方法。在现代使用SQLAlchemy的情况下，一系列的SQL语句总是在此事务状态内被调用，假设没有启用  :ref:`DBAPI autocommit模式<dbapi_autocommit>` （有关详细信息，请参见下一节），这意味着没有单个语句自动提交；如果操作失败，则当前事务中所有语句的影响将丢失。

这对于“重试”语句的概念具有的含义是，在默认情况下，当连接丢失时，**整个事务会丢失**。由于已经丢失了数据，因此没有有用的方法可以“重新连接和重试”，并继续停留在它离开的地方。因此SQLAlchemy没有一个透明的“重新连接”功能，可以在连接已断开连接而同时处于使用状态的情况下正常工作。在交易开始时和提交时，将“重试”显式地构建到应用程序中仍然是更好的方法，因为应用程序级的交易方法最了解如何重新运行其步骤。

此外，如果我们**没有**使用事务，则可以使用更多选项，如下一节所述。

.. _faq_execute_retry_autocommit:

使用DBAPI Autocommit使透明重新连接成为可用的只读版本
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

随着大多数DBAPI现在提供原生的“autocommit”设置，我们可以利用这些功能为仅 **读，仅autocommit操 作**提供有限形式的透明重新连接。可以应用到DBAPI级别的``cursor.execute()``方法的透明语句重试机制，但仍然不安全应用到 DBAPI 的``cursor.executemany()``方法，因为语句可能已经处理了任何给定的参数部分。

.. warning:: 不应将以下配方用于写入数据的操作。用户应仔细阅读并了解如何使该配方工作，以及针对特定后端在生产环境中仔细测试故障模式。在所有情况下，重试机制不保证在所有情况下阻止断开连接错误。

透明语句重试机制可以通过使用  :meth:`_events.DialectEvents.do_executexecute`  和  :meth:` _events.DialectEvents.do_execute_no_params`  钩子应用于 DBAPI 级 ``cursor.execute()`` 方法来实现。在语句执行期间拦截断开连接。对于具有 DBAPI 级 autocommit 的数据库支持，该配方要求不能保证适用于特定的后端。提供一个名为 "reconnecting_engine()" 的简单函数，将事件钩子应用于给定的   :class:`_engine.Engine`  对象，返回一个始终 执行autocommit的对象，启用 DBAPI 级autocommit操作。单参数和无参数语句执行时一个连接将会透明重新连接:

    import time

    from sqlalchemy import event


    def reconnecting_engine(engine, num_retries, retry_interval):
        def _run_with_retries(fn, context, cursor_obj, statement, *arg, **kw):
            for retry in range(num_retries + 1):
                try:
                    fn(cursor_obj, statement, context=context, *arg)
                except engine.dialect.dbapi.Error as raw_dbapi_err:
                    connection = context.root_connection
                    if engine.dialect.is_disconnect(raw_dbapi_err, connection, cursor_obj):
                        if retry > num_retries:
                            raise
                        engine.logger.error(
                            "disconnection error, retrying operation",
                            exc_info=True,
                        )
                        connection.invalidate()

                        # use SQLAlchemy 2.0 API if available
                        if hasattr(connection, "rollback"):
                            connection.rollback()
                        else:
                            trans = connection.get_transaction()
                            if trans:
                                trans.rollback()

                        time.sleep(retry_interval)
                        context.cursor = cursor_obj = connection.connection.cursor()
                    else:
                        raise
                else:
                    return True

        e = engine.execution_options(isolation_level="AUTOCOMMIT")

        @event.listens_for(e, "do_execute_no_params")
        def do_execute_no_params(cursor_obj, statement, context):
            return _run_with_retries(
                context.dialect.do_execute_no_params, context, cursor_obj, statement
            )

        @event.listens_for(e, "do_execute")
        def do_execute(cursor_obj, statement, parameters, context):
            return _run_with_retries(
                context.dialect.do_execute, context, cursor_obj, statement, parameters
            )

        return e

提供上述的配方后，可以通过以下证明脚本演示重新连接操作。

运行后，它将在每五秒钟内向数据库发出“SELECT 1”语句。

    from sqlalchemy import create_engine
    from sqlalchemy import select

    if __name__ == "__main__":
        engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True)

        def do_a_thing(engine):
            with engine.begin() as conn:
                while True:
                    print("ping: %s" % conn.execute(select([1])).scalar())
                    time.sleep(5)

        e = reconnecting_engine(
            create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True),
            num_retries=5,
            retry_interval=2,
        )

        do_a_thing(e)

重启数据库，以演示透明的重新连接操作:

.. sourcecode:: text

    $ python reconnect_test.py
    ping: 1
    ping: 1
    disconnection error, retrying operation
    Traceback (most recent call last):
      ...
    MySQLdb._exceptions.OperationalError: (2006, 'MySQL server has gone away')
    2020-10-19 16:16:22,624 INFO sqlalchemy.pool.impl.QueuePool Invalidate connection <_mysql.connection open to 'localhost' at 0xf59240>
    ping: 1
    ping: 1
    ...

.. versionadded:: 1.4  上述配方利用了1.4-specific behaviors，因此在以前的SQLAlchemy版本中不起作用。

此配方已经在SQLAlchemy的1.4版本中测试完毕。

为什么SQLAlchemy发出这么多回滚操作？
--------------------------------------------

SQLAlchemy当前假设DBAPI连接处于“非自动提交”模式-这是Python数据库API的默认行为，这意味着必须假设始终存在事务。当连接返回的时候，连接池会发出``connection.rollback()``。为了释放连接上剩余的事务资源。在像 PostgreSQL 或 MSSQL 这样会强制锁定表资源的数据库上，这一点非常重要，因为连接不再使用时表和行不会保持锁定状态。否则，应用程序可能会挂起。在 MySQL 的InnoDB上也是同样的道理，只要在连接已静止的情况下，任何仍处于旧事务中的连接都会返回陈旧的数据。
你可以看一下有关重试的官方文档。
如果我使用多个SQLite数据库连接（通常用于测试事务操作），那么我的测试程序不起作用！
若使用SQLite ":memory:"数据库，则默认的连接池是   :class:`.SingletonThreadPool` ，该线程池每个线程都会维护一个SQLite连接。因此，在同一线程中使用的两个连接实际上是相同的SQLite连接。请确保不使用 :memory: 数据库，这样引擎将使用   :class:` .QueuePool` （在当前的SQLAlchemy版本中，这是非内存数据库的默认值）。

..参见::

      :ref:`pysqlite_threading_pooling`  - PySQLite的行为信息。

.. _faq_dbapi_connection:

使用Engine时如何获取原始的DBAPI连接？
----------------------------------------------

对于常规的 SA 引擎级别 Connection，可以通过   :class:`_engine.Connection`  的  :attr:` _engine.Connection.connection`  属性获取池代理的DBAPI连接版本，并且对于真正的DBAPI连接可以在上面调用  :attr:`.PoolProxiedConnection.dbapi_connection`  属性。在常规同步驱动程序上通常不需要访问非池委托的 DBAPI 连接，因为所有方法都会通过代理执行::

    engine = create_engine(...)
    conn = engine.connect()

    # PEP-249样式 PoolProxiedConnection（历史上称为“连接妖精”）
    connection_fairy = conn.connection

    # 通常，将从该对象获得一个游标
    # ... 与 cursor_obj 协作

    # 要绕过“connection-fairy”，例如在未代理的 PEP-249 DBAPI 连接上设置属性，请使用.dbapi_connection属性，例如：
    raw_dbapi_connection = connection_fairy.dbapi_connection

    # 同样的结果也可以通过使用 .driver_connection 属性获得（有关更多信息，请参见下一节）。
    also_raw_dbapi_connection = connection_fairy.driver_connection

.. versionchanged:: 1.4.24  添加了  :attr:`.PoolProxiedConnection.dbapi_connection`  属性，并且替代了以前的  :attr:` .PoolProxiedConnection.connection`  属性，该属性仍然可用；此属性始终提供同步风格的 pep-249 连接对象。还增加了  :attr:`.PoolProxiedConnection.driver_connection`  属性，它将始终引用实际的驱动程序级别的连接，而不管其呈现什么样的API.

如何在使用asyncio驱动程序时访问底层连接?
--------------------------------------------------

当使用asyncio驱动程序时，该上述方法有两个更改。第一个是当使用   :class:`_asyncio.AsyncConnection`  时，必须使用可等待的方法  :meth:` _asyncio.AsyncConnection.get_raw_connection` 。返回的   :class:`.PoolProxiedConnection`  在这种情况下保留同步样式的 pep-249 使用模式，并且  :attr:` .PoolProxiedConnection.dbapi_connection`  属性引用将 asyncio 连接适配为同步样式 pep-249 API 的 sqlalchemy 适配连接对象，换句话说，在使用 asyncio 驱动程序时有 * 两个*代理层。使用实际的 asyncio  连接可以从  :attr:`.PoolProxiedConnection.driver_connection`  获取。

重申上述示例，在asyncio视角下的运行时如下::

    async def main():
        engine = create_async_engine(...)
        conn = await engine.connect()

        # pep-249样式ConnectionFairy连接池代理对象
        # 提供同步接口
        connection_fairy = await conn.get_raw_connection()

        # 在该协议的下层是第二个代理，其适应 asyncio
        # 驱动程序到一个 PEP-249 连接对象，通过 .dbapi_connection 访问
        # 就像同步API一样。
        sqla_sync_conn = connection_fairy.dbapi_connection

        # 真正的内部驱动程序连接可以通过 .driver_connection 属性访问
        raw_asyncio_connection = connection_fairy.driver_connection

        # 使用原始的 asyncio 连接进行工作
        result = await raw_asyncio_connection.execute(...)

.. versionchanged:: 1.4.24  添加  :attr:`.PoolProxiedConnection.dbapi_connection` 
   和  :attr:`.PoolProxiedConnection.driver_connection`  属性，以允许使用一致接口来访问 pep-249 连接、pep-249 适配层和底层驱动程序连接。

当使用 asyncio 驱动程序时，上述“DBAPI”连接实际上是经过 SQLALchemy 适配的连接形式，它呈现出同步式 pep-249 风格的API。要访问实际的 asyncio 连接，这可以通过 :class：`.PoolProxiedConnection` 的   :attr:`.PoolProxiedConnection.driver_connection`  属性访问。对于标准 pep-249 驱动程序，  :attr:` .PoolProxiedConnection.dbapi_connection`   和  :attr:`.PoolProxiedConnection.driver_connection`  是同义词。

在将连接返回到池之前，请确保将连接上的任何隔离级别设置或其他操作特定设置还原为正常状态。

作为一种替代方法，您可以调用  :meth:`_engine.Connection.detach`  方法来从   :class:` _engine.Connection`  或代理连接上解除关联，而不是还原设置。这将将连接从池中分离，以便在调用  :meth:`_engine.Connection.close`  时关闭并丢弃：

.. sourcecode:: text

    conn = engine.connect()
    conn.detach()  # 将DBAPI连接从连接池中分离
    conn.connection.<go nuts>
    conn.close()  # 真正关闭连接，池会用一个新连接代替它

如何在Python multiprocessing或 os.fork() 中使用连接/引擎/会话？
--------------------------------------------------------

这在   :ref:`pooling_multiprocessing` （池化 多处理）中提到了。