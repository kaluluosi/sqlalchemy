配置 Relationship Joins
--------------------------

:func:`_orm.relationship` 通常会通过检查两个表之间的外键关系来创建连接，从而确定应比较哪些列。 有多种情况需要自定义此行为。

处理多个连接路径
~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们需要处理的最常见情况之一是在两个表之间存在多个外键路径。

考虑“客户”类，其中包含两个对“地址”类的外键：

以上映射，当我们尝试使用它时，会产生以下错误：

上述消息非常长。 :func:`_orm.relationship` 可以返回许多潜在消息，这些消息经过精心设计，可检测到各种常见的配置问题；大多数建议提供所需的附加配置来解决比模糊或其他缺失的信息。

在这种情况下，消息需要我们对每个 :func:`_orm.relationship` 进行限定，指示每个外键列应如何处理，适当的格式如下：

上面我们指定了“foreign_keys”参数，它是一个 :class:`_schema.Column` 或 :class:`_schema.Column` 对象列表，指示应将哪些列视为“foreign”，换句话说，包含指向父表的值的列。 从 “Customer” 对象中加载 “Customer.billing_address” 关系将使用“billing_address_id” 中存在的值，以便标识要加载的 “Address” 中的行；类似地，“shipping_address_id” 用于“shipping_address”关系。 两列的联接在持久性期间也会发挥作用；刚刚插入的“Address”对象的新生成的主键将被复制到关联的“Customer”对象的相应外键列中，在刷新其间处理。

在使用 Declarative 指定“foreign_keys”时，我们还可以使用字符串名称进行指定，但重要的是，如果使用列表，则 **列表应包含在字符串中**：

在这个特定的例子中，无论如何，都不需要使用列表，因为我们只需要一个 :class:`_schema.Column` ：

警告： 当作为 Python-evaluable 字符串传递时，:paramref:`_orm.relationship.foreign_keys` 参数使用 Python 的“eval()”函数进行解释。 **不要将不受信任的输入传递给此字符串**。请参阅 :ref:`declarative_relationship_eval` ，了解关于声明性 :func:`_orm.relationship` 参数的详细信息。

指定替代联接条件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:func:`_orm.relationship` 的默认行为在构造联接时是比较一个表上的主键列与另一个表上的外键引用列的值。 我们可以使用 :paramref:`_orm.relationship.primaryjoin` 参数更改此条件，以及 :paramref:`_orm.relationship.secondaryjoin` 参数（在使用“secondary”表的情况下）。

在以下示例中，使用“User”类以及一个存储街道地址的“Address”类，创建了一个仅加载指定“Boston”城市的 “Address” 对象的关系“boston_addresses” ：

在上面的字符串 SQL 表达式中，我们使用 :func:`.and_` 连接构造来为联接条件建立两个不同的谓词 - 同时连接“User.id”和“Address.user_id”列，以及将“Address” 行限制为仅显示 “city =‘Boston’”。 使用 Declarative 时，基本 SQL 函数（例如 :func:`.and_`）会自动在字符串 :func:`_orm.relationship` 参数的评估命名空间中提供。

警告： 当作为 Python-evaluable 字符串传递时， :paramref:`_orm.relationship.primaryjoin` 参数使用 Python 的“eval()”函数进行解释。 **不要将不受信任的输入传递给此字符串**。请参阅 :ref:`declarative_relationship_eval` ，了解关于声明性 :func:`_orm.relationship` 参数的详细信息。

我们在 :paramref:`_orm.relationship.primaryjoin` 中使用的自定义条件通常只在 SQLAlchemy 为了加载或表示此关系而渲染 SQL 时有效。也就是说，当在发出每种属性惰性加载的 SQL 语句时使用，或在查询时构造连接（例如通过 :meth:`Select.join` 或通过急切的“连接”或“子查询”样式加载），它将用于生成 SQL 语句。 在操作内存中的对象时，我们可以将任何“Address”对象放入“boston_addresses”集合，而不管“.city”属性的值是什么。这些对象将保留在集合中，直到属性过期并从应用程序重新加载为止，其中将应用标准选择条件。在刷新时，键入“boston_addresses”中的对象将无条件刷新，将新插入的“Address”对象的新生成的主键值分配给关联的每个“Customer”对象的相应外键列。在此情况下， “city”条件不起作用，因为刷新过程只关心将主键值与引用外键值进行同步。

基于 SQL 函数的自定义操作符
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

另一个使用关系的用例是使用自定义操作符，例如 PostgreSQL 的“is contained within”“<<”运算符，与 :class:`_postgresql.INET` 和 :class:`_postgresql.CIDR` 类型配合使用。对于自定义布尔运算符，我们使用 :meth:`.Operators.bool_op` 函数::

例如上面的比较可以在构建 :func:`_orm.relationship` 时直接使用::

    class IPA(Base):
        __tablename__ = "ip_address"

        id = mapped_column(Integer, primary_key=True)
        v4address = mapped_column(INET)

        network = relationship(
            "Network",
            primaryjoin="IPA.v4address.bool_op('<<')" "(foreign(Network.v4representation))",
            viewonly=True,
        )


    class Network(Base):
        __tablename__ = "network"

        id = mapped_column(Integer, primary_key=True)
        v4representation = mapped_column(CIDR)

该查询被渲染为：

表达式 :meth:`.Operators.op.is_comparison` 的另一个使用情况是当我们不使用操作符，而是使用 SQL 函数时。此用例的典型示例是 PostgreSQL PostGIS 函数，但任何数据库上的任何解析为二进制条件的 SQL 函数都可以适用。为了适应此用例， :meth:`.FunctionElement.as_comparison` 方法可以修改任何 SQL 函数（例如那些从 :data:`.func` 命名空间调用的函数），以表明 ORM 函数生成两个表达式的比较。下面的示例使用 :meth:`~.FunctionElement.as_comparison` 演示了此方法：

例如，我们采取具有代币的重叠路径格式，其中我们比较字符串以便产生一种树状结构。

通过仔细使用 :func:`.foreign` 和 :func:`.remote`，我们可以构建一个关系，它实际上产生了一个粗糙的 materialized path 系统。 基本上，当 :func:`.foreign` 和 :func:`.remote` 在比较表达式的 *同一侧* 时，认为关系是“一对多”，当它们在不同的方向上时，关系被视为“多对一”。 对于我们将要在这里使用的比较，我们将处理集合，因此保持配置为“一对多”：

自我引用的多对多关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

多对多关系可以通过一个或两个参数进行自定义 :paramref:`_orm.relationship.primaryjoin` 和 :paramref:`_orm.relationship.secondaryjoin` - 对于指定使用 :paramref:`_orm.relationship.secondary` 参数的多对多引用，后者很重要。

一种常见情况是需要将一个类与自己建立多对多关系，如下所示：

在上述示例中，我们提供了所有三个 :paramref:`_orm.relationship.secondary`， :paramref:`_orm.relationship.primaryjoin` 和 :paramref:`_orm.relationship.secondaryjoin`，关于命名表"a","b","c","d"的声明式风格。从"A"到"D"的查询如下所示：

.. sourcecode:: python+sql

    sess.scalars(select(A).join(A.d)).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (
        b AS b_1 JOIN d AS d_1 ON b_1.d_id = d_1.id
            JOIN c AS c_1 ON c_1.d_id = d_1.id)
        ON a.b_id = b_1.id AND a.id = c_1.a_id JOIN d ON d.id = b_1.d_id

在上述示例中，我们利用了将多个表放置在“次要”容器中的能力，以便我们在跨多个表连接时仍然保持“简单性” :func:`_orm.relationship`。在左边和右边都只有“一个”表的情况下，复杂性保持在中间。

.. warning:: 上面的关系通常被标记为“viewonly=True”，应该被视为只读毫无疑问。虽然有时可以使上述关系可写，但这通常是非常复杂和错综复杂的。

.. _relationship_non_primary_mapper:

.. _relationship_aliased_class:

关于关系的别名类
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 1.3
    :class:`.AliasedClass`构造现在可以被指定为 :func:`_orm.relationship` 的目标，以取代使用非主要映射器的先前方法，它具有诸如不能继承映射实体的子关系的限制，以及它们要求针对替代可选择的进行复杂的配置。 本节中的示例现已更新，使用了 :class:`.AliasedClass`。

在前面的一节中，我们演示了一种将 :paramref:`_orm.relationship.secondary` 放入连接条件中的技术。在即使使用“C”、“D”等中间表进行连接时，只要“A”和“B”之间也有直接的连接条件，这里还有一个复杂的连接情况即此时不足够using一个复杂的 :paramref:`_orm.relationship.primaryjoin` 条件来表达连接从“A”到“B”，因为中介表可能需要特殊处理，并且使用 :paramref:`_orm.relationship.secondary` 对象无法表达“A->secondary->B”模式之间的引用“A”和“B”之间的直接引用。当出现这种**极其高级**的情况时，我们可以借助创建第二个映射作为关系的目标。这是使用 :class:`.AliasedClass` 的方式，以便为此连接中需要的所有其他表创建到类的映射。为了将此映射作为我们类的“替代”映射生成，我们使用 :func:`.aliased` 函数生成新的构造，然后使用 :func:`_orm.relationship` 来针对该对象进行操作，就像它是一个普通的映射类一样。

下面说明了 :func:`_orm.relationship` 与从“A”到“B”的简单连接的使用，但是主连接条件增加了两个附加实体“C”和“D”，这些实体必须同时与“A”和“B”的行相对应：

    class A(Base):
        __tablename__ = "a"

        id = mapped_column(Integer, primary_key=True)
        b_id = mapped_column(ForeignKey("b.id"))


    class B(Base):
        __tablename__ = "b"

        id = mapped_column(Integer, primary_key=True)


    class C(Base):
        __tablename__ = "c"

        id = mapped_column(Integer, primary_key=True)
        a_id = mapped_column(ForeignKey("a.id"))

        some_c_value = mapped_column(String)


    class D(Base):
        __tablename__ = "d"

        id = mapped_column(Integer, primary_key=True)
        c_id = mapped_column(ForeignKey("c.id"))
        b_id = mapped_column(ForeignKey("b.id"))

        some_d_value = mapped_column(String)


    # 1. 将 join() 设置为变量，这样我们可以在映射中多次引用它。
    j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

    # 2. 创建一个到 B 的别名类
    B_viacd = aliased(B, j, flat=True)

    A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)

在上述映射中，简单连接如下所示：

.. sourcecode:: python+sql

    sess.scalars(select(A).join(A.b)).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) ON a.b_id = b.id

在关于“A.b”的上述关系中，其引用的是“B_viacd”实体，而不是直接的“B”类。要基于“a.b”关系添加其他标准，通常需要直接引用“B_viacd”而不是使用“B”，特别是在目标实体将被转换为别名或子查询的情况下。下面说明了相同的关系，使用子查询而不是连接::

    subq = select(B).join(D, D.b_id == B.id).join(C, C.id == D.c_id).subquery()

    B_viacd_subquery = aliased(B, subq)

    A.b = relationship(B_viacd_subquery, primaryjoin=A.b_id == subq.c.id)

使用上面的“a.b”关系查询将呈现子查询：

.. sourcecode:: python+sql

    sess.scalars(select(A).join(A.b)).all()

    {execsql}SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a JOIN (SELECT b.id AS id, b.some_b_column AS some_b_column
    FROM b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) AS anon_1 ON a.b_id = anon_1.id

如果我们想要基于“A.b”联接添加其他标准，我们必须使用“B_viacd_subquery”而不是直接使用“B”。

.. _relationship_to_window_function:

使用窗口函数的行限制关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

关系到:class:`.AliasedClass`对象的另一个有趣用途是在需要关系加入到任何形式的专门SELECT时。当期望使用窗口函数时，例如限制每个集合返回的行数时。下面的示例说明了为每个集合加载前十个项的非主映射关系：

    class A(Base):
        __tablename__ = "a"

        id = mapped_column(Integer, primary_key=True)


    class B(Base):
        __tablename__ = "b"
        id = mapped_column(Integer, primary_key=True)
        a_id = mapped_column(ForeignKey("a.id"))


    partition = select(
        B, func.row_number().over(order_by=B.id, partition_by=B.a_id).label("index")
    ).alias()

    partitioned_b = aliased(B, partition)

    A.partitioned_bs = relationship(
        partitioned_b, primaryjoin=and_(partitioned_b.a_id == A.id, partition.c.index < 10)
    )

我们可以将上述“partitioned_bs”关系与大多数加载器策略一起使用，例如 :func:`.selectinload`::

    for a1 in session.scalars(select(A).options(selectinload(A.partitioned_bs))):
        print(a1.partitioned_bs)  # <-- 最多有十个对象

以上，在匹配的主键中，我们将按“b.id”排序的前十个“bs”合并在一起。通过分区“a_id”，我们确保每个“row number”对应于父“a_id”。

这样的映射通常还包括从“A”到“B”的“普通”关系，用于持久操作以及当需要“A”下的全部“B”对象时。

.. _query_enabled_properties:

构建具有查询功能的属性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

非常雄心勃勃的自定义连接条件可能无法直接进行持久化，在某些情况下可能无法正确加载。要从方程中删除持久性部分，请在 :func:`~sqlalchemy.orm.relationship` 上使用标志 :paramref:`_orm.relationship.viewonly`，以将其建立为只读属性（在 flush() 上忽略添加到集合的数据）。但是，在极端情况下，考虑使用常规的 Python 属性以及 :class:`_query.Query`，如下所示：

.. sourcecode:: python

    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)

        @property
        def addresses(self):
            return object_session(self).query(Address).with_parent(self).filter(...).all()

在其他情况下，描述符可以被构建为利用现有的 Python 数据。有关特殊 Python 属性的更一般讨论，请参见 :ref:`mapper_hybrids` 部分。

.. seealso::

    :ref:`mapper_hybrids`