.. _mypy_toplevel:

Mypy / Pep-484的ORM映射支持
============================

在使用SQLAlchemy的 :ref:`declarative <orm_declarative_mapper_config_toplevel>` 映射时，直接引用:class:`_schema.Column`
对象而非SQLAlchemy 2.0中引入的 :func:`_orm.mapped_column` 构造，支持:pep:`484`类型注解以及Mypy_类型检查工具。

.. deprecated:: 2.0

    **SQLAlchemy Mypy Plugin已不建议使用，将在SQLAlchemy 2.1发布时移除。请务必尽快进行迁移！**

    该插件无法在不断变化的Mypy版本中维护其稳定性，因此未来的稳定性无法保证。

    现代SQLAlchemy现已提供 :ref:`完全符合pep-484的映射语法<whatsnew_20_orm_declarative_typing>`的语法结构；
    可以查看相关部分以获取迁移详细信息。

.. topic:: SQLAlchemy Mypy 插件状态更新

   **更新于 2023 年 7 月**

   对于 SQLAlchemy 2.0，Mypy插件仍在SQLAlchemy 1.4版本中的级别上工作。不过，
   SQLAlchemy 2.0具有一种全新的类型系统 ，用于ORM声明模型，它消除了Mypy插件的
   需要，并具有通常情况下优秀的一致性和高级功能。
   请注意，此新功能 **不是SQLAlchemy 1.4的一部分，它仅存在于SQLAlchemy 2.0中**。

   虽然Mypy插件在技术上从未离开过“alpha”阶段，但现应**将其视为SQLAlchemy 2.0中不建议使用的插件，
   即使在使用SQLAlchemy 1.4时仍然需要该插件以支持 Mypy 的完整支持。**

   Mypy插件本身并不能解决使用诸如Pylance/Pyright、Pytype、Pycharm等其他类型工具提供正确类型的问题，
   这些工具无法使用Mypy插件。此外，开发、维护和测试Mypy插件非常困难，因为Mypy插件必须与Mypy内部的数据结构
   和进程深度集成，而Mypy内部的数据结构和进程本身也不稳定。当使用与基本模式不同的代码时，使用Mypy插件会有很多限制
   ，而这些代码可能会引发定期报告。

   出于这些原因，针对 Mypy 插件的新的非回归问题不太可能被修复。
   **在安装SQLAlchemy 2.0的同时使用Mypy插件，现有代码将继续通过所有检查而不需要任何更改**。
   SQLAlchemy 2.0的API与SQLAlchemy 1.4的API以及Mypy插件行为是完全向后兼容的。

   在SQLAlchemy 1.4上全部通过Mypy检查的代码，一旦该代码在完全运行于SQLAlchemy 2.0上，将可以逐步迁移到
   新结构。有关此迁移的背景，请参见：:ref:`whatsnew_20_orm_declarative_typing`。

   完全迁移到新声明结构的代码将完全符合pep-484，并能正常工作于IDE和其他类型工具中，而无需插件。


安装
------------

只对**SQLAlchemy 2.0有效**：不应安装任何笔尖，并且应完全卸载sqlalchemy-stubs_ 和 sqlalchemy2-stubs_等程序包。

Mypy_本身是依赖。

可以使用 pip 安装Mypy，使用“mypy”额外钩子：

.. sourcecode:: text

    pip install sqlalchemy[mypy]

该插件本身的配置方法如下所述：
`Configuring mypy to use Plugins <https://mypy.readthedocs.io/en/latest/extending_mypy.;html#configuring-mypy-to-use-plugins>`_
使用sqlalchemy的扩展.mypy.plugin模块名称，例如在
``setup.cfg`` 中::

    [mypy]
    plugins = sqlalchemy.ext.mypy.plugin

.. _sqlalchemy-stubs: https://github.com/dropbox/sqlalchemy-stubs

.. _sqlalchemy2-stubs: https://github.com/sqlalchemy/sqlalchemy2-stubs

插件作用
--------------------

Mypy插件的主要目的是截取并更改SQLAlchemy的静态定义
:ref:`declarative mappings <orm_declarative_mapper_config_toplevel>` 以使其与
:class:`_orm.Mapper` 对象对其进行*instrumented* 后的结构匹配。
对于Orm映射本身以及使用该类的代码，这允许Mypy工具的行为是可理解的，
否则会基于当前声明的已知方法**无法进行声明映射**。

与为库如`dataclasses <https://docs.python.org/3/library/dataclasses.html>`_ 所需
的类似插件相似，插件与在运行时动态修改类的类似插件不同。插件在以下
主要领域发挥作用：

