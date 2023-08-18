.. _metadata_ddl_toplevel:
.. _metadata_ddl:
.. currentmodule:: sqlalchemy.schema

自定义DDL
===============

在前面的章节中，我们讨论了各种模式结构，包括
:class:`~sqlalchemy.schema.Table`，:class:`~sqlalchemy.schema.ForeignKeyConstraint`，
:class:`~sqlalchemy.schema.CheckConstraint`和
:class:`~sqlalchemy.schema.Sequence`。在整个过程中，我们依靠
:class:`~sqlalchemy.schema.Table`和:class:`~sqlalchemy.schema.MetaData`的``create()``和
:func:`~sqlalchemy.schema.MetaData.create_all`方法，以便为所有构造发出数据定义语言(DDL)。
一旦发出，则会调用预先确定的操作顺序，并无条件地创建用于表的DDL，包括所有约束和与其相关的其他对象。
对于需要数据库特定DDL的更复杂的场景，SQLAlchemy提供了两种技术，可以用于添加任何DDL基于任何条件，可以伴随标准的表生成或单独生成。

自定义DDL
----------

最简单的自定义DDL短语使用:class:`~sqlalchemy.schema.DDL`构造。
该构造与所有其他DDL元素一样工作，只是接受作为要发出的文本的字符串：

.. sourcecode:: python+sql

    event.listen(
        metadata,
        "after_create",
        DDL(
            "ALTER TABLE users ADD CONSTRAINT "
            "cst_user_name_length "
            " CHECK (length(user_name) >= 8)"
        ),
    )

更全面的创建DDL构造库的方法是使用
自定义编译 - 有关详情，请参阅:ref:`sqlalchemy.ext.compiler_toplevel`。

控制DDL序列
-------------------------

前面引入的:class:`_schema.DDL`构造还具有条件调用的能力，
基于对数据库的检查。可以使用该功能，通过:meth:`.ExecutableDDLElement.execute_if`
方法。例如，如果我们想创建一个触发器，但仅在PostgreSQL后端上，请使用如下方式调用::

    mytable = Table(
        "mytable",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("data", String(50)),
    )

    func = DDL(
        "CREATE FUNCTION my_func() "
        "RETURNS TRIGGER AS $$ "
        "BEGIN "
        "NEW.data := 'ins'; "
        "RETURN NEW; "
        "END; $$ LANGUAGE PLPGSQL"
    )

    trigger = DDL(
        "CREATE TRIGGER dt_ins BEFORE INSERT ON mytable "
        "FOR EACH ROW EXECUTE PROCEDURE my_func();"
    )

    event.listen(mytable, "after_create", func.execute_if(dialect="postgresql"))

    event.listen(mytable, "after_create", trigger.execute_if(dialect="postgresql"))

:paramref:`.ExecutableDDLElement.execute_if.dialect`关键字也接受一个元组，
字符串方言名称::

    event.listen(
        mytable, "after_create", trigger.execute_if(dialect=("postgresql", "mysql"))
    )
    event.listen(
        mytable, "before_drop", trigger.execute_if(dialect=("postgresql", "mysql"))
    )

:meth:`.ExecutableDDLElement.execute_if`方法也可以针对一个可调用的函数工作，
在使用的数据库连接。在下面的示例中，我们使用这个来有条件地创建CHECK约束，
首先在PostgreSQL目录中查找它是否存在：

.. sourcecode:: python+sql

    def should_create(ddl, target, connection, **kw):
        row = connection.execute(
            "select conname from pg_constraint where conname='%s'" % ddl.element.name
        ).scalar()
        return not bool(row)


    def should_drop(ddl, target, connection, **kw):
        return not should_create(ddl, target, connection, **kw)


    event.listen(
        users,
        "after_create",
        DDL(
            "ALTER TABLE users ADD CONSTRAINT "
            "cst_user_name_length CHECK (length(user_name) >= 8)"
        ).execute_if(callable_=should_create),
    )
    event.listen(
        users,
        "before_drop",
        DDL("ALTER TABLE users DROP CONSTRAINT cst_user_name_length").execute_if(
            callable_=should_drop
        ),
    )

    users.create(engine)
    {execsql}CREATE TABLE users (
        user_id SERIAL NOT NULL,
        user_name VARCHAR(40) NOT NULL,
        PRIMARY KEY (user_id)
    )

    SELECT conname FROM pg_constraint WHERE conname='cst_user_name_length'
    ALTER TABLE users ADD CONSTRAINT cst_user_name_length  CHECK (length(user_name) >= 8)
    {stop}

    users.drop(engine)
    {execsql}SELECT conname FROM pg_constraint WHERE conname='cst_user_name_length'
    ALTER TABLE users DROP CONSTRAINT cst_user_name_length
    DROP TABLE users{stop}

使用内置的DDLElement类
-------------------------------------

``sqlalchemy.schema``包含SQL表达式构造，提供DDL表达式，所有这些构造都从公共基类:class:`.ExecutableDDLElement`继承。
例如，要产生一个``CREATE TABLE``语句，可以使用:class:`.CreateTable`构造：

.. sourcecode:: python+sql

    from sqlalchemy.schema import CreateTable

    with engine.connect() as conn:
        conn.execute(CreateTable(mytable))
    {execsql}CREATE TABLE mytable (
        col1 INTEGER,
        col2 INTEGER,
        col3 INTEGER,
        col4 INTEGER,
        col5 INTEGER,
        col6 INTEGER
    ){stop}

上面的:class:`~sqlalchemy.schema.CreateTable`构造类似于任何其他表达式构造(如``select()``, ``table.insert()``等)。
SQLAlchemy的所有针对DDL的结构都是:class:`.ExecutableDDLElement`基类的子类;这是所有相应于CREATE和DROP的对象的基础，以及ALTER，
不仅在SQLAlchemy中，而且在Alembic Migrations中也是如此。有关可用结构的完整参考在:ref:`schema_api_ddl`中。

用户定义的DDL结构也可以作为:class:`.ExecutableDDLElement`的子类创建。
:ref:`sqlalchemy.ext.compiler_toplevel`中的文档提供了多个示例。


控制约束和索引的DDL生成
-----------------------------------------------------
    
.. versionadded:: 2.0

虽然以前提到的:meth:`.ExecutableDDLElement.execute_if`方法对于需要调用条件的自定义:class:`.DDL`类很有用，
但常见的用途是，与特定:class:`.Table`相关联的元素，即约束和索引，
也要受到“有条件”的规则的影响，例如包括特定于特定后端的功能的索引，如PostgreSQL或SQL Server。
为此，可以使用 :meth:`.Constraint.ddl_if`和
:meth:`.Index.ddl_if`针对诸如
:class:`.CheckConstraint`、:class:`.UniqueConstraint`和:class:`.Index`之类的结构，
接受与:meth:`.ExecutableDDLElement.execute_if`方法相同的参数，
以通过其父级:class:`.Table`对象控制它们的DDL是否将被发出。这些方法可以内联使用，
例如在创建:class:`.Table`的定义时（或类似地，在ORM声明性映射中使用``__table_args__``集合），如下所示：

    from sqlalchemy import CheckConstraint, Index
    from sqlalchemy import MetaData, Table, Column
    from sqlalchemy import Integer, String

    meta = MetaData()

    my_table = Table(
        "my_table",
        meta,
        Column("id", Integer, primary_key=True),
        Column("num", Integer),
        Column("data", String),
        Index("my_pg_index", "data").ddl_if(dialect="postgresql"),
        CheckConstraint("num > 5").ddl_if(dialect="postgresql"),
    )

