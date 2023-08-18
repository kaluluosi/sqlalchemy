连接/引擎
===========

.. 目录::
    :local:
    :class: faq
    :backlinks: none


如何配置日志记录?
-------------------

请参见 :ref:`dbengine_logging`。

如何对数据库连接进行池化?我的连接是否已经池化了?
---------------------------------------------------------------

大多数情况下，SQLAlchemy自动执行应用程序级的连接池。对于 所有 包括的方言（除了SQLite，使用 “memory” 数据库时），
:class:`_engine.Engine`对象都会将: class:`.QueuePool`作为其连接性来源。

更多详细信息，请参见 :ref:`engines_toplevel` and :ref:`pooling_toplevel`。

如何将自定义连接参数传递给我的数据库API?
--------------------------------------------------------

:func:`_sa.create_engine` 调用直接接受其他参数作为关键字参数 "connect_args"::

    e = create_engine(
        "mysql+mysqldb://scott:tiger@localhost/test", connect_args={"encoding": "utf8"}
    )

对于基本字符串和整数参数，它们通常可以在URL的查询字符串中指定::

    e = create_engine("mysql+mysqldb://scott:tiger@localhost/test?encoding=utf8")

.. seealso::

    :ref:`custom_dbapi_args`

“MySQL服务器已经停止工作”
-------------------------------------

此错误的主要原因是MySQL连接已超时，并已由服务器关闭。 MySQL服务器关闭已闲置一定时间的连接，该默认时间为8小时。
为了适应此问题，可立即启用：paramref:`_sa.create_engine.pool_recycle`设置，这将确保旧于设置的秒数的连接将在下一次检出时被丢弃，
并且将由新连接替换。

对于更一般情况的应对，如数据库重启和其他暂时由于网络问题而导致的连接丢失，则可以响应更广泛的断开检测技术重新使用池 中的连接。
该部分提供了关于“悲观”（例如预冲）和“乐观”（例如优雅恢复）技术的背景。现代SQLAlchemy倾向于采用“悲观”方法。

.. seealso::

    :ref:`pool_disconnects`

.. _mysql_sync_errors:

“命令不同步；您现在无法运行此命令”/“此结果对象不返回行。它已自动关闭”
-------------------------------------------------------------------------------------------------------------------------------------

MySQL驱动程序有一类比较广泛的故障模式，其中与服务器的连接状态处于无效状态。通常，当再次使用连接时，这些两个错误消息之一会发生。
原因是，因为服务器的状态已更改为客户端库未预期的状态，因此，当客户端库在连接上发出新的语句时，服务器不会按预期做出响应。

在SQLAlchemy中，由于数据库连接被池化，连接上的消息不同步问题变得更为重要，因为当操作失败时，如果连接本身处于不可用状态，
则如果它重新进入连接池，则在再次检出时将会出现问题。缓解这个问题的方法是 在这样的故障模式发生时 **将连接无效化**，
以便MySQL的基础数据库连接被丢弃。对于许多已知的故障模式，此无效化会自动发生，并且也可通过 :meth:`_engine.Connection.invalidate`方法显式调用。

此类别中还有第二类故障模式，其中上下文管理器（例如 `with session.begin_nested():`）在出现错误时希望“回滚”事务；
但是，在某些情况下，连接的故障模式会导致回滚本身（也可以是“RELEASE SAVEPOINT”操作）也失败，导致有误导性的堆栈跟踪。

最初，这个错误的原因非常简单，这意味着多线程程序从多个线程中调用单个连接上的命令。这适用于最初几乎是唯一使用的原始 
“MySQLdb”native-C驱动程序。然而，随着纯Python驱动程序（如PyMySQL和MySQL-connector-Python）的出现以及使用gevent/eventlet等工具（经常与Celery一起使用）的增多，
已知会导致此问题的一系列因素，其中一些因素 由于SQLAlchemy版本的更新而得到改进，而其他因素则无法避免:

* **在线程之间共享连接**——这是这些故障的最初原因。一个程序在同一时间在两个或多个线程中使用了同一个连接，这意味着多重消息已经被混合在了连接上，
  将服务器端会话置于客户端不再知道如何解释的状态中。然而，现在通常更有可能发生其他原因。

* **将连接文件句柄在进程之间共享**——这通常发生在程序使用``os.fork()``生成新进程时，并且父进程中存在的TCP连接会分享到一个或多个子进程中。
  现在多个进程正在以实际上相同的文件句柄发送消息，因此服务器接收交错的消息并打破了连接的状态。

  如果程序使用Python的“multiprocessing”模块并使用在父进程中创建的 :class:`_engine.Engine`，则此情况可能非常容易发生。
  当使用像 Celery 这样的工具时，通常会使用“multiprocessing”。 正确的方法应该是，当子进程首次启动时，产生一个新的:class:`_engine.Engine`，
  丢弃从父进程中获取的任何:class:`_engine.Engine` 或者，可以通过调用 :meth:`_engine.Engine.dispose`方法来释放从父进程传递下来的 :class:`_engine.Engine` 的内部池连接。

* **使用Exits的Greenlet Monkeypatch** - 当使用像 gevent 或 eventlet 这样的库 monkeypatch Python 网络 API 时，例如 PyMySQL 库现在处于异步操作模式，
  即使它们未明确针对此模型进行开发。常见问题是 greenthread 被中断，通常是由于应用程序中的超时逻辑而引起。这导致 "GreenletExit" 异常被触发，
  纯Python MySQL驱动程序被中断，可能是因为它正在接收来自服务器的响应或准备以其他方式重置连接的状态。当异常截断所有这些工作时，
  客户端和服务器之间的对话现在不同步，并且连接的后续使用可能会失败。 
  SQLAlchemy 从版本1.1.0开始知道如何针对此进行保护，因此，如果数据库操作因所谓的“退出异常”而中断，即包括GreenletExit以及非同时是BaseException子类的Python Exception 子类，
  则使连接无效 。

* **回滚 / SAVEPOINT发布失败**- 一些类别的错误会导致连接在事务上下文中变得无法使用，以及在“SAVEPOINT”块中进行操作的情况。
  在这些情况下，连接上的故障使得任何 SAVEPOINT 都不再存在，但是当SQLAlchemy或应用程序试图“回滚”此 savepoint时（也可以是“RELEASE SAVEPOINT”操作)，
  回滚本身就失败了，通常会出现类似“savepoint does not exist”的消息。 在此情况下，在Python 3下将输出一系列异常链，其中“原因”的错误最终也会被显示出来。
  在Python 2下，没有“链接”的异常，但是近期的SQLAlchemy版本会尝试发出警告，其中说明原始失败原因，同时仍然抛出即时错误，即ROLLBACK的失败。

.. _faq_execute_retry:

如何自动执行“重试”语句执行?
----------------------------------------

文档 :ref:`pool_disconnects` 阐述了用于连接从上次检查特定连接以来已经断开连接的池的策略。在这方面最现代的功能是 
:paramref:`_sa.create_engine.pre_ping`参数，该参数允许在检索自池中时在数据库连接上发出“ping”，以便在当前连接已断开时重新连接。

重要的是要注意，此“ping”仅在实际使用连接执行操作之前进行。一旦将连接传递给调用器，根据Python :term:`DBAPI`规范，现在在使用操作之前
将自动进行autobegin操作，这意味着当第一次使用它时，它将自动 BEGIN 一个新事务，该事务对随后的语句保持有效，并在 DBAPI 
级别的 ``connection.commit()`` 或 ``connection.rollback()`` 方法被调用时保持在有效状态。

在使用现代SQLAlchemy时，一系列SQL语句总是在此事务状态内调用，
假设并未启用:ref:`DBAPI autocommit mode <dbapi_autocommit>`（在下一节中会更详细地介绍），这意味着不会自动提交任何单个语句；
如果操作失败，则会丢失当前事务中所有语句的作用。

这对于“重试”语句的概念具有的含义是，在默认情况下，当连接丢失时，“整个事务会丢失”。没有有用的方式，可以确保数据库“重新连接并重试”，
并继续上次停止的位置，因为数据已丢失。因此，SQLAlchemy 没有透明的“重新连接”特性，这在事务进行时，当数据库连接在使用时断开连接时非常重要。
由于一旦事务结束，数据库在新事务中的状态可能完全不同，因此在这种情况下，没有固定的方法来保证这些DML语句将使用相同的状态。
在处理中断操作的中间“重新连接”应该是 **"重试"整个操作的最佳方法**，通常通过使用自定义的Python装饰器，该装饰器将重试多次才能成功，
或者以某种方式建立应用程序，使其针对失败的事务具有弹性。

还有关于扩展的概念，这些扩展可以跟踪在事务内进行的所有语句，然后在一个新事务内重播它们以近似“重试”操作。
SQLAlchemy的 :ref:`事件系统<core_event_toplevel>` 允许构建这样的系统，但是这种方法通常不是很有用，因为没有办法保证这些DML语句将工作在相同的状态下，
因为一旦事务结束，数据库在新事务中的状态可能完全不同。在事务开始和提交事务的时间点将“重试”明确地构建到应用程序中仍然是更好的方法，
因为应用程序级事务方法最了解如何在不同阶段重复运行它们的步骤。

否则，如果SQLAlchemy提供了一种透明且悄无声息地“重新连接”在事务中工作的功能，那么效果将是数据被静默丢失。通过试图隐藏问题，
SQLAlchemy会使情况变得更糟。

但是，如果我们 **没有使用事务**，则有更多的选择，下一节将介绍这一点。

.. _faq_execute_retry_autocommit:

使用DBAPI自动提交允许对透明重新连接的只读版本
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

随着DBAPI现在提供原生的“自动提交”功能，我们可以利用这些功能提供有限的对于**按只读、自动提交操作试图重新连接的透明度** 。
“透明语句重试”可能适用于 DBAPI 级别 的 ``cursor.execute()``
方法，但是对于DBAPI 的 ``cursor.executemany()`` 方法不安全，因为该语句可能消耗给定参数的任何部分。

.. warning:: 以下食谱 **不应** 用于写数据的操作。 用户应该仔细阅读和了解食谱的工作方式，
   并在针对特定后端的情况下非常谨慎地针对故障模式进行测试才能在生产中使用该食谱。 
   在某些情况下，重试机制不能保证防止所有断开连接的错误。

可以通过使用 :meth:`_events.DialectEvents.do_execute` 和 :meth:`_events.DialectEvents.do_execute_no_params` 钩子将简单的重试机制应用于
DBAPI级别的 ``cursor.execute()``方法。这些钩子将能够拦截语句执行期间断开的情况。
当使用一个 :class:`_engine.Engine` 时，表示一个总是自动提交的引擎版本，可以将这些事件钩子应用于该版本以实现透明语句重试，
该版本使得 DBAPI 水平的自动提交可以使用。 对于某些后端，此操作是 **不保证** 的，可以演示使用单个函数“reconnecting_engine()”，
该函数应用于给定的 :class:`_engine.Engine` 对象产生一个总是自动提交的版本，并返回一个小参数不管是单参或无参语句执行的连接将透明的重新连接：

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

给定上述食谱，可以通过以下POC脚本演示事务过程中的透明重新连接。运行后，它将每5秒向数据库发出“SELECT 1”语句：

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

重启数据库同时保证透明重新连接操作：

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

.. versionadded:: 1.4  该食谱利用了1.4特定行为并且在之前的SQLAlchemy版本中不起作用。

上述食谱对于SQLAlchemy 1.4进行了测试。

为什么SQLAlchemy会发出很多ROLLBACKs命令？
----------------------------------------------------

SQLAlchemy目前假定DBAPI连接处于“非自动提交”模式-这是Python数据库API的默认行为，这意味着必须假定始终存在事务。
当连接被返回时，连接池会发出 ``connection.rollback()``，以便释放连接上仍然存在的任何事务资源。在像 PostgreSQL 或 MSSQL 这样锁定表的
数据库上，这很关键，以便行和表不会保留在不再使用的连接中。否则，应用程序可能会挂起。然而，并不仅是针对锁，对于任何具有任何事务隔离的数据库，
包括使用 InnoDB 的 MySQL，这也同样重要。在 MySQL 上即使使用传统的 InnoDB 来说，任何仍处于旧事务中的连接将返回旧数据。
有关即使在MySQL上，也可能看到旧数据的背景信息。请参见 https://dev.mysql.com/doc/refman/5.1/en/innodb-transaction-model.html

我正在使用MyISAM，该怎么关闭它？
------------------------------------

连接池返回行为的行为可以使用 ``reset_on_return`` 配置::

    from sqlalchemy import create_engine
    from sqlalchemy.pool import QueuePool

    engine = create_engine(
        "mysql+mysqldb://scott:tiger@localhost/myisam_database",
        pool=QueuePool(reset_on_return=False),
    )

我正在使用SQL Server - 如何将这些ROLLBACK转换为COMMIT？
---------------------------------------------------------------

“reset_on_return”还可以接受 “commit”、“rollback”中的值，除了 “True”、“False” 和 “None”。将其设置为 “commit” 将导致每次返回到池中的连接都会提交 ：

    engine = create_engine(
        "mssql+pyodbc://scott:tiger@mydsn", pool=QueuePool(reset_on_return="commit")
    )

我正在使用SQLite数据库的多个连接（通常是用于测试事务操作），但我的测试程序无效！
-------------------------------------------------------------------------------

如果使用SQLite :memory:数据库，则默认的连接池为 :class:`.SingletonThreadPool`，
它为每个线程维护正好一个SQLite连接。因此，在同一线程中使用的两个连接实际上是相同的SQLite连接。
确保您不使用 :memory: 数据库，以便引擎将使用 :class:`.QueuePool` （当前SQLAlchemy版本中非内存数据库的默认设置）。

.. seealso::

    :ref:`pysqlite_threading_pooling` - PySQLite的行为信息。

.. _faq_dbapi_connection:

在使用Engine时如何获得原始DBAPI连接?
-------------------------------------------------------

对于常规的SA引擎级连接，您可以通过 :class:`_engine.Connection` 的    :attr:`_engine.Connection.connection` 属性获得池代理的DBAPI连接版本，
对于真正的DBAPI连接，可以通过调用 :attr:`.PoolProxiedConnection.dbapi_connection` 属性来获得。在常规的同步驱动程序上，通常无需访问
非连接池代理的DBAPI连接，因为所有方法都是通过代理的：

    engine = create_engine(...)
    conn = engine.connect()

    connection_fairy = conn.connection  # pep-249样式的 PoolProxiedConnection（历史上被称为“connection fairy”）
    cursor_obj = connection_fairy.cursor()  # 通常要从此对象获取游标()
    raw_dbapi_connection = connection_fairy.dbapi_connection # 此对象上访问.dbapi_connection以绕开“connection_fairy”，例如在未代理的DBAPI连接上设置属性
    also_raw_dbapi_connection = connection_fairy.driver_connection # 相同的对象也可以通过.driver_connection访问

.. versionchanged:: 1.4.24 添加了：attr:`.PoolProxiedConnection.dbapi_connection`属性，
   这个属性取代了之前的attr:`.PoolProxiedConnection.connection`属性,它仍然可用;
   这个属性始终提供同步的pep-249连接对象。添加了：attr:`.PoolProxiedConnection.driver_connection`属性，
   它将始终引用真实的驱动级别连接，而与其API所表示的相同。

当使用asyncio驱动程序时，上述方案有两个更改。首先，使用 :class:`_asyncio.AsyncConnection` 时，必须使用可以等待的方法 
: meth:`_asyncio.AsyncConnection.get_raw_connection` 才能管理 :class:`.PoolProxiedConnection`。在这种情况下，返回
的 :class:`.PoolProxiedConnection` 保留了一种同步样式的 pep-249 使用模式，而 :attr:`.PoolProxiedConnection.dbapi_connection` 属性引用了
适应器，该适配器将asyncio连接适配为同步样式pep-249 API，换句话说：
当使用asyncio驱动程序时，使用中的“DBAPI”连接实际上是一种将asyncio连接适配为同步样式pep-249 API的SQLAlchemy-adapted连接。
:class:`.PoolProxiedConnection.driver_connection`属性始终引用实际的asyncio级的连接，而这将显示与primordial API匹配的原始asyncio API。
以下是用asyncio重新表述以前的示例的方式：

当使用asyncio驱动程序时，上述“DBAPI”连接实际上是一种将asyncio驱动程序转换成同步样式pep-249 API的SQLAlchemy-adapted连接。
可以通过 :class:`~.PoolProxiedConnection` 的 :attr:`.PoolProxiedConnection.driver_connection` 属性访问实际的 asyncio 驱动程序连接

.. versionchanged:: 1.4.24 添加了：attr:`.PoolProxiedConnection.dbapi_connection`和 :attr:`.PoolProxiedConnection.driver_connection` 属性，
   以允许使用一致的接口访问pep-249连接，pep-249适配器和底层的驱动程序连接。

如何在Python multiprocessing或os.fork()中使用引擎/连接/会话?
-----------------------------------------------------------------------------------------

这是在 :ref:`pooling_multiprocessing` 部分中讨论的。

