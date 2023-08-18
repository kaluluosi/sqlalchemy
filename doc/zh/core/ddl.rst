自定义 DDL
==================

在前面的章节中，我们讨论了许多模式构造，包括  :class:`~sqlalchemy.schema.Table` 、
  :class:`~sqlalchemy.schema.ForeignKeyConstraint` 、
  :class:`~sqlalchemy.schema.CheckConstraint` ~sqlalchemy.schema.Sequence` 。
在整个过程中，我们都依赖于 :class:`~sqlalchemy.schema.Table` 和
  :class:`~sqlalchemy.schema.MetaData` ` create()``和
 :func:`~sqlalchemy.schema.MetaData.create_all` 方法来发出数据定义语言 (DDL)，以便创建所
有构造的视图。当发出时，会调用一个预定义的操作顺序，并创建用于创建每张表的DDL，包括所
有与之相关的约束和其他对象。对于需要特定于数据库的DDL的更复杂场景，SQLAlchemy提供了
两种技术，可以用于添加任何DDL，基于任何条件，要么随着表的标准生成，要么仅仅是DDL自
身。

用户自定义 DDL
---------------

最简单的自定义 DDL 语句是使用   :class:`~sqlalchemy.schema.DDL` 。该构造类似于所有其他 DDL 
元素，只不过它接受一个字符串，即要发出的文本：

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

创建 DDL 库的更全面的方法是使用自定义编译 - 有关详细信息，请参见
  :ref:`sqlalchemy.ext.compiler_toplevel` 。

控制 DDL 序列
-------------------------

前面介绍的   :class:`_schema.DDL`  元素也可以根据数据库的检查而有条件地调用。需要使
用  :meth:`.ExecutableDDLElement.execute_if`  方法。例如，如果我们想创建触发器，但仅在
PostgreSQL 后端上，我们可以这样调用：

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

  :meth:`.ExecutableDDLElement.execute_if.dialect`  关键字也接受一组字符串方言名称：

    event.listen(
        mytable, "after_create", trigger.execute_if(dialect=("postgresql", "mysql"))
    )
    event.listen(
        mytable, "before_drop", trigger.execute_if(dialect=("postgresql", "mysql"))
    )

  :meth:`.ExecutableDDLElement.execute_if`  方法也可以针对将使用数据库连接的可调用函数，例如，
在下面的示例中，我们使用此功能有条件地创建 CHECK 约束，首先在 PostgreSQL 目录中查找
它是否存在：

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

使用内置 DDLElement 类
----------------------------

``sqlalchemy.schema`` 包包含 SQL 表达式构造，这些构造提供了 DDL 表达式，所有这些构造都扩展
自公共基类  :class:`.ExecutableDDLElement` 。例如，要生成` `CREATE TABLE``语句，可以使用
  :class:`.CreateTable`  构造：

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

上面的  :class:`~sqlalchemy.schema.CreateTable`  构造像其他任何表达式构造一样工作（例如` `select()``
、``table.insert()``
等）。所有 SQLAlchemy 的 DDL 导向构造都是   :class:`.ExecutableDDLElement`  基类的子类；这
是所有对应 CREATE和 DROP 的对象的基础，不仅在 SQLAlchemy 中，而且在 Alembic 迁移中也是如此。
可用构造的完整参考在   :ref:`schema_api_ddl`  中。

用户定义的 DDL 构造也可以作为  :class:`.ExecutableDDLElement`  的子类进行创建。参见   :ref:` sqlalchemy.ext.compiler_toplevel`  中的几个示例。

控制约束和索引的 DDL 生成
-------------------------------------------

.. versionadded:: 2.0

虽然先前提到的  :meth:`.ExecutableDDLElement.execute_if`  方法对于需要有条件地调用的自定义 
  :class:`.DDL`  类非常有用，但约束和索引通常与特定的   :class:` .Table`  相关联，即它们也应
该遵循"条件"规则，
例如包含特定于特定后端的功能，例如 PostgreSQL 或 SQL Server 的索引。对于这种用例，
  :meth:`.Constraint.ddl_if`   和  :meth:` .Index.ddl_if`  方法可能会针对 :
  :class:`.CheckConstraint` 、
  :class:`.UniqueConstraint`  和   :class:` .Index`  等构造，接受与
  :meth:`.ExecutableDDLElement.execute_if`   方法相同的参数，以控制它们是否会根据其父 
  :class:`.Table`  对象发出DDL。这些方法可以在创建   :class:` .Table`  的定义时内联使用
（或类似地，在 ORM 声明性映射的 ``__table_args__`` 集合中使用）, 如下所示：

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

在上面的示例中，  :class:`.Table`  构造引用了一个  :class:` .Index`  和一个 
  :class:`.CheckConstraint`  构造，两者都表示使用了` `.ddl_if(dialect="postgresql")``，这
表示这些元素仅会针对 PostgreSQL 方言包含在 CREATE TABLE 序列中。例如，如果我们针对 SQLite 方言运行 
``meta.create_all()``，则两个构造都不包含在内：

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

但是，如果我们将相同的命令针对 PostgreSQL 数据库运行，则将看到 CHECK 约束的内联 DDL 构造，
以及单独的 CREATE 语句针对索引发出：

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

  :meth:`.Constraint.ddl_if`   和  :meth:` .Index.ddl_if`  方法创建一个事件挂钩，该挂钩可以不仅在
DDL 执行时间上进行参考，就像  :meth:`.ExecutableDDLElement.execute_if`  的行为一样，而
且还可以在   :class:`.CreateTable`  对象的 SQL 编译阶段中进行查看，该对象负责渲染
``CHECK (num > 5)`` DDL 在 CREATE TABLE 语句中内联。因此，  :meth:`.Constraint.ddl_if.callable_` 
参数接收到的事件挂钩具有更丰富的参数集，其中包括传递了``dialect`` 关键字参数，以及类
  :meth:`.DDLCompiler`   的实例通过“编译器”关键字参数进行内联呈现。当在  :meth:` .DDLCompiler`  
序列内触发事件时，``bind`` 参数**不**存在，因此希望检查数据库版本信息的现代事件挂钩最好使
用给定的   :class:`.Dialect`  对象，例如测试 PostgreSQL 版本：

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

.. _schema_api_ddl:

DDL 表达式构造 API
----------------------------------

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