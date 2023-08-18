.. highlight:: pycon+sql

.. |prev| replace::  :doc:`dml` 
.. |next| replace::  :doc:`relationships` 

.. include:: queryguide_nav_include.rst


.. doctest-include _deferred_setup.rst

.. currentmodule:: sqlalchemy.orm

.. _loading_columns:

======================
列加载选项
======================

.. admonition:: 关于该文档

    本节提供与列加载相关的其他选项。采用的映射包括列存储
    大字符串值，我们可能希望在加载这些值时进行限制。

     :doc:`查看此页面的ORM设置 <_deferred_setup>` 。一些
    下面的示例将重新定义“Book”映射器以修改
    一些列定义。

.. _orm_queryguide_column_deferral:

使用列延迟选择要加载的列
------------------------------------------------

**列延迟**是指在查询该类型的对象时省略了ORM映射的列。
SELECT语句。 一般的理由是性能，在表具有很少使用的列的情况下，
具有潜在大数据值的情况下，完全加载这些列的每个查询可能是
时间和/或内存密集型。 SQLAlchemy ORM提供了各种方法来
控制加载实体时列的方式。

本节中的大多数示例都说明**ORM加载器选项**。这些
是传递给  :meth:`_sql.Select.options`  方法的小构造
 :class:`_sql.Select` 对象，它们然后由ORM消耗
当对象编译为SQL字符串时。

.. _orm_queryguide_load_only:

使用“load_only()”减少加载列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

 :func:`_orm.load_only` 加载器选项是载入对象时使用的最快的选项
已知仅访问少量列。此选项接受可变数量的类绑定属性
对象，指示应加载哪些映射为列的属性，其中
所有主键之外的其他列映射的属性都不会成为
检索出的列。在下面的示例中，“Book”类包含
列“。title”，“.summary”和“。cover_photo”。使用
  :func:`_orm.load_only` ，我们可以指示ORM仅加载
“。title”和“。summary”列::

    >>> from sqlalchemy import select
    >>> from sqlalchemy.orm import load_only
    >>> stmt = select(Book).options(load_only(Book.title, Book.summary))
    >>> books = session.scalars(stmt).all()
    SELECT book.id, book.title, book.summary
    FROM book
    [...] ()
    >>> for book in books:
    ...     print(f"{book.title}  {book.summary}")
    100 Years of Krabby Patties  some long summary
    Sea Catch 22  another long summary
    The Sea Grapes of Wrath  yet another summary
    A Nut Like No Other  some long summary
    Geodesic Domes: A Retrospective  another long summary
    Rocketry for Squirrels  yet another summary

上面，SELECT语句省略了“.cover_photo”列，
仅包括“。title”和“。summary”，以及主键列。
“.id”；ORM通常会始终获取主键列，因为这些列是必需的，
用于建立行的标识。

一旦加载，对象通常具有  :term:`lazy loading`  行为
应用于其余的未加载属性，这意味着在首次访问任何属性时，
将在当前事务中发出SQL语句以加载值。下面，访问“.cover_photo”会发出SELECT
语句来加载其值::

    >>> img_data = books[0].cover_photo
    SELECT book.cover_photo AS book_cover_photo
    FROM book
    WHERE book.id = ?
    [...] (1,)

无论其附加状态如何，惰性负载始终使用 :class:`_orm.Session` 发出来完成
对象处于  :term:`persistent`  状态。如果对象是  :term:` detached`  
从任何 :class:`_orm.Session` 中，该操作将失败，引发异常。

作为访问时的懒惰加载的替代方法，还可以配置延迟处理的列在访问时引发
一个信息丰富的异常，无论其附加状态如何。当使用：func:`_orm.load_only`时
可以使用：paramref:`_orm.load_only.raiseload`参数进行指示。
有关背景和示例，请参见第：ref:`orm_queryguide_deferred_raiseload`节。

.. tip:: 如其他地方所述，使用时无法使用懒加载
     :ref:`asyncio_toplevel` 。

与多个实体一起使用“load_only()”
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 :func:`_orm.load_only` 仅限于其属性列表中引用的单个实体
（当前不允许传递跨越多个实体的属性列表）。在下面的示例中，选择的内容
 :func:`_orm.load_only` 选项仅适用于“Book”实体。另一方面，“User”
选择的实体不受影响；在生成的SELECT中，所有for“user_account”的列都存在，
而只有“book.id”和“book.title”存在于“book”表中::

    >>> stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, user_account.fullname,
    book.id AS id_1, book.title
    FROM user_account JOIN book ON user_account.id = book.owner_id

如果我们想要对 ``User`` 和 ``Book`` 都应用   :func:`_orm.load_only` ，我们需要使用两个单独的选项：

    >>> stmt = (
    ...     select(User, Book)
    ...     .join_from(User, Book)
    ...     .options(load_only(User.name), load_only(Book.title))
    ... )
    >>> print(stmt)
    {printsql}SELECT user_account.id, user_account.name, book.id AS id_1, book.title
    FROM user_account JOIN book ON user_account.id = book.owner_id

.. _orm_queryguide_load_only_related:

在相关对象和集合上使用 ``load_only()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当使用   :ref:`relationship loaders <loading_toplevel>`  在控制相关对象的加载时，任何关系载入器的  :meth:` .Load.load_only`  方法都可以用于在子实体上应用   :func:`_orm.load_only`  规则。在下面的示例中，我们使用了   :func:` _orm.selectinload`  来加载每个 ``User`` 对象上的相关 ``books`` 集合。通过在生成的选项对象上应用  :meth:`.Load.load_only` ，当为关系加载对象时，生成的 SELECT 将仅参考 ` `title`` 列，除了主键列：

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

可以同时应用   :func:`_orm.load_only`  到子实体上，而不需要为关系本身指定加载样式。如果我们不想改变 ` `User.books`` 的默认加载方式，但仍要对 ``Book`` 应用 load only 规则，我们将使用   :func:`_orm.defaultload`  选项进行链接，在这种情况下，将保留关系的默认加载样式 "lazy"，并将自定义的   :func:` _orm.load_only`  规则应用于为每个 ``User.books`` 集合发出的 SELECT 语句：

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

使用 ``defer()`` 省略特定列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :func:`_orm.defer`  载入选项是   :func:` _orm.load_only`  的更细粒度的替代方法，它允许将单个特定的列标记为 "不加载"。在下面的示例中，   :func:`_orm.defer`  直接应用于 ` `.cover_photo`` 列，留下了所有其他列的行为不变：

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

与   :func:`_orm.load_only`  一样，默认情况下未加载的列将在使用  :term:` lazy loading`  时加载自己：

    >>> img_data = books[0].cover_photo
    {execsql}SELECT book.cover_photo AS book_cover_photo
    FROM book
    WHERE book.id = ?
    [...] (4,)

可以在一个语句中使用多个   :func:`_orm.defer`  选项，以便将多个列标记为延迟加载。

与   :func:`_orm.load_only`  一样，  :func:` _orm.defer`  选项会延迟加载的列还可以使用“raiseload”属性在访问时引发异常，而不是惰性加载。在下面的章节“:ref：`orm_queryguide_deferred_raiseload`”中给出了示例。

 .. _orm_queryguide_deferred_raiseload:

使用raiseload防止延迟列加载
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. comment

  >>> session.expunge_all()

在使用   :func:`_orm.load_only`  或   :func:` _orm.defer`  加载选项时，将对象上标记为deferred的属性具有默认行为何时首次访问，当前事务内将发出SELECT语句以加载它们的值。通常需要防止这种负载发生，而是在访问属性时引发异常，表示不期望为此列查询数据库。一个典型的场景是加载全部已知所需的列的对象以使操作继续进行，然后将它们传递到视图层中。在视图层内发出的任何其他SQL操作都应该被捕获，以便可以调整启动时的加载操作以适应该附加数据，而不是产生额外的惰性加载。

