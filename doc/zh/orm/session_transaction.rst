======================================
事务和连接管理
======================================

.. _unitofwork_transaction:

管理事务
=====================

.. versionchanged:: 1.4 会话事务管理已经得到了修订，使其更加清晰和易于使用。特别是，现在支持“自动开始”操作，这意味着可以控制事务开始的时间，而不需要使用传统的“自动提交”模式。

  :class:`_orm.Session`  在任何时候跟踪单个“虚拟”事务的状态，使用一个名为   :class:` _orm.SessionTransaction`  的对象。然后，该对象利用   :class:`_orm.Session`  对象绑定到的底层   :class:` _engine.Engine`  或引擎，在需要时使用   :class:`_engine.Connection`  对象开始实际的连接级事务。

这个“虚拟”事务将在需要时自动创建，或者可以使用  :meth:`_orm.Session.begin`  方法启动。尽可能地支持 Python 上下文管理器用于创建   :class:` _orm.Session`  对象以及维护   :class:`_orm.SessionTransaction`  的范围。

下面假设我们从一个   :class:`_orm.Session`  开始：

    from sqlalchemy.orm import Session
    
    session = Session(engine)

现在我们可以使用上下文管理器在标记的事务内运行操作：

    with session.begin():
        session.add(some_object())
        session.add(some_other_object())

    # 在结束时提交事务，或者如果有异常抛出，则回滚事务

在上述上下文中，假设没有引发异常，则任何待处理的对象都将刷新到数据库中，并且数据库事务将提交。如果在上面的块中引发了异常，则将回滚事务。在两种情况下，上述   :class:`_orm.Session`  在退出块之后准备好用于后续事务。

  :meth:`_orm.Session.begin`   方法是可选的，还可以使用逐个提交的方法来使用   :class:` _orm.Session` ，其中它会根据需要自动开始事务；这些只需要提交或回滚：

    session = Session(engine)

    session.add(some_object())
    session.add(some_other_object())

    session.commit()  # 提交

    # 会自动重新开始
    result = session.execute("< some select statement >")
    session.add_all([more_objects, ...])
    session.commit()  # 提交

    session.add(still_another_object)
    session.flush()  # 刷新 still_another_object
    session.rollback()  # 回滚 still_another_object

  :class:`_orm.Session`  本身具有  :meth:` _orm.Session.close`  方法。如果在尚未提交或回滚的事务内开始了   :class:`_orm.Session` ，则该方法将取消该事务（即回滚），并且还将清除   :class:` _orm.Session`  对象状态中包含的所有对象。如果以这种方式使用   :class:`_orm.Session` ，即不能保证调用  :meth:` _orm.Session.commit`  或  :meth:`_orm.Session.rollback` （例如不在上下文管理器或类似的工具中），则可使用   :class:` _orm.Session.close`  方法释放所有资源：

    # 取消 (rollback) 所有事务，释放所有对象
    # (包括那些未提交的)，释放所有数据库连接到它们的
    # 引擎
    session.close()

最后，会话构造 / 关闭过程本身也可以通过上下文管理器运行。这是确保   :class:`_orm.Session`  对象使用范围在固定块中作用域的最佳方式。首先通过   :class:` _orm.Session`  构造函数进行说明：

    with Session(engine) as session:
        session.add(some_object())
        session.add(some_other_object())

        session.commit()  # 提交

        session.add(still_another_object)
        session.flush()  # 刷新 still_another_object

        session.commit()  # 提交

        result = session.execute("<some SELECT statement>")

    # 抛弃 execute() 调用的剩余事务状态

类似地，  :class:`_orm.sessionmaker`  可以以相同方式使用：

    Session = sessionmaker(engine)

    with Session() as session:
        with session.begin():
            session.add(some_object)
        # 提交

    # 关闭 Session

  :class:`_orm.sessionmaker`  本身包括一个  :meth:` _orm.sessionmaker.begin`  方法，允许同时执行两个操作：

    with Session.begin() as session:
        session.add(some_object)

.. _session_begin_nested:

使用 SAVEPOINT
---------------

