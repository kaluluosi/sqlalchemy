.. currentmodule:: sqlalchemy.schema

.. _metadata_defaults_toplevel:

.. _metadata_defaults:

列INSERT/UPDATE默认值
=============================

列INSERT和UPDATE默认值是指在执行对行的INSERT或UPDATE语句时为某个列创建一个默认值
当**在INSERT或UPDATE语句中，没有为该列提供值的情况下**，用作插入到“时间戳”列的值创建新值，如当前时间。
如果语句*不*包含此列的值，则将不会发生默认值。

列默认值可以是与架构中的  :term:`DDL`  一起定义的服务器端函数或常量值，
也可以是作为SQL表达式直接渲染在SQLAlchemy发出的INSERT或UPDATE语句中的服务器端SQL表达式；
它们也可以是由SQLAlchemy在将数据传递​​到数据库之前调用的客户端Python函数或常量值。

.. note::

    不要将列默认处理程序与拦截并修改INSERT和UPDATE语句中提供的值的构造混淆
    应用程序应用程序会更改某个列的值。这是称为  :term:`数据装配`  ，
    它修改某个列的值，从而在发送到数据库之前通过应用程序以某种方式进行修改。
    SQLAlchemy提供了一些实现此目的的方法，包括使用:ref：自定义数据类型<types_typedecorator>，
     :ref:`SQL执行事件<core_sql_events>` 及在ORM中使用的:
    ref：定制验证器<simple_validators>以及 :ref:`orm_attribute_events <orm_attribute_events>` 中的属性事件。
    仅当在SQL :term: DML语句中不存在**值**时，列默认值才会被调用。

SQLAlchemy提供了一系列有关在INSERT和UPDATE语句中生成缺失值的默认生成功能。
选项包括：

* 插入和更新操作期间用作默认值的标量值
* 执行INSERT和UPDATE操作的Python函数
* 嵌入在INSERT语句中的SQL表达式（或在某些情况下先执行）
* 嵌入在UPDATE语句中的SQL表达式
* 用于INSERT的服务器端默认值
* 用于UPDATE时使用的服务器端触发器标记

对于所有插入/更新默认值的通用规则是，仅在没有传递特定列的值作为“execute()”参数时才会生效
否则使用给定的值。

标量默认值
---------------

最简单的默认值是用作列默认值的标量值::

    Table("mytable", metadata_obj, Column("somecolumn", Integer, default=12))

上面，如果没有提供其他值，"12"将在INSERT时绑定为列值。

标量值也可以与UPDATE语句关联，但这不太常见（因为UPDATE语句通常正在寻找动态默认值）。：

    Table("mytable", metadata_obj, Column("somecolumn", Integer, onupdate=25))

Python-Executed Functions
-------------------------

  :paramref:`_schema.Column.default`  和  :paramref:` _schema.Column.onupdate`  关键字参数也接受Python
函数。如果没有为该列提供其他值，则在插入或更新时调用这些函数，然后返回的值用于列的值。下面说明了简单的“序列”
将递增计数器分配给主键列的方法::

    # a function which counts upwards
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

应该注意，对于真正的“递增序列”行为，应正常使用数据库的内置功能
此包括序列对象或其他自动递增功能。对于主键列，SQLAlchemy在大多数情况下将自动使用这些功能
参见  :class:`~sqlalchemy.schema.Column`  标志，
以及本章后面有关 :class:`~sqlalchemy.schema.Sequence` 的部分，用于了解标准主键生成技术的背景信息。

为说明onupdate，我们将Python的datetime函数“now”分配给  :paramref:`_schema.Column.onupdate`  属性：：

    import datetime

    t = Table(
        "mytable",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        # define 'last_updated' to be populated with datetime.now()
        Column("last_updated", DateTime, onupdate=datetime.datetime.now),
    )

更新语句执行并且没有为“last_updated”传递值时，将执行“datetime.datetime.now（）”Python函数，并且其返回值用作“last_updated”的值。
请注意，我们将“now”作为函数本身提供，而无需调用它（即，没有括号以下）-SQLAlchemy将在语句执行时执行函数。

.. _context_default_functions:

上下文敏感的默认函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于某个语句的上下文会包含所有关于正在执行的语句的信息，因此  :paramref:`_schema.Column.default`  和
  :paramref:`_schema.Column.onupdate`  中使用的Python函数可以使用当前语句的上下文来确定值。典型的用例与
默认生成相关，可以使用上下文访问在行上插入或更新的其他值。要访问上下文，请提供一个接受一个“context”参数的函数::

    def mydefault(context):
        return context.get_current_parameters()["counter"] + 12


    t = Table(
        "mytable",
        metadata_obj,
        Column("counter", Integer),
        Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
    )

上面的默认生成功能将应用于所有INSERT和UPDATE语句，其中并没有提供“counter_plus_twelve”的值，
该值将是执行插入或更新时存在于“counter”列的任何值加上数字12的值。

对于使用“executemany”样式执行的单个语句（即使用传递给  :meth:`_engine.Connection.execute`  的多个参数集的语句），
会为每组参数调用用户定义的函数。对于多值 :class:`_expression.Insert` 构造（例如使用多个VALUES
由  :meth:`_expression.Insert.values`  方法设置的子句），也会为每个参数集调用用户定义的函数。

调用函数时，从上下文对象中可用  :meth:`.DefaultExecutionContext.get_current_parameters`  特殊方法
(一个 :class:`.DefaultExecutionContext` 子类)。
此方法返回一个字典，其中列键对应于值，表示INSERT或UPDATE语句的完整值集。针对
多值INSERT构造，将仅从完整参数字典中分离与单个VALUES子句对应的子集参数，并将其单独返回。

.. versionadded:: 1.2

    添加了  :meth:`.DefaultExecutionContext.get_current_parameters`  方法，
    它改进了仍然存在的  :attr:`.DefaultExecutionContext.current_parameters`  属性，
    通过提供将多个VALUES子句组织到单个参数字典中的服务来提供服务。

.. _defaults_client_invoked_sql:

由客户端调用的SQL表达式
------------------------------

  :paramref:`_schema.Column.default`  和  :paramref:` _schema.Column.onupdate`  关键字也可以传递SQL表达式，
在大多数情况下，这些表达式直接在
INSERT或UPDATE语句内联呈现：

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

上面，“create_date”列将使用“now（）”SQL函数（根据后端，大多数情况下编译为“NOW（）”
或“CURRENT_TIMESTAMP”）在INSERT语句中填充，而“key”列的结果是从其他表中查询的SELECT子查询的结果。
“last_modified”列将在发出此表的UPDATE语句时与SQL“UTC_TIMESTAMP（）” MySQL函数的值一起填充。

.. note::

    在使用  :attr:`.func`  构造使用SQL函数时，我们“调用”命名函数，
    例如使用括号，如``func.now（）``。这与指定Python可调用对象略有不同，例如
    ``datetime.datetime``，其中我们传递函数本身，但是我们不
    手动调用。在使用SQL函数时，调用``func.now（）``会返回这种
    执行“NOW”函数的SQL表达式对象正在发出的SQL。

由  :paramref:`_schema.Column.default`  引用的默认值和  :paramref:` _schema.Column.onupdate`  是在INSERT或UPDATE语句发生时显式调用的
大多数情况下内联呈现在DML语句中，除非在下面列出的某些情况下。这与无法更改的“服务器端”默认值不同，
该值作为表的DDL定义的一部分存在，例如作为“CREATE TABLE”语句的一部分，这些DDL定义可能更常见。要
有关服务器端默认值，请参见下一节：ref：`server_defaults`。

使用PRIMARY KEY列的SQL表达式指示由于某些其他服务器端数据库机制（例如某些平台上的TIMESTAMP列）或自定义触发器而在INSERT或UPDATE
上生成了新值的列，可以使用 :class：` .FetchedValue`作为标记：：

    from sqlalchemy.schema import FetchedValue

    t = Table(
        "test",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("abc", TIMESTAMP, server_default=FetchedValue()),
        Column("def", String(20), server_onupdate=FetchedValue()),
    )

 :class:`.FetchedValue` 指示符不会影响CREATE TABLE的渲染DDL。它标记列，用作将在执行INSERT或UPDATE语句期间为该列填充一个新值的列。
该值用于支持数据库可能用于此列的RETURNING或输出子句。工具，例如
ORM然后将使用此标记，以便知道如何访问操作后的列的值。特别是，的
  :meth:`.ValuesBase.return_defaults`  方法可用于  :class:` _expression.Insert` 
或 :class:`_expression.Update` 构造指定要返回这些值。

有关在ORM中使用 :class:`. FetchedValue` 的详细信息，请参见
：ref：`orm_server_defaults`。

.. warning:: :paramref:`_schema.Column.server_onupdate` 指令
   **并不**目前会生成MySQL的
   “ON UPDATE CURRENT_TIMESTAMP（）”子句。看
   :ref：`mysql_timestamp_onupdate`了解如何生成
   此子句的背景。

.. seealso::

      :ref:`orm_server_defaults` 

.. _defaults_sequences:

定义序列
-------------------

SQLAlchemy使用 :class:`~sqlalchemy.schema.Sequence` 对象表示数据库序列，它被认为是一种
特殊情况的“列默认值”。它仅对具有明确支持序列的数据库有效，
在SQLAlchemy包括的方言之间，包括PostgreSQL、Oracle、MS SQL Server和MariaDB。  :class:`~sqlalchemy.schema.Sequence` 
对象将被忽略。

.. tip::

    在较新的数据库引擎中，  :class:`.Identity` .Sequence` 
    用于生成整数主键值。请参见该部分：ref:`identity_ddl`以获取有关此构造的背景信息。

 :class:`~sqlalchemy.schema.Sequence` 可以作为“默认”生成器放置在任何列上，
在INSERT操作期间使用，也可以在需要时配置在UPDATE操作期间触发的操作。
它通常与单个整数主键列结合使用：

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

在上面的示例中，表``cartitems``与名为``cart_id_seq``的序列相关联。
执行  :meth:`.MetaData.create_all`  以用于上述表将包括：

.. sourcecode:: sql

    CREATE SEQUENCE cart_id_seq START WITH 1

    CREATE TABLE cartitems (
        cart_id INTEGER NOT NULL,
        description VARCHAR(40),
        createdate TIMESTAMP WITHOUT TIME ZONE,
        PRIMARY KEY (cart_id)
    )

.. tip::

  在使用具有显式模式名称的表时（在详细了解中）中，  :class:`.Table` .Sequence` 分享
  相反，指定  :paramref:`.Sequence.schema`  ：

    Sequence("cart_id_seq", start=1, schema="some_schema")

    :class:`.Sequence` ` `.MetaData.schema``设置在使用中的   :class:`.MetaData` ;
  有关背景信息，请参见：ref：`sequence_metadata`。

当使用  :meth:`_dml.Insert`   DML构造执行针对` `cartitems``表时，如果没有为``cart_id``列传递显式值，则将使用``cart_id_seq``序列生成值。
通常，序列函数将嵌入INSERT语句中，该语句与返回新生成的值的PYTHON过程或函数组合：

.. sourcecode:: sql

    INSERT INTO cartitems (cart_id, description, createdate)
    VALUES (next_val(cart_id_seq), 'some description', '2015-10-15 12:00:15')
    RETURNING cart_id

通过使用  :meth:`.Connection.execute`  将 :class:` _dml.Insert`构造传递给执行来调用：
在此类操作中生成的新的主键标识符，包括但不限于
使用  :class:`.Sequence` .CursorResult` 构造使用
使用  :attr:`.CursorResult.inserted_primary_key`  属性。

当为  :meth:`_dml.Insert`  构造调用RETURNING时，这些新生成的主键标识符通常也可用于  :class:` .CursorResult` 
使用  :meth:`.CursorResult.last_inserted_params`  或  :meth:` .CursorResult.last_updated_params`  中的集合。
  :attr:`_engine.CursorResult.inserted_primary_key`  集合包含插入的行的主键
（使用列表，以便单列和复合列主键以相同的格式表示）。

.. _server_defaults:

基于服务器调用DDL显式默认表达式
-----------------------------------------------

SQL表达式默认值的变体是  :paramref:`_schema.Column.server_default`  ，在CREATE TABLE操作期间将其放置在
使用 : meth：`_schema.Table.create`操作：

.. sourcecode:: python+sql

    t = Table(
        "test",
        metadata_obj,
        Column("abc", String(20), server_default="abc"),
        Column("created_at", DateTime, server_default=func.sysdate()),
        Column("index_value", Integer, server_default=text("0")),
    )

上述示例说明了用于SQL函数（在上述示例中为SYSDATE）以及服务器端常量值（上述示例中的整数“0”）的两种典型用例
。建议对任何文本SQL值使用:func：`_expression.text`构造而不是传递原始值，
因为SQLAlchemy通常不对这些值执行任何引用或转义。

与使用  :paramref:`_schema.Column.default`  或  :paramref:` _schema.Column.onupdate`  的用户定义Python函数一样，
  :paramref:`_schema.Column.server_default`  可以容纳SQL表达式，但通常情况下这些将是简单的表达式
函数和表达式，而不是更复杂的情况，如嵌入SELECT。

.. _triggered_columns:

标记隐式生成的值、时间戳和触发列
----------------------------------------------------------------------

对于在INSERT或UPDATE上生成新值的列，这些值基于其他服务器端数据库机制，
例如某些平台上的特定于数据库的自动生成行为，例如在某些平台上的TIMESTAMP列，以及
在INSERT或UPDATE触发器上调用自定义触发器，可以使用:class：`.FetchedValue`作为标记：：

    from sqlalchemy.schema import FetchedValue

    t = Table(
        "test",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("abc", TIMESTAMP, server_default=FetchedValue()),
        Column("def", String(20), server_onupdate=FetchedValue()),
    )

 :class:`.FetchedValue` 指示符不会影响 CREATE TABLE 的呈现 DDL。它标记
该列将在进行INSERT或UPDATE语句处理期间填充一个新值，并且对于支持的数据库可能用于返回子句的该列应该包括在其中。
ORM之类的工具然后使用此标记，以便知道如何访问操作后的列的值。特别是
  :meth:`~.ValuesBase.return_defaults`  方法可用于  :class:` _expression.Insert` 
或 :class:`_expression.Update` 构造指定要返回这些值。

有关使用ORM中的  :class:`.FetchedValue` 。

.. warning:: :paramref:`_schema.Column.server_onupdate` 指令
    **不会**目前生成MySQL的
    “ON UPDATE CURRENT_TIMESTAMP（）”子句。看
    :ref：`mysql_timestamp_onupdate`了解如何生成
    此子句的背景。

.. seealso::

      :ref:`orm_server_defaults` 


.. _defaults_sequences:

定义序列
-------------------

SQLAlchemy使用 :class:`~sqlalchemy.schema.Sequence` 对象表示数据库序列，它被认为是一种
特殊情况的“列默认值”。它仅对具有明确支持序列的数据库有效，
在SQLAlchemy包括的方言之间，包括PostgreSQL、Oracle、MS SQL Server和MariaDB。  :class:`~sqlalchemy.schema.Sequence` 
对象将被忽略。

.. tip::

    在较新的数据库引擎中，  :class:`.Identity` .Sequence` 
    用于生成整数主键值。请参见该部分：ref:`identity_ddl`以获取有关此构造的背景信息。

 :class:`~sqlalchemy.schema.Sequence` 可以作为“默认”生成器放置在任何列上，
在INSERT操作期间使用，也可以在需要时配置在UPDATE操作期间触发的操作。
它通常与单个整数主键列结合使用：

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

在上面的示例中，表``cartitems``与名为``cart_id_seq``的序列相关联。
执行  :meth:`.MetaData.create_all`  以用于上述表将包括：

.. sourcecode:: sql

    CREATE SEQUENCE cart_id_seq START WITH 1

    CREATE TABLE cartitems (
        cart_id INTEGER NOT NULL,
        description VARCHAR(40),
        createdate TIMESTAMP WITHOUT TIME ZONE,
        PRIMARY KEY (cart_id)
    )

.. tip::

  在使用具有显式模式名称的表时（在详细了解中）中，  :class:`.Table` .Sequence` 分享
  相反，指定  :paramref:`.Sequence.schema`  ：

    Sequence("cart_id_seq", start=1, schema="some_schema")

    :class:`.Sequence` ` `.MetaData.schema``设置在使用中的   :class:`.MetaData` ;
  有关背景信息，请参见：ref：`sequence_metadata`。

当使用  :meth:`_dml.Insert`   DML构造执行针对` `cartitems``表时，如果没有为``cart_id``列传递显式值，则将使用``cart_id_seq``序列生成值。
通常，序列函数将嵌入INSERT语句中，该语句与返回新生成的值的PYTHON过程或函数组合：

.. sourcecode:: sql

    INSERT INTO cartitems (cart_id, description, createdate)
    VALUES (next_val(cart_id_seq), 'some description', '2015-10-15 12:00:15')
    RETURNING cart_id

通过使用  :meth:`.Connection.execute`  将 :class:` _dml.Insert`构造传递给执行来调用：
在此类操作中生成的新的主键标识符，包括但不限于
使用  :class:`.Sequence` .CursorResult` 构造使用
使用  :attr:`.CursorResult.inserted_primary_key`  属性。

当为  :meth:`_dml.Insert`  构造调用RETURNING时，这些新生成的主键标识符通常也可用于  :class:` .CursorResult` 
使用  :meth:`.CursorResult.last_inserted_params`  或  :meth:` .CursorResult.last_updated_params`  中的集合。
  :attr:`_engine.CursorResult.inserted_primary_key`  集合包含插入的行的主键
（使用列表，以便单列和复合列主键以相同的格式表示）。

.. _server_defaults:

基于服务器调用DDL显式默认表达式
-----------------------------------------------

SQL表达式默认值的变体是  :paramref:`_schema.Column.server_default`  ，在CREATE TABLE操作期间将其放置在
使用: meth:`_schema.Table.create`操作：

.. sourcecode:: python+sql

    t = Table(
        "test",
        metadata_obj,
        Column("abc", String(20), server_default="abc"),
        Column("created_at", DateTime, server_default=func.sysdate()),
        Column("index_value", Integer, server_default=text("0")),
    )

上述示例说明了用于SQL函数（在上述示例中为SYSDATE）以及服务器端常量值（上述示例中的整数“0”）的两种典型用例
。建议对任何文本SQL值使用:func：`_expression.text`构造而不是传递原始值，
因为SQLAlchemy通常不对这些值执行任何引用或转义。

与使用  :paramref:`_schema.Column.default`  或  :paramref:` _schema.Column.onupdate`  的用户定义Python函数一样，
  :paramref:`_schema.Column.server_default`  可以容纳SQL表达式，但通常情况下这些将是简单的表达式
函数和表达式，而不是更复杂的情况，如嵌入SELECT。

.. _triggered_columns:

标记隐式生成的值、时间戳和触发列
----------------------------------------------------------------------

对于在INSERT或UPDATE上生成新值的列，这些值基于其他服务器端数据库机制，
例如某些平台上的特定于数据库的自动生成行为，例如在某些平台上的TIMESTAMP列，以及
在INSERT或UPDATE触发器上调用自定义触发器，可以使用:class：`.FetchedValue`作为标记：：

    from sqlalchemy.schema import FetchedValue

    t = Table(
        "test",
        metadata_obj,
        Column("id", Integer, primary_key=True),
        Column("abc", TIMESTAMP, server_default=FetchedValue()),
        Column("def", String(20), server_onupdate=FetchedValue()),
    )

 :class:`.FetchedValue` 指示符不会影响 CREATE TABLE 的呈现 DDL。它标记
该列将在进行INSERT或UPDATE语句处理期间填充一个新值，并且对于支持的数据库可能用于返回子句的该列应该包括在其中。
ORM之类的工具然后使用此标记，以便知道如何访问操作后的列的值。特别是
  :meth:`~.ValuesBase.return_defaults`  方法可用于  :class:` _expression.Insert` 
或 :class:`_expression.Update` 构造指定要返回这些值。

有关使用ORM中的  :class:`.FetchedValue` 。

.. warning:: :paramref:`_schema.Column.server_onupdate` 指令
    **不会**目前生成MySQL的
    “ON UPDATE CURRENT_TIMESTAMP（）”子句。看
    :ref：`mysql_timestamp_onupdate`了解如何生成
    此子句的背景。

.. seealso::

     :ref:`orm_server_defaults` 完全关联于 :class:`_schema.MetaData` 集合的参数，包括默认架构（如果有）。

.. seealso::

      :ref:`postgresql_sequences`  - 在PostgreSQL方言文档中

      :ref:`oracle_returning`  - 在Oracle方言文档中

.. _computed_ddl:

计算列 (GENERATED ALWAYS AS)
--------------------------------------

.. versionadded:: 1.3.11

  :class:`.Computed`  构造函数以文本形式声明，与   :class:` .CheckConstraint`  类似。然后，数据库服务器将解释SQL表达式，以确定行中列的值。

例如::

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

当在PostgreSQL 12后端上运行时，``square``表的DDL将如下所示::

    CREATE TABLE square (
        id SERIAL NOT NULL,
        side INTEGER,
        area INTEGER GENERATED ALWAYS AS (side * side) STORED,
        perimeter INTEGER GENERATED ALWAYS AS (4 * side) STORED,
        PRIMARY KEY (id)
    )

值是在插入和更新时是否持久化，还是在获取时计算，这是数据库的一个实现细节；前者称为 "stored"，后者称为 "virtual"。某些数据库实现支持两者，但有些只支持其中一个。可以指定 **computed.persisted** 标志，指定 "STORED" 或 "VIRTUAL" 关键字是否应呈现在DDL中，如果目标后端不支持关键字，则会引发错误；如果不设置，则使用目标后端的默认值。

  :class:`.Computed` .FetchedValue` 对象的子类，将自己设置为目标   :class:`_schema.Column`  的 "server default" 和 "server onupdate" 生成器，这意味着在生成INSERT和UPDATE语句时，它将被视为一个默认生成列，当使用ORM时也将被获取为一个生成列。这包括对于支持RETURNING的数据库：它将成为数据库的RETURNING子句的一部分，并且生成的值将被急切地获取。

.. note:: 用   :class:`.Computed`  定义的   :class:` _schema.Column`  可以在服务器应用的值之外不存储任何值；当向这样的列中传递值以应写入INSERT或UPDATE时，SQLAlchemy在当前实现中的行为是将忽略该值。

现在“GENERATED ALWAYS AS”已知支持以下数据库：

* MySQL版本5.7及更高版本

* MariaDB10.x系列及更高版本

* PostgreSQL自版本12起

* Oracle - 带有警告，即UPDATE.. RETURNING中包括计算列时，RETURNING无法正确工作，会发出一个警告。

* Microsoft SQL Server

* SQLite自版本3.31起

当不支持   :class:`.Computed`  的后端使用时，如果目标方言不支持它，则会在尝试渲染构造时引发   :class:` .CompileError` 。否则，如果方言支持它，但所使用的特定数据库服务器版本不支持它，则在将DDL发布到数据库时引发   :class:`.DBAPIError`  子类，通常是   :class:` .OperationalError` 。

.. seealso::

      :class:`.Computed` 

.. _identity_ddl:

标识列 (GENERATED {ALWAYS | BY DEFAULT} AS IDENTITY)
-----------------------------------------------------

.. versionadded:: 1.4

  :class:`.Identity` .Sequence` 控件控制数据库行为的选项的共享。

例如::

    from sqlalchemy import Table, Column, MetaData, Integer, Identity, String

    metadata_obj = MetaData()

    data = Table(
        "data",
        metadata_obj,
        Column("id", Integer, Identity(start=42, cycle=True), primary_key=True),
        Column("data", String),
    )

当在PostgreSQL 12后端上运行时，``data``表的DDL将如下所示::

    CREATE TABLE data (
        id INTEGER GENERATED BY DEFAULT AS IDENTITY (START WITH 42 CYCLE) NOT NULL,
        data VARCHAR,
        PRIMARY KEY (id)
    )

在插入时，数据库将为"``id``"列生成一个值，从"``42``"开始，如果语句中不存在"``id``"列的值。标识列还可以要求数据库生成该列的值，无论传递给语句的值如何，或者引发错误，具体取决于后端。要激活此模式，在  :class:`.Identity`  设置为 ` `True`` 。将上一个示例更新以包括此参数将生成以下DDL：

.. sourcecode:: sql

    CREATE TABLE data (
        id INTEGER GENERATED ALWAYS AS IDENTITY (START WITH 42 CYCLE) NOT NULL,
        data VARCHAR,
        PRIMARY KEY (id)
    )

  :class:`.Identity`  构造是   :class:` .FetchedValue`  对象的子类，并将自身设置为目标   :class:`_schema.Column`  的 "server default" 生成器，这意味着在生成INSERT语句时，它将被视为一个默认生成列，当使用ORM时也将被获取为一个生成列。这包括对于支持RETURNING的数据库：它将成为数据库的RETURNING子句的一部分，并且生成的值将被急切地获取。

目前已知支持   :class:`.Identity`  构造的有：

* PostgreSQL自版本10起。

* Oracle自版本12起。它还支持将always=None传递以启用默认生成模式，并支持将参数on_null=True与 "BY DEFAULT" 标识列结合使用来指定 "ON NULL"。

* Microsoft SQL Server。MSSQL使用自定义语法，仅支持 "start" 和 "increment" 参数，并忽略所有其他参数。

当在  :class:`_schema.Column` .Identity` 且同时设置  :paramref:`_schema.Column.autoincrement`  为 ` `False`` 时，将引发错误。

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