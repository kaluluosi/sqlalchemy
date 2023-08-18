.. _baked_toplevel:

Baked Queries
=============

.. module:: sqlalchemy.ext.baked

``baked`` 提供了一种替代的创建   :class:`~.query.Query`  对象的模式，它允许缓存对象的构建和字符串编译步骤。这意味着对于一个常用的   :class:` ~.query.Query`  构建场景，从它的初始构造到生成 SQL 字符串的每个 Python 函数调用将仅发生 **一次**，而不是每次构建和执行查询时都要发生。

建立这个系统的原因是为了极大地减少 Python 解释器**在 SQL 发出之前** 的开销。 "baked" 系统的缓存**不会**以任何方式减少 SQL 调用或缓存从数据库的返回结果。一种演示缓存 SQL 调用和结果集本身的技术在   :ref:`examples_caching`  中。

.. deprecated:: 1.4 SQLAlchemy 1.4 和 2.0 具有全新的直接查询缓存系统，完全不需要   :class:`.BakedQuery`  系统。对于所有 Core 和 ORM 查询，缓存现在都是透明激活的，无需用户进行任何操作，使用在   :ref:` sql_caching`  中描述的系统。

.. deepalchemy::

     :mod:`sqlalchemy.ext.baked`  扩展不适合初学者。正确使用它需要对 SQLAlchemy，数据库驱动程序和后端数据库如何相互交互拥有良好的高级别理解。该扩展程序提供一种非常特定的优化类型，通常不需要。如上所述，它**不会缓存查询**，只会缓存 SQL 本身的字符串表述。

概述
--------

使用 baked 系统的方法通过生成所谓的“面包屑”，该面包屑代表了存储特定查询对象系列的存储方法开始：

    from sqlalchemy.ext import baked

    bakery = baked.bakery()

上面的“bakery”将默认在 LRU 缓存中存储缓存数据，缓存值为 200 个元素，注意 ORM 查询通常会包含一个针对调用的 ORM 查询条目，以及每个 SQL 字符串的数据库语言。

这个 bakery 允许我们通过将其构造方式指定为一系列 Python 可调用来构建一个   :class:`~.query.Query`  对象，这些可调用通常是 lambda，对于简洁的用法，它重载了 "+=" 运算符，以便典型的查询构建看起来像：

    from sqlalchemy import bindparam


    def search_for_user(session, username, email=None):
        baked_query = bakery(lambda session: session.query(User))
        baked_query += lambda q: q.filter(User.name == bindparam("username"))

        baked_query += lambda q: q.order_by(User.id)

        if email:
            baked_query += lambda q: q.filter(User.email == bindparam("email"))

        result = baked_query(session).params(username=username, email=email).all()

        return result

下面是对上面代码的一些观察：

1. ``baked_query`` 对象是   :class:`.BakedQuery`  的一个实例。这个对象实际上是一个真正的 orm   :class:` ~.query.Query`  对象的“构建器”，但它本身不是*实际*   :class:`~.query.Query`  对象。

2. 实际的   :class:`~.query.Query`  对象不会在所有情况下构建，直到最后调用  :meth:` _baked.Result.all`  时才真正构建。

3. 添加到 ``baked_query`` 对象的步骤都表现为 Python 函数，通常是 lambda，传递给   :func:`.bakery`  函数的第一个 lambda 接收一个   :class:` .Session`  作为其参数。其余 lambdas 每个接收一个 ：class:`~ .query.Query` 作为其参数。

4. 在上面的代码中，即使我们的应用可能在很多情况下调用 ``search_for_user()``，即使在每次调用中我们构建一个全新的   :class:`.BakedQuery`  对象，*所有 lambda 都只被调用一次*。每个 lambda 只是在该查询在面包屑缓存中缓存期间**不会**再次被调用。

5. 通过储存引用的**lambda 对象本身**来实现缓存。即 Python 解释器在这些函数中为函数分配了一个 Python 标识（identity），这就决定了如何在后续运行中标识查询。对于那些指定了“email”参数的 ``search_for_user()`` 调用，可调用 ``lambda q: q.filter(User.email == bindparam('email'))`` 将成为检索的缓存键的一部分; 当“email”是“None”时，此可调用不是缓存键的一部分。

6. 因为这些 lambda 只被调用一次，所以非常重要的是在 lambda 中**不引用可以在调用间更改的变量**； 相反，假设这些是要绑定到 SQL 字符串中的值，我们使用   :func:`.bindparam`  来构造命名参数，在  :meth:` _baked.Result.params`  中稍后应用实际值。

