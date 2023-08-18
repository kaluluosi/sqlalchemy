.. _metadata_constraints_toplevel:
.. _metadata_constraints:

.. currentmodule:: sqlalchemy.schema

================================
定义约束和索引
================================

本节将介绍SQL  :term:`constraints`  和索引。在SQLAlchemy中，关键类包括  :class:` _schema.ForeignKeyConstraint` .Index`。

.. _metadata_foreignkeys:

定义外键
---------------------

在SQL中，*foreign key* 是一个表级构造，它约束该表中的一个或多个列，只允许存在于不同列集中的值（通常但不总是位于不同的表中）。我们称被约束的列为*foreign key*列，被约束至的列称为*referenced*列。被约束列通常定义为其拥有表（但有例外情况）的主键，外键是连接有关联的成对行的“联合”，SQLAlchemy赋予这个概念在其操作的几乎每个领域中非常重要的地位。

在SQLAlchemy中，以及在DDL中，外键约束可以被定义为表子句中的附加属性，对于单列外键，它们可以在单列的定义中指定。单列外键更常见，在列级别上，是通过构造  :class:`~sqlalchemy.schema.ForeignKey` ~sqlalchemy.schema.Column` 对象的参数来指定的::

    user_preference = Table(
        "user_preference",
        metadata_obj,
        Column("pref_id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
        Column("pref_name", String(40), nullable=False),
        Column("pref_value", String(100)),
    )

以上，我们定义了一个名为``user_preference``的新表。该表的每一行都必须包含一个与``user``表的``user_id``的列值相对应的值。

  :class:`~sqlalchemy.schema.ForeignKey` ~sqlalchemy.schema.Column` 对象，我们将在稍后看到，它可以通过其``c``集合从现有的 :class:`~sqlalchemy.schema.Table` 对象中访问::

    ForeignKey(user.c.user_id)

使用字符串的优点是当第一次需要时才会解决``user``和``user_preference``之间在Python中的链接，因此表对象可以轻松地分布在多个模块中，并且可以以任何顺序定义。

使用  :class:`~sqlalchemy.schema.ForeignKeyConstraint` ` invoice``::

    invoice = Table(
        "invoice",
        metadata_obj,
        Column("invoice_id", Integer, primary_key=True),
        Column("ref_num", Integer, primary_key=True),
        Column("description", String(60), nullable=False),
    )

然后，定义一个具有复合外键的表``invoice_item``，该复合外键引用``invoice``::

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

需要注意的是  :class:`~sqlalchemy.schema.ForeignKeyConstraint` ` invoice_item.invoice_id``和``invoice_item.ref_num``列上存储单独的 :class:`~sqlalchemy.schema.ForeignKey` 对象，但SQLAlchemy将不知道这两个值应该配对在一起，而不是两个单独的外键，而是一个包含两个列引用的复合外键。

.. _use_alter:

通过ALTER创建/删除外键约束
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在DDL中使用外键的行为通常意味着将约束“内联”在CREATE TABLE语句中，例如：

.. sourcecode:: sql

    CREATE TABLE addresses (
        id INTEGER NOT NULL,
        user_id INTEGER,
        email_address VARCHAR NOT NULL,
        PRIMARY KEY (id),
        CONSTRAINT user_id_fk FOREIGN KEY(user_id) REFERENCES users (id)
    )

使用“CONSTRAINT..FOREIGN KEY”指令在创建表定义中“内联”创建约束。 :meth：`_schema.MetaData.create_all`和  :meth:`_schema.MetaData.drop_all`  方法默认使用它，使用所有涉及:class :` _schema.Table`对象的拓扑排序，使得表按其外键依赖性的顺序进行创建和删除（此排序也可通过  :attr:`_schema.MetaData.sorted_tables`  访问器获得）。

这种方法在两个或多个外键约束涉及“依赖关系周期”的情况下不起作用，其中一组表相互依赖，假设后端强制执行外键（SQLite、MySQL/MyISAM除外）。因此，这两种方法将在所有未支持大部分ALTER的后端上将循环中的约束分离成单独的ALTER语句。给定类似于下面的模式：

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

在PostgreSQL后端中调用  :meth:`_schema.MetaData.create_all`  时，这两个表之间的循环将被解决，并且约束将分别被创建：

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

在发出DROP的情况下，相同的逻辑也适用，不过请注意，在SQL中，发出DROP CONSTRAINT需要约束有一个名称。在上面的“node”表的情况下，我们没有为这个约束命名；因此，系统将仅尝试发出DROP名称约束时才存在的约束：

.. sourcecode:: pycon+sql

    >>> with engine.connect() as conn:
    ...     metadata_obj.drop_all(conn, checkfirst=False)
    {execsql}ALTER TABLE element DROP CONSTRAINT fk_element_parent_node_id
    DROP TABLE node
    DROP TABLE element
    {stop}


如果循环无法解析，例如我们在这里没有应用名称到任何一个约束时，我们会收到以下错误：

.. sourcecode:: text

    sqlalchemy.exc.CircularDependencyError: Can't sort tables for DROP;
    an unresolvable foreign key dependency exists between tables:
    element, node.  Please ensure that the ForeignKey and ForeignKeyConstraint
    objects involved in the cycle have names so that they can be dropped
    using DROP CONSTRAINT.

这个错误仅适用于DROP案例，因为我们可以在CREATE案例中发出“ADD CONSTRAINT”而不需要名称；数据库通常会自动赋一个名称。

当使用DROP操作时，  :paramref:`_schema.ForeignKeyConstraint.use_alter`  和  :paramref:` _schema.ForeignKey.use_alter`  关键字参数可以用于手动解决依赖关系循环。我们可以将此标志仅添加到“element”表中，如下所示：

    element = Table(
        "element",
        metadata_obj,
        Column("element_id", Integer, primary_key=True),
        Column("parent_node_id", Integer),
        ForeignKeyConstraint(
            ["parent_node_id"],
            ["node.node_id"],
            use_alter=True,
            name="fk_element_parent_node_id",
        ),
    )

在我们的CREATE DDL中，我们只会看到此约束的ALTER语句，而不是其他约束：

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
        PRIMARY KEY (node_id),
        FOREIGN KEY(primary_element) REFERENCES element (element_id)
    )

    ALTER TABLE element ADD CONSTRAINT fk_element_parent_node_id
    FOREIGN KEY(parent_node_id) REFERENCES node (node_id)
    {stop}

  :paramref:`_schema.ForeignKeyConstraint.use_alter`  和  :paramref:` _schema.ForeignKey.use_alter`  与drop操作一起使用时，将要求约束有一个名称，否则会生成以下错误：

.. sourcecode:: text

    sqlalchemy.exc.CompileError: Can't emit DROP CONSTRAINT for constraint
    ForeignKeyConstraint(...); it has no name

.. seealso::

      :ref:`约束命名约定` 

      :func:`.sort_tables_and_constraints` 

.. _on_update_on_delete:

ON UPDATE和ON DELETE
~~~~~~~~~~~~~~~~~~~~~~~

大多数数据库支持外键值的*级联*，这意味着当父行被更新时，新值将放置在子行中，或者当父行被删除时，所有对应的子行都将设置为null或删除。在数据定义语言中，可以在外键约束中使用短语，例如“ON UPDATE CASCADE”，“ON DELETE CASCADE”和“ON DELETE SET NULL”，与外键约束相对应。在  :class:`~sqlalchemy.schema.ForeignKey` ~sqlalchemy.schema.ForeignKeyConstraint` 对象中可以通过``onupdate``和``ondelete``关键字参数生成此子句。该值是任何字符串，该字符串将在适当的“ON UPDATE”或“ON DELETE”短语后输出::

    child = Table(
        "child",
        metadata_obj,
        Column(
            "id",
            Integer,
            ForeignKey("parent.id", onupdate="CASCADE", ondelete="CASCADE"),
            primary_key=True,
        ),
    )

    composite = Table(
        "composite",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("rev_id", Integer),
        Column("note_id", Integer),
        ForeignKeyConstraint(
            ["rev_id", "note_id"],
            ["revisions.id", "revisions.note_id"],
            onupdate="CASCADE",
            ondelete="SET NULL",
        ),
    )

请注意，这些子句需要在使用MySQL时使用``InnoDB``表。它们也可能在其他数据库上不受支持。

.. seealso::

    关于ORM  :func:`_orm.relationship` 结构与“ON DELETE CASCADE”的集成的背景，请参见以下部分：

      :ref:`passive_deletes` 

      :ref:`passive_deletes_many_to_many` 

.. _schema_unique_constraint:

唯一约束
-----------------

可以使用  :class:`~sqlalchemy.schema.Column` ~sqlalchemy.schema.UniqueConstraint` 表级结构显式命名唯一约束和/或具有多列的约束。

.. sourcecode:: python+sql

    from sqlalchemy import UniqueConstraint

    metadata_obj = MetaData()
    mytable = Table(
        "mytable",
        metadata_obj,
        # Per-column anonymous unique constraint.
        Column("col1", Integer, unique=True),
        Column("col2", Integer),
        Column("col3", Integer),
        # Explicit/composite unique constraint.  'name' is optional.
        UniqueConstraint("col2", "col3", name="uix_1"),
    )

检查约束
----------------

检查约束可以命名或未命名，并且可以在列或表级别上创建，使用 :class:`~sqlalchemy.schema.CheckConstraint` 结构。检查约束的文本直接通过到数据库，因此有限制的“数据库无关”行为。列级别的检查约束通常只能引用其所放置的列，而表级别约束可以引用表中的任何列。

请注意，某些数据库不支持主动支持约束，例如旧版的MySQL（8.0.16之前）。

.. sourcecode:: python+sql

    from sqlalchemy import CheckConstraint

    metadata_obj = MetaData()
    mytable = Table(
        "mytable",
        metadata_obj,
        # per-column CHECK constraint
        Column("col1", Integer, CheckConstraint("col1>5")),
        Column("col2", Integer),
        Column("col3", Integer),
        # table level CHECK constraint.  'name' is optional.
        CheckConstraint("col2 > col3 + 5", name="check1"),
    )

    mytable.create(engine)
    {execsql}CREATE TABLE mytable (
        col1 INTEGER  CHECK (col1>5),
        col2 INTEGER,
        col3 INTEGER,
        CONSTRAINT check1  CHECK (col2 > col3 + 5)
    ){stop}

主键约束
----------------------

任何  :class:`_schema.Table`  标志。  :class:` .PrimaryKeyConstraint`对象提供了对此约束的显式访问，其中包括直接进行配置的选项::

    from sqlalchemy import PrimaryKeyConstraint

    my_table = Table(
        "mytable",
        metadata_obj,
        Column("id", Integer),
        Column("version_id", Integer),
        Column("data", String(50)),
        PrimaryKeyConstraint("id", "version_id", name="mytable_pk"),
    )

.. seealso::

      :class:`.PrimaryKeyConstraint`  - 详细API文档。

使用Declarative ORM扩展时设置约束
---------------------------------------------------------------

 :class:`_schema.Table` 是SQLAlchemy Core构造，它允许定义表元数据，其中包括可以由SQLAlchemy ORM用作映射类的目标。  :ref:`Declarative <declarative_toplevel>` 扩展程序允许自动创建 :class:`_schema.Table` 对象，根据主要是 :class:`_schema.Column` 对象的映射内容。

要将表级别约束对象例如  :class:`_schema.ForeignKeyConstraint` ` __table_args__``属性，如 :ref:`declarative_table_args` 中所述。

.. _constraint_naming_conventions:

配置约束命名约定
-----------------------------------------

关系数据库通常对所有约束和索引分配显式名称。在使用``CREATE TABLE``创建表的常见情况下，例如CHECK，UNIQUE和主键约束内联在表定义中生成，如果没有另外指定名称，则数据库通常会有一个系统以自动分配名称的方式。在使用诸如``ALTER TABLE``之类的命令对现有数据库表进行更改时，此命令通常需要为新约束指定显式名称，以及能够指定要删除或修改的现有约束的名称。

可以使用事件来构建自动命名方案。这种方法的优点是在代码中无需显式名称参数即可收到约束的一致命名方案，而且该约定还适用于使用：paramref：`_schema.Column.unique`和：paramref：`_schema.Column.index`参数创建的约束和索引。从SQLAlchemy 0.9.2开始，该基于事件的方法已包括在内，可以使用参数：paramref：`_schema.MetaData.naming_convention`进行配置。

配置  :paramref:`_schema.MetaData.naming_convention` .Index` 类或单个约束类作为键，并接受Python字符串模板作为值。它还接受一系列字符串代码作为备用密钥，例如“fk”，“pk”，“ix”，“ck”，“uq”，用于外键，主键，索引，检查和唯一约束，例如。在这个字典中使用的字符串模板，是在该 :class:`_schema.MetaData` 对象关联到的约束或索引没有现有名称时使用的。

适用于所有约束的示例命名约定如下：

    convention = {
        "ix": "ix_%(column_0_label)s",
        "uq": "uq_%(table_name)s_%(column_0_name)s",
        "ck": "ck_%(table_name)s_%(constraint_name)s",
        "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
        "pk": "pk_%(table_name)s",
    }

    metadata_obj = MetaData(naming_convention=convention)

上面的约定将为目标  :class:`_schema.MetaData` .UniqueConstraint` 时产生的名称：

    >>> user_table = Table(
    ...     "user",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30), nullable=False),
    ...     UniqueConstraint("name"),
    ... )
    >>> list(user_table.constraints)[1].name
    'uq_user_name'

即使我们只使用了：paramref:`_schema.Column.unique`标志，此功能也会生效：

    >>> user_table = Table(
    ...     "user",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30), nullable=False, unique=True),
    ... )
    >>> list(user_table.constraints)[1].name
    'uq_user_name'

使用命名约定的一个重要优点是，在Python构造时建立了名称，而不是在DDL发出时建立名称。使用Alembic的“—autogenerate”功能时，该名称约定在新的移动脚本生成时将是明确的::

    def upgrade():
        op.create_unique_constraint("uq_user_name", "user", ["name"])

上面的“uq_user_name”字符串是从“—autogenerate”在我们的元数据中找到的 :class:`.UniqueConstraint` 对象复制的。

可用的标记包括``%(table_name)s``，``%(referred_table_name)s``，``%(column_0_name)s``，``%(column_0_label)s``，``%(column_0_key)s``，``%(referred_column_0_name)s``和``%(constraint_name)s``，以及每个标记的多列版本，包括 ``%(column_0N_name)s``，``%(column_0_N_name)s``，``%(referred_column_0_N_name)s``，它们将所有列名分隔或不分隔线呈现。  :paramref:`_schema.MetaData.naming_convention`  文档详细介绍了每个约定的其他细节。

.. _constraint_default_naming_convention:

默认命名约定
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :paramref:`_schema.MetaData.naming_convention`  的默认值处理了长期存在的SQLAlchemy行为，即为使用  :param:` _schema.Column.index`  参数创建的 :class:`.Index` 对象分配名称::

    >>> from sqlalchemy.sql.schema import DEFAULT_NAMING_CONVENTION
    >>> DEFAULT_NAMING_CONVENTION
    immutabledict({'ix': 'ix_%(column_0_label)s'})

长名称的截断
~~~~~~~~~~~~~~~~~~~~~~~~~

当生成的名称，特别是使用多列标记的那些名称，超出目标数据库的标识符长度限制时（例如，PostgreSQL有一个63个字符的限制），这个名称将根据长名称的md5散列进行确定性截断，该名称的4个字符后缀。例如，下面的命名约定将根据使用的列名称生成非常长的名称::

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

在PostgreSQL dialect上，超过63个字符的名称将被截断，如下面的示例：

.. sourcecode:: sql

    CREATE TABLE long_names (
        information_channel_code INTEGER,
        billing_convention_name INTEGER,
        product_identifier INTEGER,
        CONSTRAINT uq_long_names_information_channel_code_billing_conventi_a79e
        UNIQUE (information_channel_code, billing_convention_name, product_identifier)
    )

上面的后缀“a79e”是基于长名称的md5哈希而确定的，并且将为给定模式生成相同的值。

为命名约定创建自定义标记
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以通过在:named-param:`naming_convention`字典中指定其他令牌和可调用来添加新令牌。例如，如果我们想使用GUID方案命名外键约束，我们可以这样做：

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

上面的，当我们创建一个新的 :class:`_schema.ForeignKeyConstraint` 时，我们将获得以下名称：

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

.. seealso::

     :paramref:`_schema.MetaData.naming_convention`  - 用于附加使用详细信息的其他用法说明
    以及所有可用的命名组件。

    `The Importance of Naming Constraints <https://alembic.sqlalchemy.org/en/latest/naming.html>`_ - 位于Alembic文档中


.. versionadded:: 1.3.0添加了多列命名标记，例如“%(column_0_N_name)”。
   超出目标数据库的字符限制的生成名称将被确定性截断。

.. _naming_check_constraints:

命名检查约束
~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`.CheckConstraint` .CheckConstraint` 的常见约定是我们期望该对象已经有一个名称，然后我们使用其他约定元素将其增强。典型的约定是“ck_%（table_name）s_%（constraint_name）s”::


    metadata_obj = MetaData(
        naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
    )

    Table(
        "foo",
        metadata_obj,
        Column("value", Integer),
        CheckConstraint("value > 5", name="value_gt_5"),
    )

上面的表会产生名称“ck_foo_value_gt_5”：

.. sourcecode:: sql

    CREATE TABLE foo (
        value INTEGER,
        CONSTRAINT ck_foo_value_gt_5 CHECK (value > 5)
    )

  :class:`.CheckConstraint` ` ％（columns_0_name）s``令牌；我们可以使用这个令牌，通过在约束的表达式中确保我们使用：class:`_schema.Column`或 :func:`_expression.column` 元素，对其执行操作，将约束与表分离：

配置 Check 约束和 Naming
--------------------------

在定义表结构时，可以通过   :class:`.CheckConstraint`  类和约束名，来添加 ` `CHECK`` 条件约束。可以用两种方式来指定 CHECK 约束的名字，一种是手动指定，以字符串形式传入。另一种是使用 SQLAlchemy 的命名惯例。

一种方法是使用   :class:`.CheckConstraint`  类::

    metadata_obj = MetaData()
    foo = Table("foo", metadata_obj, Column("value", Integer))
    CheckConstraint(foo.c.value > 5, name="foo_value_check")

另一种方法是使用   :class:`.CheckConstraint`  类和 SQLAlchemy 的命名惯例，以下代码将产生名为 ` `ck_foo_value`` 的 CHECK 约束：

.. sourcecode:: python

    metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})
    foo = Table("foo", metadata_obj, Column("value", Integer))
    CheckConstraint(foo.c.value > 5)

或者使用内联的   :func:`_expression.column`  函数::

    from sqlalchemy import column

    metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

    foo = Table(
        "foo", metadata_obj, Column("value", Integer), CheckConstraint(column("value") > 5)
    )

两种方法都会产生名为 ``ck_foo_value`` 的约束：

.. sourcecode:: sql

    CREATE TABLE foo (
        value INTEGER,
        CONSTRAINT ck_foo_value CHECK (value > 5)
    )

程序会扫描所给表达式中存在的列对象，来确定“第 0 列”的名称。如果表达式中存在多个列，则使用确定性搜索扫描，但表达式的结构将决定哪个列将被标示为“第 0 列”。

.. _naming_schematypes:

配置布尔值、枚举值和其他模式类型的命名
----------------------------------------

  :class:`.SchemaType`  类是指代一些类型信息的类，如  :class:` .Boolean` 。这些类生成了一个与类型相应的 ``CHECK`` 条件约束。这里约束的名字可以通过直接发送 "name" 参数来设定，例如使用  :paramref:`.Boolean.name`  中的代码：

    Table("foo", metadata_obj, Column("flag", Boolean(name="ck_foo_flag")))

缺省情况下，为这些类型设定的名字是依靠所设定的 "naming_convention" 来完成的。

这些类型也可以和命名惯例一起使用，为其应用约定本身（通常是包含 ``%(constraint_name)s`` 的命名约定），然后再给类型应用一个名字：

    metadata_obj = MetaData(
        naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
    )

    Table("foo", metadata_obj, Column("flag", Boolean(name="flag_bool")))

上面的表会产生名为 ``ck_foo_flag_bool`` 的约束：

.. sourcecode:: sql

    CREATE TABLE foo (
        flag BOOL,
        CONSTRAINT ck_foo_flag_bool CHECK (flag IN (0, 1))
    )

  :class:`.SchemaType`  类使用特殊的内部符号，以便约定仅在 DDL 编译时确定。在 PostgreSQL 上，有一个原生的 BOOLEAN 类型，因此   :class:` .Boolean`  的 ``CHECK`` 约束是不必要的；即使在定义检查约束的命名约定的情况下，我们也可以安全地设置一个未命名的   :class:`.Boolean`  类型。只有在运行在诸如 SQLite 或 MySQL 等没有原生 BOOLEAN 类型的数据库中时，才会参考这个约定来确定 CHECK 约束。

在调用   :class:`.SchemaType`  中定义约束时，也可以使用 ` `column_0_name`` 标记:

    metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})
    foo = Table("foo", metadata_obj, Column("flag", Boolean()))

上述表结构会产生如下 DDL：

.. sourcecode:: sql

    CREATE TABLE foo (
        flag BOOL,
        CONSTRAINT ck_foo_flag CHECK (flag IN (0, 1))
    )

约束 API
--------

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


.. autofunction:: sqlalchemy.schema.conv

.. _schema_indexes:

索引
----

可以使用 ``index`` 关键词来为单个列创建匿名索引（使用自动生成的名称 ``ix_<column label>``）。``unique`` 的使用方式则将唯一性应用到该索引本身，而不是添加单独的 ``UNIQUE`` 约束。对于具有特定名称或涵盖多个列的索引，请使用   :class:`~sqlalchemy.schema.Index`  结构，该结构需要名称。

以下是一个   :class:`~sqlalchemy.schema.Table`  示例，包含了多个关联的   :class:` ~sqlalchemy.schema.Index`  对象。"CREATE INDEX" 的 DDL 语句会在表创建语句后立即发出：

.. sourcecode:: python+sql

    metadata_obj = MetaData()
    mytable = Table(
        "mytable",
        metadata_obj,
        # an indexed column, with index "ix_mytable_col1"
        Column("col1", Integer, index=True),
        # a uniquely indexed column with index "ix_mytable_col2"
        Column("col2", Integer, index=True, unique=True),
        Column("col3", Integer),
        Column("col4", Integer),
        Column("col5", Integer),
        Column("col6", Integer),
    )

    # place an index on col3, col4
    Index("idx_col34", mytable.c.col3, mytable.c.col4)

    # place a unique index on col5, col6
    Index("myindex", mytable.c.col5, mytable.c.col6, unique=True)

    mytable.create(engine)
    {execsql}CREATE TABLE mytable (
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
    CREATE INDEX idx_col34 ON mytable (col3, col4){stop}

请注意，在上面的示例中，使用   :class:`.Index`  结构在与其对应的表之外创建，直接使用   :class:` _schema.Column`  对象。  :class:`.Index`  也支持在   :class:` _schema.Table`  中"内联"定义，并使用字符串作为列的标识符：

    metadata_obj = MetaData()
    mytable = Table(
        "mytable",
        metadata_obj,
        Column("col1", Integer),
        Column("col2", Integer),
        Column("col3", Integer),
        Column("col4", Integer),
        # place an index on col1, col2
        Index("idx_col12", "col1", "col2"),
        # place a unique index on col3, col4
        Index("idx_col34", "col3", "col4", unique=True),
    )

  :class:`~sqlalchemy.schema.Index`  对象也支持自己的 ` `create()`` 方法：

.. sourcecode:: python+sql

    i = Index("someindex", mytable.c.col5)
    i.create(engine)
    {execsql}CREATE INDEX someindex ON mytable (col5){stop}

函数式索引
----------

  :class:`.Index`  支持所有 SQL 和函数表达式，具体取决于目标后端。可以使用  :meth:` _expression.ColumnElement.desc`  以降序修改所要使用的列，对该列创建索引：

    from sqlalchemy import Index

    Index("someindex", mytable.c.somecol.desc())

或在后端支持函数索引，例如 PostgreSQL，可以使用 ``lower()`` 函数来创建“不区分大小写”的索引：

    from sqlalchemy import func, Index

    Index("someindex", func.lower(mytable.c.somecol))

索引 API
--------

.. autoclass:: Index
    :members:
    :inherited-members: