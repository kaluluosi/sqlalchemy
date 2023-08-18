========================
SQLAlchemy 1.2有哪些新特性?
========================

.. admonition:: 关于本文档

    本文档介绍的是SQLAlchemy 1.1版本和SQLAlchemy 1.2版本间的变更。

介绍
====

该指南介绍了SQLAlchemy1.2版本中的新功能，并记录了对从SQLAlchemy 1.1迁移应用程序的用户产生影响的变更。

请仔细查看可能影响行为的变化部分。

平台支持
========

针对Python 2.7及以上版本
-------------------------

SQLAlchemy 1.2现在将最低Python版本移动到2.7，不再支持2.6。预计会将新的语言特性合并到1.2系列中，而这些特性在Python 2.6中不支持。对于Python 3的支持，SQLAlchemy目前在版本3.5和3.6上进行了测试。

ORM中的新特性和改进
=====================

.. _change_3954:

“baked”贪婪加载现在是默认的懒加载方式
----------------------------------------------

允许构造所谓的:BakedQuery对象的sqlalchemy.ext.baked扩展在1.0系列中首次引入，该对象生成与表示查询结构的缓存键一起的   :class:`_query.Query`  对象。然后将此缓存键链接到生成的字符串SQL语句，以便于后面使用相同结构的 :class:` .BakedQuery`将绕过构建 :class:`_query.Query` 对象的所有开销，其中包括构建核心 :func:`_expression.select` 对象以及将 :func:`_expression.select` 编译成字符串，很大程度上消除了通常与构造和发出ORM  :class:`_query.Query` 对象相关的函数调用开销。

在ORM生成“lazy”查询的情况下，如默认的“选择”关系加载程序策略的  :func:`_orm.relationship` .BakedQuery` 。这将允许应用程序在使用惰性加载查询加载集合和相关对象的范围内大量减少函数调用。以前，这一功能通过使用全局API方法或使用“baked_select”策略在1.0和1.1中可用，现在是此行为的唯一实现。该功能也已得到改进，使得缓存仍然可以针对在惰性加载之后对具有其他加载器选项在作用的对象进行缓存。
 
可以使用  :paramref:`_orm.relationship.bake_queries`  标志在每个关联基础上禁用缓存行为，该标志对于非常异常的情况如关联使用与缓存不兼容的自定义   :class:` _query.Query`  实现等，这是可用的。

  :ticket:`3954`  

.. _change_3944:

新的"selectin"贪婪加载，使用IN全部加载集合
--------------------------------------------------

添加了一个名为"selectin"加载的新的贪婪加载，它在许多方面类似于"subquery"加载，但生成一个更简单的SQL语句，可以缓存，并且效率更高。

    给定查询如下:

    q = (
        session.query(User)
        .filter(User.name.like("%ed%"))
        .options(subqueryload(User.addresses))
    )

则产生的SQL将会是针对``User``查询，然后是针对``User.addresses``的子查询（请注意也列出了参数）：

.. sourcecode:: sql

    SELECT users.id AS users_id, users.name AS users_name
    FROM users
    WHERE users.name LIKE ?
    ('%ed%',)

    SELECT addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           addresses.email_address AS addresses_email_address,
           anon_1.users_id AS anon_1_users_id
    FROM (SELECT users.id AS users_id
    FROM users
    WHERE users.name LIKE ?) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id
    ('%ed%',)

对于"selectin"贪婪加载，我们得到了一个SELECT，该SELECT引用了父查询中加载的实际主键值：

    q = (
        session.query(User)
        .filter(User.name.like("%ed%"))
        .options(selectinload(User.addresses))
    )

则产生的SQL为：

.. sourcecode:: sql

    SELECT users.id AS users_id, users.name AS users_name
    FROM users
    WHERE users.name LIKE ?
    ('%ed%',)

    SELECT users_1.id AS users_1_id,
           addresses.id AS addresses_id,
           addresses.user_id AS addresses_user_id,
           addresses.email_address AS addresses_email_address
    FROM users AS users_1
    JOIN addresses ON users_1.id = addresses.user_id
    WHERE users_1.id IN (?, ?)
    ORDER BY users_1.id
    (1, 3)

上面的SELECT语句具有以下优点：

* 它不使用子查询，只使用INNER JOIN，因此对于像MySQL这样不喜欢子查询的数据库来说，它的性能要好得多。

* 它的结构与原始查询无关; 与新的  :ref:`扩展IN参数系统<change_3953>` 结合使用，我们在大多数情况下可以使用"baked"查询来缓存字符串SQL，从而显著减少每个查询开销。

* 因为查询仅针对给定的一组主键标识符进行提取，所以"选择"贪婪加载与  :meth:`_query.Query.yield_per`  兼容，以操作SELECT结果的一次; 只要数据库驱动程序允许多个同时工作的游标（SQLite，PostgreSQL; **不是** MySQL驱动程序或SQL Server ODBC驱动程序）。Join Eager Loading和Subquery Eager Loading都不兼容  :meth:` _query.Query.yield_per`  。

"选择"贪婪加载的缺点是可能会产生大量的SQL查询，具有大量的IN参数列表。 IN参数本身的列表被分组成500个一组，因此超过500个导出对象的结果集将有更多的“SELECT IN”查询跟随。另外，对复合主键的支持取决于数据库使用元组是否兼容IN，例如``(table.column_one, table_column_two) IN ((?, ?), (?, ?) (?, ?))`` 。目前，已知PostgreSQL和MySQL与此语法兼容，而SQLite则不兼容。


.. seealso::

      :ref:`selectin_eager_loading` 

  :ticket:`3944`  

.. _change_3948:

“selectin”分表加载，使用单独的IN查询加载子类
--------------------------------------------------------

与刚描述的"selectin"关系加载功能类似的是"selectin"多态式加载。这是一种面向通过连接贪婪加载的基础实体的多态式加载功能，它允许在不增加对子类的复杂的JOIN操作的情况下对基础实体进行加载，但是需要使用额外的SELECT语句加载其他子类的属性::

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import selectin_polymorphic

    >>> query = session.query(Employee).options(
    ...     selectin_polymorphic(Employee, [Manager, Engineer])
    ... )

    >>> query.all()
    {execsql}SELECT
        employee.id AS employee_id,
        employee.name AS employee_name,
        employee.type AS employee_type
    FROM employee
    ()

    SELECT
        engineer.id AS engineer_id,
        employee.id AS employee_id,
        employee.type AS employee_type,
        engineer.engineer_name AS engineer_engineer_name
    FROM employee JOIN engineer ON employee.id = engineer.id
    WHERE employee.id IN (?, ?) ORDER BY employee.id
    (1, 2)

    SELECT
        manager.id AS manager_id,
        employee.id AS employee_id,
        employee.type AS employee_type,
        manager.manager_name AS manager_manager_name
    FROM employee JOIN manager ON employee.id = manager.id
    WHERE employee.id IN (?) ORDER BY employee.id
    (3,)

.. seealso::

      :ref:`polymorphic_selectin` 

  :ticket:`3948`  

.. _change_3058:

ORM属性可以接收即席SQL表达式
---------------------------------

添加了一个新的ORM属性类型  :func:`_orm.query_expression` ，它类似于  :func:` _orm.延迟` ，但其SQL表达式是使用一个新的选项  :func:`_orm.with_expression` ` None``::

    from sqlalchemy.orm import query_expression
    from sqlalchemy.orm import with_expression


    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        x = Column(Integer)
        y = Column(Integer)

        # 正常情况下为None...
        expr = query_expression()


    # 但是让我们给它赋值x + y
    a1 = session.query(A).options(with_expression(A.expr, A.x + A.y)).first()
    print(a1.expr)