对于此用例，   :func:`_orm.defer`  和   :func:` _orm.load_only`  选项包括一个 boolean 参数  :parg:`~_orm.defer.raiseload`  ，当设置为 ` `True`` 时，将导致受影响的属性在访问时引发异常。在下面的示例中，将取消该:deferred列`.cover_photo`上的属性访问权限::

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

当使用   :func:`_orm.load_only`  指定一组特定的未延迟的列时， ` `raiseload`` 行为可能适用于剩余列，使用  :paramref:`_orm.load_only.raiseload`  参数，该参数将应用于所有  :term:` deferred attribute`  ::

  >>> session.expunge_all()
  >>> book = session.scalar(
  ...     select(Book).options(load_only(Book.title, raiseload=True)).where(Book.id == 5)
  ... )
  {execsql}SELECT book.id, book.title
  FROM book
  WHERE book.id = ?
  [...] (5,)
  {stop}>>> book.summary
  Traceback (most recent call last):
  ...
  sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True

.. note::

   目前尚不能在一个语句中混合   :func:`_orm.load_only`  和   :func:` _orm.defer`  选项，这些选项引用相同的实体以更改某些属性的 ``raiseload`` 行为；当前，这样做会产生属性的未定义加载行为。

.. seealso::

    :paramref:`_orm.defer.raiseload`  一节。

.. _orm_queryguide_deferred_declarative:

配置映射的列缓期
---------------------------------------

.. comment

    >>> class Base(DeclarativeBase):
    ...     pass

对于默认情况下不应在每个查询中无条件加载的列，可以将   :func:`_orm.defer`  功能作为映射列的默认行为。为了配置，使用   :func:` _orm.mapped_column`  的  :paramref:`_orm.mapped_column.deferred`  参数。下面的示例说明了适用于 ` `Book`` 的映射，将默认列延迟应用于 ``summary`` 和 ``cover_photo`` 列：

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

使用上述映射，针对 ``Book`` 的查询将自动不包括 ``summary`` 和 ``cover_photo`` 列：

    >>> book = session.scalar(select(Book).where(Book.id == 2))
    {execsql}SELECT book.id, book.owner_id, book.title
    FROM book
    WHERE book.id = ?
    [...] (2,)

在所有缓期中都是这样，当首次访问加载对象上的deferred属性时，它们将  :term:`lazy load`  他们的值。

   >>> img_data = book.cover_photo
   {execsql}SELECT book.cover_photo AS book_cover_photo
   FROM book
   WHERE book.id = ?
   [...] (2,)

与   :func:`_orm.defer`  和   :func:` _orm.load_only`  加载器选项一样，映射器级别的延迟也包括一个 ``raiseload`` 选项。当没有其他选项时，由于行为发生，而不是延迟加载。这允许映射在某些列默认情况下不加载，而且不会在语句中使用显式指令的情况下懒惰地加载。请参阅   :ref:`orm_queryguide_mapper_deferred_raiseload`  部分，了解如何配置和使用此行为。

.. _orm_queryguide_deferred_imperative:

使用 ``deferred()`` 用于命令式映射，映射 SQL 表达式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在引入   :func:`_orm.mapped_column`  构造之前，   :func:` _orm.deferred`  函数是早期的、更通用的“延迟列”映射指令。

当配置 ORM 映射器时，可以使用   :func:`_orm.deferred`  ，并接受任意 SQL 表达式或   :class:` _schema.Column`  对象作为参数。因此，当使用非声明性   :ref:`imperative mappings <orm_imperative_mapping>`  时，它适用于传递到  :paramref:` _orm.registry.map_imperatively.properties`  字典中：

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

当映射的 SQL 表达式应该延迟加载时，   :func:`_orm.deferred`  也可以用于替代   :func:` _orm.column_property`  ：

.. sourcecode:: python

    from sqlalchemy.orm import deferred


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        firstname: Mapped[str] = mapped_column()
        lastname: Mapped[str] = mapped_column()
        fullname: Mapped[str] = deferred(firstname + " " + lastname)

.. seealso::

      :ref:`mapper_column_property_sql_expressions` - 参见   :ref:` mapper_sql_expressions`  部分

      :ref:`orm_imperative_table_column_options` - 参见   :ref:` orm_declarative_table_config_toplevel`  部分

使用 ``undefer()`` 来“急切”加载延迟列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

默认情况下，如果某些列的映射已经被延迟，   :func:`_orm.undefer`  选项将会导致任何一列被快速地加载，即与映射的所有其他列一起提前加载。例如，我们可以将   :func:` _orm.undefer`  应用于以前映射中标记为延迟的 ``Book.summary`` 列：

    >>> from sqlalchemy.orm import undefer
    >>> book = session.scalar(select(Book).where(Book.id == 2).options(undefer(Book.summary)))
    {execsql}SELECT book.id, book.owner_id, book.title, book.summary
    FROM book
    WHERE book.id = ?
    [...] (2,)

现在 "Book.summary" 行被急切地加载，可以在不发出额外 SQL 的情况下访问：

    >>> print(book.summary)
    another long summary

.. _orm_queryguide_deferred_group:

按组加载待决列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. comment

    >>> class Base(DeclarativeBase):
    ...     pass

通常情况下，当映射了延迟属性时，在访问一个对象上的属性时，将仅加载该列，而不加载其他列，即使映射中具有其他标记为延迟的列。通常情况下，“延迟”属性是与一组属性共同加载的。在公共情况下，  :paramref:`_orm.mapped_column.deferred_group`  参数可以使用，它接受一个任意的字符串，用于定义一组共同的要不加载的列 ：

    >>> class Book(Base):
    ...     __tablename__ = "book"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     title: Mapped[str]
    ...     summary: Mapped[str] = mapped_column(
    ...         Text, deferred=True, deferred_group="book_attrs"
    ...     )
    ...     cover_photo: Mapped[bytes] = mapped_column(
    ...         LargeBinary, deferred=True, deferred_group="book_attrs"
    ...     )
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Book(id={self.id!r}, title={self.title!r})"

使用上述映射，访问 ``summary`` 或 ``cover_photo`` 时将一次性加载这两个列，只需使用一条 SQL 语句：

    >>> book = session.scalar(select(Book).where(Book.id == 2))
    {execsql}SELECT book.id, book.owner_id, book.title
    FROM book
    WHERE book.id = ?
    [...] (2,)
    {stop}>>> img_data, summary = book.cover_photo, book.summary
    {execsql}SELECT book.summary AS book_summary, book.cover_photo AS book_cover_photo
    FROM book
    WHERE book.id = ?
    [...] (2,)


使用 ``undefer_group()`` 按组取消加载延迟列
=============================================如果在之前的章节中使用了  :paramref:`_orm.mapped_column.deferred_group`  对延迟加载的列进行了配置，可以使用 :func:` _orm.undefer_group`选项一次指定整个组加载方式为立即加载，需要传递组的字符串名称::

    >>> from sqlalchemy.orm import undefer_group
    >>> book = session.scalar(
    ...     select(Book).where(Book.id == 2).options(undefer_group("book_attrs"))
    ... )
    {execsql}SELECT book.id, book.owner_id, book.title, book.summary, book.cover_photo
    FROM book
    WHERE book.id = ?
    [...] (2,)

这时，不需要进行其他操作就可以访问到所有延迟加载的列内容::

    >>> img_data, summary = book.cover_photo, book.summary

使用通配符undefer
^^^^^^^^^^^^^^^^^^

ORM加载器的大多数选项都接受通配符表达式，用 "*" 表示，它表示将选项应用于所有相关属性。如果映射具有一系列延迟加载的列，则可以通过使用通配符一次性未定义所有这些列来引用它们，而无需使用组名::

    >>> book = session.scalar(select(Book).where(Book.id == 3).options(undefer("*")))
    {execsql}SELECT book.id, book.owner_id, book.title, book.summary, book.cover_photo
    FROM book
    WHERE book.id = ?
    [...] (3,)

.. _orm_queryguide_mapper_deferred_raiseload:

配置默认的 mapper-level "raiseload" 行为
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. comment

    >>> class Base(DeclarativeBase):
    ...     pass

首先介绍的   :ref:`orm_queryguide_deferred_raiseload`  "raiseload" 行为也可以使用   :func:` _orm.mapped_column`  的  :paramref:`_orm.mapped_column.deferred_raiseload`  参数作为默认的映射级别实现。
使用此参数时，受影响的列将始终在访问时抛出异常，除非在查询时专门使用   :func:`_orm.undefer`  或   :func:` _orm.load_only`  取消延迟加载::

    >>> class Book(Base):
    ...     __tablename__ = "book"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     title: Mapped[str]
    ...     summary: Mapped[str] = mapped_column(Text, deferred=True, deferred_raiseload=True)
    ...     cover_photo: Mapped[bytes] = mapped_column(
    ...         LargeBinary, deferred=True, deferred_raiseload=True
    ...     )
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Book(id={self.id!r}, title={self.title!r})"

使用上述映射时，默认情况下不会加载 ``.summary`` 和 ``.cover_photo`` 列::

    >>> book = session.scalar(select(Book).where(Book.id == 2))
    {execsql}SELECT book.id, book.owner_id, book.title
    FROM book
    WHERE book.id = ?
    [...] (2,)
    {stop}>>> book.summary
    Traceback (most recent call last):
    ...
    sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True

只有通过在查询时重写其行为，通常使用   :func:`_orm.undefer`  或   :func:` _orm.undefer_group` ，
或者不常规使用   :func:`_orm.defer` ，才能加载这些属性。
下面的示例将 ``undefer('*')`` 应用于取消延迟加载所有属性，并使用   :ref:`orm_queryguide_populate_existing`  刷新已加载的对象的加载选项::

    >>> book = session.scalar(
    ...     select(Book)
    ...     .where(Book.id == 2)
    ...     .options(undefer("*"))
    ...     .execution_options(populate_existing=True)
    ... )
    {execsql}SELECT book.id, book.owner_id, book.title, book.summary, book.cover_photo
    FROM book
    WHERE book.id = ?
    [...] (2,)
    {stop}>>> book.summary
    'another long summary'



.. _orm_queryguide_with_expression:

将任意SQL表达式加载到对象上
--------------------------------------------

.. comment

    >>> class Base(DeclarativeBase):
    ...     pass
    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str]
    ...     fullname: Mapped[Optional[str]]
    ...     books: Mapped[List["Book"]] = relationship(back_populates="owner")
    ...
    ...     def __repr__(self) -> str:
    ...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"
    >>> class Book(Base):
    ...     __tablename__ = "book"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     title: Mapped[str]
    ...     summary: Mapped[str] = mapped_column(Text)
    ...     cover_photo: Mapped[bytes] = mapped_column(LargeBinary)
    ...     owner: Mapped["User"] = relationship(back_populates="books")
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Book(id={self.id!r}, title={self.title!r})"


如   :ref:`orm_queryguide_select_columns`  和其他地方所述，
  :func:`.select`  构造可以用于在结果集中加载任意的 SQL 表达式。例如，如果我们想要发出一个查询来加载 ` User` 对象，同时也包括每个 `User` 拥有的书籍数量的计数，我们可以使用 `func.count(Book.id)` 向一个包含 Join 到 `Book` 的查询中添加一个"count"列，以及一个根据拥有者 id 分组的 GROUP BY。这将产生   :class:`.Row`  对象，每个对象包含两个条目，一个是 ` User`，另一个是 `func.count(Book.id)`。

::

    >>> from sqlalchemy import func
    >>> stmt = select(User, func.count(Book.id)).join_from(User, Book).group_by(Book.owner_id)
    >>> for user, book_count in session.execute(stmt):
    ...     print(f"Username: {user.name}  Number of books: {book_count}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname,
    count(book.id) AS count_1
    FROM user_account JOIN book ON user_account.id = book.owner_id
    GROUP BY book.owner_id
    [...] ()
    {stop}Username: spongebob  Number of books: 3
    Username: sandy  Number of books: 3

在上面的例子中，`User` 实体和"book count" SQL 表达式是分别返回的。但是，一个流行的用例是生成一个查询，它将仅返回 `User` 对象，这些对象可以被迭代，例如使用  :meth:`_orm.Session.scalars` ，其中 ` func.count(Book.id)` SQL 表达式将被 *动态*地应用到每个 `User` 实体上。最终结果将类似于将任意的 SQL 表达式映射到类中，但 SQL 表达式可以在查询时被修改。对于这种用例，SQLAlchemy 提供了   :func:`_orm.with_expression`  加载选项，当与映射器级别的   :func:` _orm.query_expression`  指令结合使用时，可以产生这个结果。

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


要应用   :func:`_orm.with_expression`  到查询，映射类必须预先使用   :func:` _orm.query_expression`  指令配置一个 ORM 映射属性；该指令将产生一个适合接收查询时 SQL 表达式的映射类属性。下面我们在 `User` 中添加一个新属性 `User.book_count`。这个 ORM 映射属性是只读的，没有默认值；在已经加载的实例上访问它通常会产生 `None`。

::

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

使用 ``User.book_count`` 属性配置我们的映射后，我们可以使用   :func:`_orm.with_expression`  加载器选项将自定义 SQL 表达式应用于每个加载的 ` User` 对象中::

    >>> from sqlalchemy.orm import with_expression
    >>> stmt = (
    ...     select(User)
    ...     .join_from(User, Book)
    ...     .group_by(Book.owner_id)
    ...     .options(with_expression(User.book_count, func.count(Book.id)))
    ... )
    >>> for user in session.scalars(stmt):
    ...     print(f"Username: {user.name}  Number of books: {user.book_count}")
    {execsql}SELECT count(book.id) AS count_1, user_account.id, user_account.name,
    user_account.fullname
    FROM user_account JOIN book ON user_account.id = book.owner_id
    GROUP BY book.owner_id
    [...] ()
    {stop}Username: spongebob  Number of books: 3
    Username: sandy  Number of books: 3

上面，我们将我们的 `func.count(Book.id)` 表达式从   :func:`_sql.select`  构造的 columns 参数中移到   :func:` _orm.with_expression`  加载器选项中。ORM 然后将其视为一个特殊的列加载选项，它在运行时动态应用到语句中。

注意：

* 对于没有使用   :func:`_orm.with_expression`  来填充这个属性的对象，对象实例上的属性值将为 ` None`，除非映射中  :paramref:`_orm.query_expression.default_expr`  参数被设置为默认的 SQL 表达式。*   :func:` _orm.with_expression`  的值 **不会填充已经加载的对象**，除非使用   :ref:`orm_queryguide_populate_existing` 。下面的示例不起作用，因为 ` `A`` 对象已经加载：

  .. sourcecode:: python

    # 加载第一个 A
    obj = session.scalars(select(A).order_by(A.id)).first()

    # 加载相同带选项的 A；不会应用已经加载的对象上的表达式
    obj = session.scalars(select(A).options(with_expression(A.expr, some_expr))).first()

  为确保重新加载现有对象上的属性，请使用   :ref:`orm_queryguide_populate_existing`  执行选项以确保重新填充所有列：

  .. sourcecode:: python

    obj = session.scalars(
        select(A)
        .options(with_expression(A.expr, some_expr))
        .execution_options(populate_existing=True)
    ).first()

* 当对象过期时，无法保存此 SQL 表达式，无论是通过  :meth:`.Session.expire`  还是通过  :meth:` .Session.commit`  的 expire_on_commit 行为。一旦对象过期，SQL 表达式及其值将不再与属性关联，随后的访问将返回 ``None``。

*   :func:`_orm.with_expression`  作为对象加载选项，仅在查询的 **最外层部分** 作用，并且仅针对针对完整实体的查询，而不是针对任意列选择、子查询或复合语句（如 UNION） 的查询。参见下一节   :ref:` orm_queryguide_with_expression_unions` ，了解示例。

* 映射属性无法应用于查询的其他部分，例如 WHERE 子句、ORDER BY 子句，并利用临时表达式；也就是说，以下代码不起作用：

  .. sourcecode:: python

    # 无法在查询的其他地方引用 A.expr 表达式
    stmt = (
        select(A)
        .options(with_expression(A.expr, A.x + A.y))
        .filter(A.expr > 5)
        .order_by(A.expr)
    )

  上述 WHERE 子句和 ORDER BY 子句中的 ``A.expr`` 表达式将解析为 NULL。要在整个查询中使用表达式，请先将其分配给变量，然后使用：

  .. sourcecode:: python

    # 预先指定期望的表达式，然后在查询中引用
    a_expr = A.x + A.y
    stmt = (
        select(A)
        .options(with_expression(A.expr, a_expr))
        .filter(a_expr > 5)
        .order_by(a_expr)
    )

.. seealso::

      :func:`_orm.with_expression`  选项是一种特殊选项，用于在查询时动态应用 SQL 表达式到映射类。对于在映射器上配置的普通固定 SQL 表达式，请参见   :ref:` mapper_sql_expressions`  部分。

.. _orm_queryguide_with_expression_unions:

在 UNION 和其他子查询中使用 ``with_expression()``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. comment

  >>> session.close()

  :func:`_orm.with_expression`  构造是 ORM 加载器选项，因此仅可应用于 SELECT 语句的最外层，这些语句将载入特定的 ORM 实体。如果在   :func:` _sql.select`  中使用，将作为子查询或作为 UNION 等复合语句的元素使用，则不起作用。

为了在子查询中使用任意 SQL 表达式，应使用添加表达式的常规 Core 样式方法。要将子查询派生的表达式组合到 ORM 实体的   :func:`_orm.query_expression`  属性上，需要在 ORM 对象加载的顶层使用   :func:` _orm.with_expression` ，引用子查询中的 SQL 表达式。

在下面的示例中，使用两个带有附加 SQL 表达式标记为“expr”的   :func:`_sql.select`  构造针对 ORM 实体 ` `A`` 进行操作，并使用   :func:`_sql.union_all`  进行组合。然后，在最高层次上，使用   :ref:` orm_queryguide_unions`  中描述的查询方法从此 UNION 中 SELECT 了 ``A`` 实体。在新加载的 ``A`` 实例上添加选项来使用   :func:`_orm.with_expression`  提取此 SQL 表达式::

    >>> from sqlalchemy import union_all
    >>> s1 = (
    ...     select(User, func.count(Book.id).label("book_count"))
    ...     .join_from(User, Book)
    ...     .where(User.name == "spongebob")
    ... )
    >>> s2 = (
    ...     select(User, func.count(Book.id).label("book_count"))
    ...     .join_from(User, Book)
    ...     .where(User.name == "sandy")
    ... )
    >>> union_stmt = union_all(s1, s2)
    >>> orm_stmt = (
    ...     select(User)
    ...     .from_statement(union_stmt)
    ...     .options(with_expression(User.book_count, union_stmt.selected_columns.book_count))
    ... )
    >>> for user in session.scalars(orm_stmt):
    ...     print(f"Username: {user.name}  Number of books: {user.book_count}")
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname, count(book.id) AS book_count
    FROM user_account JOIN book ON user_account.id = book.owner_id
    WHERE user_account.name = ?联合所有
-------------------

.. 以下内容是SQL代码实例：

::

    UNION ALL
    SELECT user_account.id, user_account.name, user_account.fullname, count(book.id) AS book_count
    FROM user_account JOIN book ON user_account.id = book.owner_id
    WHERE user_account.name = ?
    [...] ('spongebob', 'sandy'){stop}
    用户名: spongebob  书籍数量: 3
    用户名: sandy  书籍数量: 3


列加载 API
-------------------

.. autofunction:: defer

.. autofunction:: deferred

.. autofunction:: query_expression

.. autofunction:: load_only

.. autofunction:: undefer

.. autofunction:: undefer_group

.. autofunction:: with_expression

.. comment

  >>> session.close()
  >>> conn.close()
  回滚...