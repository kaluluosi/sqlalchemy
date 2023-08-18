.. highlight:: pycon+sql

.. |prev| replace:: :doc:`orm_data_manipulation`
.. |next| replace:: :doc:`further_reading`

.. include:: tutorial_nav_include.rst

.. rst-class:: orm-header

.. _tutorial_orm_related_objects:

处理 ORM 相关对象（Related Objects）
======================================

本节中，我们将介绍 ORM 的另一个重要概念，即 ORM 如何与引用其他对象的映射类进行交互。在“声明映射类”一节中，映射类示例通过使用称为 :func:`_orm.relationship` 的结构来实现。这个结构定义了两个不同的映射类之间的链接关系，或者自映射类，后者称为**自引用关系**。

为了描述 :func:`_orm.relationship` 的基本思想，首先我们将简短地回顾一下映射，省略 :func:`_orm.mapped_column` 映射和其他指令：

.. sourcecode:: python


    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship


    class User(Base):
        __tablename__ = "user_account"

        # ... mapped_column() mappings

        addresses: Mapped[List["Address"]] = relationship(back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        # ... mapped_column() mappings

        user: Mapped["User"] = relationship(back_populates="addresses")

上面，``User`` 类现在有一个 ``User.addresses`` 属性，``Address`` 类有一个 ``Address.user`` 属性。 :func:`_orm.relationship` 结构与 :class:`_orm.Mapped` 结构配合使用，用于检查 :class:`_schema.Table` 对象之间的表关系，这些对象与 ``User`` 和 ``Address`` 类映射。由于代表 ``address`` 表的 :class:`_schema.Table` 对象具有引用到 ``user_account`` 表的 :class:`_schema.ForeignKeyConstraint`， :func:`_orm.relationship` 可以明确地确定 ``User`` 类向 ``Address`` 类存在一种 :term:`一对多` 关系，以及 ``User.addresses`` 关系；``user_account`` 表中的一个特定行可以被 ``address`` 表中的多行引用。

所有一对多关系自然都对应着另一方的 :term:`多对一` 关系，在本例中，是被称为 ``Address.user`` 的关系。如上面看到的 :paramref:`_orm.relationship.back_populates` 参数，两个 :func:`_orm.relationship` 对象都配置了另一个名称，这表明两个 :func:`_orm.relationship` 结构应被视为相互补充；下一节中我们将看到它是如何运作的。

持久化和加载关系对象
------------------------

我们可以通过演示 :func:`_orm.relationship` 对对象实例的行为来开始。如果我们创建一个新的 ``User`` 对象，我们可以注意到，在访问 ``.addresses`` 元素时，有一个 Python 列表::

    >>> u1 = User(name="pkrabs", fullname="Pearl Krabs")
    >>> u1.addresses
    []

此对象是 Python ``list`` 的 SQLALchemy 特定版本，它具有跟踪并响应其变化的能力。即使我们从未将其分配给对象，这个集合也会自动出现，这类似于在 :ref:`tutorial_inserting_orm` 中观察到的行为，其中注意到我们没有明确将值赋给的基于列的属性也会自动显示为 ``None``，而不是像 Python 的通常行为那样引发 ``AttributeError``。

由于 ``u1`` 对象仍然是 :term:`transient` 的，而我们从 ``u1.addresses`` 中获取的 ``list`` 未被更改，因此它实际上还没有与该对象相关联，但是随着我们对其进行更改，它将成为 ``User`` 对象的状态的一部分。

该集合特定于 ``Address`` 类，它是唯一可以在其中持久化的 Python 对象类型。使用 ``list.append()`` 方法，我们可以添加 ``Address`` 对象：

  >>> a1 = Address(email_address="pearl.krabs@gmail.com")
  >>> u1.addresses.append(a1)

此时，预期的 ``u1.addresses`` 集合包含新的 ``Address`` 对象：

  >>> u1.addresses
  [Address(id=None, email_address='pearl.krabs@gmail.com')]

由于我们使用了 :paramref:`_orm.relationship.back_populates` 参数，在两个引用不同名称的 :func:`_orm.relationship` 对象上进行了配置，可以从 ``User`` 对象导航到 ``Address`` 对象，我们还可以从 ``Address`` 对象导航回“父” ``User`` 对象：

  >>> a1.user
  User(id=None, name='pkrabs', fullname='Pearl Krabs')

在这两个  :func:`_orm.relationship` 对象之间的互补属性赋值/列表变异将被考虑。若我们创建另一个 ``Address`` 对象并将其分配给 ``Address.user`` 属性，那么``Address`` 对象将成为该``User`` 对象上的``User.addresses`` 集合的一部分：

  >>> a2 = Address(email_address="pearl@aol.com", user=u1)
  >>> u1.addresses
  [Address(id=None, email_address='pearl.krabs@gmail.com'), Address(id=None, email_address='pearl@aol.com')]

我们实际上在 ``Address`` 构造函数中使用了 ``user`` 参数作为关键字参数，其与在 ``Address`` 类上声明的其他映射属性一样被接受。它等效于在事实之后分配 ``Address.user`` 属性：

  # 等效作用于 a2 = Address(user=u1)
  >>> a2.user = u1


.. _tutorial_orm_cascades:

将对象添加到会话（Session）中
---------------------------------------

我们现在有一个关联的双向结构中的 ``User`` 和两个 ``Address`` 对象，但正如在 :ref:`tutorial_inserting_orm` 中所述，在它们与 :class:`_orm.Session` 对象相关联之前，这些对象被称为 :term:`transient`。

我们使用 :class:`_orm.Session`，并注意到当我们将 :meth:`_orm.Session.add` 方法应用于“主” ``User`` 对象时，相关的 ``Address`` 对象也被添加到同一个 :class:`_orm.Session` 中:

  >>> session.add(u1)
  >>> u1 in session
  True
  >>> a1 in session
  True
  >>> a2 in session
  True

上面的行为中，当 :class:`_orm.Session` 接收到 ``User`` 对象，并沿着 ``User.addresses`` 关系寻找一个相关的 ``Address`` 对象时，它会遵循“save-update cascade”，并决定使用同一个 :class:`_orm.Session` 对象插入所有相关的记录。ORM 参考文档在 :ref:`unitofwork_cascades` 中详细介绍了这种行为。

三个对象现在处于 :term:`pending` 状态；这意味着它们已准备好成为插入操作的对象，但这尚未进行；所有三个对象还没有分配主键，并且 ``a1`` 和 ``a2`` 对象还有一个名为 ``user_id`` 的属性，引用有一个 :class:`_schema.ForeignKeyConstraint` 引用 ``user_account.id`` 列的 :class:`_schema.Column`；因为这些对象尚未与真正的数据库行相关联，因此它们也是 ``None``：

    >>> print(u1.id)
    None
    >>> print(a1.user_id)
    None

此时，我们可以看到单元操作（unit of work）过程提供的特别方便；回顾 :ref:`tutorial_core_insert_values_clause` 中，在 ``user_account`` 和 ``address`` 表中插入行时，使用了一些复杂的语法，以便自动将 ``address.user_id`` 列与 ``user_account`` 行的那些列相关联。此外，在 ``address`` 行之前必须先执行插入 ``user_account`` 行，因为 ``address`` 行**依赖于**它们的父行 ``user_account``，在其 ``user_id`` 列中需要一个值。当使用 :class:`_orm.Session` 时，所有这些繁琐的工作都是由它来处理的，即使是最执着的 SQL 纯洁主义者也可以从 INSERT、UPDATE 和 DELETE 语句的自动化中获益。当我们通过 :meth:`_orm.Session.commit` 提交事务时，所有步骤按正确顺序执行，此外新生成的 ``user_account`` 行的主键将正确应用于 ``address.user_id`` 列：

.. sourcecode:: pycon+sql

  >>> session.commit()
  {execsql}INSERT INTO user_account (name, fullname) VALUES (?, ?)
  [...] ('pkrabs', 'Pearl Krabs')
  INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
  [... (insertmanyvalues) 1/2 (ordered; batch not supported)] ('pearl.krabs@gmail.com', 6)
  INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
  [insertmanyvalues 2/2 (ordered; batch not supported)] ('pearl@aol.com', 6)
  COMMIT




.. _tutorial_loading_relationships:

加载关联
---------------------

在上一步中，我们调用了 :meth:`_orm.Session.commit`，它发出了一个事务的 COMMIT命令，然后使用 :paramref:`_orm.Session.commit.expire_on_commit` 对所有对象进行过期，以便它们在下一个事务中刷新。

下一次访问这些对象的属性时，我们将看到主键属性的 SELECT 语句已经发出，例如当我们查看 ``u1`` 对象的新生成主键时：

.. sourcecode:: pycon+sql

  >>> u1.id
  {execsql}BEGIN (implicit)
  SELECT user_account.id AS user_account_id, user_account.name AS user_account_name,
  user_account.fullname AS user_account_fullname
  FROM user_account
  WHERE user_account.id = ?
  [...] (6,){stop}
  6

``u1`` ``User`` 对象现在具有持久化的集合 ``User.addresses``，我们也可以访问它。由于该集合由 ``address`` 表中的另一组行组成，因此当我们再次访问此集合时，将再次发出 :term:`lazy load` 以检索对象：

.. sourcecode:: pycon+sql

  >>> u1.addresses
  {execsql}SELECT address.id AS address_id, address.email_address AS address_email_address,
  address.user_id AS address_user_id
  FROM address
  WHERE ? = address.user_id
  [...] (6,){stop}
  [Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')]

ORM 中的集合和关联属性在内存中是持久的。一旦集合或属性被填充，直到重新 :term:`expired`，SQL 才不会再次发出。我们可以再次访问 ``u1.addresses``，添加或删除项目，这不会产生任何新的 SQL 调用：

  >>> u1.addresses
  [Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')]

如果我们不采取明确的步骤对其进行优化，懒加载可能会很快地变得耗费资源。所有那些未加载的属性的网络构成“懒加载”的状态都被精心地优化，以便不执行冗余操作;由于 ``u1.addresses`` 集合被刷新，根据 :term:`identity map`，这实际上是与我们已经处理过的 ``a1`` 和 ``a2`` 对象相同的 ``Address`` 实例，因此我们已经完成了加载此特定对象图中的所有属性。

关系载入的加载方式是一个非常重要的主题，需要单独讨论。下一节在 :ref:`tutorial_orm_loader_strategies` 中会介绍这些概念。

.. _tutorial_select_relationships:

在查询中使用关联
------------------------------

在上一节中，我们介绍了 :func:`_orm.relationship` 在处理**映射类实例**时的行为，也就是 ``u1``、``a1`` 和 ``a2`` 等 ``User`` 和 ``Address`` 类的实例。在本节中，我们将介绍 :func:`_orm.relationship` 作为映射类**类级别行为**的行为，其中它有助于自动化 SQL 查询的构建。

.. _tutorial_joining_relationships:

使用关联进行连接
^^^^^^^^^^^^^^^^^^^^^^^^^^^

“选择 JOIN”一节和“选择 ON Clause”一节介绍了 :meth:`_sql.Select.join` 和 :meth:`_sql.Select.join_from` 方法来组成 SQL 连接子句。为了描述如何在表之间进行连接，这些方法可以根据表元数据结构中单个含糊的 :class:`_schema.ForeignKeyConstraint` 对象的存在推断出 ON 子句，或者我们可以提供显示的 SQL 表达式构造，表示一个特定的 ON 子句。

在使用 ORM 实体时，为了帮助我们设置 join 的 ON 子句，还提供了另一种机制，即利用我们在映射中设置的 :func:`_orm.relationship` 对象，例如在“声明映射类”一节中演示的方式。与 :class:`_schema.Table` 对象之间的表关系将在 :func:`_orm.relationship` 中使用，该对象将作为**单个参数**传递到 :meth:`_sql.Select.join` 方法中，其中它表示右侧的连接，并在一次指定中同时指定On子句::

    >>> print(select(Address.email_address).select_from(User).join(User.addresses))
    {printsql}SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

如果没有指定 ON 子句，那么在 ``User`` 到 ``Address`` 的连接中仍然起作用，是因为两个映射的 :class:`_schema.Table` 对象之间有一个 :class:`_schema.ForeignKeyConstraint`，而不是在 ``User`` 和 ``Address`` 类上的 :func:`_orm.relationship` 对象：

    >>> print(select(Address.email_address).join_from(User, Address))
    {printsql}SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

有关如何使用 :meth:`.Select.join` 和 :meth:`.Select.join_from` 的示例，可参见 :ref:`orm_queryguide_joins`。

.. seealso::

    在 :ref:`queryguide_toplevel` 中的 :ref:`orm_queryguide_joins`

.. _tutorial_relationship_operators:

关系 WHERE 运算符
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_orm.relationship` 中还有一些其他类型的 SQL 生成帮助程序，通常在生成语句的 WHERE 子句时非常有用。请参见 :ref:`queryguide_relationship_operators` 中的示例。

.. seealso::

    在 :ref:`queryguide_toplevel` 中 :ref:`orm_queryguide_relationship_operators`

.. _tutorial_orm_loader_strategies:

载入策略
-----------------

在 :ref:`tutorial_loading_relationships` 中，介绍了当我们处理映射对象的实例时，访问使用 :func:`_orm.relationship` 映射的属性在默认情况下将发出 :term:`lazy load`，以加载应该存在于该集合中的对象。

延迟加载是 ORM 模式中最著名的模式之一，也是最具争议的。当许多 ORM 对象都引用同一相关对象（例如，许多 ``Address`` 对象都引用相同的 ``User``），操作这些对象时会产生大量冗余查询，可能会加起来（也被称为 :term:`N加1` 问题），更糟糕的是它们是隐式发出的。如果不能注意到这些隐式查询，当他们在数据库事务不再可用时尝试访问他们或者在使用其他并发模式（如 :ref:`asyncio <asyncio_toplevel>`）时都会引发错误。

同时，懒加载是一个非常受欢迎和有用的模式，如果与正在使用的并发方法兼容并且没有引起问题，则可以多方面受益于 SQLAlchemy 的 ORM。

首先，使用 ORM 懒加载的有效方法是**测试应用程序，启用 SQL 追踪，查看已发出的 SQL 语句**。如果出现许多看起来非常像可以合并在一起的冗余 SELECT 语句，如果对已经与其 :class:`_orm.Session` 分离的对象尝试执行这些语句，则可能会引发错误。如果使用另一种并发方法，例如 :ref:`asyncio <asyncio_toplevel>`，这些隐式查询实际上根本不起作用。

最重要的是，使用 ORM 加载延迟式加载的首要步骤是**测试应用程序，启用 SQL 追踪，并查看输出的 SQL**语句。

总之，负责管理对象状态的 ORM 查询层放置了很多重点在能够控制和优化这种加载行为方面。

最重要的基本概念是要掌握如何使用加载器策略来提高性能。

加载器策略被表示为对象，可以使用 :meth:`_sql.Select.options` 方法将其与 SELECT 语句关联起来，例如::

      for user_obj in session.execute(
          select(User).options(selectinload(User.addresses))
      ).scalars():
          user_obj.addresses  # 访问已经加载的地址集合

它们也可以在 :func:`_orm.relationship` 中默认配置，使用 :paramref:`_orm.relationship.lazy` 选项，例如：

.. sourcecode:: python

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship


    class User(Base):
        __tablename__ = "user_account"

        addresses: Mapped[List["Address"]] = relationship(
            back_populates="user", lazy="selectin"
        )

每种加载器策略对象都会向一条 SELECT 语句中添加某种类型的信息，稍后，当 :class:`_orm.Session`决定应该如何加载和/或在访问属性时如何行事时，这些信息将被使用。

下面的章节将介绍一些最常用的加载器策略。

.. seealso::

    两个 :ref:`loading_toplevel` 中的部分：

    * :ref:`relationship_lazy_option` - 关于如何在 :func:`_orm.relationship`` 中配置此策略

    * :ref:`relationship_loader_options` - 关于使用查询时加载器策略的详细信息

选择 In Load
^^^^^^^^^^^^^^^

在现代 SQLAlchemy 中，最有用的加载器是 :func:`_orm.selectinload` 加载程序。此选项解决了“N加1”问题的最常见形式，即一组对象引用相关集合。 :func:`_orm.selectinload` 将确保使用单个查询前端加载要加载的集合的完整一系列对象。它使用的 SELECT 语句的形式通常可以针对相关表本身进行发射，无需加入连接或次查询，并且仅针对未加载集合的那些父对象查询进行查询。下面的例子演示了 :func:`_orm.selectinload`，加载所有的 ``User`` 对象及其所有相关的 ``Address`` 对象；虽然我们仅在 :meth:`_orm.Session.execute` 中传递一次 :func:`_sql.select` 构造，但在访问数据库时，实际上发出了两个 SELECT 语句，第二个 SELECT 用于获取相关的 ``Address`` 对象：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import selectinload
    >>> stmt = select(User).options(selectinload(User.addresses)).order_by(User.id)
    >>> for row in session.execute(stmt):
    ...     print(
    ...         f"{row.User.name}  ({', '.join(a.email_address for a in row.User.addresses)})"
    ...     )
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account ORDER BY user_account.id
    [...] ()
    SELECT address.user_id AS address_user_id, address.id AS address_id,
    address.email_address AS address_email_address
    FROM address
    WHERE address.user_id IN (?, ?, ?, ?, ?, ?)
    [...] (1, 2, 3, 4, 5, 6){stop}
    spongebob  (spongebob@sqlalchemy.org)
    sandy  (sandy@sqlalchemy.org, sandy@squirrelpower.org)
    patrick  ()
    squidward  ()
    ehkrabs  ()
    pkrabs  (pearl.krabs@gmail.com, pearl@aol.com)

.. seealso::

    :ref:`selectin_eager_loading` - 在 :ref:`loading_toplevel` 中

Joined Load
^^^^^^^^^^^

:func:`_orm.joinedload` 构建一个 JOIN 语句，用于修改即将传递到数据库的 :class:`_sql.Select` 的方式。具体而言，在返回一个主实体行时，它向其中添加了几列，可以加载相关联的对象。

:func:`_orm.joinedload` 策略最适合加载相关的多对一对象，因为仅需要为要选取的主实体行添加一些附加列。为了提高效率，它还接受一个 :paramref:`_orm.joinedload.innerjoin` 选项，因此可以使用内连接代替外连接等选项，例如下面这个选项，我们知道所有 ``Address`` 对象都有关联的 ``User``：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import joinedload
    >>> stmt = (
    ...     select(Address)
    ...     .options(joinedload(Address.user, innerjoin=True))
    ...     .order_by(Address.id)
    ... )
    >>> for row in session.execute(stmt):
    ...     print(f"{row.Address.email_address} {row.Address.user.name}")
    {execsql}SELECT address.id, address.email_address, address.user_id, user_account_1.id AS id_1,
    user_account_1.name, user_account_1.fullname
    FROM address
    JOIN user_account AS user_account_1 ON user_account_1.id = address.user_id
    ORDER BY address.id
    [...] (){stop}
    spongebob@sqlalchemy.org spongebob
    sandy@sqlalchemy.org sandy
    sandy@squirrelpower.org sandy
    pearl.krabs@gmail.com pkrabs
    pearl@aol.com pkrabs

:func:`_orm.joinedload` 也适用于集合，也就是一对多关系，但是它会导致递归方式将主要行乘以关联项，以一种让结果集按级数以数量级增长的递归方式增加发送给的数据量，对于嵌套集合和/或更大的集合来说非常低效，因此与其他选项（如 :func:`_orm.selectinload`）相比，必须在每种情况下进行评估。

重要提示：

对于许多业务对象，往往没有必要使用多对一的即时加载，因为“N加1”问题在常见情况下的存在要少得多。例如，如果许多对象都引用同一个相关对象（例如，许多 ``Address`` 对象都引用相同的 ``User``），SQL 仅一次针对该 ``User`` 对象进行查询，使用正常的懒加载。懒加载例程尽可能地在当前 :class:`_orm.Session` 中查找相关对象主键。

.. seealso::

  :ref:`joined_eager_loading` - 在 :ref:`loading_toplevel` 中

.. _tutorial_orm_loader_strategies_contains_eager:

显式 JOIN + Eager load
^^^^^^^^^^^^^^^^^^^^^^^^^^^

假设我们使用 :meth:`_sql.Select.join` 渲染连接从 ``user_account`` 表中加载 ``Address`` 行，我们还可以利用它来在返回的每个 ``Address`` 对象上即时加载 ``Address.user`` 属性的内容。本质上，这就是我们使用“连接加载”（joined eager loading），并自己渲染连接。使用 :func:`_orm.contains_eager` 选项可以实现这个普遍使用的用例。该选项与 :func:`_orm.joinedload` 非常相似，只是它假设我们已经设置了 JOIN，并将其用于返回的每个对象上的相关属性中，例如：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import contains_eager
    >>> stmt = (
    ...     select(Address)
    ...     .join(Address.user)
    ...     .where(User.name == "pkrabs")
    ...     .options(contains_eager(Address.user))
    ...     .order_by(Address.id)
    ... )
    >>> for row in session.execute(stmt):
    ...     print(f"{row.Address.email_address} {row.Address.user.name}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    address.id AS id_1, address.email_address, address.user_id
    FROM address JOIN user_account ON user_account.id = address.user_id
    WHERE user_account.name = ? ORDER BY address.id
    [...] ('pkrabs',){stop}
    pearl.krabs@gmail.com pkrabs
    pearl@aol.com pkrabs

上面，我们过滤了 ``user_account.name`` 行并在 ``Address.user`` 属性中加载了 ``user_account`` 的行。如果我们单独应用 :func:`_orm.joinedload`，则会多次加入 SQL 查询：

    >>> stmt = (
    ...     select(Address)
    ...     .join(Address.user)
    ...     .where(User.name == "pkrabs")
    ...     .options(joinedload(Address.user))
    ...     .order_by(Address.id)
    ... )
    >>> print(stmt)  # SELECT has a JOIN and LEFT OUTER JOIN unnecessarily
    {printsql}SELECT address.id, address.email_address, address.user_id,
```查询与提前加载（Querying and Eager Loading）
----------------------------------------

.. seealso::

    :ref:`查询与提前加载 <querying_eagerloading_toplevel>`

    Two sections in :ref:`加载顶层 <loading_toplevel>`:

    * :ref:`急加载的基础知识 <zen_of_eager_loading>` - 详细描述了上述问题

    * :ref:`包含急加载 <contains_eager>` - 使用 :func:`.contains_eager`

Raiseload
^^^^^^^^^

值得一提的另一种加载器策略是 :func:`_orm.raiseload`。
此选项用于完全阻止应用程序出现 :term:`N plus one` 问题，
这是通过导致通常会是懒加载的内容引发错误来实现的。
它有两个变体，可以通过 :paramref:`_orm.raiseload.sql_only` 选项进行控制，
以阻止需要 SQL 的懒加载，还是所有 "load" 操作，包括那些只需要查询 current
:class:`_orm.Session`。

可以使用 :func:`_orm.raiseload` 的一种方法是在 :func:`_orm.relationship` 上直接配置它，
方法是将 :paramref:`_orm.relationship.lazy` 设置为值 ``"raise_on_sql"``，以使特定的映射中，
某个关系永远不会尝试发出 SQL：

.. setup code

    >>> class Base(DeclarativeBase):
    ...     pass

::

    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import relationship


    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     addresses: Mapped[List["Address"]] = relationship(
    ...         back_populates="user", lazy="raise_on_sql"
    ...     )


    >>> class Address(Base):
    ...     __tablename__ = "address"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     user: Mapped["User"] = relationship(back_populates="addresses", lazy="raise_on_sql")

使用这样的映射，应用程序被阻止懒加载，
表明特定查询需要指定加载器策略：

    >>> u1 = session.execute(select(User)).scalars().first()
    {execsql}SELECT user_account.id FROM user_account
    [...] ()
    {stop}>>> u1.addresses
    Traceback (most recent call last):
    ...
    sqlalchemy.exc.InvalidRequestError: 'User.addresses' is not available due to lazy='raise_on_sql'

异常将表明该集合应在前期加载：

    >>> u1 = (
    ...     session.execute(select(User).options(selectinload(User.addresses)))
    ...     .scalars()
    ...     .first()
    ... )
    {execsql}SELECT user_account.id
    FROM user_account
    [...] ()
    SELECT address.user_id AS address_user_id, address.id AS address_id
    FROM address
    WHERE address.user_id IN (?, ?, ?, ?, ?, ?)
    [...] (1, 2, 3, 4, 5, 6)

``lazy="raise_on_sql"`` 选项也会尝试在许多对一关系方面变得更加智能；
在上面的示例中，如果一个 ``Address`` 对象的 ``Address.user`` 属性未加载，
但该 ``User`` 对象在同一 :class:`_orm.Session` 中，则 "raiseload" 策略不会引发错误。

.. seealso::

    :ref:`防止懒加载 <prevent_lazy_with_raiseload>` - 在 :ref:`加载顶层 <loading_toplevel>`