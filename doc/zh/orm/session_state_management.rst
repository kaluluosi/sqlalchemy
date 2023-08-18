状态管理
================

.. _session_object_states:

对象状态简介
-------------------

了解一个实例在会话中可以具有的状态是有帮助的：

* **瞬态(Transient)** - 不在会话中的实例，也没有保存到数据库中；即它没有数据库身份。这个对象与 ORM 之间唯一的关系是其类上有一个与 :class:`_orm.Mapper` 相关联的映射器。

* **挂起(Pending)** - 当您  :meth:`~.Session.add`  一个短暂的实例时，它就变成了挂起状态。虽然它还没有实际刷新到数据库中，但在下一次刷新时将刷新到数据库中。

* **持久(Persistent)** - 在会话中存在一个记录在数据库中的实例。您可以通过刷新，使仍处于挂起状态的实例变成持久实例，或通过查询已存在的实例（或将其他会话中的持久实例移动到您本地会话中）获得持久实例。

* **删除(Deleted)** - 在刷新中被删除的实例，但是事务尚未结束。这种状态的对象实际上与“挂起”状态相反；当会话的事务提交时，对象将移动到分离状态。或者，当会话的事务回滚时，删除的对象会*返回*到持久状态。

* **分离(Detached)** - 与数据库中的记录对应或以前对应的实例，但目前没有任何会话。分离的对象将包含一个数据库标识标记，但是由于它与会话无关，无法确定这个数据库标识是否实际存在于目标数据库中。分离对象可以正常使用，但是它们无法加载未加载的属性或先前标记为“过期”的属性。

有关所有可能的状态转换的更深入的探讨，请参见   :ref:`session_lifecycle_events`  节，该节描述了每个转换以及如何以编程方式跟踪每个转换。


获取对象的当前状态
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

任何映射对象的实际状态可以在任何时候使用   :func:`_sa.inspect`  函数查看映射实例；此函数将返回相应的   :class:` .InstanceState`  对象，该对象管理对象的内部 ORM 状态。   :class:`.InstanceState`  提供了许多访问器，其中包括指示对象持久状态的布尔属性，包括：

*  :attr:`.InstanceState.transient` 
*  :attr:`.InstanceState.pending` 
*  :attr:`.InstanceState.persistent` 
*  :attr:`.InstanceState.deleted` 
*  :attr:`.InstanceState.detached` 

例如：：

    >>> from sqlalchemy import inspect
    >>> insp = inspect(my_object)
    >>> insp.persistent
    True

.. seealso::

    :ref:`orm_mapper_inspection_instancestate`  - 更多   :class:` .InstanceState`  示例

.. _session_attributes:

会话属性
----------------

  :class:`~sqlalchemy.orm.session.Session`  本身的行为有点像一个集合。可以使用迭代器接口访问所有当前存在的元素：

    for obj in session:
        print(obj)

并且可以使用常规“包含”语义测试其存在性：

    if obj in session:
        print("Object is present")

会话还跟踪所有新创建的（即挂起）对象、自上次加载或保存以来发生更改的所有对象（即“脏”对象）以及所有被标记为删除的对象：

    # 最近添加到 Session 中的挂起对象
    session.new

    # 有更改检测到的持久对象
    # (此集合在每次调用属性时都是实时创建的)
    session.dirty

    # 已被标记为删除的持久对象，通过session.delete(obj)方法
    session.deleted

    # 所有持久对象的字典，以其身份键为键
    session.identity_map

(Documentation:  :attr:`.Session.new` ,  :attr:` .Session.dirty` ,
  :attr:`.Session.deleted`  ,  :attr:` .Session.identity_map` ).


.. _session_referencing_behavior:

会话引用行为
--------------------------

会话中的对象是*弱引用*的。这意味着当它们在外部应用程序中解除引用时，在  :class:`~sqlalchemy.orm.session.Session`  中它们也会失去作用，并且受 Python 解释器的垃圾收集的影响。这其中的例外包括挂起对象、标记为删除的对象或具有挂起更改的持久对象。在完全刷新后，这些集合都为空，并且所有对象再次成为弱引用。

使  :class:`.Session`  中的对象保持强引用，通常只需要简单的方法。外部管理强引用行为的示例包括将对象加载到以其主键为键的本地字典中，或将其放入列表或集合中以使其保持引用状态。如果需要，这些集合可以与  :class:` .Session`  关联。  :attr:`.Session.info`   字典。

事件驱动方法也是可行的。一个提供所有对象“强引用”行为的简单方案，当它们保持在  :term:`persistent`  状态内时，如下所示：

::

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

在上面的代码中，我们拦截  :meth:`.SessionEvents.pending_to_persistent` ,
  :meth:`.SessionEvents.detached_to_persistent`  ,
  :meth:`.SessionEvents.deleted_to_persistent`   和
  :meth:`.SessionEvents.loaded_as_persistent`   事件挂钩，以拦截对象进入  :term:` persistent`  转换的过程，以及
  :meth:`.SessionEvents.persistent_to_detached`   和
  :meth:`.SessionEvents.persistent_to_deleted`  挂钩截取对象离开持久状态的时刻。

可以对任何  :class:`.Session` .Session` 的强引用行为：

::

    from sqlalchemy.orm import Session

    my_session = Session()
    strong_reference_session(my_session)

也可以对任何 :class:`.sessionmaker` 进行调用：

::

    from sqlalchemy.orm import sessionmaker

    maker = sessionmaker()
    strong_reference_session(maker)


.. _unitofwork_merging:

合并
----

  :meth:`~.Session.merge`   将状态从外部对象传输到一个新的或已存在的实例中，同时还会将传入数据对比数据库的状态，产生一个历史流，该流将被应用于下一个刷新，或者可以被设置为生成简单的状态“转移”，而不会产生更改历史或访问数据库。使用方法如下：

::

    merged_object = session.merge(existing_object)

当给定一个实例时，它会按照以下步骤进行：

* 它检查实例的主键。如果存在，则尝试在本地标识映射中定位该实例。如果将“load=True”标志保留在默认状态，如果找不到本地实例，则还会检查该主键是否存在于数据库中。
* 如果给定的实例没有主键，或者没有使用给定的主键找到实例，则创建一个新实例。
* 然后将给定实例的状态复制到定位/新创建的实例上。对于源实例中存在的属性值，该值将传输到目标实例。对于源实例上不存在的属性值，目标实例上相应的属性将从内存中  :term:`过期` ，这会丢弃目标实例中该属性的任何本地存在的值，但不会直接修改该属性的存储在数据库中的值的记忆。
* 如果将“load=True”标志保留在默认状态，则此副本过程将发出事件，并为来源对象上存在的每个属性加载目标对象未加载的集合，从而可以将传入状态与数据库中存在的状态进行协调。如果传递“load”操作，则直接将传入数据“盖章”，而不产生任何历史记录。
* 将该操作级联到相关对象和集合中，如所示``merge``级联 (参见：ref:`unitofwork_cascades`)。
* 返回新实例。

使用  :meth:`~.Session.merge` ，给定的“源”实例不会修改，也不会与目标   :class:` .Session` .Session` 对象中。 :meth:`~.Session.merge` 用于将任何类型的对象结构的状态复制到新会话，而不考虑其起源或当前会话关联。以下是一些示例：

* 一个读取对象结构并希望将其保存到数据库的应用程序可能会解析文件，构建结构，然后使用  :meth:`~.Session.merge`  将其保存到数据库中，确保文件中的数据用于制定该结构每个元素的主键。稍后，当文件更改时，可以重新运行同一进程，生成略微不同的对象结构，然后再次将其"合并"， :class:` ~sqlalchemy.orm.session.Session`将自动更新数据库以反映这些更改，通过主键从数据库中加载每个对象，然后使用新状态更新其状态。

* 一个应用程序正在将对象存储在内存中的缓存中，许多  :class:`.Session` ~.Session.merge` 。将一个已存在的对象转移到其他session或者从缓存中查询对象时， :meth:`~.Session.merge` 将会更新有相同主键的对象实例的属性。同时它也是一种安全的非插入式方法，能够交换不同会话中的状态对象的属性。 ​

关键字参数 load:是否需要映射对象的状态到数据库中,这在缓存查询的情况下很有用。另外，还有一种  :meth:`_query.Query.merge_result`  的“批量”版本，该方法被设计用于缓存扩展的   :class:` _query.Query`  对象中——请参阅 :ref:`examples_caching` 部分。

​ **使用技巧** **~~~~~~~~~~**

  :meth:`~.Session.merge`   对于许多情况非常有用。但是，它处理着临时对象和永久对象以及状态自动转移之间复杂的边界。这个过程中有各种各样的情况存在，这些情况通常需要更谨慎的方法去处理对象的状态。 merge的一般问题通常涉及一些关于被传递给  :meth:` ~.Session.merge`  方法的对象的意外状态。 我们将使用 User 和 Address 对象来举一个具体的例子：

```python
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
```

首先，我们需要创建 User 对象，有一个已经存在的 Address：

```python
u1 = User(name="ed", addresses=[Address(email_address="ed@ed.com")])
session.add(u1)
session.commit()
```

然后，我们创建了一个 Address 对象 a1，该对象位于 Session 之外，我们希望将它合并到现有的 Address 上：

```python
existing_a1 = u1.addresses[0]
a1 = Address(id=existing_a1.id)
```

如果我们做了这个操作：

```python
a1.user = u1
a1 = session.merge(a1)
session.commit()
```

会出现一个奇怪的问题，为什么呢？我们没有仔细考虑级联关系。a1.user 对一个持久化的对象的赋值会级联到影响到 User.addresses 的 backref 上，当作新添加的对象。这样我们就有了两个 Address 对象了：

```python
a1 = Address()
a1.user = u1
a1 in session
True
existing_a1 in session
True
a1 is existing_a1
False
```

以上，我们的 a1 已经在 session 中了。随后的  :meth:`~.Session.merge`  操作本质上什么也没有做。Cascade 可以通过在   :func:` _orm.relationship`  上设置参数  :paramref:`_orm.relationship.cascade`  来进行配置，虽然在这种情况下，它的行为通常是非常方便的。通常的解决方案是不要将` a1.user`分配给目标会话中已经存在的对象。

  :func:`_orm.relationship`  的 cascade_backrefs=False 参数也可以通过防止 Address 通过` a1.user = u1` 添加到会话中。

对于其他不是预期的 merge 状态:

```python
a1 = Address(id=existing_a1.id, user_id=u1.id)
a1.user = None
a1 = session.merge(a1)
session.commit()
```

上面的代码中，user 的赋值优先于 user_id 的赋值，最后就会导致 user_id 等于 None，因此产生错误。

大多数  :meth:`~.Session.merge`  问题可以检查对象是否过早地在会话中。（是否出现了两个相同的对象）或者，对象上是否有我们不想要的状态? 通过检查 ` `__dict__`` 可以快速确定:

    >>> a1 = Address(id=existing_a1, user_id=user.id)
    >>> a1.user
    >>> a1.__dict__
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x1298d10>,
        'user_id': 1,
        'id': 1,
        'user': None}
    >>> # 我们不希望 user=None 被合并，将其移除
    >>> del a1.user
    >>> a1 = session.merge(a1)
    >>> # 成功
    >>> session.commit()

分离
----

Expunge从Session中删除对象，将持久实例发送到分离状态，将挂起实例发送到短暂状态:

.. sourcecode:: python+sql

    session.expunge(obj1)

要删除所有项目，请调用  :meth:`~.Session.expunge_all` 
(此方法以前称为 ``clear()``)。

.. _session_expire:

刷新/到期
----------

  :term:`到期`  是指内部保持的数据库持久化数据被擦除，下次访问这些属性时，
将会发出SQL查询，以便从数据库中刷新该数据。

当我们谈论数据的过期时，通常我们指的是处于  :term:`持久`  状态的对象。
例如，如果我们按以下方式加载一个对象::

    user = session.scalars(select(User).filter_by(name="user1").limit(1)).first()

上述 `User` 对象是持久的，具有一系列属性; 如果我们查看其 ``__dict__`` 中的状态，则会发现已加载::

    >>> user.__dict__
    {
      'id': 1, 'name': u'user1',
      '_sa_instance_state': <...>,
    }

其中 ``id`` 和 ``name`` 引用数据库中的那些列。
``_sa_instance_state`` 是SQLAlchemy内部使用的非数据库持久值(它引用该实例的
  :class:`.InstanceState` )。虽然与本节内容不直接相关，但如果我们想获取它，应使用
  :func:`_sa.inspect`  函数来访问它。

此时，我们 ``User`` 对象的状态与加载的数据库行的状态相匹配。但是，通过调用诸如
  :meth:`.Session.expire`   这样的方法来过期对象时，我们会看到状态被删除::

    >>> session.expire(user)
    >>> user.__dict__
    {'_sa_instance_state': <...>}

我们发现，虽然内部的 “状态” 仍然存在，但对应于 ``id`` 和 ``name`` 列的值都已消失。
如果我们访问这些列之一，并正在查看SQL，则会看到此操作:

.. sourcecode:: pycon+sql

    >>> print(user.name)
    {execsql}SELECT user.id AS user_id, user.name AS user_name
    FROM user
    WHERE user.id = ?
    (1,)
    {stop}user1

上述，访问过期属性 ``user.name`` 时，ORM启动了一个  :term:`延迟加载`  ，以检索
该用户所引用的用户行的最新状态。之后， ``__dict__`` 再次被填充::

    >>> user.__dict__
    {
      'id': 1, 'name': u'user1',
      '_sa_instance_state': <...>,
    }

.. 注意:: 虽然我们正在查看 ``__dict__`` 的内容以查看SQLAlchemy如何处理对象属性，
   但是我们 **不应该直接修改** SQLAlchemy ORM正在维护的那些属性的内容(其他在SQLA领域之外的属性可以更改)。
   这是因为SQLAlchemy使用  :term:`描述符`  以跟踪我们对对象所做的更改，而当我们直接修改 ` `__dict__``
   时，ORM 将无法跟踪我们更改了什么。

  :meth:`~.Session.expire`   和  :meth:` ~.Session.refresh`  的另一个关键行为是，
所有未flush的更改都将被丢弃。也就是说，
如果我们修改了 ``User`` 的某个属性::

    >>> user.name = "user2"

但然后我们在不首先调用  :meth:`~.Session.flush`  的情况下调用  :meth:` ~.Session.expire` ，
我们挂起的值 ``'user2'`` 就被丢弃了:

    >>> session.expire(user)
    >>> user.name
    'user1'

  :meth:`~.Session.expire`   方法可用于将一个实例的所有ORM映射属性标记为“已过期”::

    # 将 obj1 的所有ORM映射属性都过期
    session.expire(obj1)

也可以将其传递一个字符串属性名称列表，以引用要标记为过期的特定属性::

    # 仅过期 obj1.attr1, obj1.attr2 的属性
    session.expire(obj1, ["attr1", "attr2"])

  :meth:`.Session.expire_all`   方法允许我们将   :class:` .Session`  中包含的所有对象
都标记为“已过期”::

    session.expire_all()  :meth:`~.Session.refresh`  方法具有类似的接口，但是不是过期，而是立即发出一个查询以获取对象的行::

    # 重新加载obj1的所有属性
    session.refresh(obj1)

  :meth:`~.Session.refresh`   还接受一个字符串属性名称列表，但不同于  :meth:` ~.Session.expire` ，它预期至少有一个名称是列映射属性的名称::

    # 重新加载 obj1.attr1, obj1.attr2
    session.refresh(obj1, ["attr1", "attr2"])

.. tip::

    另一种常用的刷新方法是使用ORM的   :ref:`orm_queryguide_populate_existing`  功能，
    可在  :term:`2.0 style`  查询中使用   :func:` _sql.select`  以及在  :term:`1.x style`  查询中
    的  :meth:`_orm.Query.populate_existing`  方法中使用。使用这种执行方法，
    语句结果集中返回的所有ORM对象将使用来自数据库的数据进行刷新::

        stmt = (
            select(User)
            .execution_options(populate_existing=True)
            .where((User.name.in_(["a", "b", "c"])))
        )
        for user in session.execute(stmt).scalars():
            print(user)  # 将使用从查询返回的列进行刷新

    有关详细信息，请参见   :ref:`orm_queryguide_populate_existing` 。


实际加载了什么
~~~~~~~~~~~~~~

当标记为  :meth:`~.Session.expire`  或使用  :meth:` ~.Session.refresh`  加载对象时，
所发出的SELECT语句会根据几个因素而异，包括：

* 过期属性的加载仅从 **列映射属性** 触发。任何类型的属性都可以标记为过期，包括   :func:`_orm.relationship` 
  - 映射属性，但访问过期的   :func:`_orm.relationship`  属性仅会为该属性发出加载操作，使用标准的关系导向的
  懒加载。即使是过期的列导向属性，在此操作的一部分中也不会加载，而是在访问任何列导向属性时加载。

* 在访问已过期的基于列的属性时，不会响应   :func:`_orm.relationship`  - 映射属性的加载。

* 关于关系，与  :meth:`.Session.expire`  相比，  :meth:` .Session.refresh`  在不是列映射属性的属性方面更为严格。
  调用  :meth:`.Session.refresh`  并传递仅包括关系映射属性的名称列表将会导致错误。
  在任何情况下，非急加载   :func:`_orm.relationship`  属性都不会包括在任何刷新操作中。

* 通过  :paramref:`_orm.relationship.lazy`  参数配置为“急加载”的属性，将在
   :meth:`~.Session.refresh`  的情况下加载，如果没有指定属性名称，或者
  如果它们的名称包括在要刷新的属性列表中。

* 配置为   :func:`.deferred`  的属性通常不会在过期属性加载或刷新期间加载。
  一个未加载的被   :func:`.deferred`  修饰的属性当直接访问或作为一个未加载属性组的一部分时，才会单独加载。

* 对于加载时按需加载的属性，继承表映射将发出一个SELECT，该SELECT通常仅包括存在未加载属性的表。
  在这里，该操作足够复杂，可以仅加载父表或子表，例如，如果最初过期的列的子集仅涵盖这两个表中的一个。

* 当在继承表映射上使用  :meth:`~.Session.refresh`  时，所发出的SELECT将类似于在目标对象类上使用  :meth:` .Session.query`  的情况。
  这通常是映射的组成部分的所有表。


何时过期或刷新
~~~~~~~~~~~~~~~~

  :class:`.Session`  当事务结束时自动使用过期功能。这意味着每当调用  :meth:` .Session.commit` 
或  :meth:`.Session.rollback`  时，   :class:` .Session`  中的所有对象都会被过期，使用等效于  :meth:`.Session.expire_all`  方法的功能。
其原因是事务结束是一个划分点，在这个点上没有更多的上下文可用以知道数据库的当前状态，
因为其他任何数量的事务都可能会影响它。只有在开始新事务时，我们才能再次访问数据库的当前状态，
此时任何数量的更改都可能已经发生。

.. sidebar:: 事务隔离

    当然，大多数数据库都能够处理多个事务，甚至包括涉及相同数据行的事务。当
    关系数据库处理涉及相同表或行的多个事务时，这就是数据库的  :term:`隔离`  属性发挥作用的时候了。
    不同数据库的隔离行为差别很大，即使在单个数据库上也可以配置为以不同的方式进行操作。在一个事务的范围内，  :class:`.Session`  无法完全预测第二次发出的相同的 SELECT 语句是否会返回我们已经拥有的数据，还是会返回新的数据，这与所谓的  :term:` 隔离级别`  设置有关。因此，最好的假设是，在事务的范围内，除非它已知发出了一个修改特定行的 SQL 表达式，否则没有必要重新刷新行，除非明确要求这样做。

  :meth:`.Session.expire`   和  :meth:` .Session.refresh`  方法用于在数据当前状态可能过时的情况下，强制对象从数据库重新加载其数据的情况下。这种情况可能包括：

* 一些 SQL 已经在 ORM 的对象处理范围之外在事务中发出，例如，如果使用  :meth:`.Session.execute`  方法发出  :meth:` _schema.Table.update`  构造；

* 如果应用程序试图获取已知在并发事务中被修改的数据，并且也已知生效的隔离规则允许此数据可见。

第二个要点有一个重要的警告："还知道生效的隔离规则允许此数据可见。"这意味着不能假定外部数据库连接上的更新将在本地透明可见，但在许多情况下不会。这就是为什么，如果想要使用  :meth:`.Session.expire`  或  :meth:` .Session.refresh`  在进行 ongoing 事务之间的数据查看时，必须理解生效的隔离行为的原因。

.. seealso::

    :meth:`.Session.expire` 

    :meth:`.Session.expire_all` 

    :meth:`.Session.refresh` 

      :ref:`orm_queryguide_populate_existing`  - 使任何 ORM 查询能够刷新对象，就像通常加载对象一样，在身份映射中刷新所有匹配的对象。

     :term:`isolation`  - 隔离级别及其解释，并包含到维基百科的链接。

    `SQLAlchemy Session In-Depth <https://techspot.zzzeek.org/2012/11/14/pycon-canada-the-sqlalchemy-session-in-depth/>`_ - 视频和幻灯片讲述了关于对象生命周期的深入讨论，包括数据过期的作用。