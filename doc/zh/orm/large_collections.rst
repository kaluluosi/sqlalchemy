.. highlight:: pycon+sql
.. doctest-enable

.. currentmodule:: sqlalchemy.orm

.. _largecollections:

操作大型集合
==============

默认情况下，  :func:`_orm.relationship`  来控制何时以及如何从数据库中加载这些内容。相关集合可能不仅在访问它们时被加载到内存中（即急切地预加载），而且在大多数情况下，在集合本身被改变以及属于工作单元系统的对象被删除时都需要加载这些内容。

如果一个相关集合可能非常大，可能在任何情况下都不能将这样的集合加载到内存中，因为操作可能会对时间、网络和内存资源产生过多的消耗。

本节包括的 API 功能旨在允许在保持足够性能的同时使用   :func:`_orm.relationship`  操作大型集合。


.. _write_only_relationship:

只写关系
---------

**只写**加载器策略是配置   :func:`_orm.relationship`  的主要方法，该方法将保持可写性，但不会将其内容加载到内存中。下面是一个现代声明式的 Declarative 形式的只写 ORM 配置：

.. sourcecode:: python

    >>> from decimal import Decimal
    >>> from datetime import datetime

    >>> from sqlalchemy import ForeignKey
    >>> from sqlalchemy import func
    >>> from sqlalchemy.orm import DeclarativeBase
    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import mapped_column
    >>> from sqlalchemy.orm import relationship
    >>> from sqlalchemy.orm import Session
    >>> from sqlalchemy.orm import WriteOnlyMapped

    >>> class Base(DeclarativeBase):
    ...     pass

    >>> class Account(Base):
    ...     __tablename__ = "account"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     identifier: Mapped[str]
    ...
    ...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
    ...         cascade="all, delete-orphan",
    ...         passive_deletes=True,
    ...         order_by="AccountTransaction.timestamp",
    ...     )
    ...
    ...     def __repr__(self):
    ...         return f"Account(identifier={self.identifier!r})"

    >>> class AccountTransaction(Base):
    ...     __tablename__ = "account_transaction"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     account_id: Mapped[int] = mapped_column(
    ...         ForeignKey("account.id", ondelete="cascade")
    ...     )
    ...     description: Mapped[str]
    ...     amount: Mapped[Decimal]
    ...     timestamp: Mapped[datetime] = mapped_column(default=func.now())
    ...
    ...     def __repr__(self):
    ...         return (
    ...             f"AccountTransaction(amount={self.amount:.2f}, "
    ...             f"timestamp={self.timestamp.isoformat()!r})"
    ...         )
    ...
    ...     __mapper_args__ = {"eager_defaults": True}

.. setup code not for display

    >>> from sqlalchemy import create_engine
    >>> from sqlalchemy import event
    >>> engine = create_engine("sqlite://", echo=True)
    >>> @event.listens_for(engine, "connect")
    ... def set_sqlite_pragma(dbapi_connection, connection_record):
    ...     cursor = dbapi_connection.cursor()
    ...     cursor.execute("PRAGMA foreign_keys=ON")
    ...     cursor.close()

    >>> Base.metadata.create_all(engine)
    BEGIN...


以上的 ``account_transactions`` 关系不使用普通的   :class:`.Mapped`  注释进行配置，而是使用   :class:` .WriteOnlyMapped`  类型的注释，在运行时将把 ``lazy="write_only"`` 的   :ref:`loader strategy <orm_queryguide_relationship_loaders>`  分配给目标   :func:` _orm.relationship` 。   :class:`.WriteOnlyMapped`  注释是   :class:` _orm.Mapped`  注释的另一种形式，它指示在对象实例的集合上使用   :class:`_orm.WriteOnlyCollection`  集合类型。

上面的   :func:`_orm.relationship`  配置还包括几个特定于何时删除 ` `Account`` 对象以及何时从 ``account_transactions`` 集合中删除 ``AccountTransaction`` 对象的元素。这些元素是：

* ``passive_deletes=True`` - 允许  :term:`unit of work`  在删除“Account”时避免加载集合；请查看   :ref:` passive_deletes` 。
* 在  :class:`.ForeignKey`  约束上配置的 ` `ondelete="cascade"``。这也在   :ref:`passive_deletes`  中详细说明。
* ``cascade="all, delete-orphan"`` - 指示  :term:`unit of work`  在将 ` `AccountTransaction`` 对象从集合中删除时将其删除。请参见   :ref:`unitofwork_cascades`  文档中的   :ref:` cascade_delete_orphan` 。

.. versionadded:: 2.0  Added "Write only" relationship loaders.


创建和持久化新的只写集合
~~~~~~~~~~~~~~~~~~~~~~~~

只写集合仅允许直接将集合作为整体分配到   :class:`_orm.Session`  中的已挂起或待定的对象上。使用上面的映射，这表示我们可以创建一个新的 ` `Account`` 对象，并添加一系列 ``AccountTransaction`` 对象以添加到   :class:`_orm.Session`  中。任何 Python 可迭代对象都可以用作对象源，下面我们使用 Python ` `list`` 作为源：

    >>> new_account = Account(
    ...     identifier="account_01",
    ...     account_transactions=[
    ...         AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
    ...         AccountTransaction(description="transfer", amount=Decimal("1000.00")),
    ...         AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
    ...     ],
    ... )

    >>> with Session(engine) as session:
    ...     session.add(new_account)
    ...     session.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO account (identifier) VALUES (?)
    [...] ('account_01',)
    INSERT INTO account_transaction (account_id, description, amount, timestamp)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
    [... (insertmanyvalues) 1/3 (ordered; batch not supported)] (1, 'initial deposit', 500.0)
    INSERT INTO account_transaction (account_id, description, amount, timestamp)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
    [insertmanyvalues 2/3 (ordered; batch not supported)] (1, 'transfer', 1000.0)
    INSERT INTO account_transaction (account_id, description, amount, timestamp)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
    [insertmanyvalues 3/3 (ordered; batch not supported)] (1, 'withdrawal', -29.5)
    COMMIT


一旦对象已被持久化（即处于  :term:`persistent`  或  :term:` detached`  状态），集合就可以扩展以添加新条目，并且可以删除个别条目。但是，不再可以重新分配整个集合，因为此类操作需要将先前的集合完全加载到内存中，以便将旧条目和新条目相互协调：

    >>> new_account.account_transactions = [
    ...     AccountTransaction(description="some transaction", amount=Decimal("10.00"))
    ... ]
    Traceback (most recent call last):
    ...
    sqlalchemy.exc.InvalidRequestError: Collection "Account.account_transactions" does not
    support implicit iteration; collection replacement operations can't be used

向现有集合中添加新项目
~~~~~~~~~~~~~~~~~~~~~~~

对于持久对象的只写集合，使用  :term:`unit of work`  过程修改集合的修改只能通过使用下列方法之一进行：

.. _collections_raiseload:

设置 RaiseLoad
-----------------

使用“raise”加载的关系将在属性通常会发出惰性加载的位置抛出  :exc:`~sqlalchemy.exc.InvalidRequestError`  异常：

    class MyClass(Base):
        __tablename__ = "some_table"

        # ...

        children: Mapped[List[MyRelatedClass]] = relationship(lazy="raise")

以上，对 ``children`` 集合的属性访问将在先前未加载集合时引发异常。这包括读取访问，但对于集合来说，还会影响写入访问，因为无法在加载集合之前对集合进行更改。采用此策略的原因是为了确保应用程序在一定上下文中未发出任何意外的惰性加载。使用Raise Strategy
-------------------

与其阅读SQL日志来判断是否预加载了所有必要的属性，"raise"策略会在访问未加载的属性时立即引发异常。该策略也可在查询选项基础上使用，使用   :func:`_orm.raiseload`  loader option。

.. seealso::

      :ref:`prevent_lazy_with_raiseload` 

使用被动删除（Passive Deletes）
---------------------------

在SQLAlchemy中,与集合相关的对象被删除时，SQLAlchemy需要考虑该集合中包含的对象。这些对象需要从父对象取消关联，对于 "一对多" 集合，这意味着需将外键列设置为NULL，或根据   :ref:`cascade <unitofwork_cascades>`  设置需要发出删除这些行的DELETE。

在  :term:`工作单元 (unit of work)`  过程中，只考虑逐行地处理对象。 也就是说，DELETE 操作意味着集合中的所有行必须全部加载到内存中。对于大型集合，这是不可行的。因此，我们寻求依赖数据库自身的能力，通过外键 ON DELETE 规则自动更新或删除行，指示工作单元放弃实际上需要加载这些行以处理它们。可以通过在   :func:` _orm.relationship`  中配置  :paramref:`_orm.relationship.passive_deletes`  使工作单元采取这种方式，并确保正确配置了正在使用的外键约束。

有关完整的 "被动删除" 配置的更多详细信息，请参见   :ref:`passive_deletes` 。