* 使用 :func:`_orm.declarative_base` 生成“Base”的动态类将继承其子类的
  映射确定。它还可以在 :ref:`orm_declarative_decorator`中描述的类装饰器
  方法中实现。

* 针对以“内联”样式定义的ORM映射属性的类型推理。例如，在上面的例子中，
  ``User`` 类的 ``id`` 和 ``name`` 属性。这包括 ``User`` 实例将使用类型
  ``int``表示 ``id`` 和 ``str`` 表示 ``name``。这还包括当访问 ``User.id``
  和 ``User.name`` 类级别属性时（如上面的 ``select()`` 语句中），它们与
  派生自 :class:`_orm.InstrumentedAttribute` 的 SQL 表达式行为兼容。

* 为没有显式构造函数的映射类应用 ``__init__()`` 方法，该构造函数接受特定类型的
  关键字参数（如果已检测到）。

当Mypy插件处理以上文件时，传递给Mypy工具的静态类定义和Python代码将转化为以下内容：

    from sqlalchemy import Column, Integer, String, select
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm.decl_api import DeclarativeMeta


    class Base(metaclass=DeclarativeMeta):
        __abstract__ = True


    class User(Base):
        __tablename__ = "user"

        id: Mapped[Optional[int]] = Mapped._special_method(
            Column(Integer, primary_key=True)
        )
        name: Mapped[Optional[str]] = Mapped._special_method(Column(String))

        def __init__(self, id: Optional[int] = ..., name: Optional[str] = ...) -> None:
            ...


    some_user = User(id=5, name="user")

    print(f"Username: {some_user.name}")

    select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains("s"))

以上重要步骤包括：

* ``Base``类现在明确地定义为使用 :class:`_orm.DeclarativeMeta` 类，而不是
  动态类。

* 在 :class:`_orm.Mapped` 类中定义具有其特定Python类型的属性，代表Python检查器和
  SQLAlchemy orm检查器的兼容性，即从类级别访问这些属性和从实例级别访问这些属性之间
  应有的行为不同。

* 从声明映射属性分配中删除右侧。因为 :class:`_orm.Mapper`
  类通常会将这些属性替换为 :class:`_orm.InstrumentedAttribute` 的特定实例。
  原始表达式移动到函数调用中，这将允许仍然可以对其进行类型检查而不会与左侧冲突。
  Mypy仅需左侧类型注释即可理解其属性的行为。

* 为未包含显式构造函数的映射类添加 ``__init__()`` 方法，该函数接受已检测到的所有
  映射属性类型的关键字参数。

插件的使用
------

以下子章节将介绍迄今为止已考虑为pep-484兼容性的各个用例。


基于 TypeEngine 的列的内省
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于包括显式数据类型的映射列，当它们作为内联属性映射时，
映射类型将自动进行内省：

    class MyClass(Base):
        # ...

        id = Column(Integer, primary_key=True)
        name = Column("employee_name", String(50), nullable=False)
        other_name = Column(String(50))

以上代码段中的 ``id``，``name`` 和 ``other_name`` 的最终类级别数据类型
将被内省为 ``Mapped[Optional[int]]``，``Mapped[Optional[str]]`` 和 ``Mapped[Optional[str]]``。
默认情况下，这些类型始终被视为 ``Optional``，即使是主键和非空列也是如此。原因是
尽管数据库列“id”和“name”不能为NULL，但Python属性“id”和“name”肯定可以是
在没有显示构造函数的情况下为“None”的情况下：

    >>> m1 = MyClass()
    >>> m1.id
    None

以上列的类型可以明确说明为更清晰的自我文档和可控制的可选类型的两个优点：

    class MyClass(Base):
        # ...

        id: int = Column(Integer, primary_key=True)
        name: str = Column("employee_name", String(50), nullable=False)
        other_name: Optional[str] = Column(String(50))

Mypy插件将接受上述``int``, ``str``和``Optional[str]``,并将其转换为包围它们的``Mapped[]``类型。
也可以显式使用``Mapped[]``构造函数：

    from sqlalchemy.orm import Mapped


    class MyClass(Base):
        # ...

        id: Mapped[int] = Column(Integer, primary_key=True)
        name: Mapped[str] = Column("employee_name", String(50), nullable=False)
        other_name: Mapped[Optional[str]] = Column(String(50))

当类型是非可选的时，仅表示访问``MyClass`` 的实例时
将被视为非``None``：

    mc = MyClass(...)

    # 将通过mypy –strict测试
    name: str = mc.name

对于可选的属性，Mypy认为这种类型必须包含空值
或者是``Optional``：

    mc = MyClass(...)

    # 将通过mypy –strict测试
    other_name: Optional[str] = mc.name

无论映射属性被标记为``Optional``或不可选，
``__init__()`` 方法的生成仍将**考虑所有关键字都是可选的**。这再次
与验证系统（如Python ``dataclasses``）的行为不同，后者将生成一个
与注释匹配的构造函数，其中包括必需和可选属性的注释。


没有显式类型的列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于包含:class:`_schema.ForeignKey`修饰符的列，在SQLAlchemy声明映射中，
它们不需要声明显式数据类型。对于此类属性，插件将需要在左侧指定明确的类型注释：

    # .. 其他导入项
    from sqlalchemy.sql.schema import ForeignKey

    Base = declarative_base()


    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column(String)


    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        user_id = Column(ForeignKey("user.id"))

插件将以以下方式输出消息：

.. sourcecode:: text

    $ mypy test3.py --strict
    test3.py:20: error: [SQLAlchemy Mypy plugin] Can't infer type from
    ORM mapped expression assigned to attribute 'user_id'; please specify a
    Python type or Mapped[<python type>] on the left hand side.
    Found 1 error in 1 file (checked 1 source file)

为解决此问题，需要在 ``Address.user_id`` 列上应用明确的类型注释：

    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        user_id: int = Column(ForeignKey("user.id"))

映射带有命令性表的列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 :ref:`imperative table style <orm_imperative_table_configuration>` 中，
:class:`_schema.Column` 定义包含在:class:`_schema.Table` 构造函数内，该构造函数与映射属性本身分离。
插件不考虑此 :class:`_schema.Table`，而是支持可以使用的显式完整注释，必须使用 :class:`_orm.Mapped`
类来将其识别为映射属性：

    class MyClass(Base):
        __table__ = Table(
            "mytable",
            Base.metadata,
            Column(Integer, primary_key=True),
            Column("employee_name", String(50), nullable=False),
            Column(String(50)),
        )

        id: Mapped[int]
        name: Mapped[str]
        other_name: Mapped[Optional[str]]

上述 :class:`_orm.Mapped` 注释被认为是映射列，并将包含在默认构造函数中，同时为在类级别和实例级别正确提供
``MyClass`` 的输入行为提供了正确的定型概要。

映射关系
^^^^^^^^^^^^^^^^^^^^^^^

该插件仅能支持极少的使用类型推理检测关系类型能力。对于不能检测到其类型的所有这些关系，它将发出
⼀个 informative error message ，即可在所有案例中明确指定
类型，无论使用:class:`_orm.Mapped`类还是可以忽略其进行内联声明的类型。插件还需要确定关系是引用集合
还是标量，其中依赖于 :paramref:`_orm.relationship.uselist` 和 / 或 :paramref:`_orm.relationship.collection_class`
参数的显式值。如果这两个参数都不存在，则需要使用显式类型说明，就像 :func:`_orm.relationship` 目标类型是字符串或callable，而不是类一样：

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column(String)


    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        user_id: int = Column(ForeignKey("user.id"))

        user = relationship(User)

以上映射将产生以下错误：

.. sourcecode:: text

    test3.py:22: error: [SQLAlchemy Mypy plugin] Can't infer scalar or
    collection for ORM mapped expression assigned to attribute 'user'
    if both 'uselist' and 'collection_class' arguments are absent from the
    relationship(); please specify a type annotation on the left hand side.
    Found 1 error in 1 file (checked 1 source file)

通过使用 ``relationship(User, uselist=False)`` 或提供类型（在这种情况下为单个标量``User`` 对象），
可以解决此问题：

    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        user: User = relationship(User)

对于集合，类似的模式适用，如果找不到 ``uselist=True`` 或 :paramref:`_orm.relationship.collection_class`，
则可以使用诸如``List``的注释。可以将类的字符串名称作为注释中支持的pep-484，确保采用
使用该类的 `TYPE_CHECKING block <https://www.python.org/dev/peps/pep-0484/#runtime-or-type-checking>`_ 的类导入：

    from typing import TYPE_CHECKING, List

    from .mymodel import Base

    if TYPE_CHECKING:
        # 如果关系目标位于另一个模块中，
        # 当无法在运行时正常导入该模块时
        from .myaddressmodel import Address


    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column(String)
        addresses: List["Address"] = relationship("Address")

与列一样， :class:`_orm.Mapped` 类也可以显式应用于这些标注：

    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column(String)

        addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        user_id: int = Column(ForeignKey("user.id"))

        user: Mapped[User] = relationship(User, back_populates="addresses")

.. _mypy_declarative_mixins:

使用@declared_attr和Declarative Mixins
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:class:`_orm.declared_attr` 类允许在类级别函数中声明declarative映射属性，当使用:ref:`declarative mixins <orm_mixins_toplevel>`时特别有用。
对于这些函数，函数的返回值应使用``Mapped[]`` 构造函数来注释，或指示函数返回的确切对象类型。
另外，没有映射的“mixin”类（即不扩展 :func:`_orm.declarative_base` 类，也没有使用任何方法如 :meth:`_orm.registry.mapped` 进行映射）
应使用 :func:`_orm.declarative_mixin` 装饰器进行修饰，它为Mypy插件提供了关于特定类用作declarative混合类的提示：

    from sqlalchemy.orm import declarative_mixin, declared_attr


    @declarative_mixin
    class HasUpdatedAt:
        @declared_attr
        def updated_at(cls) -> Column[DateTime]:  # uses Column
            return Column(DateTime)


    @declarative_mixin
    class HasCompany:
        @declared_attr
        def company_id(cls) -> Mapped[int]:  # uses Mapped
            return Column(ForeignKey("company.id"))

        @declared_attr
        def company(cls) -> Mapped["Company"]:
            return relationship("Company")


    class Employee(HasUpdatedAt, HasCompany, Base):
        __tablename__ = "employee"

        id = Column(Integer, primary_key=True)
        name = Column(String)

请注意，像 ``HasCompany.company`` 这样的方法的实际返回类型与其注释之间的不匹配之处。
插件将所有 ``@declared_attr`` 函数转换为简单的注释属性，以避免此种复杂性：

    # Mypy看到的内容
    class HasCompany:
        company_id: Mapped[int]
        company: Mapped["Company"]

结合Dataclasses或其他类型敏感的属性系统
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Python dataclasses 集成的示例在 :ref:`orm_declarative_dataclasses` 中显示了一个问题：
Python dataclasses 需要一个显式的类型，它将使用它构建类，而给定的值在每个分配语句中都是有意义的。
更具体地说，类必须明确声明如下才能被dataclasses接受：

    mapper_registry: registry = registry()


    @mapper_registry.mapped
    @dataclass
    class User:
        __table__ = Table(
            "user",
            mapper_registry.metadata,
            Column("id", Integer, primary_key=True),
            Column("name", String(50)),
            Column("fullname", String(50)),
            Column("nickname", String(12)),
        )
        id: int = field(init=False)
        name: Optional[str] = None
        fullname: Optional[str] = None
        nickname: Optional[str] = None
        addresses: List[Address] = field(default_factory=list)

        __mapper_args__ = {  # type: ignore
            "properties": {"addresses": relationship("Address")}
        }

我们不能对属性“id”，“name”等应用 ``Mapped[]`` 类型，因为它们将被 ``@dataclass`` 装饰器拒绝。
此外，Mypy还有另一个针对dataclasses的插件，可以干扰到我们所做的事情。

上述类将通过Mypy的类型检查而不会产生问题。我们错过的唯一东西是``User`` 上的属性并
 不能用于SQL表达式，例如::

    stmt = select(User.name).where(User.id.in_([1, 2, 3]))

为解决此问题，插件有一个附加功能，即我们可以指定一个额外的属性 ``_mypy_mapped_attrs``，它是一个
包含类级对象或它们的字符串名称的列表。此属性可以在 ``TYPE_CHECKING`` 变量中条件化：

    @mapper_registry.mapped
    @dataclass
    class User:
        __table__ = Table(
            "user",
            mapper_registry.metadata,
            Column("id", Integer, primary_key=True),
            Column("name", String(50)),
            Column("fullname", String(50)),
            Column("nickname", String(12)),
        )
        id: int = field(init=False)
        name: Optional[str] = None
        fullname: Optional[str]
        nickname: Optional[str]
        addresses: List[Address] = field(default_factory=list)

        if TYPE_CHECKING:
            _mypy_mapped_attrs = [id, name, "fullname", "nickname", addresses]

        __mapper_args__ = {  # type: ignore
            "properties": {"addresses": relationship("Address")}
        }

使用以上建议，列在``_mypy_mapped_attrs`` 字段中将被赋予等效的 :class:`_orm.Mapped` 类型信息，
以便当以class-bound上下文使用``User`` 类时，``User`` 类会像SQLAlchemy映射类一样行为。