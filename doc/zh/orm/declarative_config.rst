.. _orm_declarative_mapper_config_toplevel:

=============================================
使用声明式进行Mapper配置
=============================================

本文档中的   :ref:`orm_mapper_configuration_overview`  部分讨论了   :class:` _orm.Mapper`  构造的一般配置元素，该结构定义了特定用户定义的类如何映射到数据库表或其他 SQL 结构。接下来的章节描述了声明式系统如何构建   :class:`_orm.Mapper`  的具体细节。

.. _orm_declarative_properties:

使用声明式定义映射属性
--------------------------------------------

在   :ref:`orm_declarative_table_config_toplevel`  中给出的示例说明了映射到基于表的列的映射，使用了   :func:` _orm.mapped_column`  构造方法。除基于表的列之外，还有几种ORM映射构造方法可以配置，最常见的是   :func:`_orm.relationship`  构造方法。其它类型的属性包括使用   :func:` _orm.column_property`  构造方法定义的 SQL 表达式和使用   :func:`_orm.composite`  构造方法的多列映射。

在   :ref:`orm_mapping_properties`  中，   :ref:` orm_imperative_mapping`  使用   :ref:`properties <orm_mapping_properties>`  字典来定义所有映射类属性。在声明式映射中，这些属性都在类定义中内联指定，在声明式表映射的情况下，这些属性都与将用于生成   :class:` _schema.Table`  对象的   :class:`_schema.Column`  对象内联。

在完成“用户”和“地址”的示例映射工作之后，我们可以说明一个声明式表映射，其中包括不仅有   :func:`_orm.mapped_column`  对象，还包括关系和 SQL 表达式：

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

上述声明式表映射包含两个表，每个表都有指向另一个表的   :func:`_orm.relationship` ，以及一个由   :func:` _orm.column_property`  映射的简单 SQL 表达式和一个额外的   :func:`_orm.mapped_column` ，该 _orm_class="literal" 加载应根据  :paramref:` _orm.mapped_column.deferred`  关键字定义的“延迟”方式进行。关于这些特定概念的更多文档信息可以在文档   :ref:`relationship_patterns` 、  :ref:` mapper_column_property_sql_expressions`  和   :ref:`orm_queryguide_column_deferral`  中找到。

使用声明式映射也可以使用“混合表”样式来指定属性；直接属于表的   :class:`_schema.Column`  对象会移动到   :class:` _schema.Table`  定义中，但包括组成的 SQL 表达式在内的所有其他内容仍将内联到类定义中。需要引用   :class:`_schema.Column`  的构造方法会将其用   :class:` _schema.Table`  对象表示。为了使用混合式表格样式说明上述映射：

    # 使用声明式映射属性并带有命令式表格 i.e. __table__

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

上述示例中需要注意的事项：

* address   :class:`_schema.Table`  包含一个名为 ` `address_statistics`` 的列，我们将此列重新映射到相同的属性名称下，以便其受   :func:`_orm.deferred`  构造方法的控制。

* 对于声明性表和混合表映射，当我们定义   :class:`_schema.ForeignKey`  构造方法时，我们总是使用表格名称而不是映射类名称作为目标表格。

* 当我们定义   :func:`_orm.relationship`  构造方法时，由于这些构造方法在将一个映射类与另一个映射类链接在一起时，必定有一个类会比另一个类先定义，因此我们可以使用该类的字符串名称来引用远程类。此功能还扩展到   :func:` _orm.relationship`  上指定的其它参数，如“primary join”和“order by”参数。请参阅   :ref:`orm_declarative_relationship_eval`  部分获取有关详细信息。

.. _orm_declarative_mapper_options:

用声明方式进行Mapper配置选项
----------------------------------------------

使用所有映射形式时，该类的映射是通过成为   :class:`_orm.Mapper`  对象的一部分的参数进行配置的。接受这些参数的函数是   :class:` _orm.Mapper`  函数，这些参数从   :class:`_orm.registry`  可以看到的前线映射函数中传递给它。声明性映射的形式中，映射程序参数使用 ` `__mapper_args__`` 声明式类变量指定，该变量是作为关键字参数传递给   :class:`_orm.Mapper`  函数的字典。一些示例：

**映射具体的主键列**

下面的示例说明了针对  :paramref:`_orm.Mapper.primary_key`  参数的声明式级别设置，该参数独立于基于 schema 级别的主键约束将特定列作为 ORM 应将其视为主键的一部分：

    class GroupUsers(Base):
        __tablename__ = "group_users"

        user_id = mapped_column(String(40))
        group_id = mapped_column(String(40))

        __mapper_args__ = {"primary_key": [user_id, group_id]}

.. seealso::

      :ref:`mapper_primary_key`  - 有关显式列映射为主键列的 ORM 映射的后台信息

**版本 ID 列**

下面的示例说明了  :paramref:`_orm.Mapper.version_id_col`  和  :paramref:` _orm.Mapper.version_id_generator`  参数的声明式级别设置，它们配置了 ORM 维护的版本计数器，该计数器在  :term:`unit of work`  刷新过程中进行更新和检查：

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

      :ref:`mapper_version_counter`  - ORM 版本计数器功能的背景信息

**单表继承**

下面的示例说明了针对  :paramref:`_orm.Mapper.polymorphic_on`  和  :paramref:` _orm.Mapper.polymorphic_identity`  参数的声明式级别设置，这些参数用于配置单表继承映射：

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

      :ref:`single_inheritance`  - ORM 单表继承映射功能背景

动态构造映射器参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以通过使用   :func:`_orm.declared_attr`  构造方法从类绑定的描述符方法生成 ` `__mapper_args__`` 字典，而不是从固定字典中生成，这对于从表配置或映射类的其他方面编程派生出的 Mapper 参数是有用的。动态 ``__mapper_args__`` 属性通常在使用声明式 Mixin 或抽象基类时非常有用。

例如，要从具有特殊  :attr:`.Column.info`  值的列中省略映射的任何列，mixin 可以使用扫描这些列并将它们传递给  :paramref:` _orm.Mapper.exclude_properties`  集合的 ``__mapper_args__`` 方法从 ``cls.__table__`` 属性中获取这些列的名称 "键" 建议::

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

以上，``ExcludeColsWFlag`` mixin 提供了一个每个类的 ``__mapper_args__`` 钩子，该钩子将扫描传递给  :paramref:`.Column.info`  参数的“exclude”:True 的   :class:` .Column`  对象，然后将其“键”名称加入  :paramref:`_orm.Mapper.exclude_properties`  集合中，从而防止生成的   :class:` .Mapper`  将这些列考虑为任何 SQL 操作。

.. seealso::

      :ref:`orm_mixins_toplevel` 


其它声明式映射指令
--------------------------------------

``__declare_last__()``
~~~~~~~~~~~~~~~~~~~~~~

``__declare_last__()`` 钩子允许定义一个类级别函数，该函数由  :meth:`.MapperEvents.after_configured`  事件自动调用，该事件在映射被认为已完成并且 'configure' 步骤已完成时发生：

    class MyClass(Base):
        @classmethod
        def __declare_last__(cls):
            """ """
            # 使用映射

``__declare_first__()``
~~~~~~~~~~~~~~~~~~~~~~~

与 ``__declare_last__()`` 相似，但是使用  :meth:`.MapperEvents.before_configured`  事件在 Mapper 配置开始时调用：

    class MyClass(Base):
        @classmethod
        def __declare_first__(cls):
            """ """
            # 在映射配置之前做一些事情

.. _declarative_metadata:

``metadata``
~~~~~~~~~~~~

通常用于分配新   :class:`_schema.Table`  的   :class:` _schema.MetaData`  集合是与   :class:`_orm.registry`  对象相关联的  :attr:` _orm.registry.metadata`  属性。当使用由   :class:`_orm.DeclarativeBase`  超类生成的声明性基类时，以及使用诸如   :func:` _orm.declarative_base`  和  :meth:`_orm.registry.generate_base`  的遗留函数时，该   :class:` _schema.MetaData`  也通常存在于基类中作为名为“metadata”的属性，因此也存在于通过继承映射类。声明式将使用此属性（如果存在）来确定目标   :class:`_schema.MetaData`  集合，否则将使用直接关联   :class:` _orm.registry`  的   :class:`_schema.MetaData` 。

此属性也可以被分配，以便基于每个映射层次结构对单个基础和/或   :class:`_orm.registry`  使用的   :class:` _schema.MetaData`  集合。这在使用 Declarative Mixin 或抽象基类的元数据 - 每抽象基类一次 的模式中生效，例如，下一节   :ref:`declarative_abstract`  就是这样的示例。也可以使用  :meth:` _orm.registry.mapped`  作为下列示例说明：

    reg = registry()


    class BaseOne:
        metadata = MetaData()


    class BaseTwo:
        metadata = MetaData()


    @reg.mapped
    class ClassOne:
        __tablename__ = "t1"  # 将使用 reg.metadata

        id = mapped_column(Integer, primary_key=True)


    @reg.mapped
    class ClassTwo(BaseOne):
        __tablename__ = "t1"  # 将使用 BaseOne.metadata

        id = mapped_column(Integer, primary_key=True)


    @reg.mapped
    class ClassThree(BaseTwo):
        __tablename__ = "t1"  # 将使用 BaseTwo.metadata

        id = mapped_column(Integer, primary_key=True)

以上，从 ``DefaultBase`` 继承的类将使用一个   :class:`_schema.MetaData`  作为表的数据中心，在从 ` `OtherBase`` 继承的类中使用一个不同的   :class:`_schema.MetaData` 。然后可以创建这些表格，比如在不同的数据库中：

    DefaultBase.metadata.create_all(some_engine)
    OtherBase.metadata.create_all(some_other_engine)

.. seealso::

      :ref:`declarative_abstract` 

.. _declarative_abstract:

``__abstract__``
~~~~~~~~~~~~~~~~

``__abstract__`` 导致声明式跳过为类完全生成表或 Mapper。类可以像 Mixin 一样添加到层次结构中，以允许子类只扩展来自特殊类的内容：

    class SomeAbstractBase(Base):
        __abstract__ = True

        def some_helpful_method(self):
            """ """

        @declared_attr
        def __mapper_args__(cls):
            return {"helpful mapper arguments": True}


    class MyMappedClass(SomeAbstractBase):
        pass

``__abstract__`` 的一个可能的用途是为不同的基类使用不同的   :class:`_schema.MetaData` ：

    class Base(DeclarativeBase):
        pass


    class DefaultBase(Base):
        __abstract__ = True
        metadata = MetaData()


    class OtherBase(Base):
        __abstract__ = True
        metadata = MetaData()

以上，在从 ``DefaultBase`` 继承的类中，将使用一个   :class:`_schema.MetaData`  作为表的注册中心，在从 ` `OtherBase`` 继承的类中将使用一个不同的   :class:`_schema.MetaData` 。然后，可以创建这些表格，例如在不同的数据库中：

    DefaultBase.metadata.create_all(some_engine)
    OtherBase.metadata.create_all(some_other_engine)

.. seealso::

      :ref:`orm_inheritance_abstract_poly`  - 一种适用于继承层次结构的替代“抽象”映射类。
    
.. _declarative_table_cls:

``__table_cls__``
~~~~~~~~~~~~~~~~~

允许自定义用于生成   :class:`_schema.Table`  的可调用 / 类。这是一个非常开放的钩子，可以允许特殊自定义到一个   :class:` _schema.Table`  中，在此处生成：

    class MyMixin:
        @classmethod
        def __table_cls__(cls, name, metadata_obj, *arg, **kw):
            return Table(f"my_{name}", metadata_obj, *arg, **kw)

上述 mixin 将导致生成的所有   :class:`_schema.Table`  包括以 ` `"my_"`` 为前缀，后面是通常使用 ``__tablename__`` 属性指定的名称。

``__table_cls__`` 还支持返回 ``None`` 的情况，这会导致类被视为子类而不是单个表。在某些自定义方案中，这可能很有用，例如，如果没有主键存在，则将定义为单继承：

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

上述 ``Employee`` 类将被映射为单表继承映射的子类 ``Person``；``employee_name`` 列将添加为 ``Person`` 表的成员。