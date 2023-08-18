.. currentmodule:: sqlalchemy.orm

.. _mapper_sql_expressions:

将SQL表达式映射为属性
=============================

在映射类上的属性可以链接到SQL表达式，在查询中可以使用。

使用Hybrid
--------------

将相对简单的SQL表达式链接到类的最简单和最灵活的方法是使用所谓的“混合属性”，
详见:ref：`混合属性 <hybrids_toplevel>`一节。混合属性提供了既可以在Python级别上使用，也可以在SQL表达式级别上使用的表达式。例如，我们将类“User”映射为包含属性“firstname”和“lastname”的类，并包含一个混合，该混合将为我们提供“fullname”，即两者的字符串连接：

    from sqlalchemy.ext.hybrid import hybrid_property

    
    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        firstname = mapped_column(String(50))
        lastname = mapped_column(String(50))

        @hybrid_property
        def fullname(self):
            return self.firstname + " " + self.lastname

在上面的代码中，“fullname”属性在实例和类级别上都是可用的，因此可以从实例中使用：

    some_user = session.scalars(select(User).limit(1)).first()
    print(some_user.fullname)

以及可在查询中使用：

    some_user = session.scalars(
        select(User).where(User.fullname == "John Smith").limit(1)
    ).first()

上面的字符串连接示例是一个简单的示例，Python表达式可以在实例和类级别上同时使用。通常，SQL表达式必须与Python表达式区分开来，可以使用:meth：`.hybrid_property.expression`来实现。下面我们列出了需要在混合中存在条件的情况，使用Python的“if”语句和 :func:`_expression.case` 结构来执行SQL表达式：

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

.. _mapper_column_property_sql_expressions:

使用column_property
---------------------

可以使用:func：`_ orm.column_property`函数以类似于常规映射的方式映射SQL表达式：class:`_schema.Column`。使用这种技术，属性将在加载时与其他列映射属性一起加载。在某些情况下，这比使用混合的优点是，可以在对象的父行同时将该值加载到前面，尤其是如果表达式是链接到其他表（通常作为相关子查询）以访问不会正常可用于已加载的对象的数据的情况。

使用 :func:`_ orm.column_property` 进行SQL表达式的缺点包括，表达式必须与发射给整个类的SELECT语句兼容，并且在使用:func：`_ orm.column_property`从声明性mixin使用时，还可能出现一些配置性问题。

我们可以使用 :func:`_ orm.column_property` 来表示上面提到的“fullname”示例：

from sqlalchemy.orm import column_property

    
    class User(Base):
        __tablename__ = "user"
        id = mapped_column(Integer, primary_key=True)
        firstname = mapped_column(String(50))
        lastname = mapped_column(String(50))
        fullname = column_property(firstname + " " + lastname)

相关子查询也可以用。下面，我们使用  :func:`_expression.select` ，表示以列为导向的SELECT语句，其中计算可用于特定的'User'的“Address”对象的数量：

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

在上面的示例中，我们定义了像下面这样的 :func:`_expression.ScalarSelect` 结构：

    stmt = (
        select(func.count(Address.id))
        .where(Address.user_id == id)
        .correlate_except(Address)
        .scalar_subquery()
    )

在上面的代码中，我们首先使用  :func:`_sql.select`  _sql.Select` 结构，然后使用  :meth:`_sql.Select.scalar_subquery`  方法将其转换为单值子查询，指示我们打算使用此:class :` _sql.Select` 语句在列表达式上下文中使用。

在  :class:`_sql.Select` （请注意，' 'id'也是Python内置函数的名称，这不是我们要在这里使用的——如果我们在User类定义之外，则会使用User.id）。

  :meth:`_sql.Select.correlate_except`  方法表示  :func:` _expression.select`  FROM子句中的每个元素都可以从FROM列表中省略（也就是说，相关到封闭的User中的SELECT语句）除了与Address对应的。这不是必需的，但可以防止在“User”和“Address”表之间的长字符串的连接中，在SELECT语句和“Address”之间进行嵌套时意外地省略“Address” FROM列表。

对于引用自多对多关系的列的column_property，使用 :func:`.and_` 将关联表的字段连接到关系中的两个表中的一个。示例如下：

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

向现有Declarative映射类添加column_property（）^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果导入问题防止在类的内置使用  :func:`.column_property` ::_orm.Mapper.add_property` 以在事实上添加一个其他属性：

    # 仅适用于使用Declarative基类的情况
    User.address_count = column_property(
        select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
    )

对于不使用Declarative基类的映射样式，例如使用  :meth:`_orm.registry.mapped`  装饰器，可以在底层 :class：` _orm.Mapper`对象上明确调用  :meth:`_orm.Mapper.add_property`  方法，该方法可以使用 :func:` _sa.inspect`获取：

    from sqlalchemy.orm import registry

    reg = registry()

    
    @reg.mapped
    class User:
        __tablename__ = "user"

        # ... additional mapping directives

    # 后来 ...

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

在映射时从列属性组合
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以创建将多个  :class:`.ColumnProperty` .ColumnProperty` 将被解释为SQL表达式，前提是存在指向现有表达式对象的其他表达式对象；这是通过Core检测到该对象具有“__clause_element__()”方法返回SQL表达式来实现的。但是，如果在表达式中单独使用  :class:`.ColumnProperty` ，例如在没有其他Core SQL表达式对象将其作为目标的情况下，  :attr:` .ColumnProperty.expression`  属性将返回基础SQL表达。下面，'File'类包含一个属性'File.path'，该属性将字符串令牌连接到'File.filename'属性上，后者本身是类  :class:`.ColumnProperty` ：


    class File(Base):
        __tablename__ = "file"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(64))
        extension = mapped_column(String(8))
        filename = column_property(name + "." + extension)
        path = column_property("C:/" + filename.expression)

当正常使用文件类时，分配给'filename'和'path'的属性是直接可用的。仅当在映射定义中直接使用  :class:`.ColumnProperty` .ColumnProperty.expression` 属性：


    stmt = select(File.path).where(File.filename == "foo.txt")

使用“column_property（）”添加列延迟^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 :ref:`queryguide_toplevel` 中介绍的列延迟功能可以在映射时应用于由 :func:`_orm.column_property` 映射的SQL表达式，方法是在 :func:`_orm.column_property` 的位置使用 :func:`_orm.deferred` 函数：：

    from sqlalchemy.orm import deferred

    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        firstname: Mapped[str] = mapped_column()
        lastname: Mapped[str] = mapped_column()
        fullname: Mapped[str] = deferred(firstname + " " + lastname)

.. seealso::

      :ref:`orm_queryguide_deferred_imperative` 


使用普通描述符
------------------------

在发射超出  :class:`_orm.column_property` .hybrid_property` 能力范围的SQL查询时，可以使用访问为属性的普通Python函数，假设表达式仅需要在已加载的实例上可用。将函数用Python自己的``@property``装饰器装饰为只读属性。在函数内部，使用  :func:`.object_session` .Session` ，然后使用它发射查询：

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

对于query_expression配置的映射属性，还可以使用 :func:`_orm.with_expression` 加载程序选项，在查询时将对象加载为任意SQL表达式的结果。请参见 :ref:`orm_queryguide_with_expression` 一节中的映射和用法示例。

