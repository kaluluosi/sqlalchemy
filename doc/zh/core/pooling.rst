.. _pooling_toplevel:

连接池
==========

.. module:: sqlalchemy.pool

连接池是一种标准技术，用于在内存中维护长期运行的连接以便于高效复用，并提供了控制应用程序可能同时使用的连接总数的管理。

特别地，对于服务器端的Web应用程序，连接池是维护在内存中的“池”所必需的，这些池的活动数据库连接被跨请求复用。

SQLAlchemy包括几个连接池实现，它们集成到:class:`_engine.Engine`中。它们也可以直接用于希望将连接池添加到另一个普通DBAPI方法的应用程序中。

连接池配置
-------------------------

在大多数情况下，:func:`~sqlalchemy.create_engine`函数创建的:class:`_engine.Engine`返回的对象中都集成了一个预配置合理的:class:`QueuePool`。如果您只是想了解如何启用连接池，请阅读本节！您已经完成了这一步骤。

最常用的:class:`QueuePool`调整参数可以直接通过关键字参数传递到:func:`~sqlalchemy.create_engine`：``pool_size``，``max_overflow``，``pool_recycle``和 ``pool_timeout``。例如::

    engine = create_engine(
        "postgresql+psycopg2://me@localhost/mydb", pool_size=20, max_overflow=0
    )

所有SQLAlchemy连接池实现都具有共同点，即它们都没有"预创建(connection pre create)"连接——所有实现都等待第一次使用之前不创建连接。在此时，如果没有进行更多的请求以获取更多的连接，则不会创建更多的连接。因此，对于:func:`_sa.create_engine`的行为涉及到是否真正需要五个连续的连接，即使应用程序未使用五个连接并行，使用一组大小为五的连接池完全适用。

.. _pool_switching:

切换池实现
------------------------

使用:func:`_sa.create_engine`使用不同类型的池的常规方法是使用``poolclass``参数。此参数接受从“sqlalchemy.pool”模块导入的类，并处理构建池的细节。这里的一个常见用例是禁用连接池，可以通过使用:class:`.NullPool`实现来实现::

    from sqlalchemy.pool import NullPool

    engine = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test", poolclass=NullPool
    )

使用自定义连接函数
----------------------------------

请参阅 :ref:`custom_dbapi_args` 章节，其中描述了各种自定义连接选项。


构建一个池
-------------------

要单独使用:class:`_pool.Pool`，需要使用“creator”的函数作为第一个参数，后跟任何其他选项::

    from sqlalchemy import pool
    import psycopg2


    def getconn():
        c = psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")
        return c


    mypool = pool.QueuePool(getconn, max_overflow=10, pool_size=5)

可以使用:meth:`_pool.Pool.connect`函数从池中取出DBAPI连接。这个方法的返回值是一个包含在透明代理中的DBAPI连接::

    # 获取连接
    conn = mypool.connect()

    # 使用它
    cursor_obj = conn.cursor()
    cursor_obj.execute("select foo")

透明代理的目的是截取“close()”调用，因此DBAPI连接不会被关闭，而是会被返回到池中::

    #“关闭”连接。返回
    #它到连接池中。
    conn.close()

如果回收方法 failed,这个代理有时不是捕捉到执行的问反而立即立即退出对模块的计数，虽然在cPython大多数情况下,通常会这么做。但不建议使用这种方法，特别是在使用asyncio DBAPI驱动程序时不支持这种方法。

.. _pool_reset_on_return:

checkin时重置
----------------

该池包括“checkin时重置(reset on return)”行为，当连接返回到池中时，将调用DBAPI连接的“rollback()”方法。这是确保在连接中删除任何现有的事务状态(包括未提交的数据以及表和行锁定)。对于大多数DBAPI来说，“rollback()”方法是一种低效的方法，如果DBAPI已经完成了一个事务，那么该方法的执行应该是无操作。

对于此功能不可用的特定情况，例如配置为 :ref:`autocommit <dbapi_autocommit_understanding>` 或使用没有ACID可能性的数据库(例如MySQL)的连接时，可以禁用reset-on-return行为，这通常是出于性能原因。这可以通过使用:paramref:`_pool.Pool.reset_on_return`参数来实现，它也可以从:func:`_sa.create_engine`访问，作为 :paramref:`_sa.create_engine.pool_reset_on_return` 传递 None 值即可。下面的示例说明了如何使用“AUTOCOMMIT”参数设置的 :paramref:`.create_engine.isolation_level` 与此参数实现：

    non_acid_engine = create_engine(
        "mysql://scott:tiger@host/db",
        pool_reset_on_return=None,
        isolation_level="AUTOCOMMIT",
    )

上述引擎在返回连接到池时实际上不会执行回滚；由于已经启用了AUTOCOMMIT，因此驱动程序也不会执行任何BEGIN操作。

自定义reset-on-return方案
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

“checkin时重置”仅包括一个单独的"回滚"可能不足够适用于某些用例；特别是，使用临时表的应用程序可能希望自动删除这些表来完成连接校验。一些内部机制包括重置临时表的特定SQL异常或回收服务器端会话句柄和服务器端语句缓存。同样地，一些(但不是所有)后端可能支持"重置"这种情况下的状态。(注意：PostgreSQL及Microsoft SQL Server是支持重置的两个已读的数据库)。包括在Python语言中重置状态这样的操作由于线程管理的不确定行可能需要在释放到池中之前被实际地执行多次。

下面的代码示例说明了如何使用:meth:`.PoolEvents.reset`事件钩子将重置on-return重置为Microsoft SQL Server的“sp_reset_connection”存储过程。

from sqlalchemy import create_engine
from sqlalchemy import event

mssql_engine = create_engine(
    "mssql+pyodbc://scott:tiger^5HHH@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    # disable default reset-on-return scheme
    pool_reset_on_return=None,
)


@event.listens_for(mssql_engine, "reset")
def _reset_mssql(dbapi_connection, connection_record, reset_state):
    if not reset_state.terminate_only:
        dbapi_connection.execute("{call sys.sp_reset_connection}")

    # so that the DBAPI itself knows that the connection has been
    # reset
    dbapi_connection.rollback()

.. versionchanged:: 2.0.0b3  Added additional state arguments to
   the :meth:`.PoolEvents.reset` event and additionally ensured the event
   is invoked for all "reset" occurrences, so that it's appropriate
   as a place for custom "reset" handlers.   Previous schemes which
   use the :meth:`.PoolEvents.checkin` handler remain usable as well.


记录重置事件
-----------------------

池事件的日志记录(包括返回时的重置)可以设置日志记录器的``sqlalchemy.pool``和``logging.DEBUG``记录功能或正在使用 :func:`_sa.create_engine`时将 :paramref:`_sa.create_engine.echo_pool` 设置为 ``"debug"``中。如下所示：

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("postgresql://scott:tiger@localhost/test", echo_pool="debug")

上面的池将显示详细的记录信息，包括返回时的重置：

    >>> c1 = engine.connect()
    DEBUG sqlalchemy.pool.impl.QueuePool Created new connection <connection object ...>
    DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> checked out from pool
    >>> c1.close()
    DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> being returned to pool
    DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> rollback-on-return


池事件
---------------

连接池支持事件接口，允许在第一次连接时、每个新连接时以及在连接的checkin和 checkout时执行钩子。有关详细信息，请参见:class:`_events.PoolEvents`。

.. _pool_disconnects:

处理断开连接
------------------------

连接池具有刷新单个连接及其各自的全部连接集的能力，将已经池化的连接设为"无效"。常见的用例是在数据库服务器已经重新启动的情况下让连接池优雅地恢复，这样所有之前建立的连接都不再是可用的。在这方面有两种方法：

.. _pool_disconnects_pessimistic:

断开连接处理-悲观
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

悲观的方法是在每次连接池检查时发出一个测试语句，以测试数据库连接是否仍然有效。该实现是特定于方言的，会使用特定于DBAPI的ping方法，或使用一个简单的SQL语句（例如，"SELECT 1"）来测试连接是否实时。该方法在连接检查时添加了一点开销，但是是消除由于旧的池化连接而引起的所有数据库错误的最简单、最可靠的方法。调用程序不必考虑如何组织操作，使其能够恢复显然失效的连接。

使用 :paramref:`_pool.Pool.pre_ping` 参数可以实现对连接进行pessimistic测试。它可以通过:func:`_sa.create_engine`设置为 :paramref:`_sa.create_engine.pool_pre_ping` 的参数来获得：

    engine = create_engine("mysql+pymysql://user:pw@host/db", pool_pre_ping=True)

"pre ping" 功能在各个方言之间都是逐一有效操作，无论是调用DBAPI专用的"ping"方法还是，如果适当的话，将SQL发送到数据库服务(例如"SELECT 1")。如果发现"disconnect"，则说明无法使用连接，连接将被立即回收，而所有连接池中的连接都将被其时间检查所代替，并在下次checkout时被替换。

如果预检失败,则该初始连接将失败，并应按常规传播连接失败的错误。在数据库可用但无法响应"ping"的不常见情况下，"pre_ping"将尝试最多三次，然后放弃传播上一次收到的数据库错误。

请注意，"pre-ping"方法仅适用于:class:`.QueuePool`使用的连接池。

.. _pool_disconnects_pessimistic_custom:

自定义/pessimistic ping 需要注意
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在添加 :paramref:`_sa.create_engine.pool_pre_ping`之前，"pre-ping"方法是使用: meth:`_events.ConnectionEvents.engine_connect引擎连接`手动执行的。在下面的示例中，提供了一个最常见的配方，以供参考，如果应用程序已经使用这种配方或者需要特殊行为，则可用：

    from sqlalchemy import exc
    from sqlalchemy import event
    from sqlalchemy import select

    some_engine = create_engine(...)


    @event.listens_for(some_engine, "engine_connect")
    def ping_connection(connection, branch):
        if branch:
            # 此参数在SQLAlchemy 2.0中总是为False，
            # 但安装了事件挂钩。在以前的 1.x 版本中
            # 用于“断口”连接应被跳过。
            return

        try:
            # 运行SELECT 1。使用核心select()，以便于
            # 在没有表的情况下选择标量值时，能匹配后端
            # 正确的格式。
            connection.scalar(select(1))
        except exc.DBAPIError as err:
            # .connection_invalidated属性描述了此连接是“disconnect”情况，基于使用方言的原始错误异常进行检查
            if err.connection_invalidated:
                # 再次运行SELECT作为连接将被重新预验证，建立新连接
                # 连接这里的断开检测也会导致整个连接池被无效化，以便丢弃所有旧的，
                # 可能过期的连接。
                connection.scalar(select(1))
            else:
                raise

上述配方的优点是，我们使用了SQLAlchemy察觉那些已知意味着“断开”情况的DBAPI异常和:class:`_engine.Engine`对象的能力。当发生此情况时，:class:`_engine.Connection`将检测到"断开(disconnect)"情况并且重用连接而且使剩余连接池失效，这种情况下连的操作将丢失，重新进行这一操作，或者再次尝试整个事务。如果Engine是使用了DBAPI级别的自动连接配置，如:ref: `dbapi_autocommit`所述，则连接会在事件中间"透明"重新连接。请参阅:ref:`faq_execute_retry`示例。

对于包括几个后端可能有需要使用 :meth:`_events.DialectEvents.handle_error` 钩子定制"断开"异常代码。

.. _pool_disconnects_pessimistic_custom:

支持断开连接情况下新数据库错误码
------------------------------------

