.. _migration_14_toplevel:

=============================
SQLAlchemy 1.4有哪些新内容？
=============================

.. admonition:: 关于本文档

    本文档介绍了SQLAlchemy版本1.3和SQLAlchemy版本1.4之间的更改。

    1.4版本在很多方面与其他SQLAlchemy版本有所不同，它试图作为一个潜在的迁移点，迁移对象是当前计划发布的2.0 API更改。SQLAlchemy 2.0的重点是现代化、精简化的API，去除很多长期以来被反对的使用模式，并将SQLAlchemy中最好的思想作为第一类API特性，目的是API的使用更加明确，移除了一系列影响性能和加重内部复杂度的隐式行为和很少使用的API标记。

    关于SQLAlchemy 2.0的当前状态，请参见：  :ref:`migration_20_toplevel` 。

主要API更改和功能-常规
=========================

.. _change_5634:

Python 3.6是最低的Python 3版本；Python 2.7仍然受支持
-------------------------------------------------------------

由于Python 3.5已于2020年9月到期，因此SQLAlchemy 1.4现在将版本3.6作为最低要求版本。Python 2.7仍然受支持，但SQLAlchemy 1.4系列将是最后一个支持Python 2的系列。


.. _change_5159:

ORM查询与select、update、delete内部统一;提供2.0的方式执行
------------------------------------------------------------------------------------------

SQLAlchemy 2.0和SQLAlchemy 1.4最大的概念变化，基本上是去除了Core中的  :class:`_sql.Select`  _ orm.Query.update` 和  :meth:` _ orm.Query.delete`  方法与 :class:` _ dml.Update`和 :class:`_dml.Delete` 有关。

关于  :class:`_sql.Select` ，这两个对象在很多版本中具有相似的重叠API，甚至有一些能够在两者之间进行更改的能力，同时仍然在它们的使用模式和行为上有很大的不同。之所以出现这种历史背景，是因为 :class:` _orm.Query`对象被引入来克服 :class:`_sql.Select` 对象的某些缺点，因为它们必须在 :class:`_schema.Table` 元数据的基础上进行查询。但是 :class:`_orm.Query` 仅具有一种简单的接口，用于加载对象，并且仅在许多重大版本中才获得了大部分的 :class:`_sql.Select` 对象的灵活性，这导致了一些尴尬的事情，这两个对象变得非常相似，但仍然在很大程度上与对方不兼容。

在1.4版本中，所有的Core和ORM SELECT语句都是直接从一个  :class:`_sql.Select`  执行在内部调用。在未来，  :class:` _orm.Query`  执行，该执行方式允许Core构造自由地针对ORM实体使用::

    with Session(engine, future=True) as sess:
        stmt = (
            select(User)
            .where(User.name == "sandy")
            .join(User.addresses)
            .where(Address.email_address.like("%gmail%"))
        )

        result = sess.execute(stmt)

        for user in result.scalars():
            print(user)

关于上面的示例需要注意以下几点：

*   :class:`_orm.Session` ` with:`` 语句）功能；请参见：ref:`session_getting`。

* 在1.4系列中，所有  :term:`2.0 style`   ORM调用都使用设置  :paramref:` _orm.Session.future`  标志为``True``的  :class:`_orm.Session` ；此标志表示， :class:` _orm.Session`应具有2.0样式的行为，包括ORM查询可以从 :class:`_orm.Session.execute` 调用，以及一些事务特性的更改。在2.0版本中，此标志将始终为“True”。

*   :func:`_sql.select` 。

*   :func:`_sql.select` /  :class:` _sql.Select`  方法，其作用类似于  :class:`_orm.Query` 。

* 通过预期返回ORM结果并且被期望的语句使用  :meth:`.orm.Session.execute`  进行调用。请参见：  :ref:` session_querying_20` 。另请参见以下说明：  :ref:`change_session_execute_result` 。

* 返回  :class:`_engine.Result` ， :ref:` change_4710_core`和 :ref:`change_4710_orm` 了解详细信息。

在SQLAlchemy的文档中，将有许多对  :term:`1.x style`  和  :term:` 2.0 style`  执行的引用。这是为了区分两种查询样式并尝试向前记录新的调用样式。在SQLAlchemy 2.0中，虽然 :class:`_orm.Query` 对象可能仍然是一个遗留构造，但它将不再出现在大多数文档中。

相似的调整已经被应用于"批量更新和删除" ，使得Core中的 :func:`_sql.update` 和 :func:`_sql.delete` 可以用于批量操作。现在，像下面这样的批量更新：

    session.query(User).filter(User.name == "sandy").update(
        {"password": "foobar"}, synchronize_session="fetch"
    )

可以通过  :term:`2.0 style`  （实际上以上面的方式内部运行）来实现：

    with Session(engine, future=True) as sess:
        stmt = (
            update(User)
            .where(User.name == "sandy")
            .values(password="foobar")
            .execution_options(synchronize_session="fetch")
        )

        sess.execute(stmt)

请注意使用  :meth:`_sql.Executable.execution_options`  方法来传递ORM相关选项。现在，在Core和ORM之间，许多来自  :class:` _orm.Query`  以获取一些示例）。

.. seealso::

      :ref:`migration_20_toplevel` 

  :ticket:`5159`  


.. _change_session_execute_result:

ORM“Session.execute()”在所有情况下使用“future”样式“Result”集
--------------------------------------------------------------------------

如：  :ref:`change_4710_core`  参数设置为 ` `True``的 :class:`_engine.Engine` 时， :class:`_engine.Result` 和 :class:`_engine.Row` 对象现在具有“匿名元组”行为。

在使用  :paramref:`_sa.create_engine.future`  参数设置为“False”时，将返回传统的“LegacyRow”对象，它具有之前SQLAlchemy版本的部分命名元组行为，其中包括与列名具有相同的行为。

在使用  :meth:`_orm.Session.execute`  时，**完整的命名元组样式**永远**被启用**，这意味着` `"name" in row``将使用**值内容**作为测试，而不是**键内容**。这是因为  :meth:`_orm.Session.execute`  现在返回一个  :class:` _engine.Result` ，它还适用于ORM结果，即使是传统ORM结果，如  :meth:`_orm.Query.all`  返回的结果行，也使用值包含。

这是从SQLAlchemy 1.3到1.4的行为更改。为了继续接收键包含集合，请使用  :meth:`_engine.Result.mappings`  方法来获得  :class:` _engine.MappingResult` ，该方法以字典形式返回行：

    for dict_row in session.execute(text("select id from table")).mappings():
        assert "id" in dict_row

.. _change_4639:

所有DQL、DML语句现在都包含透明的SQL编译缓存
----------------------------------------------------------------------------------

这是SQLAlchemy单个版本中最广泛的变化之一，经过几个月的所有查询系统的重新组织和重构，从Base Core开始一直到ORM，现在允许在使用用户构造的语句时，大多数涉及生成SQL字符串和相关语句元数据的Python计算都可以在内存中缓存，随后对相同语句构造的任何后续调用都将使用35-60%更少的CPU资源。

这种缓存不仅限于构造SQL字符串，还包括构造将SQL构造链接到结果集的结果提取结构，在ORM中还包括ORM启用的属性加载程序、关系预加载器和其他选项以及由ORM查询从结果集运行和构建ORM对象必须构建的对象构造例程。

为了介绍该特性的一般思路，请参见  :ref:`examples_performance`  ：

    session = Session(bind=engine)
    for id_ in random.sample(ids, n):
        result = session.query(Customer).filter(Customer.id == id_).one()

此示例在Dell XPS13上运行Linux的1.3版本中完成如下所示：

.. sourcecode:: text

    test_orm_query : (10000 iterations); total time 3.440652 sec

在1.4中，未经修改的上述代码完成：

.. sourcecode:: text

    test_orm_query : (10000 iterations); total time 2.367934 sec

这个测试表明，使用缓存的常规ORM查询运行多次的速度可以提高**30%**。

特性的第二个变体是为了使用Python lambdas延迟查询本身的构造而选择的。这是由版本1.0.0
引入的"烤查询"扩展的更复杂的变体，该"lambda"特性可以以与烤查询类似的方式使用，不同之处在于它对于任何SQL构造都可用。它还包括扫描每次lambda调用以查找在每次调用中更改的绑定文字值以及其他结构（例如，每次查询都从不同的实体或列中查询），而仍然不必每次运行实际代码。

使用此API如下所示::

    session = Session(bind=engine)
    for id_ in random.sample(ids, n):
        stmt = lambda_stmt(lambda: future_select(Customer))
        stmt += lambda s: s.where(Customer.id == id_)
        session.execute(stmt).scalar_one()

上面的代码完成：

.. sourcecode:: text

    test_orm_query_newstyle_w_lambdas : (10000 iterations); total time 1.247092 sec

这个测试表明，使用新的"select()"ORM查询样式以及完整的缓存风格调用使得运行超过许多迭代的循环速度提高了**60%**，并提供了大约与最新的烤查询系统相同的性能。

新系统利用了现有的  :paramref:`_engine.Connection.execution_options.compiled_cache`  执行选项，并直接添加到了  :class:` _engine.Engine`  的  :paramref:`_query_cache_size`  参数进行配置。

整个1.4中的API和行为更改受到了支持这一新特性的影响。


.. seealso::

      :ref:`sql_caching` 

  :ticket:`4639`  
  :ticket:`5380`  
  :ticket:`4645`  
  :ticket:`4808`  
  :ticket:`5004`  

.. _change_5508:

声明式现在与ORM集成并具有新功能
-------------------------------------------------------------

在流行了十多年之后，``sqlalchemy.ext.declarative``包现已与``sqlalchemy.orm``命名空间集成，但声明式"扩展"类除外。

添加到``sqlalchemy.orm``中的新类包括：

*   :class:`_orm.registry` -一个新的类，取代了"声明式基类"类的作用，充当映射类的注册表，可以通过名称字符串在 :func:` _orm.relationship`调用中引用，并且不关心任何特定类的映射方式。

