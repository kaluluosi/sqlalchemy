.. _whatsnew_20_toplevel:

================================
SQLAlchemy 2.0 有什么新特性？
================================

.. admonition:: 关于此文档

    此文档记录了 SQLAlchemy 1.4 版本和 SQLAlchemy 2.0 版本之间的更改，独立于  :term:`1.x style`  和  :term:` 2.0 style`  用法之间的主要更改。阅读者应始终先从   :ref:`migration_20_toplevel`  文档开始，这篇文档能让你最初了解 1.x 和 2.x 系列之间实现兼容所需的主要兼容更改。

    除了 1.x - > 2.x 迁移路径之外，SQLAlchemy 2.0 中最大的范式转变是与  :pep:`484`  类型有关的深度集成, 特别是在 ORM 中。灵感源自 Python dataclasses_ 的新类型驱动的 ORM 声明样式，以及与 dataclasses_ 本身的新集成，这些都是一种最终不再需要存根(Stubs)且从 SQL 语句到结果集提供类型感知方法链的方法。

    Python 开发人员对类型感知的重视程度已经越来越高，这不仅是为了让类型检查器像 mypy_ 这样的运行无插件，更重要的是它使得 IDEs 像 vscode_ 和 pycharm _ 更加积极地协助开发者组合 SQLAlchemy 应用程序。

.. admonition:: 阅读者须知

    SQLAlchemy 2.0 的迁移文档分为 **两个** 文档 - 一个文档详细说明从 1.x 到 2.x 系列的 API 显著更改，另一个文档详细说明了相对于 SQLAlchemy 1.4 的新功能和行为:

    *   :ref:`migration_20_toplevel`  - 1.x 到 2.x API 变化
    *   :ref:`whatsnew_20_toplevel`  - 此文档, SQLAlchemy 2.0 的新功能和行为

    如果阅读者尚未将他们的 1.4 应用程序更新为遵循 SQLAlchemy 2.0 引擎和 ORM 惯例，则可以导航到   :ref:`migration_20_toplevel` ，了解如何确保 SQLAlchemy 2.0 兼容性，这是版本 2.0 下运行代码的先决条件。

.. _typeshed: https://github.com/python/typeshed

.. _dataclasses: https://docs.python.org/3/library/dataclasses.html

.. _mypy: https://mypy.readthedocs.io/en/stable/

.. _vscode: https://code.visualstudio.com/

.. _pylance: https://github.com/microsoft/pylance-release

.. _pycharm: https://www.jetbrains.com/pycharm/

核心和 ORM 中的新类型支持 - 不再使用存根 / 扩展
---------------------------------------------------------------


与版本 1.4 通过 sqlalchemy2-stubs_ 包提供的中间方法相比，SQLAlchemy 的类型方法在 Core 和 ORM 中的应用已彻底被重新设计。这种全新的方法从最基本的元素开始，即: class:`_schema.Column`，或者更准确地说：用于处理所有具有类型的 SQL 表达式的   :class:`.ColumnElement` 。这种表达级别的类型然后延伸到构建语句、执行语句和结果集的领域，最终到 ORM 的领域，其中新的   :ref:` declarative <orm_declarative_mapper_config_toplevel>`  表单允许用于完全类型化的 ORM 模型，其从语句到结果集集成。

.. Tip:: 类型支持应被视为**Beta Level**软件，但类型详细信息可能会发生变化，但不计划进行重大的不向后兼容的更改。

SQL 表达式 / 语句 / 结果集类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

本节提供了 SQLAlchemy 新的 SQL 表达式类型化方法的背景和示例，该方法从基本的   :class:`.ColumnElement`  构造物开始，并贯穿整个 SQL 语句和结果集的领域，最终进入 ORM 映射的领域。

工作原理和概述
^^^^^^^^^^^^^^^^^^^^

.. Tip::

    本节进行了架构讨论。如果你想要快速了解新的类型化看法，可跳到   :ref:`whatsnew_20_expression_typing_examples` 。

在 sqlalchemy2-stubs_ 中，SQL 表达式被作为常规数据类型编写，并引用它们的   :class:`.TypeEngine`  对象，如   :class:` .Integer` ，  :class:`.DateTime`  或   :class:` .String` （例如 'Column [Integer]'）作为它们的泛型实参。这本身就是与原始 Dropbox sqlalchemy-stubs_ 包有所分歧之处，后者将   :class:`.Column`  和其基本构造物直接作为 Python 类型的泛型，例如 'int'，'datetime' 和 'str'。人们希望由于   :class:` .Integer`  /   :class:`.DateTime`  /   :class:` .String`  本身是相对于 `int` / `datetime` / `str` 泛型的，因此将有方法来维护两级信息，并且通过   :class:`.TypeEngine`  作为中间构造物从列表达式中提取 Python 类型。然而在大多数情况下这是不可能的，因为  :pep:` 484`  不具备足够丰富的功能（缺乏重要功能，例如用于更高阶的 TypeVars ），因此这种方法不再可行。

因此，在对  :pep:`484`  的现有功能进行认真评估之后，SQLAlchemy 2.0 认识到了 sqlalchemy-stubs_ 在这个领域的原始智慧，并直接链接列表达式到 Python 类型，这意味着，如果一个 SQL 表达式被不同于常规数据类型的任何子类型（如 'Column (VARCHAR)' vs 'Column (Unicode)'），那么它们的   :class:` .String`  子类型的具体特定内容将不会被保留为 Python 数据，但在实践中，这通常不是问题，并且如果 Python 类型直接存在，一般情况下这会更有用。

具体地说，这意味着像“Column（'id'，Integer）”这样的表达式的类型定义为“Column [int]”。这可以使得从 SQLAlchemy 构件到 Python 数据类型的可行的管道被设置起来，而无需使用类型插件。重要的是，它允许完全的互操作性，即 ORM 将使用   :func:`_sql.select`  和   :class:` _engine.Row`  包含用户映射实例，例如我们教程中使用的"User" 和"Address"示例）。虽然 Python 类型目前对元组类型（其中  :pep:`646` , 第一蒟蒻尝试处理类似元组的对象的pep，被故意限制了其功能并因自身还不适用于任意元组操作而被限制到仅使用针对某些具有 \**args 和 keyword 限制的原始函数的hack来工作）支持非常有限（  :pep:` 484` ），但已制定了一个相当不错的方法，使得基本   :func:`_sql.select（）-->  :class:` _engine.Result` `-->  :class:`_engine.Row`  的类型化方法链能够正常工作，包括 ORM 类，其中在   :class:` _engine.Row`  对象被取消包装为单个列条目的点上，加入了一个小型与类型有关的访问器，它允许每个 Python 值维护与其源SQLExpresssion链接的特定 Python 类型（翻译：它有效）。

.. _sqlalchemy-stubs: https://github.com/dropbox/sqlalchemy-stubs

.. _sqlalchemy2-stubs: https://github.com/sqlalchemy/sqlalchemy2-stubs

.. _generics: https://peps.python.org/pep-0484/#generics



.. _whatsnew_20_expression_typing_examples:

SQL 表达式类型化 - 示例
^^^^^^^^^^^^^^^^^^^^^^^^^^^

