关系属性加载技术
===================

.. admonition:: 本文档说明

    本节详细介绍了如何在查询时加载相关对象的详细信息。
    读者应该熟悉 :ref: `relationship_config_toplevel`
    和基本用法。

    大部分示例都假定了一个类似于
    :doc:`设置从查询 <_plain_setup>`的
    "User /Address" 映射设置。

SQLAlchemy的主要之一，是提供了广泛的控制方式，来加载关系对象和集合。
通常使用 :func:`_orm.relationship` 来配置一个映射器上的集合或标量关联。
这种行为可以在构造映射器时使用 :paramref:`_orm.relationship.lazy` 参数来配置，
也可以使用 **ORM 加载器选项** 来使用:class:`_sql.Select`构造器。

关系的加载有三种方法; **lazy** 加载，**eager** 加载和 **no** 加载。
懒惰加载是指在查询的一开始，相关对象没有被加载的情况下返回的对象。
当在特定对象上首次访问给定的集合或引用时，
会发射附加的SELECT语句，以加载所请求的集合。

**Eager**加载是指在加载与查询一起返回的相关集合或标量引用的对象时，会立即完成。
ORM 要么通过在通常发射的 SELECT 语句中使用 JOIN 来同时加载相关行，或者通过在主要 SELECT 语句之后发射附加的 SELECT 语句来一次性加载集合或标量引用。

“没有” 加载是指在给定的关系上禁用加载，可能是属性为空并且不会被加载，
也可能在访问时引发错误，以防止不需要的懒惰加载。

关系加载风格总结
--------------------------------------

关系加载的主要形式包括：

* **lazy loading** - 可用于 ``lazy='select'`` 或 :func:`.lazyload` 选项，这是一种在属性访问时间发出 SELECT 语句，以懒惰地加载单个对象上的相关引用的加载形式。对于不以其他方式指示 :paramref:`_orm.relationship.lazy` 选项的所有 :func:`_orm.relationship` 构造，都使用惰性加载。惰性加载在 :ref:`lazy_loading` 中具体阐述。
* **select IN loading** - 可用于 ``lazy='selectin'`` 或 :func:`.selectinload` 选项，此加载形式会发射第二个（或更多）SELECT语句, 将父对象的主键标识符汇编到一个IN语句中，以便一次性通过主键加载所有相关集合/标量引用成员。在 :ref:`selectin_eager_loading` 中详细介绍了选择IN加载。
* **joined loading** - 可用于 ``lazy='joined'`` 或 :func:`_orm.joinedload` 选项，此加载形式将应用 JOIN 到给定的 SELECT 语句中，以便关联行可以在同一结果集中加载。详细介绍了连接式“通常负载” 的 :ref:`joined_eager_loading`。
* **raise loading** - 可使用 ``lazy='raise'``， ``lazy='raise_on_sql'`` 或 :func:`.raiseload` 选项，此加载形式在通常会发生懒惰加载的同时触发，但是引发 ORM 异常以防止应用程序进行不必要的懒惰加载。 :ref:`prevent_lazy_with_raiseload` 中介绍了教程。
* **subquery loading** - 可用于 ``lazy='subquery'`` 或 :func:`.subqueryload` 选项，此加载形式发射第二个 SELECT 语句，其中重新声明原始查询嵌入在子查询中，然后将该子查询与要加载的相关表连接以一次性加载所有相关集合/标量引用成员。在 :ref:`subquery_eager_loading` 中详细介绍了子查询快照加载。
* **write only loading** - 可通过 ``lazy='write_only'`` 或通过 :class:`_orm.WriteOnlyMapped` 注释来注释左侧的 :class:`_orm.Relationship` 对象。该加载器只会生成一种备选的属性检查，该属性检查从不从数据库中隐式加载记录，而只允许 :meth:`.WriteOnlyCollection.add`，:meth:`.WriteOnlyCollection.add_all` 和 :meth:`.WriteOnlyCollection.remove` 方法。通过调用 :meth:`.WriteOnlyCollection.select` 方法执行查询集合。 :ref:`write_only_relationship` 详细讨论了只写装载。
* **dynamic loading** - 可用于 ``lazy='dynamic'``，或通过 :class:`_orm.DynamicMapped` 注释将左侧的 :class:`_orm.Relationship` 对象注释掉。这是一个传统的仅限于集合的装载器风格，当访问集合时，它会生成 :class:`_orm.Query` 对象，允许针对集合内容发出自定义 SQL。但是，在各种情况下，动态 loader 会隐式迭代基础集合，这使它们不适用于管理真正大的集合。动态加载器的优先级高于:ref:`"只写"<write_only_relationship>` 集合，后者将阻止基础集合在任何情况下隐式加载。 :ref:`dynamic_relationship` 讨论了动态关系。

.. _relationship_lazy_option:

在映射时配置加载器策略
---------------------------------------------

可以在映射时配置一个特定关系的加载程序策略，以在加载映射器类型的对象时，
缺少修改任何修改它的查询级选项。这是使用 :func:`_orm.relationship` 中的
:paramref:`_orm.relationship.lazy` 参数配置的；此参数的常见值包括 ``select``，``selectin`` 和 ``joined``。

下面的示例说明了在 :ref:`relationship_patterns_o2m` 中配置 ``Parent.children`` 关系，
使其在发出 ``Parent`` 对象的 SELECT 语句时使用 :ref:`selectin_eager_loading`::

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Parent(Base):
        __tablename__ = "parent"

        id = mapped_column(int, primary_key=True)
        children = relationship(lazy="selectin")


    class Child(Base):
        __tablename__ = "child"

        id = mapped_column(int, primary_key=True)
        parent_id = mapped_column(ForeignKey("parent.id"))

在上面的示例中，每次加载一组“父”对象时，都会将每个“父”使用“selectin”的加载程序策略
拉入到其“children”集合中。

:param:`_orm.relationship.lazy` 默认值是 ``"select"``，表示懒惰加载。

.. _relationship_loader_options:

使用加载程序选项加载关系
----------------------------------------

配置加载策略的另一种常见方法是在特定属性上为每个查询设置它们，使用方法是 :meth:`_sql.Select.options`。
使用加载程序选项可获得对关系加载的非常详细的控制，
最常见的是 :func:`_orm.joinedload`，:func:`_orm.selectinload` 和 :func:`_orm.lazyload`。
该选项接受一个 class-bound 属性，引用特定类/属性，应该以此为目标::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    # 将 children 设置为懒惰加载
    stmt = select(Parent).options(lazyload(Parent.children))


    from sqlalchemy.orm import joinedload

    # 设置 joined load 来一次性加载 children
    stmt = select(Parent).options(joinedload(Parent.children))

可以使用 **方法链接** 来链接多个外键，以指定多个深度的加载方式::

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload

    stmt = select(Parent).options(
        joinedload(Parent.children).subqueryload(Child.subelements)
    )

链式加载选项可以应用于“懒惰”加载的集合。这意味着当一个集合或关联在访问时被惰性加载，
然后这个指定的选项将会生效::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))

上述查询将返回未加载 ``children`` 集合的 `Parent` 对象。在首次访问某个特定“父”对象上
的“children”集合时，它会懒惰地加载相关对象，但还会对每个 `children` 成员上的 `subelements` 集合应用积极加载。

.. _loader_option_criteria:

加载程序选项添加筛选条件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

用于指示加载程序选项的关系属性包括向 JOIN ON 子句附加其他筛选条件的功能，
或者将它们包含在涉及的 WHERE 条件中，具体取决于加载程序策略。
可以使用 :meth:`.PropComparator.and_` 方法实现此目的，传递选项，
以使加载的结果仅限于给定的筛选条件::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = select(A).options(lazyload(A.bs.and_(B.id > 5)))

如果已经加载了特定集合，则不会刷新它;
为确保新条件生效，应使用 :ref:`orm_queryguide_populate_existing` 执行选项::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = (
        select(A)
        .options(lazyload(A.bs.and_(B.id > 5)))
        .execution_options(populate_existing=True)
    )

为所有出现在查询中的一个实体添加过滤条件，而不考虑加载程序策略或它在加载过程中出现的位置，请参见 :func:`_orm.with_loader_criteria` 函数。

.. versionadded:: 1.4

.. _orm_queryguide_relationship_sub_options:

使用 Load.options() 指定子选项
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用方法链接，将路径中每个链接的加载样式明确说明。 若要沿着路径导航，
而不更改特定属性的现有加载程序样式，请使用 :func:`.defaultload` 方法/函数：

    from sqlalchemy import select
    from sqlalchemy.orm import defaultload

    stmt = select(A).options(defaultload(A.atob).joinedload(B.btoc))

可以使用类似的方法指定多个子选项， 使用 :meth:`_orm.Load.options` 方法::

    from sqlalchemy import select
    from sqlalchemy.orm import defaultload, joinedload

    stmt = select(A).options(
        defaultload(A.atob).options(
            joinedload(B.btoc), joinedload(B.btod))
        )

请参阅 :ref:`orm_queryguide_load_only_related` - 构造关系和列向量加载器选项的组合示例。

.. note:: 加载已懒惰加载的对象的加载程序选项
   “黏性”，针对特定对象实例是“黏性” 的，这意味着它们会在特定对象所加载的集合上保持，
   只要它存在于内存中。例如，给定上述查询：

      stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))

   如果以上查询加载的 ``children`` 集合过期（例如，当 :class:`.Session` 对象的事务
   提交或回滚时使用或使用 :meth:`.Session.expire_all` 方法时），则
   当下次访问 ``Parent.children`` 集合以重新加载它时，将再次使用子查询快照
   加载 ``Child.subelements`` 集合。即使从指定不同的查询访问上述“父”对象时，
   此情况始终如此。要在不撤消它并重新加载的情况下更改现有对象上的选项，
   必须结合使用 :ref:`orm_queryguide_populate_existing` 执行选项来显式设置它们：

      # 更改已加载的 Parent 对象上的选项
      stmt = (
          select(Parent)
          .execution_options(populate_existing=True)
          .options(lazyload(Parent.children).lazyload(Child.subelements))
          .all()
      )

   如果从 :class:`.Session` 完全清除了以上加载对象，例如由于垃圾回收或使用了
   :meth:`.Session.expunge_all`，则“黏性”选项也将消失，如果再次加载新创建的
   对象，则会使用新选项。

   将来的 SQLAlchemy 版本可能会添加更多操作已加载对象中的加载程序选项的选项。

.. _lazy_loading:

延迟加载
------------

默认情况下，所有的对象关系都是**懒惰加载**。与 :func:`_orm.relationship` 关联的标量或集合属性
包含一个触发器，该触发器在访问属性时首次发生。此触发器通常会在访问点上发出 SQL 调用，
以便加载相关的对象或对象：

.. sourcecode:: pycon+sql

    >>> spongebob.addresses
     {execsql}SELECT
        addresses.id AS addresses_id,
        addresses.email_address AS addresses_email_address,
        addresses.user_id AS addresses_user_id
    FROM addresses
    WHERE ? = addresses.user_id
    [5]
    {stop}[<Address(u'spongebob@google.com')>, <Address(u'j25@yahoo.com')>]

唯一的一种情况不发射 SQL 的情况是一个简单的多对一关系，
当相关对象可以仅通过其主键值来标识且该对象已经存在于当前 :class:`.Session` 中时。
因此，尽管惰性加载可能对相关集合来说代价高昂，但在加载许多对象并针对相对较小的可能目标对象集合中使用
许多对一个简单的多对一引用时，惰性加载可能能够将这些对象本地引用起来，而不需要发射与父对象数目相等的 SELECT 语句。

此默认行为的“load upon attribute access"，被称为“lazy”或“select”加载 -
因为当第一次访问属性时通常会发出“ SELECT” 语句。

可以使用 :func:`.lazyload` 加载程序选项启用通常以其他方式配置的给定属性的懒惰加载::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    # 强制使用其他方式设置的属性进行延迟加载
    stmt = select(User).options(lazyload(User.addresses))

.. _prevent_lazy_with_raiseload:

使用 raiseload 来防止不必要的惰性加载
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`.lazyload` 策略会产生一个最常见的问题对象关系映射中所指示的，
也就是 :term:`N plus one problem`，表示对于任何N个已加载的对象，
访问它们的懒惰加载属性意味着将发射 N+1 个 SELECT 语句。
在 SQLAlchemy 中，解决 N+1 问题的常见方法是使用其非常能干的急加载系统。然而，急加载要求在 :class:`_sql.Select` 前
指定要加载的属性。代码可能因通过懒加载访问其他属性和集合时不需要懒加载时的问题而产生混淆。
可以使用 :func:`.raiseload` 策略来解决这个问题;这个加载程序策略将懒惰加载的行为替换为引发一个信息型 ORM 异常::

    from sqlalchemy import select
    from sqlalchemy.orm import raiseload

    stmt = select(User).options(raiseload(User.addresses))

上面的查询将不会加载 `User` 对象，它的 ``.addresses`` 集合；
如果稍后的某些代码尝试访问此属性，将引发 ORM 异常。

:func:`.raiseload` 可以使用所谓的“通配符”指定符来进行使用，以表明所有关系都应使用此策略。例如，
要设置仅一个属性作为紧急加载，而所有其他属性作为 raise：

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload
    from sqlalchemy.orm import raiseload

    stmt = select(Order).options(joinedload(Order.items), raiseload("*"))

上面的通配符将适用于 **所有** 注释除 ``items`` 以外的关系，不仅适用于 ``Order``，而且适用于 ``Item`` 对象上的所有关系。要对整个查询中存在的所有实体添加筛选条件，
而不考虑加载程序策略或它在加载过程中的位置，请参见 :func:`_orm.with_loader_criteria` 函数。

.. tip:: “raise”策略在 flush 过程中的 :term:`工作单元` 中不适用。因此，如果 :meth:`_orm.Session.flush` 过程需要加载集合才能完成其工作，则会在绕过任何 :func:`_orm.raiseload` 指令的情况下加载该集合。

.. seealso::

    :ref:`wildcard_loader_strategies`

    :ref:`orm_queryguide_deferred_raiseload`

.. _joined_eager_loading:

连接式“通常负载”
---------------

连接式“通常负载”是包含在 SQLAlchemy ORM 中最古老的急加载样式。 默认情况下，它通过连接一个 JOIN （默认为左外连接）连接到发射的 SELECT 语句，并将目标标量/集合填充到与父级信息相同的结果集中。

在映射层面，它看起来像这样：

    class Address(Base):
        # ...

        user = relationship(lazy="joined")

连接式“通常负载”通常作为查询选项应用于特定属性，而不是应用于映射的默认加载选项，特别是在集合而不是多对一引用中使用。 这是使用 :func:`_orm.joinedload`
加载程序选项来实现的：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import joinedload
    >>> stmt = select(User).options(joinedload(User.addresses)).filter_by(name="spongebob")
    >>> spongebob = session.scalars(stmt).unique().all()
    {execsql}SELECT
        addresses_1.id AS addresses_1_id,
        addresses_1.email_address AS addresses_1_email_address,
        addresses_1.user_id AS addresses_1_user_id,
        users.id AS users_id, users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    LEFT OUTER JOIN addresses AS addresses_1
        ON users.id = addresses_1.user_id
    WHERE users.name = ?
    ['spongebob']


.. tip:: 当包括 :func:`_orm.joinedload` 引用到一个一对多或多对多集合时，
  必须将 :meth:`_result.Result.unique` 方法应用于返回的结果，这将通过主键将传入行唯一化。
  ORM 将在没有这个方法的情况下引发错误。

  由于这在返回结果集时更改了结果集的行为，使其少返回 ORM 对象而不是语句通常返回的行数，
  因此，这个作法对于现代 SQLAlchemy 来说不是自动的。因此 SQLAlchemy 明确保留了使用
  :meth:`_result.Result.unique`，这样就不会有任何模糊性，返回的对象按主键去重。

左外部连接是默认发射的 JOIN ，以便允许不引用相关行的主对象。 对于被保证的属性（例如，引用一个多对一关系到一个相关对象的地方，其中引用的外键是 NOT NULL），此查询可以通过使用内部 JOIN 来使其更加有效率；这可以通过使用 :paramref:`_orm.relationship.innerjoin` 标识符来在映射级别上实现::

    class Address(Base):
        # ...

        user_id = mapped_column(int, ForeignKey("users.id"))
        user = relationship(lazy="joined", innerjoin=True)

在查询选项级别上，通过使用 :paramref:`_orm.joinedload.innerjoin` 标识符来实现：

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload

    stmt = select(Address).options(joinedload(Address.user, innerjoin=True))

当被用于外部连接具有包含 OUTER JOIN 的链时，发出的 JOIN 将会右嵌套：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import joinedload
    >>> stmt = select(User).options(joinedload(User.addresses).joinedload(Address.widgets, innerjoin=True))
    >>> results = session.scalars(stmt).unique().all()
    {execsql}SELECT
        widgets_1.id AS widgets_1_id,
        widgets_1.name AS widgets_1_name,
        addresses_1.id AS addresses_1_id,
        addresses_1.email_address AS addresses_1_email_address,
        addresses_1.user_id AS addresses_1_user_id,
        users.id AS users_id, users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users LEFT OUTER JOIN (addresses AS addresses_1 JOIN widgets AS widgets_1 ON addresses_1.widget_id = widgets_1.id)
        ON users.id = addresses_1.user_id

.. tip:: 如果通过 :meth:`_sql.Select.with_for_update` 方法使用数据库行锁定技术，刚刚意味着使用来自于 :term:`后端` 的连接
    SELECT..FOR UPDATE，由于使用 OUTER JOIN 的预设，选项中加入了它，以确保连接上的表也被锁定。

连接式加载通常会产生混淆，因为它看起来与 :meth:`_sql.Select.join` 的使用方式相似。重要的是要理解区别，虽然 :meth:`_sql.Select.join` 用于更改查询的结果，但 :func:`_orm.joinedload` 会尽力避免更改查询结果，而是隐藏渲染的连接的效果，只允许关联对象存在的原因。

加载程序策略的哲学是可以将任何一组加载方案应用于特定的查询，并且*结果不会改变* - 只会改变完全加载相关对象和集合所需的 SQL 语句的数量。一个特定的查询可以从所有懒惰加载开始。在使用中揭示出来特定的属性或集合总是被访问，而希望为这些更改加载程序策略，例如减少发射的的 SQL 语句，其余的查询保持不变，结果将保持不变。理论上说（基本上实践上），无论您对:class:`_sql.Select` 做什么，都无法使其根据更改的加载程序策略加载不同的主要或相关对象集。

特别地， :func:`_orm.joinedload` 如何实现不影响连接到父对象的行返回在任何方式的加载不同属性的目的？ 通过创建连接的匿名别名，使其无法通过查询的其他部分引用，可实现此目的。 例如，下面的查询使用 :func:`_orm.joinedload` 来创建一个从 ``users`` 到 ``addresses`` 的 LEFT OUTER JOIN，但其中添加的``Address.email_address`` 上的 ``ORDER BY`` 是无效的 - ``Address`` 实体未在查询中命名：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import joinedload
    >>> stmt = (
    ...     select(User)
    ...     .options(joinedload(User.addresses))
    ...     .filter(User.name == "spongebob")
    ...     .order_by(Address.email_address)
    ... )
    >>> result = session.scalars(stmt).unique().all()
    {execsql}SELECT
        addresses_1.id AS addresses_1_id,
        addresses_1.email_address AS addresses_1_email_address,
        addresses_1.user_id AS addresses_1_user_id,
        users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    LEFT OUTER JOIN addresses AS addresses_1
        ON users.id = addresses_1.user_id
    WHERE users.name = ?
    ORDER BY addresses.email_address   <-- 这部分是错误的！
    ['spongebob']

上面的 ``ORDER BY addresses.email_address`` 是无效的，因为 ``addresses`` 不在 FROM 列表中。读取 “User” 记录并按电子邮件地址排序的正确方法是使用 :meth:`_sql.Select.join`：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> stmt = (
    ...     select(User)
    ...     .join(User.addresses)
    ...     .filter(User.name == "spongebob")
    ...     .order_by(Address.email_address)
    ... )
    >>> result = session.scalars(stmt).unique().all()
    {execsql}
    SELECT
        users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    JOIN addresses ON users.id = addresses.user_id
    WHERE users.name = ?
    ORDER BY addresses.email_address
    ['spongebob']

上述语句当然不同于以前的语句，因为完全未包含 ``addresses`` 的列。我们可以重新添加 :func:`_orm.joinedload`，以便有两个连接 - 一个连接是我们正在排序的连接，另一个是匿名使用以加载 ``User.addresses`` 集合：

.. sourcecode:: pycon+sql


    >>> stmt = (
    ...     select(User)
    ...     .join(User.addresses)
    ...     .options(joinedload(User.addresses))
    ...     .filter(User.name == "spongebob")
    ...     .order_by(Address.email_address)
    ... )
    >>> result = session.scalars(stmt).unique().all()
    {execsql}SELECT
        addresses_1.id AS addresses_1_id,
        addresses_1.email_address AS addresses_1_email_address,
        addresses_1.user_id AS addresses_1_user_id,
        users.id AS users_id, users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users JOIN addresses
        ON users.id = addresses.user_id

在上面的示例中，我们使用 :meth:`_sql.Select.join` 来提供我们希望在后续查询条件中使用的JOIN子句，而我们使用 :func:`_orm.joinedload` 来处理只与加载 ``User.addresses`` 集合相关的加载，而对查询本身并没有任何影响。在这种情况下，这两个 JOIN 可能看起来是多余的 - 而它们确实是多余的。如果我们想要仅使用一个 JOIN 用于加载集合和排序，我们可以使用下面将要描述的 :func:`.contains_eager` 选项。但是，想要知道 :func:`joinedload` 为什么会生成目前的结果，我们可以考虑某个 ``Address`` 上的**过滤**：
```
stmt = (
    select(User)
    .join(User.addresses)
    .options(joinedload(User.addresses))
    .filter(User.name == "spongebob")
    .filter(Address.email_address == "someaddress@foo.com")
)
```
以上代码中，我们可以看到两个 JOIN 具有非常不同的作用。一个将精确匹配完全一行，即 ``Address.email_address=='someaddress@foo.com'`` 的 ``User`` 和 ``Address`` 的联接。另外一个 LEFT OUTER JOIN 将匹配``User``相关的 **所有 ** ``Address`` 行，并且仅用于填充返回的那些 ``User`` 对象的 ``User.addresses`` 集合。

通过将 :func:`_orm.joinedload` 的使用更改为另一种加载样式，我们可以完全独立地更改如何加载集合，而与检索实际要获取的 ``User`` 行使用的 SQL 无关。下面我们将将 :func:`_orm.joinedload` 更改为 :func:`.selectinload`：

```
stmt = (
    select(User)
    .join(User.addresses)
    .options(selectinload(User.addresses))
    .filter(User.name == "spongebob")
    .filter(Address.email_address == "someaddress@foo.com")
)
```
以上代码会加载与前面几个示例相同的所有 ``User`` 行，但是使用的SQL将被优化，不再使用笛卡尔积的形式。表示为 ``SELECT DISTINCT ON`` 从句。

当使用 joined eager loading 时，如果包含影响 JOIN 外接的返回行数的修饰符时(例如：DISTINCT, LIMIT，OFFSET或其等效项)，则会首先将语句包装在一个子查询内，并将用于joined eager loading的JOIN应用于子查询。 SQLAlchemy 的 joined eager loading 走了额外的路，以确保它不会影响查询结果中的最终结果，仅影响数据获取方式。 

**选择IN加载**

在大多数情况下，选择加载是最简单,达到较高效率的方法,用来急切地加载对象集合。唯一不适用选择急切加载的情况是当模型是使用复合主键,且后端数据库不支持含有IN的元组,目前只包括 SQL Server。

可以通过将``“selectin”`` 参数传递给 :paramref:`_orm.relationship.lazy` 或使用 :func:`.selectinload` 加载程序选项来提供“选择IN”类似加载。该加载程序策略发出一个SELECT，该SELECT引用父对象的主键值或在“多对一”关系的情况下引用子对象的主键值，位于IN子句中，以便加载相关联的对象。

上面这些是选择IN加载的注意点：

- 该策略最多为每次查询发出500个主键值的SELECT,因为主键值被渲染为SQL语句中的一个大型IN表达式。一些数据库（比如Oracle）有IN表达式大小的硬限制，而作为SQL字符串的长度不可以任意大。 

- 对于具有复合主键的映射，“selectin”无法使用IN的类似语法，而必须使用“元组”形式的IN，它看起来像：WHERE (table.column_a，table.column_b) IN ((？，？)，（？，？），（？，？）)。当前，SQL Server不支持此语法，将至少需要SQLite版本3.15。 SQLAlchemy中没有特殊的逻辑来提前检查哪些平台支持此语法或不支持此语法；如果针对不支持平台运行，则数据库将立即返回错误。 SQLAlchemy之所以只运行这个SQL字符串只是因为，如果特定数据库开始支持这个语法，那就不需要对SQLAlchemy进行任何更改就可以正常工作（SQLite的情况就是如此）。

**子查询急切加载**

子查询加载在操作上类似于选择急切加载，但是由生成从原始语句导出并具有更复杂查询结构的SELECT语句。使用子查询急切加载是通过在 :paramref:`_orm.relationship.lazy` 中提供“子查询”参数或使用 :func:`.subqueryload` 加载程序选项来完成的。

在使用子查询急切加载时，它们不会用于 :meth:`_sql.Select.limit` 或 :meth:`_sql.Select.offset` 之类的限制操作，因为它们具有独立的生成SELECT和应用限制的方法。在查询中使用了selectin load或其他涉及加载具有限制操作的选项时，只需简单地尝试使用 :func:`.subqueryload` 选项以避免出现限制带来的性能和正确性问题。例如： 
``
stmt = select(MyClass).options(subqueryload(MyClass.items))
``

在使用多级急切加载时，它还会导致更多的性能/复杂性问题，这是因为子查询将被重复嵌套。

使用子查询加载时，还需要考虑以下事项：

- “子查询”加载发出的 SELECT 语句非常复杂。特别是原始SELECT具有较大的复杂性，包含很多表JOIN语法。

- “子查询”加载在运行时可以通过检查查询输出来发现其正确性。 更准确地说， :func:`.subqueryload` 输出的SQL应在它们正在加载的查询的SELECT语句之后包含附加输出以验证其正确性。 如果出现不匹配或其它协变，将快速或暂时地禁止使用 subqueryload 直到问题修复。

- “子查询”加载在处理来自多级关系操作（例如，级联的集合操作）的异常情况方面可能会受到更明显的限制。

- 在尝试使用带有复合主键的实体进行子查询加载时，SQLAlchemy会在子查询中使用多个子表达式，因为只有一部分平台支持使用元组进行多数据写入操作。

下面是一个具体的示例，我们可以仅在 ``User.addresses`` 集合中使用显式 JOIN/语句来急切地加载特定地址，通过在查询中过滤加入的数据来实现；然后使用 :func:`_orm.contains_eager` 路由返回结果:

```
stmt = (
    select(User)
    .join(User.addresses)
    .filter(Address.email_address.like("%@aol.com"))
    .options(contains_eager(User.addresses))
    .execution_options(populate_existing=True)
)
```

上面的查询仅会加载至少包含 ``'aol.com'`` 子字符串的 `` Address`` 对象的 ``User`` 对象； ``User.addresses`` 集合仅会包含 **仅** 这些 ``Address`` 条目，而不包括任何与集合相关的其他 ``Address`` 条目。

值得注意的是，使用 :func:`_orm.contains_eager` 加载的自定义集合在下一次加载时不会被 "粘滞上去"；也就是说，如果该集合已重新加载，它将会使用其通常的默认内容重新加载。如果对象过期，可使用 SQLAlchemy ORM 自带的 :func:`_orm.populate_existing` 方法重新加载已经存在的集合。所有在内存中的已加载的对象均由 SQLAlchemy ORM 构造的一个 :term:`identity map` 保存。因此，使用 :func:`_orm.contains_eager` 定义了新方式加载集合后，通常最好使用上文中的方法来重新加载集合，以便用新数据刷新内存中的对象。请确保在使用它之前刷新所有数据，使用其默认行为的 :class:`_orm.Session` 肯定够用。