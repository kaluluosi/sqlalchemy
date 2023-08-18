元数据/模式
========

.. contents::
    :local:
     :class: faq
    :backlinks: none

当我使用``table.drop()``/``metadata.drop_all()``时，我的程序会停止运行。

这通常对应于两个情况：
1. 使用PostgreSQL，这在表锁方面非常严格。
2. 您仍有一个打开的连接，其中包含锁定表的连接，与用于DROP语句的连接不同。以下是最简单的模式：

    connection = engine.connect()
    result = connection.execute(mytable.select())

    mytable.drop(engine)

上面的连接池连接仍然被检查出;此外，上面的结果对象还保持与此连接的链接。如果使用“隐式执行”，则结果将保持此连接打开状态，直到关闭结果对象或遍历完所有行。

调用``mytable.drop(engine)``会尝试在从 :class:`_engine.Engine` 获取的第二个连接上发出DROP TABLE命令。

解决方法是在发出DROP TABLE之前关闭所有连接：

    connection = engine.connect()
    result = connection.execute(mytable.select())

    # 全部读取结果集
    result.fetchall()

    # 关闭连接
    connection.close()

    # 现在锁已被删除
    mytable.drop(engine)

SQLAlchemy是否支持ALTER TABLE、CREATE VIEW、CREATE TRIGGER、模式升级功能？

SQLAlchemy没有直接支持ALTER支持。可以使用DDL和相关构造的特殊DDL进行特殊DDL。有关此主题的讨论，请参见  :ref:`metadata_ddl_toplevel` 。

更全面的选项是使用模式迁移工具，例如Alembic或SQLAlchemy-Migrate;请参见 :ref:`schema_migrations` 以获得有关此功能的讨论。

如何按其依赖关系对表对象进行排序？

可以通过  :attr:`_schema.MetaData.sorted_tables`  函数实现：

    metadata_obj = MetaData()
    # ... 添加Table对象到metadata
    ti = metadata_obj.sorted_tables
    for t in ti:
        print(t)

如何将CREATE TABLE/DROP TABLE输出作为字符串获取？

现代SQLAlchemy具有表示DDL操作的子句构造。这些可以像任何其他SQL表达式一样呈现为字符串：

    from sqlalchemy.schema import CreateTable

    print(CreateTable(mytable))

要获取特定于某个引擎的字符串：

    print(CreateTable(mytable).compile(engine))

还有通过 :func:`.create_mock_engine` 获得的 :class:`_engine.Engine` 的特殊形式，允许将整个元数据创建序列转储为字符串，使用以下配方：

    from sqlalchemy import create_mock_engine

    def dump(sql, *multiparams, **params):
        print(sql.compile(dialect=engine.dialect))

    engine = create_mock_engine("postgresql+psycopg2://", dump)
    metadata_obj.create_all(engine, checkfirst=False)

`Alembic <https://alembic.sqlalchemy.org>`_工具还支持将数据库迁移呈现为SQL脚本的“脱机”SQL生成模式。

如何子类化Table/Column以提供某些行为/配置？

 :class:`_schema.Table` 和 :class:`_schema.Column` 不适合直接子类化。然而，可以使用创建函数来获取构建期间的行为，使用约束公约或命名公约等架构对象之间的链接相关行为，例如。可以在“命名约定<https://www.sqlalchemy.org/trac/wiki/UsageRecipes/NamingConventions>`_”中看到这些技术的示例。