.. highlight :: pycon+sql

.. |prev| replace::  :doc:`orm_data_manipulation` 
.. |next| replace::  :doc:`further_reading` 

.. include :: tutorial_nav_include.rst

.. rst-class:: orm-header

.. _tutorial_orm_related_objects:

使用 ORM 相关对象
=================

在本节中，我们将介绍 ORM 的另一个重要概念，即 ORM 如何与引用其他对象的映射类交互。在本教程的   :ref:`tutorial_declaring_mapped_classes`  部分中，映射类的示例使用了一个叫做   :func:` _orm.relationship`  的构造。此构造定义了两个不同映射类之间或者从映射类到它本身的关联，后者被称为**自引用**关系。

为了描述   :func:`_orm.relationship`  的基本思想，首先让我们简要回顾映射的形式，省略了   :func:` _orm.mapped_column`  映射和其他指令：

.. sourcecode:: python

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship

    class User(Base):
        __tablename__ = "user_account"

        # ... mapped_column() 映射

        addresses: Mapped[List["Address"]] = relationship(back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        # ... mapped_column() 映射

        user: Mapped["User"] = relationship(back_populates="addresses")

以上， ``User`` 类现在有一个属性 ``User.addresses``， ``Address`` 类有一个属性 ``Address.user``。   :func:`_orm.relationship`  构造与   :class:` _orm.Mapped`  构造相结合指示类型行为，将用来检测与映射到 ``User`` 和 ``Address`` 类的  :class:`_schema.Table`  对象之间的表格关系。由于代表 ` `address`` 表的   :class:`_schema.Table`  对象具有引用 ` `user_account`` 表的   :class:`_schema.ForeignKeyConstraint` ，因此   :func:` _orm.relationship`  可以明确确定从 ``User`` 类到 ``Address`` 类存在一个一对多关系，即 ``User.addresses`` 关系。 ``user_account`` 表中的一个特定行可由 ``address`` 表中的多个行引用。

所有一对多关系自然对应另一方向的**多对一**关系，在本例中是通过 ``Address.user`` 标识的一个关系。前面配置在   :func:`_orm.relationship`  对象上的  :paramref:` _orm.relationship.back_populates`  参数，建立了互补对应的两个   :func:`_orm.relationship`  构造，我们将在下一部分看到它的作用。

保持和加载关系
--------------

我们可以开始说明   :func:`_orm.relationship`  对对象实例所做的操作。如果我们创建一个新的 ` `User`` 对象，我们可以注意到当访问 ``.addresses`` 元素时，有一个 Python 列表：

    >>> u1 = User(name="pkrabs", fullname="Pearl Krabs")
    >>> u1.addresses
    []

这个对象是一个特定于 SQLAlchemy 的 Python 版本的 ``list``，具有跟踪和响应其更改的能力。即使我们从未将其分配给对象，当我们访问该属性时，该集合也会自动出现。这类似于   :ref:`tutorial_inserting_orm`  中所观察到的行为，其中指出我们不明确为其分配值的基于列的属性也将自动显示为 ` `None```，而不是像 Python 的通常行为一样引发 ``AttributeError``。

由于 ``u1`` 对象仍然处于  :term:`瞬态` ，我们从 ` `u1.addresses`` 得到的 ``list`` 尚未被改变（即添加或扩展），因此它尚未与对象关联，但是随着我们对其进行更改，它将成为 ``User`` 对象状态的一部分。

该集合特定于 ``Address`` 类，这是唯一可能存储在其中的 Python 对象类型。我们可以使用 ``list.append()`` 方法添加一个 ``Address`` 对象：

  >>> a1 = Address(email_address="pearl.krabs@gmail.com")
  >>> u1.addresses.append(a1)

此时，``u1.addresses`` 集合按预期包含新的 ``Address`` 对象：

  >>> u1.addresses
  [Address(id=None, email_address='pearl.krabs@gmail.com')]

由于我们将 ``Address`` 对象与 ``u1`` 实例的 ``User.addresses`` 集合关联起来，另一个行为也发生了变化，即 ``User.addresses`` 关系与 ``Address.user`` 关系同步，从而我们不仅可以从 ``User`` 对象导航到 ``Address`` 对象，还可以从 ``Address`` 对象导航回“父”``User`` 对象：

  >>> a1.user
  User(id=None, name='pkrabs', fullname='Pearl Krabs')

通过使用  :paramref:`_orm.relationship.back_populates`  参数配置这些关系，我们确保了彼此之间的同步，这将在下面几节中继续讨论。使用` `back_populates``参数，两个   :func:`_orm.relationship`  之间可以反向關聯。该参数命名了另一个   :func:` _orm.relationship` ，该关系需要进行互补属性赋值 / 列清单变异。这同样适用于另一个方向，即如果我们创建另一个“Address”对象并将其分配给其“Address.user”属性，则该“Address”将成为该“User”对象上的“User.addresses“集合的一部分：

  >>> a2 = Address（email_address =“pearl@aol.com”，user = u1）
  >>> u1.addresses
  [Address（id = None，email_address ='pearl.krabs@gmail.com'），Address（id = None，email_address ='pearl@aol.com'）]

我们实际上在“Address”构造函数中使用了关键字参数“user”，其像任何其他映射到“Address”类上声明的映射属性一样得到接受。 这相当于在事后分配“Address.user”属性：

  # 等同于 a2 = Address（user = u1）
  >>> a2.user = u1


.. _tutorial_orm_cascades:

将对象级联到会话
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们现在有一个“User”和两个“Address”对象，它们在内存中以双向结构相关联，但正如之前在   :ref:`tutorial_inserting_orm`  中指出的那样，这些对象直到与   :class:` _orm.Session`  对象关联之前都是处于  :term:`transient`  状态的。

我们使用仍在进行的   :class:`_orm.Session` ，并注意到当我们将  :meth:` _orm.Session.add`  方法应用到主要的“User”对象时，相关的“Address”对象也被添加到同一个   :class:`_orm.Session`  中：

  >>> session.add（u1）
  >>> u1 in session
  True
  >>> a1 in session
  True
  >>> a2 in session
  True

以上行为，其中   :class:`_orm.Session`  收到了一个“User”对象，并跟随“User.addresses”关系以查找相关的“Address”对象，称为 **save-update cascade**，并在ORM参考文件的   :ref:` unitofwork_cascades`  中详细讨论。

这三个对象现在处于  :term:`pending`  状态； 这意味着它们已经准备好成为 INSERT 操作的主题，但尚未进行； 这三个对象都没有分配主键，而且“a1”和“a2”对象还具有称为“user_id”的属性，该属性引用具有指向“user_account.id”列的   :class:` _schema.ForeignKeyConstraint`  的   :class:`_schema.Column` ； 因为这些对象尚未与真正的数据库行相关联，因此这些列也是 ` `None`` ：

    >>> print（u1.id）
    None
    >>> print（a1.user_id）
    None

在这个阶段，我们可以看到单位操作过程提供的非常大的实用程序; 请回想一下，在   :ref:`tutorial_core_insert_values_clause`  部分中插入“user_account”和“address”表中的行时，使用了一些复杂的语法，以便自动关联 “address.user_id” 列与 “user_account” 行的那些列。此外，需要在“address”行之前插入“user_account”行的 INSERT，因为“address”中的行**依赖于“user_account”中的父行**以获得其“user_id”列中的值。

使用   :class:`_orm.Session`  时，所有这些繁琐的操作都会为我们处理，即使是最死心眼的 SQL 纯粹主义者也可以从 INSERT、UPDATE 和 DELETE 语句的自动化中受益。当我们  :meth:` _orm.Session.commit`  事务时，所有步骤都按正确的顺序调用，并且还将新生成的“user_account”行的主键适当地应用于“address.user_id”列：

.. sourcecode:: pycon+sql

  >>> session.commit（）
  {execsql}INSERT INTO user_account（name，fullname）VALUES（？，？）
  [...]（'pkrabs'，'Pearl Krabs'）
  INSERT INTO address（email_address，user_id）VALUES（？，？）RETURNING id
  [...（insertmanyvalues）1/2（有序的;不支持批处理）]（'pearl.krabs@gmail.com'，6）
  INSERT INTO address（email_address，user_id）VALUES（？，？）RETURNING id
  [insertmanyvalues 2/2（有序的;不支持批处理）]（'pearl@aol.com'，6）
  COMMIT




.. _tutorial_loading_relationships:

加载关系
---------------------

在最后一步中，我们调用了  :meth:`_orm.Session.commit`  方法，该方法向事务提交了 COMMIT，然后按  :paramref:` _orm.Session.commit.expire_on_commit`  的要求使所有对象过期，以使它们为下一个事务刷新。

当我们下一次访问这些对象的属性时，我们将在其主要属性的 SELECT 中发现 SELECT，例如当我们查看“u1”对象的新生成主键时：

.. sourcecode:: pycon+sql

  >>> u1.id
  {execsql}BEGIN（implicit）
  SELECT user_account.id AS user_account_id，user_account.name AS user_account_name，
  user_account.fullname AS user_account_fullname
  FROM user_account
  WHERE user_account.id = ？使用关系在查询语句中
------------------------------

前面的章节介绍了当使用**映射类的实例**时，  :func:`_orm.relationship` ` u1``，``a1``和``a2``实例。在本节中，将介绍  :func:`_orm.relationship` 在**映射类的类级别行为**中的行为，其在多个方面上有助于自动化SQL查询的构建。

使用关系连接
^^^^^^^^^^^^^^^^^^^^^

章节   :ref:`tutorial_select_join`  和   :ref:` tutorial_select_join_onclause`  和  :meth:`_sql.Select.join_from`  方法来组合SQL JOIN子句的用法。为了描述如何在表之间建立连接，这些方法通过以下两种方式之一来推断ON子句：基于表元数据结构中存在的单个明确的 :class:` _schema.ForeignKeyConstraint`对象连接两个表，或者我们可以提供一个明确的SQL表达式构造，指示具有特定ON子句。

使用ORM实体，一个额外机制可帮助我们设置JOIN的ON子句，即使用在用户映射中设置的   :func:`_orm.relationship`  对象，如在   :ref:` tutorial_declaring_mapped_classes`  对应的类绑定属性可以作为唯一的参数传递给  :meth:`_sql.Select.join`  ，它用于同时指示连接的右边以及ON子句：

    >>> print(select(Address.email_address).select_from(User).join(User.addresses))
    {printsql}SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

如果我们不指定ON子句，   :method:`_sql.Select.join`  或   :method:` _sql.Select.join_from`  不会使用映射上的ORM   :func:`_orm.relationship` 。这意味着，如果我们从` `User`` ``Address``进行连接而不使用ON子句，它之所以可以工作，是因为这两个映射的   :class:`_schema.Table`  对象之间的   :class:` _schema.ForeignKeyConstraint` ，而不是因为在 ``User`` 和 ``Address`` 类上的   :func:`_orm.relationship`  对象。

    >>> print(select(Address.email_address).join_from(User, Address))
    {printsql}SELECT address.email_address
    FROM user_account JOIN address ON user_account.id = address.user_id

在   :ref:`queryguide_toplevel`  中的   :ref:` orm_queryguide_joins`  部分，有许多如何使用  :meth:`.Select.join`  和  :meth:` .Select.join_from`  与  :func:`_orm.relationship` 构造的示例。

.. seealso::

      :ref:`orm_queryguide_joins`  在    :ref:` queryguide_toplevel`  中

关系 WHERE 运算符
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当构建WHERE子句时，   :func:`_orm.relationship`  提供了一些额外的SQL生成帮助程序，这些程序通常在使用ORM时非常有用。特别是，当我们的查询中涉及到限制查询的结果集，而这个限制涉及到关系时。

下面是一个带有where()方法的查询示例，查询结果只包含User和他们的Address。使用User.addresses属性来自动包含适当的WHERE子句：

.. sourcecode:: pycon+sql

  >>> session.query(User).join(User.addresses).filter(Address.email_address== 'pearl.krabs@gmail.com').all()
  {execsql}SELECT user_account.id AS user_account_id, user_account.name AS user_account_name
  FROM user_account JOIN address ON user_account.id = address.user_id
  WHERE address.email_address = ?
  [...] ('pearl.krabs@gmail.com',){stop}
  [<User(name=krusty, address=[Address(id=4, email_address='pearl.krabs@gmail.com')])>]

在本例中，User.addresses被  :meth:`.Query.join`  和  :meth:` .Query.filter`  方法使用，用作限制结果集的WHERE条件。

通过将  :meth:`.Query.join`  与   :class:` _sqlalchemy.orm.contains_eager`  结合使用，进一步优化连接。这使得查询中早产生的JOIN操作（也称为'早加载'）得以保留，并允许ORM在这些连接之后立即从父对象中读取数据而不必再次发出查询语句。

  >>> session.query(User).join(User.addresses)\\
  ...        .options(contains_eager(User.addresses)).all()
  {execsql}SELECT user_account.id AS user_account_id,
  user_account.name AS user_account_name,
  address_1.id AS address_1_id,
  address_1.email_address AS address_1_email_
  address,
  address_1.user_id AS address_1_user_id
  FROM user_account JOIN address AS address_1
  ON user_account.id = address_1.user_id
  [...]
  [<User(name=krusty, addresses=[Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')])>]WHERE子句
--------------------------------------------

参见   :ref:`queryguide_toplevel`  中的    :ref:` orm_queryguide_relationship_operators`  章节。

.. seealso::

      :ref:`queryguide_toplevel`  中的   :ref:` orm_queryguide_relationship_operators`  章节。



.. _tutorial_orm_loader_strategies:

加载策略
--------------------------------------------

在   :ref:`tutorial_loading_relationships`  章节中，我们引入了一个
概念，即当我们使用映射对象的实例时，访问使用   :func:`_orm.relationship` 
映射的属性时，默认情况下，如果未填充集合，则会发出  :term:`懒加载`  命令，
以加载此集合中应存在的对象。

懒加载是最著名的ORM模式之一，也是最具争议的模式之一。当内存中有几十个ORM对象均引用一些未加载的属性时，
对这些对象进行例行操作可能会产生许多额外的查询，并可能导致错误（也称为  :term:`N+1`  问题）。更糟糕的是，
它们会被隐式调用。这些隐式查询可能不会被注意到，在数据库事务不可用时试图进行操作时可能会导致错误，
或者在使用替代并发模式（如   :ref:`asyncio <asyncio_toplevel> ` ）时，它们实际上无法工作。

同时，当懒加载与使用的并发方法相兼容且不会引起问题时，懒加载也是一个非常受欢迎并且有用的模式。出于这些原因，
SQLAlchemy的ORM非常注重能够控制和优化加载行为。

使用ORM懒加载最有效的方法是首先**测试应用程序，打开SQL跟踪功能，并查看所发出的SQL语句**。如果有大量的冗余SELECT语句，它们看起来非常像
它们可以更有效地合并成一个语句，如果不存在于其   :class:`_orm.Session`   中的对象不适当地进行加载，
那么这就是要使用**加载程序策略的**时候。

加载程序策略表示为可使用  :meth:`_sql.Select.options`  方法与SELECT语句关联的对象，例如：

.. sourcecode:: python

      for user_obj in session.execute(
          select(User).options(selectinload(User.addresses))
      ).scalars():
          user_obj.addresses  # access addresses collection already loaded

它们也可以用  :paramref:`_orm.relationship.lazy`  选项作为   :func:` _orm.relationship`  的默认值进行配置，例如：

.. sourcecode:: python

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship


    class User(Base):
        __tablename__ = "user_account"

        addresses: Mapped[List["Address"]] = relationship(
            back_populates="user", lazy="selectin"
        )

每个加载策略对象都向将在稍后由   :class:`_orm.Session`  在决定如何加载和/或在访问时如何行为的各种属性提供一些信息。

下面的部分将介绍几种最为常用的加载策略。

参见：   :ref:`loading_toplevel`  中的两个部分：

    *   :ref:`relationship_lazy_option`  - 关于在   :func:` _orm.relationship`  上配置策略的详细信息
    *   :ref:`relationship_loader_options`  - 关于使用查询时加载程序策略的详细信息

Selectin Load
^^^^^^^^^^^^^

在现代SQLAlchemy中最常用的加载程序是   :func:`_orm.selectinload`  加载程序选项。
此选项解决了“N + 1”问题中最常见的一个，即涉及对相关集合引用的对象集的问题。
  :func:`_orm.selectinload`  将确保使用单个查询提前加载整个对象序列的特定集合。在大多数情况下，
它使用一种 SELECT 表单，通常可以只针对相关表发出，而无需引入 JOIN 或子查询，仅查询
尚未加载集合的那些父对象。下面我们使用   :func:`_orm.selectinload`  加载所有 ` `User`` 对象及其所有相关的 ``Address`` 对象。
虽然我们只调用了一次  :meth:`_orm.Session.execute`  给定了一个   :func:` _sql.select`  构造，
但在访问数据库时实际上会发出两个 SELECT 语句，第二个是用于获取相关联的 ``Address`` 对象的：

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
sandy(sandy@sqlalchemy.org, sandy@squirrelpower.org)
    patrick()
    squidward()
    ehkrabs()
    pkrabs(pearl.krabs@gmail.com, pearl@aol.com)

.. seealso::

      :ref:`selectin_eager_loading`  - in   :ref:` loading_toplevel` 

Joined Load
^^^^^^^^^^^

  :func:`_orm.joinedload`  是SQLAlchemy中最古老的“急加载器”，它会增强要传递给数据库的SELECT语句，以JOIN的形式排序加载关联对象。

  :func:`_orm.joinedload`  策略最适合于加载相关的多对一对象，因为这只需要将额外的列添加到主实体行中，而要获取所有这些列需要进行任何操作。为了更高效，它还接受一个  :paramref:` _orm.joinedload.innerjoin`  选项，这样就可以在以下情况下使用内部连接而不是外部连接：

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

  :func:`_orm.joinedload`  也适用于集合，即一对多关系，但它会以指数方式递归地乘出每个相关项目的主行，从而增加了通过嵌套集合和/或 较大集合发送的结果集数据量。所以它的使用与其他选项（如   :func:` _orm.selectinload` ）应该在每种情况下进行评估。

重要提示：

**WHERE**和**ORDER BY**限制含有   :func:`_orm.joinedload`  的查询语句 **不会针对 joinedload() 渲染的表**。以上面的代码为例，可以看到在SQL中，一个**匿名别名**从应用到“user_account”表上，从而无法直接在查询中定位到它。这个概念在   :ref:` zen_of_eager_loading`  章节中有更详细的讨论。

.. tip::

  重要提示：常常并不需要对多对一进行急加载，因为“N加一”问题在常见情况下更少见；当许多对象都引用相同的相关对象，例如每个都引用相同的“User”对象的许多“Address”对象时，常规惰性加载只会为该“User”对象发出一次SQL。当可能时，懒加载程序将在当前   :class:`_orm.Session`  中通过主键查找相关对象而不发出任何SQL。

.. seealso::

    :ref:`joined_eager_loading`  - in   :ref:` loading_toplevel` 

.. _tutorial_orm_loader_strategies_contains_eager:

显式Join + 急加载
^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果我们使用  :meth:`_sql.Select.join`  这样的方法来渲染JOIN从而加载“Address”表的行时，我们也可以利用该JOIN以急加载返回的每个“Address”对象上的“Address.user”属性的内容。这实际上就是我们正在使用“joined eager loading”，但是我们自己渲染了JOIN。可以使用   :func:` _orm.contains_eager`  选项来实现这个常见用例。此选项与   :func:`_orm.joinedload`  非常相似，但它假设我们已经自己设置了JOIN，因此它只表示应该将COLUMNS子句中的其他列加载到返回的每个对象的相关属性中。

例如：

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

上面，我们既根据 ``user_account.name`` 过滤了行，也将 ``user_account`` 的行加载到返回行每个“Address”对象的“Address.user”属性中。如果我们分别应用   :func:`_orm.joinedload` ，我们将会获得多余的SQL查询，因为它会额外连接两次：

    >>> stmt = (
    ...     select(Address)
    ...     .join(Address.user)
    ...     .where(User.name == "pkrabs")
    ...     .options(joinedload(Address.user))
    ...     .order_by(Address.id)
    ... )    >>> print(stmt)  # SELECT语句中存在JOIN和LEFT OUTER JOIN不必要
    {printsql}SELECT address.id, address.email_address, address.user_id,
    user_account_1.id AS id_1, user_account_1.name, user_account_1.fullname
    FROM address JOIN user_account ON user_account.id = address.user_id
    LEFT OUTER JOIN user_account AS user_account_1 ON user_account_1.id = address.user_id
    WHERE user_account.name = :name_1 ORDER BY address.id

.. seealso::

     :ref:`loading_toplevel` 中的两个部分：

    *   :ref:`zen_of_eager_loading`  - 详细描述了上述问题

    *   :ref:`contains_eager`  - 使用   :func:` .contains_eager` 


RaiseLoad
^^^^^^^^^

值得一提的另一个加载器策略是   :func:`_orm.raiseload` ，此选项是通过将通常是
懒加载的内容引发错误的方式，完全阻止应用程序出现  :term:`N plus one`  问题。
它有两个变体，通过  :paramref:`_orm.raiseload.sql_only`  选项进行控制，
阻止需要SQL的懒加载，还是所有包括那些只需要查询当前   :class:`_orm.Session` 
的 "load" 操作。

使用   :func:`_orm.raiseload`  的一种方法是在   :func:` _orm.relationship`  上直接配置，
通过将  :paramref:`_orm.relationship.lazy`  设置为值 ` `"raise_on_sql"``，
以便对于特定映射，某个关系将永远不会尝试发出SQL：

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

使用这样的映射，应用程序被阻止懒加载，表明需要指定一种加载策略来执行特定查询：

    >>> u1 = session.execute(select(User)).scalars().first()
    {execsql}SELECT user_account.id FROM user_account
    [...] ()
    {stop}>>> u1.addresses
    Traceback (most recent call last):
    ...
    sqlalchemy.exc.InvalidRequestError: 'User.addresses' is not available due to lazy='raise_on_sql'

异常将表明此集合应该提前加载：

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

``lazy="raise_on_sql"`` 选项也会智能地处理多对一关系；上述情况，如果某个 ``Address`` 
对象的 ``Address.user`` 属性未加载，但该 ``User`` 对象在本地存在于同一个   :class:`_orm.Session`  中，
"raiseload" 策略将不会引发错误。

.. seealso::

      :ref:`prevent_lazy_with_raiseload`  - in   :ref:` loading_toplevel` 