SQLAlchemy各方言都包括称为``is_disconnect()``的特定程序，该方法在遇到DBAPI异常时被调用。将DBAPI异常对象传递给此方法，其中方言特定的启发式将确定错误代码是否指示数据库连接已"断开"，或者是否处于另外一种不可用状态，这些状态表明应该重用连接。在此处应用的启发式应使用:meth:`_events.DialectEvents.handle_error` 事件钩子进行自定义，通常通过拥有的 :class:`_engine.Engine` 对象。使用此挂钩，提供的所有错误都能实现向下传递一个称为 :class:`.ExceptionContext` 的上下文对象，它会传递象征具体错误信息的 context.original_exception。自定义事件挂钩可以控制特定的错误是否应视为“disconnect”情况，以及此断开是否应导致整个连接池失效。

例如，在Oracle错误代码中添加支持“DPY-1001”和“DPY-4011”的代码：

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

上述错误处理函数将被用于所有Oracle错误，包括那些使用:ref:`池断开检查<pool_disconnects_pessimistic>`特性的后端使用断开错误处理程序的那些后端（2.0 新特性）。

请参阅:

    :meth:`_events.DialectEvents.handle_error`

.. _pool_disconnects_optimistic:

断开连接处理 - 乐观
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当不使用悲观处理时，以及当数据库在连接周期内的使用期间关闭和/或重新启动时，
另一种处理失效(connection stale /closed)的方法是让SQLAlchemy检测到断开连接事件，并在发生
此类事件时使所有连接池失效，这意味着它们被认为是stale、将在下次checkout时被刷新。 无需特殊处理即可使池失效，这将导致在检测到断开连接事件时继续进行。然而会有一个或多个Python解释器状态协商的TCP连接表示文件描述符，因此这将导致两个或多个独立的Python解释器状态并发访问文件描述符代表的socket连接，从而导致消息发送中断(后者通常是最常见的情况)。

SQLAlchemy :class:`_engine.Engine`对象引用现有数据库连接的连接池。当将此对象复制到子进程中时，
目标是确保不会转移任何数据库连接。这里有三种一般方法：

1.使用:class:`.NullPool`禁用连接池。这是最简单、最单一的系统，可以禁止 :class:`_engine.Engine` 多次使用任何连接。

2.在子进程的初始化阶段中，使用`某个引擎的:meth:`_engine.Engine.dispose`，并将参数传递给:paramref:`.Engine.dispose.close`的值为False。这样，新进程将不会触摸到父进程的任何连接，而是将直接用新连接池开始。

3.在子进程创建之前直接`调用:. Engine.dispose`。这也将使子进程使用新的连接池，同时确保父程序的连接不会转移到子程序中。

下面是每个方法的示例：

1.禁用连接池

    from sqlalchemy.pool import NullPool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname", poolclass=NullPool)

2.在使用:meth:`multiprocessing.Pool`时，不允许孵化出新进程使用旧的连接。

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname")


    def initializer():
        """ensure the parent proc's database connections are not touched
        in the new connection pool"""
        engine.dispose(close=False)


    with Pool(10, initializer=initializer) as p:
        p.map(run_in_process, data)


3.直接在子程序

    engine = create_engine("mysql://user:pass@host/dbname")


    # enable multiprocessing based on this recipe:
    # http://stackoverflow.com/a/21659588/328565
    def run_in_process():
        with engine.connect() as conn:
            conn.execute(text("..."))


    # before process starts, ensure engine.dispose() is called
    engine.dispose()
    p = Process(target=run_in_process)
    p.start()


.. _pool_setting_recycle:

设置池重建
---------------


增加设置 :paramref:`.QueuePool.pool_recycle` ,以避免一些默认关闭连接的后端中的连接问题(例如MySQL)，这会在连接到达一定年龄后防止连接的过多被使用。

举个例子::

    from sqlalchemy import create_engine

    e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", pool_recycle=3600)

上面的代码中的任何超过一小时的DBAPI连接都将被设置为无效并更换。请注意，使连接池中的连接失效只适用于 :class:`_pool.Pool` 本身，而与是否使用:class:`_engine.Engine` 无关。

.. _pool_connection_invalidation:

更多禁用
---------------

:class:`_pool.Pool` 提供了“连接禁用”服务，允许明确禁用连接以及响应其可用性为不可用的条件而自动禁用连接。

"使发"的意思是将特定DBAPI连接从池中移除并丢弃它。如果这一方法失败，将记录此异常，但操作仍将继续进行。

如下情况将完成DBAPI连接的“使发”：

* 检测到DBAPI异常，例如:ref:  `OperationalError`，当调用像"connection.execute()"这样的方法时，该异常被检测出表示叫做"disconnect"状态。因为Python DBAPI没有确定异常所具有的标准系统，所以SQLAlchemy各个方言都包括一个称为"``is_disconnect()``"的系统，此方法将检查异常对象的内容，包括字符串消息及其所带的所有可能的错误代码，以确定是否指示该异常表明连接不再可用。如果这是情况，将调用:meth:`._ConnectionFairy.invalidate`方法，DBAPI连接将被丢弃。

*当连接返回到池中，并调用“汇总(Pool reset on return)”行为确定需要is直接禁用连接时，调用"connection.commit()"或"connection.rollback()"方法时会抛出异常。将尝试最后一次调用"``.close()``"方法来关闭连接，然后它将被丢弃。

* 当监听器实施:meth:`_events.PoolEvents.checkout` 并且抛出:class:`~sqlalchemy.exc.DisconnectionError`异常时，表明连接将无法使用，需要进行一次新的连接尝试。

所有无效化的连接都将调用:meth:`_events.PoolEvents.invalidate`事件。

.. _pool_new_disconnect_codes:

支持新数据库错误码
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQLAlchemy各个方言都包括称为"``is_disconnect()``"的特定程序，该方法在遇到此类异常时被调用。将DBAPI异常对象传递给该方法，各方言特定的启发将然后确定错误代码是否指示了数据库连接“断开”，或者是处于另外一种不可用状态，表明它应该回收。应用到此处的启发应使用:meth:`_events.DialectEvents.handle_error`事件钩子进行自定义。使用此钩子，传递的所有错误都将传递一个称为:class:`.ExceptionContext`的上下文对象，该对象传递了在上下文中携带的具体错误信息。自定义事件挂钩可以控制特定错误是否应视为“disconnect”情况以及是否应该让整个连接池失效。

例如，如果要添加支持将Oracle错误代码“DPY-1001”和“DPY-4011”视为处理断开连接，则应在创建引擎后应用事件处理程序：

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

上述错误处理函数将用于所有Oracle错误，包括在使用 :ref:`与disconnect错误处理相关的池断开<pool_disconnects_pessimistic>` 特性的后端使用 disconnect 错误处理程序的那些后端(2.0中的新特性)。

请参阅:

    :meth:`_events.DialectEvents.handle_error`


.. _pool_use_lifo:

使用先进后出和先进先出
-------------------

:class:`.QueuePool`类特征有一个叫做:paramref:`.QueuePool.use_lifo`的标志，该标志也可从:func:`_sa.create_engine`中使用参数 :paramref:`_sa.create_engine.pool_use_lifo` 访问。将此标志设置为 ``True``将修改池的“队列”行为，使其变成“堆栈”，即上次返回池中的连接会被首先在下一次请求中使用。与池的长期行为不同，即先进先出，可以产生一系列的循环效应，以便对池中的每个连接进行等级排序，此次效果被开启时允许多余的连接保持空闲，以允许服务器端超时方案关闭这些连接。FIFO和LIFO之间的区别基本上是连接是否能在空闲期间保持完整的开放状态：

    engine = create_engine("postgreql://", pool_use_lifo=True, pool_pre_ping=True)

上面的代码中，我们使用了 :paramref:`_sa.create_engine.pool_pre_ping` 标志，以便保证了服务器端关闭连接时的连接池能够进行优雅处理，并通过新的连接进行替换。

请注意，此标志仅适用于使用:class:`.QueuePool`的情况。

.. versionadded:: 1.3

.. seealso::

    :ref:`pool_disconnects`


.. _pooling_multiprocessing:

在多进程或者 os.fork() 中使用连接池
--------------------------------------------------------

当使用连接池时，特别是使用:func:`_sa.create_engine`创建的 :class:`_engine.Engine`时，关键的一点是**连接池不能共享给派生进程**，因为TCP连接被表示为文件描述符，通常跨进程边界不工作，这意味着它将导致两个或多个完全独立Python解释器状态并发访问表示的socket连接文件描述符。

具体取决于驱动程序和操作系统，出现的问题范围从不起作用的连接到由多个进程(并发)使用的套接字连接引起的损坏消息(后者通常是最常见的情况)。

SQLAlchemy :class:`_engine.Engine`对象引用现有数据库连接的连接池。因此，当这个对象复制到一个子进程中时，目标是确保不会传递任何数据库连接。有三种方法：

1.使用:class:`.NullPool`禁用连接池。这是最简单、最单一的系统，可以禁止:class:`_engine.Engine`多次使用任何连接。

2.在独立进程的初始化阶段内，调用:class:`_engine.Engine`的:meth:`_engine.Engine.dispose`方法，将参数:paramref:`.Engine.dispose.close`传递一个值为``False``，以确保派生进程不会接触到父进程的任何连接，而是开始新的。

3.在直接之前：meth:`.Engine.dispose`派生进程将从新连接池开始，同时确保父进程的连接不会转移到子进程中：

举个例子：

1. 禁用连接池

    from sqlalchemy.pool import NullPool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname", poolclass=NullPool)

2. 在:meth:`multiprocessing.Pool`中，使用:meth:`_engine.Engine.dispose`，在初始化具体化时调用它，并且在:paramref:`.Engine.dispose.close`参数传递False的情况下，替换掉子进程中的连接池。这就是推荐的方式：

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

   .. versionadded:: 1.4.33  Added the :paramref:`.Engine.dispose.close`
      parameter to allow the replacement of a connection pool in a child
      process without interfering with the connections used by the parent
      process.

3.在派生进程之前直接:meth:`.Engine.dispose`。这将使子进程从新的连接池开始，同时确保父程序的连接不会转移到子程序中：

        engine = create_engine("mysql://user:pass@host/dbname")


        def run_in_process():
            with engine.connect() as conn:
                conn.execute(text("..."))


        # before process starts, ensure engine.dispose() is called
        engine.dispose()
        p = Process(target=run_in_process)
        p.start()4. 可以将事件处理程序应用于连接池，并测试是否跨进程共享连接，并使它们失效：

   在上述代码中，我们使用了与 :ref:`pool_disconnects_pessimistic` 中描述的类似方法来处理在不同父进程中生成的 DBAPI 连接，将连接记录强制更改为“无效”连接，以强制连接池循环利用连接记录以创建一个新连接。

   上述策略将适应 :class:`_engine.Engine` 共享在不同进程之间的情况。但这些步骤仅适用于在进程边界共享 :class:`_engine.Connection` 的情况，最好将特定的 :class:`_engine.Connection` 作用域限制在单个进程（和线程）内。还不支持直接在进程边界共享任何类型的持续事务状态，如已开启事务并引用了活动的 :class:`_orm.Connection` 实例的 ORM :class:`_orm.Session` 对象；同样最好在新进程中创建新的 :class:`_orm.Session` 对象。

直接使用池实例
--------------------------

可以直接使用池实现而无需引入一个引擎，这可用于只想使用池行为而不想使用 SQLAlchemy 中的所有其他功能的应用程序。
在下面的示例中，使用 :func:`_sa.create_pool_from_url` 获取了 ``MySQLdb`` 方言的默认池：

   如果没有指定要创建的池的类型，则将使用该方言的默认池。可以直接使用 ``poolclass`` 参数来指定它，例如以下示例：

API 文档 - 可用的池实现
--------------------

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