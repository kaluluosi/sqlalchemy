======================================
事务和连接管理
======================================

.. _unitofwork_transaction:

管理事务
=====================

.. versionchanged:: 1.4 会话事务管理已经更新以更清晰和更易于使用。 特别是现在拥有“自动开始”操作，这意味着可以控制事务开始的时间，而无需使用传统的“自动提交”模式。

:class:`_orm.Session` 在一个时间内跟踪单个“虚拟”事务的状态，使用一个称为 :class:`_orm.SessionTransaction` 的对象。 然后，此对象利用了给定 :class:`_orm.Session` 绑定的底层 :class:`_engine.Engine` 或者 Engines，以使用需要的 :class:`_engine.Connection` 对象启动真实的连接级别事务。

这个“虚拟”事务会在需要时自动创建，或者可以使用 :meth:`_orm.Session.begin` 方法启动。 尽可能地，Python上下文管理器的使用在创建 :class:`_orm.Session` 对象的级别以及维护: class:`_orm.SessionTransaction` 的范围级别上都得到了支持。

在以下示例中，假设我们从一个 :class:`_orm.Session` 开始：

    from sqlalchemy.orm import Session

    session = Session(engine)

现在，我们可以使用上下文管理器在标有事务的情况下运行操作：

    with session.begin():
        session.add(some_object())
        session.add(some_other_object())
    # 在结束时提交事务，或者如果发生异常回滚

在上下文结束时，假设未引发任何异常，任何未决的对象将被刷新到数据库中，并且数据库事务将被提交。 如果上面的块中引发了异常，则会回滚事务。 在两种情况下，在退出上面的 :class:`_orm.Session` 后，其后续将准备好用于接续的交易。

:meth:`_orm.Session.begin` 方法是可选的，同时 :class:`_orm.Session` 也可以以一种逐步提交的方法进行使用，在需要时会自动开始这些操作；这些操作只需要提交或者进行回滚即可:

    session = Session(engine)

    session.add(some_object())
    session.add(some_other_object())

    session.commit()  # 提交

    # 将自动再次开始
    result = session.execute("<some select statement>")
    session.add_all([more_objects, ...])
    session.commit()  # 提交

    session.add(still_another_object)
    session.flush()  # 刷新 still_another_object
    session.rollback()  # 回滚 still_another_object

:class:`_orm.Session` 本身具有一个 :meth:`_orm.Session.close` 方法。 如果 :class:`_orm.Session` 是在尚未提交或回滚的事务中开始的，此方法将取消该事务（即回滚），并且会使包含在 :class:`_orm.Session` 对象状态中的所有对象都被 expunge。 如果 :class:`_orm.Session` 在一个不保证进行 :meth:`_orm.Session.commit` 或 :meth:`_orm.Session.rollback` 的方式下使用（即不在上下文管理器或类似机制中），则可以使用 :class:`_orm.Session.close` 方法确保释放所有资源：

    # 删除所有对象并无条件的释放所有事务（包括回滚），将所有数据库连接返回到其 Engines 中
    session.close()

最后，会话构造 / 关闭过程本身也可以通过上下文管理器运行。 这是确保 :class:`_orm.Session` 对象的使用范围在一个固定块内的最佳方式。 首先通过 :class:`_orm.Session` 构造函数进行说明：

    with Session(engine) as session:
        session.add(some_object())
        session.add(some_other_object())

        session.commit()  # 提交

        session.add(still_another_object)
        session.flush()  # 刷新 still_another_object

        session.commit()  # 提交

        result = session.execute("<some SELECT statement>")

    # .execute() 调用的事务状态已经消除

同样，:class:`_orm.sessionmaker` 也可以以同样的方式使用：

    Session = sessionmaker(engine)

    with Session() as session:
        with session.begin():
            session.add(some_object)
        # 提交

    # 关闭 Session

:class:`_orm.sessionmaker` 本身包括 :meth:`_orm.sessionmaker.begin` 方法，允许两个操作同时执行：

    with Session.begin() as session:
        session.add(some_object)

.. _session_begin_nested:

使用保存点（SAVEPOINT）
-----------------------------------

如果支持底层引擎的 SAVEPOINT 事务，则可以使用 :meth:`~.Session.begin_nested` 方法突出显示此类事务:

    Session = sessionmaker()

    with Session.begin() as session:
        session.add(u1)
        session.add(u2)

        nested = session.begin_nested()  # 建立保存点
        session.add(u3)
        nested.rollback()  # 回滚 u3，保留 u1 和 u2

    # 提交 u1 和 u2

每次调用 :meth:`_orm.Session.begin_nested` 时，一个新的“BEGIN SAVEPOINT”命令会在当前数据库事务的范围内发出（如果尚未开始，则会启动一个），并且将返回一个类型为 :class:`_orm.SessionTransaction` 的对象，它表示对此 SAVEPOINT 的句柄。 当调用此对象的 ``.commit()`` 方法时，将向数据库发送 "RELEASE SAVEPOINT"，如果调用 ``.rollback()`` 方法，将发送 "ROLLBACK TO SAVEPOINT"。 父数据库事务仍然处于进行中。

:meth:`_orm.Session.begin_nested` 通常用作上下文管理器，特别是当可以捕获特定于每个实例的错误时，需要与这些错误对应的回滚，而无需回滚整个事务，例如下面的示例：

    for record in records:
        try:
            with session.begin_nested():
                session.merge(record)
        except:
            print(f"Skipped record {record}")
    session.commit()

当由 :meth:`_orm.Session.begin_nested` 发起的 SAVEPOINT 被回滚时，会过期在 SAVEPOINT 范围内被修改的内存中的对象状态，而未在 SAVEPOINT 开始时被更改的其他对象状态保持不变。 这是为了使后续操作可以继续使用
不影响的数据，而无需从数据库中刷新数据。

.. seealso::

    :meth:`_engine.Connection.begin_nested` - 核心 SAVEPOINT API

.. _orm_session_vs_engine:

会话级别与引擎级别的事务控制
--------------------------------------------------

核心的 :class:`_engine.Connection` 和 ORM 的 :class:`_session.Session` 具有等效的事务语义，在 :class:`_engine.Engine` 和 :class:`_engine.Connection` 级别以及 :class:`_orm.Session` 和 :class:`_engine.Connection` 级别。 以下几节详细讨论了这些方案，这是基于以下方案：

.. sourcecode:: text

    ORM                                           Core
    -----------------------------------------     -----------------------------------
    sessionmaker                                  Engine
    Session                                       Connection
    sessionmaker.begin()                          Engine.begin()
    some_session.commit()                         some_connection.commit()
    with some_sessionmaker() as session:          with some_engine.connect() as conn:
    with some_sessionmaker.begin() as session:    with some_engine.begin() as conn:
    with some_session.begin_nested() as sp:       with some_connection.begin_nested() as sp:

逐步提交
~~~~~~~~~~~~~~~~

:class:`_orm.Session` 和 :class:`_engine.Connection` 均具有 :meth:`_engine.Connection.commit` 和 :meth:`_engine.Connection.rollback`方法。 通过 SQLAlchemy 2.0样式的操作使用这些方法将在所有情况下影响**最外层**事务。 对于 :class:`_orm.Session`，假定 :paramref:`_orm.Session.autobegin` 保持默认值 ``True``。

:class:`_engine.Engine` ::

    engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

    with engine.connect() as conn:
        conn.execute(
            some_table.insert(),
            [
                {"data": "some data one"},
                {"data": "some data two"},
                {"data": "some data three"},
            ],
        )
        conn.commit()

:class:`_orm.Session` ::

    Session = sessionmaker(engine)

    with Session() as session:
        session.add_all(
            [
                SomeClass(data="some data one"),
                SomeClass(data="some data two"),
                SomeClass(data="some data three"),
            ]
        )
        session.commit()

开启事务
~~~~~~~~~~

:class:`_orm.sessionmaker` 和 :class:`_engine.Engine` 均具有 :meth:`_engine.Engine.begin` 方法，它将获得足以执行 SQL 语句的新对象（:class:`_orm.Session` 和 :class:`_engine.Connection`，分别）然后返回极维持 begin / commit / rollback 上下文管理器的句柄。

Engine::

    engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

    with engine.begin() as conn:
        conn.execute(
            some_table.insert(),
            [
                {"data": "some data one"},
                {"data": "some data two"},
                {"data": "some data three"},
            ],
        )
    # 自动提交并关闭

Session::

    Session = sessionmaker(engine)

    with Session.begin() as session:
        session.add_all(
            [
                SomeClass(data="some data one"),
                SomeClass(data="some data two"),
                SomeClass(data="some data three"),
            ]
        )
    # 自动提交并关闭

嵌套事务
~~~~~~~~~~~~~~~~~~~~

当使用 :meth:`_orm.Session.begin_nested` 或 :meth:`_engine.Connection.begin_nested` 方法时，返回的事务对象必须用于提交或回滚 SAVEPOINT。调用 :meth:`_orm.Session.commit` 或 :meth:`_engine.Connection.commit` 方法将始终提交**最外层**事务；这是从 1.x 系列中反转的 SQLAlchemy 2.0 特定行为。

Engine::

    engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

    with engine.begin() as conn:
        savepoint = conn.begin_nested()
        conn.execute(
            some_table.insert(),
            [
                {"data": "some data one"},
                {"data": "some data two"},
                {"data": "some data three"},
            ],
        )
        savepoint.commit()  # 或回滚

    # 自动提交

Session::

    Session = sessionmaker(engine)

    with Session.begin() as session:
        savepoint = session.begin_nested()
        session.add_all(
            [
                SomeClass(data="some data one"),
                SomeClass(data="some data two"),
                SomeClass(data="some data three"),
            ]
        )
        savepoint.commit()  # 或回滚
    # 自动提交

.. _session_explicit_begin:

显式开启事务
---------------

:class:`_orm.Session` 具有“autobegin”行为，这意味着只要操作开始，它就会确保存在 :class:`_orm.SessionTransaction` 以跟踪正在进行的操作。 当调用 :meth:`_orm.Session.commit` 时，此事务将完成。

通常，特别是在框架集成中，控制“开始”操作发生的时间很有用。 为此，:class:`_orm.Session` 使用“autobegin”策略，这样 :meth:`_orm.Session.begin` 方法就可以在尚未开始事务的 :class:`_orm.Session` 上直接调用：

    Session = sessionmaker(bind=engine)
    session = Session()
    session.begin()
    try:
        item1 = session.get(Item, 1)
        item2 = session.get(Item, 2)
        item1.foo = "bar"
        item2.bar = "foo"
        session.commit()
    except:
        session.rollback()
        raise

上面的模式更常规地使用上下文管理器：

    Session = sessionmaker(bind=engine)
    session = Session()
    with session.begin():
        item1 = session.get(Item, 1)
        item2 = session.get(Item, 2)
        item1.foo = "bar"
        item2.bar = "foo"

:meth:`_orm.Session.begin` 方法和会话的“autobegin”进程使用相同的步骤开始事务。 这包括调用 :meth:`_orm.SessionEvents.after_transaction_create` 事件，通常此钩子用于将其自己的事务处理集成到 ORM :class:`_orm.Session` 中。

.. _session_twophase:

启用两段提交
-------------------------

对于支持两段操作（目前为止是 MySQL 和 PostgreSQL）的后端，可以指示会话使用两段提交语义。 这将协调在数据库中对事务的提交，以便在所有数据库中提交或回滚事务。 您还可以通过 :meth:`_orm.Session.prepare` 以准备与由 SQLAlchemy 不管理的事务进行交互。 要使用两段提交，请在会话中设置标志 ``twophase=True``：

    engine1 = create_engine("postgresql+psycopg2://db1")
    engine2 = create_engine("postgresql+psycopg2://db2")

    Session = sessionmaker(twophase=True)

    # 将用户操作绑定到 engine 1，将帐户操作绑定到 engine 2
    Session.configure(binds={User: engine1, Account: engine2})

    session = Session()

    # .... 与账户和用户一起操作

    # 提交。 会话将对所有数据库执行 flush 和 prepare 步骤，然后提交两个事务
    session.commit()

.. _session_transaction_isolation:

设置事务隔离级别 / DBAPI AUTOCOMMIT
-------------------------------------------------------

大多数 DBAPI 都支持可配置的事务 :term:` 隔离` 级别。 通常为四个级别“READ UNCOMMITTED”、“READ COMMITTED”、“REPEATABLE READ”和“SERIALIZABLE”。 这些通常在 DBAPI 连接开始新事务之前应用。请注意，大多数 DBAPI 都会自动开始此事务并发出“BEGIN”。

支持隔离级别的 DBAPI 通常还支持真正的“autocommit”，这意味着 DBAPI 连接本身将处于非事务自动提交模式。 这通常意味着不会出现自动向数据库发出“BEGIN”的典型 DBAPI 行为，但它也可能包括其他指令。在使用此模式时，**DBAPI 不会在任何情况下使用事务**。 SQLAlchemy 方法如 ``.begin()``, ``.commit()`` 和 ``.rollback()`` 自动通过。

SQLAlchemy 的方言支持按 :class:`_engine.Engine` 或 :class:`_engine.Connection` 的基础设置可选的隔离模式，这些选项可以在 :func:`_sa.create_engine` 级别以及 :meth:`_engine.Connection.execution_options` 级别使用。

当使用 ORM :class:`.Session` 时，它充当 engines 和 connections 的 *接口*，但不直接公开事务隔离级别。 因此，要影响事务隔离级别，我们需要在 :class:`_engine.Engine` 或 :class:`_engine.Connection` 上进行操作。

.. seealso::

    :ref:`dbapi_autocommit` - 确保查看 SQLAlchemy :class:`_engine.Connection` 对象级别的隔离级别如何工作。

.. _session_transaction_isolation_enginewide:

为 :class:`.Session` 或 :class:`.sessionmaker` 全局设置隔离级别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了使 :class:`.Session` 或 :class:`.sessionmaker` 全局设置特定的隔离级别，第一种技术是可以在所有情况下针对特定的隔离级别建立 :class:`_engine.Engine`，然后将其用作 :class:`_orm.Session` 和 / 或 :class:`_orm.sessionmaker` 的连接资源：

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    eng = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test",
        isolation_level="REPEATABLE READ",
    )

    Session = sessionmaker(eng)

另一个选项，如果有两个引擎具有不同的隔离级别，则是使用 :meth:`_engine.Engine.execution_options` 方法，该方法将生成原始 :class:`_engine.Engine` 的浅拷贝，它与父引擎共享相同的连接池。当将操作分为“事务性”和“自动提交”操作时，这通常更可取：

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

    transactional_session = sessionmaker(eng)
    autocommit_session = sessionmaker(autocommit_engine)

上面，“eng”和``autocommit_engine`` 具有相同的 dialect 和连接池。但是，当从 ``autocommit_engine`` 获取连接时，“AUTOCOMMIT”模式将被设置为连接将被放置在非事务式自动提交模式下。这通常意味着不再发生自动发出“BEGIN”的典型 DBAPI 行为，但是它可能包括其他指令。此时，**“自动提交”模式下的 Session 仍具有事务语义**，其中包括 :meth:`_orm.Session.commit` 和 :meth:`_orm.Session.rollback` 仍将认为它们在“提交”和“回滚”对象，然后事务将悄悄消失。因此，**通常建议在只读模式下使用隔离级别为 AUTOCOMMIT 的会话**，如下例所示：

    with autocommit_session() as session:
        some_objects = session.execute("<statement>")
        some_other_objects = session.execute("<statement>")

    # 关闭连接

为多个绑定配置隔离级别
~~~~~~~~~~~~~~~~~~~~~~~~

对于 :class:`.Session` 或 :class:`.sessionmaker` 配置了多个 "binds" 的情况，可以重新完全指定 "binds" 参数，或者如果想仅替换单个 "binds"，则可以使用 :meth:`.Session.bind_mapper` 或 :meth:`.Session.bind_table` 方法：

    with Session() as session:
        session.bind_mapper(User, autocommit_engine)

为单个事务设置隔离级别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

关于隔离级别的一个关键警告是，在已经开始事务的 :class:`_engine.Connection` 上安全修改该设置是不可能的。数据库不能更改正在进行事务的隔离级别，并且一些 DBAPI 及 SQLAlchemy 方言在这个问题上具有不一致的行为。

因此，最好使用始终与所需隔离级别一起绑定的 :class:`_orm.Session`。 但是，可以通过在事务开始之前使用 :meth:`_orm.Session.connection` 方法影响每个连接上的每个事务的隔离级别：

    from sqlalchemy.orm import Session

    # 假定刚刚构造了会话
    sess = Session(bind=engine)

    # 在其他任何操作之前，使用选项调用 connection()。
    # 这将从绑定的 engine 中亲自摆脱新的连接并开始真实的数据库事务。
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # 在 SERIALIZABLE 隔离级别下操作会话 ...

    # 提交事务。连接被释放，
    # 然后重新设置为其之前的隔离级别。
    sess.commit()

    # 在上面的提交（）之后，如果需要，可以开始新的事务，
    # 除非再次设置了此隔离级别，否则将使用先前的默认隔离级别。

以上，我们首先使用构造函数或 :class:`.sessionmaker` 创建了 :class:`.Session`。 然后，我们通过在调用数据库级别事务之前调用 :meth:`.Session.connection` 来显式设置了数据库级别事务上的每个连接的隔离级别。使用此选择的事务会继续使用该选定隔离级别。 事务完成后，将重置隔离级别，以便将连接返回到连接池中之前。

可以使用 :meth:`_orm.Session.begin` 方法启动 :class:`_orm.Session` 级别的事务；在调用 :meth:`_orm.Session.connection` 之后，可以使用该方法来设置每个连接的事务隔离级别：

    sess = Session(bind=engine)

    with sess.begin():
        # 在其他任何操作之前，使用 options 调用 connection()。
        # 这将从绑定的 engine 中摆脱新的连接并开始真实的数据库事务。
        sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

        # 在 SERIALIZABLE 隔离级别下操作会话数据 ...

    # 在块外，事务已提交。 连接被释放，
    # 然后重新设置为其之前的隔离级别。

用事件跟踪事务状态
--------------------------------------

有关会话事务状态更改可用的事件挂钩，请参见 :ref:`session_transaction_events` 部分。

.. _session_external_transaction:

将会话加入外部事务中（例如用于测试套件）
========================================================================

如果使用的 :class:`_engine.Connection` 已经处于事务状态（即已建立 :class:`.Transaction`），则可以通过将 :class:`.Session` 绑定到该 :class:`_engine.Connection` 上，使:class:`.Session` 参与该事务。 理解这种原理的通常逻辑是测试套件，允许 ORM 代码自由地使用 :class:`.Session`，包括调用 :meth:`.Session.commit`，之后整个数据库交互被回滚。

.. versionchanged:: 2.0“加入外部事务”的配方再次得到优化（不需要事件处理程序来“重置”嵌套事务）。

该配方通过建立处于事务状态（即具有 :class:`.Transaction`）的 :class:`_engine.Connection` 以及可能的 SAVEPOINT，然后将其作为 "bind" 传递给 :class:`.Session`，该参数为 :paramref:`_orm.Session.join_transaction_mode` 传递了设置``"create_savepoint"`，这意味着应创建新的 SAVEPOINT，以实现 :class:`.Session` 中的 BEGIN/COMMIT/ROLLBACK，这将使外部事务保持与传递进来时相同的状态。

当测试代码撤销时，将回滚外部事务，以便将测试期间的所有数据更改全部恢复::

    from sqlalchemy.orm import sessionmaker
    from sqlalchemy import create_engine
    from unittest import TestCase

    # 全局应用程序范围。创建 Session 类、engine
    Session = sessionmaker()

    engine = create_engine("postgresql+psycopg2://...")


    class SomeTest(TestCase):
        def setUp(self):
            # 连接到数据库
            self.connection = engine.connect()

            # 开始一个非 ORM 事务
            self.trans = self.connection.begin()

            # 将个别会话绑定到连接上，选择“create_savepoint”join_transaction_mode
            self.session = Session(
                bind=self.connection, join_transaction_mode="create_savepoint"
            )

        def test_something(self):
            # 使用会话进行测试。

            self.session.add(Foo())
            self.session.commit()

        def test_something_with_rollbacks(self):
            self.session.add(Bar())
            self.session.flush()
            self.session.rollback()

            self.session.add(Foo())
            self.session.commit()

        def tearDown(self):
            self.session.close()

            # 回滚 - 将上面针对 Session 的所有内容（包括对 commit() 的调用）全部回滚。
            self.trans.rollback()

            # 将连接返回到 Engine。
            self.connection.close()

上述配方是 SQLAlchemy 自己使用的一部分，以确保其按预期工作。