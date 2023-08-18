.. currentmodule:: sqlalchemy.orm

.. _mapper_sql_expressions:

将 SQL 表达式作为映射属性
============================

在映射类中，可以将属性链接到 SQL 表达式，以便在查询中使用。

使用混合类型
----------------

将相对简单的 SQL 表达式链接到类上的最简单和最灵活的方法是使用所谓的“混合类型属性”，
它在 :ref:`混合类型的顶层` 中描述。混合类型提供了一个可以在 Python 级别和 SQL 表达式级别都适用的表达式。例如，下面我们映射了一个类 User，包含属性 firstname 和 lastname，并包括一个 hybrid，它将为我们提供 fullname，也就是两者的字符串拼接::

    from sqlalchemy.ext.hybrid import hybrid_property


    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        firstname = mapped_column(String(50))
        lastname = mapped_column(String(50))

        @hybrid_property
        def fullname(self):
            return self.firstname + " " + self.lastname

在上面的示例中，fullname 属性在实例和类级别上都被解释，因此可以从实例中使用::

    some_user = session.scalars(select(User).limit(1)).first()
    print(some_user.fullname)

并且可以在查询中使用::

    some_user = session.scalars(
        select(User).where(User.fullname == "John Smith").limit(1)
    ).first()

字符串连接示例是一个简单的示例，Python 表达式可以在实例和类级别上都实现双重目的。通常，必须区分 SQL 表达式和 Python 表达式，可以使用 :meth:`.hybrid_property.expression` 来实现。下面我们演示调用混合类型的过程中需要存在条件时的情况，使用 Python 中的 if 語句和 SQL 表达式中的 :func:`_expression.case` 构造：


    from sqlalchemy.ext.hybrid import hybrid_property
    from sqlalchemy.sql import case


    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        firstname = mapped_column(String(50))
        lastname = mapped_column(String(50))

        @hybrid_property
        def fullname(self):
            if self.firstname is not None:
                return self.firstname + " " + self.lastname
            else:
                return self.lastname

        @fullname.expression
        def fullname(cls):
            return case(
                (cls.firstname != None, cls.firstname + " " + cls.lastname),
                else_=cls.lastname,
            )

使用 column_property
---------------------

可以使用 :func:`_orm.column_property` 函数以类似于已映射的 :class:`_schema.Column` 的方式映射 SQL 表达式。使用此技术，属性与在加载时同时加载的所有其他列映射的属性一起加载。在某些情况下，与使用混合类型相比，这是一个优点，因为加载的值可以在对象的父行同时加载的同时加载，特别是如果表达式是一个与其他表格链接（通常作为相关子查询）以访问通常不可用的数据的表达式。

使用 :func:`_orm.column_property` 映射名为 fullname 的示例如下:

    from sqlalchemy.orm import column_property


    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        firstname = mapped_column(String(50))
        lastname = mapped_column(String(50))
        fullname = column_property(firstname + " " + lastname)

关联子查询也可以使用。下面使用 :func:`_expression.select` 构造函数创建一个 :class:`_sql.ScalarSelect`，它表示一个面向列的 SELECT 语句，将 ``Address`` 对象的计数链接到特定的 ``User`` 身上::

    from sqlalchemy.orm import column_property
    from sqlalchemy import select, func
    from sqlalchemy import Column, Integer, String, ForeignKey

    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class Address(Base):
        __tablename__ = "address"
        id = mapped_column(Integer, primary_key=True)
        user_id = mapped_column(Integer, ForeignKey("user.id"))


    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        address_count = column_property(
            select(func.count(Address.id))
            .where(Address.user_id == id)
            .correlate_except(Address)
            .scalar_subquery()
        )

在上面的示例中，我们创建一个类似于以下所示的 :func:`_expression.ScalarSelect` ——

    stmt = (
        select(func.count(Address.id))
        .where(Address.user_id == id)
        .correlate_except(Address)
        .scalar_subquery()
    )

上面，我们首先使用 :func:`_sql.select` 创建一个 :class:`_sql.Select` 构造函数，然后使用 :meth:`_sql.Select.scalar_subquery` 方法将其转换为标量子查询，表示我们打算在列表达式上使用此 :class:`_sql.Select` 语句。

在 :class:`_sql.Select` 中，我们选择所有的 ``Address`` 表中 ``Address.user_id`` 列等于 ``id`` 的 ``Address.id`` 行数，其中在 ``User`` 类中， ``id`` 是名为 ``id`` 的 :class:`_schema.Column`（请注意， ``id`` 也是 Python 内置函数的名称，这不是我们要在这里使用的 - 如果在 ``User`` 类定义之外，我们将使用 ``User.id``）。

:meth:`_sql.Select.correlate_except` 方法指示在这个 :func:`_expression.select` 中，FROM 子句中的每个元素都可能被省略，除了对应于 ``Address`` 的元素。这不是必需的，但它可以避免在 ``User`` 和 Address 表之间的长连接字符串中的情况下意外地省略了 ``Address``。

对于引用来自多对多关系的列的 :func:`.column_property`，请使用 :func:`.and_` 来将关联表的字段连接到关系中的两个表中的每一个表::

    from sqlalchemy import and_


    class Author(Base):
        # ...

        book_count = column_property(
            select(func.count(books.c.id))
            .where(
                and_(
                    book_authors.c.author_id == authors.c.id,
                    book_authors.c.book_id == books.c.id,
                )
            )
            .scalar_subquery()
        )

将 column_property() 添加到现有的 Declarative 映射类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果导入问题阻止 :func:`.column_property` 与类内联，可以在两者配置后将其分配给类。当使用使用 Declarative 基类的映射（即由 :class:`_orm.DeclarativeBase` 超类或 :func:`_orm.declarative_base` 等旧函数生成的映射）时，此属性分配的效果是调用 :meth:`_orm.Mapper.add_property` 来在事实上添加一个属性::

    # 仅适用于使用声明式基类
    User.address_count = column_property(
        select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
    )

使用不使用 Declarative 基类的映射风格，例如 :meth:`_orm.registry.mapped` 装饰器，可以在查询时显式调用底层 :class:`_orm.Mapper` 对象上的 :meth:`_orm.Mapper.add_property` 方法，该方法可以使用 :func:`_sa.inspect` 获取：

    from sqlalchemy.orm import registry

    reg = registry()


    @reg.mapped
    class User:
        __tablename__ = "user"

        # ... additional mapping directives


    # later ...

    # 适用于任何类型的映射
    from sqlalchemy import inspect

    inspect(User).add_property(
        column_property(
            select(func.count(Address.id))
            .where(Address.user_id == User.id)
            .scalar_subquery()
        )
    )

.. seealso::

  :ref:`orm_declarative_table_adding_columns`


.. _mapper_column_property_sql_expressions_composed:

在映射时间从列属性构成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

通常情况下，可以使用多个 :class:`.ColumnProperty` 对象将多个映射属性组合在一起。在 Core 表达式上下文中，当 :class:`.ColumnProperty` 被定位到现有表达式对象时，它将被解释为 SQL 表达式；它是通过 Core 检测到该对象具有返回 SQL 表达式的 ``__clause_element__()`` 方法实现的。然而，在表达式中使用 :class:`.ColumnProperty` 作为主对象，如果没有其他 Core SQL 表达式对象来定位它，则会返回 :attr:`.ColumnProperty.expression` 属性，以便可以将其用于一致地构建 SQL 表达式。下面，类 File 包含一个属性 File.path，它将字符串令牌连接到 File.filename 属性上，而后者本身是 :class:`.ColumnProperty`::


    class File(Base):
        __tablename__ = "file"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(64))
        extension = mapped_column(String(8))
        filename = column_property(name + "." + extension)
        path = column_property("C:/" + filename.expression)

当在通常情况下使用 File 类时，分配给 filename 和 path 的属性可直接使用。只有当 :class:`.ColumnProperty` 直接在映射定义中使用时才需要使用 :attr:`.ColumnProperty.expression` 属性::

    stmt = select(File.path).where(File.filename == "foo.txt")

使用列递延与 ``column_property()`` 
-------------------------------------

查询指南 :ref:`queryguide_toplevel` 中介绍的列递延功能可以通过在映射时间使用 :func:`_orm.deferred` 函数而不是 :func:`_orm.column_property` 映射由 :func:`_orm.column_property` 映射的 SQL 表达式来应用，例如::

    from sqlalchemy.orm import deferred


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        firstname: Mapped[str] = mapped_column()
        lastname: Mapped[str] = mapped_column()
        fullname: Mapped[str] = deferred(firstname + " " + lastname)

.. seealso::

    :ref:`orm_queryguide_deferred_imperative`


使用普通描述符 attribute 
--------------------------

在必须发出比 :func:`_orm.column_property` 或 :class:`.hybrid_property` 更多的 SQL 查询的情况下，可以使用作为属性访问的普通 Python 函数，只要表达式仅需要在已加载的实例上可用即可。使用 Python 自己的 ``@property`` 装饰器将其标记为只读属性。在功能内部，使用 :func:`.object_session` 来定位对应于当前对象的 :class:`.Session`，然后用它来发出查询::

    from sqlalchemy.orm import object_session
    from sqlalchemy import select, func


    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        firstname = mapped_column(String(50))
        lastname = mapped_column(String(50))

        @property
        def address_count(self):
            return object_session(self).scalar(
                select(func.count(Address.id)).where(Address.user_id == self.id)
            )

普通的描述符方法在必要时很有用，但通常在性能上不如混合类型或 column_property 方法，因为它需要在每次访问时发出 SQL 查询。

.. _mapper_querytime_expression:

映射时间 SQL 表达式加载属性
-----------------------------------------------

除了能够在映射类上配置固定的 SQL 表达式外，SQLAlchemy ORM 还包括一种特性，即在查询时间设置的任意 SQL 表达式作为其状态的一部分加载对象。此行为可以使用 :func:`_orm.query_expression` 配置 ORM 映射属性，然后在查询时间使用 :func:`_orm.with_expression` 加载器选项来实现。有关示例映射和用法，请参见本节 :ref:`orm_queryguide_with_expression`。

