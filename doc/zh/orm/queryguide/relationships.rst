关系加载技术
===============================

.. admonition:: 关于本文档

    本节介绍了在查询中如何加载相关对象的详细信息。读者应该熟悉
      :ref:`relationship_config_toplevel`  和基本用法。

    这里的大多数示例假定 "User/Address" 映射设置类似于 :doc:`selects
    的设置 <_plain_setup>`。

SQLAlchemy 中的一个重要组成部分是在查询相关对象时提供了广泛的控制方式。所谓“相关对象”是指在映射器上使用
  :func:`_orm.relationship`  配置的集合或标量关联。可以在映射器构造时使用
  :func:`_orm.relationship`  函数中的  :paramref:` _orm.relationship.lazy` 
参数，或使用   :class:`_sql.Select`  构造函数的 **ORM loader 选项** 来配置此行为。

关系的加载分为三种类型：**惰性**（lazy）加载，**急切**（eager）加载和**未**（no）加载。惰性加载是指从查询返回对象时首先
不加载相关对象，当在特定对象上第一次访问给定的集合或引用时，会发出另一个 SELECT 语句，以便加载所请求的集合。急切加载是
指从查询返回已加载的相关集合或标量引用。ORM 通过在 SELECT 语句之后发出额外的 SELECT 语句以一次加载集合或标量引用，或
通过向 JOIN 结构添加行来实现这一点。

"未"加载指禁用给定关系的加载，即该属性为空且永远不被加载，或者在访问它时引发错误，以防止不需要的惰性加载。

关系加载类型的概述
--------------------------------------

关系加载的主要形式包括：

* **惰性加载** - 可通过 ``lazy='select'`` 或   :func:`.lazyload`  选项获得，这种加载方式在属性访问时会发出一个 SELECT
  语句，以每次惰性加载单个对象的相关引用。　惰性加载是所有不另外明确指示
   :paramref:`_orm.relationship.lazy`  选项的   :func:` _orm.relationship`  构造函数的 **默认加载方式**。有关惰性加载的详细信息
  请参见   :ref:`lazy_loading` 。

* **select IN 加载** - 可通过 ``lazy='selectin'`` 或   :func:`.selectinload`  选项获得，该加载方式发出第二个（或更多）SELECT
  语句，将父对象的主键标识符组装成 IN 子句，因此通过主键同时加载所有相关集合 / 标量引用的成员。关于　select　IN 加载的详细信息
  请参见   :ref:`selectin_eager_loading` 。

* **连接加载** - 可通过 ``lazy='joined'`` 或   :func:`_orm.joinedload`  选项获得，此加载方式对指定的 SELECT 语句应用 JOIN
  ，以便在同一结果集中加载相关行。连接急切加载的详细信息请参阅   :ref:`joined_eager_loading` 。

* **极限加载** - 可通过 ``lazy='raise'``, ``lazy='raise_on_sql'``,
  或   :func:`.raiseload`  选项获得，该加载方式在通常会发生惰性加载的同时触发，但它会引发 ORM 提示以防止应用程序进行不必要的惰性
  加载。关于提高加载的概述请参阅：　  :ref:`prevent_lazy_with_raiseload` 。

* **子查询加载** - 可通过 ``lazy='subquery'`` 或   :func:`.subqueryload`  选项获得，此加载方式发出第二个 SELECT 语句，
  该语句在子查询中将原始查询重新表述，然后将该子查询连接到要加载的相关表以一次加载所有相关集合 / 标量引用的成员。有关子
  查询急切加载的详细信息，请参见   :ref:`subquery_eager_loading` 。

* **只写加载** - 可通过 ``lazy='write_only'`` 或通过使用   :class:`_orm.WriteOnlyMapped`  注释来注释   :class:` _orm.Relationship` 
  对象的左侧获得。该仅限集合加载方式生成一种替代的属性标记，从不隐式从数据库加载记录，而仅允许  :meth:`.WriteOnlyCollection.add` ,
   :meth:`.WriteOnlyCollection.add_all`  和  :meth:` .WriteOnlyCollection.remove`  方法。通过调用使用  :meth:`.WriteOnlyCollection.select` 
  方法构造的 SELECT 语句来查询集合。有关只写加载的讨论，请参见   :ref:`write_only_relationship` 。

* **动态加载** - 可通过 ``lazy='dynamic'`` 或通过使用   :class:`_orm.DynamicMapped`  注释来注释   :class:` _orm.Relationship` 
  对象的左侧。这是一个遗留的仅限集合加载方式。配置映射时的加载策略
------------------------------

可以在映射时配置特定关系的加载策略，使其在加载特定映射类型对象时发挥作用，
即在没有任何修改的查询级别选项的情况下。这是通过将  :paramref:`_orm.relationship.lazy` 
参数传递给   :func:`_orm.relationship`  来配置的。该参数的常用值包括` `select``、
``selectin`` 和 ``joined``。

下面的示例示例了   :ref:`relationship_patterns_o2m`  中的关系实例，该示例配置了
``Parent.children`` 关系，在发出针对 ``Parent`` 对象的 SELECT 语句时使用
  :ref:`selectin_eager_loading` ：

    from typing import List

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Parent(Base):
        __tablename__ = "parent"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(lazy="selectin")


    class Child(Base):
        __tablename__ = "child"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))


在上述示例中，每当加载 ``Parent`` 对象集合时，每个 ``Parent`` 对象都将使用
``"selectin"`` 加载器策略填充其 ``children`` 集合。

  :paramref:`_orm.relationship.lazy`   参数的默认值为` `"select"``，即实现
  :ref:`lazy_loading` 。

使用加载器选项加载关系
----------------------------------------

另一种可能更常见的配置加载策略的方法是对特定属性在每个查询中使用
  :meth:`_sql.Select.options`   方法进行设置。使用加载器选项可以非常详细地控制
关系加载，最常用的选项包括   :func:`_orm.joinedload` 、  :func:` _orm.selectinload` 
和   :func:`_orm.lazyload` 。这些选项接受指向应针对的特定类/属性的绑定属性类引用。

下面的示例集中介绍：

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    # 将子项设置为懒加载
    stmt = select(Parent).options(lazyload(Parent.children))

    from sqlalchemy.orm import joinedload

    # 将子项配置为使用 join 技术进行急加载
    stmt = select(Parent).options(joinedload(Parent.children))

也可以使用 **方法链接** 进一步的指定如何在更深一层次上进行加载：

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload

    stmt = select(Parent).options(
        joinedload(Parent.children).subqueryload(Child.subelements)
    )

可以将链接加载器选项应用于“懒”加载的集合。这意味着延迟加载在访问时，将采用指定的选项：

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))

在上述示例中，查询将返回未加载 ``children`` 集合的 ``Parent`` 对象。
当首次访问某个特定 ``Parent`` 对象的 ``children`` 集合时，将延迟加载相关对象，
但对于 ``children`` 的每个成员都将同时应用快速的 ``subelements`` 加载。


加载器选项添加筛选条件
________________________

用于指示加载器选项的关系属性具有向创建的 join 的 ON 子句中添加附加筛选条件或
涉及 WHERE 条件的能力，这取决于加载器策略。这可以通过使用  :meth:`PropComparator.and_` 
方法来实现，传递一个选项，以便仅限于给定的过滤条件在加载结果时生效：

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = select(A).options(lazyload(A.bs.and_(B.id > 5)))

使用限制条件时，如果已加载特定的集合，则不会刷新该集合；为确保新条件生效，
应用   :ref:`orm_queryguide_populate_existing`  执行选项：


    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = (
        select(A)
        .options(lazyload(A.bs.and_(B.id > 5)))
        .execution_options(populate_existing=True)
    )

要在查询中的实体所有出现的地方添加筛选条件，请参见   :func:`_orm.with_loader_criteria`  函数。

.. versionadded:: 1.4

.. _orm_queryguide_relationship_sub_options:使用 Load.options() 指定子选项
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
使用方法链接，可以明确说明路径中每个链接的加载器类型。
要沿着路径导航而不改变特定属性的现有加载器类型，可以使用   :func:`.defaultload`  方法/函数：

    from sqlalchemy import select
    from sqlalchemy.orm import defaultload

    stmt = select(A).options(defaultload(A.atob).joinedload(B.btoc))

类似的方法可以用于一次指定多个子选项，使用  :meth:`_orm.Load.options`  方法：
    
    from sqlalchemy import select
    from sqlalchemy.orm import defaultload
    from sqlalchemy.orm import joinedload

    stmt = select(A).options(
        defaultload(A.atob).options(joinedload(B.btoc), joinedload(B.btod))
    )

.. seealso::

      :ref:`orm_queryguide_load_only_related`  - 演示了结合关系和面向列的加载器选项的示例。

.. note::  应用于对象的惰性加载集合的加载器选项是特定对象实例的**"粘性"**，这意味着它们将持续存在于该特定对象加载的集合中，只要它在内存中存在。
   例如，给定上面的例子：

      stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))

   如果通过上述查询加载的特定 ``Parent`` 对象的 ``children`` 集合过期了（例如当事务被提交或回滚时，或使用  :meth:`.Session.expire_all`  时），当再次访问 ` `Parent.children`` 集合以重新加载时， ``Child.subelements`` 集合将再次使用子查询急加载。即使从指定了不同的一组选项的后续查询中访问了上述 ``Parent`` 对象，情况也是如此。要在不清除对象并重新加载的情况下更改现有对象上的选项，必须同时使用   :ref:`orm_queryguide_populate_existing`  执行选项显式地设置它们：

      # 更改已加载的 Parent 对象上的选项
      stmt = (
          select(Parent)
          .execution_options(populate_existing=True)
          .options(lazyload(Parent.children).lazyload(Child.subelements))
          .all()
      )

   如果加载的对象完全从   :class:`.Session`  中清除，例如由于垃圾收集或使用  :meth:` .Session.expunge_all` ，则“粘性”选项也会消失，如果重新加载，则新创建的对象将使用新选项。

   未来 SQLAlchemy 的版本可能会添加更多操作已加载对象上的加载器选项的替代方法。

.. _lazy_loading:

惰性加载
-----------------------

默认情况下，所有对象之间的关系都是**惰性加载**的。与   :func:`_orm.relationship`  相关联的标量或集合属性包含一个触发器，
该触发器首次访问属性时触发。此触发器通常在访问点发出 SQL 调用，以加载相关对象或对象：

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

唯一不发出 SQL 的情况是在一个简单的多对一关系中，当只使用主键即可识别相关对象并且该对象已经存在于当前的   :class:`.Session`  时。因此，
尽管对相关集合进行惰性加载可能很费时，但在加载大量对象并对一组相对较小的可能目标对象进行简单多对一加载的情况下，
惰性加载可能能够在不发出与父对象数量相同的 SELECT 语句的情况下引用这些对象。

"load upon attribute access" 的默认行为被称为“lazy”或“select”加载 - 名称“select”是因为在首次访问属性时通常会发出“SELECT”的语句。

可以使用   :func:`.lazyload`  加载器选项为通常以某种方式配置的给定属性启用惰性加载：

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    # 强制一个属性的惰性加载，该属性通常以其他方式加载
    stmt = select(User).options(lazyload(User.addresses))

.. _prevent_lazy_with_raiseload:

使用 raiseload 防止意外惰性加载
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :func:`.lazyload`  策略产生了一种最常见的 object relational mapping (对象关系映射) 问题之一；即 N+1 问题，
该问题表明，对于任何 N 个加载的对象，访问它们的惰性加载属性意味着将发出 N+1 个 SELECT 语句。
在 SQLAlchemy 中，解决 N+1 问题的常规方法是使用其功能非常强大的急加载系统。但是，急加载需要事先通过   :class:`_sql.Select` 
指定要加载的属性。对于可能访问其他未极速加载的属性的代码，不希望进行惰性加载的问题，可以使用   :func:`.raiseload`  策略解决；
这个加载器策略将惰性加载的行为替换为引发信息型错误：从上面的查询中加载的 ``User`` 对象将不会加载 ``.addresses`` 集合； 如果稍后的某些代码尝试访问此属性，则会引发 ORM 异常。

可以使用所谓的“通配符”指示符与   :func:`.raiseload`  一起使用，以指示所有关系都应使用此策略。例如，要设置仅将一个属性设置为及早加载，而将其余所有属性都设置为 ` `raise``：

.. sourcecode:: python

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload
    from sqlalchemy.orm import raiseload

    stmt = select(Order).options(joinedload(Order.items), raiseload("*"))

上述通配符将适用于**所有**不仅在 ``Order`` 上的关系，而且也适用于 ``Item`` 对象上的所有关系。要为仅 ``Order`` 对象设置   :func:`.raiseload` ，请使用   :class:` _orm.Load`  带有完整路径的指定：

.. sourcecode:: python

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload
    from sqlalchemy.orm import Load

    stmt = select(Order).options(joinedload(Order.items), Load(Order).raiseload("*"))

反之，要将提高设置为仅用于 ``Item`` 对象：

.. sourcecode:: python

    stmt = select(Order).options(joinedload(Order.items).raiseload("*"))

  :func:`.raiseload`  选项仅适用于关系属性。对于定向列属性，  :func:` .defer`  选项支持工作相同的  :paramref:`.orm.defer.raiseload`  选项。

.. tip:: “raiseload” 策略 **不适用** 在  :term:`unit of work`  刷新过程中。这意味着如果  :meth:` _orm.Session.flush`  过程需要加载集合以完成其工作，则会在绕过任何   :func:`_orm.raiseload`  指令的情况下执行此操作。

.. seealso::

      :ref:`wildcard_loader_strategies` 

      :ref:`orm_queryguide_deferred_raiseload` 

.. _joined_eager_loading:

连接的及早加载
--------------------

连接的及早加载是 SQLAlchemy ORM 中包含的最旧式的及早加载样式。它的工作原理是将连接 (默认情况下为 LEFT OUTER join) 连接到发出的 SELECT 语句，并将目标标量/集合从与父项相同的结果集中填充。

在映射级别，它看起来像是这样的：

.. sourcecode:: python

    class Address(Base):
        # ...

        user: Mapped[User] = relationship(lazy="joined")

通常将连接的及早加载作为查询的选项应用，而不是作为映射的默认加载选项，特别是在处理集合而不是 many-to-one 引用时。可以使用   :func:`_orm.joinedload`  加载程序选项来实现这一点：

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

.. tip:: 当引用到 one-to-many 或者 many-to-many 集合时，并且使用   :func:`_orm.joinedload`  时，必须应用  :meth:` _result.Result.unique`  方法到返回结果上，它会通过主键将传入的行进行唯一化，否则就会由 join 进行相应的放大。如果不这样做，ORM 就会引发错误。
在现代 SQLAlchemy 中，这不是自动的，因为它会改变结果集的行为，返回少于语句中返回的 ORM 对象数量的结果行数。因此，SQLAlchemy 使得使用  :meth:`_result.Result.unique`  显式，所以在主键上进行唯一处理的对象被返回。

默认发出的 JOIN 是 LEFT OUTER JOIN，以允许不引用相关行的前导对象。对于保证具有元素的属性，例如 many-to-one 引用到引用外键不为 NULL 的相关对象的引用，可以通过使用内部 join 使查询更有效率；在映射级别通过  :paramref:`_orm.relationship.innerjoin`  标志实现：

.. sourcecode:: python

    class Address(Base):
        # ...

        user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
        user: Mapped[User] = relationship(lazy="joined", innerjoin=True)

在查询选项级别通过  :paramref:`_orm.joinedload.innerjoin`  标志实现：

.. sourcecode:: python

    from sqlalchemy import select
    from sqlalchemy.orm import joinedload

    stmt = select(Address).options(joinedload(Address.user, innerjoin=True))

当应用于包括 OUTER JOIN 的链时，JOIN 将右嵌套自身：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import joinedload
    >>> stmt = select(User).options(joinedload(User.orders, Order.items))
    >>> result = session.exec(stmt).scalars().all().. tip:: 如果在发出SELECT语句时使用了数据库行锁定技术，这意味着  :meth:`_sql.Select.with_for_update`  方法正在用于发出SELECT..FOR UPDATE，则取决于使用的后端的行为，连接表也可能被锁定。不建议同时使用连接式预加载和SELECT..FOR UPDATE，因为这个原因。

.. NOTE: 哇，这一部分。特别长。它不是参考资料，而是概念性的。

.. _zen_of_eager_loading:

连接预加载的哲学
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

由于连接预加载似乎与使用  :meth:`_sql.Select.join`  方法具有许多相似之处，因此经常产生混淆，不清楚何时以及如何使用它。关键在于理解以下区别：  :meth:` _sql.Select.join`  用于更改查询的结果，而 :func:`_orm.joinedload` 则极力**不要**更改查询结果，而是隐藏渲染连接的效果，只允许相关对象存在。

后代策略背后的哲学是，任何一组加载方案都可以应用于特定查询，并且*结果不会改变*——只有发出SQL语句以完全加载相关对象和集合所需的数量发生了变化。特定查询可能是使用所有惰性加载。在上下文中使用它之后，可能发现特定属性或集合始终被访问，更改这些内容的加载程序策略将更有效。该策略可以在不对查询进行其他修改的情况下更改，结果将保持相同，但会发出更少的SQL语句。理论上（实际上基本上是如此），您可以对 :class:`_sql.Select` 执行的所有操作都不能使其根据加载程序策略的变化而加载不同的主要或相关对象集。

特别是，为了不以任何方式影响返回的实体行，  :func:`joinedload` ` users``到``addresses``的左外连接，但是添加到``Address.email_address`` 的``ORDER BY``是无效的-``Address``实体没有在查询中命名：

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

上面的``ORDER BY addresses.email_address``是无效的，因为``addresses``不在FROM列表中。加载``User``记录并按电子邮件地址排序的正确方法是使用  :meth:`_sql.Select.join`  ：

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

上面的语句当然不同于以前的语句，因为``addresses``的列根本没有包含在结果中。我们可以再次添加  :func:`_orm.joinedload` ，使得有两个连接——一个是我们正在其中排序的连接，另一个是匿名使用的连接，以加载` `User.addresses``集合的内容：

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
        users.nickname AS users_nickname上面我们看到了我们使用  :meth:`_sql.Select.join`  是为了提供我们想在后续查询条件中使用的 JOIN 子句，而我们使用   :func:` _orm.joinedload`  只关心加在每个 ``User`` 对象的 ``User.addresses`` 集合上的加载。在这种情况下，这两个 JOIN 看起来可能是冗余的——它们就是冗余的。如果我们想要仅使用一个 JOIN 用于集合加载以及排序，我们可以使用   :func:`.contains_eager`  选项，在下面的   :ref:` contains_eager`  中进行了描述。但是为了看到   :func:`joinedload`  做了什么，考虑我们是否在过滤特定的 ` `Address``：

.. sourcecode:: pycon+sql

    >>> stmt = (
    ...     select(User)
    ...     .join(User.addresses)
    ...     .options(joinedload(User.addresses))
    ...     .filter(User.name == "spongebob")
    ...     .filter(Address.email_address == "someaddress@foo.com")
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
    LEFT OUTER JOIN addresses AS addresses_1
        ON users.id = addresses_1.user_id
    WHERE users.name = ? AND addresses.email_address = ?
    ['spongebob', 'someaddress@foo.com']

上面，我们可以看到这两个 JOIN 所起的作用有很大的不同。其中一个JOIN将精确匹配一行，即 ``User`` 和 ``Address`` 的联接，其中 ``Address.email_address == 'someaddress@foo.com'``。另一个左外部 JOIN 将匹配 *所有* 与 ``User`` 相关的 ``Address`` 行，并且仅用于填充 ``User`` 对象中返回的 ``User.addresses`` 集合。

通过将   :func:`_orm.joinedload`  的使用方式更改为另一种加载样式，我们可以完全独立于用于检索所需 ` `User`` 行的 SQL 更改如何加载集合。下面我们将   :func:`_orm.joinedload`  改为   :func:` .selectinload` ：

.. sourcecode:: pycon+sql

    >>> stmt = (
    ...     select(User)
    ...     .join(User.addresses)
    ...     .options(selectinload(User.addresses))
    ...     .filter(User.name == "spongebob")
    ...     .filter(Address.email_address == "someaddress@foo.com")
    ... )
    >>> result = session.scalars(stmt).all()
    {execsql}SELECT
        users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    JOIN addresses ON users.id = addresses.user_id
    WHERE
        users.name = ?
        AND addresses.email_address = ?
    ['spongebob', 'someaddress@foo.com']
    # ... selectinload() emits a SELECT in order
    # to load all address records ...

在使用连接的 eager 加载时，如果查询包含影响到连接外部返回的行的修改器，例如使用 DISTINCT、LIMIT、OFFSET 或等效的修改器时，完成的语句首先包装在一个子查询中，并将专门用于连接的 eager 加载应用于子查询。SQLAlchemy 的连接的 eager 加载走出了不一般的距离，甚至还走了十英里之远，以确保它绝不会影响查询的最终结果，仅影响加载集合和相关对象的方式，无论查询的格式如何。

.. seealso::

      :ref:`contains_eager` ——使用   :func:` .contains_eager` 

.. _selectin_eager_loading:

选择 IN 的加载
-----------------

在大多数情况下，选择 IN 的加载是最简单和高效的急切地加载对象集合的方法。唯一不能使用 selectin 急切加载的情形是，当模型使用复合主键，并且后端数据库不支持包含元组的 IN 时，目前包括 SQL Server。

使用  :paramref:`_orm.relationship.lazy`  的 ` `"selectin"`` 参数，或使用   :func:`.selectinload`  加载器选项，提供了“选择 IN”急切加载。这种加载方式发出一个 SELECT，该 SELECT 引用父对象的主键值，或者在一对多关系的情况下，引用子对象的主键值，内部包含 IN 子句，以便加载相关联的对象：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import selectinload
    >>> stmt = (
    ...     select(User)
    ...     .options(selectinload(User.addresses))
    ...     .filter(or_(User.name == "spongebob", User.name == "ed"))
    ... )
    >>> result = session.scalars(stmt).all()
    {execsql}SELECT
        users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    WHERE users.name = ? OR users.name = ?
    ('spongebob', 'ed')
    SELECT
        addresses.id AS addresses_id,
        addresses.email_address AS addresses_email_address,
        addresses.user_id AS addresses_user_id
    FROM addresses
    WHERE addresses.user_id IN (?, ?)
    (5, 7)上面的第二个 SELECT 引用了 ``addresses.user_id IN (5, 7)``，其中的“5”和“7”是先前加载的两个 ``User`` 对象的主键值；当一个对象批量完全加载后，它们的主键值会注入到第二个 SELECT 的 ``IN`` 子句中。由于 ``User`` 和 ``Address`` 之间的关系具有简单的主键联接条件并提供了从 ``Address.user_id`` 可以派生出 ``User`` 的主键值，因此该语句没有任何联接或子查询。

对于简单的一对多加载，也不需要使用 JOIN，而是使用父对象的外键值：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import selectinload
    >>> stmt = select(Address).options(selectinload(Address.user))
    >>> result = session.scalars(stmt).all()
    {execsql}SELECT
        addresses.id AS addresses_id,
        addresses.email_address AS addresses_email_address,
        addresses.user_id AS addresses_user_id
        FROM addresses
    SELECT
        users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    WHERE users.id IN (?, ?)
    (1, 2)

.. tip::

    这里所说的“简单”是指  :paramref:`_orm.relationship.primaryjoin`  条件在“一”侧的主键和“多”侧的直接外键之间表达了等式比较，没有任何其他条件。

选择 IN 加载也支持多对多关系，在这种情况下它将跨越所有三个表进行 JOIN，以将一侧的行匹配到另一侧。

有关这种类型的加载需要注意以下事项：

* 该策略一次会针对多达 500 个父主键值发出 SELECT，因为这些主键值在 SQL 语句中被渲染成一个大的 IN 表达式。一些数据库（如 Oracle）对 IN 表达式的大小有一个硬性限制，而且总体上 SQL 字符串的大小不应该是任意大小的。

* 由于“selectin”加载依赖于 IN，对于具有复合主键的映射，它必须使用 IN 的“元组”形式，它的形式看起来像 ``WHERE (table.column_a, table.column_b) IN ((?, ?), (?, ?), (?, ?))``。目前这种语法在 SQL Server 上不被支持，在 SQLite 上需要至少 3.15 版本。 SQLAlchemy 中没有特殊的逻辑来提前检查这些平台是否支持此语法；如果针对不支持此语法的平台运行，数据库将立即返回错误。SQLAlchemy 只是直接运行 SQL 来失败的一个好处是，如果某个特定的数据库确实开始支持这种语法，它将在不需要对 SQLAlchemy 进行任何更改的情况下工作（这就是 SQLite 的情况）。

.. _subquery_eager_loading:

子查询急加载
----------------------

.. legacy::   :func:`_orm.subqueryload`  急加载器在大多数情况下都是遗留的，已被   :func:` _orm.selectinload`  策略代替，后者具有更简单的设计，更灵活的功能，例如   :ref:`分批 <orm_queryguide_yield_per>`  和在大多数情况下发出更高效的 SQL 语句。由于   :func:` _orm.subqueryload`  依赖于重新解释原始 SELECT 语句，因此当给定非常复杂的源查询时，它可能无法有效地工作。

   对于具有复合主键的对象的急加载集合，特定情况下   :func:`_orm.subqueryload`  仍可能有用，特别是在仍然不支持“元组 IN”语法的 Microsoft SQL Server 后端的情况下。

子查询加载的操作与 selectin 急加载类似，不过所发出的 SELECT 语句是从原始语句派生出来的，并且具有比 selectin 急加载更复杂的查询结构。

子查询急加载是使用  :paramref:`_orm.relationship.lazy`  的 ` `"subquery"`` 参数或使用   :func:`.subqueryload`  加载器选项来提供的。

子查询急加载的操作是为了对要加载的每个关系发出第二个 SELECT 语句，一次跨所有结果对象。该 SELECT 语句引用原始 SELECT 语句，在一个子查询中包装起来，以便我们检索返回的主对象的相同主键列表，然后将其链接到所有要加载的集合成员的总和：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import subqueryload
    >>> stmt = select(User).options(subqueryload(User.addresses)).filter_by(name="spongebob")
    >>> results = session.scalars(stmt).all()
    {execsql}SELECT
        users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
    FROM users
    WHERE users.name = ?
    ('spongebob',)
    SELECT
        addresses.id AS addresses_id,
        addresses.email_address AS addresses_email_address,
        addresses.user_id AS addresses_user_id,
        anon_1.users_id AS anon_1_users_id
    FROM (
        SELECT users.id AS users_id
        FROM users
        WHERE users.name = ?) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id, addresses.id
    ('spongebob',)


有关这种类型的加载需要注意以下事项：

* “子查询”加载策略所发出的 SELECT 语句与原始的 SELECT 语句不同，因此对于较大的结果集，这可能意味着大幅增加的总 SQL 执行时间和大量的网络数据流量。

* 在使用“子查询”加载程序时，无法使用   :ref:`挂起加载 <orm_queryguide_yield_per>` ，因为在这个过程中我们必须立即将所有集合加载到内存中才能继续进行。

* 因为 SELECT 语句通过“包装”原始查询派生，所以在使用   :func:`_orm.subqueryload`  加载时更改原始查询多少，也就是 SELECT 列表，仅会反映于加载的对象集合之上，而不是原始查询中的满足条件的行数。

* 同样地，由于此特定的 SELECT 语句实际上是在另一个内部查询中执行的，因此无法使用   :func:`.options`  中的大多数 ORM 上下文选项，例如   :func:` .with_for_update`  或   :func:`.add_columns` 。何种类型的加载方法？

选择何种类型的加载方法通常取决于优化SQL执行次数、发射的SQL的复杂性和获取的数据量之间的平衡。

对于“one-to-many”和“many-to-many collection”类型

   :func:`_orm.selectinload` 是一般最好的加载策略。它会产生一个额外的SELECT，使用尽可能少的表，使原始声明毫不受影响，并且在任何种类的源查询中都是最灵活的。它的唯一主要限制是当使用一个带有组合主键的表并且使用的后端不支持“tuple IN”时，目前包括SQL Server和旧版SQLite；所有其他包括的后端都支持它。

对于“many-to-one”类型

    :func:`_orm.joinedload`  获取对象，如果相关对象已存在，则不会发出任何SQL。

多态贪婪加载

支持根据每个贪婪加载基础规范的多态选项。请参阅   :ref:`eagerloading_polymorphic_subtypes`  部分，其中包含使用   :func:` _orm.with_polymorphic`  函数的  :meth:`.PropComparator.of_type`  方法示例。

通配符加载策略

每个   :func:`_orm.joinedload` ，  :func:` .subqueryload` ，  :func:`.lazyload` ，  :func:` .selectinload` ，  :func:`.noload`  和   :func:` .raiseload`  默认加载方式，影响所有   :func:`_orm.relationship`  的映射的属性，如果在语句中没有另外指定。通过将字符串 ` `'*'`` 作为这些选项中任何一个的参数进行传递即可使用此功能::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload

    stmt = select(MyClass).options(lazyload("*"))

在上述示例中， ``lazyload('*')`` 选项将取代该查询中所有   :func:`_orm.relationship`  构造的 ` `lazy`` 设置，但不包括使用 ``lazy='write_only'`` 或 ``lazy='dynamic'`` 的关系。如果一些关系指定了``lazy='joined'`` 或 ``lazy='selectin'``，例如，使用 ``lazyload('*')`` 将单方面导致所有这些关系使用 ``'select'`` 加载，例如在访问每个属性时发出SELECT语句.

该选项不会取代查询中声明的加载程序选项，例如   :func:`.joinedload` ，  :func:` .selectinload`  等。下面的查询仍将使用joined加载来处理“widget”关系::

    from sqlalchemy import select
    from sqlalchemy.orm import lazyload
    from sqlalchemy.orm import joinedload

    stmt = select(MyClass).options(lazyload("*"), joinedload(MyClass.widget))

在上述声明   :func:`.joinedload`  的指令中，无论   :func:` .lazyload` `选项位于该指令之前还是之后，都将执行.如果传入了多个包含 ``"*"`` 的选项，则最后一个选项将生效。

.. _orm_queryguide_relationship_per_entity_wildcard:

每个实体通配符加载策略
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

通配符加载策略的一个变种是能够在每个实体上设置策略。例如，如果查询“User”和“Address”，可以指示“Address”上的所有关系都使用惰性加载，同时让“User”的加载策略保持不变，方法是先应用   :class:`_orm.Load`  对象，然后将 ` `*`` 指定为链式选项::

    from sqlalchemy import select
    from sqlalchemy.orm import Load

    stmt = select(User, Address).options(Load(Address).lazyload("*"))

上面，``Address`` 上的所有关系都将设为惰性加载。

.. _joinedload_and_join:

.. _contains_eager:

将显式连接/语句路由到急加载集合中
-------------------------------------------------- -------------------

   :func:`_orm.joinedload()`  的行为是自动创建连接，使用匿名别名作为目标，其结果被路由到集合和
已加载对象的标量引用。通常情况下，查询已包含表示特定集合或标量引用的必要连接，并且连接加入了要求加入的集合/引用，但已加入的加载特性是冗余的。
然而你仍希望填充集合/引用。

为此，SQLAlchemy 提供了   :func:`_orm.contains_eager`  选项。使用方法与   :func:` _orm.joinedload()`  选项相同，除了它默认假定   :class:`_sql.Select`  对象会显式包含适当的联接，通常使用像  :meth:` _sql.Select.join`  这样的方法。下面，我们在 ``User`` 和 ``Address`` 之间指定了一个连接，并进一步将其作为急加载 ``User.addresses`` 的基础来建立：：

    from sqlalchemy.orm import contains_eager

    stmt = select(User).join(User.addresses).options(contains_eager(User.addresses))

如果语句的“急”部分是“别名化”的，则路径应该使用  :meth:`.PropComparator.of_type`  指定，允许传递特定   :func:` _orm.aliased`  建构：：

.. sourcecode:: python+sql

    # 使用一个 Address 实体的别名
    adalias = aliased(Address)

    # 构造一个期望“地址”结果的语句

    stmt = (
        select(User)
        .outerjoin(User.addresses.of_type(adalias))
        .options(contains_eager(User.addresses.of_type(adalias)))
    )

    # 正常获取结果
    r = session.scalars(stmt).unique().all()
    {execsql}SELECT
        users.user_id AS users_user_id,
        users.user_name AS users_user_name,
        adalias.address_id AS adalias_address_id,
        adalias.user_id AS adalias_user_id,
        adalias.email_address AS adalias_email_address,
        (...other columns...)
    FROM users
    LEFT OUTER JOIN email_addresses AS email_addresses_1
    ON users.user_id = email_addresses_1.user_id

作为参数传递给  :func:`.contains_eager` 的路径需要是从起始实体开始的完整路径。例如，如果我们正在加载 ` `Users->orders->Order->items->Item``，则选项应该这样使用：：

    stmt = select(User).options(contains_eager(User.orders).contains_eager(Order.items))

使用 contains_eager() 加载自定义过滤的集合结果
------------------------------------------------------ ----------

当我们使用   :func:`.contains_eager`  时，*我们*自己构造将用于填充集合的 SQL。因此，自然地我们可以选择通过加载一小部分元素来 **修改** 集合的目标值，通过编写 SQL 来过滤连接的数据，并将其路由到   :func:` _orm.contains_eager` 。同时使用   :ref:`orm_queryguide_populate_existing`  来确保任何已加载的集合被覆盖：：

    stmt = (
        select(User)
        .join(User.addresses)
        .filter(Address.email_address.like("%@aol.com"))
        .options(contains_eager(User.addresses))
        .execution_options(populate_existing=True)
    )

以上查询将仅加载包含至少一个包含子字符串 ``'aol.com'`` 的地址字段的 ``User`` 对象； ``User.addresses`` 集合仅包含这些 ``Address`` 条目，而不包含与集合关联的所有其他 ``Address`` 条目。

.. tip:: 在所有情况下，SQLAlchemy ORM 都不会覆盖已加载的属性和集合，除非告诉它这样做。
当一个`.ORM`查询正在使用`identity map`时，通常情况下该查询返回的对象实际上已经存在并且已经在内存中加载。因此，在使用`_orm.contains_eager`以另一种方式填充集合时，通常最好使用如上所示的`orm_queryguide_populate_existing`。这样，一个已经加载的集合将用新数据刷新。`populate_existing`选项将重置已经存在的**所有**属性，包括挂起的更改，因此在使用它之前请确保所有数据已被清除。使用`_orm.Session`及其默认行为`autoflush`就足够了。

.. note:: 我们使用`_orm.contains_eager`加载的自定义集合不是“固定不变的”；也就是说，下一次加载该集合时，它将以其通常的默认内容加载。如果对象过期，该集合可能会被重新加载，这发生在使用: meth:`.Session.commit`，: meth:`.Session.rollback`方法（假定默认会话设置），或使用`meth`：`.Session.expire_all`或`meth`：`.Session.expire`方法时。

.. seealso::

      :ref:`loader_option_criteria`  - modern API allowing WHERE criteria directly
    within any relationship loader option


Relationship Loader API
-----------------------

.. autofunction:: contains_eager

.. autofunction:: defaultload

.. autofunction:: immediateload

.. autofunction:: joinedload

.. autofunction:: lazyload

.. autoclass:: sqlalchemy.orm.Load
    :members:
    :inherited-members: Generative

.. autofunction:: noload

.. autofunction:: raiseload

.. autofunction:: selectinload

.. autofunction:: subqueryload