*   :func:`_orm.declarative_base`  - 这是整个声明式系统中一直在使用的声明式基类，除了它现在在内部引用  :class:` _orm.registry`  方法实现的，可以直接从  :class:`_orm.registry` ` sqlalchemy.ext.declarative.declarative_base``名称仍然存在，当启用 :ref:`2.0 deprecations mode <deprecation_20_mode>` 时，将发出2.0弃用警告。

*   :func:`_orm.declared_attr`  - 相同的“声明属性”函数调用现在已经是` `sqlalchemy.orm``的一部分了。 ``sqlalchemy.ext.declarative.declared_attr``名称仍然存在，当启用 :ref:`2.0 deprecations mode <deprecation_20_mode>` 时，将发出2.0弃用警告。

* 其他移至``sqlalchemy.orm``的名称包括  :func:`_orm.has_inherited_table` ,   :func:` _orm.synonym_for` ,   :class:`_orm.DeclarativeMeta` ,   :func:` _orm.as_declarative` 。

此外，  :func:`_declarative.instrument_declarative`  取代。  :class:` _declarative.ConcreteBase` ,   :class:`_declarative.AbstractConcreteBase` ,和 :class:` _declarative.DeferredReflection`仍然作为扩展保留在 :ref:`declarative_toplevel` 包中。

映射样式现已组织，它们全都从 :class:`_orm.registry` 对象扩展，并分为以下类别：

*   :ref:`orm_declarative_mapping` 
    * 使用   :func:`_orm.declarative_base`  Base class w/ metaclass
        *   :ref:`orm_declarative_table` 
        *   :ref:`Imperative Table <orm_imperative_table_configuration>`  (即“混合表”)
    * 使用  :meth:`_orm.registry.mapped`  声明式装饰器
        * 声明式表
        * Imperial Table（Hybrid） 
            *   :ref:`orm_declarative_dataclasses` 
*   :ref:`Imperative <orm_imperative_mapping>`  （即“古典”映射）
    * 使用  :meth:`_orm.registry.map_imperatively` 
        *   :ref:`orm_imperative_dataclasses` 

现有的经典映射函数``sqlalchemy.orm.mapper()``仍然存在，但是在直接调用``sqlalchemy.orm.mapper()``上，它已被弃用；新的  :meth:`_orm.registry.map_imperatively`  方法现在将请求路由到  :meth:` _orm.registry`  ，以使其与其他声明式映射不会产生歧义。

新方法还与第三方类仪表盘系统集成，这些系统必须先对类进行调整，以便与声明式映射配合使用，允许声明式映射以装饰器而不是声明式基类的形式工作，以便像dataclasses_和attrs_这样的包可以与声明式映射一起使用，除了使用传统映射。

声明式文档现已完全集成到ORM映射器配置文档中，并包括针对所有映射样式的示例，这些示例组织在一个地方。请参阅：ref:`orm_mapping_classes_toplevel`，以开始重新组织的文档。


.. _dataclasses: https://docs.python.org/3/library/dataclasses.html
.. _attrs: https://pypi.org/project/attrs/

.. seealso::

    :ref:`orm_mapping_classes_toplevel` 

    :ref:`change_5027` 

  :ticket:`5508`  


.. _change_5027:

Python Dataclasses和attrs与声明式、经典映射互相兼容
-----------------------------------------------------------------------

除了在  :ref:`change_5508`  中引入的新声明式装饰样式之外，  :class:` _orm.Mapper` `dataclasses``模块，并且将识别以这种方式配置的属性，并继续映射它们，这与以前的情况不同，以前类似的属性会被跳过。对于``attrs``模块，``attrs``已经将其自己的属性从类中删除，因此已与SQLAlchemy经典映射兼容。现在有了  :meth:`_orm.registry.mapped`  装饰器，这两个属性系统也可以与声明式映射一起使用。

.. seealso::

    :ref:`orm_declarative_dataclasses` 

    :ref:`orm_imperative_dataclasses` 

  :ticket:`5027`  


.. _change_3414:

Core和ORM现在支持异步IO
------------------------------------------

SQLAlchemy现在支持Python“asyncio”兼容的数据库驱动程序，使用新的asyncio前端接口连接

.. note:: 新的asyncio功能应被认为是**alpha level**曾经在SQLAlchemy 1.4版本中。

首选的数据库API是PostgreSQL的  :ref:`dialect-postgresql-asyncpg`  asyncio驱动程序。

SQLAlchemy的内部功能完全集成，使用`greenlet <https://greenlet.readthedocs.io/en/latest/>`_库以适应SQLAlchemy的内部流的执行，以将asyncio“await”关键字从数据库驱动程序向外传播到端用户API，该API具有async方法。使用此方法，asyncpg驱动程序在SQLAlchemy自己的测试套件内完全可用，并支持大多数psycopg2功能。来自greenlet项目开发者的开发人员经过验证和改进此方法，因此感激SQLAlchemy。

用户面向的“async”API本身集中在IO导向的方法中，例如  :meth:`_asyncio.AsyncEngine.connect`  和  :meth:` _asyncio.AsyncConnection.execute`  。新的Core构造仅支持  :term:`2.0 style`  的用法；这意味着所有语句必须给出连接对象，即：class:` _asyncio.AsyncConnection`。

在ORM中，支持  :term:`2.0 style`  查询执行，使用  :func:` _sql.select`  配合使用；不支持传统的 :class:`_orm.Query` 对象。注：
也就是说，ORM特性（例如懒加载相关属性和过期属性）这些会隐式地运行在Python IO中，所以在ORM中的常规**异步**应用程序中，应该谨慎使用:eager loading <loading_toplevel>技术，以及避免使用像 :ref:`expire on commit <session_committing>` 等的特性，因此不需要进行这样的加载。

对于选择与传统不同的应用程序开发人员   :class:`_asyncio.AsyncSession.run_sync`  方法提供了**完全可选的功能**，可以将与数据库相关的代码组织到函数中，在:greenlet环境下执行函数利用  :meth:` _asyncio.AsyncSession.run_sync`  。请参阅：  :ref:`examples_asyncio`  中的` `greenlet_orm.py`` 示例。

还提供了异步游标支持，使用新方法  :meth:`_asyncio.AsyncConnection.stream`  和  :meth:` _asyncio.AsyncSession.stream`  ，它们支持新的  :class:`_asyncio.AsyncResult`  和 meth:` _asyncio.AsyncResult.fetchmany`。Core和ORM都与该功能集成，这对应于在传统SQLAlchemy中使用“服务器端游标”时的情况。

.. seealso::

    :ref:`asyncio_toplevel` 

    :ref:`examples_asyncio` 

  :ticket:`3414`  

.. _change_deferred_construction:


许多Core和ORM语句对象现在在编译阶段执行大量构建和验证
--------------------------------------------------------------------------------------------------------------

1.4系列的一个重大倡议是以允许高效、可缓存的语句创建和编译模型来考虑Core SQL语句和ORM查询的方式，其中编译步骤将基于由创建语句对象生成的缓存密钥缓存，这些对象本身每次使用都会新创建。为此，特别是ORM查询 :class:`_query.Query` 的Python计算的大多数部分，以及当 :func:`_sql.select` 构造用于调用ORM查询时涉及的一些Python计算，正在被移动到的语句编译阶段，在语句已被调用并且只有在语句的编译形式没有被缓存时才会发生编译阶段。

从最终用户的角度来看，这意味着，某些根据参数传递到对象中而引发的错误消息不会立即引发，而是只有在语句首次被调用时才会发生。这些条件始终是结构性的，而不是数据驱动的，因此不会因为缓存语句而错过这种情况。

基于此类别的错误条件包括：

* 当构建  :class:`_selectable.CompoundSelect` （例如UNION、EXCEPT等）时，传递的SELECT语句不具有相同数量的列时，现在会引发  :class:` .CompileError` ；以前，在语句构造时会立即引发  :class:`.ArgumentError` 。

* 启动  :meth:`.Query.join`  时可能出现的各种错误条件将在语句编译时进行评估，而不是在第一次调用该方法时调用。

其他可能发生更改的是 :class:`_orm.Query` 直接涉及的对象：

* 调用  :attr:`_orm.Query.statement`  访问器的行为可能稍有不同。返回的  :class:` _sql.Select`  等方法，可能需要适应了FROM子句。

.. seealso::

      :ref:`change_4639` 

.. _change_4656:修复了内部导入规则，以便代码检查工具可以正确工作
---------------------------------------------------

长期以来，SQLAlchemy一直使用一个参数注入装饰器来帮助解决模块之间相互依赖的问题，例如：


    @util.dependency_for("sqlalchemy.sql.dml")
    def insert(self, dml, *args, **kw):
        ...


上面的函数被重写，不再在外部具有“dml”参数。这会导致代码检查工具看到缺少函数参数。现在已经实现一种新的方法，使得函数的签名不再被修改，而是在函数内部获得模块对象。


  :ticket:`4656`  

  :ticket:`4689`  


.. _change_1390:

支持 SQL 正则表达式运算符
--------------------------------------------

期待已久的特性，增加了对数据库正则表达式运算符的基本支持，以补充  :meth:`_sql.ColumnOperators.like`  和
  :meth:`_sql.ColumnOperators.match`   操作组。 新功能包括：  :meth:` _sql.ColumnOperators.regexp_match`  实施一个
正则表达式匹配函数，和  :meth:`_sql.ColumnOperators.regexp_replace`  实现正则表达式字符串替换功能。

受支持的后端包括 SQLite、PostgreSQL、MySQL/MariaDB 和 Oracle。SQLite 后端仅支持“regexp_match”，而不支持
“regexp_replace”。

正则表达式的语法和标志是不依赖后端的。未来的版本将允许一次指定多个正则表达式语法来在不同的后端之间切换。

对于 SQLite，Python 的“re.search()”函数是已经确定的实现。

.. 也可以看看：


     :meth:`_sql.ColumnOperators.regexp_match` 

     :meth:`_sql.ColumnOperators.regexp_replace` 

      :ref:`pysqlite_regexp`  - SQLite 实现说明书


  :ticket:`1390`  


.. _deprecation_20_mode:

SQLAlchemy 2.0 退化模式
---------------------------------

1.4 版本的主要目标之一是提供一个“过渡”版本，以便应用程序可以逐步迁移到 SQLAlchemy 2.0。为此，1.4 版本的主要
功能之一是“2.0 退化模式”，这是一系列的退化警告，对每个可检测的 API 模式都会发出警告，这些 API 模式在版本 2.0 中
将有所不同。所有的警告都使用了   :class:`_exc.RemovedIn20Warning`  类。由于这些警告涉及到基础模式，
包括   :func:`_sql.select`  和   :class:` _engine.Engine`  构造，即使是简单的应用程序也可能会生成许多警告，直到进行
适当的 API 更改。因此，直到开发人员启用环境变量 “SQLALCHEMY_WARN_20=1” 为止，警告模式将默认关闭。

有关使用 2.0 退化模式的完整步骤，请参阅   :ref:`migration_20_deprecations_mode` 。

.. 也可以参见：

    :ref:`migration_20_toplevel` 

    :ref:`migration_20_deprecations_mode` 



API 和行为上的变化 - 核心
==================================

.. _change_4617:

不再将 SELECT 语句隐式视为 FROM 子句
--------------------------------------------------------------------------

这是多年来 SQLAlchemy 中最大的概念性变化之一，但希望对最终用户的影响相对较小，因为变化更加符合 MySQL 和
PostgreSQL 等数据库的要求。

最明显的影响是，现在无法直接在另一个   :func:`_expression.select`  中嵌入   :func:` _expression.select` 。
可以使用  :meth:`_expression.SelectBase.alias`  方法，将内部的   :func:` _expression.select`  显式转换为子查询。
但是，这些方法仍然可以使用新的  :meth:`_expression.SelectBase.subquery`  方法，这个方法和  :meth:` _expression.SelectBase.alias` 
方法基本相同。 最终返回的对象是   :class:`.Subquery` ，它非常类似于   :class:` _expression.Alias`  对象，并且共享一个公共的
基本   :class:`.AliasedReturnsRows` 。

即，现在会引发：

    stmt1 = select(user.c.id, user.c.name)
    stmt2 = select(addresses, stmt1).select_from(addresses.join(stmt1))

抛出：

.. sourcecode:: text

    sqlalchemy.exc.ArgumentError: 需要列表达式或 FROM 子句，而得到了 <...Select 对象 ...>。要从一个 <class
    'sqlalchemy.sql.selectable.Select'> 对象创建 FROM 子句，请使用 .subquery() 方法。

正确的调用方式为（还请注意，不再需要为 select() 使用方括号 <change_5284>）：


    sq1 = select(user.c.id, user.c.name).subquery()
    stmt2 = select(addresses, sq1).select_from(addresses.join(sq1))

上面提到的  :meth:`_expression.SelectBase.subquery`  方法基本等同于使用  :meth:` _expression.SelectBase.alias`  方法。


这种变化的理由基于以下事实：

* 为了支持将   :class:`_sql.Select`  与   :class:` _orm.Query`  统一起来，  :class:`_sql.Select`  对象需要具有
   :meth:`_sql.Select.join`  和  :meth:` _sql.Select.outerjoin`  方法，它们实际上将 JOIN 条件添加到现有 FROM 子句中，
  这是用户一直期望的行为。以前的行为是必须与   :class:`.FromClause`  的行为保持一致，即生成一个未命名的子查询，
  然后加入到该查询中，这仅仅会让那些不幸尝试这样做的用户感到困惑而已。这种变化在   :ref:`change_select_join`  中讨论。

* 在 FROM 子句中包含 SELECT，而不首先创建别名或子查询的行为将导致创建一个未命名的子查询。虽然标准 SQL 支持此
  语法，但实际上大多数数据库都会拒绝它。例如， MySQL 和 PostgreSQL 都直接拒绝未命名子查询 的用法：

  .. sourcecode:: sql

      # MySQL / MariaDB:

      MariaDB [(none)]> select * from (select 1);
      ERROR 1248 (42000): Every derived table must have its own alias


      # PostgreSQL:

      test=> select * from (select 1);
      ERROR:  subquery in FROM must have an alias
      LINE 1: select * from (select 1);
                            ^
      HINT:  For example, FROM (SELECT ...) [AS] foo.

  像 SQLite 这样的数据库接受它们，但通常情况下，从这样的子查询生成的名称太模糊了，无法使用：

  .. sourcecode:: sql

      sqlite> CREATE TABLE a(id integer);
      sqlite> CREATE TABLE b(id integer);
      sqlite> SELECT * FROM a JOIN (SELECT * FROM b) ON a.id=id;
      Error: ambiguous column name: id
      sqlite> SELECT * FROM a JOIN (SELECT * FROM b) ON a.id=b.id;
      Error: no such column: b.id

      # 给它取一个名字
      sqlite> SELECT * FROM a JOIN (SELECT * FROM b) AS anon_1 ON a.id=anon_1.id;

  ..

由于   :class:`_expression.SelectBase`  对象不再是   :class:` _expression.FromClause`  对象，因此像 ``.c`` 属性这样的属性，
以及像 ``.select()`` 这样的方法现已弃用，因为它们暗示了对子查询的隐式生成。 ``.join()`` 和 ``.outerjoin()`` 方法现
在已经被   :ref:`重新定位为将 JOIN 条件附加到现有查询 <change_select_join>` ，这是用户一直希望这些方法所做的事情。

为了代替 ``.c`` 属性，添加了一个新属性  :attr:`_expression.SelectBase.selected_columns` 。
该属性解析为一个列集合，是大多数人希望 ``.c`` 完成的操作（但并没有）。一个常见的初学者错误是下面这样的代码：


    stmt = select(users)
    stmt = stmt.where(stmt.c.name == "foo")

上述代码看起来很合理，应该会生成 "SELECT * FROM users WHERE name='foo'"，但是老练的 SQLAlchemy 用户将会认识到，
它实际上生成了一个类似 "SELECT * FROM (SELECT * FROM users) WHERE name='foo'" 的无用的子查询。

然而，新的属性  :attr:`_expression.SelectBase.selected_columns`  符合上面的用例，对于上述情况，它直接连接到
``users.c`` 集合中的列：


    stmt = select(users)
    stmt = stmt.where(stmt.selected_columns.name == "foo")

  :ticket:`4617`  


.. _change_select_join:

select().join() 和 outerjoin() 将添加 JOIN 条件到当前查询中，而不是创建子查询
-------------------------------------------------------------------------------------------------------

为了实现“2.0 风格”的   :class:`_sql.Select`  和   :class:` _orm.Query`  的统一，尤其是对于  :term:`2.0 style`  的
  :class:`_sql.Select`  使用方式，必须首先确保  :meth:` _sql.Select.join`  方法可以正确添加 JOIN 条件到现有 SELECT 当中，
而不是将该对象包装在一个无名的子查询中，并从该子查询返回加入的 JOIN，这是一个完全无用的特性，只会让使用这些的
不幸者感到困惑。这种变化在   :ref:`change_4617`  中讨论。

从那时开始，由于  :meth:`_sql.Select.join`  和  :meth:` _sql.Select.outerjoin`  已经存在一种现有的行为，最初
计划将这些方法弃用，并将新的“有用”版本的方法作为一个单独的“未来”   :class:`_sql.Select`  对象可用于另外一个导入。

然而，在工作了一段时间后，我们决定：有两种不同类型的   :class:`_sql.Select`  对象存在的情况是更加具有误导性和
不方便的，因此我们决定硬性改变这两个方法的行为，而不是等待一年时间，过渡期间会让API更加的笨拙。虽然 SQLAlchemy 的
开发人员不轻易做出完全破坏性的改变，但这是一个非常特殊的案例，以前实现这些方法的人基本上不会使用；
正如   :ref:`change_4617`  中所述，主要数据库（如 MySQL 和 PostgreSQL）实际上不允许使用未命名的子查询，从句法
上看，使用未命名子查询的 JOIN 最为困难，因为无法明确地引用其中的列。

使用新的实现，  :meth:`_sql.Select.join`   和  :meth:` _sql.Select.outerjoin` 
现在的行为与  :meth:`_orm.Query.join`  非常相似，将 JOIN 条件添加到现有语句中匹配左侧的实体：：

    stmt = select(user_table).join(
        addresses_table, user_table.c.id == addresses_table.c.user_id
    )

生成：

.. sourcecode:: sql

    SELECT user.id, user.name FROM user JOIN address ON user.id=address.user_id

与   :class:`_sql.Join`  一样，如果有可能自动确定 ON 子句：

    stmt = select(user_table).join(addresses_table)

当语句中使用 ORM 实体时，这本质上是使用  :term:`2.0 style`  调用构建 ORM 查询的方式。ORM 实体将在内部将
“插件”分配给语句，因此当语句编译为 SQL 字符串时，将使用 ORM 相关的编译规则。更明确地说，  :meth:`_sql.Select.join`  
方法可以容纳 ORM 关系，而不会破坏 Core 和 ORM 内部之间的硬分离。因此，还添加了一个新方法  :meth:`_sql.Select.join_from` ，
它允许更容易地一次指定连接的左右两个部分：

    stmt = select(Address.email_address, User.name).join_from(User, Address)

生成：

.. sourcecode:: sql

    SELECT address.email_address, user.name FROM user JOIN address ON user.id == address.user_id


.. 也可以参见：

    :class:`_sql.Select` 


Changes to CreateEnginePlugin
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_engine.CreateEnginePlugin`  也受到了这个变化的影响，因为自定义插件的文档指出应使用` `dict.pop()``方法从
URL 对象中删除使用过的参数。现在应该使用  :meth:`_engine.CreateEnginePlugin.update_url`  方法。反向兼容的方法看起来像：


    from sqlalchemy.engine import CreateEnginePlugin


    class MyPlugin(CreateEnginePlugin):
        def __init__(self, url, kwargs):
            # 检查是否 2.0 风格
            if hasattr(CreateEnginePlugin, "update_url"):
                self.my_argument_one = url.query["my_argument_one"]
                self.my_argument_two = url.query["my_argument_two"]
            else:
                # 老效果
                self.my_argument_one = url.query.pop("my_argument_one")
                self.my_argument_two = url.query.pop("my_argument_two")

            self.my_argument_three = kwargs.pop("my_argument_three", None)

        def update_url(self, url):
            # 此方法仅在 1.4 中运行，并应用于消耗插件特定参数
            return url.difference_update_query(["my_argument_one", "my_argument_two"])

详见   :class:`_engine.CreateEnginePlugin`  中关于如何使用这个类的文档字符串。

  :ticket:`5526`  


.. _change_5284:

select()、case() 现在接受位置表达式
---------------------------------------------------

正如在本文档的别处可能所见，  :func:`_sql.select`  构造现在将接受“列子句”参数的位置，而不是要求将其作为列表进行传递。


    # 新方式，支持 2.0
    stmt = select(table.c.col1, table.c.col2, ...)

当使用位置参数发送参数时，不允许使用任何其他关键字参数。在 SQLAlchemy 2.0 中，上述调用样式将是唯一支持的调用样式。

在 1.4 的整个时期中，原始调用样式仍将继续运行，它会将列子句或其他表达式作为一个列表进行传递：


    # 旧方式，在 1.4 中仍然有效
    stmt = select([table.c.col1, table.c.col2, ...])

上面的古老调用样式也接受了早期被大多数叙述文档删除的旧关键字参数。这些关键字参数的存在让列子句在第一次时作为列表传递。
例如：


    # 旧方式，但在1.4中仍然有效
    stmt = select([table.c.col1, table.c.col2, ...], whereclause=table.c.col1 == 5)

位置参数和列表的检测基于第一个位置参数是否为列表这一事实。不幸的是，仍然可能会出现以下一些使用情况，
在这种情况下，省略了“whereclause”的关键词：


    # 旧方式，但在1.4中仍然有效
    stmt = select([table.c.col1, table.c.col2, ...], table.c.col1 == 5)

作为这种变化的一部分，也修改了   :func:`_sql.case`  构造，使其接受其 WHERE 子句列表的位置，与旧调用样式相同。
其余的依赖于列表表达的 SQLAlchemy 构造，如  :meth:`_sql.ColumnOperators.in_` ，也基于扩展 IN 功能。所以上下文中出现
“使用位置参数进行结构性说明，使用列表进行数据说明”的惯例，例如  :meth:`_sql.ColumnOperators.in_`  表达式使用列表。
：class:`_sql.Select` 构造还具有 2.0 风格的“未来” API，其中包括更新的  :meth:`.Select.join`  方法
以及如  :meth:`.Select.filter_by`  和  :meth:` .Select.join_from`  这样的方法。

相关变化：  :ref:`migration_20_5284` 、  :ref:` error_c9ae` 


  :ticket:`5284`  

.. _change_4645:

所有的 IN 表达式将即兴地针对列表中的每个值呈现参数（例如扩展参数）
---------------------------------------------------

“扩展 IN”功能首次在   :ref:`change_3953`  中引入，现在已经成熟到足以显然优于以前的渲染 IN 表达式的方法了。
随着该方法的改进可以处理空值列表，现在它是 Core/ORM 渲染 IN 参数列表的唯一方法。

之前的方法是，在将值列表传递给  :meth:`.ColumnOperators.in_`  方法时，该列表会在语句构建时扩展成单个个体
  :class:`.BindParameter`  对象序列。这个方法的缺点是无法根据参数字典在语句执行时伸缩参数列表，这意味着无法将字符串
SQL 语句与其参数独立缓存，并且不能完全使用参数字典来处理包含 IN 表达式的语句。

为了为   :ref:`baked_toplevel`  中描述的“baked query”功能提供服务，需要 IN 的可缓存版本，这就带来了“扩展 IN”功能。
与现有行为不同，即在语句构建时将参数列表扩展为多个   :class:`.BindParameter`  对象，这个功能使用单个 class：` !BindParameter`，
一次存储所有值的值列表；当   :class:`_engine.Engine`  执行语句时，它根据传递给  :meth:` _engine.Connection.execute`  的参数，
以及已从先前执行中检索到的现有 SQL 字符串，根据正在执行的参数集将其“展开”为单个绑定参数位置。
这允许相同的  :class:`.Compiled`  对象多次调用，可以针对 IN 表达式所传递的不同参数集调用它，而仍然保持贴近标量的参数
作为 DBAPI 传递。虽然某些 DBAPI 直接支持此功能，但通常不可用；扩展 IN 功能现在为所有后端都提供了一致的行为。

1.4 的主要焦点之一是允许在 Core 和 ORM 中进行真正的语句缓存，而不需要“烘焙”系统的笨拙，因此，现在每当将值列表
传递给 IN 表达式时，它自动调用“扩展 IN”特性。

例如：          

    stmt = select(A.id, A.data).where(A.id.in_([1, 2, 3]))


表达式的预执行字符串表示为：

.. sourcecode:: pycon+sql

    >>> print(stmt)
    {printsql}SELECT a.id, a.data
    FROM a
    WHERE a.id IN ([POSTCOMPILE_id_1])


直接呈现值的方法，与以前的方法相同：


    stmt = select(A.id, A.data).where(A.id.in_([1, 2, 3]).literal_binds)


.. 也可以看看：


      :ref:`baked_toplevel` 

      :ref:`change_3953` 

``"CAST(data AS VARCHAR)"`` - we always apply a label to a column expression that gets
a name from the   :class:`_expression.Column`  object involved, or other named
object such as a   :class:`_expression.Label`  or   :class:` _expression.TextualColumn`  object.
The above query would appear in SQLAlchemy as:

.. sourcecode:: python

    from sqlalchemy import cast, VARCHAR

    query = select(cast(foo.c.data, VARCHAR))

    # prints: SELECT CAST(foo.data AS VARCHAR) AS "data" FROM foo
    print(query)

As the name is taken from the   :class:`_expression.Column`  object, the above
query has the name ``"data"`` applied to it.  However, the result of this query,
when fetched, will produce a row with a label derived from the full SQL
expression "``CAST(data AS VARCHAR)``", and not a label of just ``"data"``.

A change was made to fully apply the name of the column to the label
generated by expressions like CAST, so that the result column label becomes
"``data``" instead of "``CAST(data AS VARCHAR)``" under PostgreSQL:

.. sourcecode:: python

    query = select(cast(foo.c.data, VARCHAR).label(foo.c.data.name))

    # prints: SELECT CAST(foo.data AS VARCHAR) AS data FROM foo
    print(query)

In this example, the name is explicitly applied to the label; of course,
the name would already be present for most   :class:`_expression.Column`  objects,
so it only need be done when creating an expression that is not otherwise
bound to a named column.

  :ticket:`4449`  上述功能现在SQLAlchemy将对此类表达式应用自动标签，这些表达式一直是所谓的“匿名”表达式：

.. sourcecode:: pycon+sql

    >>> print(select(cast(foo.c.data, String)))
    {printsql}SELECT CAST(foo.data AS VARCHAR) AS anon_1     # old behavior
    FROM foo

这些匿名表达式是必要的因为SQLAlchemy的  :class:`_engine.ResultProxy` .String` 数据类型以前具有结果行处理行为以便正确地匹配到正确列，因此名称最重要的是它们必须适用于数据库无关的方式并且在所有情况下都是唯一的。在SQLAlchemy 1.0中作为:ticket："918"的一部分减少了对结果行中命名列（特别是PEP-249游标的“cursor.description”元素）的依赖，因此大多数核心SELECT构造都不需要使用命名的列；在1.4版本中，整个系统变得更加舒适，适用于拥有重复列或标签名称的SELECT语句，例如  :ref:`change_4753` 。因此，我们现在模拟PostgreSQL对单个列进行简单修改的合理行为，其中最突出的一点是使用CAST：

.. sourcecode:: pycon+sql

    >>> print(select(cast(foo.c.data, String)))
    {printsql}SELECT CAST(foo.data AS VARCHAR) AS data
    FROM foo

对于没有名称的表达式的CAST，使用以前的逻辑生成通常的“匿名”标签：

.. sourcecode:: pycon+sql

    >>> print(select(cast("hi there," + foo.c.data, String)))
    {printsql}SELECT CAST(:data_1 + foo.data AS VARCHAR) AS anon_1
    FROM foo

对于  :class:`.Label` ，尽管由于这些不会在CAST内部呈现，因此必须省略标签表达式名称，但仍将使用给定名称：

.. sourcecode:: pycon+sql

    >>> print(select(cast(("hi there," + foo.c.data).label("hello_data"), String)))
    {printsql}SELECT CAST(:data_1 + foo.data AS VARCHAR) AS hello_data
    FROM foo

当然，一直以来都是如此， :class:`.Label` 可以应用于外部的表达式以直接应用“AS <name>”标签：

.. sourcecode:: pycon+sql

    >>> print(select(cast(("hi there," + foo.c.data), String).label("hello_data")))
    {printsql}SELECT CAST(:data_1 + foo.data AS VARCHAR) AS hello_data
    FROM foo


:ticket：`4449`

.. _change_4808:

Oracle，SQL Server中的新“后编译”绑定参数用于LIMIT / OFFSET
-------------------------------------------------------------------------------

1.4系列的主要目标是确保所有Core SQL结构都是完全可缓存的，这意味着特定的 :class:`.Compiled` 结构将生成相同的SQL字符串，而不管与之一起使用的任何SQL参数，这通常包括用于分页和“ top N”样式结果的限制和偏移量。

虽然SQLAlchemy多年来一直使用绑定参数进行限制/偏移量方案，但仍有一些不太正常的情况，在这种情况下不允许使用这些参数，包括SQL
Server的“TOP N”语句，例如：

.. sourcecode:: sql

    SELECT TOP 5 mytable.id, mytable.data FROM mytable

以及对于Oracle，FIRST_ROWS（）提示（如果使用了Oracle URL并传递了“ optimize_limits = True”参数到: func：`_sa.create_engine`），不允许它们，但同时，已经报告使用已绑定参数的ROWNUM比较会产生较慢的查询计划：

.. sourcecode:: sql

    SELECT anon_1.id, anon_1.data FROM (
        SELECT / * + FIRST_ROWS（5）* /
        anon_2.id AS id，
        anon_2.data AS data，
        ROWNUM AS ora_rn
从（
        SELECT mytable.id，mytable.data FROM mytable
    ）anon_2
        WHERE ROWNUM <=：param_1
    ）anon_1 WHERE ora_rn>：param_2

为了在编译级别上允许所有语句不受条件地可缓存，增加了一种名为“后编译”参数的绑定形式，其使用与“展开IN参数”相同的机制。这是一个：func：`.bindparam`，在行为上与任何其他参数绑定方法相同，除了该参数值将在发送到DBAPI ``cursor.execute（）``方法之前被字面呈现到SQL字符串中。 SQL Server和Oracle方言在内部使用了新参数，以便驱动程序接收到文字呈现的值，但SQLAlchemy的其余部分仍可将其视为绑定参数。将两个语句字符串化为``str（statement.compile（dialect = <dialect>））``时，现在看起来如下所示：

.. sourcecode:: sql

    SELECT TOP [POSTCOMPILE_param_1] mytable.id, mytable.data FROM mytable

和：

.. sourcecode:: sql

    SELECT anon_1.id, anon_1.data FROM (
        SELECT /*+ FIRST_ROWS([POSTCOMPILE__ora_frow_1]) */
        anon_2.id AS id,
        anon_2.data AS data,
        ROWNUM AS ora_rn FROM (
            SELECT mytable.id, mytable.data FROM mytable
        ) anon_2
        WHERE ROWNUM <= [POSTCOMPILE_param_1]
    ) anon_1 WHERE ora_rn > [POSTCOMPILE_param_2]

“[POSTCOMPILE_ <param>]”格式也是使用“展开IN”时看到的形式。

在查看SQL日志输出时，将看到语句的最终形式：

.. sourcecode:: sql

    SELECT anon_1.id, anon_1.data FROM (
        SELECT /*+ FIRST_ROWS(5) */
        anon_2.id AS id,
        anon_2.data AS data,
        ROWNUM AS ora_rn FROM (
            SELECT mytable.id AS id, mytable.data AS data FROM mytable
        ) anon_2
        WHERE ROWNUM <= 8
    ) anon_1 WHERE ora_rn > 3


通过:paramref：`。bindparam.literal_execute`参数公开“后编译参数”功能，但目前不打算用于一般用途。使用:meth：`.TypeEngine.literal_processor`渲染文字值的底层数据类型，其中在SQLAlchemy中的范围**极为有限**，仅支持整数和简单字符串值。

  :ticket:`4808`  

.. _change_4712:

基于子事务，连接级事务现在可以处于非活动状态
----------------------------------------------

现在可以在  :class:`_engine.Connection` .Transaction` 是由于内部事务中的回滚而变为非活动状态，但是 :class:`.Transaction` 不会清除直到本身被回滚。

这实际上是一种新的错误状态，如果内部“子”事务已经回滚了，将阻止在  :class:`_engine.Connection` .Session` 非常相似，其中如果已经开始了外部事务，则需要将其回滚以清除无效事务；此行为在:ref：'faq_session_rollback'中说明。

虽然  :class:`_engine.Connection` .Session` 更宽松，但是由于它有助于确定子事务已回滚DBAPI事务，但是外部代码并不知道这一点，并且尝试继续进行，这确实会导致在新事务上运行操作。在 :ref:`session_external_transaction` 处将会出现这种情况的常见地方使用“测试套件”模式。

Core和ORM的“子事务”功能本身已被弃用，在2.0中将不再存在。因此，子事务操作的此新错误状态本身是暂时的，因为一旦删除子事务，就不再适用。

为了使用不包括子事务的2.0样式行为，请在: func：`_sa.create_engine`上使用:paramref：`future`参数。

错误消息在错误页面中说明: ref：`错误_8s2a`。

.. _change_5367:

Enum和Boolean数据类型不再默认为“create constraint”
--------------------------------------------------------

现在，:paramref：`。Enum.create_constraint`和:paramref：`。Boolean.create_constraint`参数默认为False，表示创建这两种数据类型的“非本机”版本时，**不会**生成CHECK约束。这些CHECK约束会带来模式管理维护复杂性，应该选择而不是默认打开。

要确保为这些类型发出CREATE CONSTRAINT，请将这些标志设置为“True” ::

    class Spam(Base):
        __tablename__ = "spam"
        id = Column(Integer, primary_key=True)
        boolean = Column(Boolean(create_constraint=True))
        enum = Column(Enum("a", "b", "c", create_constraint=True))

  :ticket:`5367`  

ORM的新功能
=============

.. _change_4826:

列的Raiseload
---------------

使用:paramref：`.orm.defer.raiseload`参数的  :class:`_query.Query` .raiseload` 选项与此相同。

对于映射的列级raiseload配置，可以使用:paramref：`.deferred.raiseload`参数。然后可以在查询时使用:func：`.undefer`选项来急切地加载属性::：

    class Book(Base):
        __tablename__ = "book"

        book_id = Column(Integer, primary_key=True)
        title = Column(String(200), nullable=False)
        summary = deferred(Column(String(2000)), raiseload=True)
        excerpt = deferred(Column(Text), raiseload=True)


    book_w_excerpt = session.query(Book).options(undefer(Book.excerpt)).first()

最初考虑到的是将现有的:class：`。raiseload`选项扩展到支持列导向属性。但是，这将打破:func：`。raiseload`的“通配符”行为，该行为被记录为允许阻止所有关系加载：：

    session.query(Order).options(joinedload(Order.items), raiseload("*"))

以上，如果我们将:func：`。raiseload`扩展为同时支持列表达式和关系，则通配符也将阻止列加载，并且对于没有添加新API的情况下仍然不清楚如何实现关系加载的上面的效果。因此，列的选项仍保持在:func：`.defer`上：

      :func:`.raiseload`  - query option to raise for relationship loads

     :paramref:`.orm.defer.raiseload`  - query option to raise for column expression loads


作为此更改的一部分，发生了“ 被推迟”的行为与属性过期的属性过期行为已更改。以前，当对象被标记为过期并通过访问其中一个过期属性使得未映射为“被延迟”的属性也将加载时，属性将“过期”。现在，映射器中被延迟的属性永远不会“过期”，仅在作为延迟加载器的一部分被访问时才会加载。

在映射为“未被延迟”的属性的情况下被作为标记为“延迟加载”选项的查询时间，该属性将在对象被过期时重置。这与以前存在的行为相同。

.. seealso::

      :ref:`orm_queryguide_deferred_raiseload` 

  :ticket:`4826`  

.. _change_5263:

ORM批量插入使用psycopg2现在在大多数情况下使用RETURNING批量语句
---------------------------------------------------------------------------------

在  :ref:`change_5401` ` execute_values（）``扩展程序的psycopg2方言。所有ORM刷新过程现在都使用此功能，以便能够获取新生成的主键值和服务器默认值，同时不会失去将多个INSERT语句组合在一起的性能优势。此外，psycopg2的``execute_values（）``扩展程序本身提供了五倍的性能改进，通过将一个INSERT语句重写为在一个语句的许多“VALUES”表达式中包含多个表达式，而不是重复调用相同的语句来实现，因为psycopg2缺少像预准备语句那样准备语句以便此方法变得有效。

SQLAlchemy在其示例中包括一个:ref：`performance suite <examples_performance>`，在其中我们可以将“batch_inserts”运行程序的时间与1.3和1.4进行比较，显示对于大多数插入的批处理类型实现了3倍到5倍的加速：

.. sourcecode:: text

    # 1.3
    $ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
    test_flush_no_pk : (100000 iterations); total time 14.051527 sec
    test_bulk_save_return_pks : (100000 iterations); total time 15.002470 sec
    test_flush_pk_given : (100000 iterations); total time 7.863680 sec
    test_bulk_save : (100000 iterations); total time 6.780378 sec
    test_bulk_insert_mappings :  (100000 iterations); total time 5.363070 sec
    test_core_insert : (100000 iterations); total time 5.362647 sec

    # 1.4 with enhancement
    $ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
    test_flush_no_pk : (100000 iterations); total time 3.820807 sec
    test_bulk_save_return_pks : (100000 iterations); total time 3.176378 sec
    test_flush_pk_given : (100000 iterations); total time 4.037789 sec
    test_bulk_save : (100000 iterations); total time 2.604446 sec
    test_bulk_insert_mappings : (100000 iterations); total time 1.204897 sec
    test_core_insert : (100000 iterations); total time 0.958976 sec

请注意，默认情况下该功能将行批处理到每个1000组中，这可以影响: ref：`psycopg2_executemany_mode`中记录的``executemany_values_page_size``参数。

  :ticket:`5263`  


.. _change_orm_update_returning_14:

ORM批量更新和删除使用一旦可用就使用RETURNING进行“提取”策略
-------------------------------------------------------------

使用“ 提取”策略的ORM批量更新或删除::

    sess.query(User).filter(User.age > 29).update(
        {"age": User.age - 10}, synchronize_session="fetch"
    )

现在将使用RETURNING（如果后端数据库支持）；这目前包括PostgreSQL和SQL Server（Oracle方言不支持多行RETURNING）：：

.. sourcecode:: text

    UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s RETURNING users.id
    [generated in 0.00060s] {'age_int_1': 10, 'age_int_2': 29}
    Col ('id',)
    Row (2,)
    Row (4,)

对于不支持RETURNING多个行的后端，仍将使用以前的向前请求先发出的SELECT：

.. sourcecode:: text

    SELECT users.id FROM users WHERE users.age_int > %(age_int_1)s
    [generated in 0.00043s] {'age_int_1': 29}
    Col ('id',)
    Row (2,)
    Row (4,)
    UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s
    [generated in 0.00102s] {'age_int_1': 10, 'age_int_2': 29}

此更改的棘手之处之一是在水平分片扩展中的情况下，其中单个批量更新或删除可能会在一些后端之间多路复用，其中一些支持RETURNING，而另一些则不支持。新的1.4执行架构支持了这种情况，以便“提取”策略可以保持不变，从而可以优雅地退化为使用SELECT，而不是必须添加新的“返回”策略，该策略不是后端通用的。

作为此更改的一部分，现在“提取”策略的效率大大提高，因为它将不再过期所定位的与行匹配的对象，用于可以在Python中求值的SET子句中使用的Python表达式; 相反，仅对于不能求值的SQL表达式，它才会退回到过期属性。用于值无法求值的“评估”策略已经得到改进，以便对于无法求值的值返回“过期”。

ORM的行为变化
===================

.. _change_4710_orm:

查询返回的“键值元组”对象更改为行
-----------------------------------

如:ref：`change_4710_core`所述，Core的  :class:`.RowProxy` .Row` 的类。现在，基本的:class：`.Row`对象的行为更像是命名元组，因此现在它用作由以前的“KeyedTuple”类返回的与元组相关的结果的基础。

这样做的目的是使得到SQLAlchemy 2.0，Core和ORM SELECT语句将使用类似命名元组的  :class:`.Row` .Row._mapping` 属性，可以从:class：`.Row`中获得字典类似的功能。在此期间，Core结果集将使用:class：`.Row`子类“LegacyRow”，该子类仍保留以前的dict / tuple混合行为，以向后兼容，而将在ORM元组结果中直接使用:class：`.Row`类返回由:class：`_query.Query`对象返回。

已投入大量的工作，以使：class：`.Row`的大多数功能集在ORM中可用，这意味着按字符串名称以及实体/列访问的应该工作::

    row = s.query(User, Address).join(User.addresses).first()

    row._mapping[User]  # same as row[0]
    row._mapping[Address]  # same as row[1]
    row._mapping["User"]  # same as row[0]
    row._mapping["Address"]  # same as row[1]

    u1 = aliased(User)
    row = s.query(u1).only_return_tuples(True).first()
    row._mapping[u1]  # same as row[0]


    row = s.query(User.id, Address.email_address).join(User.addresses).first()

    row._mapping[User.id]  # same as row[0]
    row._mapping["id"]  # same as row[0]
    row._mapping[users.c.id]  # same as row[0]

.. seealso::

      :ref:`change_4710_core` 

  :ticket:`4710`  .

.. _change_5074:

会话功能新的“自动开始”行为
-----------------------------

以前，在默认模式“autocommit = False”的情况下，:class：`.Session`会立即在构造时内部开始完成：class：`.SessionTransaction`对象，并且此外会在每次调用:meth：`.Session.rollback`或:meth：`.Session.commit`之后创建一个新的事务。

新行为是仅在需要时才在调用:meth：`.Session.add`或:meth：`.Session.execute`等方法时就会即时创建此类:class：`.SessionTransaction`对象（“ autocommit = False”）模式下，但是也现在可以显式调用:meth：`.Session.begin`以便开始事务即使在“ autocommit = False”模式中也是如此，从而匹配将来样式:class：`_base.Connection`。

这表明的行为更改是：

* :class：`.Session`现在可以处于无事务状态，即使在“ autocommit = False”模式下。 以前，此状态仅在“自动提交”模式下可用。
* 在此状态下，:meth：`.Session.commit`和:meth：`.Session.rollback`方法都是无操作的。 依赖这些方法来使所有对象过期的代码应明确使用:meth：`.Session.begin`或:meth：`.Session.expire_all`以适应其用例。
* 当:class：`.Session`最初构造时或:meth：`.Session.rollback`或:meth：`.Session.commit`完成时，:meth：`.SessionEvents.after_transaction_create`事件挂钩也不会发出事件。
* :meth：`.Session.close`方法也不会暗示隐式启动新的:class：`.SessionTransaction`.

.. seealso::

      :ref:`session_autobegin` 

原理
^^^^^

:class：`.Session`对象在其默认模式“ autocommit = False”中意味着始终存在与:class：`.SessionTransaction`
对象关联的:class：`.SessionTransaction`对象通过 :attr：`.Session.transaction`属性。 当放置
:class：`.SessionTransaction`完成之后，基于组成为 :class：`.SessionTransaction`
对象仅想一并指定对象的状态与理解的行为，而 :class：`.SessionTransaction`
本身并不意味着使用任何与连接相关的资源，因此长期以来，这种默认行为具有特殊的优雅性。
例如，在制作 :meth：`.Session.commit` 然后是  :meth：`.Session.close`的情况下发生了什么。

但是，在 :ticket：5056 改进所述的减少参考循环的倡议中，这种假设意味着当调用 :meth：`.Session.close`
时，这个假设将后果是   :class:`.Session`  对象仍然存在引用循环，并 更昂贵的方式进行清理，更不用说有点开销
在构建:class：`.SessionTransaction`对象，这意味着在例如调用以前，为这样的 :class：`.Session` 带来令人费解的性能代价
:meth：`.Session.commit`然后是 :meth：`.Session.close`。

因此，决定 :meth：`.Session.close`要用None离开内部状态 :attr：`.Session.transaction`，现在称为内部状态 :attr：`.Session._transaction` ，只有在需要时才会创建：class：`.SessionTransaction`
对象。出于一致性和代码覆盖的原因，将这种行为也扩展到包括“自动开始”的每个点，而不仅仅是在调用 :meth：`.Session.close`时。

特别是，这需要为订阅 :meth：`.SessionEvents.after_transaction_create`事件挂钩的应用程序标识符改变；以前，当第一次组合 :class：`.Session` 对象时，此事件将会被发射，就像处理了大多数将 :meth：`.Session.rollback`或:meth：`.Session.commit`结束并发出 :meth：`.SessionEvents.after_transaction_end`的操作一样。新行为是在需要时发出 :meth：`.SessionEvents.after_transaction_create`，当:class：`.Session`尚未创建新:class：`.SessionTransaction`对象并且映射对象 关联的会话线程 未初始化 （“ autocommit = False”时 仅限）。

.. seealso::

      :ref:`session_autobegin` 

  :ticket:`5074`  .当  :attr:` .Session.transaction`  被调用时，在调用  :meth:`.Session.add `  和  :meth:` .Session.delete`  之类的方法时，以及在  :meth:`.Session.flush`  方法有任务完成时等都会在方法中发出。
此外，依赖于  :meth:`.Session.commit`   或  :meth:` .Session.rollback`  方法的代码使所有对象不受条件限制地到期，现在不再这样做。需要在没有发生更改时使所有对象过期的代码应为此调用  :meth:`.Session.expire_all`  。
除了后面描述的  :meth:`.Session.commit`  或  :meth:` .Session.rollback`  的noop性质，该变更对  :class:` Session`  Session` 将继续具有在调用  :meth:` Session.close`  后可用于新操作的行为，以及 :class:` _engine.Engine`和数据库本身的交互的排序方式也应该保持不受影响，因为这些操作已经按需运作了。
:ticket:5074

.. _change_5237_14:

非常重要不要转换原文中的标点符号
只翻译描述性语言文本。

可查看的关系不会同步回传递引用
-------------------------------------------------

在1.3.14中的  :ticket:`5149`  中，当且仅当  :paramref:` _orm.relationship.backref`  或  :paramref:`_orm.relationship.back_populates`  关键字与目标关系中的  :paramref:` _orm.relationship.viewonly`  标志同时使用时，SQLAlchemy开始发出警告。这是因为“只查看”关系实际上不会保留所做的更改，这可能会导致出现一些误导性的行为。但是，在  :ticket:`5237`  中，我们试图改善这种行为，因为在仅查看关系上设置后向引用是合法用例，包括在某些情况下后向填充属性由关系惰性加载器用于确定不需要在另一个方向上进行附加的急切加载，以及后向填充可以用于映射器内省， :func:` _orm.backref`也可以是设置双向关系的便捷方法。

那么解决方案是使后向引用的“突变”成为可选项，使用  :paramref:`_orm.relationship.sync_backref`  标志。1.4的  :paramref:` _orm.relationship.sync_backref`  值默认为False，用于设置  :paramref:`_orm.relationship.viewonly`  的关系目标。这表示通过viewonly查看的关系上的任何更改都不会对另一侧或 :class:` _orm.Session`的状态产生影响：
```python
    class User(Base):
        # ...

        addresses = relationship(Address, backref=backref("user", viewonly=True))


    class Address(Base):
        ...


    u1 = session.query(User).filter_by(name="x").first()

    a1 = Address()
    a1.user = u1
```
上面，“a1”对象将**不会**添加到“u1.addresses”集合中，也不会将“a1”对象添加到会话中。在以前，这两个结果都将成立。现在不再发出  :paramref:`.relationship.sync_backref`  应被设置为 ` `False``的警告，因为现在这是默认行为。
:ticket:5237

.. _change_5150:

级联反向引用在2.0中被弃用以移除
-------------------------------------------------------

SQLAlchemy一直有一个基于反向引用分配的级联对象到  :class:`_orm.Session` ::
```python
    u1 = User()
    session.add(u1)

    a1 = Address()
    a1.user = u1  # <--- adds "a1" to the Session
```
以上行为是反向引用行为的意外副作用，在因为“a1.user”隐含着“u1.addresses.append(a1)”时，“a1”会被级联到  :class:`_orm.Session`  来禁用上述行为，以及  :paramref:` _orm.backref.cascade_backrefs`  用于在关系由“relationship.backref”指定时设置，因为它可能会令人惊讶，并妨碍一些操作，在哪个对象将被过早地放置在 :class:`_orm.Session` 中并提前刷新。
在2.0中，默认行为将是“cascade_backrefs”为False，此外也没有“True”行为，因为这通常不是理想行为。当启用2.0弃用警告时，发生“backref cascade”时将发出警告。要获得新行为，请将  :paramref:`_orm.relationship.cascade_backrefs`  设置为` `False``在任何目标关系上，就像在1.3和更早版本中已经支持的那样，或者使用  :paramref:`_orm.Session.future`  标志，将其应用于  :term:` 2.0=style `  模式，如下所示：
```python
    Session = sessionmaker(engine, future=True)

    with Session() as session:
        u1 = User()
        session.add(u1)

        a1 = Address()
        a1.user = u1  # <--- will not add "a1" to the Session
```
:ticket:5150

.. _change_1763:

急切加载器在取消过期操作期间发出
---------------------------------------------

长期以来，一直在寻求一种行为，即当访问取消过期的对象时，配置急切加载器将运行，以便在取消刷新对象或未过期时急切加在过期对象上的关系。在2.0中，级联操作将检查是否存在依赖项，这被添加因为实际上只是在2.0中的主加载循环周围添加了距离。
这适用于直接应用于  :func:`_orm.relationship`  的选项，前提是对象最初是由该查询加载的。
对于“二级”急切加载器“selectinload”和“subqueryloading”，这些加载器不需要SQL策略来急切加载单个对象的属性；因此，在刷新方案下，它们会调用“立即加载”策略，该策略类似于“lazyload”发出的查询，作为其他查询的额外查询：
```python
    >>> a1 = session.query(A).options(selectinload(A.bs)).first()
    >>> a1.data = "new data"
    >>> session.commit()

```
上面，通过with_expression加载的“A1”对象将“bs”集合转换为内联JOIN，然后将“new data”将添加到“a1”。如果“flush()”或“commit()”触发复制操作，则可以将数据提交到数据库中。
:ticket:1763

.. _change_8879:

如“deferred()”、 “with_expression()”的列加载器只在外部完整实体查询上生效

.. 注意：此更改注释早期版本的文档中不存在，但对所有SQLAlchemy 1.4版本都相关。
在1.3和以前的版本中从未支持的行为，但仍会产生特定效果的行为是，将列加载器选项，例如 :func:`_orm.defer` 和 :func:`_orm.with_expression` 重新用于子查询，以便控制每个子查询的列子句中的SQL表达式。一个典型的例子是构建UNION查询，如下所示：
```python
    q1 = session.query(User).options(with_expression(User.expr, literal("u1")))
    q2 = session.query(User).options(with_expression(User.expr, literal("u2")))

    q1.union_all(q2).all()
```
在1.3中， :func:`_orm.with_expression` 选项将在每个UNION元素中生效，例如：
```sql
    SELECT anon_1.anon_2 AS anon_1_anon_2, anon_1.user_account_id AS anon_1_user_account_id,
    anon_1.user_account_name AS anon_1_user_account_name
    FROM (
        SELECT ? AS anon_2, user_account.id AS user_account_id, user_account.name AS user_account_name
        FROM user_account
        UNION ALL
        SELECT ? AS anon_3, user_account.id AS user_account_id, user_account.name AS user_account_name
        FROM user_account
    ) AS anon_1
    ('u1', 'u2')
```
SQLAlchemy 1.4的加载器选项的概念已经变得更加严格，并且在所有SQLAlchemy版本中仅应用于顶层的实体查询（SELECT）预期为要填充实际要返回的ORM实体；查询将以1.4版本的上面的方式产生：
```sql
    SELECT ? AS anon_1, anon_2.user_account_id AS anon_2_user_account_id,
    anon_2.user_account_name AS anon_2_user_account_name
    FROM (
        SELECT user_account.id AS user_account_id, user_account.name AS user_account_name
        FROM user_account
        UNION ALL
        SELECT user_account.id AS user_account_id, user_account.name AS user_account_name
        FROM user_account
    ) AS anon_2
    ('u1',)
```
这意味着 :func:`_orm.Query` 的所有加载器选项都将从UNION的第一个元素中复制到查询的最高水平，并且仅从第一个元素获取选项，而忽略任何其他查询部分中的选项。上述情况下，对第二个查询的选项将被忽略。
原因
--------
此行为现在更接近同样针对其他类型的加载器选项，如  :meth:`_orm.joinedload`  的关系加载器选项，它们在UNION情况下已经拷贝到查询的最高层级，仅从UNION的第一个元素获取，丢弃任何其他查询部分中的选项。上述情况的隐式复制和选择性忽略选项的行为是  :class:` _orm.Query`  的缺陷之一，因为它不清楚将单个SELECT转换为其本身和另一个查询的UNION如何将加载器选项应用到该新语句，并且没有考虑在该新语句中应用加载器选项的方式。
1.4版本的行为可证明是优于1.3年的行为，对于更常见的使用 :func:`_orm.defer` 来说尤其如此。下面的查询：
```python
    q1 = session.query(User).options(defer(User.name))
    q2 = session.query(User).options(defer(User.name))

    q1.union_all(q2).all()
```
在1.3年会向内部查询尴尬地添加NULL，然后将其选择出来：
```sql
    SELECT anon_1.anon_2 AS anon_1_anon_2, anon_1.user_account_id AS anon_1_user_account_id
    FROM (
        SELECT NULL AS anon_2, user_account.id AS user_account_id
        FROM user_account
        UNION ALL
        SELECT NULL AS anon_2, user_account.id AS user_account_id
        FROM user_account
    ) AS anon_1
```
如果所有查询没有完全相同的选项安装，以上的情况将引发错误，由于在 :func:`_orm.Query.union_all` 操作中的对象将处于 :class:`_orm.Session` 中，因此这会遇到多个错误。
相比之下，在1.4中，选项仅适用于顶层查询，省略了``User.name``的获取，避免了这种复杂性：
```sql
    SELECT anon_1.user_account_id AS anon_1_user_account_id
    FROM (
        SELECT user_account.id AS user_account_id, user_account.name AS user_account_name
        FROM user_account
        UNION ALL
        SELECT user_account.id AS user_account_id, user_account.name AS user_account_name
        FROM user_account
    ) AS anon_1
```
正确的方法
---------
使用  :term:`2.0-style`  查询，目前不会发出警告，但了嵌套中的 :func:` _orm.with_expression`选项是一致地被忽略，因为它们不适用于将要加载的实体，并且不会自动复制到任何地方。下面的查询不会为 :func:`_orm.with_expression` 调用产生输出：
```python
    s1 = select(User).options(with_expression(User.expr, literal("u1")))
    s2 = select(User).options(with_expression(User.expr, literal("u2")))

    stmt = union_all(s1, s2)

    session.scalars(select(User).from_statement(stmt)).all()
```
产生的SQL：
```sql
    SELECT user_account.id, user_account.name
    FROM user_account
    UNION ALL
    SELECT user_account.id, user_account.name
    FROM user_account
```
要将  :func:`_orm.with_expression` ` User``实体，请将其应用于查询的最高层，分别使用每个SELECT中的普通SQL表达式：
```python
    s1 = select(User, literal("u1").label("some_literal"))
    s2 = select(User, literal("u2").label("some_literal"))

    stmt = union_all(s1, s2)
    session.scalars(
        select(User)
        .from_statement(stmt)
        .options(with_expression(User.expr, stmt.selected_columns.some_literal))
    ).all()
```
将产生预期的SQL：
```sql
    SELECT user_account.id, user_account.name, ? AS some_literal
    FROM user_account
    UNION ALL
    SELECT user_account.id, user_account.name, ? AS some_literal
    FROM user_account
```
``User``对象本身将在``User.expr``下包含该表达式。
: ticket:4519

.. _change_4662:

“新实例与现有标识冲突 ”错误现在是警告
-------------------------------------------------- -----

SQLAlchemy一直有逻辑来检测要插入的 :class:`.Session` 中是否存在具有与已经存在的对象相同的主键的对象::
```python
    class Product(Base):
        __tablename__ = "product"

        id = Column(Integer, primary_key=True)


    session = Session(engine)

    session.add(Product(id=1))
    session.flush()

    session.add(Product(id=1))
    s.commit()  # <-- will raise FlushError
```
该更改是将 :class:`.FlushError` 更改为只是警告：
.. sourcecode:: text
    sqlalchemy/orm/persistence.py:408: SAWarning: New instance <Product at 0x7f1ff65e0ba8> with identity key (<class '__main__.Product'>, (1,), None) conflicts with persistent instance <Product at 0x7f1ff60a4550>
在这之后，条件将尝试将行插入数据库，这将发出  :class:`.IntegrityError` ，如果主键标识已经不存在于 :class:` .Session`中，则会引发此错误。
其理由是使得使用  :class:`.IntegrityError` .Session` 的现有状态的情况下运行，同时仍在使用savepoints时被使用。例如：
```python
    try:
        with session.begin_nested():
            session.add(Product(id=1))
    except exc.IntegrityError:
        print("row already exists")
```
上述逻辑在早期是不可行的，因为在存在标识符的情况下，对任何带有异常的代码块由于会同时引发 :class:`.FlushError` 导致该“Product”对象无法合并。随着更改，获取函数的警告输出成功并且现有代码被改造为正确参考比较。
由于涉及主键的逻辑，所有数据库在插入时会出现主键冲突时发出完整性错误。其中不会发出错误的情况是理论上，即将映射到联接的表或未实际在数据库模式中约束的复合主键的附加列的情况。
 :class:`.IntegrityError` 捕获重复条目。 警告可以使用Python警告过滤器设置为引发异常。
: ticket:4662

.. _change_4994:

禁止与viewonly=True相关的持久性相关的级联操作
--------------------- ---------------------------------------------------
当使用  :paramref:`_orm.relationship.viewonly`  标志将  :func:` _orm.relationship` `viewonly=True``时，表示应该仅使用此关系从数据库加载数据，并且不应对其进行改变或涉及持久性操作。为了确保这个契约成功工作，关系不能再指定从“viewonly”方面不合理的  :paramref:`_orm.relationship.cascade`  设置。
主要目标是“删除，删除孤儿”级联回收，即使viewonly为真，也将通过父级被删除，如果对象被删除或分离，一个对象仍然会级联这两个操作到关联对象中。而不是修改级联回收以检查viewonly，而是直接禁止这两个操作配置在一起：
```python
    class User(Base):
        # ...

        # this is now an error
```addresses = relationship("Address", viewonly=True, cascade="all, delete-orphan")

上述代码将会抛出以下异常:

.. sourcecode:: text

    sqlalchemy.exc.ArgumentError: 级联设置
    "delete, delete-orphan, merge, save-update" 用于执行操作，不能在
    viewonly=True 关联对象中使用。

由于应用程序出现此问题，因此自SQLAlchemy 1.3.12开始会发出警告，针对上述错误，解决方法是删除对viewonly关联关系的级联设置。

  :ticket:`4993`  
  :ticket:`4994`  

.. _change_5122:

使用自定义查询查询继承映射时查询方式更加严格
------------------------------------------------

本更改适用于查询面向已完成SELECT子查询的联接或单个表继承子类实体的情况。
如果给定的子查询返回与请求的多态标识或标识不对应的行，则会引发错误。
以前，此条件在联接表继承中会默默地通过，返回一个无效的子类，
而在单表继承中， :class:`_query.Query` 会添加额外的条件限制子查询，
这可能会与查询的意图不当干预。

在1.3系列中，如果针对联接属性映射发出以下查询：

    s = Session(e)

    s.add_all([Engineer(), Manager()])

    s.commit()

    print(s.query(Manager).select_entity_from(s.query(Employee).subquery()).all())

子查询选择了ESngineer和Manager两行，即使外部查询是针对Manager的，
我们也会得到一个非Manager的对象的返回：

.. sourcecode:: text

    SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
    FROM (SELECT employee.type AS type, employee.id AS id
    FROM employee) AS anon_1
    2020-01-29 18:04:13,524 INFO sqlalchemy.engine.base.Engine ()
    [<__main__.Engineer object at 0x7f7f5b9a9810>, <__main__.Manager object at 0x7f7f5b9a9750>]

新的更改是当遇到不符合请求的多态类型时会引发错误:

.. sourcecode:: text

    sqlalchemy.exc.InvalidRequestError: 行的标识键
    (<class '__main__.Employee'>, (1,), None)无法加载到一个对象中;
    多态鉴别列 '%(140205120401296 anon)s.type' 引用的
    映射类 Engineer->engineer不是要求的映射类 Manager->manager 的子映射类。

上述错误仅在该实体的主键列为非NULL时引发。如果该行的给定实体没有主键，
则不会尝试构造实体。

在单一继承映射的情况下，行为的更改稍微更为复杂；如果Engineer和Manager 
以上是使用单一表继承映射，那么在1.3中，将发出以下查询，
并仅返回"Manager"对象：

.. sourcecode:: text

    SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
    FROM (SELECT employee.type AS type, employee.id AS id
    FROM employee) AS anon_1
    WHERE anon_1.type IN (?)
    2020-01-29 18:08:32,975 INFO sqlalchemy.engine.base.Engine ('manager',)
    [<__main__.Manager object at 0x7ff1b0200d50>]

：class：`_query.Query` 将“单表继承”的标准添加到子查询中，对查询意图进行了编辑。
这个行为是在1.0中的 :ticket:`3891` 中添加的，它在"联接”和"单表”继承之间
创建了一种行为上的不一致，同时还改变了给定查询的意图，这可能意图
返回其中的列与继承实体对应的列均为 NULL 的更多行，这是一个有效的用例。
现在，行为等同于联接表继承，其中假定子查询返回正确的行，
如果遇到意外的多态类型，就会引发错误:

.. sourcecode:: text

    SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
    FROM (SELECT employee.type AS type, employee.id AS id
    FROM employee) AS anon_1
    2020-01-29 18:13:10,554 INFO sqlalchemy.engine.base.Engine ()
    Traceback (most recent call last):
    # ...
    sqlalchemy.exc.InvalidRequestError: 行的标识键
    (<class '__main__.Employee'>, (1,), None) 无法加载到一个对象中;
    多态鉴别列 '%(140700085268432 anon)s.type' 引用的
    映射类 Engineer->employee 不是要求的映射类 Manager->employee 的子映射类。

如上所示，应对上述情况的正确方法是调整给定的子查询，
以正确地基于鉴别器列过滤行：

    print(
        s.query(Manager)
        .select_entity_from(
            s.query(Employee).filter(Employee.discriminator == "manager").subquery()
        )
        .all()
    )

.. sourcecode:: sql

    SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
    FROM (SELECT employee.type AS type, employee.id AS id
    FROM employee
    WHERE employee.type = ?) AS anon_1
    2020-01-29 18:14:49,770 INFO sqlalchemy.engine.base.Engine ('manager',)
    [<__main__.Manager object at 0x7f70e13fca90>]


  :ticket:`5122`  

方言变化
========

pg8000最低版本1.16.6，仅支持Python 3
--------------------------------------

通过该项目的维护者的帮助，大大改进了 pg8000 方言的支持。

由于API更改，pg8000方言现在需要1.16.6及更高版本。 pg8000系列已
自1.13系列开始不支持Python 2。需要使用pg8000的Python 2用户应确
保其要求固定于 "SQLAlchemy<1.4"。

  :ticket:`5451`  

PostgreSQL psycopg2方言需要2.7或更高版本
----------------------------------------—

psycopg2方言依赖于过去几年中发布的psycopg2的许多功能。为了
简化dialect，现在的最低版本要求数为2017年3月发布的版本2.7。

.. _change_5941:

psycopg2方言不再限制绑定参数名
---------------------------------

SQLAlchemy 1.3不能在psycopg2方言下容纳包括百分号或括号在内的绑
定参数名称。这么做意味着包括这些字符的列名也存在问题，因为插入
和其他DML语句将生成与该列匹配的参数名，这将导致失败。解决方法是
使用  :paramref:`_schema.Column.key`  参数的访问控制这个问题，
从而产生一个替代名称来生成参数，或者在   :func:`_sa.create_engine` 
级别更改dialect的参数样式。从SQLAlchemy 1.4.0beta3开始，所有命名限
制均已删除，并且在所有情况下参数都完全转义，因此这些解决方法
不再需要。

  :ticket:`5941`  

  :ticket:`5653`  

.. _change_5401:

psycopg2方言默认情况下使用"execute_values"插入语句和RETURNING 
----------------------------------------------------------------

针对 PostgreSQL，旨在实现ORM和Core的显著性能改进的前半部分，
psycopg2方言现在默认情况下使用“psycopg2.extras.execute_values()”
对已编译的INSERT语句进行优化，并在此模式下实现了RETURNING支持。
该更改的另一半  :ref:`change_5263` 使ORM可以利用带有executemany的
RETURNING(即批量插入INSERT语句)而使ORM使用psycopg2的批量插入可以
根据具体情况的不同实现快达4倍的速度提升。

这个扩展方法允许使用扩展的VALUES子句来在单个语句中插入多个行。
虽然SQLAlchemy的  :func:`_sql.insert` 构造已经通过该方法支持此语法
通过  :meth:`_sql.Insert.values`  方法，但是扩展方法允许在“executemany”
执行时动态地构造VALUES从而会在  :meth:`._Engine.Connection.execute` 
将参数字典的列表传递给时发生。它还在缓存界限之外发生，以便在渲染
VALUES之前可以将INSERT语句缓存。

通过在   :ref:`examples_performance`  示例套件中的 bulk_inserts.py
脚本中快速测试 ``execute_values()`` 方法来显示该方法的性能优势：
最初的版本也是扩展此构造方法:

.. sourcecode:: text

    $ python -m examples.performance bulk_inserts --test test_core_insert --num 100000 --dburl postgresql://scott:tiger@localhost/test

    # 1.3
    test_core_insert :使用单个Core INSERT语句批量插入映射。(100000 iterations);总时间5.229326秒

    # 1.4
    test_core_insert :使用单个Core INSERT语句批量插入映射。(100000 iterations);总时间0.944007秒

扩展功能方法允许在执行插入语句时返回生成的行。如果给定的   :func:`._sql.insert` 
构造请求通过  :meth:`.Insert.returning`  方法或类似方法来返回生成的默认值，
则psycopg2方言现在将检索此列表；然后这些行被安装在结果中，就像
它们直接从游标中返回一样。这使得像ORM这样的工具可以在所有场景下使用批量插入，
这将提供显着的性能改进。

psycopg2方言的“executemany_mode”功能已作以下更改：

* 添加了一个名为“ values_only”的新模式。该模式
  对于使用已编译的INSERT语句的executemany()使用非常快的
  ``psycopg2.extras.execute_values()``扩展方法，但是不对
  UPDATE和DELETE语句使用``execute_batch（）``。这种新模式现
  在是psycopg2方言的默认设置。

* 现有的“值”模式现在被命名为"values_plus_batch"。此模式将使用 
  ``execute_values`` 用于INSERT语句，并使用 ``execute_batch`` 
   用于UPDATE和DELETE语句。此模式不是默认启用的，因为它使用
  ``executemany（）``执行UPDATE和DELETE语句时会禁用``cursor.rowcount``的
   正确功能。

* INSERT语句的RETURNING支持已启用“ values_only”和“ values”
  为。如果给定的   :func:`._sql.insert`  构造请求
  通过  :meth:`.Insert.returning`  方法或类似方法来返回生成的默认值，
  则psycopg2方言现在将接收从psycopg2回调的行，
  并将其安装在结果中，就像它们直接来自游标一样。
  这允许像ORM这样的工具在所有情况下使用批量插入，
  从而提供了明显的性能改进。

* “execute_values”扩展功能的默认“page_size”设置从100增加到1000。
   对于 ``execute_batch`` 该默认值仍为 100。这些参数
  可以像以前一样进行修改。

* 1.2中添加了对“batch”扩展的支持
    :ref:`change_4109` ，并在1 .3中通过
   :ticket:`4623` ` execute_values``扩展。
  在1.4版本中，``execute_values`` 扩展现在默认
  用于INSERT语句；UPDATE和DELETE的“batch”扩展默认仍关闭。

此外，``execute_values`` 扩展功能支持返回由RETURNING生成的行作为聚合列表。
如果给定的   :func:`._sql.insert` .Insert.returning` 方法
或类似方法来返回生成的默认值，则psycopg2方言将获取此列表；然后，这些行被安装在结果中，
就像它们直接来自游标一样。这使得像ORM这样的工具可以在所有场景下使用批量插入， 这将提供显着的性能改进。

Core engine和dialect已增强以通过提供新的
   :attr:`_engine.CursorResult.inserted_primary_key_rows`  和 
   :attr:`_engine.CursorResult.returned_default_rows`  类型支持批量插入
  以及返回模式。目前只在 psycopg2中提供。

.. seealso::

      :ref:`psycopg2_executemany_mode` 


  :ticket:`5401`  


.. _change_4895:

删除SQLite方言中的“join rewriting”逻辑；更新导入
-------------------------------------------------

删除了支持2013年之前的旧版SQLite（即小于或等于3.7.16版本）的右嵌套关
联重写。不期望任何现代 Python 版本依赖于该限制。

该行为是在0.9中首次引入的，并且是为了允许像在   :ref:`feature_joins_09` 
所描述的那样支持右嵌套连接。但是，SQLite的变通方法在2013-2014期间产生了
许多退步，因为它的复杂性。2016年，下面被用于对SQLite版本3.7.16（SQLite对
此构造支持终止的版本）之前的版本进行重写。之后，未对该行为报告任何问题
（即使在内部发现了一些错误），因此预计完成构建Python2.7或3.5及以上支持的
SQLite版本（支持的Python版本）中不会包括一个SQLite版本之前的版本，并且在
更复杂的ORM连接场景中才使用该行为。现在，如果安装的SQLite版本旧于3.7.16，则会发出警告。

在相关更改中，SQLite的模块导入不再在Python 3上尝试导入"pysqlite2"驱动
程序，因为该驱动程序在Python 3上不存在。也删除了很久以前对旧pysqlite2版本的警告。

  :ticket:`4895`  


.. _change_4976:

为MariaDB 10.3添加“Sequence”支持
------------------------------------------

自MariaDB 10.3以来，该数据库支持“ Sequence”。 SQLAlchemy的MySQL
dialect现在在严格检查dialect的服务器版本后，处理 :class:`.Sequence` 对象，
这意味着“CREATE SEQUENCE" DDL将会针对在  :class:`_schema.Table`  or 
  :class:`_schema.MetaData` .Sequence` 发出，就像PostgreSQL、Oracle 等后端一样。另外，如果在这些情况下将 :class:`.Sequence` 用于
列默认值和主键生成对象，则该   :class:`.Sequence`  将发挥作用。

由于此更改将影响DDL以及INSERT语句的行为，因此，如果在创建表的定义中明确
使用  :class:`.Sequence` ，则此类应用程序在升级到 SQLAlchemy 1.4 时需要注意。
如果要插入数据，并且目标数据库没有其他生成整数主键值的方式，则  :class:`._Sequence` 
将用于DDL和插入语句。即使在已部署升级到SQLAlchemy 1.4的现有应用程序中，
将插入维护类的整数主键值处理为一个 :class:`._Sequence` 会更改假定的情况，
因此语法构造将失败，除非针对该构造没有DDL针对其支持的backing database。

由于此更改的影响，如果 :class:`._Sequence` 附有“optional”标记，则尤其重要，
该标记用于限制 :class:`._Sequence` 可以采取作用的情况。 当在一个表的整数主键
列上使用"optional"时：

    Table(
        "some_table",
        metadata,
        Column(
            "id", Integer, Sequence("some_seq", start=1, optional=True),
            primary_key=True
        ),
    )

以上   :class:`._Sequence`  仅在目标数据库不支持生成整数主键列的任何其他
方式时用于DDL和 INSERT语句。也就是说，在上述代码中，如果方言服务器在
DDL中指定了IDENTITY列，则本质上不会使用  :class:`.Sequence` 。如果没有这种方法，
则INSERT语句会失败。

.. seealso::

      :ref:`defaults_sequences` 

  :ticket:`4976`  


.. _change_4235:

为 SQL Server 添加不同的 Sequence 支持，不同于“IDENTITY”
-------------------------------------------------------------

  :class:`.Sequence`  结构现在在 Microsoft SQL Server 中完全可用。
应用于列时，表的DDL将不再包括IDENTITY关键字，而是依赖于“CREATE SEQUENCE”
来确保序列存在，然后用于INSERT语句。

 :class:`._Sequence` 在1.3之前用于控制SQL Server的IDENTITY列参数；
通过在1.3中引发弃用警告，这种用法即将删除。 如需控制IDENTITY列的参数，
应使用“mssql_identity_start”和“mssql_identity_increment”参数；
请参见下面链接的 MSSQL 方言文档。

.. seealso::

      :ref:`mssql_identity` 

  :ticket:`4235`  

  :ticket:`4633`  