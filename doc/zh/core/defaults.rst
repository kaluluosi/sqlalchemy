.. currentmodule:: sqlalchemy.schema

.. _metadata_defaults_toplevel:

.. _metadata_defaults:

列的INSERT/UPDATE默认值
=======================

列的INSERT和UPDATE默认值是指函数，用于在针对指定行的INSERT或UPDATE语句执行过程中，为该行中的特定列**创建默认值**，在该列的INSERT或UPDATE语句中**没有提供值时**使用。换言之，如果一张表有一个名为“timestamp”的列，而一个INSERT语句执行时没有包含这个列的值，那么INSERT默认就会创建一个新值，如当前时间，该值用作要INSERT到“timestamp”列中的值。如果语句 *确实* 包含该列的值，那么默认值就**不会**起作用。

列默认值可以是在 :term:`DDL` 中与数据库一起定义的服务器端函数或常量值；也可以是在SQLAlchemy中直接呈现到INSERT或UPDATE语句中的SQL表达式；也可以是在数据传递到数据库之前由SQLAlchemy调用的客户端端Python函数或常量值。

.. note::

    列默认处理程序不应混淆与拦截和修改输入到INSERT和UPDATE语句中的值的构造方法。
    这就是 :term:`数据交换`，其中应用程序以某种方式修改列值，然后将其发送到数据库。
    SQLAlchemy提供了一些实现这一目的的方法，包括使用 :ref:`自定义数据类型
    <types_typedecorator>`、:ref:`SQL执行事件 <core_sql_events>`和
    在ORM中 :ref:`自定义验证器<simple_validators>`，以及
    :ref:`属性事件 <orm_attribute_events>`。只有当在SQL
    :term:`DML` 语句中**不存在任何值**的情况下，才会调用列默认值。


SQLAlchemy提供了一系列有关INSERT和UPDATE语句中不存在值时默认生成函数的特性。选项包括：

* 标量值用于INSERT和UPDATE操作期间的默认值
* 执行于INSERT和UPDATE操作期间的Python函数
* 嵌入在INSERT语句中的SQL表达式（或在某些情况下提前执行）
* 嵌入在UPDATE语句中的SQL表达式
* 用于INSERT的服务器端默认值
* 用于UPDATE的服务器端触发器标记

对于所有insert/update默认值的一般规则是，如果没有对于特定列传递参数，那么它们才会生效；否则，它们指定的值将被使用。

标量默认值
----------

最简单的默认值是标量值，用作列的默认值::

    Table("mytable", metadata_obj, Column("somecolumn", Integer, default=12))

上面，如果没有提供其他值，则将值“12”绑定为INSERT操作期间的列值。

标量值也可以与UPDATE语句关联，但这不是很常见（因为UPDATE语句通常是动态查找默认值）::

    Table("mytable", metadata_obj, Column("somecolumn", Integer, onupdate=25))

Python执行函数
-------------------------

:paramref:`_schema.Column.default`和 :paramref:`_schema.Column.onupdate`关键字参数也接受Python函数。如果未为该列提供其他值，则在插入或更新时调用这些函数，并使用返回的值作为该列的值。下面给出了将一个粗略的“序列”分配到主键列的示例::

    # 一个简单的计数器函数
    i = 0


    def mydefault():
        global i
        i += 1
        return i


    t = Table(
        "mytable",
        metadata_obj,
        Column("id", Integer, primary_key=True, default=mydefault),
    )

应该指出的是，对于真正的“递增顺序”行为，通常应该使用数据库的内置功能，这可能包括序列对象或其他自动增量功能。对于主键列，SQLAlchemy在大多数情况下将自动使用这些功能。有关 :class:`~sqlalchemy.schema.Column` 的API文档的详细信息，请参见 :paramref:`_schema.Column.autoincrement` 标志，以及本章后面关于 :class:`~sqlalchemy.schema.Sequence` 的部分，了解标准主键生成技术的背景。

为了说明对于`onupdate`，我们将Python `datetime`函数中的`now`分配给 :paramref:`_schema.Column.onupdate`属性::

    import datetime

    t = Table(
        "mytable",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        # 定义‘last_updated’应使用datetime.now()填充
        Column("last_updated", DateTime, onupdate=datetime.datetime.now),
    )

执行更新语句并且对于“last_updated”列没有传递值时，将执行`datetime.datetime.now()` Python函数，并将其返回值用作“last_updated”的值。请注意，我们将函数本身（`now`）提供为，而不是调用它（即，在函数名称后没有括号）。SQLAlchemy将在执行语句时执行函数。

.. _context_default_functions:

上下文敏感的默认函数
~~~~~~~~~~~~~~~~~~~~~~~~~

:paramref:`_schema.Column.default`和 :paramref:`_schema.Column.onupdate`使用的Python函数
在插入或更新时也可以使用当前语句的上下文来确定值。语句的上下文是一个内部的SQLAlchemy对象，它包含有关正在执行的语句的所有信息，包括其源表达式、与之相关的参数和游标。带有默认生成的上下文通常用于访问要在行上插入或更新的其他值。是使用一个接受单个 `context` 参数的函数提供上下文来访问上下文信息。

    def mydefault(context):
        return context.get_current_parameters()["counter"] + 12


    t = Table(
        "mytable",
        metadata_obj,
        Column("counter", Integer),
        Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
    )

上面的默认生成函数应用于否则没有提供“counter_plus_twelve”值的所有INSERT和UPDATE语句，其值将为插入或更新语句中对于“counter”列的任何值，加上数字12。

对于使用“executemany”样式执行的单个语句（例如使用:meth:`_engine.Connection.execute`向其传递多个参数集的语句）的用例，用户定义的函数会为每个参数集调用一次。对于一个包含多个VALUES子句的 :class:`_expression.Insert` 构造的用例（例如通过 :meth:`_expression.Insert.values` 方法设置了多个值子句的情况），为每个参数集调用用户定义的函数。

当调用该函数时，上下文对象（ :class:`.DefaultExecutionContext` 的子类）可以从上下文对象使用特殊方法:meth:`.DefaultExecutionContext.get_current_parameters`可以获得。该方法返回一个字典，其中包含将列键映射到INSERT或UPDATE语句的值。对于多值的INSERT构造，对应于单个VALUES子句的参数子集从完整参数字典中隔离并单独返回。

.. versionadded:: 1.2

    添加了:meth:`.DefaultExecutionContext.get_current_parameters` 方法，它比仍然存在的 :attr:`.DefaultExecutionContext.current_parameters` 属性更为出色，它提供了多个VALUES子句的服务，将其组织成单个参数字典。

.. _defaults_client_invoked_sql:

客户端调用的SQL表达式
-----------------------------------

:paramref:`_schema.Column.default`和:paramref:`_schema.Column.onupdate`关键字也可以传递SQL表达式，这些表达式在大多数情况下嵌入到INSERT或UPDATE语句中：

    t = Table(
        "mytable",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        # define 'create_date' to default to now()
        Column("create_date", DateTime, default=func.now()),
        # define 'key' to pull its default from the 'keyvalues' table
        Column(
            "key",
            String(20),
            default=select(keyvalues.c.key).where(keyvalues.c.type="type1"),
        ),
        # define 'last_modified' to use the current_timestamp SQL function on update
        Column("last_modified", DateTime, onupdate=func.utc_timestamp()),
    )

上面，如果没有来自其他值，则“create_date”列将用“现在()”SQL函数的结果填充（取决于后端，在大多数情况下，将其编译成“NOW()”或“CURRENT_TIMESTAMP”）。在“key”列中，该结果来自其他表的SELECT子查询。当更新此表时，将使用SQL“UTC_TIMESTAMP()” MySQL函数中的值填充“last_modified”列。

.. note::

    使用 :attr:`.func` 构造与 SQL 函数一起使用时，我们使用名称函数，例如括号内的 :func:`func.now()`。这与指定Python可调用对象的情况不同，例如 :func:`datetime.datetime` ，其中传递函数本身，但是不调用函数自身。在使用SQL函数的情况下，调用 :func:`func.now()` 返回将 NOW 函数呈现到正在发出的 SQL 中的 SQL表达式对象。

由 :paramref:`_schema.Column.default` 和 :paramref:`_schema.Column.onupdate` 指定的默认值和更新SQL表达式在发生 INSERT 或 UPDATE 语句时显式调用 SQLALchemy 处理，通常嵌入 INSERT 或 UPDATE 语句中，除非在下面列出的某些情况下进行处理。这与作为表DDL定义的“服务器端”默认值不同，例如作为“CREATE TABLE”语句的一部分，这更常见。有关服务器端默认值，请参见下一部分 :ref:`server_defaults`。

当使用SQL表达式指示 :paramref:`_schema.Column.default` 时，如果要在主键列上使用SQL函数，则在某些情况下，SQLAlchemy 必须“预先执行”默认生成SQL函数，即在单独的 SELECT 语句中调用它，并将结果作为参数传递给 INSERT。这仅适用于希望返回此主键值的 INSERT 语句中的主键列，因为不能使用 RETURNING 或 `cursor.lastrowid`。对于指定 :paramref:`~.expression.insert.inline` 标志的 :class:`_expression.Insert` 构造始终会内联渲染默认表达式。

当用单个参数集执行语句时（即它不是 "executemany" 样式执行），返回的 :class:`~sqlalchemy.engine.CursorResult` 将包含一个通过 :meth:`_engine.CursorResult.postfetch_cols` 访问的集合，其中包含所有具有内联执行的默认源的 :class:`~sqlalchemy.schema.Column` 对象。同样，绑定到语句的所有参数，包括所有Python和SQL表达式以及预执行的参数，均存在于:meth:`_engine.CursorResult.last_inserted_params` 或 :meth:`_engine.CursorResult.last_updated_params` 集合中。 :attr:`_engine.CursorResult.inserted_primary_key` 集合包含插入行的主键值（列表，以便从单列和复合列主键以相同格式表示）。

.. _server_defaults:

DDL显式默认表达式
------------------------

SQL表达式默认值的一个变种是 :paramref:`_schema.Column.server_default`，它在 :meth:`_schema.Table.create` 操作中放置到CREATE TABLE中：

.. sourcecode:: python+sql

    t = Table(
        "test",
        metadata_obj,
        Column("abc", String(20), server_default="abc"),
        Column("created_at", DateTime, server_default=func.sysdate()),
        Column("index_value", Integer, server_default=text("0")),
    )

对于上述表的create调用，将生成：

.. sourcecode:: sql

    CREATE TABLE test (
        abc varchar(20) default 'abc',
        created_at datetime default sysdate,
        index_value integer default 0
    )

上面的示例说明了 :paramref:`_schema.Column.server_default` 的两种典型用法，即SQL
函数（上例中的 SYSDATE）和服务器端常量值（上面的整数“0”）。使用 :func:`_expression.text` 构造将任何字面SQL值作为一个文本传递而不是传递原始值是可取的。

与客户会生成表达式一样，:paramref:`_schema.Column.server_default` 可以容纳任意SQL表达式。然而，通常情况下，这些将是简单的函数和表达式，而不是嵌入SELECT之类的复杂情况。

.. _triggered_columns:

标记隐式生成的值、Timestamp 和激发列
-------------------------------------------------------------------

根据其他基于服务器端数据库机制生成新值的列（如某些平台的TIMESTAMP列）以及在插入或更新期间调用的自定义触发器，生成新值的列的默认值，可以通过:class:`.FetchedValue` 作为标记进行调用：

    from sqlalchemy.schema import FetchedValue

    t = Table(
        "test",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("abc", TIMESTAMP, server_default=FetchedValue()),
        Column("def", String(20), server_onupdate=FetchedValue()),
    )

:class:`.FetchedValue` 不影响CREATE TABLE的呈现DDL。相反，将其标记为将在INSERT或UPDATE语句的过程中使用数据库生成新值的列，并且可以被支持的数据库用作表格在RETURNING或 OUTPUT 子句中。工具例如SQLAlchemy ORM 在知道如何访问这个标记后使用这个标记。具体来说，可以使用 :meth:`.ValuesBase.return_defaults` 方法与 :class:`_expression.Insert` 或 :class:`_expression.Update` 构造一起使用以指示应返回这些值。

有关在ORM中使用 :class:`.FetchedValue` 的详细信息，请参阅 :ref:`orm_server_defaults`。

.. warning::  :paramref:`_schema.Column.server_onupdate` 指令不能直接产生MySQL的“ON UPDATE CURRENT_TIMESTAMP（）”子句。请参见 :ref:`mysql_timestamp_onupdate` 了解如何生成此子句。

.. seealso::

    :ref:`orm_server_defaults`

.. _defaults_sequences:

定义序列
--------

SQLAlchemy对数据库序列使用:class:`~sqlalchemy.schema.Sequence`对象，它被认为是列默认值的一种特殊情况。它仅影响带有明确序列支持的数据库，这些数据库包括SQLAlchemy所包含的方言之一，包括PostgreSQL、Oracle、MS SQL Server和MariaDB。否则，将忽略 :class:`~sqlalchemy.schema.Sequence`对象。

:class:`~sqlalchemy.schema.Sequence`可以作为"default"生成器被放置在任何列上，用于INSERT操作，并且如果需要的话也可以在UPDATE操作期间触发。它通常与单个整数主键列配对使用：

    table = Table(
        "cartitems",
        metadata_obj,
        Column(
            "cart_id",
            Integer,
            Sequence("cart_id_seq", start=1),
            primary_key=True,
        ),
        Column("description", String(40)),
        Column("createdate", DateTime()),
    )

在上面的示例中，表`cartitems`与名为`cart_id_seq`的序列相关联。发出:meth:`.MetaData.create_all`用于以上表将包括：

.. sourcecode:: sql

    CREATE SEQUENCE cart_id_seq START WITH 1

    CREATE TABLE cartitems (
      cart_id INTEGER NOT NULL,
      description VARCHAR(40),
      createdate TIMESTAMP WITHOUT TIME ZONE,
      PRIMARY KEY (cart_id)
    )

.. tip::

  当使用具有明确模式名称的表时（详见 :ref:`schema_table_schema_name`）， :class:`.Sequence`  的配置模式不会自动共享嵌入式 :class:`.Sequence`，而应指定 :paramref:`.Sequence.schema` ::

    Sequence("cart_id_seq", start=1, schema="some_schema")

  也可以通过在使用中表示所有 :class:`_schema.Table` / :class:`_schema.Column` 的 :class:`_schema.MetaData`，将 :class:`.Sequence` 明确地关联到 :class:`.Sequence`
 ：参见“ :ref:`sequence_metadata`”部分了解详情。

当 :class:`_schema.Column` 的 **Python端** 默认生成器与 :class:`.Sequence` 相关联时， :class:`.Sequence` 也将受到 “CREATE SEQUENCE” 和 “DROP SEQUENCE” DDL 的影响，就像与表一起使用时一样。只要发出 :meth:`.MetaData.create_all` 或 :meth:`.Table.create`，针对该列（无论是在此表或在其他表中）或任何使用该序列的其他项进行 CREATE/ALTER/DROP 操作，将始终为该序列执行此操作。

:class:`.Sequence` 也可以直接与:class:`.MetaData` 构造相关联。这使:class:`.Sequence` 在一次可以与多个:class:`.Table` 结合使用的时间，并且还允许从所使用的 :class:`.Sequence` 或 :class:`.Table` 继承 :paramref:`.MetaData.schema` 参数。有关详情，请参见 :ref:`sequence_metadata` 部分。

在一个SERIAL列上关联序列
---------------------------

PostgreSQL的SERIAL数据类型是一种自动递增类型，当发出CREATE TABLE时，它意味着隐式创建PostgreSQL序列。可以在 :class:`_schema.Column` 上将 :class:`.Sequence` 结合使用，以防止在CREATE TABLE过程中使用该特定列的PostgreSQL自动生成序列，并且可以通过为 :paramref:`.Sequence.optional` 参数指定值“True”，忽略该特定情况。 这使我们可以使用PATHNAME时使用相同的序列：

    table = Table(
        "cartitems",
        metadata_obj,
        Column(
            "cart_id",
            Integer,
            # use an explicit Sequence where available, but not on
            # PostgreSQL where SERIAL will be used
            Sequence("cart_id_seq", start=1, optional=True),
            primary_key=True,
        ),
        Column("description", String(40)),
        Column("createdate", DateTime()),
    )

在上面的示例中，对于PostgreSQL的“CREATE TABLE”将使用“SERIAL”数据类型为“cart_id”列，并且将忽略“cart_id_seq”序列。但是在Oracle上，“cart_id_seq”序列将显式创建。

.. tip::

    SERIAL 和 SEQUENCE 的这种交互比较古老，在其他情况下，最好使用 :class:`.Identity` 来代替 :class:`.Sequence` 来生成整数主键值。请参阅 :ref:`identity_ddl` 部分了解有关此构造的背景。

单独执行序列
-----------------

SEQUENCE 是 SQL 的第一类模式对象，可以独立于数据库中生成值。如果您有:class:`.Sequence` 对象，可以通过将其直接传递到SQL执行方法来通过其“next value”指令来调用它：

    with my_engine.connect() as conn:
        seq = Sequence("some_sequence", start=1)
        nextid = conn.execute(seq)

要将 :class:`.Sequence` 的“next value”函数嵌入到诸如SELECT或INSERT之类的SQL语句中，请使用 :meth:`.Sequence.next_value` 方法，在语句编译时进行渲染，生成适合目标后端的SQL函数 ：

.. sourcecode:: pycon+sql

    >>> my_seq = Sequence("some_sequence", start=1)
    >>> stmt = select(my_seq.next_value())
    >>> print(stmt.compile(dialect=postgresql.dialect()))
    {printsql}SELECT nextval('some_sequence') AS next_value_1

.. _sequence_metadata:

将序列与MetaData相关联
----------------------------

对于要与任意 :class:`.Table` 相关联的 :class:`.Sequence`，可以将 :class:`.Sequence` 与特定 :class:`_schema.MetaData` 关联，使用 :paramref:`.Sequence.metadata` 参数::

    seq = Sequence("my_general_seq", metadata=metadata_obj, start=1)

这样的序列可以像独立架构构造一样独立存在，也可以在表之间共享。

将 :class:`.Sequence` 显式关联到 :class:`_schema.MetaData` 可以实现以下行为：

* :class:`.Sequence` 会继承放在目标 :class:`_schema.MetaData` 上指定的 :paramref:`_schema.MetaData.schema` 参数，则影响生成 CREATE/DROP DDL，以及在SQL语句中呈现 :meth:`.Sequence.next_value` 函数的方式。

* :meth:`_schema.MetaData.create_all` 和:meth:`_schema.MetaData.drop_all` 方法将发出CREATE / DROP此 :class:`.Sequence` 的DDL，即使 :class:`.Sequence` 没有与任何 :class:`_schema.Table` / :class:`_schema.Column` 关联到此 :class:`_schema.MetaData` 中。

将序列与服务器端默认值相关联
----------------------------------------

.. note:: 下面的技术已知仅适用于PostgreSQL数据库。它不适用于Oracle。

在前面的部分中，说明了如何将 :class:`.Sequence` 作为Python端默认生成器与 :class:`_schema.Column` 关联：

    Column(
        "cart_id",
        Integer,
        Sequence("cart_id_seq", metadata=metadata_obj, start=1),
        primary_key=True,
    )

在上面的情况下，当相关联的 :class:`.Sequence` 也是 +server-side default generator+ 中Server一方时，该 :class:`.Sequence` 必须在 CREATE TABLE发出的CREATE TABLE状态下进行使用，当与 :paramref:`_schema.Column.server_default` 参数配合使用时可以自动成为服务器端默认生成器。从该序列函数生成的值可用从 :class:`.MetaData.create_all` 或 :class:`_schema.Table` 的DDL记述中使用但它不得作为该列的连接默认值嵌入到创建该表的SQL表示：

    cart_id_seq = Sequence("cart_id_seq", metadata=metadata_obj, start=1)
    table = Table(
        "cartitems",
        metadata_obj,
        Column(
            "cart_id",
            Integer,
            cart_id_seq,
            server_default=cart_id_seq.next_value(),
            primary_key=True,
        ),
        Column("description", String(40)),
        Column("createdate", DateTime()),
    )

或使用ORM：

    class CartItem(Base):
        __tablename__ = "cartitems"

        cart_id_seq = Sequence("cart_id_seq", metadata=Base.metadata, start=1)
        cart_id = Column(
            Integer, cart_id_seq, server_default=cart_id_seq.next_value(), primary_key=True
        )
        description = Column(String(40))
        createdate = Column(DateTime)

在上面的示例中，在PostgreSQL上发出“CREATE TABLE”语句时将使用“SERIAL”数据类型为cart_id列，并且将忽略cart_id_seq序列。在Oracle上，将显式创建cart_id_seq序列。

.. 小贴士::

    为 :class:`.Sequence` 显式关联 :class:`_schema.MetaData`，这会确保 :class:`.Sequence`
    可以独立创建（例如，可以通过自动化脚本完成），也可以与不同的
    :class:`_schema.Table` 结合使用，从而确保 :class:`.Sequence` 适应不同的表和模式名称约定::参见“ :ref:`sequence_metadata`”部分了解详情。


将:classmethod:`~.Sequence.next_value` 用作 :paramref:`_schema.Column.server_default` 产生的序列的值生成器函数，由于并不在INSERT语句本身中，因此确保了“主键提取”逻辑在所有情况下都有效。通常，支持序列的数据库也支持INSERT语句的RETURNING，这由SQLAlchemy在emit时自动使用。但是，如果针对特定的插入不使用RETURNING，则SQLAlchemy会首选在INSERT语句本身外部“预执行”序列，该预执行仅在将序列包含为Python-side默认值生成器函数的情况下有效。

示例还将 :class:`.Sequence` 直接关联到所包含的 :class:`_schema.MetaData` 中，这再次确保了:class:`.Sequence` 在 :class:`_schema.Table` 的 INSERT 期间“primary key fetch”逻辑的所有情况下都有效。

与:class:`_schema.MetaData`集合的参数完全关联，包括默认模式（如果有）。

.. seealso::

    :ref:`postgresql_sequences` - PostgreSQL方言文档中的说明

    :ref:`oracle_returning` - Oracle方言文档中的说明

.. _computed_ddl:

计算列（GENERATED ALWAYS AS）
--------------------------------------

.. versionadded:: 1.3.11

:class:`.Computed`结构允许将:class:`_schema.Column`声明为“GENERATED ALWAYS AS”列，
即由数据库服务器计算其值的列。结构接受SQL表达式，通常使用字符串或:func:`_expression.text`构造函数声明为文本，类似于 :class:`.CheckConstraint`。
然后，数据库服务器解释SQL表达式以确定在行内列的值。

例子::

    from sqlalchemy import Table, Column, MetaData, Integer, Computed

    metadata_obj = MetaData()

    square = Table(
        "square",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("side", Integer),
        Column("area", Integer, Computed("side * side")),
        Column("perimeter", Integer, Computed("4 * side")),
    )

在PostgreSQL 12后端上运行``square``表的DDL将如下所示：

.. sourcecode:: sql

    CREATE TABLE square (
        id SERIAL NOT NULL,
        side INTEGER,
        area INTEGER GENERATED ALWAYS AS (side * side) STORED,
        perimeter INTEGER GENERATED ALWAYS AS (4 * side) STORED,
        PRIMARY KEY (id)
    )

值是在插入和更新时持久化，还是在获取时计算，是数据库的实现细节；
前者称为“存储”，后者称为“虚拟”。一些数据库实现支持两者，但一些只支持其中一个。
可指定可选的:paramref:`.Computed.persisted`标志，其值为``True``或``False``, 以指示DDL应渲染“STORED”或“VIRTUAL”关键字，但是如果目标后端不支持关键字，则会引发错误；留空将为目标后端使用工作默认值。 

:class:`.Computed`结构是:class:`.FetchedValue`基本结构的子类，并将自己设置为目标
使用时，“服务器默认值”和“服务器更新”生成器:type:`_schema.Column`，这意味着在生成INSERT和UPDATE语句时，它将被视为默认生成列，并且在使用ORM时将被获取为生成列。
包括对于支持RETURNING且需要急切获取生成值的数据库，它将是数据库RETURNING子句的一部分。

.. note:: 使用:class:`.Computed`声明的 :class:`_schema.Column`可能不存储除服务器应用之外的任何值；目前，当传递值用于插入或更新写入此类列时，SQLAlchemy的行为是忽略这个值。

“GENERATED ALWAYS AS”当前可以支持：

* MySQL版本5.7及以上

* MariaDB 10.x系列及以上

* PostgreSQL从12版本开始

* Oracle - 附加条件是RETURNING与UPDATE不正常工作
  渲染包括计算列的UPDATE..RETURNING时将发出警告)

* Microsoft SQL Server

* SQLite从版本3.31开始

使用不支持的后端时，如果目标方言不支持，尝试呈现构造时，会引发:class:`.CompileError`。
否则，如果方言支持它，但所使用的特定数据库服务器版本不支持，则会引发:class:`.DBAPIError`的子类，通常为:class:`.OperationalError`，当将DDL发送到数据库时。

.. seealso::

    :class:`.Computed`

.. _identity_ddl:

身份列（GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY）
-----------------------------------------------------------------

.. versionadded:: 1.4

:class:`.Identity`结构允许将基于数据库中使用的自增或自减序列，
将:class:`_schema.Column`声明为身份列，并将其呈现为DDL中的“GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY”。此结构共享其大多数选项来控制数据库行为，以与 :class:`.Sequence` 相关联。

例子::

    from sqlalchemy import Table, Column, MetaData, Integer, Identity, String

    metadata_obj = MetaData()

    data = Table(
        "data",
        metadata_obj,
        Column("id", Integer, Identity(start=42, cycle=True), primary_key=True),
        Column("data", String),
    )

在PostgreSQL 12后端上运行``data``表的DDL将如下所示：

.. sourcecode:: sql

    CREATE TABLE data (
        id INTEGER GENERATED BY DEFAULT AS IDENTITY (START WITH 42 CYCLE) NOT NULL,
        data VARCHAR,
        PRIMARY KEY (id)
    )

数据库会在插入时生成``id``列的值，从``42``开始，如果语句中没有包含``id``列的值。
身份列也可以要求数据库生成该列的值，忽略语句传递的值或引发错误，具体取决于后端。
为了激活此模式，请在:class:`.Identity`结构中将参数:paramref:`_schema.Identity.always`设置为 ``True``。
将前一个示例更新以包括此参数将生成以下DDL：

.. sourcecode:: sql

    CREATE TABLE data (
        id INTEGER GENERATED ALWAYS AS IDENTITY (START WITH 42 CYCLE) NOT NULL,
        data VARCHAR,
        PRIMARY KEY (id)
    )

:class:`.Identity`构造函数是:class:`.FetchedValue`基本结构的子类, 并且将其本身设置为目标:type:`_schema.Column`的“服务器默认值”生成器，这意味着在生成INSERT语句时，它将被视为默认生成列，并且在使用ORM时将其作为生成列获取。
包括对于支持RETURNING且需要急切获取生成值的数据库，它将是数据库返回的子句的一部分。

目前已知:class:`.Identity`结构受以下支持：

* PostgreSQL从10版本开始。

* Oracle从12版本开始。它还支持将“always=None”传递给启用默认生成模式，
  参数“on_null=True”指定“BY DEFAULT”身份列中的“ON NULL”。

* Microsoft SQL Server。MSSQL使用自定义语法，只支持“start”和“increment”参数，并忽略所有其他语句。

使用不受支持的后端时，会忽略:class:`.Identity`，并使用自动增量列的默认SQLAlchemy逻辑。
当:class:`_schema.Column`同时指定:class:`.Identity`并设置:paramref:`_schema.Column.autoincrement`为 ``False``时，会引发错误。

.. seealso::

    :class:`.Identity`


默认对象API
-------------------

.. autoclass:: Computed
    :members:


.. autoclass:: ColumnDefault


.. autoclass:: DefaultClause


.. autoclass:: DefaultGenerator


.. autoclass:: FetchedValue


.. autoclass:: Sequence
    :members:


.. autoclass:: Identity
    :members: