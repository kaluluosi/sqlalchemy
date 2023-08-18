========================
非传统映射
========================

.. _orm_mapping_joins:

.. _maptojoin:

将一个类映射到多个表
=======================================

Mapper可以针对任意关系单元（称为*selectable*）进行构建，而不仅仅是普通的表。例如，:func:`_expression.join`函数创建了一个包含多个表的可选单元，其中包括自己的复合主键，可以与:class:`_schema.Table`一样映射::

    from sqlalchemy import Table, Column, Integer, String, MetaData, join, ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import column_property

    metadata_obj = MetaData()

    # 定义两个 Table 对象
    user_table = Table(
        "user",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("name", String),
    )

    address_table = Table(
        "address",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String),
    )

    # 定义它们之间的连接。它们之间是通过 user.id 和 address.user_id
    # 列进行连接的。
    user_address_join = join(user_table, address_table)


    class Base(DeclarativeBase):
        metadata = metadata_obj


    # 映射到这个连接
    class AddressUser(Base):
        __table__ = user_address_join

        # 使用 column_property 指定一个特殊的列映射关系。这个 id 属性来自两张表的 id 列和 address.user_id 列
        id = column_property(user_table.c.id, address_table.c.user_id)

        # 将 address.id 列映射到一个名为 address_id 的属性。这是为了明确其映射
        address_id = address_table.c.id

在上面的例子中，连接表达了 "user" 和 "address" 表的列。用户表和地址表的"id"和"user_id"列通过外键进行匹配，因此在映射时它们定义为一个属性，"AddressUser.id"。根据配置的这部分内容，当刷新发生时，映射将从"user.id"复制新的主键值到"address.user_id"列中。

此外， "address.id"列显式映射到名为 "address_id"的属性。这是为了**区分**"address.id"列的映射和同名的"AddressUser.id"属性，后者已指定为引用与"address.user_id"外键相结合的"user"表。

上面映射的自然主键是"(user.id，address.id)"的组合，因为这些是"user"和"address"表一起组合的主键列。一个"AddressUser"对象的标识将基于这两个值，并且从一个"AddressUser"对象表示为"(AddressUser.id，AddressUser.address_id)"。

在引用"AddressUser.id"列时，大多数SQL表达式将只使用映射的列列表中的第一列，因为这两列是同义词。但是，对于一些特殊的用例，例如GROUP BY表达式，在同时引用两个列时必须使用proper context，即适应别名和相似文本，可以使用 :attr:`.ColumnProperty.Comparator.expressions` 访问器::

    stmt = select(AddressUser).group_by(*AddressUser.id.expressions)

.. versionadded:: 1.3.17 添加了 :attr:`.ColumnProperty.Comparator.expressions` 访问器。


.. note::

    如上所示，针对多个表的映射支持持久性，即插入、更新和删除有关目标表中的行。但是，它不支持同时将更新一个表并在同一行上执行INSERT或DELETE的操作。也就是说，如果将一行“PtoQ”映射到“p”和“q”表，其中它基于“p”和“q”的LEFT OUTER JOIN，如果进行一个要在现有的记录中更改“q”表中的数据的UPDATE，行中必须有“q”，否则，对于大多数支持报告UPDATE所影响的行数的DBAPI驱动程序，ORM将无法检测到已更新的行并引发错误；否则，数据会被默默忽略。

    一个允许在其他任一工作的"插入"相关行的食谱可以利用.MapperEvents.before_update事件，例如::

        from sqlalchemy import event


        @event.listens_for(PtoQ, "before_update")
        def receive_before_update(mapper, connection, target):
            if target.some_required_attr_on_q is None:
                connection.execute(q_table.insert(), {"id": target.id})

    在上面的例子中，通过使用 :meth:`_schema.Table.insert` 创建一个INSERT构造，然后使用给定的:class:`_engine.Connection`执行它，改变了“q_table”表中的行，在 emit 的过程中使用了相同的 SQL，从而将行 INSERT 到 "q_table" 表中。用户提供的逻辑必须检测从“p”到“q”左外连接是否没有一个条目关于“q”侧。

.. _orm_mapping_arbitrary_subqueries:

将类映射到任意子查询
============================================

与映射连接类似，也可以使用简单的 :func:`_expression.select` 对象作为映射器。下面的例子片段说明了如何将名为 "Customer" 的类映射到一个包括对子查询的连接的 :func:`_expression.select` 中::

    from sqlalchemy import select, func

    subq = (
        select(
            func.count(orders.c.id).label("order_count"),
            func.max(orders.c.price).label("highest_order"),
            orders.c.customer_id,
        )
        .group_by(orders.c.customer_id)
        .subquery()
    )

    customer_select = (
        select(customers, subq)
        .join_from(customers, subq, customers.c.id == subq.c.customer_id)
        .subquery()
    )


    class Customer(Base):
        __table__ = customer_select

在上面的例子中，"customer_select"所表示的完整行将是"customers" 表的所有列，以及 "subq" 子查询公开的那些列，即 "order_count"， "highest_order"和 "customer_id"。将"Customer"类映射到此可选择会创建一个包含这些属性的类。

当ORM持久化新的“Customer”实例时，实际上只有“customers”表会收到INSERT。这是因为“orders”表的主键没有在映射中表示；ORM仅会为它已映射主键的表发出INSERT。

.. note::

    针对任意SELECT语句进行映射的做法，尤其是像上面那样的复杂语句，几乎不需要；它必然会产生复杂的查询，往往比直接查询构造的效率低得多。这种做法在某种程度上是基于SQLAlchemy非常早期的历史，其中 :class:`_orm.Mapper` 结构被认为是主要的查询接口；在现代使用中， :class:`_query.Query` 对象可以用于构建任何 SELECT 语句，包括复杂的组合，并且应该优先使用“map-to-selectable”方法。

一个类的多个映射器
==============================

在现代SQLAlchemy中，一个特定的类同时只被一个所谓的**primary**映射器映射。这个映射器涉及三个主要功能区域：查询、持久性和对映射类进行仪器化。主映射器的原理与事实相符，即 :class:`_orm.Mapper` 修改类本身，不仅将其持久化到特定的:class:`_schema.Table`，而且还在类上调用 :term:`instrumenting` 属性，这些属性根据表元数据特定地结构化。由于只有一个映射器可以实际仪器化类，因此不可能让多个映射器与等量地与其关联。


"非主" 映射器的概念已经存在于许多版本的SQLAlchemy中，但是从版本1.3开始，这个特性已经被弃用。唯一需要这样一个非主映射器的情况是针对可替代选择性地构建关系到一个类的情况。这种用例现在适用于 :class:`.aliased` 构造，并在 :ref:`relationship_aliased_class` 中描述。

关于通过在不同情况下完全持久化到不同表的类的用例，早期版本的SQLAlchemy提供了一种改编自Hibernate的功能，称为“entity name”特性。但是，在映射的类本身成为SQL表达式构造的源之后，即类自身的属性直接链接到映射的表列，这种用法在SQLAlchemy中变得不可行。该特性被删除，并替换为一种简单的面向方案的方法，用于在没有任何仪器化的歧义的情况下实现此任务 - 创建新的子类，每个子类单独映射。现在，这种模式作为一个食谱可用于`Entity Name<https://www.sqlalchemy.org/trac/wiki/UsageRecipes/EntityName>`_。