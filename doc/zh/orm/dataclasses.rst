.. _orm_dataclasses_toplevel:

======================================
与dataclasses和attrs集成
======================================

SQLAlchemy自2.0版本开始提供"原生dataclass"集成，使用单个混合类型或装饰器将   :ref:`带注释的声明表 <orm_declarative_mapped_column>`  映射转换为Python dataclass_就可将映射类转换为数据类。

.. versionadded:: 2.0 使用ORM Declarative类进行集成数据类的创建

还有其他的模式可以使用，例如映射现有的dataclasses，或映射由attrs_第三方集成库创建的类。

.. _orm_declarative_native_dataclasses:

声明数据类映射
-------------------------------

SQLAlchemy   :ref:`Annotated Declarative Table <orm_declarative_mapped_column>`  映射可以通过添加另一个混合类型类或装饰器指令来增强，这会在映射完成后增加一个步骤，将映射类**原地**转换为Python dataclass_, 然后应用特定于ORM的 :term:` instrumentation` 以完成映射过程。这当中最显著的行为变化就是生成一个具有细粒度控制的 `__init __（）` 方法, 可以在没有默认值的情况下接受有位置和关键字参数，同时生成像 `__repr __（）` 和 `__eq __（）` 这样的方法。

从  :pep:`484`  类型的角度讲，数据类特定的行为被识别为包含数据类转换的工具，特别是通过  :pep:` 681`  "Dataclass Transforms" 来利用，这可以使类型工具将该类视为已显式使用 `@dataclasses.dataclass` 装饰器装饰。

.. 注意:: 截至 **2023年4月4日**，typing工具对  :pep:`681`  的支持有限，目前已知 Pyright_ 和 Mypy_ 版本 1.2 支持。请注意，Mypy 1.1.1 引入  :pep:` 681`  支持，但没有正确适应 Python 描述符，这会在使用 SQLAlchemy 的ORM映射方案时导致错误。

   .. seealso::
   
      https://peps.python.org/pep-0681/#the-dataclass-transform-decorator - 关于类似SQLAlchemy的库如何启用  :pep:`681`  支持的背景
   
将数据转换类添加到任何声明式类中，可以将   :class:`_orm.MappedAsDataclass`  混合类型添加到   :class:` _orm.DeclarativeBase`  类层次结构中或通过使用   :meth:`_orm.registry.mapped_as_dataclass`  类装饰器的装饰器映射。

可将   :class:`_orm.MappedAsDataclass`  混合类型应用于声明式 "Base" 类或任何超类，例如下面的示例 ::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Base(MappedAsDataclass, DeclarativeBase):
        """该子类将被转换为dataclasses"""


    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

或者可以直接应用于从声明式base派生的类：

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Base(DeclarativeBase):
        pass


    class User(MappedAsDataclass, Base):
        """User类将被转换为数据类"""

        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

使用装饰器形式时，仅支持  :meth:`_orm.registry.mapped_as_dataclass`  装饰器::

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry


    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

类级特征配置
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

支持数据类特性部分。目前**支持**的有“init”、“repr”、“eq”、“order”和“unsafe_hash”特性，Python 3.10+上支持“match_args”和“kw_only”。目前**不支持**的有“frozen”和“slots”特性。

在使用   :class:`_orm.MappedAsDataclass`  的混合类形式时，类配置参数作为类级参数传递::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Base(DeclarativeBase):
        pass


    class User(MappedAsDataclass, Base, repr=False, unsafe_hash=True):
        """User类将被转换为DataClass"""

        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

使用  :meth:`_orm.registry.mapped_as_dataclass`  装饰器形式时，类配置参数直接传递给装饰器::

    from sqlalchemy.orm import registry
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    reg = registry()


    @reg.mapped_as_dataclass(unsafe_hash=True)
    class User:
        """User类将被转换为DataClass"""

        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

有关数据类类选项的背景，请参见数据类_文档中的 `@dataclasses.dataclass <https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass>`_。

属性配置
^^^^^^^^^^^^^^^^^^^^^^^^^

SQLAlchemy原生数据类与普通数据类的区别在于所有要映射的属性都是使用   :class:`_orm.Mapped`  通用注释容器描述的。映射遵循与   :ref:` orm_declarative_table`  中记录的相同形式，支持   :func:`_orm.mapped_column`  和   :class:` _orm.Mapped`  的所有功能。

