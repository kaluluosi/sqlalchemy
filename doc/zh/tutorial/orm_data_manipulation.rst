.. |prev| replace::   :doc:`data`  
.. |next| replace::   :doc:`orm_related_objects`  

.. include:: tutorial_nav_include.rst

.. rst-class:: orm-header

.. _tutorial_orm_data_manipulation:

使用ORM操作数据
==============================

前一节    :ref:`tutorial_working_with_data`   集中介绍了核心层的SQL表达式语言, 以便提供跨重要SQL语句构造的连续性。本节将构建ORM的生命周期，并介绍它如何与这些构造交互。

**前提章节** - ORM中心的章节建立在本文档的两个ORM中心章节的基础上:

*    :ref:`tutorial_executing_orm_session`   - 介绍如何创建ORM    :class:` _orm.Session`   对象

*    :ref:`tutorial_orm_table_metadata`   - 在这里我们设置了ORM映射的“User”和“Address”实体

*    :ref:`tutorial_selecting_orm_entities`   - 介绍如何对命名为 “User” 的实体运行 SELECT 语句的几个示例

.. _tutorial_inserting_orm:

使用ORM Unit of Work模式插入行
----------------------------------

当使用ORM时，  :class:`_orm.Session`  对象负责构造   :class:` _sql.Insert`   构造并在ongoing transaction中发射它作为INSERT语句。我们告诉   :class:`_orm.Session`   这样做的方式是通过**添加**对象条目到其中;    :class:` _orm.Session`   确保这些新条目在需要时被发射到数据库中，使用称为**flush**的进程将其保存到数据库中。    :class:`_orm.Session`   使用的用于持久化对象的整个过程称为   :term:` unit of work`    模式。

类的实例表示行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在前面的示例中，我们使用Python字典发射INSERT语句来指示要添加的数据，ORM中直接使用自定义Python类，回到    :ref:`tutorial_orm_table_metadata`  。在类级别，"User" 和 "Address" 类用作定义相应数据表应该是什么样子的地方。这些类还用作可扩展数据对象，我们使用它们在事务处理中创建和操作行。下面，我们将创建两个 "User" 对象，每个User对象表示可能的要插入的数据库行：：

    >>> squidward = User(name="squidward", fullname="Squidward Tentacles")
    >>> krabs = User(name="ehkrabs", fullname="Eugene H. Krabs")

我们能够使用映射的列名作为构造函数中的关键字参数构造这些对象。这是可能的，因为 “User”类包括了由ORM映射提供的自动生成的“__init __（）”构造函数，以便我们可以使用构造函数中的列名称作为键来创建每个对象。 

与我们在   :class:`_sql.Insert`  的常数示例中所做的类似，我们没有包括主键（假设未包括“id”列的条目），因为我们想使用数据库的自动递增主键功能，SQLite在这种情况下，ORM也与之集成。上面这些对象的"id"属性的值，如果我们查看它，会显示为"None"：：

    >>> squidward
    User(id=None, name='squidward', fullname='Squidward Tentacles')

"None"值是由SQLAlchemy提供的，表明该属性目前没有值。SQLAlchemy映射的属性总是返回Python值，如果在处理尚未被赋值的新对象时它们缺失，则不会引发“AttributeError”异常。

目前，上面的两个对象处于称为   :term:`transient`   的状态 - 它们未与任何数据库状态关联，并且尚未与 :  :class:` _orm.Session`  对象相关联，该对象可以为它们生成INSERT语句。

向会话中添加对象
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

为了一步步说明添加过程，我们将创建一个   :class:`_orm.Session` 对象，而不使用上下文管理器（因此我们必须稍后关闭它！）：：

    >>> session = Session(engine)

然后，使用   :meth:`_orm.Session.add`   方法将对象添加到    :class:` _orm.Session`   状态，尚未插入：：

    >>> session.add(squidward)
    >>> session.add(krabs)

当我们有未决的对象时，我们可以查看   :class:`_orm.Session`   ：：

  >>> session.new
  IdentitySet([User(id=None, name='squidward', fullname='Squidward Tentacles'), User(id=None, name='ehkrabs', fullname='Eugene H. Krabs')])

上述视图使用称为:class`.IdentitySet`的集合本质上，是一个使用Python内置的``id（）``函数在所有情况下（即使用Python ``hash（）``函数）以对象标识哈希的Python集合。

刷新
^^^^^^^^

  :class:`_orm.Session` 使用一种称为：term：` 工作单元`的模式。
这通常意味着它一次累计一个更改，但实际上不会将它们实际发送到数据库，直到需要。
这使它能够根据给定的挂起更改集合在事务中发出SQL DML时做出更好的决策。
当它向数据库发出SQL以推出当前更改集时，该过程称为**flush**。

