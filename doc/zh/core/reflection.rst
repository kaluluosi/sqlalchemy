.. currentmodule:: sqlalchemy.schema

.. _metadata_reflection_toplevel:
.. _metadata_reflection:


反射数据库对象
==============

:class:`~sqlalchemy.schema.Table`对象可以指示从相应的数据库模式对象中加载关于自身的信息。这个过程叫做*反射*。在最简单的情况下，你只需要指定表名、一个:class:`~sqlalchemy.schema.MetaData`对象，和``autoload_with``参数：

    >>> messages = Table(
    ...     "messages",
    ...     metadata_obj,
    ...     autoload_with=engine
    ... )
    >>> [c.name for c in messages.columns]
    ['message_id', 'message_name', 'date']


上面的操作将使用给定的engine查询关于``messages``表的信息，然后生成对应于这些信息的:class:`~sqlalchemy.schema.Column`，:class:`~sqlalchemy.schema.ForeignKey`和其他对象，就像:class:`~sqlalchemy.schema.Table`对象在Python中手动构建一样。

当表反射时，如果一个给定的表通过外部密钥引用了另一个表，那么会在代表连接的:class:`~sqlalchemy.schema.MetaData`对象中创建第二个:class:`~sqlalchemy.schema.Table`对象。下面，假设``shopping_cart_items``表引用了一个名为``shopping_carts``的表。反射``shopping_cart_items``表会导致``shopping_carts``表也被加载：

    >>> shopping_cart_items = Table("shopping_cart_items", metadata_obj, autoload_with=engine)
    >>> "shopping_carts" in metadata_obj.tables
    True

:class:`~sqlalchemy.schema.MetaData`具有类似"单例"的有趣行为，以便如果您单独请求两个表，:class:`~sqlalchemy.schema.MetaData`将确保为每个不同的表名创建一个:class:`~sqlalchemy.schema.Table`对象。如果已经存在具有给定名称的:class:`~sqlalchemy.schema.Table`对象，则:class:`~sqlalchemy.schema.Table`构造函数将返回已经存在的:class:`~sqlalchemy.schema.Table`对象。如下，我们可以通过名称访问已经生成的``shopping_carts``表：

    shopping_carts = Table("shopping_carts", metadata_obj)

当然，最好使用``autoload_with=engine``来处理上述表。这样，如果尚未加载表的属性，则会加载该表的属性。该自动加载操作仅在表还未加载时发生；一旦加载，对于具有相同名称的新调用:class:`~sqlalchemy.schema.Table`，不会重新发出任何反射查询。

.. _reflection_overriding_columns:

覆盖反映的列
------------------

可以在反射表时使用显式值覆盖单个列；这对于指定自定义数据类型、可能未在数据库中配置主键的约束等非常有用。例子如下所示：

    >>> mytable = Table(
    ...     "mytable",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),  #覆盖反射的'id'列作为主键
    ...     Column("mydata", Unicode(50)),  #覆盖反射的'mydata'列作为Unicode
    ...     autoload_with=some_engine,
    ... )

.. seealso::

    :ref:`custom_and_decorated_types_reflection` -说明了上述列覆盖技术如何适用于使用自定义数据类型进行表反射。

反射视图
------------

反射系统也可以反射视图。基本用法与表的用法相同：

    my_view = Table("some_view", metadata_obj, autoload_with=engine)

上面，“my_view”是一个:class:`~sqlalchemy.schema.Table`对象，包含每个列的名字和类型，它们是视图“some_view”中的每个列。

通常，在反映视图时希望至少有一个主键约束，如果没有外键约束的话。视图反射不会展开这些约束。

使用"override"技术进行优化，为具有主键或外键约束的列指定显式属性：

    my_view = Table(
        "some_view",
        metadata_obj,
        Column("view_id", Integer, primary_key=True),
        Column("related_thing", Integer, ForeignKey("othertable.thing_id")),
        autoload_with=engine,
    )

