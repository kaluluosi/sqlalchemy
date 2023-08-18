.. _mypy_toplevel:

Mypy / Pep-484 ORM映射支持
=========================

使用依赖于 SQLAlchemy 的   :ref:`declarative <orm_declarative_mapper_config_toplevel>`  映射就能支持  :pep:` 484`  类型标注以及 MyPy_ 类型检查工具, 这种映射直接引用   :class:`_schema.Column`  对象而非 SQLAlchemy 2.0 引入的   :func:` _orm.mapped_column`  构造。

.. deprecated:: 2.0

    **SQLAlchemy Mypy 插件已被弃用，可能在 SQLAlchemy 2.1 版本甚至更早的版本中移除。我们强烈建议用户尽快迁移到新的映射语法。**

    该插件无法跟随 mypy 不断变化的版本，因此其稳定性未来无法保障。

    现代的 SQLAlchemy 提供了完全符合 pep-484 的映射语法; 迁移详见链接部分。

.. topic:: SQLAlchemy Mypy 插件状态更新

   **更新于2023年7月**

   针对 SQLAlchemy 2.0，Mypy 插件在 SQLAlchemy 1.4 发布时能够正常工作。但是，SQLAlchemy 2.0 提供了一个   :ref:`新的类型系统 <whatsnew_20_orm_declarative_typing>`  用于ORM Declarative模型，该类型系统可以替代 Mypy 插件并且功能更加一致也更强大。注意，该新功能**不在 SQLAlchemy 1.4中提供，只在 SQLAlchemy 2.0 中提供**。

   SQLAlchemy Mypy 插件即使从技术上来说，也从未脱离"alpha"阶段。但是，对于 SQLAlchemy 2.0，该插件目前应被视为已废弃状态，尽管在使用 SQLAlchemy 1.4 时仍需要它以获取全部的 MyPy 支持。

   Mypy 插件本身并不能解决在其他类型工具（如 Pylance/Pyright、Pytype、Pycharm 等）中提供正确类型定义的问题，这些工具不能使用 Mypy 插件。此外，开发、维护和测试 Mypy 插件非常困难，因为 Mypy 插件必须与作为 Mypy 插件内部数据和处理的 Mypy 内部数据结构进行深度集成，而该数据结构本身在 Mypy 项目本身中并不稳定。使用模式与基本模式偏差的代码将面临多种限制。

   因此，对于 Mypy 插件的新回归问题，不太可能进行修复。**在安装 SQLAlchemy 1.4 的情况下，使用 Mypy 插件检测的现有代码将无需进行任何更改即可通过 SQLAlchemy 2.0 中的所有检查，前提是仍然使用该插件。SQLAlchemy 2.0 的 API 与 SQLAlchemy 1.4 的 API 和 Mypy 插件行为完全向后兼容。**

   完全通过 SQLAlchemy 1.4 中使用 Mypy 插件检查的终端用户代码，可能会根据   :ref:`whatsnew_20_orm_declarative_typing`  部分所述的方式逐步迁移至新结构。

   在仅在 SQLAlchemy 2.0 上运行且已完全迁移到新延展结构的代码将按照 pep-484 规范工作，并且可以在IDE和其他打字工具中正常工作，而无需使用插件。


安装
------------

仅适用于 **SQLAlchemy 2.0**：不应安装存根，诸如 sqlalchemy-stubs_ 和 sqlalchemy2-stubs_ 等包应完全卸载。

这里是使用“mypy”额外钩子通过pip安装Mypy的命令：

.. sourcecode:: text

    pip install sqlalchemy[mypy]

插件的配置与 mypy 所有插件一样，在 `为 mypy 配置插件 <https://mypy.readthedocs.io/en/latest/extending_mypy.html#configuring-mypy-to-use-plugins>`_
中描述，可以使用 ``sqlalchemy.ext.mypy.plugin`` 模块名称，例如在 ``setup.cfg`` 中使用::

    [mypy]
    plugins = sqlalchemy.ext.mypy.plugin

.. _sqlalchemy-stubs: https://github.com/dropbox/sqlalchemy-stubs

.. _sqlalchemy2-stubs: https://github.com/sqlalchemy/sqlalchemy2-stubs


插件所做的事情
--------------------

Mypy 插件的主要目的是截获并修改 SQLAlchemy 的静态定义   :ref:`declarative mappings <orm_declarative_mapper_config_toplevel>` ，以便它们与它们   :class:` _orm.Mapper`  对象在进行  :term:`instrumented`  操作后的结构相匹配。这使得类结构和使用该类的代码对 Mypy 工具有意义，否则根据声明式映射当前的功能不会有任何效果。该插件不会与需要为数据类（如 ` dataclasses <https://docs.python.org/3/library/dataclasses.html>`_ ）类动态修改的类似插件类似。

为了涵盖在此发生的主要区域，请考虑 ORM 映射，使用一个典型的 "User" 类示例::

    from sqlalchemy import Column, Integer, String, select
    from sqlalchemy.orm import declarative_base

    #“Base”是一个在 declarative_base() 函数中创建的动态类
    Base = declarative_base()

    
    class User(Base):
        __tablename__ = "user"
        id = Column(Integer, primary_key=True)
        name = Column(String)

    #“some_user”是 User 类的一个实例，基于映射接受“id”和“name” kwargs
    some_user = User(id=5, name="user")

    # 它有一个称为 .name 的字符串属性
    print(f"Username: {some_user.name}")

    # select() 构造使用从 User 类派生的 SQL 表达式

.. code-block:: python
    select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains("s"))

在上面的示例中，Mypy 扩展可以采用以下步骤：

* 解释由   :func:`_orm.declarative_base`  生成的动态 ` `Base`` 类，使得继承自它的类被认为是已映射的。也可以采用类装饰器的方式，详见   :ref:`orm_declarative_decorator` 。

* 对 ORM 映射属性进行类型推断，这些属性是以声明式 "内联" 样式定义的，在上面的示例中是 ``User`` 类的 ``id`` 和 ``name`` 属性。这包括：``User`` 的实例将使用 ``int`` 作为 ``id`` 类型，使用 ``str`` 作为 ``name`` 类型。它也包括当访问 ``User.id`` 和 ``User.name`` 类级属性时（如在上面的 ``select()`` 语句中），它们与 SQL 表达式行为兼容，这是从   :class:`_orm.InstrumentedAttribute`  属性描述符类中继承而来的。

* 对于没有显式构造函数的映射类，应用一个 ``__init__()`` 方法，它接受特定类型的关键字参数，这些参数对所有检测到的映射属性进行定义。

当 Mypy 插件处理上面的文件时，生成的静态类定义和传递给 Mypy 工具的 Python 代码等价于以下代码：

.. code-block:: python
    from sqlalchemy import Column, Integer, String, select
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm.decl_api import DeclarativeMeta


    class Base(metaclass=DeclarativeMeta):
        __abstract__ = True


    class User(Base):
        __tablename__ = 'user'

        id: Mapped[Optional[int]] = Mapped._special_method(
            Column(Integer, primary_key=True)
        )
        name: Mapped[Optional[str]] = Mapped._special_method(Column(String))

        def __init__(self, id: Optional[int] = ..., name: Optional[str] = ...) -> None:
            pass


    some_user = User(id=5, name='user')

    print(f'Username: {some_user.name}')

    select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains('s'))

上面采取的关键步骤包括：

* ``Base`` 类现在以   :class:`_orm.DeclarativeMeta`  类显式定义，而不是动态类。

* ``id`` 和 ``name`` 属性以   :class:`_orm.Mapped`  类为基础定义，该类表示 Python 描述符，具有类级和实例级不同的行为。现在   :class:` _orm.Mapped`  类是   :class:`_orm.InstrumentedAttribute`  类的基类，后者用于所有 ORM 映射属性。

    :class:`_orm.Mapped`  被针对任意 Python 类型定义为泛型类别，这意味着特定的   :class:` _orm.Mapped`  实例与特定的 Python 类型相关联，例如上面的 ``Mapped[Optional[int]]`` 和 ``Mapped[Optional[str]]``。

* 声明式映射属性赋值的右侧被 **移除**，因为这类似于   :class:`_orm.Mapper`  类通常要执行的操作，即它将这些属性替换为特定的   :class:` _orm.InstrumentedAttribute`  实例。原始表达式被移入一个函数调用中，这将允许它在不与表达式左侧冲突的情况下进行类型检查。对于 Mypy，左侧的类型注释足以让属性的行为被理解。

* 添加了 ``User.__init__()`` 方法的类型存根，其中包括正确的关键字和数据类型。

用法
----------

下面的子部分将介绍已经考虑到的 PEP 484 合规性的单个用例。


基于 TypeEngine 的列的内省
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于包括显式数据类型的映射列，当它们被映射为内联属性时，将自动进行类型内省：

.. code-block:: python
    class MyClass(Base):
        # ...

        id = Column(Integer, primary_key=True)
        name = Column('employee_name', String(50), nullable=False)
        other_name = Column(String(50))

上面的 ``id``、``name`` 和 ``other_name`` 的最终类级数据类型将被自动内省为 ``Mapped[Optional[int]]``、``Mapped[Optional[str]]`` 和 ``Mapped[Optional[str]]``。类型默认情况下始终被视为 ``Optional``，即使对于主键和非空列也是如此。原因是虽然数据库列 "id" 和 "name" 不能为 NULL，但 Python 属性 ``id`` 和 ``name`` 很可能是 ``None``，没有显式构造函数：

.. code-block:: python
    >>> m1 = MyClass()
    >>> m1.id
    None

上述列的类型可以被 **显式地** 声明，提供两个优点，即更清晰的自我文档化以及能够控制哪些类型是可选的：

.. code-block:: python
    class MyClass(Base):
        # ...

        id: int = Column(Integer, primary_key=True).. _mypy_mapping_relationships:

映射关系
--------

插件对于使用类型推断来检测关系类型具有有限的支持。
对于无法检测类型的所有情况，它都会发出信息性的错误消息，
而在所有情况下，都可以明确提供适当的类型，
使用   :class:`_orm.Mapped`  类标识它们为映射属性，
或者在内联声明中省略它。

插件还需要确定关系是引用集合还是标量，
为此，它依赖于  :paramref:`_orm.relationship.uselist`  和/或
  :paramref:`_orm.relationship.collection_class`   参数的显式值。
如果这些参数都不存在，则需要明确的类型，
如果   :func:`_orm.relationship`  的目标类型是字符串或可调用对象而不是类，则也需要明确的类型。

.. literalinclude:: /../../examples/orm/relationships.py
   :language: python
   :lines: 11-21

以上映射将会产生以下错误：

.. sourcecode:: pytb

  test3.py:16: error: Argument 1 to "Mapped" has incompatible type "Union[str, Type[User]]"; expected "Type[Any]"
  Found 1 error in 1 file (checked 1 source file)

继续使用   :class:`_orm.Mapped` ，或者在属性上使用常规类型声明来修复错误：

.. literalinclude:: /../../examples/orm/relationships.py
   :language: python
   :lines: 24-26

在这种情况下，我们省略了   :class:`_orm.Mapped`  类，而是使用了 int 和 str 作为映射示例。.. sourcecode:: text

    test3.py:22: 错误：[SQLAlchemy Mypy插件]无法推断ORM映射表达式分配给关系“user”属性的标量或集合，如果关系（）中既没有“uselist”也没有“collection_class”参数，则请在左侧指定类型注释。在1个文件（检查了1个源文件）中找到1个错误

可以通过使用 ``relationship(User, uselist=False)`` 或者在这种情况下提供类型-标量 User 参数来解决错误：

    class Address(Base):
        __tablename__ = "address"

        id = Column(Integer, primary_key=True)
        user_id: int = Column(ForeignKey("user.id"))

        user: User = relationship(User)

对于集合，类似的模式适用，其中在缺少 ``uselist=True`` 或  :paramref:`_orm.relationship.collection_class`  的情况下，可以使用集合注释，例如 ` `List``。在注释中还完全适当地使用类的字符串名称，这是 pep-484 支持的，确保在需要时在 `TYPE_CHECKING 块中导入类`_::

    from typing import TYPE_CHECKING, List

    from .mymodel import Base

    if TYPE_CHECKING:
        # 如果关系的目标在另一个模块中，无法在运行时正常导入
        from .myaddressmodel import Address


    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column(String)
        addresses: List["Address"] = relationship("Address")

与列一样，  :class:`_orm.Mapped`  类也可以显式应用：

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

使用 @declared_attr 和 Declarative Mixins
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_orm.declared_attr`  类允许在类级函数中声明声明性映射属性，并且在使用   :ref:` declarative mixins <orm_mixins_toplevel>` `Mapped[]``结构进行注释，或者通过指示函数返回的确切对象类型来进行注释。此外，还应该使用   :func:`_orm.declarative_mixin`  装饰未映射的“mixin”类（即不扩展   :func:` _orm.declarative_base`  类，也没有使用  :meth:`_orm.registry.mapped`  等方法映射的类），该装饰符为 Mypy 提供了提示，表明特定类意图作为声明性 mixin:

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

请注意，方法如 ``HasCompany.company`` 的实际返回类型与所注释的类型之间的不匹配。Mypy 插件将所有 ``@declared_attr`` 函数转换为简单的注释属性，以避免这种复杂性:

    # Mypy看到的
    class HasCompany:
        company_id: Mapped[int]
        company: Mapped["Company"]

与数据类或其他类型敏感属性系统结合
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Python 数据类集成的例子   :ref:`orm_declarative_dataclasses`  呈现了一个问题；Python 数据类期望一个明确的类型，它将使用该类型构建类，并且在给出的每个赋值语句中给出的值是重要的。也就是说，必须准确地声明像下面的类才能被数据类接受：

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

我们不能将 ``Mapped[]`` 类型应用于属性 ``id``，``name``，等等。因为它们将会被 `@dataclass` 装饰器拒绝。此外，
 Mypy 已经有另一个专门用于 dataclasses 的插件，它可能也会干扰到我们现在所做的事情。

以上代码实际上能够通过 Mypy 的类型检查而没有问题；我们唯一缺少的是让 ``User`` 类
的属性可以被用在 SQL 表达式里的能力，例如：：

    stmt = select(User.name).where(User.id.in_([1, 2, 3]))

为了提供一个解决方案，Mypy 的插件有一个额外的特性，我们可以指定额外的属性 ``_mypy_mapped_attrs``，即
将一个包含类级别对象或它们字符串名称的列表。这个属性可以根据 ``TYPE_CHECKING`` 变量条件进行定义：

.. code-block:: python

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

使用上面的代码，列在 ``_mypy_mapped_attrs`` 中的属性将会被应用于   :class:`_orm.Mapped`  类型信息中，
这样，当 ``User`` 类被用在类绑定的上下文中时，它将会像一个 SQLAlchemy 映射类一样运作。

.. _Mypy: https://mypy.readthedocs.io/