.. _session_events_toplevel:

使用事件追踪查询、对象和会话更改
===========================================

SQLAlchemy在Core和ORM中都使用了广泛的   :ref:`事件监听<event_toplevel>` 。这些事件的集合已经发展了多年，包括许多非常有用的新事件以及一些不再像曾经那样相关的旧事件。本节将尝试介绍主要的事件钩子及其使用场景。

.. _session_execute_events:

执行事件
---------------

.. versionadded:: 1.4    :class:`_orm.Session`  事件，还包括  :meth:` _orm.QueryEvents.before_compile_update`  和  :meth:`_orm.QueryEvents.before_compile_delete`  。

  :class:`_orm.Session`  方法调用的所有查询，包括  :class:` _orm.Query`  事件钩子以及 :class:`_orm.ORMExecuteState` 对象来表示事件状态。


基本查询拦截
^^^^^^^^^^^^^^^^^^^^^^^^^

  :meth:`_orm.SessionEvents.do_orm_execute`  首先对查询的任何拦截非常有用，包括使用  :term:` 1.x风格`  的  :class:`_orm.Query`  的  :func:` _sql.select` ，  :func:`_sql.update`  。 :class:` _orm.ORMExecuteState`结构提供访问器，允许修改语句、参数和选项:: 

    Session = sessionmaker(engine)


    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        if orm_execute_state.is_select:
            # 为所有SELECT语句添加populate_existing

            orm_execute_state.update_execution_options(populate_existing=True)

            # 检查SELECT是否针对特定实体，并在是的情况下加入ORDER BY
            col_descriptions = orm_execute_state.statement.column_descriptions

            if col_descriptions[0]["entity"] is MyEntity:
                orm_execute_state.statement = statement.order_by(MyEntity.name)

上面的示例说明了如何修改SELECT语句。在这个级别上，  :meth:`_orm.SessionEvents.do_orm_execute`  事件钩子旨在替换之前使用的  :meth:` _orm.QueryEvents.before_compile`  事件的使用，对于各种装载器，它没有被一致地触发。此外，  :meth:`_orm.QueryEvents.before_compile`  仅适用于  :class:` _orm.Query`  ，而不适用于  :meth:`_orm.Session.execute`  的  :term:` 2.0风格`  。

.. _do_orm_execute_global_criteria:

添加全局WHERE/ON条件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最常请求的查询扩展特性之一是能够向所有查询的实体添加WHERE条件。这可以通过使用  :func:`_orm.with_loader_criteria`  事件中使用，这是最理想的选择：

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

上面的示例向所有SELECT语句添加了一个选项，该选项将限制对``MyEntity``的所有查询，以过滤``public == True``。该条件将应用于立即查询范围内该类的所有加载项。默认情况下， :func:`_orm.with_loader_criteria` 选项会自动传播到关系加载程序，这将应用于后续关系加载，包括lazyloads、selectinloads等。

