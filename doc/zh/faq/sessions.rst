会话/查询
===========

.. contents::
    :local:
     :class: faq
    :backlinks: none

.. _faq_session_identity:

我正在使用会话重新加载数据，但是没有看到在其他地方提交的更改
------------------------------------------------------------

这种行为的主要问题是会话将事务视为*串行化*隔离状态，即使它没有
（通常并不在）。
实际上，这意味着会话不会更改其在事务范围内已读取的任何数据。

如果对“隔离级别”的术语不熟悉，则需要先阅读此链接：

`Isolation Level <https://en.wikipedia.org/wiki/Isolation_%28database_systems%29>`_

简而言之，可串行化的隔离级别通常意味着
一旦在事务中选择了一系列行，就会获得
每次重新发出该SELECT都是*相同的数据*。如果您在
下一个较低的隔离级别“可重复读取”，您将会
看到新添加的行（不再看到删除的行），但对于已加载的行
您将不会看到任何更改。只有如果在
更低的隔离级别中，例如“读取提交的”，才会变得可能
看到一行数据更改其值。

有关在使用ORM时控制隔离级别的信息，请参见  :ref:`session_transaction_isolation` 。

为了简化事情，  :class:`.Session`  本身是在
一个完全孤立的事务中运作，并且不覆盖任何映射的属性
它已经在事务的范围内读取，除非您告诉它。尝试重新读取数据的用例
您已经在正在进行的事务中加载的数据是*不常见的*用例，在许多情况下没有效果，
因此被认为是异常而不是规范；为了在此例外中工作，提供了几种方法
允许在持续事务的上下文中重新加载指定的数据。

要了解我们何谓“事务”，当我们谈论时的含义
  :class:`.Session` ，您的 :class:` .Session`旨在仅在
事务内工作。这方面的概述请参见：  :ref:`unitofwork_transaction` 。

一旦我们确定了隔离级别，我们认为
我们的隔离级别设置得低到可以重新选择一行，
我们应该在我们的 :class:`.Session` 中看到新数据，我们如何看到它？

从最常见到最不常见的三种方法：

1. 我们简单地结束当前事务，并在下一次访问时启动新事务
   使用  :class:`.Session.commit`  (请注意
   如果 :class:`.Session` 处于深度使用的“自动提交”
   模式，也将调用  :meth:`.Session.begin`  )。大多数应用程序
   和用例在不需要“查看”其他事务中的数据方面没有任何问题，因为
   他们遵循此模式，这是最佳实践的核心之一：
   **短暂的交易**。请参见 :ref:`session_faq_whentocreate` 中
   关于此的一些想法。

2. 我们告诉我们的 :class:`.Session` 重新读取它已经读取的行，
   要么在我们使用  :meth:`.Session.expire_all`  查询下一次查询它们时
   或使用  :meth:`.Session.expire`  或立即在对象上使用
     :class:`.Session.refresh` 。有关此详细信息，请参见  :ref:` session_expire` 。
   。

3. 我们可以运行整个查询，同时设置它们以确定性地覆盖
   当前正在加载的对象，同时读取行，使用“填充现有”。
   这是在 :ref:`orm_queryguide_populate_existing` 中描述的执行选项。

但记住，**如果我们的隔离设置为可重复读或更高级别，则ORM无法看到行中的更改，除非我们启动新事务**。

.. _faq_session_rollback:

"This Session's transaction has been rolled back due to a previous exception during flush." (或类似消息)
-----------------------------------------------------------------------------------------------------------

这是一个错误，在  :meth:`.Session.flush`  引发异常时发生，
回滚了事务，但在不显式调用  :meth:`.Session.rollback`  或  :meth:` .Session.close`  的情况下调用了 :class:`.Session` 上的进一步命令。

这通常对应于应用程序捕获  :ref:`.Session.flush` .Session.commit` 时
异常，而不正确地处理异常。例如::

    from sqlalchemy import create_engine, Column, Integer
    from sqlalchemy.orm import sessionmaker
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base(create_engine("sqlite://"))


    class Foo(Base):
        __tablename__ = "foo"
        id = Column(Integer, primary_key=True)


    Base.metadata.create_all()

    session = sessionmaker()()

    # 约束违规
    session.add_all([Foo(id=1), Foo(id=1)])

    try:
        session.commit()
    except:
        # 忽略错误
        pass

    # 继续使用未回滚的session
    session.commit()

使用 :class:`.Session` 应该符合与此类似的结构：

    try:
        # <使用session>
        session.commit()
    except:
        session.rollback()
        raise
    finally:
        session.close()  # 可选，取决于用例

许多事情都可能导致try/except失败flushes以外的块。
应用程序应确保对ORM定向的某种“框架”适用于处理ORM的过程，
以便连接和事务资源具有明确定义的边界，并且如果任何失败条件发生，则可以显式回滚事务。

这并不意味着整个应用程序都应该放置try/except块，这是
可伸缩的体系结构。相反，一种典型的方法是
当首次调用ORM方法和函数时，从最上层调用函数的过程
块以在一系列操作成功完成时提交事务，
如果操作由于任何原因失败，则回滚事务，
包括失败的flushes。也有使用函数装饰器的方法
或上下文管理器实现类似的结果。采取的方法
非常取决于正在编写的应用程序的类型。

有关如何组织 :class:`.Session` 使用的详细讨论，
请参见  :ref:`session_faq_whentocreate` 。

为什么flush()要发出ROLLBACK？
---------------------------------------

如果  :meth:`.Session.flush`  无法部分完成并
随后不回滚将是非常好的，但目前不支持这样做，因为它
需要修改其内部簿记，以使其可以随时停止并
与数据库的刷新保持精确一致。
尽管这在理论上是可能的，但增强的有用性
大大降低了许多数据库操作在任何情况下都需要ROLLBACK。
特别是Postgres具有操作，一旦失败，
事务就不允许继续：

.. sourcecode:: text

    test=> create table foo(id integer primary key);
    NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "foo_pkey" for table "foo"
    CREATE TABLE
    test=> begin;
    BEGIN
    test=> insert into foo values(1);
    INSERT 0 1
    test=> commit;
    COMMIT
    test=> begin;
    BEGIN
    test=> insert into foo values(1);
    ERROR:  duplicate key value violates unique constraint "foo_pkey"
    test=> insert into foo values(2);
    ERROR:  current transaction is aborted, commands ignored until end of transaction block

SQLAlchemy为解决这两个问题提供了“SAVEPOINT”支持，
通过  :meth:`.Session.begin_nested`  。使用  :meth:` .Session.begin_nested`  ，您可以构建一个可能会
在事务内部失败的操作，然后“回滚”到失败之前的点
并保持封闭事务。

为什么不滚动本来应该是一个自动的调用ROLLBACK？为什么我必须再次ROLLBACK？
--------------------------------------------------

由  :meth:`.Session.flush`  引起的回滚不是整个交易块的结束；
尽管它结束了正在进行的数据库事务，但从 :class:`.Session` 的角度来看，仍然存在处于非活动状态的事务。

有了以下块：

    sess = Session()  # 开始逻辑事务
    try:
        sess.flush()

        sess.commit()
    except:
        sess.rollback()

上述代码中，在第一次创建  :class:`.Session` .Session` 中建立逻辑事务。
这个事务是“逻辑的”，因为它实际上不使用任何数据库
资源，直到调用SQL语句，此时会启动连接级别和DBAPI级别事务。
但是，无论是否
数据库级事务是其状态的一部分，逻辑事务将保持不变，
直到使用  :meth:`.Session.commit`  ，  :meth:` .Session.rollback`  或  :meth:`.Session.close`  结束为止。

当上述“flush()”失败时，代码仍然在由try/commit/except/rollback块框定的事务内。
如果“flush()”完全回滚了逻辑事务，那么就意味着，当然到达时我们再次到达
“except：”块时， :class:`.Session` 将处于清洁状态，已准备好对所有新事物发出新SQL，
并且误传了:meth:.Session.rollback`将被错误地调用。特别地，
  :class:`.Session` .Session.rollback` 在此时将被错误地调用。
与“rollback”在即将发生回滚的地方不同，此时正常情况下应该发生回滚，  :class:`.Session` 
相反，拒绝在这个位置（即通常要发生回滚的地方）之前继续进行SQL操作，
此时，只有显式回滚之后， :class:`.Session` 才能继续。

换句话说，期望调用代码将始终调用  :meth:`.Session.commit`  ` :`或  :meth:`.Session.rollback`  或  :meth:` .Session.close`  
以对应于当前事务块。 ：meth:`.Session.flush`会将 :class:`.Session` 保留在该事务块中，
以便列表计数方法的行为是可预测且一致的。

如何创建总是为每个查询添加一定过滤器的查询？
----------------------------------------------------

参见`FilteredQuery <https://www.sqlalchemy.org/trac/wiki/UsageRecipes/FilteredQuery>`_中的配方。

.. _faq_query_deduplicating:

我的查询返回的对象数与查询计数()告诉我的对象数不相同-为什么？
---------------------------------------------------------------

  :class:`_query.Query`  对象在要求返回ORM映射对象列表时，将根据主键进行**去重**。换句话说，
如果我们例如使用所述“用户”映射：在 :ref:`tutorial_orm_table_metadata` 中，
并且我们拥有如下SQL查询：

    q = session.query(User).outerjoin(User.addresses).filter(User.name == "jack")

上述教程中使用的示例数据中，在具有名称“'jack'”，主键值为5的``users''行中，在``addresses''表中有两行。
如果我们要求上面的查询中返回：meth:`_query.Query.count`，我们将会得到答案为** 2 **：

    >>> q.count()
    2

但是，如果我们运行  :meth:`_query.Query.all`  或迭代查询，我们将返回
**一个元素**：

  >>> q.all()
  [User(id=5, name='jack', ...)]

这是因为当 :class:`_query.Query` 对象返回完整的实体时，它会**去重**。不会发生这种情况
如果我们改为请求单个返回列::

  >>> session.query(User.id, User.name).outerjoin(User.addresses).filter(
  ...     User.name == "jack"
  ... ).all()
  [(5, 'jack'), (5, 'jack')]

 :class:`_query.Query` 按照以下两个主要原因进行了去重：

* **为了使加入的急切装载正常工作**–   :ref:`joined_eager_loading`  通过使用相关表之间的连接查询出
  行，然后将这些行路由到主对象上的集合中。为了做到这一点，
  它必须获取其中主对象的主键为每个子输入重复的行。这种模式可以继续
  进入进一步的子集合，以便可以以某个单向主对象的多值的倍数进行处理，
  例如“User(id = 5)”。去重不论是否
  建立了joinedload，因为急切加载背后的关键哲学是这些选项永远不会影响结果。

* **排除有关身份图的混淆**– 这显然是次要原因。由于  :class:`.Session` 
  使用的身份图，即使我们的SQL结果集具有两个
  主键为5的行，对于每个主键/类组合，只有一个``User（id = 5）``对象
  必须根据其身份唯一地维护，即其主键/类组合。如果对于这种查询
  “用户（）”对象，不会在列表中多次获得相同的对象。可以使用有序集合代替
    :class:`_query.Query`  返回整个对象时更好地表示要返回的内容。但是
   :class:`_query.Query` 的去重问题仍然是有问题的，主要原因是
  :meth:`_query.Query.count` 方法是不一致的，而
  在最近的版本中使用联接急切的数量超过了针对该行为的使用。由于这一演变持续下去，
  SQLAlchemy可能会更改 :class:`_query.Query` 的这种行为，这也可能涉及到
  新的API，以更直接地控制此行为，并且还可以修改连接急切装载的行为以创建更一致的用法。
  

我已经创建了一个映射，使其针对Outer Join，虽然该查询返回行，但没有返回对象。为什么？
---------------------------------------------------------------

由Outer Join返回的行可能只包含主键的部分NULL，
因为主键是两个表的组合。 :class:`_query.Query` 对象忽略了
没有可接受主键的传入行-
根据 :class:`_orm.Mapper` 上“allow_partial_pks”
标志，接受主键取决于值是否具有至少一个非NULL
值或是否没有NULL值。请参见  :class:`_orm.Mapper` ` allow_partial_pks``。

当我调用"Session.delete(myobject)"时，它没有从父集合中删除！
---------------------------------------------------------------

有关此行为的说明，请参见  :ref:`session_deleting_from_collections` 。

为什么在加载对象时不调用我的“__init __()”函数？
------------------------------------------

请参见 :ref:`mapping_constructors` 以了解此行为的说明。

如何在ORM查询中使用文本SQL？
------------------------

请参见：

*   :ref:`orm_queryguide_selecting_text` -使用 :class:` _query.Query`进行自定义文本块
*   :ref:`session_sql_expressions` -直接使用文本SQL使用  :class:` .Session` 。

我调用“Session.delete(myobject)”，但它没有从父集合中删除！
---------------------------------------------------------

有关此行为的说明，请参见  :ref:`session_deleting_from_collections` 。

为什么我的“foo_id”实例属性设置为“7”，但“foo”属性仍为“None”-它没有加载ID为#7的Foo吗？
---------------------------------------------------------

ORM的构造方式不支持从外键属性更改立即填充
驱动——相反它是设计成反过来的——ORM在背后处理外键属性，
最终用户自然设置对象关系。因此，设置“o.foo”最简单的方法是这样做——设置它！::

    foo = session.get(Foo, 7)
    o.foo = foo
    Session.commit()

当然，操纵外键属性是完全合法的。但是，
将外键属性设置为新值当前不会触发
与其中涉及的：func:`_orm.relationship`有关的“过期”事件。这意味着
对于以下序列：

    o = session.scalars(select(SomeClass).limit(1)).first()

    # 假设现有的o.foo_id值为None;
    # 访问o.foo将使它与``None``合一，但实际上有效地是
    # “加载” None 的值。
    assert o.foo is None

    # 现在设置foo_id为其他值。o.foo不会立即受到影响
    o.foo_id = 7

“o.foo”在其第一次访问时以其有效的数据库值（即“None”）加载。设置
“o.foo_id = 7”将有“7”作为待处理更改的值，但没有冲洗
- 因此“o.foo”仍然是``None``::

    # 属性已经加载为“None”，尚未与o.foo_id = 7协调
    assert o.foo is None

对于“o.foo”基于外键突变的情况通常会自然地加载
通常在提交后实施，这样既可以刷新新的外键值
也重复所有状态::

    session.commit()  # 刷新所有属性

    foo_7 = session.get(Foo, 7)

    # 调用o.foo将再次进行懒加载，这次获取新对象
    assert o.foo is foo_7

一个更小的操作是单独设置属性——您可以对任何:term: persistent 的对象使用  :meth:`.Session.expire`  进行此操作：

    o = session.scalars(select(SomeClass).limit(1)).first()
    o.foo_id = 7
    Session.expire(o, ["foo"])  # 对象必须是persistent

foo_7 = session.get(Foo, 7)

    assert o.foo is foo_7  # 延迟加载：o.foo在访问时懒惰加载

注意，如果对象不是持久的但出现在 :class:`.Session` 中，则其称为:term: pending。
这意味着该对象的行尚未插入到数据库中。对于这样的对象，设置``foo_id``没有意义
直到插入该行为止；否则还没有行::

    new_obj = SomeClass()
    new_obj.foo_id = 7

    Session.add(new_obj)

    # 返回 None，但这不是“懒加载”，因为该对象不在
    # 数据库中是不稳定的，而None值不是对象的状态的一部分
    assert new_obj.foo is None

    Session.flush()  # 发出INSERT

    assert new_obj.foo is foo_7  # 现在它加载


.. topic:: 针对非持久对象的属性加载

    上述“挂起”行为的一个变异是使用标志
    ``load_on_pending``在：func:`_orm.relationship`上。当设置此标志时，
    懒惰加载器将在INSERT继续之前为“new_obj.foo”发出信号；另外
    一种变种是使用  :meth:`.Session.enable_relationship_loading`  方法，
    它可以将对象“附加”到 :class:`.Session` 中，以使多对一关系加载为按外键属性方法进行加载，而不管对象是否处于任何特定状态。
    这两种技术都**不建议通用使用**，它们是添
    加上ORM通常的对象状态的特定编程方案遇到的用户特定编程方案。

配方`ExpireRelationshipOnFKChange <https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange>`_
提供了一个示例，它使用SQLAlchemy事件来协调外键属性的设置与一对多
关系。

.. _faq_walk_objects:

如何遍历与给定对象相关的所有对象？
-------------------------------------------

具有其他对象相关联的对象将对应于
  :meth:`_orm.relationship`  在映射器之间设置。此代码片段将
遍历所有对象，并进行周期性修正：

    from sqlalchemy import inspect

    def walk(obj):
        deque = [obj]

        seen = set()

        while deque:
            obj = deque.pop(0)
            if obj in seen:
                continue
            else:
                seen.add(obj)
                yield obj
            insp = inspect(obj)
            for relationship in insp.mapper.relationships:
                related = getattr(obj, relationship.key)
                if relationship.uselist:
                    deque.extend(related)
                elif related is not None:
                    deque.append(related)

您可以使用以下方式展示此函数：

    Base = declarative_base()

    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        bs = relationship("B", backref="a")

    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)

该功能可能如下所示：

    obj = A()
    obj.bs = [B(), B()]

    for row in walk(obj):
        print(row)        a_id = Column(ForeignKey("a.id"))
        c_id = Column(ForeignKey("c.id"))
        c = relationship("C", backref="bs")


    class C(Base):
        __tablename__ = "c"
        id = Column(Integer, primary_key=True)


    a1 = A(bs=[B(), B(c=C())])


    for obj in walk(a1):
        print(obj)

输出:

.. sourcecode:: text

    <__main__.A object at 0x10303b190>
    <__main__.B object at 0x103025210>
    <__main__.B object at 0x10303b0d0>
    <__main__.C object at 0x103025490>

有没有一种自动为关键词提取唯一性的方式（或者其他类型的对象），而不需要查询关键词并获得包含该关键词的行的引用？

人们读完docs中的many-to-many示例后，会发现如果您创建了两次相同的“关键词(Keyword)”，它将在DB中插入两次，这相当麻烦。为解决此问题，我们创建了这个`UniqueObject <https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject>`_ 。

.. _faq_post_update_update:

为什么“post_update”除了第一个“UPDATE”操作之外，还会发出“UPDATE”操作？

“Post_update”功能在   :ref:`post_update`  中记录，它涉及在对特定的关系绑定外键(relationship-bound foreign key)进行更改时，除了通常会对目标行发出的INSERT / UPDATE / DELETE操作之外，还会发出一个UPDATE语句。虽然此UPDATE语句的主要目的是与该行的INSERT或DELETE配对，以便在断开彼此依赖的外键之间自动地进行后置设定或前取消设置，但目前它也作为第二次发出UPDATE语句捆绑在一起，并且在目标行本身受到UPDATE操作的情况下常常是不必要的，并且通常看起来是浪费的。

但是，尝试删除此“ UPDATE / UPDATE”行为的某些研究表明，需要对工作单元的过程进行重大更改，这不仅涉及到post_update的实现，还涉及到与post_update无关的其他领域，因为在某些情况下，非post_update侧的操作顺序需要反转，这反过来又会影响其他案例，例如正确处理引用主键值的UPDATE（请参阅  :ticket:`1063`  中的概念证明）。

答案是，“post_update”用于断开两个彼此依赖的外键之间的循环，并且仅在目标表的INSERT / DELETE操作中进行此循环破坏意味着Update语句的顺序需要在其他地方得到宽松，从而导致其他边缘情况的破坏。