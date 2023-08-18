.. _orm_declarative_styles_toplevel:

====================
声明式映射风格
====================

正如在   :ref:`orm_declarative_mapping`  中介绍的那样，“声明式映射”是现代 SQLAlchemy 中构建映射的典型方法。本节将提供用于Declarative 映射配置的概述。

.. _orm_explicit_declarative_base:

.. _orm_declarative_generated_base_class:

使用声明式基类
-------------------

最常见的方法是通过继承   :class:`_orm.DeclarativeBase`  超类来生成“Declarative Base”类::

    from sqlalchemy.orm import DeclarativeBase


    # 声明基类
    class Base(DeclarativeBase):
        pass

也可以通过将其分配为名为“registry”的类变量来创建声明式基类，并给出一个现有的   :class:`_orm.registry` ，如下所示::

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import registry

    reg = registry()


    # 声明基类
    class Base(DeclarativeBase):
        registry = reg

.. versionchanged:: 2.0   :class:`_orm.DeclarativeBase`  超类取代了   :func:` _orm.declarative_base`  函数和  :meth:`_orm.registry.generate_base`  方法的使用。超类方法可以在不使用插件的情况下与  :pep:` 484`  工具集成。
   有关迁移说明，请参阅   :ref:`whatsnew_20_orm_declarative_typing` 。

使用声明基类，新映射类被声明为基类的子类::

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

在上面的示例中，``Base`` 类作为一个新类的基类，在此基础上构建了新的映射类 ``User`` 和 ``Address``。

对于构造的每个子类，类的主体遵循声明式映射方法，该方法在幕后定义了   :class:`_schema.Table`  和一个   :class:` _orm.Mapper`  对象，这些对象组成了一个完整的映射。

.. seealso::

      :ref:`orm_declarative_table_config_toplevel`  - 描述如何指定要生成的映射   :class:` _schema.Table`  的组件，其中包括有关   :func:`_orm.mapped_column`  构造的注释类型   :class:` _orm.Mapped`  如何与之交互的注释和选项

      :ref:`orm_declarative_mapper_config_toplevel`  - 描述 Declarative 中 ORM Mapper 配置的所有其他方面，包括   :func:` _orm.relationship`  配置、SQL 表达式和   :class:`_orm.Mapper`  参数


.. _orm_declarative_decorator:

使用装饰器声明式映射（无声明式基类）
------------------------------------------------------------

与使用“声明式基类”不同，另一种方法是显式地将声明式映射应用于类，采用类似于“经典”映射的命令技术，或者更简洁地使用装饰器。  :meth:`_orm.registry.mapped`  函数是一个可以应用于任何 Python 类的类装饰器，而D Python 类则通常采用声明表达式样式配置。

以下示例使用  :meth:`_orm.registry.mapped`  装饰器而不是使用   :class:` _orm.DeclarativeBase`  超类设置相同的映射::

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

使用上述样式时，只有在直接应用装饰器的类映射到，特定类的映射才会进行。 对于继承映射（在   :ref:`inheritance_toplevel`  中详细描述），应将装饰器应用于要映射的每个子类::

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

使用 Declarative 映射的声明式表   :ref:`orm_declarative_table`  和命令式表   :ref:` orm_imperative_table_configuration`  都可以与 Declarative 基类或装饰器风格的 Declarative 映射一起使用。

当将 SQLAlchemy 声明式映射与其他类仪表系统（如 dataclasses_ 和 attrs_）组合使用时，装饰器形式的映射是非常有用的，虽然 SQLAlchemy 2.0 现在也具有使用 Declarative 基类的 dataclasses 集成能力。

.. _dataclass: https://docs.python.org/3/library/dataclasses.html
.. _dataclasses: https://docs.python.org/3/library/dataclasses.html
.. _attrs: https://pypi.org/project/attrs/