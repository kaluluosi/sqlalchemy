.. _relationship_configure_joins:

配置关系联接
------------

:function:`_orm.relationship` 通常通过检查两个表之间的外键关系来创建一个联接，以确定应该比较哪些列，但需要定制的情况有多种。

.. _relationship_foreign_keys:

处理多个连接路径
~~~~~~~~~~~~~~~~~~

处理两个表之间有多个外键路径的情况是最常见的问题之一。

考虑一个包含两个外键指向“地址”类的“客户”类::

    from sqlalchemy import Integer, ForeignKey, String, Column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Customer(Base):
        __tablename__ = "customer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)

        billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
        shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

        billing_address = relationship("Address")
        shipping_address = relationship("Address")


    class Address(Base):
        __tablename__ = "address"
        id = mapped_column(Integer, primary_key=True)
        street = mapped_column(String)
        city = mapped_column(String)
        state = mapped_column(String)
        zip = mapped_column(String)

上述映射，当我们试图使用它时，将产生一个错误：

.. sourcecode:: text

    sqlalchemy.exc.AmbiguousForeignKeysError: Could not determine join
    condition between parent/child tables on relationship
    Customer.billing_address - there are multiple foreign key
    paths linking the tables.  Specify the 'foreign_keys' argument,
    providing a list of those columns which should be
    counted as containing a foreign key reference to the parent table.

上面的消息很长。  :function:`_orm.relationship`  可以返回很多可能的消息，它们都可以检测出一系列常见的配置问题；大多数消息都会提示需要的附加配置来解决歧义或其他缺少信息的问题。

在这种情况下，消息要求我们针对每个   :func:`_orm.relationship`  ，通过指示每个外键列被视为哪个column,来限定   :func:` _orm.relationship` ，适当的方法如下::

    class Customer(Base):
        __tablename__ = "customer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)

        billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
        shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

        billing_address = relationship("Address", foreign_keys=[billing_address_id])
        shipping_address = relationship("Address", foreign_keys=[shipping_address_id])

在上述代码中，我们指定了“foreign_keys”参数，它是一个或多个   :class:`_schema.Column`  对象的列表，这些对象指示要认为“foreign”的列或表，换句话说，包含指向父表的值的列。从“客户”对象加载“billing_address”关系将使用“billing_address_id”中的值来识别要加载的“Address”中的行。同样，“shipping_address_id”用于“shipping_address”关系。这两个列的链接在持久性过程中也起到了作用；刚刚插入的“地址”对象的新生成的主键将被复制到关联的“客户”对象中的适当外键列中。

在使用 Declarative 时，我们还可以使用字符串名称进行指定，但如果使用列表，则非常重要的是，**列表是字符串的一部分**::

        billing_address = relationship("Address", foreign_keys="[Customer.billing_address_id]")

在此特定示例中，在任何情况下也不需要列表，因为只需要一个   :class:`_schema.Column` ：

        billing_address = relationship("Address", foreign_keys="Customer.billing_address_id")

.. warning:: 作为 Python 可评估的字符串进行传递时，  :paramref:`_orm.relationship.foreign_keys`  参数使用 Python 的 eval() 函数进行解释。**不要将不受信任的输入传递给此字符串**。有关   :func:` _orm.relationship`  参数的声明性评估的详细信息，请参阅   :ref:`declarative_relationship_eval` 。

.. _relationship_primaryjoin:

指定替代连接条件
~~~~~~~~~~~~~~~~~~~~

  :function:`_orm.relationship`   在构建连接时的默认行为是将一个表的主键列的值与另一表上引用外键的列的值相等。我们可以使用  :paramref:` _orm.relationship.primaryjoin`  参数来更改此条件，以便让其等于任何我们想要的条件。如果使用“secondary”表，情况如下。

在下面的示例中，使用“User”类以及一个存储街道地址的“Address”类，创建了一个“boston_addresses”关系，该关系仅会装载“Boston”城市的那些“Address”对象::

    from sqlalchemy import Integer, ForeignKey, String, Column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String)
        boston_addresses = relationship(
            "Address",
            primaryjoin="and_(User.id==Address.user_id, " "Address.city=='Boston')",
        )


    class Address(Base):
        __tablename__ = "address"
        id = mapped_column(Integer, primary_key=True)
        user_id = mapped_column(Integer, ForeignKey("user.id"))

        street = mapped_column(String)
        city = mapped_column(String)
        state = mapped_column(String)
        zip = mapped_column(String)

在上面的字符串 SQL 表达式中，我们使用   :func:`.and_`  连接词语来建立两个不同的谓词的连接条件 - 将“User.id”和“Address.user_id”列相互连接起来，以及限制“Address”中的行仅为“city='Boston'”时。当使用 Declarative 时，基本 SQL 函数如   :func:` .and_`  自动可用于字符串   :func:`_orm.relationship`  参数的计算名称空间。以声明式样式直接引用名为“a”，“b”，“c”，“d”的表。从“A”到“D”的查询如下所示:

.. sourcecode:: python+sql

    sess.scalars(select(A).join(A.d)).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (
        b AS b_1 JOIN d AS d_1 ON b_1.d_id = d_1.id
            JOIN c AS c_1 ON c_1.d_id = d_1.id)
        ON a.b_id = b_1.id AND a.id = c_1.a_id JOIN d ON d.id = b_1.d_id

在上面的示例中，我们利用能够将多个表放入“辅助”容器中的机制，以便我们可以在跨多个表连接的同时仍然为  :func:`_orm.relationship` 保持“简单性”，因为左右双方只有“一个”表；复杂性在中间保持即可。

.. warning:: 类似上面的关系通常被标记为“viewonly=True”，应视为只读。虽然有时候可以使类似于上面的关系可写，但这通常是复杂且容易出错的。

.. _relationship_non_primary_mapper:

.. _relationship_aliased_class:

与别名类的关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 1.3
      :class:`.AliasedClass`  的目标，取代了先前使用非主映射的方法，该方法有限制，例如它们不能像已映射实体一样继承子关系以及它们需要针对备选可选项进行复杂的配置。本节中的示例现已更新为使用   :class:` .AliasedClass` 。

在前一节中，我们演示了一种将  :paramref:`_orm.relationship.secondary`  用于将其他表放入连接条件中的技术。 有一种复杂的连接情况，甚至使用此技术也不足;当我们试图从“A”到“B”连接，并在其中使用任意数量的“C”，“D”等，但是“A”和“B”之间也有直接连接条件。在这种情况下，从“A”到“B”的连接可能很难仅使用复杂的  :paramref:` _orm.relationship.primaryjoin`  条件来表达，因为中间表可能需要特殊处理，并且还不能用  :paramref:`_orm.relationship.secondary`  对象来表示，因为“A->secondary->B”模式不支持“A”和“B”之间的任何引用。当出现这种 **极端高级** 情况时，我们可以创建第二个映射作为关系的目标。这是我们使用   :class:` .AliasedClass`  来创建到包含我们需要的所有附加表的类的映射的地方。为了将此映射作为我们类的“备选”映射生成，我们使用   :func:`.aliased`  函数以产生新的构造，然后针对对象使用   :func:` _orm.relationship` ，就像它是一个普通的映射类一样。

下面说明了   :func:`_orm.relationship` ，它具有从“A”到“B”的简单连接，但是主连接条件增加了两个附加实体“C”和“D”，这些实体在同一时间必须具有与“A和B”中的行相对应的行::


    class A(基类):
        __tablename__ = "a"

        id = mapped_column(Integer, primary_key=True)
        b_id = mapped_column(ForeignKey("b.id"))


    class B(基类):
        __tablename__ = "b"

        id = mapped_column(Integer, primary_key=True)


    class C(基类):
        __tablename__ = "c"

        id = mapped_column(Integer, primary_key=True)
        a_id = mapped_column(ForeignKey("a.id"))

        some_c_value = mapped_column(String)


    class D(基类):
        __tablename__ = "d"

        id = mapped_column(Integer, primary_key=True)
        c_id = mapped_column(ForeignKey("c.id"))
        b_id = mapped_column(ForeignKey("b.id"))

        some_d_value = mapped_column(String)


    # 1. 将 join() 设置为变量，以便我们多次引用映射。
    j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

    # 2. 创建一个关于 B 的别名类
    B_viacd = aliased(B, j, flat=True)

    A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)

有了上述映射，简单连接如下所示:

.. sourcecode:: python+sql

    sess.scalars(select(A).join(A.b)).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) ON a.b_id = b.id

在查询中使用目标别名类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在上一个示例中，“A.b”关系将“B_viacd”实体用作目标，而并不是直接使用“B”类。要添加关于“A.b”的附加条件，通常需要直接引用“B_viacd”而不是使用“B”，特别是在目标实体的“A.b”需要转换为别名或子查询的情况下。下面的示例说明了相同的关系，但是使用子查询而不是连接::

    subq = select(B).join(D, D.b_id == B.id).join(C, C.id == D.c_id).subquery()

    B_viacd_subquery = aliased(B, subq)

    A.b = relationship(B_viacd_subquery, primaryjoin=A.b_id == subq.c.id)

使用上述“A.b”关系的查询将呈现子查询:

.. sourcecode:: python+sql

    sess.scalars(select(A).join(A.b)).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (SELECT b.id AS id, b.some_b_column AS some_b_column
    FROM b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) AS anon_1 ON a.b_id = anon_1.id

如果我们要基于“A.b”关系添加其他条件，则必须使用“B_viacd_subquery”而不是直接使用“B”:

.. sourcecode:: python+sql

    sess.scalars(
        select(A)
        .join(A.b)
        .where(B_viacd_subquery.some_b_column == "some b")
        .order_by(B_viacd_subquery.id)
    ).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (SELECT b.id AS id, b.some_b_column AS some_b_column
    FROM b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) AS anon_1 ON a.b_id = anon_1.id
    WHERE anon_1.some_b_column = ? ORDER BY anon_1.id

.. _relationship_to_window_function:

使用窗口函数的行限制关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

其他有趣的用例，以便让关系连接到任何形式的专门的SELECT时，是使用Python列表达式的标志  :paramref:`_orm.relationship.viewonly` ，它将其建立为只读属性（刷新时将忽略写入集合的数据）。但是，在极端情况下，考虑与   :class:` _query.Query`  结合使用常规Python属性，如下所示:

.. sourcecode:: python

    class User(基类):
        __tablename__ = 'user'
        id = mapped_column(Integer, primary_key=True)

        @property
        def addresses(self):
            return object_session(self).query(Address).with_parent(self).filter(...).all()

在其他情况下，可以构建描述符以利用现有的Python数据。有关特殊Python属性的更一般讨论，请参见   :ref:`mapper_hybrids`  部分。

.. seealso::

      :ref:`mapper_hybrids` 