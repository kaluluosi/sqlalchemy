从数据库反射对象
===========================

一个   :class:`~sqlalchemy.schema.Table`  对象可以被传输到数据库中，从而加载关于自身的信息。这个过程被称为“自反”（reflection）。在最简单的情况下，您只需要指定表名、一个   :class:` ~sqlalchemy.schema.MetaData`  对象和“autoload_with”参数。

    >>> messages = Table("messages", metadata_obj, autoload_with=engine)
    >>> [c.name for c in messages.columns]
    ['message_id', 'message_name', 'date']

上述操作将使用给定的引擎查询数据库对“messages”表的信息，然后根据此信息生成与手动构造的 Python 相同的   :class:`~sqlalchemy.schema.Column`  and   :class:` ~sqlalchemy.schema.ForeignKey`  等对象。

当反映表时，如果给定表通过外键引用另一个表，则将创建一个第二个   :class:`~sqlalchemy.schema.Table`  对象，表示连接   :class:` ~sqlalchemy.schema.MetaData`  对象。例如，假设表“shopping_cart_items”引用名为“shopping_carts”的表。反射“shopping_cart_items”表会导致也加载“shopping_carts”表：

    >>> shopping_cart_items = Table("shopping_cart_items", metadata_obj, autoload_with=engine)
    >>> "shopping_carts" in metadata_obj.tables
    True

如果您只命名表，则可以访问已生成的“shopping_carts”表:

    shopping_carts = Table("shopping_carts", metadata_obj)

当然，最好使用 “autoload_with=engine” 参数。这是因为如果尚未加载表的属性，则只加载该表。如果表没有被加载，那么自反操作会自动加载。加载后，对同名的   :class:`~sqlalchemy.schema.Table`  的新调用不会发出任何反射查询。

重写反映列
----------------------------

可以在反映表时使用显式值覆盖单个列。这对于指定自定义数据类型、数据库中可能未配置的主键等约束非常有用。

    >>> mytable = Table(
    ...     "mytable",
    ...     metadata_obj,
    ...     Column(
    ...         "id", Integer, primary_key=True
    ...     ),  # override reflected 'id' to have primary key
    ...     Column("mydata", Unicode(50)),  # override reflected 'mydata' to be Unicode
    ...     # additional Column objects which require no change are reflected normally
    ...     autoload_with=some_engine,
    )

反映视图
----------------

反射系统也可以反映视图。基本用法与表相同:

    my_view = Table("some_view", metadata, autoload_with=engine)

上面，“my_view”是一个   :class:`~sqlalchemy.schema.Table`  对象，其中包含   :class:` ~sqlalchemy.schema.Column`  对象，表示视图“some_view”中每个列的名称和类型。

通常，在反射视图时，当未在数据库中配置主键时，至少需要一个主键约束，如果有外键，那么也应该配置外键。

使用“override”功能对此进行说明，显式指定作为主键或具有外键约束的那些列。

    my_view = Table(
        "some_view",
        metadata,
        Column("view_id", Integer, primary_key=True),
        Column("related_thing", Integer, ForeignKey("othertable.thing_id")),
        autoload_with=engine,
    )

一次反映所有表格
-----------------------------

  :class:`~sqlalchemy.schema.MetaData`  对象也可以获取表的列表并反射整个表集。这是通过使用   :func:` ~sqlalchemy.schema.MetaData.reflect`  方法来实现的。调用后，所有定位的表都存在于   :class:`~sqlalchemy.schema.MetaData`  对象的表字典中：

    metadata_obj = MetaData()
    metadata_obj.reflect(bind=someengine)
    users_table = metadata_obj.tables["users"]
    addresses_table = metadata_obj.tables["addresses"]

“metadata.reflect()”还提供了一种方便的方法来清除或删除数据库中的所有行：

    metadata_obj = MetaData()
    metadata_obj.reflect(bind=someengine)
    for table in reversed(metadata_obj.sorted_tables):
        someengine.execute(table.delete())

从其他模式反映表
------------------------------------

部分   :ref:`schema_table_schema_name`  介绍了表模式的概念，这是数据库中包含表和其他对象的命名空间，可以通过显式指定来指定。当反应   :class:` _schema.Table`  对象以及其他对象时，如视图、索引和序列，可以使用  :paramref:`_schema.Table.schema`  参数设置此模式。还可以使用  :paramref:` _schema.MetaData.schema`  参数将此模式设置为   :class:`_schema.MetaData`  对象的缺省模式。

此模式参数的使用直接影响反射功能在其被请求反射对象时将要寻找的位置。例如，通过使用  :paramref:`_schema.MetaData.schema`  参数配置的默认模式名称“project”：

    >>> metadata_obj = MetaData(schema="project")

然后，  :meth:`.MetaData.reflect`  将使用配置的“project”类作为所反映对象的位置：

    >>> # uses `schema` configured in metadata_obj
    >>> metadata_obj.reflect(someengine)

结果就是   :class:`_schema.Table`  对象从 “project” 模式反射而来，并以该名称的模式限定形式填充。

同样，在包括  :paramref:`_schema.Table.schema`  参数的个别   :class:` _schema.Table`  对象中，将从该数据库模式反射该模式中的项目。如果以“schema-qualified” 的方式进行反射，则会覆盖正在拥有项目的   :class:`_schema.MetaData`  集合上配置的缺省模式：

    >>> messages = Table("messages", metadata_obj, schema="project", autoload_with=someengine)
    >>> messages
    Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')

最后，  :meth:`_schema.MetaData.reflect`  方法本身也允许传递  :paramref:` _schema.MetaData.reflect.schema`  参数，因此我们可以为一个默认配置的   :class:`_schema.MetaData`  对象加载来自“project”模式的表：

    >>> metadata_obj = MetaData()
    >>> metadata_obj.reflect(someengine, schema="project")

我们可以调用  :meth:`_schema.MetaData.reflect`  任何次数，以传递不同的  :paramref:` _schema.MetaData.schema`  参数（或根本不传递任何参数）来继续将对象填充到   :class:`_schema.MetaData`  对象中：

    >>> # add tables from the "customer" schema
    >>> metadata_obj.reflect(someengine, schema="customer")
    >>> # add tables from the default schema
    >>> metadata_obj.reflect(someengine)

反射的限制
-------------------------

反射过程是使用仅在关系数据库中表示的信息重新创建   :class:`_schema.Table`  元数据。这个过程本质上无法恢复未实际存储在数据库中的架构方面：客户端默认值，无论是使用 Python 函数还是使用   :class:` _schema.Column`  的“默认”关键字定义的 SQL 表达式（请注意，这与 “server_default” 特别是反射的“default”关键字不同），列信息，例如放置在  :attr:`_schema.Column.info`  字典中的数据等等。

关系数据库还在许多情况下以不同于 SQLAlchemy 中指定的格式报告表元数据。从反射返回的   :class:`_schema.Table`  对象不能始终可靠地用于生成与原始 Python 定义的   :class:` _schema.Table`  对象相同的 DDL。这种情况发生在包括服务器默认值、列关联序列和有关约束和数据类型的各种怪癖等方面。服务器端默认值可能会用“::<type>”转换指令（通常 PostgreSQL 将包含一个“::<type>”转换）或不同的引用方案返回。

另一类限制包括部分或尚未完全定义反射的架构结构。最近提高了反映处理事项方面的功能，例如可以反映视图、索引和外键选项等。截止到此写作，像 CHECK 约束、table comments 和触发器等结构不能反映。

从数据库无关类型反映
---------------------------------------

反映列时，无论是  :paramref:`_schema.Table.autoload_with`  的  :meth:` _reflection.Inspector.get_columns`  方法，数据类型都会尽可能地特定于目标数据库。这意味着如果从 MySQL 数据库反射 “integer” 数据类型，则类型将由   :class:`sqlalchemy.dialects.mysql.INTEGER`  类表示，其中包括 MySQL 特定的属性，例如“display_width”。或在 PostgreSQL 上，可能返回 PostgreSQL 特定数据类型，如   :class:` sqlalchemy.dialects.postgresql.INTERVAL`  或   :class:`sqlalchemy.dialects.postgresql.ENUM` 。

反射中存在这样的用例，即给定一个   :class:`_schema.Table`  要传输到不同的数据库供应商。为了适应这种用例，可以使用  :meth:` _events.DDLEvents.column_reflect`  事件拦截列反射。 使用  :meth:`_types.TypeEngine.as_generic`  方法，可以将这些特定于供应商的数据类型实时转换为 SQLAlchemy 后端无关数据类型，用于上面的例子。例如，类型如   :class:` _types.Integer` 、  :class:`_types.Interval` 、  :class:` _types.Enum`  可以替换特殊数据类型   :class:`sqlalchemy.dialects.mysql.MEDIUMINT`  和   :class:` sqlalchemy.dialects.mysql.TINYINT` ，我们可以选择使用  :meth:`_events.DDLEvents.column_reflect`  事件来处理这些问题。自定义的处理程序将使用  :meth:` _types.TypeEngine.as_generic`  方法将上述 MySQL 特定类型对象转换为通用类型，方法是在传递给事件处理程序的列字典条目中将“type”条目替换为列字典。这个字典格式至  :meth:`_reflection.Inspector.get_columns`  中已经被描述：

    >>> from sqlalchemy import event
    >>> metadata_obj = MetaData()

    >>> @event.listens_for(metadata_obj, "column_reflect")
    ... def genericize_datatypes(inspector, tablename, column_dict):
    ...     column_dict["type"] = column_dict["type"].as_generic()

    >>> my_generic_table = Table("my_table", metadata_obj, autoload_with=mysql_engine)

现在我们获得了一个新的   :class:`_schema.Table`  对象，它是通用的并使用   :class:` _types.Integer`  代替这些数据类型。我们现在可以在 PostgreSQL 数据库（例如）上发出“CREATE TABLE”语句：

    >>> pg_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
    >>> my_generic_table.create(pg_engine)
    {execsql}CREATE TABLE my_table (
        id SERIAL NOT NULL,
        data1 VARCHAR(50),
        data2 INTEGER,
        data3 INTEGER,
        PRIMARY KEY (id)
    )

注意，SQLAlchemy 通常会为其他行为提供良好的猜测，例如 MySQL 的“AUTO_INCREMENT”指令通常使用“SERIAL”自动增加的数据类型最接近地在 PostgreSQL 中表示。

.. versionadded:: 1.4 添加了  :meth:`_types.TypeEngine.as_generic`  方法， 并且还通过改进  :meth:` _events.DDLEvents.column_reflect`  事件的使用，以便可以方便地应用于   :class:`_schema.MetaData`  对象。

反射的限制
-------------------------

反射过程是使用仅在关系数据库中表示的信息重新创建   :class:`_schema.Table`  元数据。这个过程本质上无法恢复未实际存储在数据库中的架构方面：客户端默认值、列信息、例如放置在  :attr:` _schema.Column.info`  字典中的数据等等。反映从反射返回的对象不能始终可靠地用于生成与原始 Python 定义的   :class:`_schema.Table`  对象相同的 DDL。这种情况发生在包括服务器默认值、列关联序列和有关约束和数据类型的各种怪癖等方面。服务器端默认值可能会用“::<type>”转换指令（通常 PostgreSQL 将包含一个“::<type>”转换）或不同的引用方案返回。

另一类限制包括部分或尚未完全定义反射的架构结构。最近提高了反映处理事项方面的功能，例如可以反映视图、索引和外键选项等。截止到此写作，像 CHECK 约束、table comments、触发器等结构还没有反映。