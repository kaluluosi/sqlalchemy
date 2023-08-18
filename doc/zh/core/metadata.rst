.. _metadata_toplevel:

.. _metadata_describing_toplevel:

.. _metadata_describing:

==================================
使用 MetaData 描述数据库
==================================
本节讨论了基本的 :class:`_schema.Table`, :class:`_schema.Column`, 和 :class:`_schema.MetaData` 对象。

.. module:: sqlalchemy.schema

.. seealso::

    :ref:`unified_tutorial` 的 :ref:`tutorial_working_with_metadata` - SQLAlchemy 数据库元数据概念的教程介绍

元数据实体的集合存储在一个名为 :class:`~sqlalchemy.schema.MetaData` 的对象中::

    from sqlalchemy import MetaData

    metadata_obj = MetaData()

:class:`~sqlalchemy.schema.MetaData` 是一个容器对象，可以将描述一个或多个数据库的许多不同特征存储在一起。

要表示一个表，请使用 :class:`~sqlalchemy.schema.Table` 类。它的两个主要参数是表名，然后是这个表将要关联的 :class:`~sqlalchemy.schema.MetaData` 对象。其余的位置参数大多数都是描述每一列的 :class:`~sqlalchemy.schema.Column` 对象::

    from sqlalchemy import Table, Column, Integer, String

    user = Table(
        "user",
        metadata_obj,
        Column("user_id", Integer, primary_key=True),
        Column("user_name", String(16), nullable=False),
        Column("email_address", String(60)),
        Column("nickname", String(50), nullable=False),
    )

上面描述了名为 ``user`` 的表，其中包含四列。表格的主键由 ``user_id`` 列组成。多个列可以被分配给 ``primary_key=True`` 标志，这表示多列主键，称为 *组合* 主键。

还要注意，每个列都使用相应的通用化类型对象（如 :class:`~sqlalchemy.types.Integer` 和 :class:`~sqlalchemy.types.String`）来描述其数据类型。 SQLAlchemy 具有许多不同级别的特定类型以及创建定制类型的能力。类型系统的文档可以在 :ref:`types_toplevel` 找到。

.. _metadata_tables_and_columns:

访问表格和列
----------------------------

:class:`~sqlalchemy.schema.MetaData` 对象包含与其关联的所有模式构造。它支持访问这些表对象的一些方法，例如 ``sorted_tables`` 访问器，它按外键依赖关系的顺序返回每个 :class:`~sqlalchemy.schema.Table` 对象的列表 (也就是说，每个表在其引用的所有表之前)::

    >>> for t in metadata_obj.sorted_tables:
    ...     print(t.name)
    user
    user_preference
    invoice
    invoice_item

在大多数情况下，单个 :class:`~sqlalchemy.schema.Table` 对象已经被显式地声明，这些对象通常作为应用程序中的模块级变量来直接访问。一旦定义了 :class:`~sqlalchemy.schema.Table`，它就有了一整套访问器，这些访问器允许检查其特性。给定以下:class:`~sqlalchemy.schema.Table` 定义::

    employees = Table(
        "employees",
        metadata_obj,
        Column("employee_id", Integer, primary_key=True),
        Column("employee_name", String(60), nullable=False),
        Column("employee_dept", Integer, ForeignKey("departments.department_id")),
    )

请注意，此表中使用的 :class:`~sqlalchemy.schema.ForeignKey` 对象——该构造定义了对远程表的引用，并且在 :ref:`metadata_foreignkeys` 中被完全描述。关于这张表的信息访问方法包括:

    # 访问列"employee_id":
    employees.columns.employee_id

    # 或
    employees.c.employee_id

    # 可以用字符串方式访问
    employees.c["employee_id"]

    # 可以使用多个字符串返回列的元组（在2.0中新添加的）
    emp_id, name, type = employees.c["employee_id", "name", "type"]

    # 迭代所有列
    for c in employees.c:
        print(c)

    # 获取表的主键列
    for primary_key in employees.primary_key:
        print(primary_key)

    # 获取表的外键对象：
    for fkey in employees.foreign_keys:
        print(fkey)

    # 访问表的 MetaData:
    employees.metadata

    # 访问列的名称、类型、可为空、主键、外键
    employees.c.employee_id.name
    employees.c.employee_id.type
    employees.c.employee_id.nullable
    employees.c.employee_id.primary_key
    employees.c.employee_dept.foreign_keys

    # 获取列的 "key"，它默认为名字，但可以是任何用户定义的字符串:
    employees.c.employee_name.key

    # 访问列的表:
    employees.c.employee_id.table is employees

    # 获取由外键关联的表格
    list(employees.c.employee_dept.foreign_keys)[0].column.table

.. tip::

  :attr:`_sql.FromClause.c` 集合，相当于 :attr:`_sql.FromClause.columns` 集合，是 :class:`_sql.ColumnCollection` 的实例，它提供了对列集合的类似字典的接口。 名称通常像属性名称一样访问，例如 ``employees.c.employee_name```。 但是，对于具有空格或与字典方法名称相匹配的名称（例如 :meth:`_sql.ColumnCollection.keys` 或 :meth:`_sql.ColumnCollection.values`），必须使用索引访问，例如``employees.c['values']``或``employees.c["some column"]``。有关更多信息，请参见：class:`_sql.ColumnCollection`。

创建和删除数据库表
-------------------------------------

一旦您定义了一些 :class:`~sqlalchemy.schema.Table` 对象，假设您正在使用全新的数据库的话，您可能需要做的一件事情是为这些表发出 CREATE 语句以及它们的相关构造（顺便说一句，如果您已经有一些首选方法，如与您的数据库一起提供的工具或现有的脚本系统，那么您可能*不想*这样做 - 如果是这样，请随意跳过此部分 - SQLAlchemy 不要求使用它来创建您的表）。

通常发出 CREATE 语句的方法是在 :class:`~sqlalchemy.schema.MetaData` 对象上使用 :func:`~sqlalchemy.schema.MetaData.create_all`。此方法将发出查询，先检查每个单独的表是否存在，如果没有找到则发出 CREATE 语句：

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

:func:`~sqlalchemy.schema.MetaData.create_all` 创建了通常在表格定义本身中内联定义的外键约束，因此它还按依赖顺序以它们的依赖关系顺序生成表格。 有选项可以更改此行为，使其使用 ``ALTER TABLE``。

删除所有表也可以通过 :func:`~sqlalchemy.schema.MetaData.drop_all` 方法实现。此方法的功能与 :func:`~sqlalchemy.schema.MetaData.create_all` 完全相反 - 首先检查每个表是否存在，然后按依赖项的相反顺序删除各个表。

可以通过 :class:`~sqlalchemy.schema.Table` 的 ``create()`` 和 ``drop()`` 方法创建和删除单个表。默认情况下，这些方法会发出 CREATE 或 DROP 命令，而不管表是否存在：

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

``drop()`` 方法：

.. sourcecode:: python+sql

    employees.drop(engine)
    {execsql}DROP TABLE employees
    {}

要启用 "检查表是否存在" 的逻辑，请将 ``create()`` 或 ``drop()`` 配置项中添加 ``checkfirst=True`` 参数：

    employees.create(engine, checkfirst=True)
    employees.drop(engine, checkfirst=False)

.. _schema_migrations:

通过迁移更改数据库对象
---------------------------------------------

虽然 SQLAlchemy 直接支持发出 CREATE 和 DROP 语句来创建模式构造，但更改这些构造的能力，通常通过 ALTER 语句以及其他针对特定数据库的构造，超出了 SQLAlchemy 自身的范围。虽然您可以很容易地手动发出 ALTER 语句和类似语句，例如通过将 :func:`_expression.text` 构造传递给 :meth:`_engine.Connection.execute` 或使用 :class:`.DDL` 构造，但在多租户应用程序中，使用架构迁移工具自动维护数据库模式是一种常见的实践。

SQLAlchemy 项目提供了 `Alembic <https://alembic.sqlalchemy.org>`_ 迁移工具来实现这一目的。 Alembic 具有高度可定制的环境和极简使用模式，支持诸如事务 DDL，自动生成 "候选" 迁移，生成 SQL 脚本的 "离线" 模式以及分支解析等功能。

Alembic 取代了 `SQLAlchemy-Migrate <https://github.com/openstack/sqlalchemy-migrate>`_ 工程，后者是 SQLAlchemy 的原始迁移工具，现在已经被视为已过时。

.. _schema_table_schema_name:

指定模式名称
--------------------------

大多数数据库都支持多个 "模式" - 引用替代表格和其他构造的名称空间。 服务器端的 "模式" 几种形式存在，包括特定数据库范围内的 "模式" 名称（例如 PostgreSQL 模式），具有相同服务器上其他数据库名称的指定同级数据库（例如 MySQL 或 MariaDB 访问相同服务器上其他数据库），以及其他概念，如由其他用户名拥有的表（ Oracle、SQL Server）或甚至是指向备用数据库文件名的名称（SQLite ATTACH）或远程服务器（Oracle DBLINK with synonyms）。

所有以上方法（大多数）通常都有方法使用字符串名称引用这些替代表格的方式。 SQLAlchemy 认为此名称为 **模式名称**。 在 SQLAlchemy 中，这不过是一个字符串名称，它与 :class:`_schema.Table` 对象关联，并且然后以适合目标数据库的方式将其呈现为 SQL 语句，以便表被引用远程位置中的模式，无论该数据库是什么机制。

可以使用 :paramref:`_schema.Table.schema` 参数将 "schema" 的名称直接与 :class:`_schema.Table` 的 Core 对象一起使用，如下所示::

    metadata_obj = MetaData()

    financial_info = Table(
        "financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("value", String(100), nullable=False),
        schema="remote_banks",
    )

此 :class:`_schema.Table` 生成的 SQL, 如下面的 SELECT 语句，将以 ``remote_banks`` 模式名称明确地限定表名 ``financial_info``：

.. sourcecode:: pycon+sql

    >>> print(select(financial_info))
    {printsql}SELECT remote_banks.financial_info.id, remote_banks.financial_info.value
    FROM remote_banks.financial_info

当使用 :ref:`declarative table <orm_declarative_table_config_toplevel>` 配置与 ORM 结合使用时，可以使用 ``__table_args__`` 参数字典传递该参数。

:class:`_schema.Table` 对象显式定义架构名称时，组合名称将存储在内部 :class:`_schema.MetaData` 命名空间中。我们可以在 :attr:`_schema.MetaData.tables` 集合中搜索键 ``'remote_banks.financial_info'`` 来查看此情况::

    >>> metadata_obj.tables["remote_banks.financial_info"]
    Table('financial_info', MetaData(),
    Column('id', Integer(), table=<financial_info>, primary_key=True, nullable=False),
    Column('value', String(length=100), table=<financial_info>, nullable=False),
    schema='remote_banks')

在引用该表时，甚至是在同一模式中的引用表采用此指定的点名称如 ``remote_banks.financial_info``:

    customer = Table(
        "customer",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("financial_info_id", ForeignKey("remote_banks.financial_info.id")),
        schema="remote_banks",
    )

当使用 :class:`_schema.ForeignKey` 或 :class:`_schema.ForeignKeyConstraint` 对象引用此表时，无论是使用 schema 名称限定也好，使用非 schema 名称限定也好，都可以引用 ``remote_banks.financial_info`` 表::

    # 两者都可以：

    refers_to_financial_info = Table(
        "refers_to_financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("fiid", ForeignKey("financial_info.id")),
    )


    # 或

    refers_to_financial_info = Table(
        "refers_to_financial_info",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("fiid", ForeignKey("remote_banks.financial_info.id")),
    )

当使用默认模式 :class:`_schema.Table` 时，如果 

:param:`_schema.Table.schema` 设置为 :const:`_schema.BLANK_SCHEMA`；

元数据对象设置 :paramref:`_schema.MetaData.schema` 在此情况下默认为基础的数据库。

.. topic::  什么是 "schema" ?

    SQLAlchemy 对数据库 "schema" 的支持是带着 PostgreSQL-style schema 的第一方支持而设计的。在这个风格中，首先有一个 "database"，通常具有单个 "owner"。在这个数据库中，可以有任意多个 "schemas"，它们包含实际的表对象。

    特定模式内的表格通过 "<schema>.<table>" 语法明确引用。 与这种体系结构相反的是 MySQL 的体系结构，其中只有 "databases"，然而 SQL 语句可以使用相同的语法引用多个数据库，语法是 "<database>.<tablename>"。对于 Oracle，此语法引用了另一个概念，即表的 "owner"。无论使用哪种数据库，SQLAlchemy 都使用短语 "schema" 来引用符合 "<qualifier>.<tablename>" 一般语法的限定符。

.. seealso::

    :ref:`orm_declarative_table_schema_name`- ORM 结合使用时的架构名称规定
    :ref:`declarative table <orm_declarative_table_config_toplevel>` 配置


:class:`_schema.Table` 支持特定于数据库的选项。 例如，MySQL 有不同的表格后端类型，包括 "MyISAM" 和 "InnoDB"。这可以使用 :class:`_schema.Table` 来表示 ``mysql_engine``：

    addresses = Table(
        "engine_email_addresses",
        metadata_obj,
        Column("address_id", Integer, primary_key=True),
        Column("remote_user_id", Integer, ForeignKey(users.c.user_id)),
        Column("email_address", String(20)),
        mysql_engine="InnoDB",
    )

其他后端可能还支持表级选项-对于每个 dialet 都在各自的文档部分中进行了描述。

Column、Table 和 MetaData API
---------------------------

.. attribute:: sqlalchemy.schema.BLANK_SCHEMA
    :noindex:

    引用 :attr:`.SchemaConst.BLANK_SCHEMA`。

.. attribute:: sqlalchemy.schema.RETAIN_SCHEMA
    :noindex:

    引用 :attr:`.SchemaConst.RETAIN_SCHEMA`。

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