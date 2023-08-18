.. _orm_declarative_mapper_config_toplevel:

=============================================
使用 Declarative 进行映射器配置
=============================================

在 :ref:`orm_mapper_configuration_overview` 章节中讲述了 :class:`_orm.Mapper` 的一般配置元素，该结构定义了特定用户定义类如何映射到数据库表或其他 SQL 结构。以下章节描述了 Declarative 系统如何构建 :class:`_orm.Mapper` 的具体细节。

.. _orm_declarative_properties:

使用 Declarative 定义映射属性
--------------------------------

:ref:`orm_declarative_table_config_toplevel` 章节中给出的示例演示了针对基于表的列使用 :func:`_orm.mapped_column` 构造进行映射。除了基于表的列之外，还可以配置多种其他类型的 ORM 映射结构，其中最常见的是 :func:`_orm.relationship` 构造。其他类型的属性包括使用 :func:`_orm.column_property` 构造定义的 SQL 表达式和使用 :func:`_orm.composite` 构造的多列映射。

使用命令式映射时，使用 :ref:`properties <orm_mapping_properties>` 字典来建立所有映射类属性，在声明式映射中，这些属性都在与类定义内联的位置指定，就像在声明式表映射中与将用于生成 :class:`_schema.Column` 对象的对象类一样。

在使用 ``User`` 和 ``Address`` 的示例映射中，我们可以演示一个声明式表映射，其中不仅包括 :func:`_orm.mapped_column` 对象，还包括关系和 SQL 表达式::

    from typing import List
    from typing import Optional

    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import String
    from sqlalchemy import Text
    from sqlalchemy.orm import column_property
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        firstname: Mapped[str] = mapped_column(String(50))
        lastname: Mapped[str] = mapped_column(String(50))
        fullname: Mapped[str] = column_property(firstname + " " + lastname)

        addresses: Mapped[List["Address"]] = relationship(back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        user_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
        email_address: Mapped[str]
        address_statistics: Mapped[Optional[str]] = mapped_column(Text, deferred=True)

        user: Mapped["User"] = relationship(back_populates="addresses")

上述声明式表映射包含两个表，每个表都有一个 :func:`_orm.relationship` 引用另一个表，并且由 :func:`_orm.column_property` 映射的一个简单的 SQL 表达式以及一个额外的 :func:`_orm.mapped_column`，它表示加载应以由 :paramref:`_orm.mapped_column.deferred` 关键字定义的“延迟”方式。有关这些特定概念的详细信息可以在 :ref:`relationship_patterns`、:ref:`mapper_column_property_sql_expressions` 和 :ref:`orm_queryguide_column_deferral` 中找到。

可以使用混合表风格来指定使用声明性映射的上述属性；直接与表相关的 :class:`_schema.Column` 对象移到 :class:`_schema.Table` 定义中，但所有其他内容，包括组成的 SQL 表达式，仍然内联于类定义中。需要引用 :class:`_schema.Column` 的构造函数需要使用 :class:`_schema.Table` 对象来引用它。使用混合表风格来说明上述映射：

    # mapping attributes using declarative with imperative table
    # i.e. __table__

    from sqlalchemy import Column, ForeignKey, Integer, String, Table, Text
    from sqlalchemy.orm import column_property
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import deferred
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __table__ = Table(
            "user",
            Base.metadata,
            Column("id", Integer, primary_key=True),
            Column("name", String),
            Column("firstname", String(50)),
            Column("lastname", String(50)),
        )

        fullname = column_property(__table__.c.firstname + " " + __table__.c.lastname)

        addresses = relationship("Address", back_populates="user")


    class Address(Base):
        __table__ = Table(
            "address",
            Base.metadata,
            Column("id", Integer, primary_key=True),
            Column("user_id", ForeignKey("user.id")),
            Column("email_address", String),
            Column("address_statistics", Text),
        )

        address_statistics = deferred(__table__.c.address_statistics)

        user = relationship("User", back_populates="addresses")

请注意以下几点：

* address :class:`_schema.Table` 中包含一个名为 ``address_statistics`` 的列，但我们将该列重新映射到属性名相同的属性，以便受 :func:`_orm.deferred` 构造的控制。
* 当我们使用 :class:`_schema.ForeignKey` 构造时，始终使用 **表名称** 而不是映射类名称来命名目标表。
* 当我们定义 :func:`_orm.relationship` 构造时，由于这些构造在一个映射类之间创建链接，其中一个必然在另一个之前定义，所以我们可以使用字符串名称来引用远程类。对 :func:`_orm.relationship` 上指定的其他参数（如“primary join”和“order by”参数）的功能也可以扩展到此功能。有关详细信息，请参见 :ref:`orm_declarative_relationship_eval` 节。
 
.. _orm_declarative_mapper_options:

使用 Declarative 进行 Mapper 配置选项
---------------------------------------

在所有映射形式中，类的映射是通过成为 :class:`_orm.Mapper` 对象的一部分的参数来配置的。最终接收这些参数的函数是 :class:`_orm.Mapper` 函数，并且是从 :class:`_orm.registry` 对象上定义的其中一个前置映射函数将其传递给该函数。

对于映射形式，使用 ``__mapper_args__`` 声明类变量来指定映射器参数，它是一个将作为关键字参数传递给 :class:`_orm.Mapper` 函数的字典。下面是一些示例：

**映射特定的主键列**

下面的示例演示了针对 :paramref:`_orm.Mapper.primary_key` 参数的 Declarative 级别设置，这些参数定义了特定列作为 ORM 应该视为主键，而独立于架构级别的主键约束：

    class GroupUsers(Base):
        __tablename__ = "group_users"

        user_id = mapped_column(String(40))
        group_id = mapped_column(String(40))

        __mapper_args__ = {"primary_key": [user_id, group_id]}

.. seealso::

    :ref:`mapper_primary_key` - 关于显式列映射为主键列的 ORM 映射的更多背景信息

**Version ID 列**

下面的示例演示了关于 :paramref:`_orm.Mapper.version_id_col` 和
:paramref:`_orm.Mapper.version_id_generator` 参数的 Declarative 级别设置，它们配置了一个在 :term:`unit of work` 刷新过程中更新并检查的 ORM 维护的版本计数器：

    from datetime import datetime


    class Widget(Base):
        __tablename__ = "widgets"

        id = mapped_column(Integer, primary_key=True)
        timestamp = mapped_column(DateTime, nullable=False)

        __mapper_args__ = {
            "version_id_col": timestamp,
            "version_id_generator": lambda v: datetime.now(),
        }

.. seealso::

    :ref:`mapper_version_counter` - 关于 ORM 版本计数器功能的说明

**单表继承**

下面的示例演示了关于 :paramref:`_orm.Mapper.polymorphic_on` 和 :paramref:`_orm.Mapper.polymorphic_identity` 参数的 Declarative 级别设置，它们用于配置单表继承映射：

    class Person(Base):
        __tablename__ = "person"

        person_id = mapped_column(Integer, primary_key=True)
        type = mapped_column(String, nullable=False)

        __mapper_args__ = dict(
            polymorphic_on=type,
            polymorphic_identity="person",
        )


    class Employee(Person):
        __mapper_args__ = dict(
            polymorphic_identity="employee",
        )

.. seealso::

    :ref:`single_inheritance` - 有关 ORM 单表继承映射功能的背景

动态构造映射器参数
~~~~~~~~~~~~~~~~~~~~

可以使用类绑定的描述符方法来从可调用对象或类中生成 ``__mapper_args__`` 字典，而不是从固定字典中生成。这对于从表配置或映射类的其他方面编程派生映射器参数非常有用。动态 ``__mapper_args__`` 属性通常在使用 Declarative Mixin 或抽象基类时是有用的。

例如，为了从映射表中排除具有特殊 :attr:`.Column.info` 值的任何列，Mixin 可以使用一个 ``__mapper_args__`` 方法，该方法扫描 ``cls.__table__`` 属性中的这些列并将它们添加到 :paramref:`_orm.Mapper.exclude_properties` 集合中：

    from sqlalchemy import Column
    from sqlalchemy import Integer
    from sqlalchemy import select
    from sqlalchemy import String
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import declared_attr


    class ExcludeColsWFlag:
        @declared_attr
        def __mapper_args__(cls):
            return {
                "exclude_properties": [
                    column.key
                    for column in cls.__table__.c
                    if column.info.get("exclude", False)
                ]
            }


    class Base(DeclarativeBase):
        pass


    class SomeClass(ExcludeColsWFlag, Base):
        __tablename__ = "some_table"

        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String)
        not_needed = mapped_column(String, info={"exclude": True})

上述 ``ExcludeColsWFlag`` Mixin 提供了一个每类的 ``__mapper_args__`` 钩子，该钩子将扫描包括 ``'exclude': True`` 在内的参数传递给 :paramref:`.Column.info` 的 :class:`.Column` 对象，然后将它们的字符串“键”名称添加到 :paramref:`_orm.Mapper.exclude_properties` 集合中，这将防止产生的 :class:`.Mapper` 考虑这些列进行任何 SQL 操作。

.. seealso::

    :ref:`orm_mixins_toplevel`

其他 Declarative 映射指令
--------------------------------------

``__declare_last__()``
~~~~~~~~~~~~~~~~~~~~~~

``__declare_last__()`` 钩子允许定义一个类级函数，它将在 :meth:`.MapperEvents.after_configured` 事件自动调用，该事件在映射被视为完成并且“配置”步骤已完成后发生：

    class MyClass(Base):
        @classmethod
        def __declare_last__(cls):
            """ """
            # do something with mappings

``__declare_first__()``
~~~~~~~~~~~~~~~~~~~~~~~

像 ``__declare_last__()`` 一样，但通过 :meth:`.MapperEvents.before_configured` 事件在映射配置开始时调用：

    class MyClass(Base):
        @classmethod
        def __declare_first__(cls):
            """ """
            # do something before mappings are configured

.. _declarative_metadata:

``metadata``
~~~~~~~~~~~~

通常用于分配新 :class:`_schema.Table` 的 :class:`_schema.MetaData` 集合是 :class:`_orm.registry` 对象附带的 :attr:`_orm.registry.metadata` 属性。当使用 Declarative 基类（例如，由 :class:`_orm.DeclarativeBase` 超类产生的），以及 :func:`_orm.declarative_base` 和 :meth:`_orm.registry.generate_base` 等遗留函数时，该 :class:`_schema.MetaData` 通常作为名为 ``.metadata`` 的属性直接存在于基类上，因此也存在于映射类通过继承。 Declarative 使用此属性（如果存在）来确定要使用的目标 :class:`_schema.MetaData` 集合，如果不存在，则使用与 :class:`_orm.registry` 直接相关联的 :class:`_schema.MetaData`。

此属性也可以分配给每个映射层次结构以基类和/或 :class:`_orm.registry` 为基础，以影响要针对单个基类和/或 :class:`_orm.registry` 使用的 :class:`_schema.MetaData` 集合。这将生效，无论使用的是否是声明性基类或直接使用 :meth:`_orm.registry.mapped` 装饰器，从而允许模式（例如每个抽象基类的元数据）在多个数据库中生成表。下面的示例说明了使用 :meth:`_orm.registry.mapped` 语句来说明此映射：

    reg = registry()


    class BaseOne:
        metadata = MetaData()


    class BaseTwo:
        metadata = MetaData()


    @reg.mapped
    class ClassOne:
        __tablename__ = "t1"  # will use reg.metadata

        id = mapped_column(Integer, primary_key=True)


    @reg.mapped
    class ClassTwo(BaseOne):
        __tablename__ = "t1"  # will use BaseOne.metadata

        id = mapped_column(Integer, primary_key=True)


    @reg.mapped
    class ClassThree(BaseTwo):
        __tablename__ = "t1"  # will use BaseTwo.metadata

        id = mapped_column(Integer, primary_key=True)

上述示例中，从 ``DefaultBase`` 继承的类将使用一个 :class:`_schema.MetaData` 作为表的注册表，从 ``OtherBase`` 继承的类将使用不同的 :class:`_schema.MetaData`。然后表本身可以在不同的数据库中生成，例如：

    DefaultBase.metadata.create_all(some_engine)
    OtherBase.metadata.create_all(some_other_engine)

.. seealso::

    :ref:`declarative_abstract`

.. _declarative_abstract:

``__abstract__``
~~~~~~~~~~~~~~~~

``__abstract__`` 使 Declarative 完全跳过为此类生成表或映射器的过程。类可以像 Mixin 一样添加到层次结构中，允许子类仅从特殊的类继承：

    class SomeAbstractBase(Base):
        __abstract__ = True

        def some_helpful_method(self):
            """ """

        @declared_attr
        def __mapper_args__(cls):
            return {"helpful mapper arguments": True}


    class MyMappedClass(SomeAbstractBase):
        pass

``__abstract__`` 的一个可能用途是使用不同的 :class:`_schema.MetaData` 为不同基类生成表::

    class Base(DeclarativeBase):
        pass


    class DefaultBase(Base):
        __abstract__ = True
        metadata = MetaData()


    class OtherBase(Base):
        __abstract__ = True
        metadata = MetaData()

上述类继承自 ``DefaultBase`` 的类将使用一个 :class:`_schema.MetaData` 作为表的注册表，而继承自 ``OtherBase`` 的类将使用不同的 :class:`_schema.MetaData`。然后表本身可以在不同的数据库中生成。

.. seealso::

    :ref:`orm_inheritance_abstract_poly` - 适用于继承层次结构的“抽象”映射类的另一种形式。

.. _declarative_table_cls:

``__table_cls__``
~~~~~~~~~~~~~~~~~

允许自定义用于生成 :class:`_schema.Table` 的可调用/类。这是一个非常开放的钩子，可以允许特殊自定义到一个 :class:`_schema.Table` 中，该表在此处生成：

    class MyMixin:
        @classmethod
        def __table_cls__(cls, name, metadata_obj, *arg, **kw):
            return Table(f"my_{name}", metadata_obj, *arg, **kw)

以上 Mixin 将导致生成所有 :class:`_schema.Table` 对象时包含前缀 ``"my_"``，后跟通常使用 ``__tablename__`` 属性指定的名称。

``__table_cls__`` 还支持返回 ``None`` 的情况，这将导致将此类视为单表继承的子类。这在某些自定义方案中可能会有用，以便基于表本身的参数（例如，没有主键时定义为单继承）确定单表继承是否应该发生：

    class AutoTable:
        @declared_attr
        def __tablename__(cls):
            return cls.__name__

        @classmethod
        def __table_cls__(cls, *arg, **kw):
            for obj in arg[1:]:
                if (isinstance(obj, Column) and obj.primary_key) or isinstance(
                    obj, PrimaryKeyConstraint
                ):
                    return Table(*arg, **kw)

            return None


    class Person(AutoTable, Base):
        id = mapped_column(Integer, primary_key=True)


    class Employee(Person):
        employee_name = mapped_column(String)

上述 ``Employee`` 类将映射为单表继承，针对 ``Person``； ``employee_name`` 列将添加为 ``Person`` 表的成员。