SQLAlchemy 0.4有什么新功能？
=============================

.. admonition:: 关于本文档

    本文档介绍了SQLAlchemy 0.3和0.4之间的更改，最近发布于2008年10月12日。

    文档日期：2008年3月21日

首先要做什么
==================

如果您正在使用任何ORM功能，请确保从``sqlalchemy.orm``导入：

::

    from sqlalchemy import *
    from sqlalchemy.orm import *

其次，在任何以前使用``engine =` `，``connectable =` `，``bind_to =` `，``something.engine``，``metadata.connect（）``的地方，请使用``bind =``：

::

    myengine = create_engine("sqlite://")

    meta = MetaData(myengine)

    meta2 = MetaData()
    meta2.bind = myengine

    session = create_session(bind=myengine)

    statement = select([table], bind=myengine)

这些知道了吗？好的！您现在（95％）兼容0.4了。如果您使用的是0.3.10，可以立即进行这些更改；它们也将适用于该位置。

模块导入
==============

在0.3中，“``from sqlalchemy import *``”将所有sqlalchemy的子模块导入到您的命名空间中。版本0.4不再将子模块导入命名空间。这可能意味着您需要在代码中添加额外的导入。

在0.3中，此代码有效：

::

    from sqlalchemy import *


    class UTCDateTime(types.TypeDecorator):
        pass

在0.4中，必须这样做：

::

    from sqlalchemy import *
    from sqlalchemy import types


    class UTCDateTime(types.TypeDecorator):
        pass

对象关系映射
=========================

查询
--------

新的查询API
^^^^^^^^^^^^^

查询基于生成器接口进行标准化（旧接口仍然存在，只是已弃用）。虽然0.3中的大多数生成接口在0.4查询中仍然可用，但是0.4查询具有内部部件以匹配生成器外部，并且具有更多技巧。所有结果缩小都通过``filter（）``和``filter_by（）``，限制/偏移量通过数组切片或``limit（）/ offset（）``进行，连接通过``join（）``和``outerjoin（）``进行（或更手动地，通过``select_from（）``以及手动形成的标准）。

要避免已弃用的警告，请对03代码进行一些更改

User.query.get_by( \**kwargs )

::

    User.query.filter_by(**kwargs).first()

User.query.select_by( \**kwargs )

::

    User.query.filter_by(**kwargs).all()

User.query.select()

::

    User.query.filter(xxx).all()

新的基于属性的表达式构造
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

到目前为止，在ORM中最明显的区别是现在您可以直接使用基于类属性构建查询标准。当使用映射类时，不再需要“``.c.``”前缀：

::

    session.query(User).filter(and_(User.name == "fred", User.id > 17))

虽然简单的基于列的比较并不是什么大问题，但是类属性具有一些新的“更高级别”的结构可用，包括之前仅在``filter_by（）``中可用的结构：

::

    # comparison of scalar relations to an instance
    filter(Address.user == user)

    # return all users who contain a particular address
    filter(User.addresses.contains(address))

    # return all users who *dont* contain the address
    filter(~User.address.contains(address))

    # return all users who contain a particular address with
    # the email_address like '%foo%'
    filter(User.addresses.any(Address.email_address.like("%foo%")))

    # same, email address equals 'foo@bar.com'.
    # for simple comparisons,% keyword
    filter(User.addresses.any(email_address="foo@bar.com"))

    # return all Addresses whose user attribute has the username 'ed'
    filter(Address.user.has(name="ed"))

    # return all Addresses whose user attribute has the username 'ed'
    # and an id > 5 (mixing clauses with kwargs)
    filter(Address.user.has(User.id > 5, name="ed"))

在映射类的``.c``属性中仍然可用``Column``集合。请注意，只有映射类的映射属性可用于基于属性的表达式。 ``.c``仍用于访问来自SQL表达式生成的常规表格和可选择对象中的列。

自动连接别名
^^^^^^^^^^^^^^^^^^^^^^^

我们已经有一段时间使用join（）和outerjoin（）了：

::

    session.query(Order).join("items")

现在，您可以将它们别名：

::

    session.query(Order).join("items", aliased=True).filter(Item.name="item 1").join(
        "items", aliased=True
    ).filter(Item.name == "item 3")

上述代码将创建从订单->项目使用别名的两个连接。每个后续“filter（）”调用都将其表述准则调整为该别名的准则。使用``add_entity（）``并用“id”定位每个连接就可以获得``Item``对象：

::

    session.query(Order).join("items", id="j1", aliased=True).filter(
        Item.name == "item 1"
    ).join("items", aliased=True, id="j2").filter(Item.name == "item 3").add_entity(
        Item, id="j1"
    ).add_entity(
        Item, id="j2"
    )

以``（Order，Item，Item）``形式返回元组。

自我引用查询
^^^^^^^^^^^^^^^^^^^^^^^^

因此，查询。join（）现在可以别名了。那给了我们什么？自引用查询！无需任何“Alias”对象即可进行连接：

::

    # standard self-referential TreeNode mapper with backref
    mapper(
        TreeNode,
        tree_nodes,
        properties={
            "children": relation(
                TreeNode, backref=backref("parent", remote_side=tree_nodes.id)
            )
        },
    )

    # query for node with child containing "bar" two levels deep
    session.query(TreeNode).join(["children", "children"], aliased=True).filter_by(
        name="bar"
    )

要在别名连接的每个表上添加标准，您可以使用``from_joinpoint``，以便针对相同的别名行继续连接：

::

    # search for the treenode along the path "n1/n12/n122"

    # first find a Node with name="n122"
    q = sess.query(Node).filter_by(name="n122")

    # then join to parent with "n12"
    q = q.join("parent", aliased=True).filter_by(name="n12")

    # join again to the next parent with 'n1'.  use 'from_joinpoint'
    # so we join from the previous point, instead of joining off the
    # root table
    q = q.join("parent", aliased=True, from_joinpoint=True).filter_by(name="n1")

    node = q.first()

``query.populate_existing（）``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``query.load()``的热情版本（或``session.refresh（）``）。从查询中加载的每个实例都将即时刷新，包括所有急切地加载的项目，如果已经存在于会话中，则会立即刷新它们：

::

    session.query(Blah).populate_existing().all()

关系
---------

嵌入式更新/插入SQL子句
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在``flush（）``期间，在更新或插入中嵌入SQL子句的内联执行：

::

    myobject.foo = mytable.c.value + 1

    user.pwhash = func.md5(password)

    order.hash = text("select hash from hashing_table")

操作后的列属性设置为延迟加载器，以便在下一次访问时发出加载新值的SQL。

自我引用和循环急切加载
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

由于别名化的改进，``relation（）``现在可以沿着相同的表*任意次数*加入；告诉它你想要走多深。让我们更清楚地显示自我引用的“``TreeNode``”：

::

    nodes = Table(
        "nodes",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("parent_id", Integer, ForeignKey("nodes.id")),
        Column("name", String(30)),
    )


    class TreeNode(object):
        pass


    mapper(
        TreeNode,
        nodes,
        properties={"children": relation(TreeNode, lazy=False, join_depth=3)},
    )

那么当我们说：

::

    create_session().query(TreeNode).all()

什么会发生？一次沿着别名，深入三个级别的父级：

.. sourcecode:: sql

    SELECT
    nodes_3.id AS nodes_3_id, nodes_3.parent_id AS nodes_3_parent_id, nodes_3.name AS nodes_3_name,
    nodes_2.id AS nodes_2_id, nodes_2.parent_id AS nodes_2_parent_id, nodes_2.name AS nodes_2_name,
    nodes_1.id AS nodes_1_id, nodes_1.parent_id AS nodes_1_parent_id, nodes_1.name AS nodes_1_name,
    nodes.id AS nodes_id, nodes.parent_id AS nodes_parent_id, nodes.name AS nodes_name
    FROM nodes LEFT OUTER JOIN nodes AS nodes_1 ON nodes.id = nodes_1.parent_id
    LEFT OUTER JOIN nodes AS nodes_2 ON nodes_1.id = nodes_2.parent_id
    LEFT OUTER JOIN nodes AS nodes_3 ON nodes_2.id = nodes_3.parent_id
    ORDER BY nodes.oid, nodes_1.oid, nodes_2.oid, nodes_3.oid

请注意，还有一个好看的干净的别名名称。即使连接与同一个立即表不同的对象无关，连接也不会在本身上循环。任何连接可以通过指定``join_depth``而返回到自身。当不存在时，贪婪装载自动停止贪婪装载。

复合类型
^^^^^^^^^^^

这是Hibernate阵营的一种。复合类型允许您定义由多个列（或一个列，如果您愿意）组成的自定义数据类型。让我们定义一个名为``Point``的新类型。存储x / y坐标：

::

    class Point(object):
        def __init__(self, x, y):
            self.x = x
            self.y = y

        def __composite_values__(self):
            return self.x, self.y

        def __eq__(self, other):
            return other.x == self.x and other.y == self.y

        def __ne__(self, other):
            return not self.__eq__(other)

定义``Point``对象的方式特定于自定类型；构造函数接受一个参数列表，``__composite_values __（）``方法生成这些参数的序列。命令将匹配我们的映射，如下所示：

让我们创建一个存储每行两个点的顶点表：

::

    vertices = Table(
        "vertices",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("x1", Integer),
        Column("y1", Integer),
        Column("x2", Integer),
        Column("y2", Integer),
    )

然后，映射它！我们将创建一个``Vertex``对象，其中存储了两个“``Point``”对象：

::

    class Vertex(object):
        def __init__(self, start, end):
            self.start = start
            self.end = end


    mapper(
        Vertex,
        vertices,
        properties={
            "start": composite(Point, vertices.c.x1, vertices.c.y1),
            "end": composite(Point, vertices.c.x2, vertices.c.y2),
        },
    )

一旦设置了复合类型，它的使用方式就与任何其他类型一样：

::


    v = Vertex(Point(3, 4), Point(26, 15))
    session.save(v)
    session.flush()

    # works in queries too
    q = session.query(Vertex).filter(Vertex.start == Point(3, 4))

如果要定义映射属性在表达式中使用时生成SQL子句的方式，请创建自己的``sqlalchemy.orm.PropComparator``子类，并将其发送到``composite（）``中。复合类型也可以作为主键使用，并且可用于``query.get()``：

::

    # a Document class which uses a composite Version
    # object as primary key
    document = query.get(Version(1, "a"))

“dynamic_loader（）”关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``relation()``会为所有读操作返回一个实时的``Query``对象。写操作仅限于``append（）``和``remove（）``，直到会话刷新为止，集合的更改才可见。此功能在“自动刷新”会话的情况下特别方便，它将在每个查询之前刷新。

::

    mapper(
        Foo,
        foo_table,
        properties={
            "bars": dynamic_loader(
                Bar,
                backref="foo",
                # <other relation() opts>
            )
        },
    )

    session = create_session(autoflush=True)
    foo = session.query(Foo).first()

    foo.bars.append(Bar(name="lala"))

    for bar in foo.bars.filter(Bar.name == "lala"):
        print(bar)

    session.commit()

新选项：“undefer_group（）”，“eagerload_all（）”
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

几个查询选项非常方便。 ``undefer_group()``将整个“推迟”的列组标记为未推迟：

::

    mapper(
        Class,
        table,
        properties={
            "foo": deferred(table.c.foo, group="group1"),
            "bar": deferred(table.c.bar, group="group1"),
            "bat": deferred(table.c.bat, group="group1"),
        },
    )

    session.query(Class).options(undefer_group("group1")).filter(...).all()

