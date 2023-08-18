.. _unitofwork_cascades:

级联操作
========

映射器支持在   :func:`~sqlalchemy.orm.relationship`  上配置可配置的级联行为。这个级联操作指的是，在特定   :class:` .Session`  上针对 "parent" 对象执行的操作应该如何传播到该关系所引用的项目（例如，"child" 对象），并且受到  :paramref:`_orm.relationship.cascade`  选项的影响。

级联的默认行为仅限于级联所谓的   :ref:`cascade_save_update`  和   :ref:` cascade_merge`  设置。对于级联的典型 "可选" 设置是添加   :ref:`cascade_delete`  和   :ref:` cascade_delete_orphan`  选项。这些设置适用于相关对象，只要它们附加到其父级上，就存在，并且在否则删除时。

使用  :paramref:`_orm.relationship.cascade`  选项在   :func:` ~sqlalchemy.orm.relationship`  上配置级联行为：

    class Order(Base):
        __tablename__ = "order"

        items = relationship("Item", cascade="all, delete-orphan")
        customer = relationship("User", cascade="save-update")

对于 backref，同样可以使用相同的标志和   :func:`~.sqlalchemy.orm.backref`  函数，在其最终将其参数提供回   :func:` ~sqlalchemy.orm.relationship` ：

    class Item(Base):
        __tablename__ = "item"

        order = relationship(
            "Order", backref=backref("items", cascade="all, delete-orphan")
        )

.. sidebar:: Cascade 的起源

    SQLAlchemy 的级联行为的概念以及配置它们的选项，主要来源于 Hibernate ORM 中类似的功能。Hibernate 在几个地方使用 "级联（cascade）"，如在 `Example: Parent/Child <https://docs.jboss.org/hibernate/orm/3.3/reference/en-US/html/example-parentchild.html>`_ 中。如果级联操作令人困惑，我们将参考他们的结论，表明 "我们刚刚涵盖的这些部分可能有一些混淆。但是，在实践中，它们都能很好地工作。"

  :paramref:`_orm.relationship.cascade`   的默认值为 ` `save-update, merge``。该参数的典型替代设置为 ``all`` 或更常见的是 ``all, delete-orphan``。``all`` 符号是 ``save-update, merge, refresh-expire, expunge, delete`` 的同义词，并且结合使用时，与 ``delete-orphan`` 一起使用表示子对象应在所有情况下随父对象而动，并一旦不再与该父对象关联就将其删除。

.. warning::``all`` 级联选项暗示使用   :ref:`cascade_refresh_expire`  级联设置，当使用   :ref:` asyncio_toplevel`  扩展时可能不适用，因为它会比通常在显式 IO 上下文中适当的更积极地过期相关对象。有关更多背景信息，请参见   :ref:`asyncio_orm_avoid_lazyloads`  中的注释。

可以在以下子部分中找到可指定  :paramref:`_orm.relationship.cascade`  参数的可用值列表。

.. _cascade_save_update:

save-update
-----------

``save-update`` 级联表示，当通过  :meth:`.Session.add`  将对象放置到   :class:` .Session`  时，使用此   :func:`_orm.relationship`  引用相关对象的所有对象也应添加到同一个   :class:` .Session`  中。假设我们有一个具有两个相关对象 ``address1`` 和 ``address2`` 的对象 ``user1``：

    >>> user1 = User()
    >>> address1, address2 = Address(), Address()
    >>> user1.addresses = [address1, address2]

如果我们将 ``user1`` 添加到   :class:`.Session`  中，它也会隐式添加 ` `address1``，``address2``：

    >>> sess = Session()
    >>> sess.add(user1)
    >>> address1 in sess
    True

``save-update`` 级联还影响到已经存在于   :class:`.Session`  的对象的属性操作。如果我们向 ` `user1.addresses`` 集合中添加第三个对象 ``address3``，它将成为该   :class:`.Session`  的状态的一部分：

    >>> address3 = Address()
    >>> user1.addresses.append(address3)
    >>> address3 in sess
    True

从集合中删除项目或取消将对象从标量属性中取消关联时，``save-update`` 级联可能会出现意外的行为。在某些情况下，孤立的对象仍然可能被拖入前父级的   :class:`.Session`  中，以使 flush 过程可以适当地处理。这种情况通常只在从一个   :class:` .Session`  中删除对象并添加到另一个   :class:`.Session`  的场景下出现：

    >>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
    >>> address1 = user1.addresses[0]
    >>> sess1.close()  # user1、address1 不再与 sess1 有关联
    >>> user1.addresses.remove(address1)  # address1 不再与 user1 有关联
    >>> sess2 = Session()
    >>> sess2.add(user1)  # ... 但它仍会添加到新会话中，
    >>> address1 in sess2  # 因为它仍然处于"待处理"状态
    True

