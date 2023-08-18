==============
会话基础知识
==============


会话的作用是什么？
--------------------------

从最一般的角度来看，   :class:`~.Session`  建立所有与数据库的对话，并代表“持有区”包含您在它的寿命期内加载或关联的所有对象。它是 SELECT 和其他查询的接口，这些查询将返回和修改 ORM 映射的对象。 ORM 对象本身是在   :class:` .Session`  中维护的，在一个叫做  :term:`身份映射(identity map)`  的结构中 - 一个数据结构，维护每个对象的唯一副本，其中 “唯一” 意味着“只有一个具有特定主键的对象”。

  :class:`.Session`  开始时基本上是一个无状态的，在发出查询或持久化其他对象后，它会从与   :class:` .Session`  关联的   :class:`_engine.Engine`  中请求一个连接资源，然后在连接上建立一个事务。这个事务将一直保持到   :class:` .Session`  被指示提交或回滚事务为止。

被   :class:`_orm.Session`  维护的 ORM 对象是  :term:` instrumented` ，当 Python 程序中的属性或集合被修改时，会生成一个更改事件，这个事件会被记录到   :class:`_orm.Session`  中。当要查询数据库时，或者在事务将要提交时，  :class:` _orm.Session`  首先将内存中存储的所有待决更改刷新到数据库。这被称为  :term:`unit of work`  模式。

使用一个   :class:`.Session`  时，将 ORM 映射的对象维护为 **代理对象(proxy object)**，而这些对象对于保持与当前交易匹配的状态非常有用。为了保持对象在数据库中的状态与实际情况一致，有各种事件会导致对象重新访问数据库以保持同步。可以“分离”   :class:` .Session`  中的对象，并继续使用，虽然这种做法是有危险的。通常建议，当要再次使用这些对象时，应将它们重新关联到另一个   :class:`.Session`  中，以便它们可以恢复正常的数据库状态。

.. _session_basics:

使用会话的基础
-------------------------

这里介绍了最基本的   :class:`.Session`  使用模式。

.. _session_getting:

开启和关闭会话
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`_orm.Session`  可以独立构建，也可以使用   :class:` _orm.sessionmaker`  类。通常情况下，会在开始时预先传递一个   :class:`_engine.Engine`  对象作为连接资源。一种典型的用法可能是::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import Session

    # 连接资源由Session使用
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

    # 创建会话并添加对象
    with Session(engine) as session:
        session.add(some_object)
        session.add(some_other_object)
        session.commit()

上面的   :class:`_orm.Session`  被实例化为关联到特定数据库 URL 的   :class:` _engine.Engine` 。然后它在 Python 上下文管理器（即 `with:` 语句）中使用，以便在块结束时它会被自动关闭。这相当于调用  :meth:`_orm.Session.close`  方法。

调用  :meth:`_orm.Session.commit`  是可选的，并且仅在与   :class:` _orm.Session`  的工作包括新数据需要持久化到数据库时才需要。如果我们只是发出 SELECT 调用并且不需要写任何更改，则不需要调用  :meth:`_orm.Session.commit` 。

.. note::

    注意，在调用  :meth:`_orm.Session.commit`  之后，无论是显式调用还是使用上下文管理器，与   :class:` .Session`  关联的所有对象都被  :term:`expired` ，意味着它们的内容被擦除以在下一个事务中重新加载。如果这些对象被  :term:` detach` ，除非使用  :paramref:`.Session.expire_on_commit `  参数来禁用这个行为，否则它们将无法使用，直到重新关联到新的   :class:` .Session` 。请参阅本节   :ref:`session_committing`  中的详细信息。

.. _session_begin_commit_rollback_block:

构建/提交/回滚块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们还可以在块中将  :meth:`_orm.Session.commit`  调用和整体“框架”的事务包含在一个上下文管理器中，用于那些将提交数据写入数据库的情况。发挥“框架”的意思是，如果所有操作都成功，那么将调用  :meth:` _orm.Session.commit`  方法，但如果出现任何异常，则将调用  :meth:`_orm.Session.rollback`  方法，以便立即回滚事务，然后将异常向外传播。在 Python 中，这通常会使用类似于一个“try：/except：/else：”块来表达，如下所示::

    # 上一段示例的详细版
    with Session(engine) as session:
        session.begin()
        try:
            session.add(some_object)
            session.add(some_other_object)
        except:
            session.rollback()
            raise
        else:
            session.commit()

如上所示的操作序列，可以更简洁地实现，方法是使用由  :meth:`_orm.Session.begin`  方法返回的   :class:` _orm.SessionTransaction`  对象，该对象为相同的操作序列提供了上下文管理器接口::

    # 创建会话并添加对象
    with Session(engine) as session:
        with session.begin():
            session.add(some_object)
            session.add(some_other_object)
        # 内部上下文调用 session.commit()，如果没有异常
    # 外部上下文调用 session.close()

更简洁的是，可以合并这两个上下文::

    # 创建会话并添加对象
    with Session(engine) as session, session.begin():
        session.add(some_object)
        session.add(some_other_object)
    # 内部上下文调用 session.commit()，如果没有异常
    # 外部上下文调用 session.close()

使用 sessionmaker
~~~~~~~~~~~~~~~~~~~~

  :class:`_orm.sessionmaker`  的作用是提供一个具有固定配置的   :class:` _orm.Session`  对象的工厂。由于应用程序通常会在模块范围内拥有一个   :class:`_engine.Engine`  对象，因此   :class:` _orm.sessionmaker`  可以为针对此引擎的   :class:`_orm.Session`  对象提供一个工厂::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    # 连接资源由 Session 使用
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

    # sessionmaker()，在与引擎相同的范围内使用
    Session = sessionmaker(engine)

    # 我们现在可以构建一个 Session()，而不需要每次传递引擎
    with Session() as session:
        session.add(some_object)
        session.add(some_other_object)
        session.commit()
    # 关闭会话

与   :class:`_engine.Engine`  的行为相似，  :class:` _orm.sessionmaker`  是模块级函数级会话/连接的工厂。因此，它也具有它自己的  :meth:`_orm.sessionmaker.begin`  方法，类似于  :meth:` _engine.Engine.begin` ，该方法返回一个   :class:`_orm.Session`  对象，同时也维护一个 begin/commit/rollback 块::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    # 连接资源由Session使用
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

    # 此处engine是父级提供程序
    Session = sessionmaker(engine)

    # 我们现在可以使用 Session.begin() 创建 Session() 并包含 begin()/commit()/rollback()
    with Session.begin() as session:
        session.add(some_object)
        session.add(some_other_object)
    # 提交事务，关闭会话

上面的例子中，当上述“with:”块结束时，   :class:`_orm.Session`  将在提交事务并关闭：class:` _orm.Session`。

在编写应用程序时，应该将   :class:`.sessionmaker`  工厂与由   :func:` _sa.create_engine`  创建的   :class:`_engine.Engine`  对象具有相同的作用域，通常在模块级或全局级别。由于这些对象都是工厂，它们可以被任意数量的函数和线程同时使用。

.. seealso::

      :class:`_orm.sessionmaker` 

      :class:`_orm.Session` 


.. _session_querying_20:

查询
~~~~~~~~

查询的主要方式是利用   :func:`_sql.select`  构造函数创建一个   :class:` _sql.Select`  对象，然后使用  :meth:`_orm.Session.execute`  和  :meth:` _orm.Session.scalars`  等方法执行它以返回结果。然后以类似于   :class:`_result.Result`  对象的形式返回结果，其中包括子变量，例如   :class:` _result.ScalarResult` 。

ORM 查询的完整指南可以在   :ref:`queryguide_toplevel`  中找到。以下是一些简短的示例::

    from sqlalchemy import select
    from sqlalchemy.orm import Session

    with Session(engine) as session:
        # 查询 ``User`` 对象
        statement = select(User).filter_by(name="ed")

        # ``User`` 对象列表
        user_obj = session.scalars(statement).all()

        # 查询单个列
        statement = select(User.name, User.fullname)

        # Row 对象列表
        rows = session.execute(statement).all()

.. versionchanged:: 2.0

    2.0 查询方式是标准的。有关从 1.x 系列的迁移说明，请参见   :ref:`migration_20_query_usage` 。

.. seealso::

     :ref:`queryguide_toplevel` 

.. _session_adding:


添加新项或现有项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :meth:`~.Session.add`   用于将实例放置在会话中。对于  :term:` transient` （即全新的）实例，这将对下一个刷新时进行插入。对于  :term:`persistent` （i.e. 是由此会话加载的）实例，它们已经存在，并不需要添加。可以使用这种方法来  :term:` detach` （即已从会话移除）的实例重新关联会话::

    user1 = User(name="user1")
    user2 = User(name="user2")
    session.add(user1)
    session.add(user2)

    session.commit()  # 将更改写入数据库

要一次将项目列表添加到会话中，请使用  :meth:`~.Session.add_all` ::

    session.add_all([item1, item2, item3])

  :meth:`~.Session.add`   操作沿着 “save-update” 级联， 对其属性和集合进行更改时发生级联操作的相关重要行为，请参阅   :ref:` unitofwork_cascades`  节。

.. _session_deleting:

删除
~~~~~~~~

  :meth:`~.Session.delete`   方法将实例放置到会话的对象删除列表中::


    session.delete(obj1)
    session.delete(obj2)

    session.commit()

  :meth:`_orm.Session.delete`   标记要删除的对象，将对每个受影响的主键发出 DELETE 语句。在待删除的行在刷新之前，标有“删除”的对象将存在于  :attr:` _orm.Session.deleted`  集合中。在执行 DELETE 之后，他们将从   :class:`_orm.Session`  中删除，并且在事务提交后，   :class:` _orm.Session`  变为永久状态。

