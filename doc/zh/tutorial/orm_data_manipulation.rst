.. |prev| replace:: :doc:`data`
.. |next| replace:: :doc:`orm_related_objects`

.. include:: tutorial_nav_include.rst

.. rst-class:: orm-header

.. _tutorial_orm_data_manipulation:

使用ORM进行数据操作
==============================

前面的章节 :ref:`tutorial_working_with_data` 集中讲解了Core的SQL Expression语言，以提供横跨主要SQL语句结构的连续性。本节将构建 :class:`_orm.Session` 的生命周期以及它如何与这些构造交互。

**先决条件章节** - ORM集中的教程部分构建在本文档中的两个以ORM为中心的部分之上。

* :ref:`tutorial_executing_orm_session` - 介绍了如何在ORM中创建 :class:`_orm.Session` 对象

* :ref:`tutorial_orm_table_metadata` - 在其中设置了 ``User`` 和 ``Address`` 实体的ORM映射

* :ref:`tutorial_selecting_orm_entities` - 演示了如何运行针对 ``User`` 等实体的SELECT语句的几个示例。

.. _tutorial_inserting_orm:

使用ORM Unit of Work模式插入行
-------------------------------------------------

当使用ORM时，:class:`_orm.Session` 对象负责构建 :class:`_sql.Insert` 并在进行中的事务中将它们作为INSERT语句发出。指示 :class:`_orm.Session` 如何执行此操作的方法是通过**添加**对象条目到其中来完成的；当需要时，:class:`_orm.Session` 会确保将这些新条目发射到数据库中，使用一种称为**flush**的过程。最终 :class:`_orm.Session` 用来持久化对象的过程被称为：term:`unit of work` 模式。

类的实例表示行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在上一个示例中，我们使用Python字典来表示要添加的数据，请注意，使用ORM时，我们直接使用了在 :ref:`tutorial_orm_table_metadata` 定义的自定义Python类。在类级别上， ``User`` 和 ``Address`` 类用作定义相应数据库表应如何查看的地方。

这些类也用作扩展数据对象，我们在其中创建和操纵事务中的行。下面，我们会创建两个 ``User`` 对象，每个对象表示一个可以插入的数据库行::

    >>> squidward = User(name="squidward", fullname="Squidward Tentacles")
    >>> krabs = User(name="ehkrabs", fullname="Eugene H. Krabs")

我们能够使用构造函数中映射的列名作为关键字参数构造这些对象。这是因为 ``User`` 类包含了ORM映射所提供的自动生成的 ``__init__()`` 构造函数，以便我们可以使用列名称作为构造函数中的键来创建每个对象。

与之前的Core示例一样，在这里我们没有包括一个主键 (即 ``id`` 列的条目)，因为我们想利用数据库（在本例中是SQLite）的自动增加主键功能，ORM同样集成了这个功能。如果我们查看上面的对象的 ``id`` 值，则显示为 ``None``：

    >>> squidward
    User(id=None, name='squidward', fullname='Squidward Tentacles')

``None`` 值由SQLAlchemy提供，表示属性尚未具有值。在处理新对象的情况下，SQLAlchemy映射的属性总是在Python中返回一个值，并且在找不到它们的情况下不会引发 ``AttributeError``。

目前，上面的两个对象被称为处于 :term:`transient` 状态 - 它们不与任何数据库状态相关联，尚未与可能生成针对它们的INSERT语句的 :class:`_orm.Session` 对象相关联。

向会话添加对象
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

为了逐步展示添加过程，我们将创建一个 :class:`_orm.Session`而不使用上下文管理器（因此我们必须稍后确保关闭它！）::

    >>> session = Session(engine)

然后使用 :meth:`_orm.Session.add` 方法将对象添加到 :class:`_orm.Session` 中。当调用这个方法时，对象处于 :term:`pending` 状态，尚未被插入::

    >>> session.add(squidward)
    >>> session.add(krabs)

当我们有挂起的对象时，我们可以通过查看 :class:`_orm.Session` 的名称为 :attr:`_orm.Session.new` 的集合来查看该状态::

    >>> session.new
    IdentitySet([User(id=None, name='squidward', fullname='Squidward Tentacles'), User(id=None, name='ehkrabs', fullname='Eugene H. Krabs')])

上面的视图使用称为 :class:`.IdentitySet` 的集合，它实际上是Python集合，在所有情况下都使用对象标识哈希化（即使用Python内置的 ``id()`` 函数，而不是Python ``hash()`` 函数）。

刷新
^^^^^^^^

:class:`_orm.Session` 使用称为 :term:`unit of work` 的模式。这通常意味着它一个一个积累更改，但仅在需要时才向数据库实际通信进行更改。这允许使基于给定挂起更改的SQL DML如何在事务中发出更好的决策。在实际发出用于将当前状态的更改推入数据库的SQL时，该进程称为 **flush**。

我们可以通过手动调用 :meth:`_orm.Session.flush`函数来说明刷新过程：

.. sourcecode:: pycon+sql

    >>> session.flush()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [... (insertmanyvalues) 1/2 (ordered; batch not supported)] ('squidward', 'Squidward Tentacles')
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [insertmanyvalues 2/2 (ordered; batch not supported)] ('ehkrabs', 'Eugene H. Krabs')

上面我们观察到:class:`_orm.Session`首先被调用以发出SQL，因此它创建了一个新事务并为两个对象发出了适当的INSERT语句。在对挂起对象的任何更改都被其他flush处理之前，该事务现在**保持打开状态**。 常规情况下可以忽略:meth:`_orm.Session.flush`，因为:class:`_orm.Session`具有称为 **autoflush** 的特性，我们稍后将加以说明。每当调用 :meth:`_orm.Session.commit` 时，它还会 刷新更改。


自动生成的主键属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一旦插入行，我们创建的两个Python对象现在处于称为 :term:`persistent` 的状态，它们与添加或加载它们的:class:`_orm.Session`对象相关联，并具有许多其他可用特性，稍后将介绍。

插入操作的另一个影响是ORM已检索每个新对象的主键标识符。内部通常使用与先前介绍的 :attr:`_engine.CursorResult.inserted_primary_key` 访问器相同的方法。 现在，“Squidward”和“Krabs”对象的主键标识符已与它们相关联，我们可以通过访问 ``id`` 属性来查看它们：

    >>> squidward.id
    4
    >>> krabs.id
    5

.. tip:: 为什么ORM发出了两个单独的INSERT语句，而不是使用 :ref:`executemany <tutorial_multiple_parameters>`? 正如我们将在接下来的一节中看到的，当刷新对象时，:class:`_orm.Session`始终需要知道新插入对象的主键。如果使用类似SQLite的自动增量功能（其他示例包括使用序列的PostgreSQL IDENTITY或SERIAL等），则 :attr:`_engine.CursorResult.inserted_primary_key` 功能通常要求一次插入一个插入语句。 如果我们事先提供了主键的值，则ORM将能够更好地优化操作。一些数据库后端，如 :ref:`psycopg2 `，也可以在同时插入多行的情况下插入多行数据并能够检索主键值。

通过主键从Identity Map获取对象
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:term:`identity map` 对于:class:`_orm.Session`是至关重要的特性，因为该对象开始将对象持续性地链接到它们的主键标识中，使用一个名为 :term:`identity map`的特性。在本地存在时，通过使用 :meth:`_orm.Session.get` 方法检索其中一个以上对象会返回对于自己来说特定于主键的条目，否则会发出SELECT:

    >>> some_squidward = session.get(User, 4)
    >>> some_squidward
    User(id=4, name='squidward', fullname='Squidward Tentacles')

关于恒等图的重要事项是，它在特定于:class:`_orm.Session`，只作用于内存中当前已加载的对象，将所有当前已加载的Python对象与其主键标识匹配。我们可以观察到， ``some_squidward`` 引用与先前 ``squidward`` 相同的对象。

    >>> some_squidward is squidward
    True

该恒等图是允许在没有让事情不同步时操作复杂对象集的必要条件。

提交
^^^^^^^^^^^

本节讲解的是 :class:`_orm.Session` 的工作原理的更多内容。 我们现在将提交事务，以便在我们检查ORM的其余行为和功能之前建立对如何选择行的知识:

.. sourcecode:: pycon+sql

    >>> session.commit()
    COMMIT

我们处理的对象仍然处于 :class:`.Session` 中的状态处于 :term:`attached` 状态，直到 :class:`.Session` 被关闭（在 :ref:`tutorial_orm_closing` 中引入）。

.. tip::

  重要的一点是，我们刚刚处理过的对象的属性已经 :term:`expired`，这意味着当我们下一次访问它们的任何属性时， :class:`.Session` 会开始一个新的事务并重新加载它们的状态。这个选项有时会因性能原因或者如果我们想在关闭 :class:`.Session` 后使用对象（这被称为 :term:`detached` 状态），而导致问题，因为他们将没有任何状态，也没有 :class:`.Session`（-引发“detached instance”错误）。该行为可以使用称为 :paramref:`.Session.expire_on_commit` 的参数进行控制。详见 :ref:`tutorial_orm_closing`。

.. _tutorial_orm_updating:

使用Unit of Work模式更新ORM对象
----------------------------------------------------

在 :ref:`tutorial_core_update_delete` 中，我们介绍了 :class:`_sql.Update` 含义，它表示SQL UPDATE语句。当使用ORM时，有两种方式可以使用该构造。主要的途径是ORM使用 :term:`unit of work` 过程自动发出 UPDATE 语句，该过程对应于具有更改的单个对象的每个主键，而这些更改可以进行。

假设我们将 ``User`` 对象加载为用户名 ``sandy``，以下是 ：meth:`_sql.Select.filter_by` 方法的使用示例，还演示了 :meth:`_engine.Result.scalar_one` 方法::

.. sourcecode:: pycon+sql

    >>> sandy = session.execute(select(User).filter_by(name="sandy")).scalar_one()
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('sandy',)

Python对象 ``sandy`` 如前所述用作代表数据库中的一行，更具体地说是 **关于当前事务的数据库行**，该事务具有主键标识符为 ``2``：

    >>> sandy
    User(id=2, name='sandy', fullname='Sandy Cheeks')

如果我们更改此对象的属性，则 :class:`_orm.Session` 会跟踪此更改：

    >>> sandy.fullname = "Sandy Squirrel"

该对象出现在名为 ` :attr:`_orm.Session.dirty` 的集合中，表示该对象处于“脏状态”：

    >>> sandy in session.dirty
    True

:class:`_orm.Session` 在下一次刷新时会将其更改提交到数据库中。如前面所述，在还没有刷新其他挂起的更改之前，任何刷新都会导致挂起对象的更改。

我们可以直接查询此行中的 ``User.fullname`` 列，并且将会看到已经将更新值返回：

.. sourcecode:: pycon+sql

    >>> sandy_fullname = session.execute(select(User.fullname).where(User.id == 2)).scalar_one()
    {execsql}UPDATE user_account SET fullname=? WHERE user_account.id = ?
    [...] ('Sandy Squirrel', 2)
    SELECT user_account.fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (2,){stop}
    >>> print(sandy_fullname)
    Sandy Squirrel

我们可以看到上面的查询语句中会进行 UPDATE 和查询操作，这是因为刷新过程会推出所有等待提交的更改。 现在``sandy`` Python对象不再被认为是脏的：

    >>> sandy in session.dirty
    False

请注意，我们仍然在一个事务中，并且我们的更改尚未被推到数据库的永久存储。因为Sandy的姓氏实际上是“ Cheeks”而不是“ Squirrel”，我们将在回滚事务时稍后更正此错误。但是首先，我们将进行一些其他数据更改操作。


.. seealso::

    :ref:`session_flushing`-更详细地介绍了刷新过程以及 :paramref:`_orm.Session.autoflush` 设置的信息。

.. _tutorial_orm_deleting:


使用Unit of Work模式删除ORM对象
----------------------------------------------------

为了完成基本的持久化操作，可以使用 :class:`_orm.Session` 全局删除单个ORM对象，这个过程也是通过 :term:`unit of work` 完成的，使用 :meth:`_orm.Session.delete` 方法，比如从数据库中加载 “patrick”：

.. sourcecode:: pycon+sql

    >>> patrick = session.get(User, 3)
    {execsql}SELECT user_account.id AS user_account_id, user_account.name AS user_account_name,
    user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (3,)

如果我们标记 ``patrick`` 进行删除（如其他操作一样，除非刷新，否则现在什么也不会发生）：

    >>> session.delete(patrick)

当前的ORM行为是，``patrick`` 在 :class:`_orm.Session` 中保持不变，直到刷新为止，这意味着它保持在注册表中。在再次发出查询之前，没有删除行为实际发出：

.. sourcecode:: pycon+sql

    >>> session.execute(select(User).where(User.name == "patrick")).first()
    {execsql}SELECT address.id AS address_id, address.email_address AS address_email_address,
    address.user_id AS address_user_id
    FROM address
    WHERE ? = address.user_id
    [...] (3,)
    DELETE FROM user_account WHERE user_account.id = ?
    [...] (3,)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('patrick',)

上面的查询中我们通过WHERE条件所对应的ID，获得了一条关于``patrick``的反向外键数据，ORM试图查找与目标行相关的其他表中的行的数据，这种行为被称为级联，。关于ORM如何操作与级联相关的行，有非常详细的文档说明，参见 :ref:`cascade_delete`。

.. seealso::

    :ref:`cascade_delete` -描述如何在大量行匹配的情况下微调 :meth:`_orm.Session.delete` 的行为，以处理其他表中的值。

除此之外，正在删除的 ``patrick`` 对象实例现在不再被认为是持久的 :class:`_orm.Session`，由下面的包含检查来表示：

    >>> patrick in session
    False

但是请注意，我们仍然仍处于一个事务中，而且我们所做的所有更改都只是局限于当前事务中，如果我们不提交事务，则不会变为永久存储。现在我们将进行回滚事务，这更有趣。


.. _tutorial_orm_bulk:

批量/多行INSERT、upsert、UPDATE和DELETE
---------------------------------------------------

本章讨论的ORM :class:`_orm.Session`的 :term:`unit of work` 技术是旨在将 :term:`dml` 或INSERT（/ UPDATE / DELETE）语句与Python对象机制集成在一起的，通常涉及到互相关联对象的复杂图。当使用这些对象使用class:`_orm.Session.add`集成时，处理的单个ORM对象的更改会自动地执行INSERT / UPDATE / DELETE。然而，ORM :class:`_orm.Session` 还可以在没有传递ORM持久化对象的情况下进行处理，而是通过传递值列表来进行INSERT, UPDATE或删除，或者传递WHERE条件，以使UPDATE或DELETE语句能够匹配多行。这种使用方式的特点是，大量行必须在不需要构造和操作映射的对象的情况下受到影响，这对于简单、性能密集的任务（例如大量批量插入）可能会很麻烦并且不必要。

如果想获取更多的背景与用例，请参见 :ref:`orm_expression_update_delete`。

.. seealso::

    :ref:`orm_expression_update_delete`-在 :ref:`queryguide_toplevel`,中介绍了如何使用这些功能的操作示例。


回滚
-------------

:class:`_orm.Session` 具有 :meth:`_orm.Session.rollback` 方法，按预期在进行中的SQL连接上发出一个ROLLBACK。但是，它还会影响当前与 :class:`_orm.Session` 关联的对象，如前面的Python对象``sandy``。虽然我们已将``sandy``的姓氏更改为``"Sandy Squirrel"``，但我们想回滚此更改。调用 :meth:`_orm.Session.rollback` 不仅会回滚事务，而且还会使 :class:`_orm.Session` 当前关联的所有对象 **过期**，这将导致它们在下一次使用 :term:`lazy loading` 来自动开始一个 **新事务**来刷新，默认情况下也是如此：

.. sourcecode:: pycon+sql

    >>> session.rollback()
    ROLLBACK

为了更仔细地查看“expiration”处理，我们可以观察Python对象``sandy``在其Python ``__dict__``中没有任何状态，除了一个特殊的SQLAlchemy内部状态对象：

    >>> sandy.__dict__
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>}

这是 ":term:`expired`" 状态；再次访问属性时，会自动开始一个新事务并将 ``sandy`` 刷新为当前数据库行：

.. sourcecode:: pycon+sql

    >>> sandy.fullname
    {execsql}BEGIN (implicit)
    SELECT user_account.id AS user_account_id, user_account.name AS user_account_name,
    user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (2,){stop}
    'Sandy Cheeks'

我们现在可以观察到 Python 对象 ``sandy`` 的``__dict__``中已经包含数据库行：

    >>> sandy.__dict__  # doctest: +SKIP
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>,
     'id': 2, 'name': 'sandy', 'fullname': 'Sandy Cheeks'}

对于删除对象，当我们前面提到 ``patrick`` 不再处于会话中时，该对象的身份也得到了恢复：

    >>> patrick in session
    True

当然，数据库数据也再次出现：

.. sourcecode:: pycon+sql

    >>> session.execute(select(User).where(User.name == "patrick")).scalar_one() is patrick
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('patrick',){stop}
    True

.. _tutorial_orm_closing:

关闭会话
------------------

在上面的各个部分中，我们在Python上下文管理器之外使用了 :class:`_orm.Session` 对象，也就是说，我们没有使用 ``with`` 语句。没有问题，但是如果我们以这种方式进行操作，最好在完成所有操作后明确关闭 :class:`_orm.Session` ：

.. sourcecode:: pycon+sql

    >>> session.close()
    {execsql}ROLLBACK

关闭 :class:`_orm.Session` 需要完成以下几件事情：

* 它释放连接资源到连接池，取消任何正在进行的事务。

  这意味着当我们使用会话执行一些只读任务然后关闭它时，我们不需要显式调用 :meth:`_orm.Session.rollback` 以确保事务被回滚；连接池处理此事。

* 它**取消了** :class:`_orm.Session` 中的所有对象。

  这意味着我们在此 :class:`_orm.Session` 中加载的所有Python对象，例如 ``sandy``， ``patrick`` 和 ``squidward``，现在都处于称为 :term:`detached` 的状态。 特别地，我们将注意到，处于 :term:`expried` 状态的对象，例如由调用 :meth:`_orm.Session.commit` 导致的，现在已经失效，因为它们不包含当前行的状态，也不再与任何数据库事务关联。

  可以使用 :meth:`_orm.Session.add` 方法重新关联具有相同数据库行的已分隔对象与同一 :class:`_orm.Session`，这会重新建立它们与它们各自的数据库行的关系：

  .. sourcecode:: pycon+sql

      >>> session.add(squidward)
      >>> squidward.name
      {execsql}BEGIN (implicit)
      SELECT user_account.id AS user_account_id, user_account.name AS user_account_name, user_account.fullname AS user_account_fullname
      FROM user_account
      WHERE user_account.id = ?
      [...] (4,){stop}
      'squidward'

  ..

  .. tip::

      尽量避免在分离状态下使用对象。如果必要，请使用 :paramref:`_orm.Session.expire_on_commit` 标志选择立即在Web应用程序中显示刚提交的对象与 :class:`_orm.Session` 关闭之前的情况，将它们设置为“False”。