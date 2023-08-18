==============================
SQLAlchemy 0.9 的新特性
==============================

.. admonition:: 关于本文档

    本文档描述的是SQLAlchemy 0.8版本已进行维护的变更情况，时间为2013年5月；
    与SQLAlchemy 0.9版本（第一次生产发布于2013年12月30日）之间的变更。
    文档更新时间: 2015年6月10日

引言
=====

本指南介绍了SQLAlchemy 0.9版的新特性，并记录了影响用户从SQLAlchemy 0.8系列迁移应用程序的变更。

请仔细查阅:ref:`behavioral_changes_orm_09`和:ref:`behavioral_changes_core_09`以获取可能具有向后不兼容的更改的相关信息。

平台支持
==========

针对Python 2.6, 支持Python 3 (无需使用2to3).
------------------------------------------------------

0.9版本中的第一项成就是消除了与2to3工具相关的Python 3兼容性依赖。为了更加直观，最低Python版本现在为2.6，它具有与Python 3兼容的高度交叉性。
现在所有的SQLAlchemy模块和单元测试都可以在任何Python 2.6及以上的版本中同样良好地解释运行，包括3.1和3.2解释器。

:ticket:`2671`

C扩展在Python 3中可用
-----------------------

C扩展已经支持Python3环境下编译和运行。

:ticket:`2161`

.. _behavioral_changes_orm_09:

行为变更 - ORM
========================

.. _migration_2824:

复合属性现在在按属性基础上查询时作为它们的对象形式返回
---------------------------------------------------------

使用:class:`_query.Query`和复合属性时，现在返回的是组成该复合属性的对象类型，而不是拆分为单个列。使用:ref:`mapper_composite`中设置的映射：

    >>> session.query(Vertex.start, Vertex.end).filter(Vertex.start == Point(3, 4)).all()
    [(Point(x=3, y=4), Point(x=5, y=6))]

这个改变在代码期望将单个属性展开为单独的列时与原代码不兼容。要获得此行为，请使用".clauses"访问器:

    >>> session.query(Vertex.start.clauses, Vertex.end.clauses).filter(
    ...     Vertex.start == Point(3, 4)
    ... ).all()
    [(3, 4, 5, 6)]

.. seealso::

    :ref:`change_2824`

:ticket:`2824`


.. _migration_2736:

:meth:`_query.Query.select_from`不再将子句应用于对应的实体
---------------------------------------------------------

:meth:`_query.Query.select_from`方法在最近的版本中已被广泛应用，它是控制:class:`_query.Query`对象“从何处选择”的手段，通常用于控制如何呈现JOIN.

考虑下面的例子::

    select_stmt = select([User]).where(User.id == 7).alias()

    q = (
        session.query(User)
        .join(select_stmt, User.id == select_stmt.c.id)
        .filter(User.name == "ed")
    )

预期上面的语句会返回如下SQL：

.. sourcecode:: sql

    SELECT "user".id AS user_id, "user".name AS user_name
    FROM "user" JOIN (SELECT "user".id AS id, "user".name AS name
    FROM "user"
    WHERE "user".id = :id_1) AS anon_1 ON "user".id = anon_1.id
    WHERE "user".name = :name_1

如果我们想要交换JOIN操作中左侧和右侧的元素，我们可以使用:data:`_query.Query.select_from`方法来实现::

    q = (
        session.query(User)
        .select_from(select_stmt)
        .join(User, User.id == select_stmt.c.id)
        .filter(User.name == "ed")
    )

但在0.8及以下的版本，上述使用:data:`_query.Query.select_from`的示例会将"select_stmt"应用于**替换**“User”实体，因为它从与“User”兼容的“user”表中选择：

.. sourcecode:: sql

    -- SQLAlchemy 0.8及以下版本...
    SELECT anon_1.id AS anon_1_id, anon_1.name AS anon_1_name
    FROM (SELECT "user".id AS id, "user".name AS name
    FROM "user"
    WHERE "user".id = :id_1) AS anon_1 JOIN "user" ON anon_1.id = anon_1.id
    WHERE anon_1.name = :name_1

