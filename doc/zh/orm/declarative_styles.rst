.. _orm_declarative_styles_toplevel:

=============================================
声明式映射风格
=============================================

如 :ref:`orm_declarative_mapping` 中所介绍的，**声明式映射(Declarative Mapping)**是现代SQLAlchemy中构造映射的典型方式。本节将提供用于声明性mapper配置的概述。

.. _orm_explicit_declarative_base:

.. _orm_declarative_generated_base_class:

使用声明基类
-------------------------------

最常见的做法是通过子类化 :class:`_orm.DeclarativeBase` 超类来生成"声明基类"::

    from sqlalchemy.orm import DeclarativeBase


    # 声明基类
    class Base(DeclarativeBase):
        pass

也可以使用现有的 :class:`_orm.registry` ，将其分配为名为 "registry" 的类变量即可创建声明基类::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import registry

    reg = registry()


    # 声明基类
    class Base(DeclarativeBase):
        registry = reg

.. versionchanged:: 2.0 
   :class:`_orm.DeclarativeBase` 超类取代了使用 :func:`_orm.declarative_base` 函数和 
   :meth:`_orm.registry.generate_base` 方法的方式；超类的方法可以和 :pep:`484` 工具一起使用。 
   请参阅 :ref:`whatsnew_20_orm_declarative_typing` 获取迁移说明。

使用声明基类，新的映射类被声明为基类的子类::

    from datetime import datetime
    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import func
    from sqlalchemy import Integer
    from sqlalchemy import String
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        name: Mapped[str]
        fullname: Mapped[Optional[str]]
        nickname: Mapped[Optional[str]] = mapped_column(String(64))
        create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

        addresses: Mapped[List["Address"]] = relationship(back_populates="user")


    class Address(Base):
        __tablename__ = "address"

        id = mapped_column(Integer, primary_key=True)
        user_id = mapped_column(ForeignKey("user.id"))
        email_address: Mapped[str]

        user: Mapped["User"] = relationship(back_populates="addresses")

上面的代码中，"Base" 类用作将要映射的新类的基础，对于新的映射类 ``User`` 和 ``Address`` 也是如此构造的。

对于每个创建的子类，类体遵循声明式映射方法，该方法在幕后定义了一个 :class:`_schema.Table` 和一个 :class:`_orm.Mapper` 对象，这些对象组成了一个完整的映射。

.. seealso::

    :ref:`orm_declarative_table_config_toplevel` - 描述了如何指定将要生成的映射 :class:`_schema.Table` 的组件，包括关于使用 :func:`_orm.mapped_column`构造函数及其与 :class:`_orm.Mapped` 注释类型的交互的注意事项和选项

    :ref:`orm_declarative_mapper_config_toplevel` - 描述了在声明式中配置ORM映射器的所有其他方面，包括 :func:`_orm.relationship` 配置、 SQL 表达式和 :class:`_orm.Mapper` 参数


.. _orm_declarative_decorator:

使用装饰器进行声明式映射（无基类）
------------------------------------------------------------

作为使用"声明基类"类的替代方式，可以显式地将声明式映射应用于一个类，使用与"经典"映射类似的命令式技术，或者更简洁地使用装饰器。:meth:`_orm.registry.mapped` 函数是一个类装饰器，可以应用于没有现有结构的任何Python类。否则，Python类通常以声明式样式配置。

以下示例使用了 :meth:`_orm.registry.mapped` 装饰器而不是使用 :class:`_orm.DeclarativeBase` 超类来设置相同的映射：


    from datetime import datetime
    from typing import List
    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import func
    from sqlalchemy import Integer
    from sqlalchemy import String
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import registry
    from sqlalchemy.orm import relationship

    mapper_registry = registry()


    @mapper_registry.mapped
    class User:
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        name: Mapped[str]
        fullname: Mapped[Optional[str]]
        nickname: Mapped[Optional[str]] = mapped_column(String(64))
        create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

        addresses: Mapped[List["Address"]] = relationship(back_populates="user")


    @mapper_registry.mapped
    class Address:
        __tablename__ = "address"

        id = mapped_column(Integer, primary_key=True)
        user_id = mapped_column(ForeignKey("user.id"))
        email_address: Mapped[str]

        user: Mapped["User"] = relationship(back_populates="addresses")

使用上述样式时，仅当将装饰器直接应用于该类时，才会进行特定类的映射。对于继承映射（详见：:ref:`inheritance_toplevel`）, 应该分别将装饰器应用于要进行映射的每个子类::

    from sqlalchemy.orm import registry

    mapper_registry = registry()


    @mapper_registry.mapped
    class Person:
        __tablename__ = "person"

        person_id = mapped_column(Integer, primary_key=True)
        type = mapped_column(String, nullable=False)

        __mapper_args__ = {
            "polymorphic_on": type,
            "polymorphic_identity": "person",
        }


    @mapper_registry.mapped
    class Employee(Person):
        __tablename__ = "employee"

        person_id = mapped_column(ForeignKey("person.person_id"), primary_key=True)

        __mapper_args__ = {
            "polymorphic_identity": "employee",
        }

无论是声明基类还是装饰器风格的声明式映射，都可以使用 :ref:`declarative table <orm_declarative_table>` 和 :ref:`imperative table <orm_imperative_table_configuration>`两种表配置样式。

装饰器形式的映射在将 SQLAlchemy 声明式映射与其他类内部机制，例如 dataclasses_ 和 attrs_ 等合并时，非常有用。但请注意，SQLAlchemy 2.0 现在还将数据类(dataclasses)与声明式基类的集成作为常见用例支持。

.. _dataclass: https://docs.python.org/zh-cn/3/library/dataclasses.html
.. _dataclasses: https://docs.python.org/zh-cn/3/library/dataclasses.html
.. _attrs: https://pypi.org/project/attrs/