我们可以通过调用   :meth:`_orm.Session.flush`   方法手动说明刷新过程：

.. sourcecode:: pycon+sql

    >>> session.flush()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [... (insertmanyvalues) 1/2 (ordered; batch not supported)] ('squidward', 'Squidward Tentacles')
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [insertmanyvalues 2/2 (ordered; batch not supported)] ('ehkrabs', 'Eugene H. Krabs')

在上面的例子中，我们观察到首先调用    :class:`_orm.Session`   发出SQL，所以它创建了一个新的事务并发出了适当的INSERT语句
两个对象。直到我们调用   :meth:`_orm.Session.commit`  ,   :meth:` _orm.Session.rollback`  , or
   :meth:`_orm.Session.close`   方法关闭该事务，该事务现在**仍处于打开状态**。

虽然   :meth:`_orm.Session.flush`   可以用于手动推出待处理的更改到当前事务，但通常是不必要的，
因为    :class:`_orm.Session`   具有称为**autoflush**的行为，我们稍后将对其进行说明。
每当调用   :meth:`_orm.Session.commit`    时，它也会推出更改。


自动生成的主键属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一旦插入了行，我们创建的两个Python对象处于一种称为：term:`persistent`状态，其中它们与添加或加载它们的  :class:`_orm.Session` 对象相关联，并且具有许多后面将介绍的其他行为。

插入发生的另一个影响是ORM已检索了每个新对象的新主键标识符。“内部”通常使用相同的  :attr:`_engine.CursorResult.inserted_primary_key` 访问器。
现在，“Squidward”和“Krabs”对象已将这些新的主键标识符与它们相关联，我们可以通过访问“id”属性来查看它们:

    >>> squidward.id
    4
    >>> krabs.id
    5

.. 提示:: ORM在只使用一个INSERT语句时为什么发出两个单独的INSERT语句？如我们将在下面的部分中看到的那样，
  :class:`_orm.Session` 当刷新对象时总是需要知道新插入对象的主键。
如果使用autoincrement之类的功能（其他示例包括PostgreSQL IDENTITY或SERIAL，使用序列等），
则  :attr:`_engine.CursorResult.inserted_primary_key` 特性通常要求每个INSERT以一行的形式发出。
如果我们预先为主键提供了值，则ORM将能够更好地优化该操作。某些数据库后端，例如   :ref:`psycopg2 <postgresql_psycopg2>`  ，也可以同时INSERT多行，同时仍能检索主键值。

通过主键从Identity Map获取对象
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对象的主键标识对于    :class:`_orm.Session`   非常重要，因为对象现在使用称为：term:` identity map`的功能与此标识符在内存中链接起来将其关联。
identity map 是一个内存存储器，将所有当前加载到内存中的对象与它们的主键标识链接起来。我们可以通过使用   :meth:`_orm.Session.get`   方法检索上述对象之一，并在本地存在时从identity map返回条目，否则会发出SELECT:

    >>> some_squidward = session.get(User, 4)
    >>> some_squidward
    User(id=4, name='squidward', fullname='Squidward Tentacles')

关于identity map 的重要事项是，它在一个特定的  :class:`_orm.Session` 对象的范围内维护特定Python对象的唯一实例，对应于特定的数据库标识。
我们可以观察到，``some_squidward`` 引用前面的 ``squidward`` 同一个对象::

    >>> some_squidward is squidward
    True

identity map是一个关键功能，允许在事务中操作复杂的对象集而不会发生不同步的情况。


提交
^^^^^^^^^^^

  :class:`_orm.Session` 还有很多要说的内容，我们将进一步讨论。现在我们将提交事务，以便在查看更多ORM行为和特征之前可以建立有关如何SELECT行的知识:.. sourcecode :: pycon+sql

    >>> session.commit()
    COMMIT

上述操作将提交正在进行的事务。我们处理的对象仍会被附加在  :class:`.Session` 上，
它们的状态将保持不变，直到关闭    :class:`.Session`  
（在    :ref:`tutorial_orm_closing`   中介绍）。

.. tip::

  重要的一点是我们刚才处理的对象上的属性已经过期，即下次访问它们的任何属性时，
     :class:`.Session`   会启动一个新的事务并重新加载它们的状态。这个选项有时对性能
  有问题，或者如果希望在关闭    :class:`.Session`   后使用对象（即处于   :term:` detached`   状态），
  它们将没有任何状态，并且没有    :class:`.Session`   与其一起加载该状态，从而导致“detached instance”错误。
  可以使用名为   :paramref:`.Session.expire_on_commit`   的参数来控制行为。更多信息请查看    :ref:` tutorial_orm_closing`  。

.. _tutorial_orm_updating:

使用工作单元模式更新ORM对象
----------------------------------------------------

在前面的章节    :ref:`tutorial_core_update_delete`   中，我们介绍了   :class:` _sql.Update`   构造，它表示SQL UPDATE语句。
当使用ORM时，这个构造有两种用法。 主要的方式是它自动作为    :class:`_orm.Session`   使用的
   :term:`unit of work`    过程的一部分发出，其中对具有更改的单个对象基于每个主键基础发出一个UPDATE语句。

假设我们将用户名为 ``sandy`` 的 ``User`` 对象加载到事务中（还展示了   :meth:`_sql.Select.filter_by`   方法
以及   :meth:`_engine.Result.scalar_one`   方法）：

.. sourcecode:: pycon+sql

    >>> sandy = session.execute(select(User).filter_by(name="sandy")).scalar_one()
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('sandy',)

如前所述，Python对象 ``sandy`` 充当了该数据库行的**代理**，
更具体地说，该数据库行对于当前事务来说，在具有主键标识 ``2`` 的情况下：

    >>> sandy
    User(id=2, name='sandy', fullname='Sandy Cheeks')

如果更改此对象的属性，则   :class:`_orm.Session`   将跟踪此更改：

    >>> sandy.fullname = "Sandy Squirrel"

表示对象出现在集合   :attr:`_orm.Session.dirty`   中，表示对象已经是“dirty”：

    >>> sandy in session.dirty
    True

当    :class:`_orm.Session`   次次发出flush时，将发出UPDATE以将此值更新到数据库中。
如前所述，我们在emit any SELECT之前自动刷新，
使用称为**autoflush** 的行为。我们可以直接查询此行的 ``User.fullname`` 列，
我们会得到更新后的值：

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

我们可以看到上面有一个：func：`_sql.select` 语句。但是，显示的发出SQL表明还发出了UPDATE，这是刷新进程在推出挂起的更改时发出的。
Python对象 ``sandy`` 现在不再被认为是“dirty”对象：

    >>> sandy in session.dirty
    False

但是请注意我们仍处于一个事务中，我们的更改尚未推送到数据库的永久存储。
由于Sandy的姓实际上是“Cheeks”而不是“Squirrel”，因此稍后当我们回滚事务时我们会纠正这个错误。
但首先，我们将进行更多数据更改。

.. seealso::

       :ref:`session_flushing`   - 详细介绍了flush过程以及有关   :paramref:` _orm.Session.autoflush`   设置的信息。

.. _tutorial_orm_deleting:

使用工作单元模式删除ORM对象
----------------------------------------------------

为了完成基本的持久化操作，可以使用   :meth:`_orm.Session.delete`   方法将单个ORM对象标记为
在   :term:`unit of work`   过程中删除。 我们让从数据库中加载` `patrick``：

.. sourcecode:: pycon+sql    >>> patrick = session.get(User, 3)
    {execsql}SELECT user_account.id AS user_account_id, user_account.name AS user_account_name,
    user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (3,)

如果我们像其他操作那样将``patrick``标记为删除，还没有什么事情发生，直到刷新为止:

    >>> session.delete(patrick)

当前 ORM 的行为是 ``patrick`` 仍然在    :class:`_orm.Session`   中，直到刷新为止，如前所述，如果我们发送一个查询：

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

上面的SELECT之前是一条DELETE，表示要删除 ``patrick``，ORM 会查找与该行存在关联的 ``address`` 表中的行；这个行为被称为   :term:`级联`  ，可以通过允许数据库自动处理 ` `address`` 中的相关行来更高效地实现；   :ref:`cascade_delete`   部分对此进行了详细的说明。

.. seealso::

       :ref:`cascade_delete`   - 描述了如何调整   :meth:` _orm.Session.delete`   的行为以处理其他表中的相关行。

除此以外，即将被删除的 ``patrick`` 对象实例不再被认为是在    :class:`_orm.Session`   中持久化的，这可以通过   :meth:` _orm.Session`   中的包含检查来证明：

    >>> patrick in session
    False

然而，与我们对 ``sandy`` 对象所做的更新一样，在正在进行的事务中进行的每个更改都是局部的，如果不提交，它们将不会变得永久。当前我们更感兴趣的是回滚事务，在下一节中我们将介绍它。

.. _tutorial_orm_bulk:


批量处理：插入，更新和删除多行
---------------------------------------------------

本节中讨论的   :term:`工作单元`   技术旨在将   :term:` dml`  （INSERT/UPDATE/DELETE 语句）与 Python 对象机制集成，通常涉及复杂的相互关联的对象图。一旦通过   :meth:`.Session.add`    将对象添加到    :class:` .Session`   中，作为增加属性值或修改属性值的结果，工作单元进程会自动透明地发出 INSERT/UPDATE/DELETE。

然而，ORM    :class:`.Session`   也可以处理允许本身发出 INSERT、UPDATE 和 DELETE 语句的命令，而不是传递任何经过ORM持久化的对象。通过传递要插入、更新、合并或 WHERE 条件的列表，以便一次性调用可匹配多行的的 UPDATE 或 DELETE 语句，在无需构建和操作映射对象的情况下，可以影响多行。当必须在不需要构建和操作映射对象的情况下影响大量行时，这种用法特别重要。

ORM    :class:`_orm.Session`  、   :func:` _dml.update`   和    :func:`_dml.delete`  ，它们的使用方式与在 SQLAlchemy Core中使用方式相似，与    :class:` _engine.Connection`   不同的仅是其构造、执行和结果处理与 ORM 完全集成。

有关使用这些功能的背景和示例，请参见    :ref:`queryguide_toplevel`   中的    :ref:` orm_expression_update_delete`   章节。

.. seealso::

       :ref:`orm_expression_update_delete`   - in the    :ref:` queryguide_toplevel`  


回滚
-------------

   :class:`_orm.Session`   有一个   :meth:` _orm.Session.rollback`   方法，在 SQL 连接上发出一个 ROLLBACK。但是，它也会影响当前与   :class:`_orm.Session`   相关联的对象（在我们以前的示例中是 Python 对象 ` `sandy``）。虽然我们已经更改了 ``.fullname`` 的 ``sandy`` 对象，但是我们希望撤消此更改。调用   :meth:`_orm.Session.rollback`   将不仅回滚事务，而且将**过期**与该    :class:` _orm.Session`   相关联的所有对象，这将导致它们在下次访问时自动刷新。使用名为“   :term:`懒加载`   ”的进程：

.. sourcecode:: pycon+sql

    >>> session.rollback()
    ROLLBACK

为了更仔细地查看“到期”过程，我们可以观察到Python对象“sandy”在其Python“__dict__”中没有剩余状态，除了一个特殊的SQLAlchemy内部状态对象::

    >>> sandy.__dict__
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>}

这是“   :term:`过期`   ”状态；再次访问属性将自动开始新事务，并使用当前数据库行刷新“sandy”：

.. sourcecode:: pycon+sql

    >>> sandy.fullname
    {execsql}BEGIN (implicit)
    SELECT user_account.id AS user_account_id, user_account.name AS user_account_name,
    user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (2,){stop}
    'Sandy Cheeks'

现在，我们可以观察到完整的数据库行也被填充到``sandy``对象的``__dict__``中::

    >>> sandy.__dict__  # doctest: +SKIP
    {'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>,
     'id': 2, 'name': 'sandy', 'fullname': 'Sandy Cheeks'}

对于已删除的对象，当我们早先注意到``patrick``不再在会话中时，该对象的身份也被恢复了::

    >>> patrick in session
    True

当然，数据库数据也再次出现了：

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

在上面的部分中，我们在Python上下文管理器之外使用了   :class:`_orm.Session`  ` with``语句。没关系，然而，如果我们按照这种方式做事情，最好在完成时明确地关闭   :class:`_orm.Session`  ：

.. sourcecode:: pycon+sql

    >>> session.close()
    {execsql}ROLLBACK

关闭   :class:`_orm.Session`  ，这也是使用它在上下文管理器中时发生的事情，会完成以下几件事情：

* 它：term:`释放`所有连接资源，并取消任何正在进行的事务（例如，回滚）。

  这意味着当我们使用会话执行某些只读任务然后关闭它时，我们不需要显式调用   :meth:`_orm.Session.rollback`   来确保回滚事务；连接池会处理此事。

* 它**删除**所有  :class:`_orm.Session` 中的对象。

  这意味着我们为此   :class:`_orm.Session`  ` sandy``，``patrick``和``squidward``，现在处于一种称为“   :term:`分离`    ”的状态。特别是，我们将注意到仍处于   :term:` 过期`   状态的对象，例如由于调用了   :meth:`_orm.Session.commit`   ，现在是无法使用的，因为它们不包含当前行的状态，并且不再与任何数据库事务相关联以进行刷新::

    >>> squidward.name
    Traceback (most recent call last):
      ...
    sqlalchemy.orm.exc.DetachedInstanceError: Instance <User at 0x...> is not bound to a Session; attribute refresh operation cannot proceed

  可以使用   :meth:`_orm.Session.add`   方法将分离的对象重新关联到相同或新的   :class:` _orm.Session`  ，这将重新建立它们与它们特定数据库行的关系：

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

      如果可能，尽量避免使用分离状态的对象。当关闭   :class:`_orm.Session`   标志设置为“False”。