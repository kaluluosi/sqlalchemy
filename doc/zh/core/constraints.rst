.. _metadata_constraints_toplevel:
.. _metadata_constraints:

.. currentmodule:: sqlalchemy.schema

================================
定义约束和索引
================================

本节将讨论SQL :term:`constraints`和索引。在SQLAlchemy中，关键类包括 :class:`_schema.ForeignKeyConstraint`和 :class:`.Index`。

.. _metadata_foreignkeys:

定义外键
---------------------

在SQL中，*外键*是一个表级结构，它限制该表中一个或多个列只允许存在于不同组列中的值，通常但不总是位于不同表上。我们称受到限制的列为*foreign key*列，而限制它们的列为*引用*列。引用列几乎总是为其所有者表定义了主键，尽管也有例外情况。外键是将具有彼此关系的行配对在一起的“联接”，在几乎所有操作中，SQLAlchemy都为这个概念赋予了非常深远的重要性。

在SQLAlchemy和DDL中，外键约束可以定义为表子句中的附加属性，或者对于单列外键，它们可以在单个列的定义中选择性地指定。单列外键更常见，在列级别上由构造 :class:`~sqlalchemy.schema.Column`对象的 :class:`~sqlalchemy.schema.ForeignKey`对象作为参数来指定::

    user_preference = Table(
        "user_preference",
        metadata_obj,
        Column("pref_id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
        Column("pref_name", String(40), nullable=False),
        Column("pref_value", String(100)),
    )

在上面的示例中，我们为 ``user_preference`` 定义了一个新表，其中每行都必须包含在 ``user``表的 ``user_id`` 列中也存在的值。

:class:`~sqlalchemy.schema.ForeignKey`的参数通常是 *<tablename>.<columnname>*的形式的字符串，或者对于位于远程架构或“所有者”中的表，格式为 <schemaname>.<tablename>.<columnname>*。它也可以是一个实际的 :class:`~sqlalchemy.schema.Column`对象，稍后我们将看到，它可以通过其``c``集合从现有的:class:`~sqlalchemy.schema.Table`对象访问::

    ForeignKey(user.c.user_id)

使用字符串的优点在于，当第一次需要时，仅当``user``和``user_preference``之间的Python链接得到解析，以便于多个模块轻松地分布表对象，并且可以按任何顺序定义。

还可以在表级别上定义外键，使用 :class:`~sqlalchemy.schema.ForeignKeyConstraint` 对象。此对象可以描述单列或多列外键。多列外键称为*组合*外键，并且几乎总是引用具有组合主键的表。下面我们定义一个具有组合主键的表“invoice”::

    invoice = Table(
        "invoice",
        metadata_obj,
        Column("invoice_id", Integer, primary_key=True),
        Column("ref_num", Integer, primary_key=True),
        Column("description", String(60), nullable=False),
    )

然后再定义一个具有组合外键，引用``invoice``的表“invoice_item”::

    invoice_item = Table(
        "invoice_item",
        metadata_obj,
        Column("item_id", Integer, primary_key=True),
        Column("item_name", String(60), nullable=False),
        Column("invoice_id", Integer, nullable=False),
        Column("ref_num", Integer, nullable=False),
        ForeignKeyConstraint(
            ["invoice_id", "ref_num"], ["invoice.invoice_id", "invoice.ref_num"]
        ),
    )

重要的是要注意，*ForeignKeyConstraint*是定义组合外键的唯一方法。虽然我们还可以在``invoice_item.invoice_id``和``invoice_item.ref_num``两个列上放置单独的:class:`~sqlalchemy.schema.ForeignKey`对象，但SQLAlchemy不会意识到这两个值应该配对在一起——它将是单个组合外键，引用两个列，而不是两个独立的外键约束。

.. _use_alter:

通过ALTER创建/删除外键约束
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ ~~

DDL中涉及的外键带有的行为，是约束通常在"内联"在CREATE TABLE语句中创建，例如::

    CREATE TABLE addresses (
        id INTEGER NOT NULL,
        user_id INTEGER,
        email_address VARCHAR NOT NULL,
        PRIMARY KEY (id),
        CONSTRAINT user_id_fk FOREIGN KEY(user_id) REFERENCES users (id)
    )

“CONSTRAINT .. FOREIGN KEY”指令用于在“内联”方式内创建约束在CREATE TABLE定义中。 :meth:`_schema.MetaData.create_all`和 :meth:`_schema.MetaData.drop_all`方法将默认情况下在所有涉及:class:`_schema.Table`对象的拓扑排序中执行此操作，因此表将按照其外键依赖性的顺序创建和删除（此排序也可以通过 :attr:`_schema.MetaData.sorted_tables`访问器获得）。

当两个或多个外键约束涉及"依赖循环"的情况时，这种方法无法工作，其中一组表彼此相互依赖，假设后端执行外键的一致性（在SQLite，MySQL/MyISAM之外）。因此，这些方法将在除不支持大多数更改形式的SQLite之外的所有后端上，将这些约束断开为单独的ALTER语句。例如，对于类似这样的模式，我们将其称为：：

    node = Table(
        "node",
        metadata_obj,
        Column("node_id", Integer, primary_key=True),
        Column("primary_element", Integer, ForeignKey("element.element_id")),
    )

    element = Table(
        "element",
        metadata_obj,
        Column("element_id", Integer, primary_key=True),
        Column("parent_node_id", Integer),
        ForeignKeyConstraint(
            ["parent_node_id"], ["node.node_id"], name="fk_element_parent_node_id"
        ),
    )

当我们在诸如 PostgreSQL 后端上调用 :meth:`_schema.MetaData.create_all` 时，这两个表之间的循环会被解决，约束将分别创建：：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     metadata_obj.create_all(conn, checkfirst=False)
    {execsql}CREATE TABLE element (
        element_id SERIAL NOT NULL,
        parent_node_id INTEGER,
        PRIMARY KEY (element_id)
    )

    CREATE TABLE node (
        node_id SERIAL NOT NULL,
        primary_element INTEGER,
        PRIMARY KEY (node_id)
    )

    ALTER TABLE element ADD CONSTRAINT fk_element_parent_node_id
        FOREIGN KEY(parent_node_id) REFERENCES node (node_id)
    ALTER TABLE node ADD FOREIGN KEY(primary_element)
        REFERENCES element (element_id)
    {stop}

在删除所有表时需要发出DROP命令，同样的逻辑适用，但是请注意，在SQL中，发出DROP CONSTRAINT需要指定约束的名称。在上面的例子中，我们尚未为“node”表指定此约束的名称。因此，系统将仅尝试发出命名约束的DROP：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     metadata_obj.drop_all(conn, checkfirst=False)
    {execsql}ALTER TABLE element DROP CONSTRAINT fk_element_parent_node_id
    DROP TABLE node
    DROP TABLE element
    {stop}


在循环无法解决的情况下，例如如果我们在这里没有给任何约束命名，我们将收到以下错误消息：

.. sourcecode:: text

    sqlalchemy.exc.CircularDependencyError: Can't sort tables for DROP;
    an unresolvable foreign key dependency exists between tables:
    element, node.  Please ensure that the ForeignKey and ForeignKeyConstraint
    objects involved in the cycle have names so that they can be dropped
    using DROP CONSTRAINT.

此错误仅适用于DROP，因为我们可以在CREATE CASE中发出`ADD CONSTRAINT`，而不需要名称; 数据库通常会自动分配一个名称。

当在可查迁移工具中使用：func:`alembic < https://alembic.sqlalchemy.org/>`时，可通过模式迁移来处理现有表和约束; 但是，Alembic和SQLAlchemy当前未为没有指定名称的约束对象创建名称，这意味着要能够改变现有约束，我们必须反向工程用于自动分配名称到关系数据库的命名系统，或者必须注意确保所有约束都已命名。

与分配所有 :class:`.Constraint` 和 :class:`.Index` 对象的明确名称相比，可以使用事件构建自动命名方案。这种方法的优点是，约束将获得一致的命名方案，无需在整个代码中显式名称参数，而且约定在使用 :paramref:`_schema.MetaData.naming_convention` 时可以用于通过 :paramref:`_schema.Column.unique` 和 :paramref:`_schema.Column.index` 参数生成的所有约束和索引。从SQLAlchemy 0.9.2开始，此事件驱动的方法已经包含在内，并可以使用参数 :paramref:`_schema.MetaData.naming_convention` 进行配置。

为元数据集合配置命名约定
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:paramref:`_schema.MetaData.naming_convention` 指的是一个接受 :class:`.Index` 类或单个 :class:`.Constraint` 类作为键，并接受Python字符串模板作为值的字典。它还接受多个字符串代码作为备用键， ``fk`` ， ``pk`` ， ``ix`` ， ``ck`` ， ``uq`` 用于外键、主键、索引、检查和唯一键约束，分别。在字典中使用的字符串模板用于在与此 :class:`_schema.MetaData` 对象关联的约束或索引未具有现有名称时使用。除了一个例外情况外，此名称可以由现有名称进一步修饰。

一个适用于基本情况的示例命名约定如下：

    convention = {
        "ix": "ix_%(column_0_label)s",
        "uq": "uq_%(table_name)s_%(column_0_name)s",
        "ck": "ck_%(table_name)s_%(constraint_name)s",
        "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
        "pk": "pk_%(table_name)s",
    }

    metadata_obj = MetaData(naming_convention=convention)

上面的约定将为目标 :class:`_schema.MetaData` 集合中的所有约束都建立名称。例如，我们可以观察到创建未命名的 :class:`.UniqueConstraint`时生成的名称：

    >>> user_table = Table(
    ...     "user",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30), nullable=False),
    ...     UniqueConstraint("name"),
    ... )
    >>> list(user_table.constraints)[1].name
    'uq_user_name'

当我们只使用 :paramref:`_schema.Column.unique` 标志时，同样的特性也会出现：

    >>> user_table = Table(
    ...     "user",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30), nullable=False, unique=True),
    ... )
    >>> list(user_table.constraints)[1].name
    'uq_user_name'

命名约定方法的一个关键优点是，这些名称在Python构建时间而不是在DDL发出时间建立。在使用Alembic的“--自动生成”功能时，这对于生成新的迁移脚本意味着命名约定将变得明确：

    def upgrade():
        op.create_unique_constraint("uq_user_name", "user", ["name"])

上面的 ``"uq_user_name"`` 字符串从我们在元数据中找到的 :class:`.UniqueConstraint` 对象中复制，这是``--autogenerate``生成新的迁移脚本时的效果。

可用的令牌包括 ``%(table_name)s`` ， ``%(referred_table_name)s`` ， ``%(column_0_name)s`` ， ``%(column_0_label)s`` ， ``%(column_0_key)s`` ， ``%(referred_column_0_name)s``和 ``%(constraint_name)s`` 以及每个的多列版本，包括 ``%(column_0N_name)s`` ， ``%(column_0_N_name)s`` ， ``%(referred_column_0_N_name)s`` ，其中所有列名以符号分隔或没有符号分隔的方式呈现。 :paramref:`_schema.MetaData.naming_convention` 文档对每个约定的详细信息有更多细节。

