=============================
SQLAlchemy 1.1新功能
=============================

.. admonition:: 关于本文档

    该文档描述的是SQLAlchemy版本1.0和版本1.1之间的更改。

简介
============

本指南介绍了SQLAlchemy版本1.1中的新功能，并记录了影响从SQLAlchemy 1.0系列迁移应用程序的用户的更改。
请仔细查看潜在的不兼容行为的行为更改部分。

平台/安装器更改
============================

安装时现在需要Setuptools
--------------------------------------

多年来，SQLAlchemy的``setup.py``文件支持安装得有Setuptools，也支持不使用Setuptools进行操作；支持使用直接的Distutils的“后备”模式。
在现在不再支持没有Setuptools安装的Python环境的情况下，为了更充分地支持Setuptools的特征集，特别是支持pytest与之进行集成以及像“extras”之类的功能，``setup.py``现在完全依赖于Setuptools。

.. seealso::

      :ref:`installation` 

  :ticket:`3489`  

启用/禁用C扩展的构建仅通过环境变量完成
------------------------------------------------------------------------

默认情况下，在安装时构建C扩展是可能的。自SQLAlchemy 0.8.6 / 0.9.4起，通过将``DISABLE_SQLALCHEMY_CEXT``环境变量提供给安装过程可以禁用C扩展版本。以前使用``--without-cextensions``参数的方法已删除，因为该方法使用了setuptools的已弃用特性。

.. seealso::

      :ref:`c_extensions` 

  :ticket:`3500`  


新功能和改进 - ORM
===================================

.. _change_2677:

新会话生命周期事件
----------------------------

长期以来，  :class:`.Session`  支持事件，这些事件允许在一定程度上跟踪对象的状态更改，包括  :meth:` .SessionEvents.before_attach`  ，  :meth:`.SessionEvents.after_attach`  和  :meth:` .SessionEvents.before_flush`  .
会话文档还记录了主要的对象状态，参见   :ref:` session_object_states` 。但是，从未存在过通过这些转换精确定位对象的系统。此外，“已删除”对象的状态历史上一直模糊不清，因为这些对象在“持久”状态和“分离”状态之间具有某种功能。

为了清理这个区域并允许会话状态转换的领域完全透明，添加了一系列新事件，这些事件旨在覆盖对象可能发生的所有状态之间的任何方式，此外该“已删除”状态已在会话对象状态的范围内获得了其自己的官方状态名称。

新状态转换事件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

单位作为:期间的对象状态之间转换，例如：术语:`persistent`，期间等，现在可以拦截以特定转换进行覆盖功能。
现在，在  :class:`.SessionEvents` .Session` 对象移入、移出一个对象o, 还有所述:return  :meth:` .Session.rollback`  , 满足所有状态转换。

总共有**十个新事件**。一个新的文档部分   :ref:`session_lifecycle_events`  阐述了这些事件的概要。

新增的对象状态“deleted”，已删除对象不再是“persistent”
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :ref:`persistent` .Session` 中一直被记录作为具有有效数据库标识符的对象；但是，对于在刷新时被删除的对象，它们一直处于一片灰色地带，在其中它们不是真正的“分离”从  :class:`.Session` ,因为在回滚中它们仍然可以被恢复，但是也不是真正的“persistent”，因为它们的数据库标识符已被删除，它们不在标识映射中。

为了解决这种灰色地带，在添加的新事件中，介绍了一种新的对象状态：term:`deleted`。在新的“deleted”状态之间存在“persistent”状态和“detached”状态。使用  :meth:`.Session.delete`  标记删除的对象将保留在“persistent”状态，直到执行刷新；在那时，它将从标识映射中删除，将移至“deleted”状态，并调用  :meth:` .SessionEvents.persistent_to_deleted`  钩子。如果  :class:`.Session` .SessionEvents.deleted_to_persistent` 转换。否则，如果  :class:` .Session` .SessionEvents.deleted_to_detached` 转换。

此外，  :attr:`.InstanceState.persistent`  读取器**不再返回true**用于处于新的“deleted”状态的对象；相反，  :attr:` .InstanceState.deleted`  读取器已增强，以可靠地报告此新状态。当对象分离时，  :attr:`.InstanceState.deleted`  返回:False，并将  :attr:` .InstanceState.detached`  读取器设置为True。要确定对象是否在当前事务或先前事务中已删除，请使用  :attr:`.InstanceState.was_deleted`  读取器。

强标识映射已被弃用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

强标识映射的新系列转换事件是检测对象在其进出标识映射过程中的泄漏的来源灵感，以便可以维护“强引用”，以反映该对象在映射中的输入输出。具有此新功能，不再需要  :paramref:`.Session.weak_identity_map`  参数和相应的   :class:` .StrongIdentityMap`  对象。多年来，该选项一直作为“强引用”行为的SQLAlchemy的一部分存在，许多应用程序都写了可以采用该行为的应用程序级构造功能。长期以来，已建议将对象的强引用跟踪方法区分为   :class:`.Session`  的固有工作，而应作为应用程序级别的构造而建立应用程序所需。新 Event 模型使甚至可以复制强标识映射的确切行为。请参阅   :ref:` session_referencing_behavior` ，了解如何替换强标识映射的新配方。

  :ticket:`2677`  

.. _change_1311:

新的 init_scalar() 事件可截取 ORM 级别默认值
--------------------------------------------------------------

访问未设置的属性时，ORM 将生成一个“None”值，用于非永久对象:: 

    >>> obj = MyObj()
    >>> obj.some_value
    None

即使在对象持久之前，这里也有用例对应于核心生成的默认值 。为了适应这种用例，添加了一个新事件  :meth:`.AttributeEvents.init_scalar` 。新示例` `active_column_defaults.py``，演示了一种简单用法，可以实现如下效果::

    >>> obj = MyObj()
    >>> obj.some_value
    "my default"

  :ticket:`1311`  

.. _change_3499:

关于“不可Hash”类型的特定检查，对 ORM 行进行了排重处理
------------------------------------------------------------------

查询对象  :class:`_query.Query`  具有“排重”行为，用于返回包含  :term:` ORM`  映射实体（例如，完整的映射对象，而不是单个列值）的行。其主要目的是使实体的处理与标识映射一起自然而然地进行，包括为目的而使用连接的急切加载中通常表示的重复实体，以及用于过滤额外列的目的。

此排重依赖于行内元素的可哈希性。随着PostgreSQL的特殊类型（如  :class:`_postgresql.ARRAY` ,` _postgresql.HSTORE` 和  :class:`_postgresql.JSON` ）的引入，类型在行内的经验与对这里遇到问题的类型的浩瀚的描述经历了更多预变化。

事实上，自版本0.8以来，SQLAlchemy在数据类型上包含了一个被标记为“不可Hash”的标志，但是这个标志并没有在所有内置的类型上一致使用。如在   :ref:`change_3499_postgresql`  中所述，将为PostgreSQL的所有“结构型”类型设置一致的标志。

  :class:`.NullType`  类型上也设置了“不可Hash”标志，因为  :class:` .NullType`  用于引用任何类型的未知表达式。因为在大多数情况下  :attr:`.func`  不会实际知道给出的函数名称，所以  :attr:` .func`  应用于大多数用途的  :attr:`.func`  将很可能使行车去重失效。以下示例说明了对字符串表达式应用` `func.substr()``以及对datetime表达式应用``func.date()``将返回重复行，这是由于不存在应用显式键入的情况::

    result = (
        session.query(func.substr(A.some_thing, 0, 4), A).options(joinedload(A.bs)).all()
    )

    users = (
        session.query(
            func.date(User.date_created, "start of month").label("month"),
            User,
        )
        .options(joinedload(User.orders))
        .all()
    )

上述示例为了保持去重，应指定如下::

    result = (
        session.query(func.substr(A.some_thing, 0, 4, type_=String), A)
        .options(joinedload(A.bs))
        .all()
    )

    users = (
        session.query(
            func.date(User.date_created, "start of month", type_=DateTime).label("month"),
            User,
        )
        .options(joinedload(User.orders))
        .all()
    )

此外，所谓的“不可Hash”类型的处理方式与以前的版本略有不同；在这里，我们使用``id()``函数从这些结构中获取“哈希值”，就像将任何普通映射对象一样。这取代了以前的方法，该方法将计数器应用于对象。

  :ticket:`3499`  

.. _change_3321:

对映射类、实例传递 SQL 字面量的特定检查已添加
---------------------------------------------------------------------------

现在类型系统具有特定检查，用于在上下文中传递 SQLAlchemy“可检查”对象，这使得它们处理为字面值时产生错误。任何Python内置对象都可以作为SQL值合法传递（不是  :class:`_expression.ClauseElement` ` __clause_element__()``方法的方法，为该对象提供有效的SQL表达式。
对于不提供此功能的SQLAlchemy对象，例如映射类、映射程序和映射实例，在后面发生故障之前，会发出更详细的错误消息，而不会将其传递给DBAPI。下面是一个例子，其中字符串属性“User.name”与完整实例“User()”进行比较，而不是与字符串值进行比较::

    >>> some_user = User()
    >>> q = s.query(User).filter(User.name == some_user)
    sqlalchemy.exc.ArgumentError: Object <__main__.User object at 0x103167e90> is not legal as a SQL literal value

在进行比较“User.name == some_user”时，立即引发异常。以前，上面的比较将生成一个SQL表达式，只有在解决为DBAPI执行调用时才会失败；映射的“User”对象最终将成为一个拒绝DBAPI的绑定参数。

请注意，在上述示例中，该表达式失败，因为“User.name”是一个基于字符串的（例如列导向）属性。更改不*不*使用显示编写的类型时，将许多使用:`func`的其他示例将其应用于函数名称中返回。.func不实际知道大多数情况下给出的函数名称，因此  :attr:`.func`  将使行车去重失效。以下示例说明这种情况:将时期应用于日期时间表达式，将Substr()应用于字符串表达式;这两个示例将返回重复行，比如之前使用了明确类型::

    result = (
        session.query(func.substr(A.some_thing, 0, 4), A).options(joinedload(A.bs)).all()
    )

    users = (
        session.query(
            func.date(User.date_created, "start of month").label("month"),
            User,
        )
        .options(joinedload(User.orders))
        .all()
    )

上述示例为了保持排重，应该被指定为::

    result = (
        session.query(func.substr(A.some_thing, 0, 4, type_=String), A)
        .options(joinedload(A.bs))
        .all()
    )

    users = (
        session.query(
            func.date(User.date_created, "start of month", type_=DateTime).label("month"),
            User,
        )
        .options(joinedload(User.orders))
        .all()
    )

  :ticket:`3321`  

.. _feature_indexable:

新的可索引 ORM 扩展
---------------------------

  :ref:`indexable_toplevel`  扩展是混合属性功能的扩展，它允许构建属性，该属性引用“可索引”数据类型的特定元素，例如阵列或JSON 字段::

    class Person(Base):
        __tablename__ = "person"

        id = Column(Integer, primary_key=True)
        data = Column(JSON)

        name = index_property("data", "name")

上述代码中，``name``属性将从JSON列``data``中读取/写入``"name"`` 字段，并初始化为空字典::

    >>> person = Person(name="foobar")
    >>> person.name
    foobar

当修改属性时，该扩展也会触发更改事件，因此无需使用   :class:`~.mutable.MutableDict`  来跟踪此更改。

.. seealso::

      :ref:`indexable_toplevel` 

.. _change_3250:

新选项允许显式保留空值而非默认值
----------------------------------------------------------------

与   :ref:`change_3514`  一起添加到PostgreSQL的新JSON-NULL的支持相关，现在基本的   :class:` .TypeEngine`  类支持一种方法  :meth:`.TypeEngine.evaluates_none`  允许将   :func:` _expression.null`  赋予该属性的对象的“None”值正面地保留为NULL，而不是从插入语句中省略该列，从而使用列级默认值。这允许基于初始映射级别的对象级别技术，将   :func:`_expression.null`  赋给该属性。

.. seealso::

      :ref:`session_forcing_null` 

  :ticket:`3250`  


.. _change_3582:

继续单表继承查询方面的修复
--------------------------------------------------

从1.0 series的   :ref:`migration_3177`  开始，  :class:` _query.Query`  在针对子查询表达式（例如exists）时不应再不适当地添加“单一继承性”标准，例如：

    class Widget(Base):
        __tablename__ = "widget"
        id = Column(Integer, primary_key=True)
        type = Column(String)
        data = Column(String)
        __mapper_args__ = {"polymorphic_on": type}


    class FooWidget(Widget):
        __mapper_args__ = {"polymorphic_identity": "foo"}


    q = session.query(FooWidget).filter(FooWidget.data == "bar").exists()

    session.query(q).all()

会产生：

.. sourcecode:: sql

    SELECT EXISTS (SELECT 1
    FROM widget
    WHERE widget.data = :data_1 AND widget.type IN (:type_1)) AS anon_1

内部的 IN 子句是正确的，以限制为 FooWidget 对象，但是以前，也将在子查询外部生成一个 IN 子句。

  :ticket:`3582`  

.. _change_3680:

取消数据库 SAVEPOINT 时，改进会话状态
--------------------------------------------------------------------

MySQL中的常见情况是在事务内发生死锁时将SAVEPOINT取消。 :class:`.Session` 已修改为在某种程度上更加优雅地处理此失败模式，以便仍然可以使用外部的非SAVEPOINT事务::

    s = Session()
    s.begin_nested()

    s.add(SomeObject())

    try:
        # assume the flush fails, flush goes to rollback to the
        # savepoint and that also fails
        s.flush()
    except Exception as err:
        print("Something broke, and our SAVEPOINT vanished too")

    # this is the SAVEPOINT transaction, marked as
    # DEACTIVE so the rollback() call succeeds
    s.rollback()

    # this is the outermost transaction, remains ACTIVE
    # so rollback() or commit() can succeed
    s.rollback()

这个问题是  :ticket:`2696`  的延续，在Python 2上运行时，我们发出警告，以便在发生SAVEPOINT异常时可以看到原始错误，尽管SAVEPOINT异常优先纳入。在Python 3上，异常链接在一起，因此在报告原始错误时也会报告SAVEPOINT异常。


  :ticket:`3680`  

.. _change_3677:

修复了错误“新实例X与持久实例Y冲突”的刷新错误
----------------------------------------------------------------------------------

  :class:`.Session.rollback`  转移到  :term:` persistent` 。这些状态转换跟踪在弱引用集合中，如果从该集合进行垃圾回收，则 :class:`.Session` 将不再担心它（否则，无法处理插入许多新对象的操作事务）。
但是，如果应用程序在回滚发生之前重新装载该相同的已垃圾回收行，那么在下一个事务进入强引用之前，该对象的被删除状态将丢失；就是说，如果强引用的对象的状态被标记为已删除，那么刷新将错误地引发异常::

    from sqlalchemy import Column, create_engine
    from sqlalchemy.orm import Session
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)


    e = create_engine("sqlite://", echo=True)
    Base.metadata.create_all(e)

    s = Session(e)

    # persist an object
    s.add(A(id=1))
    s.flush()

    # rollback buffer loses reference to A

    # load it again, rollback buffer knows nothing
    # about it
    a1 = s.query(A).first()

    # roll back the transaction; all state is expired but the
    # "a1" reference remains
    s.rollback()

    # previous "a1" conflicts with the new one because we aren't
    # checking that it never got committed
    s.add(A(id=1))
    s.commit()

上面的程序将引发：

.. sourcecode:: text

    FlushError: New instance <User at 0x7f0287eca4d0> with identity key
    (<class 'test.orm.test_transaction.User'>, ('u1',)) conflicts
    with persistent instance <User at 0x7f02889c70d0>

错误在于，当引发上述异常时，工作单位正在处理原始对象，假定它是活行，而实际上该对象已过期，并在测试后消失。修复了此问题，因此在 SQL 日志中，我们会看到：

.. sourcecode:: sql

    BEGIN (implicit)

    INSERT INTO a (id) VALUES (?)
    (1,)

    SELECT a.id AS a_id FROM a LIMIT ? OFFSET ?
    (1, 0)

    ROLLBACK

    BEGIN (implicit)

    SELECT a.id AS a_id FROM a WHERE a.id = ?
    (1,)

    INSERT INTO a (id) VALUES (?)
    (1,)

    COMMIT

以上，在工作单元现在对试图报告为冲突的行进行SELECT检查后，看到该行不存在，然后正常处理该单元。只有当需要错误地引发异常时，才会产生SELECT。SELECT的成本仅在该字段将抛出异常的子查询或相似操作中有所体现。

  :ticket:`3677`  

.. _change_2349:

连同继承映射的级联删除
-------------------------------------------------------

将连接表继承映射扩展为仅在  :meth:`.Session.delete`  的结果下进行DELETE，该方法仅发布基表的DELETE，而不是子类表，允许 ON DELETE CASCADE 配置为针对配置的外键。这是使用  :paramref:` .orm.mapper.passive_deletes`  选项进行配置::

    from sqlalchemy import Column, Integer, String, ForeignKey, create_engine
    from sqlalchemy.orm import Session
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"
        id = Column("id", Integer, primary_key=True)
        type = Column(String)

        __mapper_args__ = {
            "polymorphic_on": type,
            "polymorphic_identity": "a",
            "passive_deletes": True,
        }


    class B(A):
        __tablename__ = "b"
        b_table_id = Column("b_table_id", Integer, primary_key=True)
        bid = Column("bid", Integer, ForeignKey("a.id", ondelete="CASCADE"))
        data = Column("data", String)

        __mapper_args__ = {"polymorphic_identity": "b"}

上述映射的  :paramref:`.orm.mapper.passive_deletes`  选项在基础映射器上进行了配置；它对所有具有该选项设置的映射器后代（非基映射器的映射器）生效。针对类型为` `B``的对象的DELETE不再需要未加载的``b_table_id``的主键值，也不需要为表本身发出DELETE语句::

    session.delete(some_b)
    session.commit()

将以如下方式发出SQL语句：

.. sourcecode:: sql

    DELETE FROM a WHERE a.id = %(id)s
    -- {'id': 1}
    COMMIT

正如往常一样，目标数据库必须具有启用的 ON DELETE CASCADE 的外键支持。

  :ticket:`2349`  

.. _change_3630:

必须不会再处理具有相同名称的反向引用，该反向引用适用于具体的继承子类
-------------------------------------------------------------------------------------------

以下映射一直没有问题的配置：

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        b = relationship("B", foreign_keys="B.a_id", backref="a")


    class A1(A):
        __tablename__ = "a1"
        id = Column(Integer, primary_key=True)
        b = relationship("B", foreign_keys="B.a1_id", backref="a1")
        __mapper_args__ = {"concrete": True}


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)

        a_id = Column(ForeignKey("a.id"))
        a1_id = Column(ForeignKey("a1.id"))

上述配置可以如上所述完成，即使类``A``和类``A1``都有名为``b``的关系也没有任何冲突，因为类``A1``被标记为“concrete”。

但是，如果使用另一种方式配置关系，则会引发错误：

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)


    class A1(A):
        __tablename__ = "a1"
        id = Column(Integer, primary_key=True)
        __mapper_args__ = {"concrete": True}


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)

        a_id = Column(ForeignKey("a.id"))
        a1_id = Column(ForeignKey("a1.id"))

        a = relationship("A", backref="b")
        a1 = relationship("A1", backref="b")

该修复增强了反向引用功能，因此不会发出错误，还增加了对更换属性的其他检查逻辑，以跳过替换属性的警告。

  :ticket:`3630`  

.. _change_3749:

当创建两个映射器在继承场景中，如果在两者上都放置具有相同名称的关系，则不会再次出现警告
-------------------------------------------------------------------------------------------------------------------

在继承情况下的两个映射器创建时，在其中放置具有相同名称的关系，将会标记警告；警告是“<name>”上的关系标记了映射“<name>”上同名关系的相同关系；此过程可能在flush操作期间导致依赖项问题。一个实例是如下的映射配置::


    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        bs = relationship("B")


    class ASub(A):
        __tablename__ = "a_sub"
        id = Column(Integer, ForeignKey("a.id"), primary_key=True)
        bs = relationship("B")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))

这个警告可追溯到2007年0.4系列的一个版本，基于一种已经完全重写了的单位的版本。目前，似乎没有已知问题有相同命名的关系位于基类和派生类中，因此取消了警告。但是，请注意，由于警告的存在，这种用例在实际使用中可能不普遍。此修复添加了对此用例的基本测试支持，但是可能会识别到该模式的某些新问题。

.. versionadded:: 1.1.0b3

  :ticket:`3749`  

.. _change_3653:

在混合属性和方法中传播docstring以及.info

改善的ORM特性和改进 - ORM
====================================

.. _change_3657:

映射器的`hybrid_property`将会反映原始 docstring 中的 `__doc__`值::

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)

        name = Column(String)

        @hybrid_property
        def some_name(self):
            """The name field"""
            return self.name

上述代码会被翻译成如下格式::

    >>> A.some_name.__doc__
    The name field

然而，为了实现这个，`hybrid_property`的机制必然变得更加复杂。之前，混合类访问器只是简单地传递的，也就是说，这个测试将会成功::

    >>> assert A.name is A.some_name

通过这个改变，`A.some_name`表达式返回的表达式将被包含在它自己的 `QueryableAttribute` 包装器中::

    >>> A.some_name
    <sqlalchemy.orm.attributes.hybrid_propertyProxy object at 0x7fde03888230>

已经经过了大量的测试，以确保这个包装器能够被正确地运作。 包括
`Custom Value Object <https://techspot.zzzeek.org/2011/10/21/hybrids-and-value-agnostic-types/>`_ 这样的精细方案，不过我们将会看到没有其它的功能回退会发生。

作为这个改变的一部分，`  :attr:` .hybrid_property.info 此属性的收集现在也从混合描述符本身传播，而不是从底层表达式传播。也就是说，访问 `  A.some_name.info` 现在会返回与 ` inspect(A).all_orm_descriptors['some_name'].info` 获得相同的字典。

但是这个 `.info` 字典是**独立**于混合描述符可能直接代理的映射属性的。这个处理改变于1.0版。包装器仍将代理镜像属性的其它有用属性，例如  :attr:`.QueryableAttribute.property`   和  :attr:` .QueryableAttribute.class_` 。

  :ticket:`3653`  

.. _change_3601:

Session.merge会将未决冲突解决为persistent的相同方式
---------------------------------------------------------------

现在，  :meth:`.Session.merge`   方法将跟踪给定图形内的对象身份，以在发出` INSERT`之前维护主键的唯一性。当遇到具有相同标识的重复对象时，non-primary-key属性被 **覆盖**，因为找到了对象，这基本上是不确定的。如果一个唯一标识贯穿整个图形，那么该行为与persistent对象的行为相匹配，因此该行为更加内部一致。

比如::

    u1 = User(id=7, name="x")
    u1.orders = [
        Order(description="o1", address=Address(id=1, email_address="a")),
        Order(description="o2", address=Address(id=1, email_address="b")),
        Order(description="o3", address=Address(id=1, email_address="c")),
    ]

    sess = Session()
    sess.merge(u1)

在上述代码中，我们将一个 `User` 对象与三个新的 `Order` 对象合并，每个 `Order` 对象都引用不同的 `Address` 对象，但是都具有相同的primary key。默认情况下，  :meth:`.Session.merge`  的当前行为是从标识映射中查找此 ` Address` 对象，并将其用作目标。 如果对象存在，即数据库已经有一个带有primary key 的“1”的 `Address` 的行，则可以看到 `Address` 的 `email_address` 字段将被覆盖 三次，在这种情况下分别为a，b，最后c。

但是，如果主键key为"1"的 `Address` 行不存在，  :meth:`.Session.merge`  将创建三个单独的 ` Address` 实例，然后我们将在INSERT时得到主键冲突。新的行为是，这些提议的 `Address` 对象的主键被跟踪在一个单独的字典中，以便将三个提议的 `Address` 对象的状态合并到一个要插入的 `Address` 对象上。

如果检测到合并树中存在冲突数据，则可能最好发出某种警告，但是多年来，用于persistent case的值非确定合并已经是行为，所以这个行为更匹配于pending case. 对于两种情况，都仍然可以实现警告存在冲突值的功能，但是这将增加相当大的性能开销，因为在合并过程中必须比较每个列值。

  :ticket:`3601`  

.. _change_3708:

修复了用户启动的外键操纵与many-to-one对象移动相关问题
------------------------------------------------------------------------------------

在替换引用到一个对象中的many-to-one reference with another object的机制中修复了一个bug。在属性操作期间，以前引用的对象对应到的数据库-committed外键值现在使用，而不是当前的外键值。修复的主要影响是，当进行many-to-one change时，后退事件向集合发出的精度更高，即使手动将外键属性移动到了新的值上，而只是在引用many-to-one变化之前调用了backref事件。假设 `Parent` 和 `SomeClass` 的地图，其中 `SomeClass.parent` 引用 `Parent`，而 `Parent.items` 引用 `SomeClass` 对象的collection::

    some_object = SomeClass()
    session.add(some_object)
    some_object.parent_id = some_parent.id
    some_object.parent = some_parent

上述代码创建了一个未决的对象 `some_object`，引用 `Parent` 的三个新的 `Order` 对象，每个都指向一个不同的`Address` 对象，但每个具有相同的主键。

在bug修复之前，backref将不会发出::

    # before the fix
    assert some_object not in some_parent.items

现在修复了，当我们寻找上一个值时，我们忽略已手动设置的父`parent_id`，而我们查找数据库-committed值. 在这种情况下，它为 `None`，因为对象是pending的，因此事件系统将 `some_object.parent` 进行标记为清晰更改::

    # after the fix, backref fired off for some_object.parent = some_parent
    assert some_object in some_parent.items

尽管将外键属性移动到新值是被不鼓励的操作，但仍有有限的支持此用例的支持。操纵外键以允许使用加载的应用程序通常会使用  :meth:`.Session.enable_relationship_loading`  和  :attr:` .RelationshipProperty.load_on_pending`  特性，这将导致relationship基于尚未持久化的内存中的外键值进行懒惰加载。无论是否使用这些特性，这个行为改进现在应该是明显的。

  :ticket:`3708`  

.. _change_3662:

通过多态实体提高了 `Query.correlate` 方法
------------------------------------------------------

在最近的SQLAlchemy版本中，大量poly的查询生成的SQL的形式比它的子查询中捆绑多个数据表的形式更加“平坦”。为适应这一变化，  :meth:`_query.Query.correlate`  方法现在提取这种多态可选择的单独表，并确保所有这些表都是子查询的一部分。 假设通过` with_polymorphic`从映射文档中的Person/Manager/Engineer->Company建立的映射::

    sess.query(Person.name).filter(
        sess.query(Company.name)
        .filter(Company.company_id == Person.company_id)
        .correlate(Person)
        .as_scalar()
        == "Elbonia, Inc."
    )

上述查询现在产生：

.. sourcecode:: sql

    SELECT people.name AS people_name
    FROM people
    LEFT OUTER JOIN engineers ON people.person_id = engineers.person_id
    LEFT OUTER JOIN managers ON people.person_id = managers.person_id
    WHERE (SELECT companies.name
    FROM companies
    WHERE companies.company_id = people.company_id) = ?

我们可以看到， `c`表在两次选择中都被选中，一次是在 `A.b.c -> c_alias_1` 的情况下，一次是在 `A.c -> c_alias_2` 的情况下. 同样，我们还可以看到，在identity map中得到的最终 `C` 对象是否已加载取决于映射是如何遍历的，即使不完全是随机的，而是基本上是不确定的。 查询选项仅要求在 `c_alias_1` 的上下文中加载属性`C.d`，而不是 `c_alias_2` 上下文中。因此，我们在最终得到的identity map中得到的 `C` 对象是否有 `C.d` 属性取决于映射的遍历顺序，这是不完全 - 预测态的。 加强处理量的测试是多次方向上找到mapping的entity的情况，这一修正将希望涵盖所有这种性质的场景。

  :ticket:`3662`  

.. _change_3081:

字符串化Query将查阅 Session 获取正确的方言
--------------------------------------------------------

在   :class:`_query.Query`  对象上调用 ` str（）` 将会查阅   :class:`.Session`  获取正确的“bind”，以便呈现SQL，该SQL将会传递到数据库。 特别是，这允许查询包含特定于方言的SQL构造的情况可以呈现出来，假设将   :class:` _query.Query`  关联到一个适当的   :class:`.Session` 。 以前，只有在   :class:` _schema.MetaData`  与映射相关联的情况下才会发挥此行为，并且   :class:`_query.Query`  对象被视为公共查询结构。

如果基础的   :class:`_schema.MetaData`  或   :class:` .Session`  都未与任何绑定的   :class:`_engine.Engine`  相关联，则将使用Fall-back到“默认”方言来生成SQL字符串。

.. seealso::

      :ref:`change_3631` 

  :ticket:`3081`  

.. _change_3431:

在一个行中多次存在相同的entity时，加入了joined eager loading机制
--------------------------------------------------------------------

在一个多样化的查询中，当一个属性通过joined eager loading加载时，即使该实体已经从另一条不包括该属性的“路径”中加载，属性也会被加载。这是一种难以重现的深度用例，但是通常的想法是如下所示::

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        b_id = Column(ForeignKey("b.id"))
        c_id = Column(ForeignKey("c.id"))

        b = relationship("B")
        c = relationship("C")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        c_id = Column(ForeignKey("c.id"))

        c = relationship("C")


    class C(Base):
        __tablename__ = "c"
        id = Column(Integer, primary_key=True)
        d_id = Column(ForeignKey("d.id"))
        d = relationship("D")


    c_alias_1 = aliased(C)
    c_alias_2 = aliased(C)

    q = s.query(A)
    q = q.join(A.b).join(c_alias_1, B.c).join(c_alias_1.d)
    q = q.options(
        contains_eager(A.b).contains_eager(B.c, alias=c_alias_1).contains_eager(C.d)
    )
    q = q.join(c_alias_2, A.c)
    q = q.options(contains_eager(A.c, alias=c_alias_2))

上述查询可能产生如下这种形式的SQL：

.. sourcecode:: sql

    SELECT
        d.id AS d_id,
        c_1.id AS c_1_id, c_1.d_id AS c_1_d_id,
        b.id AS b_id, b.c_id AS b_c_id,
        c_2.id AS c_2_id, c_2.d_id AS c_2_d_id,
        a.id AS a_id, a.b_id AS a_b_id, a.c_id AS a_c_id
    FROM
        a
        JOIN b ON b.id = a.b_id
        JOIN c AS c_1 ON c_1.id = b.c_id
        JOIN d ON d.id = c_1.d_id
        JOIN c AS c_2 ON c_2.id = a.c_id

我们可以看到，`c`表被选中了两次。一次是在 `A.b.c -> c_alias_1` 的情况下，另一次是在 `A.c -> c_alias_2` 的情况下。此外，我们可以看到，在一个单行中得到的`C` identity是相同的，尽管在identity映射中只添加了一个新对象。

上述的查询选项要求加载属性 `C.d`，并且只涉及到 `c_alias_1` 的情况。而不涉及别名为 `c_alias_2` 的情况。因此，我们得到的最终的 `C` 对象在identity map中是否已加载取决于映射是如何遍历的，即使不是完全随机，而是基本上是不确定的。修复包括两种“多路径到一个实体”的情况的测试，应该能涵盖所有这类型的场景。

  :ticket:`3431`  


为mutable_toplevel扩展了新的MutableList和MutableSet辅助类
---------------------------------------------------------------

加入新的辅助类   :class:`.MutableList`  和   :class:` .MutableSet`  配合现有的   :class:`.MutableDict`  辅助器。

  :ticket:`3297`  

.. _change_3512:

新的"raise" / "raise_on_sql"加载器策略
----------------------------------------------

为了帮助防止一系列对象加载后发生不需要的延迟加载，现在可以对关系属性应用新的"lazy='raise'"和"lazy='raise_on_sql'"策略以及相应的文章选择器   :func:`_orm.raiseload` ，当读取时，会导致它引发一个 ` InvalidRequestError` 。
两个变体都测试懒加载，包括只返回None或从identity map中检索的那些的延迟加载。::

    >>> from sqlalchemy.orm import raiseload
    >>> a1 = s.query(A).options(raiseload(A.some_b)).first()
    >>> a1.some_b
    Traceback (most recent call last):
    ...
    sqlalchemy.exc.InvalidRequestError: 'A.some_b' is not available due to lazy='raise'

或仅限于在发出SQL时发出延迟加载::

    >>> from sqlalchemy.orm import raiseload
    >>> a1 = s.query(A).options(raiseload(A.some_b, sql_only=True)).first()
    >>> a1.some_b
    Traceback (most recent call last):
    ...
    sqlalchemy.exc.InvalidRequestError: 'A.bs' is not available due to lazy='raise_on_sql'

  :paramref:`.expression.over.range_`   和  :paramref:` .expression.over.rows`  形式采用的都是 2 元组指示针对特定范围的负值和正值，0表示 “CURRENT ROW”，和 None 表示 “UNBOUNDED”。

  :ticket:`3512`  

.. _change_3394:

映射器的order_by参数已不再使用
-----------------------------

这个参数是第一个版本的ORM中设计的一部分，是ORM的原始设计的一部分，它在ORM中扮演着公共查询结构的角色。现在，这个角色已经被   :class:`_query.Query`  对象所取代，这里我们只需要使用  :meth:` _query.Query.order_by`  来指示结果的排序方式，无论是任何组合的SELECT语句，实体或SQL表达式。有许多情况下决定不清楚，诸如将查询组合到联合中，这些情况不被支持。


  :ticket:`3394`  

新功能和改进 - 核心
====================================

.. _change_3803:

Engines现在会为BaseException类型的情况使连接失效，并运行错误处理程序
-------------------------------------------------------------------------

Python ``BaseException`` 类在 ``Exception`` 的下面，但是这个类是系统级别异常超集的基类，例如 `KeyboardInterrupt`，`SystemExit`，尤其是 `GreenletExit` 异常，后者被事件和植物使用。因此，此异常类现在被   :class:`_engine.Connection`  的异常处理程序所拦截，并由  :meth:` _events.ConnectionEvents.handle_error`  事件处理程序包括。默认情况下，  :class:`_engine.Connection`  现在在不是 ` Exception` 的子类的情况下外部系统异常发生时，被 **无效废除**，因为我们假设已中断操作，并且连接可能处于不可用状态。MySQL驱动程序受此更改的影响最大，但该更改适用于所有DBAPIs。

请注意，在失效时，当前已使用 `  :class:`_engine.Connection`  的即时DBAPI连接会被处理，如果仍然在引发异常后继续使用  :class:` _engine.Connection` ，下一次会使用一个新的DBAPI连接来进行后续操作；但是，在操作中的任何事务状态将会丢失，并且在此重新使用之前，必须（如果适用）调用适当的 `.rollback()` 方法。

为了识别这种变化，可以演示在程序执行中处理 `KeyboardInterrupt`` 或 ``GreenletExit`` 并希望在同一事务中继续工作。这样的操作在理论上是可能的，因为不会受到像 psycopg2 这样的其他DBAPI被 ``KeyboardInterrupt`` 所影响，此时以下绕路将使禁用连接被重新用于特定异常：


        engine = create_engine("postgresql+psycopg2://")


        @event.listens_for(engine, "handle_error")
        def cancel_disconnect(ctx):
            if isinstance(ctx.original_exception, KeyboardInterrupt):
                ctx.is_disconnect = False

  :ticket:`3803`  


.. _change_2551:

支持INSERT，UPDATE，DELETE的CTE
-----------------------------------------

业界最广泛要求的功能之一是对插入，更新，删除进行通用表达式（CTE）的支持，现在已经实现。INSERT / UPDATE / DELETE可以从自身语句中作为CTE中派生的表中提取，以及为查询或更大的语句的CTE，举例如下。

为INSERT增加了CTE：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column, select, literal, exists
    >>> orders = table(
    ...     "orders",
    ...     column("region"),
    ...     column("amount"),
    ...     column("product"),
    ...     column("quantity"),
    ... )
    >>>
    >>> upsert = (
    ...     orders.update()
    ...     .where(orders.c.region == "Region1")
    ...     .values(amount=1.0, product="Product1", quantity=1)
    ...     .returning(*(orders.c._all_columns))
    ...     .cte("upsert")
    ... )
    >>>
    >>> insert = orders.insert().from_select(
    ...     orders.c.keys(),
    ...     select([literal("Region1"), literal(1.0), literal("Product1"), literal(1)]).where(
    ...         ~exists(upsert.select())
    ...     ),
    ... )
    >>>
    >>> print(insert)  # Note: formatting added for clarity
    {printsql}WITH upsert AS
    (UPDATE orders SET amount=:amount, product=:product, quantity=:quantity
     WHERE orders.region = :region_1
     RETURNING orders.region, orders.amount, orders.product, orders.quantity
    )
    INSERT INTO orders (region, amount, product, quantity)
    SELECT
        :param_1 AS anon_1, :param_2 AS anon_2,
        :param_3 AS anon_3, :param_4 AS anon_4
    WHERE NOT (
        EXISTS (
            SELECT upsert.region, upsert.amount,
                   upsert.product, upsert.quantity
            FROM upsert))

.. seealso::   :ref:`tutorial_cte` 

  :ticket:`2551`  

.. _change_3049:

支持窗口函数中的RANGE和ROWS指定
--------------------------------------------

新的参数  :paramref:`.expression.over.range_`  和  :paramref:` .expression.over.rows`  允许RANGE和ROWS表达式用于窗口函数。:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import func

    >>> print(func.row_number().over(order_by="x", range_=(-5, 10)))
    {printsql}row_number() OVER (ORDER BY x RANGE BETWEEN :param_1 PRECEDING AND :param_2 FOLLOWING){stop}

    >>> print(func.row_number().over(order_by="x", rows=(None, 0)))
    {printsql}row_number() OVER (ORDER BY x ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW){stop}

    >>> print(func.row_number().over(order_by="x", range_=(-2, None)))
    {printsql}row_number() OVER (ORDER BY x RANGE BETWEEN :param_1 PRECEDING AND UNBOUNDED FOLLOWING){stop}

  :paramref:`.expression.over.range_`   和  :paramref:` .expression.over.rows`  被指定为2元组，指示特定范围的负值和正值，对于特定范围，0表示“CURRENT ROW”，NOne表示“UNBOUNDED”。

.. seealso::

      :ref:`tutorial_window_functions` 

  :ticket:`3049`  

.. _change_2857:

支持SQL LATERAL关键字
-----------------------------------

LATERAL关键字目前已知仅由PostgreSQL 9.3及更高版本支持，但作为SQL标准的一部分，Core已经支持了此关键字。  :meth:`_expression.Select.lateral`  的实现 beyond 只是渲染LATERAL关键字，还允许表的相关联，这些表衍生自与可选择的相同FROM子句的selectable，例如外侧相关表达式（lateral correlation）:

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column, select, true
    >>> people = table("people", column("people_id"), column("age"), column("name"))
    >>> books = table("books", column("book_id"), column("owner_id"))
    >>> subq = (
    ...     select([books.c.book_id])
    ...     .where(books.c.owner_id == people.c.people_id)
    ...     .lateral("book_subq")
    ... )
    >>> print(select([people]).select_from(people.join(subq, true())))
    {printsql}SELECT people.people_id, people.age, people.name
    FROM people JOIN LATERAL (SELECT books.book_id AS book_id
    FROM books WHERE books.owner_id = people.people_id)
    AS book_subq ON true

.. seealso::

      :ref:`tutorial_lateral_correlation` 

      :class:`_expression.Lateral` 

     :meth:`_expression.Select.lateral` 


  :ticket:`2857`  

.. _change_3718:

支持TABLESAMPLE
----------------------

参考支持SQL的标准TABLESAMPLE使用

这是SQL标准的一部分。因此，Core现在支持渲染SQL表SAMPLE关键字。尽管PostgreSQL 9.5是唯一可知支持的数据库，但希望此功能将扩展到其他SQL数据库可能会增加对TABLESAMPLE的支持，包括Oracle和SQL Server, 等等。

看下面的例子::

    >>> select([mytable]).tablesample(10)

将产生如下所示的SQL语句：

.. sourcecode:: sql

    SELECT mytable.* FROM mytable TABLESAMPLE BERNOULLI(10)

请注意，`TABLESAMPLE BERNOULLI` 是 PostgreSQL 的默认选项，Hash和系统都是默认的。

  :ticket:`3718`    :meth:` _expression.FromClause.tablesample`  方法，返回一个类似于别名的 :class:`_expression.TableSample` 结构:

    from sqlalchemy import func

    selectable = people.tablesample(func.bernoulli(1), name="alias", seed=func.random())
    stmt = select([selectable.c.people_id])

假设`people`有一个`people_id`列，则上述语句将呈现为:

.. sourcecode:: sql

    SELECT alias.people_id FROM
    people AS alias TABLESAMPLE bernoulli(:bernoulli_1)
    REPEATABLE (random())

  :ticket:`3718`  

.. _change_3216:

现在不再针对复合主键列隐式启用`.autoincrement`指令
---------------------------------------------------------

SQLAlchemy始终有一个方便功能，即对于单列整数主键，启用后端数据库的“自动增量”功能；通过“自动增量”，我们指的是数据库列将包括任何DDL指令，以指示自增长整数标识符，例如在PostgreSQL上的SERIAL关键字或MySQL上的AUTO_INCREMENT，此外，方言将使用适合于该后端的执行从一：meth:`_schema.Table.insert`构造，以获取这些生成的值。

更改的是，此功能不再自动应用于复合主键；以前，表定义，如:

    Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True),
    )

只有因为它是主键列列表中的第一列，因此将对“x”列应用“autoincrement”语义。为了禁用它，必须关闭所有列上的“autoincrement”：

    # 旧方式
    Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True, autoincrement=False),
        Column("y", Integer, primary_key=True, autoincrement=False),
    )

使用新行为，除非某个列已明确标记为“autoincrement = True”，否则复合主键将不具有自动增量语义：

    #列“y”将是SERIAL/AUTO_INCREMENT/自动生成
    Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True, autoincrement=True),
    )

为了预期一些可能的向后不兼容情况，  :meth:`_schema.Table.insert`  构造将执行更彻底的检查，并检查不具有设置自增量的复合主键列的缺少主键值；给定一个表，例如：

    Table(
        "b",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True),
    )

使用没有插入值的INSERT将产生以下警告：

.. sourcecode:: text

    SAWarning: Column 'b.x' is marked as a member of the primary
    key for table 'b', but has no Python-side or server-side default
    generator indicated, nor does it indicate 'autoincrement=True',
    and no explicit value is passed.  Primary key columns may not
    store NULL. Note that as of SQLAlchemy 1.1, 'autoincrement=True'
    must be indicated explicitly for composite (e.g. multicolumn)
    primary keys if AUTO_INCREMENT/SERIAL/IDENTITY behavior is
    expected for one of the columns in the primary key. CREATE TABLE
    statements are impacted by this change as well on most backends.

对于从服务器端默认值或触发器接收主键值的列，可以使用 :class:`.FetchedValue` 指示存在值生成器：

    Table(
        "b",
        metadata,
        Column("x", Integer, primary_key=True, server_default=FetchedValue()),
        Column("y", Integer, primary_key=True, server_default=FetchedValue()),
    )

对于确实意图将空值存储在一个或多个列中的复合主键（仅在SQLite和MySQL上受支持），请使用“nullable = True”指定该列：

    Table(
        "b",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True, nullable=True),
    )

在一个相关的更改中，可以在客户端侧或服务器侧启用自动增量标志为True的列。在INSERT期间，这通常不会对列的行为产生太大影响。

.. seealso::

      :ref:`change_mysql_3216` 

  :ticket:`3216`  

.. _change_is_distinct_from:

支持IS DISTINCT FROM和IS NOT DISTINCT FROM
------------------------------------------------------


``.autoincrement``指令不再针对复合主键列隐式启用
---------------------------------------------------------

SQLAlchemy始终有一个方便功能，即对于单列整数主键，启用后端数据库的“自动增量”功能；通过“自动增量”，我们指的是数据库列将包括任何DDL指令，以指示自增长整数标识符，例如在PostgreSQL上的SERIAL关键字或MySQL上的AUTO_INCREMENT，此外，方言将使用适合于该后端的执行从一：meth:`_schema.Table.insert`构造，以获取这些生成的值。

更改的是，此功能不再自动应用于复合主键；以前，表定义，如:

    Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True),
    )

只有因为它是主键列列表中的第一列，因此将对“x”列应用“autoincrement”语义。为了禁用它，必须关闭所有列上的“autoincrement”：

    # 旧方式
    Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True, autoincrement=False),
        Column("y", Integer, primary_key=True, autoincrement=False),
    )

使用新行为，除非某个列已明确标记为“autoincrement = True”，否则复合主键将不具有自动增量语义：

    #列“y”将是SERIAL/AUTO_INCREMENT/自动生成
    Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True, autoincrement=True),
    )

为了预期一些可能的向后不兼容情况，  :meth:`_schema.Table.insert`  构造将执行更彻底的检查，并检查不具有设置自增量的复合主键列的缺少主键值；给定一个表，例如：

    Table(
        "b",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True),
    )

使用没有插入值的INSERT将产生以下警告：

.. sourcecode:: text

    SAWarning: 列'b.x'被标记为'Key'成员，并且没有指示Python侧或服务器端默认生成器，
    也没有指定'autoincrement = True'，而且没有显式传递值。主键列可能不存储NULL。
    注意，自_SQLAlchemy 1.1以来，对于复合（例如多列）主键，如果希望对主键中的某一列采用AUTO_INCREMENT /
    SERIAL / IDENTITY行为，则必须明确指示“autoincrement = True”。CREATE TABLE语句也受此更改影响。

对于从服务器端默认值或触发器接收主键值的列，可以使用 :class:`.FetchedValue` 指示存在值生成器：

    Table(
        "b",
        metadata,
        Column("x", Integer, primary_key=True, server_default=FetchedValue()),
        Column("y", Integer, primary_key=True, server_default=FetchedValue()),
    )

对于确实意图将空值存储在一个或多个列中的复合主键（仅在SQLite和MySQL上受支持），请使用“nullable = True”指定该列：

    Table(
        "b",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True, nullable=True),
    )

在一个相关的更改中，可以在客户端侧或服务器侧启用自动增量标志为True的列。在INSERT期间，这通常不会对列的行为产生太大影响。

.. seealso::

      :ref:`change_mysql_3216` 

  :ticket:`3216`  

.. _change_is_distinct_from:

支持IS DISTINCT FROM和IS NOT DISTINCT FROM
此外，新操作员  :meth:`.ColumnOperators.isnot_distinct_from`   allow the IS NOT DISTINCT
FROM sql operation: 新操作员  :meth:`.ColumnOperators.is_distinct_from`   和

.. sourcecode:: pycon+sql

    >>>打印（列（“x”）。is_distinct_from（None））
    {printsql}x IS DISTINCT FROM NULL{stop}

将提供有关NULL、True和False的处理：

.. sourcecode:: pycon+sql

    >>> print(column("x").isnot_distinct_from(False))
    {printsql}x IS NOT DISTINCT FROM false{stop}

对于SQLite，它没有这种运算符，它在SQLite上呈现为“IS”/“IS NOT”，这在SQLite中可以处理NULL，与其他后端不同：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.dialects import sqlite
    >>> print(column("x").is_distinct_from(None).compile(dialect=sqlite.dialect()))
    {printsql}x IS NOT NULL{stop}

.. _change_1957:

Core和ORM支持FULL OUTER JOIN
-------------------------------------------------- -----

新标志  :paramref:`。FromClause.outerjoin.full`  ，在Core和ORM级别上提供，
指示编译器呈现“FULL OUTER JOIN”而不是通常呈现“LEFT OUTER JOIN”：

    stmt = select([t1]).select_from(t1.outerjoin(t2, full=True))

标志也在ORM级别下起作用：

    q = session.query(MyClass).outerjoin(MyOtherClass, full=True)

  :ticket:`1957`  

.. _change_3501:

ResultSet列匹配增强;文本SQL的位置列安装
-------------------------------------------------- -----------------------------

在1.0系列中，通过:ticket: 918，对 :class:`_engine.ResultProxy` 系统进行了一系列改进，
将基于匹配名称而不是基于表/ORM元数据对游标绑定的结果列进行调整，这应包含完整信息
有关要返回的结果行。这允许显着降低Python开销，以及更准确地将ORM和Core链接起来
SQL表达式到结果行。在1.1中，此重新组织在内部进一步，并通过最近添加的  :meth:`_expression.TextClause.columns`  方法
使其可用于纯文本SQL构造。

TextAsFrom.columns()现在按位置工作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :meth:`_expression.TextClause.columns`  方法在0.9中添加，接受基于列的参数
按位置;在1.1中，当所有列被传递为位置时，这些列与最终结果集的关联
也进行了按位置执行。这里的关键优点是可以将文本SQL链接到ORM-
级别结果集而无需处理模棱两可或重复的列名称
要么必须将标记方案与ORM级标记方案匹配。所有
现在只需要在文本SQL中具有相同的列顺序
和传递给  :meth:`_expression.TextClause.columns`  的列参数::

    from sqlalchemy import text

    stmt = text(
        "SELECT users.id, addresses.id, users.id, "
        "users.name, addresses.email_address AS email "
        "FROM users JOIN addresses ON users.id=addresses.user_id "
        "WHERE users.id = 1"
    ).columns(User.id, Address.id, Address.user_id, User.name, Address.email_address)

    query = session.query(User).from_statement(stmt).options(contains_eager(User.addresses))
    result = query.all()

上述文本SQL包含三个id列，这可能令人困惑，但现在我们可以直接应用
从`User`和` `Address``类直接映射的列，即使在文本SQL中
将“Address.user_id”列链接到文本SQL中的“users.id”列，而  :obj:`_query.Query`  对象将获得正确的行
需要瞄准，包括针对贪婪加载。

这是**向后不兼容的**行为更改，对使用不同的列的方法进行列的应用
与文本语句中存在重要关注点的行为变化。希望这个
影响将因此方法一直记录并且
方法仅在0.9中添加，在任何情况下可能还没有广泛使用。在现有的
如何处理使用它的应用程序的行为更改说明请参见  :ref:`behavior_change_3501` 。

.. seealso::

    :ref:`tutorial_select_arbitrary_text` 

      :ref:`behavior_change_3501`  - 向后兼容性备注

对于Core/ORM SQL constructs，位置匹配优先于基于名称的匹配
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

此更改的另一个方面是修改匹配列的规则
对于已编译的SQL构造，更全面地依赖于“位置性”匹配
表/ORM级元数据。给出如下语句::

    ua = users.alias("ua")
    stmt = select([users.c.user_id, ua.c.user_id])

当执行上述语句时，该语句在1.0中将与其原始匹配
使用与SQL列进行定位匹配的编译构造，但是由于语句
包含“user_id”标签重复，因此“模棱两可的列”规则
仍然会涉及并防止列从行中获取。从1.1开始，“模棱两可的列”规则
不会影响从列构造到SQL列的精确匹配，这是ORM用于
获取列::

    result = conn.execute(stmt)
    row = result.first()

    # 这两个都匹配位置，因此没有错误
    user_id = row[users.c.user_id]
    ua_id = row[ua.c.user_id]

    # 这仍然会引发 exception
    user_id = row["user_id"]

更不可能出现“模棱两可列”错误消息
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

随着这个变化，错误消息“结果集中存在模糊不清的列名'<name>'！
尝试“在选择语句中使用'use_labels'选项”已被缩小；因为现在这个
消息在使用ORM或Core编译SQL构造获取结果列时几乎不会发生,
它只是在实际存在模糊不清的名称“模糊不清的列”时才会出现
在渲染的SQL语句本身中，而不是指示键或存在本构造中的名称
用于获取。

  :ticket:`3501`  

.. _change_3292:

在Core中添加对Python原生``enum``类型和兼容形式的支持
--------------------------------------------------------------

现在可以使用任何符合PEP-435的列举类型构造 :class:`.Enum` 类型。使用此模式，输入值和返回值都是实际的枚举对象，而不是字符串/整数等值::

    import enum
    from sqlalchemy import Table, MetaData, Column, Enum, create_engine


    class MyEnum(enum.Enum):
        one = 1
        two = 2
        three = 3


    t = Table("data", MetaData(), Column("value", Enum(MyEnum)))

    e = create_engine("sqlite://")
    t.create(e)

    e.execute(t.insert(), {"value": MyEnum.two})
    assert e.scalar(t.select()) is MyEnum.two


``Enum.enums``集合现在是一个列表而不是一个元组
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

作为  :class:`.Enum` .Enum.enums` 的元素集合
现在是一个列表，而不是一个元组。这是因为列表
适合长度可变的均质项序列，其中
元素的位置没有语义意义。

  :ticket:`3292`  

.. _change_2837:

日志记录和异常显示中现在会截断大的参数和行值
-------------------------------------------------- 

绑定到SQL语句的大值，以及在结果行中存在的大值，将在日志记录、异常报告以及行本身的``repr()``内部截断::

    >>> from sqlalchemy import create_engine
    >>> import random
    >>> e = create_engine("sqlite://", echo="debug")
    >>> some_value = "".join(chr(random.randint(52, 85)) for i in range(5000))
    >>> row = e.execute("select ?", [some_value]).first()
    ... # (lines are wrapped for clarity) ...
    2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine select ?
    2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine
    ('E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6GU
    LUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4=4:P
    GJ7HQ6 ... (4702 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=RJP
    HDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
    K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
    2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine Col ('?',)
    2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine
    Row (u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;
    NM6GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7
    >4=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=
    RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
    MK:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)

  :ticket:`2837`  

.. _change_3619:

Core中添加JSON支持
-------------------------------------------------- --------------------------

由于MySQL现在除了PostgreSQL JSON数据类型外还具有JSON数据类型，因此核心现在增加了一个 :class:`sqlalchemy.types.JSON` 数据类型，该类型是两者的基础。使用此类型允许访问“getitem”运算符以及“getpath”运算符，以一种适用于PostgreSQL和MySQL的方式。

新数据类型还具有一系列对NULL值的处理和表达式处理的改进。

.. seealso::

      :ref:`change_3547` 

      :class:`_types.JSON` 

      :class:`_postgresql.JSON` 

      :class:`.mysql.JSON` 

  :ticket:`3619`  

.. _change_3514:

JSON支持现在针对ORM操作插入“null”，在未出现时省略
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_types.JSON` .mysql.JSON` 现在有一个标志  :paramref:`.types.JSON.none_as_null`  ，
当设置为True时，表示Python值``None``应该转换为SQL NULL而不是JSON ]NULL值。默认情况下，此标志为False，这意味着Python值``None``应该导致JSON NULL值。

此逻辑将失败，并已进行更正，以下情况：

1.当列还包含默认值或server_default值时，映射到预计持久化JSON“null”的映射属性的正值
在有映射属性可以插入预期的“NULL”值的情况下，仍将导致触发列级默认值，
替换了"None"值::

    class MyObject(Base):
        # ...

        json_value = Column(JSON(none_as_null=False), default="some default")


    # 将插入“'null'”而不是“'some default'”,
    # 现在将插入“'null'”
    obj = MyObject(json_value=None)
    session.add(obj)
    session.commit()

2.当列*未*包含默认值或server_default值时，在具有none_as_null=False的JSON列上丢失
将仍然呈现JSON NULL值而不会后退以不插入任何值，会表现出
与所有其他数据类型不同的行为::

    class MyObject(Base):
        # ...

        some_other_value = Column(String(50))
        json_value = Column(JSON(none_as_null=False))


    # 某些情况下会为some_other_value结果为空，
    # 但是json_value结果是“'null'”。现在两者都为空
    # （json_value从INSERT中省略）
    obj = MyObject()
    session.add(obj)
    session.commit()

这是一个行为改变，对于依赖于默认缺少缺少数据的应用程序来说，这是向后不兼容的情况。这
本质上建立了一个**缺失的值与不存在的值有所区别**。有关此方案的更多详细信息，请参见  :ref:`behavior_change_3514` 。

3.当使用  :meth:`.Session.bulk_insert_mappings`  方法时，将在所有情况下忽略` `None``::

    # 将INSERT SQL空/触发默认值
    # 现在将插入“'null'”
    session.bulk_insert_mappings(MyObject, [{"json_value": None}])

  :class:`_types.JSON` .TypeEngine.should_evaluate_none` 标志，表示不应忽略` `None``;它是基于值
自  :paramref:`.types.JSON.none_as_null`  的值进行自动配置。感谢  :ticket:` 3061`  ，我们可以区分用户主动设置的值
与从未设置的值。

该功能也适用于新的基类 :class:`_types.JSON` 类型及其后代类型。

  :ticket:`3514`  

.. _change_3514_jsonnull:

添加新的JSON.NULL常量
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

为确保应用程序始终可以完全控制  :class:`_types.JSON` 、  :class:` _postgresql.JSON` 、 :class:`.mysql.JSON` 或
  :class:`_postgresql.JSONB` ` "null"``值，已添加常量  :attr:`.types.JSON.NULL`  ，
它与  :func:`.null` ` "null"``且无论设置：paramref:`.types.JSON.none_as_null`
如何，都可以在两者之间进行。

    from sqlalchemy import null
    from sqlalchemy.dialects.postgresql import JSON

    obj1 = MyObject(json_value=null())  # 将始终插入SQL NULL
    obj2 = MyObject(json_value=JSON.NULL)  # 将始终插入JSON字符串“null”

    session.add_all([obj1, obj2])
    session.commit()

该功能也适用于新的基类 :class:`_types.JSON` 类型及其后代类型。

  :ticket:`3514`  

.. _change_3516:

Core中添加数组支持;添加新的ANY和ALL操作符
-------------------------------------------------- ------------------------------

随随着对PostgreSQL  :class:`_postgresql.ARRAY` 。

数组是SQL标准的一部分，与此类似的还有几个面向数组的函数，例如 “array_agg ()”和“unnest ()”。为了支持这些构造不仅适用于PostgreSQL而且也适用于未来的其他具有数组功能的引擎，例如DB2，因此SQL表达式的大部分数组逻辑现在都在Core中。  :class:`_types.ARRAY` 类型仍然仅在 PostgreSQL上工作，但是它可以直接使用，支持特殊的数组用例，例如索引访问，以及对任何和全部的支持：

    mytable = Table("mytable", metadata, Column("data", ARRAY(Integer, dimensions=2)))

    expr = mytable.c.data[5][6]

    expr = mytable.c.data[5].any(12)

为了支持ANY和ALL，   :class:`_types.ARRAY` .types.ARRAY.Comparator.any` 和 :meth:`.types.ARRAY.Comparator.all` 方法，但还将这些操作导出到新的独立操作函数  :func:`_expression.any_` 和  :func:`_expression.all_` 中。这两个函数以更传统的SQL方式工作，允许一个右侧表达式形式，例如：

    from sqlalchemy import any_, all_

    select([mytable]).where(12 == any_(mytable.c.data[5]))

对于PostgreSQL特定的运算符“ contains”，“ contained_by”和“ overlaps”，应继续直接使用  :class:`_postgresql.ARRAY` 类型，该类型提供了  :class:`_types.ARRAY` 类型的所有功能。

现在，  :func:`_expression.any_` 和  :func:`_expression.all_` 操作符是在Core级别开放的，但是它们在后端数据库中的解释是有限制的。在PostgreSQL后端上，这两个运算符仅接受数组值。然而，在MySQL后端上，它们仅接受子查询值。在MySQL上，可以使用以下表达式：

    from sqlalchemy import any_, all_

    subq = select([mytable.c.value])
    select([mytable]).where(12 > any_(subq))

 $& 

.. _change_3132:

新功能函数，"WITHIN GROUP"，array_agg和set集合函数
------------------------------------------------------

有了新的   :class:`_types.ARRAY` ` array_agg ()`` SQL函数，这可以使用   :class:`_functions.array_agg`  获得：

    from sqlalchemy import func

    stmt = select([func.array_agg(table.c.value)])

通过引入   :class:`_postgresql.aggregate_order_by` ，还添加了一个面向PostgreSQL的聚合ORDER BY元素：

    from sqlalchemy.dialects.postgresql import aggregate_order_by

    expr = func.array_agg(aggregate_order_by(table.c.a, table.c.b.desc()))
    stmt = select([expr])

生成结果为：

.. sourcecode:: sql

    SELECT array_agg(table1.a ORDER BY table1.b DESC) AS array_agg_1 FROM table1

PG方言本身还提供了   :func:`_postgresql.array_agg`  包装器，以确保   :class:` _postgresql.ARRAY`  类型：

    from sqlalchemy.dialects.postgresql import array_agg

    stmt = select([array_agg(table.c.value).contains("foo")])

此外，像 ``percentile_cont()``, ``percentile_disc()``,
``rank()``, ``dense_rank()``和其他需要此功能的函数
“WITHIN GROUP（ORDER BY <expr>）”现已通过  :meth:`.FunctionElement.within_group`  修饰符提供：

    from sqlalchemy import func

    stmt = select(
        [
            department.c.id,
            func.percentile_cont(0.5).within_group(department.c.salary.desc()),
        ]
    )

上述语句将生成类似的SQL：

.. sourcecode:: sql

  SELECT department.id, percentile_cont(0.5)
  WITHIN GROUP (ORDER BY department.salary DESC)

这些函数的正确返回类型现在提供了占位符，
包括   :class:`_sql.expression.percentile_cont` 、   :class:` _sql.expression.percentile_disc` 、
  :class:`_sql.expression.rank` 、   :class:` _sql.expression.dense_rank` 、
   :class:`_sql.expression.mode` 、   :class:` _sql.expression.percent_rank` 
和   :class:`_sql.expression.cume_dist` 。

 $&   $& 

.. _change_2919:

TypeDecorator现在自动使用Enum，Boolean，“schema”类型
---------------------------------------------------------

  :class:`.SchemaType` .Enum`  和  :class:` .Boolean`的类型，除了与相应的数据库类型相对应之外，
还会生成CHECK约束或在PostgreSQL ENUM的情况下创建新的CREATE TYPE语句，现在它们将自动与   :class:`.TypeDecorator` .TypeDecorator` 必须如下所示：

    # old way
    class MyEnum(TypeDecorator, SchemaType):
        impl = postgresql.ENUM("one", "two", "three", name="myenum")

        def _set_table(self, table):
            self.impl._set_table(table)

现在，  :class:`.TypeDecorator` 传播了这些附加事件，因此可以像任何其他类型一样完成::

    # new way
    class MyEnum(TypeDecorator):
        impl = postgresql.ENUM("one", "two", "three", name="myenum")

 $& 

.. _change_2685:

Table对象的多租户模式翻译
--------------------------

为了支持在许多模式中使用相同的一组   :class:`_schema.Table`  对象的应用程序（例如，每个用户的架构），现在添加了一个新的执行选项  :paramref:` .Connection.execution_options.schema_translate_map` 。使用此映射，可以在每个连接基础上使一组   :class:`_schema.Table`  对象引用任何一组模式，而不是它们分配给的  :paramref:` _schema.Table.schema` 。DDL和SQL生成以及ORM都可以使用翻译。

例如，如果“User”类被分配了模式“per_user”：

    class User(Base):
        __tablename__ = "user"
        id = Column(Integer, primary_key=True)

        __table_args__ = {"schema": "per_user"}

每次请求时，   :class:`.Session`  可以设置为引用不同的模式：

    session = Session()
    session.connection(
        execution_options={"schema_translate_map": {"per_user": "account_one"}}
    )

    # will query from the ``account_one.user`` table
    session.query(User).get(5)

.. seealso::

      :ref:`schema_translating` 

 $& 

.. _change_3631:

Core SQL构造的“友好”字符串化没有发现方言现在
------------------------------------------

在Core SQL构造上调用 ``str()`` 现在会在比以前更多的情况下生成字符串，支持各种默认SQL中通常不存在的 SQL 构造，例如 RETURNING，数组索引和非标准数据类型：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column
    t>>> t = table('x', column('a'), column('b'))
    >>> print(t.insert().returning(t.c.a, t.c.b))
    {printsql}INSERT INTO x (a, b) VALUES (:a, :b) RETURNING x.a, x.b

现在， ``str()`` 函数调用单独的方言/编译器，旨在进行纯字符串打印，而不设置特定的方言，
因此在出现更多“只有显示给我一个字符串！”的情况时，这些可添加到此语言/编译器中，而不会影响实际方言上的行为。

.. seealso::

      :ref:`change_3081` 

 $& 

.. _change_3531:

type_coerce函数现在是一个持久的SQL元素
-------------------------------------------------------

以前，函数   :func:`_expression.type_coerce`  会返回一个   :class:` .BindParameter`  或   :class:`.Label`  类型的对象，这取决于输入。这会产生这样的影响：如果使用表达式转换，例如将一个元素从   :class:` _schema.Column`  转换为   :class:`.BindParameter` ，这对于ORM级别的懒惰加载是至关重要的，那么类型转换信息将不会被使用，因为它已经丢失了。

为了改进此行为，该函数现在返回一个持久的   :class:`.TypeCoerce`  容器，该容器包围给定表达式，但是该表达式本身不受影响; 此结构由SQL编译器明确评估。这允许内部表达式的类型强制转换保持不变，无论如何修改语句，包括如果包含元素替换为不同元素的情况，这在ORM的懒惰加载功能中很常见。

用于说明效果的测试用例使用了异构的 primaryjoin 条件，以及自定义类型和懒惰加载。在给定应用程序中，假设数据库的字符串“id”列等于另一个表中的整数“id”列：

    class Person(Base):
        __tablename__ = "person"
        id = Column(StringAsInt, primary_key=True)

        pets = relationship(
            "Pets",
            primaryjoin=(
                "foreign(Pets.person_id)" "==cast(type_coerce(Person.id, Integer), Integer)"
            ),
        )


    class Pets(Base):
        __tablename__ = "pets"
        id = Column("id", Integer, primary_key=True)
        person_id = Column("person_id", Integer)

在上述代码中，在  :paramref:`_orm.relationship.primaryjoin`  表达式中，我们使用   :func:` .type_coerce`  来处理以整数形式传递的绑定参数，因为我们已经知道这些将来自我们将该值作为整数在Python中维护的“StringAsInt”类型。然后，我们正在使用   :func:`.cast` ，以便作为SQL表达式，VARCHAR“id”列将被CAST为整数，用于正常的非转换连接，例如  :meth:` _query.Query.join`  或   :func:`_orm.joinedload`  的连接。例如，加载 ` `.pets`` 的 joinedload：

.. sourcecode:: sql

    SELECT person.id AS person_id, pets_1.id AS pets_1_id,
           pets_1.person_id AS pets_1_person_id
    FROM person
    LEFT OUTER JOIN pets AS pets_1
    ON pets_1.person_id = CAST(person.id AS INTEGER)

在连接中不使用CAST，在PostgreSQL等强类型数据库上将会无法隐式地比较整数并失败。

“ .pets” 的lazyload情况仅依赖于在加载时用绑定参数替换 “Person.id” 列，该绑定参数接收Python加载的值。在需要替换列使用语句时，类型强制转换信息将丢失的情况下，此替换是特定的地方，现在将使用   :func:`.type_coerce`  函数维护包装，即使在列为绑定参数替换为其他列的情况下，查询也会像上面所述那样进行。

 $& 

主要行为更改 - ORM
===========================

.. _behavior_change_3514:

如果没有提供值且未建立默认值，则JSON列将不插入JSON NULL
----------------------------------------------------------------

如   :ref:`change_3514`  中所述，  :class:` _types.JSON`  不会再在值完全丢失时呈现JSON“null”值。为了防止SQL NULL，应该设置默认值。给定以下映射：

    class MyObject(Base):
        # ...

        json_value = Column(JSON(none_as_null=False), nullable=False)

以下提交操作将失败并引发完整性错误：

    obj = MyObject()  # 注意没有 json_value
    session.add(obj)
    session.commit()  # 将引发完整性错误

如果列的默认值应该是JSON NULL，则应设置该值：

    class MyObject(Base):
        # ...

        json_value = Column(JSON(none_as_null=False), nullable=False, default=JSON.NULL)

否则，请确保对象上存在该值：

    obj = MyObject(json_value=None)
    session.add(obj)
    session.commit()  # 将插入JSON NULL

请注意，将  :paramref:`.types.JSON.none_as_null`  标志设置为 ` `None`` 与完全省略它相同；该标志对于传递给  :paramref:`_schema.Column.default`  或  :paramref:` _schema.Column.server_default`  的 ``None`` 值不产生影响。

.. seealso::

      :ref:`change_3514` 

.. _change_3641:

DISTINCT + ORDER BY的相同命名的@validates装饰符现在会引发异常
-------------------------------------------------------------------------------

  :class:`_orm.validates`  装饰器只应为特定属性名称的类创建一次。创建多个现在会引发错误，虽然先前它会默默选择最后定义的验证器：

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)

        data = Column(String)

        @validates("data")
        def _validate_data_one(self):
            assert "x" in data

        @validates("data")
        def _validate_data_two(self):
            assert "y" in data


    configure_mappers()

将引发以下错误：

.. sourcecode:: text

    sqlalchemy.exc.InvalidRequestError: A validation function for mapped attribute 'data'
    on mapper Mapper|A|a already exists.

 $& 

主要行为更改 - Core
=============================

.. _behavior_change_3501:

当按位置传递列时，TextClause.columns()将按位置而不是按名称进行匹配
--------------------------------------------------------------------------

  :meth:`_expression.TextClause.columns`  方法的新行为，它本身实际上是在0.9系列中添加的，现在，在位置上传递列而不添加任何其他关键字参数时，它们与最终结果集的列位置相关联，而不是名称。由于该方法一直都在文档中说明列以与文本SQL中列的相同顺序传递，因此对该方法的内部不再进行检查，并且应用程序使用此方法将在将   :class:` _schema.Column`  对象按位置传递给该方法时确保这些   :class:`_schema.Column`  对象的位置与文本SQL中声明这些列的位置相同。

例如，如下代码：

    stmt = text("SELECT id, name, description FROM table")

    # no longer matches by name
    stmt = stmt.columns(my_table.c.name, my_table.c.description, my_table.c.id)

现在不再像预期的那样工作了；现在给出的列的顺序已经具有重要意义：

    stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)

可能更有可能的是，类似如下的语句：

    stmt = text("SELECT * FROM table")
    stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)

现在稍微有些冒险，因为“*”规范通常按表中的存在顺序提供列。如果表结构因模式更改而更改，则此排序可能不再相同。
因此，使用  :meth:`_expression.TextClause.columns`  方法建议在文本SQL中明确列出所需的列，尽管在文本SQL中名称本身不再需要担心。

.. seealso::

      :ref:`change_3501` 

.. _change_3809:

字符串server_default现在是文字引用
------------------------------------------

现在，将字符串 server_default 作为普通Python字符串传递到  :paramref:`_schema.Column.server_default`  的服务器默认值现在将通过文字引用系统传递：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.schema import MetaData, Table, Column, CreateTable
    >>> from sqlalchemy.types import String
    >>> t = Table("t", MetaData(), Column("x", String(), server_default="hi ' there"))
    >>> print(CreateTable(t))
    {printsql}CREATE TABLE t (
        x VARCHAR DEFAULT 'hi '' there'
    )

先前的引号会直接呈现。对于具有此类用例的应用程序，此更改可能是不向后兼容的。

 $& 

.. _change_2528:

带有LIMIT / OFFSET / ORDER BY的SELECT的UNION或类似操作现在在嵌入式选​​择中加括号
-------------------------------------------------------------------------------

“ UNION”查询由多个 包含行限制或排序行为（包括LIMIT，OFFSET和/或 ORDER BY） 的 SELECT语句组成，例如：

.. sourcecode:: sql

    (SELECT x FROM table1 ORDER BY y LIMIT 1) UNION
    (SELECT x FROM table2 ORDER BY y LIMIT 2)

此查询需要在每个子选择中使用括号以便正确组合子结果。
在 SQLAlchemy Core 中生成上述查询的方法类似于此：

    stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1)
    stmt2 = select([table1.c.x]).order_by(table2.c.y).limit(2)

    stmt = union(stmt1, stmt2)

先前，上述结构不会产生内部SELECT语句的括号，从而产生了一个在所有后端上均失败的查询。

以上格式将在SQLite上 **继续失败**；此外，在Oracle上， **仅按顺序排列** 而没有 LIMIT / SELECT 的格式将 **继续失败**。
这不是向后不兼容的更改，因为查询在没有括号的情况下失败；修复后，查询顶多在所有其他数据库上正常工作。

在所有情况下，在调用与嵌套自描述查询一起使用很少用到的情况下，为了产生一个有限的SELECT语句的UNION语句存在工作风险，因此建议使用子查询：：

    stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1).alias().select()
    stmt2 = select([table2.c.x]).order_by(table2.c.y).limit(2).alias().select()

    stmt = union(stmt1, stmt2)

这种解决方法在所有 SQLAlchemy 版本中都可以使用。在 ORM 中，看起来像：

    stmt1 = session.query(Model1).order_by(Model1.y).limit(1).subquery().select()
    stmt2 = session.query(Model2).order_by(Model2.y).limit(1).subquery().select()

    stmt = session.query(Model1).from_statement(stmt1.union(stmt2))

此行为具有与 SQLAlchemy 0.9 中   :ref:`feature_joins_09`  中介绍的 "join rewriting" 行为的许多相似之处；
但是，在这种情况下，我们选择不添加新的重写行为以适应此用例的 SQLite，现有的重写行为已经非常复杂了。
使用此功能的 UNION 与带有括号的 SELECT 语句的情况比   :ref:`feature_joins_09`  中的案例要少得多。

 $& 


Dialect Improvements and Changes - PostgreSQL
=============================================

.. _change_3529:

支持INSERT.. ON CONFLICT（DO UPDATE | DO NOTHING）
--------------------------------------------------------

自PostgreSQL 9.5起新增加的“ON CONFLICT”子句在   :func:`sqlalchemy.dialects.postgresql.dml.insert`  中现在已经通过   :class:` _expression.Insert`  子类支持。此   :class:`_expression.Insert`  子类添加了两个新方法  :meth:` _expression.Insert.on_conflict_do_update`  和  :meth:`_expression.Insert.on_conflict_do_nothing` ，这两个方法实现了这个领域支持的所有语法：

    from sqlalchemy.dialects.postgresql import insert

    insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

    do_update_stmt = insert_stmt.on_conflict_do_update(
        index_elements=[my_table.c.id], set_=dict(data="some data to update")
    )

    conn.execute(do_update_stmt)

以上将呈现：

.. sourcecode:: sql

    INSERT INTO my_table (id, data)
    VALUES (:id, :data)
    ON CONFLICT id DO UPDATE SET data=:data_2

.. seealso::

      :ref:`postgresql_insert_on_conflict` 

 $& 

.. _change_3499_postgresql:

ARRAY和JSON类型现在正确指定为“不可哈希的”
-----------------------------------------------------------------

如   :ref:`change_3499`  中所述，ORM依靠能够为列值生成散列函数，例如当查询的所选实体混合了完整的ORM实体和列表达式时。 ` `hashable=False`` 现在已正确设置在PG的所有“数据结构”类型上，包括   :class:`_postgresql.ARRAY`  和   :class:` _postgresql.JSON` 。
  :class:`_postgresql.JSONB`  和   :class:` .HSTORE`  类型已包含此标志。对于   :class:`_postgresql.ARRAY` ，这取决于  :paramref:` .postgresql.ARRAY.as_tuple`  标志的条件，但是现在应该不再需要设置此标志才能在组合的ORM行中具有数组值。

.. seealso::

      :ref:`change_3499` 

      :ref:`change_3503` 

 $& 

.. _change_3503:

从ARRAY，JSON，HSTORE的索引访问现在正确建立了正确的SQL类型
--------------------------------------------------------------------------

对于   :class:`_postgresql.ARRAY` 、   :class:` _postgresql.JSON`  和   :class:`.HSTORE` ，返回的表示为索引访问的表达式的SQL类型应在所有情况下正确。

其中包括：

* 分配给   :class:`_postgresql.ARRAY`  索引访问的SQL类型会考虑配置的维数数量。具有三个维度的   :class:` _postgresql.ARRAY`  将返回类型为   :class:`_postgresql.ARRAY`  的SQL表达式的表达式

减小一维数组大小
---------------------------------

对于具有类型``ARRAY(Integer, dimensions=3)``的列，现在可以执行以下表达式：

    int_expr = col[5][6][7]  # 返回整数表达式对象

以前，访问``col [5]``的索引访问将返回类型为 :class:`.Integer` 
的表达式，我们无法再访问剩余维度，除非使用  :func:`.Cast` 
或  :func:`.type_coerce` 。

  :class:`_postgresql.JSON` 
--------------------------------------------------------

现在， :class:`_postgresql.JSON` 和  :class:`_postgresql.JSONB` 类型反映了PostgreSQL本身
对于索引访问所做的操作。这意味着所有针对  :class:`_postgresql.JSON` 或
 :class:`_postgresql.JSONB` 类型的索引访问都将返回一个总是  :class:`_postgresql.JSON` 或 
  :class:`_postgresql.JSONB` ~.postgresql.JSON.Comparator.astext` 
修饰符，否则不会将JSON结构的索引访问归类为其他原生类型，比如字符串、列表、数字等等。
这意味着，和   :class:`_postgresql.ARRAY`  类型一样，现在可以直接生成具有多层索引访问的JSON表达式：

    json_expr = json_col["key1"]["attr1"][5]

对于 :class:`.HSTORE` 的索引访问，返回的“文本”类型以及
与  :class:`_postgresql.JSON` 和  :class:`_postgresql.JSONB` 的索引
访问一起使用  :attr:`~.postgresql.JSON.Comparator.astext`  修饰符的“文本”类型
是可配置的；默认值为   :class:`~_expression.TextClause` ，但可以使用  :paramref:` .postgresql.JSON.astext_type`  或
 :paramref:`.postgresql.HSTORE.text_type` 参数设置为用户定义的类型。

.. seealso::

      :ref:`change_3503_cast` 

  :ticket:`3499`  
  :ticket:`3487`  

The JSON cast() 操作现在要求显式调用 `.astext`
-------------------------------------------------

作为   :ref:`change_3503`   运算符的操作
  :class:`_postgresql.JSON` .postgresql.JSON.Comparator.astext` 
修饰符；PostgreSQL的JSON/JSONB类型支持彼此之间的CAST操作，而不需要“astext”方面。

这意味着在大多数情况下，做了这样的应用：

    expr = json_col ["somekey"].cast(Integer)

现在需要更改为：

    expr = json_col ["somekey"].astext.cast(Integer)

.. _change_2729:

ARRAY与ENUM将为ENUM生成 CREATE TYPE
-----------------------------------------------------

类似如下的表定义现在将如预期一样生成 CREATE TYPE：

    enum = Enum(
        "manager",
        "place_admin",
        "carwash_admin",
        "parking_admin",
        "service_admin",
        "tire_admin",
        "mechanic",
        "carwasher",
        "tire_mechanic",
        name="work_place_roles",
    )

    class WorkPlacement(Base):
        __tablename__ = "work_placement"
        id = Column(Integer, primary_key=True)
        roles = Column(ARRAY(enum))
    e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
    Base.metadata.create_all(e)

输出如下：

.. sourcecode:: sql

    CREATE TYPE work_place_roles AS ENUM (
        'manager', 'place_admin', 'carwash_admin', 'parking_admin',
        'service_admin', 'tire_admin', 'mechanic', 'carwasher',
        'tire_mechanic')

    CREATE TABLE work_placement (
        id SERIAL NOT NULL,
        roles work_place_roles[],
        PRIMARY KEY (id)
    )


  :ticket:`2729`  

```check```约束现在被反映
-----------------------------------------

PostgreSQL dialect 现在支持  :meth:`_reflection.Inspector.get_check_constraints`  中检查约束的反射，以及
  :class:`_schema.Table`  值得一提的是，反映了  :attr:` _schema.Table.constraints`  集合内的 CHECK 约束。

可以分别检查“普通”和“材质化”视图
-----------------------------------------

新的参数  :paramref:`.PGInspector.get_view_names.include`  允许指定应返回哪些视图的子类型：

    from sqlalchemy import insp

    insp = inspect(engine)

    plain_views = insp.get_view_names(include="plain")
    all_views = insp.get_view_names(include=("plain", "materialized"))

  :ticket:`3588`  

向Index添加tablespace选项
-----------------------------------

  :class:`.Index` ` postgresql_tablespace``，以指定TABLESPACE，
就像 :class:`_schema.Table` 对象一样。

.. seealso::

      :ref:`postgresql_index_storage` 

  :ticket:`3720`  

对于FOR UPDATE SKIP锁定/ FOR NO KEY UPDATE / FOR KEY SHARE的支持
---------------------------------------------------------------------

Core 和 ORM 中的  :paramref:`.GenerativeSelect.with_for_update.skip_locked` 
和  :paramref:`.GenerativeSelect.with_for_update.key_share`  新参数在 PostgreSQL 后端查询时应用了
“SELECT...FOR UPDATE”或“SELECT...FOR SHARE”的修改：

* SELECT FOR NO KEY UPDATE::

    stmt = select([table]).with_for_update(key_share=True)

* SELECT FOR UPDATE SKIP LOCKED::

    stmt = select([table]).with_for_update(skip_locked=True)

* SELECT FOR KEY SHARE::

    stmt = select([table]).with_for_update(read=True, key_share=True)

Dialect Improvements and Changes - MySQL
========================================

.. _change_3547:

MySQL JSON 支持
----------------------

MySQL 5.7 新添加了 JSON 类型，MySQL 方言增加了新的类型  :class:`.mysql.JSON` ，支持该类型。此类型同时提供JSON的持久性和基于“JSON_EXTRACT”函数的基本索引访问。可以通过使用通用于MySQL和PostgreSQL的 :class:` _types.JSON`数据类型来实现跨MySQL和PostgreSQL的可索引JSON列。

.. seealso::

      :ref:`change_3619` 

  :ticket:`3547`  

.. _change_3332:

新增对 AUTOCOMMIT“隔离级别”的支持
-------------------------------------------

MySQL 方言现在接受值“AUTOCOMMIT”作为 :paramref:`_sa.create_engine.isolation_level` 和
  :paramref:`.Connection.execution_options.isolation_level`   参数：

    connection = engine.connect()
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

隔离级别使用大多数 MySQL DBAPI 提供的各种“autocommit”的属性。

  :ticket:`3332`  

.. _change_mysql_3216:

不再为具有AUTO_INCREMENT的组合主键生成隐式键
---------------------------------------------------------

MySQL 方言的行为是，如果InnoDB表上的复合主键通过一个或多个自增列进行配置，而自增列并不是第一列，如：

    t = Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True, autoincrement=False),
        Column("y", Integer, primary_key=True, autoincrement=True),
        mysql_engine="InnoDB",
    )

上述 ddl 将生成：

.. sourcecode:: sql

    CREATE TABLE some_table (
        x INTEGER NOT NULL,
        y INTEGER NOT NULL AUTO_INCREMENT,
        PRIMARY KEY (x, y),
        KEY idx_autoinc_y (y)
    )ENGINE=InnoDB

注意上面的“KEY”和自动生成的名称；这是许多年前改编该方言的响应之后的一项更改，该方言已经发出多年的警告，现在应该调用“sqlalchemy.dialects.postgresql”。

现在，这个解决方案已被删除并替换为更好的系统，只需要在主键中明确声明自动增量列 * 在主键列中* ：

.. sourcecode:: sql

    CREATE TABLE some_table (
        x INTEGER NOT NULL,
        y INTEGER NOT NULL AUTO_INCREMENT,
        PRIMARY KEY (y, x)
    )ENGINE=InnoDB

为了保持对主键列顺序的明确控制，您可以显式使用  :class:`.PrimaryKeyConstraint` 构造（1.1.0b2）（以及一个必需的自动增量列的KEY，如需要的MySQL），例如：

    t = Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True, autoincrement=True),
        PrimaryKeyConstraint("x", "y"),
        UniqueConstraint("y"),
        mysql_engine="InnoDB",
    )

随着   :ref:`change_3216`  的改变，具有或不具有自动增量的组合主键现在更容易指定；
  :paramref:`_schema.Column.autoincrement`  现在默认为值 ` `"auto"``，不再需要 ``autoincrement=False`` 指令：

    t = Table(
        "some_table",
        metadata,
        Column("x", Integer, primary_key=True),
        Column("y", Integer, primary_key=True, autoincrement=True),
        mysql_engine="InnoDB",
    )

Dialect Improvements and Changes - SQLite
=========================================

.. _change_3634:

SQLite 3.7.16 版本解决了的正确处理右嵌套链接
-----------------------------------------------------

在版本0.9中，通过   :ref:`feature_joins_09`  引入了新特性，它花费了很多的功夫来支持重写在SQLite上的联接，以始终使用子查询来实现“right-nested-join”效果，因为SQLite多年来没有支持这个语法的方法。有趣的是，标记在这个迁移注释中的SQLite版本，即3.7.15.2版本，实际上是最后一个实际上拥有此限制的SQLite版本！下一个发布版本是3.7.16，而恰恰是支持了右嵌套链接。在1.1中，执行了查找这个更改发生的特定SQLite版本和源提交的工作（SQLite的更改日志以加密短语“增强查询优化器以利用传递式连接限制”而没有链接任何问题编号，更改编号或更多解释），并且这种解决方法现在已经被 SQLite 的版本检测到，并针对版本3.7.16或更高版本关闭了这种行为。

  :ticket:`3634`  

.. _change_3633:

取消 SQLite 版本 3.10.0 解决了集合的列名问题的“点”符号列名的解决方法
-------------------------------------------------------------------------

SQLite 方言长期以来都使用了一个解决方法来解决这个问题，即数据库驱动程序没有为某些 SQL 结果集正确报告列名，特别是在使用 UNION 时；
  :ref:`sqlite_dotted_column_names`  详细描述了这个解决方法，其中要求 SQLAlchemy 假定任何具有一个字符点的列名实际上是通过此错误行为传递的 ` `tablename.columnname`` 组合，并通过``sqlite_raw_colnames `` 执行选项将其关闭。

截至 SQLite 3.10.0 版本，修复了 UNION 和其他查询中的错误，就像在   :ref:`change_3634`  中描述的那样。就像更改描述中所述的那样，SQLite 的更改日志仅以加密短语“已将colUsed字段添加到sqlite3_index_info中，供sqlite3_module.xBestIndex方法使用”命名，没有链接到任何问题编号、更改编号或更多的解释。

总的来说，从1.0系列开始，SQLAlchemy   :class:`_engine.ResultProxy`  不再依赖于结果集中的列名，以提供 Core 和 ORM 的 SQL 构造结果。

  :ticket:`3633`  

.. _change_sqlite_schemas:

对远程模式的支持做出了改进
------------------------------------------------

SQLite方言通过实现  :meth:`_reflection.Inspector.get_schema_names`  ，并显著改进了被创建并反射的来自远程模式的表和索引，远程模式是通过 ` `ATTACH`` 语句分配名称的数据库。以前，数据库在一个分隔名表上创建索引无法正确工作，并且
  :meth:`_reflection.Inspector.get_foreign_keys`   方法现在会在结果中指示给定的模式。不支持跨模式外键。

.. _change_3629:

反射主键约束名称
----------------------------------------------------------

SQLite 后端现在利用 SQLite 的 “sqlite_master” 视图，以从原始 DDL 提取表的主键约束名称，就像在近期 SQLAlchemy 版本中为外键约束所实现的那样。

  :ticket:`3629`  

现在不再强制字符串/变长类型在反映时显式表示为“max”
---------------------------------------------------------------------------

反映   :class:`.String` 、  :class:` _expression.TextClause` （等含长度的类型）的类型时，一个“未设长度”的类型在 SQL Server 上会将“长度”参数复制为值 ``"max"``：

    >>> from sqlalchemy import create_engine, inspect
    >>> engine = create_engine("mssql+pyodbc://scott:tiger@ms_2008", echo=True)
    >>> engine.execute("create table s (x varchar(max), y varbinary(max))")
    >>> insp = inspect(engine)
    >>> for col in insp.get_columns("s"):
    ...     print(col["type"].__class__, col["type"].length)
    <class 'sqlalchemy.sql.sqltypes.VARCHAR'> max
    <class 'sqlalchemy.dialects.mssql.base.VARBINARY'> max

在基本类型中，“长度”参数预期是一个整数值或仅为None的值；
None 表示无界限制长度，在 SQL Server 方言中将其解释为“max”。因此，修复长度应设置为 None，以便类型对象在非 SQL Server 上下文中工作：

    >>> for col in insp.get_columns("s"):
    ...     print(col["type"].__class__, col["type"].length)
    <class 'sqlalchemy.sql.sqltypes.VARCHAR'> None
    <class 'sqlalchemy.dialects.mssql.base.VARBINARY'> None

应用程序可能依靠将“长度”值直接与字符串“max”进行比较，应考虑使用值 ``None`` 代替相同的含义。