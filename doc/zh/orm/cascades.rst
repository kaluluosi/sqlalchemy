级联
========

映射器支持在 :func:`~sqlalchemy.orm.relationship` 构造中可配置 :term:`级联` 行为的概念。这涉及到对相对于关于特定 :class:`.Session` 的 "父" 对象执行操作应如何传播到由该关系引用的项目（例如 "子" 对象），并受到 :paramref:`_orm.relationship.cascade` 选项的影响。

级联的默认行为仅限于所谓的 :ref:`Cascade Save-Update <cascade_save_update>` 和 :ref:`Cascade Merge <cascade_merge>` 设置的级联。级联的典型 "替代" 设置是添加 :ref:`Cascade Delete <cascade_delete>` 和 :ref:`Cascade Delete-Orphan <cascade_delete_orphan>` 选项；这些设置适用于只存在于它们附加到其父对象的情况下，并且否则会被删除的相关对象。

使用 :paramref:`_orm.relationship.cascade` 选项配置级联行为的示例在下面的代码中::

    class Order(Base):
        __tablename__ = "order"

        items = relationship("Item", cascade="all, delete-orphan")
        customer = relationship("User", cascade="save-update")

在回推（backref）上设置级联，可以使用相同的标志与 :func:`~.sqlalchemy.orm.backref` 函数，它最终将其参数反馈到 :func:`~sqlalchemy.orm.relationship` 中::

    class Item(Base):
        __tablename__ = "item"

        order = relationship(
            "Order", backref=backref("items", cascade="all, delete-orphan")
        )

.. 侧边栏:: Cascade 的由来

    SQLAlchemy 中的级联行为的概念以及配置它们的选项，主要源自 Hibernate ORM 中的类似特征；Hibernate 在 `栗子：父 / 子 <https://docs.jboss.org/hibernate/orm/3.3/reference/en-US/html/example-parentchild.html>`_ 等地方都称之为 "级联"。如果级联令人困惑，我们将引用它们的结论，说明 "我们刚刚涵盖的章节可能有点令人困惑。但是，在实践中，所有内容都很好。"

:paramref:`_orm.relationship.cascade` 的默认值为 ``save-update, merge``。对于此参数的典型替代设置为 ``all`` 或更常见的 ``all, delete-orphan``。``all`` 符号是 ``save-update, merge, refresh-expire, expunge, delete`` 的同义词，并且在与 ``delete-orphan`` 结合使用时表示，子对象应在所有情况下跟随其父对象，并且一旦与该父对象不再关联，就应将其删除。

.. warning:: ``all`` 级联选项意味着 :ref:`Cascade Refresh-Expire <cascade_refresh_expire>`  级联设置，使用 :ref:`asyncio_toplevel` 扩展时可能不是理想的，因为它会比在显式 IO 上下文中通常适当的方式更积极地到期相关对象。有关详细背景，请参见 :ref:`asyncio_orm_avoid_lazyloads` 中的备注。

可以指定的 :paramref:`_orm.relationship.cascade` 参数的可用值在以下各小节中描述。

.. _cascade_save_update:

save-update
-----------

``save-update`` 级联指示当使用 :meth:`.Session.add` 将对象放置到 :class:`.Session` 中时，
所有通过此 :func:`_orm.relationship` 关系关联的对象也应添加到相同的 :class:`.Session` 中。使用 ``save-update`` 级联将为你自动将对象和它的关系添加到会话中。例如，我们有一个带有两个相关对象，``address1``，``address2`` 的对象 ``user1``::

    >>> user1 = User()
    >>> address1, address2 = Address(), Address()
    >>> user1.addresses = [address1, address2]

如果我们将 ``user1`` 添加到 :class:`.Session` 中，它也会隐式地添加 ``address1`` 和 ``address2``::

    >>> sess = Session()
    >>> sess.add(user1)
    >>> address1 in sess
    True

``save-update`` 级联也影响已经存在于 :class:`.Session` 中的对象的属性操作。如果我们将第三个对象``address3`` 添加到 ``user1.addresses`` 集合中，它将成为该 :class:`.Session` 的状态的一部分::

    >>> address3 = Address()
    >>> user1.addresses.append(address3)
    >>> address3 in sess
    True

使用 ``save-update`` 级联可能会出现令人惊讶的行为，当从集合中删除项或将对象从标量属性中取消关联时。在某些情况下，孤立的对象仍可能被拉入前父级的 :class:`.Session`；
原因是刷新过程需要正确处理相关对象。通常只有在将对象从一个 :class:`.Session` 中删除并添加到另一个 :class:`.Session` 中时才会出现这种情况::

    >>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
    >>> address1 = user1.addresses[0]
    >>> sess1.close()  # user1，address1不再与sess1关联
    >>> user1.addresses.remove(address1)  # address1不再与user1相关联
    >>> sess2 = Session()
    >>> sess2.add(user1)  # ...但它仍然会被添加到新会话中
    >>> address1 in sess2  # 因为它仍处于“挂起”状态等待刷新
    True

``save-update`` 级联默认开启，通常为方便起见。它通过允许单个调用 :meth:`.Session.add` 一次为该 :class:`.Session` 注册整个对象结构来简化代码。
虽然可以禁用它，但通常不需要这样做。

.. _back_populates_cascade:

.. _backref_cascade:

双向关系下的 save-update 级联行为
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

双向关系上的 ``save-update`` 级联行为在单向情况下进行，即使用 :paramref:`_orm.relationship.back_populates` 或 :paramref:`_orm.relationship.backref` 参数创建互相引用的两个 :func:`_orm.relationship` 对象时。

当将与 :class:`._orm.Session` 关联的“父”对象上的属性或集合分配给未与 :class:`._orm.Session` 关联的子对象时，将自动将对象添加到该同一 :class:`._orm.Session` 中。但是，反向操作不会产生此效果；将与 :class:`._orm.Session` 关联的子对象分配给与 :class:`._orm.Session` 不关联的对象时，将不会自动将该父对象添加到 :class:`._orm.Session` 中。这种行为的整体主题为“级联回推（cascade backrefs）”，它表示自 SQLAlchemy 2.0 以来标准化的行为更改。

举例来说，给定一个 ``Order`` 对象的映射，该对象通过关系 ``Order.items`` 和 ``Item.order`` 与一系列 ``Item`` 对象相关双向关系::

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

如果在与 :class:`._orm.Session` 关联的 ``Order`` 上添加一个 ``Item`` 对象并将其添加到该 ``Order.items`` 集合中，则此 ``Item`` 将自动级联到该 :class:`._orm.Session` 中::

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

上面，``Order.items`` 和 ``Item.order`` 的双向特性意味着将 ``Item`` 添加到 ``Order.items`` 中还分配给了``Item.order``。同时，``save-update`` 级联允许为对象添加到与父对象相同的 :class:`._orm.Session` 中已经相关的内容。

但是，如果在相反的方向上执行上述操作，即将 ``Item.order`` 分配给该子对象而不是直接附加到 ``Order.item`` 上，则不会自动执行级联操作，即便在建立的对象分配上``Order.items`` 和 ``Item.order`` 的状态也是相同的：

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

在上面的情况下，创建 ``Item`` 对象并进行所需的所有状态设置后，它应显式添加到 :class:`._orm.Session` 中::

    >>> session.add(i1)

在旧版本的 SQLAlchemy 中，级联的 save-update 行为在所有情况下双向执行。然后通过称为 ``cascade_backrefs`` 的选项可切换到它的可选设置，针对 SQLAlchemy 2.0 移除了旧行为，不再提供选择在 ORM 中间切换不同的工作方式，这包括了在 ORM 中的学习曲线以及文档和用户支持的负担。

.. seealso::

    :ref:`change_5150` - 有关“级联回推”行为变更的背景信息

.. _cascade_delete:

delete
------

``delete`` 级联指示将标记为删除的“父”对象的相关的“子”对象也应标记为删除。例如，如果我们的关系为 ``User.addresses`` 并配置了 ``delete`` 级联，则：

    class User(Base):
        # ...

        addresses = relationship("Address", cascade="all, delete")

如果使用上面的映射，则有一个 ``User`` 对象和两个相关的 ``Address`` 对象：

    >>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
    >>> address1, address2 = user1.addresses

如果将 ``user1``  标记为删除，则在进行刷新操作后，``address1`` 和 ``address2`` 也将被删除：

.. sourcecode:: pycon+sql

    >>> sess.delete(user1)
    >>> sess.commit()
    {execsql}DELETE FROM address WHERE address.id = ?
    ((1,), (2,))
    DELETE FROM user WHERE user.id = ?
    (1,)
    COMMIT

或者，如果我们的 ``User.addresses`` 关系没有 ``delete`` 级联，则 SQLAlchemy 的默认行为是将其与 ``user1`` 解除关联，使其外键引用设置为 ``NULL``。使用下面的映射::

    class User(Base):
        # ...

        addresses = relationship("Address")

在删除父 ``User`` 对象时，不会删除 ``address`` 中的行，但会将其解除关联：

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

:ref:`cascade_delete` 级联在一对多关系上通常与 :ref:`cascade_delete_orphan` 级联结合使用，如果“子”对象与父对象分离，则会发出有关相关行的 DELETE。将 ``delete`` 和 ``delete-orphan`` 级联组合起来涵盖了 SQLAlchemy 不得不在设置外键列为 NULL 与完全删除行之间进行决策的两种情况。

该功能默认与数据库配置的 ``FOREIGN KEY`` 约束完全独立，后者本身会配置 ``CASCADE`` 行为。为了更有效地与此配置集成，需要使用 :ref:`passive_deletes` 中描述的附加指令。

.. seealso::

    :ref:`passive_deletes`

    :ref:`cascade_delete_many_to_many`

    :ref:`cascade_delete_orphan`

.. _cascade_delete_many_to_many:

使用 delete 级联与多对多关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``cascade="all, delete"`` 选项同样适用于使用 :paramref:`_orm.relationship.secondary` 表示关联的关系；表示联结表。当父对象被删除，且因此从其相关对象中解除关联时，单元操作流程通常会从联接表中删除行，但保留相关对象。当与 ``"all, delete"`` 组合使用时，还将为子行本身执行额外的 ``DELETE`` 语句。

以下示例调整了 :ref:`relationships_many_to_many` 的例子以说明在关系的 **一个** 方向上设置了 ``cascade="all, delete"`` ：

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

上述示例中，当使用 :meth:`_orm.Session.delete` 标记父行时，删除流程将按照惯例从 ``association`` 表中删除行，但根据级联规则，还将删除所有相关的 ``Child`` 行。

.. warning::

    如果在两个关系上同时设置了 ``cascade="all, delete"`` 设置，则级联操作将继续级联所有 ``Parent`` 和 ``Child`` 对象，加载遇到的每个 ``children`` 和 ``parents`` 集合，然后删除连接的然后删除连缀；这通常不适用于 "delete" 级联在双向设置上。

.. seealso::

  :ref:`relationships_many_to_many_deletion`

  :ref:`passive_deletes_many_to_many`

.. _passive_deletes:

使用 ORM 关系的外键 ON DELETE 级联
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQLAlchemy 的级联行为与关系数据库的 ``ON DELETE`` 功能重叠。
SQLAlchemy 使用 :class:`_schema.ForeignKey` 和 :class:`_schema.ForeignKeyConstraint` 构造对象允许配置这些架构级别的 DDL 行为。使用这些对象与 :class:`_schema.Table` metadata 在 :ref:`on_update_on_delete` 中进行描述；

为了使用 ``ON DELETE`` 外键级联与 :func:`_orm.relationship` 结合使用，首先需要注意的是 :paramref:`_orm.relationship.cascade` 设置必须匹配所需的 "delete" 的或 "set null" 行为（使用 ``delete`` 级联或不使用它），以便无论 ORM 还是数据库级别的约束将处理实际修改数据库中的数据时，ORM 都能够适当地跟踪在本地存在的对象的状态。

然后，在 :func:`_orm.relationship` 上还有一个附加选项，指示 ORM 应在多大程度上尝试自行运行相关行的 DELETE/UPDATE 操作，而非依赖于期望数据库 SIDE FOREIGN KEY 约束级联来处理任务；这是 :paramref:`_orm.relationship.passive_deletes` 参数，它接受选项 ``False``（默认值）、``True`` 和 ``"all"``。

最典型的示例是将孩子行与父行分离时删除子行，并且在相关的 ``FOREIGN KEY`` 约束上配置了 ``ON DELETE CASCADE``：

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

上述配置中，当父行被删除时，删除行为将遵循以下步骤：

1. 应用使用 :meth:`_orm.Session.delete` 标记删除的父行。

2. 在下次 flush 更改到数据库时，所有 **当前加载的** ``my_parent.children`` 集合中的项都将被 ORM 删除，这意味着每个记录都将发出一个 ``DELETE`` 语句。

3. 如果 ``my_parent.children`` 集合 **未加载**，则不会发出 ``DELETE`` 语句。如果在此 :func:`_orm.relationship` 上未设置 :paramref:`_orm.relationship.passive_deletes` 标志，则会发出查询未加载的 ``Child`` 对象的 SELECT 语句。

4. 然后，为 ``my_parent.row`` 中对应的行发出 ``DELETE`` 语句。

5. 数据库级别的 ``ON DELETE CASCADE`` 使得与受影响的行在 ``parent`` 中有引用的所有行都会被删除。

6. ``my_parent`` 对象，以及与其相关的所有 ``Child``，在操作完成后均与 :class:`._orm.Session` 分离。

.. note::

    要使用 "ON DELETE CASCADE"，底层数据库引擎必须支持 ``FOREIGN KEY`` 约束，并且它们必须被实施：

    * 在使用 MySQL 时，必须选择适当的存储引擎。有关详细信息，请参见 :ref:`mysql_storage_engines`。

    * 使用 SQLite 时，必须显式启用外键支持。有关详细信息，请参见 :ref:`sqlite_foreign_keys`。

.. 主题:: 关于被动删除的注释

    需要注意 ORM 和关系数据库的“级联”概念之间的差异，以及它们如何集成：

    * 数据库级别的 ``ON DELETE`` 级联通常相对于“多”这一侧的关系进行配置；也就是说，我们相对于是关系的 "many" 这一侧进行配置。在 ORM 级别上，**这个方向相反**。SQLAlchemy 从“parent”一侧处理删除“child”对象的操作，这意味着 ``delete`` 和 ``delete-orphan`` 级联在 "one-to-many" 这一侧进行配置。

    * 在数据库级别上，没有 ``ON DELETE`` 设置的外键常常用于**防止**父行被删除，因为这必然会导致出现未处理的关联行。如果在一对多关系中需要这种行为，则可以在 SQLAlchemy 默认行为在数据库级别设置一个外键保持，方法是在数据库模式级别将包含外键的列设置为 ``NOT NULL``。在极少数特殊情况下，可以通过在 :paramref:`_orm.relationship.passive_deletes` 标志上设置字符串 ``"all"`` 来禁用 SQLAlchemy 设置外键列为 NULL 的行为，而是对父行进行 DELETE 操作，而不对 Child 行造成任何影响，即使这些行在内存中存在。当需要在父行删除时激活数据库级别外键触发器（包括特殊的“ON DELETE”设置或其他设置）的情况下，可能会出现这种情况。

    * 数据库级别的 ``ON DELETE`` 级联通常比依赖于 SQLAlchemy 的级联删除要更有效率。数据库可以将多个关系的串级操作链接到一起；例如，如果删除了行 A，则可以删除表 B 中的所有相关行以及每个这些 B 行相关的 C 行等，所有这些操作都可以在一个 DELETE 语句的范围内发生。另一方面，为了完全支持级联删除操作，SQLAlchemy 必须逐个加载每个相关集合，以便针对然后可能具有更多相关集合的所有行进行处理。也就是说，SQLAlchemy 并不像同时实现所有相关行的 DELETE 语句那样先进。

    * SQLAlchemy **不需要** 这么复杂，因为我们提供平滑地与数据库自己的 ``ON DELETE`` 功能集成的方式，即将 :paramref:`_orm.relationship.passive_deletes` 选项与正确配置的外键约束结合使用。在此行为下，SQLAlchemy 只为已经本地存在于 :class:`.Session` 中的行发出 DELETE；对于未加载的任何集合，它不会发出 SELECT，而是将其留给数据库来处理。在 :ref:`passive_deletes` 如果需要使用，请提供一个示例。

    * 虽然数据库级别的 ``ON DELETE`` 功能仅在关系中的“多”一侧起作用，但 SQLAlchemy 的“delete” cascade 却有**有限的**能力，反向方向上使其在“多”这一侧上配置，即当删除“many”关系的引用时，一侧上的对象将被删除。但是，如果有其他对象从“many”这一侧引用到此“one”一侧的情况下，这样的配置容易产生约束冲突，因此它通常仅在关系实际上是 “one-to-one” 时有用。应使用 :paramref:`_orm.relationship.single_parent` 标志为此情况建立 Python 内部断言。

