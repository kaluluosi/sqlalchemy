.. _session_events_toplevel:

使用事件跟踪查询、对象和会话更改
===================================

SQLAlchemy拥有一个广泛的 :ref:`事件监听系统 <event_toplevel>`，在Core和ORM中都有使用。在ORM中，有许多事件监听器钩子，这些钩子记录在API级别，如 :ref:`orm_event_toplevel`所述。这些事件集多年来不断增长，包括许多非常有用的新事件以及一些不再像过去那样相关的旧事件。本节将尝试介绍主要的事件钩子以及它们可能被使用的情况。

.. _session_execute_events:

执行事件
---------------

.. versionadded:: 1.4  :class:`_orm.Session`现在拥有一个单一的全面挂钩，旨在拦截ORM代表进行的所有SELECT语句以及批量UPDATE和DELETE语句。此挂钩取代了先前的:meth:`_orm.QueryEvents.before_compile`事件和:meth:`_orm.QueryEvents.before_compile_update`以及:meth:`_orm.QueryEvents.before_compile_delete`。

:class:`_orm.Session` 通过在 :meth:`_orm.Session.execute` 方法中调用所有查询，这包括由 :class:`_orm.Query` 发出的所有 SELECT 语句以及由列和关系加载器代表发出的所有 SELECT 语句 ，能拦截和修改查询。该系统使用 :meth:`_orm.SessionEvents.do_orm_execute` 事件挂钩以及 :class:`_orm.ORMExecuteState` 对象来表示事件状态。

基本查询截获
^^^^^^^^^^^^^^^^^^^^^^^^^

首先要注意的是，:meth:`_orm.SessionEvents.do_orm_execute` 首先对查询截取非常有用，包括由 :class:`_orm.Query` 以 :term:`1.x 样式` emit 以及当启用了 ORM 的 :term:`2.0 样式` 的 :func:`_sql.select`、 :func:`_sql.update` 或 :func:`_sql.delete` 构造函数被传递到 :meth:`_orm.Session.execute` 时。 :class:`_orm.ORMExecuteState` 构造提供访问器，以允许修改语句、参数和选项。

下面的示例说明了一些简单的修改 SELECT 语句。

    Session = sessionmaker(engine)


    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        if orm_execute_state.is_select:
            # 为所有SELECT语句添加populate_existing

            orm_execute_state.update_execution_options(populate_existing=True)

            # 检查SELECT是针对某个实体的，如果是，则添加 ORDER BY

            col_descriptions = orm_execute_state.statement.column_descriptions

            if col_descriptions[0]["entity"] is MyEntity:
                orm_execute_state.statement = statement.order_by(MyEntity.name)

上述示例说明了一些简单的修改 SELECT 语句。:meth:`_orm.SessionEvents.do_orm_execute` 事件钩子旨在替换前面使用的 :meth:`_orm.QueryEvents.before_compile` 事件的使用，后者未对各种加载程序的各种类型合一致地触发；另外， :meth:`_orm.QueryEvents.before_compile` 仅适用于使用 :class:`_orm.Query` 的 :term:`1.x 样式`，而不适用于使用 :meth:`_orm.Session.execute` 的 :term:`2.0 样式`。

.. _do_orm_execute_global_criteria:

添加全局WHERE/ON条件
^^^^^^^^^^^^^^^^^^^^^^^^^

其中一个最常请求的查询扩展功能是添加 WHERE 条件到所有查询实体中的所有出现。这可以通过使用 :func:`_orm.with_loader_criteria` 查询选项来实现。该选项可以单独使用，也适用于 :meth:`_orm.SessionEvents.do_orm_execute` 事件::

    from sqlalchemy.orm import with_loader_criteria

    Session = sessionmaker(engine)


    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        if (
            orm_execute_state.is_select
            and not orm_execute_state.is_column_load
            and not orm_execute_state.is_relationship_load
        ):
            orm_execute_state.statement = orm_execute_state.statement.options(
                with_loader_criteria(MyEntity.public == True)
            )

上面的示例将选项添加到所有SELECT语句中，以将所有针对 ``MyEntity`` 的查询限制为使用 ``public == True`` 过滤。这些条件将应用于该查询范围内该类的 **所有** 加载。默认情况下， :func:`_orm.with_loader_criteria` 选项将自动传播到关系加载程序，这将适用于后续关系加载，包括 lazy loads, selectinloads等。

如果一系列类都具有某些常见列结构，如果使用 :ref:`declarative mixins <declarative_mixins>` 组成这些类，则可以使用 mixin 类本身与 :func:`_orm.with_loader_criteria` 选项结合使用，通过使用 Python lambda。Python lambda 将针对匹配条件的特定实体在查询编译时调用。例如，对基于名为 ``HasTimestamp`` 的 mixin 的一系列类进行操作：

    import datetime


    class HasTimestamp:
        timestamp = mapped_column(DateTime, default=datetime.datetime.now)


    class SomeEntity(HasTimestamp, Base):
        __tablename__ = "some_entity"
        id = mapped_column(Integer, primary_key=True)


    class SomeOtherEntity(HasTimestamp, Base):
        __tablename__ = "some_entity"
        id = mapped_column(Integer, primary_key=True)

上述类 SomeEntity 和 SomeOtherEntity 都将有一个默认为当前日期和时间的 timestamp 列。一个事件可以用于拦截所有从 HasTimestamp 扩展的对象，并过滤它们的 timestamp 列，使其不早于一个月前的日期：

    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        if (
            orm_execute_state.is_select
            and not orm_execute_state.is_column_load
            and not orm_execute_state.is_relationship_load
        ):
            one_month_ago = datetime.datetime.today() - datetime.timedelta(months=1)

            orm_execute_state.statement = orm_execute_state.statement.options(
                with_loader_criteria(
                    HasTimestamp,
                    lambda cls: cls.timestamp >= one_month_ago,
                    include_aliases=True,
                )
            )

.. warning:: 在 :func:`_orm.with_loader_criteria` 调用中使用lambda会一次性执行 **每个唯一类** 。不应在这个lambda中调用自定义函数。请参见 :ref:`engine_lambda_caching` 查看 "lambda SQL" 特性的概述，这只适用于高级用途。

.. seealso::

    :ref:`examples_session_orm_events` - 包括上述 :func:`_orm.with_loader_criteria` 示例的工作示例。

.. _do_orm_execute_re_executing:

重新执行语句
^^^^^^^^^^^^^^^^^^^^^^^

.:class:`_orm.ORMExecuteState` 可以控制给定语句的执行，包括只要使用预先构造的结果集检索即可将查询语句替换为接收到的结果集，以及可以反复使用相同语句，可根据需要更改其状态，例如，在多个数据库连接上调用，然后在内存中合并结果。这两种高级模式都作为SQLAlchemy的示例套件中的示例提供，如下所述。

当在 :meth:`_orm.SessionEvents.do_orm_execute` 事件钩子中的内部调用 :meth:`_orm.Session.execute` 来使用新的嵌套调用执行语句时，会涉及到略微复杂的递归序列，旨在在SQL语句在各个非SQL上下文之间进行重定向时解决相当复杂的问题。下面链接互联网中的"dogpile caching" 和 "horizontal sharding" 是使用该特性时的指南。

:meth:`_orm.ORMExecuteState.invoke_statement` 方法可用于在新的嵌套调用 :meth:`_orm.Session.execute` 函数时，使用新的嵌套的 :meth:`_orm.Session.execute` 触发当前正在处理的执行的后续处理，而反过来检索返回的:class:`_engine.Result`。在这嵌套调用中，所触发的:meth:`_orm.SessionEvents.do_orm_execute` 事件处理程序也会被跳过。

:meth:`_orm.ORMExecuteState.invoke_statement` 方法返回一个 :class:`_engine.Result` 对象; 该对象具有将其冻结为可缓存格式和“解冻”为新的:class:`_engine.Result` 对象以及将其数据与其他 :class:`_engine.Result` 对象合并的功能。

例如：在 :meth:`_orm.SessionEvents.do_orm_execute` 中，使用缓存实现缓存::

    from sqlalchemy.orm import loading

    cache = {}


    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        if "my_cache_key" in orm_execute_state.execution_options:
            cache_key = orm_execute_state.execution_options["my_cache_key"]

            if cache_key in cache:
                frozen_result = cache[cache_key]
            else:
                frozen_result = orm_execute_state.invoke_statement().freeze()
                cache[cache_key] = frozen_result

            return loading.merge_frozen_result(
                orm_execute_state.session,
                orm_execute_state.statement,
                frozen_result,
                load=False,
            )

通过上面的钩子，可以实现以下示例中使用缓存的方法::

    stmt = (
        select(User).where(User.name == "sandy").execution_options(my_cache_key="key_sandy")
    )

    result = session.execute(stmt)

在上述代码中，使用了自定义的执行选项，以确立一个“缓存键”，该缓存键将由 :meth:`_orm.SessionEvents.do_orm_execute` 钩子拦截。如果这个缓存键匹配到缓存中的 :class:`_engine.FrozenResult` 对象，则使用该对象。这个示例使用 :meth:`_engine.Result.freeze` 方法来“冻结”一个 :class:`_engine.Result` 对象，该对象在上述情况下将包含ORM的结果，以便它可以存储在缓存中并被多次使用。为了从“冻结”结果返回一个实时结果，可以使用 :func:`_orm.loading.merge_frozen_result` 函数将结果对象中的“冻结”数据合并到当前会话中。

上面的示例在 :ref:`examples_caching` 中作为完整示例实现。

:meth:`_orm.ORMExecuteState.invoke_statement` 方法还可以被多次调用，每次传递不同的信息到
:paramref:`_orm.ORMExecuteState.invoke_statement.bind_arguments` 参数，以便:meth:`.Session` 每次使用不同的 :class:`_engine.Engine` 对象。这将每次都返回一个不同的 :class:`_engine.Result` 对象；这些结果可以使用 :meth:`_engine.Result.merge` 方法合并。这是 :ref:`horizontal_sharding_toplevel` 所采用的技术；请参见源代码以熟悉。

.. seealso::

    :ref:`examples_caching`

    :ref:`examples_sharding`




.. _session_persistence_events:

持久性事件
------------------

可能是最广泛使用的系列的事件是"持久性"事件，它对应于 :ref:`flush process<session_flushing>`。 flush 是在其中对待定更改的所有决策都被做出，并以 INSERT、UPDATE 和 DELETE 语句的形式发布到数据库的地方。

before_flush()
^^^^^^^^^^^^^^^

当应用程序希望确保在刷新进行时进行一些其他的持久性更改以及在对象被持久化之前验证其状态并在这之后组合附加对象和引用时， :meth:`.SessionEvents.before_flush` 挂钩是默认和最广泛使用的事件。在这个事件中，可以安全地操作 :class:`.Session`的状态，也就是说，可以自由地添加对象，删除对象，并自由地更改对象的单个属性，在事件钩子完成时，这些更改将被纳入刷新进程中。

典型的 :meth:`.SessionEvents.before_flush` 钩子将被要求扫描 :attr:`.Session.new`、 :attr:`.Session.dirty` 和 :attr:`.Session.deleted` 集合，以查找将发生的对象。

示例，请参见 :ref:`examples_versioned_history` 和 :ref:`examples_versioned_rows`。

after_flush()
^^^^^^^^^^^^^^

:meth:`.SessionEvents.after_flush` 挂钩在 SQL 发出刷新 process 之后被调用，但在被持久对象的状态被修改之前。也就是说，您仍然可以检查 :attr:`.Session.new`、 :attr:`.Session.dirty` 和 :attr:`.Session.deleted` 集合，以查看刚刚刷新的情况，还可以使用 :class:`.AttributeState` 提供的诸如跟踪历史记录这样的特性，以查看刚刚持久化的更改。在 :meth:`.SessionEvents.after_flush` 事件中，可以根据所观察到的情况向数据库发出其他 SQL。

after_flush_postexec()
^^^^^^^^^^^^^^^^^^^^^^^^

:meth:`.SessionEvents.after_flush_postexec` 和 :meth:`.SessionEvents.after_flush` 相比，会尽快在修改对象的状态以承认刚查询操作时被调用。 :attr:`.Session.new`、 :attr:`.Session.dirty` 和 :attr:`.Session.deleted` 集合通常在此处完全为空。在这个钩子中，有能力在对象上进行新的变更，这意味着所述 :class:`.Session` 再次进入"dirty"状态. 如果在 :meth:`.Session.commit` 上下文中检测到新变化，则 Session 的机制会导致它再次进行刷新。在此挂钩中检测到新变化时，:meth:`.SessionEvents.after_flush_postexec` 钩子将被跨过。

.. _session_persistence_mapper:

Mapper级别的 Flush 事件
^^^^^^^^^^^^^^^^^^^^^^^^^

除了 flush 层钩子之外，还有一系列钩子，它们更细粒度，即基于 INSERT、UPDATE或 DELETE 的每个对象进行单独处理，并根据 flush process 进行了细分。这些是 mapper 持久性钩子，它们也很受欢迎，但是这些事件需要谨慎处理，因为它们在已经进行的 flush 过程的上下文内进行；许多操作在此处是不安全的。

这些事件包括：

* :meth:`.MapperEvents.before_insert`
* :meth:`.MapperEvents.after_insert`
* :meth:`.MapperEvents.before_update`
* :meth:`.MapperEvents.after_update`
* :meth:`.MapperEvents.before_delete`
* :meth:`.MapperEvents.after_delete`

.. note::
  重要的一点是，这些事件 **仅** 适用于 :ref:`session flush操作<session_flushing>`，而不应用于在 :ref:`orm_expression_update_delete` 中所述的 ORM-level INSERT/UPDATE/DELETE 功能。要拦截 ORM-level DML，请使用 :meth:`_orm.SessionEvents.do_orm_execute` 事件。

每个事件都传递了 :class:`_orm.Mapper`、映射的对象本身以及使用的 :class:`_engine.Connection` 来发出 INSERT、UPDATE 或 DELETE 语句。这类事件很有吸引力，因为如果一个应用程序想要将一些活动与定期使用 INSERT 持久化的特定类型的对象相关联，那么该挂钩具有很高的特异性；与 :meth:`.SessionEvents.before_flush` 不同，不需要搜索 :attr:`.Session.new` 集合以查找目标。但是，刷新计划，它代表有决定要发出的每个单独的 INSERT、UPDATE、DELETE 语句的列表已经被决定，不会在这个阶段进行任何更改。因此，只有在该对象行的属性**本地**上操作是可使用的。任何对对象或其他对象的其他更改都会影响 :class:`.Session` 的状态，这将导致它无法正常工作。

在这些 mapper 级别的持久化事件中不支持的操作包括：

* :meth:`.Session.add`
* :meth:`.Session.delete`
* 映射集合的 append、add、remove、delete、discard 等。
* 映射关系属性设置/del 事件，即 ``someobject.related =someotherobject``

传递 :class:`_engine.Connection` 的原因是，建议在这里直接在 :class:`_engine.Connection` 上进行 **简单 SQL 操作**，例如增量计数器或在 log 表内插入额外行。

此外，如果您的应用程序代码动态添加属性到对象上，则这些属性中的设置和删除也不适用于 mapper级别的钩子。要在每个属性上捕获动态更改，应使用类似于 :ref:`simple_validators` 中所描述的验证程序或使用 :attr:`.SessionEvents.before_flush` 事件。对于这两种方法，我们建议在实例的 ``__init__()`` 方法中建立其他状态，例如创建其他要与新对象关联的对象。这些活动也可以使用 :attr:`.InstanceState.mutate` 访问器来进行。

.. _session_lifecycle_events:

对象生命周期事件
-----------------------

事件的另一个用途是跟踪对象的生命周期。这是指在 :ref:`session_object_states` 中首次介绍的状态。所有上述状态都可以完全通过事件进行跟踪。每个事件都表示不同的状态转换，也就是说，起始状态和目标状态都是被跟踪的。除了最初的暂态事件之外，所有事件都以:class:`.Session`对象或类的形式呈现，这意味着它们可以与特定的 :class:`.Session` 对象或与 :class:`.sessionmaker` 关联。:class:`.Session` 对象：

    from sqlalchemy import event
    from sqlalchemy.orm import Session

    session = Session()


    @event.listens_for(session, "transient_to_pending")
    def object_is_pending(session, obj):
        print("new pending: %s" % obj)

或者与:class:`.Session`类本身一起使用以及与特定的 :class:`.sessionmaker`。这可能是最有用的形式::

    from sqlalchemy import event
    from sqlalchemy.orm import sessionmaker

    maker = sessionmaker()


    @event.listens_for(maker, "transient_to_pending")
    def object_is_pending(session, obj):
        print("new pending: %s" % obj)

当然，依次为一个方程堆叠这些侦听器，例如检测到所有已进入持久状态的对象：

        @event.listens_for(maker, "pending_to_persistent")
        @event.listens_for(maker, "deleted_to_persistent")
        @event.listens_for(maker, "detached_to_persistent")
        @event.listens_for(maker, "loaded_as_persistent")
        def detect_all_persistent(session, instance):
            print("object is now persistent: %s" % instance)

暂态
^^^^^^^^^

当映射对象的所有值首次创建时，它们都以 :term:`transient` 的状态开始，即该对象仅自身存在，不与任何 :class:`.Session` 关联。在这种状态下，没有特定的“转换”事件，因为没有 :class:`.Session`，但是如果一个人想拦截任何瞬态对象的创建，那么 :meth:`.InstanceEvents.init` 方法可能是最好的事件。此事件适用于特定的类或超类。例如，在所有新对象上进行拦截的事件::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy import event


    class Base(DeclarativeBase):
        pass


    @event.listens_for(Base, "init", propagate=True)
    def intercept_init(instance, args, kwargs):
        print("new transient: %s" % instance)

暂态转持久态
^^^^^^^^^^^^^^^^^^^^

当该对象被通过 :meth:`.Session.add` 或等效方法与 :class:`.Session` 相关联时，该暂态对象将变为 :term:`pending` 对象。如果一个对象作为显式添加的结构引用的级联的结果，那么一个对象也可能变为 :term:`persistent` 对象（嵌套级联结构在持久化事件处理中被自动处理）。跟踪从暂态到 pending 的转换，使用 :meth:`.SessionEvents.transient_to_pending` 事件::

    @event.listens_for(sessionmaker, "transient_to_pending")
    def intercept_transient_to_pending(session, object_):
        print("transient to pending: %s" % object_)

pending 转 persistent
^^^^^^^^^^^^^^^^^^^^^

当刷新执行并对实例进行 INSERT 操作时， :term:`pending` 对象将变为 :term:`persistent` 对象。这个对象现在有一个身份标识符。使用 :meth:`.SessionEvents.pending_to_persistent` 事件来跟踪 pending 到 persistent 的过程。

    @event.listens_for(sessionmaker, "pending_to_persistent")
    def intercept_pending_to_persistent(session, object_):
        print("pending to persistent: %s" % object_)

pending 转 transient
^^^^^^^^^^^^^^^^^^^^

当 :meth:`.Session.rollback` 方法在待定对象被 Flush 之前被调用或在 Flush 之前对象被删除使用 :meth:`.Session.expunge` 方法时， :term:`pending` 对象可以回归到 :term:`transient` 状态。使用 :meth:`.SessionEvents.pending_to_transient` 事件来跟踪 pending 到 transient 的过程。

    @event.listens_for(sessionmaker, "pending_to_transient")
    def intercept_pending_to_transient(session, object_):
        print("transient to pending: %s" % object_)

作为持久态加载
^^^^^^^^^^^^^^^^^^^^

对象可以直接以 :term:`persistent` 状态出现在 :class:`.Session` 中，当它们从数据库中加载时就是如此。跟踪这个状态转换等同于跟踪对象荷载时，也就是使用 :meth:`.InstanceEvents.load` 实例级别事件。但是， :meth:`.SessionEvents.loaded_as_persistent` 事件作为一个 session 中心钩子，为拦截对象通过这个特定途径进入持久状态提供了该hook。

    @event.listens_for(sessionmaker, "loaded_as_persistent")
    def intercept_loaded_as_persistent(session, object_):
        print("object loaded into persistent state: %s" % object_)

persistent 转 transient
^^^^^^^^^^^^^^^^^^^^^^^

当该事务被回滚时， :term:`persistent` 的对象可恢复到 :term:`transient`的状态。在事务回滚时，因为其所属的 :class:`.Session` 的状态已被修改，该对象回滚并从identity map中删除。跟踪从 persistent 到 transient 的过程，使用 :meth:`.SessionEvents.persistent_to_transient` 事件钩子::

    @event.listens_for(sessionmaker, "persistent_to_transient")
    def intercept_persistent_to_transient(session, object_):
        print("persistent to transient: %s" % object_)

persistent 转 deleted
^^^^^^^^^^^^^^^^^^^^^

在 Flush 过程中从数据库中删除标记为删除的对象时，该 :term:`persistent` 对象进入 :term:`deleted` 状态。请注意，这与调用 :meth:`.Session.delete` 方法删除目标对象时是**不同**的。 :meth:`.Session.delete` 此方法仅 **标记** 对象要被删除；只有当 Flush 过程的一部分有实际的 DELETE 语句时，才会实际发出 DELETE 语句。在 Flush 后，目标对象处于“deleted”状态。

在 "删除" 状态下，对象仅与 :class:`.Session` 稍微关联。它不在identity map中，也不在引用 :attr:`.Session.deleted` 集合中，该集合与它是待定要被删除的状态有关。

从“deleted”状态，对象可以在事务被提交时进入分离状态，或者在事务被回滚时恢复到持久状态。

使用 :meth:`.SessionEvents.persistent_to_deleted` 跟踪从 persistent 到 deleted 的转换::

    @event.listens_for(sessionmaker, "persistent_to_deleted")
    def intercept_persistent_to_deleted(session, object_):
        print("object was DELETEd, is now in deleted state: %s" % object_)

deleted 转 detached
^^^^^^^^^^^^^^^^^^^

当会话的事务提交时， :term:`deleted` 对象变为 :term:`detached`。在调用 :meth:`.Session.commit` 方法后，数据库事务最终，并且 :class:`.Session` 完全丢弃了 deleted 对象并删除了所有关联。使用 :meth:`.SessionEvents.deleted_to_detached` 跟踪 deleted 到 detached 的转换::

    @event.listens_for(sessionmaker, "deleted_to_detached")
    def intercept_deleted_to_detached(session, object_):
        print("deleted to detached: %s" % object_)

.. note::

    当对象处于被删除状态时， :attr:`.InstanceState.deleted` 属性可用，该属性可使用 ``inspect(object).deleted`` 访问器返回 True。然而，当对象处于删除时， :attr:`.InstanceState.deleted` 再次返回 False。为了检测对象是否被删除，无论它是分离的还是不是，应使用 :attr:`.InstanceState.was_deleted` 访问器。


persistent 转 detached
^^^^^^^^^^^^^^^^^^^^^^^

当使用 :meth:`.Session.expunge`、 :meth:`.Session.expunge_all`或 :meth:`.Session.close` 方法将对象与 :class:`.Session` 清除关联时， :term:`persistent` 对象变为 :term:`detached`。事实上，如果应用程序的引用被垃圾回收丢弃，导致所归属的 :class:`.Session` 隐式解除引用，则对象可能变为**隐式分离 **；在这种情况下，**不会发出任何事件**。

使用 :meth:`.SessionEvents.persistent_to_detached` 跟踪对象从 persistent 到 detached 的过程::

    @event.listens_for(sessionmaker, "persistent_to_detached")
    def intercept_persistent_to_detached(session, object_):
        print("object became detached: %s" % object_)

detached 转 persistent
^^^^^^^^^^^^^^^^^^^^^^

当使用 :meth:`.Session.add` 或等效方法重新与会话关联分离的对象时，该分离对象将变为 :term:`persistent`。使用 :meth:`.SessionEvents.detached_to_persistent` 事件来跟踪从 detached 到 persistent 的对象转换：

    @event.listens_for(sessionmaker, "detached_to_persistent")
    def intercept_detached_to_persistent(session, object_):
        print("object became persistent again: %s" % object_)

deleted 转 persistent
^^^^^^^^^^^^^^^^^^^^^

如果事务被回滚，则可以将 :term:`deleted` 对象恢复为 :term:`persistent` 状态。这就是当该事务回滚时 :meth:`.Session.rollback` 调用。有助于使用 :meth:`.SessionEvents.persistent_to_deleted` 事件来跟踪从 persistent 到 deleted 的对象转换：

    @event.listens_for(sessionmaker, "persistent_to_deleted")
    def intercept_persistent_to_deleted(session, object_):
        print("object was DELETEd, is now in deleted state: %s" % object_)使用:meth:`.Session.rollback`方法回滚会话。使用:meth:`.SessionEvents.deleted_to_persistent`事件跟踪回到持久状态的已删除对象：

    @event.listens_for(sessionmaker, "deleted_to_persistent")
    def intercept_deleted_to_persistent(session, object_):
        print("deleted to persistent: %s" % object_)

.. _session_transaction_events:

事务事件
------------------

事务事件允许通知应用程序当事务边界在 :class:`.Session` 级别发生时，以及当 :class:`.Session` 在 :class:`_engine.Connection` 对象上更改事务状态时。


* :meth:`.SessionEvents.after_transaction_create`，:meth:`.SessionEvents.after_transaction_end` - 这些事件跟踪 :class:`.Session` 的逻辑事务作用域，不特定于个别数据库连接。这些事件旨在帮助集成事务跟踪系统，例如"zope.sqlalchemy"。在应用程序需要将某些外部作用域与 :class:`.Session` 的事务作用域对齐时，请使用这些事件。这些钩子反映 :class:`.Session` 的"嵌套"事务行为，因为它们跟踪逻辑的"子事务"以及"嵌套"(例如，SAVEPOINT)事务。

* :meth:`.SessionEvents.before_commit`，:meth:`.SessionEvents.after_commit`，:meth:`.SessionEvents.after_begin`，:meth:`.SessionEvents.after_rollback`，:meth:`.SessionEvents.after_soft_rollback` - 这些事件允许从数据库连接的角度跟踪事务事件。特别是，:meth:`.SessionEvents.after_begin`是一个每个连接的事件。维护多个连接的 :class:`.Session` 将为每个连接分别发出此事件，因为那些连接在当前事务中被使用。然后回滚和提交事件再引用数据库API连接直接接受回滚或提交指令的时间。

属性更改事件
-----------------------

属性更改事件允许拦截对象上特定属性被修改的情况。这些事件包括：:meth:`.AttributeEvents.set`，:meth:`.AttributeEvents.append`和 :meth:`.AttributeEvents.remove`。这些事件非常有用，特别是对于每个对象的验证操作；然而，使用"验证器"钩子通常更加方便，这个钩子在幕后使用这些钩子；有关此背景的详细信息，请参阅:ref:`simple_validators`。属性事件也负责反向引用的机制。:ref:`examples_instrumentation`中有一个使用属性事件的示例。