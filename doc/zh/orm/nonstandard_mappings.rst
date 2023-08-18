========================
非传统映射
========================

.. _orm_mapping_joins:

.. _maptojoin:

将类映射到多个表
=========================

Mapper不仅可以针对纯粹的表进行构造，还可以针对任意关系单元（称为“可选择的”）进行构造。例如，   :func:`_expression.join`  函数创建了一个包含多个表的可选择单元（具有自己的复合主键），这些表可以以与   :class:` _schema.Table`  相同的方式进行映射。

.. note::
    在下文中，原文有一个术语“mapper”，我未予翻译，因为有上下文即可理解。下同。

以下是一个例子：

    from sqlalchemy import Table, Column, Integer, String, MetaData, join, ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import column_property

    metadata_obj = MetaData()

    # 定义两个表格对象
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

    # 定义它们之间的连接。这
    # 是通过 user.id 和 address.user_id
    # 两列进行的。
    user_address_join = join(user_table, address_table)


    class Base(DeclarativeBase):
        metadata = metadata_obj


    # 将它们映射到类
    class AddressUser(Base):
        __table__ = user_address_join

        id = column_property(user_table.c.id, address_table.c.user_id)
        address_id = address_table.c.id

在上面的例子中，连接表达了 ``user`` 和 ``address`` 表格的列. ``user.id`` 和 ``address.user_id`` 列是通过外键相等，因此在映射中他们被定义为一个属性， ``AddressUser.id``，使用   :func:`.column_property`  来指示一个专门的列映射。根据配置的这一部分，当刷新发生时，映射将会将新的主键值从 ` `user.id`` 复制到 ``address.user_id`` 列中。

此外，``address.id`` 列映射显式地映射到名为 ``address_id`` 的属性。这是为了**消除歧义**，从 ``address.id`` 列的映射中区分同名的 ``AddressUser.id`` 属性，后者被指定为引用与 ``address.user_id`` 外键结合的 ``user`` 表格。

上述映射的自然主键是 ``(user.id, address.id)`` 的组合，因为这些是合并为一起的 ``user`` 和 ``address`` 表格的主键列。 ``AddressUser`` 对象的标识将是根据这两个值中的任意值，从 ``AddressUser`` 对象中表示为 ``(AddressUser.id，AddressUser.address_id)``。

当引用 ``AddressUser.id`` 列时，大多数 SQL 表达式将只使用映射列列表中的第一列，因为这两列是同义词。然而，对于特殊的用例，例如必须同时引用两个列的 GROUP BY 表达式，同时利用适当的上下文，即适应别名和类似的内容，在访问器  :attr:`.ColumnProperty.Comparator.expressions`  中使用。

    stmt = select(AddressUser).group_by(*AddressUser.id.expressions)

.. versionadded:: 1.3.17 添加了  :attr:`.ColumnProperty.Comparator.expressions`  访问器。

.. note::
    如上例所示，映射到多个表的映射支持持久性，即针对目标表格中的行的 INSERT、UPDATE 和 DELETE 操作。但是，它不支持在同一记录中在一个表上执行 UPDATE 操作并在其他表上执行 INSERT 或 DELETE 操作。也就是说，如果记录 PtoQ 被映射到“p”和“q”两个表格，其中一个基于“p”和“q”的 LEFT OUTER JOIN，如果进行修改“q” 表格中的数据的更新，行在 “q” 中必须存在; 如果主键标识已经存在，则不会发出 INSERT。如果行不存在，则对于大多数支持报告 UPDATE 所影响行数的 DBAPI 驱动程序，ORM 将无法检测到已更新的行并引发错误；否则，数据将被默默忽略。

    允许对相关行进行即时“插入”的一个示例配方可能会利用 .MapperEvents.before_update 事件，代码如下：

        from sqlalchemy import event

        @event.listens_for(PtoQ, "before_update")
        def receive_before_update(mapper, connection, target):
            if target.some_required_attr_on_q is None:
                connection.execute(q_table.insert(), {"id": target.id})

    在上述代码中，首先使用  :meth:`_schema.Table.insert`  创建一个 INSERT 构造，并且使用已经给出的   :class:` _engine.Connection`  执行它，这正是用于发出 SQL 用于刷新过程的那个连接。用户提供的逻辑必须检测到“p”到“q”的 LEFT OUTER JOIN 没有“q”一侧的条目。

.. _orm_mapping_arbitrary_subqueries:

将类映射到任意子查询
============================================

与映射到连接一样，一个普通的   :func:`_expression.select`  对象也可以与 mapper 一起使用。下面的示例片段说明了如何将名为“Customer”的类映射到一个包括对子查询的连接的   :func:` _expression.select`  ::

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

以上例子中，``customer_select`` 代表一个完整的行，包含 ``customers`` 表格的所有列和子查询 ``subq`` 公开的那些列，分别是 ``order_count``、``highest_order`` 和 ``customer_id``。将 ``Customer`` 类映射到此可选择项，然后创建包含这些属性的类。

当 ORM 持久化“Customer”的新实例时，实际上仅会向 ``customers`` 表格插入数据。这是因为 ``ordres`` 表格的主键在映射中没有被表示；ORM 仅会向已映射主键的表格插入数据。

.. note::
    映射到任意 SELECT 语句的做法，尤其是像上面的那样复杂的 SELECT 语句，是几乎不需要的。这种方法必然导致生成复杂的查询，往往比直接构造查询的查询效率低。这种做法在 SQLalchemy 的早期历史上就存在了，   :class:`_orm.Mapper`  构造旨在代表主要的查询界面，但在现代使用中，   :class:` _query.Query`  对象可以用来构造几乎任何 SELECT 语句，包括复杂的复合语句，应优先考虑“映射到可选择”的方法。

一个类的多个映射器
==============================

在现代 SQLalchemy 中，特定类同时只由一个所谓的**主要**映射器进行映射。这个映射器涉及三个主要功能领域：查询、持久化和对映射类的仪表化。主要映射器的理论基础是，   :class:`_orm.Mapper`  并非只将其持久地编制到特定的   :class:` _schema.Table`  上，而是也会 **仪表化** 类上的属性，特定地基于表格元数据进行结构化。由于只有一个映射器实际上可以更改类本身，而不仅仅是将其持久化到特定的   :class:`_schema.Table`  中，因此没有多个映射器能够与类以相同的度量关联。

“非主要”映射器的概念在 SQLalchemy 的早期版本中一直存在，但从版本 1.3 开始，该特性已被弃用。当仅构建与可选择项关联的关系时，这种非主要映射器是有用的。现在可以使用   :class:`.aliased`  来实现该用例，关于这个：ref:` relationship_aliased_class`.

关于将一个类完全持久化到不同的表格中的用例， SQLalchemy 的早期版本提供了一种功能，从 Hibernate 改编而来，称为“实体名称”功能。但是，在 SQLalchemy 将映射类本身成为 SQL 表达式构造的来源后，这个用例变得不可行，即，类的属性直接链接到映射表列。该特性已被删除，并替换为一种无歧义仪表化方法的简单配方，即创建单独映射的新子类。现在，这种模式可以作为配方使用，位于 `Entity Name <https://www.sqlalchemy.org/trac/wiki/使用/配方/EntityName>`_ 。