.. _passive_deletes_many_to_many:

使用外键 ON DELETE 与多对多关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如 :ref:`Cascade Delete Many-to-Many <cascade_delete_many_to_many>` 中所述，“delete” 级联在多对多关系上同样适用。要使使用 ``foreign key`` 来配置联接表上的 ``ON DELETE CASCADE`` 与多对多关系结合使用，必须在联接表上配置 ``FOREIGN KEY`` 指令。这些指令可以处理自动从联接表上删除，但不能自动删除相关对象本身。

在这种情况下，可以使用 :paramref:`_orm.relationship.passive_deletes` 指令在删除操作期间在执行某些额外的 ``SELECT`` 语句时节省一些时间，但仍有一些集合需要 ORM 继续加载以查找受影响的子对象并正确处理它们。

.. note::

  这种情况的假设优化是以单个 ``DELETE`` 语句一次删除联接表的所有父相关行，然后使用 ``RETURNING`` 查找受影响的相关子行，但这目前不是 ORM 工作单元实现的一部分。

在此配置中，必须在联接表的两个外键约束上配置 ``ON DELETE CASCADE``。在父（parent）->子（child）关系的一侧上配置了 ``cascade="all, delete"``，可以在双向关系的 **其他** 一侧上配置 ``passive_deletes=True``，如下所示：

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

使用上述配置，删除 ``Parent`` 对象的过程如下:

1. 使用 :meth:`_orm.Session.delete` 标记删除 ``Parent`` 对象。

2. 在刷新时，如果没有加载 ``Parent.children`` 集合，则 ORM 将首先发出 SELECT 语句以加载与 ``Parent.children`` 相应的 ``Child`` 对象。

3. 然后，对于与该父行对应的行，将为 ``association`` 中的行发出 ``DELETE`` 语句。

4. 对于受此直接删除影响的每个 ``Child`` 对象。因为配置了 ``passive_deletes=True``，因此在“Child.parents”集合上的每个“Child”对象不需要在单独的查询中检索，因为假定与“association”中相应的行将被删除。

5. 然后，对于从 ``Parent.children`` 中加载的每个 ``Child`` 对象发出 ``DELETE`` 语句。与父对象解绑定的子对象在父对象标记为删除时，而不是在其父对象删除时，被自动删除。这在处理“由其父对象拥有”的相关对象时是一项常见功能，其外键为NOT NULL，以便从父则删除集合中的项目会导致其删除。

``delete-orphan``级联意味着每个子对象一次只能有一个父对象，并且在**绝大多数情况下，只配置在一对多关系上。**对于在多对一或多对多关系上设置这种级联的少见情况，“多”端可以通过配置:paramref:`_orm.relationship.single_parent`参数来强制一次只允许一个对象，这会建立验证-python端的Python验证，以确保该对象一次只与一个父对象相关联，但这严重限制了“多”关系的功能，而且通常不是所需的。

.. seealso::

  :ref:`error_bbf0` - 关于涉及delete-orphan级联的常见错误情况的背景。

.. _cascade_merge:

合并
-----

``merge``级联表示应从作为:meth:`.Session.merge`调用的主体的父对象向下传播:meth:`.Session.merge`操作。默认情况下，此级联也开启。

.. _cascade_refresh_expire:

刷新-过期
--------------

``refresh-expire``是不常见的选项，表示从父对象向下传播:meth:`.Session.expire`操作。使用:meth:`.Session.refresh`时，只有被引用的对象被过期，但实际上并没有刷新。

.. _cascade_expunge:

删减
-------

``expunge``级联表示当从:class:`.Session`使用:meth:`.Session.expunge`删除父对象时，应向下传播操作以删除所引用的对象。

.. _session_deleting_from_collections:

删除-删除从集合和标量关系引用的对象
----------------------------------------------------------------------------------------

ORM通常在刷新过程中不会修改集合或标量关系的内容。这意味着，如果你的类具有一个引用了对象集的:func:`_orm.relationship`，或者引用了单个对象的引用，例如多对一，则此属性的内容将在刷新过程发生时不被修改。相反，预计:class:`.Session`最终将过期，通过:meth:`.Session.commit`的commit-expire行为或通过:meth:`.Session.expire`的显式使用进行。此时，与该:class:`.Session`相关联的任何引用对象或集合都将被清除，并在下一次访问时重新加载。

关于这种行为经常引起困惑的一个常见问题涉及使用:meth:`~.Session.delete`方法。当:meth:`.Session.delete`在对象上调用并且:class:`.Session`被刷新时，从数据库中删除该行。引用到目标行的行，假设它们使用:func:`_orm.relationship`之间的跟踪，这两个映射对象类型之间的，将还会发现它们的外键属性被更新为null，或者如果设置了级联删除，则相关行也将被删除。但是，即使与删除的对象相关联的行本身可能也会发生修改，代表该行的集合或对象引用也在刷新本身范围内未被修改。这意味着，如果对象是相关集合的成员，则在该集合过期之前，它仍将存在于python端。同样，如果通过另一个对象通过多对一或一对一引用，则该引用也将保留在该对象上，直到该对象过期为止。

下面，我们演示了在标记了要删除的“地址”对象后，它仍存在与“用户”的父级关联集合中，在刷新之后仍然存在的情况：

    >>> address = user.addresses[1]
    >>> session.delete(address)
    >>> session.flush()
    >>> address in user.addresses
    True

上述会话提交后，所有属性都过期。再次访问“user.addresses”将重新加载集合，显示所需状态：

    >>> session.commit()
    >>> address in user.addresses
    False

有一个配方可以拦截:meth:`.Session.delete`并自动调用过期；请参见`ExpireRelationshipOnFKChange <https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange>`_。但是，删除集合内的项目的常见做法是放弃直接使用:meth:`~.Session.delete`的用法，而是使用级联行为，以便自动调用删除作为将对象从父集合中删除的结果。``delete-orphan``级联可以实现这一点，如下例所示：

    class User(Base):
        __tablename__ = "user"

        # ...

        addresses = relationship("Address", cascade="all, delete-orphan")


    # ...

    del user.addresses[1]
    session.flush()

在上述示例中，从“User.addresses”集合中删除“Address”对象后，“delete-orphan”级联的效果相当于将其传递给:meth:`~.Session.delete`。

``delete-orphan``级联也可以应用于多对一或一对一关系，以便在解除与其父对象的关联时自动标记为删除。在多对一或一对一上使用“delete-orphan”级联需要额外的标志:paramref:`_orm.relationship.single_parent`，它会调用断言，表明此相关对象不应与任何其他父对象同时共享：

    class User(Base):
        # ...

        preference = relationship(
            "Preference", cascade="all, delete-orphan", single_parent=True
        )

上面的代码，如果从“User”中删除一个假设的“Preference”对象，则将在刷新时将其删除:

    some_user.preference = None
    session.flush()  # will delete the Preference object

.. seealso::

    :ref:`unitofwork_cascades`有关级联详细信息。