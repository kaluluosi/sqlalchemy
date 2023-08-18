.. _orm_dataclasses_toplevel:

======================================
与dataclasses和attrs集成
======================================

从SQLAlchemy 2.0版本开始，可以通过向映射类添加单个mixin或装饰器来将:ref:`带注释的声明性表<orm_declarative_mapped_column>`映射转换为Python dataclass_ ，从而实现“原生数据类”集成。

.. versionadded:: 2.0 集成ORM声明类的数据类创建

还有一些可用的模式，可以用于映射现有的数据类，以及映射由attrs_第三方集成库进行装饰的类。

.. _orm_declarative_native_dataclasses:

声明性数据类映射
------------------------------

SQLAlchemy :ref:`带注释的声明性表<orm_declarative_mapped_column>`映射可以使用一个额外的mixin类或装饰器指令进行增强，增强后，映射过程完成后会将映射的类**就地**转换为Python dataclass_ ，然后才应用ORM-specific :term:`instrumentation`到class上。最显著的行为补充是生成一个具有对位置和关键字参数的细粒度控制的``__init__()``方法，具有或没有默认值，以及生成如``__repr__()``或``__eq__()``之类的方法。

从:pep:`484` typing的角度来看，该类被认为具有Dataclass-specific的行为，尤其是通过利用:pep:`681`，Dataclass变换，允许使用typing工具将该类视为明确使用``@dataclasses.dataclass``装饰器装饰的类。

.. note::  从**2023年4月4日**起，typing工具对:pep:`681`的支持受到限制，并且目前已知Pyright_以及Mypy_作为**版本1.2**已支持。请注意，Mypy 1.1.1引入了:pep:`681`支持，但未正确适应Python描述符，这将导致在使用SQLAlhcemy的ORM映射方案时出现错误。

   .. seealso::

      https://peps.python.org/pep-0681/#the-dataclass-transform-decorator - 有关像SQLAlchemy这样的库如何启用:pep:`681`支持的背景说明

可以通过向任何Declarative类添加Dataclass conversion，具体方法为向:class:`_orm.MappedAsDataclass` mixin添加:class:`_orm.DeclarativeBase`类继承级别，或对于装饰器映射，使用``_orm.registry.mapped_as_dataclass``装饰器。

:class:`_orm.MappedAsDataclass` mixin可以应用于Declarative ``Base``类或任何超类，如下例所示::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Base(MappedAsDataclass, DeclarativeBase):
        """子类将转换为dataclasses"""


    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

或者可直接应用于从Declarative base扩展的类::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Base(DeclarativeBase):
        pass


    class User(MappedAsDataclass, Base):
        """User class will be converted to a dataclass"""

        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

在使用装饰器形式时，只支持:meth:`_orm.registry.mapped_as_dataclass`装饰器::

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry


    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

类级特性配置
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

dataclasses特性的支持是部分的。 :一个:`被支持`的是``init``，``repr``，``eq``，``order``和``unsafe_hash``特性，``match_args``和``kw_only``支持Python 3.10+。 :一个:`无法支持`的是``frozen``和``slots``特性。

当使用:meth:`_orm.MappedAsDataclass`的mixin class形式进行Declarative类配置时，类配置参数作为类级别参数传递::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import MappedAsDataclass


    class Base(DeclarativeBase):
        pass


    class User(MappedAsDataclass, Base, repr=False, unsafe_hash=True):
        """User class将转换为dataclass"""

        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

在使用装饰器形式时，类配置参数直接传递给装饰器::

    from sqlalchemy.orm import registry
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    reg = registry()


    @reg.mapped_as_dataclass(unsafe_hash=True)
    class User:
        """User class将转换为dataclass"""

        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(init=False, primary_key=True)
        name: Mapped[str]

关键字配置
^^^^^^^^^^^^^^^^^^^^^^^^^

支持dataclasses特性是部分的。 :一个:`被支持`的是``init``，``repr``，``eq``，``order``和``unsafe_hash``特性，``match_args``和``kw_only``支持Python 3.10+。 :一个:`无法支持`的是``frozen``和``slots``特性。

当使用Declarative with Imperative Table形式的mixin class :class:`_orm.MappedAsDataclass`进行Declarative类配置时，类配置参数作为类级别参数传递::

当使用装饰器形式时，类配置参数直接传递给装饰器::

属性配置
^^^^^^^^^^^^^^^^^^^^^^

SQLAlchemy的本机dataclasses与正常dataclasses不同之处在于映射的属性使用:class:`_orm.Mapped`泛型注释容器进行描述。映射遵循与:ref:`orm_declarative_table`文档记录的相同的格式，并支持:func:`_orm.mapped_column`和:class:`_orm.Mapped`的所有功能。

此外，ORM属性配置构造，包括:func:`_orm.mapped_column`，:func:`_orm.relationship`和:func:`_orm.composite`支持**每个属性字段选项**，包括``init``，``default``，``default_factory``和``repr``。这些参数的名称固定为:pep:`681`。功能相当于dataclasses：

* ``init``，如:paramref:`_orm.mapped_column.init`，
  :paramref:`_orm.relationship.init`，如果为false，则表示该字段不应作为``__init__()``方法的一部分
* ``default``，如:paramref:`_orm.mapped_column.default`，
  :paramref:`_orm.relationship.default`
  表示该字段的默认值，可以作为关键字参数在``__init__()``方法中提供。
* ``default_factory``，如:paramref:`_orm.mapped_column.default_factory`，
  :paramref:`_orm.relationship.default_factory`，表示可调函数
  如果没有明确传递给``__init__()``方法，则将调用该函数来生成新的默认值。
* ``repr``默认为True，表示该字段应包含在生成的``__repr__()``方法中


与dataclasses的另一个关键区别是，属性的默认值必须使用ORM构造的``default``参数来配置，例如``mapped_column(default=None)``。使用类似于dataclass语法的语法，它接受简单的Python值作为默认值而不使用``@dataclases.field()``，不被支持。

使用:func:`_orm.mapped_column`的示例，如下映射将生成一个``__init__()``方法，仅接受``name``和``fullname``字段，其中``name``是必需的，可以作为位置传递，而``fullname``是可选的。 我们希望由数据库生成``id``字段不是构造函数的一部分::

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

为了适应``default``参数的名称重叠与当前:sparamref:`_schema.Column.default`的重叠参数 :class:`_schema.Column` 构造，:func:`_orm.mapped_column`构造通过增加新的参数:sparamref:`_orm.mapped_column.insert_default`进行消歧，该参数将直接填充到 :class:`_schema.Column`的 :paramref:`_schema.Column.default`参数中，独立于在:sparamref:`_orm.mapped_column.default`上可以设置的内容，该内容始终用于dataclasses配置。 例如，要配置具有``func.utc_timestamp()``,的datetime列的:paramref:`_schema.Column.default`但构造函数中的该参数是可选的::

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

使用上述映射时，当未传递``created_at``参数的新``User``对象的``INSERT``过程执行如下步骤：

.. sourcecode:: pycon+sql

    >>> with Session(e) as session:
    ...     session.add(User())
    ...     session.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO user_account (created_at) VALUES (utc_timestamp())
    [generated in 0.00010s] ()
    COMMIT



与Annotated集成
~~~~~~~~~~~~~~~~~~~~~~~~~~

在:ref:`orm_declarative_mapped_column_pep593`中引入的方法说明了如何使用:pep:`593`中的``Annotated``对象将整个:func:`_orm.mapped_column`构造打包以进行重复使用。该特征支持使用dataclasses特性。但是，其中的某些方面需要解决措施，因为typing工具可能无法正确解释对属性的:pep:`681`特殊配置。例如，鉴于以下方法，该方法在运行时将完全正常，但typing工具将认为``User()``构造函数是无效的，因为它们看不到present`` init = False``参数::

    from typing import Annotated

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    # typing工具会忽略init=False
    intpk = Annotated[int, mapped_column(init=False, primary_key=True)]

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"
        id: Mapped[intpk]


    # typing错误：参数“id”缺失
    u1 = User()

然而，:func:`_orm.mapped_column`必须在右侧也必须存在，其中具有:paramref:`_orm.mapped_column.init`的pep-681特定参数被封装到显式的:func:`_orm.mapped_column`构造中，以使typing工具正确解释属性。例如，下面的方法将正常工作，但typing工具将不会将``User()``构造函数视为有效，因为它们看不到`` init = False``参数::

    from typing import Annotated

    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry

    intpk = Annotated[int, mapped_column(primary_key=True)]

    reg = registry()


    @reg.mapped_as_dataclass
    class User:
        __tablename__ = "user_account"

        # init=False和其他pep-681参数必须内联
        id: Mapped[intpk] = mapped_column(init=False)


    u1 = User()

.. _orm_declarative_dc_mixins:

使用mixin和抽象超类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在:class:`_orm.MappedAsDataclass`映射类中使用任何mixin或基类，这些类包括:class:`_orm.Mapped`属性，它们自己必须是:class:`_orm.MappedAsDataclass`层次结构的一部分，例如，在使用mixin的示例中::

    class Mixin(MappedAsDataclass):
        create_user: Mapped[int] = mapped_column()
        update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)


    class Base(DeclarativeBase, MappedAsDataclass):
        pass


    class User(Base, Mixin):
        __tablename__ = "sys_user"

        uid: Mapped[str] = mapped_column(
            String(50), init=False, default_factory=uuid4, primary_key=True
        )
        username: Mapped[str] = mapped_column()
        email: Mapped[str] = mapped_column()

:pep:`681`支持作为属于数据类的一部分的非数据类mixin的类型工具将不起作用。

.. deprecated:: 2.0.8 在:class:`_orm.MappedAsDataclass`或:meth:`_orm.registry.mapped_as_dataclass`层次结构中使用mixin和抽象基类，它们本身不是数据类，并非都支持:pep:`681`作为属于数据类的一部分的字段。会出现这种情况的警告，后续将成为错误。

   .. seealso::

       :ref:`error_dcmx` - 背景说明


关系配置
^^^^^^^^^^^^^^^^^^^^^^^^^^

如在配置记录的:ref:`relationship_patterns`中记录的，在:class:`_orm.Mapped`注释中与:func:`_orm.relationship`结合使用的方式相同。当将基于集合的:func:`_orm.relationship`作为可选关键字参数指定时，必须传递:paramref:`_orm.relationship.default_factory`参数，并且它必须引用要使用的集合类。One-to-one和scalar对象引用可以利用:paramref:`_orm.relationship.default`，如果默认值为``None``时可以将其用于构造函数::

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

上述映射将在新的``Parent()``对象构造时为``Parent.children``生成一个空列表，类似地，在新的``Child()``对象构造时为``Child.parent``生成``None``值而无需传递``parent``。

当:class:`_orm.relationship`单独声明时，它需要直接在:paramref:`_orm.Mapper.properties`字典中指定，该字典本身指定在``__mapper_args__``字典中，以便将其传递给:class:`_orm.Mapper`的构造函数。这个方法的替代方法在下面的示例中。

.. _orm_declarative_native_dataclasses_non_mapped_fields:

使用非映射数据类字段
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当使用Declarative数据类时，也可以使用非映射字段，这些字段将成为数据类构造过程的一部分，但不会映射。通过使用正常的Dataclass语法，可以定义不使用:class:`.Mapped`的字段。不使用:class:`.Mapped`的任何字段都将被映射过程忽略。在以下示例中，字段``ctrl_one``和``ctrl_two``将成为对象的实例级状态，但不会被ORM持久化：


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

上述Data的实例可以创建如下::

    d1 = Data(status="s1", ctrl_one="ctrl1", ctrl_two="ctrl2")


一个更现实的例子可能是结合:initvar来使用``__post_init__()``特性来接收仅初始化的字段，这些字段可以用于组合持久化数据。在下面的示例中，``User``类使用``id``、``name``和``password_hash``作为映射特性，但使用初始化-only``password``和``repeat_password``字段来表示用户创建过程（注意：要运行此示例，请将函数``your_crypt_function_here()``替换为第三方加密函数，例如``bcrypt``或``argon2-cffi``）：


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

上述对象使用``password``和``repeat_password``参数创建，这些参数最先被消耗，以便在flush中从自动增量或其他默认值生成器中获取其值。允许在构造函数中明确指定它们，将为它们赋予``None``默认值。

从:class:`_orm.MappedAsDataclass`或直接应用了:meth:`_orm.registry.mapped_as_dataclass`的mixin中包括非数据类mixin的字段将被忽略，而无需显式地指定``__allow_unmapped__``类属性。在以前的2.0 beta版本中，即使只是为了使旧的ORM typed mappings继续正常工作，这个属性也需要被显式定义。

.. _dataclasses_pydantic:

与Pydantic等备用数据类提供者的集成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. warning::

    Pydantic版本1.x的数据类层与SQLAlchemy的类instrumentation并不完全兼容，而且许多功能（例如相关的collections）可能无法正确工作。

    对于Pydantic的兼容性，请考虑构建在SQLAlchemy ORM之上的Pydantic的`SQLModel <https://sqlmodel.tiangolo.com>`_ ORM，其中包含特定的实现细节，**明确解决**这些不兼容性。

SQLAlchemy的:class:`_orm.MappedAsDataclass` class和:meth:`_orm.registry.mapped_as_dataclass`方法直接调用Python标准库``dataclasses.dataclass``类装饰器，在对类进行声明性映射处理之后。可以使用``dataclass_callable``参数将此函数调用换成备用的数据类提供者，例如Pydantic，作为类关键字参数传递给:class:`_orm.MappedAsDataclass`以及:meth:`_orm.registry.mapped_as_dataclass`方法::

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

上述``User``类将被应用为数据类，并使用Pydantic的``pydantic.dataclasses.dataclasses``可调用。该过程既适用于映射类，也适用于扩展从:class:`_orm.MappedAsDataclass`或直接应用:meth:`_orm.registry.mapped_as_dataclass`的mixin。

.. versionadded：2.0.4 为:class:`_orm.MappedAsDataclass`和:meth:`_orm.registry.mapped_as_dataclass`方法添加了``dataclass_callable``类和方法参数，并调整了数据类内部以适应更严格的数据类函数，例如Pydantic的函数实现。


.. _orm_declarative_dataclasses:

将ORM映射应用于现有的数据类（传统数据类使用）
---------------------------------------------------------------------

.. legacy::

   这些方法的描述已被新的功能:ref:`orm_declarative_native_dataclasses`所取代。这个在1.4中首次添加了Dataclass的支持，这通.过在此部分中描述这个旧方法。

要将映射应用于数据类，无法直接使用SQLAlchemy的“inline”声明性指令;将ORM指令分配给类时，使用以下三种技术之一：

* 使用“Declarative with Imperative Table”，使用:class:`_schema.Table`对象定义要映射的表/列，并将其分配给类的``__table__``属性;关系在``__mapper_args__``字典内定义。使用:meth:`_orm.registry.mapped`装饰器映射类。以下是下方的示例:ref:`orm_declarative_dataclasses_imperative_table`。

* 使用完整的“Declarative”，将Declarative-interpreted指令，例如:class:`_schema.Column`，:func:`_orm.relationship`添加到``.metadata``字典的``dataclasses.field()``构造中，其中它们由声明性过程消耗。重复使用:meth:`_orm.registry.mapped`装饰器，另请参见下面显示的示例:ref:`orm_declarative_dataclasses_declarative_table`。

* 可以使用:meth:`_orm.registry.map_imperatively`方法将Imperative映射应用于现有的数据类，以完全相同的方式生成映射如:ref:`orm_imperative_mapping`中所述。这将在下面的示例:ref:`orm_imperative_dataclasses`中说明。

将ORM映射应用于数据类的一般过程与普通类的过程相同，但还包括SQLAlchemy将检测到在数据类中作为声明过程一部分的类级属性，并在运行时将其替换为通常的SQLAlchemy ORM映射属性。由dataclasses生成的``__init__``方法保持不变，其他所有由dataclasses生成的方法也保持不变，例如``__eq__()``和``__repr__()``等。

.. _orm_declarative_dataclasses_imperative_table:

使用Declarative With Imperative Table映射预存在的数据类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用:ref:`orm_imperative_table_configuration`的示例将描述如何使用``@dataclass``。显式构造完整的:class:`_schema.Table`对象并将其分配给``__table__``属性。使用正常的dataclass语法定义实例字段。其他:class:`.MapperProperty`定义，例如:func:`.relationship`，都放置在:class:`__mapper_args__ <orm_declarative_mapper_options>`类级字典中，对应于:paramref:`_orm.Mapper.properties`参数::


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

上述示例中，``User.id``、``Address.id``和``Address.user_id``属性被定义为``field(init=False)``。这意味着这些参数不会添加到``__init__()``方法中，但是Session仍将能够在获取值后在flush期间将它们设置。将它们显式指定为构造函数参数，它们将被赋予``None``默认值。

对于:func:`_orm.relationship`通过单独声明，需要直接在:paramref:`_orm.Mapper.properties`字典中指定，这个字典本身在``__mapper_args__``字典内置，以便将它传递给:class:`_orm.Mapper`的构造函数。此方法的另一种方法在下面的示例中。


.. _orm_declarative_dataclasses_declarative_table:

使用Declarative-style字段映射预存在的数据类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. legacy:: 完全声明式方法需要将:class:`_schema.Column`对象声明为类属性。这将与使用dataclass级别的属性的冲突。结合使用时，可以利用``metadata``属性。在``dataclass.field``对象中，可以提供SQLAlchemy特定的映射信息。Declarative支持在类指定属性``__sa_dataclass_metadata_key__``时的这些参数的提取。这还提供了一种更简洁的方法来指示:func:`_orm.relationship`关联::

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

addresses: List[Address] = field默认值为列表，元素类型为Address，其中metadata参数用来设置属性的元数据信息，这里设置了一个sa键，值为关系数据的配置信息，即关联Address表。

由此可见metadata参数的类似于声明方式给与了dataclasses通过注释生成ORM对象的灵活性，使其能够通过注释继承和自定义ORM属性和关系的实现。

对于Address类也有类似的注释实现，id、user_id和email_address分别对应自增主键、用户id和email地址信息。

在本文档的另一个章节orm_declarative_dataclasses_mixin中，详细介绍了如何将声明性的混合类用于已有的dataclass中，其中介绍了基于声明式的orm混合类的要求，并给出了基于dataclass的实现方式。实现的主要步骤是使用Lambda函数在metadata中表示ORM construct（例如Column、relationship等）。

最后本文档通过使用属性包装函数_declared_attr_修饰lambda函数来实现定义属性函数的目标，并给出了一个将dataclass映射为ORM类的例子。在此例子中，实体类User和Address类使用了两个已有的ORM类UserMixin和AddressMixin，并通过数据类混合方式继承他们的功能。值得一提的是，这里的ORM类是通过Python的数据类实现的，实现了类似SQLAlchemy的声明式混合，使得我们的ORM层的语法看起来比纯Python代码简洁并且易于阅读及维护。

在orm_imperative_dataclasses章节中，介绍了如何利用一种叫做_map_imperatively_的技术将已有的dataclass映射为ORM类。同时还给出了数据类的另一个实现方式：使用attrs模块的示例代码，并介绍了如何进行Python类型注解。

最后，本文档还详细的讲述了在Python的ORM领域中两种主流的类型检查工具mypy和pyright在可读性和错误提示方面的优越性以及对这些注释的支持程度。