在上面的示例中，:class:`.Table`结构引用
:class:`.Index`和:class:`.CheckConstraint`结构，两者都
表示已经``.ddl_if(dialect="postgresql"）``,
这表明这些元素只会出现在PostgreSQL方言中的CREATE TABLE序列中。
如果我们对SQLite方言运行``meta.create_all()``，例如，将不包含任何结构：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import create_engine
    >>> sqlite_engine = create_engine("sqlite+pysqlite://", echo=True)
    >>> meta.create_all(sqlite_engine)
    {execsql}BEGIN (implicit)
    PRAGMA main.table_info("my_table")
    [raw sql] ()
    PRAGMA temp.table_info("my_table")
    [raw sql] ()

    CREATE TABLE my_table (
        id INTEGER NOT NULL,
        num INTEGER,
        data VARCHAR,
        PRIMARY KEY (id)
    )

但是，如果我们对PostgreSQL数据库运行相同的命令，我们将会
看到CHECK约束的内联DDL以及为索引发出的单独CREATE语句：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import create_engine
    >>> postgresql_engine = create_engine(
    ...     "postgresql+psycopg2://scott:tiger@localhost/test", echo=True
    ... )
    >>> meta.create_all(postgresql_engine)
    {execsql}BEGIN (implicit)
    select relname from pg_class c join pg_namespace n on n.oid=c.relnamespace where pg_catalog.pg_table_is_visible(c.oid) and relname=%(name)s
    [generated in 0.00009s] {'name': 'my_table'}

    CREATE TABLE my_table (
        id SERIAL NOT NULL,
        num INTEGER,
        data VARCHAR,
        PRIMARY KEY (id),
        CHECK (num > 5)
    )
    [no key 0.00007s] {}
    CREATE INDEX my_pg_index ON my_table (data)
    [no key 0.00013s] {}
    COMMIT

:meth:`.Constraint.ddl_if`和:meth:`.Index.ddl_if`方法创建了
事件挂钩，可以在DDL执行时咨询，就像:meth:`.ExecutableDDLElement.execute_if`的行为一样，
但也可以在：class:`.CreateTable`对象的SQL编译阶段内存储，该对象负责内联呈现``CHECK (num > 5)``
DDL，嵌入在CREATE TABLE语句中。因此，:meth:`.Constraint.ddl_if.callable_`参数所接收的事件挂钩具有更丰富的参数设置，
包括传递了``dialect``关键字参数，以及通过``compiler``关键字参数传递的: class:`.DDLCompiler`实例，
用于“内联呈现”部分的序列。当在:class:`.DDLCompiler`序列中触发该事件时，
``bind``参数不存在，因此，希望检查数据库版本信息的现代事件挂钩最好使用给定的:class:`.Dialect`对象，
例如测试PostgreSQL版本信息：

.. sourcecode:: python+sql

    def only_pg_14(ddl_element, target, bind, dialect, **kw):
        return dialect.name == "postgresql" and dialect.server_version_info >= (14,)


    my_table = Table(
        "my_table",
        meta,
        Column("id", Integer, primary_key=True),
        Column("num", Integer),
        Column("data", String),
        Index("my_pg_index", "data").ddl_if(callable_=only_pg_14),
    )

.. seealso::

    :meth:`.Constraint.ddl_if`

    :meth:`.Index.ddl_if`


DDL表达式构造API
-----------------------------

.. autofunction:: sort_tables

.. autofunction:: sort_tables_and_constraints

.. autoclass:: BaseDDLElement
    :members:

.. autoclass:: ExecutableDDLElement
    :members:

.. autoclass:: DDL
    :members:

.. autoclass:: _CreateDropBase

.. autoclass:: CreateTable
    :members:


.. autoclass:: DropTable
    :members:


.. autoclass:: CreateColumn
    :members:


.. autoclass:: CreateSequence
    :members:


.. autoclass:: DropSequence
    :members:


.. autoclass:: CreateIndex
    :members:


.. autoclass:: DropIndex
    :members:


.. autoclass:: AddConstraint
    :members:


.. autoclass:: DropConstraint
    :members:


.. autoclass:: CreateSchema
    :members:


.. autoclass:: DropSchema
    :members: