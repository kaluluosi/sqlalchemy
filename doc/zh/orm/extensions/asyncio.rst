.. _asyncio_toplevel:

异步 I/O (asyncio)
==========================

支持 Python asyncio。支持使用 asyncio 兼容方言，包括 Core 和 ORM。

.. versionadded:: 1.4

.. warning:: 请阅读 :ref:`asyncio_install`，以获取包括 Apple M1 架构在内的重要平台安装注意事项。

.. seealso::

    :ref:`change_3414` - 初始功能公告

    :ref:`examples_asyncio` - 例子脚本说明了在 asyncio 扩展中使用 Core 和 ORM 的使用范例。

.. _asyncio_install:

异步 I/O 平台安装注意事项（包括 Apple M1）
---------------------------------------------------------

异步 io 扩展仅支持 Python 3。它还依赖于 `greenlet <https://pypi.org/project/greenlet/>`_ 库。这
一依赖关系默认安装在包括以下常见机器平台：

.. sourcecode:: text

    x86_64 aarch64 ppc64le amd64 win32

对于上述平台，“greenlet”已知提供预构建的轮文件。对于其他平台，“greenlet”不会默认安装;
“greenlet”的当前文件清单可以在 `Greenlet - Download Files <https://pypi.org/project/greenlet/#files>`_ 中查看。
请注意，其中有许多架构被省略，包括 Apple M1。

