ORM配置
============

.. contents::
    :local:
    :class: faq
    :backlinks: none

.. _faq_mapper_primary_key:

如何映射一个没有主键的表？
-----------------------------------

SQLAlchemy ORM要映射到特定的表，需要至少有一列标识为主键列；多列的复合主键当然也是可行的。 但这些列不需要实际上被数据库知道为主键列，虽然最好是。重要的是列的行为要像主键一样，例如对于一个行是一个唯一且不可为空的标识符。

大多数ORM都要求对象定义了一些主键，因为内存中的对象必须对应于数据库表中的唯一可识别的行；至少，这允许将对象定位为仅影响该对象行而不影响其他行的UPDATE和DELETE语句。 但是，主键的重要性远远超出了这一点。 在SQLAlchemy中，所有ORM映射的对象始终链接到其特定的数据库行，使用的模式称为：term：`identity map`，它是SQLAlchemy使用的工作单元系统的中心模式，也是最常见的模式（以及不太常见的）ORM使用。

.. 注意::

    值得注意的是，我们只谈论SQLAlchemy ORM；建立在Core上并仅处理：class:`_schema.Table`对象，
    :func:`_expression.select`构造和类似的应用程序，不需要任何主键以任何形式存在于表格上（尽管在SQL中，所有表格
    应该实际上有某种形式的主键，否则您需要实际更新或删除特定的行）。

几乎所有情况下，表格都有名为term：`candidate key`的东西，它是唯一标识行的一列或一系列列。
如果表格确实没有这种情况，并且拥有真正重复的行，则该表格不对应于
“第一范式”于关系型数据库，不能映射。否则，组成最佳候选键的列可以直接应用于映射器：

    class SomeClass(Base):
        __table__ = some_table_with_no_pk
        __mapper_args__ = {
            "primary_key": [some_table_with_no_pk.c.uid, some_table_with_no_pk.c.bar]
        }

最好的方法是在使用完全声明的表元数据时，在这些列上使用primary_key=True标志：

    class SomeClass(Base):
        __tablename__ = "some_table_with_no_pk"

        uid = Column(Integer, primary_key=True)
        bar = Column(String, primary_key=True)

关系型数据库中的所有表都应该有主键。甚至一个多对多的关联表 - 主键是两个关联列的复合：

.. sourcecode:: sql

    CREATE TABLE my_association (
      user_id INTEGER REFERENCES user(id),
      account_id INTEGER REFERENCES account(id),
      PRIMARY KEY (user_id, account_id)
    )


如何配置一个Python保留字或类似的列？
--------------------------------------------

在映射中，基于列的属性可以被赋予任何所需的名称。参见
:ref:`mapper_column_distinct_names`。

如何获得一个映射类的所有列、关系、映射属性等的列表？
-------------------------------------------------------------------------------------------------

此信息都可以从:class:`_orm.Mapper`对象中获取。

要获取特定映射类的:attr:`_orm.Mapper`，请对其调用:func:`_sa.inspect`函数：

    from sqlalchemy import inspect

    mapper = inspect(MyClass)

从那里，可以通过以下属性访问有关类的所有信息：

* :attr:`_orm.Mapper.attrs` - 所有映射属性的名称空间。这些属性
  本身是:class:`.MapperProperty`的实例，其中包含其他属性，如果适用，则可以导致映射的SQL表达式或列。

* :attr:`_orm.Mapper.column_attrs` - 仅列和SQL表达式属性的映射属性名称空间。
  您可能希望使用:attr:`_orm.Mapper.columns`来直接获取:class:`_schema.Column`对象。

* :attr:`_orm.Mapper.relationships` - 所有:class:`.RelationshipProperty`属性的名称空间。

* :attr:`_orm.Mapper.all_orm_descriptors` - 所有映射属性的名称空间，以及使用系统定义的用户定义属性，例如：class:`.hybrid_property`、:class:`.AssociationProxy`等。

* :attr:`_orm.Mapper.columns` - 一个:class:`_schema.Column`对象和与映射相关的其他命名SQL表达式的名称空间。

* :attr:`_orm.Mapper.mapped_table` - 这个映射程序映射到的:class:`_schema.Table`或其他可选择的表。

* :attr:`_orm.Mapper.local_table` - "本地"表格是该映射程序的:class:`_schema.Table`；
  对于使用继承映射到组合可选择性的映射程序，这与: attr:`_orm.Mapper.mapped_table`不同。

.. _faq_combining_columns:

我收到有关“隐式组合列X的属性Y”的警告或错误消息
---------------------------------------------------------

这种情况指的是当映射包含两个由于名称而被映射到相同属性名称的列时，
但没有迹象表明这是有意的。映射类需要对每个将存储独立值的属性指定显式名称；
当两个列具有相同的名称并且未被消除歧义时，它们将落在同一属性下
效果是从一个列复制到另一个列，基于哪个列首先被分配到属性中。

这种行为通常是可取的，并且仅在由具有外键关系链接在一起的两个列时，
使用继承映射。当警告或异常发生时，可以通过将列分配给不同命名的属性来解决问题，或者如果希望将它们组合在一起，则使用:func:`.column_property`使这种情况明确。

例如：

    from sqlalchemy import Integer, Column, ForeignKey
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(A):
        __tablename__ = "b"

        id = Column(Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey("a.id"))

在SQLAlchemy版本0.9.5中，检测到上述条件，并警告“将“A”和“B”的”id“列”组合起来“，这是一个严重的问题，因为这意味着“B”对象的主键将始终镜像其“A”的主键。

可以解决缺陷的映射如下：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(A):
        __tablename__ = "b"

        b_id = Column("id", Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey("a.id"))

假设我们确实希望“A.id”和“B.id”相互镜像，尽管“B.a_id”是与“A.id”相关的。我们可以使用：func:`.column_property`组合它们在一起：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(A):
        __tablename__ = "b"

        # probably not what you want, but this is a demonstration
        id = column_property(Column(Integer, primary_key=True), A.id)
        a_id = Column(Integer, ForeignKey("a.id"))

我正在使用Declarative并使用“and_()”或“or_()”设置primaryjoin / secondaryjoin，我收到有关外键的错误消息。
------------------------------------------------------------------------------------------------------------------------------------------------------------------

您是否在做这个？：

    class MyClass(Base):
        # ....

        foo = relationship(
            "Dest", primaryjoin=and_("MyClass.id==Dest.foo_id", "MyClass.foo==Dest.bar")
        )

那是两个字符串表达式的“and_()”，SQLAlchemy无法应用任何映射。Declarative允许将：func:`_orm.relationship`参数指定为字符串，这些字符串使用“eval()”转换为表达式对象。但这不会在“and_()”表达式内发生 - 这是declarative仅适用于*作为字符串传递给primaryjoin或其他参数的整体性*的特殊操作：

   class MyClass(Base):
        # ....

        foo = relationship(
            "Dest", primaryjoin="and_(MyClass.id==Dest.foo_id, MyClass.foo==Dest.bar)"
        )

或者，如果您已经拥有所需的对象，则跳过字符串：

    class MyClass(Base):
        # ....

        foo = relationship(
            Dest, primaryjoin=and_(MyClass.id == Dest.foo_id, MyClass.foo == Dest.bar)
        )

同样的想法适用于所有其他参数，例如“foreign_keys”：

    # wrong !
    foo = relationship(Dest, foreign_keys=["Dest.foo_id", "Dest.bar_id"])

    # correct !
    foo = relationship(Dest, foreign_keys="[Dest.foo_id, Dest.bar_id]")

    # also correct !
    foo = relationship(Dest, foreign_keys=[Dest.foo_id, Dest.bar_id])


    # if you're using columns from the class that you're inside of, just use the column objects !
    class MyClass(Base):
        foo_id = Column(...)
        bar_id = Column(...)
        # ...

        foo = relationship(Dest, foreign_keys=[foo_id, bar_id])

.. _faq_subqueryload_limit_sort:

为什么推荐使用“LIMIT”与“ORDER BY”（特别是与“subqueryload()”一起）？
------------------------------------------------------------------------------------

当不为返回行的SELECT语句使用ORDER BY时，关系型数据库可以自由地以任意顺序返回匹配的行。虽然此排序很多时候对应于表格中行的自然顺序，但并非对于所有数据库和所有查询都是如此。这意味着即使有多行符合查询的标准，
使用“LIMIT”或“OFFSET”的任何查询，或者仅选择结果的第一行，舍弃其余部分，
在返回什么结果行方面是不确定的，假设存在多个匹配的行与查询的标准。
虽然我们对通常返回将行按自然顺序排列的数据库上的简单查询可能不会注意到这一点，但如果我们还使用：func:`_orm.subqueryload`将相关集合加载，则这将成为更大的问题，并且我们可能不会
正在按预期加载集合。

SQLAlchemy通过发出单独的查询来实现：func:`_orm.subqueryload`，其结果与第一个查询匹配。
我们看到如下两个发出的查询：

.. sourcecode:: pycon+sql

    >>> session.scalars(select(User).options(subqueryload(User.addresses))).all()
    {execsql}-- the "main" query
    SELECT users.id AS users_id
    FROM users
    {stop}
    {execsql}-- the "load" query issued by subqueryload
    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id FROM users) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id

第二个查询将第一个查询作为行的来源嵌入其中。当内部查询使用“OFFSET”和/或“LIMIT”而没有排序时，
这两个查询可能不会看到相同的结果：

.. sourcecode:: pycon+sql

    >>> user = session.scalars(
    ...     select(User).options(subqueryload(User.addresses)).limit(1)
    ... ).first()
    {execsql}-- the "main" query
    SELECT users.id AS users_id
    FROM users
     LIMIT 1
    {stop}
    {execsql}-- the "load" query issued by subqueryload
    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id FROM users LIMIT 1) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id

取决于数据库的具体情况，我们可能会得到以下结果的结果两个查询：

.. sourcecode:: text

    -- query #1
    +--------+
    |users_id|
    +--------+
    |       1|
    +--------+

    -- query #2
    +------------+-----------------+---------------+
    |addresses_id|addresses_user_id|anon_1_users_id|
    +------------+-----------------+---------------+
    |           3|                2|              2|
    +------------+-----------------+---------------+
    |           4|                2|              2|
    +------------+-----------------+---------------+

上面，我们接收了两个与“user.id”为2相对应的“addresses”行，并且没有任何行与1相对应。
我们浪费了两行并未实际加载集合。这是一个隐蔽的错误，因为除非查看SQL和结果，否则ORM不会显示任何问题；如果我们访问对于我们已经有的“User”的“addresses”，它将向懒加载发出集合查询
和我们看不出实际上出了什么问题。

解决此问题的方法是始终指定确定性排序，以便主查询始终返回相同的行集。这通常
意味着您应该:meth:`_sql.Select.order_by`在表上选择一个唯一的列。
主键对此是一个不错的选择：

    session.scalars(
        select(User).options(subqueryload(User.addresses)).order_by(User.id).limit(1)
    ).first()

请注意：func:`_orm.joinedload`贪婪加载器策略不会遇到相同的问题，因为只发出一个查询，因此加载查询
不能与主查询不同。类似地，：func:`.selectinload`贪婪加载策略也没有此问题，因为它将其集合加载直接链接到已加载的主键值。

.. seealso::

    :ref:`subquery_eager_loading`