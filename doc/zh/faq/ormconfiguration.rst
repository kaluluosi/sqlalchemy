ORM配置
======

.. contents::
    :local:
     :class: faq
    :backlinks: none

.. _faq_mapper_primary_key:

如何映射没有主键的表？
----------------------------

为了映射到特定的表，SQLAlchemy ORM 至少需要有一个标为主键的列；
多列即复合主键当然也是可行的。这些列不必实际上被数据库识别为主键列，
不过最好是这样。 唯一需要的是这些列的行为与主键相同，
例如它们应该是行的唯一标识符并且不可以为空。

大多数 ORM 需要定义某种主键，因为内存中的对象必须对应于唯一标识的数据库表的行；
至少，这允许只影响该对象的行的 UPDATE 和 DELETE 语句。 然而，主键的重要性
远远超出此。在 SQLAlchemy 中，所有 ORM 映射的对象始终通过一个称为
  :term:`identity map`   的模式与其特定数据库行唯一链接在一起，
这是 SQLAlchemy 使用的工作单元系统的核心模式，也是 ORM 用法的最常见（和不常见的）模式之一。

.. 注意::
    重要的是要注意，我们只谈论 SQLAlchemy ORM；在基于 Core 构建的应用程序中，
    仅处理   :class:`_schema.Table`  对象、  :func:` _expression.select`  构造和类似内容，
    没有任何主键需要在表上存在或与表相关。

几乎所有情况下，表都有所谓的  :term:`候选键` ，这是唯一标识行的一列或多列。
如果表确实没有这个，而且具有实际上完全重复的行，则表不符合 `第一正规化 <https://en.wikipedia.org/wiki/First_normal_form>`_ 
并且无法映射。 否则，组成最佳候选键的任何列都可以应用于映射器::

    class SomeClass(Base):
        __table__ = some_table_with_no_pk
            __mapper_args__ = {
            "primary_key": [some_table_with_no_pk.c.uid, some_table_with_no_pk.c.bar]
        }

更好地使用完全声明的表元数据时，在这些列上使用 ``primary_key=True`` 标记：

    class SomeClass(Base):
        __tablename__ = "some_table_with_no_pk"

        uid = Column(Integer, primary_key=True)
        bar = Column(String, primary_key=True)

关系数据库中的所有表都应该具有主键。即使是多对多的关联表，
主键也将是两个关联列的组合：

.. sourcecode:: sql

    CREATE TABLE my_association (
      user_id INTEGER REFERENCES user(id),
      account_id INTEGER REFERENCES account(id),
      PRIMARY KEY (user_id, account_id)
    )


如何配置为 Python 保留字或类似的列？
--------------------------------------------------

映射时基于列的属性可以被赋予任何所需的名称。请参见   :ref:`mapper_column_distinct_names` 。

如何获取给定映射类的所有列，关系，映射属性等的列表？
-------------------------------------------------------------------------------------------------

所有这些信息都可以从   :class:`_orm.Mapper`  对象中获得。

要获取特定映射类的   :class:`_orm.Mapper` ，请在其上调用   :func:` _sa.inspect`  函数::

    from sqlalchemy import inspect

    mapper = inspect(MyClass)

从那里，可以通过属性访问有关类的所有信息，例如：

*  :attr:`_orm.Mapper.attrs`  - 所有映射属性的名称空间。属性本身是   :class:` .MapperProperty`  的实例，如果适用，
  它们包含可导致映射的 SQL 表达式或列的其他属性。

*  :attr:`_orm.Mapper.column_attrs`  - 映射属性名称空间
涵盖列和SQL 表达式属性。您可能希望使用  :attr:`_orm.Mapper.columns`  直接获取   :class:` _schema.Column`  对象。

*  :attr:`_orm.Mapper.relationships`  - 所有关系属性的名称空间。

*  :attr:`_orm.Mapper.all_orm_descriptors`  - 所有映射属性的名称空间，以及使用诸如   :class:` .hybrid_property` 、
  :class:`.AssociationProxy`  等系统定义的用户定义属性。

*  :attr:`_orm.Mapper.columns`  - 用于映射的   :class:` _schema.Column`  对象和其他命名 SQL 表达式的名称空间。

*  :attr:`_orm.Mapper.mapped_table`  - 此映射器所映射到的   :class:` _schema.Table`  或其他可选择项。

*  :attr:`_orm.Mapper.local_table`  - 与此映射器“本地”的   :class:` _schema.Table` ;
  在使用继承将映射器映射到复合可选择项的情况下，这与  :attr:`_orm.Mapper.mapped_table`  不同。

.. _faq_combining_columns:

我收到有关“隐式组合列 X 在属性 Y 下”的警告或错误消息
--------------------------------------------------

当映射包含两个由于名称而被映射到同一属性名称的列，并且没有表明这是有意的，
则出现该情况。由于映射类需要显式为存储独立值的每个属性指定名称；
当两个列具有相同的名称并且未加区别时，它们将落入同一属性下，
其效果是该属性从一个列复制值到另一个列，基于先将哪个列分配给属性的原则。

此行为通常是可取的，并且在使用外键关系将两个列链接在继承映射中的情况下，
可以无需警告地执行操作。 当发生警告或异常时，可以通过将这些列分配给具有不同名称的属性来解决问题，
或者如果希望将它们组合在一起，则使用   :func:`.column_property`  明确说明这一点。

给出如下示例：

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

截至 SQLAlchemy 版本 0.9.5，上述条件被检测到，并且将警告表示“id”列（即 A 和 B 中的 id）正在组合在同名属性“id”下，
这是一个严重的问题，因为它意味着“B”对象的主键将始终与其“a”的主键镜像。

解决此问题的映射如下：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(A):
        __tablename__ = "b"

        b_id = Column("id", Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey("a.id"))

假设我们确实希望“ A.id ”和“ B.id ”相互映射，尽管 “B.a_id” 是 “A.id” 相关的。我们可以
使用   :func:`.column_property`  将它们组合在一起：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(A):
        __tablename__ = "b"

        # probably not what you want, but this is a demonstration
        id = column_property(Column(Integer, primary_key=True), A.id)
        a_id = Column(Integer, ForeignKey("a.id"))

我正在使用 Declarative，并使用 ``and_()`` 或 ``or_()`` 设置 primaryjoin/secondaryjoin，
为此我收到有关外键的错误消息。
--------------------------------------------------------------------------------------------------

您这样做了吗？

    class MyClass(Base):
        # ....

        foo = relationship(
            "Dest", primaryjoin=and_("MyClass.id==Dest.foo_id", "MyClass.foo==Dest.bar")
        )

这是两个字符串表达式的 ``and_()``，SQLAlchemy 不能将其应用于任何映射。Declarative 允许将   :func:`_orm.relationship`  
参数指定为字符串，这些字符串使用 ``eval()`` 转换为表达式对象。但这不会在 ``and_()`` 表达式内发生 -
这是 Declarative 仅应用于字符串作为一个整体的特殊操作：  :func:`_orm.relationship` 。

    class MyClass(Base):
        # ....

        foo = relationship(
            "Dest", primaryjoin="and_(MyClass.id==Dest.foo_id, MyClass.foo==Dest.bar)"
        )

或者，如果您需要的对象已经可用，则可以跳过字符串：

    class MyClass(Base):
        # ....

        foo = relationship(
            Dest, primaryjoin=and_(MyClass.id == Dest.foo_id, MyClass.foo == Dest.bar)
        )

相同的思想适用于所有其他参数，如 ``foreign_keys``：

    # 不正确！
    foo = relationship(Dest, foreign_keys=["Dest.foo_id", "Dest.bar_id"])

    # 正确！
    foo = relationship(Dest, foreign_keys="[Dest.foo_id, Dest.bar_id]")

    # 还是正确的！
    foo = relationship(Dest, foreign_keys=[Dest.foo_id, Dest.bar_id])


    # 如果您正在使用来自所在类的列，请使用列对象！
    class MyClass(Base):
        foo_id = Column(...)
        bar_id = Column(...)
        # ...

        foo = relationship(Dest, foreign_keys=[foo_id, bar_id])

.. _faq_subqueryload_limit_sort:

为什么建议使用带有“LIMIT”（特别是使用“subqueryload()”）的“ORDER BY”？
---------------------------------------------------------------------------------

当未使用 ORDER BY 返回行时，关系数据库可自由以任何任意顺序返回匹配的行。
尽管此排序通常对应于表中的自然行顺序，但这并不适用于所有数据库和所有查询。
这意味着对使用 LIMIT 或 OFFSET 限制行或仅选择结果的查询而言，
如果匹配的行不止一行，则查询结果在某种程度上是不确定的。

尽管我们在通常返回自然排序行的数据库上可能不注意到这一点，
但如果我们还使用   :func:`_orm.subqueryload`  加载相关集合，则可能会更成为问题。
这可能不会按预期加载集合。

SQLAlchemy 通过发出单独的查询来实现   :func:`_orm.subqueryload` ，其结果与第一次查询的结果保持匹配。

我们看到如下的两个发出查询：

.. sourcecode:: pycon+sql

    >>> session.scalars(select(User).options(subqueryload(User.addresses))).all()
    {execsql}-- 主要查询
    SELECT users.id AS users_id
    FROM users
    {stop}
    {execsql}-- subqueryload 发出的“加载”查询
    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id FROM users) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id

第二个查询将第一个查询作为行源嵌入其中。
当内部查询使用 OFFSET 和/或 LIMIT 而不进行排序时，
两个查询可能不会看到相同的结果：

.. sourcecode:: pycon+sql

    >>> user = session.scalars(
    ...     select(User).options(subqueryload(User.addresses)).limit(1)
    ... ).first()
    {execsql}-- 主要查询
    SELECT users.id AS users_id
    FROM users
     LIMIT 1
    {stop}
    {execsql}-- subqueryload 发出的“加载”查询
    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id FROM users LIMIT 1) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id

根据数据库具体情况，我们可能会为这两个查询获取以下结果：

.. sourcecode:: text

    -- 查询 #1
    +--------+
    |users_id|
    +--------+
    |       1|
    +--------+

    -- 查询 #2
    +------------+-----------------+---------------+
    |addresses_id|addresses_user_id|anon_1_users_id|
    +------------+-----------------+---------------+
    |           3|                2|              2|
    +------------+-----------------+---------------+
    |           4|                2|              2|
    +------------+-----------------+---------------+

上述示例中，我们收到了 “帐户” 的两个行，其 “user.id” 均为 2，但没有获得 1 的行。
我们浪费了两行并未正确加载集合。这是一个隐蔽的错误，因为如果不查看 SQL 和结果，
则 ORM 将不会显示任何问题；如果我们访问我们拥有的“ addresses ”，
它将为集合发出惰性加载，并且我们将看不出实际上发生了什么错误。

解决此问题的方法是始终指定确定性排序顺序，
以便主查询始终返回相同的行集。这通常意味着您应该  :meth:`_sql.Select.order_by`  
表上的唯一列。主键是此目的的一个好选择::

    session.scalars(
        select(User).options(subqueryload(User.addresses)).order_by(User.id).limit(1)
    ).first()

请注意，  :func:`_orm.joinedload`  极端加载程序策略不受此问题的影响，
因为仅发出了一个查询，因此加载查询无法与主查询不同。
类似地，  :func:`.selectinload`  也不会出现此问题，因为它将其集合加载直接链接到刚加载的主键值。

.. seealso::

      :ref:`subquery_eager_loading` 