对于一系列具有公共列结构的类，如果使用了一个  :ref:`声明性混合类<declarative_mixins>` ` HasTimestamp``的混合类：

    import datetime


    class HasTimestamp:
        timestamp = mapped_column(DateTime, default=datetime.datetime.now)


    class SomeEntity(HasTimestamp, Base):
        __tablename__ = "some_entity"
        id = mapped_column(Integer, primary_key=True)


    class SomeOtherEntity(HasTimestamp, Base):
        __tablename__ = "some_entity"
        id = mapped_column(Integer, primary_key=True)

上述类``SomeEntity``和``SomeOtherEntity``将分别具有一个列``timestamp``，其默认值为当前日期和时间。事件可以用于拦截所有扩展自``HasTimestamp``并将其``timestamp``列过滤为一个月之内日期的对象： 

    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        if (如果满足下列条件之一：

.. code-block:: python

    orm_execute_state.is_select
    and not orm_execute_state.is_column_load
    and not orm_execute_state.is_relationship_load
：

一个月前的日期是：

.. code-block:: python

    one_month_ago = datetime.datetime.today() - datetime.timedelta(months=1)

可以使用   :func:`_orm.with_loader_criteria`  选项限制查询结果中数据的范围，使其仅包括在给定时间范围内的行。例如，要获取所有   :class:` HasTimestamp`  类都具有 `timestamp` 属性的查询，并且该属性小于或等于一个月前的行（以及任何其他在别名列中存在 `HasTimestamp` 的模型），可以使用以下代码：

.. code-block:: python

    orm_execute_state.statement = orm_execute_state.statement.options(
        with_loader_criteria(
            HasTimestamp,
            lambda cls: cls.timestamp >= one_month_ago,
            include_aliases=True,
        )
    )

.. warning:: 在   :func:`~sqlalchemy.orm.with_loader_criteria`  中使用 lambda 函数的时候其仅 **针对每个唯一的类** 调用一次。自定义函数不应该在此 lambda 函数中调用。 了解“lambda SQL”特性以供高级使用，请参阅   :ref:` engine_lambda_caching` 。

.. seealso::

      :ref:`examples_session_orm_events`  - 包括使用接受   :func:` ~sqlalchemy.orm.with_loader_criteria`  的完整示例。

.. _do_orm_execute_re_executing:

重新执行语句
^^^^^^^^^^^^

.. deepalchemy:: 语句重新执行功能涉及到一个稍微复杂的递归顺序，旨在解决将执行SQL语句重新定向到各种非SQL上下文的相当棘手的问题。下面链接的“犬舍缓存”和“水平分片”是使用此功能时的指南。

  :class:`_orm.ORMExecuteState`  能够控制给定语句的执行；这包括能够完全不调用语句，允许从缓存中返回已检索到的预构建结果集，以及在不同状态下重复调用同一语句的能力，例如针对多个数据库连接调用该语句，然后将结果在内存中合并。这两种高级模式在下面的 SQLAlchemy 示例套件中都有展示。

在  :meth:`_orm.SessionEvents.do_orm_execute`  事件挂钩内，可以使用  :meth:` _orm.ORMExecuteState.invoke_statement`  方法，使用  :meth:`_orm.Session.execute`  的新嵌套调用来调用语句，这将预占正在进行的当前执行的后续处理，并返回内部执行返回的   :class:` _engine.Result` 。因此，在此嵌套调用中，至此为止激发的事件处理程序均会被跳过。

  :meth:`_orm.ORMExecuteState.invoke_statement`   方法返回一个   :class:` _engine.Result`  对象；此对象然后具有将其“冻结”为可缓存格式并“解冻”为新的   :class:`_engine.Result`  对象，以及将其数据与其他   :class:` _engine.Result`  对象合并的能力。

例如，使用  :meth:`_orm.SessionEvents.do_orm_execute`  实现缓存：

从 sqlalchemy.orm 中导入 loading::

    from sqlalchemy.orm import loading

缓存是一个字典：

    cache = {}

添加缓存创建的回调函数

.. code-block:: python

    @event.listens_for(Session, "do_orm_execute")
    def _do_orm_execute(orm_execute_state):
        # 检查是否使用了特定的自定义选项
        if "my_cache_key" in orm_execute_state.execution_options:
            # 检索该选项的值进行匹配
            cache_key = orm_execute_state.execution_options["my_cache_key"]

            # 如果缓存中存在与之匹配的值，则使用匹配的值
            if cache_key in cache:
               frozen_result = cache[cache_key]
            # 如果没有，则使用 invoke_statement() 获取结果集的 FrozenResult 冻结格式，并将其传递给缓存
            else:
                frozen_result = orm_execute_state.invoke_statement().freeze()
                cache[cache_key] = frozen_result

            # 调用 `merge_frozen_result()` 方法将 frozen_result 与 session 中的语句合并并返回结果
            return loading.merge_frozen_result(
                orm_execute_state.session,
                orm_execute_state.statement,
                frozen_result,
                load=False,
            )

使用缓存：

不同的查询通过传递不同的关键字参数 ``my_cache_key`` 设定不同的缓存键值。下面的例子中，查询名为 `sandy` 的 ``User`` 将使用 ``key_sandy`` 作为它的 `my_cache_key`，并且在会话中执行。如果缓存中存在与 `key_sandy` 匹配的值，它将被加载。否则会进行查询，结果将写入缓存。

    stmt = (
        select(User).where(User.name == "sandy").execution_options(my_cache_key="key_sandy")
    )

    result = session.execute(stmt)

以上，自定义执行选项作为参数传递给  :meth:`_sql.Select.execution_options`  以建立“缓存键”，该缓存键然后会被  :meth:` _orm.SessionEvents.do_orm_execute`  回调拦截。此缓存键将匹配可能存在于缓存中的   :class:`_engine.FrozenResult`  对象，并在存在时重新使用，将 ORM 结果组合在一起。

在   :ref:`examples_caching`  的完整示例中实现了上述示例。

  :meth:`_orm.ORMExecuteState.invoke_statement`   方法也可以多次调用，同时通过  :paramref:` _orm.ORMExecuteState.invoke_statement.bind_arguments`  参数传递不同的信息，使   :class:`_orm.Session`  每次使用不同的   :class:` _engine.Engine`  对象。这将每次返回一个不同的   :class:`_engine.Result`  对象；这些结果可以使用  :meth:` _engine.Result.merge`  方法合并。这是由   :ref:`horizontal_sharding_toplevel`  扩展采用的技术；请查阅源代码以熟悉。

.. seealso::

      :ref:`examples_caching` 

      :ref:`examples_sharding` 



.. _session_persistence_events:

持久化事件
----------

“持久化”事件可能是最广泛使用的一系列事件，对应   :ref:`flush 过程<session_flushing>` 。冲洗是在该过程中做出有关待处理对象的决策，并以 INSERT，UPDATE 和 DELETE 语句的形式发出的地方。

``before_flush()``
^^^^^^^^^^^^^^^^^^  :meth:`.SessionEvents.before_flush`  
-----------------------------------

  :meth:`.SessionEvents.before_flush`  是应用程序在执行flush时需要确保进行额外数据持久化更改的最常用事件。使用  :meth:` .SessionEvents.before_flush`  
操作对象以验证它们的状态，并在持久化之前构建额外的对象和引用。在此事件中，**可以安全地操作Session的状态**。也就是说，
可以将新对象连接到Session，可以删除对象，并且可以自由更改对象上的单个属性。这些更改在事件完成后会被纳入到flush过程中。

典型的  :meth:`.SessionEvents.before_flush`  挂钩将扫描集合:attr:` .Session.new`，  :attr:`.Session.dirty`  和  :attr:` .Session.deleted`  
以查找要发生的更改的对象。

有关  :meth:`.SessionEvents.before_flush`  的示例，请参见：ref:` examples_versioned_history` 和 :ref:` examples_versioned_rows` 等示例。

``after_flush()``
-----------------------

  :meth:`.SessionEvents.after_flush`  是在SQL发出flush流程之后被调用，但在刷新的对象状态发生更改之前被调用。也就是说，
仍然可以检查  :attr:`.Session.new`  ，  :attr:` .Session.dirty`  和  :attr:`.Session.deleted`   collections，
以查看刚刚刷新的内容，还可以使用历史记录跟踪功能（例如由 :class:`.AttributeState` 提供的功能），
以查看刚刚持久化的更改。在  :meth:`.SessionEvents.after_flush`  事件中，可以根据所观察到的更改向数据库发出其他SQL。

``after_flush_postexec()``
-------------------------

  :meth:`.SessionEvents.after_flush_postexec`  在  :meth:` .SessionEvents.after_flush`  之后不久被调用，但在已经修改对象状态以反映刚执行的flush操作之后被调用。
  :attr:`.Session.new`  ，  :attr:` .Session.dirty`  和  :attr:`.Session.deleted`  集合在此处通常为空。
使用  :meth:`.SessionEvents.after_flush_postexec`  检查标识映射以获取已完成对象并可能发出其他SQL。在此挂钩中，有能力在对象上进行新的更改，
这意味着**Session**将再次进入“dirty”状态;如果在  :meth:`.Session.commit`  上下文中检测到新变化，则此处会引发flush**again**，
否则，未处理的更改将作为下一个正常flush的一部分打包。当钩子在  :meth:`.Session.commit`  上检测到新变化时，
在此方面，一个计数器保证在循环100次后停止，以防止  :meth:`.SessionEvents.after_flush_postexec`  钩子在每次调用时不断添加要刷新的新状态。

.. _ session_persistence_mapper:

Mapper级别的Flush事件
------------------------------

除了flush-level挂钩之外，还有一组更细粒度的挂钩，因为它们以每个对象为基础进行调用，并根据flush过程内的插入、更新或删除对它们进行拆分。这些是映射器持久性挂钩，
它们也非常受欢迎，但是需要更加谨慎地处理这些事件，因为它们在已经进行的flush过程环境中进行; 许多操作无法安全进行。

事件为：

*  :meth:`.MapperEvents.before_insert` 
*  :meth:`.MapperEvents.after_insert` 
*  :meth:`.MapperEvents.before_update` 
*  :meth:`.MapperEvents.after_update` 
*  :meth:`.MapperEvents.before_delete` 
*  :meth:`.MapperEvents.after_delete` 

.. note::

   重要的是要注意，这些事件仅适用于  :ref:`session flush operation <session_flushing>` ，而不适用于ORM级别的INSERT/UPDATE/DELETE功能，
   描述在  :ref:`orm_expression_update_delete` 。要拦截ORM级别的DML，请使用  :meth:` _orm.SessionEvents.do_orm_execute`  事件。

每个事件都传递了  :class:`_orm.Mapper` ，映射的对象本身以及正在使用其发出INSERT，UPDATE或DELETE语句的  :class:` _engine.Connection` 。这些事件的吸引力很明显，
如果应用程序想要将某些活动与插入特定类型的对象同时进行，那么此钩子非常具体;与  :meth:`.SessionEvents.before_flush`  事件不同，
无需通过  :attr:`.Session.new`  集合搜索目标。但是，这些事件被调用时已经决定了代表要发出的每个INSERT，UPDATE，DELETE语句的flush计划，
并且在此阶段无法进行任何更改。因此，可能对给定对象的仅限于对象行的本地属性进行更改。对于任何其他更改对象或其他对象都不会进行更改。影响 :class:`.Session` 状态的操作，将无法正常运行。

以下映射器级别持久性事件不支持以下操作：

*  :meth:`.Session.add` 
*  :meth:`.Session.delete` 
* 映射的集合附加、添加、移除、删除、丢弃等操作。
* 映射的关系属性设置/删除事件，即：``someobject.related = someotherobject``。

之所以需要传递  :class:`_engine.Connection` ，是因为鼓励**简单的SQL操作直接在此执行**，直接在 :class:` _engine.Connection`上执行，比如在日志表中插入额外的行或者递增计数器。

还有许多每个对象单独执行的操作，它们根本不需要在刷新事件中处理。最常见的替代方法是在对象的 ``__init__()`` 方法中与一个对象一起建立附加状态，例如创建预关联与新对象关联的其他对象。使用如   :ref:`simple_validators`  中所述的验证器是另一种方法；这些函数可以拦截属性变化，并响应属性变化在目标对象上建立附加状态更改。使用这两种方法时，对象在到达刷新步骤之前就处于正确状态。

.. _session_lifecycle_events:

对象生命周期事件
-------------------

另一种使用事件的用例是跟踪对象的生命周期。它指的是在   :ref:`session_object_states`  中首次引入的状态。

所有上述状态均可以完全使用事件跟踪。每个事件都表示明确的状态转换，意味着起始状态和目标状态均作为跟踪的一部分。除了初始瞬态事件外，所有事件都是与  :class:`.Session` .Session` 对象关联：

    from sqlalchemy import event
    from sqlalchemy.orm import Session

    session = Session()


    @event.listens_for(session, "transient_to_pending")
    def object_is_pending(session, obj):
        print("新的待处理对象: %s" % obj)

或者与  :class:`.Session` .sessionmaker` 一起关联，后者可能是最有用的形式：

    from sqlalchemy import event
    from sqlalchemy.orm import sessionmaker

    maker = sessionmaker()


    @event.listens_for(maker, "transient_to_pending")
    def object_is_pending(session, obj):
        print("新的待处理对象: %s" % obj)

当然，监听器可以堆叠在一个函数之上，这可能是经常发生的情况。例如，要跟踪所有进入持久状态的对象：

        @event.listens_for(maker, "pending_to_persistent")
        @event.listens_for(maker, "deleted_to_persistent")
        @event.listens_for(maker, "detached_to_persistent")
        @event.listens_for(maker, "loaded_as_persistent")
        def detect_all_persistent(session, instance):
            print("对象现在是持久的: %s" % instance)

瞬态
^^^^^^^^^

所有映射的对象在构建时首先以  :term:`瞬态`  形式开始。在此状态下，对象是独立的，没有与任何  :class:` .Session` .Session`，但是如果需要拦截创建任何瞬态对象，最好使用  :meth:`.InstanceEvents.init`   方法，例如，要拦截特定声明基础的所有新对象：

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy import event


    class Base(DeclarativeBase):
        pass


    @event.listens_for(Base, "init", propagate=True)
    def intercept_init(instance, args, kwargs):
        print("新的瞬态对象: %s" % instance)

瞬态到待处理
^^^^^^^^^^^^^^^^^^^^

当通过  :meth:`.Session.add`   或  :meth:` .Session.add_all`  方法首次将瞬态对象与  :class:`.Session`  。某个对象也可以通过显式添加的引用对象的   :ref:` "级联" <unitofwork_cascades>`  作为  :class:`.Session`  的一部分。可以使用  :meth:` .SessionEvents.transient_to_pending`  事件来检测瞬态到待处理的转换：

    @event.listens_for(sessionmaker, "transient_to_pending")
    def intercept_transient_to_pending(session, object_):
        print("瞬态到待处理: %s" % object_)

待处理到持久
^^^^^^^^^^^^^^^^^^^^^

当刷新进行并且实例的INSERT语句执行时，  :term:`待处理`  对象变为  :term:` 持久` 。该对象现在具有标识键。使用  :meth:`.SessionEvents.pending_to_persistent`  事件来跟踪待处理到持久的转换：

    @event.listens_for(sessionmaker, "pending_to_persistent")
    def intercept_pending_to_persistent(session, object_):
        print("待处理到持久: %s" % object_)

待处理到瞬态
^^^^^^^^^^^^^^^^^^^^

如果在等待处理对象的任何INSERT语句执行之前调用了  :meth:`.Session.rollback`   方法，则  :term:` 待处理`  对象可以回退到  :term:`瞬态`  。持续生成实例
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果一个对象被删除，则它的状态将从持久状态转换为删除状态。对象的状态可以通过调用``Session.delete``方法来改变，但该对象直到``Session``刷新之后才会被标记为“被删除”。

从删除状态向持久状态转变是通过执行：meth：``Session.flush``或在刷新之前调用：meth：``Session.expunge``方法从而将该对象从会话中删除。使用：meth：``SessionEvents.deleted_to_persistent``事件将已删除到持久转换跟踪::

    @event.listens_for(sessionmaker, "deleted_to_persistent")
    def intercept_deleted_to_persistent(session, object_):
        print("object was re-inserted from DELETEd, now back in persistent state: %s" % object_)

.. note::

   如果对象是使用新的主键从磁盘重新加载的，则可以使一个对象从删除状态移回持久状态，因为需要在新的主键插入和SELECT之间执行匹配。这通常不是由用户代码做的，但可以通过调用插件系统中存在的``InstanceEvents.load``事件来追踪。

脱离中间状态
---------------

在很多情况下，保持对象状态的完整性要求追踪对象状态的中间转换。例如，跟踪从删除状态返回到持久状态的情况非常重要，因为这会在过程中自动更新数据库。

在每种情况下，迁移都可能是任意的；然而，在特定的情况下，跟踪可能是有意义的。例如，在审计或历史记录方案中，记录对象状态的变化可以是有益的。

除了直接追踪到回调函数和事件之外，还可以通过增加一个对象上的属性来跟踪。如果然后需要调整迁移，这样做时常有用的。

.. _`special tracker attribute`:

特殊追踪器属性 (Special Tracker Attribute)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
追踪器属性的示例是将特定状态标记为``deleted``或``modified``，或跟踪对象已经被修改了几次（使用``_modification_token``或类似）。

一个常见的做法是使用一个名为``_sa_instance_state``的类状态属性。``_sa_instance_state``是一个带有许多锅炉板属性的内部状态对象，每当转换到下一个状态时，它都会被更新。

例如，下面的示例为对象添加了一个名为``was_deleted``的属性，它当对象是删除状态时为``True``，否则为``False``。使用mapper级``set_committed_value``方法设置属性的默认状态，并使用``synthesized``事件覆盖属性。这在 ad-hoc 跨两次刷新跟踪该对象创建/删除的方案中很有用 :

.. sourcecode:: python

    from sqlalchemy.orm import mapper, Session
    from sqlalchemy.orm.attributes import (
        get_state_history,
        set_committed_value
    )
    from sqlalchemy.orm.events import (
        SessionEvents,
        mapper_configured,
        event
    )

    class CustomInstanceState(object):
        def __init__(self, was_deleted=False):
            self.was_deleted = was_deleted

        @property
        def deleted(self):
            return self.was_deleted

        def _check_modification(self, uow):
            if uow.attributes.get(('modified', id(self)), False):
                self._modification_token = uow.transaction.nested

    @mapper_configured.dispatch_for_events("instrument_class")
    def on_orm_instrument_class(mapper_, cls_):
        state_cls = mapper_._equivalent_columns['state']
        state_cls_attr = state_cls.property.key
        mapper_.add_property(
            'state_instance',
            state_cls_attr,
            synonym=state_cls_attr,
            comparator_factory=CustomInstanceState,
            instrumentation_cls=state_cls.instrumentation_cls
        )
        create_state = mapper_.dispatch._create_state
        create_deferred_state = mapper_.dispatch._create_deferred_state

        state_mapper_ = mapper_.dispatch['init_state_mapper']
        def _init_state(self):
            self.state_instance = self.state_instance.__class__()

        create_state.append(_init_state)
        create_deferred_state.append(_init_state)
        state_mapper_.add_listener(
            'before_insert', '_check_modification',
            retval=True, propagate=True
        )
        state_mapper_.add_listener(
            'before_update', '_check_modification',
            retval=True, propagate=True
        )

    @event.listens_for(Session, "synthesized")
    def link_custom_state(session, state, dict_, instance):
        state.instance = instance.state_instance
        state.load_path.append('state_instance')

    @event.listens_for(SessionEvents.modified_detached, propagate=True)
    def instance_was_modified(uow, instance):
        uow.attributes[('modified', id(instance))] = True
        if hasattr(instance.state_instance, '_modification_token'):
            instance.state_instance._modification_token = \
                uow.transaction.nested

    @event.listens_for(SessionEvents.loaded_as_persistent, propagate=True)
    def instance_loaded(instance, *args):
        mapper_ = object_mapper(instance.__class__)
        state_cls = mapper_._equivalent_columns['state']
        session = object_session(instance)
        state_data = get_state_history(
            session, instance, session.identity_map
        )
        set_committed_value(
            mapper_.class_manager,
            instance,
            state_cls.prop,
            CustomInstanceState(
                was_deleted=(
                    state_data.deleted and
                    not state_data.detached and
                    not state_data.pending
                ),
            )
        )

如果“synthesized”事件被发射（简单地调用“instance.state_instance”将发射此事件），则从 mapper.configured 事件使用的CustomInstanceState类中创建的CustomInstanceState的实例将被分配给实例的“state_instance”属性。

此属性和标准实例状态是一样的或更好的，因为它允许使用特定于实例的方式覆盖标准行为。如果初始迁移被建模为使用标准的ORM事件，而默认行为不符合某些要求，则使用此模式可以很容易地调整当前的代码路径（暂时使用内部API）被  :term:`deleted`  的对象可以在使用  :meth:` .Session.rollback`  方法进行回滚的事务中恢复到  :term:`persistent`  状态。使用  :meth:` .SessionEvents.deleted_to_persistent`  事件跟踪返回到持久状态的已删除对象::

    @event.listens_for(sessionmaker, "deleted_to_persistent")
    def intercept_deleted_to_persistent(session, object_):
        print("deleted to persistent: %s" % object_)

.. _session_transaction_events:

事务事件
--------
事务事件允许应用程序在   :class:`.Session`  级别上的事务边界发生时以及在
  :class:`.Session`  更改   :class:` _engine.Connection`  对象的事务状态时得到通知。

*  :meth:`.SessionEvents.after_transaction_create` ,
   :meth:`.SessionEvents.after_transaction_end`  - 这些事件以不特定于单个数据库连接的方式跟踪
    :class:`.Session`  的逻辑事务范围。这些事件旨在帮助集成事务跟踪系统（例如
  ``zope.sqlalchemy``）。当应用程序需要将某些外部范围与   :class:`.Session`  的事务范围对齐时，请使用这些
  事件。这些挂钩反映了   :class:`.Session`  的“嵌套”事务行为，因为它们
  跟踪逻辑“子事务”以及“嵌套”（例如 SAVEPOINT）事务。

*  :meth:`.SessionEvents.before_commit` ,  :meth:` .SessionEvents.after_commit` ,
   :meth:`.SessionEvents.after_begin` ,
   :meth:`.SessionEvents.after_rollback` ,  :meth:` .SessionEvents.after_soft_rollback`  -
  这些事件允许从数据库连接的角度跟踪事务事件。特别是  :meth:`.SessionEvents.after_begin` 
  是一个每个连接的事件；维护多个连接的   :class:`.Session`  将在当前事务中用到的每个连接中
  分别发出此事件。之后的回滚和提交事件是指 DBAPI 连接本身接收到回滚或提交指令的时间。

属性更改事件
------------
属性更改事件允许在对象上特定属性被修改时进行拦截。这些事件包括  :meth:`.AttributeEvents.set` 、
  :meth:`.AttributeEvents.append`   和  :meth:` .AttributeEvents.remove` 。这些事件非常有用，
特别是对于每个对象的验证操作；然而，使用一个“验证器”钩子通常更方便，
该钩子在幕后使用这些钩子；有关此背景的详细信息，请参见   :ref:`simple_validators` 。属性事件也在背引用的机制中。
使用属性事件的示例在   :ref:`examples_instrumentation`  中。