一次性反射所有表
-----------------------

:class:`~sqlalchemy.schema.MetaData`对象还可以获得表列表并反射整个列表。这是通过使用:func:`~sqlalchemy.schema.MetaData.reflect`方法实现的，在调用后，所有定位的表都在:class:`~sqlalchemy.schema.MetaData`对象的表字典中：

    metadata_obj = MetaData()
    metadata_obj.reflect(bind=someengine)
    users_table = metadata_obj.tables["users"]
    addresses_table = metadata_obj.tables["addresses"]

``metadata.reflect()``还为清除或删除数据库中所有行提供了一种便利的方法：

    metadata_obj = MetaData()
    metadata_obj.reflect(bind=someengine)
    for table in reversed(metadata_obj.sorted_tables):
        someengine.execute(table.delete())

.. _metadata_reflection_schemas:

从其他模式中反射表
-------------------

在:ref:`schema_table_schema_name`中介绍了表模式的概念，这是数据库内包含表和其他对象的名称空间，并且可以显式地指定。 :class:`_schema.Table`对象以及用于表示其他对象如视图、索引和序列的对象的模式，可以使用参数:paramref:`_schema.Table.schema`设置，并且还可以在:class:`_schema.MetaData`对象上使用 :paramref:`_schema.MetaData.schema`参数作为默认模式。

此模式参数对反映对象的位置产生直接影响。例如，对于一个:class:`_schema.MetaData`对象，如果是假设默认模式名称“project”，则可以使用其:paramref:`_schema.MetaData.schema`参数配置它：

    >>> metadata_obj = MetaData(schema="project")

然后，:meth:`.MetaData.reflect`将使用配置的``.schema``进行反射：

    >>> #使用 metadata_obj 中配置的'schema'
    >>> metadata_obj.reflect(engine)

最终，:class:`_schema.Table`中在“project”架构中的对象将被反映，并且将作为带有该名称的架构限定符被填充：

    >>> metadata_obj.tables["project.messages"]
    Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')

类似地，包括 :paramref:`_schema.Table.schema`参数的 individual :class:`_schema.Table` 对象还可以从该数据库模式中反射，并覆盖可能已经在 :class:`_schema.MetaData` 集合上拥有的默认模式 :paramref:`_schema.MetaData.schema`：

    >>> messages = Table("messages", metadata_obj, schema="project", autoload_with=someengine)
    >>> messages
    Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')

最后，:meth:`_schema.MetaData.reflect`方法本身也允许传递一个 :paramref:`_schema.MetaData.reflect.schema`参数，以供适用于默认配置的:class:`_schema.MetaData`对象从“project”架构中加载表：

    >>> metadata_obj = MetaData()
    >>> metadata_obj.reflect(engine, schema="project")

我们可以使用 :meth:`_schema.MetaData.reflect` 方法任意多次调用具有不同 :paramref:`_schema.MetaData.schema`参数（或者根本不应用）以继续使用 :class:`_schema.MetaData`对象为载体加载对象：


    >>> #从“customer”架构添加表
    >>> metadata_obj.reflect(engine, schema="customer")
    >>> #从默认架构添加表
    >>> metadata_obj.reflect(engine)

.. _reflection_schema_qualified_interaction:

使用模式限定反映的交互
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. admonition:: 整理最佳实践

    在本节中，我们将讨论SQLAlchemy有关表的反射行为，这些表在数据库会话的“缺省模式”中可见，并且这些表如何与显式包含模式的SQLAlchemy指令进行交互。
    作为最佳实践，确保数据库的“默认”模式只是一个名称

如 :ref:`schema_metadata_schema_name` 中所述，具有模式概念的数据库通常也包括模式的“默认”模式概念。自然而然的是，当引用表名称而没有架构时，具有模式功能的数据库仍将考虑该表在某个地方处于“模式”中。例如，一些数据库（如PostgreSQL）将此概念进一步扩展为“模式搜索路径”，其中一个特定数据库会话中的**多个**模式名称被认为是“隐式的”；这将不要求在特定的数据库模式名称中包含模式名称（同时这是在模式名称中包含的也是完全可以的）。

因为许多关系数据库因此拥有这样一种特定的表对象，既可以在模式合格的方式下引用，也可以在“隐式”方式下引用，这对于SQLAlchemy的反射功能构成了复杂性。在模式合格的方式下反映表将始终使用 :attr:`_schema.Table.schema` 属性填充它，并且另外影响如何将此:class:`_schema.Table`组织到 :attr:`_schema.MetaData.tables` 集合中，即以带有schema作为限定符的方式。相反，在非限定方式下反映**同一张**表将以不带schema的方式将其组织到 :attr:`_schema.MetaData.tables` 集合中。因此，单个:class:`_schema.MetaData`集合中存在两个不同的:class:`_schema.Table`对象，表示实际数据库中的相同表。

为了说明这个问题的影响，考虑上面的例子中来自“project”模式的表，并假设“project”模式是数据库连接的默认模式，或者如果使用如PostgreSQL这样的数据库，则假设“project”模式在PostgreSQL中设置为``search_path``。这将意味着如果执行以下两个SQL语句，则数据库将接受为同等的：

.. sourcecode:: sql

    -- 带模式限定
    SELECT message_id FROM project.messages

    -- 不带模式限定
    SELECT message_id FROM messages

这并不是一个问题，因为该表可以找到两种方式。但是在SQLAlchemy中，它的:class:`_schema.Table`对象的**身份**决定了它在SQL语句中的语义角色。根据SQLAlchemy当前的决策，这意味着如果反映明了相同的“messages”表，另一方面没有限制模式和这两个对象是在同一个模式中的，则我们会得到**两个** :class:`_schema.Table`对象，但它们将**不**被视为具有相同的语义：

    >>> #在非限定方式下反射
    >>> messages_table_1 = Table("messages", metadata_obj, autoload_with=someengine)
    >>> #在限定方式下反射
    >>> messages_table_2 = Table(
    ...     "messages", metadata_obj, schema="project", autoload_with=someengine
    ... )
    >>> #两个不同的对象
    >>> messages_table_1 is messages_table_2
    False
    >>> #以不同的方式存储
    >>> metadata.tables["messages"] is messages_table_1
    True
    >>> metadata.tables["project.messages"] is messages_table_2
    True

上述问题在反映的表包含到其他表的外键引用时变得更加复杂。例如，“messages”具有引用到另一表“projects”的“project_id”列，这意味着该引用定义了一个 :class:`_schema.ForeignKeyConstraint` 对象。

我们可能会遇到这样一种情况：单个 : class:`_schema.MetaData` 集合可能会包含四个 : class:`_schema.Table`对象，其中一个或两个附加表被反映过程生成；这是因为当遇到要反射的表上的外键约束时，反射过程会将其引用的表分支为取消引用该模式名称的 :class:`_schema.Table`也包括它的模式，除非省略其模式名称，则在同一个模式中这两个对象属于拥有的 :class:`_schema.Table` 的默认架构会把默认.schema 从反射的: class:`_schema.ForeignKeyConstraint`对象中省略掉，但是在其他情况下包含或要求。常见情况是在架构合格的方式下反射表时，根据已省略其模式名称并且这两个对象在同一模式中这个决策，将且只将该表作为一个附加表：

    >>> #在一个架构合格的模式下反射“messages”
    >>> messages_table_1 = Table(
    ...         "messages", metadata_obj, schema="project", autoload_with=someengine
    ...     )

以上的 `messages_table_1` 将与 `projects` 表相同一样都是以合格的模式方式引用。 接下来，你会得到一个自动生成的 :class:`_schema.Table`，并定义了外键约束，假设它也是以类似的方式引用的：

    >>> #这里在一个现在以限定方式反映的 'puchase' 模式中反射'projects‘
    >>> projects_table_2 = Table(
    ...         "project.projects", metadata_obj, autoload_with=someengine
    ...     )