默认情况下，``save-update`` 级联处于开启状态，并且经常被认为是理所当然的；它通过允许一次调用  :meth:`.Session.add`  来一次性注册整个对象结构来简化代码。虽然它可以被禁用，但通常没有必要这样做。

.. _back_populates_cascade:

.. _backref_cascade:

双向关系中 save-update 级联的行为
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在双向关系的上下文中，``save-update`` 级联仅按 **单向** 方向进行处理，即在使用  :paramref:`_orm.relationship.back_populates`  或  :paramref:` _orm.relationship.backref`  参数创建互相引用的两个   :func:`_orm.relationship`  对象时。

当将未与   :class:`_orm.Session`  关联的对象分配给与   :class:` _orm.Session`  关联的父对象上的属性或集合时，该对象将自动添加到同一个   :class:`_orm.Session`  中。但是，在相反方向上执行的相同操作将不会自动触发此效果。即当一个未与   :class:` _orm.Session`  关联的对象分配为具有与   :class:`_orm.Session`  关联的子对象的属性时，不会自动将该父对象添加到相同的   :class:` _orm.Session`  中。这种行为的总体主题被称为 "级联反向引用"，并代表从 SQLAlchemy 2.0 开始的行为更改。

例如，假设我们有一个 ``Order`` 对象的映射，该对象通过关系 ``Order.items`` 双向关联一系列 ``Item`` 对象，并且 ``Item.order`` 也引用 ``Order`` 对象：

    mapper_registry.map_imperatively(
        Order,
        order_table,
        properties={"items": relationship(Item, back_populates="order")},
    )

    mapper_registry.map_imperatively(
        Item,
        item_table,
        properties={"order": relationship(Order, back_populates="items")},
    )

如果已将 ``Order`` 与   :class:`_orm.Session`  关联，并且创建了一个 ` `Item`` 对象并将其附加到该订单的 ``Order.items`` 集合中，则该 ``Item`` 将自动级联到同一个   :class:`_orm.Session`  中：

    >>> o1 = Order()
    >>> session.add(o1)
    >>> o1 in session
    True

    >>> i1 = Item()
    >>> o1.items.append(i1)
    >>> o1 is i1.order
    True
    >>> i1 in session
    True

上述代码中，``Order.items`` 和 ``Item.order`` 的双向性意味着向 ``Order.items`` 追加也将分配到 ``Item.order``。同时，``save-update`` 级联允许 ``Item`` 对象被添加到相同   :class:`_orm.Session`  中，其中父级 ` `Order`` 已经关联。

但是，在相反方向上执行上述操作，即分配给 ``Item.order`` 而不是直接追加到 ``Order.item`` 中时，级联操作不会自动进行，即使对象分配的状态与之前的情况相同：

    >>> o1 = Order()
    >>> session.add(o1)
    >>> o1 in session
    True

    >>> i1 = Item()
    >>> i1.order = o1
    >>> i1 in order.items
    True
    >>> i1 in session
    False

在上面的代码中，创建 ``Item`` 对象并设置其所有所需状态后，应将其添加到   :class:`_orm.Session`  中：

    >>> session.add(i1)

在早期版本的 SQLAlchemy 中，save-update 级联将在所有情况下双向发生。然后可以使用一个名为 ``cascade_backrefs`` 的选项选择禁用该行为。最后，在 SQLAlchemy 1.4 中，旧的行为被弃用，并删除了 ``cascade_backrefs`` 选项。其原因是，通常不直观，将对象上的属性与对象的持久性状态联系起来，如上所示通过 ``i1.order = o1`` 的形式进行联系，并且可能会出现后续问题，其中 autoflush 将过早地刷新对象并导致错误，即在仍在构造对象且尚未准备好刷新的情况下。可以使用  :paramref:`_orm.relationship.single_parent`  标志在这种情况下建立 Python 中的断言。

.. seealso::

      :ref:`change_5150`  - 解释了关于 "cascade backrefs" 的行为变化

.. _cascade_delete:

delete
------

``delete`` 级联表示，当标记 "parent" 对象进行删除时，其相关的 "child" 对象也应该被标记为删除。例如，如果我们有一个关系 ``User.addresses``，其中配置了 ``delete`` 级联：

    class User(Base):
        # ...

        addresses = relationship("Address", cascade="all, delete")

