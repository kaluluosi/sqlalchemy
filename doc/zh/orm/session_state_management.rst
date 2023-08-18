状态管理
===========

.. _session_object_states:

对象状态简介
--------------

了解实例在会话/session中的可能状态将有所帮助：

* **临时**（Transient）- 不在会话/session中，未保存到数据库中；即没有数据库标识符。一个对象与ORM的唯一相关关系是其类与:class:`_orm.Mapper`关联。
  
* **挂起**（Pending）- 当你使用:meth:`~.Session.add`将临时实例添加到会话/session中时，它变成挂起状态。它还未被持久化到数据库中，但在下一个 flush 发生时会被刷新到数据库中。
  
* **持久**（Persistent）- 一个已经存在于会话/session中，且在数据库中拥有记录的实例。使一个实例持久的方法可以是一个 flush 操作以使所有的挂起实例变成持久实例，或者查询数据库以找到现有实例（或从其他会话中添加持久实例到本地会话中）。

* **删除**（Deleted）- 在 flush 期间被删除的实例，但事务还未完成。对象处于这一状态相当于位于"挂起"状态的反面；当会话/session的事务提交时，对象将转移到分离状态。或者当会话的事务回滚时，删除的对象将“回归”到持久状态。

* **分离**（Detached）- 对应于一个在之前与数据库中的记录对应（或者现在还是对应），但当前不位于任何会话/session中的实例。分离的对象实例会包含数据库标识符，但由于不与会话/session关联，所以不知道这个标识符在目标数据库中是否存在。分离的对象实例可以正常使用，但是它们无法加载未加载的属性或以前标记为“过期”的属性。

关于所有可能状态转换更深入的探讨，详见 :ref:`session_lifecycle_events` 这部分，它描述了每个转换以及如何在程序中跟踪每个转换。

获得对象的当前状态
~~~~~~~~~~~~~~~~~~~~~~

可以使用:meth:`~sqlalchemy.inspect`方法在任何时候查看任何映射对象的状态；这个方法将返回相应的 :class:`.InstanceState` 对象，管理对象的内部ORM状态。除其他访问器之外， :class:`.InstanceState` 还提供布尔属性，指示对象的持久性状态，包括：

* :attr:`.InstanceState.transient`
* :attr:`.InstanceState.pending`
* :attr:`.InstanceState.persistent`
* :attr:`.InstanceState.deleted`
* :attr:`.InstanceState.detached`

例如： ::

  >>> from sqlalchemy import inspect
  >>> insp = inspect(my_object)
  >>> insp.persistent
  True

.. seealso::

  :ref:`orm_mapper_inspection_instancestate` - 进一步关于:class:`.InstanceState`的例子。

.. _session_attributes:

Session 属性
------------------

:class:`~sqlalchemy.orm.session.Session` 本身可以看作一组类似于“集合”的对象。可以使用迭代器接口访问所有项目： ::

    for obj in session:
        print(obj)

可以使用常规"包含"语义测试存在性： ::

    if obj in session:
        print("Object is present")

会话/session还跟踪所有新创建（即挂起）的对象，所有自上次加载或保存后发生更改的对象（即“脏”的对象）
以及所有已标记为已删除的对象： ::

    # 会话/session中最近添加的挂起对象
    session.new

    # 当前检测到有更改的持久对象（该集合现在是在每次调用属性时动态创建的）
    session.dirty

    # 标记为删除的持久对象，例如通过session.delete(obj)标记的。
    session.deleted

    # 包含所有持久对象的字典，以它们的标识键来索引
    session.identity_map

（文档： :attr:`.Session.new`，:attr:`.Session.dirty`，
:attr:`.Session.deleted`，:attr:`.Session.identity_map`）。

.. _session_referencing_behavior:

会话引用行为
------------------------

会话/session中的对象是 *弱引用* 的。这意味着当它们在外部应用程序中反引用时，它们也会从:class:`~sqlalchemy.orm.session.Session`中移出范围，并受Python解释器的垃圾回收。这其中的例外情况包括标记为已删除的对象、标记为挂起的对象、以及有挂起更改的持久化对象。在 flush 后，这些集合都会被清空，所有对象再次成为弱引用。

使会话/:class:`.Session` 中的对象保持强引用的通常简单方法可能会很有用。外部管理强引用行为的示例包括将对象加载到一个本地词典中，以其主键为键，或者在它们需要保持引用的时间段内将它们加载到列表或集合中。如果需要，这些集合可以与一个 :class:`.Session` 关联，将它们放入 :attr:`.Session.info` 字典中。

也可以选择采用事件驱动的方式。为了使所有对象在一直处于 :term:`persistent` 状态时都保持强引用行为，实现其中一种简单做法是： ::

    from sqlalchemy import event


    def strong_reference_session(session):
        @event.listens_for(session, "pending_to_persistent")
        @event.listens_for(session, "deleted_to_persistent")
        @event.listens_for(session, "detached_to_persistent")
        @event.listens_for(session, "loaded_as_persistent")
        def strong_ref_object(sess, instance):
            if "refs" not in sess.info:
                sess.info["refs"] = refs = set()
            else:
                refs = sess.info["refs"]

            refs.add(instance)

        @event.listens_for(session, "persistent_to_detached")
        @event.listens_for(session, "persistent_to_deleted")
        @event.listens_for(session, "persistent_to_transient")
        def deref_object(sess, instance):
            sess.info["refs"].discard(instance)

上述的方法中，我们捕获了 :meth:`.SessionEvents.pending_to_persistent`、:meth:`.SessionEvents.detached_to_persistent`、
:meth:`.SessionEvents.deleted_to_persistent` 和 :meth:`.SessionEvents.loaded_as_persistent`
等事件钩子，以拦截对象进入 :term:`persistent` 过渡的情况，以及 :meth:`.SessionEvents.persistent_to_detached` 和 :meth:`.SessionEvents.persistent_to_deleted` 钩子，拦截对象离开持久状态的情况。

我们可以在任何 :class:`.Session` 中调用上述函数来提供基于每个 :class:`.Session` 的强引用行为： ::

    from sqlalchemy.orm import Session

    my_session = Session()
    strong_reference_session(my_session)

也可以在任何 :class:`.sessionmaker` 中调用该函数： ::

    from sqlalchemy.orm import sessionmaker

    maker = sessionmaker()
    strong_reference_session(maker)

.. _unitofwork_merging:

合并
------

:meth:`~.Session.merge`将状态从外部对象传输到会话/:class:`.Session`中的新实例或已存在实例中。它还会根据数据库的状态来对入站数据进行调解，并产生一个历史记录流，该流将被应用到下一个 flush 中，或者可以要求产生一个简单的状态“传输”，而不会产生变更记录或访问数据库。使用方式如下：

    merged_object = session.merge(existing_object)

当给出一个实例，其遵循以下步骤：

* 它检查实例的主键。如果存在，则尝试在本地标识映射中找到该实例。如果将 ``load=True`` 标志保留为其默认值，则在本地未找到主键时还会检查数据库。
* 如果给定的实例没有主键，或者未找到具有给定主键的实例，则创建一个新实例。
* 然后，将给定实例的状态复制到已定位/新已创建的实例上。对于存在于来源实例上的属性值，该值将传输到目标实例。对于不在源实例上存在的属性值，从目标实例删除对应属性的当前值（即，该属性会被标记为过期，将目标对象中相关的本地值丢弃，但不会改变该属性在数据库中已持久化的值）。

  如果将 ``load=True`` 标志保留为其默认值，则此复制过程会生成事件并为来源对象的每个属性加载目标对象未加载的收集，以便数据库中的现有状态可被纳入已到达/incoming 状态中。如果传递了 ``load=False``，则生成的数据将直接打标签，而不生成任何更改记录（即使不存在任何目标对象上的历史记录）。
* 这个操作被级联到相关对象和收集上，就像所指示的那样，通过:paramref:`_orm.relationship`

  merge 级联（参见 :ref:`unitofwork_cascades`）。

* 返回新的实例。


用于 :meth:`~.Session.merge` 的“来源”实例未被修改，且未与目标:class:`.Session`相关联，仍可用于与任意数量的其他:class:`.Session`对象合并。
:meth:`~.Session.merge`对于许多目的来说是非常有用的方法。然而，它处理的是瞬时/分离对象和那些持久对象复杂边界，以及自动传输状态所涉及的场景多种多样，通常需要更仔细的方法来处理对象的状态。:meth:`~.Session.merge`通常涉及一些不可预期的状态以及传递给它的对象的问题。

让我们使用经典的用户和地址对象示例。 ::

    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50), nullable=False)
        addresses = relationship("Address", backref="user")


    class Address(Base):
        __tablename__ = "address"

        id = mapped_column(Integer, primary_key=True)
        email_address = mapped_column(String(50), nullable=False)
        user_id = mapped_column(Integer, ForeignKey("user.id"), nullable=False)

假设已经有一个拥有一个“Address”对象的“User”对象，而这个对象已经持久化: ::

    >>> u1 = User(name="ed", addresses=[Address(email_address="ed@ed.com")])
    >>> session.add(u1)
    >>> session.commit()

我们现在创建了一个在会话之外的“a1”对象，并希望将其合并到现有的“Address”上:：
    >>> existing_a1 = u1.addresses[0]
    >>> a1 = Address(id=existing_a1.id)

如果我们进行了这些操作，将会出现一个惊喜:：

    >>> a1.user = u1
    >>> a1 = session.merge(a1)
    >>> session.commit()
    sqlalchemy.orm.exc.FlushError: New instance <Address at 0x1298f50>
    with identity key (<class '__main__.Address'>, (1,)) conflicts with
    persistent instance <Address at 0x12a25d0>

为什么会这样呢？我们在级联中不太小心，将``a1.user``分配给了已存在的对象，因为它是一个持久化对象，级联将返回到了 users.addresses，使``a1``对象以为自己已经添加进了地址列表中。然而这样它还会再次进行添加的操作，这会导致一个错误。上述问题通常解决方法是像上文中描述的一样，在所需的情况下选择不将持久化对象分配给“a1.user”。
使用 :meth:`~.Session.expunge_all` 方法可以移除 :class:`.Session` 中的所有项目。因此，最佳猜测是，在事务的范围内，除非已知发出一个SQL表达式以修改特定行，否则没有必要刷新行，除非明确告知这样做。在那些已知当前数据状态可能过期的情况下，可以使用:meth:`. Session.expire`和:meth:`. Session.refresh`方法，强制对象重新从数据库加载数据。这些情况可能包括：

*在ORM对象处理范围之外的事务中发出了一些SQL，例如使用:meth:`. Session.execute`方法发出:meth:`_schema.Table.update`构造;

*如果应用程序试图获取已知在并发事务中已修改的数据，并且还知道生效的隔离规则允许查看此数据。

第二个要点有一个重要的警告，即“也知道在生效的隔离规则允许查看此数据。”这意味着不能假设在另一个数据库连接上发生的UPDATE在本地这里仍然可见；在许多情况下，它是不可见的。这就是为什么如果希望使用:meth:`. Session.expire`或:meth:`. Session.refresh`来在进行中的事务之间查看数据，则必须理解有效的隔离行为。

.. seealso::

    :meth:`.Session.expire`

    :meth:`.Session.expire_all`

    :meth:`.Session.refresh`

    :ref:`orm_queryguide_populate_existing`-使任何ORM查询都可以刷新对象，就像正常加载对象一样，在身份映射中刷新所有匹配对象以匹配SELECT语句的结果。

    :term:`isolation`-带有指向Wikipedia的链接的隔离词表解释。

    The SQLAlchemy Session In-Depth-有关对象生命周期的深入讨论，包括数据过期的作用的视频+幻灯片。