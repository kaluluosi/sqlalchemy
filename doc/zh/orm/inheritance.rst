.. _继承顶级:

映射类继承层次结构
=================

SQLAlchemy支持三种继承形式：单表继承，多表继承和联结表继承。其中，单表继承表示几种类型的类由单个表表示；多表继承表示每种类型的类都由独立的表表示，联结表继承则通过依赖表从中断开类层次结构，每个类都由自己的表表示，只包括本地类特定的属性。

最常见的继承形式是单表继承和联结表继承，而多表继承则提出了更多的配置挑战。

在映射器配置继承关系时，SQLAlchemy可以多态地加载元素，这意味着单个查询可返回多种类型的对象。

.. seealso::

      :ref:`loading_joined_inheritance` ——在   :ref:` queryguide_toplevel`  中查看

      :ref:`examples_inheritance` ——完整的联结，单个和具体的继承示例

.. _joined_inheritance:

联结表继承
------------------------

在联结表继承中，沿着类的层次结构，每个类都由一个不同的表表示。查询类中的一个特定子类将呈现为沿着其继承路径中的所有表进行SQL JOIN。如果查询的类是基类，则查询基表，同时有选项包括其他表，或者允许特定于子表的属性稍后加载。

在所有情况下，要为给定行实例化的最终类是由基类上定义的鉴别器列或SQL表达式确定的，其将产生与特定子类相关联的标量值。

联结继承层次结构中的基类配置有用于多态鉴别器列的附加参数，以及可选的基类本身的多态标识符：

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

在上面的示例中，鉴别器是 ``type`` 列，可使用  :paramref:`_orm.Mapper.polymorphic_on`  参数配置。此参数接受以列为导向的表达式，可以使用映射属性的值作为字符串名称，也可以使用如   :class:` _schema.Column`  或   :func:`_orm.mapped_column`  构造的列表达式对象。

鉴别器列将存储一个值，该值表示行内表示的对象类型。列可以是任何数据类型，但字符串和整数是最常见的。为特定行应用于此列的实际数据值是使用  :paramref:`_orm.Mapper.polymorphic_identity`  参数指定的，默认值如下。

虽然多态鉴别器表达式并不是必需的，但如果需要多态加载，则必需。建立基表上的列是实现此功能的最简单方法，但非常复杂的继承映射可能会使用SQL表达式，例如CASE表达式，作为多态鉴别器。

.. note::

   目前，“整个继承层次结构”通常意味着基类最底层的列配置有 **只有一个鉴别器列或SQL表达式** ，不支持“级联”多态鉴别器表达式。

接下来，我们定义 ``Engineer`` 和 ``Manager`` 分别作为 ``Employee`` 的子类。每个子类包含代表其表示的子类唯一属性的列。每个表也必须包含主键列（或多列），以及对父表的外键：

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

在上面的示例中，每个映射都在其映射器参数中指定  :paramref:`_orm.Mapper.polymorphic_identity`  参数。该值填充由  :paramref:` _orm.Mapper.polymorphic_on`  参数建立的列。  :paramref:`_orm.Mapper.polymorphic_identity`  参数应在整个层次结构中唯一，并且每个映射类应该只有一个“identity”；如上所述，不支持一些子类引入第二个标识的“级联”身份。

ORM使用由  :paramref:`_orm.Mapper.polymorphic_identity`  设置的值，以确定在多态方式下加载行时行所属的类。在上面的示例中，表示 ` `Employee`` 的每一行都将在其“type”列中包含值 ``'employee'``，同样的，每个 ``Engineer`` 将得到值 ``'engineer'``，每个 ``Manager`` 将得到值 ``'manager'``。不论继承映射是否像联结表继承一样使用不同的单独的表来表示子类，或者所有子类像单表继承一样使用相同的表，都期望将此值保留并在查询时向ORM提供。  :paramref:`_orm.Mapper.polymorphic_identity`  参数也适用于具体表继承，但实际上并没有持久化；请参见   :ref:` concrete_inheritance`  部分获取详细信息。

在多态设置中，最常见的情况是将外键约束建立在与主键本身相同的列或列上，但这不是必需的；也可以使用与主键不同的列来指示父项。从基表到子表构造 JOIN 的方式也是可以直接自定义的，只是这很少使用。

.. topic:: 联结继承主键

    联结表继承配置的一个自然结果是，任何映射对象的标识都可以完全从基表中的行中确定。这具有明显的优点，因此SQLAlchemy始终将联结继承类的主键列视为仅基表的主键列。换句话说， ``engineer`` 表和 ``manager`` 表的 ``id`` 列不用于定位 ``Engineer`` 或 ``Manager`` 对象-只考虑 ``employee.id`` 中的值。当然， ``engineer.id`` 和 ``manager.id`` 仍然对模式的正确操作至关重要，因为一旦在语句中确定了父行，它们将用于定位连接行。

联结继承映射完成后，针对“Employee”进行的查询将返回“Employee”、“Engineer”和“Manager”对象的组合。在这种情况下，新保存的“Engineer”、“Manager”和“Employee”对象将自动使用正确的“鉴别器”值填充“employee.type”列，即“'engineer'”、“'manager'”或“'employee'”。

联结继承中的关系
++++++++++++++++++++++++

与联结表继承一样，联结继承完全支持关系。涉及联结继承类的关系应将目标类定向到对应于外键约束的类；下面是相应的代码，因为 ``employee`` 表在单向具有外键约束指向 ``company`` 表：

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

如果外键约束在与子类相对应的表上，关系应定向到该子类。在下面的示例中，从 ``manager`` 到 ``company`` 有一个外键约束，因此关系应定向到 ``Manager`` 和 ``Company`` 类之间：

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

上述代码中，“Manager”类将具有 “Manager.company” 属性；“Company”则将具有一个始终针对 “employee” 和 “manager” 表一起加载的 “Company.managers” 属性。

联结继承映射的加载
++++++++++++++++++++++++

有关继承加载技术（包括在映射器配置时间以及查询时间配置要查询的表格）的背景信息，请参见   :ref:`inheritance_loading_toplevel` 。只返回该类的实例，并启用具有多态性类的多态加载，映射器内部配置特殊的SELECT，通常通过所有表的UNION生成。

.. warning:: 具体表继承的复杂程度远远高于连接或单表继承，特别是涉及使用涉及关系、急切加载和多态加载时，其功能极为有限。当用于多态时，它会生成UNION的非常大的查询，这些查询将不如简单连接执行。如果需要关系加载和多态加载的灵活性，则强烈建议尽可能使用连接或单表继承。如果不需要多态加载，则可以使用纯非继承映射，如果每个类都完全引用其自己的表。

相对于连接和单表继承，具体继承在“多态”加载中的应用不那么顺畅。因此，当不需要多态加载时，具体继承更为合适。在涉及具体继承类的关系时，建立关系也更加困难。

要将一个类添加为使用具体继承，请在“__mapper_args__”中添加  :paramref:`_orm.Mapper.concrete`  参数。这表明对于声明式和映射，都不应将超类表视为映射的一部分：

示例代码：

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

应据此注意两个关键点：

* 我们必须**在每个子类中明确定义所有列**，即使那些名称相同。例如，这里的“Employee.name”并**不会**被复制到由“Manager”或“Engineer”映射的表中。
* 虽然“Engineer”和“Manager”类在继承关系中被映射，但他们仍然**不具有多态加载**。也就是说，如果我们查询“Employee”对象，则“manager”和“engineer”表根本不会被查询。

.. _concrete_polymorphic:

具体多态加载配置
++++++++++++++++++++++++++++++

使用具体继承进行多态加载需要配置针对每个应该具有多态加载的基类的专用SELECT。该SELECT需要能够单独访问所有映射的表，通常是使用SQLAlchemy助手 :func:`.polymorphic_union` 构造的UNION语句。

如在  :ref:`inheritance_loading_toplevel`  参数默认配置为从特殊可选择载入。当前公共API要求必须在首次构建  :paramref:` _orm.Mapper`  时设置此参数。

然而，在使用声明性时，映射器和映射到的  :class:`_schema.Table` .ConcreteBase` 和 :class:`.AbstractConcreteBase` 辅助类，它们在幕后处理此问题。

使用  :class:`.ConcreteBase` ，我们可以以与其他形式的继承映射几乎相同的方式设置我们的具体映射：

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

上面，Declarative在“initialization”操作的映射器中设置了“Employee”类的多态可选择载入；此类映射器对于解决其他相关映射器的依赖关系是可执行配置步骤。  :class:`.ConcreteBase` .polymorphic_union` 函数为所有具体映射表创建了一个UNION，在创建好所有其他类之后配置该语句并将其应用于已存在的基类映射器。

在选择时，多态联合生成如下查询：

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

上面的UNION查询需要为每个子表制造“NULL”列，以适应那些不是特定子类成员的列。

相关链接：
      :class:`.ConcreteBase` 