性能烘培查询可能看上去有点奇怪、有点笨重、略显繁琐。然而，当一个查询在应用程序中被频繁调用时，通过使用烘培查询可以大幅提高 Python 的性能。
示例套件 `short_selects` 在 :re和 `examples_performance` 说明中演示了每次仅返回一行记录的查询语句的比较，例如下面的普通查询语句：

.. sourcecode:: python

    session = Session(bind=engine)
    for id_ in random.sample(ids, n):
        session.query(Customer).filter(Customer.id == id_).one()

与相应的“烘培”查询语句进行比较：

.. sourcecode:: python

    bakery = baked.bakery()
    s = Session(bind=engine)
    for id_ in random.sample(ids, n):
        q = bakery(lambda s: s.query(Customer))
        q += lambda q: q.filter(Customer.id == bindparam("id"))
        q(s).params(id=id_).one()

对于每个块进行 10000 次迭代的 Python 函数调用数量之间的差异为：

.. sourcecode:: text

    test_baked_query : test a baked query of the full entity.
                       (10000 iterations); total fn calls 1951294

    test_orm_query :   test a straight ORM query of the full entity.
                       (10000 iterations); total fn calls 7900535

以强大的笔记本电脑的秒数为单位，结论如下：

.. sourcecode:: text

    test_baked_query : test a baked query of the full entity.
                       (10000 iterations); total time 2.174126 sec

    test_orm_query :   test a straight ORM query of the full entity.
                       (10000 iterations); total time 7.958516 sec

需要注意的是，这个测试非常重视只返回一行记录的查询。对于返回多行记录的查询，使用烘培查询的性能优势将会越来越小，比例与获取记录所花费的时间成正比。需要特别注意的是，**烘培查询功能仅适用于构建查询本身，而不是提取结果**。使用烘培功能并非一定能够提高应用程序的性能；它仅仅是对于那些由此类开销影响的应用程序而言的一个潜在的有用功能。

.. topic:: 量力而行

    关于如何对 SQLAlchemy 应用程序进行性能分析的背景，请参见 :re这一章节。对于尝试改善应用程序性能，必须使用性能度量技术。

原理
------

以上“lambda”方法是更传统的“参数”方法的超集。假设我们希望构建一个简单的系统，其中我们只构建一个   :class:`~.query.Query` ，然后将其存储在字典中以供重复使用。现在可以直接通过构建查询并通过调用 ` `my_cached_query = query.with_session(None)`` 删除其   :class:`.Session`  来实现这一点：

.. sourcecode:: python

    my_simple_cache = {}


    def lookup(session, id_argument):
        if "my_key" not in my_simple_cache:
            query = session.query(Model).filter(Model.id == bindparam("id"))
            my_simple_cache["my_key"] = query.with_session(None)
        else:
            query = my_simple_cache["my_key"].with_session(session)

        return query.params(id=id_argument).all()

以上方法为我们带来了非常微小的性能收益。通过重复使用   :class:`~.query.Query` ，我们可以节省在 ` `session.query(Model)`` 构造函数中所执行的 Python 工作，以及调用 ``filter(Model.id == bindparam('id'))`` 所执行的工作，这将为我们跳过构建 Core 表达式以及将其发送到  :meth:`_query.Query.filter` 。但是，通过在调用  :meth:` _query.Query.all`  时每次重新生成全新的   :class:`_expression.Select`  对象，这种方法仍然会产生所有额外的开销，并且会额外地把这个全新的   :class:` _expression.Select`  对象传递给字符串编译步骤，这对于上面这个简单情况来说可能是开销的 70%。

为了减少额外的开销，我们需要一些更专业的逻辑，一些可以记忆化选择对象和 SQL 构造的方式。在 wiki 中的 `BakedQuery <https://bitbucket.org/zzzeek/sqlalchemy/wiki/UsageRecipes/BakedQuery>`_ 部分有一个示例，它是此功能的前身，然而在那个系统中，我们没有缓存查询的 *构造*。为了消除所有开销，我们需要缓存查询的构造以及 SQL 编译。假设我们按照这种方式改进了该方法，并制作了一个 ``.bake()`` 方法，该方法可预编译用于查询的 SQL，生成一个可用于快速调用的新对象。我们的示例变为：

.. sourcecode:: python

    my_simple_cache = {}


    def lookup(session, id_argument):
        if "my_key" not in my_simple_cache:
            query = session.query(Model).filter(Model.id == bindparam("id"))
            my_simple_cache["my_key"] = query.with_session(None).bake()
        else:
            query = my_simple_cache["my_key"].with_session(session)

        return query.params(id=id_argument).all()以上，我们已经解决了性能问题，但仍需要处理字符串缓存密钥。

我们可以使用“面包房”方法重构以上内容，以一种看起来不那么不寻常的方式，而更像一个简单的改进，可以比简单的“重用查询”的方法更好的实现：

::

    bakery = baked.bakery()


    def lookup(session, id_argument):
        def create_model_query(session):
            return session.query(Model).filter(Model.id == bindparam("id"))

        parameterized_query = bakery.bake(create_model_query)
        return parameterized_query(session).params(id=id_argument).all()

以上，我们使用“baked”系统的方式非常类似于简单的“缓存查询”系统。然而，它使用了少了两行代码，不需要制造一个“my_key”的缓存密钥，并且还包括与我们自定义的“bake”函数相同的特性，它将查询构造函数、过滤器调用、生成   :class:`_expression.Select`  对象和字符串编译步骤的 100％ Python 调用工作缓存起来。

从以上内容出发，如果我们问自己，“如果lookup需要根据查询结构做出条件决策怎么办？”，这就是希望“baked”是这样的原因。与其从一个函数（这就是我们最初认为的baked可能起作用的方式）开始构建参数化查询，我们可以从*任意数量*的函数构建它。考虑我们的Naïve示例，如果我们需要在条件基础上在查询中添加附加子句：

::

    my_simple_cache = {}


    def lookup(session, id_argument, include_frobnizzle=False):
        if include_frobnizzle:
            cache_key = "my_key_with_frobnizzle"
        else:
            cache_key = "my_key_without_frobnizzle"

        if cache_key not in my_simple_cache:
            query = session.query(Model).filter(Model.id == bindparam("id"))
            if include_frobnizzle:
                query = query.filter(Model.frobnizzle == True)

            my_simple_cache[cache_key] = query.with_session(None).bake()
        else:
            query = my_simple_cache[cache_key].with_session(session)

        return query.params(id=id_argument).all()

我们的“简单”参数化系统现在必须负责生成缓存密钥，考虑到是否传递了“include_frobnizzle”标志，因为该标志的存在表示生成的SQL将完全不同。显然，随着查询构建的复杂性增加，缓存这些查询的任务会非常快地变得繁琐。我们可以将上面的示例转换为以下“bakery”直接使用：

::

    bakery = baked.bakery()


    def lookup(session, id_argument, include_frobnizzle=False):
        def create_model_query(session):
            return session.query(Model).filter(Model.id == bindparam("id"))

        parameterized_query = bakery.bake(create_model_query)

        if include_frobnizzle:

            def include_frobnizzle_in_query(query):
                return query.filter(Model.frobnizzle == True)

            parameterized_query = parameterized_query.with_criteria(
                include_frobnizzle_in_query
            )

        return parameterized_query(session).params(id=id_argument).all()

以上，我们不仅缓存查询对象，而且还缓存生成SQL所需的所有工作。我们也不再需要处理确保我们生成的缓存密钥准确考虑到我们所做的所有结构性修改的任务；这现在是自动处理的，没有出错的机会。

该代码示例比Naive示例要短几行，消除了处理缓存密钥的需要，并具有完整的所谓“烘烤”功能的巨大性能优势。但仍然有点冗长！因此，我们将类似于  :meth:`.BakedQuery.add_criteria`  和  :meth:` .BakedQuery.with_criteria`  这样的方法缩短为运算符，并鼓励（当然不要求！）使用简单的lambda，只作为减少冗长的手段：

::

    bakery = baked.bakery()


    def lookup(session, id_argument, include_frobnizzle=False):
        parameterized_query = bakery.bake(
            lambda s: s.query(Model).filter(Model.id == bindparam("id"))
        )

        if include_frobnizzle:
            parameterized_query += lambda q: q.filter(Model.frobnizzle == True)

        return parameterized_query(session).params(id=id_argument).all()

在上面的方法中，该方法更容易实现，并且在代码流中更类似于非缓存查询函数的样子，从而使代码更易于移植。

以上描述实际上是用于到达当前“烘焙”方法的设计过程的摘要。从“正常”方法开始，还需解决缓存密钥的构建和管理，去除所有冗余的Python执行以及需要根据条件语句构建的查询等附加问题，从而形成最终方法。

特殊查询技巧
------------------------

本节将描述特定查询情况的一些技术。

.. _baked_in:
使用IN语句
^^^^^^^^^^^^^^^^^^^^

在SQLAlchemy中，  :meth:`.ColumnOperators.in_`  方法历史上基于传递给该方法的项目列表生成可变的绑定参数集，但这不适用于“烘焙的”查询，因为该列表的长度可能会在不同的调用中更改。为了解决这个问题，  :paramref:` .bindparam.expanding`  参数支持后期渲染IN安全的表达式，可以在烘焙查询内缓存。实际元素列表在语句执行时呈现，而不是在语句编译时呈现：

::

    bakery = baked.bakery()

    baked_query = bakery(lambda session: session.query(User))
    baked_query += lambda q: q.filter(User.name.in_(bindparam("username", expanding=True)))

    result = baked_query.with_session(session).params(username=["ed", "fred"]).all()

.. seealso::

   :paramref:`.bindparam.expanding` 

   :meth:`.ColumnOperators.in_` 

使用子查询
^^^^^^^^^^^^^^^^

在使用  :class:`_query.Query` 
对象用于在另一个对象中生成子查询。在
 :class:`_query.Query` 当前处于烘焙形式的情况下，可以使用中间方法来
检索  :class:`_query.Query` .BakedQuery.to_query` 方法
检索对象。此方法传递了作为参数的  :class:`.Session` ，作为用于生成特定步骤的lambda可调用对象的参数。

::

    bakery = baked.bakery()

    # a baked query that will end up being used as a subquery
    my_subq = bakery(lambda s: s.query(User.id))
    my_subq += lambda q: q.filter(User.id == Address.user_id)

    # select a correlated subquery in the top columns list,
    # we have the "session" argument, pass that
    my_q = bakery(lambda s: s.query(Address.id, my_subq.to_query(s).as_scalar()))

    # use a correlated subquery in some of the criteria, we have
    # the "query" argument, pass that.
    my_q += lambda q: q.filter(my_subq.to_query(q).exists())

.. versionadded:: 1.3

.. _baked_with_before_compile:

使用before_compile事件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

从SQLAlchemy1.3.11开始，对于特定的  :class:`_query.Query` ，使用  :meth:` .QueryEvents.before_compile`  
事件将禁止烘焙查询系统缓存查询，如果事件钩子返回与传递的不同的新的  :class:`_query.Query` 
对象。这样，  :meth:`.QueryEvents.before_compile`   hook可以针对特定的
 :class:`_query.Query` 每次使用都被调用，以适应不同的钩子情况
每次都更改查询。为了允许
  :meth:`.QueryEvents.before_compile`  修改  :meth:` _query.Query`  对象，但仍然允许缓存结果，可以
在注册事件时传递``bake_ok=True``标志。

::

    @event.listens_for(Query, "before_compile", retval=True, bake_ok=True)
    def my_event(query):
        for desc in query.column_descriptions:
            if desc["type"] is User:
                entity = desc["entity"]
                query = query.filter(entity.deleted == False)
        return query

上述策略适用于每次以完全相同的方式修改给定的 :class:`_query.Query` 的事件，
不依赖于特定参数或更改的外部状态。

.. versionadded:: 1.3.11  - 添加了“bake_ok”标志到  :meth:`.QueryEvents.before_compile`  事件，并禁止缓存通过的“烘焙”扩展发生
   如果该标志未设置，则返回新的 :class:`_query.Query` 对象的事件处理程序。

禁用烘焙查询会话范围
------------------------------------

可以将标志  :paramref:`.Session.enable_baked_queries`  设置为False，
导致使用该标志时，所有烘焙请求都不使用高速缓存，而使用该标志时 , 当使用该请求对 :class:`.Session` 进行操作时。

::

    session = Session(engine, enable_baked_queries=False)

与所有会话标志一样，它也被工厂对象和方法接受，例如
  :class:`.sessionmaker` .sessionmaker.configure` 。

立即实现此标志的理由是，应用程序
可能由于用户定义的烘焙查询或其他烘焙问题而遇到问题，导致缓存关键字冲突的问题。
可以关闭此行为，以识别或消除烘焙查询作为问题的原因。

.. versionadded:: 1.2

懒加载集成
------------------------

.. versionchanged:: 1.4 从SQLAlchemy 1.4开始，“烘焙查询”系统不再是关系加载系统的一部分。
    相反，使用了 :ref:`本地缓存 <sql_caching>` 系统。


API文档
-----------------

.. autofunction:: bakery

.. autoclass:: BakedQuery
    :members:

.. autoclass:: Bakery
    :members:

.. autoclass:: Result
    :members:
    :noindex: