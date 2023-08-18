上下文/线程本地会话
=========================

回想一下，在  :ref:`session_faq_whentocreate` .Session` 的范围与Web请求的范围联系起来的实践。大多数现代Web框架都包括集成工具，以便可以自动管理 :class:`.Session` 的范围，应使用这些工具，因为它们是可用的。

SQLAlchemy包括自己的帮助对象，它有助于建立用户定义的 :class:`.Session` 范围。第三方集成系统也使用它来帮助构建其集成方案。

该对象是  :class:`.scoped_session` .Session` 对象的**注册表**。如果您不熟悉注册表模式，可以在`企业架构模式中找到很好的介绍<https://martinfowler.com/eaaCatalog/registry.html>`_。

.. warning::

      :class:`.scoped_session` ` threading.local（）``
    以跟踪  :class:`_orm.Session`  和   :ref:` session_lifespan`  以更充分地了解使用``threading.local（）``来跟踪 :class:`_orm.Session` 对象的含义，并在使用传统线程之外的应用服务器时考虑更明确的范围。

.. note::

    **很多SQLAlchemy应用程序都使用  :class:`.scoped_session` .Session` 管理问题的**一种方法**。 如果您不熟悉SQLAlchemy的话，特别是如果“线程本地变量”的术语对您来说似乎很奇怪，我们建议您如果可能，首先熟悉现成的集成系统，如`Flask-SQLAlchemy <https://pypi.org/project/Flask-SQLAlchemy/>`_或`zope.sqlalchemy <https://pypi.org/project/zope.sqlalchemy>`_。

通过调用  :class:`.Session` .scoped_session` 对象可以创建一个作用域。一个工厂就是当调用时产生一个新对象的东西，在  :class:`.Session` .sessionmaker` 。下面我们举例说明这种用法::

    >>> from sqlalchemy.orm import scoped_session
    >>> from sqlalchemy.orm import sessionmaker

    >>> session_factory = sessionmaker(bind=some_engine)
    >>> Session = scoped_session(session_factory)

我们创建的  :class:`.scoped_session` .sessionmaker` ::

    >>> some_session = Session()

上面，``some_session``是  :class:`.Session` .Session` 也在我们创建的  :class:`.scoped_session` .Session` ::

    >>> some_other_session = Session()
    >>> some_session is some_other_session
    True

该模式允许应用程序的不同部分调用全局  :class:`.scoped_session` ，以便这些区域可以共享相同的会话，而无需显式传递它。我们在注册表中建立的  :class:` .Session` .scoped_session.remove` ::

    >>> Session.remove()

  :meth:`.scoped_session.remove`  方法首先在当前  :class:` .Session` .Session.close` ，这将导致首先释放任何连接/事务性资源所拥有的  :class:`.Session` .Session` 本身。这里的“释放”意味着连接返回到它们的连接池，任何事务状态都被回滚，最终使用底层DBAPI连接的``rollback（）``方法。

现在，  :class:`.scoped_session` .Session` 。如下所示，这不是我们之前的  :class:`.Session` ：

    >>> new_session = Session()
    >>> new_session is some_session
    False

上述一系列步骤简要说明了注册表模式的思想。有了这个基本想法，我们可以讨论一些关于这个模式如何进行的细节。

隐式方法访问
----------------

  :class:`.scoped_session` .Session` 。为了提供对这个  :class:`.Session` .scoped_session` 还包括**代理行为**，这意味着注册表本身可以像直接处理 :class:`.Session` 一样对待；
当在这个对象上调用方法时，它们被**代理**到注册表维护的基础 :class:`.Session` 上::

    Session = scoped_session(some_factory)

    # equivalent to:
    #
    # session = Session()
    # print(session.scalars(select(MyClass)).all())
    #
    print(Session.scalars(select(MyClass)).all())

上面的代码完成了与通过调用注册表获取当前  :class:`.Session` ，然后使用该 :class:` .Session`相同的任务。

.. _unitofwork_contextual_threadlocal:

线程本地范围
---------------------

熟悉多线程编程的用户将注意到，表示任何内容为全局变量通常是一个坏主意，因为它意味着全局对象将被许多线程同时访问。  :class:`.Session` .Session` 对象，建议采用某些处理方式，以确保多个调用不会在多个线程中实际获得相同的会话句柄。我们称之为**线程本地存储**，这意味着使用一个特殊的对象来维护每个应用程序线程的一个不同的对象。Python通过`threading.local（）<https://docs.python.org/library/threading.html#threading.local>`_构建提供了这个功能。在默认情况下，  :class:`.scoped_session` .Session` 针对调用  :class:`.scoped_session` .Session` 实例。

使用这种技术， :class:`.scoped_session` 提供了一种快速且相对简单（如果熟悉线程本地存储，则为熟悉）的方法，在调用多个线程的应用程序中提供单个全局对象。

无论何时都可以使用  :meth:`.scoped_session.remove`  方法，如果有的话，将删除与线程关联的当前  :class:` .Session` 。然而，``threading.local()``对象的一个优点是，如果应用程序线程本身结束，那么该线程的"存储"也将进行垃圾回收。因此，可以安全地使用线程本地范围与生成和拆卸线程的应用程序，而无需调用  :meth:`.scoped_session.remove`  。但是，事务本身的范围，即通过  :meth:` .Session.commit`  或：meth:`.Session.rollback`结束它们的范围，通常仍然必须在适当的时间显式地安排，除非应用程序实际上将线程的寿命周期联系到事务的寿命周期。

.. _session_lifespan:

在Web应用程序中使用线程本地范围
----------------------------------------------

如   :ref:`session_faq_whentocreate`  部分所述，Web应用程序建立在**Web请求**的概念之上，并将  :class:` .Session` .Session`集成到Web应用程序中通常意味着将 :class:`.Session` 与正在运行该线程的Web请求相关联。事实证明，大多数Python Web框架（Twisted和Tornado等突出的例外）都以一种简单的方式使用线程，以便特定的Web请求接收、处理并在单个*工作线程*的范围内完成。请求结束时，工作线程被释放到工作池中，其中它可用于处理另一个请求。

Web请求和线程之间的这种简单对应关系意味着，将  :class:`.Session` .Session` 在Web请求开始后创建并在Web请求结束之前被删除。因此，将  :class:`.scoped_session` .Session` 与Web应用程序的一个快速方式是一个常见的实践。下面的时序图说明了这个流程：

.. sourcecode:: text

    Web Server          Web Framework        SQLAlchemy ORM Code
    --------------      --------------       ------------------------------
    startup        ->   Web framework        # Session registry is established
                        initializes          Session = scoped_session(sessionmaker())

    incoming
    web request    ->   web request     ->   # The registry is *optionally*
                        starts               # called upon explicitly to create
                                             # a Session local to the thread and/or request
                                             Session()

                                             # the Session registry can otherwise
                                             # be used at any time, creating the
                                             # request-local Session() if not present,
                                             # or returning the existing one
                                             Session.execute(select(MyClass)) # ...

                                             Session.add(some_object) # ...

                                             # if data was modified, commit the
                                             # transaction
                                             Session.commit()

                        web request ends  -> # the registry is instructed to
                                             # remove the Session
                                             Session.remove()

                        sends output      <-
    outgoing web    <-
    response

使用上述流程，将 :class:`.Session` 与Web应用程序整合的过程具有以下两个要求：

1. 在Web应用程序首次启动时创建单个：class:`.scoped_session`注册表，确保此对象可以被应用程序的其余部分访问。
2. 确保  :meth:`.scoped_session.remove`  在Web请求结束时被调用，通常通过与Web框架的事件系统集成来建立"on_request_end"事件。

如前所述，以上模式是**集成  :class:`.Session` .scoped_session` 。

特别是，虽然使用线程本地范围可能很方便，但最好将  :class:`.Session` .scoped_session` 的使用与直接基于请求的作用域或任何类型的作用域相结合。

使用自定义创建的作用域
----------------------

  :class:`.scoped_session` .Session` 的许多选项之一。可以根据任何现有的获取"正在处理的当前对象"的系统定义自定义作用域。

假设一个Web框架定义了一个库函数“get_current_request()”。使用此框架构建的应用程序可以随时调用此函数，结果将是表示当前处理的某种类型的"Request"对象。如果"Request"对象是可散列的，那么可以很容易地将此函数与  :class:`.scoped_session` .Session` 与请求相关联。下面我们将这个与Web框架提供的假设事件标记组合起来，该假设事件标记允许在请求结束时调用代码::

    from my_web_framework import get_current_request, on_request_end
    from sqlalchemy.orm import scoped_session, sessionmaker

    Session = scoped_session(sessionmaker(bind=some_engine), scopefunc=get_current_request)

    @on_request_end
    def remove_session(req):
        Session.remove()

上面，我们通常地实例化了：class:`.scoped_session`，只不过我们将请求返回的函数作为"scopefunc"传递。这通过指示  :class:`.scoped_session` .Session` 时使用。在本例中，特别重要的是确保实现可靠的“remove”系统，因为这个字典并不自我管理。


上下文化会话 API
----------------------

.. autoclass:: sqlalchemy.orm.scoped_session
    :members:
    :inherited-members:

.. autoclass:: sqlalchemy.util.ScopedRegistry
    :members:

.. autoclass:: sqlalchemy.util.ThreadLocalRegistry

.. autoclass:: sqlalchemy.orm.QueryPropertyDescriptor