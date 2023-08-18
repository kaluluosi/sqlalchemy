.. highlight:: pycon+sql

.. |prev| replace:: :doc:`dml`
.. |next| replace:: :doc:`relationships`

.. include:: queryguide_nav_include.rst


.. doctest-include _deferred_setup.rst

.. currentmodule:: sqlalchemy.orm

.. _loading_columns:

======================
列载入选项
======================

.. admonition:: 关于本文档

    本节介绍有关列载入的其他选项。使用的映射包括用于存储大字符串值的列，我们可能希望限制它们何时被载入。

    :doc:`查看本页的ORM设置 <_deferred_setup>`。以下示例中，将重新定义“Book”映射器以修改某些列定义。

.. _orm_queryguide_column_deferral:

通过延迟列来限制载入哪些列
-------------------------------------------------- ------

**列延迟** 是指在查询类型的对象时忽略的ORM映射列。这里的一般基本原则是性能，或者当表存在很少使用的可能具有大量数据的列时，每个查询完全加载这些列可能是时间和/或内存密集型的。 SQLAlchemy ORM提供了各种方式来控制在加载实体时控制列的载入。

本节中的大多数示例都说明了**ORM加载程序选项**。这些是小的结构，它们传递给:class:`_sql.Select`对象的:meth:`_sql.Select.options`方法，然后在将对象编译为SQL字符串时由ORM使用。

.. _orm_queryguide_load_only:

使用``load_only()``来减少载入的列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_orm.load_only`加载程序选项是特别适合在已知仅将访问少量列时加载对象时使用的选项。此选项接受变量数量的类属性对象，指示应该加载那些映射列的属性，其他不在主键之外的列映射属性将不包括在获取的列中。在下面的示例中，“Book”类包含列“ title”，“ summary”和“ cover_photo”。使用：func:`_orm.load_only`，我们可以指示ORM仅在开始时加载“ title”和“ summary”列：

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import load_only
    >>> stmt = select(Book).options(load_only(Book.title, Book.summary))
    >>> books = session.scalars(stmt).all()
    {execsql}SELECT book.id, book.title, book.summary
    FROM book
    [...] ()
    {stop}>>> for book in books:
    ...     print(f"{book.title}  {book.summary}")
    100 Years of Krabby Patties  some long summary
    Sea Catch 22  another long summary
    The Sea Grapes of Wrath  yet another summary
    A Nut Like No Other  some long summary
    Geodesic Domes: A Retrospective  another long summary
    Rocketry for Squirrels  yet another summary

上面，SELECT语句省略了“ cover_photo”列，仅包括“ title”和“ summary”，以及主键列“.id”；ORM通常会始终获取主键列，因为这些列对于建立行的标识是必需的。

一旦加载，对象通常会在其余未加载的属性上应用：term：'lazy loading'行为，这意味着在首次访问任何属性时，当前事务中会发出一个SQL语句以加载值。下面，访问“ cover_photo”会发出SELECT语句以加载其值：

    >>> img_data = books[0].cover_photo
    {execsql}SELECT book.cover_photo AS book_cover_photo
    FROM book
    WHERE book.id = ?
    [...] (1,)

懒惰加载始终使用对象处于：term：'persistent'状态的:class:`_orm。会话`发出。如果对象处于任何：class：`_orm。会话`中：term：'detached'，则操作失败，引发异常。

与懒惰加载不同，延迟列也可以配置为在访问时引发有关其附加状态的信息性异常。当使用：func:`_orm.load_only`构造时，可以使用：paramref:`_orm.load_only.raiseload`参数指示。有关背景和示例，请参见:ref：`orm_queryguide_deferred_raiseload`部分。

.. tip :: 如其他地方所述，协同错误时不可使用懒惰加载。 :ref：`asyncio_toplevel`。

使用``load_only()``作用于多个实体
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_orm.load_only`仅限于在其属性列表中所引用的单个实体（尚未允许跨越多个实体跨度的属性列表）。在下面的示例中，给定的：func:`_orm.load_only`选项仅适用于“书籍”实体。未受影响的“用户”实体具有所有“user_account”列；在生成的SELECT语句中，“book”表只有“id”和“ title”列：

    >>> stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname,
    book.id AS id_1, book.title
    FROM user_account JOIN book ON user_account.id = book.owner_id

如果要将：func:`_orm.load_only`选项应用于“用户”和“书籍”，则应使用两个单独的选项：

    >>> stmt = (
    ...     select(User, Book)
    ...     .join_from(User, Book)
    ...     .options(load_only(User.name), load_only(Book.title))
    ... )
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, book.id AS id_1, book.title
    FROM user_account JOIN book ON user_account.id = book.owner_id

.. _orm_queryguide_load_only_related:

对相关对象和集合使用``load_only()``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在使用关系加载器 :ref:`loading_toplevel` 时，任何关系加载器的对象都可以使用:meth:`.Load.load_only`来应用 :func:`_orm.load_only`规则，以控制子实体上的列。在下面的示例中，使用：func:`_orm.selectinload`来加载每个“用户”对象上的相关“books”集合。通过:meth:`.Load.load_only`应用下文中的 :func:`_orm.load_only`，当加载关系的对象时，发出的SELECT将仅涉及“title”列，以及主键列：

    >>> from sqlalchemy.orm import selectinload
    >>> stmt = select(User).options(selectinload(User.books).load_only(Book.title))
    >>> for user in session.scalars(stmt):
    ...     print(f"{user.fullname}   {[b.title for b in user.books]}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] ()
    SELECT book.owner_id AS book_owner_id, book.id AS book_id, book.title AS book_title
    FROM book
    WHERE book.owner_id IN (?, ?)
    [...] (1, 2)
    {stop}Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
    Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']


.. comment

    >>> session.expunge_all()

如上所述，在不需要指定要使用的加载样式的情况下，也可以将：func:`_orm.load_only`应用于子实体，例如，使用：func:`_orm.defaultload`选项进行链接，该选项在本例中将保留关系加载样式“ lazy”，并将自定义：func:适用于本例的`_orm.load_only`规则SELECT语句发出：

    >>> from sqlalchemy.orm import defaultload
    >>> stmt = select(User).options(defaultload(User.books).load_only(Book.title))
    >>> for user in session.scalars(stmt):
    ...     print(f"{user.fullname}   {[b.title for b in user.books]}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    [...] ()
    SELECT book.id AS book_id, book.title AS book_title
    FROM book
    WHERE ? = book.owner_id
    [...] (1,)
    {stop}Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
    {execsql}SELECT book.id AS book_id, book.title AS book_title
    FROM book
    WHERE ? = book.owner_id
    [...] (2,)
    {stop}Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']

.. _orm_queryguide_defer:

使用``defer()``省略特定列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_orm.defer`加载程序选项是比：func:`_orm.load_only`更细的选择，允许标记单个特定列为“不要加载”。在下面的示例中，直接应用：func:`_orm.defer`到“.cover_photo”列，保留所有其他列的行为：

    >>> from sqlalchemy.orm import defer
    >>> stmt = select(Book).where(Book.owner_id == 2).options(defer(Book.cover_photo))
    >>> books = session.scalars(stmt).all()
    {execsql}SELECT book.id, book.owner_id, book.title, book.summary
    FROM book
    WHERE book.owner_id = ?
    [...] (2,)
    {stop}>>> for book in books:
    ...     print(f"{book.title}: {book.summary}")
    A Nut Like No Other: some long summary
    Geodesic Domes: A Retrospective: another long summary
    Rocketry for Squirrels: yet another summary

在默认情况下，与：func:`_orm.load_only`一样，未加载的列将在访问时 :term：' lazy loading'其值：

    >>> img_data = books[0].cover_photo
    {execsql}SELECT book.cover_photo AS book_cover_photo
    FROM book
    WHERE book.id = ?
    [...] (4,)

在一个语句中使用多个：func:`_orm.defer`选项，以将多个列标记为延迟。

与: func:`_orm.load_only`和: func:`_orm.defer`加载程序选项一样，: func`:` _orm.defer选项也包括在属性访问时引发有关其附加状态的信息性异常。当使用：func:`_orm.load_only`构造来命名一组特定的非延迟列时，可以使用：paramref：`_orm.load_only.raiseload`参数指示该行为。 有关如何配置和使用这种行为的背景，请参见:ref：`orm_queryguide_deferred_raiseload`部分。

.. _orm_queryguide_deferred_raiseload:

使用 raiseload 防止延迟列载入
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. comment

  >>> session.expunge_all()

使用：func:`_orm.load_only`或：func:`_orm.defer`加载程序选项时，当对一个对象上标记为延迟的属性进行访问时，默认行为是发出一个SQL语句以加载其值。通常需要防止此加载发生，并且在访问属性时引发异常，指示未预计在访问该列的数据库进行查询。典型的场景是使用所有已知所需进行操作的列来加载对象，然后将其传递到视图层。视图层发出的任何其他SQL操作都应该被捕获，以便可以用于调整初始加载操作以适应该附加数据，而不是在附加信息时再次进行延迟加载。

对于此用例，：func:`_orm.defer`和：func:`_orm.load_only`加载程序选项包括布尔参数：paramref:`_orm.defer.raiseload`，当设置为``True``时，受影响的属性将在访问时引发。在下面的示例中，延迟列“.cover_photo”将禁止属性访问：

  >>> book = session.scalar(
  ...     select(Book).options(defer(Book.cover_photo, raiseload=True)).where(Book.id == 4)
  ... )
  {execsql}SELECT book.id, book.owner_id, book.title, book.summary
  FROM book
  WHERE book.id = ?
  [...] (4,)
  {stop}>>> book.cover_photo
  Traceback (most recent call last):
  ...
  sqlalchemy.exc.InvalidRequestError: 'Book.cover_photo' is not available due to raiseload=True

当使用：func:`_orm.load_only`来命名一组非延迟列时，可以使用：paramref:`_orm.load_only.raiseload`参数将``raiseload``行为应用于所有延迟的属性。请参见 :ref：`orm_queryguide_deferred_raiseload` 部分了解详细信息。

.. note::

    目前尚不能将引用相同实体的：func:`_orm.load_only`和：func:`_orm.defer`选项混合在一个语句中，以更改某些属性的“ raiseload”行为；目前，这样做将在属性的不确定加载行为中产生未定义的加载行为。

.. seealso::

    :paramref:`_orm.defer.raiseload`功能是适用于关系的相同的顶级功能
    可用于"raiseload"特性。对于关系中的"raiseload"，请参见
    这个指南的 :ref:`loading_toplevel` 部分中的 :ref:`prevent_lazy_with_raiseload`。

.. _orm_queryguide_deferred_declarative:

在映射中配置列 Deferral
----------------------------------

.. comment

    >>> class Base(DeclarativeBase):
    ...     pass

映射器的默认行为是让执行默认延迟。该默认延迟设置适用于应延迟的列，并且非常适合不希望在每个查询中无条件加载的列。要配置，请使用：paramref:`_orm.mapped_column.deferred`参数。下面的示例说明了适用于“.summary”和“.cover_photo”列的默认列 deferral 的映射：

    >>> class Book(Base):
    ...     __tablename__ = "book"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     title: Mapped[str]
    ...     summary: Mapped[str] = mapped_column(Text, deferred=True)
    ...     cover_photo: Mapped[bytes] = mapped_column(LargeBinary, deferred=True)
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Book(id={self.id!r}, title={self.title!r})"

使用上述映射，对“ Book”进行的查询将自动不包括“.summary”和“.cover_photo”列：

    >>> book = session.scalar(select(Book).where(Book.id == 2))
    {execsql}SELECT book.id, book.owner_id, book.title
    FROM book
    WHERE book.id = ?
    [...] (2,)

与所有延迟一样，默认情况下，查询状态下未加载的属性的默认行为是懒惰加载其值：

    >>> img_data = book.cover_photo
    {execsql}SELECT book.cover_photo AS book_cover_photo
    FROM book
    WHERE book.id = ?
    [...] (2,)

如前所述，映射器级的deferral也包括具有在属性访问时引发具有关键信息的异常的能力，而不管它们的附加状态如何。当使用：func:`_orm.load_only`构造时，这可以使用：paramref:`_orm.load_only.raiseload`参数来指示。请参见 :ref：`orm_queryguide_deferred_raiseload` 部分，了解如何配置和使用此选项。

.. _orm_queryguide_deferred_imperative:

使用``deferred()` `对命令式映射器，映射SQL表达式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在SQLAlchemy引入：func：`_orm.mapped_column`构造之前，：func：`_orm.deferred`函数是早期的、更通用的"deferred列"映射指令。

在配置ORM映射器时使用：func:`_orm.deferred`，并接受任意SQL表达式或：class:`_schema.Column`对象。因此，它适合用于非声明性：ref：`orm_imperative_mapping`映射，将其传递给：paramref:`_orm.registry。`以：func:`_orm.registry.map_imperatively.properties`字典为键：

.. sourcecode:: python

    from sqlalchemy import Blob
    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy import Text
    from sqlalchemy.orm import registry

    mapper_registry = registry()

    book_table = Table(
        "book",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("title", String(50)),
        Column("summary", Text),
        Column("cover_image", Blob),
    )


    class Book:
        pass


    mapper_registry.map_imperatively(
        Book,
        book_table,
        properties={
            "summary": deferred(book_table.c.summary),
            "cover_image": deferred(book_table.c.cover_image),
        },
    )

如 : ref:`orm_queryguide_select_columns` 和其他地方所述，可以使用 :func:`.select` 结构来加载任意 SQL 表达式结果集中使用计数和其他SQL表达式的添加
-----------------

例如，如果我们想要发出一个查询以加载“User”对象，但是也包括每个“User”拥有的书籍数量的计数，我们可以使用“func.count（Book.id）”将“计数”列添加到查询中，该查询包括连接到“Book”，以及按所有者ID分组的GROUP BY 。这将产生:class:`.Row`对象，每个对象都包含两个条目，一个条目是“User”，另一个条目是“func.count（Book.id）”：

    >>> from sqlalchemy import func
    >>> stmt = select(User, func.count(Book.id)).join_from(User, Book).group_by(Book.owner_id)
    >>> for user, book_count in session.execute(stmt):
    ...     print(f"用户名：{user.name}  书籍数量：{book_count}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    count(book.id) AS count_1
    FROM user_account JOIN book ON user_account.id = book.owner_id
    GROUP BY book.owner_id
    [...] ()
    {stop}用户名：海绵宝宝  书籍数量：3
    用户名：松鼠先生  书籍数量：3

在上面的示例中，“User”实体和“book count” SQL表达式是分别返回的。但是，一个常见的用例是生成一个查询，该查询将仅返回“User”对象，例如使用:meth:`_orm.Session.scalars`进行迭代，其中“func.count（Book.id）”的查询结果应用动态地到每个“User”实体。最终的结果将类似于使用:func:`_orm.column_property`将任意SQL表达式映射到类的情况，但是SQL表达式可以在查询时进行修改。对于这种用例，SQLAlchemy提供了:func:`_orm.with_expression`加载器选项，当与映射器级别的:func:`_orm.query_expression`指令结合使用时，可以产生此结果。

.. comment

    >>> class Base(DeclarativeBase):
    ...     pass
    >>> class Book(Base):
    ...     __tablename__ = "book"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     title: Mapped[str]
    ...     summary: Mapped[str] = mapped_column(Text)
    ...     cover_photo: Mapped[bytes] = mapped_column(LargeBinary)
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Book(id={self.id!r}, title={self.title!r})"

要在查询中应用:func:`_orm.with_expression`，映射类必须具有先前使用:func:`_orm.query_expression`指令配置的ORM映射属性。该指令将在OR映射类上生成一个属性，该属性适合接收查询时SQL表达式。以下是将“User.book_count”添加到“User”的新属性。这个ORM映射属性只读，没有默认值；在加载的实例上访问它通常会产生“None”：

    >>> from sqlalchemy.orm import query_expression
    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str]
    ...     fullname: Mapped[Optional[str]]
    ...     book_count: Mapped[int] = query_expression()
    ...
    ...     def __repr__(self) -> str:
    ...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

通过在映射中配置“User.book_count”属性，我们可以使用:func:`_orm.with_expression`加载器选项，在每次加载其时将自定义SQL表达式应用于每个“User”对象::

    >>> from sqlalchemy.orm import with_expression
    >>> stmt = (
    ...     select(User)
    ...     .join_from(User, Book)
    ...     .group_by(Book.owner_id)
    ...     .options(with_expression(User.book_count, func.count(Book.id)))
    ... )
    >>> for user in session.scalars(stmt):
    ...     print(f"用户名：{user.name}  书籍数量：{user.book_count}")
    {execsql}SELECT count(book.id) AS count_1, user_account.id, user_account.name,
    user_account.fullname
    FROM user_account JOIN book ON user_account.id = book.owner_id
    GROUP BY book.owner_id
    [...] ()
    {stop}用户名：海绵宝宝  书籍数量：3
    用户名：松鼠先生  书籍数量：3

以上，我们将“func.count（Book.id）”表达式从:func:`_sql.select`构造函数的列参数中移除，并将其移动到:func:`_orm.with_expression`加载器选项中。ORM然后将其视为特殊的列加载选项，该选项在语句上动态应用。

:func:`.query_expression`映射有以下注意事项：

* 在未使用:func:`_orm.with_expression`为属性填充值的对象上，对象实例上的属性将具有值“None”，除非在映射上将:paramref:`_orm.query_expression.default_expr`参数设置为默认的SQL表达式。

* 除非使用:ref:`orm_queryguide_populate_existing`，否则:func:`_orm.with_expression`值**不会填充已加载的对象**。下面的示例**不起作用**，因为“obj”对象已经加载：

  .. sourcecode:: python

    # 加载第一个A
    obj = session.scalars(select(A).order_by(A.id)).first()

    # 使用选项加载相同的A；该表达式**不会**应用于已加载的对象
    obj = session.scalars(select(A).options(with_expression(A.expr, some_expr))).first()

  要确保属性在现有对象上重新加载，请使用:ref:`orm_queryguide_populate_existing`执行选项以确保重新填充所有列：

  .. sourcecode:: python

    obj = session.scalars(
        select(A)
        .options(with_expression(A.expr, some_expr))
        .execution_options(populate_existing=True)
    ).first()

* 当对象过期时，:func:`_orm.with_expression` SQL表达式**将丢失**。一旦对象过期，无论是通过:meth:`.Session.expire`还是通过:meth:`.Session.commit`的expire_on_commit行为，SQL表达式及其值都不再与属性关联，并且在随后的访问中将返回“None”。

* :func:`_orm.with_expression`作为对象加载选项，仅在查询完整实体时对查询的**最外层部分**生效，而不是在任意列选择、子查询或复合语句（如UNION）的元素中生效。请参见下一节：ref:`orm_queryguide_with_expression_unions`。

* 所映射的属性**不能**应用于查询的其他部分，比如WHERE子句、ORDER BY子句，并且使用自定义的宣传表达式；也就是说，这不能使用：

  .. sourcecode:: python

    # 无法在查询中引用A.expr
    stmt = (
        select(A)
        .options(with_expression(A.expr, A.x + A.y))
        .filter(A.expr > 5)
        .order_by(A.expr)
    )

  在上面的WHERE子句和ORDERBY子句中，``A.expr``表达式将解析为NULL。要在整个查询中使用表达式，请对变量进行赋值并使用它：

  .. sourcecode:: python

    # 首先分配所需表达式，然后引用该查询
    a_expr = A.x + A.y
    stmt = (
        select(A)
        .options(with_expression(A.expr, a_expr))
        .filter(a_expr > 5)
        .order_by(a_expr)
    )

.. seealso::

    :func:`_orm.with_expression`选项是用于在查询时间动态地应用SQL表达式的ORM加载器选项。对于映射器上配置的普通SQL表达式，请参见:ref:`mapper_sql_expressions`一节。


列加载API
-------------------

.. autofunction:: defer

.. autofunction:: deferred

.. autofunction:: query_expression

.. autofunction:: load_only

.. autofunction:: undefer

.. autofunction:: undefer_group

.. autofunction:: with_expression