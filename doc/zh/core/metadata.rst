==================================
使用MetaData描述数据库
==================================

.. module:: sqlalchemy.schema

本节讨论基本的  :class:`_schema.Table` 、 :class:` _schema.Column`和 :class:`_schema.MetaData` 对象。

.. seealso::

       :ref:`tutorial_working_with_metadata`  - 统一教程中介绍了SQLAlchemy数据库元数据概念的教程

许多不同的数据库（或多个数据库）描述存储在名为 :class:`~sqlalchemy.schema.MetaData` 的对象中：

    from sqlalchemy import MetaData

    metadata_obj = MetaData()

  :class:`~sqlalchemy.schema.MetaData`  是一个容器对象，其中保存了许多描述数据库的（或多个数据库的）不同特性。

要表示表，请使用  :class:`~sqlalchemy.schema.Table` ~sqlalchemy.schema.MetaData` 对象。其余的位置参数大多数都是描述每个列的 :class:`~sqlalchemy.schema.Column` 对象：

    from sqlalchemy import Table, Column, Integer, String

    user = Table(
        "user",
        metadata_obj,
        Column("user_id", Integer, primary_key=True),
        Column("user_name", String(16), nullable=False),
        Column("email_address", String(60)),
        Column("nickname", String(50), nullable=False),
    )

上述描述了一个名为“user”的表，其中包含四个列。表的主键由“user_id”列组成。可将多个列分配`primary_key=True`标志，该标志表示多列主键，称为*组合*主键。

请注意，每个列都使用与通用化类型对应的对象描述其数据类型，例如  :class:`~sqlalchemy.types.Integer` ~sqlalchemy.types.String` 。 SQLAlchemy具有数十种具有不同特定性级别的类型以及创建自定义类型的能力。类型系统的文档可以在   :ref:`types_toplevel`  中找到。

.. _metadata_tables_and_columns:

访问表和列
---------------------

  :class:`~sqlalchemy.schema.MetaData` ` sorted_tables``访问者，它以对外键依赖性的顺序返回每个 :class:`~sqlalchemy.schema.Table` 对象的列表，即，每个表前面都有它所引用的所有表：

    >>> for t in metadata_obj.sorted_tables:
    ...     print(t.name)
    user
    user_preference
    invoice
    invoice_item

在大多数情况下，已经显式声明了单个  :class:`~sqlalchemy.schema.Table` ~sqlalchemy.schema.Table` ，它就具有完整的访问器，在检查其特性时允许检查其所有属性。给定以下 :class:`~sqlalchemy.schema.Table` 定义：

    employees = Table(
        "employees",
        metadata_obj,
        Column("employee_id", Integer, primary_key=True),
        Column("employee_name", String(60), nullable=False),
        Column("employee_dept", Integer, ForeignKey("departments.department_id")),
    )

请注意，本表中使用的 :class:`~sqlalchemy.schema.ForeignKey` 对象 - 这个构造定义了对远程表的引用，并且在  :ref:`metadata_foreignkeys` 中完全描述。 访问该表信息的方法包括::

    # 访问“employee_id”列：
    employees.columns.employee_id

    # 或者：
    employees.c.employee_id

    # 通过字符串
    employees.c ["employee_id"]

    # 可以使用多个字符串返回一组列（新的2.0功能）
    emp_id，name，type = employees.c ["employee_id"、"name"、"type"]

    # 遍历所有列
    for c in employees.c:
        print(c)

    # 获取表的主键列
    for primary_key in employees.primary_key:
        print(primary_key)

    # 获取表的外键对象：
    for fkey in employees.foreign_keys:
        print(fkey)

    # 访问表的MetaData：
    employees.metadata

    # 访问列的名称、类型、可空性、主键、外键
    employees.c.employee_id.name
    employees.c.employee_id.type
    employees.c.employee_id.nullable
    employees.c.employee_id.primary_key
    employees.c.employee_dept.foreign_keys

    # 获取列的“键”，默认为其名称，但可以是任何用户定义字符串：
    employees.c.employee_name.key

    # 访问列的表：
    employees.c.employee_id.table是employees

    # 获取由外键关联的表
    列表（employees.c.employee_dept.foreign_keys）[0] .column.table

   

.. tip::

   :attr:`_sql.FromClause.c`  集合同义），是  :class:` _sql.ColumnCollection` `employees.c.employee_name``。但是，对于带有空格或与字典方法名称匹配的名称（例如  :meth:`_sql.ColumnCollection.keys`  或  :meth:` _sql.ColumnCollection.values`  ），必须使用索引访问，例如``employees.c ['values']``或``employees.c [ "some column"]``。 有关详细信息，请参见  :class:`_sql.ColumnCollection` 。

创建和删除数据库表
-------------------------------------

一旦定义了一些 :class:`~sqlalchemy.schema.Table` 对象，假设您正在使用全新的数据库，则可能要做的一件事是为这些表及其相关构造发出CREATE语句（顺便提一下，如果您已经有了一些首选方法，例如包含在您的数据库中的工具或现有脚本系统 ，请很高兴跳过此节 - SQLAlchemy不要求使用它来创建表）。

通常发出CREATE的方法是在  :class:`~sqlalchemy.schema.MetaData` ~sqlalchemy.schema.MetaData.create_all` 。此方法将发出查询，首先检查每个单独的表是否存在，如果找不到，则会发出CREATE语句：

.. sourcecode:: python+sql

    engine = create_engine("sqlite:///:memory:")

    metadata_obj = MetaData()

    user = Table(
        "user",
        metadata_obj,
        Column("user_id", Integer, primary_key=True),
        Column("user_name", String(16), nullable=False),
        Column("email_address", String(60), key="email"),
        Column("nickname", String(50), nullable=False),
    )

    user_prefs = Table(
        "user_prefs",
        metadata_obj,
        Column("pref_id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
        Column("pref_name", String(40), nullable=False),
        Column("pref_value", String(100)),
    )

    metadata_obj.create_all(engine)
    {execsql}PRAGMA table_info(user){}
    CREATE TABLE user(
            user_id INTEGER NOT NULL PRIMARY KEY,
            user_name VARCHAR(16) NOT NULL,
            email_address VARCHAR(60),
            nickname VARCHAR(50) NOT NULL
    )
    PRAGMA table_info(user_prefs){}
    CREATE TABLE user_prefs(
            pref_id INTEGER NOT NULL PRIMARY KEY,
            user_id INTEGER NOT NULL REFERENCES user(user_id),
            pref_name VARCHAR(40) NOT NULL,
            pref_value VARCHAR(100)
    )

  :func:`~sqlalchemy.schema.MetaData.create_all` ` ALTER TABLE``。

通过使用  :class:`~sqlalchemy.schema.Table` ` create()``和``drop()``方法可以创建和删除单个表。默认情况下，这些方法无论表是否存在都会发出CREATE或DROP：

.. sourcecode:: python+sql

    engine = create_engine("sqlite:///:memory:")

    metadata_obj = MetaData()

    employees = Table(
        "employees",
        metadata_obj,
        Column("employee_id", Integer, primary_key=True),
        Column("employee_name", String(60), nullable=False, key="name"),
        Column("employee_dept", Integer, ForeignKey("departments.department_id")),
    )
    employees.create(engine)
    {execsql}CREATE TABLE employees(
        employee_id SERIAL NOT NULL PRIMARY KEY,
        employee_name VARCHAR(60) NOT NULL,
        employee_dept INTEGER REFERENCES departments(department_id)
    )
    {}

``drop()``方法：

.. sourcecode:: python+sql

    employees.drop(engine)
    {execsql}DROP TABLE employees
    {}

要启用“首先检查表存在”的逻辑，请将``create()``或``drop()``的``checkfirst = True``参数添加到中：

    employees.create(engine，checkfirst = True)
    employees.drop(engine，checkfirst = False)

.. _schema_migrations:

通过Migrations修改数据库对象
---------------------------------------------

尽管SQLAlchemy直接支持发射CREATE和DROP语句以进行模式构造，但通常通过ALTER语句以及其他特定于数据库的构造来更改这些构造的能力超出了SQLAlchemy自身的范围。尽管手动发射ALTER语句等很容易，例如通过将  :func:`_expression.text`  或使用  :class:` .DDL`  构造，但常规做法是使用模式迁移工具来自动维护数据库架构并与应用程序代码相对应。

SQLAlchemy项目提供了用于此目的的迁移工具 `Alembic <https://alembic.sqlalchemy.org>`_。
Alembic具有高度自定义的环境和最简单的使用模式，支持诸如事务DDL、自动生成“候选”迁移、生成SQL脚本的“脱机”模式以及分支解决方案等功能。

Alembic取代了 `SQLAlchemy-Migrate <https://github.com/openstack/sqlalchemy-migrate>`_ 项目，该项目是SQLAlchemy的原始迁移工具，现在已被视为遗留程序。

.. _schema_table_schema_name:

指定模式名称
--------------------------

大多数数据库支持多个“模式”的概念 - 指代备选表集和其他构造的空间。具有不同名称空间的“模式”的服务器端几何形状有许多形式，包括特定于特定数据库的“schemas”名称（例如PostgreSQL schemas），同级别命名的兄弟数据库（例如MySQL / MariaDB访问同一服务器上的其他数据库），以及其他概念，比如表由其他用户名（Oracle，SQL Server）拥有或甚至是引用替代数据库文件（SQLite ATTACH）或远程服务器（Oracle DBLINK与 synonym）的名称。

以上所有方法（大多数）共同使用一种方式来使用字符串名称引用此备选表集，即。SQLAlchemy将此名称称为**模式名称**。在SQLAlchemy内部，这只是与 :class:`_schema.Table` 对象关联的字符串名称，并且随后以适合于目标数据库的方式呈现到SQL语句中，使表表现为在其远程“模式”中引用的表，无论这个机制在目标数据库上是什么。

“模式”名称可能直接与  :class:`_schema.Table`  参数相关联；当使用ORM时，来自 :ref:` orm_declarative_table_config_toplevel`配置的参数字典将传递该参数。

还可以使用   :class:`_schema.MetaData`  对象将“模式”名称与之自动生效于未指定自己的名称的所有   :class:` _schema.Table`  对象关联。最后，SQLAlchemy还支持一种动态模式名称系统，该系统通常用于每个连接或每个语句基础上的多租户应用程序，以便单个   :class:`_schema.Table`  元数据集可以引用按每个连接或每个语句基础上动态配置的模式名称集。

.. 主题:: 什么是“模式”？

      SQLAlchemy对数据库“模式”的支持是针对PostgreSQL样式模式具有一流支持。在这种风格中，首先有一个通常具有单个“所有者”的“数据库”。在该数据库中可以有任意数量的“模式”，然后包含实际的表对象。

      特定模式中的表使用“<schema> .<tablename>”语法显式引用。与这些数据库不同，如MySQL，其中仅有“数据库”，但SQL语句可以使用相同的语法引用多个数据库，该语法类似，除了是“<database> .<tablename>”。在Oracle上，“schema”名称引用了另一个概念，即表的“owner”。无论使用哪种类型的数据库，SQLAlchemy在通用语法“<qualifier>.<tablename>”中使用短语“schema”来指代限定标识符。

.. seealso::

      :ref:`orm_declarative_table_schema_name`  - 在ORM使用时的模式名称指定

      :ref:`declarative table <orm_declarative_table_config_toplevel>`  配置

最基本的示例是使用Core   :class:`_schema.Table` ，如下所示::

    metadata_obj = MetaData()

    financial_info = Table(
        "financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("value", String(100), nullable=False),
        schema="remote_banks",
    )

使用该  :class:`_schema.Table` ` remote_banks``模式名称的表名称 ``financial_info``：

.. sourcecode:: pycon+sql

    >>> print(select(financial_info))
    {printsql}SELECT remote_banks.financial_info.id, remote_banks.financial_info.value
    FROM remote_banks.financial_info

定义有显式模式名称的  :class:`_schema.Table` ` 'remote_banks.financial_info'``来在  :attr:`_schema.MetaData.tables`  集合中查看它：

    >>> metadata_obj.tables["remote_banks.financial_info"]
    Table('financial_info', MetaData(),
    Column('id', Integer(), table=<financial_info>, primary_key=True, nullable=False),
    Column('value', String(length=100), table=<financial_info>, nullable=False),
    schema='remote_banks')

这是用于引用此表的“点”名称，并且可以使用它在  :class:`_schema.ForeignKey`  或 :class:` _schema.ForeignKeyConstraint`对象中引用表，即使引用表也位于同一模式中也是如此：

    customer = Table(
        "customer",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("financial_info_id", ForeignKey("remote_banks.financial_info.id")),
        schema="remote_banks",
    )

  :paramref:`_schema.Table.schema`  参数可以与某些方言一起使用，以指示对特定表的多令牌（例如点分）路径。这在特定于Microsoft SQL Server这样的数据库上特别重要，其中常常有点号“database/owner”。这些令牌可以直接放入名称中，例如::

    schema = "dbo.scott"

.. seealso::

      :ref:`multipart_schema_names`  - 描述了使用SQL Server方言的点分模式名称

      :ref:`metadata_reflection_schemas` 

.. _schema_metadata_schema_name:

使用MetaData指定默认模式名称
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_schema.MetaData`  参数的显式默认选项，通过将  :paramref:` _schema.MetaData.schema`  参数传递给顶级   :class:`_schema.MetaData`  构造函数：


    metadata_obj = MetaData(schema="remote_banks")

    financial_info = Table(
        "financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("value", String(100), nullable=False),
    )

在  :paramref:`_schema.Table.schema`  参数保持其默认值： ` `None``时，将使用值 ``"remote_banks"``。这包括，  :class:`_schema.Table`  使用模式限定的名称目录，即::

    metadata_obj.tables["remote_banks.financial_info"]

当使用   :class:`_schema.ForeignKey`  或 :class:` _schema.ForeignKeyConstraint`对象引用此表时，可以引用其限定模式名称或非限定模式名称：

    # 这两个都是可以的：

    refers_to_financial_info = Table(
        "refers_to_financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("fiid", ForeignKey("financial_info.id")),
    )


    #或者

    refers_to_financial_info = Table(
        "refers_to_financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("fiid", ForeignKey("remote_banks.financial_info.id")),
    )

当使用设置了  :paramref:`_schema.MetaData.schema`  参数的 :class:` _schema.MetaData`对象时，
要指定   :class:`_schema.Table`  不使用模式限定名称，请使用特殊符号  :data:` _schema.BLANK_SCHEMA`  ::

    from sqlalchemy import BLANK_SCHEMA

    metadata_obj = MetaData(schema="remote_banks")

    financial_info = Table(
        "financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("value", String(100), nullable=False),
        schema=BLANK_SCHEMA,  # 表示不使用 "remote_banks"
    )

.. seealso::

     :paramref:`_schema.MetaData.schema` 

.. _schema_dynamic_naming_convention:

应用动态模式命名约定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :paramref:`_schema.Table.schema`   参数使用时，也可以基于每个连接或每个执行动态地应用于动态查找，以便例如在多租户情况下，每个事务或语句可以针对特定的模式名称集。  :ref:` schema_translating`部分描述了如何使用此特征。

.. seealso::

      :ref:`schema_translating` 


.. _schema_set_default_connections:

为新连接设置默认模式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

以上所有方法都是涉及在SQL语句中包含显式模式名称的方法。实际上，数据库连接具有“默认”模式的概念，即如果表名没有明确指定模式限定，则该默认模式是“模式”（或数据库所有者，等等）的名称。这些名称通常在登录级别进行配置，例如连接到PostgreSQL数据库时，默认“模式”称为“Public”。

通常情况下，无法通过登录本身设置默认的“模式名称”，而是每次创建连接时使用诸如在PostgreSQL上“SET SEARCH_PATH”或在Oracle上“ALTER SESSION”的语句进行配置设置。这些方法可以通过使用  :meth:`_pool.PoolEvents.connect`  事件来实现，该事件允许在创建了第一个DBAPI连接时访问DBAPI连接。例如，将Oracle CURRENT_SCHEMA变量设置为替代名称：

    from sqlalchemy import event
    from sqlalchemy import create_engine

    engine = create_engine("oracle+cx_oracle://scott:tiger@tsn_name")


    @event.listens_for(engine, "connect", insert=True)
    def set_current_schema(dbapi_connection, connection_record):
        cursor_obj = dbapi_connection.cursor()
        cursor_obj.execute("ALTER SESSION SET CURRENT_SCHEMA=%s" % schema_name)
        cursor_obj.close()

上述“set_current_schema（）”事件处理程序将在上面的 :class:`_engine.Engine` 首次连接时立即发生。由于事件被“插入”到处理程序列表的开头，因此它也将在dialect自己的事件处理程序运行之前运行，这主要包括将确定连接的“默认模式”的事件处理程序。

有关其他数据库的信息，请参见关于如何配置默认模式的数据库和/或特定于方言的文档。

.. versionchanged:: 1.4.0b2 该文件现在无需建立其他事件处理程序。

.. seealso::

      :ref:`postgresql_alternate_search_path`  - 在   :ref:` postgresql_toplevel`  骨架文档中。

模式和反射
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

模式   :class:`_schema.Table`  中引入的表反射特性交互。有关此工作方式的详细信息，请参见   :ref:` metadata_reflection_schemas` 。

特定于后端的选项
------------------------

  :class:`~sqlalchemy.schema.Table` ~sqlalchemy.schema.Table` 使用``mysql_engine``表示此内容：

    addresses = Table(
        "engine_email_addresses",
        metadata_obj,
        Column("address_id", Integer, primary_key=True),
        Column("remote_user_id", Integer, ForeignKey(users.c.user_id)),
        Column("email_address", String(20)),
        mysql_engine="InnoDB",
    )

其他后端也可能支持表级选项 - 这些将在每个方言的个别文档部分中描述。

列，表，MetaData API
---------------------------

.. attribute:: sqlalchemy.schema.BLANK_SCHEMA
    :noindex:

    引用  :attr:`.SchemaConst.BLANK_SCHEMA` 。

.. attribute:: sqlalchemy.schema.RETAIN_SCHEMA
    :noindex:

    引用  :attr:`.SchemaConst.RETAIN_SCHEMA` 


.. autoclass:: Column
    :members:
    :inherited-members:


.. autoclass:: MetaData
    :members:

.. autoclass:: SchemaConst
    :members:

.. autoclass:: SchemaItem
    :members:

.. autofunction:: insert_sentinel

.. autoclass:: Table
    :members:
    :inherited-members: