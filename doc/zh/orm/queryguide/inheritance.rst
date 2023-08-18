.. highlight:: pycon+sql
.. |prev| replace::  :doc:`select` 
.. |next| replace::  :doc:`dml` 

.. include:: queryguide_nav_include.rst

.. doctest-include _inheritance_setup.rst

.. _inheritance_loading_toplevel:


.. currentmodule:: sqlalchemy.orm

.. _loading_joined_inheritance:

继承映射中编写SELECT语句
=========================

.. admonition::关于本文件

    本环节使用了使用 ORM 继承特性配置的 ORM 映射详见   :ref:`inheritance_toplevel` , 因为它是最复杂的 ORM 查询案例。详见  :doc:` 查看本页的ORM设置 <_inheritance_setup>` 。

从基类和特定子类中选择
--------------------------------------------------------

一个加入继承层次结构的类的 SELECT 语句将会查询它所映射的表，以及查询可能涉及到的表格，使用 JOIN 将这些表连接起来。然后使用每行中的  :term:`discriminator`  来确定正确的类型，该查询将返回请求类型及其所有的子类型。下面的查询建立在 ` Manager` 类的子类之上，然后将返回仅包含 `Manager` 类型对象的结果::

    >>> from sqlalchemy import select
    >>> stmt = select(Manager).order_by(Manager.id)
    >>> managers = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT manager.id, employee.id AS id_1, employee.name, employee.type, employee.company_id, manager.manager_name
    FROM employee JOIN manager ON employee.id = manager.id ORDER BY manager.id
    [...] ()
    {stop}>>> print(managers)
    [Manager('Mr. Krabs')]

..  用于展示的设置代码


    >>> session.close()
    ROLLBACK

当 SELECT 子句针对继承层次结构中的基类时，默认行为是仅包含该类的表，并且不使用 JOIN。在所有情况下，将使用  :term:`discriminator`  列来区分不同的子类型，这将导致返回任何可能的子类型对象。返回的对象将会填充对应基表属性的特征，并且子表属性的特征将从未加载到已自动加载状态。可以按照各种方式更具体地配置子属性的加载状态，这将在本节后面讨论。

以下示例创建了针对 `Employee` 超类的查询，这表示结果集中可能包含任何类型的对象，包括 `Manager`, `Engineer` 和 `Employee`。

    >>> from sqlalchemy import select
    >>> stmt = select(Employee).order_by(Employee.id)
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT employee.id, employee.name, employee.type, employee.company_id
    FROM employee ORDER BY employee.id
    [...] ()
    {stop}>>> print(objects)
    [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]

上面的查询并未包含 `Manager` 和 `Engineer` 的附加表，这意味着返回的对象将不包含表示这些表的数据，例如 `Manager` 类的 `.manager_name` 属性和 `Engineer` 类的 `.engineer_info` 属性。这些属性以已过期的状态开始，当首次访问时，会自动填充自己，使用  :term:`lazy loading`  功能加载，该功能通过访问时自动加载来实现。

如果已加载大量对象，则此惰性加载行为是不可行的，在消费应用程序需要访问子类特定属性的情况下，这将是产生额外 SQL 查询的  :term:`N plus one`  问题。这将导致性能下降，并且对诸如使用   :ref:` asyncio <asyncio_toplevel>`  的做法也不兼容。此外，在我们针对 `Employee` 对象的查询中，由于查询只针对基表，我们无法通过子类特定属性添加 SQL 标准，例如 `Manager` 或 `Engineer`。下面的两个部分详细介绍了两种解决这两个问题的实现方法，即   :func:`_orm.selectin_polymorphic`  加载器选项和   :func:` _orm.with_polymorphic`  实体结构。


.. _polymorphic_selectin:

使用 selectin_polymorphic()
----------------------------

..  用于展示的设置代码


    >>> session.close()
    ROLLBACK

为了解决访问子类特定属性时的性能问题，可以使用   :func:`_orm.selectin_polymorphic`  加载策略来  :term:` 急加载`  跨许多对象一起提前加载附加属性。此加载器选项的工作方式与  :func:`_orm.selectinload` ` IN``根据主键查询附加行。

 :func:`_orm.selectinload` 将其参数作为正在查询的基本实体，然后是该实体的一个子类序列，用于加载其特定属性以供传入行使用：

    >>> from sqlalchemy.orm import selectin_polymorphic
    >>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])

然后使用  :func:`_orm.selectin_polymorphic` .Select` 的  :meth:`.Select.options`  方法。该示例说明了如何使用  :func:` _orm.selectin_polymorphic` `Manager``和``Engineer``子类的本地列：

    >>> from sqlalchemy.orm import selectin_polymorphic
    >>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
    >>> stmt = select(Employee).order_by(Employee.id).options(loader_opt)
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT employee.id, employee.name, employee.type, employee.company_id
    FROM employee ORDER BY employee.id
    [...] ()
    SELECT manager.id AS manager_id, employee.id AS employee_id,
    employee.type AS employee_type, manager.manager_name AS manager_manager_name
    FROM employee JOIN manager ON employee.id = manager.id
    WHERE employee.id IN (?) ORDER BY employee.id
    [...] (1,)
    SELECT engineer.id AS engineer_id, employee.id AS employee_id,
    employee.type AS employee_type, engineer.engineer_info AS engineer_engineer_info
    FROM employee JOIN engineer ON employee.id = engineer.id
    WHERE employee.id IN (?, ?) ORDER BY employee.id
    [...] (2, 3)
    {stop}>>> print(objects)
    [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]

上面的示例说明了发出两个额外的SELECT语句以急性地获取其他属性，例如``Engineer.engineer_info``和``Manager.manager_name``。现在，我们可以在加载的对象上访问这些子属性，而不需要发出任何其他SQL语句：

    >>> print(objects[0].manager_name)
    Eugene H. Krabs

.. tip::   :func:`_orm.selectin_polymorphic` ` employee``表无需包含在后两个“急性加载”查询中的事实。因此，在上面的示例中，尽管已经加载了``employee``的列，但我们看到了从``employee``到``manager``和``engineer``的JOIN。这与：func:`_orm.selectinload`关系策略有所不同，后者在这方面更为复杂，并且可以在不需要时将JOIN因素化。

.. _polymorphic_selectin_w_loader_options:

将其他加载器选项与selectin_polymorphic()子类加载器结合使用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  设置代码，不用显示


    >>> session.close()
    ROLLBACK

由  :func:`_orm.selectin_polymorphic` ` Manager``映射器有到名为``Paperwork``的实体的  :ref:`one to many <relationship_patterns_o2m>` ` Manager``对象上急性加载此集合，其中``Manager``对象的子属性也本身已经急性加载：

    >>> from sqlalchemy.orm import selectinload
    >>> from sqlalchemy.orm import selectin_polymorphic
    >>> stmt = (
    ...     select(Employee)
    ...     .order_by(Employee.id)
    ...     .options(
    ...         selectin_polymorphic(Employee, [Manager, Engineer]),
    ...         selectinload(Manager.paperwork),
    ...     )
    ... )
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT employee.id, employee.name, employee.type, employee.company_id
    FROM employee ORDER BY employee.id
    [...] ()
    SELECT manager.id AS manager_id, employee.id AS employee_id, employee.type AS employee_type, manager.manager_name AS manager_manager_name
    FROM employee JOIN manager ON employee.id = manager.id
    WHERE employee.id IN (?) ORDER BY employee.id
    [...] (1,)
    SELECT paperwork.manager_id AS paperwork_manager_id, paperwork.id AS paperwork_id, paperwork.document_name AS paperwork_document_name
    FROM paperwork
    WHERE paperwork.manager_id IN (?)
    [...] (1,)
    SELECT engineer.id AS engineer_id, employee.id AS employee_id, employee.type AS employee_type, engineer.engineer_info AS engineer_engineer_info
    FROM employee JOIN engineer ON employee.id = engineer.id
    WHERE employee.id IN (?, ?) ORDER BY employee.id
    [...] (2, 3)
    {stop}>>> print(objects[0])
    Manager('Mr. Krabs')
    >>> print(objects[0].paperwork)
    [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]

.. _polymorphic_selectin_as_loader_option_target:

将selectin_polymorphic()应用于现有的急性加载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了能够在  :func:`_orm.selectin_polymorphic` 。
由于我们的  :doc:`设置 <_inheritance_setup>`  映射包括一个父级
``Company`` 实体和一个引用 ``Employee`` 实体的 ``Company.employees``   :func:`_orm.relationship` ,
我们可以演示一个针对 ``Company`` 实体的 SELECT 请求，它将急切地加载所有的 ``Employee`` 对象以及
它们子类型的所有属性，如下所示使用  :meth:`.Load.selectin_polymorphic`  作为链式加载器选项；在这种情况下，第一个参数是从
前面的加载器选项（在这种情况下为   :func:`_orm.selectinload` ）隐含的，因此，我们只指示我们希望加载的附加目标子类::

    >>> stmt = select(Company).options(
    ...     selectinload(Company.employees).selectin_polymorphic([Manager, Engineer])
    ... )
    >>> for company in session.scalars(stmt):
    ...     print(f"company: {company.name}")
    ...     print(f"employees: {company.employees}")
    {execsql}SELECT company.id, company.name
    FROM company
    [...] ()
    SELECT employee.company_id AS employee_company_id, employee.id AS employee_id,
    employee.name AS employee_name, employee.type AS employee_type
    FROM employee
    WHERE employee.company_id IN (?)
    [...] (1,)
    SELECT manager.id AS manager_id, employee.id AS employee_id, employee.name AS employee_name,
    employee.type AS employee_type, employee.company_id AS employee_company_id,
    manager.manager_name AS manager_manager_name
    FROM employee JOIN manager ON employee.id = manager.id
    WHERE employee.id IN (?) ORDER BY employee.id
    [...] (1,)
    SELECT engineer.id AS engineer_id, employee.id AS employee_id, employee.name AS employee_name,
    employee.type AS employee_type, employee.company_id AS employee_company_id,
    engineer.engineer_info AS engineer_engineer_info
    FROM employee JOIN engineer ON employee.id = engineer.id
    WHERE employee.id IN (?, ?) ORDER BY employee.id
    [...] (2, 3)
    {stop}company: Krusty Krab
    employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]

.. seealso::

      :ref:`eagerloading_polymorphic_subtypes`  - 用   :func:` _orm.with_polymorphic`  相当于上面的例子


在映射器上配置 selectin_polymorphic()
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以在特定映射器上配置   :func:`_orm.selectin_polymorphic`  的行为，以便在默认情况下执行，方法是使用
  :paramref:`_orm.Mapper.polymorphic_load`   参数，在 per-subclass 的基础上使用值 ` `"selectin"``。下面的示例说明了
在 ``Engineer`` 和 ``Manager`` 子类中使用此参数的方法: 

.. sourcecode:: python

    class Employee(Base):
        __tablename__ = "employee"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        type = mapped_column(String(50))

        __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
        engineer_info = mapped_column(String(30))

        __mapper_args__ = {
            "polymorphic_load": "selectin",
            "polymorphic_identity": "engineer",
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
        manager_name = mapped_column(String(30))

        __mapper_args__ = {
            "polymorphic_load": "selectin",
            "polymorphic_identity": "manager",
        }

使用上述映射，在 SELECT 语句使用 ``Employee`` 类时，当发出语句时，将自动假定使用``selectin_polymorphic(Employee, [Manager, Engineer])`` 为加载器选项。

.. _with_polymorphic:

使用 with_polymorphic()
------------------------

与   :func:`_orm.selectin_polymorphic`  相反，后者仅影响对象的加载方式，  :func:` _orm.with_polymorphic` 
构造则影响多态结构的 SQL 查询如何呈现，通常作为到每个包含的子表的 LEFT OUTER JOIN
系列。这个连接结构被称为**多态可选择项**。通过为多个子表提供视图，  :func:`_orm.with_polymorphic`  提供了一种在一个 SELECT 语句中同时
写入几个继承类的方法，并能基于单个子表添加过滤条件。

  :func:`_orm.with_polymorphic`  实质上是   :func:` _orm.aliased`  构造的一种特殊形式。它接受一个类似于   :func:`_orm.selectin_polymorphic` ` 的参数形式
，即进行查询的基础实体，之后是该实体的一系列子类，它们的特定属性应该为传入行加载::

    >>> from sqlalchemy.orm import with_polymorphic
    >>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])

为了表示所有子类都应该成为实体的一部分，  :func:`_orm.with_polymorphic`  还将接受字符串 ` `"*"``，
可以代替类的序列，以表示所有类（请注意，这尚未得到   :func:`_orm.selectin_polymorphic`  的支持）::

    >>> employee_poly = with_polymorphic(Employee, "*")下面的示例演示了与上一节中演示的相同操作，一次加载所有“Manager”和“Engineer”的列：

    >>> stmt = select(employee_poly).order_by(employee_poly.id)
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT employee.id, employee.name, employee.type, employee.company_id,
    manager.id AS id_1, manager.manager_name, engineer.id AS id_2, engineer.engineer_info
    FROM employee
    LEFT OUTER JOIN manager ON employee.id = manager.id
    LEFT OUTER JOIN engineer ON employee.id = engineer.id ORDER BY employee.id
    [...] ()
    {stop}>>> print(objects)
    [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]

与   :func:`_orm.selectin_polymorphic`  的情况一样，子类的属性已经加载：

    >>> print(objects[0].manager_name)
    Eugene H. Krabs

由于   :func:`_orm.with_polymorphic`  生成的默认可选使用了 LEFT OUTER JOIN，从数据库的角度来看，不像   :func:` _orm.selectin_polymorphic`  方法那样优化查询，每个表只使用 JOIN 语句。

.. _with_polymorphic_subclass_attributes:

使用 with_polymorphic() 过滤子类的属性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通过包含允许对子类进行引用的命名空间，  :func:`_orm.with_polymorphic`  构造可用于包含子类映射器上的属性。在上一节中创建的 ` `employee_poly`` 构造包括名为 ``.Engineer`` 和 ``.Manager`` 的属性，这些属性提供了关于多态 SELECT 中 ``Engineer`` 和 ``Manager`` 的命名空间。在下面的示例中，我们可以使用   :func:`_sql.or_`  构造同时创建两个类的条件：

    >>> from sqlalchemy import or_
    >>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])
    >>> stmt = (
    ...     select(employee_poly)
    ...     .where(
    ...         or_(
    ...             employee_poly.Manager.manager_name == "Eugene H. Krabs",
    ...             employee_poly.Engineer.engineer_info
    ...             == "Senior Customer Engagement Engineer",
    ...         )
    ...     )
    ...     .order_by(employee_poly.id)
    ... )
    >>> objects = session.scalars(stmt).all()
    {execsql}SELECT employee.id, employee.name, employee.type, employee.company_id, manager.id AS id_1,
    manager.manager_name, engineer.id AS id_2, engineer.engineer_info
    FROM employee
    LEFT OUTER JOIN manager ON employee.id = manager.id
    LEFT OUTER JOIN engineer ON employee.id = engineer.id
    WHERE manager.manager_name = ? OR engineer.engineer_info = ?
    ORDER BY employee.id
    [...] ('Eugene H. Krabs', 'Senior Customer Engagement Engineer')
    {stop}>>> print(objects)
    [Manager('Mr. Krabs'), Engineer('Squidward')]

.. _with_polymorphic_aliasing:

使用 with_polymorphic 进行别名处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :func:`_orm.with_polymorphic`  构造作为   :func:` _orm.aliased`  的特殊情况，也提供了   :func:`_orm.aliased`  的基本功能，即对多态可选进行别名处理。具体来说，这意味着一个语句中可以同时使用两个或多个引用相同类层次结构的   :func:` _orm.with_polymorphic`  实体。要在具有联接继承映射的情况下使用此功能，通常需要传递两个参数，即  :paramref:`_orm.with_polymorphic.aliased`  和  :paramref:` _orm.with_polymorphic.flat` 。  :paramref:`_orm.with_polymorphic.aliased`   参数表示多态可选应该通过唯一于此构造体的别名名称引用。  :paramref:` _orm.with_polymorphic.flat`  参数适用于默认的 LEFT OUTER JOIN 多态可选，表示应在语句中使用更优化的别名处理形式。

为了说明这个特点，下面的示例发出了一个 SELECT 查询，分别使用 ``Employee`` 和 ``Engineer`` 加入的 polymorphic 实体，以及加入 ``Employee`` 和 ``Manager`` 的 polymorphic 实体。由于这两个 polymorphic 实体都将基本的「employee」表包含在它们的 polymorphic 可选中，因此必须应用别名来区分该表在其两个不同的上下文中。这两个 polymorphic 实体被视为两个单独的表，因此通常需要以某种方式将这些实体彼此连接，如下面的示例所示，其中按照 ``company_id`` 列将这两个实体与某些附加限制条件一起进行连接：

    >>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True, flat=True)
    >>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True, flat=True)
    >>> stmt = (
    ...     select(manager_employee, engineer_employee)
    ...     .join(
    ...         engineer_employee,
    ...         engineer_employee.company_id == manager_employee.company_id,
    ...     )
    ...     .where(
    ...         or_(
    ...             manager_employee.name == "Mr. Krabs",
    ...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
    ...         )
    ...     )
    ...     .order_by(engineer_employee.name, manager_employee.name)
    ... )
    >>> for manager, engineer in session.execute(stmt):
    ...     print(f"{manager} {engineer}")
    {execsql}SELECT
    employee_1.id, employee_1.name, employee_1.type, employee_1.company_id,
    manager_1.id AS id_1, manager_1.manager_name,
    employee_2.id AS id_2, employee_2.name AS name_1, employee_2.type AS type_1,
    employee_2.company_id AS company_id_1, engineer_1.id AS id_3, engineer_1.engineer_info
    FROM employee AS employee_1
    LEFT OUTER JOIN manager AS manager_1 ON employee_1.id = manager_1.id
    JOIN在上面的例子中，  :paramref:`_orm.with_polymorphic.flat`   的行为是保持多态可选择的表作为它们各自表的左外连接，这些表本身被赋予匿名别名。还产生了一个右侧嵌套的 JOIN。

当省略  :paramref:`_orm.with_polymorphic.flat`  参数时，通常的行为是每个多态可选择的表格都包含在一个子查询中，从而产生一个更冗长的形式：

::

    >>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True)
    >>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True)
    >>> stmt = (
    ...     select(manager_employee, engineer_employee)
    ...     .join(
    ...         engineer_employee,
    ...         engineer_employee.company_id == manager_employee.company_id,
    ...     )
    ...     .where(
    ...         or_(
    ...             manager_employee.name == "Mr. Krabs",
    ...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
    ...         )
    ...     )
    ...     .order_by(engineer_employee.name, manager_employee.name)
    ... )
    >>> print(stmt)
    {printsql}SELECT anon_1.employee_id, anon_1.employee_name, anon_1.employee_type,
    anon_1.employee_company_id, anon_1.manager_id, anon_1.manager_manager_name, anon_2.employee_id AS employee_id_1,
    anon_2.employee_name AS employee_name_1, anon_2.employee_type AS employee_type_1,
    anon_2.employee_company_id AS employee_company_id_1, anon_2.engineer_id, anon_2.engineer_engineer_info
    FROM
    (SELECT employee.id AS employee_id, employee.name AS employee_name, employee.type AS employee_type,
    employee.company_id AS employee_company_id,
    manager.id AS manager_id, manager.manager_name AS manager_manager_name
    FROM employee LEFT OUTER JOIN manager ON employee.id = manager.id) AS anon_1
    JOIN
    (SELECT employee.id AS employee_id, employee.name AS employee_name, employee.type AS employee_type,
    employee.company_id AS employee_company_id, engineer.id AS engineer_id, engineer.engineer_info AS engineer_engineer_info
    FROM employee LEFT OUTER JOIN engineer ON employee.id = engineer.id) AS anon_2
    ON anon_2.employee_company_id = anon_1.employee_company_id
    WHERE anon_1.employee_name = :employee_name_2 OR anon_1.manager_manager_name = :manager_manager_name_1
    ORDER BY anon_2.employee_name, anon_1.employee_name

历史上，以上形式在不一定具有支持右嵌套 JOIN 的后端上更具可移植性，而且当使用   :ref:`具体的表继承 <concrete_inheritance>`  映射以及一般情况下使用替代多态可选择表格时，也可能是适当的选择。

.. _with_polymorphic_mapper_config:

在映射器上配置 with_polymorphic()
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

与   :func:`_orm.selectin_polymorphic`  一样，  :func:` _orm.with_polymorphic`  还支持在基类上使用  :paramref:`.mapper.with_polymorphic`  参数的映射器配置版本，或者以更现代的形式在每个子类上使用  :paramref:` _orm.Mapper.polymorphic_load`  参数，在这里传递值 ``"inline"``。

.. warning::

   对于连接的继承映射，建议在查询中显式使用   :func:`_orm.with_polymorphic` ，或者在使用  :paramref:` _orm.Mapper.polymorphic_load`  时使用 ``"selectin"`` 进行隐式急切子类加载，而不是使用在本节中描述的映射器级  :paramref:`.mapper.with_polymorphic`  参数。该参数调用复杂的启发式算法，旨在重写 SELECT 语句中的 FROM 子句，可能会干扰构造更复杂的语句，特别是那些引用相同映射实体的嵌套子查询。

例如，我们可以使用如下所示的  :paramref:`_orm.Mapper.polymorphic_load`  将 ` `Employee`` 映射声明为 ``"inline"``：

.. sourcecode:: python

    class Employee(Base):
        __tablename__ = "employee"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        type = mapped_column(String(50))

        __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
        engineer_info = mapped_column(String(30))

        __mapper_args__ = {
            "polymorphic_load": "inline",
            "polymorphic_identity": "engineer",
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
        manager_name = mapped_column(String(30))

        __mapper_args__ = {
            "polymorphic_load": "inline",
            "polymorphic_identity": "manager",
        }

使用上面的映射，在针对 ``Employee`` 类的 SELECT 语句发出时，将自动假定使用 ``with_polymorphic(Employee, [Engineer, Manager])`` 作为主实体。

::

    print(select(Employee))
    {printsql}SELECT employee.id, employee.name, employee.type, engineer.id AS id_1,
    engineer.engineer_info, manager.id AS id_2, manager.manager_name
    FROM employee
    LEFT OUTER JOIN engineer ON employee.id = engineer.id
    LEFT OUTER JOIN manager ON employee.id = manager.id

使用映射器级“with polymorphic”，查询也可以直接引用子类实体，其中它们隐含地表示多态查询中的连接表格。例如，我们可以自由地针对默认的 ``Employee`` 实体直接引用 ``Manager`` 和 ``Engineer``。如果我们需要在单独的别名上下文中引用``Employee``实体或它的子实体，我们需要再次直接使用   :func:`_orm.with_polymorphic`  将这些别名实体定义为所示的方式。在   :ref:` with_polymorphic_aliasing`  中。

如果需要更加集中地控制多态可选择的内容，可以使用更遗留的形式映射器级多态控制，即基类上配置的  :paramref:`_orm.Mapper.with_polymorphic`  参数。该参数接受与   :func:` _orm.with_polymorphic`  构建相当的参数，但是使用连接继承映射的常见方法是使用平面的星号，表示所有子表都应左外连接，如下：

.. sourcecode:: python

    class Employee(Base):
        __tablename__ = "employee"
        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50))
        type = mapped_column(String(50))

        __mapper_args__ = {
            "polymorphic_identity": "employee",
            "with_polymorphic": "*",
            "polymorphic_on": type,
        }


    class Engineer(Employee):
        __tablename__ = "engineer"
        id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
        engineer_info = mapped_column(String(30))

        __mapper_args__ = {
            "polymorphic_identity": "engineer",
        }


    class Manager(Employee):
        __tablename__ = "manager"
        id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
        manager_name = mapped_column(String(30))

        __mapper_args__ = {
            "polymorphic_identity": "manager",
        }

总的来说，   :func:`_orm.with_polymorphic`  和  :paramref:` _orm.Mapper.with_polymorphic`  使用的 LEFT OUTER JOIN 格式可能在 SQL 和数据库优化器方面非常麻烦；对于连接继承映射中的子类属性的普遍加载，应该首选   :func:`_orm.selectin_polymorphic`  方法，或者其映射器级别的等效设置  :paramref:` _orm.Mapper.polymorphic_load`  为 ``"selectin"``，仅在需要时按需在每个查询上使用   :func:`_orm.with_polymorphic`  。

.. _inheritance_of_type:

连接到特定子类型或 with_polymorphic() 实体
------------------------------------------------------------

作为   :func:`_orm.with_polymorphic`  实体是   :func:` _orm.aliased`  的一种特殊情况，为了将多态实体作为一个连接的目标，在使用   :func:`_orm.relationship`  构造作为 ON 子句时，
我们需要使用与详细说明了正常别名的相同技术，详见   :ref:`orm_queryguide_joining_relationships_aliased` ，最简单的方法是使用  :meth:` _orm.PropComparator.of_type` 。
在下面的示例中，我们演示了从父级 ``Company`` 实体沿着一对多关系 ``Company.employees`` 加入的方式，这在  :doc:`setup <_inheritance_setup>`  中配置，以链接到 ` `Employee`` 对象，使用   :func:`_orm.with_polymorphic`  实体作为目标：


    >>> employee_plus_engineer = with_polymorphic(Employee, [Engineer])
    >>> stmt = (
    ...     select(Company.name, employee_plus_engineer.name)
    ...     .join(Company.employees.of_type(employee_plus_engineer))
    ...     .where(
    ...         or_(
    ...             employee_plus_engineer.name == "SpongeBob",
    ...             employee_plus_engineer.Engineer.engineer_info
    ...             == "Senior Customer Engagement Engineer",
    ...         )
    ...     )
    ... )
    >>> for company_name, emp_name in session.execute(stmt):
    ...     print(f"{company_name} {emp_name}")
    {execsql}SELECT company.name, employee.name AS name_1
    FROM company JOIN (employee LEFT OUTER JOIN engineer ON employee.id = engineer.id) ON company.id = employee.company_id
    WHERE employee.name = ? OR engineer.engineer_info = ?
    [...] ('SpongeBob', 'Senior Customer Engagement Engineer')
    {stop}Krusty Krab SpongeBob
    Krusty Krab Squidward

更直接地，  :meth:`_orm.PropComparator.of_type`  也用于任何类型的继承映射，以将连接限定在   :func:` _orm.relationship`  的目标的特定子类型上。上面的查询可以按如下方式严格编写为“引擎师”目标：


    >>> stmt = (
    ...     select(Company.name, Engineer.name)
    ...     .join(Company.employees.of_type(Engineer))
    ...     .where(
    ...         or_(
    ...             Engineer.name == "SpongeBob",
    ...             Engineer.engineer_info == "Senior Customer Engagement Engineer",
    ...         )
    ...     )
    ... )
    >>> for company_name, emp_name in session.execute(stmt):
    ...     print(f"{company_name} {emp_name}")
    {execsql}SELECT company.name, employee.name AS name_1
    FROM company JOIN (employee JOIN engineer ON employee.id = engineer.id) ON company.id = employee.company_id
    WHERE employee.name = ? OR engineer.engineer_info = ?
    [...] ('SpongeBob', 'Senior Customer Engagement Engineer')
    {stop}Krusty Krab SpongeBob
    Krusty Krab Squidward如上所示，与“多态可选择对象” ``with_polymorphic(Employee，[Engineer])`` 相比，
直接加入到 ``Engineer`` 目标具有使用内部 JOIN 而不是 LEFT OUTER JOIN 的有用特性，
从 SQL 优化器的角度来看，通常更有效。

.. _eagerloading_polymorphic_subtypes:

多态子类型的急加载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在前一节中，使用  :meth:`_orm.PropComparator.of_type`  和  :meth:` .Select.join` 
方法演示可以等效地应用到   :ref:`relationship loader options <orm_queryguide_relationship_loaders>` 
中，例如   :func:`_orm.selectinload`  和   :func:` _orm.joinedload` 。

例如，如果我们希望加载 ``Company`` 对象，并且另外希望使用 ``_orm.with_polymorphic`` 构建充分的层次结构
来急切地加载 ``Company.employees`` 的所有元素，则可以编写如下代码：

```
>>> all_employees = with_polymorphic(Employee, "*")
>>> stmt = select(Company).options(selectinload(Company.employees.of_type(all_employees)))
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
{execsql}SELECT company.id, company.name
FROM company
[...] ()
SELECT employee.company_id AS employee_company_id, employee.id AS employee_id,
employee.name AS employee_name, employee.type AS employee_type, manager.id AS manager_id,
manager.manager_name AS manager_manager_name, engineer.id AS engineer_id,
engineer.engineer_info AS engineer_engineer_info
FROM employee
LEFT OUTER JOIN manager ON employee.id = manager.id
LEFT OUTER JOIN engineer ON employee.id = engineer.id
WHERE employee.company_id IN (?)
[...] (1,)
company: Krusty Krab
employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

以上查询可以直接与前一节   :ref:`polymorphic_selectin_as_loader_option_target`  中
演示的   :func:`_orm.selectin_polymorphic`  版本进行比较。

.. seealso::

     :ref:`polymorphic_selectin_as_loader_option_target`  - 演示了与上述相同的例子，但使用   :func:` _orm.selectin_polymorphic` 


.. _loading_single_inheritance:

单继承映射的 SELECT 语句
-------------------------------------------------

..  安装代码，请勿显示

    >>> session.close()
    ROLLBACK  # 回滚
    >>> conn.close()

.. doctest-include _single_inheritance.rst

.. 警告:: 单表继承设置

    本节讨论单表继承，参见   :ref:`single_inheritance` ，使用单个表表示层次结构中
    的多个类。

     :doc:`查看此部分的 ORM 设置 <_single_inheritance>` 。

与联接继承映射不同，对于单继承映射的 SELECT 语句构造通常更简单，因为
对于全单继承层次结构，只有一个表。

无论继承层次结构是否都是单继承还是混合使用联接继承和单继承，对于单继承来说，
基类与子类的不同查询仅通过限制 SELECT 语句的 WHERE 条件来区分。

例如，对 ``Employee`` 的单继承示例映射的查询将使用对表的简单 SELECT 来加载
``Manager``，``Engineer`` 和 ``Employee`` 类型的对象：

```
>>> stmt = select(Employee).order_by(Employee.id)
>>> for obj in session.scalars(stmt):
...     print(f"{obj}")
{execsql}BEGIN (implicit)
SELECT employee.id, employee.name, employee.type
FROM employee ORDER BY employee.id
[...] ()
{stop}Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
```

当发出特定子类的加载时，会向 SELECT 添加附加条件来限制行数，
例如以下是针对 ``Engineer`` 实体执行的 SELECT：

```
>>> stmt = select(Engineer).order_by(Engineer.id)
>>> objects = session.scalars(stmt).all()
{execsql}SELECT employee.id, employee.name, employee.type, employee.engineer_info
FROM employee
WHERE employee.type IN (?) ORDER BY employee.id
[...] ('engineer',)
{stop}>>> for obj in objects:
...     print(f"{obj}")
Engineer('SpongeBob')
Engineer('Squidward')
```

提高单继承属性的加载性能
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

..  安装代码，请勿显示

    >>> session.close()
    ROLLBACK

单继承映射对于如何选择子类上的属性的默认行为类似于联接继承，因此在
默认情况下，特定子类的属性仍然会发出第二个 SELECT。在下面的示例中，
加载单个类型为 ``Manager`` 的 ``Employee``，但由于请求的类是 ``Employee``，
因此默认情况下不包含属性 ``Manager.manager_name``，因此在访问该属性时会发出
另一个 SELECT：

```
>>> mr_krabs = session.scalars(select(Employee).where(Employee.name == "Mr. Krabs")).one()
{execsql}BEGIN (implicit)
SELECT employee.id, employee.name, employee.type
FROM employee
WHERE employee.name = ?
[...] ('Mr. Krabs',)
{stop}>>> mr_krabs.manager_name
{execsql}SELECT employee.manager_name AS employee_manager_name
FROM employee
```WHERE employee.id = ? AND employee.type IN (?)
    [...] (1, 'manager')
    {stop}'Eugene H. Krabs'

为了更改这种行为，使用在joined inheritance loading中使用的相同的常规概念也可以用于单一继承，包括使用   :func:`_orm.selectin_polymorphic`  选项以及   :func:` _orm.with_polymorphic`  选项，后者仅包括其他列并从SQL角度而言对于单一继承mapper来说更加高效：

    >>> employees = with_polymorphic(Employee, "*")
    >>> stmt = select(employees).order_by(employees.id)
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN（隐式）
    SELECT employee.id, employee.name, employee.type,
    employee.manager_name, employee.engineer_info
    FROM employee ORDER BY employee.id
    [...] ()
    {stop}>>> for obj in objects:
    ...     print(f"{obj}")
    Manager('Mr. Krabs')
    Engineer('SpongeBob')
    Engineer('Squidward')
    >>> objects[0].manager_name
    'Eugene H. Krabs'

由于加载单一继承子类映射的开销通常很小，因此建议单一继承映射包括  :paramref:`_orm.Mapper.polymorphic_load`  参数，并对那些预计常见加载其特定子类属性的子类设置为 ` `"inline"``。下面是演示  :doc:`设置 <_single_inheritance>`  的示例，其中包含此选项的修改版本：

    >>> class Base(DeclarativeBase):
    ...     pass
    >>> class Employee(Base):
    ...     __tablename__ = "employee"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str]
    ...     type: Mapped[str]
    ...
    ...     def __repr__(self):
    ...         return f"{self.__class__.__name__}({self.name!r})"
    ...
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "employee",
    ...         "polymorphic_on": "type",
    ...     }
    >>> class Manager(Employee):
    ...     manager_name: Mapped[str] = mapped_column(nullable=True)
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "manager",
    ...         "polymorphic_load": "inline",
    ...     }
    >>> class Engineer(Employee):
    ...     engineer_info: Mapped[str] = mapped_column(nullable=True)
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "engineer",
    ...         "polymorphic_load": "inline",
    ...     }


有了以上的映射，``Manager`` 和 ``Engineer`` 类将自动包含在针对 ``Employee`` 实体的 SELECT 语句中：

    >>> print(select(Employee))

继承加载API
-----------------------

.. autofunction:: sqlalchemy.orm.with_polymorphic

.. autofunction:: sqlalchemy.orm.selectin_polymorphic


..  设置代码，不要显示

    >>> session.close()
    ROLLBACK
    >>> conn.close()