.. _constraint_default_naming_convention:

默认命名约定
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:paramref:`_schema.MetaData.naming_convention` 的默认值处理了长期以来SQLAlchemy的行为，即为使用 :paramref:`_schema.Column.index` 参数创建的 :class:`.Index` 对象分配名称：

    >>> from sqlalchemy.sql.schema import DEFAULT_NAMING_CONVENTION
    >>> DEFAULT_NAMING_CONVENTION
    immutabledict({'ix': 'ix_%(column_0_label)s'})

截断长名称
~~~~~~~~~~~~~~~~~~~~~~~~~

当一个生成的名称，特别是使用多列令牌的名称，太长而无法符合目标数据库的标识符长度限制时（例如，PostgreSQL有一个限制为63个字符），名称将会被明确定义的后缀截断为源名称的md5哈希基础上的4个字符。例如，以下是将带有使用的列名创建时生成很长名称的约定：

    metadata_obj = MetaData(
        naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
    )

    long_names = Table(
        "long_names",
        metadata_obj,
        Column("information_channel_code", Integer, key="a"),
        Column("billing_convention_name", Integer, key="b"),
        Column("product_identifier", Integer, key="c"),
        UniqueConstraint("a", "b", "c"),
    )

在PostgreSQL方言上，长度超过63个字符的名称将被截断如下：

.. sourcecode:: sql

    CREATE TABLE long_names (
        information_channel_code INTEGER,
        billing_convention_name INTEGER,
        product_identifier INTEGER,
        CONSTRAINT uq_long_names_information_channel_code_billing_conventi_a79e
        UNIQUE (information_channel_code, billing_convention_name, product_identifier)
    )

上面的后缀 ``a79e`` 基于长名称的md5哈希值，并且每次生成相同的值，以便为给定模式生成一致的名称。

创建自定义的命名约定令牌
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以添加新令牌，通过在命名_convention字典中指定额外的令牌和可调用对象。例如，如果我们想使用GUID方案命名外键约束，我们可以按以下方式执行：

    import uuid


    def fk_guid(constraint, table):
        str_tokens = (
            [
                table.name,
            ]
            + [element.parent.name for element in constraint.elements]
            + [element.target_fullname for element in constraint.elements]
        )
        guid = uuid.uuid5(uuid.NAMESPACE_OID, "_".join(str_tokens).encode("ascii"))
        return str(guid)


    convention = {
        "fk_guid": fk_guid,
        "ix": "ix_%(column_0_label)s",
        "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    }

以上，当我们创建一个新的 :class:`_schema.ForeignKeyConstraint`时，我们将为名称获得一个名称如下：

    >>> metadata_obj = MetaData(naming_convention=convention)

    >>> user_table = Table(
    ...     "user",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("version", Integer, primary_key=True),
    ...     Column("data", String(30)),
    ... )
    >>> address_table = Table(
    ...     "address",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("user_id", Integer),
    ...     Column("user_version_id", Integer),
    ... )
    >>> fk = ForeignKeyConstraint(["user_id", "user_version_id"], ["user.id", "user.version"])
    >>> address_table.append_constraint(fk)
    >>> fk.name
    fk_0cd51ab5-8d70-56e8-a83c-86661737766d

参见：

    :paramref:`_schema.MetaData.naming_convention` - 有关其他用法详细信息以及可用命名组件的参考手册。

    `约束命名的重要性 <https://alembic.sqlalchemy.org/en/latest/naming.html>`_ - Alembic文档中。


.. versionadded:: 1.3.0 添加了多列命名令牌，例如 ``%(column_0_N_name)s``。生成的名称超出目标数据库的字符限制将被确定性地截断。

.. _naming_check_constraints:

命名CHECK约束
~~~~~~~~~~~~~~~~~~~~~~~~

:class:`.CheckConstraint`对象针对任意SQL表达式进行配置，它可以存在任意数量的列，并且通常使用原始SQL字符串进行配置。因此，与 :class:`.CheckConstraint` 一起使用的常见约定是我们预期对象已经具有名称，然后我们使用其他约定元素增强该名称。一个典型的约定是 ``"ck_%(table_name)s_%(constraint_name)s"``：

    metadata_obj = MetaData(
        naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
    )

    Table(
        "foo",
        metadata_obj,
        Column("value", Integer),
        CheckConstraint("value > 5", name="value_gt_5"),
    )

上面的表将生成名称``ck_foo_value_gt_5``：

.. sourcecode:: sql

    CREATE TABLE foo (
        value INTEGER,
        CONSTRAINT ck_foo_value_gt_5 CHECK (value > 5)
    )

:class:`.CheckConstraint`还支持 ``%(columns_0_name)s`` 令牌；我们可以通过确保在约束的表达式中使用 :class:`_schema.Column` 或 :func:`_expression.column` 元素来利用它，无论是将约束声明与表分开还是将其定义在表中：：约束（Constraints）和索引（Indexes）
---------------------------------------

配置约束名称
~~~~~~~~~~~~~~~

可以使用 ``naming_convention`` 参数为约束设置一个命名规则。只要在命名规则中使用了可用的参数，即可使用这个功能。

.. note::

   如果所使用的约束类型的名称参数已经明确指定，将不会应用使用的命名规则。

例如，以下代码：

.. sourcecode:: python

    metadata_obj = MetaData(
        naming_convention={
            "ck": "ck_%(table_name)s_%(column_0_name)s",
            "uq": "uq_%(table_name)s_%(column_0_name)s",
            "ix": "ix_%(table_name)s_%(column_0_name)s",
            "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
            "pk": "pk_%(table_name)s"
        }
    )

创建约束
~~~~~~~

- 命名约束

我们可以使用约束名称创建约束。例如以下代码：

.. sourcecode:: python+sql

    CheckConstraint("value > 5", name="myconstraint")

生成的结果为：

.. sourcecode:: sql

    CONSTRAINT myconstraint CHECK (value > 5)

- 匿名约束

以下两种方式都可以使用匿名约束：

  1. 直接将表达式作为约束。

  例如以下代码：

  .. sourcecode:: python+sql

      metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

      foo = Table("foo", metadata_obj, Column("value", Integer))

      CheckConstraint(foo.c.value > 5)

  生成的结果为：

  .. sourcecode:: sql

      CREATE TABLE foo (
          value INTEGER,
          CONSTRAINT ck_foo_value CHECK (value > 5)
      )

  2. 在表中定义约束并使用 :func:`_expression.column` 来表示这个约束中的值。

  例如以下代码：

  .. sourcecode:: python+sql

      from sqlalchemy import column

      metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

      foo = Table(
          "foo",
           metadata_obj,
           Column("value", Integer),
           CheckConstraint(column("value") > 5)
      )

  生成的结果同样为：

  .. sourcecode:: sql

      CREATE TABLE foo (
          value INTEGER,
          CONSTRAINT ck_foo_value CHECK (value > 5)
      )

- Schema（模式）类型约束

:class:`.SchemaType` 指的是可以生成与该类型一起附带的 CHECK 约束的类型对象，例如 :class:`.Boolean` 和 :class:`.Enum`。此时约束的名称可以通过发送参数“name”来直接设置。例如：

.. sourcecode:: python+sql

    Table("foo", metadata_obj, Column("flag", Boolean(name="ck_foo_flag")))

也可以将命名约束特性与这些类型结合使用，通常使用一个包含 ``%(constraint_name)s`` 的惯例，然后将一个名称应用于该类型。例如：

.. sourcecode:: python+sql

    metadata_obj = MetaData(
        naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
    )

    Table("foo", metadata_obj, Column("flag", Boolean(name="flag_bool")))

以上代码将会产生约束名为“ck_foo_flag_bool”的结果：

.. sourcecode::

    CREATE TABLE foo (
        flag BOOL,
        CONSTRAINT ck_foo_flag_bool CHECK (flag IN (0, 1))
    )

- 约束 API

可以使用以下类直接生成约束：

.. autoclass:: Constraint
    :members:
    :inherited-members:

.. autoclass:: ColumnCollectionMixin
    :members:

.. autoclass:: ColumnCollectionConstraint
    :members:
    :inherited-members:

.. autoclass:: CheckConstraint
    :members:
    :inherited-members:

.. autoclass:: ForeignKey
    :members:
    :inherited-members:

.. autoclass:: ForeignKeyConstraint
    :members:
    :inherited-members:

.. autoclass:: HasConditionalDDL
    :members:
    :inherited-members:

.. autoclass:: PrimaryKeyConstraint
    :members:
    :inherited-members:


.. autoclass:: UniqueConstraint
    :members:
    :inherited-members:

创建 Indexes
~~~~~~~~~~~~

对于单个列使用默认名称为 ``ix_<column label>`` 的匿名索引可以使用类似如下代码：

.. sourcecode:: python

    Column("some_column", String(50), index=True)

或者您可以使用 :class:`~sqlalchemy.schema.Index` 类来为索引分配更具描述性的名称。

下面我们使用 :class:`~sqlalchemy.schema.Index` 为表设置多个索引：

.. sourcecode:: python+sql

    metadata_obj = MetaData()
    mytable = Table(
        "mytable",
        metadata_obj,
        Column("col1", Integer, index=True),
        Column("col2", Integer, index=True, unique=True),
        Column("col3", Integer),
        Column("col4", Integer),
        Column("col5", Integer),
        Column("col6", Integer),
    )

    Index("idx_col34", mytable.c.col3, mytable.c.col4)
    Index("myindex", mytable.c.col5, mytable.c.col6, unique=True)

最后通过 mytable.create 命令生成 DDL 语句：

.. sourcecode:: python+sql

    mytable.create(engine)

生成的结果为：

.. sourcecode:: sql

    CREATE TABLE mytable (
      col1 INTEGER,
      col2 INTEGER,
      col3 INTEGER,
      col4 INTEGER,
      col5 INTEGER,
      col6 INTEGER
    )
    CREATE INDEX ix_mytable_col1 ON mytable (col1)
    CREATE UNIQUE INDEX ix_mytable_col2 ON mytable (col2)
    CREATE UNIQUE INDEX myindex ON mytable (col5, col6)
    CREATE INDEX idx_col34 ON mytable (col3, col4)

- 索引 API：

    .. autoclass:: Index
        :members:
        :inherited-members:

功能性索引
~~~~~~~~~~

:class:`.Index` 支持 SQL 和函数表达式，支持目标后端支持的函数表达式。例如，可以使用 :meth:`_expression.ColumnElement.desc` 修改符号来使用下降值对列进行索引：

.. sourcecode:: python

    from sqlalchemy import Index

    Index("someindex", mytable.c.somecol.desc())

或在支持功能性索引的后端（如 PostgreSQL）中，可以使用 ``lower()`` 函数创建不区分大小写的索引：

.. sourcecode:: python

    from sqlalchemy import func, Index

    Index("someindex", func.lower(mytable.c.somecol))