如果使用上述映射，则存在一个 ``User`` 对象和两个相关的 ``Address`` 对象：

    >>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
    >>> address1, address2 = user1.addresses

如果我们标记 ``user1`` 进行删除，在 flush 操作执行后，``address1`` 和 ``address2`` 也将被删除：

.. sourcecode:: pycon+sql

    >>> sess.delete(user1)
    >>> sess.commit()
    {execsql}DELETE FROM address WHERE address.id = ?
    ((1,), (2,))
    DELETE FROM user WHERE user.id = ?
    (1,)
    COMMIT

另外，如果我们的 ``User.addresses`` 关系没有配置 ``delete`` 级联，SQLAlchemy 的默认行为是将 "address1" 和 "address2" 从 "user1" 中分离出来，将它们的外键引用设置为 ``NULL``。使用以下映射：

    class User(Base):
        # ...

        addresses = relationship("Address")

删除父级 ``User`` 对象时，不会删除 ``address`` 中的行，而是将它们取消关联：

.. sourcecode:: pycon+sql

    >>> sess.delete(user1)
    >>> sess.commit()
    {execsql}UPDATE address SET user_id=? WHERE address.id = ?
    (None, 1)
    UPDATE address SET user_id=? WHERE address.id = ?
    (None, 2)
    DELETE FROM user WHERE user.id = ?
    (1,)
    COMMIT

对于一对多关系，通常将   :ref:`cascade_delete`  级联与   :ref:` cascade_delete_orphan`  级联结合在一起使用，如果 "child" 对象与 "parent" 分离则会发出一个 DELETE 语句。``delete`` 和 ``delete-orphan`` 级联的组合涵盖了 SQLAlchemy 必须在设置一个外键列为 NULL 与完全删除该行之间进行决策的情况。

该功能默认情况下完全独立于可能对 ``CASCADE`` 行为进行配置的数据库配置的 ``FOREIGN KEY`` 约束。为了与此配置更有效地集成，应使用   :ref:`passive_deletes`  中描述的其他指令。

.. seealso::

      :ref:`passive_deletes` 

      :ref:`cascade_delete_many_to_many` 

      :ref:`cascade_delete_orphan` 

.. _cascade_delete_many_to_many:

在多对多关系中使用 delete 级联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``cascade="all, delete"`` 选项在多对多关系中同样适用，可以使用  :paramref:`_orm.relationship.secondary`  来指示关联表。当删除父对象并因此与相关对象分离时，UnitOfWork 过程通常会从关联表中删除行，但仍然保留相关的对象本身。当与 ` `all, delete`` 结合使用时，将为子行本身进行额外的 ``DELETE`` 语句。

以下示例将   :ref:`relationships_many_to_many`  示例适应为 **一个** 方向上的级联："all, delete" 的设置示例：

    association_table = Table(
        "association",
        Base.metadata,
        Column("left_id", Integer, ForeignKey("left.id")),
        Column("right_id", Integer, ForeignKey("right.id")),
    )


    class Parent(Base):
        __tablename__ = "left"
        id = mapped_column(Integer, primary_key=True)
        children = relationship(
            "Child",
            secondary=association_table,
            back_populates="parents",
            cascade="all, delete",
        )


    class Child(Base):
        __tablename__ = "right"
        id = mapped_column(Integer, primary_key=True)
        parents = relationship(
            "Parent",
            secondary=association_table,
            back_populates="children",
        )

使用上述配置，删除 ``Parent`` 对象的过程如下：

1. 应用程序调用 ``session.delete(my_parent)``，其中 ``my_parent`` 是 ``Parent`` 实例。

2. 在   :class:`_orm.Session`  执行下一个 flush 时，所有 **当前加载** 在 ` `my_parent.children`` 集合中的项目都会被 ORM 删除，这意味着为每个记录都会发出 ``DELETE`` 语句。

3. 如果 ``my_parent.children`` 集合未加载，则不会发出任何 ``DELETE`` 语句。如果此   :func:`_orm.relationship`  上 **未** 设置 ` `passive_deletes`` 标志，则会发出一个 SELECT 语句，以加载未加载的 ``Child`` 对象。

4. 对应于此直接删除，对于受影响的每个 ``Child`` 对象，因为配置了 ``passive_deletes=True``，单元操作不需要尝试为每个 ``Child.parents`` 集合发出 SELECT 语句，因为假定已删除 ``association`` 中相应的行。

