.. _asyncio_toplevel:

异步 I/O (asyncio)
==========================

支持Python asyncio。包括Core和ORM使用，使用asyncio兼容的方言。

.. versionadded:: 1.4

.. warning:: 请阅读   :ref:`asyncio_install`  获取关于安装平台的重要提示，包括**Apple M1 Architecture**。

.. seealso::

      :ref:`change_3414`  - 初始特性公告

      :ref:`examples_asyncio`  - 例子脚本展示了Core和ORM在asyncio扩展中的使用。

.. _asyncio_install:

异步io平台安装说明(包括Apple M1)
---------------------------------------------------------

asyncio扩展仅适用于Python 3。它还依赖于 `greenlet <https://pypi.org/project/greenlet/>`_ 库。
这个依赖已经在常见的机器平台，包括以下机型默认安装。

.. sourcecode:: text

    x86_64 aarch64 ppc64le amd64 win32

对于上述平台，“greenlet”已知可以提供预构建的wheel文件。
对于其他平台，“greenlet”**默认不会安装**；
目前greenlet的文件列表可以在`Greenlet -
下载文件<https://pypi.org/project/greenlet/#files>`_中看到。
请注意，**有许多架构被省略了，包括Apple M1**。

为了在确保`greenlet`依赖存在时安装SQLAlchemy，无论使用什么平台,
可以如下步骤安装 ``[asyncio]`` `setuptools 额外选项 <https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras>`_,
这样会将 ``greenlet`` 也加入安装。:

.. sourcecode:: text

  pip install sqlalchemy[asyncio]

在没有预先构建wheel文件的平台上安装``greenlet``，意味着需要从源代码构建``greenlet``，
这需要使用Python的开发库。

概要 - Core
---------------

对于Core使用，  :func:`_asyncio.create_async_engine`  函数可以创建一个  :class:` _asyncio.AsyncEngine`实例，
该实例通过其 :meth:`_asyncio.AsyncEngine.connect` 和 :meth:`_asyncio.AsyncEngine.begin` 方法提供了传统
  :class:`_engine.Engine`  API的异步版本。   :class:` _asyncio.AsyncEngine`  通过提供   :class:`_asyncio.AsyncConnection` 
来交付异步上下文管理器。可以使用  :meth:`_asyncio.AsyncConnection.execute`  方法通过缓冲   :class:` _engine.Result` 
执行语句，或者使用  :meth:`_asyncio.AsyncConnection.stream`  方法通过流服务器端   :class:` _asyncio.AsyncResult` 
执行语句::

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

        # 为函数范围中创建的async引擎关闭和清理持久连接
        await engine.dispose()


    asyncio.run(async_main())

上述方法中，  :meth:`_asyncio.AsyncConnection.run_sync`   方法可用于调用特殊的 DDL 函数，
例如  :meth:`_schema.MetaData.create_all` ，该函数不包含可等待挂钩。

.. tip:: 推荐使用 ``await`` 调用  :meth:`_asyncio.AsyncEngine.dispose`  方法，
   如果在一个上下文中使用   :class:`_asyncio.AsyncEngine`  对象并且会被垃圾回收，在上面的示例中的
   ``async_main`` 函数中，就像这样确保连接池持有的所有连接都在可等待的上下文中正确处理。与使用阻止IO不同，
   SQLAlchemy 不能在 ``__del__`` 或 weakref终结器中正确处理这些连接，因为没有机会调用 ``await``。
   如果在范围内未明确处置引擎，则可能会导致发出类似于形式的警告标准输出
   ``RuntimeError: Event loop is closed`` 中的垃圾回收。

  :class:`_asyncio.AsyncConnection`  还通过  :meth:` _asyncio.AsyncConnection.stream`   方法提供了一个 "streaming" API，
返回一个   :class:`_asyncio.AsyncResult`  对象。该结果对象使用服务器端光标，并提供了一个async \ await API，例如一个async迭代器::

    async with engine.connect() as conn:
        async_result = await conn.stream(select(t1))

        async for row in async_result:
            print("row: %s" % (row,))

.. _asyncio_orm:


概要 - ORM
---------------使用  :term:`2.0 风格`  类提供了完整的ORM功能。

在默认使用模式下，必须特别小心，以避免涉及ORM关系和列属性的  :term:`延迟加载`  或其他过期属性访问；下一节  :ref:` asyncio_orm_avoid_lazyloads`详细介绍了这一点。

.. warning::

   一个   :class:`_asyncio.AsyncSession`  实例 **不能安全地用于多个并发任务**。请参阅有关   :ref:` asyncio_concurrency`  和   :ref:`session_faq_threadsafe`  的部分内容。

下面的示例说明了包括映射器和会话配置的完整示例::

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

在上面的示例中，使用可选的   :class:`_asyncio.async_sessionmaker`  助手实例化了   :class:` _asyncio.AsyncSession` ，该助手提供了一个固定参数集的新   :class:`_asyncio.AsyncSession`  对象工厂，这里包括将其与具有特定数据库URL的   :class:` _asyncio.AsyncEngine`  关联。然后将其传递给其他方法，其中它可以在 Python 异步上下文管理器（即 ``async with:`` 语句）中使用，以便在块结束时自动关闭；这相当于调用  :meth:`_asyncio.AsyncSession.close`  方法。

.. _asyncio_concurrency:

使用异步会话处理并发任务
~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`_asyncio.AsyncSession`  对象是一个**可变的、有状态的对象**，它表示正在进行的**单个有状态的数据库事务**。使用 asyncio 中的并发任务，例如使用 ` `asyncio.gather()`` 等 API，应**对每个单独的任务使用单独的**   :class:`_asyncio.AsyncSession` 。

有关   :class:`_orm.Session`  和   :class:` _asyncio.AsyncSession`  的一般描述，以及它们在处理并发工作负载时应如何使用的信息，请参见   :ref:`session_faq_threadsafe`  部分。

.. _asyncio_orm_avoid_lazyloads:

使用 AsyncSession 时避免隐式 IO 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用传统的 asyncio，应用程序需要避免任何可能发生 IO 的属性访问点。可以使用以下技术来帮助解决这个问题，其中许多方法在上面的示例中有展示。

* 延迟加载关系、延迟的列或表达式，或正在过期场景下访问的属性可以利用   :class:`_asyncio.AsyncAttrs`  mixin。当添加到特定类或更普遍地添加到 Declarative ` `Base`` 超类时，此 mixin 提供访问器  :attr:`_asyncio.AsyncAttrs.awaitable_attrs` ，可以将任何属性交付为等待的形式：.. _async_orm_collections:

异步ORM的集合
-----------------------

在异步程序中使用ORM集合存在的关键问题是访问集合时有可能想要访问未被提前加载的集合，此时将会使用惰性加载来访问集合。然而使用惰性加载需要隐式的IO操作，这在asyncio下会失败。为了直接访问该异步属性，你可以通过指定  :attr:`_asyncio.AsyncAttrs.awaitable_attrs`  前缀将属性访问作为一个 awaitable 进行访问。

Add your translations here, for example:
class B(Base):
    __tablename__ = "b"

    # ... rest of mapping ...

访问 ``A.bs`` 集合时可以通过提前加载来避免使用惰性加载所需执行的IO操作。最有用的急切加载策略是   :func:`_orm.selectinload`  急切加载器，该急切加载器在前面的例子中使用，用于在 ` `await session.execute()`` 在的范围内急切地加载 ``A.bs`` 集合。

如果不使用   :class:`_asyncio.AsyncAttrs` ，可以使用` `lazy="raise"``声明关系，以便默认情况下它们将不尝试发出SQL操作。为了加载集合将使用急切加载而不是使用惰性加载。

在构建新对象时，**集合总是被分配一个默认的空集合**，例如在上面的示例中是一个列表，以使该 ``A`` 对象上的 ``.bs`` 集合在此对象被flush后可以存在和可读；否则，当 ``A`` 刷新时， ``.bs`` 将被卸载并在访问时引发错误。

  :class:`_asyncio.AsyncSession`  配置使用   :paramref:` _orm.Session.expire_on_commit`  设置为False，这样我们可以在调用  :meth:`_asyncio.AsyncSession.commit`  之后访问对象的属性，例如在末尾访问属性的代码行。

其他一些指导包括：

- 应该避免使用  :meth:`_asyncio.AsyncSession.expire`  这样的方法，而要使用  :meth:` _asyncio.AsyncSession.refresh` ；**如果**绝对需要过期刷新。
- 可以使用  :meth:`_asyncio.AsyncSession.refresh`  **显式地在asyncio下加载关系**，**如果**将期望的属性名称明确定义为  :paramref:` _orm.Session.refresh`  的参数  :paramref:`_orm.Session.refresh.attribute_names` 。

.. versionadded:: 2.0.13

.. seealso::

      :class:`_asyncio.AsyncAttrs` 

集合可以使用   :ref:`write_only_relationship`  功能被替换为永不发出隐式IO的**只写集合**。使用此功能，集合不会被读取，仅使用显式SQL调用查询。请参阅   :ref:` examples_asyncio`  部分中的 ``async_orm_writeonly.py`` 示例，这是使用asyncio的写集合的示例，其中许多项目解决使用异步加载集合的具体技术，需要更精细的处理。

注意，使用只写集合的程序行为是简单和易于预测的。但缺点是没有任何内置系统来加载这些集合中的许多集合，这需要手动完成。因此，下面的许多内容都针对使用异步懒惰加载关系进行处理时，需要更多注意事项的情况。* 避免使用   :ref:`unitofwork_cascades`  中记录的 ` `all`` 级联选项，而是显式地列出所需的级联特性。``all``级联选项暗示了   :ref:`cascade_refresh_expire`  等级联设置，这意味着  :meth:` .AsyncSession.refresh`  方法将会使相关对象的属性过期，但不一定刷新这些相关对象 ，假设   :func:`_orm.relationship`  中没有配置急切加载，它们将保持过期状态。

* 如果使用   :func:`_orm.deferred`  列，则应使用适当的加载器选项（如果使用），除了上述注意事项中的   :func:` _orm.relationship`  构造。有关延迟列加载的背景，请参阅   :ref:`orm_queryguide_column_deferral` 。

.. _dynamic_asyncio:

* 在   :ref:`dynamic_relationship`  中描述的“动态”关系加载程序策略默认不兼容 asyncio 方法。只有在  :meth:` _asyncio.AsyncSession.run_sync`  方法中调用或使用其 ``.statement`` 属性获取普通选择时，才能直接使用它：

      user = await session.get(User, 42)
      addresses = (await session.scalars(user.addresses.statement)).all()
      stmt = user.addresses.statement.where(Address.email_address.startswith("patrick"))
      addresses_filter = (await session.scalars(stmt)).all()

  在 SQLAlchemy 2.0 中，引入了   :ref:`write only <write_only_relationship>`  技术，这种技术与 asyncio 完全兼容，并且应该优先考虑使用。

  .. seealso::

      :ref:`migration_20_dynamic_loaders`  - 关于迁移到 2.0 样式的说明

* 如果在不支持 RETURNING 的数据库（例如 MySQL 8）上使用 asyncio，则新刷新的对象上的生成的时间戳等服务器默认值将不可用，除非使用  :paramref:`_orm.Mapper.eager_defaults`  选项。在 SQLAlchemy 2.0 中，这种行为已自动应用于像 PostgreSQL、SQLite 和 MariaDB 这样的后端，当插入行时使用 RETURNING 获取新值。

.. _session_run_sync:

在 asyncio 下运行同步方法和函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. deepalchemy:: 这种方法本质上是公开了 SQLAlchemy 首次提供 asyncio 接口的机制。虽然这样做没有技术上的问题，但整体而言，这种方法可能被认为是“有争议的”，因为它与 asyncio 编程模型的一些中心哲学相悖，即任何可能导致调用 I/O 的编程语句 **必须** 有一个 ``await`` 调用，以免程序不明确地在哪些行可能发生 I/O。这种方法不改变这个概念，但它允许一系列同步 I/O 指令在函数调用的范围内被豁免免于这个规则，从而被打包成一个可等待对象。

作为在 asyncio 事件循环中集成传统 SQLAlchemy“懒加载”的备选手段，提供了一个名为  :meth:`_asyncio.AsyncSession.run_sync`  的 **可选** 方法，该方法将在一个 greenlet 中运行任何 Python 函数，传统的同步编程概念将被翻译为在到达数据库驱动程序时使用 ` `await``。这里的一个假设方法是，面向 asyncio 的应用程序可以将数据库相关方法打包成函数，这些函数将使用  :meth:`_asyncio.AsyncSession.run_sync`  被调用。

如果我们没有对 ``A.bs`` 集合使用   :func:`_orm.selectinload` ，那么我们可以在一个单独的函数中实现对这些属性访问的处理方式，如下所示：

    import asyncio

    from sqlalchemy import select
    from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine


    def fetch_and_update_objects(session):
        """在将同步风格的 ORM 代码打包到将调用函数的可等待对象中运行。

        """

        # 这里的会话对象是传统的 ORM 会话。
        # 所有功能都可以在这里使用，包括旧版查询。

        stmt = select(A)

        result = session.execute(stmt)
        for a1 in result.scalars():
            print(a1)

            # 延迟加载
            for b1 in a1.bs:
                print(b1)

        # 旧版查询
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

        # 对于在函数作用域中创建的 AsyncEngine，关闭它并清理池化的连接
        await engine.dispose()


    asyncio.run(async_main())

在“sync”运行器中运行某些函数的上述方法与使用“gevent”这样的基于事件的编程库上的 SQLAlchemy 应用程序有一些相似之处。它们的区别如下：

1. 与使用“gevent”时不同，我们可以继续使用标准的 Python。使用asyncio扩展处理事件
--------------------------

SQLAlchemy  :ref:`事件系统 <event_toplevel>` 没有直接扩展到asyncio，意味着还没有 SQLalchemy 事件处理程序的“异步”版本。

但是，由于asyncio扩展围绕着通常的同步SQLAlchemy API，因此常规的“同步”风格的事件处理程序可以自由地使用，就像不使用asyncio一样。

如下所述，有两种当前的策略可以注册事件：

* 实例级别可以注册事件（例如，特定的   :class:`_asyncio.AsyncEngine`  实例） ，通过关联与直接代理对象引用有关` `sync``属性的事件。 例如，要将  :meth:`_events.PoolEvents.connect`  属性作为目标。目标包括：

       :attr:`_asyncio.AsyncEngine.sync_engine` 

       :attr:` _asyncio.AsyncConnection.sync_connection` 

       :attr:`_asyncio.AsyncConnection.sync_engine` 

       :attr:`_asyncio.AsyncSession.sync_session` 

* 要在类级别上注册事件，以针对同一类型的所有实例（例如所有   :class:`_asyncio.AsyncSession`  实例） ，请使用相应的同步级别类 。例如，要针对   :class:` _asyncio.AsyncSession`  类注册  :meth:`_ormevents.SessionEvents.before_commit`  事件，请使用   :class:` _orm.Session`  作为目标类。

* 要在   :class:`_orm.sessionmaker`  级别上注册，请结合一个显式的   :class:` _orm.sessionmaker`  和  :class:`_asyncio.async_sessionmaker`  ，使用  :paramref:` _asyncio.async_sessionmaker.sync_session_class`  ，将事件与   :class:`_orm.sessionmaker`  相关联。

在执行位于asyncio上下文中的事件处理程序时，像  :class:`_engine.Connection`  这样的对象会在不需要` `await``或``async``使用的通常“同步”方式下工作; 当最终由asyncio数据库适配器接收到消息时，就会透明地将调用样式适配回asyncio调用样式。 对于传递DBAPI级别的“连接”对象（如  :meth:`_events.PoolEvents.connect`  ）的事件，对象是一个符合  :term:` pep-249`  标准的“连接”对象，它会将同步样式调用适配到asyncio驱动程序。

使用AsyncEngine / Sessions / Sessionmakers的事件监听器示例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

下面是与async-facing API构造关联的同步样式事件处理程序的一些示例：

* **在AsyncEngine上处理核心事件**

  在此示例中，我们访问   :class:`_asyncio.AsyncEngine.sync_engine`  属性，将其作为  :class:` .ConnectionEvents`  和   :class:`.PoolEvents`  的目标：：

    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.engine import Engine
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")


    #实例Engine上的connect事件
    @event.listens_for(engine.sync_engine, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)
        cursor = dbapi_con.cursor()

        # 在适配的DBAPI连接/游标上使用同步样式API
        cursor.execute("select 'execute from event'")
        print(cursor.fetchone()[0])


    # 所有引擎实例的before_execute事件
    @event.listens_for(Engine, "before_execute")
    def my_before_execute(
        conn,
        clauseelement,
        multiparams,
        params,
        execution_options,
    ):
        print("before execute!")


    async def go():
        async with engine.connect() as conn:
            await conn.execute(text("select 1"))
        await engine.dispose()


    asyncio.run(go())

  输出：

  .. sourcecode:: text

    New DBAPI connection: <AdaptedConnection <asyncpg.connection.Connection object at 0x7f33f9b16960>>
    execute from event
    before execute!


* **在AsyncSession上处理ORM事件**

  在此示例中，我们访问  :attr:`_asyncio.AsyncSession.sync_session`  作为   :class:` _orm.SessionEvents`  的目标：：

    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.orm import Session

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    session = AsyncSession(engine)


    # 实例Session上的before_commit事件
    @event.listens_for(session.sync_session, "before_commit")
    def my_before_commit(session):
        print("before commit!")

        # 在Connection上使用同步样式API
        connection = session.connection()

        # 在游标上使用同步样式API
        result = connection.execute(text("select 'execute from event'"))异步操作中的SQLAlchemy Events
------------------------------------

在那些异步应用中需要事件处理的场景下，SQLAlchemy也提供了相应的方法来实现。不同于同步的事件处理，在异步操作中需要使用   :class:`_asyncio.async_sessionmaker`  和   :class:` _orm.sessionmaker`  结合使用来实现。

* **ORM Events on async_session**

  首先需要定义一个 ORM 会话类，在这个类上我们可以使用 SQLAlchemy 提供的事件装饰器实现相关的事件监听。

  代码示例：

  .. code-block:: python

    import asyncio

    from sqlalchemy import create_engine, event, text
    from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

    async def test_async_session_events():

        engine = create_async_engine('postgresql+asyncpg://user:pass@localhost/db')
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)

        async with engine.begin() as conn:
            await conn.execute("delete from mytable")

        async with AsyncSession(engine) as session:

            @event.listens_for(session, "before_commit")
            def before_commit(session):
                print("before commit!")

            @event.listens_for(session, "after_commit")
            def after_commit(session):
                print("after commit!")

            result = await session.execute(text("select 1"))
            print(result)
            print(result.first())

        async with AsyncSession(engine) as session:
            result = await session.execute(text("select 1"))
            print(result.first())

        async with AsyncSession(engine) as session:
            result = await session.execute(text("select 1"))
            print(result.first())

        async with engine.begin() as conn:
            await conn.execute("delete from mytable")

        async with AsyncSession(engine) as session:
            result = await session.execute(text("select 1"))
            print(result.first())


  返回结果：

  .. sourcecode:: text

    SELECT 1
    1
    1
    1

* **ORM Events on sessionmaker**

  在这种管理多个数据库操作会话的场景中，我们可以创建一个   :class:`_orm.sessionmaker`  作为我们的事件监听对象。这个会话工厂类可以被赋值给   :class:` _asyncio.async_sessionmaker`  ，在使用时需要注意设置  :paramref:`_asyncio.async_sessionmaker.sync_session_class`  参数。

  示例代码：

  .. code-block:: python

    import asyncio

    from sqlalchemy import event, text
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

  返回结果：

  .. sourcecode:: text

    before commit


.. _asyncio_events_run_async:

异步操作下使用仅接受 awaitable 的数据库驱动方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在上文提到的对于   :class:`.PoolEvents`  事件的处理方式，接收到的连接对象是一个以 sync-style "DBAPI" 连接对象包装的内部 asyncio "driver" 连接对象，该对象能够被 SQLAlchemy 的内部方法使用。当用户自定义事件处理器需要直接使用底层 "driver" 连接对象异步调用连接对象的方法时，就需要使用到 ` `.set_type_codec()`` 方法提供的 asyncpg 驱动程序。

为了解决这个问题，SQLAlchemy's   :class:`.AdaptedConnection`  类提供了一个  :meth:` .AdaptedConnection.run_async`  方法，该方法可以在事件处理器或其他 SQLAlchemy 内部执行异步函数。这个方法直接类比于  :meth:`_asyncio.AsyncConnection.run_sync`  方法，后者能够让同步方法在异步环境下执行。

在  :meth:`.AdaptedConnection.run_async`  方法中，我们需要传入一个接受 driver 连接对象的参数并返回一个 awaitable 对象的可调用函数。注意这个传入的函数不需要被声明为 ` `async``，可以使用 Python 的``lambda``声明一个返回 awaitable 对象的可调用函数：

  代码示例：

  .. code-block:: python

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine(...)


    @event.listens_for(engine.sync_engine, "connect")
    def register_custom_types(dbapi_connection, *args):
        dbapi_connection.run_async(
            lambda connection: connection.set_type_codec(
                "MyCustomType",
                encoder,
                decoder,
            )
        )

在 ``register_custom_types`` 事件处理器中传入的对象是   :class:`.AdaptedConnection`  实例，这个实例提供了一个异步数据库驱动程序连接对象的类似 DBAPI 的接口。然后  :meth:` .AdaptedConnection.run_async`  方法可以在事件处理器或其他 SQLAlchemy 内部执行异步方法。可等待环境下的SQLAlchemy Asyncio
-------------------------

SQLAlchemy Asyncio 支持可等待协程，它基于 asyncio 驱动层进行操作。

.. versionadded:: 1.4.30


使用多个 asyncio 事件循环
----------------------------------

不同的事件循环一般不共享 :class:`_asyncio.AsyncEngine` 当使用默认的连接池实现时。这在很少的情况下发生，例如在将 asyncio 与多线程结合使用时。

如果必须要传递   :class:`_asyncio.AsyncEngine`  给另一个事件循环，下面的方法必须被调用  :meth:` _asyncio.AsyncEngine.dispose()` ，因为在新事件循环开始之前必须在上一个事件循环中清理它。如果不这样做，会导致类似于“Task <Task Pending...>got Future attached to a different loop”的运行时错误。

如果必须将相同的引擎在不同的循环之间共享，可以使用   :class:`~sqlalchemy.pool.NullPool`  禁用池，防止引擎在使用的任何连接上使用多个了：

    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.pool import NullPool

    engine = create_async_engine(
        "postgresql+asyncpg://user:pass@host/dbname",
        poolclass=NullPool,
    )

.. _asyncio_scoped_session:

使用 asyncio scoped session
----------------------------

在使用   :class:`.scoped_session`  对象的线程化 SQLAlchemy 中使用的 “scoped session” 模式也可以在 asyncio 中使用，只需使用一个经过适配的版本   :class:` _asyncio.async_scoped_session` 。

.. tip:: 在一般情况下，SQLAlchemy 不建议新开发项目中使用“scoped”模式，因为它依赖于必须在大多数情况下显式地关闭其变化的可变全局状态。特别是在使用 asyncio 时，把   :class:`_asyncio.AsyncSession`  直接传递给需要它的可等待函数可能会更好。

在使用   :class:`_asyncio.async_scoped_session`  时，由于 asyncio 上下文中没有 “thread-local” 的概念，必须在构造函数中提供“scopefunc”参数。下面的示例展示了使用 ` `asyncio.current_task()`` 函数来实现这个作用：

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

.. warning::   :class:`_asyncio.async_scoped_session`  使用的“scopefunc”在任务中被调用了任意次数（每次访问基础的   :class:` _asyncio.AsyncSession` ），必须作为幂等函数和小型函数来使用，不应尝试创建或修改任何状态（例如创建回调等）。

.. warning:: 在范围中使用 ``current_task()`` 时，需要从 outermost 可等待项中调用  :meth:`_asyncio.async_scoped_session.remove`  方法，以确保键在任务完成时从注册表中删除，否则任务句柄以及   :class:` _asyncio.AsyncSession`  将留在内存中，从而形成存储器泄漏。请参阅下面的示例，其中演示了  :meth:`_asyncio.async_scoped_session.remove`  的正确使用方法。

  :class:`_asyncio.async_scoped_session`  包括类似于   :class:` .scoped_session`  的**代理行为**，这意味着它可以直接被视为   :class:`_asyncio.AsyncSession` ，需要包括通常的 ` `await`` 关键字，包括  :meth:`_asyncio.async_scoped_session.remove`  方法：

    async def some_function(some_async_session, some_object):
        # 直接使用 AsyncSession
        some_async_session.add(some_object)

        # 通过上下文本地代理使用 AsyncSession
        await AsyncScopedSession.commit()

        # "remove" 当前被代理 AsyncSession，使其脱离本地上下文
        await AsyncScopedSession.remove()

.. versionadded:: 1.4.19

.. currentmodule:: sqlalchemy.ext.asyncio


.. _asyncio_inspector:

使用 inspector 检查架构对象
---------------------------------------------------

SQLAlchemy 还没有支持 asyncio 版本的   :class:`_reflection.Inspector` （在   :ref:` metadata_reflection_inspector`  中介绍），但是在 asyncio 上下文中可以通过使用   :class:`_asyncio.AsyncConnection.run_sync`  方法的现有接口来使用它。

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
----------------------------------  :class:`_asyncio.AsyncResult`  对象是   :class:` _result.Result`  对象的异步版本。使用
  :meth:`_asyncio.AsyncConnection.stream`   或  :meth:` _asyncio.AsyncSession.stream`  方法时，才会返回此对象，该方法返回基于活动数据库游标的结果对象。

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

ORM 会话 API 文档
---------------------

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