此外，ORM属性配置构造，包括   :func:`_orm.mapped_column` 、  :func:` _orm.relationship`  和   :func:`_orm.composite`  支持 **每个属性字段选项**，包括 ` `init``、``default``、``default_factory`` 和 ``repr``。这些参数的名称被固定为  :pep:`681`  中指定的名称。功能与数据类等效：

* ``init``，与  :paramref:`_orm.mapped_column.init <sqlalchemy.orm.mapped_column>` ，  :paramref:` _orm.relationship.init <sqlalchemy.orm.relationship>`  相同，如果为 False，则表示该字段不应作为``__init __()`` 方法的一部分。

* ``default``，如  :paramref:`_orm.mapped_column.default <sqlalchemy.orm.mapped_column>` ，  :paramref:` _orm.relationship.default <sqlalchemy.orm.relationship>` ，指示字段的默认值，该值在``__init__（）``方法中按关键字参数给出。

* ``default_factory``，如  :paramref:`_orm.mapped_column.default_factory <sqlalchemy.orm.mapped_column>` ，  :paramref:` _orm.relationship.default_factory <sqlalchemy.orm.relationship>` ，表示可调用函数，该函数会在未将参数传递明确传递给``__init__（）``方法的情况下生成新的默认值。

* ``repr``默认为True，表示该字段应作为生成的``__repr __()`` 方法的一部分。

与数据类的另一个主要区别是，将属性的默认值 **必须**使用ORM构造中的``default``参数进行配置，例如``mapped_column（default = None）``。不支持类似数据类的语法，该语法接受简单的Python值作为默认值，而不使用 `@dataclases.field（）``。

通过使用   :func:`_orm.mapped_column` ，下面的映射将生成一个` `__init__()`` 方法，该方法仅接受参数``name`` 和``fullname``，其中``name``是必需的，可以按位置传递，``fullname``是可选的。我们预计由数据库生成的 ``id`` 字段根本不在构造函数中:


    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]
        fullname: Mapped[str] = mapped_column(default=None)


    # 'fullname'是可选的关键字参数
    u1 = User("name")


列默认值
~~~~~~~~~~~~~~~

为了适应 ``default`` 参数与现有  :paramref:`_schema.Column.default`  参数的重叠，   :func:` _orm.mapped_column`  构造将这两个名称区分开来，通过添加一个新参数  :paramref:`_orm.mapped_column.insert_default`  直接将其填充到   :class:` _schema.Column`  的  :paramref:`_schema.Column.default`  参数中，而不考虑在  :paramref:` _orm.mapped_column.default`  上设置了什么，后者始终用于数据类配置。例如，要将默认值设置为 ``func.utc_timestamp()`` SQL 函数的 datetime 列，但在构造函数中该参数是可选的::

    from datetime import datetime

    from sqlalchemy import func
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        created_at: Mapped[datetime] = mapped_column(
            insert_default=func.utc_timestamp(), default=None
        )

上述映射将根据默认值生成新的``User``对象的 ``INSERT`` 语句，而在 ``created_at`` 参数未显式传递的情况下，会执行如下操作：

.. sourcecode:: pycon+sql

    >>> with Session(e) as session:
    ...     session.add(User())
    ...     session.commit()
    {execsql} BEGIN（隐式）
    INSERT INTO user_account（created_at）VALUES（utc_timestamp（））
    [generated in 0.00010s] ()
    COMMIT

与注释
~~~~~~~~~~~~~~~~~~~~~~~~

在   :ref:`orm_declarative_mapped_column_pep593`  中介绍的方法说明如何使用  :pep:` 593`  中的“注释”对象来打包整个   :func:`_orm.mapped_column`  构造以供重用。此功能与数据类特性一起使用。 但是，该特性的一个方面需要处理以下情况：当使用类型工具时，额外的内部更改是，  :pep:` 681`  特定参数 `init`、`default`、`repr` 和 `default_factory` 必须在右侧中打包到显式的   :func:`_orm.mapped_column`  构造中，以便类型工具可以正确解释属性。例如，下面的方式将完美地在运行时工作，但是当未在其中看到` `init=False`` 参数时，类型工具将认为 ``User()`` 构造是无效的：

    from typing import Annotated

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    # typing工具将忽略init = False
    intpk = Annotated[int， mapped_column(init=False， primary_key=True)]

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"
        id: Mapped[intpk]


    # typing错误：缺少参数“id”
    u1 = User()

相反，需要在右边也使用   :func:`_orm.mapped_column` ，并使用此函数显式设置  :paramref:` _orm.mapped_column.init`  的值；其他参数可以保留在 ``Annotated`` 构造内::

    from typing import Annotated

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    intpk = Annotated[int， mapped_column(primary_key=True)]

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        # init=False和其他pep-681参数必须是inline的
        id: Mapped[intpk] = mapped_column(init=False)


    u1 = User()

.. _orm_declarative_dc_mixins:

使用基类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

任何包含   :class:`_orm.Mapped`  属性的混合类型或基类在   :class:` _orm.MappedAsDataclass`  映射类中使用时，它们本身必须是   :class:`_orm.MappedAsDataclass`  层次结构的一部分，比如在下面使用混合类型的示例::

    class Mixin(MappedAsDataclass):
        create_user: Mapped[int] = mapped_column()
        update_user: Optional[int] = mapped_column(default=None, init=False)


    class Base(DeclarativeBase, MappedAsDataclass):
        pass


    class User(Base, Mixin):
        __tablename__ = "sys_user"

        uid: Mapped[str] = mapped_column(
            String(50), init=False, default_factory=uuid4, primary_key=True
        )
        username: Mapped[str] = mapped_column()
        email: Mapped[str] = mapped_column()

支持  :pep:`681`  的 Python 类型检查器否则不认为数据类产生的类属性与数据类相同。

.. deprecated:: 2.0.8 在   :class:`_orm.MappedAsDataclass`  或
    :meth:`_orm.registry.mapped_as_dataclass`  层次结构内使用混合类型和抽象基类，如果它们本身不是数据类，则不支持这些字段作为属于数据类的  :pep:` 681`  .针对此情况发出警告，该警告后来将变成错误。

   .. seealso::

         :ref:`error_dcmx`  - 关于理由

关系配置
^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_orm.Mapped`  注释结合   :func:` _orm.relationship`  与   :ref:`relationship_patterns`  中描述的使用方法相同。在将集合类作为可选关键字参数指定的情况下，必须传递  :paramref:` _orm.relationship.default_factory`  参数，并且它必须引用要使用的集合类。如果默认值将是 ``None``，则可以使用  :paramref:`_orm.relationship.default`  来定义对多和标量对象引用::

    from typing import List

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry
    from sqlalchemy.orm import relationship

    reg = registry()


    @reg.mapped_as_dataclass
    class Parent:
        __tablename__ = "parent"
        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(
            default_factory=list, back_populates="parent"
        )


    @reg.mapped_as_dataclass
    class Child:
        __tablename__ = "child"
        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
        parent: Mapped["Parent"] = relationship(default=None)

上述映射将为 ``Parent.children`` 栏位生成一个空列表，当构造一个不传递``children`` 的新 ``Parent() ``对象时，而类似的，对于 ``Child()`` 对象，当构造一个未传递 ``parent`` 的新对象时，会生成 ``None`` 值的 ``Child.parent``。

注意,  :paramref:`_orm.relationship.default_factory`  可以从   :func:` _orm.relationship`  派生的给定集合类自动推导出来，但这会破坏与Data class的兼容性，因为  :paramref:`_orm.relationship.default_factory`  或  :paramref:` _orm.relationship.default`  的存在决定了将其呈现为 ``__init__()`` 方法时，该参数是必需还是可选。

使用   :ref:`与 _的非映射数据类字段 <orm_declarative_native_dataclasses_non_mapped_fields>`  时，将使用未映射的字段作为实例级状态的一部分，但不会被ORM保存。任何不使用   :class:` .Mapped`  的字段将被映射过程忽略。在下面的示例中，字段“ctrl_one”和“ctrl_two”将是对象的实例级状态，但不会被ORM保存：


    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    reg = registry()


    @reg.mapped_as_dataclass
    class Data:
        __tablename__ = "data"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        status: Mapped[str]

        ctrl_one: Optional[str] = None
        ctrl_two: Optional[str] = None

可以像这样创建Data的实例：

    d1 = Data(status="s1", ctrl_one="ctrl1", ctrl_two="ctrl2")

更真实的例子可能是将Dataclasses的 ``InitVar`` 特性与 ``__post_init __（）`` 特性一起使用，以接收可用于构成持续数据的仅初始化字段。在下面的示例中，声明了使用 ``id``， ``name`` 和 ``password_hash`` 作为映射属性的 ``User`` 类，但是使用仅初始化 ``password``和 ``repeat_password``字段来表示用户创建过程（注意：要运行此示例，将 ``your_crypt_function_here（）`` 函数替换为类似于`bcrypt <https://pypi.org/project/bcrypt/>`_或 `argon2-cffi <https://pypi.org/project/argon2-cffi/>`_的第三方加密函数）:

    from dataclasses import InitVar
    from typing import Optional

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

        password: InitVar[str]
        repeat_password: InitVar[str]

        password_hash: Mapped[str] = mapped_column(init=False, nullable=False)

        def __post_init__(self, password: str, repeat_password: str):
            if password != repeat_password:
                raise ValueError("passwords do not match")

            self.password_hash = your_crypt_function_here(password)

上述对象使用参数 ``password`` 和 ``repeat_password`` 创建，这些参数会被提前使用，以便可以从模拟器中返回它们的值并从中生成 ``password_hash`` 参数：

    >>> u1 = User(name="some_user", password="xyz", repeat_password="xyz")
    >>> u1.password_hash
    '$6$9ppc... (example crypted string....)'

.. versionchanged:: 2.0.0rc1 当使用  :meth:`_orm.registry.mapped_as_dataclass`  或
    :class:`.MappedAsDataclass`  时，不使用   :class:` .Mapped`  注释的字段可以被包含在内，这将被视为结果数据类的一部分，但不会被映射，无需也没有始终需要注释合法性的额外 `__allow_unmapped__` 类属性。以前的2.0 beta版本将要求显式存在该属性，即使该属性的目的仅是允许旧版ORM类型映射继续运行。

.. _dataclasses_pydantic:

与Pydantic等替代数据类提供程序集成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. warning::

   Pydantic 1.x版本的数据类层不完全兼容于SQLAlchemy的类仪器化，许多特性，如相关集合可能不正确地工作。

   为了与Pydantic兼容，请考虑使用基于SQLAlchemy ORM构建的使用Pydantic的 `SQLModel <https://sqlmodel.tiangolo.com>`_ ORM，其中包括特定实现细节，这些细节 **显式解决了** 这些不兼容性。

SQLAlchemy的   :class:`_orm.MappedAsDataclass`  类 和  :meth:` _orm.registry.mapped_as_dataclass`  方法直接调用 Python标准库的 `dataclasses.dataclass <https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass>`_类装饰器，在将类声明过程应用于类之后。可以使用 ``dataclass_callable`` 参数直接为其他数据类提供程序换掉此函数调用，例如Pydantic，作为类关键字参数及作为  :meth:`_orm.registry.mapped_as_dataclass`  方法的参数::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass
    from sqlalchemy.orm import registry


    class Base(
        MappedAsDataclass,
        DeclarativeBase,
        dataclass_callable=pydantic.dataclasses.dataclass,
    ):
        pass


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]

上述 ``User`` 类将被处理成一个dataclass，使用Pydantic的 ``pydantic.dataclasses.dataclasses`` 调用。这个过程适用于映射类以及直接扩展   :class:`_orm.MappedAsDataclass`  或直接应用到  :meth:` _orm.registry.mapped_as_dataclass`  的混合类。

.. versionadded:: 2.0.4 添加了   :class:`_orm.MappedAsDataclass`  和  :meth:` _orm.registry.mapped_as_dataclass`  的 ``dataclass_callable`` 类和方法参数，并调整了一些数据类的内部以适应更严格的数据类函数，例如 Pydantic 数据类函数。

.. _orm_declarative_dataclasses:

将ORM映射应用于现有数据类（旧数据类用法）
---------------------------------------------------------------------

.. legacy::

   这些方法不再适用于2.0系列的新特性   :ref:`orm_declarative_native_dataclasses` 。这种新方法基于首次添加到版本1.4中的数据类支持，其在本节中介绍。

要将映射应用于数据类，不能直接使用 SQLAlchemy 的 "inline" 声明式指令;需要通过以下三种技术之一分配 ORM 指令：

* 使用“自声明式表”；表格/列与映射被定义为分配给类的``__ table__`` 属性的   :class:`_schema.Table`  对象；关系是在 ` `__mapper_args__`` 字典内定义的.使用  :meth:`_orm.registry.mapped`  装饰器将类映射.下面的示例显示了：   :ref:` orm_declarative_dataclasses_imperative_table` .

* 使用完全的“声明式”类型，如   :class:`_schema.Column` 、  :func:` _orm.relationship`  被添加到 ``dataclasses.field()`` 构造函数的 ``.metadata`` 字典中，在声明式过程中进行消耗。再次使用  :meth:`_orm.registry.mapped`  装饰器将映射类.请参考下面的示例解释：  :ref:` orm_declarative_dataclasses_declarative_table` .

* 非声明式映射可以使用  :meth:`_orm.registry.map_imperatively`  来应用到现有的数据类中，以完全相同的方式生成映射如   :ref:` orm_imperative_mapping`  中所述。这是   :ref:`orm_imperative_dataclasses`  中所示的。

SQLAlchemy将映射应用于数据类的一般过程与普通类相同，但还包括SQLAlchemy检测到作为数据类声明过程的一部分的类级属性，并在运行时用通常的SQLAlchemy ORM映射属性替换它们，如dataclasses生成的 ``__init __`` 方法和 ``__eq__()``, ``__repr__()`` 等方法将保持不变。

.. _orm_declarative_dataclasses_imperative_table:

使用“自声明式表”​​映射现有数据类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用到   :ref:`orm_imperative_table_configuration`  的方式举例使用 ` `@dataclass``.显式地构造完整的  :class:`_schema.Table`  对象，并将其分配给 ` `__table__`` 属性。使用以下数据类语法定义实例字段。其他   :class:`.MapperProperty`  的定义，例如   :func:` .relationship` ，将被放置在 ``properties`` 键下的   :ref:`__mapper_args__ <orm_declarative_mapper_options>`  类级字典中，对应于  :paramref:` _orm.Mapper.properties`  参数的方式::

    from __future__ import annotations

    from dataclasses import dataclass, field
    from typing import List, Optional

    from sqlalchemy import Column, ForeignKey, Integer, String, Table
    from sqlalchemy.orm import registry, relationship

    mapper_registry = registry()


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
            "properties": {
                "addresses": relationship("Address"),
            }
        }


    @mapper_registry.mapped
    @dataclass
    class Address:
        __table__ = Table(
            "address",
            mapper_registry.metadata,
            Column("id", Integer, primary_key=True),
            Column("user_id", Integer, ForeignKey("user.id")),
            Column("email_address", String(50)),
        )
        id: int = field(init=False)
        user_id: int = field(init=False)
        email_address: Optional[str] = None

在上面的示例中，``User.id``、``Address.id`` 和 ``Address.user_id`` 属性定义为 ``field(init = False)``。这意味着这些参数不会被添加到 ``__init__()`` 方法中，但是   :class:`.Session`  在进行 flush（从自动增量或其他默认值生成器中获取其值）时仍然可以将其设置。要允许它们明确在构造函数中指定，请改为将它们设置为默认值 ` `None``。

对于   :func:`_orm.relationship`  单独声明一个关系必须直接在  :paramref:` _orm.Mapper.properties`  字典中指定，该字典本身在 ``__mapper_args__`` 字典内指定，以便将其传递到   :class:`_orm.Mapper`  的构造函数。与此方法的替代方法相比，下面的方法更简洁，其中使用 ` `metadata`` 属性上的 SQLAlchemy 特定映射信息来指示   :func:`_orm.relationship`  关联是使用以下方式：


    from __future__ import annotations

    from dataclasses import dataclass, field
    from typing import List

    from sqlalchemy import Column, ForeignKey, Integer, String
    from sqlalchemy.orm import registry, relationship

    mapper_registry = registry()


    @mapper_registry.mapped
    @dataclass
    class User:
        __tablename__ = "user"

        __sa_dataclass_metadata_key__ = "sa"
        id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
        name: str = field(default=None, metadata={"sa": Column(String(50))})
        fullname: str = field(default=None, metadata={"sa": Column(String(50))})
        nickname: str = field(default=None, metadata={"sa": Column(String(12))})

addresses: List[Address] = field(
            default_factory=list, metadata={"sa": relationship("Address")}
        )


    @mapper_registry.mapped
    @dataclass
    class Address:
        __tablename__ = "address"
        __sa_dataclass_metadata_key__ = "sa"
        id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
        user_id: int = field(init=False, metadata={"sa": Column(ForeignKey("user.id"))})
        email_address: str = field(default=None, metadata={"sa": Column(String(50))})

.. _orm_declarative_dataclasses_mixin:

使用声明性 mixins 与预先存在的 dataclasses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在   :ref:`orm_mixins_toplevel`  章节中，我们介绍了声明性 Mixin 类。
声明性 mixins 的一种要求是某些难以复制的结构必须以可调用对象的方式给出，
使用   :class:`_orm.declared_attr`  装饰器，例如   :ref:` orm_declarative_mixins_relationships`  中的示例：

::

    class RefTargetMixin:
        @declared_attr
        def target_id(cls):
            return Column("target_id", ForeignKey("target.id"))

        @declared_attr
        def target(cls):
            return relationship("Target")

可以使用 lambda 表示 SQLAlchemy 构造函数，在 dataclasses 中的 ``field()`` 中支持使用
此形式。使用   :func:`_orm.declared_attr`  包含 lambda 是可选的。
如果我们要创建一个类似上面 ``User`` 类的类，其中 ORM 字段来自于一个自身就是 dataclass 的
mixin 的情况，形式将是：

::

    @dataclass
    class UserMixin:
        __tablename__ = "user"

        __sa_dataclass_metadata_key__ = "sa"

        id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})

        addresses: List[Address] = field(
            default_factory=list, metadata={"sa": lambda: relationship("Address")}
        )


    @dataclass
    class AddressMixin:
        __tablename__ = "address"
        __sa_dataclass_metadata_key__ = "sa"
        id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
        user_id: int = field(
            init=False, metadata={"sa": lambda: Column(ForeignKey("user.id"))}
        )
        email_address: str = field(default=None, metadata={"sa": Column(String(50))})


    @mapper_registry.mapped
    class User(UserMixin):
        pass


    @mapper_registry.mapped
    class Address(AddressMixin):
        pass

.. versionadded:: 1.4.2  Added support for "declared attr" style mixin attributes,
   namely   :func:`_orm.relationship`  constructs as well as   :class:` _schema.Column` 
   objects with foreign key declarations, to be used within "Dataclasses
   with Declarative Table" style mappings.



.. _orm_imperative_dataclasses:

使用命令式映射映射预先存在的 dataclasses
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

正如之前所述，在使用 ``@dataclass`` 装饰器将类设置为 dataclass 后，我们可以使用
  :meth:`_orm.registry.mapped`   装饰器将 ORM 映射应用于该类。我们还可以使用
  :meth:`_orm.registry.map_imperatively`   方法将该类作为命令式映射来使用，
以便我们可以将所有   :class:`_schema.Table`  和   :class:` _orm.Mapper`  配置以命令式的方式传递给函数，
而不是在类本身上定义它们作为类变量::

    from __future__ import annotations

    from dataclasses import dataclass
    from dataclasses import field
    from typing import List

    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import MetaData
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy.orm import registry
    from sqlalchemy.orm import relationship

    mapper_registry = registry()


    @dataclass
    class User:
        id: int = field(init=False)
        name: str = None
        fullname: str = None
        nickname: str = None
        addresses: List[Address] = field(default_factory=list)


    @dataclass
    class Address:
        id: int = field(init=False)
        user_id: int = field(init=False)
        email_address: str = None


    metadata_obj = MetaData()

    user = Table(
        "user",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )

    address = Table(
        "address",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )

    mapper_registry.map_imperatively(
        User,
        user,
        properties={
            "addresses": relationship(Address, backref="user", order_by=address.c.id),
        },
    )

    mapper_registry.map_imperatively(Address, address)

.. _orm_declarative_attrs_imperative_table:

将 ORM 映射应用于现有的 attrs 类
-------------------------------------------------

attrs_ 库是一个流行的第三方库，提供类似于 dataclasses 的功能，还提供了许多在普通 dataclass 中
无法找到的其他特性。

使用 attrs_ 扩充的类使用 ``@define`` 装饰器。这个装饰器启动一个过程，扫描类寻找定义
类行为的属性，然后使用这些属性来生成方法、文档和注释。

SQLAlchemy ORM 支持使用声明性命令式数据类或命令式映射将 attrs_ 类映射。这两种样式的通用形式
完全等同于用于 dataclasses 的   :ref:`orm_declarative_dataclasses_declarative_table`  和
  :ref:`orm_declarative_dataclasses_imperative_table`  映射形式，其中 dataclass 或 attrs 中使用的内联属性指令未更改，
并且在运行时应用了 SQLAlchemy 的面向表的工具。

attrs_ 的默认 ``@define`` 装饰器将带注释的类替换为一个基于 __slots__ 的新类，这是不受支持的。
使用旧样式注释 ``@attr.s`` 或使用 ``define(slots=False)``，此类将不被替换。此外，attrs 在装饰器运行后会删除其本身的类绑定属性，
以便 SQLAlchemy 的映射过程接管这些属性。``@attr.s`` 装饰器和 ``@define(slots=False)`` 装饰器都与 SQLAlchemy 兼容。

使用声明性“命令式表”
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在“声明性表命令式”样式中，使用声明性类内联声明   :class:`_schema.Table`  对象。类首先应用 ` `@define`` 装饰器，
然后应用  :meth:`_orm.registry.mapped`  装饰器::

    from __future__ import annotations

    from typing import List
    from typing import Optional

    from attrs import define
    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import MetaData
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import registry
    from sqlalchemy.orm import relationship

    mapper_registry = registry()


    @mapper_registry.mapped
    @define(slots=False)
    class User:
        __table__ = Table(
            "user",
            mapper_registry.metadata,
            Column("id", Integer, primary_key=True),
            Column("name", String(50)),
            Column("FullName", String(50), key="fullname"),
            Column("nickname", String(12)),
        )
        id: Mapped[int]
        name: Mapped[str]
        fullname: Mapped[str]
        nickname: Mapped[str]
        addresses: Mapped[List[Address]]

        __mapper_args__ = {  # type: ignore
            "properties": {
                "addresses": relationship("Address"),
            }
        }


    @mapper_registry.mapped
    @define(slots=False)
    class Address:
        __table__ = Table(
            "address",
            mapper_registry.metadata,
            Column("id", Integer, primary_key=True),
            Column("user_id", Integer, ForeignKey("user.id")),
            Column("email_address", String(50)),
        )
        id: Mapped[int]
        user_id: Mapped[int]
        email_address: Mapped[Optional[str]]

.. note:: 激活 attrs 的 ``slots=True`` 选项（在映射类上启用 ``__slots__`` ）
   无法在没有完全实现替代属性工具（参见   :ref:`examples_instrumentation` ）的情况下与 SQLAlchemy 映射一起使用，
   映射类通常依赖于对 ``__dict__`` 的直接访问以进行状态存储。出现此选项时行为是未定义的。



使用命令式映射映射 attrs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

与 dataclasses 一样，我们也可以使用  :meth:`_orm.registry.map_imperatively`  将现有的 attrs 类定义为 ORM 映射：

::

    from __future__ import annotations

    from typing import List

    from attrs import define
    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import MetaData
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy.orm import registry
    from sqlalchemy.orm import relationship

    mapper_registry = registry()


    @define(slots=False)
    class User:
        id: int
        name: str
        fullname: str
        nickname: str
        addresses: List[Address]


    @define(slots=False)
    class Address:
        id: int
        user_id: int
        email_address: Optional[str]


    metadata_obj = MetaData()

    user = Table(
        "user",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )

    address = Table(
        "address",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )

    mapper_registry.map_imperatively(
        User,
        user,
        properties={
            "addresses": relationship(Address, backref="user", order_by=address.c.id),
        },
    )

    mapper_registry.map_imperatively(Address, address)

上述形式等价于之前使用“声明性命令式表”的示例。


.. _dataclass: https://docs.python.org/zh-cn/3/library/dataclasses.html
.. _dataclasses: https://docs.python.org/zh-cn/3/library/dataclasses.html
.. _attrs: https://pypi.org/project/attrs/
.. _mypy: https://mypy.readthedocs.io/en/stable/
.. _pyright: https://github.com/microsoft/pyright