5. 由于从 ``Parent.children`` 加载了 instances 对象，因此将为每个因此被 "loaded" 的 ``Child`` 对象发出 DELETE 语句。

.. warning::

    如果在一侧上配置了 "cascade" 级联选项的 **两个** 关系，则级联操作将继续在所有 `Parent`` 和 ``Child`` 对象上级联，加载遇到的每个 ''children" 和 "parents" 集合，删除一切相关的对象。通常情况下，不应在双向上配置 "delete" 级联。

.. seealso::

    :ref:`relationships_many_to_many_deletion` 

    :ref:`passive_deletes_many_to_many` 

.. _passive_deletes:

在 ORM 关系中使用 FOREIGN KEY ON DELETE 级联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQLAlchemy 的级联删除行为与关系数据库的 ``ON DELETE`` 功能重叠。SQLAlchemy 允许使用   :class:`_schema.ForeignKey`  和   :class:` _schema.ForeignKeyConstraint`  构造来配置这些模式级的  :term:`DDL`  行为；在与   :class:` _schema.Table`  元数据一起使用这些对象的用法在   :ref:`on_update_on_delete`  中有描述。

为了将 ``ON DELETE`` 外键级联与   :func:`_orm.relationship`  结合使用，首先首先重要的是注意  :paramref:` _orm.relationship.cascade`  设置必须匹配所需的 "delete" 或 "set null" 行为（使用 ``delete`` 级联或留空），从而使 ORM 能够适当地跟踪可能会受影响的本地存在对象的状态。

然后，在   :func:`_orm.relationship`  中有一个附加选项，指示 ORM 应尽可能地在相关行上运行 DELETE/UPDATE 操作，而不是依赖于预计数据库端 FOREGIN KEY 约束级联回滚任务。这是  :paramref:` _orm.relationship.passive_deletes`  参数，它接受 ``False``（默认值）、``True`` 和 ``"all"`` 选项。

最典型的例子是，当删除父行时应删除子行，并且配置了相应的 ``ON DELETE CASCADE``：

    class Parent(Base):
        __tablename__ = "parent"
        id = mapped_column(Integer, primary_key=True)
        children = relationship(
            "Child",
            back_populates="parent",
            cascade="all, delete",
            passive_deletes=True,
        )


    class Child(Base):
        __tablename__ = "child"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("parent.id", ondelete="CASCADE"))
        parent = relationship("Parent", back_populates="children")

上述配置在删除父行时的行为如下：

1. 使用  :meth:`_orm.Session.delete`  标记 ` `Parent`` 对象进行删除。

2. 该数据库触发器自动在删除父行时删除子行。

3. 与 ``Parent`` 相关联的 ``Parent`` 实例，以及与之关联的所有 ``Child`` 实例（已经与此实例整理并处于加载状态），将从   :class:`._orm.Session`  中去除。

.. note::

    要使用 "ON DELETE CASCADE"，底层数据库引擎必须支持 ``FOREIGN KEY`` 约束并且必须正在实施：

    * 在使用 MySQL 时，必须选择适当的存储引擎。请参见   :ref:`mysql_storage_engines`  了解详情。

    * 在使用 SQLite 时，必须显式启用外键支持。请参见   :ref:`sqlite_foreign_keys`  了解详情。

.. topic:: 注意 passsive_deletes 的一些说明点

    重要的是要注意 ORM 和关系数据库的“级联”概念之间的差异，以及它们如何集成：

    * 数据库级别的 ``ON DELETE`` 级联适用于关系的 **多对一** 方向；也就是说，我们相对于一个 "many" 的关系配置它。在 ORM 级别上，**这个方向是相反的**。SQLAlchemy 通过处理与 "parent" 相关的 "child" 来实现 "child" 对象的删除处理，这意味着在 **部分-全部** 上配置了 "delete" 和 "delete-orphan" 级联。

    * 没有 ``ON DELETE`` 设置的数据级别外键经常被用来防止 "parent" 行被删除，因为这必然会使相关的 "child" 行存在但无人管理。如果在一对多关系中需要这种行为，则在数据库模式级别上将持有外键的列设置为 ``NOT NULL`` 即可。SQLAlchemy 的默认行为将设置 FOREIGN KEY 为 NULL 编程在外键约束例外错误。

    * 数据库级别的 ``ON DELETE`` 级联通常比依赖于 SQLAlchemy 的 "cascade" 级联删除功能要高效得多.数据库可以在许多关系之间链接一系列级联操作；例如，如果删除了行 A，则可以删除表 B 中的所有相关行，并且与每个这些 B 行相关的 C 行，并且进行转换，依次，所有在同一个 DELETE 语句的范围内. SQLAlchemy 此功能相对简单，无法在此上下文中一次性为所有相关行发出 DELETE。

    * SQLAlchemy 没有必要变得如此复杂；我们提供了与数据库自身的 ``ON DELETE`` 功能的平稳集成，通过在外键约束上配置  :paramref:`_orm.relationship.passive_deletes`  选项. 在此行为下，SQLAlchemy 仅为所有 **当前加载** 在   :class:` .Session`  中的行发出 DELETE；对于任何未加载的集合，它将由数据库处理，而不是发出 SELECT 语句。该节   :ref:`passive_deletes`  提供了这种用法的示例。

    * 虽然数据库级别的 ``ON DELETE`` 功能仅适用于关系的 "many" 方向，但 SQLAlchemy 的 "delete" 级联在 *相反* 方向上有**有限**的操作能力，这意味着它可以在 "many" 一侧上进行配置，以在删除引用时删除 "one" 侧上的对象。然而，如果有其他对象从 "many" 引用到该 "one" 侧，这容易导致约束违规，因此仅在关系实际上是 "one to one" 时才非常有用。应使用  :paramref:`_orm.relationship.single_parent`  标志来为此情况建立 Python 断言。

.. _passive_deletes_many_to_many:

关于基于 foreign key 的 ON DELETE 和多对多关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如   :ref:`cascade_delete_many_to_many`  中所述，“delete” 级联在多对多关系中也适用。为了与 ` `many-to-many`` 结合使用 ``ON DELETE CASCADE`` 外键，可以在关联表上配置 ``FOREIGN KEY`` 指令。这些指令可以处理从关联表中自动删除的任务，但无法适应相关的对象本身的自动删除。

在这种情况下，  :paramref:`_orm.relationship.passive_deletes`  指令可以在删除操作期间为我们节省一些额外的 ` `SELECT`` 语句，但仍然有一些集合需要 ORM 继续加载，以便定位受影响的子对象并正确处理它们。

.. note::

  假设性优化包括一次删除所有父关联行的关联表，并使用 ``RETURNING`` 来定位受影响的相关的子行，然而，ORM 单元的工作实现目前不支持这样的优化。

在此配置中，我们在关联表的两个外键约束上配置了 ``ON DELETE CASCADE``。我们在父级 -> 子级关系的一边上配置了 ``cascade="all, delete"``，我们可以在双向关系的 **另一侧** 上配置 ``passive_deletes=True``：

    association_table = Table(
        "association",
        Base.metadata,
        Column("left_id", Integer, ForeignKey("left.id", ondelete="CASCADE")),
        Column("right_id", Integer, ForeignKey("right.id", ondelete="CASCADE")),
    )


    class Parent(Base):
        __tablename__ = "left"
        id = mapped_column(Integer, primary_key=True)
        children = relationship(
            "Child",
            secondary=association_table,
            back_populates="parents",
            cascade="all, delete",
        )


    class Child(Base):
        __tablename__ = "right"
        id = mapped_column(Integer, primary_key=True)
        parents = relationship(
            "Parent",
            secondary=association_table,
            back_populates="children",
            passive_deletes=True,
        )

使用上述配置时，删除 ``Parent`` 对象的过程如下：

1. 属性标记为使用  :meth:`_orm.Session.delete`  删除 ` `Parent`` 对象。

2. 在 flush 执行时，如果未加载 ``Parent.children`` 集合，则 ORM 将首先发出 SELECT 语句，以加载与之对应的 ``Child`` 对象。

3. 然后，ORM 将为与该父行相应的 ``association`` 中的行发出 ``DELETE`` 语句。

4. 对于受此次直接删除影响的每个 "Child" 对象，由于配置了 ``passive_deletes=True``，UnitOfWork 不需要为每个 ``Child.parents`` 集合发出 SELECT 语句，因为假定 ``association`` 中相应行将被删除。

5. 然后，对于从 ``Parent.children`` 加载的每个 ``Child`` 对象，将发出 DELETE 语句。

.. _cascade_delete_orphan:

delete-orphan
-------------

在 ``delete`` 级联增加了 ``delete-orphan`` 级联，这意味着当子对象与 **未关联的 "parent" 分离时**，其应被标记为删除。解除关联
----------------

当子对象是“由”其父对象拥有的相关对象，带有非空外键时，
从父集合中删除项会导致其删除。``delete-orphan``级联
意味着每个子对象一次只能有一个父对象，并且在绝大多数
情况下只配置为一对多关系。对于不太常见的在多对一或多对多
关系上设置的情况，“多”端可以通过配置
 :paramref:`_orm.relationship.single_parent` 参数，强制一次
只允许一个对象，该参数建立了Python端验证，确保对象
一次只与一个父对象相关联，但是这严重限制了“多”关系
的功能，通常不是期望的。
 
.. seealso::

    :ref:`error_bbf0`  -有关删除孤立对象时常见的错误场景的详细信息


合并
----------------

``merge`` 级联指定了从作为  :meth:`.Session.merge` .Session.merge` 操作。此级联默认打开。

刷新-过期
--------------------

``refresh-expire`` 是一个不常见的选项，指示应将 :meth:`.Session.expire` 操作从父级传播到被引用的对象。当使用
  :meth:`.Session.refresh`  时，引用的对象只会变成过期状态，但不会实际上进行刷新。

剔除
--------------------

``expunge``级联指明当父对象被  :meth:`.Session.expunge` .Session` 中删除时，该操作应传播到被引用对象。

在集合和标量关系中删除关联对象的注意事项
--------------------------------------------------------

ORM在整个刷新过程中从不修改集合或标量关系的内容。这意味着，如果你的类
有一个指向对象集合或单个对象引用的   :func:`_orm.relationship` ，
例如，多对一，当执行刷新过程时，该属性的内容不会被修改。 相反，
预计   :class:`.Session`  最终将过期，通过  :meth:` .Session.commit`  的提交操作或通过显式使用  :meth:`.Session.expire` 。在那时，与
连接到该  :class:`.Session`  的引用对象或集合将被清除，并在下次访问时重新加载。

关于此行为常见的混淆涉及使用  :meth:`~.Session.delete`  方法。当
  :meth:`.Session.delete`   在对象上调用时，并刷新   :class:` .Session`  时，该行从
数据库中删除。 如果通过外键引用了目标行，假设它们
使用两个映射对象类型之间的  :func:`_orm.relationship` 进行追踪，也将看到它们的外键属性被更新为null，如果设置了
删除级联，则相关行也将被删除。但是，尽管与已删除对象相关的行本身可能会被稍作修改，删除对象的
关系绑定集合或对象引用**在刷新本身的范围内不发生更改**。这意味着如果对象是
相关集合的成员，则它仍将存在于Python端，直到该集合被过期。同样，如果对象
通过多对一或一对一从另一个对象进行引用，该引用也将保留在该对象上，直到该对象过期为止。


下面，我们将示例说明在将 ``Address``对象标记为删除后，它仍存在于与其父对象``User``相关联的集合中，即使在刷新后仍是如此::

    >>> address = user.addresses[1]
    >>> session.delete(address)
    >>> session.flush()
    >>> address in user.addresses
    True

当提交上述会话后，所有属性都过期了。接下来访问 ``user.addresses``将重新加载集合，显示所需的状态::

    >>> session.commit()
    >>> address in user.addresses
    False

有一个配方可以拦截  :meth:`.Session.delete` ~.Session.delete` ，而是使用级联行为自动调用删除，因为从父集合中移除对象会导致自动将其标记为删除。 ` `delete-orphan``级联就是这样实现的，如下例所示::


    class User(Base):
        __tablename__ = "user"

        # ...

        addresses = relationship("Address", cascade="all, delete-orphan")


    # ...

    del user.addresses[1]
    session.flush()

在上面的示例中，从 ``User.addresses``集合中移除``Address``对象时， ``delete-orphan`` 级联会将该 ``Address``对象标记为删除，就像将其传递给  :meth:`~.Session.delete`  的效果一样。

可以将 ``delete-orphan``级联应用于多对一或一对一关系，以便在将对象取消关联时，也会自动将其标记为删除。在多对一或一对一上使用``delete-orphan``级联需要另一个标志  :paramref:`_orm.relationship.single_parent` ，它调用断言，规定此相关对象不与任何其他父对象同时共享::

    class User(Base):
        # ...

        preference = relationship(
            "Preference", cascade="all, delete-orphan", single_parent=True
        )

如果从其父对象上删除某个假想的 ``Preference`` 对象，则在刷新操作后，它将被删除::

    some_user.preference = None
    session.flush()  # 将删除该Preference对象


.. seealso::

     :ref:`unitofwork_cascades` 对级联有详细描述。