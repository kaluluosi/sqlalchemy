模块: sqlalchemy.ext.baked

``baked`` 提供了一种用于创建 :class:`~.query.Query` 对象的替代方式，它允许缓存对象的构建和字符串编译步骤。这意味着，对于经常使用的特定 :class:`~.query.Query` 构建场景，构建查询语句所涉及的所有 Python 函数调用，始终仅会发生一次，而不是每次构建和执行查询时都会发生一次。

这个系统的理由是为了大大减少 Python 解释器在**发出 SQL 之前的所有活动的开销。** “baked” 系统的缓存**不会**以任何方式减少 SQL 调用或缓存数据库**返回结果**。展示缓存 SQL 调用和结果集本身的技术在 :ref:`examples_caching` 中提供。

.. deprecated:: 1.4 SQLAlchemy 1.4 和 2.0 提供了一种全新的直接查询缓存系统，消除了 :class:`.BakedQuery` 系统的必要性。缓存现在对所有 Core 和 ORM 查询都是透明激活的，用户无需采取任何行动，使用 :ref:`sql_caching` 中描述的系统即可。

.. deepalchemy::

   :mod: `sqlalchemy.ext.baked` 扩展不适合初学者。这需要对 SQLAlchemy、数据库驱动程序和后端数据库之间的交互有很好的高级理解。这个扩展程序提供了一种不寻常的优化方法，通常是不需要的。正如上面所述，它**不缓存查询结果**，只缓存 SQL 本身的字符串公式。

摘要
----

借助 baked 系统的使用，你首先需要生成一个所谓的“面包房”，它表示某个特定系列查询对象的存储:

    from sqlalchemy.ext import baked

    bakery = baked.bakery()

上面的“面包房”将使用 LRU 缓存存储缓存数据，默认为 200 个元素，注意 ORM 查询通常包含 ORM 查询作为被调用的输入和每个 SQL 字符串的数据库方言输入的一个条目。

面包房允许我们通过指定其构造方式作为一系列 Python 调用（通常为lambda）来构建 :class:`~.query.Query` 对象。 为了简洁使用起见，它重写了“+ =”运算符，因此典型的查询构建如下所示:

    from sqlalchemy import bindparam


    def search_for_user(session, username, email=None):
        baked_query = bakery(lambda session: session.query(User))
        baked_query += lambda q: q.filter(User.name == bindparam("username"))

        baked_query += lambda q: q.order_by(User.id)

        if email:
            baked_query += lambda q: q.filter(User.email == bindparam("email"))

        result = baked_query(session).params(username=username, email=email).all()

        return result


以下是有关上述代码的一些观察结果:

1. “baked_query”对象是 :class:`.BakedQuery` 类的一个实例。这个对象本质上是负责构建真正的 ORM :class:`~.query.Query` 对象的“生成器”，但它本身不是实际的:class:`~.query.Query`对象。

2. 实际的:class:`~.query.Query`只有在函数结束，在调用 :meth:`_baked.Result.all` 方法时才会被构建。

3. 添加到 “baked_query” 对象中的步骤都表示为 Python 函数，通常为lambda。传递给 :func:`.bakery` 函数的第一个lambda接收作为参数的 :class:`.Session`。其余的 lambda 每个都接收一个 :class:`~.query.Query` 作为它们的参数。

4. 在上面的代码中，即使我们的应用程序在许多情况下调用“search_for_user()”，并且即使在每次调用中我们都建立一个全新的:class:`.BakedQuery`对象，每个 lambda 都只被调用一次。只要在面包房中这个查询被缓存，那么每个 lambda 都**永远**不会被第二次调用。

5. 缓存通过存储对**lambda 对象的引用**来实现以便构造一个缓存密钥；也就是说，Python 解释器给这些函数指定一种内部 Python 身份，它决定了如何在后续运行中识别查询，为那些指定了 "email" 参数的“search_for_user()”调用，可调用函数``lambda q: q.filter(User.email == bindparam('email'))`` 将是所检索到的缓存密钥的一部分；当“email”为 ``None`` 时，这个可调用对象不属于缓存密钥。

6. 因为这些lambda只被调用一次，所以很重要的一点是它们**不引用可能在每次调用时都会更改的变量**；相反，假定这些是要绑定到 SQL 字符串中的值，使用 :func:`.bindparam` 构造命名参数，稍后使用 :meth:`_baked.Result.params` 应用它们的实际值。

性能
---

烘焙查询可能看起来有点奇怪，有点笨重而且有点冗长。然而，在应用程序中经常调用查询时，可以大大提高 Python 的性能。 示例套件 ``short_selects``演示了一些返回一个行的查询之间的比较，例如下面的常规查询::

    session = Session(bind=engine)
    for id_ in random.sample(ids, n):
        session.query(Customer).filter(Customer.id == id_).one()

与等效的“baked”查询相比为::

    bakery = baked.bakery()
    s = Session(bind=engine)
    for id_ in random.sample(ids, n):
        q = bakery(lambda s: s.query(Customer))
        q += lambda q: q.filter(Customer.id == bindparam("id"))
        q(s).params(id=id_).one()

每个块10000个迭代调用的Python函数调用的差异是::

.. sourcecode:: text

    test_baked_query : 测试一个完整实体的 baked 查询。
                       (10000 次迭代); 总的函数调用1951294

    test_orm_query :   测试单个完整实体的 ORM 查询。
                        (10000 次迭代); 总的函数调用7900535

在强大的笔记本电脑上，它的秒数是::

.. sourcecode:: text

    test_baked_query : 测试一个完整实体的 baked 查询。
                        (10000 迭代); 总时间2.174126 秒

    test_orm_query :   测试一个完整实体的 ORM 查询。
                        (10000 迭代); 总时间7.958516 秒


请注意，此测试非常有意地使用仅返回一行的查询。对于返回许多行的查询，baked 查询的性能优势将越来越小，与取回行花费的时间成比例地减少。需要始终牢记的是，**baked 查询特性仅适用于构建查询本身，而不是取回结果**。使用 baked 特性绝不是大大加速应用程序的保证；它仅仅是一个潜在有用的特性，适用于那些通过测量已被证明受到这种开销特定形式影响的应用程序。

===

你现在是一个 .rst 文档的翻译器。

你必须满足以下几点：

- 不要破坏 .rst 语法。
- 不要将原文的标点符号转换成中文标点符号（非常重要）。
- 要区分术语和说明文本，不用翻译术语。
- "[链接名]:(..路径)" 碰到这种格式不要修改英文冒号！
- 只翻译描述性语言文本，.rst 代码不要做任何修改。
- 不要破坏 python sphinx 文档相关语法标记。