.. _orm_quickstart:


ORM快速入门
===============

对于想要快速了解基本ORM使用的新用户，这里是在:ref:`unified_tutorial`中使用的映射和示例的缩略形式。此处的代码可以在干净的命令行上完全运行。

由于本节中的描述是有意**非常简短**的，请转到完整的:ref:`unified_tutorial`以获取更详细的各个概念的说明。

.. versionchanged:: 2.0 ORM快速入门已更新，以使用包括:func:`_orm.mapped_column`在内的新构造update，以适应最新的:pep:`484`感知特性。有关迁移信息，请参见:ref:`whatsnew_20_orm_declarative_typing`部分。

声明模型
---------------

在这里，我们定义将形成我们将从数据库查询的结构的模块级别构造。 这个结构，称为:ref:`Declarative Mapping <orm_declarative_mapping>`，同时定义了一个Python对象模型以及描述存在于特定数据库中实际SQL表的:term:`database metadata`：

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

映射从一个基类开始，上面称为“Base”，并通过对:class:`_orm.Declarativebase`类进行简单的子类化来创建。

然后通过制作``Base``的子类来创建单个映射类。映射类通常是指特定的单个数据库表，其名称由使用``__tablename__``类级别属性来指示。

接下来，通过添加属性来声明组成表格的列，这些属性包括称为:class:`_orm.Mapped`的特殊类型注释。每个属性的名称对应于将成为数据库表的列。每个列的数据类型首先是从与每个:class:`_orm.Mapped`注释关联的Python数据类型中取出。``INTEGER``对应于``int``，``VARCHAR``对应于``str``，等等。可选型修饰符的空值衍生自其是否使用``Optional[]``类型修饰符。可以使用右侧:func:`_orm.mapped_column`指令中的SQLAlchemy类型对象指定更具体的类型信息，例如上面在``User.name``列中使用的:String数据类型。Python类型和SQL类型之间的关联可以使用:ref:`type annotation map<orm_declarative_mapped_column_type_map>`进行自定义。

所有基于列的属性都需要至少声明一个列作为主键，通常可以在应在主键上使用:paramref:`_schema.Column.primary_key`参数上的：func:`_orm.mapped_column`对象。在上面的示例中，“User.id”和“Address.id”列被标记为主键。

综合考虑，将字符串表名称和列声明列表的组合称为SQLAlchemy中的：term:`table metadata`。使用Core和ORM方法设置表元数据在:ref:`tutorial_working_with_metadata`中介绍。以上映射是指：ref:`带注释的声明表<orm_declarative_mapped_column>`配置的示例。

其他类型的:class:`_orm.Mapped`也可用，最常见的为上面指定的:func:`_orm.relationship`构造。与基于列的属性不同，:func:`_orm.relationship`表示两个ORM类之间的链接。在上面的示例中，``User.addresses``将``User``链接到``Address``，``Address.user``将``Address``链接到``User``。:func:`_orm.relationship`构造在:ref:`tutorial_orm_related_objects`中介绍。

最后，上面的示例类包括一个``__repr __（）``方法，这不是必需的，但对于调试很有用。可以使用自动生成方法，例如通过dataclasses，创建带有映射类。关于dataclass映射的更多内容请参见:ref:`orm_declarative_native_dataclasses`。


创建引擎
------------------

:class:`_engine.Engine`是一个**工厂**，它可以为我们创建新的数据库连接，还可以在:ref:`Connection Pool<pooling_toplevel>`内持有连接以快速重用。为了方便起见，我们通常使用一个只针对记忆的:ref:`SQLite<sqlite_toplevel>`数据库：

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite://", echo=True)

.. tip::

    参数``echo=True``表示将SQL发出到连接中记录在标准输出中。

有关:class:`_engine.Engine`的完整介绍从:ref:`tutorial_engine`开始。

发出CREATE TABLE DDL
----------------------

使用我们的表元数据和引擎，我们可以使用称为:meth:`_schema.MetaData.create_all`的方法在我们目标SQLite数据库中一次生成模式：

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

上面的Python代码发生了很多事情。在:ref:`tutorial_working_with_metadata`中，讨论了有关表元数据的所有文章。

创建对象并进行持久化
--------------------------

我们现在已准备好将数据插入数据库。我们可以通过创建``User``和``Address``类的实例来实现这一点这个过程已经通过自动化通过声明映射过程的``__init__（）``方法建立。然后，我们通过一个称为：ref:`Session <tutorial_executing_orm_session>`的对象将它们传递到数据库中，该对象使用:class:`_engine.Engine`与数据库进行交互。这里使用:meth:`_orm.Session.add_all`方法一次添加多个对象，使用:meth:`_orm.Session.commit`方法将提交任何待处理的更改到数据库中并将当前数据库事务提交，在使用:class:`_orm.Session`的情况下，这始终是在进行中的：

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

    建议采用以上样式的:class:`_orm.Session` ，也就是使用python中的“with：”语句。:class:`_orm.Session`对象表示活动数据库资源，因此在一系列操作完成后关闭它是非常好的。在下一节中，我们将保持 :class:`_orm.Session`的打开形式，仅用于说明目的。

