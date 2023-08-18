.. highlight:: pycon+sql
.. |prev| replace:: :doc:`select`
.. |next| replace:: :doc:`dml`

.. include:: queryguide_nav_include.rst

.. doctest-include _inheritance_setup.rst

.. _inheritance_loading_toplevel:


.. currentmodule:: sqlalchemy.orm

.. _loading_joined_inheritance:

编写继承映射的SELECT语句
==================================================

.. admonition:: 关于此文档

    本文档使用了：使用 :ref:`ORM Inheritance <inheritance_toplevel>` 功能来配置的 ORM 映射，
    描述于 :ref:`inheritance_toplevel`。笔者将重点介绍 :ref:`joined_inheritance` ，因为这是最复杂的 ORM 查询案例。

    :doc:`查看此页面的 ORM 设置 <_inheritance_setup>`。

从基类或特定子类中进行查询
--------------------------------------------------------

在查询连接的继承层次结构中的一个类的情况下，SELECT语句将针对类映射到的表以及任何超级表（如果存在），使用JOIN将它们链接在一起。因此，查询将返回所请求类型的对象以及请求类型的任何子类型，使用每行中的 :term:`discriminator` 值来确定正确的类型。下面的查询建立在 ``Manager`` 子类的 ``Employee`` 上，然后返回一个结果，其中仅包含类型为Manager的对象::

    >>> from sqlalchemy import select
    >>> stmt = select(Manager).order_by(Manager.id)
    >>> managers = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT manager.id, employee.id AS id_1, employee.name, employee.type, employee.company_id, manager.manager_name
    FROM employee JOIN manager ON employee.id = manager.id ORDER BY manager.id
    [...] ()
    {stop}>>> print(managers)
    [Manager('Mr. Krabs')]

..  配置代码，不显示


    >>> session.close()
    ROLLBACK

当SELECT语句针对继承层次结构中的基类时，默认情况下，只有该类的表将包括在生成的SQL中，而JOIN不会被使用。在所有情况下， :term:`discriminator` 列用于区分不同的请求子类型，这会导致返回任何可能的子类型的对象。返回的对象将具有对应基表的属性，子表对应属性的状态将从未加载状态开始，访问时自动加载。加载子属性是可配置的，可以以多种方式更加 "eager" 进行，稍后在本节中讨论。

下面的示例创建一个针对 ``Employee`` 超类的查询。这表明结果集中可能包含任何类型的对象，包括 ``Manager``、``Engineer`` 和 ``Employee`` 。

    >>> from sqlalchemy import select
    >>> stmt = select(Employee).order_by(Employee.id)
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT employee.id, employee.name, employee.type, employee.company_id
    FROM employee ORDER BY employee.id
    [...] ()
    {stop}>>> print(objects)
    [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]

在上面的语句中， ``manager`` 和 ``engineer`` 的其他表未包含在SELECT中，这意味着返回的对象将不包含代表来自这些表的数据，在本示例中是 ``.manager_name`` 属性和 ``.engineer_info`` 属性。这些属性的状态从未加载状态开始，并在访问时自动填充。如果加载的数据很多，则惰性加载并不合适，因为在使用 N+1 问题 时，这会导致每行额外的SQL被访问子类特定的属性。这会影响性能，还与使用 :ref:`asyncio <asyncio_toplevel>` 的方法不兼容。另外，在我们的 ``Employee`` 对象的查询中，由于查询针对的是基表，我们没有办法以 ``Manager`` 或 ``Engineer`` 的术语添加涉及子类特定属性的SQL条件。接下来的两个部分将详细介绍两个构造，它们以不同的方式提供解决这两个问题的解决方案：使用 :func:`_orm.selectin_polymorphic`的加载程序选项 和使用 :func:`_orm.with_polymorphic` 实体构造。

.. _polymorphic_selectin:

使用 selectin_polymorphic() 
----------------------------

为了解决访问子类时的性能问题， :func:`_orm.selectin_polymorphic`加载器选项可用于提前一次性 :term:`eagerly load` 许多对象的额外属性。此加载选项的工作方式类似于 :func:`_orm.selectinload` 关系加载器策略，它对于接收到的行将发出针对子表的额外SELECT语句，使用主键基于WHERE IN查询其他行。

:func:`_orm.selectinload`接受查询的基本实体，后跟应为其加载特定属性的该实体的子类所组成的序列::

    >>> from sqlalchemy.orm import selectin_polymorphic
    >>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])

然后将 :func:`_orm.selectin_polymorphic` 用作加载选项，将其传递到 :class:`.Select` 的: meth:`.Select.options` 方法。示例说明使用 :func:`_orm.selectin_polymorphic` 来立即加载本地对于 ``Manager`` 和 ``Engineer`` 子类的列::

    >>> from sqlalchemy.orm import selectin_polymorphic
    >>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
    >>> stmt = select(Employee).order_by(Employee.id).options(loader_opt)
    >>> objects = session.scalars(stmt).all()
    {execsql}BEGIN (implicit)
    SELECT employee.id, employee.name, employee.type, employee.company_id FROM employee ORDER BY employee.id
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

上面的示例说明，为了及时获取额外的属性，发出了两个额外的SELECT语句，例如 ``Engineer.engineer_info`` ，以及 ``Manager.manager_name`` 。我们现在可以在加载的对象上访问这些子属性，而不必发出任何其他的SQL语句::

    >>> print(objects[0].manager_name)
    Eugene H. Krabs

.. tip:: :func:`_orm.selectin_polymorphic` 加载器选项还未优化基表 ``employee`` 不需要包含到第二个“eager load”查询中，因此在上面的例子中我们看到从 ``employee`` 到 ``manager`` 和 ``engineer`` 的 JOIN，尽管已经加载了 ``employee`` 的列。这与 :func:`_orm.selectinload` 关系策略形成对比，在这种情况下，后者更加复杂，可以在不需要时删除JOIN。


.. _polymorphic_selectin_w_loader_options:

将其他加载器选项与 selectin_polymorphic() 子类加载项结合使用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于 :func:`_orm.selectin_polymorphic` 生成的SELECT语句本身是ORM语句，因此我们还可以添加其他加载器选项（例如那些在 :ref:`orm_queryguide_relationship_loaders` 中记录的选择器）来更具体地加载子类。例如，如果我们认为 ``Manager`` 映射有一个 :ref:`one to many <relationship_patterns_o2m>` 关系，引用一个名为 ``Paperwork`` 的实体，我们可以结合使用 :func:`_orm.selectin_polymorphic` 和 :func:`_orm.selectinload` 来在所有 ``Manager`` 对象上急切地加载此集合。当情况如此，子属性的加载是特别“急切”的，示例如下::

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
    SELECT employee.id, employee.name, employee.type, employee.company_id FROM employee ORDER BY employee.id
    [...] ()
    SELECT manager.id AS manager_id, employee.id AS employee_id, employee.type AS employee_type,
    manager.manager_name AS manager_manager_name
    FROM employee JOIN manager ON employee.id = manager.id
    WHERE employee.id IN (?) ORDER BY employee.id
    [...] (1,)
    SELECT paperwork.manager_id AS paperwork_manager_id, paperwork.id AS paperwork_id,
    paperwork.document_name AS paperwork_document_name
    FROM paperwork WHERE paperwork.manager_id IN (?)
    [...] (1,)
    SELECT engineer.id AS engineer_id,
    employee.id AS employee_id, employee.type AS employee_type,
    engineer.engineer_info AS engineer_engineer_info
    FROM employee JOIN engineer ON employee.id = engineer.id
    WHERE employee.id IN (?, ?) ORDER BY employee.id
    [...] (2, 3)
    {stop}>>> print(objects[0])
    Manager('Mr. Krabs')
    >>> print(objects[0].paperwork)
    [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]

.. _polymorphic_selectin_as_loader_option_target:

将 selectin_polymorphic() 应用于现有急切加载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了能够在 :func:`_orm.selectin_polymorphic` 加载之后添加加载器选项之外，我们还可以指示将 :func:`_orm.selectin_polymorphic` 应用于现有加载器的目标。在我们的 :doc:`setup <_inheritance_setup>` 映射中包括一个名为 ``Company`` 的父实体，其 ``Company.employees`` :func:`_orm.relationship` 引用 ``Employee`` 实体，我们可以将SELECT针对 ``公司`` 实体的查询的所有 ``Employee`` 对象及其子类型的所有属性立即加载，如下所示，使用 :meth:`.Load.selectin_polymorphic` 作为一个链接的加载器选项； 在这种形式中，第一个参数是由前面的加载选项（在这种情况下为 :func:`_orm.selectinload` ）隐含地表示的，因此我们只指定我们希望加载的附加目标子类::

    >>> stmt = select(Company).options(
    ...     selectinload(Company.employees).selectin_polymorphic([Manager, Engineer])
    ... )
    >>> for company in session.scalars(stmt):
    ...     print(f"company: {company.name}")
    ...     print(f"employees: {company.employees}")
    {execsql}SELECT company.id, company.name FROM company ORDER BY company.id
    [...] ()
    SELECT employee.id AS employee_id, employee.name AS employee_name,
    employee.type AS employee_type, employee.company_id AS employee_company_id,
    manager.id AS id_1, manager.manager_name AS manager_manager_name
    FROM employee JOIN manager ON employee.id = manager.id
    WHERE employee.company_id IN (?)
    [...] (1,)
    SELECT paperwork.manager_id AS paperwork_manager_id, paperwork.id AS paperwork_id,
    paperwork.document_name AS paperwork_document_name
    FROM paperwork WHERE paperwork.manager_id IN (?)
    [...] (1,)
    SELECT engineer.id AS engineer_id, employee.id AS employee_id,
    employee.type AS employee_type, engineer.engineer_info AS engineer_engineer_info,
    employee.company_id AS employee_company_id
    FROM employee JOIN engineer ON employee.id = engineer.id
    WHERE employee.company_id IN (?, ?) ORDER BY employee.id
    [...] (2, 3)
    {stop}company: Krusty Krab
    employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]

.. seealso::

    :ref:`eagerloading_polymorphic_subtypes` - 演示使用 :func:`_orm.with_polymorphic` 配置的等效示例。


将选择 selectin_polymorphic() 工具配置到映射器上
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:func:`_orm.selectin_polymorphic` 的行为可以在$特定映射器上进行配置，以便默认情况下进行，使用 :paramref:`.mapper.with_polymorphic` 参数，在基类上进行配置或使用每个子类的 :paramref:`_orm.Mapper.polymorphic_load` 参数，传递值 ``"selectin"`` 。

.. warning::

   对于加入的映射，更喜欢在查询中显式使用
   :func:`_orm.with_polymorphic` 或在 :paramref:`_orm.Mapper.polymorphic_load` 中使用值为 ``"selectin"`` 的方式进行隐式子类加载，而不是在本节中描述的映射器级：paramref:`.mapper.with_polymorphic` 参数的方法。此参数会调用旨在重写SELECT语句中的FROM子句的复杂启发式方法，可能会干扰构造更复杂的语句，特别是对于引用相同映射实体的嵌套子查询。


例如，我们可以在以下情况下表示我们的“Employee”映射，使用 ``"inline"``：

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

在上述映射中，对于 ``Employee`` 类型的SELECT语句将自动在发出语句时使用 ``with_polymorphic(Employee, [Engineer, Manager])`` 作为主实体。::

    print(select(Employee))
    {printsql}SELECT employee.id, employee.name, employee.type, engineer.id AS id_1,
    engineer.engineer_info, manager.id AS id_2, manager.manager_name
    FROM employee
    LEFT OUTER JOIN engineer ON employee.id = engineer.id
    LEFT OUTER JOIN manager ON employee.id = manager.id

当使用映射器级“with polymorphic”时，查询还可以直接引用子类实体，其中它们隐式地表示多态查询中的连接表。在上述示例中，我们可以自由地引用 ``Manager`` 和 ``Engineer`` 直接返回默认 ``Employee`` 实体::

    print(
        select(Employee).where(
            or_(Manager.manager_name == "x", Engineer.engineer_info == "y")
        )
    )
```{printsql}SELECT employee.id, employee.name, employee.type, engineer.id AS id_1,
engineer.engineer_info, manager.id AS id_2, manager.manager_name FROM employee 
LEFT OUTER JOIN engineer ON employee.id = engineer.id 
LEFT OUTER JOIN manager ON employee.id = manager.id 
WHERE manager.manager_name = :manager_name_1 OR engineer.engineer_info = :engineer_info_1```

然而，如果我们需要在不同的别名上下文中引用“Employee”实体或其子实体，我们将再次直接使用:func：`_orm.with_polymorphic`来定义这些别名实体，如在:ref：`with_polymorphic_aliasing`中所示。

为了更集中地控制多态可选项，可使用更遗留的映射器级多态控制形式，即配置在基类上的：paramref：`_orm.Mapper.with_polymorphic`参数。此参数接受与:func：`_orm.with_polymorphic`构造可比较的参数，但使用联接继承映射的常见用法是纯星号，表示应LEFT OUTER JOIN所有子表，如下所示：

.. sourcecode :: python

    类Employee（基类）：
        __tablename__ =“employee”
        id =映射列（整数，primary_key = True）
        name =映射列（字符串（50））
        type =映射列（字符串（50））

        __mapper_args__ = {
            “polymorphic_identity”：“employee”，
            “with_polymorphic”：“*”，
            “polymorphic_on”：type，
        }


    类工程师（员工）：
        __tablename__ =“engineer”
        id =映射列（整数，ForeignKey（“employee.id”），primary_key = True）
        engineer_info =映射列（字符串（30））

        __mapper_args__ = {
            “polymorphic_identity”：“engineer”，
        }


    类经理（员工）：
        __tablename__ =“manager”
        id =映射列（整数，ForeignKey（“employee.id”），primary_key = True）
        manager_name =映射列（字符串（30））

        __mapper_args__ = {
            “polymorphic_identity”：“manager”，
        }

总的来说，：func：`_orm.with_polymorphic`使用的LEFT OUTER JOIN格式和选项（例如：paramref：`_orm.Mapper.with_polymorphic`）在SQL和数据库优化器方面可能很繁琐；对于加载已加入继承映射中的子类属性，应优先使用：func：`_orm.selectin_polymorphic`方法或将其映射器级等效设置为：paramref：`_orm.Mapper.polymorphic_load`为“selectin”，仅在需要每个查询时根据需要使用:func：`_orm.with_polymorphic`。

.. _inheritance_of_type:

加入特定的子类型或带多态()的实体
-------------------------------------------------- ------------