而“eagerload_all（）”在一次传递中设置一系列属性为贪婪：

::

    mapper(Foo, foo_table, properties={"bar": relation(Bar)})
    mapper(Bar, bar_table, properties={"bat": relation(Bat)})
    mapper(Bat, bat_table)

    # eager load bar and bat
    session.query(Foo).options(eagerload_all("bar.bat")).filter(...).all()

新的集合API
^^^^^^^^^^^^^^^^^^

集合现在不再由身份列表（{{{InstrumentedList}}} proxy）代理，对成员，方法和属性的访问是直接的。装饰器现在截取进入和离开集合的对象，并且现在可以轻松编写自定义集合类来管理其自己的成员身份。灵活的装饰器还替换了0.3中定制集合的命名方法界面，允许任何类轻松适应用作集合容器。

基于字典的集合现在更易于使用，完全类似于``dict``。不再需要更改“__iter__”对于字典和新的内置“dict”类型可以满足许多需求：

::

    # use a dictionary relation keyed by a column
    relation(Item, collection_class=column_mapped_collection(items.c.keyword))
    # or named attribute
    relation(Item, collection_class=attribute_mapped_collection("keyword"))
    # or any function you like
    relation(Item, collection_class=mapped_collection(lambda entity: entity.a + entity.b))

现有的0.3类``dict`` -like和基于自由对象的集合类需要更新其新的API。在大多数情况下，这仅涉及向类定义添加几个装饰器。

来自外部表/子查询的映射关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

此功能在0.3中悄然出现，但是在0.4中得到了改进，由于更好地能够将针对表的子查询转换为该表别名的子查询；这对于贪婪的装载，查询中的别名连接等非常重要。这减少了对选择语句进行映射器的创建的必要性，当您只需要添加一些额外的列或子查询时。

::

    mapper(
        User,
        users,
        properties={
            "fullname": column_property(
                (users.c.firstname + users.c.lastname).label("fullname")
            ),
            "numposts": column_property(
                select([func.count(1)], users.c.id == posts.c.user_id)
                .correlate(users)
                .label("posts")
            ),
        },
    )

典型查询的外观如下：

.. sourcecode:: sql

    SELECT (SELECT count(1) FROM posts WHERE users.id = posts.user_id) AS count,
    users.firstname || users.lastname AS fullname,
    users.id AS users_id, users.firstname AS users_firstname, users.lastname AS users_lastname
    FROM users ORDER BY users.oid

继承
-----------

没有联接或联合的多态继承
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

继承的新文档： https://www.sqlalchemy.org/docs/04
/mappers.html#advdatamapping_mapper_inheritance_joined

更好的多态行为与“get（）”
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

联接表继承层次结构中的所有类都使用基类获取“_instance_key”，即``(BaseClass，（1，）,None)``。那么，当您调用基类上的``get（）``从基类的``Query``中查找子类实例时，它可以在当前标识映射中查找子类实例，而无需查询数据库。

类型
-----

sqlalchemy.types.TypeDecorator的自定义子类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^■

有一个`新API <https://www.sqlalchemy.org/docs/04/types
.html#types_custom>`_可用于子类化TypeDecorator。在某些情况下，使用0.3 API会导致编译错误。

SQL表达式
===============

全新，确定性Label / Alias Generation
---------------------------------------

所有“匿名”标签和别名现在使用简单的<name>_<number>格式。 SQL更容易阅读，并与计划优化器缓存兼容。只需查看教程中的一些示例：
https://www.sqlalchemy.org/docs/04/ormtutorial.html
https://www.sqlalchemy.org/docs/04/sqlexpression.html

生成选择（）构造
------------------------------

这显然是使用 “select（）” 的方法。请参阅htt
p://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_transf
orm 。

新的运营商系统
-------------------

SQL运算符和更多或更多SQL关键字现在已抽象为编译器层。它们现在可以智能地运作，并具有类型/后端感知性，请参见：https：//www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators

所有“type”关键字参数重命名为“type_”
--------------------------------------------------

就像它所说的那样：

::

       b = bindparam("foo", type_=String)

变为_函数改为接受序列或可选择的
--------------------------------------------------------

in\_函数现在需要一个值序列或可选择的可接受其唯一参数。将值作为位置参数传递的以前API仍然有效，但是现已被弃用。这意味着

::

    my_table.select(my_table.c.id.in_(1, 2, 3))
    my_table.select(my_table.c.id.in_(*listOfIds))

应更改为

::

    my_table.select(my_table.c.id.in_([1, 2, 3]))
    my_table.select(my_table.c.id.in_(listOfIds))

架构和反射
=====================

“MetaData”，“BoundMetaData”，“DynamicMetaData”...
-------------------------------------------------------

在0.3.x系列中，``BoundMetaData``和
``DynamicMetaData``已弃用，预计更换为``MetaData``
和``ThreadLocalMetaData``。旧名称已在0.4中删除。更新很简单：

.. sourcecode:: text

    +-------------------------------------+-------------------------+
    |如果您有                            | 现在使用                |
    +=====================================+=========================+
    | ``MetaData``                        | ``MetaData``            |
    +-------------------------------------+-------------------------+
    | ``BoundMetaData``                   | ``MetaData``            |
    +-------------------------------------+-------------------------+
    | ``DynamicMetaData``（有一个引擎或   | ``MetaData``            |
    | threadlocal = False）                |                         |
    +-------------------------------------+-------------------------+
    | ``DynamicMetaData``                 | ``ThreadLocalMetaData`` |
    |（每个线程具有不同的引擎）          |                         |
    +-------------------------------------+-------------------------+

Seldom-used“name”指针已被删除。 ``ThreadLocalMetaData``构造函数现在不带参数。这两种类型都可以绑定到``Engine``或单个``Connection``。

一步多表反射
-------------------------------

现在可以在一步中从整个数据库或模式中加载表定义并自动创建“Table”对象：

::

    >>> metadata = MetaData(myengine, reflect=True)
    >>> metadata.tables.keys()
    ['table_a', 'table_b', 'table_c', '...']

``MetaData``还增加了一个``.reflect()``
方法，可对加载过程进行更好的控制，包括指定要加载的可用表的子集。

SQL执行
=============

“engine”，“connectable”和“bind_to”现在都是“bind”
----------------------------------------------------------

“Transactions”、“NestedTransactions”和“TwoPhaseTransactions”
----------------------------------------------------------------------

连接池事件
----------------------

连接池现在在创建新的DB-API时应发出事件
连接，检查并将其检查回池中。您可以使用这些在新连接上执行会话范围的SQL设置语句，例如。

Oracle Engine固定
-------------------

在0.3.11中，Oracle Engine存在有关主键处理方式的错误。这些错误可能会导致程序出现问题。当使用SQLite等其他引擎时并没有问题，但在使用Oracle引擎时会出现失败。
在0.4版本中，Oracle引擎已经重新设计，修复了这些主键问题。

Oracle的输出参数
-------------------

::

    result = engine.execute(
        text(
            "begin foo(:x, :y, :z); end;",
            bindparams=[
                bindparam("x", Numeric),
                outparam("y", Numeric),
                outparam("z", Numeric),
            ],
        ),
        x=5,
    )
    assert result.out_parameters == {"y": 10, "z": 75}

基于连接的“MetaData”和“Sessions”
------------------------------------

“MetaData”和“Session”可以显式地绑定到连接：

::

    conn = engine.connect()
    sess = create_session(bind=conn)

更快、更可靠的“ResultProxy”对象
----------------------------------------