有关创建:class:`_orm.Session`的基础知识请参见:ref:`tutorial_executing_orm_session`，有关更多信息请参见:ref:`session_basics`。

然后，在:ref:`tutorial_inserting_orm`中介绍了一些基本的持久性操作变体。

简单选择
--------------

拥有数据库中的一些行后，这里是发出SELECT语句以加载一些对象的最简单形式。要创建SELECT语句，我们使用:func:`_sql.select`函数创建一个新的:class:`_sql.Select`对象，然后使用:class:`_orm.Session`调用它。在ORM对象查询时，通常有用的方法是:meth:`_orm.Session.scalars`方法，该方法将返回一个:class:`_result.ScalarResult`对象，该对象将迭代我们选择的ORM对象：

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


上面的查询还使用了:meth:`_sql.Select.where`方法添加WHERE条件，还使用了所有SQLAlchemy类似列的构造的:meth:`_sql.ColumnOperators.in_`方法来使用SQL IN运算符。

有关如何选择对象和单个列的详细信息，请参见:ref:`tutorial_selecting_orm_entities`。

带有JOIN的SELECT
-----------------

查询多个表通常非常常见，在SQL中JOIN关键字是主要的方式。:class:`_sql.Select`构造使用:meth:`_sql.Select.join`方法创建连接：

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

上面的查询说明了自动连接多个WHERE条件，以及如何使用类似列的SQLAlchemy对象创建“相等”比较，这使用重写的Python方法:meth:`_sql.ColumnOperators.__eq__`来生成SQL准则对象。

有关上述概念的更多背景信息，请参见:ref:`tutorial_select_where_clause`和 :ref:`tutorial_select_join`。

进行更改
------------

:class:`_orm.Session`对象与我们的ORM映射类``User``和``Address``结合使用，自动跟踪对对象所做的更改，这将导致将来:class:`_orm.Session`flush时发出SQL语句。下面，我们更改与"sandy"关联的一个电子邮件地址，并在发出SELECT以检索"patrick"的行后向"patrick"添加一个新的电子邮件地址：

.. sourcecode:: pycon+sql

    >>> stmt = select(User).where(User.name == "patrick")
    >>> patrick = session.scalars(stmt).one()
    {execsql}SELECT user_account.id AS user_account_id, user_account.name AS user_account_name, user_account.fullname AS user_account_fullname
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

请注意，当我们访问``patrick.addresses``时，会发出一个SELECT。这称为:term:`lazy load`。:ref:` tutorial_orm_loader_strategies`介绍了使用更多或更少SQL访问相关项目的不同方法。

有关ORM数据操作的详细介绍始于:ref:`tutorial_orm_data_manipulation`。

一些删除
------------

总有一天，所有事情都会结束，就像数据库中的某些行一样——这里是两种不同类型的删除的快速演示，这两种不同类型的删除都基于特定的用例。

首先，我们将从"sandy"用户中删除一个``Address``对象。当:class:`_orm.Session`下一次flush时，这将导致行被删除。此行为是我们在配置中称为：term:`delete cascade`的行为。我们可以通过主键使用:meth:`_orm.Session.get`获得" sandy "对象，然后使用该对象：

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

上面的SELECT是：term:`lazy load` 运行，以便可以加载"sandy.addresses"集合，以便我们可以删除"sandy_address"成员。有其他方法可以处理此过程，这些方法不会发出太多SQL。

我们可以选择发出要更改的DELETE SQL（而不提交事务），使用:meth:`_orm.Session.flush`方法：

.. sourcecode:: pycon+sql

    >>> session.flush()
    {execsql}DELETE FROM address WHERE address.id = ?
    [...] (2,)

接下来，我们将完全删除“patrick”用户。对于唯一的顶级对象的删除本身，我们使用:meth:`_orm.Session.delete`方法；此方法实际上不执行删除，而是在下一次刷新时设置待删除的对象。此操作也将根据我们配置的级联选项:term:`cascade`到相关对象，此处是关联对象"Address"：

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

在:meth:`_orm.Session.delete`方法本身中，在本例中发出了两个SELECT语句，即使没有发出DELETE，这可能看起来令人惊讶。这是因为当方法执行检查对象时，事情不顺利。“patrick”对象是:term:`expired`，这在最后一次调用:meth:`_orm.Session.commit`时发生，发出的SQL是从新交易中重新加载行。这种过期是可选的，通常情况下，我们将会将其关闭，以适用范围比较差的情况。

为了说明即将删除的行，这是提交的提交：

.. sourcecode:: pycon+sql

    >>> session.commit()
    {execsql}DELETE FROM address WHERE address.id = ?
    [...] (4,)
    DELETE FROM user_account WHERE user_account.id = ?
    [...] (3,)
    COMMIT
    {stop}

在:ref:`tutorial_orm_deleting`中讨论了ORM删除。在:ref:`session_expiring`中介绍对象过期，:ref:`unitofwork_cascades`中讨论级联。

深入了解上述概念
---------------------------------

对于新用户，上面的部分很可能是一个快速的旅行。每个步骤中都有很多重要的概念没有涵盖。如果要对上述真正发生的情况有扎实的工作知识，则建议详细阅读:ref:`unified_tutorial`。祝你好运！