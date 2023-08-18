使用传统的“backref”关系参数
--------------------------------------------------

.. note:: 应该将  :paramref:`_orm.relationship.backref`  关键字视为遗留问题，使用具有显式
     :func:`_orm.relationship`  
   的优先选择。使用单个 :func:`_orm.relationship` 构造提供了优势，包括ORM映射类将其属性
   作为类构建时从一开始就包括，而不是作为延迟步骤，并且配置更加直观，因为所有参数都是明确的。
   SQLAlchemy 2.0中的新  :PEP:`484`  功能也利用了显式出现在源代码中的属性，而不是使用动态属性生成。

.. seealso::

    有关双向关系的一般信息，请参阅以下各节：

      :ref:`tutorial_orm_related_objects` -在 :ref:` 统一教程`中，
    提供对使用  :paramref:`_orm.relationship.back_populates`  进行双向关系配置和行为的概述。

      :ref:`back_populates_cascade` -有关 :class:` _orm.Session`级联行为的双向 :func:`_orm.relationship` 行为的备注。

     :paramref:`_orm.relationship.back_populates` 

在  :func:`_orm.relationship`  关键字参数允许
自动生成一个新的  :func:`_orm.relationship` ，该关键字将自动 添加到相关类的ORM映射中。
然后，它将放在与当前正在配置的 :func:`_orm.relationship` 中的
  :paramref:`_orm.relationship.back_populates`  配置中，两个  :func:` _orm.relationship` 
构造相互引用。

从以下示例开始::

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

以上配置在``User``上建立了一个集合，其中包含名为``User.addresses``的``Address``
对象。它还在``Address``上建立了一个``.user``属性，该属性将引用父``User``对象。
使用  :paramref:`_orm.relationship.back_populates`  等效于以下内容::

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

``User.addresses``和``Address.user``关系的行为表示它们现在以**双向**方式
进行操作，表示关系的一侧上的更改会影响另一侧。在 :ref:`统一教程` 中
的 :ref:`tutorial_orm_related_objects` 中有一个示例和讨论此行为。

backref默认参数
~~~~~~~~~~~~~~~~~~~~~~~~~

由于  :paramref:`_orm.relationship.backref`  生成一个全新的  :func:` _orm.relationship` ，
默认情况下生成过程将尝试在新的 :func:`_orm.relationship` 中包括与原始参数相对应的
相应参数。例如，下面是一个包含  :paramref:`_orm.relationship.backref`  关键字及
包含  :ref:`自定义连接条件 <relationship_configure_joins>` 
作为示例::

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

当生成"backref"时，  :paramref:`_orm.relationship.primaryjoin`  条件也被复制到了新的
 :func:`_orm.relationship` 中::

    >>> print(User.addresses.property.primaryjoin)
    "user".id = address.user_id AND address.email LIKE :email_1 || '%%'
    >>>
    >>> print(Address.user.property.primaryjoin)
    "user".id = address.user_id AND address.email LIKE :email_1 || '%%'
    >>>

可转移的其他参数包括  :paramref:`_orm.relationship.secondary`  参数，该参数引用
多对多关联表，以及"join"参数  :paramref:`_orm.relationship.primaryjoin`  
和  :paramref:`_orm.relationship.secondaryjoin`  ；"backref"足够聪明
可以知道在生成相反方面时这两个参数也应该被“颠倒”。

指定backref参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

"backref"的许多其他参数不是隐式的，包括参数如  :paramref:`_orm.relationship.lazy`  ，
  :paramref:`_orm.relationship.remote_side`  ,  :paramref:` _orm.relationship.cascade` 
和  :paramref:`_orm.relationship.cascade_backrefs`  。对于这种情况，
我们在字符串的位置上使用 :func:`.backref` 函数；这将存储一组特定的参数，这些参数在生成
新的 :func:`_orm.relationship` 时将被传递：

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

在上面的示例中，我们只在``Address.user``侧指定了``lazy="joined"``指令，
表示当对``Address``进行查询时，应自动连接到``User``实体，
这将填充每个返回的``Address``的``.user``属性。 :func:`.backref` 函数格式化我们给
它的参数成为由接收方 :func:`_orm.relationship` 解释为要应用于其创建新关系的附加参数。