与  :meth:`_orm.Session.delete`  相关的各种重要行为，特别是如何处理其他对象和集合的关系，有各种信息。在   :ref:` unitofwork_cascades`  节中有更多关于这公司工作的信息，但通常的规则是：

* 通过   :func:`_orm.relationship`  指令与删除对象相关联的映射对象对应的行默认情况下是不会删除的，如果这些对象对待删除的行具有反向的外键约束，则这些列将被设置为 NULL。如果这些列是不可为空的，则会导致约束冲突。

* 通过在   :func:`~_orm.relationship`  上使用   :ref:` cascade_delete`  级联将“SET NULL”更改为删除相关对象的行。

* 对于通过  :paramref:`_orm.relationship.secondary`  参数连接为“多对多”的表中的行，在删除它们所引用的对象时，在所有情况下它们都将被删除。

* 当相关对象包括对被删除对象的外键约束并且它们所属的关联集合当前未加载到内存中时，单元操作将发出一个 SELECT，以便其主键值或在这些相关行上发出 UPDATE 或 DELETE 语句。这样，ORM 将在不需要其他指令的情况下执行 ON DELETE CASCADE 的功能，即使在使用此功能时未对 Core 的   :class:`_schema.ForeignKeyConstraint`  进行配置。

* 可以使用  :paramref:`_orm.relationship.passive_deletes`  参数调整此行为，并自然地依赖于“ON DELETE CASCADE”。当设置为 True 时，此 SELECT 操作将不再发生，但是适用本地存在的行仍会受到显式 SET NULL 或 DELETE 的影响。将  :paramref:` _orm.relationship.passive_deletes`  设置为字符串 `"all"` 将禁用所有相关对象更新/删除。

* 当删除一个标记为删除的对象时，不会自动从引用它的集合或对象引用中移除对象。当   :class:`_orm.Session`  过期时，这些集合可以再次加载，以便对象不再存在。然而，与其使用  :meth:` _orm.Session.delete`  删除这些对象，不如将对象从其集合中移除，然后使用   :ref:`cascade_delete_orphan`  使它在移除集合时作为次要效应被删除。请参阅   :ref:` session_deleting_from_collections`  节以获取此类示例。

.. seealso::

      :ref:`cascade_delete`  —— 描述“删除级联”，当引导对象被删除时，它标记相关对象以被删除。

      :ref:`cascade_delete_orphan`  —— 描述了“删除孤儿级联 orphan”，它标记相关对象以在其与引导对象的关系被取消时删除。

      :ref:`session_deleting_from_collections`  —— 重要的背景信息，删除涉及关系在内存中刷新的方式。

.. _session_flushing:

刷新
~~~~~~~~

当使用其默认配置的   :class:`~sqlalchemy.orm.session.Session`  时，刷新步骤几乎总是透明执行的。特别是，在执行任何单个 SQL 语句的结果之前，都会刷新所有未处理的 SQL 语句，或使用查询返回结果（最终使用  :meth:` _orm.Session.execute` ），或者如果在  :meth:`.Session.commit`  调用之前更改了内部   :class:` .Session`  对象的状态，那么在  :meth:`.Session.commit`  之前也会进行刷新。在使用  :meth:` .Session.begin_nested`  时发出保存点时，也会进行刷新。

可以随时通过调用  :meth:`~.Session.flush`  方法来强制执行会话刷新::

    session.flush()

几乎总是在特定方法的范围内执行自动刷新步骤，这些方法包括:

* 当针对启用 ORM 的 SQL 结构，例如引用了 ORM 实体和/或 ORM 映射属性的   :func:`_sql.select`  对象或其他 SQL 执行方法时，例如  :meth:` _orm.Session.execute`  和其他 SQL 执行方法。
* 当使用   :class:`_query.Query`  执行以返回结果的方式执行时（这最终使用  :meth:` _orm.Session.execute` ）。
* 当在  :meth:`.Session.merge`  方法之前进行本地未处理对象刷新时。
* 懒加载操作针对未加载的对象属性时。

此外，还有一些点无条件地执行 **刷新**。这些点是键事务边界，包括：

* 在  :meth:`.Session.commit`  方法的过程中。
* 当调用  :meth:`.Session.begin_nested`  时
* 当使用  :meth:`.Session.prepare`  的 2PC 方法。

所谓的 **自动刷新** 行为，用于上面的列表中的行为，可以通过构造使用  :paramref:`.Session.autoflush`  参数为 ` `False`` 的   :class:`.Session`  或   :class:` .sessionmaker`  来禁用::

    Session = sessionmaker(autoflush=False)

此外，可在使用   :class:`.Session`  时使用  :attr:` .Session.no_autoflush`  上下文管理器临时禁用自动刷新::

    with mysession.no_autoflush:
        mysession.add(some_object)
        mysession.flush()

需要重申的是，   :class:`.Session`  仅在以  :term:` DBAPI`  事务上下文执行 SQL 命令时才会发出刷新，只要 DBAPI 不在   :ref:`driver level autocommit <dbapi_autocommit>`  模式下。这意味着假设数据库连接在其事务设置中提供  :term:` 原子性` ，如果在刷新中有任何单个 DML 语句失败，则整个操作将被回滚。

当刷新失败时，为了继续使用同一   :class:`_orm.Session` ，必须在刷新失败后显式调用  :meth:` ~.Session.rollback` ，即使底层事务已经回滚了（即使数据库驱动程序在技术上处于驱动程序级别的自动提交模式）。这是为了始终保持所谓的 “子事务” 嵌套模式的一致性。FAQ 节   :ref:`faq_session_rollback`  中包含有关此行为的更详细说明。

.. seealso::

      :ref:`faq_session_rollback`  —— 关于刷新失败时为什么必须调用  :meth:` _orm.Session.rollback`  的进一步背景信息。

.. _session_get:

按主键获取
~~~~~~~~~~~~~~~~~~

由于   :class:`_orm.Session`  使用  :term:` identity map`  引用当前内存中的对象和主键，因此提供了用于按主键定位对象的  :meth:`_orm.Session.get`  方法，首先在当前身份映射中查找对象，然后在数据库中查询缺失数据。例如，要定位主键标识符为 ` `(5,)`` 的 ``User`` 实体::

    my_user = session.get(User, 5)

  :meth:`_orm.Session.get`   还包括调用形式，用于传递组合主键值，这些值可以作为元组或字典传递，以及允许特定的加载程序和执行选项的附加参数。有关完整的参数列表，请参见  :meth:` _orm.Session.get` 。

.. seealso::

     :meth:`_orm.Session.get` 

.. _session_expiring:

过期/刷新
~~~~~~~~~~~~~~~~~~~~~

使用   :class:`_orm.Session`  时，一个重要的考虑因素通常是处理从数据库加载的对象的状态，以保持它们与当前事务的最新状态同步。SQLAlchemy ORM 基于一个  :term:` identity map`  的概念，因此，当从 SQL 查询中“加载”一个对象时，将维护对应于特定数据库标识符的唯一 Python 对象实例。这意味着，如果我们发出两个分开的查询，每个查询，针对同一行，得到一个映射对象，那么这两个查询将返回同一个 Python 对象::

  >>> u1 = session.scalars(select(User).where(User.id == 5)).one()
  >>> u2 = session.scalars(select(User).where(User.id == 5)).one()
  >>> u1 is u2
  True

由此引申出来，当 ORM 从查询中获得行时，它将跳过为已经加载的对象进行属性复制。这里设计的假设是假设事务是完全隔离的，然后到某种程度上事务不是隔离的，应用程序可以根据需要采取步骤刷新对象从数据库事务。FAQ 链接   :ref:`faq_session_identity`  更详细地讨论了这个概念。

当 ORM 映射对象被加载到内存中时，有三种一般方法可以使用当前事务的新数据刷新其内容：

* **过期() 方法** ——  :meth:`_orm.Session.expire`  方法可擦除对象的选择性或所有属性的内容，以便在下一次访问它们时从数据库中加载，例如，使用  :term:` lazy loading`  模式::

    session.expire(u1)
    u1.some_attribute  # <-- 从事务中进行的懒加载

  ..

* **刷新() 方法** ——  :meth:`_orm.Session.refresh`  方法密切相关，它表现得和  :meth:` _orm.Session.expire`  方法相同，但还会立即发出一次或多次 SQL 查询，以实际刷新对象的内容::

    session.refresh(u1)  # <-- 发出 SQL 查询
    u1.some_attribute  # <-- 从事务中进行刷新

  ..

* **populate_existing() 方法或执行选项** —— 这是一个在   :ref:`orm_queryguide_populate_existing`  中记录的执行选项；在遗留形式中，它在   :class:` _orm.Query`  对象上被称为  :meth:`_orm.Query.populate_existing`  方法。在任一形式中，这个操作指示返回查询的对象应该从它们在数据库中的内容中进行不可条件地的重新生成::

    u2 = session.scalars(
        select(User).where(User.id == 5).execution_options(populate_existing=True)
    ).one()

  ..

有关刷新/过期概念的更多讨论可以在   :ref:`session_expire`  找到。

.. seealso::

    :ref:`session_expire` 

    :ref:`faq_session_identity` 

使用任意 WHERE 子句进行 UPDATE 和 DELETE
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0 包括增强功能，以发出几种启用 ORM 的 INSERT、UPDATE 和 DELETE 语句。请参见  :doc:`queryguide/dml`  中的文档。

.. seealso::

     :doc:`queryguide/dml` 

      :ref:`orm_queryguide_update_delete_where` 


.. _session_autobegin:

自动启动
~~~~~~~~~~

  :class:`_orm.Session`  对象具有被称为 **autobegin** 的行为。这表示   :class:` _orm.Session`  在内部将自身视为“事务”状态，只要使用   :class:`_orm.Session`  执行任何工作，无论是涉及到修改   :class:` _orm.Session`  的内部状态与对象状态更改的情况，还是涉及到需要数据库连接的操作。

当首次构造   :class:`_orm.Session`  时，不存在事务状态。当调用像  :meth:` _orm.Session.add`  或  :meth:`_orm.Session.execute`  这样的方法时，或者类似地，如果执行查询以返回结果（最终使用  :meth:` _orm.Session.execute` ），或者修改了  :term:`persistent`  对象上的属性，那么会自动开始处理事务状态。

可以通过访问  :meth:`_orm.Session.in_transaction`  方法来检查   :class:` _orm.Session`  的事务状态，它返回 ``True`` 或 ``False``，指示“autobegin”步骤是否已经进行。虽然通常不需要，但  :meth:`_orm.Session.get_transaction`  方法将返回表示此事务状态的实际   :class:` _orm.SessionTransaction`  对象。

也可以通过调用  :meth:`_orm.Session.begin`  显式启动   :class:` _orm.Session`  的事务状态。当调用此方法时，无条件地将   :class:`_orm.Session`  置于“事务”状态。  :meth:` _orm.Session.begin`  可以按照   :ref:`session_begin_commit_rollback_block`  中所述的方式作为上下文管理器使用。

.. _session_autobegin_disable:

禁用 Autobegin 以防止隐式事务
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以使用  :paramref:`_orm.Session.autobegin`  参数将“autobegin”行为设置为 ` `False`` 以禁用它。使用此参数，   :class:`_orm.Session`  将要求必须显式调用  :meth:` _orm.Session.begin`  才能使用   :class:`_orm.Session` 。在构造后，在调用任何  :meth:` _orm.Session.rollback` 、  :meth:`_orm.Session.commit`   或  :meth:` _orm.Session.close`  方法后，   :class:`_orm.Session`  将不会在隐式地开始任何新的事务；如果在不首先调用  :meth:` _orm.Session.begin`  的情况下尝试使用   :class:`_orm.Session` ，则会引发错误::

    with Session(engine, autobegin=False) as session:
        session.begin()  # <-- 如有必要，否则下一次调用将抛出 InvalidRequestError

        session.add(User(name="u1"))
        session.commit()

        session.begin()  # <-- 如有必要，否则下一次调用将抛出InvalidRequestError

        u1 = session.scalar(select(User).filter_by(name="u1"))

.. versionadded:: 2.0 添加了  :paramref:`_orm.Session.autobegin` ，允许禁用“autobegin”行为

.. _session_committing:

提交
~~~~~~~~

  :meth:`~.Session.commit`   用于提交当前事务。从本质上讲，这表示在事务性方法（例如  :meth:` .Session.commit`  和  :meth:`.Session.begin_nested` ）发出之前在内部发出 ` COMMIT` 语句。同时也会在  :meth:`.Session.commit`  之前进一步持久化任何待处理的 SQL 操作。

以下是一个简单的例子::

    from sqlalchemy.orm import Session

    with Session(engine) as session:
        user = User("ed", "Ed Jones", "edspassword")
        session.add(user)

        session.commit()

.. seealso::

     :meth:`_orm.Session.rollback` 

     :meth:`_orm.Session.begin_nested` 


.. _session_close:

关闭
~~~~~~~~

  :class:`.Session`  对象被设计为不可重用。也就是说，在调用  :meth:` .Session.commit` 、  :meth:`.Session.rollback`   或  :meth:` .Session.close`  之一之后，必须丢弃对象并创建新对象，才能继续执行新的数据库操作。

.. seealso::

     :meth:`_orm.Session.commit` 

    :meth:`_orm.Session.rollback` 所有当前拥有事务的数据库连接；
从  :term:`DBAPI`  的视角来看，这意味着会在每个 DBAPI 的连接上调用 ` `connection.commit()`` 方法。

当   :class:`.Session`  中没有进行事务，表示自上一次调用  :meth:` .Session.commit`  以来，没有在此   :class:`.Session`  上执行任何操作时，该方法将开始并提交一个仅在内部使用的“逻辑”事务，这通常不会对数据库产生影响，除非检测到挂起的刷新更改，但仍将调用事件处理程序和对象过期规则。

  :meth:`_orm.Session.commit`   操作始终在发出 COMMIT 命令之前无条件地发出  :meth:` ~.Session.flush` 。如果没有检测到任何挂起的更改，则不会向数据库发出 SQL。此行为不可配置，并且不受  :paramref:`.Session.autoflush`  参数的影响。

在此之后，如果存在实际的数据库事务，  :meth:`_orm.Session.commit`   将提交这些事务。

最后，在事务结束时，   :class:`_orm.Session`  中的所有对象都将被  :term:` 过期` 。这样，当实例下次被访问时，无论是通过属性访问还是通过它们出现在 SELECT 的结果中，它们都会接收到最新的状态。此行为可以通过  :paramref:`_orm.Session.expire_on_commit`  参数控制，如果不希望发生该行为，则可以将其设置为 ` `False``。

.. seealso::

      :ref:`session_autobegin` 

.. _session_rollback:

回滚
~~~~~~

  :meth:`~.Session.rollback`   会回滚当前事务（如果有）。如果没有正在进行的事务，则该方法会静默传递。

对于默认配置的会话，
在既定的事务开始通过   :ref:`autobegin <session_autobegin>`  方法时，
Session 在回滚后的状态如下：

* 所有事务都将被回滚，所有连接都将返回到连接池中，除非 Session 直接绑定到连接，否则连接仍然保持（但仍会回滚）。
* 在事务生命周期内最初处于  :term:`待定`  状态的对象将被清除，与将其 INSERT 语句回滚相对应。它们的属性状态保持不变。
* 在事务生命周期中标记为  :term:`删除`  的对象将被提升回  :term:` 持久性`  状态，与将它们的 DELETE 语句回滚相对应。请注意，如果这些对象在事务中首先处于  :term:`待定`  状态，则优先级较高。
* 所有未被清除的对象都将完全过期-这与  :paramref:`_orm.Session.expire_on_commit`  设置无关。

了解了这种状态之后，   :class:`_orm.Session`  可以在回滚发生后安全地继续使用。

.. versionchanged:: 1.4

      :class:`_orm.Session`  对象现在具有延迟“begin”行为，如   :ref:` autobegin <session_autobegin>`  中所述。如果没有启动事务，
    如  :meth:`_orm.Session.commit`  和  :meth:` _orm.Session.rollback`  方法将没有效果。
    在非自动提交模式下，这种行为在 1.4 之前不会被观察到，因为事务总是隐式存在。

当  :meth:`_orm.Session.flush`  失败时，通常是由于违反主键、外键或“非 null”约束等原因，将自动发出 ROLLBACK（目前不可能在部分故障后继续刷新）。但是，在此时，   :class:` _orm.Session`  进入一种称为“非活动”的状态，调用应用程序必须始终显式调用  :meth:`_orm.Session.rollback`  方法，以便   :class:` _orm.Session`  可以回到可用状态（也可以简单地关闭和丢弃）。如需详细的讨论，请参见   :ref:`faq_session_rollback`  中的 FAQ。

.. seealso::

    :ref:`session_autobegin` 

.. _session_closing:

关闭
~~~~~~~

  :meth:`~.Session.close`   方法会发出  :meth:` ~.Session.expunge_all` ，从会话中删除所有 ORM 映射对象，并将所有事务性 / 连接资源从固定到它的   :class:`_engine.Engine`  对象中释放。当将连接返回到连接池时，事务状态也会回滚。

当   :class:`_orm.Session`  关闭时，它本质上与初始构造时相同的状态，并且**可以再次使用**。在这方面，  :meth:` _orm.Session.close`  方法更像是一种“重置”回到清洁状态的方法，而不是像一种“数据库关闭”方法。

建议在   :class:`_orm.Session`  的作用域内限制  :meth:` _orm.Session.close`  的调用，特别是如果没有使用  :meth:`_orm.Session.commit`  或  :meth:` _orm.Session.rollback`  方法。   :class:`_orm.Session`  可以作为上下文管理器使用，以确保调用  :meth:` _orm.Session.close` ::

    with Session(engine) as session:
        result = session.execute(select(User))

    # 自动关闭会话

.. versionchanged:: 1.4

      :class:`_orm.Session`  对象具有延迟“begin”行为，如   :ref:` autobegin <session_autobegin>`  中所述。
    在调用  :meth:`_orm.Session.close`  方法后不再立即开始新事务。

.. _session_faq:

会话常见问答
---------------------------

到这个时候，许多用户已经对会话有疑问了。本节提供一个迷你 FAQ（注意我们也有  :doc:`完整的 FAQ </faq/index>` ），涵盖了使用   :class:` .Session`  时最基本的问题。

何时创建   :class:`.sessionmaker` ？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

只要在您应用程序的全局范围内的某个地方一次。它应该被视为您应用程序的配置的一部分。如果您的应用程序在一个包中有三个 .py 文件，则例如，您可以将   :class:`.sessionmaker`  放在您的 ` `__init__.py`` 文件中；从那时起，其他模块可以说“from mypackage import Session”。这样，其他人只使用  :class:`.Session()` ，而该会话的配置由该中心点控制。

如果应用程序启动，执行导入操作，但不知道将要连接到哪个数据库，则可以在稍后使用  :meth:`.sessionmaker.configure`  将   :class:` .Session`  在“类”级别绑定到引擎。

在本节中的示例中，我们经常会展示   :class:`.sessionmaker`  在我们实际上调用   :class:` .Session`  之前创建。但这只是为了举例说明！实际上，  :class:`.sessionmaker`  将出现在模块级别的某个地方。由此生成   :class:` .Session`  的调用将放置在应用程序开始进行其工作的地方。

.. _session_faq_whentocreate:

何时构建   :class:`.Session` ，何时提交，何时关闭？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. topic:: 简而言之;

    1. 通常情况下，应将会话的生命周期与访问和/或操作数据库数据的函数和对象**分开**。这将极大地有助于实现可预测且一致的事务范围。

    2. 确保您清楚地知道何时开始和结束事务，并使事务**短暂**，即在一系列操作结束时结束，而不是无限期地保持打开状态。

通常会在可能预期访问数据库的逻辑操作开始时构建   :class:`.Session` 。

  :class:`.Session`  每次与数据库通信时，都会立即开始数据库事务。该事务将保持在   :class:` .Session`  回滚，提交或关闭时，如无正在进行的事务。如果再次使用   :class:`.Session` ，则会开始新的事务，而非在前一个事务结束后。从这个角度来看，   :class:` .Session`  可以跨越多个事务的生命周期，但一次只能有一个。我们将这两个概念称为**事务范围**和**会话范围**。

通常并不难确定在何处开始和结束   :class:`.Session`  的范围，尽管可能会引入各种应用程序结构。

其中一些示例场景包括：

* Web 应用程序。在这种情况下，最好使用所用 Web 框架提供的 SQLAlchemy 集成。否则，基本模式是在 Web 请求开始时创建   :class:`_orm.Session` ，在执行 POST、PUT 或 DELETE 的 Web 请求结束时调用  :meth:` _orm.Session.commit`  方法，然后在 Web 请求结束时关闭会话。通常，最好将  :paramref:`_orm.Session.expire_on_commit`  设置为 False，这样前端代码中从   :class:` _orm.Session`  返回的对象不需要发出新的 SQL 查询以刷新对象，如果事务已经提交。

* 后台守护程序生成子进程。在这种情况下，最好为每个子进程创建一个本地的   :class:`.Session` ，在处理的“作业”生命周期内使用该会话，然后在完成作业时将其关闭。

* 对于命令行脚本，应用程序将创建一个单独的全局   :class:`.Session` ，该 Session 在程序开始执行其工作时建立，并在程序完成其任务时立即提交它。

* 对于基于 GUI 界面的应用程序，  :class:`.Session`  的作用范围可能最好位于用户生成的事件的范围内，例如按钮推送。或者，作用范围可能对应于明确的用户交互，例如用户"打开"一系列记录，然后将其“保存”。

作为一般规则，应用程序应将会话的生命周期与特定数据处理函数和方法之外的数据分开*外部*来处理。这是一个基本的问题分离，使得数据特定操作能够获得对访问和操纵该数据的上下文的暴露。

例如，**不要这样做**：：

    ### 这是**错误的** 范例 ###


    class ThingOne:
        def go(self):
            session = Session()
            try:
                session.execute(update(FooBar).values(x=5))
                session.commit()
            except:
                session.rollback()
                raise


    class ThingTwo:
        def go(self):
            session = Session()
            try:
                session.execute(update(Widget).values(q=18))
                session.commit()
            except:
                session.rollback()
                raise


    def run_my_program():
        ThingOne().go()
        ThingTwo().go()

将会话（和通常是事务）的生命周期**分开**对于使用特定数据的操作是不太可能的。下面的示例说明了这可能如何看起来，同时还使用 Python 上下文管理器（即``with:`` 关键字）来自动处理   :class:`_orm.Session`  和其事务的范围：：

    ### 这是一个更好但不是唯一的方式 ###


    class ThingOne:
        def go(self, session):
            session.execute(update(FooBar).values(x=5))


    class ThingTwo:
        def go(self, session):
            session.execute(update(Widget).values(q=18))


    def run_my_program():
        with Session() as session:
            with session.begin():
                ThingOne().go(session)
                ThingTwo().go(session)

.. versionchanged:: 1.4   :class:`_orm.Session`  可以作为上下文管理器使用，无需使用外部帮助器函数。

Session 是缓存吗？
~~~~~~~~~~~~~~~~~~~~~~~

有点是，因为它实现了  :term:`标识映射` (identity map) 模式，并存储按其主键编为键的对象。但是，它不会缓存任何查询。这意味着，即使 ` `Foo(name='bar')`` 在唯一映射中，当您说``session.scalars(select(Foo).filter_by(name='bar'))`` 时，Session 也不知道。它不得不向数据库发送 SQL，获取行，然后当它看到行中的主键时，*然后*才可以在本地标识映射中查找该对象。只有在您说 ``query.get({some primary key})`` 时，   :class:`~sqlalchemy.orm.session.Session`  才不必发出查询。

此外，默认情况下，   :class:`.Session`  使用弱引用存储对象实例。这也破坏了将 Session 用作缓存的目的。

  :class:`.Session`  不是设计成每个人都可以作为“注册表”查询的全局对象。这更多地是第二级缓存的工作。SQLAlchemy 提供了一种使用 ` dogpile.cache <https://dogpilecache.readthedocs.io/>`_ 实现第二级缓存的方式，通过   :ref:`examples_caching`  示例。

如何获取某个对象的   :class:`~sqlalchemy.orm.session.Session` ？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用  :meth:`~.Session.object_session`  类方法，该方法可用于  :class:` ~sqlalchemy.orm.session.Session` ::

    session = Session.object_session(someobject)

较新的   :ref:`core_inspection_toplevel`  系统也可用于获取会话::

    from sqlalchemy import inspect

    session = inspect(someobject).session

.. _session_faq_threadsafe:

Session 是线程安全的吗？在并发任务中共享 AsyncSession 是安全的吗？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`.Session`  是表示单个数据库事务的**可变状态对象**。因此，**单个   :class:` .Session`  实例不能在并发线程或 asyncio 任务之间共享，除非小心同步**。  :class:`.Session`  旨在以**非并发**方式使用，即特定的   :class:` .Session`  实例应仅在一个线程或任务中使用。

在使用 SQLAlchemy 的   :ref:`asyncio_toplevel`  扩展中的   :class:` _asyncio.AsyncSession`  对象时，该对象仅是在   :class:`_orm.Session`  顶部的一个薄代理，因此相同的规则适用;它是一个**不同步，可变的有状态对象**，因此一个单独的   :class:` _asyncio.AsyncSession`  实例不能方便地在多个 asyncio 任务中使用。

  :class:`.Session`  或   :class:` _asyncio.AsyncSession`  的一个实例代表一个单一的逻辑数据库事务，只引用其绑定到的   :class:`_engine.Engine`  或   :class:` .AsyncEngine`  的一个   :class:`_engine.Connection` 。请注意，这些对象均支持绑定到多个引擎，但在事务范围内，仍然每个引擎仅有一个连接处于活动状态。

在事务中的数据库连接也是一个有状态的对象，它旨在以非并发序列方式进行操作。命令按一定的顺序在连接上发出，这些命令由数据库服务器按其发出的确切顺序处理。当   :class:`_orm.Session`  发出命令并接收结果时，它本身正在通过内部状态转换，与此连接上的命令和数据状态相一致。这些状态包括是否启动、提交或回滚了事务，任何正在发挥作用的 SAVEPOINT，以及将数据库行状态与本地 ORM 映射的对象进行微调粒度的同步。

在设计并发性数据库应用程序时，应采用每个并发任务/线程使用自己的数据库事务的适当模型。因此，当讨论数据库并发问题时，使用的标准术语是**多个并发事务**。在传统的 RDMS 中，不存在接收和处理多个并发命令的单个数据库事务的类似物。

因此，SQLAlchemy 的   :class:`_orm.Session`  和   :class:` _asyncio.AsyncSession`  的并发模型为**每个线程一个 Session，每个任务一个 AsyncSession**。使用多个线程或在 asyncio 中使用多个任务（例如，使用诸如 ``asyncio.gather()``的 API）的应用程序将确保每个线程都有自己的   :class:`_orm.Session` ，每个 asyncio 任务都有自己的   :class:` _asyncio.AsyncSession` 。

确保使用本地范围内的标准上下文管理器模式在 Python 函数或任务的顶级内部管理   :class:`_orm.Session`  或   :class:` _asyncio.AsyncSession`  的范围，将确保   :class:`_orm.Session`  或   :class:` _asyncio.AsyncSession`  的生命周期在本地范围内维护。

对于应用程序从全局方面受益的全局   :class:`.Session` ，其中将 Session 对象传递给特定的函数和方法的选项不是一个选择，   :class:` .scoped_session` .Session` 对象；请参见   :ref:`unitofwork_contextual`  一节对此进行了说明。在 asyncio 上下文中，   :class:` .async_scoped_session`  对象是   :class:`.scoped_session`  的 asyncio 对应物，但更难以配置，因为它需要一个自定义的“上下文”函数。