要在确保存在“greenlet”依赖关系的情况下安装 SQLAlchemy，请使用
下面的“[asyncio]”'setuptools extra'  <https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras>`_ 安装，这也将指示“pip”安装“greenlet”:

.. sourcecode:: text

  pip install sqlalchemy[asyncio]

请注意，在没有预构建轮文件的平台上安装“greenlet”意味着必须构建“greenlet”，这需要
Python 的开发库也要存在。


概述 - Core
---------------

对于 Core 使用，:func:`_asyncio.create_async_engine` 函数创建
:class:`_asyncio.AsyncEngine` 的实例，它然后提供传统的 :class:`_engine.Engine`
API 的异步版本。:class:`_asyncio.AsyncEngine` 通过其
:meth:`_asyncio.AsyncEngine.connect` 和 :meth:`_asyncio.AsyncEngine.begin`
方法都会提供一个异步上下文管理器的形式，从而提供一个 :class:`_asyncio.AsyncConnection`。 
:class:`_asyncio.AsyncConnection` 然后可以使用 :meth:`_asyncio.AsyncConnection.execute` 方法来提供一个缓冲的 :class:`_engine.Result`，
或使用 :meth:`_asyncio.AsyncConnection.stream` 方法来提供一个流式服务器端的 :class:`_asyncio.AsyncResult`来调用语句::

    import asyncio

    from sqlalchemy import Column
    from sqlalchemy import MetaData
    from sqlalchemy import select
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy.ext.asyncio import create_async_engine

    meta = MetaData()
    t1 = Table("t1", meta, Column("name", String(50), primary_key=True))


    async def async_main() -> None:
        engine = create_async_engine(
            "postgresql+asyncpg://scott:tiger@localhost/test",
            echo=True,
        )

        async with engine.begin() as conn:
            await conn.run_sync(meta.create_all)

            await conn.execute(
                t1.insert(), [{"name": "some name 1"}, {"name": "some name 2"}]
            )

        async with engine.connect() as conn:
            # select a Result, which will be delivered with buffered
            # results
            result = await conn.execute(select(t1).where(t1.c.name == "some name 1"))

            print(result.fetchall())

        # for AsyncEngine created in function scope, close and
        # clean-up pooled connections
        await engine.dispose()


    asyncio.run(async_main())

上面的 :meth:`_asyncio.AsyncConnection.run_sync` 可以用于调用特殊的DDL函数，例如 :meth:`_schema.MetaData.create_all`，这些函数不包括 awaitable 钩子。

.. tip:: 建议在使用 :class:`_asyncio.AsyncEngine` 对象的上下文范围将要超出范围并被垃圾回收的作用域中使用 ``await`` 调用 :meth:`_asyncio.AsyncEngine.dispose` 方法，如上面例子中的 ``async_main``
    函数所示。这将确保由连接池保持打开的任何连接都在可等待上下文中正确地释放。与使用阻塞IO不同，由于没有机会调用``await``，所以 SQLAlchemy 无法在这些方法中适当地处理这些连接，如``__del__``或弱引用终结者中没有机会调用。未显式处理引擎超出作用域时可能会导致发出类似于运行时错误：事件循环已关闭的警告。

:class:`_asyncio.AsyncConnection` 也通过 :meth:`_asyncio.AsyncConnection.stream` 方法特性提供了“流式”API。该结果对象使用服务器端光标，并提供一个异步/ await API，例如异步迭代器::

    async with engine.connect() as conn:
        async_result = await conn.stream(select(t1))

        async for row in async_result:
            print("row: %s" % (row,))

.. _asyncio_orm:


概述 - ORM
---------------

使用 :term:`2.0 style`，:class:`_asyncio.AsyncSession` 类提供了完整的 ORM 功能。

在使用的默认模式下，必须小心避免关于ORM关系和列属性的 :term:`lazy loading` 或其他过期属性访问;以下一章 :ref:`asyncio_orm_avoid_lazyloads` 详细说明了这一点。

.. warning::

    单个 :class:`_asyncio.AsyncSession` 实例**不能安全地在多个并发任务中使用**。请参阅 :ref:`asyncio_concurrency` 和 :ref:`session_faq_threadsafe` 讨论背景知识。

下面的示例演示了一个完整的示例，包括映射器和会话配置::

    from __future__ import annotations

    import asyncio
    import datetime
    from typing import List

    from sqlalchemy import ForeignKey
    from sqlalchemy import func
    from sqlalchemy import select
    from sqlalchemy.ext.asyncio import AsyncAttrs
    from sqlalchemy.ext.asyncio import async_sessionmaker
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship
    from sqlalchemy.orm import selectinload


    class Base(AsyncAttrs, DeclarativeBase):
        pass


    class A(Base):
        __tablename__ = "a"

        id: Mapped[int] = mapped_column(primary_key=True)
        data: Mapped[str]
        create_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())
        bs: Mapped[List[B]] = relationship()


    class B(Base):
        __tablename__ = "b"
        id: Mapped[int] = mapped_column(primary_key=True)
        a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
        data: Mapped[str]


    async def insert_objects(async_session: async_sessionmaker[AsyncSession]) -> None:
        async with async_session() as session:
            async with session.begin():
                session.add_all(
                    [
                        A(bs=[B(), B()], data="a1"),
                        A(bs=[], data="a2"),
                        A(bs=[B(), B()], data="a3"),
                    ]
                )


    async def select_and_update_objects(
        async_session: async_sessionmaker[AsyncSession],
    ) -> None:
        async with async_session() as session:
            stmt = select(A).options(selectinload(A.bs))

            result = await session.execute(stmt)

            for a1 in result.scalars():
                print(a1)
                print(f"created at: {a1.create_date}")
                for b1 in a1.bs:
                    print(b1)

            result = await session.execute(select(A).order_by(A.id).limit(1))

            a1 = result.scalars().one()

            a1.data = "new data"

            await session.commit()

            # access attribute subsequent to commit; this is what
            # expire_on_commit=False allows
            print(a1.data)

            # alternatively, AsyncAttrs may be used to access any attribute
            # as an awaitable (new in 2.0.13)
            for b1 in await a1.awaitable_attrs.bs:
                print(b1)


    async def async_main() -> None:
        engine = create_async_engine(
            "postgresql+asyncpg://scott:tiger@localhost/test",
            echo=True,
        )

        # async_sessionmaker: a factory for new AsyncSession objects.
        # expire_on_commit - don't expire objects after transaction commit
        async_session = async_sessionmaker(engine, expire_on_commit=False)

        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)

        await insert_objects(async_session)
        await select_and_update_objects(async_session)

        # for AsyncEngine created in function scope, close and
        # clean-up pooled connections
        await engine.dispose()


    asyncio.run(async_main())

在上面的示例中，使用 :class:`_asyncio.async_sessionmaker` 辅助程序来实例化 :class:`_asyncio.AsyncSession` 
对象，它提供了一组固定参数的 :class:`_asyncio.AsyncSession` 对象的工厂，这些参数在这里包括将其与特定的数据库 URL 相关联的 :class:`_asyncio.AsyncEngine`。
然后将它传递给其他方法，在这些方法中，它可以在 Python 异步上下文管理器中使用 (即 ``async with:`` 语句)，以便在块结束时自动关闭；
这与调用 :meth:`_asyncio.AsyncSession.close` 方法是相当的。


.. _asyncio_concurrency:

使用 AsyncSession 与并发任务
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`_asyncio.AsyncSession` 对象是一个**可变的、有状态的对象**，它表示一个**单独的处于状态的数据库事务**。使用 asyncio 与 Python 中的 API，例如 ``asyncio.gather()``，
应该使用**每个单独任务一个单独的 ``_asyncio.AsyncSession``**。

关于 :class:`_orm.Session` 和 :class:`_asyncio.AsyncSession` 如何在并发工作负载中使用它们的一般描述，请参阅 :ref:`session_faq_threadsafe`。

.. _asyncio_orm_avoid_lazyloads:

在使用 AsyncSession 时避免隐式 I/O
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在传统的 asyncio 中，应用程序需要避免可以发生 I/O 的属性访问点。以下是可用于帮助此事的技术，其中许多在上面的示例中进行了说明。

* :class:`_asyncio.AsyncAttrs` 混入的属性（如懒加载关系、延迟列或表达式或过期属性访问）可以利用。当将其添加到特定类中或更一般地添加到 Declarative ``Base`` 超类中时，
它提供了一个访问器 :attr:`_asyncio.AsyncAttrs.awaitable_attrs`，该访问器以任何属性作为等待::

    from __future__ import annotations

    from typing import List

    from sqlalchemy.ext.asyncio import AsyncAttrs
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship


    class Base(AsyncAttrs, DeclarativeBase):
        pass


    class A(Base):
        __tablename__ = "a"

        # ... rest of mapping ...

        bs: Mapped[List[B]] = relationship()


    class B(Base):
        __tablename__ = "b"

        # ... rest of mapping ...

  当懒加载未使用时，新加载的“ A”实例上的``A.bs``集合通常会使用 :term:`懒加载`，为了成功，通常需要向数据库发出 I/O，这在 asyncio 下会失败，因为不允许隐式 I/O。 
  为了在 asyncio 下直接访问此属性而不需要任何预加载操作，可以将其作为 awaitable 访问，方法是在 :attr:`_asyncio.AsyncAttrs.awaitable_attrs` 前缀中命名::

    a1 = (await session.scalars(select(A))).one()
    for b1 in await a1.awaitable_attrs.bs:
        print(b1)

  :class:`_asyncio.AsyncAttrs` 混入为 :meth:`_asyncio.AsyncSession.run_sync` 方法所用的内部方法提供了一个简洁的外观。

  .. versionadded:: 2.0.13

  .. seealso::

      :class:`_asyncio.AsyncAttrs`


* 集合可以替换为**仅写集合**，它们永远不会隐式地发出 I/O，方法是使用 SQLAlchemy 2.0 中的 :ref:`write_only_relationship` 功能。使用此功能，永远不会从集合中读取数据，只能使用显式的查询语句来查询。请查看 :ref:`examples_asyncio` 中的 ``async_orm_writeonly.py`` 示例，以获取使用 asyncio 相关的写入仅集合的示例。使用写入仅集合，程序的行为简单并易于预测。然而，缺点是没有用于一次加载许多这些集合的内置系统，默认情况下必须手动执行。因此，很多点下面的小圆点针对的是在使用 asyncio 的传统懒加载关系时的具体技术，这需要更多的小心。

* 如果不使用 :class:`_asyncio.AsyncAttrs`，则应将关系声明为 ``lazy="raise"``，以便默认情况下不会尝试发出 SQL。为了加载集合，会使用 :term:`eager loading` 代替。

* 最有用的 eager loading 策略是 :func:`_orm.selectinload` eager loader，在 :func:`selectinload()` 的作用域内，它用于 eager
  加载 ``A.bs`` 集合::

      stmt = select(A).options(selectinload(A.bs))

* 在构建新对象时，**集合始终分配一个默认的、空的集合**，例如在上面的示例中的列表 ``A(bs=[], data="a2")``：
  这使得在刷新 ``A`` 时，该对象上的 ``.bs`` 集合可以存在并且可读；否则，当刷新 ``A`` 时，``.bs`` 将不会加载并且访问将引发错误。

* :class:`_asyncio.AsyncSession` 配置为使用 :paramref:`_orm.Session.expire_on_commit` 设置为 False，以便我们可以在调用 :meth:`_asyncio.AsyncSession.commit` 后访问对象上的属性，例如下面的一行，其中我们访问一个属性::

      async_session = AsyncSession(engine, expire_on_commit=False)

      # sessionmaker version
      async_session = async_sessionmaker(engine, expire_on_commit=False)

      async with async_session() as session:
          result = await session.execute(select(A).order_by(A.id))

          a1 = result.scalars().first()

          # commit would normally expire all attributes
          await session.commit()

          # access attribute subsequent to commit; this is what
          # expire_on_commit=False allows
          print(a1.data)

其他准则包括：

* 应该避免使用 :meth:`_asyncio.AsyncSession.expire` 方法，而应该使用 :meth:`_asyncio.AsyncSession.refresh`；**如果** 确实需要到期。通常情况下不应需要到期，应该将 :paramref:`_orm.Session.expire_on_commit` 正确设置为 ``False``。

* 懒加载的关系可以使用 :meth:`_asyncio.AsyncSession.refresh` 在 asyncio 中加载 **如果**在:paramref:`_orm.Session.refresh.attribute_names` 参数中显式命名了要访问的属性名。 

  .. versionadded:: 2.0.4 Added support for
     :meth:`_asyncio.AsyncSession.refresh` and the underlying
     :meth:`_orm.Session.refresh` method to force lazy-loaded relationships
     to load, if they are named explicitly in the
     :paramref:`_orm.Session.refresh.attribute_names` parameter.
     In previous versions, the relationship would be silently skipped even
     if named in the parameter.

* 应该避免使用 :ref:`unitofwork_cascades` 中的 ``all`` 级联选项，而应该显式列出所需的级联特性。`` all`` 级联选项意味着包括 :ref:`cascade_refresh_expire` 设置在内，这意味着 :meth:`.AsyncSession.refresh` 方法将对相关对象上的属性到期，但不一定会在假定未配置饥饿加载的情况下刷新这些相关对象，即使它们已经到期了。

* :func:`_orm.deferred` 列，如果使用的话，应该使用适当的加载程序选项，除了在  :func:`_orm.relationship` 构造中注明的以外。请参阅 :ref:`orm_queryguide_column_deferral`，了解关于延迟列加载的背景知识。

.. _dynamic_asyncio:

* 描述在 :ref:`dynamic_relationship` 中的“dynamic”关系加载程序策略默认情况下不与为 asyncio 而生的方法兼容。只有在 :meth:`_asyncio.AsyncSession.run_sync` 方法被描述的异步上下文中调用时，或者使用其“`.statement`` ”属性获取常规选择时，它才能直接使用。
  ``_asyncio.AsyncSession.run_sync``。此方法，例如.，异步 ORM 内容的应用程序可以将数据库相关方法打包到功能中，该功能使用 :meth:`_asyncio.AsyncSession.run_sync` 调用。

修改上面的例子，如果我们没有使用 :func:`_orm.selectinload` 来处理 ``A.bs`` 集合，我们可以在不同函数的分离函数中完成具体的访问::

    import asyncio

    from sqlalchemy import select
    from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine


    def fetch_and_update_objects(session):
        """run traditional sync-style ORM code in a function that will be
        invoked within an awaitable.

        """

        # the session object here is a traditional ORM Session.
        # all features are available here including legacy Query use.

        stmt = select(A)

        result = session.execute(stmt)
        for a1 in result.scalars():
            print(a1)

            # lazy loads
            for b1 in a1.bs:
                print(b1)

        # legacy Query use
        a1 = session.query(A).order_by(A.id).first()

        a1.data = "new data"


    async def async_main():
        engine = create_async_engine(
            "postgresql+asyncpg://scott:tiger@localhost/test",
            echo=True,
        )
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.drop_all)
            await conn.run_sync(Base.metadata.create_all)

        async with AsyncSession(engine) as session:
            async with session.begin():
                session.add_all(
                    [
                        A(bs=[B(), B()], data="a1"),
                        A(bs=[B()], data="a2"),
                        A(bs=[B(), B()], data="a3"),
                    ]
                )

            await session.run_sync(fetch_and_update_objects)

            await session.commit()

        # for AsyncEngine created in function scope, close and
        # clean-up pooled connections
        await engine.dispose()


    asyncio.run(async_main())