如果底层引擎支持 SAVEPOINT 事务，则可以使用  :meth:`~.Session.begin_nested`  方法确定 SAVEPOINT 事务：

    Session = sessionmaker()

    with Session.begin() as session:
        session.add(u1)
        session.add(u2)

        nested = session.begin_nested()  # 建立 savepoint
        session.add(u3)
        nested.rollback()  # 回滚 u3，保留 u1 和 u2

    # 提交 u1 和 u2

每次调用  :meth:`_orm.Session.begin_nested`  时，都会在当前数据库事务的范围内（如果尚未启动则开始）向数据库发出新的“BEGIN SAVEPOINT”命令，并返回一个   :class:` _orm.SessionTransaction`  类型的对象，该对象表示对此 SAVEPOINT 的句柄。当开始一个新的范围嵌套  :meth:`_orm.SessionTransaction.commit`  或  :meth:` _orm.SessionTransaction.rollback`  在上下文器内部或直接调用这些方法时，在 SAVEPOINT 句柄范围内开始的真正的传统事务将被提交或回滚，直到达到顶层范围的真正结束为止。在这个对象上调用``.commit()``方法时，会向数据库发出“RELEASE SAVEPOINT”的命令，而如果调用``.rollback()``方法，则会发出“ROLLBACK TO SAVEPOINT”的命令。包含的数据库事务仍在进行中。

  :meth:`_orm.Session.begin_nested`  通常用作上下文管理器，在其中可以捕获特定的每个实例错误，结合针对该事务状态的回滚来使用，而不会回滚整个事务，例如下面的示例::

    for record in records:
        try:
            with session.begin_nested():
                session.merge(record)
        except:
            print("Skipped record %s" % record)
    session.commit()

当由  :meth:`_orm.Session.begin_nested`  产生的上下文管理器完成时，它会“提交”保存点，其中包括刷新所有挂起状态的常规行为。当发生错误时，则会回滚保存点，并将更改的对象的本地   :class:` _orm.Session`  状态过期。

对于 PostgreSQL 和捕获 :class:`.IntegrityError` 的异常以检测重复行的情况，此模式非常理想。在 PostgreSQL 中，当引发此类错误时，通常会中止整个事务，但使用 SAVEPOINT 时，外部事务得以维持。在下面的示例中，将一组数据持久保存到数据库中，并跳过偶尔出现的“重复主键”记录，而不回滚整个操作：

    from sqlalchemy import exc

    with session.begin():
        for record in records:
            try:
                with session.begin_nested():
                    obj = SomeRecord(id=record["identifier"], name=record["name"])
                    session.add(obj)
            except exc.IntegrityError:
                print(f"Skipped record {record} - row already exists")

当调用  :meth:`~.Session.begin_nested`  时，  :class:` _orm.Session`  首先将当前所有挂起状态刷新到数据库中；这不受  :paramref:`_orm.Session.autoflush`  参数值的影响，该参数通常可以用于禁用自动刷新。这种行为的理由是，在嵌套事务发生回滚时，  :class:` _orm.Session`  可以使范围内创建的任何内存状态过期，同时确保当刷新这些过期对象时，开始 SAVEPOINT 之前的对象图的状态将可从数据库重新加载。

在现代版本的 SQLAlchemy 中，当由  :meth:`_orm.Session.begin_nested`  发起的 SAVEPOINT 被回滚时，与 SAVEPOINT 创建后进行修改的内存中对象状态过期，但自 SAVEPOINT 开始后未被修改的其他对象状态将保持不变。这样，就可以使后续操作继续利用未受影响的数据，而无需将其从数据库中刷新。

.. seealso::

     :meth:`_engine.Connection.begin_nested`  - 核心 SAVEPOINT API

.. _orm_session_vs_engine:

会话级和引擎级事务控制
--------------------------------------------------

Core 中的   :class:`_engine.Connection`  和 ORM 中的   :class:` _session.Session`  具有等效的事务语义，分别在  :class:`_orm.sessionmaker`  vs.   :class:` _engine.Engine`  和   :class:`_orm.Session`  vs.   :class:` _engine.Connection`  的级别上。以下部分详细说明了这些情况的各个方面，基于以下方案：

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

随时提交
~~~~~~~~~~~~~~~~

  :class:`_orm.Session`  和   :class:` _engine.Connection`  都具有  :meth:`_engine.Connection.commit`  和  :meth:` _engine.Connection.rollback`  方法。使用 SQLAlchemy 2.0 样式的操作，这些方法在所有情况下都会影响最外层的事务。对于   :class:`_orm.Session` ，假定  :paramref:` _orm.Session.autobegin`  的默认值为 ``True``。

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

只开始一次
~~~~~~~~~~

  :class:`_orm.sessionmaker`  方法，该方法将实例化SQL执行对象（分别是  :class:` _orm.Session` ），然后返回一个上下文管理器以维护此对象的事务上下文(commit/rollback)。

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
    # 自动提交并关闭连接

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
    # 自动提交并关闭连接

嵌套事务
~~~~~~~~~~~~~~~~~~~~

在使用SAVEPOINT时，需通过  :meth:`_orm.Session.begin_nested`  或  :meth:` _engine.Connection.begin_nested`  方法。
返回的事务对象用于提交或回滚SAVEPOINT。
调用  :meth:`_orm.Session.commit`  或  :meth:` _engine.Connection.commit`  方法始终会提交最外层的事务。这是Sqlalchemy 2.0 的特定行为，与之前版本（1.x）相反。

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
        savepoint.commit()  # 或回滚（rollback）

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
        savepoint.commit()  # 或回滚（rollback）

.. _session_explicit_begin:

显式开始
---------------

默认情况下，当ORM Session 执行操作时，会自动创建  :class:`_orm.SessionTransaction`  时完成。而在一些框架集成的场景中，需要手动控制"开始"操作的时机。为了满足这个需求，  :class:` _orm.Session`  使用"autobegin"策略。
  :meth:`_orm.Session.begin`  方法可以在没有已开始的事务的情况下，直接对 :class:` _orm.Session`进行调用。例如::

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

上述模式更常见的是使用上下文管理器来调用::

    Session = sessionmaker(bind=engine)
    session = Session()
    with session.begin():
        item1 = session.get(Item, 1)
        item2 = session.get(Item, 2)
        item1.foo = "bar"
        item2.bar = "foo"

  :meth:`_orm.Session.begin`  方法和Session的"autobegin"过程使用相同的步骤来开始事务。
这包括当执行时调用  :meth:`_orm.SessionEvents.after_transaction_create`  事件。
在框架中使用此钩子，可以将其自身的事务处理过程与ORM :class:`_orm.Session` 集成。


.. _session_twophase:

启用双阶段提交（two-phase commit）机制
-------------------------

对于支持两阶段commit操作（MySQL和PostgreSQL）的后端，可以通过设置参数``twophase=True``来启用双阶段提交机制。这将协调所有数据库中的三个对象的提交或回滚。你也可以为已有事务 prepare 一个session，使之可以与ORM未解决的事务进行交互。对于需要在两个实例之间提交事务的负载平衡设置中必须启用此机制。

例如：

.. code-block:: python

    engine1 = create_engine("postgresql+psycopg2://db1")
    engine2 = create_engine("postgresql+psycopg2://db2")

    Session = sessionmaker(twophase=True)

    # 将 User 操作绑定到 engine1，将 Account 操作绑定到 engine2
    Session.configure(bind={User: engine1, Account: engine2})

    session = Session()

    # ... ...处理帐户以及用户

    # 提交。会将事务提交到所有DBs，包括一个flush处理过程，和一个prepare并提交的操作
    session.commit()

.. _session_transaction_isolation:

设置事务隔离级别 / DBAPI_AUTOCOMMIT
-------------------------------------------------------

大多数 DBAPI 都支持配置事务隔离级别。可能需要了解更多关于DBAPI 隔离级别的相关知识后使用。传统上有四个等级："READ UNCOMMITTED"，"READ COMMITTED", "REPEATABLE READ"和"SERIALIZABLE"。这些通常在一个DBAPI连接在开始新事务之前应用，注意大多数DBAPI在第一次发出SQL语句时会隐式开始这个事务。

