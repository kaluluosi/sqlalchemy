Sessions / Queries
==================

.. contents::
    :local:
    :class: faq
    :backlinks: none

.. _faq_session_identity:

重载数据到我的Session中，但是它没有看到我在其他地方提交的更改
------------------------------------------------------------------------------------------

关于这种行为的主要问题是，即使不是在某些情况下（通常不是如此），Session也会像事务处于*serializable*隔离状态。从实际角度来说，这意味着Session不会改变任何已经在事务范围内读取的数据。

如果“隔离级别”这个术语不熟悉，那么您需要先阅读此链接：

隔离级别

简而言之，“serializable”隔离级别通常意味着一旦在事务中选择了一系列行，每次重新发出该SELECT时，都将返回“相同的数据”。 如果您处于较低的隔离级别“可重复读取”，则会看到新添加的行（不再看到已删除的行），但对于您已经加载的行，您将不会看到任何更改。 仅当您处于较低的隔离级别（例如“可读取的”)时，才有可能看到数据行更改其值。

有关在使用SQLAlchemy ORM时控制隔离级别的信息，请参见:ref: 'session_transaction_isolation'。

要极大地简化事情，Session本身是根据完全隔离的事务来工作的，并且除非您告诉它，否则它不会覆盖它已经读取的任何映射属性。 在正在进行的事务内尝试重新读取您已经加载的数据的用例是一种**不常见的**用例，在许多情况下没有影响，因此这被认为是例外而不是规范; 为了在此例外中工作，已提供了几种方法，以允许在正在进行的事务上下文中重新加载特定数据。

要理解我们何时讨论“事务”时我们是什么意思，当我们谈论:class:`.Session`时，您的:class:`.Session`旨在仅在事务内工作。有关此的概述，请参见:ref:`unitofwork_transaction`。

一旦我们确定了我们的隔离级别是什么，并且我们认为我们的隔离级别设置的足够低，以便我们重新SELECT行时，我们应该在:class:`.Session`中看到新数据，如何做到呢？

从大多数到最少，有三种方法：

1. 我们只需在下一个访问时在我们的:class:`.Session`上结束事务并开始新的事务，
   调用 :meth:`.Session.commit`(请注意，如果:class:`.Session`处于不常用的“自动提交”模式，则还需要调用 :meth:`.Session.begin`)。 
   绝大多数应用程序和用例不会遇到无法在其他事务中“看到”数据的任何问题，因为它们遵循最佳实践的核心--**短生命周期事务**。有关此内容的一些思考，请参见:ref:`session_faq_whentocreate`。

2. 我们告诉我们的:class:`.Session`要重新读取它已经读取的行，无论是在下一次使用 :meth:`.Session.expire_all` 、 :meth:`.Session.expire`查询它们时，还是立即使用 :class:`.Session.refresh` 在对象上。有关详细信息，请参见:ref:`session_expire`。

3. 我们可以在执行期间运行整个查询，同时将它们设置为在读取行时肯定覆盖已经加载的对象，使用“populate existing”。这是在 :ref:`orm_queryguide_populate_existing` 中描述的执行选项。

但请记住，**ORM无法在隔离级别为可重复读或更高级别时查看行中的更改，除非我们开始一个新的事务**。

.. _faq_session_rollback:

此Session的事务由于刷新期间之前的异常而被回滚了。 。 （或类似的）
---------------------------------------------------------------------------------------------------------

当:meth:`.Session.flush`引发异常，回滚事务，但在没有显式调用:meth:`.Session.rollback`或:meth:`.Session.close`的情况下调用了:class:`.Session`上的进一步命令时，就会发生此错误。

通常对应于捕获:fant异的应用程序： Session.flush或Session.commit并且没有正确处理异常。 例如：

    from sqlalchemy import create_engine, Column, Integer
    from sqlalchemy.orm import sessionmaker
    from sqlalchemy.ext.declarative import declarative_base

    Base = declarative_base(create_engine("sqlite://"))


    class Foo(Base):
        __tablename__ = "foo"
        id = Column(Integer, primary_key=True)


    Base.metadata.create_all()

    session = sessionmaker()()

    #违反约束
    session.add_all([Foo(id=1), Foo(id=1)])

    try:
        session.commit()
    除外:
        #忽略错误
        pass

    #继续在不回滚的情况下使用Session
    session.commit()

善用:class:`.Session`应当遵循类似以下结构：

    尝试：
        #<使用会话>
        session.commit()
    除此之外
        session.rollback()
        抛出
    最后：
        session.close()＃可选，取决于用例

许多事情可以导致尝试/ except块内的失败除刷新外。应用程序应确保对ORM导向的进程应用某种“框架”，以便连接和事务资源具有明确定义的边界，使得在任何失败条件下都可以显式地回滚事务，包括失败的刷新。 这并不意味着整个应用程序都应该有try / except块，这不是可扩展的体系结构。相反，典型的方法是，当首次调用ORM导向的方法和函数时，调用函数的过程从顶部开始将位于块内的事务提交到一系列操作的成功完成，并且如果任何故障条件发生，则回滚事务，包括失败的刷新。也有使用函数装饰器或上下文管理器实现类似结果的方法。采用的方法非常取决于正在编写的应用程序的类型。

有关如何组织使用：class：`的详细讨论`.Session`，请参见:ref:`session_faq_whentocreate`。

但为什么刷新会强制执行回滚？
 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:meth:`.Session.flush`可能部分完成，然后不回滚我将超出其当前功能的原因是，其内部帐务记录必须进行修改，以便可以在任何时候停止，并且与已刷新到数据库的内容一致。虽然理论上可以做到这一点，但因为许多数据库操作在任何情况下都需要ROLLBACK，所以增强的有用性大大降低了。特别是Postgres具有操作，一旦失败，事务就不允许继续：

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

SQLAlchemy提供的解决这两个问题的方法是支持SAVEPOINT，通过:class:`.Session.begin_nested`。使用:class:`.Session.begin_nested`，您可以为可能失败的操作创建框架，并在维护封闭事务的同时“回滚”到之前它的故障点。

为什么只有一个ROLLBACK的自动调用不够？ 我为什么还必须ROLLBACK？
 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

从.:meth：`中导致的ROLLBACK`Session.flush``是完整事务块的结局，尽管它结束了游戏数据库的事务，但是从Session的角度来看，仍然有一个当前处于非活动状态的事务。

例如：

    sess = Session()＃开始逻辑事务
    尝试：
        sess.flush()
        sess.commit()
    except:
        sess.rollback()

上述代码在:class:`.Session`第一次创建时，假设没有使用“auto-commit mode”，则在:class:`.Session`中建立了逻辑事务。在逻辑事务中“逻辑”是由于它实际上没有使用除SQL语句外的任何数据库资源启动，此时将启动连接级别和DBAPI级别的事务。但是，无论它的状态是否存在数据库级别事务，逻辑事务将保持在那里，直到它使用:meth:`.Session.commit`,`:meth:`.Session.rollback`或:meth:`.Session.close``结束。

当上下文处于上述尝试/ except框架中的块失败时，代码仍在由try / commit / except / rollback框架构建的事务中。如果“flush()”完全回滚了逻辑事务，那么当我们到达“except：”块时，:class:`.Session`将处于一个干净的状态，准备对所有新事务发出新的SQL，而:meth:`.Session.rollback`则处于错误序列中。特别是，在这一点上，:class:`.Session``已经开始了新事务，因此:meth:`.Session.rollback`将错误地起作用。特别是，此时:class:`.Session`始终必须进行新事务，:meth:`.Session.rollback`错误地起作用。相反，:class:`.Session`应该始终:meth:`.Session.commit`，:meth:`.Session.rollback`或:meth:`.Session.close`要对应当前事务块。"flush()"在正确处理回滚的位置上很明显，类似于在试图回滚处于正常用法中的此处时，SQL操作继续在新事务上进行，而:class:`.Session`拒绝进行，直到实际回滚发生为止。

换句话说，这意味着调用代码的期望一直是:class:`.Session.commit`,`:meth:`.Session.rollback`或:meth:`.Session.close`朝向当前事务块。 "flush()"将:class:`.Session`保持在该事务块中，以便:class:`_query.Query`希望返回:class:`_query.Query`返回完整的结果集时为可预测和一致的行为。

如何使每个查询始终添加某个过滤器以过滤每个查询？
------------------------------------------------------------------------------------------------

见FilteredQuery。

.. _faq_query_deduplicating:

我的查询返回的对象数量与查询.count()测试我告诉我的不同 - 为什么？
-------------------------------------------------------------------------------------

:class:`_query.Query`对象在要求返回ORM映射的对象列表时，将根据主键进行**主键去重**。也就是说，例如使用在`ORM元数据`命名的“ User”映射，如果我们有如下SQL查询：

    q = session.query(User).outerjoin(User.addresses).filter(User.name == "jack")

上述教程中使用的示例数据的“地址”表中有两行用于名称为“'jack'”的用户行，主键值为5。如果我们要求上面的查询进行:meth:`_query.Query.count`，我们将得到答案**2**：

    >>> q.count()
    2

但是，如果我们运行:meth:`_query.Query.all`或迭代查询，我们会得到一个**元素**：

  >>> q.all()
  [User(id=5, name='jack', ...)]

这是因为当:class:`_query.Query`对象返回全部实体时，它会**去重**。如果我们对单个列请求返回，则不会发生这种情况::

  >>> session.query(User.id, User.name).outerjoin(User.addresses).filter(
  ...     User.name == "jack"
  ... ).all()
  [(5, 'jack'), (5, 'jack')]

类似地，:class:`_query.Query`将进行去重有两个主要原因：

* **为使连接的贪婪加载正常工作** - :ref:`joined_eager_loading`
  通过使用对相关表进行连接的联接来查询行，在将这些行路由到引导对象的集合时，其中的主对象的主键将重复为每个子条目。 这种模式可以继续应用于进一步的子集合中，例如为``User（id = 5）``这样的“所有” '。 对于我们来说，它在地址集合已通过``lazy = 'joined'``或通过：func:`_orm.joinedload`选项加载，以及不管是否已经建立joinedload，依然适用，因为贪婪加载背后的关键理念是这些选项从不会影响结果。

* **为消除对身份图的困惑** - 这可能是不太关键的原因。由于:class:`.Session`
  使用:term:`identity map`，因此即使我们的SQL结果集具有两个主键为5的行，所有DB中只有一个``User（id = 5）``对象，这必须根据其 主键/类组合唯一维护，也就是说，主键/
  类组合。实际上，如果要返回用户（）对象，多次在列表中获取同一个对象没有多少意义。 有序集合可能是:class:`_query.Query`希望返回的更好表示方式。

:meth:`_query.Query`的去重问题仍然很棘手，主要是因为:meth:`_query.Query.count`方法不一致，当前状态是贪婪加载最近的几个版本已被后代向子查询贪婪加载策略和更近的“选择IN 贪婪加载“策略，它们都通常更适用于集合贪婪加载。随着这种演变的持续进行，SQLAlchemy可能会更改:class:`_query.Query`上的这种行为，这可能还涉及新的API，以更直接地控制此行为，并且也可能更改联接线路产生后果的方式 查询的贪婪加载。

我已经创建了一个映射以与Outer Join相关联，而虽然查询返回了行，但没有返回任何对象。为什么不是这样呢？
------------------------------------------------------------------------------------------------------------------

由外部联接返回的行可能会包含主键的一部分为NULL的值，因为主键是两个表的合成。:class:`_query.Query`对象将忽略不具有可接受主键的传入行。根据:class:`_orm.Mapper`上的``allow_partial_pks``标志，如果该值有至少一个非NULL值，或者该值没有NULL值，则接受该主键。请参见``allow_partial_pks``在：class:`_orm.Mapper`。

我正在使用``joinedload（）``或``lazy = False``创建JOIN / OUTER JOIN，并且在尝试添加WHERE，ORDER BY，LIMIT等（依赖于（OUTER）JOIN）时，SQLAlchemy无法构建正确的查询 - 为什么？                                                       
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

joined贪婪加载生成的连接仅用于完全加载相关集合，并且旨在不会影响查询的主结果。由于它们是匿名别名，因此无法直接引用它们。

关于此行为的详细信息，请参见:ref:`zen_of_eager_loading`。

查询没有“__len __（）”方法，为什么没有？
------------------------------------

应用于对象的Python ``__len __（）``幻术方法允许使用“len（）”
builtin用于确定集合的长度。可以直觉地将SQL查询对象与``__len __（）``关联到:meth:`_query.Query.count`方法，该方法发出`SELECT COUNT`。之所以不可能，是因为将查询作为列表进行评估会造成两个SQL调用而不是一个：

    class Iterates:
        def __len__(self):
            print("LEN!")
            return 5

        def __iter__(self):
            print("ITER!")
            return iter([1, 2, 3, 4, 5])


    list(Iterates())

产生的输出：

.. sourcecode:: text

    ITER!
    LEN!

如何用ORM查询使用Textual SQL？
------------------------------------------

见:

* :ref:`orm_queryguide_selecting_text`-用:class:`_query.Query`进行即兴的文本块

* :ref:`session_sql_expressions`-直接使用文本SQL使用:class:`~sqlalchemy.orm.session.Session`。

我调用了``Session.delete（myobject）''，但它没有从父集合中删除！
------------------------------------------------------------------------------------------

请参见:ref:`session_deleting_from_collections`

为什么不在加载对象时调用我的``__init __（）''？

请参阅:ref:`mapping_constructors`，了解此行为的说明。

如何使用SA ORM使用ON DELETE CASCADE？
---------------------------------------------

SQLAlchemy始终会为当前在Session中加载的依赖行发出UPDATE或DELETE语句。对于尚未加载的依赖项行，默认情况下会发出SELECT语句，以便也可以更新/删除它们。 换句话说，它假设没有配置ON DELETE CASCADE。

要配置SQLAlchemy以与ON DELETE CASCADE进行协作，请参见:ref:`passive_deletes`。

我将"is"属性设置为"7"，但"foo"属性仍然为``None`` - 它不应该加载id＃7的Foo吗？
-------------------------------------------------------------------------------------------------------------------

ORM没有以支持从外部键属性更改驱动立即填充关系的方式构建-相反，它是设计为采用：class:`~sqlalchemy.orm.relationship`构造器设置的对象关系。因此，将“o.foo”设置为新值通常是做到这一点的方法：

    foo = session.get(Foo, 7)
    o.foo = foo
    Session.commit()

操作外键属性是完全合法的。但是，将外键属性设置为新值当前不会触发包含它的:func:`_orm.relationship`的“expire”事件。这意味着，对于以下序列：

    o = session.scalars(select(SomeClass).limit(1)).first()

    #假设现有的o.foo_id值为None;
    #访问o.foo将通过将其视为"None"对其进行协调，但是有效地
    #“加载”价值观为“None”
    assert o.foo is None

    #现在将foo_id设置为其他事情。 o.foo不会立即受到影响
    o.foo_id = 7

``o.foo``在第一次访问时加载其有效的数据库值为“None”。设置
``o.foo_id = 7``将更改暂挂的更改值，但尚未提交更改。因此，
``o.foo``仍为``None``：


    #设置已“加载”为None，但尚未协调为o.foo_id = 7
    assert o.foo is None

要根据外键突变加载“o.foo”通常在提交后自然发生，这将刷新新的外键值并使其逾期所有状态：:

    session.commit()＃过期所有属性

    foo_7 = session.get(Foo, 7)

    # o.foo将再次进行懒加载，这次获取新对象
    assert o.foo is foo_7

稍微简单一些的操作是单独过期该属性 - 对于任何:term:`persistent`对象，您都可以这样做:meth:`~sqlalchemy.orm.session.Session.expire`::

    o = session.scalars(select(SomeClass).limit(1)).first()
    o.foo_id = 7
    Session.expire(o, ["foo"])  ＃对象必须是persistent的

    foo_7 = session.get(Foo, 7)

    assert o.foo is foo_7  ＃ o.foo lazyloads on access

注意，如果对象尚未持久，但存在于:class:`.Session`中，则它被称为:term:`pending`。这意味着尚未将对象的行INSERT到数据库中。对于这样的对象，将``foo_id``设置没有意义，直到插入行; 否则还没有行：

    new_obj = SomeClass()
    new_obj.foo_id = 7

    Session.add(new_obj)

    #返回None，但这不是“lazyload”，因为对象在DB中不是persistent状态，
    #None值不是对象的状态的一部分
    assert new_obj.foo is None

    Session.flush()  ＃发出INSERT

    assert new_obj.foo is foo_7  ＃现在它加载

.. topic:: 用于非持久对象的属性加载

    上述pending行为的一个变体是在:func:`_orm.relationship`上使用标志
    ``load_on_pending``。当设置此标志时，懒汉将在INSERT
    继续之前发出：class:`.Session.begin_nested` “new_obj.foo” ；另外
    这种方法是使用:meth:`.Session.enable_relationship_loading`方法，该方法可以“连接”对象以便许多对一关系根据外键属性加载，并且与对象处于特定状态无关。这两种技术都**不推荐一般使用**；它们添加了来自ORM-usual object
    状态。

配方特色`ExpireRelationshipOnFKChange <https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange>`_演示了使用SQLAlchemy事件的示例，以便将外键属性设置与多对一关系协调起来。

.. _faq_walk_objects:

如何遍历与给定对象相关的所有对象？
-------------------------------------------------- ---------------------------

具有其他对象相关的对象将对应于标识映射器之间的:func:`_orm.relationship`构造。该代码片段将在纠正周期的情况下迭代所有对象：

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

可以如下所示演示该函数：

    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"
        id = Column(Integer, primary_key=True)
        bs = relationship("B", backref="a")


    class B(Base):
        __tablename__ = "b"
        id = Column(Integer, primary_key=True)

有没有一种自动将关键字（或其他类型的对象）设置为唯一的方法，而不需要查询关键字并获取包含该关键字的行的引用？
---------------------------------------------------------------------------------------------------------------------------

当人们阅读文档中的多对多示例时，它们会发现如果您创建了相同的“ Keyword”两次，则会将其放入数据库中两次。这有点不方便。

创建这个 UniqueObject 来解决这个问题。可以在 https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject 进行查看。

.. _faq_post_update_update:

为什么 post_update 会发出 UPDATE 以及第一次 UPDATE？
-----------------------------------------------------

post_update 特性, 文档 在 :ref:`post_update` , 在外键关联的更改时涉及发出 UPDATE 语句，除了通常会发出的目标行的 INSERT/UPDATE/DELETE 之外，还会向中断与互相依赖的外键的循环的前／后置的方式 发出 UPDATE 语句。虽然此 UPDATE 语句的主要目的是与该行的插入或删除配对，以便可以后设置或预取消设置外键引用以打破互相依赖的外键，但是当前它也被捆绑为生成目标行发生UPDATE 时发出的第二个 UPDATE。在这种情况下，post_update 发出的 UPDATE 通常是不必要的，而且经常会显示浪费的情况。

然而，尝试去除此“ UPDATE / UPDATE”行为的一些研究表明，不仅在 post_update 实现中需要进行工作单元进程的主要更改，还需要在与 post_update 无关的区域进行更改，因为在某些情况下非 post_update 侧的操作顺序需要相反，这反过来又会影响其他情况，例如正确处理引用的主键值的 UPDATE（有关“ 1063”中的概念证明），会导致其他情况的故障。

答案是“post_udpate”用于打破两个互相依赖的外键之间的循环，并且为了使这种循环破坏仅限于目标表的 INSERT/DELETE，需要放宽 UPDATE 语句的顺序，这会导致其他边缘情况的故障。