在上面的示例中，用 :class:`_asyncio.AsyncSession` 代替 :class:`_orm.Session` 以实现异步任务。使用 :paramref:`_asyncio.AsyncSession.run_sync` 包装处理使用 ORM 功能，使 ORM 功能为异步可用。 

使用 Session 退出异步模式，创建新的运行时 numpy 的异步环境    def my_after_commit(session):
        print("after commit!")


    async def go():
        await session.execute(text("select 1"))
        await session.commit()

        await session.close()
        await engine.dispose()


    asyncio.run(go())

  输出：

  .. sourcecode:: text

    before commit!
    execute from event
    after commit!

* **ORM Events on async_sessionmaker（async_sessionmaker 上的 ORM 事件）**

  对于这种情况，我们将 :class:`_orm.sessionmaker` 作为事件目标，然后将它赋给 :class:`_asyncio.async_sessionmaker`，
  并使用 :paramref:`_asyncio.async_sessionmaker.sync_session_class` 参数来指定对应的同步 session 类：：

    import asyncio

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import async_sessionmaker
    from sqlalchemy.orm import sessionmaker

    sync_maker = sessionmaker()
    maker = async_sessionmaker(sync_session_class=sync_maker)


    @event.listens_for(sync_maker, "before_commit")
    def before_commit(session):
        print("before commit")


    async def main():
        async_session = maker()

        await async_session.commit()


    asyncio.run(main())

  输出：

  .. sourcecode:: text

    before commit


.. topic:: asyncio 和事件，两个对立面

    根据其特性，SQLAlchemy 事件将在特定的 SQLAlchemy 进程**内部**发生，
    也就是说，事件总会在用户端代码调用某个特定的 SQLAlchemy API 之后、这个 API 的某个内部部分之前发生。

    而 asyncio 扩展的体系结构与此相反，它发生在 SQLAlchemy 的常规从用户端 API 到DBAPI函数的流程**外部**。

    消息流可以视为如下：

    .. sourcecode:: text

         SQLAlchemy    SQLAlchemy        SQLAlchemy          SQLAlchemy   plain
          asyncio      asyncio           ORM/Core            asyncio      asyncio
          (public      (internal)                            (internal)
          facing)
        -------------|------------|------------------------|-----------|------------
        asyncio API  |            |                        |           |
        call  ->     |            |                        |           |
                     |  ->  ->    |                        |  ->  ->   |
                     |~~~~~~~~~~~~| sync API call ->       |~~~~~~~~~~~|
                     | asyncio    |  event hooks ->        | sync      |
                     | to         |   invoke action ->     | to        |
                     | sync       |    event hooks ->      | asyncio   |
                     | (greenlet) |     dialect ->         | (leave    |
                     |~~~~~~~~~~~~|      event hooks ->    | greenlet) |
                     |  ->  ->    |       sync adapted     |~~~~~~~~~~~|
                     |            |               DBAPI -> |  ->  ->   | asyncio
                     |            |                        |           | driver -> database


    在上面的流程中，API 调用总是从 asyncio 开始，流经同步 API，最后以 asyncio 结束，然后结果通过相反的方式传播回来。
    在这之间，首先将消息转换为同步类型 API 使用，然后再转换回来成为异步类型。
    然后事件钩子位于“同步类型 API 使用”的中间位置。
    由此得出的结论是，在事件钩子中呈现的 API 发生在由 asyncio API 请求适配为同步后的过程中，
    而向数据库 API 发送的输出消息将被透明地转换为 asyncio。

.. _asyncio_events_run_async:

在连接池和其他事件中使用仅可等待的驱动方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

正如在上一节中所讨论的那样，事件处理程序（例如那些面向 :class:`.PoolEvents` 事件处理程序）接收到一个同步类型的“DBAPI”连接，
该连接是 SQLAlchemy asyncio dialects 提供的包装对象，用于将底层的 asyncio“driver”连接转换为可以被 SQLAlchemy 内部使用的连接。
当用户定义的实现需要直接使用终极“driver”连接来使用其可等待方法（即在该驱动程序连接上使用仅可等待方法）时，就会出现一种特殊情况。举个例子，
asyncpg 驱动程序提供了 ``.set_type_codec()`` 方法。

为了应对这种情况，SQLAlchemy 的 :class:`.AdaptedConnection` 类提供了一个方法 :meth:`.AdaptedConnection.run_async`，
该方法允许在事件处理程序或其他 SQLAlchemy 内部的“同步”上下文中调用可等待的函数。该方法直接类比于 :meth:`_asyncio.AsyncConnection.run_sync`，
后者允许同步风格的方法在异步环境下运行。

应将 :meth:`.AdaptedConnection.run_async` 传递一个函数作为参数，
该函数将接受最内部的“driver”连接作为单个参数，并返回一个可等待对象，
该对象将由 :meth:`.AdaptedConnection.run_async` 方法调用。给定函数本身不需要声明为 ``async``；
它可以是 Python ``lambda:``，因为返回的可等待值将在返回后被调用。::

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine(...)


    @event.listens_for(engine.sync_engine, "connect")
    def register_custom_types(dbapi_connection, *args):
        dbapi_connection.run_async(
            lambda connection: connection.set_type_codec(
                "MyCustomType",
                encoder,
                decoder,  # ...
            )
        )

以上，传递给“register_custom_types”事件处理程序的对象是 :class:`.AdaptedConnection` 的一个实例，
后者为基础的异步类型驱动程级连接对象提供了类似于 DBAPI 的接口。然后，:meth:`.AdaptedConnection.run_async` 方法
提供了一个可等待环境，其中底层的驱动程序级连接可能会被操作。

.. versionadded:: 1.4.30


使用多个 asyncio 事件循环
----------------------------------

一个使用了多个事件循环的应用程序（例如将 asyncio 与多线程结合使用的非常不寻常的情况），
在使用默认池实现时，不应在不同的事件循环之间共享同一个 :class:`_asyncio.AsyncEngine`。

如果必须将 :class:`_asyncio.AsyncEngine` 从一个事件循环传递到另一个事件循环，
则应该在它被重用在一个新事件循环上之前调用 :meth:`_asyncio.AsyncEngine.dispose()` 方法。
如果不这样做，可能会导致类似于“Task <Task pending ...> got Future attached to a different loop”的运行时错误。

如果必须在不同的循环之间共享相同的引擎，那么应该将它配置为使用 :class:`~sqlalchemy.pool.NullPool` 来禁用池，
以防止引擎使用任何一个连接多次：

    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.pool import NullPool

    engine = create_async_engine(
        "postgresql+asyncpg://user:pass@host/dbname",
        poolclass=NullPool,
    )

.. _asyncio_scoped_session:

使用 asyncio scoped session（使用 asyncio scoped session）
----------------------------

“scoped session” 模式在 SQLAlchemy 中的使用方式是 :class:`.scoped_session` 对象，在 asyncio 中也有相应的版本，称为使用了适配器的 :class:`_asyncio.async_scoped_session`。

.. tip:: SQLAlchemy 通常不推荐使用“scoped”模式进行新开发，因为它依赖于一个可以被修改的全局状态，在线程或任务完成后必须在显式地被拆除。
   特别是在使用 asyncio 时，将 :class:`_asyncio.AsyncSession` 直接传递给需要它的可等待函数，很可能是一个更好的想法。

当使用 :class:`_asyncio.async_scoped_session` 时，由于在 asyncio 上下文中不存在“线程本地”概念，
所以必须在构造函数中提供“scopefunc”参数。下面的示例说明了使用 ``asyncio.current_task()`` 函数的情况：

    from asyncio import current_task

    from sqlalchemy.ext.asyncio import (
        async_scoped_session,
        async_sessionmaker,
    )

    async_session_factory = async_sessionmaker(
        some_async_engine,
        expire_on_commit=False,
    )
    AsyncScopedSession = async_scoped_session(
        async_session_factory,
        scopefunc=current_task,
    )
    some_async_session = AsyncScopedSession()

.. warning:: 由 :class:`_asyncio.async_scoped_session` 使用的“scopefunc”会在任务中**任意多次调用**，
    每次访问底层 :class:`_asyncio.AsyncSession` 时都会调用一次。因此，函数必须是**幂等**和轻量级的，
    不得尝试创建或更改任何状态（例如建立回调等）。

.. warning:: 使用“当前任务”作为作为上下文的“关键字”需要从最外层可等待对象中调用 :meth:`_asyncio.async_scoped_session.remove` 方法，
    以确保在任务完成时从注册表中删除该键，否则，任务句柄以及 :class:`_asyncio.AsyncSession` 将留在内存中，从而实际上创建了内存泄漏。
    参见下面的示例，展示了正确使用 :meth:`_asyncio.async_scoped_session.remove` 方法的方法。

:class:`_asyncio.async_scoped_session` 包括了相似于 :class:`.scoped_session` 的**代理行为**，因此它可以直接被视为 :class:`_asyncio.AsyncSession`，
需要注意的是，对于方法（包括 :meth:`_asyncio.async_scoped_session.remove` 方法）来说，通常需要使用一般的 ``await`` 关键字。

.. versionadded:: 1.4.19

.. currentmodule:: sqlalchemy.ext.asyncio


.. _asyncio_inspector:

使用 Inspector 检查模式对象
---------------------------------------------------

SQLAlchemy 尚未提供 asyncio 版本的 :class:`_reflection.Inspector`（介绍参见 :ref:`metadata_reflection_inspector`），
但是，现有的接口可以在 asyncio 上下文中使用，方法是利用 :class:`_asyncio.AsyncConnection` 的 :meth:`_asyncio.AsyncConnection.run_sync` 方法：

    import asyncio

    from sqlalchemy import inspect
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost/test")


    def use_inspector(conn):
        inspector = inspect(conn)
        # use the inspector
        print(inspector.get_view_names())
        # return any value to the caller
        return inspector.get_table_names()


    async def async_main():
        async with engine.connect() as conn:
            tables = await conn.run_sync(use_inspector)


    asyncio.run(async_main())

.. seealso::

    :ref:`metadata_reflection`

    :ref:`inspection_toplevel`

引擎 API 文档
-------------------------

.. autofunction:: create_async_engine

.. autofunction:: async_engine_from_config

.. autofunction:: create_async_pool_from_url

.. autoclass:: AsyncEngine
   :members:

.. autoclass:: AsyncConnection
   :members:

.. autoclass:: AsyncTransaction
   :members:

结果集 API 文档
----------------------------------

:class:`_asyncio.AsyncResult` 对象是 :class:`_result.Result` 对象的适配异步版本的对象。
只有当使用 :meth:`_asyncio.AsyncConnection.stream` 或 :meth:`_asyncio.AsyncSession.stream` 方法时，
才会返回一个结果对象，该对象在一个活动的数据库游标上。

.. autoclass:: AsyncResult
   :members:
   :inherited-members:

.. autoclass:: AsyncScalarResult
   :members:
   :inherited-members:

.. autoclass:: AsyncMappingResult
   :members:
   :inherited-members:

.. autoclass:: AsyncTupleResult

ORM Session API 文档
-----------------------------

.. autofunction:: async_object_session

.. autofunction:: async_session

.. autoclass:: async_sessionmaker
   :members:
   :inherited-members:

.. autoclass:: async_scoped_session
   :members:
   :inherited-members:

.. autoclass:: AsyncAttrs
   :members:

.. autoclass:: AsyncSession
   :members:
   :exclude-members: sync_session_class

   .. autoattribute:: sync_session_class

.. autoclass:: AsyncSessionTransaction
   :members: