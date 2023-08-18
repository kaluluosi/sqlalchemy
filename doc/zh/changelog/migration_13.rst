=============================
SQLAlchemy 1.3有什么新特性？
=============================

.. admonition:: 关于本文档

    本文档描述了SQLAlchemy版本1.2和1.3之间的变化。

介绍
=====

本指南介绍了SQLAlchemy版本1.3中的新特性，并记录了影响从1.2系列迁移应用程序的用户的更改。

请仔细查看行为变更部分，因为其中可能存在不兼容的行为更改。

常规
=====

.. _change_4393_general:

所有已弃用的元素都会生成弃用警告；添加了新的弃用
------------------------------------------------------------

1.3版本确保所有已弃用的行为和API（包括所有长期列为“遗留”状态多年的元素）都会生成``DeprecationWarning``警告。这包括使用参数（例如 :paramref:`.Session.weak_identity_map`）和类（例如 :class:`.MapperExtension`）时。虽然所有弃用项都已记录在文档中，但通常它们没有使用适当的重构文本指令或翻译成版本号。无论某个API功能是否实际上会生成弃用警告都是不一致的。通常的态度是，这些已弃用功能都被视为长期遗留功能，没有计划删除它们。

该更改包括：所有已记录的弃用现在使用文档中的适当重构文本指令和版本号进行记录，表达了特性或用例将在未来版本中删除的措辞被明确地表示（例如，不再使用永久遗留用例），并且使用任何这样的特性或用例肯定会生成``DeprecationWarning``，这在Python 3中以及使用Pytest等现代测试工具时现在在标准错误流中更明确。目标是将这些长期弃用的功能（追溯到版本0.7或0.6）全部开始完全删除，而不是将它们保留为“遗留”功能。此外，从版本1.3开始添加了一些主要的新弃用。由于SQLAlchemy有数千个开发人员在14年的现实世界中使用，因此可以指向一种流畅混合使用实例的方式，以消除广泛使用的特性和模式。

更大的背景是SQLAlchemy试图调整面向Python 3-only的世界，以及类型注释的世界，为此，SQLAlchemy有一些**暂定的**计划，即进行大规模的重新构建 SQLAlchemy，希望大大减少API的认知负荷，以及对核心和ORM之间的许多实现和使用差异进行重大修补。由于这两个系统在SQLAlchemy的首次发布后剧烈演变，特别是ORM仍然保留了许多“附加”行为，这些行为使得Core和ORM之间的隔离墙过高。通过提前将API聚焦于每个受支持的用例的单一模式，将来迁移到显著更改的API的工作变得更加简单。

有关1.3中添加的大部分主要弃用，请参见下面链接的部分。

.. 参见::

   :ref:`change_4393_threadlocal`
   :ref:`change_4393_convertunicode`
   :ref:`change_4423`

:ticket:`4393`

新功能和改进 - ORM
===================

.. _change_4423:

关系到AliasedClass可以替换非主要映射器
-----------------------------------------

“非主要映射器”是在:ref:`orm_imperative_mapping`风格中创建的:class:`_orm.Mapper`，它充当已经映射到与不同类型的可选择关联的类的附加映射器。非主要映射器的根源在0.1、0.2系列的SQLAlchemy版本中，当时预计:class:`_orm.Mapper`对象将是主要的查询构造界面，而:class:`_query.Query`对象尚不存在。

随着:class:`_query.Query`和以后的:class:`.AliasedClass`构造的出现，大多数非主要映射器的用途都消失了。这是一个好事，因为SQLAlchemy从0.5系列开始完全停用了“经典”映射，转而采用声明性系统。

当一个非主要映射器与选择性可选择的不同选择集合上的对象存在困难定义的:func:`_orm.relationship`配置实现时，仍然存在一个用例，这反映了一个用于非主要映射器的需求。在这种情况下，将非主要映射器与可选的选择集合作为映射目标，而不是尝试构建涵盖特定对象之间关系所有复杂性的:paramref:`_orm.relationship.primaryjoin`，可以使过程变得更加容易。

随着这种用例的普及，它的局限性变得明显，其中包括：非主要映射器难以针对添加新列的可选择执行配置，映射器不继承原始映射器的关系，在非主要映射器上显式配置的关系在使用程序装载器选项时无法正常工作，非主要映射器还没有提供可用于查询的完全功能的以列为基础的属性的全功能命名空间（再次提醒，在旧的0.1-0.4天中，人们会直接使用:class:`_schema.Table`对象来与ORM配合使用）。

缺失的一部分是允许 :func:`_orm.relationship` 直接引用 :class:`.AliasedClass`。:class:`.AliasedClass`已经做了我们希望非主要映射器执行的所有操作。它允许从可选的可选择执行加载现有映射的类，它继承已存在映射器的所有属性和关系，它非常适合执行装载程序选项，它提供了可以像类一样混合到查询中的类似于类的对象。通过这个变化，前非主要映射器的配方在 :ref:`relationship_aliased_class` 中更改为预支类。

在新方法中，所有这些冗verbose都消失了，并且在创建关系时直接引用了附加列::

    j = join (B，D，D.b_id == B.id).join (C，C.id == D.c_id)

    B_viacd = aliased（B，j，flat = True）

    A.b = relationship(B_viacd, primaryjoin = A.b_id == j.c.b_id)

非主要映射器现在已经被弃用，最终的目标是将传统的映射全部取消。declarative API将成为映射的单一手段，这有望允许内部改进和简化，以及更清晰的文档故事。

:ticket:`4423`

.. _change_4340:

选择加载不再使用简单一对多的连接
------------------------------------------------------------

