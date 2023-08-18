=============================
SQLAlchemy 1.0有哪些新特性？
=============================

.. admonition:: 关于本文档

    本文档描述SQLAlchemy 0.9版本至2014年5月维护在进行中的发布
    与于2015年4月发布的SQLAlchemy版本1.0之间的更改。

    最新更新时间：2015年6月9日


引言
============

该指南介绍了SQLAlchemy 1.0版本中的新特性，同时记录了影响用户将应用程序从SQLAlchemy 0.9系列迁移到1.0的更改。

请仔细查看行为更改部分，以确定是否存在潜在的不兼容行为更改。


新功能和改进-ORM
===================================

新的Session批量INSERT/UPDATE API
----------------------------------

创建了一系列新的  :class:`.Session`  方法，这些方法直接提供了向队列操作单元中的插入和更新语句发送钩子的功能。在正确使用的情况下，这个专家级的系统可以允许ORM-mappings生成批量插入和更新语句，并分组进行执行，以便让这些语句速度与核心的直接使用竞争。

.. seealso::

      :ref:`bulk_operations (批量操作)`  - 引言和完整文档

  :ticket:`3100`  

新的性能范例套件
-----------------------------

受到块操作 (bulk_operations) 特性的基准测试和FAQ的影响，增加了一个新的例子章节，其中包含了几个设计用于说明各种Core和ORM技术的相对性能特征的脚本。这些脚本根据用例进行组织，并打包在单一的控制台接口下，使得可以运行任意组合的演示，以便输出时间，Python概要结果和/或RunSnake概要展示。

.. seealso::

      :ref:`examples_performance (性能示例)` 

“烘培”查询
---------------

“烘培”查询是一种不同寻常的新方法，它允许简单地构造和调用：class:`_query.Query` 对象，使用缓存，每次调用时都使用比Python函数调用开销减少75%以上的方法。通过指定  :class:`_query.Query`  对象为一系列仅被调用一次的lambdas，查询作为预编译单元开头的映射变得可行：

    from sqlalchemy.ext import baked
    from sqlalchemy import bindparam

    bakery = baked.bakery()


    def search_for_user(session, username, email=None):
        baked_query = bakery(lambda session: session.query(User))
        baked_query += lambda q: q.filter(User.name == bindparam("username"))

        baked_query += lambda q: q.order_by(User.id)

        if email:
            baked_query += lambda q: q.filter(User.email == bindparam("email"))

        result = baked_query(session).params(username=username, email=email).all()

        return result

.. seealso::

      :ref:`baked_toplevel` 

  :ticket:`3054`  

.. _feature_3150:

提高declarative mixins，“@declared_attr”和相关功能
---------------------------------------------------------------------------

在 :class:`.declared_attr` 与声明式系统联合使用时，支持新的能力进行了重大的改进。

用 :class:`.declared_attr` 装饰的函数现在仅在生成基于冗余的列副本之后才会被调用。这意味着函数可以调用利用冗余建立的列，并将会接收与正确的 :class:`_schema.Column` 对象相关的引用：

    class HasFooBar(object):
        foobar = Column(Integer)

        @declared_attr
        def foobar_prop(cls):
            return column_property("foobar: " + cls.foobar)


    class SomeClass(HasFooBar, Base):
        __tablename__ = "some_table"
        id = Column(Integer, primary_key=True)

例如，上面的“SomeClass.foobar_prop”将用于“SomeClass”，而“SomeClass.foobar”将作为最终映射到“SomeClass”的 :class:`_schema.Column` 对象，而不是直接存在于“HasFooBar”上的未复制对象上。

 :class:`.declared_attr` 函数现在**记忆**它以类为基础返回的值，以使得对同一属性的重复调用返回相同的值。我们可以修改以下示例以说明这一点：

    class HasFooBar(object):
        @declared_attr
        def foobar(cls):
            return Column(Integer)

        @declared_attr
        def foobar_prop(cls):
            return column_property("foobar: " + cls.foobar)


    class SomeClass(HasFooBar, Base):
        __tablename__ = "some_table"
        id = Column(Integer, primary_key=True)

以前，“SomeClass”将被映射为某个特定的“foobar”列的一个拷贝，但是无论属性被映射多少次，“foobar_prop”每次调用“foobar”都会生成一个不同的列。在声明式设置时间内，“SomeClass.foobar”的值现在已被记忆下来，因此即使属性被映射器映射之前，中间列值的值始终保持一致，而不管被调用多少次。

以上的两个行为应该有助于很多类型的映射器属性的声明式定义，这些属性可以导出自其他属性，其中 :class:`.declared_attr` 函数在类实际映射之前被本地存在。

对于一种较瘦的特例，希望构建一个声明性mixin，该mixin根据子类分别建立不同的列，添加了一个新的修饰符  :attr:`.declared_attr.cascading`  。使用这个修饰符，装饰的函数将为映射继承层次结构中的每个类单独调用。尽管这已经是类似于“__table_args__”和“__mapper_args__”等特殊属性的行为，对于默认情况下假定属性仅附加到基类并且只继承自子类的列和其他属性，可以应用单独的行为：

    class HasIdMixin(object):
        @declared_attr.cascading
        def id(cls):
            if has_inherited_table(cls):
                return Column(ForeignKey("myclass.id"), primary_key=True)
            else:
                return Column(Integer, primary_key=True)


    class MyClass(HasIdMixin, Base):
        __tablename__ = "myclass"
        # ...


    class MySubClass(MyClass):
        """ """

        # ...

.. seealso::

      :ref:`mixin_inheritance_columns` 

最后， :class:`.AbstractConcreteBase` 类已经重新进行了改进，以便可以在抽象基类中内联地设置第二个关系或其他映射器属性：

    from sqlalchemy import Column, Integer, ForeignKey
    from sqlalchemy.orm import relationship
    from sqlalchemy.ext.declarative import (
        declarative_base,
        declared_attr,
        AbstractConcreteBase,
    )

    Base = declarative_base()


    class Something(Base):
        __tablename__ = "something"
        id = Column(Integer, primary_key=True)


    class Abstract(AbstractConcreteBase, Base):
        id = Column(Integer, primary_key=True)

        @declared_attr
        def something_id(cls):
            return Column(ForeignKey(Something.id))

        @declared_attr
        def something(cls):
            return relationship(Something)


    class Concrete(Abstract):
        __tablename__ = "cca"
        __mapper_args__ = {"polymorphic_identity": "cca", "concrete": True}

上面的映射将设置一个带有“id”和“something_id”列的“cca”表，而“Concrete”还将具有一个名为“something”的关系。新功能是，“Abstract”还将具有针对基础的多态联接的独立配置关系“something”。

  :ticket:`3150`    :ticket:` 2670`   :ticket:`3149`   :ticket:` 2952`   :ticket:`3050` 

ORM完整对象获取的速度提高了25%
----------------------------------

“loading.py”模块的机制以及标识图现在进行了多次内联，重构和裁减，以便使得加载的原始行现在以约快25%的速度填充了基于ORM的对象。假设有一个100万行的表，像下面这样的脚本可以说明被改进了的加载类型：

    import time
    from sqlalchemy import Integer, Column, create_engine, Table
    from sqlalchemy.orm import Session
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()


    class Foo(Base):
        __table__ = Table(
            "foo",
            Base.metadata,
            Column("id", Integer, primary_key=True),
            Column("a", Integer(), nullable=False),
            Column("b", Integer(), nullable=False),
            Column("c", Integer(), nullable=False),
        )


    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)

    sess = Session(engine)

    now = time.time()

    #避免使用all()，以便减少一次性处理内存中一大堆完整对象的开销
    for obj in sess.query(Foo).yield_per(100).limit(1000000):
        pass

    print("总时间：%d" %（time.time() - now））

本地MacBookPro运行19秒的0.9版现在运行14秒的1.0版。当对大量的行进行分批处理时，：meth:`_query.Query.yield_per` 是一个不错的选择，因为它避免了Python解释器一次性分配大量内存以获取所有对象及其指令的开销。没有：meth:`_query.Query.yield_per`，上述脚本在MacBookPro上的0.9版本为31秒，在1.0版本为26秒，额外的时间花费在设置很大的内存缓冲区上。

.. _feature_3176:

新的KeyedTuple实现速度大大提高
-------------------------------------------------

我们研究了 :class:`.KeyedTuple` 实现，以期提高以下查询的性能::

    rows = sess.query(Foo.a, Foo.b, Foo.c).all()

使用  :class:`.KeyedTuple` ` collections.namedtuple()``的benchmark要比  :class:`.KeyedTuple` ` collections.namedtuple()``很快就超过了：class:`.KeyedTuple`，因为实例调用增加。

该怎么办？一个新型号既兼顾了两种方法的优点。对于“大小”（返回行数）和“编号”（返回不同查询的数量）对所有三个类型进行基准测试，新的“轻量级键值元组”无论是在哪种情况下，都优于另外两种类型，而是基于哪种方案，性能会略有不同::

.. sourcecode:: text

    -----------------
    size=10 num=10000                 #少数据多查询
    namedtuple: 3.60302400589         # namedtuple 不行（失败了）
    keyedtuple: 0.255059957504        # KeyedTuple非常快
    lw keyed tuple: 0.582715034485    # lw keyed 比KeyedTuple慢
    -----------------
    size=100 num=1000                 # <---sweet spot
    namedtuple: 0.365247011185
    keyedtuple: 0.24896979332
    lw keyed tuple: 0.0889317989349   #轻量级键值快于KeyedTuple!
    -----------------
    size=10000 num=100
    namedtuple: 0.572599887848
    keyedtuple: 2.54251694679
    lw keyed tuple: 0.613876104355
    -----------------
    size=1000000 num=10               #少查询多数据
    namedtuple: 5.79669594765         # namedtuple非常快
    keyedtuple: 28.856498003          # KeyedTuple不行
    lw keyed tuple: 6.74346804619     #轻量级键值快于namedtuple

  :ticket:`3176`  

.. _feature_slots:

结构性内存使用的显著改进
-------------------------------------------------

通过为许多内部对象更广泛地使用``__slots__``进行了结构性内存使用的改进。这种优化特别针对具有许多表和列的大型应用程序的基本内存大小，并减少了内存大小，包括事件监听内部，比较器对象和ORM属性等系统的部分。

一个基于heapy的bench可衡量启动时的Nova的大小差异，该bench利用了比应用程序中更大的对象，包括共享字典以及使用的它们的SVN外壳，以及打开的Gunicorn工作器：

.. sourcecode:: text

    #在heapy中报告，SQLAlchemy对象+关联的字典+弱引用相关对象以core of Nova导入为基础占用的总计数和总字节数：

        Before: total count 26477 total bytes 7975712
        After: total count 18181 total bytes 4236456

    #报告Python模块空间的总体字节数在core of Nova导入的情况下：

        Before: Partition of a set of 355558 objects. Total size = 61661760 bytes.
        After: Partition of a set of 346034 objects. Total size = 57808016 bytes.

.. _feature_updatemany:

在刷新中，UPDATE语句会使用executemany()批量批处理
---------------------------------------------------------------

UPDATE语句现在可以在ORM flush中批量处理为更高效的executemany()调用，类似于可以批量处理INSERT语句的方式；这将基于以下标准在flush中引发：

● 两个或更多UPDATE语句连续涉及要修改的相同列集合。

● 语句在SET子句中没有嵌入的SQL表达式。

●映射未使用:paramref:~.orm.mapper.version_id_col，或后端方言支持执行executemany()操作的“明确” rowcount；现在大多数DBAPI们都正确支持这个。

.. _feature_3178:


.. _bug_3035:

Session.get_bind()现在可以处理更广泛的继承场景
-------------------------------------------------------------------

当一个查询或工作单元内的刷新进程试图定位与特定类相对应的数据库引擎时，将调用  :meth:`.Session.get_bind`  方法。方法已经改进，以处理各种面向继承的场景，包括以下：

● 绑定到Mixin或抽象类::

        class MyClass(SomeMixin, Base):
            __tablename__ = "my_table"
            # ...


        session = Session(binds={SomeMixin: some_engine})

● 根据表单独继承的具体子类绑定：

        class BaseClass(Base):
            __tablename__ = "base"

            # ...


        class ConcreteSubClass(BaseClass):
            __tablename__ = "concrete"

            # ...

            __mapper_args__ = {"concrete": True}


        session = Session(binds={base_table: some_engine, concrete_table: some_other_engine})

  :ticket:`3035`  


.. _bug_3227:

Session.get_bind()现在通过所有相关的查询用例接收Mapper
----------------------------------------------------------------------

一系列问题已得到修复，之前：meth:`.Session.get_bind`并没有接收到主要的  :class:`_orm.Mapper` ，即使该映射器是可用的（主要的映射器是单个映射器，或者也可以是第一个映射器，与  :class:` _query.Query` .Session.binds` 一样运用到这个问题上，这个参数使映射器与一系列引擎相关联（尽管在这个用例中，即使在映射表对象获得了绑定时，大多数情况下仍会“工作”）。或者更具体地实现用户定义的  :meth:`.Session.get_bind`  方法来提供基于Mapper的一些选择引擎的方式，例如水平分片或所谓的“路由”会话，该会话路由查询到不同的后端。

这几个场景包括：

●  :meth:`_query.Query.count`  ::

        session.query(User).count()

●  :meth:`_query.Query.update`  和  :meth:` _query.Query.delete`  ，都是针对UPDATE/DELETE的
  语句以及由“fetch”策略使用的SELECT：

        session.query(User).filter(User.id == 15).update(
            {"name": "foob"}, synchronize_session="fetch"
        )

        session.query(User).filter(User.id == 15).delete(synchronize_session="fetch")

●针对单个列的查询::

        session.query(User.id, User.name).all()

●针对间接映射对象（如  :obj:`.column_property`  ）的SQL函数和其他表达：

        class User(Base):
            ...

            score = column_property(func.coalesce(self.tables.users.c.name, None))


        session.query(func.max(User.score)).scalar()

  :ticket:`3227`    :ticket:` 3242`   :ticket:`1326` 

.. _feature_2963:

.info词典改进
-----------------------------

现在，  :attr:`.InspectionAttr.info`  集合可以适用于从  :attr:` _orm.Mapper.all_orm_descriptors` .hybrid_property`和  :func:`.association_proxy` 。但是，由于这些对象是类绑定描述符，因此必须**分别**访问这些对象以获取属性。一下使用  :attr:` _orm.Mapper.all_orm_descriptors`  命名空间进行说明：

    class SomeObject(Base):
        # ...

        @hybrid_property
        def some_prop(self):
            return self.value + 5


    inspect(SomeObject).all_orm_descriptors.some_prop.info["foo"] = "bar"

它也可用作所有  :class:`.SchemaItem` ，  :class:` .UniqueConstraint` ）的构造函数参数。

  :ticket:`2971`  

  :ticket:`2963`  

.. _bug_3188:

ColumnProperty结构与别名以及order_by较好地生成
------------------------------------------------------------------

已经修复了有关  :func:`.column_property` .aliased` 构造以及在0.9中引入的“order by label”的逻辑有关（参见：ref :`migration_1068`）。

给定以下映射::

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)


    class B(Base):
        __tablename__ = "b"

        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))


    A.b = column_property(select([func.max(B.id)]).where(B.a_id == A.id).correlate(A))

简单的情况下" A.b "出现两次将无法正确渲染::

    print(sess.query(A, a1).order_by(a1.b))

这将会排序错误的列：

.. sourcecode:: sql

    SELECT a.id AS a_id, (SELECT max(b.id) AS max_1 FROM b
    WHERE b.a_id = a.id) AS anon_1, a_1.id AS a_1_id,
    (SELECT max(b.id) AS max_2
    FROM b WHERE b.a_id = a_1.id) AS anon_2
    FROM a, a AS a_1 ORDER BY anon_1

现在的输出是

.. sourcecode:: sql

    SELECT a.id AS a_id, (SELECT max(b.id) AS max_1
    FROM b WHERE b.a_id = a.id) AS anon_1, a_1.id AS a_1_id,
    (SELECT max(b.id) AS max_2
    FROM b WHERE b.a_id = a_1.id) AS anon_2
    FROM a, a AS a_1 ORDER BY anon_2

“order by标签”的各种场景也会存在排序失败的问题，例如如果映射是“多态性”的话：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        type = Column(String)

        __mapper_args__ = {"polymorphic_on": type, "with_polymorphic": "*"}

在标签上命名的排序会标记失败，例如给定以下查询：：

    print(sess.query(A).filter(A.type == "foo").order_by(A.b))

现在会正确地按标记排序：

.. sourcecode:: sql

    SELECT a.id AS a_id, a.type AS a_type, (SELECT max(b.id) AS max_1
    FROM b WHERE b.a_id = a.id) AS anon_1
    FROM a
    WHERE a.type = ?
    ORDER BY anon_1

这些问题的包括各种heisenbug，可能会破坏“aliased()”构造的状态，使标记逻辑再次失效；这些均已得到修复。

  :ticket:`3148`    :ticket:` 3188` 

新功能和改进-Core
====================================

.. _feature_3034:

选择/查询LIMIT/OFFSET可以指定为任意的SQL表达式
---------------------------------------------------------------------------

现在，  :meth:`_expression.Select.limit`   和  :meth:` _expression.Select.offset`  方法接受任何SQL表达式作为参数，同时也适用于ORM :class:`_query.Query` 对象。通常，这通常用于允许通过绑定参数传递完整SQL表达式，稍后可以将该参数替换为一个值：

    sel = select([table]).limit(bindparam("mylimit")).offset(bindparam("myoffset"))

不支持非整数LIMIT或OFFSET表达式的方言可能继续不支持这种行为。比第三方方言可能需要修改以便利用新的行为。目前使用“._limit”或“._offset”属性的方言将在访问这两个属性时继续为指定为简单整数值的限制/偏移量。如果指定了SQL表达式，则这两个属性将在访问时抛出一个  :class:`.CompileError` 。如果第三方方言希望支持新功能，它应该现在调用` `._limit_clause``和``._offset_clause``属性来获取完整的SQL表达式，而不是整数值。

.. _feature_3282:

除了在生成整体性的外键约束时指定use_alter标志外
--------------------------------------------------------------------------------

现在，  :meth:`_schema.MetaData.create_all`  和  :meth:` _schema.MetaData.drop_all`  方法将自动生成ALTER语句用于外键约束，这些约束涉及表之间的相互依赖循环，而不需要指定,  :paramref:`_schema.ForeignKeyConstraint.use_alter`  。此外，外键约束现在不需要名称即可通过ALTER创建；只需要在DROP操作中需要名称。在DROP的情况下，如果没有名称，这个功能将确保只有显式名称的约束才会实际上包括在ALTER语句中。在不存在可解决的循环的情况下执行DROP，如果DROP无法继续，则该功能会输出简明和清晰的错误信息。

  :paramref:`_schema.ForeignKeyConstraint.use_alter`  和  :paramref:` _schema.ForeignKey.use_alter`  标志仍然存在，并且继续具有相同的效果来确定需要ALTER的约束，例如在CREATE/DROP场景中需要的约束。

从版本1.0.1开始，在SQLite的情况下，特殊逻辑会接管，如果在DROP过程中给定表存在一个无法解决的循环，则发出警告，而表已按**无序**排列删除，这通常对于SQLite来说是可以的，除非启用了约束。要解决警告并在SQLite数据库上进行至少部分排序，特别是在启用了约束的SQLite数据库上进行至少部分排序，请将“use_alter”标志重新应用于那些应显式省略拍摄的  :class:`_schema.ForeignKey` .ForeignKeyConstraint` 对象。

.. seealso::

      :ref:`use_alter` -全新行为的完整描述。

  :ticket:`3282`  

.. _change_3330:

ResultProxy的“自动关闭”现在是“软关闭”
----------------------------------------------

多个版本中， :meth:`_engine.ResultProxy.close`  的情况下` 使用对象；由于所有DBAPI资源都已被释放，该对象是安全的，可以丢弃。然而，对象保持严格的“关闭”状态，这意味着任何  :meth:`_engine.ResultProxy.fetchone`  ,` _engine.ResultProxy.fetchmany`或  :meth:`_engine.ResultProxy.fetchall`  的后续调用都会引发` :.ResourceClosedError`：

    >>> result = connection.execute(stmt)
    >>> result.fetchone()
    (1, 'x')
    >>> result.fetchone()
    None  # indicates no more rows
    >>> result.fetchone()
    exception: ResourceClosedError

这个行为与pep-249中的描述不一致，pep-249（Python的DBAPI接口）认为即使在结果已用完的情况下，也可以反复调用fetch方法。它也干扰了某些实现的result proxy的行为，如cy_oracle方言使用的  :class:`.BufferedColumnResultProxy` 。

为了解决这个问题，“关闭”的状态已经

被拆分为两个状态；"soft close" 和 "closed"。
"soft close"会释放DBAPI游标和连接，"close with result"对象将释放连接，但不会影响cursor。
  :meth:`_engine.ResultProxy.close`  不再被隐式调用，只有  :meth:` _engine.ResultProxy._soft_close`  被非公开地调用：

    >>> result = connection.execute(stmt)
    >>> result.fetchone()
    (1, 'x')
    >>> result.fetchone()
    None  # 表示没有更多的数据
    >>> result.fetchone()
    None  # 仍然是None
    >>> result.fetchall()
    []
    >>> result.close()
    >>> result.fetchone()
    异常: ResourceClosedError  # 现在会抛出这个异常

CHECK约束现在支持命名约定中的"column_0_name"标记
-----------------------------------------------------------------------------------

"column_0_name"将从 :class:`.CheckConstraint` 表达式的第一个列派生::

    metadata = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

    foo = Table("foo", metadata, Column("value", Integer))

    CheckConstraint(foo.c.value > 5)

将生成：

.. sourcecode:: sql

    CREATE TABLE foo (
        value INTEGER,
        CONSTRAINT ck_foo_value CHECK (value > 5)
    )

此命名约定结合  :class:`.Boolean` .Enum` 等 :class:`.SchemaType` 约束的约束也将使用所有CHECK约束约定。

.. seealso::

      :ref:`naming_check_constraints` 

      :ref:`naming_schematypes` 

  :ticket:`3299`  

.. _change_3341:

引用未附加列的约束可以在列关联到表时自动附加到表上
-----------------------------------------------------------------------------------------------------------------

从版本0.8开始，  :class:`.Constraint` ::

    from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

    m = MetaData()

    t = Table("t", m, Column("a", Integer), Column("b", Integer))

    uq = UniqueConstraint(t.c.a, t.c.b)  # 将自动附加到表

    assert uq in t.constraints

为了帮助声明式产生的某些情况，即使   :class:`_schema.Column`  对象尚未关联到   :class:` _schema.Table`  中，也可以让此自动附加逻辑执行；指定了附加到   :class:`.Constraint`  的列时，当   :class:` _schema.Column`  对象关联到   :class:`_schema.Table`  后，   :class:` .Constraint`  也会被添加::

    from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

    m = MetaData()

    a = Column("a", Integer)
    b = Column("b", Integer)

    uq = UniqueConstraint(a, b)

    t = Table("t", m, a, b)

    assert uq in t.constraints  # constraint自动附加

以上功能是自1.0.0b3版本后的一个过晚的补丁。1.0.4版本的修复脚本为  :ticket:`3411`  。
如果约束引用了一组混合类型的   :class:`_schema.Column`  对象和字符串列名，则忽略此逻辑。
因为我们尚未对 :class:`_schema.Table` 上的名称添加进行跟踪::

    from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

    m = MetaData()

    a = Column("a", Integer)
    b = Column("b", Integer)

    uq = UniqueConstraint(a, "b")

    t = Table("t", m, a, b)

    # 约束不会自动附加，因为我们没有跟踪任何情况
    # 能够定位名称'b'在哪个时刻在表上可用
    assert uq not in t.constraints

以上,列"a"被显式声明，但列"b"尚未显式声明；因此，在查询列"a"附加到表"t"之后，
查询不知道何时olumn "b"将被附加，约束将无法获取"b"，因此将不会进行自动附加的操作。因此，
如果约束使用任何字符串名称，则跳过自动添加-on-column-attach逻辑。

如果在 :class:`_schema.Table` 上下文中已经存在所需的  :class:`_schema.Column` 对象，则原始的自动附加逻辑仍然适用。

    from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

    m = MetaData()

    a = Column("a", Integer)
    b = Column("b", Integer)


    t = Table("t", m, a, b)

    uq = UniqueConstraint(a, "b")

    # 约束自动附加，与较旧的版本相同
    assert uq in t.constraints

  :ticket:`3341`  
  :ticket:`3411`  

.. _change_2051:

.. _feature_insert_from_select_defaults:

"INSERT FROM SELECT"现在包括Python和SQL表达式默认值
------------------------------------------------------

现在，即使另有说明，  :meth:`_expression.Insert.from_select`  也会包括Python和SQL表达式默认值。
此前，约束并未包括在"INSERT FROM SELECT"中的默认值。

例如::

    from sqlalchemy import Table, Column, MetaData, Integer, select, func

    m = MetaData()

    t = Table(
        "t", m, Column("x", Integer), Column("y", Integer, default=func.somefunction())
    )

    stmt = select([t.c.x])
    print(t.insert().from_select(["x"], stmt))

将生成：

.. sourcecode:: sql

    INSERT INTO t (x, y) SELECT t.x, somefunction() AS somefunction_1
    FROM t

可以使用  :paramref:`.Insert.from_select.include_defaults`  禁用此功能。

  :ticket:`3184`  


Column Server Defaults现在呈现为文字值
------------------------------------------------

现在，在需要编译SQL表达式时，如果  :class:`.DefaultClause` 设置的值是一个文字值，编译器会打开"literal binds"编译器标志。这使得嵌入到SQL中的文字值能够被编译器正确呈现，例如：

    from sqlalchemy import Table, Column, MetaData, Text
    from sqlalchemy.schema import CreateTable
    from sqlalchemy.dialects.postgresql import ARRAY, array
    from sqlalchemy.dialects import postgresql

    metadata = MetaData()

    tbl = Table(
        "derp",
        metadata,
        Column("arr", ARRAY(Text), server_default=array(["foo", "bar", "baz"])),
    )

    print(CreateTable(tbl).compile(dialect=postgresql.dialect()))

现在呈现：

.. sourcecode:: sql

    CREATE TABLE derp (
        arr TEXT[] DEFAULT ARRAY['foo', 'bar', 'baz']
    )

以前，文字值"foo"、“bar”和“baz”将呈现为绑定参数，这在DDL中是无用的。

  :ticket:`3087`  

.. _feature_3184:

``UniqueConstraint``现在是数据表反射过程的一部分
--------------------------------------------------------

使用``autoload=True``填充的  :class:`_schema.Table` .UniqueConstraint` 和 :class:`.Index` 结构。 这对于PostgreSQL和MySQL有一些注意事项：

PostgreSQL
^^^^^^^^^^

PostgreSQL的行为是这样的，当创建UNIQUE约束时，它隐含地同时创建了一个对应的唯一的索引。  :meth:`_reflection.Inspector.get_indexes`  和  :meth:` _reflection.Inspector.get_unique_constraints`  方法仍然都会分别返回这些条目，其中  :meth:`_reflection.Inspector.get_indexes`  现在的特性中包括一个令人惊讶的“重复约束”，其中包含：如果检测到相应的约束，则对于具有此标识符的索引条目，将被映射到相应的约束。然而，使用 ` `Table(..., autoload=True)``
执行完整的表反射时，   :class:`.Index` .UniqueConstraint` 构造有关的，因此不会出现  :attr:`_schema.Table.indexes` ，只有  :class:` .UniqueConstraint`  中。这种去重逻辑通过加入到查询"pg_index"时连接"pg_constraint"表来完成。

MySQL
^^^^^

MySQL在没有可区分UNIQUE索引和UNIQUE约束的概念。虽然在创建表和索引时支持这两种语法，但实际上不会对它们进行任何不同的存储。  :meth:`_reflection.Inspector.get_indexes`  和  :meth:` _reflection.Inspector.get_unique_constraints`  仍然会分别返回UNIQUE索引的条目，在MySQL中只需将其视为UNIQUE约束。但是，在使用 ``Table(..., autoload=True)`` 执行完整的表反射时，无论什么情况下，  :class:`.UniqueConstraint` .Index` 表示，  :attr:` _schema.Table.indexes`  中存在包含“unique=True”的设置。

.. seealso::

      :ref:`postgresql_index_reflection` 

      :ref:`mysql_unique_constraints` 

  :ticket:`3184`  


新系统安全发出参数化警告
--------------------------------------------

长期以来，警告消息不能引用数据元素，这意味着某个特定函数可能会发出一个无限数量的唯一警告消息。  :class:`.Unicode` 类型部分中的警告消息"Unicode type received non-unicode bind param value"就是这个问题的例子。将数据值放入该消息中将意味着在Python "__warningregistry__"中记录的模块，或在某些情况下是Python全局"warnings.onceregistry"中记录的模块，会不断增加新的unique警告消息。

这个改变在实际使用中利用了一个特殊的"string"类型，该类型有意改变了字符串的哈希方式，使得大量参数化消息仅使用一小组可能的哈希值进行哈希，以便例如"Unicode type received non-unicode bind param value"警告可被定制地仅限制发送一定数量的次数; 不再发送新的警告。因此，在执行此类操作时，您可能会看到以下警告:

.. sourcecode:: text

    SAWarning: Got None for value of column user.id; this is unsupported
      for a relationship comparison and will not currently produce an
      IS comparison (but may in a future release)

请注意，这种模式在大多数情况下已经被破坏了，特别是当我们没有将值发送到ORM中时，即没有设置属性，则   :class:`.Unicode`  等类似类型不再将通知我们。如果您的应用程序依赖于"NULL = NULL"在所有情况下都失败的事实，并且运行风险较高，则应该检查 :ticket:` 3178`的解决方案。

  :ticket:`3178`  

Key Behavioral Changes - ORM
============================

.. _bug_3228:

：meth:`_query.Query.update`现在可以将字符串名称解析为映射属性名称
--------------------------------------------------------------

方法  :meth:`_query.Query.update`  的文档说明指示给定的"values"字典是"一个使用属性名称为key的字典"，这暗示这些都是映射属性名称。 但不幸的是，该函数的设计更多考虑了属性和SQL表达式，而没有考虑字符串; 当字符串被传递时，这些字符串将直接通过，进入核心更新语句，而不考虑这些名称在映射类中的表示方式，这意味着该名称必须完全匹配表列，而不是该名称映射到该类上的属性名称。

现在，字符串名称已经解析为属性名称::

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column("user_name", String(50))

上面是一个映射，其中列"user_name"映射为"name"。 以前，在  :meth:`_query.Query.update`  中传递字符串名称，必须这样写::

    session.query(User).update({"user_name": "moonbeam"})

现在，传递的字符串将根据实体解析为属性名称::

    session.query(User).update({"name": "moonbeam"})

通常最好直接使用属性，以避免任何模棱两可:: 

    session.query(User).update({User.name: "moonbeam"})

此更改还表示， 同义词和混合属性也可以通过字符串名称引用::

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column("user_name", String(50))

        @hybrid_property
        def fullname(self):
            return self.name

    session.query(User).update({"fullname": "moonbeam"})

  :ticket:`3228`  

.. _bug_3371:

在将带有None值的对象与关系比较时发出警告
-------------------------------------------------------------------------

此更改是从1.0.1开始的新更改，一些用户正在执行基本上是这种形式的查询：

     session.query(Address).filter(Address.user == User(id=None))

这种模式在SQLAlchemy中目前不受支持。对于所有版本，在查询标量或一对多关系时，如果尚未设置该值，则将返回“None”值。

例如，考虑上面定义的两个映射，一对多关系将定义如下：

    class User(Base):
        __tablename__ = "user"
        id = Column(Integer, primary_key=True)
        addresses = relationship("Address", backref="user")

    class Address(Base):
        __tablename__ = "address"
        id = Column(Integer, primary_key=True)
        user_id = Column(Integer, ForeignKey(User.id))
        email_address = Column(String)


现在，如果链接到一个User对象，然后执行：

    session.query(Address).filter(Address.user == User(id=None))

在所有版本中，将发出SQL语句：

.. sourcecode:: sql

    SELECT address.id AS address_id, address.user_id AS address_user_id,
    address.email_address AS address_email_address
    FROM address WHERE ? = address.user_id
    (None,)

请注意，上述模式一直不符合SQL语义。 "WHERE？= address.user_id"类似于“WHERE NULL = address.user_id”，这不论如何都将产生“FALSE”。 此模式已经破坏了大多数情况，因为在关系数据库中，“缺失值”通常被视为NULL 。一个单独的模式是比如：：func:`~sqlalchemy.sql.expression.or_` (`Address.user_id`==None，`Address.user``==``None`)

但仍然有某些人知道这种模式，因为它可能已经存在于许多年前的某个应用项目中。 仍然支持此查询模式，但是现在会出现警告:

.. sourcecode:: text

    SAWarning: Got None for value of column user.id; this is unsupported
    for a relationship comparison and will not currently produce an
    IS comparison (but may in a future release)

  :ticket:`3371`  

.. _bug_3374:

"否定contains或equals"关系比较将使用属性的当前值而不是数据库值
-------------------------------------------------------------------------------------------------------------------------

此更改从1.0.1开始。现在，在查询将关系定向到目标对象的情况下，将动态地返回每次访问的默认返回值，而不是在首次访问时隐式设置属性状态，调用一个“设置”函数。上面的更改可见的结果是此时不再在__dict__对象上隐式修改值，并且与   :func:`.attributes.get_history`  和相关函数的历史上也存在一些小行为变化。

给出一个没有状态的对象：

    >>> obj = Foo()

如果我们访问一个未设置过的标量或一对多属性，SQLAlchemy的行为始终是返回“None”：

    >>> obj.someattr
    None

实际上，这个“None”的值现在是该对象的状态的一部分，并且类似于专门设置属性，例如“obj.someattr = None”。但是，“get”操作的“set on get”行为在历史和事件方面会有所不同。当首次调用“get”并返回“None”时，不会发出任何属性事件。另外，如果查看历史记录，则会发现：

    >>> inspect(obj).attrs.someattr.history
    History(added=(), unchanged=[None], deleted=())  # 0.9

意思是属性始终为“None”，从未更改过。这是明确不同的，如果首先设置了该属性：

    >>> obj = Foo()
    >>> obj.someattr = None
    >>> inspect(obj).attrs.someattr.history
    History(added=[None], unchanged=(), deleted=())  # 极旧的版本和最新版本都相同

以上意味着，在要求“get”操作时存在使属性事件出现的不一致性。无论
将 None 值设置到属性上还是其他值，取决于 get 操作的表现方式。既而，对于一个“set”操作，现在永远不会实际上设置属性原本的“默认值”了。

    >>> obj = Foo()
    >>> obj.someattr
    None
    >>> inspect(obj).attrs.someattr.history
    History(added=(), unchanged=(), deleted=())  # 1.0
    >>> obj.someattr = None
    >>> inspect(obj).attrs.someattr.history
    History(added=[None], unchanged=(), deleted=())

上面所说的行为意味着，在“set”操作在关系上属性取代的情况下，始终会使用默认操作，而不是赋值操作，如果 None 也是值之一。

主要变更 - 流程管理
============================

.. _bug_3228:

query.update()现在解析字符串名称以映射属性名称
--------------------------------------------------------------

  :meth:`_query.Query.update`  的文档说明指出，给定的“values”字典是“一个使用属性名称为键的字典”，这暗示这些都是映射属性名称。 :meth:` _query.Query.update`的设计更多考虑了属性和SQL表达式，而不是字符串; 当字符串被传递时，这些字符串将直接通过核心更新语句，而不考虑这些名称在映射类中的表示方式，这意味着，该名称必须完全匹配表列，而不是该名称映射到该类上的属性名称。

现在，字符串名称已经解析为属性名称::

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column("user_name", String(50))

邮件中指出，在  :meth:`_query.Query.update`  中传递字符串名称，必须这样写::

    session.query(User).update({"user_name": "moonbeam"})

现在，传递的字符串将解析为属性名称::

    session.query(User).update({"name": "moonbeam"})

通常地，建议直接使用属性来避免模棱两可的情况::

    session.query(User).update({User.name: "moonbeam"})

此更改还表示,synonyms和混合属性也可以按字符串名称引用::

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column("user_name", String(50))

        @hybrid_property
        def fullname(self):
            return self.name

    session.query(User).update({"fullname": "moonbeam"})

  :ticket:`3228`  

.. _bug_3371:

当将对象与None值进行比较时发出警告
-------------------------------------------------------------------------

这个改动从1.0.1开始。对于长时间以来的查询行为状态在 :ref:`migration_3060` 中被说明，对于一对多/多对一关系，设置关系绑定属性，可以覆盖为外键所设置的值。 建议使用传统的单列 NULL 值来列出这个模式。 但是，此更改仍支持此查询模式，但是现在会出现警告：

.. sourcecode:: text

    SAWarning: Got None for value of column user.id; this is unsupported
    for a relationship comparison and will not currently produce an
    IS comparison (but may in a future release)

  :ticket:`3371`  

.. _bug_3374:

已删除的对象全部深度解绑过程已更正
-----------------------------------------------------------

此浅实例在  :meth:`.Session.expunge`  中存在缺陷，导致在删除对象时出现一致性问题。即使在解绑之后，函数：func:` .object_session`和  :attr:`.InstanceState.session`  属性仍然会将对象视为属于   :class:` .Session`  。 

    assert inspect(u1).session is sess  #提交之前正常

    sess.expunge(u1)

    assert u1 not in sess
    assert inspect(u1).session is None  #失败！

请记住，“u1 not in sess”为True，而“inspect（u1）.session”仍然引用会话，同时删除操作之后的事务尚未进行完全脱离；通常，在提交事务后，完全分离会话。 这个问题也会影响使用“Session.expunge”的函数，例如“make_transient”。

看到“[链接名]：（..路径）”，不要修改英文的冒号！请区分术语和说明文本，不用翻译术语。不要破坏python sphinx文档相关语法标记。非常重要的是，不要将原文的标点符号转换成中文标点符号！

:ticket：``3139``

.. _migration_yield_per_eager_loading:

当使用yield_per时，禁止显式使用Joined/Subquery eager loading + 嵌套加载
-------------------------------------------------- ----------------

为了使  :meth:`_query.Query.yield_per`  方法更容易使用，如果任何子查询急加载器，
或将使用集合的连接急加载器
在使用yield-per时生效，则会引发异常，因为这些当前不兼容
具有yield-per的急加载（子查询可以在理论上设置）。当
引发此错误时，可以将：func：`.lazyload`选项发送到带有
asterisk ::

    q = sess.query(Object).options(lazyload（“*”）).yield_per(100)

或使用：meth:`_query.Query.enable_eagerloads` ::

    q = sess.query(Object).enable_eagerloads(False).yield_per(100)

：func：`.lazyload`选项的优点是仍然可以使用附加的多对一
加入的装载程序选项::

    q = (
        sess.query（Object）
        .options（lazyload（“*”），joinedload（“some_manytoone”））
        .yield_per(100)
    )

.. _bug_3233:

重复的连接采取措施的更改和修复
-------------------------------------------------- -------

此处的更改包括加入一个实体时，在没有基于关系的ON子句的情况下添加到同一表格或多个单表实体中的某些场景中
连接两次，以及连接多次到相同的目标关系。

从以下映射开始：：

    from sqlalchemy import Integer，Column，String，ForeignKey
    from sqlalchemy.orm import Session，relationship
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base（）


    class A（Base）：
        __tablename__ =“a”
        id = Column（Integer，primary_key = True）
        bs = relationship（“B”）


    class B（Base）：
        __tablename__ =“b”
        id = Column（Integer，primary_key = True）
        a_id =列（ForeignKey（“a.id”））

加入“A.bs”两次的查询：：

    print（s.query（A）。join（A.bs）。join（A.bs））

将呈现：

.. sourcecode :: sql

    SELECT a.id AS a_id
    FROM a JOIN b ON a.id = b.a_id

该查询去重了冗余的“A.bs”，因为它试图支持以下情况之一：：func：`.make_transient`

    s.query（A）。join（A.bs）。filter（B.foo ==“bar”）。reset_joinpoint（）。join（A.bs，B.cs）。filter（
        C.bar ==“bat”
    ））

也就是说，“A.bs”是“路径”的一部分。随着：ticket：`3367`到到达相同的终点点两次而不是
在较大的路径中，现在将产生警告：

.. sourcecode :: text

    SAWarning：路径连接目标A.bs已经被连接;跳过

更大的变化涉及当连接到一个实体时，在不使用
ON绑定路径。如果我们两次连接到“B”：

    打印（s.query（A）.join（B，B.a_id == A.id）。join（B，B.a_id == A.id））

在0.9中，这将呈现如下：

.. sourcecode :: sql

    SELECT a.id AS a_id
    FROM a JOIN b ON b.a_id = a.id JOIN b AS b_1 ON b_1.a_id = a.id

这是有问题的，因为隐式别名是隐含的，并且在不
在不同的ON子句中的情况下产生不可预测的结果。

在1.0中，不会自动应用别名，并且我们得到：

.. sourcecode :: sql

    SELECT a.id AS a_id
    FROM a JOIN b ON b.a_id = a.id JOIN b ON b.a_id = a.id

这将从数据库中抛出错误。虽然它可能很好
如果“重复连接目标”在我们从冗余关系vs中连接到两个对象时执行相同的操作
重复的非关系目标时，现在我们仅在更严重的情况下更改行为，其中
以前发生了隐式别名，而只对关系的情况进行警告。最终，在所有情况下，连接到同一个对象两次而不需要任何别名以进行消歧义的行为应该引发错误。

更改还影响单表继承目标。使用
以下映射：

    from sqlalchemy import Integer，Column，String，ForeignKey
    from sqlalchemy.orm import Session，relationship
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base（）


    class A（Base）：
        __tablename__ =“a”

        id = Column（Integer，primary_key = True）
        type = Column（String）

        __mapper_args__ = {“polymorphic_on”：type，“polymorphic_identity”：“a”}


    class ASub1(A)：
        __mapper_args__ = {“polymorphic_identity”：“asub1”}


    class ASub2（A）：
        __mapper_args__ = {“polymorphic_identity”：“asub2”}


    class B(Base)：
        __tablename__ =“b”

        id = Column（Integer，primary_key = True）

        a_id =列（Integer，ForeignKey（“a.id”））

        a = relationship（“A”,primaryjoin =“B.a_id == A.id”, backref =“b”）


    s = Session（）

    print（s.query（ASub1）。join（B，ASub1.b）。join（ASub2，B.a））

    print（s.query（ASub1）。join（B，ASub1.b）。join（ASub2，ASub2.id == B.a_id））

底部的两个查询等效，并且应该同时呈现相同的SQL 中Text嵌套的总数在ORM而不是Core表达式语言中递增。为此，应该使用函数：func：`sa.orm.load_only`：

    print（s.query（A.id，A.name，A.nickname，A.ordercount，
        B.id，B.name）。\
            join（A.b）.join（B，B.id == A.id）。\
            options（load_only（“name”））.join（B.c）\
            .all（））

单独使用" load_only"对于PostgreSQL来说并不是一个好的主意，因为它通常在客户端和服务器之间包装结果组，并且包装后的列名称已经被\" as\"重新命名，这会变成：SELECT anon_1.some_name_ AS some_name_1，而不是：SELECT some_name_，可重载的SELECT之后，因此不一定会在结果组之后到达客户端。

。。 _migration_3222：

单表继承类型的所有ON子句都被添加到条件语句中
--------------------------------------------------

当连接到单表继承子类目标时，ORM总是添加了“单一表”条件，当连接关系时。
从以下映射开始：：

    class Widget(Base)：
        __tablename__ =“widget”
        id =列（Integer，primary_key = True）
        type = Column（String）
        related_id =列（ForeignKey（“related.id”））
        related = relationship（“Related”，backref =“widget”）
        __mapper_args__ = {“polymorphic_on”：type}


    class FooWidget(Widget)：
        __mapper_args__ = {“polymorphic_identity”：“foo”}


    class Related(Base)：
        __tablename__ =“related”
        id =列（Integer，primary_key = True）

它一直是行为对于关系的连接“A.b”使用以下这种形式的ON子句：

    s.query（Related）。join（FooWidget，Related.widget）。全部（）

SQL输出：

.. sourcecode :: sql

    SELECT related.id AS related_id
    FROM related JOIN widget ON related.id = widget.related_id AND widget.type IN (:type_1)

注意：因为我们连接到一个子类“FooWidget”，所以  :meth:`_query.Query.join`  
知道要向ON子句添加“AND widget.type IN（'foo'）”条件。

这里的更改是将“AND widget.type IN（）”条件附加到*任何*ON子句，而不仅仅是从关系中生成的，
包括显式状态的一种...：meth：`_query.Query.join`，

    # ON子句现在将被呈现为
    # related.id = widget.related_id AND widget.type IN (:type_1)
    s.query（关联）。join（FooWidget，FooWidget.related_id == Related.id）。all（）

以及没有任何类型的东西都没有ON子句时的“隐式”连接：

    # ON子句现在将被呈现为
    # related.id = widget.related_id AND widget.type IN (:type_1)
    s.query（Related）。join（FooWidget）。all（）

先前，“这些ON子句”不包括单一继承条件。应用了此问题的应用程序现在将希望移除其显式使用的条件，在其中进行操作与此同时，应该可以正常工作，即使在其间被渲染两次。

.. seealso ::

    :ref：`bug_3233`

:ticket：``3222``

退役事件钩子已被删除
--------------------------------

在ORM事件钩子的使用案例，其中一些从0.5开始就被弃用的事件钩子已被移除：``translate_row``，``populate_instance``,
``append_result``，``create_instance``。 这些挂钩的用例
起源于很早的0.1 / 0.2系列的SQLAlchemy，而且早就不再需要。 特别是用钩子
主要无法使用，因为在这些事件中行为合同与
周围内部的相关性如何初始化和创建实例以及如何定位ORM生成的列。
删除这些挂钩大大简化了ORM对象加载的机制。

.. _bundle_api_change:

出于自定义行加载器被用于新Bundle功能的API更改
--------------------------------------------------------------

在0.9中，新的：class：`。Bundle`对象在自定义类上覆盖了``create_row_processor()``方法
当部分或完全由文本片段组成的SQL时，发出了警告。
``create_row_processor()``方法被覆盖时，默认的示例代码如下：

    from sqlalchemy.orm import Bundle


    class DictBundle(Bundle)：
        def create_row_processor(self，query，procs，labels)：
            """覆盖create_row_processor以将值作为字典返回"""

            def proc（row，result）：
                return dict(zip(labels，（proc(row，result) for proc in procs)))

            返回过程

未使用的“结果”成员现已删除：

    from sqlalchemy.orm import Bundle


    class DictBundle(Bundle)：
        def create_row_processor(self，query，procs，labels)：
            """覆盖create_row_processor以将值作为字典返回"""

            def proc（row）：
                return dict(zip(labels，（proc(row) for proc in procs)))

            返回过程

..另请参阅 ::

    :ref：`bundles`

:ticket：``3155``

使用join（），“synchronize_session ='evaluate'”时出现多表更新时出现
-------------------------------------------------- --------------------------

“Evaluator”：“_query.Query.update”不适用于多表
更新，并且在存在多个表时需要将其设置为“synchronize_session = False”或
“synchronize_session ='fetch'”。新行为是现在显式引发异常，
其中包含一条消息以更改同步设置。
这使从SA0.9.7起发出的警告升级为异常。

:ticket：``3117``

query.update（）/ query.delete（）在使用join（），select_from（），from_self（）时引发，反之亦然
-------------------------------------------------- -----------------------------

SA0.9.10（截至2015年6月9日尚未发布）在使用方法时会发出警告
“_query.Query.update”或“_query.Query.delete”方法
与调用：meth：“_query.Query.join”，：meth：“_query.Query.outerjoin”，
：meth：“_query.Query.select_from”或：meth：“_query.Query.from_self”。这些是不受支持的
使用情况在0.9系列中默默地失败，直到0.9.10添加了警告
之后，这些情况现在会引发异常。

:ticket：``3349``

具有“synchronize_session = 'evaluate'”的query.update（）引发具有多表更新的异常
--------------------------------------------------------------------

“评估程序”：：“_query.Query.update”不适用于多个表
更新，并且当有多个表时需要将其设置为False或'synchronize_session = fetch'
synchronize_session。新行为是现在显式引发异常，
其中包含一条消息以更改同步设置。

:ticket：``3117``

已删除“Resurrect”事件
--------------------------------

完全删除了“复活”ORM事件。自0.8开始，这个事件就不再有任何功能了
在工作单位中从早期版本的0.1 / 0.2中删除了更改系统。
你现在是一个.rst文档翻译器。在上面的语句中，我们期望看到"ORDER BY id_count"，而不是该函数的重新陈述。在编译期间，该字符串参数将与列子句中的一个条目进行匹配，因此该声明将按我们的期望进行处理，而不会发出警告（尽管请注意，"name"表达式已解析为"users.name"！）：

.. sourcecode:: sql

    SELECT users.name, count(users.id) AS id_count
    FROM users GROUP BY users.name ORDER BY id_count

但是，如果我们引用无法定位的名称，则再次会出现警告，如下所示::

    stmt = select([user.c.name, func.count(user.c.id).label("id_count")]).order_by(
        "some_label"
    )

该输出按照我们说的做，但再次警告我们：

.. sourcecode:: text

    SAWarning: Can't resolve label reference 'some_label'; converting to text() (this warning may be suppressed after 10 occurrences)

.. sourcecode:: sql

    SELECT users.name, count(users.id) AS id_count
    FROM users ORDER BY some_label

上述行为适用于我们可能希望引用所谓的“标签引用”的所有位置；ORDER BY和GROUP BY，还包括OVER子句以及引用列的DISTINCT ON子句（例如，PostgreSQL语法）。

我们仍然可以使用：func：`_expression.text`指定任意的表达式进行ORDER BY或其他操作：

    stmt = select([users]).order_by(text("some special expression"))

整个更改的要点是，SQLAlchemy现在希望我们告诉它当发送一个字符串时，此字符串显式地是：func：`_expression.text`构造，列，表等，如果我们将其用作ORDER BY，GROUP BY或其他表达式中的标签名称，SQLAlchemy希望该字符串解析为已知的内容，否则应再次限定为：func：`_expression.text`或类似的构造。

  :ticket:`2992`  

.. _bug_3288:

使用多值插入时为每个行单独调用Python端默认值
--------------------------------------------------------------------------------------

使用:meth :`_expression.Insert.values`的多值版本时，对Python端列默认值的支持基本上未实现，并且仅在特定情况下“偶然”工作，当使用的方言时使用非位置（例如，命名）样式的绑定参数，并且在不需要为每行调用Python端可调用的情况下。

该功能已进行了大修，以便它更类似于“执行多个”样式的调用::

    import itertools

    counter = itertools.count(1)
    t = Table(
        "my_table",
        metadata,
        Column("id", Integer, default=lambda: next(counter)),
        Column("data", String),
    )

    conn.execute(
        t.insert().values(
            [
                {"data": "d1"},
                {"data": "d2"},
                {"data": "d3"},
            ]
        )
    )

上面的示例将为每个行单独调用``next(counter)``，就像预期的那样：

.. sourcecode:: sql

    INSERT INTO my_table (id, data) VALUES (?, ?), (?, ?), (?, ?)
    (1, 'd1', 2, 'd2', 3, 'd3')

以前，默认情况下，位置方言将失败，因为不会为附加的位置生成绑定：

.. sourcecode:: text

    没有提供错误数量的绑定。当前语句使用6，
    并且提供了4个。
    [SQL：u'INSERT INTO my_table（id，data）VALUES（？，？），（？，？），（？，？）']
    [parameters：（1，“d1”，“d2”，“d3”）]

对于“命名”的方言，如果仅引用服务器端默认值，则将重复使用“id”的相同值在每行中：

.. sourcecode:: sql

    INSERT INTO my_table（id，data）VALUES（：id，：data_0），（：id，：data_1），（：id，：data_2）
     - {u'data_2'：'d3'，u'data_1'：'d2'，u'data_0'：'d1'，'id'：1}

还拒绝用内联呈现的SQL作为“服务器端”默认值，因为不能保证服务器端默认值与此兼容。如果VALUES子句为特定列呈现，则需要Python-side值；如果省略的值仅引用服务器端默认值，则会引发异常::

    t = Table(
        "my_table",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("data", String, server_default="some default"),
    )

    conn.execute(
        t.insert().values(
            [
                {"data": "d1"},
                {"data": "d2"},
                {},
            ]
        )
    )

将引发以下异常：

.. sourcecode:: text

    sqlalchemy.exc.CompileError：INSERT value for column my_table.data is
在VALUES子句中显式呈现为boundparameter；a
需要Python-side value or SQL expression.

以前，该值“d1”将复制到第三行的值（但仅对于命名格式！）：

.. sourcecode:: sql

    INSERT INTO my_table（data）VALUES（：data_0），（：data_1），（：data_0）
    - {u'data_1'：'d2'，u'data_0'：'d1'}

  :ticket:`3288`  

.. _change_3163:

无法从该事件的运行程序中添加或删除事件侦听器
---------------------------------------------------------------------------

从事件自身中删除事件侦听器将导致对列表的元素进行修改，这将导致仍附加的事件侦听器无法触发。为防止这种情况，同时仍然保持性能，使用``collections.deque（）``替换了列表，该列表不允许在迭代期间进行任何添加或删除，并引发``RuntimeError``。

  :ticket:`3163`  

.. _change_3169:

INSERT...FROM SELECT构造现在暗示“inline = True”
--------------------------------------------------------------

使用:func：`_expression.from_select`现在意味着在:func：`_expression.insert`上有``inline = True``，从而修复了错误，后端支持“隐式返回”的情况下，INSERT...FROM SELECT结构将错误地被编译为，在该情况下，如果INSERT插入零行（因为implicit返回值需要一行），会导致断点，以及在INSERT插入多个行的情况下，返回任意数据（例如，仅多行中的第一行）。还将类似的更改应用于具有多个参数设置的INSERT..VALUES；此语句不再发出implicit RETURNING。由于这些构造处理可变数量的行，因此
  :attr:`_engine.ResultProxy.inserted_primary_key`  访问器不适用。以前，有一条文档注释说，如果某些数据库不支持返回值，可以使用` `inline = True``来使用INSERT... FROM SELECT，但在任何情况下，INSERT... FROM SELECT都不需要implicit返回，如果需要返回插入的数据行数，则应使用常规显式:func：`_expression.Insert.returning`.

  :ticket:`3169`  

.. _change_3027:

``autoload_with``现在意味着``autoload = True``
----------------------------------------------

可以通过仅传递  :paramref:`_schema.Table.autoload_with`  设置反射的 :class:` _schema.Table`对象：

    my_table = Table("my_table", metadata, autoload_with=some_engine)

  :ticket:`3027`  

.. _change_3266:

DBAPI异常包装和handle_error（）事件改进
--------------------------------------------------------------

当使用多行插入的多值版本时，SQLAlchemy对DBAPI异常的包装没有在
 :class:`_engine.Connection` 无效后重连并遇到错误时进行。这
已得到解决。此外，最近添加的:meth：`_events.ConnectionEvents.handle_error`
事件现在在读取期间发生错误时触发，并在通过  :paramref:`_sa.create_engine.creator`  通过自定义连接使用:func：` _sa.create_engine`时触发。

了上:`.ExceptionContext`对象具有一个新的数据成员
:attr：`.ExceptionContext.engine`，它将始终引用  :class:`_engine.Engine` ，在这些情况下， :class:` _engine.Connection`对象不可用（例如，初始连接时）。

  :ticket:`3266`  

.. _change_3243:

ForeignKeyConstraint.columns现在是ColumnCollection
------------------------------------------------------

:attr：`_schema.ForeignKeyConstraint.columns`先前是一个普通的列表
包含字符串或 :class:`_schema.Column` 对象，具体取决于如何进行
如果使用了  :class:`_schema.ForeignKeyConstraintM` ，并且仅在 :class:` _schema.ForeignKeyConstraint`与 :class:`_schema.Table` 关联后才初始化。添加了一个新的访问器
  :attr:`_schema.ForeignKeyConstraint.column_keys`  
无条件返回本地列集的字符串键，无论如何构造对象或其当前状态如何。

.. _feature_3084:

MetaData.sorted_tables访问器是“确定性的”
--------------------------------------------------

对于  :attr:`_schema.MetaData.sorted_tables`  访问器所产生的表的排序是“确定性的”，所有情况下的排序应该是相同的，而无需任何Python散列。 通过首先按名称对表进行排序，然后将它们传递给拓扑算法来维护该排序。

请注意，此更改尚未应用于在发出  :meth:`_schema.MetaData.create_all`  或  :meth:` _schema.MetaData.drop_all`  时应用的排序。

  :ticket:`3084`  

.. _bug_3170:

null（），false（）和true（）常量不再是单例
--------------------------------------------------


在0.9中，这三个常量被更改为返回“单例”值;不幸的是，这将导致如下查询不能按预期渲染：

    select([null()，null()])

仅呈现``SELECT NULL AS anon_1``，因为两个  :func:`.null` 
构造将会成为相同的''NULL''对象，并且SQLAlchemy的核心模型基于对象身份来确定词汇意义。 0.9的更改除了想要节省对象开销之外没有任何重要性;通常，未命名的构造需要保持词汇唯一以获得唯一标识。

  :ticket:`3170`  

.. _change_3204:

PostgreSQL / Oracle在报告临时表/视图名称时具有不同的方法
---------------------------------------------------------------------------

在PostgreSQL / Oracle的情况下，  :meth:`_reflection.Inspector.get_table_names`  和  :meth:` _reflection.Inspector.get_view_names`  在适用于临时表和视图的情况下也会返回名称，而对于任何其他方言都没有提供。 在MySQL的情况下，至少对于临时表来说，它甚至不可能。 在这种情况下，对于名为“临时表”或“临时视图”的名称，重构的   :class:`_schema.Table`  构造将于存在于数据库中的名称进行匹配。

  :ticket:`3204`  

Dialect Improvements and Changes - PostgreSQL
=============================================

.. _change_3319:

ENUM类型的创建/删除规则彻底改写
---------------------------------------

PostgreSQL的 :class:`_postgresql.ENUM` 的规则已更严格，以便更好地支持TYPE的创建和删除。

创建**未显式**与  :class:`_schema.MetaData`  和  :meth:` _schema.Table.drop `  对应::

    table = Table(
        "sometable", metadata, Column("some_enum", ENUM("a", "b", "c", name="myenum"))
    )

    table.create(engine)  # 将发出CREATE TYPE和CREATE TABLE
    table.drop(engine)  # 将发出DROP TABLE和DROP TYPE - 1.0的新功能

这意味着如果第二个表也具有名为'myenum'的枚举，则上述DROP操作现在将失败。为了适应共享的常见枚举类型使用案例，增强了具有元数据关联枚举的行为。

**显式**与  :class:`_schema.MetaData`  和  :meth:` _schema.Table.drop`  创建或删除:  :tparamref:`_schema.Table.create`  调用了` `checkfirst = True``标志。

:

    my_enum = ENUM("a", "b", "c", name="myenum", metadata=metadata)
    table = Table("sometable", metadata, Column("some_enum", my_enum))

    #将失败：ENUM'my_enum'不存在
    table.create(engine)

    #将检查enum并发出CREATE TYPE
    table.create(engine, checkfirst=True)

    table.drop(engine)  #将发出DROP TABLE，*不会* DROP TYPE
    metadata.drop_all(engine)  # 将发出DROP TYPE
    metadata.create_all(engine)  # 将发出CREATE TYPE

  :ticket:`3319`  

PostgreSQL表选项
----------------------------

添加了表空间，ON COMMIT，WITH（OUT）OIDS和INHERITS的PG表选项，在通过
 :class:`_schema.Table` 构造的DDL渲染时提供支持。

.. seealso::

      :ref:`postgresql_table_options` 
    
  :ticket:`2051`  

.. _feature_get_enums:

PGInspector.get_enums()方法与PostgreSQL语言配合使用
----------------------------------------------

对PostgreSQL的  :func:`_sa.inspect` .PGInspector` 对象，在
可以使用新的  :meth:`.PGInspector.get_enums`  方法返回所有可用的` `ENUM``类型的信息::


    from sqlalchemy import inspect, create_engine

    engine = create_engine("postgresql+psycopg2://host/dbname")
    insp = inspect(engine)
    print(insp.get_enums())

.. seealso::

     :meth:`.PGInspector.get_enums` 

.. _feature_2891:

PostgreSQL dialect反映Materialized Views，Foreign Tables
-----------------------------------------------------

变化如下：

* 通过传递autoload=True的Table`构造将匹配作为材料化视图或外部表存在于数据库中的名称。

* .._reflection.Inspector.get_view_names将返回纯视图和Materialized View名称。

* .._reflection.Inspector.get_table_names对于PostgreSQL**不会更改**，它
  继续仅返回纯表名。

* 新方法  :meth:`.PGInspector.get_foreign_table_names`  添加，它将返回PostgreSQL模式表中专门标记为"foreign"的表的名称。

此更改涉及在查询``pg_class.relkind``时添加“m”和“f”列表限定符，但是对于正在生产中运行0.9的任何人来说，此更改是新的，以避免任何向后不兼容的惊喜。

  :ticket:`2891`  

.. _change_3264:

MySQL内部“没有此表”异常未传递到事件处理程序
----------------------------------------------------------

MySQL方言现在将禁用由于它内部使用的语句触发: meth:`_events.ConnectionEvents.handle_error`事件来检测表是否存在或不存在。这是通过执行选项“skip_user_error_events”实现的，该选项在该执行的范围内禁用了处理错误处理程序。因此，重写异常的用户代码不必担心MySQL方言或其他偶尔需要捕获SQLAlchemy特定异常的方言。


改变了MySQL-Connector的“raise_on_warnings”默认值
-------------------------------------------------------

将MySQL-Connector的“raise_on_warnings”的默认值更改为False。 这个
由于某种原因设置为True。 不幸地，“缓冲区”标志必须保持为True，因为
MySQL连接器不允许关闭游标，除非已完全提取所有结果。


.. _bug_3186:

MySQL布尔符号“true”、“false”再次有效
------------------------------------------------

0.9中IS/IS NOT运算符以及布尔类型中的所有Boolean类型的彻底翻新
在  :ticket:`2682`  中导致MySQL方言无法在“IS”/“IS NOT”的上下文中使用“真”和“假”符号。 显然，即使MySQL没有“boolean”类型，但它在使用“true”和“false”符号时支持IS / IS NOT，尽管这些符号在其他地方与“1”和“0”同义（并且IS / IS NOT不能与数字一起使用）。

因此，这里的更改是MySQL方言仍然是“非本机布尔类型”，但  :func:`.true` .false` 符号再次生成关键字“true”和“false”，因此像``column.is_(true())``表达式在MySQL上再次有效。

  :ticket:`3186`  

.. _change_3263:

match（）运算符现在返回与MySQL的浮点返回值兼容的MatchType

列操作的  :meth:`.ColumnOperators.match`  表达式的返回类型现在是一个名为  :class:` .MatchType` .Boolean`的一个子类，可以被方言截获，以便在SQL执行时产生不同的结果类型。

现在类似下面的代码将正确地运行，并在MySQL上返回浮点数：

    >>> connection.execute(
    ...     select(
    ...         [
    ...             matchtable.c.title.match("Agile Ruby Programming").label("ruby"),
    ...             matchtable.c.title.match("Dive Python").label("python"),
    ...             matchtable.c.title,
    ...         ]
    ...     ).order_by(matchtable.c.id)
    ... )
    [
        (2.0, 0.0, 'Agile Web Development with Ruby On Rails'),
        (0.0, 2.0, 'Dive Into Python'),
        (2.0, 0.0, "Programming Matz's Ruby"),
        (0.0, 0.0, 'The Definitive Guide to Django'),
        (0.0, 1.0, 'Python in a Nutshell')
    ]


  :ticket:`3263`  

.. _change_2984:

Drizzle方言现在是一个外部方言
-------------------------------------

“Drizzle <https://www.drizzle.org/>”的方言现在是一个外部方言，可在https://bitbucket.org/zzzeek/sqlalchemy-drizzle上获得。这个方言是在SQLAlchemy能够很好地适应第三方方言之前添加的；今后，不在“普遍使用”类别中的所有数据库都是第三方方言。方言的实现没有改变，仍然基于SQLAlchemy中的MySQL+MySQLdb方言。该方言尚未发布，处于“attic”状态；但是，如果有人想继续完善它，那么它通过了大部分测试，一般都能正常工作。

方言改进和变化 - SQLite
=========================================

SQLite有名和无名UNIQUE和FOREIGN KEY约束将检查和反射
---------------------------------------------------------------

现在，SQLite上的UNIQUE和FOREIGN KEY约束在有名和无名情况下都被完全反映出来。先前，忽略了外键名称，并跳过了没有名称的独特约束。特别是这将有助于Alembic的新SQLite迁移功能。

为了实现这一点，对于外键和唯一约束，将PRAGMA foreign_keys、index_list和index_info的结果与CREATE TABLE语句的正则表达式解析相结合，形成约束名称的完整图像，并区分作为唯一约束和未命名INDEX创建的UNIQUE约束。

  :ticket:`3244`  

  :ticket:`3261`  

方言改进和变化 - SQL Server
=============================================

.. _change_3182:

使用基于主机名的SQL Server连接需要PyODBC驱动程序名称
---------------------------------------------------------------

使用PyODBC连接到SQL Server使用DSN-less连接，例如具有显式主机名的连接，现在需要驱动程序名称--SQLAlchemy将不再尝试猜测默认值：

    engine = create_engine(
        "mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=SQL+Server+Native+Client+10.0"
    )

SQLAlchemy在Windows上以前硬编码的默认值“SQL Server”已经过时，SQLAlchemy无法通过任务基于操作系统/驱动程序检测来猜测最佳驱动程序。在使用ODBC时，始终首选使用DSN以完全避免此问题。

 $& 

SQL Server 2012大文本/二进制类型呈现为VARCHAR、NVARCHAR、VARBINARY
--------------------------------------------------------------------------------

对于SQL Server 2012及更高版本，  :class:`_expression.TextClause` .Unicode Text` 和  :class:`.Large Binary` 。

方言改进和变化 - Oracle
=========================================

.. _change_3220:

改进了Oracle中CTE的支持
------------------------

Oracle中的CTE支持已得到修复，还有一个新功能  :meth:`_expression.CTE.with_suffixes`  ，可以帮助Oracle的特殊指令：

    included_parts = (
        select([part.c.sub_part, part.c.part, part.c.quantity])
        .where(part.c.part == "p1")
        .cte(name="included_parts", recursive=True)
        .suffix_with(
            "search depth first by part set ord1",
            "cycle part set y_cycle to 1 default 0",
            dialect="oracle",
        )
    )

 $& 

DDL的新Oracle关键字
--------------------------

关键字诸如COMPRESS、ON COMMIT、BITMAP：

  :ref:`oracle_table_options` 

  :ref:`oracle_index_options` 