.. seealso::

      :ref:`mapper_querytime_expression` 

  :ticket:`3058`  

.. _change_orm_959:

ORM支持多表删除
----------------------------

ORM  :meth:`_query.Query.delete`  方法支持多个表的DELETE的标准，就像在   :ref:` change_959`  中介绍的那样。该功能的工作方式与UPDATE的多表标准相同，该过程最早出现在0.8版本中，并在   :ref:`change_orm_2365`  中描述。

以下是DELETE的示例，使用了FROM子句（具体取决于后端）与``SomeOtherEntity``使用``SomeEntity``的ID作为参考::

    query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
        SomeOtherEntity.foo == "bar"
    ).delete()

.. seealso::

      :ref:`change_959` 

  :ticket:`959`  

.. _change_3229:

混合类型，复合类型（hybrids, composites）支持批量更新
-------------------------------------------------

在  :meth:`_query.Query.update`  中使用时，混合属性（例如  :mod:` sqlalchemy.ext.hybrid` ）以及复合属性（  :ref:`mapper_composite` ）现在支持在UPDATE语句的SET子句中使用。对于混合类型，可以直接使用简单的表达式，或使用新的修饰符  :meth:` .hybrid_property.update_expression`  将值分解为多个列/表达式:

    class Person(Base):
        # ...

        first_name = Column(String(10))
        last_name = Column(String(10))

        @hybrid.hybrid_property
        def name(self):
            return self.first_name + " " + self.last_name

        @name.expression
        def name(cls):
            return func.concat(cls.first_name, " ", cls.last_name)

        @name.update_expression
        def name(cls, value):
            f, l = value.split(" ", 1)
            return [(cls.first_name, f), (cls.last_name, l)]

上面的UPDATE可以用以下方式渲染：

    session.query(Person).filter(Person.id == 5).update({Person.name: "Dr. No"})

在混合类型之前，如果属性被设置为软删除或者 Null，修饰符  :meth:`.hybrid_property.update_expression`   以及对应的ORM事件已经定义好了，例如，用户软删除情况下的查询。

类似的功能在复合属性上也是可用的，复合值将被分解成其单个列以进行批量UPDATE：

    session.query(Vertex).update({Edge.start: Point(3, 4)})

.. seealso::

      :ref:`hybrid_bulk_update` 

.. _change_3911_3912:

hybrid属性支持在子类之间的重用，@getter可重定义
-----------------------------------------------------

  :class:`sqlalchemy.ext.hybrid.hybrid_property` ` @setter``、``@expression``等的定义，并提供了一个``@getter``变异器，以便可以在多个子类或其他 类中重新使用特定的hybrid。现在，这类似于标准Python中的``@property``的行为:

    class FirstNameOnly(Base):
        # ...

        first_name = Column(String)

        @hybrid_property
        def name(self):
            return self.first_name

        @name.setter
        def name(self, value):
            self.first_name = value


    class FirstNameLastName(FirstNameOnly):
        # ...

        last_name = Column(String)

        @FirstNameOnly.name.getter
        def name(self):
            return self.first_name + " " + self.last_name

        @name.setter
        def name(self, value):
            self.first_name, self.last_name = value.split(" ", maxsplit=1)

        @name.expression
        def name(cls):
            return func.concat(cls.first_name, " ", cls.last_name)

上面的``FirstNameOnly.name``hybrid在子类中受到引用，以便专门将其重新用于新的子类。这是通过在每次调用``@getter``、``@setter``以及所有其他变异器方法（如``@expression``）时将混合对象复制到新对象中实现的，每个新对象留下先前混合的定义。以前，像``@setter``这样的方法会在现有混合里原地修改混合，从而干扰了超类上的定义。

.. 注意::请务必阅读  :ref:`hybrid_reuse_subclass` .hybrid_property.expression` 和  :meth:`.hybrid_property.comparator`  ，在某些情况下可能需要一个特殊的限定符  :attr:` .hybrid_property.overrides` ，以避免与 :class:`.QueryableAttribute` 产生名称冲突。

.. 注意::``@hybrid_property``中的此更改意味着，当向``@hybrid_property``添加setter和其他状态时，**方法必须保留原始混合的名称**，否则具有附加状态的新混合将以不匹配的名称存在于类中。这是标准Python的``@property``构造所采取的相同行为。

  :ticket:`3911`  

  :ticket:`3912`  

.. _change_3896_event:

新的bulk_replace事件
----------------------

为适应   :ref:`change_3896_validates`  中描述的验证用例，添加了  :meth:` .AttributeEvents.bulk_replace`  方法，该方法与  :meth:`.AttributeEvents.append`  和  :meth:` .AttributeEvents.remove`  事件一起使用。在“追加”和“删除”之前调用了“bulk_replace”，以便修改集合以匹配现有集合。之后，按照之前的行为，单个项目附加到新的目标集合中，对新的集合执行“附加”事件。本示例同时显示了“bulk_replace”和“append”，如果使用集合分配，则“append”将接收已由“bulk_replace”处理的对象作为输入。：attr:`~.attributes.OP_BULK_REPLACE`符号可用于确定此“追加”事件是否为批量替换过程的第二部分

    from sqlalchemy.orm.attributes import OP_BULK_REPLACE


    @event.listens_for(SomeObject.collection, "bulk_replace")
    def process_collection(target, values, initiator):
        values[:] = [_make_value(value) for value in values]


    @event.listens_for(SomeObject.collection, "append", retval=True)
    def process_collection(target, value, initiator):
        # make sure bulk_replace didn't already do it
        if initiator is None or initiator.op is not OP_BULK_REPLACE:
            return _make_value(value)
        else:
            return value

  :ticket:`3896`  

.. _change_3303:

新增：SQLAlchemy.ext.Mutable的修改事件处理程序
----------------------------------------------

新增事件处理程序  :meth:`.AttributeEvents.modified`  ，该处理程序与  :mod:` sqlalchemy.ext.mutable`  扩展从中调用  :func:`.attributes.flag_modified` ` 下面的in-place更改被用于``。例如，在``.data``字典发生就地更改时，会触发此事件处理程序。

    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy.ext.mutable import MutableDict
    from sqlalchemy import event

    Base = declarative_base()


    class MyDataClass(Base):
        __tablename__ = "my_data"
        id = Column(Integer, primary_key=True)
        data = Column(MutableDict.as_mutable(JSONEncodedDict))


    @event.listens_for(MyDataClass.data, "modified")
    def modified_json(instance):
        print("json value modified:", instance.data)

上面的事件处理程序将在``.data``字典进行就地更改时被触发。

  :ticket:`3303`  

.. _change_3769:

AssociationProxy any(), has(), contains()支持链接到联合代理
---------------------------------------------------------------

  :meth:`.AssociationProxy.any`  、  :meth:` .AssociationProxy.has`  和  :meth:`.AssociationProxy.contains`  比较方法现在支持链接到本身也是   :class:` .AssociationProxy` `A.b_values``代理是链接到``AtoB.bvalue``，它本身也是一个链接到``B``的 :ref:`mapper_association_proxy` 的代理：

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)

        b_values = association_proxy("atob", "b_value")
        c_values = association_proxy("atob", "c_value")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))
        value = Column(String)

        c = relationship("C")


    class C(Base):
        __tablename__ = "c"
        id = Column(Integer, primary_key=True)
        b_id = Column(ForeignKey("b.id"))
        value = Column(String)


    class AtoB(Base):
        __tablename__ = "atob"

        a_id = Column(ForeignKey("a.id"), primary_key=True)
        b_id = Column(ForeignKey("b.id"), primary_key=True)

        a = relationship("A", backref="atob")
        b = relationship("B", backref="atob")

        b_value = association_proxy("b", "value")
        c_value = association_proxy("b", "c")

