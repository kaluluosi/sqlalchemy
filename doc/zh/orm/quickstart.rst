.. _orm_quickstart:


ORM快速入门
===========

对于想要快速了解ORM基本使用情况的新用户来说，这里简化了在   :ref:`unified_tutorial`  中使用的映射和示例。这里的代码可以从干净的命令行完全运行。

因为本节中的说明故意非常简短，请前往全面的   :ref:`unified_tutorial`  以获得更详细的介绍。

.. versionchanged:: 2.0 ORM Quickstart已更新为最新版本的  :pep:`484`  感知特性，使用了包括    内的新构造。请参阅  :ref:` whatsnew_20_orm_declarative_typing` 章节以获取迁移信息。

声明模型
--------

在这里，我们定义将形成我们将从数据库中查询的结构的模块级别构造。这种结构称为   :ref:`Declarative Mapping <orm_declarative_mapping>` ，一次定义了Python对象模型以及描述实际存在或将存在于特定数据库中的  :term:` 数据库元数据`  的实际SQL表：

    >>> from typing import List
    >>> from typing import Optional
    >>> from sqlalchemy import ForeignKey
    >>> from sqlalchemy import String
    >>> from sqlalchemy.orm import DeclarativeBase
    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import mapped_column
    >>> from sqlalchemy.orm import relationship

    >>> class Base(DeclarativeBase):
    ...     pass

    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str] = mapped_column(String(30))
    ...     fullname: Mapped[Optional[str]]
    ...
    ...     addresses: Mapped[List["Address"]] = relationship(
    ...         back_populates="user", cascade="all, delete-orphan"
    ...     )
    ...
    ...     def __repr__(self) -> str:
    ...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

    >>> class Address(Base):
    ...     __tablename__ = "address"
    ...
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     email_address: Mapped[str]
    ...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...
    ...     user: Mapped["User"] = relationship(back_populates="addresses")
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"

映射以一个基类开始，上面称为“Base”，通过对   :class:`_orm.DeclarativeBase`  类进行简单子类化创建。

然后，通过将子类化的方式创建单个映射类实例。映射类通常指代单个数据库表，其名称由使用“__tablename__”类级别属性指示。

接下来，通过添加属性来声明表中的列。每个属性的名称对应于将成为数据库表一部分的列。每个列的数据类型首先取决于与每个  :class:`_orm.Mapped` 注释关联的Python数据类型；例如，` ` INTEGER``为``int``，``VARCHAR``为``str``等。可通过右侧   :func:`_orm.mapped_column`  指令中调用SQLAlchemy类型对象来指定每个列的更具体的类型信息，例如上面在“User.name”列中使用的:String数据类型。Python类型和SQL类型之间的关联可以使用   :ref:` type annotation map <orm_declarative_mapped_column_type_map>`  进行自定义。

对于哪些需要更具体的自定义的基于列的属性，使用   :func:`_orm.mapped_column`  指令。除了类型信息外，此指令还接受各种参数，该参数指示有关数据库列的特定详细信息，包括服务器默认值和约束信息，例如主键和外键中的成员资格。  :func:` _orm.mapped_column`  指令接受   :class:`_schema.Column`  类接受的超集参数，该类由SQLAlchemy Core用于表示数据库列。

所有ORM映射类都需要将至少一个列声明为主键的一部分，通常通过在那些应该成为键的   :func:`_orm.mapped_column`  对象上使用  :paramref:` _schema.Column.primary_key`  参数来完成。在上面的示例中，“User.id”和 “Address.id”列被标记为主键。

总之，将字符串表名称和列声明列表组合在一起，称为在SQLAlchemy中的  :term:`table metadata`  。使用Core和ORM方法同时设置表格元数据在   :ref:` tutorial_working_with_metadata`  中进行介绍。上述映射是所谓的   :ref:`Annotated Declarative Table <orm_declarative_mapped_column>` 配置的例子。

其他   :class:`_orm.Mapped`  的变体可用，其中最常见的是上文提到的   :func:` _orm.relationship`  指令。与基于列的属性相反，  :func:`_orm.relationship`  指示了两个ORM类之间的链接。在上面的示例中，“User.addresses”链接“User”到“Address”，“Address.user”链接“Address”到“User”。  :func:` _orm.relationship`  指令在   :ref:`tutorial_orm_related_objects`  中进行了介绍。

最后，上述示例类包括一个“__repr __ ()”方法，这个方法并不是必需的，但对于调试很有用。可以使用生成的自动方法（例如__repr __()）使用dataclasses创建映射类。有关dataclass映射的详细信息，请参见：ref：`orm_declarative_native_dataclasses`。

创建引擎
----------

  :class:`_engine.Engine`  是一个**工厂**，它可以为我们创建新的数据库连接，还可以在  :ref:` 连接池 <pooling_toplevel>`  中维护连接以供快速重用。为了方便起见，我们通常使用  :ref:`SQLite <sqlite_toplevel>`  内存数据库进行学习：

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite://", echo=True)

.. tip:: 

    参数 ``echo=True``表示将连接发出的SQL记录到标准输出流中。

完整介绍   :class:`_engine.Engine`  从   :ref:` tutorial_engine` 开始。

发出CREATE TABLE DDL
--------------------------

使用我们的表元数据和引擎，我们可以创建我们在目标SQLite数据库中的模式，使用方法  :meth:`_schema.MetaData.create_all`  ：

.. sourcecode:: pycon+sql

    >>> Base.metadata.create_all(engine)
    {execsql}BEGIN (implicit)
    PRAGMA main.table_...info("user_account")
    ...
    PRAGMA main.table_...info("address")
    ...
    CREATE TABLE user_account (
        id INTEGER NOT NULL,
        name VARCHAR(30) NOT NULL,
        fullname VARCHAR,
        PRIMARY KEY (id)
    )
    ...
    CREATE TABLE address (
        id INTEGER NOT NULL,
        email_address VARCHAR NOT NULL,
        user_id INTEGER NOT NULL,
        PRIMARY KEY (id),
        FOREIGN KEY(user_id) REFERENCES user_account (id)
    )
    ...
    COMMIT

从我们上面编写的Python代码中刚刚发生了很多事情。了解有关表元数据的完整概述，请参见   :ref:`tutorial_working_with_metadata` 。

创建对象并持久化
------------------------

我们现在准备在数据库中插入数据。我们通过创建``User``和``Address``类实例来完成这个过程，从而使用了“__init __()”方法，这是通过声明映射过程自动建立的。然后，我们通过使用称为  :class:`_orm.Session` 的对象将它们传递给数据库，该对象利用  :class:` _engine.Engine` 与数据库进行交互。  :meth:`_orm.Session.add_all`  方法用于同时添加多个对象，并且使用  :meth:` _orm.Session.commit`  方法将等待提交的更改刷新到数据库中，并随后提交当前的数据库事务，每当使用   :class:`_orm.Session`  时都会进行交互：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.orm import Session

    >>> with Session(engine) as session:
    ...     spongebob = User(
    ...         name="spongebob",
    ...         fullname="Spongebob Squarepants",
    ...         addresses=[Address(email_address="spongebob@sqlalchemy.org")],
    ...     )
    ...     sandy = User(
    ...         name="sandy",
    ...         fullname="Sandy Cheeks",
    ...         addresses=[
    ...             Address(email_address="sandy@sqlalchemy.org"),
    ...             Address(email_address="sandy@squirrelpower.org"),
    ...         ],
    ...     )
    ...     patrick = User(name="patrick", fullname="Patrick Star")
    ...
    ...     session.add_all([spongebob, sandy, patrick])
    ...
    ...     session.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [...] ('spongebob', 'Spongebob Squarepants')
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [...] ('sandy', 'Sandy Cheeks')
    INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
    [...] ('patrick', 'Patrick Star')
    INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
    [...] ('spongebob@sqlalchemy.org', 1)
    INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
    [...] ('sandy@sqlalchemy.org', 2)
    INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
    [...] ('sandy@squirrelpower.org', 2)
    COMMIT


.. tip::

    建议在上述与   :class:`_orm.Session`  相关的操作中像上面一样使用上下文管理器样式，即使用Python的` `with：``语句。  :class:`_orm.Session`  对象表示活动的数据库资源，因此最好确保在完成一系列操作时关闭它。在下一节中，我们将保持   :class:` _orm.Session`  打开，只是为了说明目的。

创建   :class:`_orm.Session`  的基础知识在   :ref:` tutorial_executing_orm_session`  中，更多内容请参见   :ref:`session_basics` 。

然后，在   :ref:`tutorial_inserting_orm`  中，介绍了一些基本的插入操作变体。

简单选择
----------

在数据库中有一些行后，下面是发出简单SELECT语句以加载一些对象的最简单形式。要创建SELECT语句，我们使用   :func:`_sql.select`  函数创建新的  :class:` _sql.Select` 对象，然后使用   :class:`_orm.Session`  调用该对象。在ORM对象中查询时通常有用的方法是  :meth:` _orm.Session.scalars`  方法，该方法将返回一个  :class:`_result.ScalarResult` 对象，该对象将遍历我们选择的ORM对象：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select

    >>> session = Session(engine)

    >>> stmt = select(User).where(User.name.in_(["spongebob", "sandy"]))

    >>> for user in session.scalars(stmt):
    ...     print(user)
    {execsql}BEGIN (implicit)
    SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name IN (?, ?)
    [...] ('spongebob', 'sandy'){stop}
    User(id=1, name='spongebob', fullname='Spongebob Squarepants')
    User(id=2, name='sandy', fullname='Sandy Cheeks')


上面的查询还使用了  :meth:`_sql.Select.where`  方法添加WHERE条件，并且使用了  :meth:` _sql.ColumnOperators.in_`  方法，该方法是所有SQLAlchemy类似列的构造的一部分，用于使用SQL IN操作员。

有关如何选择对象和单个列的更多详细信息，请参见   :ref:`tutorial_selecting_orm_entities` 。

SELECT与JOIN
-------------

在多个表之间进行查询是非常常见的，对于SQL来说，JOIN关键字是主要方式。   :class:`_sql.Select`  构造使用  :meth:` _sql.Select.join`  方法创建连接：

.. sourcecode:: pycon+sql

    >>> stmt = (
    ...     select(Address)
    ...     .join(Address.user)
    ...     .where(User.name == "sandy")
    ...     .where(Address.email_address == "sandy@sqlalchemy.org")
    ... )
    >>> sandy_address = session.scalars(stmt).one()
    {execsql}SELECT address.id, address.email_address, address.user_id
    FROM address JOIN user_account ON user_account.id = address.user_id
    WHERE user_account.name = ? AND address.email_address = ?
    [...] ('sandy', 'sandy@sqlalchemy.org')
    {stop}
    >>> sandy_address
    Address(id=2, email_address='sandy@sqlalchemy.org')

上面的查询说明了多个WHERE条件，这些条件会自动使用AND连接，并且说明了如何使用SQLAlchemy类似列的对象创建“等于”比较，该对象使用重写的Python方法  :meth:`_sql.ColumnOperators.__eq__`  生成SQL标准对象。

有关上述概念更多背景信息请参见：  :ref:`tutorial_select_where_clause`  和   :ref:` tutorial_select_join` 。

进行更改
----------

  :class:`_orm.Session`  对象与我们的ORM映射类 ` `User`` 和 ``Address`` 一起自动跟踪所做的更改，这些更改会导致在   :class:`_orm.Session`  下一次刷新时发出SQL语句。接下来，我们将更改与“sandy”关联的一个电子邮件地址，然后为“patrick”添加一个新电子邮件地址，在发出SELECT以检索“patrick”的行之后：

.. sourcecode:: pycon+sql

    >>> stmt = select(User).where(User.name == "patrick")
    >>> patrick = session.scalars(stmt).one()
    {execsql}SELECT user_account.id, user_account.name, user_account.fullname
    FROM user_account
    WHERE user_account.name = ?
    [...] ('patrick',)
    {stop}

    >>> patrick.addresses.append(Address(email_address="patrickstar@sqlalchemy.org"))
    {execsql}SELECT address.id AS address_id, address.email_address AS address_email_address, address.user_id AS address_user_id
    FROM address
    WHERE ? = address.user_id
    [...] (3,){stop}

    >>> sandy_address.email_address = "sandy_cheeks@sqlalchemy.org"

    >>> session.commit()
    {execsql}UPDATE address SET email_address=? WHERE address.id = ?
    [...] ('sandy_cheeks@sqlalchemy.org', 2)
    INSERT INTO address (email_address, user_id) VALUES (?, ?)
    [...] ('patrickstar@sqlalchemy.org', 3)
    COMMIT
    {stop}

请注意，当我们访问 ``patrick.addresses`` 时，会发出SELECT语句。这被称为  :term:`lazy load`  。在   :ref:` tutorial_orm_loader_strategies`  中介绍了使用更多或更少的SQL访问相关项的不同方式。

有关ORM数据操作的详细演练从：ref：`tutorial_orm_data_manipulation` 开始。

一些删除
----------

所有事情都必须结束，有些数据库行也是如此-这里是两种不同形式的删除的快速演示，基于具体用例，这两种形式都很重要。

首先，我们将从“sandy”用户中删除一个“Address”对象。下次   :class:`_orm.Session`  刷新时，这将导致行被删除。这是我们所配置的行为，称为   :ref:` cascade delete <cascade_delete>` 。我们可以通过使用  :meth:`_orm.Session.get` ，使用主键获取“sandy”对象的句柄，然后使用相应对象进行操作：

.. sourcecode:: pycon+sql

    >>> sandy = session.get(User, 2)
    {execsql}BEGIN (implicit)
    SELECT user_account.id AS user_account_id, user_account.name AS user_account_name, user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (2,){stop}

    >>> sandy.addresses.remove(sandy_address)
    {execsql}SELECT address.id AS address_id, address.email_address AS address_email_address, address.user_id AS address_user_id
    FROM address
    WHERE ? = address.user_id
    [...] (2,)

上面的最后一个SELECT是懒加载操作，以便可以加载 ``sandy.addresses`` 集合，以便我们可以删除 ``sandy_address`` 成员。有其他更适用的操作进行此系列操作，这样不会发出太多SQL。

我们可以选择发出已设置为移除的更改的DELETE SQL，而不提交事务，使用  :meth:`_orm.Session.flush`  方法：

.. sourcecode:: pycon+sql

    >>> session.flush()
    {execsql}DELETE FROM address WHERE address.id = ?
    [...] (2,)

接下来，我们将完全删除“patrick”用户。对于仅通过自身进行顶级删除的对象，我们使用  :meth:`_orm.Session.delete`  方法；该方法实际上不执行删除，而是设置要在下一次刷新时删除的对象。操作还将根据我们配置的级联选项传播到相关对象“Address”：

.. sourcecode:: pycon+sql

    >>> session.delete(patrick)
    {execsql}SELECT user_account.id AS user_account_id, user_account.name AS user_account_name, user_account.fullname AS user_account_fullname
    FROM user_account
    WHERE user_account.id = ?
    [...] (3,)
    SELECT address.id AS address_id, address.email_address AS address_email_address, address.user_id AS address_user_id
    FROM address
    WHERE ? = address.user_id
    [...] (3,)

在这个特定情况下，  :meth:`_orm.Session.delete`  方法发出了两个SELECT语句，即使它没有发出DELETE，这可能看起来令人惊讶。这是因为当方法去检查对象时，结果是发现` `patrick``对象已  :term:`expired` （当我们最后调用  :meth:` _orm.Session.commit`  时发生的），并且发出的SQL是为了在新事务中重新加载行。这是可选的，通常我们将在适用情况下将其关闭。

要说明行正在被删除，请参见提交：

.. sourcecode:: pycon+sql

    >>> session.commit()
    {execsql}DELETE FROM address WHERE address.id = ?
    [...] (4,)
    DELETE FROM user_account WHERE user_account.id = ?
    [...] (3,)
    COMMIT
    {stop}

在   :ref:`tutorial_orm_deleting`  中讨论ORM删除。  :ref:` session_expiring`  中介绍了过期对象的背景信息；级联在   :ref:`unitofwork_cascades`  中进行了详细讨论。

深入学习上述概念
-----------------

对于新用户来说，上述各个部分可能是一个快速浏览。每个步骤中都有许多重要的概念没有涉及。建议通读   :ref:`unified_tutorial` ，以获得扎实的工作知识，了解上述内容的真正情况。祝你好运！