快速浏览下 SQLAlchemy 的新类型行为。在下面的示例中，注释：指示了你在 vscode_ (或在使用 `reveal_type() <https://mypy.readthedocs.io/en/latest/common_issues.html?highlight=reveal_type#reveal-type>`_ 功能使用类型检查工具时会看到什么信息：

* 将 Python 简单数据类型分配给 SQL 表达式

  ::

    # (variable) str_col: ColumnClause[str]
    str_col = column("a", String)

    # (variable) int_col: ColumnClause[int]
    int_col = column("a", Integer)

    # (variable) expr1: ColumnElement[str]
    expr1 = str_col + "x"

    # (variable) expr2: ColumnElement[int]
    expr2 = int_col + 10

    # (variable) expr3: ColumnElement[bool]
    expr3 = int_col == 15

* 将单个 SQL 表达式分配到   :func:`_sql.select`  构造，以及任何返回行的构造，包括返回行 DML，例如带有  :meth:` _sql.Insert.returning `  的   :class:`_sql.Insert` ，将被打包到一个 ` `Tuple[]`` 类型中，该类型保留每个元素的 Python 类型。

  ::

    # (variable) stmt: Select[Tuple[str, int]]
    stmt = select(str_col, int_col)

    # (variable) stmt: ReturningInsert[Tuple[str, int]]
    ins_stmt = insert(table("t")).returning(str_col, int_col)

* 来自任何返回行的构造的 ``Tuple[]`` 类型，当使用 ``.execute()`` 方法调用它时，将通过   :class:`_engine.Result`  和   :class:` _engine.Row`  传递。为了将   :class:`_engine.Row`  对象解包为元组，必须使使用  :meth:` _engine.Row.tuple`  或  :attr:`_engine.Row.t`  访问器，本质上将   :class:` _engine.Row`  强制转换为相应的 ``Tuple[]`` (但在运行时仍保持相同的   :class:`_engine.Row`  对象)。

  ::

    with engine.connect() as conn:
        # (variable) stmt: Select[Tuple[str, int]]
        stmt = select(str_col, int_col)

        # (variable) result: Result[Tuple[str, int]]
        result = conn.execute(stmt)

        # (variable) row: Row[Tuple[str, int]] | None
        row = result.first()

        if row is not None:
            # for typed tuple unpacking or indexed access,
            # use row.tuple() or row.t  (this is the small typing-oriented accessor)
            strval, intval = row.t

            # (variable) strval: str
            strval

            # (variable) intval: int
            intval

* 单列语句的标量值使用  :meth:`_engine.Connection.scalar` ,  :meth:` _engine.Result.scalars`  等方法时，会产生正确的值。

  ::

    # (variable) data: Sequence[str]
    data = connection.execute(select(str_col)).scalars().all()

* 对于 ORM 映射类，按预期工作。

  典型用法是按 ORM 映射类选择纯粹的查询：

  ::

      # (variable) users1: Sequence[User]
      users1 = session.scalars(select(User)).all()

      # (variable) user: User
      user = session.query(User).one()

      # (variable) user_iter: Iterator[User]
      user_iter = iter(session.scalars(select(User)))

  *   :class:`_orm.Query`  的独特功能被扩展：它在返回值级别上保留类型化，并允许数据类型作为元组或单值的类型。

      # (variable) q1: RowReturningQuery[Tuple[int, str]]
      q1 = session.query(User.id, User.name)

      # (variable) rows : List[Row[Tuple[int, str]]]
      rows = q1.all()

      # (variable) q2: Query[User]
      q2 = session.query(User)

      # (variable) users: List[User]
      users = q2.all()

* 核心 Table 尚未找到有效的方法在访问  :attr:`.Table.c`  时维护   :class:` _schema.Column`  对象的类型。

  由于   :class:`.Table`  被设置为类的实例，并且  :attr:` .Table.c`  访问器通常按名称动态访问   :class:`.Column`  对象，因此当前还没有为此方法确定类型的成熟方法，因此需要其他的语法。

* ORM 类，标量等功能运行良好。

  具有典型用法，如按需选择 ORM 类，标量值或元组的用例，2.x 和 1.x 样式的查询，以不管是“单个值”还是包含在适当的容器中，如 ``Sequence[]``，``List[]`` 或 ``Iterator[]``:

  ::

    # (variable) u1: Type[User]
    u1 = aliased(User)
    # (variable) stmt: Select[Tuple[User, User, str]]
    stmt = select(User, u1, User.name).filter(User.id == 5)
    # (variable) result: Result[Tuple[User, User, str]]
    result = session.execute(stmt)
    # (variable) user_obj: User
    # (variable) address_obj: Address
    user_obj, address_obj = result.one().t


其中选择映射类的构造，例如  :class:`_orm.aliased` , 与原始映射类的列级属性以及期望的语句返回类型一样有效。

.. admonition:: 注意事项 - 必须卸载所有存根

    使用类型支持的一个关键限制是**必须卸载所有 SQLAlchemy stubs packages**，以使类型支持能够正常工作。当使用 mypy_ 对 Python 虚拟环境运行时，这仅仅是卸载这些包。但是，当前版本的 typeshed_ 包中包含了 SQLAlchemy 的存根包，因此它本身被捆绑到某些类型检查工具中，例如 Pylance_，因此在某些情况下，如果它们确实妨碍了新的打字支持正常工作，就必须定位这些包的文件并将其删除。

    一旦 SQLAlchemy 2.0 发布为最终状态，typeshed 将从其自己的 stubs 源中删除 SQLAlchemy。

.. _whatsnew_20_orm_declarative_typing:

ORM Declarative Models
~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 1.4 引入了使用 sqlalchemy2-stubs_ 和   :ref:`Mypy Plugin <mypy_toplevel>`  的第一个 SQLAlchemy-native ORM 类型支持。在 SQLAlchemy 2.0 中，Mypy 插件**仍然可用，并已更新以使用 SQLAlchemy 2.0 的类型系统**，但现在应被认为是**已弃用的**，因为应用程序现在有一个简单的路径可用于采用新的类型支持，该路径不使用插件或存根。

概述
^^^^^^^^

新系统的基本方法是：当使用完全   :ref:`Declarative <orm_declarative_table>`  模型(即，不是   :ref:` hybrid declarative <orm_imperative_table_configuration>`  或   :ref:`imperative <orm_imperative_mapping>`  配置，这些配置的更改是无效的) 时，已映射列声明，首先通过检查每个属性声明左侧的类型注释来在运行时派生。如果注释存在，则左手注释可以在  :func:` _orm.mapped_column` 上参考右手端上的  :class:`_schema.Column` ，后者用于提供关于将要生成并映射的   :class:`_schema.Column`  的 Core 级架构信息。如果左侧没有注释，则可以使用   :func:` _orm.mapped_column`  作为   :class:`_schema.Column`  指令的确切替代项，即不是在左侧存在注释的情况下。

这种方法的灵感源自 Python dataclasses_ 方法，它从左侧注释开始，然后允许在右侧使用可选的``dataclasses.field()`` 规范；与 dataclasses_ 方法的主要区别在于 SQLAlchemy 的方法是**严格选择性**的，其中没有类型注释的已存在映射，可以像以前一样工作，  :func:`_orm.mapped_column` 可以作为   :class:` _schema.Column`  指令的直接替换品，即使左侧没有明确的注释。

在需要存在实际属性级Python类型的情况下，“_orm.Mapped” 注释必须显式地用于每个属性上，这些属性在   :func:`_orm.mapped_column`  在没有显式注释的情况下将会被定义为“Any” 。

.. _whatsnew_20_orm_typing_migration:

迁移现有的 ORM 映射
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

过渡到新的 ORM 方法开始更加冗长，但随着使用的新功能越来越多，它会变得更加简洁，以下步骤详细说明了典型过渡，然后继续演示了一些更多的选项。

步骤一 -   :func:`_orm.declarative_base`  被   :class:` _orm.DeclarativeBase`  取代。
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

观察到的 Python 打字的一个限制是似乎不存在一种方法来从函数中动态生成一个类，然后再将其作为新类的基类，类型检查工具可以理解这些信息。为了解决这个问题，可以使用   :class:`_orm.DeclarativeBase`  类替换通常的   :func:` _orm.declarative_base`  调用，这将产生与以前相同的“Base”对象，但类型检查工具可以理解它：

    from sqlalchemy.orm import DeclarativeBase

    class Base(DeclarativeBase):
        pass

步骤二 - 使用   :func:`_orm.mapped_column`  替换 Declarative 使用的   :class:` _schema.Column` 
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

在使用   :class:`_schema.Column`  的 ORM 类型中，可以直接使用   :func:` _orm.mapped_column` ，它是 ORM 感知的构造，可以直接替换   :class:`_schema.Column`  的使用。考虑以下 1.x 的映射：

  ::

    from sqlalchemy import Column
    from sqlalchemy.orm import relationship
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user_account"

        id = Column(Integer, primary_key=True)
        name = Column(String(30), nullable=False)
        fullname = Column(String)
        addresses = relationship("Address", back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        email_address = Column(String, nullable=False)
        user_id = Column(ForeignKey("user_account.id"), nullable=False)
        user = relationship("User", back_populates="addresses")

将   :class:`_schema.Column`  替换为   :func:` _orm.mapped_column` ，不需要更改任何参数：

  ::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user_account"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(30), nullable=False)
        fullname = mapped_column(String)
        addresses = relationship("Address", back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        id = mapped_column(Integer, primary_key=True)
        email_address = mapped_column(String, nullable=False)
        user_id = mapped_column(ForeignKey("user_account.id"), nullable=False)
        user = relationship("User", back_populates="addresses")

上面的各列**尚未具有 Python 类型**，并且在实例级别上将被定义为“Mapped [Any]”；这是因为我们可以将任何列声明声明为可选或不可选，因此没有必要进行“猜测”，以免在明确定义类型的情况下进行类型错误。但在此步骤中，我们上面的映射已经为每个属性设置了适当的  :term:`descriptor`  类型，可以在查询中使用，也可以用于实例级替换，所有这些都将在没有插件的情况下通过 mypy - strict 模式。

步骤三 - 根据需要应用确切的 Python 类型，使用   :class:`_orm.Mapped` 。
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

这可以针对需要确切类型的所有属性完成；可以跳过可以被视为“Any”的属性。例如，为了提供 ORM 恒定类型，详细说明   :class:`_orm.Mapped`  被使用的关于   :func:` _orm.relationship` ：

  ::

      from typing import List
      from typing import Optional
      from sqlalchemy.orm import DeclarativeBase
      from sqlalchemy.orm import Mapped
      from sqlalchemy.orm import mapped_column
      from sqlalchemy.orm import relationship


      class Base(DeclarativeBase):
          pass


      class User(Base):
          __tablename__ = "user_account"

          id: Mapped[int] = mapped_column(Integer, primary_key=True)
          name: Mapped[str] = mapped_column(String(30), nullable=False)
          fullname: Mapped[Optional[str]] = mapped_column(String)
          addresses: Mapped[List["Address"]] = relationship(back_populates="user")


      class Address(Base):
          __tablename__ = "address"

          id: Mapped[int] = mapped_column(Integer, primary_key=True)
          email_address: Mapped[str] = mapped_column(String, nullable=False)
          user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"), nullable=False)
          user: Mapped["User"] = relationship(back_populates="addresses")

此时，我们的 ORM 映射已完全具有类型，并将生成精确类型化的   :func:`_sql.select` 、  :class:` _orm.Query`  和   :class:`_engine.Result`  构造。现在，我们可以开始简化映射声明中的冗余。

步骤四 - 移除   :func:`_orm.mapped_column`  的指令了
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

所有“可选”参数都可以使用 ``Optional[]`` 隐含，如果没有 ``Optional[]``，则 ``nullable`` 默认为“False”。所有 SQL 类型缺少参数，如 ``Integer`` 和 ``String`` 只能作为 Python 注释表达。没有参数的   :func:`_orm.mapped_column`  指令可以完全删除。  :func:` _orm.relationship`  现在将其类从左侧的注释中推导出来，支持用于字符串的"前向引用"(已经支持10年了；))。

  ::

    from typing import List
    from typing import Optional
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(30))
        fullname: Mapped[Optional[str]]
        addresses: Mapped[List["Address"]] = relationship(back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        email_address: Mapped[str]
        user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
        user: Mapped["User"] = relationship(back_populates="addresses")

步骤五 - 使用 pep-593 ``Annotated`` 将常用的指令打包到类型中
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

这是一个全新的功能，是作为一种提供面向类型的配置的替代或补充方法，并取代了一般的   :ref:`declarative mixins <orm_mixins_toplevel>` 。Declarative 映射允许自定义 SQL 类型与 Python 类型之间的映射，例如“str”到 ` `String`` 类型，使用  :paramref:`_orm.registry.type_annotation_map` 。使用  :pep:` 593` `Annotated`` 允许我们创建特定 Python 类型的变体，以便可以使用相同的类型，例如 ``str``，每个提供   :class:`_types.String`  的变体。使用 ` `Annotated`` 的特定示例是，使用  :func:`_orm.mapped_column` 在注释左侧中传递   :func:` _orm.mapped_column`  构造(使用者 credit to @adriangb01 <https://twitter.com/adriangb01/status/1532841383647657988> 为这个想法进行说明) 。

此功能在将来的发布版本中可以进一步扩展，以包括   :func:`_orm.relationship` ,   :func:` _orm.composite`  和其他构造，但当前仅限于   :func:`_orm.mapped_column` 。下面的示例除了我们的"str50"示例之外，还添加了其他 ` `Annotated`` 类型，以说明此功能：

  ::

    from typing_extensions import Annotated
    from sqlalchemy.orm import DeclarativeBase

    str50 = Annotated[str, 50]


    # 使用类型级别覆盖的 declarative 基础，使用一个预期在多个地方使用的类型
    class Base(DeclarativeBase):
        type_annotation_map = {
            str50: String(50),
        }

在左手类型使用   :func:`_orm.mapped_column`  时，Declarative 将从左侧类型提取完整的   :func:` _orm.mapped_column`  定义，如果使用   :func:`_orm.mapped_column`  构造作为 ` `Annotated[]`` 的任何一个参数在左侧使用，即可实现此功能。目前此功能被限制为   :func:`_orm.mapped_column`  的使用。下面的示例添加了其他 Annotated 类型以说明此功能：

    from typing_extensions import Annotated
    from typing import List
    from typing import Optional

    class Base(DeclarativeBase):
        pass

    str50 = Annotated[str, 50]
    Email = Annotated[str, 50]
    OptionalString = Annotated[Optional[str], False]

    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(30))
        fullname: Mapped[OptionalString]
        addresses: Mapped[List["Address"]] = relationship(back_populates="user")
        email: Mapped[Email]

    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        email_address: Mapped[str]
        user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
        user: Mapped[User] = relationship(back_populates="addresses")


同时使用``Annotated``和``type_annotation_map``的示例:

    from typing_extensions import Annotated
    from typing import List, Optional, Type
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column, relationship

    str50 = Annotated[str, 50]

    class Base(DeclarativeBase):
        type_annotation_map = {
            str50: String(50),
        }

    class User(Base):
        __tablename__ = "users"

        # The Mapped[...] annotations describe the types at the Python level,
        # absent the type_annotation_map dictionary which is used for leading the
        # ORM to the correct types.
        id: Mapped[int] = mapped_column(Integer, primary_key=True)
        name: Mapped[str] = mapped_column(str_)
        age: Mapped[Optional[int]] = mapped_column(Integer)

        # The List[Address] maps to Address.user, which is of type Mapped[User],
        # where Mapped[User] maps to the class itself.
        addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

    class Address(Base):
        __tablename__ = "addresses"

        id: Mapped[int] = mapped_column(Integer, primary_key=True)
        email: Mapped[str] = mapped_column(str_)
        user_id: Mapped[int] = mapped_column(ForeignKey(User.id))
        user: User.addresses.property.mapper.class_ = relationship(User, back_populates="addresses")


.. _whatsnew_20_migration:

迁移指南
----------

### SQLExpression 升级

类型支持的要点是：

1. 所有映射类中 `Column`，`relationship` 和 `mapped_column` 都应该用类型化建模。
2. ORM classes 根据 Python 类型进行建模，因此应该使用 `Mapped` 泛型。
3. `column` 作为直接函数调用仍然支持。这意味着您不必修改查询构造，但如果想增加类型信息，则需要通过 `type annotations` 实现。

<br/>

提请 Buildstream 维护者留意此次更改后 Buildstream 中的 SQLAlchemy 版本。    from sqlalchemy import ForeignKey
    from sqlalchemy import String
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship

    # declarative base from previous example
    str50 = Annotated[str, 50]

    # 基础类, 包含了一系列特定字段的注释.
    class Base(DeclarativeBase):
        type_annotation_map = {
            str50: String(50),
        }

    # 为 mapped_column() 方法进行重载, 使用常规的基础列风格, 所以要使用多个地方的格式.
    intpk = Annotated[int, mapped_column(primary_key=True)]
    user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

    # 描述用户模型的信息.
    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[intpk] 
        name: Mapped[str50] 
        fullname: Mapped[Optional[str]] 
        addresses: Mapped[List["Address"]] = relationship(back_populates="user")

    # 描述地址模型的信息.
    class Address(Base):
        __tablename__ = "address"

        id: Mapped[intpk] 
        email_address: Mapped[str50] 
        user_id: Mapped[user_fk] 
        user: Mapped["User"] = relationship(back_populates="addresses")

上面的代码展示了用 ``Mapped[str50]``, ``Mapped[intpk]``
或者是 ``Mapped[user_fk]`` 创建的列是如何从
  :paramref:`_orm.registry.type_annotation_map`   以及 ` `Annotated`` 构建中构建出来的，这样就可以重用预先建立的映射和列配置了。

可选的步骤-将映射类转换成 dataclasses_ 
++++++++++++++++++++++++++++++++++++++++++

我们可以将映射类转换成 dataclasses_, 这样
一个重要的优势之一就是我们可以构建一个严格强类型的 ``__init__()`` 方法，
其中包括了显式位置参数，使用关键字参数以及默认参数，更不必说我们还可以获得
许多本身就处于 dataclass 中的其他方法如 ``__str__()`` 和 ``__repr__()``
。dataclasses 也提供了序列化方法，例如
`dataclasses.asdict() <https://docs.python.org/3/library/dataclasses.html#dataclasses.asdict>`_ 和
`dataclasses.astuple() <https://docs.python.org/3/library/dataclasses.html#dataclasses.astuple>`_,
这些方法对于构建具有双向关系的映射所用，并不是特别适用。Python 类目前无法支持循环引用的解决方案。下一个章节   :ref:`whatsnew_20_dataclasses`  更详细地阐述了修改示例模型。

从第三步开始，开始支持类型
+++++++++++++++++++++++++++++

在上述示例中，从第三步开始的任何示例都将包含模型属性，并且将填充到  :func:`_sql.select` ,   :class:` _orm.Query` , 和   :class:`_engine.Row`  对象中:：

    # (variable) stmt: Select[Tuple[int, str]]
    stmt = select(User.id, User.name)

    with Session(e) as sess:
        for row in sess.execute(stmt):
            # (variable) row: Row[Tuple[int, str]]
            print(row)

        # (variable) users: Sequence[User]
        users = sess.scalars(select(User)).all()

        # (variable) users_legacy: List[User]
        users_legacy = sess.query(User).all()

.. seealso::

      :ref:`orm_declarative_table`  - Declarative generation and mapping of   :class:` .Table`  columns.

.. _whatsnew_20_mypy_legacy_models:

使用

**************************

使用

。。。

.. _whatsnew_20_dataclasses:

原生支持作为 ORM 模型封装的 dataclasses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 1.4 引入了新特性，此特性将  :pep:`593`  中的 “Annotated” 与基础列结合使用，可以用来通过类型注解创建映射。我们可以进一步将映射转换为 dataclasses_，其中一个重要优势是，我们可以构建一个严格强类型的 ` `__init__()`` 方法，其中包括了显式位置参数，使用关键字参数以及默认参数，不必记住

我们还可以获得 dataclass 内置的方法，例如 ``__str__()`` 和 ``__repr__()``。

dataclass 序列化方法，例如`dataclasses.asdict() <https://docs.python.org/3/library/dataclasses.html#dataclasses.asdict>`_ 和 `dataclasses.astuple() <https://docs.python.org/3/library/dataclasses.html#dataclasses.astuple>`_，此方法可以使用，但不能处理自引用结构，因此不适用于具有双向关系的映射。



SQLAlchemy 的当前集成方法将用户定义的类转换为**真实的 dataclass**，以为运行时的操作提供帮助；使用这个特性，实现使用  :pep:`681`  来识别 :ref:

dataclass 兼容的类，或已通过替代 API 声明的完全 dataclass 的类。提供了这个特性，是因为当前版本允许转换「dataclass」类型使其具备 SQLAlchemy 1.4 中所引入的现有 dataclass 特性，产生等价的运行时映射，并支持一种完全集成的配置样式，比上一个版本更完全地实现了类型。



为了使用 dataclasses 满足  :pep:`681` ，ORM 构建如   :func:` _orm.mapped_column`  和   :func:`_orm.relationship`  在其注释中接受额外的  :pep:` 681`  参数 ``init``，``default``，和 ``default_factory``，这些参数将传递到数据类创建过程中。这些参数当前必须存在于右侧的显式指令中，就像在 ``dataclasses.field()`` 中使用的方式一样；它们当前不能作为左侧“Annotated”构造中的局部变量。为了支持使用 ``Annotated`` 的方便性同时又支持数据类配置，  :func:`_orm.mapped_column`  可以将最小化的右侧参数与位于左侧的现有   :func:` _orm.mapped_column`  构建中的参数进行合并，以便保持大部分简洁性，这个后面的示例可以看到。

要在继承类中启用数据类，我们使用   :class:`.MappedAsDataclass`  混入类，可以直接应用于每个类，也可以应用于“Base”类，如下例所示，我们从 “步骤 5” of  :ref:` whatsnew_20_orm_declarative_typing`  开始修改示例映射：



    from typing_extensions import Annotated

    from typing import List

    from typing import Optional

    from sqlalchemy import ForeignKey

    from sqlalchemy import String

    from sqlalchemy.orm import DeclarativeBase

    from sqlalchemy.orm import Mapped

    from sqlalchemy.orm import MappedAsDataclass

    from sqlalchemy.orm import mapped_column

    from sqlalchemy.orm import relationship





    class Base(MappedAsDataclass, DeclarativeBase):

        """子类将被转换为 dataclasses"""





    intpk = Annotated[int, mapped_column(primary_key=True)]

    str30 = Annotated[str, mapped_column(String(30))]

    user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]
 :ref: :ref:



 :ref:
    class User(Base):

        __tablename__ = "user_account"


 :ref:
        id: Mapped[intpk] = mapped_column(init=False)

        name: Mapped[str30]

        fullname: Mapped[Optional[str]] = mapped_column(default=None)

        addresses: Mapped[List["Address"]] = relationship(

            back_populates="user", default_factory=list

        )





    class Address(Base):

        __tablename__ = "address"



        id: Mapped[intpk] = mapped_column(init=False)

        email_address: Mapped[str]

        user_id: Mapped[user_fk] = mapped_column(init=False)

        user: Mapped["User"] = relationship(back_populates="addresses", default=None)



与前例类似，该映射使用了直接应用 ``@dataclasses.dataclass`` 装饰器，同时声明了映射的声明性映射。这个装饰器会自动安装每个 ``dataclasses.field()`` 指示。 ``用户''/``地址''结构可以使用如下配置基于位置参数进行创建：



    >>> u1 = User("username", fullname="full name", addresses=[Address("email@address")])
 :ref:
    >>> u1

    User(id=None, name='username', fullname='full name', addresses=[Address(id=None, email_address='email@address', user_id=None, user=...)])





  :class:`_orm.WriteOnlyCollection`  功能集成，包括批量INSERT、UPDATE/DELETE和WHERE条件，都包括RETURNING支持。请参见完整文档：  :ref:` write_only_relationship`  。

.. seealso::

     :ref:`write_only_relationship` 

Python3.7 /类型批注动态关系支持
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

虽然“动态”关系在2.0中是遗留下来的，但是由于这些模式被认为具有长寿命，
因此在“动态”关系中添加了  :ref:`类型批注映射<whatsnew_20_orm_declarative_typing>`  支持
与新的``lazy="write_only"``，使用  :class:`_orm.DynamicMapped`  注释：

    from sqlalchemy.orm import DynamicMapped


    class Base(DeclarativeBase):
        pass


    class Account(Base):
        __tablename__ = "account"
        id: Mapped[int] = mapped_column(primary_key=True)
     :ref:dentifier: Mapped[str]
        account_transactions: DynamicMapped["AccountTransaction"] = relationship(
            cascade="all, delete-orphan",
            passive_deletes=True,
            order_by="AccountTransaction.timestamp",
        )


    class AccountTransaction(Base):
        __tablename__ = "account_transaction"
        id: Mapped[int] = mapped_column(primary_key=True)
        account_id: Mapped[int] = mapped_column(
            ForeignKey("account.id", ondelete="cascade")
        )
        description: Mapped[str]
        amount: Mapped[Decimal]
        timestamp: Mapped[datetime] = mapped_column(default=func.now())

上面的映射提供了一个``Account.account_transactions``集合，该
集合通过类型标记为返回 :class:`_orm.AppenderQuery` 集合类型，
包括其元素类型，例如``AppenderQuery[AccountTransaction]``。
这样，迭代和查询将产生打上了``AccountTransaction``类型标记的对象。

.. seealso::

     :ref:`dynamic_relationship` 


  :ticket:`7123`  

.. _change_7311:

安装现在完全支持pep-517
---------------------------

源分发现在包含一个``pyproject.toml``文件，允许完全支持  :pep:`517`  。
特别是，这允许使用``pip``对本地源构建进行自动安装
使用Cython_可选依赖项。

  :ticket:`7311`  

.. _change_7256:

C扩展现已移植到Cython
-------------------------

SQLAlchemy C扩展现已被全部采用用Cython_编写。虽然在2010年C扩展诞生之初就曾经评估过Cython，但是C扩展的性质和重点现在与当时相比发生了很大变化。与此同时，Cython显然也有了很大的发展，Python编译/分发工具链也有了显著的进步，这使得我们可以重新考虑它的使用。

转换到Cython提供了显着的新优势，但没有明显的劣势：
* 用来替代特定C扩展的Cython扩展的所有基准测试都比SQLAlchemy先前包括的几乎所有C代码**更快**，有时略微，但有时明显。虽然这似乎令人惊讶，但它似乎是Cython实现中的非明显优化的产物，如果是Python到C直接处理函数的直接传输，即包括C扩展中添加的许多自定义集合类型的情况，否则实现中就不存在这些优化，这正如许多C扩展中的情况。
* Cython扩展比原始C代码更容易编写、维护和调试，而且在大多数情况下与Python代码是一一对应的。
* Cython非常成熟和广泛使用，包括作为SQLAlchemy支持的一些知名数据库驱动程序的基础，包括“asyncpg”、“psycopg3”和“asyncmy”。

与前一个C扩展一样，Cython扩展在SQLAlchemy的轮分发中预先构建，可从PyPi的“pip”自动获取。除了Cython要求之外，手动构建说明也没有改变。

.. seealso::

     :ref:`c_extensions` 


  :ticket:`7256`  


.. _change_4379:

数据库反射的主要架构、性能和API增强
---------------------------------------------------------------

 :class:`.Table` 对象及其组件反映的内部系统已完全重构，
允许参与的方言一次性反射数千个表，获得很高的性能。
目前，**PostgreSQL**和**Oracle**方言参与了新架构，
其中PostgreSQL方言现在可以几乎三倍快地反射大量的  :class:`.Table` 
对象，而Oracle方言现在可以十倍快地反射大量的  :class:`.Table` 
对象。

重新架构最直接适用于使用SELECT查询对系统目录表进行表反射的方言，
并且剩下 :ref:从此方法中受益。",
包括"views" 和 "materialized views" 里的处理分离出来了，
因为在现实世界的案例中，这两个构造使用了不同的CREATE和DROP DDL，
这包括现在有单独的  :meth:`.Inspector.get_view_names`   和  :meth:` .Inspector.get_materialized_view_names`  方法。

新的API与以前的系统向后兼容，并且不应该要求第三方方言进行任何更改，以保持兼容性。 第三方方言也可以选择加入新系统，通过实现批处理查询以进行模式反射。

随着这种变化， :class:`.Inspector` 对象的API和行为得到了改进和增强，具有更一致的跨方言行为和更多的方法以及新的性能特性。

性能概述
~~~~~~~~~~~~~~~~~~~~

源分发包 :ref:`` test/perf/many_table_reflection.py``，它对新旧反射功能进行基准测试。
一组较老版本的测试可能会在旧版本上运行，这里我们用它来说明在本
篇文档中，我们将``metadata.reflect()``应用于250个 :class:`.Table` 对象时两个SQLA版本之间性能差异的情况：

===========================  ==================================  ====================  ====================
Dialect                      Operation                           SQLA 1.4 Time (secs)  SQLA 2.0 Time (secs)
---------------------------  ----------------------------------  --------------------  --------------------
postgresql+psycopg2          ``metadata.reflect()``, 250 tables  8.2                   3.3
oracle+cx_oracle             ``metadata.reflect()``, 250 tables  60.4                  6.8
===========================  ==================================  ====================  ====================


``Inspector()``的行为变化
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

针对包含在SQLAlchemy的方言SQLite、PostgreSQL、MySQL/MariaDB、Oracle和SQL Server中的:meth:`.Inspector
.has_table`、  :meth:`.Inspector.has_sequence`  、  :meth:` .Inspector.has_index`  、:meth:`.Inspector
.get_table_names`和  :meth:`.Inspector.get_sequence_names`  现在均具有一致的缓存行为：它们在首次针对特定的  :class:` .Inspector` .Inspector`对象时创建或删除表/序列时，将不会收到更新的状态。当执行DDL更改时应该使用  :meth:`.Inspector.clear_cache`  或新的  :class:` .Inspector` 。之前  :meth:`.Inspector.has_table`  、  :meth:` .Inspector.has_sequence`  方法既未实现缓存，也未对
  :meth:`.Inspector.get_table_names`  和  :meth:` .Inspector.get_sequence_names`  方法支持缓存，导致两种类型的方法之间结果的不一致。

对于第三方方言，行为取决于它们是否实现了dialect级的“反射缓存”装饰器的方法实现。

新方法和Inspector()的改进
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* 添加一个  :meth:`.Inspector.has_schema`   方法，返回目标数据库中是否存在模式
* 添加一个  :meth:`.Inspector.has_index`  方法，返回表部分是否具有特定的索引。
* 如  :meth:`.Inspector.get_columns`  单表操作的检查约束方法，现在治愈了DDL更改时受影响的表或视图不存在的情况。这个变化是特定针对方言的，所以当前第三方方言的处理可能还不一样。
* 将处理“视图”和“材质化视图”的方式分离，因为在实际使用中，这两个构造使用CREATE和DROP时使用不同的DDL；这包括现在有单独的  :meth:`.Inspector.get_view_names`  和  :meth:` .Inspector.get_materialized_view_names`  方法。

  :ticket:`4379`  

.. _ :ref:t_6842:

方言支持psycopg 3（也称“psycopg”）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

新增了对`psycopg 3 <https://pypi.org/project/psycopg/>`_ DBAPI的方言支持，尽管其编号为“3”，但现在已更名为“psycopg" ，取代了此前的“psycopg2”，对于现在仍然是SQLAlchemy的“默认”驱动程序的“postgresql”方言。 ``psycopg``是一个针对PostgreSQL的完全重写和现代化的数据库适配器，支持预处理语句以及Python asyncio等概念。

``psycopg``是SQLAlchemy提供的第一个既提供pep-249同步API，又提供asyncio驱动程序的DBAPI。 ``psycopg``数据库URL可用于 :func:`_sa.

create_engine`和 :func:`_asyncio.create_async_engine` 引擎创建函数，并自动选择相应键入的方言或并发dialect。

.. seealso::

     :ref:`postgresql_psycopg` 


.. _ticket_8054:

Dialect support for oracledb
----------------------------

新增了针对`oracledb <https://pypi.org/project/oracledb/>`_ DBAPI的方言支持，这是流行的cx_Oracle驱动程序的重命名、新的主要发布版本。

.. seealso::

     :ref:`oracledb` 

.. _ticket_7631:

新的约束和索引条件DDL
-----------------------------------------------

新方法  :meth:`_schema.Constraint.ddl_if`  和  :meth:` _schema.Index.ddl_if`  允许根据快速查询所接受的条件，对  :class:`_schema.CheckConstraint` 、 :class:` _schema.UniqueConstraint`和 :class:`_schema.Index` 等对象进行条件性布局。在下面的示例中，CHECK约束和索引只有在PostgreSQL后端下才会产生：

    meta = MetaData()

    my_table = Table(
        "my_table",
        meta,
        Column("id", Integer, primary_key=True),
        Column("num", Integer),
        Column("data", String),
        Index("my_pg_index", "data").ddl_if(dialect="postgresql"),
        CheckConstraint("num > 5").ddl_if(dialect="postgresql"),
    )

    e1 = create_engine("sqlite://", echo=True)
    meta.create_all(e1)  # 将不会生成CHECK和INDEX


    e2 = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
    meta.create_all(e2)  # 将生成CHECK和INDEX

在上述情况下， :class:`_orm.Session` 根本不应该影响现有的事务状态，应当自己创建保存点(nested transaction)。

.. seealso::

     :ref:`schema_ddl_ddl_if` 

.. _change_5052:

DATE，TIME，DATETIME数据类型现在在所有后端都支持文本呈现
-----------------------------------------------------------------------------

现在，使用后端特定编译，包括PostgreSQL和Oracle，实现了日期和时间类型的文字呈现：

    >>> import datetime

    >>> from sqlalchemy import DATETIME
    >>> from sqlalchemy import literal
    >>> from sqlalchemy.dialects import oracle
    >>> from sqlalchemy.dialects import postgresql

    >>> date_literal = literal(datetime.datetime.now(), DATETIME)

    >>> print(
    ...     date_literal.compile(
    ...         dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}
    ...     )
    ... )
    {printsql}'2022-12-17 11:02:13.575789'{stop}

    >>> print(
    ...     date_literal.compile(
    ...         dialect=oracle.dialect(), compile_kwargs={"literal_binds": True}
    ...     ) :ref:
    ... )
    {printsql}TO_TIMESTAMP('2022-12-17 11:02:13.575789', 'YYYY-MM-DD HH24:MI:SS.FF'){stop}
 :ref:
以前，当使用与dialect特定类型无关的字符串调用“文字呈现”时，会支持这样的文本呈现，但当使用dialect-specific type时，除非特定相关类型伪造字符串格式的实现，否则会引发```NotImplementedError```，从1.4.45开始成为  :class:`.CompileError`  )。

使用使用句柄字面值。 这里是字符串化的带有明文密码的URL，使用  :meth:`_url.URL.render_as_string`   方法。

跨端口连接以及其他情况如下：

    # to see sql with unobfuscated passwords for unittests,
    # convert url to string first and pass the hide_password=False
    # parameter to the render_as_string method call.
    >>> str(engine.url).replace("***", "password")
    'postgresql://user:password@localhost:5432/dbname'

  :ticket:`8567`  


.. _change_8710:

``Result``, ``AsyncResult``现在支持上下文管理器
-------------------------------------------------------

 :class:`.Result` 对象现在支持上下文管理器使用，这将确保在块的结束时关闭对象及其底层游标。这在使用服务器端游标时特别有用，在这种情况下，重要的是，即使发生用户定义的异常，未经提交的更改（未关闭的游标）也会在操作结束时得到关闭::

    with engine.connect() as conn:
        with conn.execution_options(yield_per=100).execute(
            text("select * from table")
        ) as result:
            for row in result:
                print(f"{row}")

使用异步的情况下，  :class:`.AsyncResult` .AsyncConnection` 已更改，以提供可选的异步上下文管理器使用，例如::

    async with async_engine.connect() as conn:
        async with conn.execution_options(yield_per=100).execute(
            text("select * from table")
        ) as result:
            for row in result:
                print(f"{row}")

  :ticket:`8710`  

行为变化
------------------

此节介绍了SQLAlchemy 2.0中的行为变化，这些变化不在主要的1.4->2.0迁移路径中，这些变化不应该对向后兼容性产生重大影响。


.. _change_9015:

新的事务加入模式 ``Session``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

“将外部事务加入到Session`中的行为已被修改和改进，允许明确控制  :class:`_orm.Session` 。新的参数  :paramref:` _orm.Session.join_transaction_mode` 。最重要的是，它允许：class:`_orm.Session`仅使用保存点开始全事务处理运输，而不将外部启动的事务提交或回滚。

这主要优化的改进是已经在文档  :ref:`session_external_transaction`  中进行了详细记录，其中还包括了从SQLAlchemy1.3到1.4的变化它现在已经简化为不再需要明确使用事件处理程序或任何提到过的显式保存点（savepoint）; 通过使用` `join_transaction_mode="create_savepoint"``， :class:`_orm.Session` 永远不会影响现有事务的状态，而将始终自己创建保存点（savepoint）。


以下是给出了示例，它在 :ref:`session_external_transaction` 中给出了；参见该节以获取完整示例。

    class SomeTest(TestCase):
        def setUp(self):
            # connect to the database
            self.connection = engine.connect()

            # begin a non-ORM transaction
            self.trans = self.connection.begin()

            # bind an individual Session to the connection, selecting
            # "create_savepoint" join_transaction_mode
            self.session = Session(
                bind=self.connection, join_transaction_mode="create_savepoint"
            )

        def tearDown(self):
            self.session.close()

            # rollback non-ORM transaction
            self.trans.rollback()

            # return connection to the Engine
            self.connection.close()

默认情况下选择的  :paramref:`_orm.Session.join_transaction_mode`  是 ` `"conditional_savepoint"``模式，如果给定的  :class:`_engine.Connection` ：class:` _engine.Engine`已经在保存点上，会使用``"create_savepoint"``行为。如果给定的 :class:`_engine.Connection` 处于已经存在的事务而不是保存点，则 :class:`_orm.Session` 将传播“rollback”呼叫，但不会传播“commit”呼叫，但是不会启动自己的保存点。这种行为被默认选定，因为它最大限度地兼容旧版SQLAlchemy的兼容性，且它会在当前给定驱动程序可以使用SAVEPOINT的情况下开始新SAVEPOINT ，因为对SAVEPOINT的支持不仅在特定后端和驱动程序，还在特定配置中变化。

加入在事务中已经启动的 :class:`_engine.Connection` 的新代码应显式选择 :paramref:`_orm.Session.join_transaction_mode` 参数，以便显式定义所需的行为。

  :ticket:`9015`  

.. _Cython: https://cython.org/

.. _change_8567:

``str(engine.url)`` 默认情况下会隐藏密码
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了避免数据库密码泄漏，在  :class:`.URL` ` str()``
现将默认启用密码隐藏功能。以前，这种隐藏会在``__repr__()``调用上启用，但在``__str__()``时不会启用。这对于企图使用另一个引擎的字符串化URL来调用 :func:`_sa.create_engine` 的应用程序和测试套件可能会产生影响，例如：

    >>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
    >>> e2 = create_engine(str(e1.url))

上述代码中，引擎``e2``不会包含正确的密码字符串，它将包含替换成``"***"``的字符串。

上述模式的首选方法是直接传递:class`URL`对象，没有必要将其转换为字符串::

    >>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
    >>> e2 = create_engine(e1.url)

否则，对于带有明文密码的字符串URL，可以使用  :meth:`_url.URL.render_as_string`  方法，并传递参数  :paramref:` _url.URL.render_as_string.hide_password`  为``False``， 如下所示：

    >>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
    >>> url_string = e1.url.render_as_string(hide_password=False)
    >>> e2 = create_engine(url_string)


  :ticket:`8567`  

.. _change_8925:

以相同名称、键的列替换Table对象严格规则
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

严格规则已经制定来做到以下几点：

* 毫无疑问，  :class:`.Table` .Column` 对象，无论它们有什么 .key。已经识别和修复了一种可能出现这种情况的边缘情况。

