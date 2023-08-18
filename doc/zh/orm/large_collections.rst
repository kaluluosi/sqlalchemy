.. highlight:: pycon+sql
.. doctest-enable

.. currentmodule:: sqlalchemy.orm

.. _largecollections:

处理大集合
==============================

默认情况下，:func:`_orm.relationship`的行为是完全加载在内存中的集合内容，基于一个配置的
:ref:`loader策略 <orm_queryguide_relationship_loaders>`来控制这些内容的何时和如何从数据库加载。
相关集合可能不仅在访问它们时或主动加载数据时被加载到内存中，而且在集合本身发生变化时也需要
加载数据, 或者在从工作单元通当中删除拥有对象的情况下都需要加载数据。

当关联集合可能非常大时，可能根本不可能在任何情况下都将这样的集合加载到内存中，因为这个
操作可能会大量消耗时间、网络和内存资源。

本章节包括旨在允许：func: `_orm.relationship` 与大集合一同使用保持足够性能的API特点。

.. _write_only_relationship:

只写关系
------------------------

**只写**加载策略是配置 :func:`_orm.relationship` 不加载它的内容到内存中的主要手段。现代化的
类型注解声明形式的Declarative形式如下所例：
：


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


在上面的代码里面，``account_transactions``关系不使用普通的:class:`.Mapped`注释配置，而是使用
:class:`.WriteOnlyMapped`类型的注释配置，在运行时将``lazy="``配置为"write_only"。``.WriteOnlyMapped``
注释是:class:`_orm.Mapped`注释的另一种形式，它表示在对象的实例上使用:class:`_orm.WriteOnlyCollection`集合类型。

上述:func:`_orm.relationship`配置还包括具体的一些元素，它们决定了在删除 "Account"对象以及从 “account_transactions”集合中删除"AccountTransaction"对象时要执行的操作:

* ``passive_deletes=True`` - 允许工作单元在删除“Account”时不必加载集合；请参见:ref:`passive_deletes`。
* ``ondelete="cascade"``在 :class:`.ForeignKey`约束上配置。这也在:ref:`passive_deletes`详述。
* ``cascade="all, delete-orphan"`` - 指示工作单元在从集合中删除“AccountTransaction”时删除它们。在:ref:`unitofwork_cascades`文档中
  参见 :ref:`cascade_delete_orphan`。

.. versionadded:: 2.0 添加了 "Write only" 技术支持.


创建和持久化新的write_only集合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现在可以直接将write-only集合作为整体分配为 :term:`transient` 或 :term:`pending` 对象。
使用以上映射代码，在:class:`_orm.Session`中可以创建新 ``Account``对象，
一系列 ``AccountTransaction``对象。 可以使用任何Python可迭代对象作为
开始对象的来源，正如下面我们使用Python ``list``那样::

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


一旦对象被写为数据库持久状态(即处于 :term:`persistent` 或 :term:`detached` 状态)，
集合就可以扩展了，旧的和新的条目以及个别条目的记录也都可以删除。但是，
集合可能**不能再被重新分配替换为整个集合**，因为这种修改操作需要前一个集合被完全加载到
内存中，才能将旧值和新值调和。

把新项目添加到现有集合中
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于 :term:`persistent` 对象，只有通过使用 :term:`unit of work` 进程才能进行 集合的修改，才可以对 write-only 集合添加新条目等操作；

使用 :meth:`.WriteOnlyCollection.add`,
:meth:`.WriteOnlyCollection.add_all` and :meth:`.WriteOnlyCollection.remove`
方法::

    >>> from sqlalchemy import select
    >>> session = Session(engine, expire_on_commit=False)
    >>> existing_account = session.scalar(select(Account).filter_by(identifier="account_01"))
    {execsql}BEGIN (implicit)
    SELECT account.id, account.identifier
    FROM account
    WHERE account.identifier = ?
    [...] ('account_01',)
    {stop}
    >>> existing_account.account_transactions.add_all(
    ...     [
    ...         AccountTransaction(description="paycheck", amount=Decimal("2000.00")),
    ...         AccountTransaction(description="rent", amount=Decimal("-800.00")),
    ...     ]
    ... )
    >>> session.commit()
    {execsql}INSERT INTO account_transaction (account_id, description, amount, timestamp)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
    [... (insertmanyvalues) 1/2 (ordered; batch not supported)] (1, 'paycheck', 2000.0)
    INSERT INTO account_transaction (account_id, description, amount, timestamp)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
    [insertmanyvalues 2/2 (ordered; batch not supported)] (1, 'rent', -800.0)
    COMMIT


上述添加的条目被保存在:class:`_orm.Session`中之后，就像存在于 :term:`transient`状态下的对象一样，
随后如果要删除对象后，则使用:meth:`.WriteOnlyCollection.remove`方法，工作单元过程会默认该对象
已经是集合的一部分。以删除一个单独的AccountTransaction条目为例，因为我们的 :ref:`cascade <unitofwork_cascades>`设置需要，
被删除的行被修整。

  >>> existing_transaction = account_transactions[0]
  >>> existing_account.account_transactions.remove(existing_transaction)
  >>> session.commit()
  {execsql}DELETE FROM account_transaction WHERE account_transaction.id = ?
  [...] (3,)
  COMMIT

对于任何的ORM-mapped集合，可以通过删除对象将对象与集合分离。对于 one-to-many关系模式 ，这种情况下键值可以设置为 :const:`None`；对于 many-to-many 关系模式，特定的联合项将被删除。

删除记录
~~~~~~~~~~~~~~

WriteOnlyCollection 对象 remove 的协议与常规 orm 集合相同：如果 project_transaction 已经存在于 session 中，传统的 orm 逻辑将项目从 Account.transactions集合中删除：

jack = session.query(Account).filter_by(name='jack').one()

proj = session.query(Project).filter_by(name='proj3').one()

jack.projects.remove(proj)

如果没有一个项目驻留在 session 中（如下文所述），则只需将其设置为可删除即可对其进行 remove，就像写操作一样：

proj = Project(name='other project')

jack.projects.remove(proj)

Querying Items
~~~~~~~~~~~~~~

:class:`_orm.WriteOnlyCollection` 没有在任何时间点存储对集合当前内容的引用，
也没有任何行为会直接发出 SELECT 查询以将其加载到内存中；
这种方法的默认设置是，集合可以包含很多千甚至百万行，因此不应作为其他操作的副作用
的内存中预加载。

:class:`_orm.WriteOnlyCollection` 包括SQL生成助手，例如:meth:`_orm.WriteOnlyCollection.select`，
该助手将预先配置带有当前父行的 WHERE / FROM条件的:class:`.Select`构造，
然后可以进一步修改以选择所需的任何范围的行，也可使用 :ref:`server side cursors <orm_queryguide_yield_per>`
用于需要以节省内存占用的方式迭代整个集合的进程。

下面是 SQLite 下生成的语句。请注意，它还包括 ORDER BY 条件，这由 :func:`_orm.relationship` 的参数 :paramref:`_orm.relationship.order_by` 指示，
如果未配置该参数，则将省略此条件::


    >>> print(existing_account.account_transactions.select())
    {printsql}SELECT account_transaction.id, account_transaction.account_id, account_transaction.description,
    account_transaction.amount, account_transaction.timestamp
    FROM account_transaction
    WHERE :param_1 = account_transaction.account_id ORDER BY account_transaction.timestamp

我们可以使用 :class:`.Select` 构造和 :class:`_orm.Session` 进行查询 `AccountTransaction` 对象，
最简单的方法是使用会返回 ORM 对象的 :meth:`_orm.Session.scalars` 方法。典型的做法是，
尽管不是必需的，但会进一步修改 :class:`.Select` 来限制返回的记录； 在下面的示例中，添加了其他 WHERE 条件来仅加载 “debit” 帐户交易记录，
以及“LIMIT 10” 来仅检索这前十个行：

    >>> account_transactions = session.scalars(
    ...     existing_account.account_transactions.select()
    ...     .where(AccountTransaction.amount < 0)
    ...     .limit(10)
    ... ).all()
    {execsql}
    BEGIN (implicit)
    SELECT account_transaction.id, account_transaction.account_id, account_transaction.description,
    account_transaction.amount, account_transaction.timestamp
    FROM account_transaction
    WHERE ? = account_transaction.account_id AND account_transaction.amount < ?
    ORDER BY account_transaction.timestamp  LIMIT ? OFFSET ?
    [...] (1, 0, 10, 0)
    {stop}
    >>> print(account_transactions)
    [AccountTransaction(amount=-29.50, timestamp='...'), AccountTransaction(amount=-800.00, timestamp='...')]

Bulk INSERT of New Items
~~~~~~~~~~~~~~~~~~~~~~~~

WriteOnlyCollection 能够 generating DML constructs 像 :class:`_dml.Insert` 对象，
它可在 ORM 的上下文中产生 bulk insert 行为。详见 ORM bulk 插入章节的概述。
仅正常的 one to many 集合的情况下,方法:meth:`.WriteOnlyCollection.insert`
生成一个 :class:`_dml.Insert` 构造器，其用于主键的 VALUES 条件与父对象相应。
由于此 VALUES 条件仅针对关联表格，因此该语句将同时插入新行，并将成为新记录添加到相关集合中::

  >>> session.execute(
  ...     existing_account.account_transactions.insert(),
  ...     [
  ...         {"description": "transaction 1", "amount": Decimal("47.50")},
  ...         {"description": "transaction 2", "amount": Decimal("-501.25")},
  ...         {"description": "transaction 3", "amount": Decimal("1800.00")},
  ...         {"description": "transaction 4", "amount": Decimal("-300.00")},
  ...     ],
  ... )
  {execsql}BEGIN (implicit)
  INSERT INTO account_transaction (account_id, description, amount, timestamp) VALUES (?, ?, ?, CURRENT_TIMESTAMP)
  [...] [(1, 'transaction 1', 47.5), (1, 'transaction 2', -501.25), (1, 'transaction 3', 1800.0), (1, 'transaction 4', -300.0)]
  <...>
  {stop}
  >>> session.commit()
  COMMIT

.. seealso::

    :ref:`orm_queryguide_bulk_insert` - in the :ref:`queryguide_toplevel`

    :ref:`relationship_patterns_o2m` - at :ref:`relationship_patterns`


Many to Many Collections
^^^^^^^^^^^^^^^^^^^^^^^^

一种 **many to many 集合** 的情况下，描述了两个类之间的关系涉及到一个第三个配置使用 :paramref:`_orm.relationship.secondary` 参数的表格。
使用 :class:`.WriteOnlyCollection` 插入此类集合的新行时，可能首先单独对其进行批量插入，并使用 RETURNING 检索到这些记录，
然后将这些记录传递给 :meth:`.WriteOnlyCollection.add_all` 方法，此时单元操作过程会将这些记录作为集合的一部分来持久化。

例如，假设一个类 ``BankAudit``  指向许多 ``AccountTransaction`` 记录，这是一个多对多表格::

    >>> from sqlalchemy import Table, Column
    >>> audit_to_transaction = Table(
    ...     "audit_transaction",
    ...     Base.metadata,
    ...     Column("audit_id", ForeignKey("audit.id", ondelete="CASCADE"), primary_key=True),
    ...     Column(
    ...         "transaction_id",
    ...         ForeignKey("account_transaction.id", ondelete="CASCADE"),
    ...         primary_key=True,
    ...     ),
    ... )
    >>> class BankAudit(Base):
    ...     __tablename__ = "audit"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
    ...         secondary=audit_to_transaction, passive_deletes=True
    ...     )

.. setup code not for display

    >>> Base.metadata.create_all(engine)
    BEGIN...

我们可以通过批量插入添加更多的 ``AccountTransaction`` 对象，
通过调用 ``existing_account.account_transactions.insert()``，添加方法返回一个 :class:`_dml.Insert` 构造器，其用于与父对象相应的 VALUES 条件。

通过 ``session.execute()`` 执行该方法，并传入带有预先绑定的参数序列的元素列表来批量插入（或插入已存在于表格中的现有 ``AccountTransaction`` 对象）
此处的 :const:`RETURNING` 选项被添加到语法以使用 :func:`sqlalchemy.orm.scalars` 函数捕获 插入/返回的所有行：

  >>> new_transactions = session.scalars(
  ...     existing_account.account_transactions.insert().returning(AccountTransaction),
  ...     [
  ...         {"description": "odd trans 1", "amount": Decimal("50000.00")},
  ...         {"description": "odd trans 2", "amount": Decimal("25000.00")},
  ...         {"description": "odd trans 3", "amount": Decimal("45.00")},
  ...     ],
  ... ).all()
  {execsql}BEGIN (implicit)
  INSERT INTO account_transaction (account_id, description, amount, timestamp) VALUES
  (?, ?, ?, CURRENT_TIMESTAMP), (?, ?, ?, CURRENT_TIMESTAMP), (?, ?, ?, CURRENT_TIMESTAMP)
  RETURNING id, account_id, description, amount, timestamp
  [...] (1, 'odd trans 1', 50000.0, 1, 'odd trans 2', 25000.0, 1, 'odd trans 3', 45.0)
  {stop}

与许多针对于持久状态对象的 orm 集合的情况相同，orm 集合可以通过 :meth:`.WriteOnlyCollection.update` 和 :meth:`.WriteOnlyCollection.delete` 对象快捷构造进行更新和修改。

WriteOnlyCollection支持 bulk update 和 delete 通过生成诸如 :class:`.Update` 和 :class:`.Delete` 构造器，其中包括 WHERE 条件，以便针对集合中的元素进行 update 和 delete 操作。

最后 :class:`_orm.WriteOnlyCollection` 能够通过 :meth:`_orm.WriteOnlyCollection.select()` 方法生成用于此查询的 SQL 部分，然后可以将其使用于外层 SELECT 语句中，以便能够针对一组较大的行进行现有的过滤操作，基于与选定的父项匹配的子项来选择行。
有关详细信息，请参阅 :class:`_orm.Query` 以及 :ref:`subquery <sql_expression_subquery>`指南。您可以在此查询中定义各种限制和过滤条件，以便只查询某些行::

    >>> from sqlalchemy import update
    >>> subq = bank_audit.account_transactions.select().with_only_columns(AccountTransaction.id)
    >>> session.execute(
    ...     update(AccountTransaction)
    ...     .values(description=AccountTransaction.description + " (audited)")
    ...     .where(AccountTransaction.id.in_(subq))
    ... )
    {execsql}UPDATE account_transaction SET description=(account_transaction.description || ?)
    WHERE account_transaction.id IN (SELECT account_transaction.id
    FROM audit_transaction
    WHERE ? = audit_transaction.audit_id AND account_transaction.id = audit_transaction.transaction_id)
    RETURNING id
    [...] (' (audited)', 1)
    <...>

要更新或删除其中一个组，最好使用一个类似于如下代码的子查询判断Related名字是 "foo" 的对象::

    from sqlalchemy.sql import exists, select

    audit_alias = select([audit.c.id]).where(audit.c.name == "audit1").scalar_subquery()

    q = (
        select([account_transactions.c.id])
        .where(
            exists()
            .where(account_transactions_audit_transaction.c.audit_id == audit_alias)
            .where(account_transactions.c.id == account_transactions_audit_transaction.c.transaction_id)
        )
        .subquery()
    )

    stmt = (
        account_transactions.update()
        .values(description=account_transactions.c.description + " (audited)")
        .where(account_transactions.c.id.in_(q))
    )

BulkUPDATE 的多个例子详见 :ref:`updating_multiple_rows`。同时在示例中， ORM 的查询操作对 write-only 集合做了操作，但是并不会执行额外的懒惰加载。

Write Only Collections - API Documentation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. autoclass:: sqlalchemy.orm.WriteOnlyCollection
    :members:
    :inherited-members:

.. autoclass:: sqlalchemy.orm.WriteOnlyMapped
    :members:

.. _dynamic_relationship:

动态关系加载器
----------------------------

.. legacy::  “dynamic” lazy loader strategy是现在称为 “write_only” 策略的旧式形式。

   "dynamic" 策略会配置 :func:`_orm.relationship`，在实例上访问它时返回一个'orm.Query'对象，而不是集合。
   :class:`_orm.Query` 然后可以进行进一步的修改，以便可以基于过滤条件来迭代数据库集合。返回的 :class:`_orm.Query` 对象是 :class:`_orm.AppenderQuery`的一个实例，它将加载和迭代 :class:`_orm.Query` 的行为与基本的集合修改方法（如 :meth:`_orm.AppenderQuery.append` 和 :meth:`_orm.AppenderQuery.remove` 等）组合在一起。

   "dynamic" 加载程序还与 :ref:`asyncio_toplevel` 扩展不兼容，虽然可以使用一些限制，但必须根据案例进行小心的编程和测试。因此，在处理真正的大的集合管理时，应该优选 :class:`.WriteOnlyCollection`。

动态关系加载策略允许配置一个 :func:`_orm.relationship`，当在实例上访问它时，它将返回 :class:`_orm.Query` 对象而不是集合。这个 :class:`_orm.Query` 可以进一步地修改以便数据库集合可以基于篮子条件进行迭代。返回的 :class:`_orm.Query` 对象是实例级别的 :class:`_orm.AppenderQuery`，组合了 :class:`_orm.Query` 的加载和迭代行为，以及 :meth:`_orm.AppenderQuery.append` 和 :meth:`_orm.AppenderQuery.remove` 等简单的集合修改方法。

"dynamic" 关系加载器可以使用 DynamicMapped 注释类进行配置::

    from sqlalchemy.orm import DynamicMapped

    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        posts: DynamicMapped[Post] = relationship()

以上，一个单独的 "User.posts" 集合属性将返回的是一个 :class:`_orm.AppenderQuery` 对象，这是 :class:`_orm.Query` 的子类，支持基本的修改集合操作，例如 :meth:`_orm.AppenderQuery.append` 和 :meth:`_orm.AppenderQuery.remove`。

动态关系支持有限的写操作，可以通过 :meth:`_orm.AppenderQuery.append` 和 :meth:`_orm.AppenderQuery.remove` 方法进行操作。

由于动态关系的访问总是会向数据库查询，此原因导致对于基本联系人分组已经进行豁免了，因此修改底层集合的更改在数据记录刷新之前将不可见。但只要 :class:`.Session` 上设置了自动flash，刷新操作就会自动执行每次集合查询返回之前。

Dynamic Relationship Loaders - API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. autoclass:: sqlalchemy.orm.AppenderQuery
    :members:
    :inherited-members: Query

.. autoclass:: sqlalchemy.orm.DynamicMapped
    :members:

.. _collections_raiseload:

设置RaiseLoad
-----------------

“raise”加载关系将在通常情况下作为一个懒惰加入发出属性时，抛出一个
:exc:`~sqlalchemy.exc.InvalidRequestError` 异常::

    class MyClass(Base):
        __tablename__ = "some_table"

        # ...

        children: Mapped[List[MyRelatedClass]] = relationship(lazy="raise")

在上述代码中，如果之前没有填充children，则访问属性将引发异常。这包括读取访问，但对于集合来说也影响了写访问，因为集合不能在不加载它们的情况下进行变更。这样做的原因是确保应用程序在某个上下文中不会发出任何意外的懒惰加载。使用raise策略
-----------------

为了避免阅读SQL日志以确定是否已经提前加载了所有必要的属性，"raise"策略将导致如果访问时未加载属性的话会立即抛出异常。 使用:func: `_orm.raiseload`加载选项的查询选项也可以基于raise策略。

.. 参见::

    :ref:`prevent_lazy_with_raiseload`

使用被动删除
---------------------

SQLAlchemy中集合管理的重要方面是，当引用集合的对象被删除时，SQLAlchemy需要考虑到位于该集合内部的对象。这些对象需要从父对象中解除关联，对于一个一对多的集合来说，这意味着外键列将设置为NULL，或者基于:ref:`cascade <unitofwork_cascades>`设置，可能希望为这些行发出一个DELETE语句。

:term:`工作单元`进程只考虑逐行操作对象，这意味着DELETE操作意味着集合中的所有行必须在刷新过程中完全加载到内存中。对于大型集合来说，这是不可行的，所以我们将寻求依赖于数据库自己的能力来自动更新或删除行，使用外键ON DELETE规则，指导工作单元在处理时不需要实际加载这些行。可以通过在:func: `_orm.relationship`构造上配置:paramref:`_orm.relationship.passive_deletes`来指示工作单元以这种方式工作;同时要正确配置正在使用的外键约束。

有关完整的"passive delete"配置的进一步细节，请参见:ref:`passive_deletes`节。