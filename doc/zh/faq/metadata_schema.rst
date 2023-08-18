=================
元数据 / 模式
=================

.. contents::
    :local:
    :class: faq
    :backlinks: none


当我说 “table.drop()” / “metadata.drop_all()” 时，程序卡住了
===========================================================================

通常有两种情况与之对应：1. 使用 PostgreSQL，它对表锁非常严格，2. 你仍然有一个打开的连接，其中包含对表的锁定，并且与用于 DROP 语句的连接不同。这是模式的最简版本：

    connection = engine.connect()
    result = connection.execute( mytable.select() )

    mytable.drop(engine)

上述代码中，仍然存在一个连接池连接；此外，上面的结果对象也维护着与此连接的链接。如果使用“隐式执行”，则结果将保持该连接打开，直到结果对象被关闭或所有行耗尽。

调用 mytable.drop(engine) 会尝试在从 :class:`_engine.Engine` 获取的第二个连接上发出 DROP TABLE 操作，将会发生锁定。

解决方法是在发出 DROP TABLE 之前关闭所有连接：

    # close out all connections before emitting DROP TABLE
    connection = engine.connect()
    result = connection.execute( mytable.select() )

    # fully read result sets
    result.fetchall()

    # close connections
    connection.close()

    # now locks are removed
    mytable.drop(engine)

SQLAlchemy 是否支持 ALTER TABLE、CREATE VIEW、CREATE TRIGGER、模式升级功能？
===============================================================================================


SQLAlchemy 直接不支持常规 ALTER。可以使用 :class:`.DDL` 和相关构造对特殊 DDL 进行操作。有关此主题的讨论，请参阅 :ref:`metadata_ddl_toplevel`。

更全面的选择是使用模式迁移工具，例如 Alembic 或 SQLAlchemy-Migrate。请参阅 :ref:`schema_migrations` 进行讨论。

如何按它们的依赖关系对表对象进行排序？
==========================================================

可以使用 :attr:`_schema.MetaData.sorted_tables` 函数来实现此功能：

    metadata_obj = MetaData()
    # ... add Table objects to metadata
    ti = metadata_obj.sorted_tables
    for t in ti:
        print(t)

.. _faq_ddl_as_string:

如何将 CREATE TABLE / DROP TABLE 输出作为字符串获取？
==============================================================

现代 SQLAlchemy 具有表示 DDL 操作的子句构造。这些可以像任何其他 SQL 表达式一样呈现为字符串：

    from sqlalchemy.schema import CreateTable

    print(CreateTable(mytable))

要获取特定引擎的字符串：

    print(CreateTable(mytable).compile(engine))

还有一种特殊形式的 :class:`_engine.Engine`，通过 :func:`.create_mock_engine` 可用，允许将整个元数据创建序列转储为字符串，使用以下方法：

    from sqlalchemy import create_mock_engine


    def dump(sql, *multiparams, **params):
        print(sql.compile(dialect=engine.dialect))


    engine = create_mock_engine("postgresql+psycopg2://", dump)
    metadata_obj.create_all(engine, checkfirst=False)

“Alembic <https://alembic.sqlalchemy.org>`_” 工具还支持“离线”SQL 生成模式，可将数据库迁移渲染为 SQL 脚本。

如何子类化 Table / Column 以提供某些行为/配置？
============================================================================

:class:`_schema.Table` 和 :class:`_schema.Column` 不是直接子类化的好目标。可以使用创建函数来获得构造行为，使用约束公约或命名公约之类的链接模式相关行为，例如使用附件事件。可以在 `命名约定`_（Naming Conventions）中看到许多这些技术的示例。