* 添加与现有的  :class:`.Column` .Column` 会始终引发  :class:`.DuplicateColumnError` ，除非存在其他参数;  :paramref:` .Table.append_column.replace_existing`  用于  :meth:`.Table.append_column`  ，以及用于构造名字相同的 :class:` .Table`作为一个现有的，带或不带反射都可以。此前这是一个弃用警告。

* 如果一个  :class:`.Table` .Column`  全部替换，则会发出警告，这意味着操作不是用户想要的。尤其是在二次反射阶段下，比如  ` `metadata.reflect(extend_existing=True)``。警告建议将  :paramref:`.Table.autoload_replace` ` False``，以防止这种情况的出现。在1.4及以前的版本中，在此情况下，传入的列仅会**另外**添加到现有列中。这是一个2.0中的bug，从2.0.0b4开始是行为上的变化，因为在此情况下，旧的key将**不再在**列集合中存在。

  :ticket:`8925`  

.. _change_9297:

ORM映射的列顺序有所不同;使用``sort_order``控制行为
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Declarative已更改了未同实际申明类的   :class:`.Column`  排序方式，以及这些的映射列作为从declare类本身派生的mixin或抽象基类中的列的方式，以将来自申明类自身的列放在前面，然后是自mixin列。以下映射::

    class Foo:
        col1 = mapped_column(Integer)
        col3 = mapped_column(Integer)


    class Bar:
        col2 = mapped_column(Integer)
        col4 = mapped_column(Integer)


    class Model(Base, Foo, Bar):
        id = mapped_column(Integer, primary_key=True)
        __tablename__ = "model"

在1.4上提供的CREATE TABLE为：

.. sourcecode:: sql

    CREATE TABLE model (
      col1 INTEGER,
      col3 INTEGER,
      col2 INTEGER,
      col4 INTEGER,
      id INTEGER NOT NULL,
      PRIMARY KEY (id)
    )

但是在2.0上，则是：

.. sourcecode:: sql

    CREATE TABLE model (
      id INTEGER NOT NULL,
      col1 INTEGER,
      col3 INTEGER,
      col2 INTEGER,
      col4 INTEGER,
      PRIMARY KEY (id)
    )

对于上面的特殊情况，可以看作是改进，因为"Model"上 pri众键列现在位于你通常会首选的位置。然而，这不能让申declare模型较早的应用了这个功能的程序获得安慰：

    class Foo:
        id = mapped_column(Integer, primary_key=True)
        col1 = mapped_column(Integer)
        col3 = mapped_column(Integer)


    class Model(Foo, Base):
        col2 = mapped_column(Integer)
        col4 = mapped_column(Integer)
        __tablename__ = "model"

现在的CREATE TABLE输出如下：

.. sourcecode:: sql

    CREATE TABLE model (
      col2 INTEGER,
      col4 INTEGER,
      id INTEGER NOT NULL,
      col1 INTEGER,
      col3 INTEGER,
      PRIMARY KEY (id)
    )

为了解决此问题，SQLAlchemy 2.0.4引入了对  :func:`_orm.mapped_column` 
的新参数  :paramref:`_orm.mapped_column.sort_order` ，这是一个整数值，缺省为` `0``，可以设置为正值或负值，这样可以将列置于其他列之前或之后，例如下面的示例::

    class Foo:
        id = mapped_column(Integer, primary_key=True, sort_order=-10)
        col1 = mapped_column(Integer, sort_order=-1)
        col3 = mapped_column(Integer)


    class Model(Foo, Base):
        col2 = mapped_column(Integer)
        col4 = mapped_column(Integer)
        __tablename__ = "model"

上述模型在所有列之前放置“id”，在“id”之后放置“col1”：

.. sourcecode:: sql

    CREATE TABLE model (
      id INTEGER NOT NULL,
      col1 INTEGER,
      col2 INTEGER,
      col4 INTEGER,
      col3 INTEGER,
      PRIMARY KEY (id)
    )

在将来的SQLAlchemy版本中，可以选择为 :class:`_orm.mapped_column` 构造提供显式排序提示，因为这种排序只针对ORM。

.. _change_7211:

``Sequence``构造现在默认不具有任何显式的默认“开始”值；影响到SQL Server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在SQLAlchemy1.4之前，如果未指定其他参数，则  :class:`.Sequence` ` CREATE SEQUENCE``DDL：

.. sourcecode:: pycon+sql

    >>> s = Sequence("my_sequence")
    >>> print(CreateSequence(s).compile(dialect=default))
    CREATE SEQUENCE my_sequence START 1 INCREMENT BY 1 MAXVALUE 9223372036854775807
    MINVALUE -9223372036854775808 CYCLE CACHE 1 NO CYCLE

但是，对于SQL Server方言，``START``关键字是一个必须的参数。为避免问题，  :class:`.Sequence` ` TypeError``，但是可以将 ``start=1``传递给构造函数来解决这个问题，如下所示：

    my_sequence = Sequence("my_sequence", start=1)

对于其他方言，使用默认值仍然可以正常工作。

这是一个从1.4.9起的行为变化。

  :ticket:`7211`      >>> # SQLAlchemy 1.3 (and 2.0)
    >>> from sqlalchemy import Sequence
    >>> from sqlalchemy.schema import CreateSequence
    >>> print(CreateSequence(Sequence("my_seq")))
    {printsql}CREATE SEQUENCE my_seq

然而，随着增加了对MS SQL Server的  :class:`.Sequence` ` -2**63``，因此，版本1.4决定，如果没有提供  :paramref:`.Sequence.start`  ，则将DDL默认发出起始值为1：

.. sourcecode:: pycon+sql

    >>> # SQLAlchemy 1.4 (only)
    >>> from sqlalchemy import Sequence
    >>> from sqlalchemy.schema import CreateSequence
    >>> print(CreateSequence(Sequence("my_seq")))
    {printsql}CREATE SEQUENCE my_seq START WITH 1

这个变化引入了其他复杂性，其中包括当包括  :paramref:`.Sequence.min_value`  参数时，这个默认值` `1``实际上应该默认为  :paramref:`.Sequence.min_value`  所指定的值，否则，小于起始值的min_value可能会被视为相互矛盾。首先，查看这个问题开始成为其他各种边缘情况的一个兔子洞，所以我们决定取消这个变化并恢复  :class:` .Sequence` `SEQUENCE``的各个参数应如何相互作用。

因此，为确保起始值在所有后端都是1，**必须明确指出起始值1**，如下所示：

.. sourcecode:: pycon+sql

    >>> # All SQLAlchemy versions
    >>> from sqlalchemy import Sequence
    >>> from sqlalchemy.schema import CreateSequence
    >>> print(CreateSequence(Sequence("my_seq", start=1)))
    {printsql}CREATE SEQUENCE my_seq START WITH 1

除此之外，在现代后端（包括PostgreSQL、Oracle、SQL Server）上自动生成整数主键的情况下，应优先使用 :class:`.Identity` 构造，它在1.4和2.0中工作方式相同，行为没有任何变化。

  :ticket:`7211`  


.. _change_6980:

``with_variant()``方法克隆原始的TypeEngine而不是更改类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :meth:`_sqltypes.TypeEngine.with_variant`  方法用于为特定类型应用替代的数据库行为，现在会返回原始的  :class:` _sqltypes.TypeEngine` `Variant``类中。

虽然以前的``Variant``方法能够使用动态属性getter维护所有在Python中的行为，但这里的改进是，当调用一个变量时，返回的类型仍然是原始类型的实例，这在类型检测器（如mypy和pylance）中更加流畅。如下程序所示：

    import typing

    from sqlalchemy import String
    from sqlalchemy.dialects.mysql import VARCHAR

    type_ = String(255).with_variant(VARCHAR(255, charset="utf8mb4"), "mysql", "mariadb")

    if typing.TYPE_CHECKING:
        reveal_type(type_)

像pyright这样的类型检测器现在将报告类型为：

.. sourcecode:: text

    info: Type of "type_" is "String"

此外，对于单个类型可以通过传递多个方言名称来提高可读性和可维护性，这在使用与``"mysql"``和``"mariadb"``方言不一样时最为有效。此时，在1.4和2.0中，多个方言名称可以用于相同的类型名称。

  :ticket:`6980`  


.. _change_4926:

Python除法操作符在所有后端上执行真除法；增加整数除法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Core表达式语言现在支持“真除法”（即``/``Python操作符）和“地板除法”（即``//``Python操作符），包括后端特定的行为，以规范化在这方面不同的数据库。

给定针对两个整数值的“真除法”操作：

     :ref:= literal(5, Integer) / literal(10, Integer)

例如，在PostgreSQL上，正常情况下，SQL除法运算符在针对整数使用时的作用是“地板除法”，这意味着上述结果将返回整数“0”。对于这些和类似的后端，SQLAlchemy现在使用了一个等价于以下形式的表单来呈现SQL：

.. sourcecode:: sql

    %(param_1)s / CAST(%(param_2)s AS NUMERIC)

其中，``param_1=5``，``param_2=10``，所以返回表达式将是NUMERIC类型，通常作为Python值``decimal.Decimal("0.5")``。

在给定两个整数值的“地板除法”操作时：

     expr = literal(5, Integer) // literal(10, Integer)

例如，对于MySQL和Oracle等后端，使用对整数的“真除法”，这意味着上述结果将返回浮点值 “0.5”。对于这些和类似的后端，SQLAlchemy现在使用了一个等价于以下形式的表单来呈现SQL：

… code-block

    FLOOR(%(param_1)s / %(param_2)s)

其中param_1=5，param_2=10，因此返回表达式将是INTEGER类型，作为Python值``0``。

这里的向后不兼容的更改是，如果一个应用程序在PostgreSQL、SQL Server或SQLite上，依赖Python“true div”操作符在所有情况下返回整数值的情况下失败。如果应用程序依赖此行为，则对于这些操作应该使用Python“floor division”运算符``//``或向前兼容性，使用之前的SQLAlchemy版本时使用floor函数：

    expr = func.floor(literal(5, Integer) / literal(10, Integer))

在SQLAlchemy 2.0之前的任何版本中，上述形式都需要使支持后端自适应地完成地板除法。

  :ticket:`4926`  


.. _change_7433:

Session提示检测到非法并发或可重入访问时引发
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现在， :class:`_orm.Session` 可以捕获更多的与多线程或其他并发情况中的非法并发状态更改和执行意外状态更改的事件挂钩相关的错误。

当同时在多个线程中使用 :class:`_orm.Session` 时，已知会发生一种错误是
``AttributeError: 'NoneType' object has no attribute 'twophase'``，这是完全神秘的。当一个线程调用  :meth:`_orm.Session.commit`  时，它内部调用  :meth:` _orm.SessionTransaction.close`  方法来结束事务上下文，同时另一个线程正在运行一个查询，如  :meth:`_orm.Session.execute`  中的查询，而  :meth:` _orm.Session.execute`  会使用当前事务获取数据库连接的内部方法开头，首先assert该会话为“active”，但是这个断言通过后，同时调用  :meth:`_orm.Session.close`  调用干扰了这个状态，导致上述未定义的条件。
 :ref:
该更改对围绕  :class:`_orm.SessionTransaction`  方法将无法更改状态，而是寻求改变状态为：正在为当前连接获取数据库查询的方法已在进程中进行，从而导致一个不允许的状态。

对于asyncio，需要注意，如果将特定的 :class:`_orm.Session` 共享到多个asyncio任务中，则此更改也适用。

:tic :ref:7433`


.. _change_7490:

SQLite方言对基于文件的数据库使用QueuePool
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当使用基于文件的数据库时，SQLite方言现在默认使用  :class:`_pool.QueuePool` 。它与设置` `check_same_thread=False``一起使用。发现默认使用  :class:`_pool.NullPool`  参数来自定义池类。

.. seealso::

     :ref:`pysqlite_threading_pooling`  :ref:

  :ticket:`7490`   :ref:

.. _change_5465_oracle:

新的带二进制精度的Oracle FLOAT类型；不直接接受十进制精度
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Oracle方言添加了一种新的数据类型  :class:`_oracle.FLOAT` ，配合添加的  :class:` _sqltypes.Double` 、 :class:`_sqltypes.DOUBLE_PRECISION` 和 :class:`_sqltypes.REAL` 数据类型。Oracle的“FLOAT”接受所谓的“二进制精度”参数，根据Oracle文档，这大致相当于将“精度”值除以0.3103：

    from sqlalchemy.dialects import oracle

    Table("some_table", metadata, Column("value", oracle.FLOAT(126)))

具有二进制精度值126等同于使用 :class:`_sqltypes.DOUBLE_PRECISION` 数据类型，值63等同于使用 :class:`_sqltypes.REAL` 数据类型。其他精度值是具体的 :class:`_oracle.FLOAT` 类型。

SQLAlchemy的  :class:`_sqltypes.Float` ，则Oracle方言现在会引发一个信息性错误。要使用显示精度值指定  :class:` _sqltypes.Float`  方法：

    from sqlalchemy.types import Float
    from sqlalchemy.dialects import oracle

    Table(
        "some_table",
        metadata,
        Column("value", Float(5).with_variant(oracle.FLOAT(16), "oracle")),
    )

.. _change_7156:

新的RANGE/MULTIRANGE支持和PostgreSQL后端的更改
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

已完全为psycopg2、psycopg3和asyncpg方言实现RANGE/MULTIRANGE支持。新支持使用新的SQLAlchemy特定的 :class:`_postgresql.Range` 对象，
该对象不依赖于不同的后端，也不需要使用后端特定的导入或扩展步骤。对于多范围支持，使用 :class:`_postgresql.Range` 对象的列表。

使用以前的psycopg2特定类型的代码应修改为使用  :class:`_postgresql.Range` ，这样可以呈现兼容的接口。

  :class:`_postgresql.Range`  和  :meth:` _postgresql.Range.contained_by`  方法，其工作方式与PostgreSQL的“@>”和“<@”相同。将来可以添加其他操作符支持。

请参阅  :ref:`postgresql_ranges`  中的文档，了解使用新特性的背景情况。


.. seealso::

     :ref:`postgresql_ranges` 

  :ticket:`7156`  
  :ticket:`8706`  

.. _change_7086:

PostgreSQL上的``match()``操作符使用``plainto_tsquery()``而不是``to_tsquery()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :meth:`.Operators.match`  函数现在使用PostgreSQL的` `plainto_tsquery(expr)``，而不是``to_tsquery()``。``plainto_tsquery()``接受纯文本，而``to_tsquery()``接受专门的查询符号，因此与其他后端比较，具有更低的跨兼容性。

通过使用  :data:`.func`  生成PostgreSQL特定函数和  :meth:` .Operators.bool_op`  （  :meth:`.Operators.op`  的布尔类型版本）来生成任意操作符，在  :meth:` .Operators.match`  中使用所有PostgreSQL搜索函数和操作符，从而在以前版本中的方式下可用，如同使用相关的示例：  :ref:`postgresql_match`  。

现有的SQLAlchemy项目使用  :meth:`.Operators.match`  中的PG特定指令，应直接使用` `func.to_tsquery()``。要在SQLAlchemy 1.4中呈现的SQL与1.4中存在的一样，请参见版本说明  :ref:`postgresql_simple_match`  。

  :ticket:`7086`  