1.2中引入的“selectin”加载特性引入了一种极其高效的新方法来急切地加载集合，在许多情况下比“子查询”急切加载的效率要高得多，因为它不依赖于重新说明原始SELECT查询，并且使用一个简单的IN子句。但是，“selectin”加载仍然依赖于渲染父代和相关表之间的连接，因为它需要该行中的父主键值以匹配行。在1.3中，添加了一种新的优化，将省略大多数简单一对多加载的最常见情况，其中相关行已包含表达式的父主键行的主键值。这可以使ORM在不使用JOIN或子查询的情况下一次加载大量集合，从而提供了巨大的性能提升。

给定一个映射::

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        bs = relationship("B", lazy="selectin")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey(“a.id”))

在1.2 “selectin”加载的版本中，加载 A 到 B 的方式看起来像是：

.. sourcecode:: sql
 
    SELECT a.id AS a_id FROM a
    SELECT a_1.id AS a_1_id, b.id AS b_id, b.a_id AS b_a_id
    FROM a AS a_1 JOIN b ON a_1.id = b.a_id
    WHERE a_1.id IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?) ORDER BY a_1.id
    (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

使用新行为时，加载看起来是这样的：

.. sourcecode:: sql

    SELECT a.id AS a_id FROM a
    SELECT b.a_id AS b_a_id, b.id AS b_id FROM b
    WHERE b.a_id IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?) ORDER BY b.a_id
    (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

该行为正在自动发布，使用类似于懒加载的启发式方法来确定相关实体是否可以直接从标识映射中获取。但是，与大多数查询功能一样，由于多态加载方案的高级方案，该功能的实现变得更加复杂。如果遇到问题，用户应报告错误，但是更改还包括一个标志：：paramref:`_orm.relationship.omit_join`，在:func:`_orm.relationship`上可以设置为``False``以禁用此优化。

:ticket:`4340`

.. _change_4359:

许多对一查询表达式的行为改进
------------------------------------------------------------

在构建查询时，将相对于对象值（例如：param：`User.id` = ``5``）将许多对一关系与对象值进行比较的查询表达式比较，例如：

    u1 = session.query(User).get(5)

    query = session.query(Address).filter(Address.user == u1)

上面的表达式``Address.user == u1``，它最终编译成基于``User``对象的主键列的SQL表达式，例如``"address.user_id = 5"``，使用延迟可调用包装器来在绑定表达式的时候尽可能晚地检索值``5``。这既适用于``Address.user == u1``表达式可能针对尚未刷新的具有服务器生成主键值的``User``对象的用例，也适用于表达式始终在属性创建之后检索正确结果的用例，即使此时属性的值已更改。

但是，这种行为的副作用是，如果在表达式评估时``u1``过期，它将会生成额外的SELECT语句，并且如果``u1``也从 :class:`.Session`中分离，则会引发错误：：

    u1 = session.query(User).get(5)

    query = session.query(Address).filter(Address.user == u1)

    session.expire(u1)
    session.expunge(u1)

    query.all()  # <-- would raise DetachedInstanceError

当对象的过期/清除时可能隐式发生此情况:class:`.Session`，因为``Address.user == u1``表达式仅引用其:class:`.InstanceState`，而不是对象本身。现在的修复方法是，允许``Address.user == u1``表达式在尝试通常在表达式编译时检索或加载值时执行该值，但如果对象已分离并且已过期，则在:class:`.DetachedInstanceError`中检索它，因为该属性试图在 :class:`.InstanceState` 上将先前值检索回来时，检索该属性。

但是，这种行为的副作用是，如果在表达式评估时``u1``过期，它将会生成额外的SELECT语句，并且如果``u1``也从:class:`.Session`中分离，则会引发错误：

    u1 = session.query(User).get(5)

    query = session.query(Address).filter(Address.user == u1)

    session.expire(u1)
    session.expunge(u1)

    query.all()  # <-- would raise DetachedInstanceError

该修复方案是，允许``Address.user == u1``表达式在尝试检索或加载该值时检索该值，但如果该对象被分离，并且已过期，则在从:class:`.InstanceState`检索该值时从一个新的机制中检索它，在该机制下，每当属性被过期时，都会为该状态记录此属性的上一个已知值。该机制仅在需要属性/: class:`.InstanceState`的表达式功能时才启用，以节省性能/内存开销。

渐进属性API特性用于指示无法计算该值时的特定错误消息，这两种情况是当列属性从未设置时以及在第一次评估时已过期和现在分离时。在所有情况下，不再引发:class:`.DetachedInstanceError`。

:ticket:`4359`

.. _change_4353:

许多对一替换不会为“old”对象的"raiseload"或"detached"引发错误
---------------------------------------

考虑在一个集合上进行延迟加载，以便加载“old”值的情况，如果关系未指定 :paramref:`_orm.relationship.active_history` 标志，将不会为已分离的对象引发``AssertionError``::

    a1 = session.query(Address).filter_by(id=5).one()

    session.expunge(a1)

    a1.user = some_user

上面，当在未附加的``a1``对象上替换``.user``属性时，``.user``属性将从标识映射中尝试检索``.user``属性的以前值，会引发:class:`.DetachedInstanceError`，因为属性试图从已分离的``some_user``对象中检索值。当``u1``对象已过期并且已分离时，通常会隐式发生对象的过期/清除，因为``Address.user == u1``表达式仅引用其:class:`.InstanceState`，而不是该对象本身。解决办法是，这个操作现在可以在不加载旧值的情况下继续。

对于“引发加载”策略（``lazy="raise"``），同样的更改也发生了：

    class Address(Base):
        # ...

        user = relationship("User", ..., lazy="raise")

以前，将``a1.user``关联对象调用“引发加载”会由于属性试图检索以前的值而引发“raiseload”异常。即使在加载“old”值时也不会引发这个断言了。

:ticket:`4353`


.. _change_4354:

为ORM属性执行del操作
--------------------------

Python的“del”操作对映射属性（标量列或对象引用）实际上没有用处。现在已经添加了修复程序，其中“del”操作正确工作，其中“del”操作大致等同于将属性设置为“None”值：：

    some_object = session.query(SomeObject).get(5)

    del some_object.some_attribute  #从SQL的角度来看，像是"= None"

:ticket:`4354`


.. _change_4257:

在InstanceState对象上添加了info字段
-------------------------------------

向:class:`.InstanceState`类添加了``.info``字典，它来自于对映射对象调用:func:`_sa.inspect`。这使得用户可以为对象添加其他信息，这些信息将随着对象在内存中的完整生命周期一起传递：

    from sqlalchemy import inspect

    u1 = User(id=7, name="ed")

    inspect(u1).info["user_info"] = "7|ed"

:ticket:`4257`

.. _change_4196:

基于水平分片的查询扩展支持批量更新和删除方法
-------------------------------------------------------------

:class:`.ShardedQuery`扩展对象支持:meth:`_query.Query.update`和:meth:`_query.Query.delete`批量更新/删除方法。当调用它们时，将查询选择器调用，以根据给定的标准在多个分片上运行更新/删除。

:ticket:`4196`

协会代理改进
--------------

虽然没有任何特定的原因，但是在本周期中，Association代理扩展经历了许多改进。

.. _change_4308:

Association代理具有新的cascade_scalar_deletes标志
------------------------------------------------------------

给定以下映射::

    class A(Base):
        __tablename__ = "test_a"
        id = Column(Integer, primary_key=True)
        ab = relationship("AB", backref="a", uselist=False)
        b = association_proxy(
            "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
        )


    class B(Base):
        __tablename__ = "test_b"
        id = Column(Integer, primary_key=True)
        ab = relationship("AB", backref="b", cascade="all, delete-orphan")


    class AB(Base):
        __tablename__ = "test_ab"
        a_id = Column(Integer, ForeignKey(A.id), primary_key=True)
        b_id = Column(Integer, ForeignKey(B.id), primary_key=True)

将``A.b``赋值将生成一个``AB``对象::


    a.b = B()

如果设置 :paramref:`.AssociationProxy.cascade_scalar_deletes`标志，将``A.b``设置为``None``将同时删除``A.ab``。默认行为仍然是使``a.ab``保持原样::

    a.b = None
    assert a.ab is None

虽然看起来直观，这个逻辑应该只查看现有关系的“级联”属性，但是，从这单独的属性本身并不明确该属性是否应该被删除，因此该行为作为一个显式选项可用。

此外，现在``del``针对标量运作方式与将属性设置为``None``大致相同::

    del a.b
    assert a.ab is None

:ticket:`4308`

.. _change_3423:

Association代理在每个类上按类存储类特定状态
------------------------------------------------

:class:`.AssociationProxy`对象根据父映射类做出许多决策。虽然:class:`.AssociationProxy`最初只是一个相对简单的“getter”，但很快就显而易见，它还需要对其关联的属性进行决策，例如它所参考的是什么类型的属性，即标量还是集合，映射对象还是简单值等等。为了实现这一点，它需要检查它引用的映射属性或其他描述符或属性，这些映射属性或其他描述符或属性是从其父类引用的。但是，在Python描述符机制中，描述符仅在它在该类的上下文中被访问时（例如将``MyClass.some_descriptor``调用``__get __（）``方法，它将传递类）。因此，:class:`.AssociationProxy`对象会存储仅特定于该类的内部状态，但仅在将:class:`.AssociationProxy`作为描述符调用``__get __（）``的基础上才会修改其内部状态，尝试在不先访问:class:`.AssociationProxy`作为描述符的情况下预先检查该状态将引发错误。此外，它假设``__get __（）``首次看到的第一个类是它需要知道的唯一父类。尽管在1.1中已经改进了此缺陷，但是它仍然留下了一些缺陷以及确定最佳“所有者”类的复杂问题。

这些问题现已得到解决，因为 :class:`.AssociationProxy`在``__get__（）``被调用时不再修改其自己的内部状态；相反，生成了一种新对象 :class:`.AssociationProxyInstance`用于每个特定于映射的父类处理所有与该映射父类相关的特定状态（当父类没有映射时，不会生成：class：`。AssociationProxyInstance`）。“拥有”类的概念在本质上改进了1.1，现在已经被更换为一种方法，其中AP现在可以同等地处理任意数量的“拥有”类。

为了适应希望检查不一定直接调用``__get __（）``的:class:`.AssociationProxy`的应用程序的状态，而不一定调用``__get __（）``，添加了一个新方法:meth:`.AssociationProxy.for_class`，它提供了直接访问特定于类的:class:`.AssociationProxyInstance`，如下所示：：

    class User(Base):
        # ...

        keywords = association_proxy("kws", "keyword")


    proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)

一旦我们有了 :class:`.AssociationProxyInstance`对象，在上面的示例中存储在``proxy_state``变量中，我们就可以查看与``User.keywords``代理特定的属性，例如``target_class``：

    >>> proxy_state.target_class
    Keyword


:ticket:`3423`

.. _change_4351:

面向列的比较运算符现在提供了用于面向列的目标的标准列运算符
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

给定了一个:class:`.AssociationProxy`，其中目标是一个数据库列，并且不是一个对象引用或另一个关联代理::


    class User(Base):
        # ...

        elements = relationship("Element")

        # column-based association proxy
        values = association_proxy("elements", "value")


    class Element(Base):
        # ...

        value = Column(String)

那么现在可以使用标准列运算符，例如``like``：

.. sourcecode:: pycon+sql

    >>> print (s. query(User).filter(User.values.like("%foo%"))）
    {printsql}SELECT "user".id AS user_id
    FROM "user"
    WHERE EXISTS (SELECT 1
    FROM element
    WHERE "user".id = element.user_id AND element.value LIKE :value_1)

``equals``：

.. sourcecode:: pycon+sql

    >>> print(s.query(User).filter(User.values == "foo"))
    {printsql}SELECT "user".id AS user_id
    FROM "user"
    WHERE EXISTS (SELECT 1
    FROM element
    WHERE "user".id = element.user_id AND element.value = :value_1)

当将其与``None``进行比较时，IS NULL表达式将与相关行不再存在的测试一起增强；这与以前相同：

.. sourcecode:: pycon+sql

    >>> print (s. query(User).filter(User.values == None)）
    {printsql}SELECT "user".id AS user_id
    FROM "user"
    WHERE (EXISTS (SELECT 1
    FROM element
    WHERE "user".id = element.user_id AND element.value IS NULL)) OR NOT (EXISTS (SELECT 1
    FROM element
    WHERE "user".id = element.user_id))

请注意， :meth:`.ColumnOperators.contains` 操作符实际上是一个字符串比较操作符； **这是一种行为更改**，因为先前，关联代理仅使用``.contains``作为列表存在运算符。对于基于列的比较，现在的行为类似于“like”：

.. sourcecode:: pycon+sql

    >>> print(s.query(User).filter(User.values.contains("foo")))
    {printsql}SELECT "user".id AS user_id
    FROM "user"
    WHERE EXISTS (SELECT 1
    FROM element
    WHERE "user".id = element.user_id AND (element.value LIKE '%' || :value_1 || '%'))

为了测试值``"foo"``是否是``User.values``集合的简单成员资格，应使用相等运算符（例如``User.values == 'foo'``）；在之前的版本中，这也可以工作。

当使用基于对象的协会代理与集合时，行为与以前相同，测试集合成员资格，例如，给定一个映射::

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        user_elements = relationship("UserElement")

        # object-based association proxy
        elements = association_proxy("user_elements", "element")


    class UserElement(Base):
        __tablename__ = "user_element"

        id = Column(Integer, primary_key=True)
        user_id = Column(ForeignKey("user.id"))
        element_id = Column(ForeignKey("element.id"))
        element = relationship("Element")


    class Element(Base):
        __tablename__ = "element"

        id = Column(Integer, primary_key=True)
        value = Column(String)

``。contains()``方法仍然产生与以前相同的表达式，测试``User.elements``的列表中是否存在``Element``对象：

.. sourcecode:: pycon+sql

    >>> print(s.query(User).filter(User.elements.contains(Element(id=1))))
    SELECT "user".id AS user_id
    FROM "user"
    WHERE EXISTS (SELECT 1
    FROM user_element

:ticket:`4351`

.. _change_4356:

当列型AssociationProxy目标时，== None协议跟empty匹配
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

给定具有非空默认值的SQLAlchemy列的映射，例如：

    sa.Column(sa.String, default="value")

如果对应于“value”的属性被映射到具有默认值的这样的列，则``.xxx == None``协议现在实际上匹配空字符串。例如，如果考虑使用内置调查机制检查一个代理对象是否为空::


    # 查询所有长度为0的元素
    query.filter_by(attr="")

那么这个查询同时还将查找那些被映射到某个具有默认值为“value”的列上的其他映射，这个值既不是数量级也不是NULL，例如：


    class SomeMappedObject(Base):
        __tablename__ = "somemappedthing"
        id = Column(Integer, primary_key=True)
        name = Column(String, default="value")

    proxy = association_proxy("columns", "name")


    mapped = SomeMappedObject()
    mapped.columns = [SomeMappedThingColumn(name="foo"), SomeMappedThingColumn(name="")]

    assert proxy == ["foo", ""]


现在，当代理也适用于代理列时，``== None``协议也会匹配空字符串，因为在SQLAlchemy中，这通常意味着相应的列具有无值的情况，如果该列是非空的，则具有默认值。如果不需要此行为，则可以使用“== None”代替以前的比较，或者显式添加``is``运算符。


:ticket:`4356`

.. _change_4365:

在ManyToOne / OneToOne加载时可以授予“duplicates”警告
------------------------------------------------------------

在尝试加载ManyToOne/OneToOne关联时，如果关联键不是唯一的，将为警告提供一个复制当前行的信息。

:ticket:`4365`

.. _change_4093:

在更新时，不需要apply_updates参数
----------------------------------

bind对象的:meth:`_engine.Cursor.execute`方法现在将不接受``apply_updates``参数，这是一个无用的参数：::

    with engine.connect() as conn:
        conn.execute(
            table.update().values(a=b),
            {table.c.a: 1},
            apply_updates=False,  # not used
        )

现应该忽略这个参数：：

    with engine.connect() as conn:
        conn.execute(
            table.update().values(a=b),
            {table.c.a: 1},
        )

:ticket:`4093`WHERE "user".id = user_element.user_id AND :param_1 = user_element.element_id)

总体而言，这项更改是基于:ref:`change_3423`的架构更改启用的；由于代理现在在生成表达式时会产生额外的状态，因此有一个对象目标和一个列目标版本的:class:`.AssociationProxyInstance`类。

:ticket:`4351`

代理现在强引用父对象
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

协会代理集合维护只有父对象的弱引用的长期行为被撤销；现在代理将维护父对象的强引用，只要代理集合本身仍然存在于内存中，也将消除“陈旧的关联代理”错误。这个改变正在实验性的基础上进行，看看是否会引起任何副作用。

例如，给定一个具有协会代理映射的情况::

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        bs = relationship("B")
        b_data = association_proxy("bs", "data")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))
        data = Column(String)


    a1 = A(bs=[B(data="b1"), B(data="b2")])

    b_data = a1.b_data

之前，如果``a1``被删除了::

    del a1

``a1``被删除后，尝试迭代``b_data``集合会导致“stale association proxy，parent object has gone out of scope”的错误。这是因为协会代理需要访问实际的``a1.bs``集合以产生视图，在此改变之前它只维护了对``a1``的弱引用。特别是，当执行内联操作时，用户通常会遇到此错误，例如::

    collection = session.query(A).filter_by(id=1).first().b_data

上面的查询由于``A``对象将在实际使用``b_data``集合之前被垃圾收集器收集而引起错误。

现在的改变是，``b_data``集合现在维护一个对``a1``对象的强引用，这样它就会留在那儿::

    assert b_data == ["b1", "b2"]

这个改变引入了一个副作用，即当应用程序像上面那样传递集合时，“**只有在集合也被丢弃时，父对象才会被垃圾回收**”。与始终如一，如果``a1``在特定的:class:`.Session`内是持久的，它将保持为该会话的状态，直到被垃圾收集。

请注意，如果此更改导致问题，可能会对其进行修改。

:ticket:`4268`

.. _change_2642:

多列代理和字典的替换实现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

将集合或字典分配给协会代理集合现在应该正常工作，而不是重新创建现有键/值对协会代理成员，从而导致由于删除+插入相同对象而潜在刷新失败，它现在应该只在适当的情况下创建新的协会对象::

    class A(Base):
        __tablename__ = "test_a"

        id = Column(Integer, primary_key=True)
        b_rel = relationship(
            "B",
            collection_class=set,
            cascade="all, delete-orphan",
        )
        b = association_proxy("b_rel", "value", creator=lambda x: B(value=x))


    class B(Base):
        __tablename__ = "test_b"
        __table_args__ = (UniqueConstraint("a_id", "value"),)

        id = Column(Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey("test_a.id"), nullable=False)
        value = Column(String)


    # ...

    s = Session(e)
    a = A(b={"x", "y", "z"})
    s.add(a)
    s.commit()

    # re-assign where one B should be deleted, one B added, two
    # B's maintained
    a.b = {"x", "z", "q"}

    # only 'q' was added, so only one new B object.  previously
    # all three would have been re-created leading to flush conflicts
    # against the deleted ones.
    assert len(s.new) == 1

:ticket:`2642`

.. _change_1103:

一对多反向引用在删除操作期间检查集合重复项
---------------------------------------------------------------

ORM映射的集合，典型的Python序列，通常为Python“列表”，包含重复的情况下，如果对象从一个位置删除而没有从其他位置删除，那么多对一的反向引用会将其属性设置为``None``，即使仍然表示一个对多的集合事实上仍存在。 尽管一对多集合在关系模型中不能有重复项，但使用ORM映射的 :func:`_orm.relationship`，可以在内存中具有有重复项的限制。 在特定的Python“交换”操作中，将一个副本暂时存在于列表中是固有的。给定标准的一对多/多对一设置。

:: 

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        bs = relationship("B", backref="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

如果我们有一个带有两个``B``成员的``A``对象，并执行一个交换：

::

    a1 = A(bs=[B(), B()])

    a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]

在上述操作期间，中继Python ``__setitem__`` ``__delitem__``方法释放中间状态，在集合中第二个``B（）``对像存在两次。当``B（）``对象被从一个位置删除时，``B.a``反向引用将设置引用为``None``，导致在刷新期间从``A``和``B``对象之间移除链接。即使在关系模型中不能在一个对多的一侧存在重复项，但是在内存中具有重复值的ORM映射的 :func:`_orm.relationship`，该活动也是固有的，重复状态的限制不能持久化或从数据库中检索。特别地，当列表中暂时存在一个副本时，具有重复项的情况是Python“交换”操作的固有属性。是一个标准的一对多/多对一设置。

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        bs = relationship("B", backref="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

如果我们有一个带有两个``B``成员的``A``对象，并执行一个交换：

::

    a1 = A(bs=[B(), B()])

    a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]

在上述操作期间，中继Python ``__setitem__`` ``__delitem__``方法释放中间状态，在集合中第二个``B（）``对像存在两次。当``B（）``对象被从一个位置删除时，``B.a``反向引用将设置引用为``None``，导致在刷新期间从``A``和``B``对象之间移除链接。即使在关系模型中不能在一个对多的一侧存在重复项，但是在内存中具有重复值的ORM映射的 :func:`_orm.relationship`，该活动也是固有的，重复状态的限制不能持久化或从数据库中检索。特别地，当列表中暂时存在一个副本时，具有重复项的情况是Python“交换”操作的固有属性。

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        bs = relationship("B", backref="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

如果我们有一个带有两个``B``成员的``A``对象，并执行一个交换：

::

    a1 = A(bs=[B(), B()])

    a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]

在上述操作期间，中继Python ``__setitem__`` ``__delitem__``方法释放中间状态，在集合中第二个``B（）``对像存在两次。当``B（）``对象被从一个位置删除时，``B.a``反向引用将设置引用为``None``，导致在刷新期间从``A``和``B``对象之间移除链接。即使在关系模型中不能在一个对多的一侧存在重复项，但是在内存中具有重复值的ORM映射的 :func:`_orm.relationship`，该活动也是固有的，重复状态的限制不能持久化或从数据库中检索。特别地，当列表中暂时存在一个副本时，具有重复项的情况是Python“交换”操作的固有属性。

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        bs = relationship("B", backref="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

如果我们有一个带有两个``B``成员的``A``对象，并执行一个交换：

::

    a1 = A(bs=[B(), B()])

    a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]

在上述操作期间，中继Python ``__setitem__`` ``__delitem__``方法释放中间状态，在集合中第二个``B（）``对像存在两次。当``B（）``对象被从一个位置删除时，``B.a``反向引用将设置引用为``None``，导致在刷新期间从``A``和``B``对象之间移除链接。即使在关系模型中不能在一个对多的一侧存在重复项，但是在内存中具有重复值的ORM映射的 :func:`_orm.relationship`，该活动也是固有的，重复状态的限制不能持久化或从数据库中检索。特别地，当列表中暂时存在一个副本时，具有重复项的情况是Python“交换”操作的固有属性。

当ORM映射的集合以Python序列的形式存在并具有重复性时，发现多对一的反向引用将其属性设置为``None``的问题，即使一对多/多对一集合在关系模型中不允许具有重复值，在ORM映射中具有重复值的情况下可以出现限制（即Python“交换”操作固有的可能性）。改变确保在类似的情况下不会清除多对一关系上的属性，从而维护了集合的完整性。

:ticket:`1103`
连接池通常由 :func:`_sa.create_engine` 使用，称为 :class:`.QueuePool`。该池使用 Python 内置的 ``Queue`` 类型的对象来存储等待使用的数据库连接。``Queue`` 具有先进先出的行为，旨在提供对持久池中的数据库连接的轮询使用。但是，这可能会带来潜在的缺点：当池的利用率较低时，一系列重用连接导致防止服务器端超时策略关闭这些连接。为适应此用例，增加了一个新标志：:paramref:`_sa.create_engine.pool_use_lifo`，它将 ``.get()`` 方法的顺序与正常相反，从队列的开头而不是结尾获取连接，从本质上将“队列”变为“堆栈”（考虑到这会太冗长，因此没有添加一个名为 ``StackPool`` 的全新池）。

.. seealso::

    :ref:`pool_use_lifo`

核心要点变化
==============

.. _change_4481:

强制转换字符串 SQL 片段为 text() 完全移除
--------------------------------------------

首次在 1.0 版本中添加的警告，在 :ref:`migration_2992` 中描述。现在已将其转换为异常。持续关注使用像 :meth:`_query.Query.filter` 和 :meth:`_expression.Select.order_by` 这样的方法传递的字符串片段强制转换为 :func:`_expression.text` 组成的内容，即使这发出了警告。对于 :meth:`_expression.Select.order_by`、:meth:`_query.Query.order_by`、:meth:`_expression.Select.group_by` 和 :meth:`_query.Query.group_by` 来说，仍然会将字符串标签或列名解析为相应的表达式结构，但是如果解析失败，则会引发 :class:`.CompileError`，从而防止直接呈现原始 SQL 文本。

:ticket:`4481`

.. _change_4393_threadlocal:

不推荐使用“threadlocal”引擎策略
------------------------------------

“threadlocal 引擎策略”是在 SQLAlchemy 0.2 左右添加的，作为解决 SQLAlchemy 0.1 中的标准操作方式总结出的问题的一种解决方案。回顾过去，可以说 SQLAlchemy 第一个发布版本在每个方面都是“alpha”，人们担心已经有太多用户已经使用现有 API，无法更改它显得相当荒谬。

SQLAlchemy 的最初使用模型如下所示：

    engine.begin()

    table.insert().execute(parameters)
    result = table.select().execute()

    table.update().execute(parameters)

几个月的现实世界使用后，很明显，假装“连接”或“事务”是隐藏的实现细节是一个坏主意，特别是当某个人需要一次处理多个数据库连接时。因此，我们今天看到的使用范例被引入了，由于当时 Python 中还不存在上下文管理器，因此省略了上下文管理器：

    conn = engine.connect()
    try:
        trans = conn.begin()

        conn.execute(table.insert(), parameters)
        result = conn.execute(table.select())

        conn.execute(table.update(), parameters)

        trans.commit()
    except:
        trans.rollback()
        raise
    finally:
        conn.close()

上述模式是人们所需的，但由于它仍然有点冗长（因为没有上下文管理器），因此仍然保留了旧的工作方式，成为“threadlocal 引擎策略”。

今天，在 Core 的工作更加简洁，甚至比原始模式更加简洁，得益于上下文管理器：

    with engine.begin() as conn:
        conn.execute(table.insert(), parameters)
        result = conn.execute(table.select())

        conn.execute(table.update(), parameters)


在此时，任何仍然依赖“threadlocal”方式的剩余代码都将通过此停用来更新为更现代的方式。该功能应在下一个主要系列（如 1.4）的 SQLAlchemy 中完全删除。连接池参数 :paramref:`_pool.Pool.use_threadlocal` 也已被弃用，因为它在大多数情况下实际上没有任何影响，还有 :meth:`_engine.Engine.contextual_connect` 方法，它通常与 :meth:`_engine.Engine.connect` 方法同义，除非使用 threadlocal 引擎。

:ticket:`4393`


.. _change_4393_convertunicode:

已弃用 "convert_unicode" 参数
-------------------------------

已弃用 :paramref:`.String.convert_unicode` 和 :paramref:`_sa.create_engine.convert_unicode` 参数。这些参数的目的是告知 SQLAlchemy 在 Python 2.x 中确保传入的 Python Unicode 对象在传递到数据库之前进行了编码，并期望从数据库接收的是 bytestring。在 Python 3 开始使用后，DBAPIs 开始开始更充分地支持 Unicode，更重要的是默认情况下支持 Unicode。但是，特定 DBAPI 在哪些条件下返回结果中的 Unicode 数据的特定情况，以及是否接受 Python Unicode 值作为参数，仍然非常复杂。这是“convert_unicode”标志过时的开始，因为它们不再足以确保仅在需要时执行编码/解码。取而代之的是，dialects 开始自动检测“convert_unicode”。其中一部分可以在引擎第一次连接时发出的 "SELECT 'test plain returns'" 和 "SELECT 'test_unicode_returns'" 的 SQL 中看到；方言正在测试当前的 DBAPI 是否以默认方式返回 Unicode 数据。

最终结果是，在任何情况下，无需使用“convert_unicode”标志，如果确实需要，则 SQLAlchemy 项目需要知道这些情况以及原因。目前，在所有主要数据库中都通过了数百个 Unicode 回路测试，而没有使用该标志，因此可以相当有把握地认为，除了争议性的非使用情况（例如从遗留数据库中访问错误编码的数据）之外，它们不再需要使用自定义类型更合适。

:ticket:`4393`


SQLAlchemy - PostgreSQL 改进和更改
=============================================

.. _change_4237:

为 PostgreSQL 分区表添加基本反射支持
--------------------------------------------

SQLAlchemy 可以使用在版本 1.2.6 中添加的 **postgresql_partition_by** 标志在 PostgreSQL 的 CREATE TABLE 语句中呈现“PARTITION BY”序列。但是，``'p'`` 类型不是直到现在使用的反射查询的一部分。

给定类似于以下架构：

    dv = Table(
        "data_values",
        metadata_obj,
        Column("modulus", Integer, nullable=False),
        Column("data", String(30)),
        postgresql_partition_by="range(modulus)",
    )

    sa.event.listen(
        dv,
        "after_create",
        sa.DDL(
            "CREATE TABLE data_values_4_10 PARTITION OF data_values "
            "FOR VALUES FROM (4) TO (10)"
        ),
    )

两个表名称 ``'data_values'`` 和 ``'data_values_4_10'`` 将从 :meth:`_reflection.Inspector.get_table_names` 返回，此外列也将从 ``Inspector.get_columns('data_values')`` 和 ``Inspector.get_columns('data_values_4_10')`` 返回。这也适用于使用这些表的 ``Table(..., autoload=True)`` 形式。


:ticket:`4237`


SQLAlchemy - MySQL 改进和更改
=============================================

.. _change_mysql_ping:

启用协议级 ping 进行预 ping
------------------------------------------

PyMySQL 和 mysql-connector-python 等 MySQL 方言现在使用 ``connection.ping()`` 方法进行预 ping 功能，该功能在连接上使用 Microsoft ODBC 驱动程序时可用。这比以前在连接上发出 “SELECT 1” 的方法要轻量得多。

.. _change_mysql_ondupordering:

在 ON DUPLICATE KEY UPDATE 中控制参数订购
------------------------------------------------------------

在“ON DUPLICATE KEY UPDATE”子句中可显式地排序 UPDATE 参数。方法是通过传递 2 元组的列表来实现：

    from sqlalchemy.dialects.mysql import insert

    insert_stmt = insert(my_table).values(id="some_existing_id", data="inserted value")

    on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
        [
            ("data", "some data"),
            ("updated_at", func.current_timestamp()),
        ],
    )

.. seealso::

    :ref:`mysql_insert_on_duplicate_key_update`


SQLAlchemy - SQLite 改进和更改
=============================================

.. _change_3850:

添加对 SQLite JSON 的支持
-----------------------------

添加了一个新的数据类型 :class:`_sqlite.JSON`，它代表 SQLite 上中的 JSON 成员访问函数。该实现使用 SQLite 的 ``JSON_EXTRACT`` 和 ``JSON_QUOTE`` 函数提供基本的 JSON 支持。

请注意，存储在数据库中的数据类型名称本身是名称“JSON”。这将创建一个带有“numeric”亲和力的 SQLite 数据类型，这通常应该不是问题，除了在 JSON 值仅由单个整数值组成的情况下。尽管如此，在 SQLite 的文档上下文中，采用“JSON”用于其熟悉度。

:ticket:`3850`


SQLAlchemy - SQLite 改进和更改
=============================================

.. _change_4360:

增加了新参数来影响 IDENTITY 起始和增量，同时弃用 Sequence
---------------------------------------------------------------------------------

从 SQL Server 2012 开始，SQL Server 现在支持序列，其具有真实的“CREATE SEQUENCE”语法。在 :ticket:`4235` 中，SQLAlchemy 将以与任何其他方言一样的方式使用 :class:`.Sequence` 支持这些功能。但是，当前的情况是，在 SQL Server 上 :class:`.Sequence` 已被重新用途，从而影响主键列上的 “start” 和 “increment” 参数。为了过渡到正常序列也可用的情况，使用 :class:`.Sequence` 将在整个 1.3 系列中发出弃用警告。为了影响“起始”和“增量”，请在 :class:`_schema.Column` 上使用新的 ``mssql_identity_start`` 和 ``mssql_identity_increment`` 参数：

    test = Table(
        "test",
        metadata_obj,
        Column(
            "id",
            Integer,
            primary_key=True,
            mssql_identity_start=100,
            mssql_identity_increment=10,
        ),
        Column("name", String(20)),
    )

为了在非主键列上发出 “IDENTITY”（这是极少使用但有效的 SQL Server 用例），使用 :paramref:`_schema.Column.autoincrement` 标志，在目标列上将其设置为 True，在任何整数主键列上设置为 False：

    test = Table(
        "test",
        metadata_obj,
        Column("id", Integer, primary_key=True, autoincrement=False),
        Column("number", Integer, autoincrement=True),
    )

.. seealso::

    :ref:`mssql_identity`

:ticket:`4362`

:ticket:`4235`


.. _change_4369:

cx_Oracle 连接参数现代化，弃用已过时的参数
------------------------------------------------------

一系列参数现代化的步骤被添加到了 cx_oracle 方言中，以及 URL 字符串：

* 已弃用的参数“auto_setinputsizes”、“allow_twophase”、“exclude_setinputsizes”已被删除。

* 参数“threaded”的值一直为 SQLAlchemy 方言的默认值 True，现在不再默认生成。SQLAlchemy :class:`_engine.Connection` 对象本身不被认为是线程安全的，因此不需要传递此标志。

* 已弃用在 :func:`_sa.create_engine` 中发出“线程化”本身的参数。要将“线程化”的值设置为 “True”，请将其传递到 :paramref:`_sa.create_engine.connect_args` 字典或使用查询字符串，例如 "?"threaded=true"。

* URL 查询字符串中未被特别使用的所有参数现已传递给 cx_Oracle.connect() 函数。其中的选择已被强制指定为 cx_Oracle 常量或布尔值，包括“mode”、“purity”、“events”和“threaded”。

* 与以前一样，所有 cx_Oracle“.connect()”参数都通过 :paramref:`_sa.create_engine.connect_args` 字典接受，该文档的准确性是不准确的。

:ticket:`4369`

SQLAlchemy - SQL Server 改进和更改
=============================================

.. _change_4158:

支持 pyodbc fast_executemany
-------------------------------

现在，当使用 Microsoft ODBC 驱动程序时，Pyodbc 的最新添加的“fast_executemany”模式可用于 pyodbc / mssql 方言。通过 :func:`_sa.create_engine` 传递：

    engine = create_engine(
        "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+13+for+SQL+Server",
        fast_executemany=True,
    )

.. seealso::

    :ref:`mssql_pyodbc_fastexecutemany`

:ticket:`4158`

.. _change_4362:

对于 Oracle，弃用了国家字符数据类型，重新启用了通用 Unicode（Generic Unicode）
----------------------------------------------------------------------------------

现在，除非更改具有 Unicode 兼容字符集的数据库在 SQL Server 中，否则 :class:`.Unicode` 和 :class:`.UnicodeText` 数据类型默认对应于 Oracle 上的 ``VARCHAR2`` 和 ``CLOB`` 数据类型，而不是 ``NVARCHAR2`` 和 ``NCLOB``（也称为“国家字符集”类型）。这将在例如在其如何呈现在“CREATE TABLE”语句中这样的行为中看到，在使用 :class:`.Unicode` 或 :class:`.UnicodeText` 处理绑定参数时，不会传递类型对象到 ``setinputsizes()``；cx_Oracle 本身进行字符串值处理。这个变化基于 cx_Oracle 的维护者的建议，即 Oracle 中的“国家”数据类型在很大程度上已过时且不具备性能。它们还会在某些情况下干扰，例如应用于像 “trunc()” 等函数的格式说明符时。

当不使用 Unicode 兼容字符集的数据库时，``NVARCHAR2`` 及相关类型可能需要。在这种情况下，可以通过传递 ``use_nchar_for_unicode`` 标志到 :func:`_sa.create_engine` 来重新启用旧行为。

与此同时，在 Python 2 下添加了用于实现 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了缓解在 Python 2 下 cx_Oracle 方言此前具有的此行为的性能问题，现在在 Python 2 下使用性能非常好（当构建了 C 扩展名时）的本地 Unicode 处理程序。自动 Unicode 强制执行可以通过将 ``coerce_to_unicode`` 标志设置为 False 来禁用。该标志现在默认为 True，并适用于未显式为 :class:`.Unicode` 或 Oracle 的 NVARCHAR2/NCHAR/NCLOB 数据类型之一的结果集中返回的所有字符串数据。

:ticket:`4242`

.. _change_4500:

更改了 StatementError 的格式（换行符和 %s）
=================================================================================

字符串表示形式的 "StatementError" 引入了两个更改。现在，“细节”和“SQL”部分的字符串表示形式在换行符之间分隔，并保留在原始 SQL 语句中存在的换行符。目标是提高可读性，同时仍然使原始错误消息在一行上以供记录目的。

这意味着以前看起来像这样的错误消息：

.. sourcecode:: text

    sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError) A value is
    required for bind parameter 'id' [SQL: 'select * from reviews\nwhere id = ?']
    (Background on this error at: https://sqlalche.me/e/cd3x)

现在会看起来像这样：

.. sourcecode:: text

    sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError) A value is required for bind parameter 'id'
    [SQL: select * from reviews
    where id = ?]
    (Background on this error at: https://sqlalche.me/e/cd3x)

该更改的主要影响是消费者不能再假设完整的异常消息都在单行上，但原始“错误”部分，由 DBAPI 驱动程序或 SQLAlchemy 内部生成，仍将在第一行中。

:ticket:`4500`