我们可以使用  :meth:`.AssociationProxy.contains`  在` `A.b_values``上进行查询，查询时跨越两个代理``A.b_values``和``AtoB.b_value``：

.. sourcecode:: pycon+sql

    >>> s.query(A).filter(A.b_values.contains("hi")).all()
    {execsql}SELECT a.id AS a_id
    FROM a
    WHERE EXISTS (SELECT 1
    FROM atob
    WHERE a.id = atob.a_id AND (EXISTS (SELECT 1
    FROM b
    WHERE b.id = atob.b_id AND b.value = :value_1)))

我们可以使用  :meth:`.AssociationProxy.any`  在` `A.c_values``上进行查询，查询时跨越两个代理``A.c_values``和``AtoB.c_value``：


.. sourcecode:: pycon+sql

    >>> s.query(A).filter(A.c_values.any(value="x")).all()
    {execsql}SELECT a.id AS a_id
    FROM a
    WHERE EXISTS (SELECT 1
    FROM atob
    WHERE a.id = atob.a_id AND (EXISTS (SELECT 1
    FROM b
    WHERE b.id = atob.b_id AND (EXISTS (SELECT 1
    FROM c
    WHERE b.id = c.b_id AND c.value = :value_1)))))

  :ticket:`3769`  


.. _change_4137:

身份键增强支持分片
-------------------------

ORM使用的身份键结构现在包含一个额外的成员，以便来自不同上下文的两个相同的Principal Key可以共存于同一身份映射。在   :ref:`examples_sharding` ` WeatherLocation``，该类引用一个依赖于``WeatherReport``对象的``WeatherReport``对象，其中``WeatherReport ``类被映射到一个存储简单整数主键的表。

两个来自不同数据库的``WeatherReport``对象可能具有相同的主键值。现在，该示例说明了一个新的``identity_token``字段，以跟踪此差异，以便两个对象可以共存于同一身份映射中。

    tokyo = WeatherLocation("Asia", "Tokyo")
    newyork = WeatherLocation("North America", "New York")

    tokyo.reports.append(Report(80.0))
    newyork.reports.append(Report(75))

    sess = create_session()

    sess.add_all([tokyo, newyork, quito])

    sess.commit()

ORM文档使用简单的整数主键列扩展了这个示例，因此在两个不同的数据库上可以使用相同的Test表和城市ID。当每个Test和City行具有唯一的主键值时，该示例不同数据库的手动建立此冲突条件。

为了说明问题，假设有两个Test和City行，它们的主键在一个数据库中号称为ID 1和2，而在另一个数据库中号称为ID 2和1。

凭据构建(Originating shard tracking)已在详细成对的文档中进行了逐步的说明——   :ref:`examples_sharding` 。

    newyork_report = newyork.reports[0]
    tokyo_report = tokyo.reports[0]

    assert inspect(newyork_report).identity_key == (Report, (1,), "north_america")
    assert inspect(tokyo_report).identity_key == (Report, (1,), "asia")

    #表示源分片的标记直接可用

    assert inspect(newyork_report).identity_token == "north_america"
    assert inspect(tokyo_report).identity_token == "asia"

  :ticket:`4137`  


核心中的新特性和改进
======================

.. _change_4102:

布尔数据类型现在强制采用严格的真/假/空值
------------------------------------------------------

1.1中描述的更改：ref:`change_3730`的一个意外副作用，它修改了  :class:`.Boolean` .Boolean` 发送字符串“0”值的代码将在后端之间不一致地破坏。

解决这个问题的最终解决方案是**不支持布尔值和字符串值**，因此在1.2中，如果传递了非整数/True/False/None值，则会引发'TypeError' 。同时，仅接受整数值0和1。

为了适应希望具有更自由解释布尔值的应用程序，应该使用  :type:`.TypeDecorator`  。以下演示了一个配方，它允许先前的1.1  :class:` .Boolean`数据类型的“自由”行为：

    from sqlalchemy import Boolean
    from sqlalchemy import TypeDecorator


    class LiberalBoolean(TypeDecorator):
        impl = Boolean

        def process_bind_param(self, value, dialect):
            if value is not None:
                value = bool(int(value))
            return value

  :ticket:`4102`  

.. _change_3919:

连接池现在增加悲观断线检测（Pessimistic Disconnection Detection）
----------------------------------------------------------------

长期以来，在连接池文档中一直提供了在检查出一个检出的连接用于测试其存活状态的功能的概要。该文档中介绍了使用  :meth:`_events.ConnectionEvents.engine_connect`  引擎事件在检出的连接上发出简单语句的配方。在适当的方言的情况下，该配方的功能现在已经添加到连接池本身中，在与任何其他操作池一起使用时检查每个连接的新参数  :paramref:` _sa.create_engine.pool_pre_ping`  。每个检出的连接在返回之前将被测试以进行新鲜测试。

    engine = create_engine("mysql+pymysql://", pool_pre_ping=True)

虽然“pre-ping”方法会在连接池检出时添加一些小延迟，但对于通常以事务为导向（包括大多数ORM应用程序）的典型应用程序来说，这种开销是很小的，并且消除了获取可能错误的连接的问题，从而需要应用程序放弃或重试操作。该功能**不**适合在进行事务或SQL操作时中断的连接。如果应用程序还需要从这些错误中进行恢复，它需要进行自己的操作重试逻辑。

.. seealso::

      :ref:`pool_disconnects_pessimistic` 

  :ticket:`3919`  

.. _change_3907:

IN / NOT IN运算符的空集合行为现在可配置；默认表达式简化表达式 ``column.in_([])`` 假定是 false，
现在默认情况下会产生表达式 ``1！=1``，
而不是 ``column != column``。
这将 **更改结果** 与 SQL 表达式或列进行比较时的查询，
当将其与空集比较时求值为 NULL 的列，生成一个 boolean 值 false 或者 true
(对于 NOT IN)，而不是 NULL。
这种情况下会发出警告。
可以使用  :paramref:`_sa.create_engine.empty_in_strategy`  参数切换回旧的行为。
在 SQL 中，IN 和 NOT IN 运算符不支持与显式空集合的值相比较；
即，这种语法是非法的：

.. sourcecode:: sql

    mycolumn IN ()

为了避免这种情况，SQLAlchemy 和其他数据库库会检测到此情况，
并渲染另一种表达式，该表达式计算为 False；
或者在 NOT IN 的情况下，计算为 true；
基于这样一种理论：“col IN ()”总是 false，因为 "empty set" 中什么都没有。
为了在跨数据库的情况下生成可移植并且适用于 WHERE 子句的 false/true 常量，
通常使用简单的自反证明，例如 ``1 != 1``（对于 false），
``1 = 1``（对于 true）；通常情况下，
作为 WHERE 子句目标的简单常量“0”或“1”并不起作用。

在 SQLAlchemy 的早期，它也从这种方法开始，
但很快就有了这样的理论：
如果“列”为空，SQL 表达式“column IN ()”将不会返回 false；
相反，表达式将产生 NULL，因为 "NULL" 的意思是 "未知"，
而 SQL 中对 NULL 的比较通常会产生 NULL。

为了模拟这个结果，SQLAlchemy 改为使用``expr != expr``这个表达式，
而不是使用一个固定的值 `1 != 1`，对于空的 "IN" 和 `expr = expr` 对于空的 "NOT IN"。
也就是说，我们使用表达式左边的实际操作数，而不是使用一个固定的值。
如果表达式的左操作数为 NULL，则整个比较也会获得 NULL 结果，而不是 false 或 true。

不幸的是，用户最终抱怨这种表达式会对某些查询规划程序产生严重的性能影响。
这时添加了一个警告，当遇到空的 IN 表达式时会发出警告，
鼓励用户避免生成空的 IN 谓词的代码，
因为通常它们可以被安全地省略。
但是，在从输入变量动态构建的查询的情况下，
这对于传入的值集合可能为空的情况非常繁琐。

近几个月来，开始质疑了该决策的最初假设。
表达式 “NULL IN ()” 应返回 NULL 只是理论上的，
由于数据库不支持该语法，因此无法测试。
然而，正如现在，您可以通过模拟空集合来询问关系数据库会返回什么值，
如下所示：

.. sourcecode:: sql

    SELECT NULL IN (SELECT 1 WHERE 1 != 1)

使用上面的测试，我们可以看到数据库本身不能达成一致的结论。
大多数人认为 PostgreSQL 是最正确的数据库，
因为即使 "NULL" 代表“未知”，“empty set”也意味着什么都不存在，
包括所有未知值。另一方面，MySQL 和 MariaDB 对上述表达式返回 NULL，
默认使用“所有与 NULL 的比较都会返回 NULL”的更常见的行为。

SQLAlchemy 的 SQL 架构比初期要复杂得多，
因此现在可以允许在 SQL 字符串编译时调用任一行为。
以前，在构建  :meth:`.ColumnOperators.in_`  或  :meth:` .ColumnOperators.notin_`  操作符进行构造时，
将转换为比较表达式。转换到比较表达式
现在由方言本身指示去调用，即静态 ``1 != 1`` 比较或动态 ``expr != expr`` 比较。
默认已 **更改** 为静态比较，
因为这与 PostgreSQL 的行为是相同的，
这也是大多数用户偏爱的行为。将影响用 null 表达式与空集进行比较的查询的结果，
特别是一个查询，该查询正在查询否定 `where(~null_expr.in_([]))`，
因为现在这将计算为 true 而不是 NULL。

现在可以使用标志  :paramref:`_sa.create_engine.empty_in_strategy`  来控制行为，
其默认设置为 ``"static"``，但也可以设置为 ``"dynamic"`` 或 ``"dynamic_warn"``。
其中``"dynamic_warn"`` 设置等效于以前的行为，即同时发出``expr != expr``和性能警告。
但是，预计大多数用户都会赞赏“static”默认设置。

  :ticket:`3907`  

.. _change_3953:

通过缓存语句引入的后期扩展 IN 参数集允许 IN 表达式
---------------------------------------------------------

添加了名为“expanding”的新类型   :func:`.bindparam` 。
这用于 IN 表达式，其中将元素列表渲染为语句执行时的单个参数，
而不是在语句编译时。这允许将单个绑定参数名称链接到具有多个元素的 IN 表达式，
也允许使用查询缓存在 IN 表达式中使用相关特性的“select”和“polymorphic in”loading。

新功能允许使用烘烤查询扩展，以减少调用开销：

.. sourcecode:: python

    stmt = select([table]).where(table.c.col.in_(bindparam("foo", expanding=True)))
    conn.execute(stmt, {"foo": [1, 2, 3]})

应在 1.2 系列中视为 **实验功能**。

  :ticket:`3953`  

.. _change_3999:

比较运算符的优先级已被降低
----------------------------

比较运算符的优先级，例如 IN、LIKE、等于、IS、MATCH 和其他比较运算符的优先级
已被降低为一级。当组合比较运算符时将生成更多括号。

例如，``(column("q") == null())！=（column("y") == null()）`` 现在会生成 ```(q IS NULL)！= (y IS NULL)``，
而不是 ``q IS NULL！= y IS NULL``。


  :ticket:`3999`  

.. _change_1546:

对表、列SQL注释提供了支持，包括DDL、reflection
-------------------------------------------------

Core 支持与表和列相关联的字符串注释。
这些是通过  :paramref:`_schema.Table.comment`  和  :paramref:` _schema.Column.comment`  参数指定的：

Table(
    "my_table",
    metadata,
    Column("q", Integer, comment="the Q value"),
    comment="my Q table",
)

上面，将在创建表时适当地渲染 DDL，
以将上述注释与架构中的表/列相关联。
在使用  :meth:`_reflection.Inspector.get_columns`  自动加载的上述表或反射时，注释也将包含在内。
表注释也可以使用  :meth:`_reflection.Inspector.get_table_comment`  方法单独使用。

当前支持的后端包括 MySQL、PostgreSQL 和 Oracle。

  :ticket:`1546`  

.. _change_959:

DELETE 支持跨多个表的标准
----------------------------

  :class:`_expression.Delete`  现在支持对支持它的引擎的多个表条件（目前这些是 PostgreSQL、MySQL 和 Microsoft SQL Server）进行实现，
这个特性在 0.7 和 0.8 系列中首次引入，和 UPDATE 中类似。

给定以下语句：

stmt = (
    users.delete()
    .where(users.c.id == addresses.c.id)
    .where(addresses.c.email_address.startswith("ed%"))
)
conn.execute(stmt)

PostgreSQL 后端对上述语句生成的 SQL 如下：

.. sourcecode:: sql

    DELETE FROM users USING addresses
    WHERE users.id = addresses.id
    AND (addresses.email_address LIKE %(email_address_1)s || '%%')

.. seealso::

      :ref:`tutorial_multi_table_deletes` 

  :ticket:`959`  

.. _change_2694:

为 startswith()、endswith() 添加了一个新的“autoescape”选项
-------------------------------------------------------

对于 autoescape 设置为 True 的  :meth:`.ColumnOperators.startswith` 、  :meth:` .ColumnOperators.endswith`   以及  :meth:`.ColumnOperators.contains` ，
此参数会自动转义所有出现的 ``%``、``_``，使用正斜杠 ``/`` 作为转义字符，默认情况下
转义字符本身也被转义。使用正斜杠是为了避免像 PostgreSQL 的 ``standard_confirming_strings`` 设置那样的设置发生冲突；
自从 PostgreSQL 9.1 后，其默认值发生了更改，而 MySQL 的 ``NO_BACKSLASH_ESCAPES`` 设置也是如此。

.. note:: 该特性已在 1.2.0b2 中的初始实现改为作为布尔值传递，而不是指定要用作转义字符的特定字符。

例如：

column("x").startswith("total%score", autoescape=True)

例如，如果参数的值包含反斜杠：

column("x").startswith("total/score", autoescape=True)

将以相同的方式呈现，参数值如下：

x LIKE :x_1 || '%' ESCAPE '/'

其中 "x_1" 参数的值为 ``'total/%score'``。

  :ticket:`2694`  

.. _change_floats_12:

增强“浮点”数据类型的强类型化
--------------------------

一系列更改允许使用   :class:`.Float`  数据类型更强地将其连接到 Python 浮点值，而不是更通用的 
  :class:`.Numeric`  类型。
这些更改与确保 Python 浮点值不会错误地强制转换为 ``Decimal()``，
如果应用程序使用普通浮点值，则结果类型将强制转换为 ``float``。

* 传递给 SQL 表达式的纯 Python“float”值现在会拉入具有类型   :class:`.Float`  的文字参数；
  先前的类型是   :class:`.Numeric` ，带有默认标志“asdecimal=True”，这意味着结果类型将强制转换为 ` `Decimal()``。
  特别是，这将发出 SQLite 上的令人困惑的警告：

        float_value = connection.scalar(
            select([literal(4.56)])  # the "BindParameter" will now be
            # Float, not Numeric(asdecimal=True)
        )

*   :class:`.Numeric` 、  :class:` .Float`  和   :class:`.Integer`  之间的数学运算将在其表达式类型中保留   :class:` .Numeric`  或   :class:`.Float`  类型，
  包括 ``asdecimal`` 标志以及类型是否应为   :class:`.Float` 。
  
  .. code-block:: python

        # asdecimal 标志保持不变
        expr = column("a", Integer) * column("b", Numeric(asdecimal=False))
        assert expr.type.asdecimal == False

        # Numeric 的 Float 子类保持不变
        expr = column("a", Integer) * column("b", Float())
        assert isinstance(expr.type, Float)

* 如果 DBAPI 已知支持本地 ``Decimal()`` 模式，则   :class:`.Float`  数据类型将无条件地将 ` `float()`` 处理器应用于结果值。
  有些后端不始终保证浮点数返回为简单浮点数，而不是精度数字，例如 MySQL。

  :ticket:`4017`  

  :ticket:`4018`  

  :ticket:`4020`  

.. change_3249:

增加了 GROUPING SETS、CUBE 和 ROLLUP 的支持
-------------------------------------------

通过  :attr:`.func`  命名空间可用所有三个 GROUPING SETS、CUBE 和 ROLLUP；
在 CUBE 和 ROLLUP 的情况下，这些函数在以前的版本中已经工作，但在 GROUPING SETS 的情况下，
编译器添加了一个占位符，以允许空间存在。所有三个函数均在文档中命名：

.. sourcecode:: python

    from sqlalchemy import select, table, column, func, tuple_
    t = table("t", column("value"), column("x"), column("y"), column("z"), column("q"))
    stmt = select([func.sum(t.c.value)]).group_by(
        func.grouping_sets(
            tuple_(t.c.x, t.c.y),
            tuple_(t.c.z, t.c.q),
        )
    )
    print(stmt)

结果：

.. sourcecode:: python

    SELECT sum(t.value) AS sum_1
    FROM t GROUP BY GROUPING SETS((t.x, t.y), (t.z, t.q))

  :ticket:`3429`  

.. _change_4075:

在上下文默认生成器中，多值 INSERT 的参数助手允许 SET
------------------------------------------------------------

一个默认生成函数，例如   :ref:`context_default_functions`  中描述的那样，可以查看上下文参数相关的当前参数
通过  :attr:`.DefaultExecutionContext.current_parameters`  属性来指定。然而，在   :class:` _expression.Insert` 
构造指定多个 VALUES 子句时，执行用户定义的函数会多次调用，每次对应于一个参数集，在比较之前执行对现有集合的操作，
但却无法知道  :attr:`.DefaultExecutionContext.current_parameters`  中哪些键值适用于该列。
添加了一个新功能  :meth:`.DefaultExecutionContext.get_current_parameters` ，
它包括关键字参数：  :paramref:`.DefaultExecutionContext.get_current_parameters.isolate_multiinsert_groups`  
默认为 ``True``，在执行操作之前执行了一些额外的操作以确保所支持的命名空间适用于当前 VALUES 子句的范围。

例如：

    def mydefault(context):
        return context.get_current_parameters()["counter"] + 12


    mytable = Table(
        "mytable",
        metadata_obj,
        Column("counter", Integer),
        Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
    )

    stmt = mytable.insert().values([{"counter": 5}, {"counter": 18}, {"counter": 20}])

    conn.execute(stmt)

  :ticket:`4075`  

ORM 的重要行为变化
==================

.. _change_3934:

after_rollback() Session 事件现在在对象过期之前发出
-----------------------------------------------------

  :meth:`.SessionEvents.after_rollback`   事件现在具有在对象过期（例如“快照删除”）之前获取属性状态的功能。
这使得该事件与  :meth:`.SessionEvents.after_commit`  事件的行为一致，
后者在删除“快照”之前发出。

例如：

    sess = Session()

    user = sess.query(User).filter_by(name="x").first()


    @event.listens_for(sess, "after_rollback")
    def after_rollback(session):
        # 'user.name' 现在存在，假设它已经被加载了。在此之前，这将引发
        # 尝试发出惰性加载时的异常。
        print("user name: %s" % user.name)


    @event.listens_for(sess, "after_commit")
    def after_commit(session):
        # 'user.name' 现在存在，假设它已经被加载了。这是现有的行为。
        print("user name: %s" % user.name)


    if should_rollback:
        sess.rollback()
    else:
        sess.commit()

请注意，  :class:`.Session`  仍然不允许在此事件中发送 SQL；
这意味着未加载的属性仍然无法在事件范围内加载。

  :ticket:`3934`  

.. _change_3891:

使用 ``select_from()`` 的单表继承问题已得到解决
------------------------------------------------------

  :meth:`_query.Query.select_from`   现在会在生成 SQL 时尊重单表继承列鉴别器；
以前，只有查询列列表中的表达式会被考虑在内。
假设 ``Manager`` 是 ``Employee`` 的子类，则以下查询：

    sess.query(Manager.id)

将生成 SQL：

.. sourcecode:: sql

    SELECT employee.id FROM employee WHERE employee.type IN ('manager')

但是，如果仅在  :meth:`_query.Query.select_from`  中指定了` `Manager`` 而没有在列列表中指定，
则不会添加鉴别器：

    sess.query(func.count(1)).select_from(Manager)

将生成以下 SQL：

.. sourcecode:: sql

    SELECT count(1) FROM employee

  :ticket:`3891`  

.. _change_3913:

替换之前，以前的收藏品不再发生变异
------------------------------------

在替换属性会等待新值实际插入之前，collections 也不再更改。

例如：

a1, a2, a3 = Address("a1"), Address("a2"), Address("a3")
user.addresses = [a1, a2]

previous_collection = user.addresses

# 将集合替换为新集合
user.addresses = [a2, a3]

previous_collection

a1 不再在 previous_collection 中。

  :ticket:`3913`  

.. _change_3896_validates:

使用 @validates 方法在批量集合设置之前接收所有值
-------------------------------------------------------------------

使用 ``@validates`` 方法的方法现在在进行“批量设置”操作时将接收到集合中所有元素的副本，
而不是要与现有集合进行比较之前仅接收添加的元素。

上面以“字典转换为 B 的实例”为例，这会将字典转换为“B”实例。

可以使用“@validates”验证器将其用作集合附加，如下所示：

a1 = A()
a1.bs.append({"data": "b1"})

但是，集合分配将失败，因为 ORM 将假定传入的对象已经是 ``B`` 对象实例，因为它试图将其与现有对象成为比较，
而实际上这个过程是会执行集合附加操作的，这实际上是调用了验证器。

a1 = A()
a1.bs = [{"data": "b1"}]

修复后，  :meth:`_orm.relationship.post_update`  的列现在与具有  :paramref:` _schema.Column.onupdate`  值集的列交互更加正确。
如果插入对象具有列的显式值，则将在 UPDATE 期间重新命令该列，
因此不会覆盖“onupdate”规则：

class A(Base):

    @validates("bs")
    def convert_dict_to_b(self, key, value):
        return B(data=value["data"])


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id"))
        data = Column(String)

上面，我们可以像这样使用验证器，将字典转换为“B”实例，以在集合附加时进行转换：

a1 = A()

b1, b2 = B(data="one"), B(data="two")

a1.bs = [b1, b2]

然后，将集合替换为与第一个集合重叠的集合：

b3 = B(data="three")
a1.bs = [b2, b3]

以前，第二次赋值将只触发一次 ``A.validate_b`` 方法，对于 “b3” 对象。``b2`` 对象将被视为已经存在于集合中，
因此不进行验证。随着新行为的应用，``A.validate_b`` 现在会在传递到集合之前将 ``b2`` 和 ``b3`` 传递到 ``A.validate_b`` 中，
然后将继续传递给集合。因此，验证方法必须对该情况进行幂等性的处理。

.. seealso::

      :ref:`change_3896_event` 

  :ticket:`3896`  

.. _change_3753:

使用 flag_dirty() 将对象标记为“脏”状态，而不更改任何属性
---------------------------------------------------------------

如果使用   :func:`.attributes.flag_modified`  函数将非加载属性标记为已修改，则会引发异常：

a1 = A(data="adf")
s.add(a1)

s.flush()

# 过期，就像我们说的那样 s.commit()
s.expire(a1, "data")

# 将引发 InvalidRequestError
attributes.flag_modified(a1, "data")


因为如果在 flush 时属性仍然未存在，刷新过程通常也会失败。
要将对象标记为“修改”，而无需引用任何特定属性，
以便在以自定义事件处理程序为例的情况下参与刷写过程，
请使用新的   :func:`.attributes.flag_dirty`  函数：

    from sqlalchemy.orm import attributes

    attributes.flag_dirty(a1)

  :ticket:`3753`  

.. _change_3796:

scoped_session 中的“scope”关键字已被删除
-------------------------------------------------

一个非常古老且未记录的关键字参数``scope`` 已被删除：

    from sqlalchemy.orm import scoped_session

    Session = scoped_session(sessionmaker())

    session = Session(scope=None)

此关键字参数的目的是尝试允许变量“作用域”（``None`` 表示“无作用域”，
从而返回新的   :class:`.Session` ）。这个关键字从未被记录过，
现在如果遇到会引发 ``TypeError‍``。 尽管我们并不预期用户使用此关键字，
但如果用户在 Beta 测试期间报告与此相关的问题，则可以对其进行弃用。

  :ticket:`3796`  

.. _change_3471:

和 onupdate 一起使用 post_update 的精细调整
------------------------------------------------------

使用  :paramref:`_orm.relationship.post_update`  功能的关系现在会更好地与具有  :paramref:` _schema.Column.onupdate` 
值集的列交互。如果插入对象具有一个明确的列的值，则在 UPDATE 期间它将重新表述该列，以便“onupdate”规则不会覆盖它。

  :ticket:`3471`          __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        favorite_b_id = Column(ForeignKey("b.id", name="favorite_b_fk"))
        bs = relationship("B", primaryjoin="A.id == B.a_id")
        favorite_b = relationship(
            "B", primaryjoin="A.favorite_b_id == B.id", post_update=True
        )
        updated = Column(Integer, onupdate=my_onupdate_function)


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey("a.id", name="a_fk"))


    a1 = A()
    b1 = B()

    a1.bs.append(b1)
    a1.favorite_b = b1
    a1.updated = 5
    s.add(a1)
    s.flush()

以上，以前的行为是，一个UPDATE会在INSERT之后发出，从而触发"onupdate"并覆盖值"5"。现在，SQL的形式如下：

.. sourcecode:: sql

    INSERT INTO a (favorite_b_id, updated) VALUES (?, ?)
    (None, 5)
    INSERT INTO b (a_id) VALUES (?)
    (1,)
    UPDATE a SET favorite_b_id=?, updated=? WHERE a.id = ?
    (1, 5, 1)

此外，如果"updated"的值没有设置，则可以在``a1.updated``上正确获取新生成的值；以前在刷新或更新属性以允许生成值时的逻辑不会为后更新而发出。在此情况下，在更新中刷新时还会发出  :meth:`.InstanceEvents.refresh_flush`  事件。

  :ticket:`3471`  

  :ticket:`3472`  

.. _change_3496:

post_update与ORM版本控制相结合
-----------------------------------

在ORM版本控制中，"post_update"功能 ，即在针对特定relationship-bound外键的更改时发出UPDATE语句，以及通常针对目标行发出的INSERT / UPDATE / DELETE被支持。现在，这个UPDATE语句参与标记的版本号，即支持   :ref:`mapper_version_counter` 。

给定一个映射 ::

    class Node(Base):
        __tablename__ = "node"
        id = Column(Integer, primary_key=True)
        version_id = Column(Integer, default=0)
        parent_id = Column(ForeignKey("node.id"))
        favorite_node_id = Column(ForeignKey("node.id"))

        nodes = relationship("Node", primaryjoin=remote(parent_id) == id)
        favorite_node = relationship(
            "Node", primaryjoin=favorite_node_id == remote(id), post_update=True
        )

        __mapper_args__ = {"version_id_col": version_id}

将节点更新为与另一个节点相关联作为“favorite”现在会递增版本计数器，如当前版本所匹配的：：
 节点=节点（） 会话添加（节点）
 会话提交（）＃节点现在是版本＃1

 node =会话查询（Node）.get（node.id）
 节点。favorite_node = Node()
 session.commit()＃node现在是2.0版本
请注意，这意味着在响应其他属性更改发出UPDATE的对象和以下UPDATE中二次发出UPDATE作为post_update关系更改，将会为一个flush 会收到**两个版本计数器更新**。如果在当前flush中插入对象，则版本计数器不会再次增加，除非使用服务器端的版本控制方案。

现在在此处讨论post_update为UPDATE发出一个的原因  :ref:`faq_post_update_update` 。

.. seealso::

      :ref:`post_update` 

      :ref:`faq_post_update_update` 


  :ticket:`3496`  

重大行为变更 - 核心
=========================

.. _change_4063:

自定义运算符的键入行为已统一
-----------------------------------

用户可以使用  :meth:`运营商。op`  ` 函数即时制作运算符。以前对于针对此类运算符的表达式的打印行为不一致，也不可控。现在表达式对此操作的键入行为与左手表达式相同:: 

    column（“x”，types.DateTime）.op（“ - ％gt;”）（无）.type
    NullType（）

其他类型将使用左手类型作为返回类型的默认行为：：

    column（“x”，types.String（50））。op（“ - ％gt;”）（无）.type
    String（length = 50）

这些行为大多是偶然的，因此更改后的行为与第二种形式一起制作，即默认返回类型与左手表达式相同：： 

    column（“x”，types.DateTime）.op（“ - ％gt;”）（无）.type
    DateTime（）

由于大多数用户定义的运算符往往是“比较”运算符，通常是由PostgreSQL定义的许多特殊运算符之一，因此现在：“运营商。op.is_comparison”标志已被维修，遵循其文档化的行为，即包括 :class:`.Boolean` 在内的所有情况都允许返回类型为?

    column（“x”，types.String（50））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

    column（“x”，types.ARRAY（types.Integer））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

    column（“x”，types.JSON（））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

为了协助布尔比较运算符，添加了一个新的简写方法  :meth:`运营商。bool_op。`  此方法应优选即时制作布尔值运算符： 

.. sourcecode :: pycon + sql

    >>> print（column（“x”，types.Integer）.bool_op（“ - ％gt;”）（5））
    {printsql} x  - ％gt;：x_1


.. _change_3740:

缩小literal_column（）中的百分号现在有条件转义
--------------------------------------------------


现在，  :obj:`_expression.literal_column`  构造在特定情况下条件地转义百分号字符，具体取决于DBAPI是否使用敏感的百分符paramstyle或不使用该字符（例如'format'或'pyformat'）。 

以前，不可能生成声明单个百分号的 :obj:_expression.literal_column构造::

    >>> from sqlalchemy import literal_column
    >>> print（literal_column（“some％symbol”））
    {printsql} some％symbol

现在，对于未设置这些paramstyles的dialects（例如大多数MySQL dialects），百分号不受影响：：

    >>> from sqlalchemy import literal_column
    >>> print（literal_column（“ some％symbol”））
    {printsql}一些％符号{stop}
    >>> from sqlalchemy.dialects import mysql
    >>> print（literal_column（“ some％symbol”）.compile（dialect = mysql.dialect（）））
    {printsql}一些％符号{stop}

另外，针对  :meth:`.Operators.contains`  ，  :meth:` .ColumnOperators.startswith`  和  :meth:`.ColumnOperators.endswith`  等运算符的使用的双倍将仅在适当时发生。 

  :ticket:`3740`  


.. _change_3785:

列级别的COLLATE关键字现在引用了排序名称
--------------------------------------------------------------–

已经修复了在   :func:`_expression.collate`  and :meth:` .ColumnOperators.collate`所使用的，用于在语句级别提供ad-hoc列排序的约束条件的一个错误，在其中一个大小写敏感的名称没有引用时：:

    sel = select([table1.c.my_name]).where(table1.c.my_name.collate('...'))

现在渲染为：

.. sourcecode:: sql

    SELECT table1.my_name
    FROM table1
    WHERE table1.my_name COLLATE "..."

之前大小写敏感名称中的“fr_FR”未被引用。当前，手动引用标识符不会被检测到，因此必须调整手动引用标识符的应用程序。请注意，此更改不影响在类型级别（例如指定在表级别上的  :class:`.String` ）上使用排序的情况，因为引用是已经应用了引用。

  :ticket:`3785`  

Sqlalchemy中的边界值改变
===========================

.. _change_4063:

自定义运算符的键入行为已统一
-----------------------------------

用户可以使用  :meth:`运营商。op`  ` 函数即时制作运算符。以前对于针对此类运算符的表达式的打印行为不一致，也不可控。现在表达式对此操作的键入行为与左手表达式相同:: 

    column（“x”，types.DateTime）.op（“ - ％gt;”）（无）.type
    NullType（）

其他类型将使用左手类型作为返回类型的默认行为：：

    column（“x”，types.String（50））。op（“ - ％gt;”）（无）.type
    String（length = 50）

这些行为大多是偶然的，因此更改后的行为与第二种形式一起制作，即默认返回类型与左手表达式相同：： 

    column（“x”，types.DateTime）.op（“ - ％gt;”）（无）.type
    DateTime（）

由于大多数用户定义的运算符往往是“比较”运算符，通常是由PostgreSQL定义的许多特殊运算符之一，因此现在：“运营商。op.is_comparison”标志已被维修，遵循其文档化的行为，即包括 :class:`.Boolean` 在内的所有情况都允许返回类型为?

    column（“x”，types.String（50））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

    column（“x”，types.ARRAY（types.Integer））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

    column（“x”，types.JSON（））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

为了协助布尔比较运算符，添加了一个新的简写方法  :meth:`运营商。bool_op。`  此方法应优选即时制作布尔值运算符： 

.. sourcecode :: pycon + sql

    >>> print（column（“x”，types.Integer）.bool_op（“ - ％gt;”）（5））
    {printsql} x  - ％gt;：x_1


.. _change_3740:

缩小literal_column（）中的百分号现在有条件转义
--------------------------------------------------


现在，  :obj:`_expression.literal_column`  构造在特定情况下条件地转义百分号字符，具体取决于DBAPI是否使用敏感的百分符paramstyle或不使用该字符（例如'format'或'pyformat'）。 

以前，不可能生成声明单个百分号的 :obj:_expression.literal_column构造::

    >>> from sqlalchemy import literal_column
    >>> print（literal_column（“some％symbol”））
    {printsql} some％symbol

现在，对于未设置这些paramstyles的dialects（例如大多数MySQL dialects），百分号不受影响：：

    >>> from sqlalchemy import literal_column
    >>> print（literal_column（“ some％symbol”））
    {printsql}一些％符号{stop}
    >>> from sqlalchemy.dialects import mysql
    >>> print（literal_column（“ some％symbol”）.compile（dialect = mysql.dialect（）））
    {printsql}一些％符号{stop}

另外，针对  :meth:`.Operators.contains`  ，  :meth:` .ColumnOperators.startswith`  和  :meth:`.ColumnOperators.endswith`  等运算符的使用的双倍将仅在适当时发生。 

  :ticket:`3740`  


.. _change_3785:

列级别的COLLATE关键字现在引用了排序名称
--------------------------------------------------------------–

已经修复了在   :func:`_expression.collate`  and :meth:` .ColumnOperators.collate`所使用的，用于在语句级别提供ad-hoc列排序的约束条件的一个错误，在其中一个大小写敏感的名称没有引用时：:

    sel = select([table1.c.my_name]).where(table1.c.my_name.collate('...'))

现在渲染为：

.. sourcecode:: sql

    SELECT table1.my_name
    FROM table1
    WHERE table1.my_name COLLATE "..."

之前大小写敏感名称中的“fr_FR”未被引用。当前，手动引用标识符不会被检测到，因此必须调整手动引用标识符的应用程序。请注意，此更改不影响在类型级别（例如指定在表级别上的  :class:`.String` ）上使用排序的情况，因为引用是已经应用了引用。

  :ticket:`3785`  

Sqlalchemy中的边界值改变
===========================

.. _change_4063:

自定义运算符的键入行为已统一
-----------------------------------

用户可以使用  :meth:`运营商。op`  ` 函数即时制作运算符。以前对于针对此类运算符的表达式的打印行为不一致，也不可控。现在表达式对此操作的键入行为与左手表达式相同:: 
	
    column（“x”，types.DateTime）.op（“ - ％gt;”）（无）.type
    NullType（）

其他类型将使用左手类型作为返回类型的默认行为：：

    column（“x”，types.String（50））。op（“ - ％gt;”）（无）.type
    String（length = 50）

这些行为大多是偶然的，因此更改后的行为与第二种形式一起制作，即默认返回类型与左手表达式相同：： 

    column（“x”，types.DateTime）.op（“ - ％gt;”）（无）.type
    DateTime（）

由于大多数用户定义的运算符往往是“比较”运算符，通常是由PostgreSQL定义的许多特殊运算符之一，因此现在：“运营商。op.is_comparison”标志已被维修，遵循其文档化的行为，即包括 :class:`.Boolean` 在内的所有情况都允许返回类型为?

    column（“x”，types.String（50））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

    column（“x”，types.ARRAY（types.Integer））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

    column（“x”，types.JSON（））。op（ “ - ％gt;”，is_comparison = True）（无）.type
    布尔（）

为了协助布尔比较运算符，添加了一个新的简写方法  :meth:`运营商。bool_op。`  此方法应优选即时制作布尔值运算符： 

.. sourcecode :: pycon+sql

    >>> print（column（“x”，types.Integer）.bool_op（“ - ％gt;”）（5））
    {printsql} x - ％gt; :x_1


.. _change_3740:

缩小literal_column（）中的百分号现在有条件转义
--------------------------------------------------


现在，  :obj:`_expression.literal_column`  构造在特定情况下条件地转义百分号字符，具体取决于DBAPI是否使用敏感的百分符paramstyle或不使用该字符（例如'format'或'pyformat'）。 

以前，不可能生成声明单个百分号的 :obj:_expression.literal_column构造::

    >>> from sqlalchemy import literal_column
    >>> print（literal_column（“some％symbol”））
    {printsql} some％symbol

现在，对于未设置这些paramstyles的dialects（例如大多数MySQL dialects），百分号不受影响：：

    >>> from sqlalchemy import literal_column
    >>> print（literal_column（“ some％symbol”））
    {printsql} some％symbol{stop}
    >>> from sqlalchemy.dialects import mysql
    >>> print（literal_column（“ some％symbol”）.compile（dialect = mysql.dialect（）））
    {printsql} some%%symbol{stop}

另外，针对  :meth:`.Operators.contains`  ，  :meth:` .ColumnOperators.startswith`  和  :meth:`.ColumnOperators.endswith`  等运算符的使用的双倍将仅在适当时发生。 

  :ticket:`3740`  

.. _change_3785:

列级别的COLLATE关键字现在引用了排序名称
-------------------------------------------------------------- 

已经修复了在   :func:`_expression.collate`  and :meth:` .ColumnOperators.collate`所使用的，用于在语句级别提供ad-hoc列排序的约束条件的一个错误，在其中一个大小写敏感的名称没有引用时：:

    sel = select([table1.c.my_name]).where(table1.c.my_name.collate('...'))

现在渲染为：

.. sourcecode:: sql

    SELECT table1.my_name
    FROM table1
    WHERE table1.my_name COLLATE "..."

之前大小写敏感名称中的“fr_FR”未被引用。当前，手动引用标识符不会被检测到，因此必须调整手动引用标识符的应用程序。请注意，此更改不影响在类型级别（例如指定在表级别上的  :class:`.String` ）上使用排序的情况，因为引用是已经应用了引用。

  :ticket:`3785`  

改进和改变的方言 - PostgreSQL
=============================================

.. _change_4109:

支持批量模式/快速执行助手
--------------------------------

已经确定Psycopg2的``cursor.executemany（）``方法表现差，特别是对于INSERT语句而言。为了缓解这种情况，psycopg2添加了`快速执行助手 <https://initd.org/psycopg/docs/extras.html#fast-execution-helpers>`_，它通过在批处理中发送多个DML语句来减少服务器往返次数。 SQLAlchemy 1.2现在包括对这些助手的支持，以便可在  :class:`_engine.Engine` ` cursor.executemany（）``对多个参数集合发出语句时透明使用。该功能默认关闭，可以通过  :func:`_sa.create_engine` ` use_batch_mode``参数启用：：

    engine = create_engine(
        "postgresql+psycopg2://scott:tiger@host/dbname", use_batch_mode=True
    )

该功能目前被认为是实验性的，但在将来的版本中可能会默认启用。请参见   :ref:`psycopg2_batch_mode` 。

  :ticket:`4109`  

.. _change_3959:

支持在INTERVAL中进行字段规范，包括完整的反射
--------------------------------------------------------

在PostgreSQL的INTERVAL数据类型中，“fields”指示符允许指定要存储的间隔的哪些字段，包括诸如“YEAR”，“MONTH”，“YEAR TO MONTH”等值。  :class:`_postgresql.INTERVAL` 数据类型现在允许指定这些值：：

    from sqlalchemy.dialects.postgresql import INTERVAL

    Table("my_table", metadata, Column("some_interval", INTERVAL(fields="DAY TO SECOND")))

此外，现在可以独立于“fields”说明符反射所有INTERVAL数据类型; 数据类型本身中的“fields”参数也将存在：：

    >>> inspect(engine).get_columns("my_table")
    [{'comment': None，
         'name'：u'some_interval'，'nullable'：True，
         'default'：None，'autoincrement'：False，
         'type'：INTERVAL(fields = u'day to second')}]

  :ticket:`3959`  

改进和改变的方言 - MySQL
========================================

.. _change_4009:

支持INSERT..ON DUPLICATE KEY UPDATE
-------------------------------------------

MySQL支持的“ON DUPLICATE KEY UPDATE”子句现在可以使用  :class:`_expression.Insert` 。这个  :class:` _expression.Insert` ~.mysql.dml.Insert.on_duplicate_key_update` ，实现了MySQL的语法：：

    from sqlalchemy.dialects.mysql import insert

    insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

    on_conflict_stmt = insert_stmt.on_duplicate_key_update(
        data=insert_stmt.inserted.data, status="U"
    )

    conn.execute(on_conflict_stmt)

上面将呈现：

.. sourcecode:: sql

    INSERT INTO my_table (id, data)
    VALUES (:id, :data)
    ON DUPLICATE KEY UPDATE data=VALUES(data), status=:status_1

请参见::

      :ref:`mysql_insert_on_duplicate_key_update` 

  :ticket:`4009`  

改进和改变的方言 - Oracle
=========================================

.. _change_cxoracle_12:

CX_Oracle方言，键入系统的重大重构
------------------------------------------

随着cx_Oracle 6.x系列的推出，SQLAlchemy的cx_Oracle方言已被重新设计和简化，以利用cx_Oracle最近的改进，并丢弃了在cx_Oracle 5.x系列之前更为相关的模式的支持模式。

    最少的cx_Oracle版本支持现在是5.1.3; 建议使用5.3或最近的6.x系列。

    数据类型的处理方式已被重构。对于除LOB类型以外的任何数据类型，不再使用“cursor.setinputsizes（）”方法，根据cx_Oracle开发人员的建议。因此，参数“auto_setinputsizes”和“exclude_setinputsizes”已弃用，并且不再具有任何效果。

    “coerce_to_decimal”标志，仅当值的精度和比例强制转换为“Decimal”时应设置为False，仅影响未键入的（例如没有  :class:`.TypeEngine` .Numeric` 类型或子类型的Core表达式现在将遵循该类型的十进制强制规则。

    方言中的“双阶段”事务支持已被删除，对于cx_Oracle 6.x系列，这个特性已经被删除，并且几乎没有工作的这个东西，也不太可能在生产中使用。因此，“allow_twophase”dialect标志已弃用，也没有任何作用。

    修复了列名RETURNING中存在的错误。给出如下语句：：

     result = conn.execute(table.insert().values(x=5).returning(table.c.a, table.c.b))

    之前的每行结果中的键将是``ret_0``和``ret_1`，这些是cx_Oracle的``RETURNING``实现中的内部标识符。这些键将是``a``和``b``，如其他方言所期望的。

    cx_Oracle的LOB数据类型将返回值表示为``cx_Oracle.LOB``对象，该对象是与游标关联的代理，通过``.read（）``方法返回最终数据值。历史上，如果在这些LOB对象被消耗之前（具体地说，更多的行比为开销阵列大小的值多被读取），读取了更多的行，这些LOB对象将引发错误“LOB variable no longer valid after subsequent fetch”。 SQLAlchemy通过在其类型系统中自动调用``.read（）``来解决此问题，并使用特殊的``BufferedColumnResultSet``，以确保在使用调用``cursor.fetchmany（）``或``cursor.fetchall（）``时缓冲数据。

    当前的方言现在使用cx_Oracle outputtypehandler来处理这些``.read（）``调用，以便它们始终首先被调用，而无论正在获取多少行，因此这个错误不再可能出现。但是，与此用例相关的内部，如``BufferedColumnResultSet``的使用，已被删除，以及Core“ResultSet”中的一些其他内部，这些内部是这个用例的特定用途。由于cx_Oracle 6.x已经删除了这个错误的条件，因此不再可能发生错误。该错误可以在生产的情况下发生，如果使用了很少使用（如果有）的``auto_convert_lobs = False``选项，结合前面的cx_Oracle 5.x系列，并且在LOB对象可以被消费之前读取更多的行。升级到cx_Oracle 6.x将解决这个问题。