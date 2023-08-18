使用传统的“backref”关系参数
-------------------------------------------------- 

.. 注意：应将 :paramref:`_orm.relationship.backref` 关键字视为遗留问题，操作时应优先使用显式的 :paramref:`_orm.relationship.back_populates` 和 :func:`_orm.relationship` 构造。使用单独的 :func:`_orm.relationship` 构造的好处包括 ORM 映射类将在构建类时将其属性一并添加，而不是作为延迟步骤，并且配置更简单，因为所有参数都是显式的。SQLAlchemy 2.0 中的新的 :pep:`484` 功能还利用了在源代码中显式存在属性，而不是使用动态属性生成。

.. 参见::

    有关双向关系的一般信息，请参见以下部分：

    :ref:`tutorial_orm_related_objects` - 在 :ref:`unified_tutorial` 中，使用 :paramref:`_orm.relationship.back_populates` 呈现了双向关系的配置和行为概述

    :ref:`back_populates_cascade` - 关于 :class:`_orm.Session` 级联行为的双向 :func:`_orm.relationship` 视图行为的注意事项。

    :`_orm.relationship.back_populates`

在 :func:`_orm.relationship` 构造上的 :paramref:`_orm.relationship.backref` 关键字参数允许自动生成一个新的 :func:`_orm.relationship`。它将自动添加到相关类的 ORM 映射中。然后将其放置在当前 :func:`_orm.relationship` 的 :paramref:`_orm.relationship.back_populates` 配置中，两个 :func:`_orm.relationship` 构造相互引用。

从以下示例开始：

    from sqlalchemy import Column, ForeignKey, Integer, String
    from sqlalchemy.orm import DeclarativeBase, relationship

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)
        addresses = relationship("Address", backref="user")

    class Address(Base):
        __tablename__ = "address"
        id = mapped_column(Integer, primary_key=True)
        email = mapped_column(String)
        user_id = mapped_column(Integer, ForeignKey("user.id"))

以上配置在“User”上建立了一个“Address”对象集合，名为“User.addresses”。它还建立了一个名为“Address.user”的“.user”属性，该属性将引用父“User”对象。使用 :paramref:`_orm.relationship.back_populates`，其等效于以下形式：

    from sqlalchemy import Column, ForeignKey, Integer, String
    from sqlalchemy.orm import DeclarativeBase, relationship

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)
        addresses = relationship("Address", back_populates="user")

    class Address(Base):
        __tablename__ = "address"
        id = mapped_column(Integer, primary_key=True)
        email = mapped_column(String)
        user_id = mapped_column(Integer, ForeignKey("user.id"))
        user = relationship("User", back_populates="addresses")

“User.addresses” 和 “Address.user” 关系的行为是它们现在以双向方式运行，表示在关系的一侧进行更改影响其他方。关于此行为的示例和讨论可以在 :ref:`tutorial_orm_related_objects` 的 :ref:`unified_tutorial` 中找到。


Backref 默认参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于 :paramref:`_orm.relationship.backref` 生成了全新的 :func:`_orm.relationship`，默认情况下，生成过程将尝试在新的 :func:`_orm.relationship` 中包括与原始参数相对应的相应参数。例如，以下是包括 :ref:`custom join condition <relationship_configure_joins>` 的 :func:`_orm.relationship`，它还包括 :paramref:`_orm.relationship.backref` 关键字的示例：

    from sqlalchemy import Column, ForeignKey, Integer, String
    from sqlalchemy.orm import DeclarativeBase, relationship

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)
        addresses = relationship(
            "Address",
            primaryjoin=(
                "and_(User.id==Address.user_id, Address.email.startswith('tony'))"
            ),
            backref="user",
        )

    class Address(Base):
        __tablename__ = "address"
        id = mapped_column(Integer, primary_key=True)
        email = mapped_column(String)
        user_id = mapped_column(Integer, ForeignKey("user.id"))

当生成“backref”时，“backref” 与新的 :func:`_orm.relationship` 中的 :paramref:`_orm.relationship.primaryjoin` 条件一起副本到新的 :func:`_orm.relationship`。

其他可传递的参数包括 :paramref:`_orm.relationship.secondary` 参数，它引用了许多与许多协会表格，以及 “join” 参数 :paramref:`_orm.relationship.primaryjoin` 和 :paramref:`_orm.relationship.secondaryjoin` 的参数;“backref” 足够聪明，知道这两个参数在创建相反作用的同时也应该“翻转”。

指定 Backref 参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

“backref” 的许多其他参数都不是隐式的，包括像 :paramref:`_orm.relationship.lazy`、:paramref:`_orm.relationship.remote_side`、:paramref:`_orm.relationship.cascade` 和 :paramref:`_orm.relationship.cascade_backrefs` 这样的参数。对于这种情况，我们在一个字符串 :func:`.backref` 函数中使用我们 ：

    # <other imports>
    from sqlalchemy.orm import backref

    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)
        addresses = relationship(
            "Address",
            backref=backref("user", lazy="joined"),
        )

在上面，我们仅在 “Address.user” 方面放置了一个 “lazy= joined”的指令，表明在对 “Address” 进行查询时，应该自动进行到 “User” 实体的连接，这将自动填充每个返回的 “Address” 的 “.user” 属性。 :func:`.backref` 函数将我们给出的参数格式化为接受者 :func:`_orm.relationship` 中的一种形式，作为它创建的新关系的附加参数。