以上代码很难移植并且故意设定了这种行为。但现在，此行为可以使用一个被称为:meth:`_query.Query.select_entity_from`的新方法来实现。这是一个较少使用的行为，在当前SQLAlchemy中大致等同于从自定义:func:`.aliased`构造中选择：

    select_stmt = select([User]).where(User.id == 7)
    user_from_stmt = aliased(User, select_stmt.alias())

    q = session.query(user_from_stmt).filter(user_from_stmt.name == "ed")

因此，在SQLAlchemy 0.9中，与“select_stmt”有关的查询将产生我们期望的SQL。

.. sourcecode:: sql

    -- SQLAlchemy 0.9
    SELECT "user".id AS user_id, "user".name AS user_name
    FROM (SELECT "user".id AS id, "user".name AS name
    FROM "user"
    WHERE "user".id = :id_1) AS anon_1 JOIN "user" ON "user".id = id
    WHERE "user".name = :name_1

从SQLAlchemy 0.8.2版本开始，:meth:`_query.Query.select_entity_from`方法将可用，因此依赖于旧行为的应用程序首先可以过渡到该方法，并确保所有测试继续正常运行，然后升级为0.9，而不会出现问题。

:ticket:`2736`


.. _migration_2833:

在“relationship ()”上使用"viewonly=True"将阻止 history 生效
---------------------------------------------------------------------------

:func:`_orm.relationship`上的"viewonly"标志用于防止对目标属性的更改在Flush过程中产生任何影响。这是通过消除在Flush过程中考虑属性来实现的。在过去，即使对属性进行更改，将仍然将父对象注册为“dirty”并触发潜在的Flush。更改是“viewonly”标志现在也防止为目标属性设置history。像backrefs和用户定义的事件之类的属性事件仍然可以正常运作。

这个改变如下面所示：

    from sqlalchemy import Column, Integer, ForeignKey, create_engine
    from sqlalchemy.orm import backref, relationship, Session
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy import inspect

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)


    class B(Base):
        __tablename__ = "b"

        id = Column(Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey("a.id"))
        a = relationship("A", backref=backref("bs", viewonly=True))


    e = create_engine("sqlite://")
    Base.metadata.create_all(e)

    a = A()
    b = B()

    sess = Session(e)
    sess.add_all([a, b])
    sess.commit()

    b.a = a

    assert b in sess.dirty

    # before 0.9.0
    # assert a in sess.dirty
    # assert inspect(a).attrs.bs.history.has_changes()

    # after 0.9.0
    assert a not in sess.dirty
    assert not inspect(a).attrs.bs.history.has_changes()

:ticket:`2833`

.. _migration_2751:

关联代理SQL表达式的改进和修复
-------------------------------------------------------

现在对于从标量关系的标量值到关联代理的“==”和“！=”运算符，会生成更完整的SQL表达式，，将会考虑当与“None”进行比较时，“关联”行是否存在。

考虑以下映射：

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)

        b_id = Column(Integer, ForeignKey("b.id"), primary_key=True)
        b = relationship("B")
        b_value = association_proxy("b", "value")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)
        value = Column(String)

从以前开始，以下查询：

    s.query(A).filter(A.b_value == None).all()

将产生以下SQL：

.. sourcecode:: sql

    SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a
    WHERE EXISTS (SELECT 1
    FROM b
    WHERE b.id = a.b_id AND b.value IS NULL)

但是现在，在SQLAlchemy 0.9中，它将生成：

.. sourcecode:: sql

    SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a
    WHERE (EXISTS (SELECT 1
    FROM b
    WHERE b.id = a.b_id AND b.value IS NULL)) OR a.b_id IS NULL

区别是SQLAlchemy 0.9不仅检查“b.value”，还检查当某些父行没有关联行时的情况。因此对于使用此类比较以及一些父行没有关联行的系统，它将产生与早期版本不同的结果。

更为重要的是，“has()”运算符已经得到了增强，现在您可以仅针对一个带有单个条件的标量列值调用它，而不需要其他标准，它将产生检查是否存在关联行的标准：

    s.query(A).filter(A.b_value.has()).all()

输出：

.. sourcecode:: sql

    SELECT a.id AS a_id, a.b_id AS a_b_id
    FROM a
    WHERE EXISTS (SELECT 1
    FROM b
    WHERE b.id = a.b_id)

这相当于``A.b.has()``，但允许直接查询“b_value”。

:ticket:`2751`

.. _migration_2810:

关联代理缺失标量将返回 None
-------------------------------------------------------------

从标量属性到标量的关联代理现在将返回``None``，如果该代理对象不存在。这与缺失的多对一关系返回None相同，应该返回代理值。例如：

    from sqlalchemy import *
    from sqlalchemy.orm import *
    from sqlalchemy.ext.declarative import declarative_base
    from sqlalchemy.ext.associationproxy import association_proxy

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        b = relationship("B", uselist=False)

        bname = association_proxy("b", "name")


    class B(Base):
        __tablename__ = "b"

        id = Column(Integer, primary_key=True)
        a_id = Column(Integer, ForeignKey("a.id"))
        name = Column(String)


    a1 = A()

    # this is how m2o's always have worked
    assert a1.b is None

    # but prior to 0.9, this would raise AttributeError,
    # now returns None just like the proxied value.
    assert a1.bname is None

:ticket:`2810`


.. _change_2787:

attributes.get_history()现在默认情况下如果值不存在，则查询DB
-------------------------------------------------------------------------------

关于:func:`.attributes.get_history`的bugfix允许一个基于列的属性查询未加载的值，假设“passive”标志保持默认值为“PASSIVE_OFF”。以前，这个标志不会被采纳。此外，还添加了一个新方法 :meth:`.AttributeState.load_history`来补充 :attr:`.AttributeState.history`属性，它将为未加载的属性发出loader可调用的历史版本。

这是如下代码的变化：

    from sqlalchemy import Column, Integer, String, create_engine, inspect
    from sqlalchemy.orm import Session, attributes
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        data = Column(String)


    e = create_engine("sqlite://", echo=True)
    Base.metadata.create_all(e)

    sess = Session(e)

    a1 = A(data="a1")
    sess.add(a1)
    sess.commit()  # a1 is now expired

    # history doesn't emit loader callables
    assert inspect(a1).attrs.data.history == (None, None, None)

    # in 0.8, this would fail to load the unloaded state.
    assert attributes.get_history(a1, "data") == (
        (),
        [
            "a1",
        ],
        (),
    )

    # load_history() is now equivalent to get_history() with
    # passive=PASSIVE_OFF ^ INIT_OK
    assert inspect(a1).attrs.data.load_history() == (
        (),
        [
            "a1",
        ],
        (),
    )

:ticket:`2787`

.. _behavioral_changes_core_09:

行为变更 - Core
=========================

类型对象不再接受被忽略的关键字参数
-------------------------------------------------------

在0.8系列之前，大多数类型对象都接受任意关键字参数，其会被默默地忽略：

    from sqlalchemy import Date, Integer

    # storage_format argument here has no effect on any backend;
    # it needs to be on the SQLite-specific type
    d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")

    # display_width argument here has no effect on any backend;
    # it needs to be on the MySQL-specific type
    i = Integer(display_width=5)

这是一个很老的bug，在0.8系列中添加了一个不受支持的警告，但由于没有人将Python与“-W”标志一起运行，因此大多数情况下并没有出现警告：

.. sourcecode:: text

    $ python -W always::DeprecationWarning ~/dev/sqlalchemy/test.py
    /Users/classic/dev/sqlalchemy/test.py:5: SADeprecationWarning: Passing arguments to
    type object constructor <class 'sqlalchemy.types.Date'> is deprecated
      d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")
    /Users/classic/dev/sqlalchemy/test.py:9: SADeprecationWarning: Passing arguments to
    type object constructor <class 'sqlalchemy.types.Integer'> is deprecated
      i = Integer(display_width=5)

自0.9系列开始，"catch all"构造函数已从:class:`.TypeEngine`移除，因此这些无意义的参数不再被接受。

正确的方法是使用特定于方言的类型，如"storage_format"和"display_width"等方言特定参数应该使用特定于方言的类型。

    from sqlalchemy.dialects.sqlite import DATE
    from sqlalchemy.dialects.mysql import INTEGER

    d = DATE(storage_format="%(day)02d.%(month)02d.%(year)04d")

    i = INTEGER(display_width=5)

如果想使用不特定于方言的类型呢？我们使用:meth:`.TypeEngine.with_variant`方法:

    from sqlalchemy import Date, Integer
    from sqlalchemy.dialects.sqlite import DATE
    from sqlalchemy.dialects.mysql import INTEGER

    d = Date().with_variant(
        DATE(storage_format="%(day)02d.%(month)02d.%(year)04d"), "sqlite"
    )

    i = Integer().with_variant(INTEGER(display_width=5), "mysql")

:meth:`.TypeEngine.with_variant`不是新功能，它已经在SQLAlchemy0.7.2中添加了。因此，在0.8系列上运行的代码可以更正为使用这种方法，并在升级到0.9之前进行测试。

**Original**

为了在多元素路径中沿用某种加载样式，必须使用"_all()"方式：

    query(User).options(joinedload_all("orders.items.keywords"))

**新方式**

现在，加载程序选项可以链式使用，因此可以对每个链接应用相同的"joinedload(x)"方法，而不需要通过点或多个属性名来连接长路径。

    query(User).options(joinedload("orders").joinedload("items").joinedload("keywords"))

**原方式**




设置基于子类的路径上的选项需要将路径中的所有链接都拼写为类绑定属性，因为需要调用:meth:`.PropComparator.of_type`方法。如下所示：

```
session.query(Company).options(
    subqueryload_all(Company.employees.of_type(Engineer), Engineer.machines)
)
```

**新方法**

路径中只有确实需要:meth:`.PropComparator.of_type`的元素需要设置为类绑定属性，字符串名称可以在之后恢复使用::

```
session.query(Company).options(
    subqueryload(Company.employees.of_type(Engineer)).subqueryload("machines")
)
```

**旧方法**

在长路径中设置最后一个链接上的加载器选项使用了一种语法，看起来很像它应该为路径中的所有链接设置选项，这会引起混乱::

```
query(User).options(subqueryload("orders.items.keywords"))
```

**新方法**

现在可以使用:func:`.defaultload`为路径中的条目拼写路径，其中现有的加载器风格不应更改。更详细，但目的更清晰::

```
query(User).options(defaultload("orders").defaultload("items").subqueryload("keywords"))
```

仍然可以利用点形式，特别是在跳过几个路径元素的情况下::

```
query(User).options(defaultload("orders.items").subqueryload("keywords"))
```

**旧方法**

路径上的:func:`.defer`选项需要为每个列的完整路径拼写::

```
query(User).options(defer("orders.description"), defer("orders.isopen"))
```

**新方法**

到达目标路径的单个:class:`_orm.Load`对象可以反复调用:meth:`_orm.Load.defer`::

```
query(User).options(defaultload("orders").defer("description").defer("isopen"))
```

加载类
^^^^^^^^^^^^^^^

可以直接使用:class:`_orm.Load`类提供“绑定”的目标，尤其是存在多个父实体的情况下::

```
from sqlalchemy.orm import Load

query(User, Address).options(Load(Address).joinedload("entries"))
```

仅加载
^^^^^^^^^

新选项:func:`.load.only`实现了“除...外延迟所有”样式的加载，仅加载给定的列，并推迟其余的列::

```
from sqlalchemy.orm import load_only

query(User).options(load_only("name", "fullname"))

# 明确指定父实体
query(User, Address).options(Load(User).load_only("name", "fullname"))

# 指定路径
query(User).options(joinedload(User.addresses).load_only("email_address"))
```

类指定通配符
^^^^^^^^^^^^^^^^^^^^^^^^^

使用:class:`_orm.Load`，可以使用通配符为给定实体上的所有关系（或可能是列）设置加载，而不影响其他实体::

```
# 惰性加载所有User关系
query(User).options(Load(User).lazyload("*"))

# 未延迟所有User列
query(User).options(Load(User).undefer("*"))

# 惰性加载所有地址关系
query(User).options(defaultload(User.addresses).lazyload("*"))

# 未延迟所有Address列
query(User).options(defaultload(User.addresses).undefer("*"))
```

:ticket:`1418`


.. _feature_2877:

``text()``新功能
---------------------------

:func:`_expression.text`构造获得了新的方法:

* :meth:`_expression.TextClause.bindparams`允许灵活地设置绑定参数类型和值::

  # 设置值
  stmt = text(
      "SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp"
  ).bindparams(name="ed", timestamp=datetime(2012, 11, 10, 15, 12, 35))

  # 设置类型和/或值
  stmt = (
      text("SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp")
      .bindparams(bindparam("name", value="ed"), bindparam("timestamp", type_=DateTime()))
      .bindparam(timestamp=datetime(2012, 11, 10, 15, 12, 35))
  )

* :meth:`_expression.TextClause.columns`取代了:func:`_expression.text`的``typemap``选项，返回一个新构造``.TextAsFrom``::

  # 将text()转换为alias()，并附加一个.c集合：
  stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
  stmt = stmt.alias()

  stmt = select([addresses]).select_from(
      addresses.join(stmt), addresses.c.user_id == stmt.c.id
  )


  # 或转换为cte()：
  stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
  stmt = stmt.cte("x")

  stmt = select([addresses]).select_from(
      addresses.join(stmt), addresses.c.user_id == stmt.c.id
  )

:ticket:`2877`

.. _feature_722:

从SELECT中插入
------------------

经过多年毫无意义的拖延，这个比较小的语法特性现在已经添加，而且也被回退到了0.8.3中，因此在0.9中实际上并不“新”。:func:`_expression.select`语句或其他兼容构造可以传递给新方法:meth:`_expression.Insert.from_select`，其中它将被用于渲染一个“INSERT .. SELECT”构造:

```
from sqlalchemy.sql import table, column
t1 = table("t1", column("a"), column("b"))
t2 = table("t2", column("x"), column("y"))
print(t1.insert().from_select(["a", "b"], t2.select().where(t2.c.y == 5)))
```

该构造体已足够智能化，以适应ORM对象，例如类和:class:`_query.Query`对象::

  s = Session()
  q = s.query(User.id, User.name).filter_by(name="ed")
  ins = insert(Address).from_select((Address.id, Address.email_address), q)

生成:

```
INSERT INTO addresses (id, email_address)
SELECT users.id AS users_id, users.name AS users_name
FROM users WHERE users.name = :name_1
```

:ticket:`722`

.. _feature_github_42:

新的对于``select()``, ``Query()``的 'FOR UPDATE' 支持
---------------------------------------------------

在Core和ORM中尝试简化在``SELECT``语句中指定 ''FOR UPDATE'' 子句的规范化，
并增加对于 PostgreSQL 和 Oracle 支持的''FOR UPDATE OF'' SQL的支持。

使用核心:meth:`_expression.GenerativeSelect.with_for_update`，可以单独指定
选项，例如``FOR SHARE``和``NOWAIT``，而不是链接到任意字符串代码::

  stmt = select([table]).with_for_update(read=True, nowait=True, of=table)

对于Posgtresql，上述语句会像这样呈现：

```
SELECT table.a, table.b FROM table FOR SHARE OF table NOWAIT
```

:class:`_query.Query`对象增加了一个类似的次:meth:`_query.Query.with_for_update`，
其行为方式相同。该方法取代了:meth:`_query.Query.with_lockmode`方法，该方法是使用一个与正常的``FOR UPDATE``子句翻译不同的系统翻译``FOR UPDATE``子句的。
目前， ``lockmode``字符串参数仍然被: meth:`.Session.refresh` 方法接受。


.. _feature_2867:

本机浮点类型的浮点字符串转换精度可进行配置
---------------------------------------------------------------------------------------

每当DBAPI返回Python浮点类型需要转换为Python ``Decimal()``
时，SQLAlchemy必须做出一个必要的中间步骤，即将浮点值转换为字符串。
此字符串转换的规模以前是硬编码为10，现在可以进行配置。
设置是通过对包含``decimal_return_scale``参数的:class:`.Numeric`和:class:`.Float`类型，所有SQL或方言特定的后代类型使用。如果类型支持``.scale`` 参数（例如:class:`.Numeric`和某些浮点类型，例如:class:`.mysql.DOUBLE`），则``.scale``的值将用作默认值``.decimal_return_scale``。如果同时缺少``.scale``和``.decimal_return_scale``，则会使用默认值10。例如::

    from sqlalchemy.dialects.mysql import DOUBLE import decimal

    data = Table(
        "data",
        metadata,
        Column("double_value", mysql.DOUBLE(decimal_return_scale=12, asdecimal=True)),
    )

    conn.execute(
        data.insert(),
        double_value=45.768392065789,
    )

    result = conn.scalar(select([data.c.double_value]))

先前，这通常会被整理为Decimal（“45.7683920658”），例如截取到10个小数位。现在的结果
在请求的情况下得到12，因为MySQL可以支持这一点水平的精度



:ticket:`2867`


.. _change_2824:

针对ORM查询的列捆绑包
------------------------------

:class:`.Bundle`允许查询一组列，然后将这些列组成一个名称，该名称与查询返回的元组中的名称相匹配
列基本用法是1。允许在基于声明的基础上返回“组合”ORM列，而不是将它们展开为单个列和2。允许使用临时
列和返回类型在ORM中创建自定义结果集构造，而无需涉及映射类的更重量级的机制。

.. seealso::

    :ref:`migration_2824`

    :ref:`bundles`

:ticket:`2824`


服务器端版本计数
-----------------------------

ORM的版本控制功能（现在也在:ref:`mapper_version_counter`中记录）现在可以利用服务器端版本计数方案，
例如触发器或数据库系统列，以及条件编程方案越过version_id_counter函数本身。通过将值FALSE提供给``version_id_generator``参数，ORM将使用已设置的版本标识符，或者在发出INSERT或UPDATE时立即获取每行的版本标识符。当使用服务器生成的版本标识符时，强烈建议仅在支持强RETURNING支持的后端（PostgreSQL，SQL Server; Oracle还支持返回值，但cx_oracle驱动程序仅具有有限的支持），否则会向额外的SELECT语句添加重大的性能开销。 :ref:`server_side_version_counter`中提供的示例说明了如何使用PostgreSQL的xmin系统列将其与ORM的版本控制功能集成。

.. seealso::

    :ref:`server_side_version_counter`

:ticket:`2793`

.. _feature_1535:

``include_backrefs = False``选项对于``@validates``
----------------------------------------------------

函数:func:`.validates`现在接受一个选项``include_backrefs = True``，它将跳过仅从backref发出的验证器的情况::

```
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship, validates
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

    @validates("bs")
    def validate_bs(self, key, item):
        print("A.bs validator")
        return item


class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))

    @validates("a", include_backrefs=False)
    def validate_a(self, key, item):
        print("B.a validator")
        return item


a1 = A()
a1.bs.append(B())  # prints only "A.bs validator"
```

:ticket:`1535`


PostgreSQL JSON类型
--------------------

PostgreSQL方言现在具有:class:`_postgresql.JSON`类型，以补充:class:`_postgresql.HSTORE`类型。

.. seealso::

    :class:`_postgresql.JSON`

:ticket:`2581`

.. _feature_automap:

Automap扩展
-----------------

在＊＊0.9.1＊＊的新扩展名为＊＊sqlalchemy.ext.automap＊＊。这是一个实验性的扩展，它扩展了声明性以及:class:`.DeferredReflection`类的功能。基本上，该扩展提供了一个基类:class:`.AutomapBase`，它根据给定的表元数据自动生成映射类和之间的关系。

通常，使用的:class:`_schema.MetaData`可能是通过反射产生的，但是并不要求使用反射。最基本的用法说明了如何:mod:`sqlalchemy.ext.automap`能够基于反射架构生成映射类，包括关系::

from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(engine, reflect=True)

# mapped classes are now created with names matching that of the table
# name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named "<classname>_collection"
print(u1.address_collection)

此外，:class:`.AutomapBase`类是一个声明性基类，并支持声明性所支持的所有功能。 “自动映射”功能可以与现有的已明确声明的架构一同使用，以生成关系和丢失的类。可以使用可调用函数放置命名方案和关系-生产例程。

希望:class:`.AutomapBase`系统提供一种快速现代化的解决方案，即在现有数据库中自动生成一个简单且未加权重的对象模型。通过严格在映射器配置级别上处理问题，并与现有的Declarative类技术完全集成，:class:`.AutomapBase`寻求为快速自动生成临时映射提供完全集成的方法。.


0.9版中，“eager_defaults”现在可以使用VERSIONING扩展来为这些值发出RETURNING语句，因此在后端有强大的RETURNING支持，特别是PostgreSQL的情况下，ORM可以与INSERT或UPDATE一起内联获取新生成的默认值和SQL表达式值。当target backend和“_schema.Table” 支持“implicit returning”时，“eager_defaults”在启用时会自动使用RETURNING。 

子查询贪婪加载将为某些查询将DISTINCT应用于最内层SELECT中 

为了减少子查询贪婪加载包含多对一关系时可能生成的重复行数，当连接的目标列不包含主键时（在加载多对一的情况下），将会在内部SELECT中应用DISTINCT关键字

也就是说，在从A到B的多对一上进行子查询加载时：

SELECT b.id AS b_id, b.name AS b_name, anon_1.b_id AS a_b_id FROM (SELECT DISTINCT a_b_id FROM a) AS anon_1 JOIN b ON b.id = anon_1.a_b_id

由于“a.b_id”是一个非唯一的外键，所以将使用DISTINCT关键字，使得冗余的“a.b_id”被消除。可以使用标记“distinct_target_key”无条件地打开或关闭特定的:func:`_orm.relationship`：

将该值设置为True表示无条件打开它，将该值设置为False表示无条件关闭它，将该值设置为None表示当目标SELECT针对不包括完整主键的列时特性发挥作用。在0.9中，“None”是默认值。该选项还在0.8中回退，“distinct_target_key”选项默认为“False”。

尽管此功能旨在通过消除重复行来帮助提高性能，但SQL中的“DISTINCT”关键字本身可能会产生负面影响。

当渴望默认时：“eager_defaults”可以将新创建的、默认和SQL表达式值与INSERT或UPDATE一起获取。现在，类型处理器可以呈现“文本绑定”值。更新了始发对象与 :class: 的事件传递机制。一个细小但广泛的更改是：列可以从ForeignKey中获取它们的类型。修改了 :class: `.RowProxy` 的行为，现在它有元素排序。Firebird“fdb”现在是默认的Firebird方言，“fdb”和“kinterbasdb”现在默认情况下将“retaining”设置为“False”。对于这些Flag，现在已经添加到create_engine方法中。

本次升级是为了解决开发旧版本的SQLAlchemy时出现的问题，以便更好地提升其性能。