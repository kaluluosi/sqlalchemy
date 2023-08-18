.. _unitofwork_contextual:

上下文/线程本地会话
======================

回想一下，在 :ref:`session_faq_whentocreate` 一节中，"会话范围"的概念被引入，强调了将 :class:`.Session` 的范围与 web 请求的范围关联的实践。大多数现代 web 框架都包括集成工具，以便自动管理 :class:`.Session` 的范围，并应该使用这些工具。

SQLAlchemy 包括它自己的辅助对象，它有助于建立用户定义的 :class:`.Session` 范围。第三方集成系统也会使用它来帮助构建它们的集成方案。

该对象是 :class:`.scoped_session` 对象，它表示一组 :class:`.Session` 对象的 **注册表**。如果您不熟悉注册表模式，则可以在 `企业级应用程序架构模式 <https://martinfowler.com/eaaCatalog/registry.html>`_ 中找到一个很好的介绍。

.. warning::

    :class:`.scoped_session` 注册表默认使用 Python 的 ``threading.local()``
    来跟踪 :class:`_orm.Session` 实例。**这不一定与所有应用程序服务器兼容**，
    特别是那些使用 greenlets 或其他替代并发控制形式的应用程序服务器，这可能会在中高并发情况下导致竞争条件（例如随机出现故障）。
    请阅读 :ref:`unitofwork_contextual_threadlocal` 和下面的 :ref:`session_lifespan`，
    以更全面地了解使用 ``threading.local()`` 来跟踪 :class:`_orm.Session` 对象的影响，并在使用不基于传统线程的应用程序服务器时考虑更明确的作用域。

.. note::

    :class:`.scoped_session` 对象是许多 SQLAlchemy 应用程序使用的非常受欢迎和有用的对象。
    然而，要注意的是，它只提供了解决 :class:`.Session` 管理问题的 **一种方法**。如果您是 SQLAlchemy 初学者，
    特别是如果术语 "线程本地变量" 对您来说似乎很奇怪，我们建议您优先选择一个成品集成系统，
    例如 `Flask-SQLAlchemy <https://pypi.org/project/Flask-SQLAlchemy/>`_ 或 `zope.sqlalchemy <https://pypi.org/project/zope.sqlalchemy>`_。

我们可以通过调用 :class:`.scoped_session` ，并传递一个**工厂函数**来构建一个 :class:`.scoped_session`，
该工厂函数可以创建新的 :class:`.Session` 对象。任何能够在被调用时生成新对象的函数都可以看作是一个工厂函数，对于 :class:`.Session`，最常见的工厂函数是本节之前介绍的 :class:`.sessionmaker`。下面演示了这种用法:

    >>> from sqlalchemy.orm import scoped_session
    >>> from sqlalchemy.orm import sessionmaker

    >>> session_factory = sessionmaker(bind=some_engine)
    >>> Session = scoped_session(session_factory)

我们创建的 :class:`.scoped_session` 对象现在将在我们“调用”注册表时调用 :class:`.sessionmaker`：

    >>> some_session = Session()

上面，``some_session`` 是 :class:`.Session` 的一个实例，现在我们可以使用它来连接到数据库。
我们创建的 :class:`.Session` 也在我们创建的 :class:`.scoped_session` 注册表中。
如果我们再次调用此注册表，我们将返回相同的 :class:`.Session`：

    >>> some_other_session = Session()
    >>> some_session is some_other_session
    True

这种模式允许应用程序的不同部分调用全局 :class:`.scoped_session`，
以便所有这些区域都可以共享相同的会话，无需显式传递它。
我们在注册表中建立的 :class:`.Session` 将保持不变，直到我们明确告诉该注册表将其处理掉，
通过调用 :meth:`.scoped_session.remove`：

    >>> Session.remove()

:meth:`.scoped_session.remove` 方法首先在当前 :class:`.Session` 上调用 :meth:`.Session.close`，
这会将 :class:`.Session` 拥有的任何连接/事务资源的占用效果释放，然后丢弃 :class:`.Session` 本身。
这里的 "释放" 意味着将连接返回到链接池，并回滚任何事务状态，最终使用底层 DBAPI 连接的 ``rollback()`` 方法。

此时，:class:`.scoped_session` 对象是“空的”，并在再次调用该注册表时创建一个**新的** :class:`.Session`。
如下所示，这不是我们之前拥有的 :class:`.Session`：

    >>> new_session = Session()
    >>> new_session is some_session
    False

上述步骤说明了注册表模式的要点。掌握这个基本思想后，我们可以讨论如何进行此模式的详细信息。

隐式方法访问
------------

:class:`.scoped_session` 的工作非常简单：将 :class:`.Session` 保留给所有请求它的人。为了更透明地访问该 :class:`.Session`，:class:`.scoped_session` 还包括 **代理行为**，这意味着可以直接如同 :class:`.Session` 一样处理该注册表；当在该对象上调用方法时，它们被**代理**到注册表维护的基础 :class:`.Session` 上：

    Session = scoped_session(some_factory)

    # 等效于:
    #
    # session = Session()
    # print(session.scalars(select(MyClass)).all())
    #
    print(Session.scalars(select(MyClass)).all())

上述代码实现的功能与通过调用上述注册表获取当前 :class:`.Session`，然后使用该 :class:`.Session` 完成的任务相同。

.. _unitofwork_contextual_threadlocal:

线程本地范围
---------------

熟悉多线程编程的用户会注意到，将任何东西表示为全局变量通常是一个坏主意，因为它意味着全局对象将被同时许多线程访问。:class:`.Session` 对象完全是为在一个线程中 **非并发** 使用而设计的，就多线程来说，这意味着“同一时间只能在一个线程中使用”。因此，我们在上面的 :class:`.scoped_session` 用法示例中表示一个相同 :class:`.Session` 对象在多次调用中维护的事实，表明必须存在某种处理方式，使跨许多线程的多个调用实际上不会获得一个句柄到相同的会话。我们称之为 **线程本地存储**，这意味着将使用一个特殊的对象，该对象将为每个应用程序线程维护一个不同的对象。Python 提供了这个对象，通过 `threading.local() <https://docs.python.org/library/threading.html#threading.local>`_ 构建。:class:`.scoped_session` 对象默认情况下使用此对象作为存储，以便单个 :class:`.Session` 为所有调用 :class:`.scoped_session` 注册表的人维护，但在单个线程的范围内。在不同的线程上调用此注册表的调用者将获得一个与该线程本地的 :class:`.Session` 实例。

使用这种技术，:class:`.scoped_session` 提供了一种快速而相对简单的（如果熟悉线程本地存储的话）方式，在多个线程中从全局角度提供单个对象的应用程序中调用。

总是使用 :meth:`.scoped_session.remove` 方法删除当前与线程关联的任何 :class:`.Session`。但是，``threading.local()`` 对象的一个优点是，如果应用程序线程本身结束，那么该线程的“存储”也将被垃圾回收。因此，使用线程本地范围的应用程序可以生成和销毁线程，而不需要调用 :meth:`.scoped_session.remove`。但是，事务本身的范围，即通过 :meth:`.Session.commit` 或 :meth:`.Session.rollback` 结束它们的范围，通常仍然是必须在适当的时间显式设置的，除非应用程序实际上将线程生存期与事务的生存期相关联。

.. _session_lifespan:

在 Web 应用程序中使用线程本地范围
------------------------------------

如 :ref:`session_faq_whentocreate` 一节中所述，一个 Web 应用程序的架构围绕 **Web 请求** 的概念建立，将 :class:`.Session` 与该请求相关联通常意味着 :class:`.Session` 将与该请求关联。事实证明，大多数 Python Web 框架，与 Twisted 和 Tornado 这样的异步框架不同，使用线程的简单方式，其中接收特定 Web 请求，处理并完成该请求在单个 **工作线程** 的范围内完成。Web 请求结束时，工作线程被释放到一个工作池中，它可用于处理另一个请求。

这种 Web 请求与线程之间的简单对应关系意味着，要将 :class:`.Session` 与线程相关联，也意味着它还与在该线程内运行的 Web 请求相关联，反之亦然，只要 :class:`.Session` 是在 Web 请求开始之后创建的，并在 Web 请求结束之前撤销，就会自然关联该 Web 请求，这是一种常见的做法，使用 :class:`.scoped_session` 快速地将 :class:`.Session` 与 Web 应用程序集成。下面的序列图说明了此流程：

.. sourcecode:: text

    Web 服务器          Web 框架              SQLAlchemy ORM 代码
    --------------      --------------        ------------------------------
    启动        ->      Web 框架          ->   # 建立会话注册表
                        初始化请求         Session = scoped_session(sessionmaker())

    进入请求
    Web 请求     ->      Web 请求进行     ->   # 可选地，该注册表
                        开始                 # 可显式调用以创建适合该线程和 / 或请求的 Session
                                              Session()

                                              # 会话注册表也可能在任何时候使用，如果不存在，则创建请求本地 Session()，
                                              # 否则返回现有的 Session()
                                              Session.execute(select(MyClass)) # ...

                                              Session.add(some_object) # ...

                                              # 如果修改了数据，请提交事务
                                              Session.commit()

                        结束并删除请求  ->     # 指示注册表删除会话
                                              Session.remove()

                        发送输出         <-
    响应 Web 请求   <-
    结束

使用上述流程，将 :class:`.Session` 与 Web 应用程序集成的流程具有以下两个要求：

1. 在 Web 应用程序一开始时创建一个 :class:`.scoped_session` 注册表，确保应用程序的所有其他部分都可以访问该对象。
2. 确保在 Web 请求结束时调用 :meth:`.scoped_session.remove`，通常通过与 Web 框架的事件系统进行集成以建立“请求结束”事件来实现。

如前所述，上述模式是 **集成 :class:`.Session` 与 Web 框架的潜在方法之一**，其中特别假设 **Web 框架将 Web 请求与应用程序线程相关联**。但是，**强烈建议使用 Web 框架提供的集成工具，如果有的话**，
而不是 :class:`.scoped_session`。

特别地，在 :class:`.Session` 直接与请求相关联而不是与当前线程相关联，比使用线程本地更加优选。下面的自定义范围部分详细介绍了一种更高级的配置，该配置可以将 :class:`.scoped_session` 的使用与直接基于请求的作用域或任何类型的作用域相结合。

使用自定义创建的作用域
----------------------------

:class:`.scoped_session` 对象的默认行为“线程本地”范围仅是可将 :class:`.Session` “限定范围”的许多选项之一。可以基于任何现有的系统定义自定义范围，以获取当前操作的“当前对象”。

假设 Web 框架定义了一个库函数``get_current_request()``。使用这个框架构建的应用程序可以随时调用此函数，结果是表示当前正在处理的请求的一种 "Request" 对象。如果 "Request" 对象是可哈希的，则可以轻松地将此函数与 :class:`.scoped_session` 集成，以将 :class:`.Session` 与该请求相关联。以下演示了这一点，结合一个假设的事件标记，该事件标记由 Web 框架提供，允许在请求结束时调用代码：

    from my_web_framework import get_current_request, on_request_end
    from sqlalchemy.orm import scoped_session, sessionmaker

    Session = scoped_session(sessionmaker(bind=some_engine), scopefunc=get_current_request)


    @on_request_end
    def remove_session(req):
        Session.remove()

上面，我们以通常的方式实例化 :class:`.scoped_session`，除了将请求返回函数作为 "scopefunc" 传递。这会指示 :class:`.scoped_session` 对象使用此函数在注册表调用时生成一个字典键来返回当前 :class:`.Session`。在这种情况下，特别重要的是，我们确保实现了可靠的“remove”系统，因为该字典否则不进行自我管理。

上下文会话 API
---------------

.. autoclass:: sqlalchemy.orm.scoped_session
    :members:
    :inherited-members:

.. autoclass:: sqlalchemy.util.ScopedRegistry
    :members:

.. autoclass:: sqlalchemy.util.ThreadLocalRegistry

.. autoclass:: sqlalchemy.orm.QueryPropertyDescriptor