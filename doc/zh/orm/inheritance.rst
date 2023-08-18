映射类继承层次结构
=====================================

SQLAlchemy支持三种继承形式：单表继承，其中一个表代表多种类形；具体的表继承，其中每种类型的类都由独立的表代表；以及联接表继承，其中类层次结构分布在相关表之间，每个类由自己的表代表，该表仅包含该类本地的那些属性。

单继承和联合继承是最常见的继承形式，而具体的继承则提供了更多的配置挑战。

当映射器在继承关系中配置时，SQLAlchemy有能力以多态的方式加载元素，意味着单个查询可以返回多种类型的对象。

.. seealso::

    :ref:`loading_joined_inheritance` - 描述 :ref:`queryguide_toplevel` 中的联接表

    :ref:`examples_inheritance` - 完整的联接、单体和具体继承示例

.. _joined_inheritance:

联接表继承
------------------------

在联接表继承中，沿着类的层次结构的每个类都由一个不同的表代表。查询特定子类时将呈现为沿其继承路径中的所有表连接的SQL JOIN。如果查询的类是基类，则将查询基表，同时包括包括在内的其他表的选项，或者允许稍后加载特定于子表的属性。

在所有情况下，对于给定行要实例化的最终类由基类上定义的多态鉴别器列或SQL表达式确定，该列将产生与特定子类相关联的标量值。


联接继承层次结构中的基类使用附加参数进行配置，以指示多态鉴别器列，以及可选的基类自身的多态标识符：

```
    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }

        def __repr__(self):
            return f"{self.__class__.__name__}({self.name!r})"
```

在上面的示例中，鉴别器是“类型”列，该列使用 :paramref:`_orm.Mapper.polymorphic_on` 参数进行配置。该参数接受面向列的表达式，可以指定为映射的属性的字符串名称，也可以指定为列表达式对象，例如 :class:`_schema.Column` 或 :func:`_orm.mapped_column` 构造。

鉴别器列将存储一个值，该值表示行内表示的对象的类型。该列可以是任何数据类型，但字符串和整数是最常见的。为数据库中特定行应用于此列的实际数据值是使用 :paramref:`_orm.Mapper.polymorphic_identity` 参数指定的，如下面所述。

虽然多态鉴别器表达式不严格必需，但如果想要多态加载，则需要此参数。在基表上建立一个列是实现此目的的最简单方法，然而，非常复杂的继承映射可以使用SQL表达式，例如CASE表达式，作为多态鉴别器。

.. note::

   目前，整个继承层次结构仅可为一个鉴别器列或SQL表达式进行配置，通常为继承层次结构中的基类。尚不支持“级联”多态鉴别器表达式。

接下来，我们定义 ``Engineer`` 和 ``Manager`` 的子类。每个子类都包含代表它所代表的子类的唯一属性的列。每个表还必须包含主键列（或多个主键列），以及对父表的外键引用：

```
    class Engineer(Employee):
        __tablename__ = "engineer"
        id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
        engineer_name: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
        manager_name: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }
```

在上面的示例中，每个映射都在其映射器参数中指定 :paramref:`_orm.Mapper.polymorphic_identity` 参数。此值填充由 :paramref:`_orm.Mapper.polymorphic_on` 参数在基础映射上指定的列。 :paramref:`_orm.Mapper.polymorphic_identity` 参数应在整个继承层次结构中为每个映射类唯一，并且每个映射类应只有一个“身份”; 如上所述，“级联”标识，在其中一些子类引入第二个标识的情况下，不受支持。

ORM使用由 :paramref:`_orm.Mapper.polymorphic_identity` 设置的值，以确定负责加载面向这些行的类。在上面的示例中，每个表示“Employee”的行都在其“type”列中具有值“employee”；类似地，每个“Engineer”将获得值“engineer”，每个“Manager”将获得值“manager”。无论继承映射是否像联接表继承中那样使用不同的联接表子类一样，在查询时都期望保存此值并为ORM在查询时可用。:paramref:`_orm.Mapper.polymorphic_identity` 参数也适用于具体表继承，但实际上没有保存；有关详细信息，请参见下面的 :ref:`concrete_inheritance` 部分。

在多态设置中，常见的做法是外键约束在与主键本身相同的列或列上，但这不是必需的；某个不同于主键本身的列也可以使“父表”引用某个列。构建从基表到子类的JOIN方式也是可以直接自定义的，但是这很少是必要的。

.. topic:: 联接继承主键

    联接表继承配置的一个自然效应是任何映射对象的标识都可以完全从基本表中的行中确定。这具有明显的优点，因此SQLAlchemy始终将联接继承类的主键列视为仅属于基本表。换句话说，``engineer``和``manager``表的``id``列不用于查找``Engineer``或``Manager``对象--只考虑``employee.id``中的值。当然，``engineer.id``和``manager.id``对于整个模式的适当操作仍然非常重要，因为一旦确定了父行，它们将用于定位联接行。

联接继承映射已经完成，请对``Employee``进行查询将返回``Employee``、``Engineer``和``Manager``对象的组合。在此情况下，新保存的``Engineer``、``Manager``和``Employee``对象将自动使用正确的"鉴别器"值填充``employee.type``列，即分别是``"engineer"``、``"manager"``和``"employee"``。


联接继承关系
+++++++++++++++++++++++++++++++++++++

联接表继承的关系完全得到支持。涉及联接继承类的关系应该针对在层次结构中对应于外键约束的类；下面，因为``employee``表有一个返回``company``表的外键约束，
关系是在``Company``和``Employee``之间建立的：

    from __future__ import annotations

    from sqlalchemy.orm import relationship


    class Company(Base):
        __tablename__ = "company"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        employees: Mapped[List[Employee]] = relationship(back_populates="company")


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]
        company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
        company: Mapped[Company] = relationship(back_populates="employees")

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }


    class Manager(Employee):
        ...


    class Engineer(Employee):
        ...

如果外键约束在对应于子类的表上，则关系应该指向该子类。在下面的示例中，从经理到公司的外键约束，因此关系应该在``Manager``和``Company``之间建立：

    class Company(Base):
        __tablename__ = "company"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        managers: Mapped[List[Manager]] = relationship(back_populates="company")


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
        manager_name: Mapped[str]

        company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
        company: Mapped[Company] = relationship(back_populates="managers")

        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }


    class Engineer(Employee):
        ...

上面，``Manager``类将具有``Manager.company``属性；``Company``将具有始终针对``employee``和``manager``表一起加载的``Company.managers``属性。

载入联接继承映射
+++++++++++++++++++++++++++++++++++

有关继承加载技术的背景信息，请参见 :ref:`inheritance_loading_toplevel` 部分，其中包括两次配置以及查询和多层抽象的映射的信息。单表和联接表继承的大部分加载技术都是相同的，因此提供了很高的抽象度，使得可以轻松地在这两种映射类型之间切换，以及在单个层次结构中混合使用它们（只需省略``__tablename__``就可以使子类单一继承）。有关继承加载技术的文档，请参见 :ref:`loading_single_inheritance` 部分和 :ref:`inheritance_loading_toplevel` 部分。


单表继承
------------------------

单表继承将来自所有子类的所有属性都表示在单个表中。具有唯一属性的特定子类将在表中的列中保持它们，如果行引用其他类型的对象，则这些列默认为NULL。

查询特定层次结构中的类将呈现为针对基本表的SELECT，其中包括一个WHERE子句，该子句将行限制为鉴别器列或表达式中存在特定值或值。


通过一个示例理解如何更好地理解``Single Table Inheritance``的用法：

```
    class Employee(Base):
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_on": "type",
            "polymorphic_identity": "employee",
        }


    class Manager(Employee):
        manager_data: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }


    class Engineer(Employee):
        engineer_info: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }
```

上面的配置中，唯一不同的是没有表名。我们已经声明了所有的列，这使得ORM可以查询到每个具体类的实例。需要注意的是，在单表继承配置中，鉴别器列必须在基表中。

单表继承具有简单性优势，而不需要像联接表继承那样涉及多个表以加载对象。


.. _orm_inheritance_column_conflicts:

使用 use_existing_column 解决列冲突
+++++++++++++++++++++++++++++++++++++++++++++++++

下面的示例中，``manager_name``和``engineer_info``列被“提升”到应用于``Employee.__table__``，由于它们声明在没有唯一表的子类上。但是，当两个子类想要指定同一个列时，会出现棘手的情况，如下所示：

```
    from datetime import datetime


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }


    class Engineer(Employee):
        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }

        start_date: Mapped[datetime] = mapped_column(nullable=True)


    class Manager(Employee):
        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }

        start_date: Mapped[datetime] = mapped_column(nullable=True)

```

在上面的示例中，``Engineer``和``Manager``中声明的``start_date``列都将导致错误：

.. sourcecode:: text

    sqlalchemy.exc.ArgumentError: Column 'start_date' on class Manager conflicts
    with existing column 'employee.start_date'.  If using Declarative,
    consider using the use_existing_column parameter of mapped_column() to
    resolve conflicts.

上述情况对Declarative映射系统而言是不确定的，它可能通过使用 :paramref:`_orm.mapped_column.use_existing_column` 参数来使用所映射的父类上的列（如果已经存在）来解决此问题，否则将映射一个新的列 ::

```
    from sqlalchemy import DateTime


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_on": "type",
            "polymorphic_identity": "employee",
        }


    class Engineer(Employee):
        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }

        start_date: Mapped[datetime] = mapped_column(
            nullable=True, use_existing_column=True
        )


    class Manager(Employee):
        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }

        start_date: Mapped[datetime] = mapped_column(
            nullable=True, use_existing_column=True
        )
```

以上，当映射``Manager``时，``start_date``列在``Employee``类上已经出现，因为已经是``Engineer``映射提供的 By ``_.Orm.Mapped_Column``。 :paramref:`_orm.mapped_column.use_existing_column` 参数表示 :func:`_orm.mapped_column` 应首先在子类中查找所请求的 :class:`_schema.Column`，如果存在，则使用已经映射的列，否则 :func:`_orm.mapped_column` 将像往常一样映射该列，将其添加为由``Employee``超类引用的 :class:`_schema.Table` 中的列之一。

.. versionadded:: 2.0.0b4 - Added :paramref:`_orm.mapped_column.use_existing_column`,
   which provides a 2.0-compatible means of mapping a column on an inheriting
   subclass conditionally.  The previous approach which combines
   :class:`.declared_attr` with a lookup on the parent ``.__table__``
   continues to function as well, but lacks :pep:`484` typing support.  

可以使用混合类相似的概念（请参见 :ref:`orm_mixins_toplevel`）从可重用的混合类定义特定列和/或其他映射属性的一系列从中继承的子类：


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_on": type,
            "polymorphic_identity": "employee",
        }


    class HasStartDate:
        start_date: Mapped[datetime] = mapped_column(
            nullable=True, use_existing_column=True
        )


    class Engineer(HasStartDate, Employee):
        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }


    class Manager(HasStartDate, Employee):
        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }

单表继承关系
+++++++++++++++++++++++++++++++++++++++++++

单表继承的关系与联接继承完全得到支持。配置方式与联接继承相同；外键属性应该位于也对应于外键约束的类。下面，因为``employee``表有一个外键约束返回到``company``表，因此关系是在``Company``和``Employee``之间建立的：

    class Company(Base):
        __tablename__ = "company"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        employees: Mapped[List[Employee]] = relationship(back_populates="company")


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]
        company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
        company: Mapped[Company] = relationship(back_populates="employees")

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }


    class Manager(Employee):
        manager_data: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }


    class Engineer(Employee):
        engineer_info: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }

此外，我们还可以创建涉及特定子类的关系。在查询时，SELECT 语句将包括 WHERE 子句，以限制类选择为特定子类或子类：

    class Company(Base):
        __tablename__ = "company"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        managers: Mapped[List[Manager]] = relationship(back_populates="company")


    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }


    class Manager(Employee):
        manager_name: Mapped[str] = mapped_column(nullable=True)

        company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
        company: Mapped[Company] = relationship(back_populates="managers")

        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }


    class Engineer(Employee):
        engineer_info: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }

以上，``Manager``类将具有``Manager.company``属性；``Company``将具有始终针对带有额外 WHERE 子句的 “employee” 和 “manager” 表一起加载的``Company.managers``属性。

使用``polymorphic_abstract``建立更深层次的层级
+++++++++++++++++++++++++++++++++++++++++++++++++

.. versionadded:: 2.0

在构建任何继承层次结构时，映射的类可以包括设置为``True``的 :paramref:`_orm.Mapper.polymorphic_abstract` 参数，这表示应在映射该类的同时，不希望直接实例化该类，并且不会包含 :paramref:`_orm.Mapper.polymorphic_identity`。然后，子类可以被声明为该映射类的子类，并且它们本身可以包括:paramref:`_orm.Mapper.polymorphic_identity`，从而可以正常使用。这允许以一个通用的基类来引用一系列子类，该基类在基线层次结构中被视为“抽象”，无论是在查询中还是在 :func:`_orm.relationship` 声明中。此用法不同于使用 Declarative 中的 :ref:`declarative_abstract` 属性，Declarative 会完全映射目标类，并且因此无法单独使用映射类。 :paramref:`_orm.Mapper.polymorphic_abstract` 可以应用于继承层次结构中的任何一级类，包括一次在多个级别上批量应用。

例如，假设要将“Manager”和“Principal”都归类为“Executive”超类，并且“Engineer”和“Sysadmin”都归类为“Technologist”超类。``Technologist`` 或 ``Executive`` 都不会被直接实例化，因此它们没有 :paramref:`_orm.Mapper.polymorphic_identity`。可以如下配置：

```
    class Employee(Base):
        __tablename__ = "employee"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        type: Mapped[str]

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "polymorphic_on": "type",
        }


    class Executive(Employee):
        """An executive of the company"""

        executive_background: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {"polymorphic_abstract": True}


    class Technologist(Employee):
        """An employee who works with technology"""

        competencies: Mapped[str] = mapped_column(nullable=True)

        __mapper_args__ = {"polymorphic_abstract": True}


    class Manager(Executive):
        """a manager"""

        __mapper_args__ = {"polymorphic_identity": "manager"}


    class Principal(Executive):
        """a principal of the company"""

        __mapper_args__ = {"polymorphic_identity": "principal"}


    class Engineer(Technologist):
        """an engineer"""

        __mapper_args__ = {"polymorphic_identity": "engineer"}


    class SysAdmin(Technologist):
        """a systems administrator"""

        __mapper_args__ = {"polymorphic_identity": "engineer"}
```

在上面的示例中，“Technologist”和“Executive”是普通映射类，并且还指出要添加到超类上的新列``executive_background`` 和 ``competencies``。但是，它们都缺少：paramref:`_orm.Mapper.polymorphic_identity` 的设置，因为不希望直接实例化 Technologist 或 Executive；我们希望始终有一个 ``Manager``、 ``Principal``、 ``Engineer`` 或 ``SysAdmin``。但是，我们仍然可以查询活动人数并且可以将它们作为 :func:`_orm.relationship` 的目标。下面的示例演示了``Employe``e 对象的 SELECT 语句：


.. sourcecode:: python+sql

    session.scalars(select(Technologist)).all()
    {execsql}
    SELECT employee.id, employee.name, employee.type, employee.competencies
    FROM employee
    WHERE employee.type IN (?, ?)
    [...] ('engineer', 'sysadmin')

在上面的示例中，``Technologist`` 和 ``Executive`` 抽象映射类是通过声明的 ``Manager``、 ``Principal``、 ``Engineer`` 和 ``SysAdmin`` 这些子类来调用的。即使我们可以通过查询使用“``Technologist``”和“``Principal``”角色，也可以对 :func:`_orm.relationship`（例如，“Company.principals``”和``Company.technologists``）进行更改。


载入单表继承映射
+++++++++++++++++++++++++++++++++++

单表继承的载入技术与联接表继承的载入技术大部分相同，并且在两者之间提供了高度的抽象化，使得它们易于切换以及在单个继承层次结构中进行混合。有关继承加载技术的文档，请参见 :ref:`loading_single_inheritance` 部分和 :ref:`inheritance_loading_toplevel` 部分。仅返回该类的实例。通过在映射器中配置一个特殊的SELECT语句才能启用具体类的多态加载，通常这个SELECT语句是由所有表的UNION生成的。

.. warning::

    具体表继承比连接或单表继承**更加复杂**，功能方面**更加受限**，特别是与关系、急加载和多态加载一起使用时。如果需要灵活性进行关系加载和多态加载，建议使用连接或单表继承，如果有可能。如果不需要多态加载，则可以使用普通的非继承映射，如果每个类都完全引用自己的表。

虽然连接和单表继承熟练于"多态"加载，但在具体继承中，这是一个更尴尬的问题。因此，具体继承更适合不需要**多态加载**的场合。建立涉及具体继承类的关系也更加麻烦。

要将一个类作为使用具体继承，请在"__mapper_args__"中添加:paramref:`_orm.Mapper.concrete`参数。这表明，在Declarative以及映射中，超类表不应被视为映射的一部分::

    class Employee(Base):
        __tablename__ = "employee"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))


    class Manager(Employee):
        __tablename__ = "manager"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        manager_data = mapped_column(String(50))

        __mapper_args__ = {
            "concrete": True,
        }


    class Engineer(Employee):
        __tablename__ = "engineer"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        engineer_info = mapped_column(String(50))

        __mapper_args__ = {
            "concrete": True,
        }

必须注意两个关键点：

* 我们必须**明确定义**每个子类上的所有列，即使这些列名称相同。例如，在此处，像"Employee.name"一样的列并**不会**自动复制到"Manager"或"Engineer"表中。
* 虽然"Engineer"和"Manager"类在继承关系中被映射，但它们仍然**不包括多态加载**。这意味着，如果我们查询"Employee"对象，则"manager"和"engineer"表根本不会被查询。

.. _concrete_polymorphic:

具体多态加载配置
+++++++++++++++++++

使用具体继承进行多态加载需要针对每个应该具有多态加载的基类配置一个特殊的SELECT。这个SELECT需要能够访问所有的映射表，并且通常是由SQLAlchemy辅助函数:func:`.polymorphic_union`生成的一个UNION语句。

正如在:ref:`inheritance_loading_toplevel`所讨论的那样，任何类型的映射器继承配置都可以配置成从默认的特殊可选择性载入，并使用:paramref:`_orm.Mapper.with_polymorphic`参数。当前的公共API要求在首次构造映射器时设置此参数。

然而，在Declarative中，映射器和被映射的:class:`_schema.Table`同时创建，即在定义映射类的时候就完成了。这意味着:paramref:`_orm.Mapper.with_polymorphic`参数尚未提供，因为对应于子类的:class:`_schema.Table`对象尚未被定义。

有几种策略可用于解决此循环，但是Declarative提供了辅助类:class:`.ConcreteBase`和:class:`.AbstractConcreteBase`，可以在幕后处理此问题。

使用:class:`.ConcreteBase`，我们可以以几乎相同的方式设置具体映射，以设置其他形式的继承映射::

    from sqlalchemy.ext.declarative import ConcreteBase
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class Employee(ConcreteBase, Base):
        __tablename__ = "employee"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "concrete": True,
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        manager_data = mapped_column(String(40))

        __mapper_args__ = {
            "polymorphic_identity": "manager",
            "concrete": True,
        }


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        engineer_info = mapped_column(String(40))

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
            "concrete": True,
        }

上面，在映射器"initialization"时间，Declarative设置了``Employee``类的多态可选操作；这是对于那些要解决其他依赖映射器的映射器的后期配置步骤。助手:class:`.ConcreteBase`使用:func:`.polymorphic_union`函数在所有其他类设置好后创建了具体映射的表的一个UNION，然后在已存在的基类映射器上设置了这个语句。

在选择时，多态联合会生成如下查询：

.. sourcecode:: python+sql

    session.scalars(select(Employee)).all()
    {execsql}
    SELECT
        pjoin.id,
        pjoin.name,
        pjoin.type,
        pjoin.manager_data,
        pjoin.engineer_info
    FROM (
        SELECT
            employee.id AS id,
            employee.name AS name,
            CAST(NULL AS VARCHAR(40)) AS manager_data,
            CAST(NULL AS VARCHAR(40)) AS engineer_info,
            'employee' AS type
        FROM employee
        UNION ALL
        SELECT
            manager.id AS id,
            manager.name AS name,
            manager.manager_data AS manager_data,
            CAST(NULL AS VARCHAR(40)) AS engineer_info,
            'manager' AS type
        FROM manager
        UNION ALL
        SELECT
            engineer.id AS id,
            engineer.name AS name,
            CAST(NULL AS VARCHAR(40)) AS manager_data,
            engineer.engineer_info AS engineer_info,
            'engineer' AS type
        FROM engineer
    ) AS pjoin

上面的UNION查询需要为每个子表制造"NULL"列，以适应不是该特定子类成员的列。

.. seealso::

    :class:`.ConcreteBase`

.. _abstract_concrete_base:

具体抽象类
+++++++++++++

具体继承的映射通常将超类和子类都映射到单独的表中，而不映射基类。换句话说，基类是"抽象"的。

当希望将两个不同的子类映射到各自的表中，并留下基类没有映射的时候，这可以非常容易地实现。当使用Declarative时，只需使用"__abstract__"指示符声明基类即可:: 

    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class Employee(Base):
        __abstract__ = True


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        manager_data = mapped_column(String(40))


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        engineer_info = mapped_column(String(40))

上面，我们实际上没有使用SQLAlchemy的继承映射设施；我们可以正常地加载和持久化"Manager"和"Engineer"实例。然而，当我们需要**查询多态**时，也就是说，我们想发出"select(Employee)"并得到一个"Manager"和"Engineer"实例集合，那就意味着我们重新进入了具体继承的领域，我们必须构建一个特殊的映射器来针对"Employee"实现这一点。

为了将我们的具体继承示例修改为表示一个"抽象"基类，它具备多态加载功能，我们只有一个"engineer"表和一个"manager"表，而没有"employee"表，然而，映射器"Employee"将直接映射到"多态联合"，而不是将其本地指定为:paramref:`_orm.Mapper.with_polymorphic`参数。

为了帮助处理这个问题，Declarative提供了一个变量:class:`.AbstractConcreteBase`，它会自动执行此操作::

    from sqlalchemy.ext.declarative import AbstractConcreteBase
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class Employee(AbstractConcreteBase, Base):
        strict_attrs = True

        name = mapped_column(String(50))


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        manager_data = mapped_column(String(40))

        __mapper_args__ = {
            "polymorphic_identity": "manager",
            "concrete": True,
        }


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        engineer_info = mapped_column(String(40))

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
            "concrete": True,
        }


    Base.registry.configure()

上述映射使用:meth:`_orm.registry.configure`方法，触发了实际映射"Employee"类，因为在配置步骤之前，该类没有映射，因为它将从未定义的任何子表中查询出来。使用上述映射，只有"Manager"和"Engineer"实例可以被持久化；针对"Employee"类的查询将始终产生"Manager"和"Engineer"对象。使用上述映射，可以使用"Employee"类及其直接声明的任何属性，例如"Employee.name"构建查询：

.. sourcecode:: pycon+sql

    >>> stmt = select(Employee).where(Employee.name == "n1")
    >>> print(stmt)
    {printsql}SELECT pjoin.id, pjoin.name, pjoin.type, pjoin.manager_data, pjoin.engineer_info
    FROM (
      SELECT engineer.id AS id, engineer.name AS name, engineer.engineer_info AS engineer_info,
      CAST(NULL AS VARCHAR(40)) AS manager_data, 'engineer' AS type
      FROM engineer
      UNION ALL
      SELECT manager.id AS id, manager.name AS name, CAST(NULL AS VARCHAR(40)) AS engineer_info,
      manager.manager_data AS manager_data, 'manager' AS type
      FROM manager
    ) AS pjoin
    WHERE pjoin.name = :name_1

:paramref:`.AbstractConcreteBase.strict_attrs`参数表示"Employee"类应直接映射仅在"Employee"类本地声明的属性，例如"Employee.name"属性。诸如"Manager.manager_data"和"Engineer.engineer_info"之类的其他属性仅存在于相应的子类。当未设置:paramref:`.AbstractConcreteBase.strict_attrs`时，那么在整个层次结构中，包括"Manager.manager_data"和"Engineer.engineer_info"在内的所有子类属性都会映射到基"Employee"类。这是一种遗留的使用模式，可能更便于查询，但是其效果是整个层次结构中的所有子类共享完整的属性集；在上述例子中，不使用:paramref:`.AbstractConcreteBase.strict_attrs`将导致生成非有用的"Engineer.manager_name"和"Manager.engineer_info"属性。

.. versionadded:: 2.0  添加:paramref:`.AbstractConcreteBase.strict_attrs`参数到:class:`.AbstractConcreteBase`，它会生成更简洁的映射；默认值为False，以便允许早期1.x版本的遗留映射继续按其原样工作。

.. seealso::

    :class:`.AbstractConcreteBase`


与具体继承的关系
+++++++++++++++++++

在具体继承方案中，映射关系很具有挑战性，因为不同的类没有共享的表。如果关系只涉及特定的类，例如在我们之前的示例中的"Company"和"Manager"之间的关系，这些就只是两个相关的表。

但是，如果"Company"想要与"Employee"建立一对多关系，则表明该集合可以包括"Engineer"和"Manager"对象，就意味着"Employee"必须具有多态加载能力，并且每个要关联的表都必须具有一个回到"company"表的外键。以下是这种情况的一个示例配置::

    from sqlalchemy.ext.declarative import ConcreteBase


    class Company(Base):
        __tablename__ = "company"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        employees = relationship("Employee")


    class Employee(ConcreteBase, Base):
        __tablename__ = "employee"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        company_id = mapped_column(ForeignKey("company.id"))

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "concrete": True,
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        manager_data = mapped_column(String(40))
        company_id = mapped_column(ForeignKey("company.id"))

        __mapper_args__ = {
            "polymorphic_identity": "manager",
            "concrete": True,
        }


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        engineer_info = mapped_column(String(40))
        company_id = mapped_column(ForeignKey("company.id"))

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
            "concrete": True,
        }

关于具体继承的下一个复杂度涉及当我们希望"Employee"、"Manager"和"Engineer"中的一个或所有对象也指回"Company"时。为了这种情况，SQLAlchemy在那些链接到"Employee"并且链接到"Company"的关系上有一个特殊的行为，当在实例级别上执行时，它**不起作用**。相反，必须向每个类应用不同的:func:`_orm.relationship`。为了实现三个关系的相反行为，它们都作为"Company.employee"的相反行为被使用:paramref:`_orm.relationship.back_populates`参数来引用：

    from sqlalchemy.ext.declarative import ConcreteBase


    class Company(Base):
        __tablename__ = "company"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        employees = relationship("Employee", back_populates="company")


    class Employee(ConcreteBase, Base):
        __tablename__ = "employee"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        company_id = mapped_column(ForeignKey("company.id"))
        company = relationship("Company", back_populates="employees")

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "concrete": True,
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        manager_data = mapped_column(String(40))
        company_id = mapped_column(ForeignKey("company.id"))
        company = relationship("Company", back_populates="employees")

        __mapper_args__ = {
            "polymorphic_identity": "manager",
            "concrete": True,
        }


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        engineer_info = mapped_column(String(40))
        company_id = mapped_column(ForeignKey("company.id"))
        company = relationship("Company", back_populates="employees")

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
            "concrete": True,
        }

上面的限制与当前实现有关，包括具体继承的类不共享父类的任何属性，因此需要设置不同的关系。

具体继承的加载
+++++++++++++++

使用具体继承的加载选项是有限的；通常情况下，如果映射的多态加载使用了Declarative具体的mixin之一，则无法在查询时修改其样式，在当前SQLAlchemy版本中不支持：func:`_orm.with_polymorphic`函数来覆盖使用的加载样式，因为当前还存在一些限制。