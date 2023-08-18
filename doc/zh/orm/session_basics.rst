==============
会话基础知识
==============


会话（Session）会做些什么？
--------------------------

从最普遍的角度来看，:class:`~.Session` 与数据库建立会话，并在其生命周期中为您加载或关联的所有对象提供“保管区域”。提供了一个接口，SELECT和其他查询将通过它返回和修改ORM映射的对象。ORM对象本身在:class:`.Session`, 一种称为 :term:`identity map` 的结构中维护，它维护每个对象的唯一副本，其中“唯一”指“只有具有特定主键的一个对象”。

:class:`.Session` 开始时通常处于大多数情况下无状态的形式。 一旦发出查询或将其他对象与其保存，它会从与:class:`.Session` 关联的:class:`_engine.Engine` 中请求连接资源，然后在该连接上建立一个事务。这个事务一直有效，直到:class:`.Session` 被要求提交或回滚事务。

:class:`_orm.Session`维护的ORM对象称为 :term:`instrumented`，这意味着在Python程序中修改属性或集合时，会生成一个更改事件，该事件由 :class:`_orm.Session` 记录。在将要查询数据库或将要提交事务时，:class:`_orm.Session`首先将所有待处理的更改刷新到数据库中，这称为 :term:`unit of work`模式。

在使用 :class:`.Session` 时，将其维护的ORM映射对象视为与本地事务相关的数据库行的 **代理对象**。 为了保持对象上的状态与实际在数据库中的状态相匹配，需要考虑各种事件，以便将其重新访问数据库以保持同步。 可以将对象“分离”到 :class:`.Session` 并继续使用它们，尽管这种做法有其警告。通常，当您再次使用这些对象时，您将使用另一个 :class:`.Session` 将分离的对象重新关联，以便它们可以恢复其表示数据库状态的正常任务。

.. _session_basics:

使用会话的基础知识
-------------------------

这里介绍了最基本的:class:`.Session`使用模式。

.. _session_getting:

打开和关闭会话
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`_orm.Session` 可以独立构建或使用 :class:`_orm.sessionmaker` 类。它通常会一开始传递一个单一的:class:`_engine.Engine`，作为连接资源的源。一个典型的用法可能是::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import Session

    # 引擎，会与该Session用于连接
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

    # 创建会话并添加对象
    with Session(engine) as session:
        session.add(some_object)
        session.add(some_other_object)
        session.commit()

上面，使用与特定数据库URL关联的:class:`_engine.Engine` 实例化了 :class:`_orm.Session`。然后，在使用Python上下文管理器（即 ``with:`` 语句）时，会自动在该块结束时关闭该实例；这与调用 :meth:`_orm.Session.close` 方法相同。

调用 :meth:`_orm.Session.commit` 是可选的，只有当我们使用 :class:`_orm.Session` 执行新数据以将其持久化到数据库中时才需要调用该方法。如果我们只发出SELECT调用，并且不需要编写任何更改，则对 :meth:`_orm.Session.commit` 的调用将是不必要的。

.. note::

    请注意，在调用 :meth:`_orm.Session.commit` 之后，无论是显式调用还是使用上下文管理器，与 :class:`.Session`关联的所有对象都会 :term:`过期`，这意味着它们的内容已被删除以在下一个事务中重新加载。如果这些对象是 :term:`分离`的，则除非使用 :paramref:`.Session.expire_on_commit` 参数禁用此行为，否则它们将无效，直到使用新的 :class:`.Session` 重新关联它们。有关更多详细信息，请参阅 :ref:`session_committing`。

.. _session_begin_commit_rollback_block:

编写起始 / 提交 / 回滚块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们还可以将:meth:`_orm.Session.commit` 调用和事务的整体“框架”包含在上下文管理器内，以便在将数据提交到数据库时执行回滚操作。"框架"指的是如果所有操作都成功，则将调用 :meth:`_orm.Session.commit` 方法，但如果引发任何异常，则将调用 :meth:`_orm.Session.rollback`方法，以便立即回滚事务，然后向外传播异常。在Python中，最根本的表达方式是使用“try: / except: / else:”块，例如::

    # 上下文管理器的详细版本将执行
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

上面示例中详细说明的操作序列可以通过使用 :meth:`_orm.Session.begin`
方法返回的 :class:`_orm.SessionTransaction` 对象更简洁地实现，该对象为相同序列提供了上下文管理器接口::

    # 创建会话并添加对象
    with Session(engine) as session:
        with session.begin():
            session.add(some_object)
            session.add(some_other_object)
        # inner context calls session.commit(), if there were no exceptions
    # outer context calls session.close()

更简洁的是，可以将两个上下文结合使用::

    # 创建会话并添加对象
    with Session(engine) as session, session.begin():
        session.add(some_object)
        session.add(some_other_object)
    # inner context calls session.commit(), if there were no exceptions
    # outer context calls session.close()

使用sessionmaker
~~~~~~~~~~~~~~~~~~~~

:class:`_orm.sessionmaker` 的目的是为具有固定配置的:class:`_orm.Session`对象提供工厂。由于一个应用程序通常会在模块范围内拥有一个:class:`_engine.Engine`对象，因此:class:`_orm.sessionmaker` 可以为此引擎提供:class:`_orm.Session` 对象的工厂::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    # 引擎，会与该Session用于连接
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

    # 一个sessionmaker()，
    # 也在与引擎相同的作用域
    Session = sessionmaker(engine)

    # 现在我们可以构造一个Session()，
    # 无需每次传递engine
    with Session() as session:
        session.add(some_object)
        session.add(some_other_object)
        session.commit()
    # 关闭会话

:class:`_orm.sessionmaker` 与 :class:`_engine.Engine` 一样，在模块级别或全局范围内进行工厂设置，因此它还有自己的:meth:`_orm.sessionmaker.begin`方法，
类似于 :meth:`_engine.Engine.begin`，它返回一个:class:`_orm.Session`对象，并同时保留一对
begin/commit/rollback块::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker

    # 引擎，会与该Session用于连接
    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

    # 一个sessionmaker()，
    # 也在与引擎相同的作用域
    Session = sessionmaker(engine)

    # 我们现在可以构造一个Session()，
    # 包括begin()/commit()/rollback()
    with Session.begin() as session:
        session.add(some_object)
        session.add(some_other_object)
    # 提交事务，关闭会话

在上面示例中，当上面的“with:”块结束时，:class:`_orm.Session`将会进行提交，并关闭:class:`_orm.Session`实例。

写应用程序时，:class:`.sessionmaker` 工厂应与通过 :func:`_sa.create_engine` 创建的:class:`_engine.Engine` 对象的范围相同，该对象通常在模块级别或全局范围内。由于这些对象都是工厂，因此可以同时由任意数量的函数和线程使用。

.. seealso::

    :class:`_orm.sessionmaker`

    :class:`_orm.Session`


.. _session_querying_20:

查询
~~~~~~~~

查询的主要方法是使用 :func:`_sql.select` 构建来创建 :class:`_sql.Select`对象，然后使用 :meth:`_orm.Session.execute`和:meth:`_orm.Session.scalars`等方法执行它以返回结果。结果随后以:class:`_result.Result`对象返回，包括子变体，例如 :class:`_result.ScalarResult`。

有关SQLAlchemy ORM查询的完整指南，可在 :ref:`queryguide_toplevel`中找到。以下是一些简短的示例::

    from sqlalchemy import select
    from sqlalchemy.orm import Session

    with Session(engine) as session:
        # 查询“User”对象
        statement = select(User).filter_by(name="ed")

        # “User”对象列表
        user_obj = session.scalars(statement).all()

        # 查询特定列
        statement = select(User.name, User.fullname)

        # Row对象列表
        rows = session.execute(statement).all()

.. versionchanged:: 2.0

    2.0式查询现在是标准。有关从1.x系列迁移的信息，请参见 :ref:`migration_20_query_usage`。

.. seealso::

   :ref:`queryguide_toplevel`

.. _session_adding:


添加新对象或现有对象
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:meth:`~.Session.add` 用于将实例放置在: class :`中。Session。对于 :term:`transient`（即全新）实例，这将导致在下一个刷新时对这些实例进行INSERT。对于 :term:`persistent`（即由此会话加载的）实例，它们已经存在并且不需要被添加。可以使用此方法将 :term:`detached`（即已从会话中删除的）实例重新关联到会话中::

    user1 = User(name="user1")
    user2 = User(name="user2")
    session.add(user1)
    session.add(user2)

    session.commit()  # 将更改写入数据库

要一次将项目列表添加到会话中，请使用 :meth:`~.Session.add_all`::

    session.add_all([item1, item2, item3])

向 :meth:`~.Session.add` 操作 *级联* 到 "save-update" 级联级别。有关详细信息，请参见 :ref:`unitofwork_cascades` 部分。

.. _session_deleting:

删除
~~~~~~~~

:meth:`~.Session.delete` 方法将实例放置在会话的“待删除对象”列表中::

    # 标记将要删除的两个对象
    session.delete(obj1)
    session.delete(obj2)

    # 提交（或刷新）
    session.commit()

:meth:`_orm.Session.delete` 操作标记要删除的对象，将为每个受影响的主键发出DELETE语句。在待删除的对象被刷新之前，这些对象在 :attr:`_orm.Session.deleted` 集合中存在。在DELETE之后，它们会从 :class:`_orm.Session` 中清除，一旦提交了事务，它将永久存在。

与删除对象相关的各种重要行为，特别是如何处理到其他对象和集合的 关系。有关如何处理此项工作的更多信息，请参见 :ref:`unitofwork_cascades` 部分，但一般规则如下：

* 使用 :func:`_orm.relationship` 指令将映射对象与已删除对象相关联的行默认情况下**不会**被删除。如果那些对象有一个指向要删除的行的外键约束，那么这些列将设置为NULL。如果这些列是不可为空的，则会导致约束违规。

* 要将“SET NULL”更改为删除相关对象的行，请在 :func:`_orm.relationship` 上使用 :ref:`cascade_delete` 级联。

* 通过 :paramref:`_orm.relationship.secondary` 参数链接为“many-to-many”的表的行，在引用对象被删除时**将被删除**。

* 当相关对象包含指向正在删除的对象的外键约束，并且这些对象所属的相关集合未加载到内存中时，工作单元将发出一个SELECT以获取所有相关行，以使其主键值可以用于在这些相关行上发出UPDATE或DELETE语句。通过这种方式，ORM将在无需进一步指令的情况下执行ON DELETE CASCADE 的功能，即使在核心 :class:`_schema.ForeignKeyConstraint` 对象上进行了配置。

* :paramref:`_orm.relationship.passive_deletes` 可用于调整此行为并更自然地依赖于“ON DELETE CASCADE”；当设置为True时，此SELECT操作将不再进行，但是本地存在的行仍将被显式设置为SET NULL或DELETE。将 :paramref:`_orm.relationship.passive_deletes` 设置为字符串 ``"all"`` 将禁用 **所有** 相关对象update/delete。

* 当删除标记为删除的对象时，不会自动将其从引用它的集合或对象的集合中删除。当过期 :class:`_orm.Session` 时，这些集合可以重载，以便对象不再存在。但是，与其使用 :meth:`_orm.Session.delete` 删除这些对象，还可以将对象从其集合中删除，然后使用 :ref:`cascade_delete_orphan`，以使它作为集合删除的次要影响而被删除。有关详细信息，请参阅 :ref:`session_deleting_from_collections`。

.. seealso::

    :ref:`cascade_delete` - 描述“删除级联”，其中将标记与引导对象相关的相关对象进行删除当引导对象被删除时。

    :ref:`cascade_delete_orphan` - 描述“删除孤儿级联”，其中将标记与引导对象相关的相关对象进行删除当它们与引导对象的关系被解除时

    :ref:`session_deleting_from_collections` - 关于在内存中刷新关系的重要背景信息

.. _session_flushing:

刷新
~~~~~~~~

当使用其默认配置的 :class:`~sqlalchemy.orm.session.Session` 时，刷新步骤几乎总是自动完成的。具体来说，它在发出任何单个SQL语句之前以及在 :class:`_query.Query` 或 :term:`2.0 风格` 的 :meth:`_orm.Session.execute` 调用等操作中自动刷新. 在 :meth:`.Session.commit` 调用之前的 :meth:`.Session.flush` 调用之前，它也在其中发生。当使用 :meth:`.Session.begin_nested` 时发出SAVEPOINT时，也会发生刷新。

可以随时使用 :meth:`.Session.flush` 方法强制进行 :class:`.Session` 刷新::

    session.flush()

仅结果的刷新称为 **自动刷新**。在进行ORM启用的SQL构造（例如引用ORM实体和/或ORM映射属性的 :func:`_sql.select` 对象）的方法包括，:meth:`_orm.Session.execute` 等执行SQL的方法以及在:meth:`_orm.Session.commit` 调用之前的方法 。在需要数据库连接的操作之后，也会自动引发自动刷新，例如在 :term:`persistent` 对象上修改属性。

可以通过构造传递 :paramref:`.Session.autoflush` 参数为 ``False`` 的 :class:`.Session` 或 :class:`.sessionmaker` 来禁用自动刷新。通过使用此参数，:class:`.Session` 将要求要先显式调用 :meth:`.Session.begin`， 才能使用 :class:`.Session`，在构建时，并在调用 :meth:`_orm.Session.rollback`、 :meth:`_orm.Session.commit` 或 :meth:`_orm.Session.close` 方法之后，该 :class:`.Session` 不会自动开始任何新事务，并在未首先调用:meth:`_orm.Session.begin` 的情况下尝试使用 :class:`.Session`时引发错误，并且不会暗示事务从而需要您首先调用 :meth:`_orm.Session.begin` 而不是使用自动设置。

.. versionadded:: 2.0 新增 :paramref:`_orm.Session.autobegin`，允许禁用“autobegin”行为

.. seealso::

    :ref:`faq_session_rollback` - 更多关于在刷新失败时为何必须调用:meth:`_orm.Session.rollback` 的背景信息

.. _session_get:

按主键检索
~~~~~~~~~~~~~~~~~~

由于 :class:`_orm.Session` 使用 :term:`identity map` 指向当前由主键标识的内存对象，因此， :meth:`_orm.Session.get` 方法提供了一种按主键查找对象的方法，首先查找当前的identity map，然后查询数据库以获取不存在的值。例如，要查找主键标识为 ``(5,)`` 的 ``User`` 实体::

    my_user = session.get(User, 5)

:meth:`_orm.Session.get` 还包括调用表单，用于传递组合主键值，可以将它们作为元组或字典传递，以及允许特定的加载程序和执行选项的其他参数。请参见 :meth:`_orm.Session.get` 以获取完整的参数列表。

.. seealso::

    :meth:`_orm.Session.get`

.. _session_expiring:

过期/刷新
~~~~~~~~~~~~~~~~~~~~~

在使用 :class:`_orm.Session` 时，重要的考虑因素之一是如何处理从数据库加载的状态的问题，以使其保持与当前事务的状态同步。 SQLAlchemy ORM 基于 :term:`identity map` 概念，因此，当从SQL查询中“加载”对象时，会维护对应于特定数据库标识的唯一Python对象实例。这意味着，如果我们发出两个单独的查询，每个查询都针对相同的行并获得映射对象，则这两个查询将返回相同的Python对象::

  >>> u1 = session.scalars(select(User).where(User.id == 5)).one()
  >>> u2 = session.scalars(select(User).where(User.id == 5)).one()
  >>> u1 is u2
  True

由此衍生的，当ORM从查询获得行时，它将**跳过**对其属性的填充。这里的设计假设是假定事务是完全隔离的，然后在事务不隔离的程度上，应用程序可以根据需要采取措施刷新来自事务的对象。:ref:`faq_session_identity` 中的FAQ条目详细讨论了这个概念。

当ORM映射对象加载到内存中时，有三种常规方法可以使用当前事务中的新数据刷新其内容：

* **expire() 方法** - :meth:`_orm.Session.expire` 方法将选择或所有属性的内容擦除，因此当它们下一次访问时，将从数据库加载它们，例如使用 :term:`lazy loading` 模式::

    session.expire(u1)
    u1.some_attribute  # <-- 从事务中延迟加载

  ..

* **refresh() 方法** - 与此密切相关的是 :meth:`_orm.Session.refresh` 方法，它做了 :meth:`_orm.Session.expire` 方法所做的一切，但还会立即发出一个或多个SQL查询，以实际刷新对象的内容：

    session.refresh(u1)  # <-- 发出SQL查询
    u1.some_attribute  # <-- 从事务中刷新

  ..

* **populate_existing() 方法或执行选项** — 现在这是一个执行选项，记录在 :ref:`orm_queryguide_populate_existing` 中；在旧版本中，它可以在 :class:`_orm.Query` 对象上找到，作为 :meth:`_orm.Query.populate_existing` 方法。以任一形式表示，此操作指示应返回查询的对象应从其在数据库中的内容中不受条件地重新填充::

    u2 = session.scalars(
        select(User).where(User.id == 5).execution_options(populate_existing=True)
    ).one()

  ..

关于刷新/过期概念的进一步讨论可以在 :ref:`session_expire` 中找到。

.. seealso::

  :ref:`session_expire`

  :ref:`faq_session_identity`



带任意 WHERE 子句的 UPDATE 和 DELETE
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0 包括增强的功能，用于发出几个支持ORM的INSERT、UPDATE和DELETE语句。请参见:doc:`queryguide/dml` 文档。

.. seealso::

    :doc:`queryguide/dml`

    :ref:`orm_queryguide_update_delete_where`


.. _session_autobegin:

自动开始
~~~~~~~~~~

:class:`_orm.Session` 对象具有称为 **autobegin** 的行为。这表示 :class:`_orm.Session` 在内部将自己视为在执行关于对象状态更改的内部状态或需要数据库连接的操作时处于“事务”状态。

当首次构建 :class:`_orm.Session` 时，没有事务状态。当调用方法比如 :meth:`_orm.Session.add` 或 :meth:`_orm.Session.execute`，或类似情况下发生会对数据库产生连接请求的操作时，关于接下来的逻辑，:class:`_orm.Session` 内部将自动开始意味着会话直接处于“事务”状态。

可以通过访问 :meth:`_orm.Session.in_transaction` 方法来检查 :class:`_orm.Session` 是否通过“autobegin" 步骤进行了处理，该方法返回 ``True`` 或 ``False``，以指示“autobegin" 步骤是否已进行。虽然通常不需要，但 :meth:`_orm.Session.get_transaction` 方法将返回表示此事务状态的实际 :class:`_orm.SessionTransaction` 对象。

也可以通过调用服务 :meth:`_orm.Session.begin` 显式地启动 :class:`_orm.Session` 的事务状态。当调用此方法时，无条件地将 :class:`_orm.Session` 放入“transactional”状态。 :meth:`_orm.Session.begin` 方法可以使用如 :ref:`session_begin_commit_rollback_block` 中描述的上下文管理器。

.. _session_autobegin_disable:

禁用 Autobegin 以防止隐式事务
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

“autobegin”行为可以使用 :paramref:`.Session.autobegin` 设定为 ``False`` 来禁用。通过使用此参数，:class:`.Session` 将要求必须显式调用 :meth:`.Session.begin`， 才能在构建时，及在调用 :meth:`_orm.Session.rollback`, :meth:`_orm.Session.commit`, 或者 :meth:`_orm.Session.close` 方法之后使用 :class:`.Session`，否则，:class:`_orm.Session` 无法自动解决新事务，并在不使用 :meth:`_orm.Session.begin`的情况下尝试使用 :class:`.Session` 时，将引发错误::

    with Session(engine, autobegin=False) as session:
        session.begin()  # <-- 需要，否则无法在下一个调用上引发InvalidRequestError

        session.add(User(name="u1"))
        session.commit()

        session.begin()  # <-- 需要，否则无法在下一个调用上引发InvalidRequestError

        u1 = session.scalar(select(User).filter_by(name="u1"))

.. versionadded:: 2.0 新增 :paramref:`_orm.Session.autobegin`，允许禁用“autobegin”行为

.. _session_committing:

提交
~~~~~~~~~~

:meth:`~.Session.commit` 用于提交当前的事务。它的核心是指示在事务提交之前发出 ``COMMIT`` 的 语句。

所有当前存在事务的数据库连接；就 :term:`DBAPI` 来说，这意味着会在每个 DBAPI 连接上调用 ``connection.commit()`` 方法。

当 :class:`.Session` 中没有任何事务（表示自上次调用 :meth:`.Session.commit` 以来没有在此 :class:`.Session` 上调用任何操作）时，该方法会开始和提交一个仅内部使用的“逻辑”事务（如果未检测到挂起的 flush 更改，则通常不会影响数据库，但仍将调用事件处理程序和对象过期规则）。

:meth:`_orm.Session.commit` 操作在发出 COMMIT 之前无条件执行 :meth:`~.Session.flush`。如果未检测到挂起的更改，则不会向数据库发出任何 SQL。此行为不可配置，并且不受 :paramref:`.Session.autoflush` 参数的影响。

在那之后，:meth:`_orm.Session.commit` 将提交实际的数据库事务（如果存在）。

最后，由于事务关闭，:class:`_orm.Session` 中的所有对象都会过期。这是为了让实例在下次访问时（通过属性访问或者通过 SELECT 结果中存在时）接收到最新的状态。您可以通过 :paramref:`_orm.Session.expire_on_commit` 标志来控制此行为，当此行为不希望时，可以将其设置为 ``False``。

.. seealso::

    :ref:`session_autobegin`

.. _session_rollback:

回滚
~~~~~~

如果当前存在事务，则 :meth:`~.Session.rollback` 回滚该事务。当不存在事务时，该方法会默默地通过。

对于默认配置的会话，负责回滚的 :meth:`~.Session.rollback` 操作，在经由 :ref:`autobegin <session_autobegin>` 或由显式调用 :meth:`_orm.Session.begin` 方法开始事务后， :class:`.Session` 的回滚状态如下：

    * 开始回滚所有事务并返回到连接池中，除非 :class:`~sqlalchemy.orm.session.Session` 直接绑定到 Connection，此时会继续维护该连接（但是依然被回滚）。
    * 生命周期内添加到 :class:`~sqlalchemy.orm.session.Session` 等待处理的对象进入 :term:`expired` 状态，对应着其 INSERT 语句被回滚。它们的属性状态保持不变。
    * 生命周期内标记为 :term:`deleted` 的对象进入 :term:`persistent` 状态，对应着其 DELETE 语句被回滚。请注意，如果这些对象首先在事务进行时处于 :term:`pending` 状态，那么该操作优先级更高。
    * 所有未过期的对象都会过期——这与 :paramref:`_orm.Session.expire_on_commit` 设置无关。

了解这些状态后，:class:`_orm.Session` 可以在发生回滚后安全地继续使用。

.. versionchanged:: 1.4

    :class:`_orm.Session` 现在具有延迟“begin”行为，如 :ref:`autobegin <session_autobegin>` 中所述。如果未开始事务，则诸如 :meth:`_orm.Session.commit` 和 :meth:`_orm.Session.rollback` 之类的方法不起作用。在 1.4 之前，由于在非自动提交模式下，事务始终存在，因此不会观察到此行为。

当 :meth:`_orm.Session.flush` 失败时，通常是由于主键、外键或“非空”约束违规等原因，将自动发出 ROLLBACK（当前不可能在部分失败后继续刷新）。但此时 :class:`_orm.Session` 进入一种称为“非活动”的状态，调用方必须始终显式调用 :meth:`_orm.Session.rollback` 方法，以便 :class:`_orm.Session` 可以重新回到可用状态（也可以简单地关闭和丢弃）。有关详细讨论，请参见 :ref:`faq_session_rollback` 中的 FAQ 条目。

.. seealso::

  :ref:`session_autobegin`

.. _session_closing:

关闭
~~~~~~~

:meth:`~.Session.close` 方法会发出 :meth:`~.Session.expunge_all` 方法，该方法会从会话中删除所有 ORM 映射的对象，并将事务/连接资源从绑定的 :class:`_engine.Engine` 对象释放。当将连接返回到连接池时，也会回滚事务状态。

当关闭 :class:`_orm.Session` 时，它基本上处于首次构造时的状态，并且**可以再次使用**。在这个意义上，:meth:`_orm.Session.close` 方法更像一种“重置”到干净状态的方法，而不是“数据库关闭”方法。

建议在结束时通过调用 :meth:`_orm.Session.close` 来限制 :class:`_orm.Session` 的范围，特别是如果不使用 :meth:`_orm.Session.commit` 或 :meth:`_orm.Session.rollback` 方法。:class:`_orm.Session` 可以作为上下文管理器使用，以确保在完成操作之后调用 :meth:`_orm.Session.close`::

    with Session(engine) as session:
        result = session.execute(select(User))

    # 自动关闭会话

.. versionchanged:: 1.4

    :class:`_orm.Session` 对象具有延迟“begin”行为，如 :ref:`autobegin <session_autobegin>` 中所述。在调用 :meth:`_orm.Session.close` 方法之后，不再立即开始新的事务。

.. _session_faq:

会话常见问题
--------------------

到这里，很多用户已经对会话（:class:`.Session`）产生了疑问。本节介绍
了最基本的一些问题，当使用 :class:`.Session` 时，您可能会遇到这些问题。如
果您遇到了其他问题，请参考 :doc:`真正的 FAQ</faq/index>`。

我什么时候创建 :class:`.sessionmaker`？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

只需要在应用程序的全局范围内创建一次。它应该被视为应用程序的配置的一
部分。如果您的应用程序在一个包中有三个.py 文件，您可以在 ``__init__.py``
文件中放置 :class:`.sessionmaker` 行；从那时起，您的其他模块会说“从 mypa
ckage 导入 Session”。这样，每个人都只使用 :class:`.Session()`，而该
会话的配置由该中心点控制。

如果您的应用程序启动、导入，但不知道将连接到哪个数据库，则可以稍后再
使用 :meth:`.sessionmaker.configure` 将 :class:`.Session` 绑定到引擎。
在本节的示例中，我们经常会在实际调用 :class:`.Session` 的行正上方创
建 :class:`.sessionmaker`。但这只是为了举例说明！实际上，:class:`.sessionm
aker` 将在模块级别的某个地方。随后调用 :class:`.Session` 的进程将
被放置在数据库对话开始的地方。

.. _session_faq_whentocreate:

何时构造 :class:`.Session`，何时提交，何时关闭？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. topic:: tl;dr;

    1. 作为一般规则，将会话的生命周期与访问和/或操作数据库数据的函数和对
       象**分离和外部**。这将极大地有助于实现可预计和一致的交易范围。

    2. 确保您清楚地知道何时开始和结束事务，并使事务**短暂**，这意味
       着它们在一系列操作的序列末端结束，而不是无限期地保持开放。

:class:`.Session` 通常是在可能需要访问数据库的逻辑操作开始时构造的。

无论何时使用 :class:`.Session` 与数据库通信，它都会在开始通信时开始数
据库交易。此交易一直处于进行中，直到 :class:`.Session` 被回滚、提交或
关闭。如果 :class:`.Session` 再次使用，会话将开始新的事务，前提是前一
个事务已经结束；因此，:class:`.Session` 能够跨多个事务的生命周期，但
一次只能有一个事务。我们称这两个概念为**交易范围**和**会话范围**。

通常没有太大难度来确定 :class:`.Session` 范围的最佳点，尽管广泛的应用
程序体系结构可能会引入一些挑战性的情况。

以下是一些实际情况：

1. Web 应用程序。在这种情况下，最好使用 Web 框架提供的 SQLAlchemy 集
成。否则，基本模式是在 Web 请求开始时创建 :class:`_orm.Session`，在执行
POST、PUT 或 DELETE 的 Web 请求结束时调用 :meth:`_orm.Session.commit` 方法，然
后在请求结束时关闭会话。通常也建议将 :paramref:`_orm.Session.expire_on_commit`
设置为 False，以便在视图层中从 :class:`_orm.Session` 返回的对象再次被访问时，
如果事务已提交，则无需发出新 SQL 查询刷新对象。

2. 后台守护进程。在这种情况下，每个子进程应该创建一个本地的 :class:`.Session`，
   这样就可以在处理子进程的“作业”的生命周期内使用该 :class:`.Session`，然后在完成“作业”时将其关闭。

3. 对于命令行脚本，应用程序应创建一个单独的全局 :class:`.Session`，
   该会话在程序开始工作时建立，并在程序完成任务时立即提交。

4. 对于由 GUI 接口驱动的应用程序，:class:`.Session` 的范围可能应该在用户生成
的事件范围内，例如按钮推送。或者，会话范围可能对应着显式用户交互，例如用户“打开”
一系列记录，然后“保存”它们。

作为一般规则，应将会话的生命周期与访问和/或操作数据库数据的函数和对象分
离和外部。这是一个基本的关注点分离，使得数据特定操作不依赖于它们访问和
操作该数据的上下文。

例如：**不要这么做**：

    ### 这是**错误**的方式 ###

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

将会话（和通常的交易）的生命周期**分离**，**外部**于特定数据访问和操
作函数：

    ### 这是**更好的**（但不是唯一的）方式 ###

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

.. versionchanged:: 1.4 :class:`_orm.Session` 可以作为上下文管理器使用，无需使用外部帮助函数。

:class:`.Session` 是缓存吗？
~~~~~~~~~~~~~~~~~~~~~~~~~

是，也不是。它在某种程度上被用作缓存，因为它实现了 :term:`identity map`
模式，并将对象按其主键键入。但是，它不会对任何查询进行缓存。这意味着，
如果您说 ``session.scalars(select(Foo).filter_by(name='bar'))``，即使 ``Foo(n
ame='bar')`` 现在位于身份映射中，会话也不知道其中的内容。它需要向数据库发出
SQL 语句，返回行后，当它看到该行中的主键时，它才能查看本地身份映射并了解
对象已经存在。只有当您说 ``query.get({some primary key})``，:class:`~sqlalchemy.orm.session.Session` 才不必发出查询。

此外，默认情况下， :class:`.Session` 使用弱引用存储对象实例。这也使得使用 :class:`.Session` 作为缓存的目的无效。

:class:`.Session` 的设计并不是作为一个每个人都可以查看的全局对象。这更
类似于第 2 层高速缓存的工作方式（Second Level Cache）。SQLAlchemy 使
用 `dogpile.cache <https://dogpilecache.readthedocs.io/>`_ 来实现第二个级别的
高速缓存模式。请参见 :ref:`examples_caching` 示例。

如何在特定对象中获取 :class:`~sqlalchemy.orm.session.Session`？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用 :meth:`~.Session.object_session` 类方法即可，在 :class:`_orm.Session` 上可用：

    session = Session.object_session(someobject)

新的 :ref:`核心检查-toplevel` 系统也可以使用：

    from sqlalchemy import inspect

    session = inspect(someobject).session

.. _session_faq_threadsafe:

:class:`.Session` 是线程安全的吗？:class:`_asyncio.AsyncSession` 在并发任务中共享安全吗？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`.Session` 是一个**可变、有状态**的对象，表示一个**单个数据库事务**。因此，
：class:`.Session` 实例**不能在并发线程或 asyncio 任务之间共享，而不经过仔细的同
步处理**。 :class:`.Session` 旨在以**非并发**方式使用，即一个特定的：class:`.Sessio
n` 实例应在同一时间只在一个线程或任务中使用。

使用 SQLAlchemy 的 :ref:`asyncio <asyncio_toplevel>` 扩展的 :class:`_asyncio.AsyncSession` 对
象，这个对象只是一个代理 :class:`_orm.Session`，相同的规则也适用，因此不能
安全地将单个 :class:`_asyncio.AsyncSession` 实例用于多个 asyncio 任务。

:class:`.Session` 或者 :class:`_asyncio.AsyncSession` 实例代表一个单个的
逻辑数据库事务，每次使用 :class:`_orm.Session` 发出命令时，都会有一个对应的状态
为“被绑定”的 :class:`_engine.Connection`，用于与该对象绑定的 :class:`.Engine` 或
:class:`.AsyncEngine`。要注意的是，这些对象都支持绑定到多个引擎，但是在事务的
作用域内仍然只有一个连接在使用。

在事务内的数据库连接也是一个具有状态的对象，旨在以非并发顺序进行操作。命
令将按顺序在连接上发出，并由数据库服务器按照它们发出的顺序处理。随着 :class:`_orm.Session`
在此连接上发出命令并接收结果， :class:`_orm.Session` 本身正在转换其内部状态，以反映此
连接上的命令和数据状态；这些状态包括是否已经开始、提交或回滚事务，是否存在 SAVEPOINT 以及
将单独的数据库行状态与本地 ORM 映射对象的状态进行了精细同步。

在为并发设计数据库应用程序时，合适的模型是每个并发任务/线程都使用自己的
数据库事务。这就是通常讨论数据库并发问题的原因。在传统的关系型数据库系
统中，不存在单个数据库事务同时接收和处理多个命令的情况。

因此，SQLAlchemy 的 :class:`_orm.Session` 和 :class:`_asyncio.AsyncSession` 的并
发模型是 **线程范围的每个会话，asyncio 的 :class:`_asyncio.AsyncSession` 应该为每个任
务单独维护**。在使用多个线程或 asyncio 任务的应用程序中，每个线程均应具有
自己的 :class:`_orm.Session`，每个 asyncio 任务均应具有自己的 :class:`_asyncio.AsyncSession` 。

最好通过在下面的 Python 函数顶层内使用 :ref:`standard context manager pattern
<session_getting>`（即 ``with`` 关键字）来保证这种使用方式在局部范围内
维护 ：class:`_orm.Session` 或 :class:`_asyncio.AsyncSession` 生命周期。

对于从不想将 :class:`.Session` 对象传递给需要它的特定功能和方法的应用程序
中受益的“全局” :class:`.Session` 的应用程序，:class:`.scoped_session`
方法可以提供“线程本地” :class:`.Session` 对象；请参见
 :ref:`unitofwork_contextual` 部分。

在 asyncio 上下文中，:class:`.async_scoped_session`
对象是 :class:`.scoped_session` 的 asyncio 版本，但配置更具挑战性，因
为它需要一个自定义“上下文”函数来生成。