由于:func：`_orm.with_polymorphic`实体是:func：`_orm.aliased`的一个特殊情况，因此为了将多态实体作为联接的目标处理，特别是在使用: 唐：`_orm.relationship`构造作为ON子句时，使用与常规别名相同的技术如下清楚地列出了在:ref： `orm_queryguide_joining_relationships_aliased`中。 最简洁的方法是使用:meth：`_orm.PropComparator.of_type`。 在下面的示例中，我们展示了从父“Company”实体沿“Company.employees”的一对多关系连接，后者在:doc帮助下进行配置<_inheritance_setup>`链接到“Employee”对象， 并使用:func：`_orm.with_polymorphic`实体作为目标::

    ＆gt;＆gt;＆gt; employee_plus_engineer = with_polymorphic（Employee，[Engineer]）
    ＆gt;＆gt;＆gt; stmt = (
    ...     select（Company.name，employee_plus_engineer.name）
    ...     .join（Company.employees.of_type（employee_plus_engineer））
    ...     .where（
    ...         or_（
    ...             employee_plus_engineer.name ==“SpongeBob”，
    ...             employee_plus_engineer.Engineer.engineer_info
    ...             ==“Senior Customer Engagement Engineer”，
    ...         ）
    ...     ）
    ... ）
    ＆gt;＆gt;＆gt; for company_name，emp_name in session.execute（stmt）：
    ...     print（f“ {company_name} {emp_name}”）
    {execsql}SELECT company.name，employee.name AS name_1 FROM company JOIN （employee LEFT OUTER JOIN engineer ON employee.id = engineer.id）ON company.id = employee.company_id WHERE employee.name =？或engineer.engineer_info =？
    [...]（'SpongeBob'，'Senior Customer Engagement Engineer'）
    {stop}Krusty Krab SpongeBob
    Krusty Krab Squidward

更直接的是，: meth：`_orm.PropComparator.of_type`也用于任何类型的继承映射，以将对: func：`_orm.relationship`的联接限制为其目标的特定子类型。 上面的查询可以严格按照以下方式编写：作为“工程师”目标::

    ＆gt;＆gt;＆gt; stmt = (
    ...     select（Company.name，Engineer.name）
    ...     .join（Company.employees.of_type（Engineer））
    ...     .where（
    ...         or_（
    ...             Engineer.name ==“SpongeBob”，
    ...             Engineer.engineer_info ==“高级客户参与工程师”，
    ...         ）
    ...     ）
    ... ）
    ＆gt;＆gt;＆gt; for company_name，emp_name in session.execute（stmt）：
    ...     print（f“ {company_name} {emp_name}”）
    {execsql}SELECT company.name，employee.name AS name_1 FROM company JOIN （employee JOIN engineer ON employee.id = engineer.id）ON company.id = employee.company_id WHERE employee.name =？或engineer.engineer_info =？
    [...]（'SpongeBob'，'Senior Customer Engagement Engineer'）
    {stop}Krusty Krab SpongeBob
    Krusty Krab Squidward

可以观察到以上，“工程师”目标的直接加入使用内部JOIN，而不是LEFT OUTER JOIN，SQL优化器通常会从SQL优化器角度考虑，更具性能。

.. _eagerloading_polymorphic_subtypes:

通过急切地加载多态子类型来

可使用:meth：`_orm.PropComparator.of_type`与继承映射的任何类型等效地将多态实体应用到:ref：`relationship loader options <orm_queryguide_relationship_loaders>`，例如:func： `_orm.selectinload`和:func：`_orm.joinedload`。

例如，如果我们想要加载“Company”对象，并使用：func：`_orm.with_polymorphic`构造对整个层次结构进行快速加载，我们可以写成::

    ＆gt;＆gt;＆gt; all_employees = with_polymorphic（Employee，“*”）
    ＆gt;＆gt;＆gt; stmt = select（Company）。options（selectinload（Company.employees.of_type（all_employees）））
    ＆gt;＆gt;＆gt; for company in session.scalars（stmt）：
    ...     print（f“company：{company.name}”）
    ...     print（f“employees：{company.employees}”）
    {execsql}SELECT company.id，company.name FROM company
    [...]（）
    SELECT employee.company_id AS employee_company_id，employee.id AS employee_id，employee.name AS employee_name，employee.type AS employee_type，manager.id AS manager_id，manager.manager_name AS manager_manager_name，engineer.id AS engineer_id，engineer.engineer_info AS engineer_engineer_info FROM employee LEFT OUTER JOIN manager ON employee.id = manager.id LEFT OUTER JOIN engineer ON employee.id = engineer.id WHERE employee.company_id IN（？）
    [...]（1，）
    company：Krusty Krab
    employees：[Manager（'Mr. Krabs'），Engineer（'SpongeBob'），Engineer（'Squidward'）]

可以将上述查询直接与在前一节:ref：`polymorphic_selectin_as_loader_option_target`中说明的使用: func: `_orm.selectin_polymorphic`的版本进行比较。

.. seealso ::

    :ref：`polymorphic_selectin_as_loader_option_target`-举例说明了
    与上述相同的示例，使用：func：`_orm.selectin_polymorphic`代替


.. _loading_single_inheritance：

用于单继承映射的SELECT语句
-------------------------------------------------- -------------

..设置代码，不显示

    >>> session.close（）
    ROLLBACK
    >>> conn.close（）

.. doctest-include_single_inheritance.rst

.. warning :: 单表继承设置

    这一部分讨论了单表继承，
    描述在：ref：`single_inheritance`中，使用单个表表示层次结构中的多个类。

    :doc：`查看此部分的ORM设置 <_single_inheritance>`。

与加入继承映射不同，单继承映射的SELECT语句的构建趋向于更简单，因为对于全单继承层次结构，只有一个表格。

无论继承层次结构是否全部为单继承，查询基类与子类之间的查询的SELECT语句均可通过添加附加的WHERE条件来进行区分。

例如，将加载类型为“Manager”，“Engineer”和“Employee”的单继承示例映射为“Employee”的查询将使用表的简单SELECT::

    >>> stmt = select（Employee）.order_by（Employee.id）
    >>> for obj in session.scalars（stmt）：
    ...     print（f“{obj}”）
    {execsql}BEGIN（implicit）
    SELECT employee.id，employee.name，employee.type
    FROM employee ORDER BY employee.id
    [...]（）
    {stop}Manager（'Mr. Krabs'）
    Engineer（'SpongeBob'）
    Engineer（'Squidward'）

针对特定的子类发出负载时，可能会向SELECT添加其他条件以限制行，例如下面针对“工程师”实体执行的SELECT：¶

    >>> stmt = select（Engineer）.order_by（Engineer.id）
    >>> objects = session.scalars（stmt）.all（）
    {execsql}SELECT employee.id，employee.name，employee.type，employee.engineer_info
    FROM employee
    WHERE employee.type IN（？）ORDER BY employee.id
    [...]（'engineer'，）
    {stop}>>> for obj in objects：
    ...     print（f“{obj}”）
    Engineer（'SpongeBob'）
    Engineer（'Squidward'）

单继承映射的默认行为与加入继承相同，即默认情况下仍通过第二个SELECT发出特定于子类的属性。 在下面的示例中，加载类型为“Manager”的单个“Employee”对象，但由于请求的类是“Employee”，因此默认情况下不包含“Manager.manager_name”属性，当它被访问时发出一个附加的SELECT链接::

    >>> mr_krabs = session.scalars（select（Employee）。where（Employee.name ==“Mr. Krabs”））。one（）
    {execsql}BEGIN（implicit）
    SELECT employee.id，employee.name，employee.type
    FROM employee
    WHERE employee.name =？
    [...] （'Mr. Krabs'，）
    {stop}>>> mr_krabs.manager_name
    {execsql}SELECT employee.manager_name AS employee_manager_name
    FROM employee
    WHERE employee.id =？ 和employee.type in（？）
    [...] （1，'manager'）
    {stop}'Eugene H. Krabs'

要更改此行为，与加入继承加载中使用的相同的常规概念适用于单继承，包括使用:func：_orm选择in_polymorphic选项以及使用:func：_orm.with_polymorphic选项，后者仅包括附加列，并且从SQL的角度来看更加有效效为：

    >>> employees = with_polymorphic（Employee，“*”）
    >>> stmt = select（employees）。order_by（employees.id）
    >>> objects = session.scalars（stmt）.all（）
    {execsql}BEGIN（implicit）
    SELECT employee.id，employee.name，employee.type，
    employee.manager_name，employee.engineer_info
    FROM employee ORDER BY employee.id
    [...]（）
    {stop}>>> for obj in objects：
    ...     print（f“{obj}””）
    Manager（'Mr. Krabs'）
    Engineer（'SpongeBob'）
    Engineer（'Squidward'）
    >>> objects [0] .manager_name
    'Eugene H. Krabs'

由于加载单继承子类映射的开销通常很小，因此建议对单继承映射包括:paramref：`_orm.Mapper.polymorphic_load`参数，具有“inline”设置那些子类，其中预期其特定子类属性的加载是常见的。 下面的示例说明:doc：`设置 <_single_inheritance>`，修改以包含此选项。

    >>> class Base（DeclarativeBase）：
    ...    过去
    >>> class Employee（Base）：
    ...     __tablename__ =“employee”
    ...     id：Mapped [int] = mapped_column（primary_key = True）
    ...     name：Mapped [str]
    ...     type：Mapped [str]
    ...
    ...     def __repr__（self）：
    ...         return f“{self.__class__._name__}（{self.name！r}）”
    ...
    ...     __mapper_args__ = {
    ...         “polymorphic_identity”：“employee”，
    ...         “polymorphic_on”：“type”，
    ...     }
    >>> class Manager（Employee）：
    ...     manager_name：Mapped [str] = mapped_column（nullable = True）
    ...     __mapper_args__ = {
    ...         “polymorphic_identity”：“manager”，
    ...         “polymorphic_load”：“inline”，
    ...     }
    >>> class Engineer（Employee）：
    ...     engineer_info：Mapped [str] = mapped_column（nullable = True）
    ...     __mapper_args__ = {
    ...         “polymorphic_identity”：“engineer”，
    ...         “polymorphic_load”：“inline”，
    ...     }

以上映射表明，“Manager”和“Engineer”类的列将自动包含在针对“ Employee”实体的SELECT语句中：

    >>> print（select（Employee））
    {printsql}SELECT employee.id，employee.name，employee.type，
    employee.manager_name，employee.engineer_info
    FROM employee

继承加载API
-----------------------

.. autofunction :: sqlalchemy.orm.with_polymorphic

.. autofunction :: sqlalchemy.orm.selectin_polymorphic


..  Setup code, not for display

    >>> session.close（）
    ROLLBACK
    >>> conn.close（）