支持隔离级别的DBAPI通常也支持真正的"自动提交"概念，这意味着DBAPI连接本身将被放置在非事务性的自动提交模式下。这通常意味着发出"BEGIN"到数据库的典型DBAPI行为不再发生，但它也可能包括其他指令。在使用此模式时，**DBAPI在任何情况下都不使用事务**。SQLAlchemy像``.begin()``, ``.commit()``和``.rollback()``之类的方法会被静默地跳过。

SQLAlchemy的方言支持在每个  :class:`_engine.Engine`  级别的标志。

在使用ORM :class:`~sqlalchemy.orm.session.Session` 时，它充当引擎和连接的*facade*，但不直接暴露事务隔离。因此，为了影响事务隔离级别，我们需要根据情况对 :class:`_engine.Engine` 或 :class:`_engine.Connection` 采取行动。

.. seealso::

      :ref:`dbapi_autocommit`  - 请务必查看isolation levels如何在SQLAlchemy  :class:` _engine.Connection`对象级别工作。

.. _session_transaction_isolation_enginewide:

为Sessionmaker / Engine Wide设置隔离
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

要全局设置特定的隔离级别，第一种技术是：可以构造针对所有情况具有特定隔离级别的  :class:`_engine.Engine` ，然后将其用作 :class:` _orm.Session`和/或 :class:`_orm.sessionmaker` 的连接源：

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    eng = create_engine(
        "postgresql+psycopg2://scott:tiger@localhost/test",
        isolation_level="REPEATABLE READ",
    )

    Session = sessionmaker(eng)

另一个选项是，如果有两个具有不同隔离级别的引擎，可以使用  :meth:`_engine.Engine.execution_options`  方法，它将生成原始 :class:` _engine.Engine`的浅拷贝，该引擎与主引擎共享相同的连接池。当操作将被分为“事务”和“自动提交”操作时，这通常是更可取的：

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

    transactional_session = sessionmaker(eng)
    autocommit_session = sessionmaker(autocommit_engine)

上面， "``eng``" 和 ``"autocommit_engine"``共享相同的方言和连接池。但是，当从``autocommit_engine``获得连接时，将设置"AUTOCOMMIT"模式。然后，这两个  :class:`_orm.sessionmaker` ` transactional_session``"和"`autocommit_session"``在使用数据库连接时继承这些特性。

"``autocommit_session``"仍具有事务语义，包括  :meth:`_orm.Session.commit`  和  :meth:` _orm.Session.rollback`  仍然认为它们在"committing"和"rolling back"对象，但事务将被静默地忽略。因此，**通常情况下，但不是严格要求，一个具有AUTOCOMMIT隔离的会话以只读方式使用**，即：

    with autocommit_session() as session:
        some_objects = session.execute("<statement>")
        some_other_objects = session.execute("<statement>")

    # closes connection

为单独的会话设置隔离
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当我们创建一个新的  :class:`.Session` ，直接使用构造函数或当我们调用由  :class:` .sessionmaker` .sessionmaker`创建我们的 :class:`_orm.Session` 并传递设置为自动提交的引擎：

    plain_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    autocommit_engine = plain_engine.execution_options(isolation_level="AUTOCOMMIT")

    # will normally use plain_engine
    Session = sessionmaker(plain_engine)

    # make a specific Session that will use the "autocommit" engine
    with Session(bind=autocommit_engine) as session:
        # work with session
        ...

对于配置有多个绑定的  :class:`.Session` .sessionmaker` 的情况，我们可以重新指定完整的"binds"参数，或者如果我们只想替换特定的绑定，我们可以使用  :meth:`.Session.bind_mapper`  或  :meth:` .Session.bind_table`  方法：

    with Session() as session:
        session.bind_mapper(User, autocommit_engine)

为独立事务设置隔离
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

关于隔离级别的一个关键警告是，不能在已经有事务的 :class:`_engine.Connection` 上安全地修改设置。开始。在进行的事务中，数据库无法更改隔离级别，某些DBAPI和SQLAlchemy方言在这个领域存在不一致的行为。

因此最好使用   :class:`_orm.Session` ，它可以提前绑定到具有所需隔离级别的引擎上。然而，可以通过在事务开始时使用  :meth:` _orm.Session.connection`  方法来影响每个连接的隔离级别:

    from sqlalchemy.orm import Session

    # 假设会话刚刚被构建
    sess = Session(bind=engine)

    # 在任何其他操作之前使用选项调用connection()。
    # 这将从绑定的引擎中获取新连接并开始一个真正的数据库事务。
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # ... 在 SERIALIZABLE 隔离级别中使用会话...

    # 提交事务。该连接将被释放并恢复到其先前的隔离级别。
    sess.commit()

    # 在上面的 commit() 之后，可以开始新的事务，该事务将继续以先前的默认隔离级别进行，除非再次设置。

上面，我们首先使用构造函数或者一个   :class:`.sessionmaker`  来生成一个   :class:` .Session` 。然后，通过调用  :meth:`.Session.connection`  来明确设置开始数据库级事务，该方法提供了执行选项，这些选项将在开始数据库级事务之前传递给连接。事务以所选隔离级别进行。当事务完成时，将在连接上重置隔离级别以恢复默认设置，然后再返回连接池。

  :meth:`_orm.Session.begin`   方法也可用于开始   :class:` _orm.Session`  级别事务。在该调用之后使用  :meth:`_orm.Session.connection`  可用于设置每个连接的事务隔离级别:

    sess = Session(bind=engine)

    with sess.begin():
        # 在任何其他操作之前使用选项调用connection()。
        # 这将从绑定的引擎中获取新连接并开始一个真正的数据库事务。
        sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

        # ... 在 SERIALIZABLE 隔离级别中使用会话...

    # 在块外面，事务已提交。连接被释放并恢复到先前的隔离级别。

使用事件跟踪事务状态
--------------------------------------

请参见部分   :ref:`session_transaction_events` ，了解可用的用于会话事务状态更改的事件挂钩的概述。

.. _session_external_transaction:

加入外部事务的会话（例如用于测试套件）
========================================================================

如果正在使用已处于事务状态（即已建立   :class:`.Transaction` ）的   :class:` _engine.Connection` ，则可以通过将   :class:`.Session`  绑定到该   :class:` _engine.Connection` ，使   :class:`.Session`  参与其中在此事务中。这样做的常见原因是测试套件，允许 ORM 代码自由地使用   :class:` .Session` ，包括调用  :meth:`.Session.commit`  的能力，在此之后，整个数据库交互将被回滚。

.. versionchanged:: 2.0 "加入外部事务" 的方法在 2.0 中得到了新的改进。不再需要 "重置" 嵌套事务的事件处理程序。

该方法的工作方式是在事务内部建立一个   :class:`_engine.Connection`  并可选择一个 SAVEPOINT，然后将其传递给   :class:` _orm.Session`  作为 "bind"；通过  :paramref:`_orm.Session.join_transaction_mode`  参数传递设置为 ` `"create_savepoint"``，指示应该创建新的 SAVEPOINT，以实现   :class:`_orm.Session`  的 BEGIN/COMMIT/ROLLBACK，这将使外部事务保持在传递它时的相同状态。

当测试结束时，将回滚外部事务，以便撤消整个测试期间的任何数据更改:

    from sqlalchemy.orm import sessionmaker
    from sqlalchemy import create_engine
    from unittest import TestCase

    # 应用全局范围。创建 Session 类和 engine
    Session = sessionmaker()

    engine = create_engine("postgresql+psycopg2://...")


    class SomeTest(TestCase):
        def setUp(self):
            # 连接数据库
            self.connection = engine.connect()

            # 开始非 ORM 事务
            self.trans = self.connection.begin()

            # 将一个单独的 Session 绑定到该连接上，并选择
            # "create_savepoint" join_transaction_mode
            self.session = Session(
                bind=self.connection, join_transaction_mode="create_savepoint"
            )

        def test_something(self):
            # 在测试中使用 session

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

            # 回滚 - 上述 Session 中发生的所有内容（包括对 commit() 的调用）
            # 都将被回滚
            self.trans.rollback()

            # 将连接返回给 Engine
            self.connection.close()

上述方法是 SQLAlchemy 的自身 CI 的一部分，以确保它仍然按预期工作。