上面的模式“purchase”中的这  `projects_table_2` 与默认模式下 `projects_table_1` 可能是不同的 :class:`_schema.Table`对象。这就引发一个问题，例如在从模式中反射表以加载应用程序级别 :class:`_schema.Table` 对象的应用程序内部，以及在迁移场景中，尤其是当使用 Alembic 迁移来检测新表或外键约束时。

上述行为可以通过坚持一个简单的规则来解决：

* 不要为希望位于数据库的**默认**模式中的任何 :class:`_schema.Table` 包含 :paramref:`_schema.Table.schema`参数。

对于 PostgreSQL 和其他支持模式搜索路径（schema search path）的数据库，请添加以下额外的实践方法：

* 将“搜索路径”缩小为**一个模式，即默认模式**。

.. seealso::

    :ref:`postgresql_schema_reflection` - 有关PostgreSQL数据库的更多详细信息。

.. _metadata_reflection_inspector:

使用 Inspector 进行细粒度反映
------------------------------

低级接口提供了一种与特定后端无关的加载来自给定数据库的模式列表、表、列和约束描述的系统。这被称为“Inspector”：

    from sqlalchemy import create_engine
    from sqlalchemy import inspect

    engine = create_engine("...")
    inspector = inspect(engine)
    print(inspector.get_table_names())


.. autoclass:: sqlalchemy.engine.reflection.Inspector
    :members:
    :undoc-members:

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedColumn
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedComputed
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedCheckConstraint
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedForeignKeyConstraint
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedIdentity
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedIndex
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedPrimaryKeyConstraint
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedUniqueConstraint
    :members:
    :inherited-members: dict

.. autoclass:: sqlalchemy.engine.interfaces.ReflectedTableComment
    :members:
    :inherited-members: dict


.. _metadata_reflection_dbagnostic_types:

使用数据库非相关类型进行反映
---------------------------------------

当反映表的列时，无论是使用:paramref:`_schema.Table.autoload_with`参数还是:meth:`_reflection.Inspector.get_columns`方法，数据类型都将尽可能与目标数据库保持一致。这意味着如果从MySQL数据库反映“integer”数据类型，则该类型将由:class:`sqlalchemy.dialects.mysql.INTEGER`类表示，该类包含MySQL特定属性，例如“display_width”。或者在PostgreSQL上，可能会返回特定于PostgreSQL的数据类型，例如 :class:`sqlalchemy.dialects.postgresql.INTERVAL` or :class:`sqlalchemy.dialects.postgresql.ENUM`。

当将:class:`_schema.Table`传输到不同的供应商数据库中时，存在一种反映的用例，可以通过拦截使用:meth:`_events.DDLEvents.column_reflect`事件的列反映来将这些供应商特定的数据类型实时转换为SQLAlchemy后端非关联数据类型, 例如上述类型中的 :class:`_types.Integer`, 我们可以选择使用 ``as_generic`` 方法来"泛化"数据类型，这种类型可以将这些供应商特定的类型对象转换为通用的 SQLAlchemy 数据库无关类型，替换特殊数据类型如:class:`sqlalchemy.dialects.mysql.MEDIUMINT` and :class:`sqlalchemy.dialects.mysql.TINYINT` 为:class:`_types.Integer`。

需要使用 :meth:`_events.DDLEvents.column_reflect` 事件建立一个处理程序来插入列反映，同时要使用 :meth:`_types.TypeEngine.as_generic` 方法。 对于MySQL数据库（可能因为MySQL拥有许多供应商特定的数据类型和选项，而选择）：

.. sourcecode:: sql

    CREATE TABLE IF NOT EXISTS my_table (
        id INTEGER PRIMARY KEY AUTO_INCREMENT,
        data1 VARCHAR(50) CHARACTER SET latin1,
        data2 MEDIUMINT(4),
        data3 TINYINT(2)
    )

上面的表包括MySQL专用的整数类型“MEDIUMINT”和“TINYINT”，以及一个“VARCHAR”列，其中包括MySQL专用的“CHARACTER SET”选项。如果我们以正常方式反映此表，则会生成包含这些MySQL特定数据类型和选项的:class:`_schema.Table`对象：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import MetaData, Table, create_engine
    >>> mysql_engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")
    >>> metadata_obj = MetaData()
    >>> my_mysql_table = Table("my_table", metadata_obj, autoload_with=mysql_engine)

以上示例将表模式反映到新的 :class:`_schema.Table` 对象中。我们可以进行简单的演示，输出使用:class:`_schema.CreateTable`构造的MySQL特定"CREATE TABLE"语句：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.schema import CreateTable
    >>> print(CreateTable(my_mysql_table).compile(mysql_engine))
    {printsql}CREATE TABLE my_table (
    id INTEGER(11) NOT NULL AUTO_INCREMENT,
    data1 VARCHAR(50) CHARACTER SET latin1,
    data2 MEDIUMINT(4),
    data3 TINYINT(2),
    PRIMARY KEY (id)
    )ENGINE=InnoDB DEFAULT CHARSET=utf8mb4

上面的MySQL特定数据类型和选项得到了保持。 如果我们要构建一个:class:`_schema.Table`，然后将其传输到不同的供应商数据库中，以此来代替这些供应商特定数据的 `MEDIUMINT` 和 `TINYINT`，使用 :meth:`_types.TypeEngine.as_generic` 方法进行处理，例如以处理上面类型的方式，将'create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP'})

.. sourcecode:: pycon+sql

    >>> pg_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
    >>> my_generic_table.create(pg_engine)
    {execsql}CREATE TABLE my_table (
        id SERIAL NOT NULL,
        data1 VARCHAR(50),
        data2 INTEGER,
        data3 INTEGER,
        PRIMARY KEY (id)
    )

请注意，SQLAlchemy通常会基于可靠猜测，例如MySQL`AUTO_INCREMENT`指令使用最接近PostgreSQL的`SERIAL`自增数据类型。

.. versionadded:: 1.4 添加 :meth:`_types.TypeEngine.as_generic` 方法，同时还使用 :meth:`_events.DDLEvents.column_reflect` 事件的使用方法得到了改善，这样可以方便地应用于 :class:`_schema.MetaData`对象。


反射的局限性
------------------

需要注意的是，反射过程仅使用在关系数据库中表示的信息来重新创建 :class:`_schema.Table` 元数据。这个过程定义不能恢复模式的任何方面，因为它们实际上并未存储在数据库中。反射不可用的状态包括但不限于：

* 客户端默认值，Python函数或使用 :class:`_schema.Column` 的“default”关键字定义的SQL表达式。(请注意，这与 ``server_default``是分开的，这是特定于reflection的。）

* 列信息，例如放置到 :attr:`_schema.Column.info` 字典中的数据

* :class:`_schema.Column` 或 :class:`_schema.Table`的 .quote 设置的值

* 特定的 :class:`.Sequence` 与给定的 :class:`_schema.Column` 的关联

关系数据库还以许多情况下以与在SQLAlchemy中指定的不同格式报告表元数据。从反响得到的:class:`_schema.Table`对象不能始终依赖于生成与原始Python定义的:class:`_schema.Table`对象相同的DDL。这些出现的区域包括服务器默认值、列关联的序列和有关约束和数据类型的各种特性。服务器端默认值可能包含转换指令（通常是PostgreSQL包括“::<type>”转换），或比原来的方式具有不同的引用模式。

另一类限制包括部分或尚未定义反射的模式结构。最近的反射改进使得诸如视图、索引和外键选项等东西可以被反射。截至本写作之时，类似 CHECK 约束、表注释和触发器等结构尚未被反映。