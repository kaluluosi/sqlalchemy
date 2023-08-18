=============
1.3 更新日志
=============

.. changelog_imports::

    .. include:: changelog_12.rst
        :start-line: 5

    .. include:: changelog_11.rst
        :start-line: 5

.. changelog::
    :version: 1.3.25
    :include_notes_from: unreleased_13

.. changelog::
    :version: 1.3.24
    :released: 2021年3月30日

    .. change::
        :tags: bug, schema
        :tickets: 6152

        修复了最初在 :ticket:`2892`, :ticket:`2919` 和 :ticket:`3832` 中引入的错误，这些错误导致对于 :class:`_types.TypeDecorator` 的附加事件将与 "impl" 类重复，如果 "impl" 也是 :class:`_types.SchemaType`，会产生更多的问题。真实场景是，在设置 ``create_constraint=True`` 标志的情况下，任何 :class:`_types.TypeDecorator` 相对于 :class:`_types.Enum` 或 :class:`_types.Boolean` 都会得到一个重复的 :class:`_schema.CheckConstraint`。

    .. change::
        :tags: bug, schema, sqlite
        :tickets: 6007
        :versions: 1.4.0

        修复了 :class:`_types.Boolean` 或 :class:`_types.Enum` 生成的 CHECK 约束无法正确渲染命名约定的问题。这是由于其名称中的意外更改而导致的。此问题最初在 :ticket:`3067` 中修复，但是，该修复操作的方式似乎比所需的方式更为复杂。现在修复的方法更加简单有效。

    .. change::
        :tags: bug, orm
        :tickets: 5983
        :versions: 1.4.0

        移除了非常旧的警告声明，表示 passvie_deletes 不适用于许多对一关系。尽管在许多情况下，在许多对一关系上放置此参数可能不是所期望的，但是可能会有不希望从此类关系中级联删除的用例。

    .. change::
        :tags: bug, postgresql
        :tickets: 5989
        :versions: 1.4.0

        修复了使用 :class:`_postgresql.aggregate_order_by` 会在某些情况下返回 ARRAY(NullType) 的问题，该问题会干扰结果对象正确返回数据的能力。

    .. change::
        :tags: bug, schema
        :tickets: 5919
        :versions: 1.4.0

        修复了使用列名/键等作为约定的主键约束的支持，尤其是当 :class:`.PrimaryKeyConstraint` 对象与 ：class:`~.schema.Table` 自动关联时，将更新其名称，当将新的主键 :class:`_schema.Column` 对象添加到表中并将其添加到约束中。可以处理与此约束构造过程相关的内部故障模式，包括没有列存在、没有名称存在或存在空名称。

    .. change::
        :tags: bug, schema
        :tickets: 6071
        :versions: 1.4.3

        调整了删除多个表之间 :class:`_schema.Sequence` 对象的 DROP 语句的逻辑，使得所有 :class:`_schema.Sequence` 对象在所有表之后被删除，即使该 :class:`_schema.Sequence` 仅与 :class:`_schema.Table` 对象而不是直接与整个 :class:`_schema.MetaData` 对象相关联。该用例支持相同的 :class:`_schema.Sequence` 将同时与多个 :class:`_schema.Table` 关联的情况。

    .. change::
        :tags: bug, orm
        :tickets: 5952
        :versions: 1.4.0

        修复了在连接两个表的过程中，如果其中一个表具有无关的、无法解析的外键约束，则会引发 :class:`_exc.NoReferenceError`，虽然这个错误可以通过绕过来完成连接过程。在这个过程中测试异常的逻辑会使假设构造失败。

    .. change::
        :tags: bug, postgresql, reflection
        :tickets: 6161
        :versions: 1.4.4

        修复了 PostgreSQL 反射中出现的问题，其中表达 “NOT NULL” 的列将覆盖对应域的空值约束。

    .. change::
        :tags: bug, engine
        :tickets: 5929
        :versions: 1.4.0

        修复了在 ``schema_translate_map`` 功能在直接执行 :class:`_schema.DefaultGenerator` 对象（例如序列）的情况下无法考虑的问题，包括它们被“预执行”以在禁用 implicit_returning 时生成主键值时的情况。

    .. change::
        :tags: bug, orm
        :tickets: 6001
        :versions: 1.4.0

        修复了 :class:`_mutable.MutableComposite` 构造可能被放置到无效状态的问题，当父对象已经加载时，然后被后续查询覆盖，由于复合属性的刷新处理程序将对象替换为不受 mutable 扩展处理的新对象。

    .. change::
        :tags: bug, types, postgresql
        :tickets: 6023
        :versions: 1.4.3

        调整了 psycopgl2 语言方言以发出绑定参数的显式 postgresql 样式转换，其中包含ARRAY 元素。这允许数组内的全部数据类型在数组中正确处理。asyncpg 方言已经在最终语句中生成了这些内部转换。这还包括阵列切片更新以及特定于 PostgreSQL 的 :meth:`~._postgresql.ARRAY.contains` 方法的支持。

    .. change::
        :tags: bug, sql
        :tickets: 5816

        修复了 :meth:`.TypeEngine.with_variant` 方法在 :class:`.TypeDecorator` 类型上失败的问题，该方法未考虑正在使用的方言特定映射。

    .. change::
        :tags: usecase, mysql
        :tickets: 5808

        在 MySQL >=(8,0,17) 和 MariaDb >=(10, 4, 5) 中现在支持向 ``FLOAT`` 进行转换。

.. changelog::
    :version: 1.3.23
    :released: 2021年2月1日

    .. change::
        :tags: bug, ext
        :tickets: 5836

        修复了列为未标记的自定义 SQL 构造且未提供默认编译形式的情况下，有时会在尝试生成可选择的列集合键时调用字符串化的问题。虽然这似乎是一个不寻常的情况，但它可能会在某些 ORM 场景中触发，例如与连接的急加载一起在 "order by" 中使用表达式。问题在缺少默认编译器函数时引发了 :class:`.CompileError` 而不是 :class:`.UnsupportedCompilationError`。

    .. change::
        :tags: bug, postgresql
        :tickets: 5645

        对于 SQLAlchemy 1.3 版本，setup.py 将 pg8000 锁定到低于 1.16.6 的版本。版本 1.16.6 及以上由 SQLAlchemy 1.4 支持。感谢 Giuseppe Lumia 贡献了拉取请求。

    .. change::
        :tags: bug, sql
        :tickets: 5691

        如果多次调用返回 :meth:`_sql.Insert.returning` 这样的返回() 方法，将引发警告。因为当前版本不支持逐渐操作。在 1.4 中将支持此行为。另外，任何组合使用 :meth:`_sql.Insert.returning` 和 :meth:`_sql.ValuesBase.return_defaults` 方法现在都会引发错误，因为这些方法是相互排斥的；之前的操作会静默失败。

    .. change::
        :tags: bug, oracle
        :tickets: 5813

        修复了 SQLAlchemy 1.3.11 中由 :ticket:`4894` 引入的 Oracle 方言中的回归错误，其中在针对 UPDATE 的 RETURNING 中使用 SQL 表达式将无法编译，因为在不是列的任意 SQL 表达式情况下，将检查 “server_default”。

    .. change::
        :tags: bug, mysql
        :tickets: 5821

        修复了由于解决 :ticket:`5462` 中出现的错误。该错误会在索引的 MySQL 函数表达式中需要双圆括号时引发警告，并且在必要时自动添加额外的圆括号。但是，这意外地扩展到了包括 Alembic 的内部文本组件以及：class:`~.schema.Index` 表达式意外地添加了双圆括号。已经将检查限制在只包括直接的 :class:`_expression.BinaryExpression`、:class:`_expression.UnaryExpression` 和 :class:`_expression.FunctionExpression` 的表达式。

    .. change::
        :tags: usecase, mysql
        :tickets: 5800

        修复了由于 :ticket:`5462` 的修复操作，对于 MySQL，函数表达式在索引中无法正确渲染。因为 MySQL 函数表达式需要包含在双圆括号中，所以修复了不包含在双圆括号中的 SQL 函数表达式的问题。之前的更改不包括 Alembic 的需要在索引表达式中使用 SQL 文本的情况。

.. change :: 
        :tags: bug, mssql
        :tickets: 5039

        添加 ``.schema`` 参数到 ``_expression.table`` 构造体中，支持临时表达式包含模式名 。来自 Dylan Modesitt 的拉取请求。

    .. change::
        :tags: bug, mssql
        :tickets: 5339

        修正了 ``datetime.time`` 参数被转换为``datetime.datetime`` 的问题。之前这使得它们无法像“>=”一样直接和实际的 :class:`_mssql.TIME` 列进行比较。

    .. change::
        :tags: bug, mssql
        :tickets: 5359

        修复了 SQL Server 的pyodbc类方言中``is_disconnect``函数错误地报告断开连接状态的问题，这是因为异常消息中有一个与 SQL Server ODBC 错误代码匹配的子字符串。


    .. change::
        :tags: bug, engine
        :tickets: 5326

        进一步修复了“重置”代理“reset”代理中的问题。如果未正确调用，则会发出警告并进行更正。发现了额外的场景并修复了这些问题，这些问题是警告被触发的位置。

        
    .. change::
        :tags: usecase, sqlite
        :tickets: 5297

        SQLite 3.31 在已添加支持计算列。对SQLite目标运行的SQLAlchemy也支持计算列。

        
    .. change::
        :tags: bug, schema
        :tickets: 5276

        在使用 :func: `.tometadata` 复制数据库对象（例如，:class：`。Table`）时，修正了忽略 ``dialect_options`` 的问题。

    .. change::
        :tags: bug, sql
        :tickets: 5344

        正确应用 type_coerce 元素中的 self_group。使用时该类型元素在表达式中没有正确应用分组规则。

        
    .. change::
        :tags: bug, oracle, reflection
        :tickets: 5421

        修正了 Oracle 方言中的一个bug，即索引中包含完整的主键列，则会将包含主键约束完整设置为主键索引本身，即使存在多个索引。现在，该检查已被细化，以将主键约束的名称与索引名称本身进行比较，而不是基于索引中存在的列来猜测。

        
    .. change::
        :tags: change, sql, sybase
        :tickets: 5294

        在Sybase方言中添加了``.offset`` 支持。通过 Alan D. Snow 的拉取请求。


    .. change::
        :tags: bug, engine
        :tickets: 5341

        修复了 :class:`.URL` 对象中存在的问题，即当对其进行字符串化时，不会对特殊字符进行URL编码，从而防止URL被重新消耗为真实URL。公关请求由 Miguel Grinberg。


    .. change::
        :tags: usecase, mysql
        :tickets: 4860

         实现了为 MySQL 提供行级锁支持。来自 Quentin Somerville 的拉取请求。


    .. change::
        :tags: change, mssql
        :tickets: 5321

        从``PyODBCConnector``一级移到``MSDialect_pyodbc``，因为在某些情况下 pyodbc 可以正常工作。

    .. change::
        :tags: change, examples

        为性能测试套件（``examples.performance``）添加了新选项 ``--raw`` ，可以将原始配置文件转储为任何数量的性能分析可视化工具。删除了“runsnake”选项，因为目前很难构建runsnake。

    .. change::
        :tags: usecase, postgresql
        :tickets: 5265

        在PostgreSQL中添加了对:class:`_sqltypes.ARRAY` of :class:`.Enum`, :class:`_postgresql.JSON` 或 :class:`_postgresql.JSONB`类型的列的支持。在这些使用案例中以前需要使用解决方法。
        
        
    .. change::
        :tags: schema
        :tickets: 4138

        在 ``_schema.Column`` 的 ``__repr__()`` 方法中增加了 ``comment`` 属性。

              
    .. change::
        :tags: orm, usecase
        :tickets: 5237

        在 ：func：`_orm.query_expression` 构造体中添加了一个新参数:paramref：``_orm.query_expression.default_expr``，它将在自动请求时自动应用于查询 因为与表达式 : func：`_orm.with_expression` 选项冲突。问候由 Haoyu Sun 提供。

.. changelog::
    :version: 1.3.17
    :released: May 13, 2020

        
    .. change::
        :tags: bug, orm
        :tickets: 5194

        修改了 :meth:`_query.Query.join` 所报告的错误消息，当无法定位到左手边时， :meth:`_query.Query.select_from` 方法是解决此问题的最佳方式。在该系列中的版本 1.3 中确定了来自给定列实体的 FROM 子句的确定性顺序，因此每次确定该表达式时都将确定每个表的该组件。

        
    .. change::
        :tags: bug, orm
        :tickets: 5196

        固定了引起1.3.13的回归 :ticket:`4849`，当flush错误会出现SYS表时，sys.exc_info()函数调用未正确调用的问题。此异常出现时增加了测试覆盖范围。

然而，执行:class:`.Compiled`对象不应该在其状态中引起任何更改，特别是考虑到一旦完全构建就可以重复使用和线程安全性。

.. change::
    :tags: tests, postgresql
    :tickets: 5057

    通过测试`max_prepared_transactions`是否设置为大于0的值来改进PostgreSQL数据库中双阶段事务需求的检测。感谢Federico Caselli提供请求。

.. change::
    :tags: bug, orm, engine
    :tickets: 5056, 5050, 5071

    添加了测试支持并修复了故意为之创建无意义引用周期的一些短暂对象（主要是ORM查询）的问题。在这方面，感谢Carson Ip的帮助。

.. change::
    :tags: orm, bug
    :tickets: 5107

    修正了加载器选项中一个引入的回归（通过：票号:`4468`），在该回归中，使用:meth:`.PropComparator.of_type`使用加载器选项定位其前导关系引用的继承子类的别名实体的能力将无法产生匹配路径。请参阅同一版本中解决类似问题的:票号:`5082`。

.. change::
    :tags: bug, tests
    :tickets: 4946

    修复了一些测试失败，这些失败会在Windows上由于SQLite文件锁定问题以及与连接池相关的测试中的一些计时问题而发生。感谢Federico Caselli提供请求。

.. change::
    :tags: orm, bug
    :tickets: 5082

    修复了joined eager加载在1.3.0b3中的回归，通过`4468`添加了跨入多态子类的:功能:`.with_polymorphic`创建联接选项的能力，然后进一步的映射关系会失败，因为多态的子类无法以可以通过加载器策略定位的方式将自己添加到加载路径中。进行了微调以解决此场景。

.. change::
    :tags: performance, orm

    通过一种基于映射关系构建连接的系统中出现的性能问题。连接表达式的子句调整系统将用于大多数连接表达式，包括在没有适应的情况下的常见情况。已经调整了发生适应的条件，以便简单关系的平均不带别名的连接而不带“secondary”表使用大约少70%的函数调用。

.. change::
    :tags: usecase, postgresql
    :tickets: 5040

    添加了前缀支持到:class:`_expression.CTE`结构，以允许支持Postgresql 12“MATERIALIZED”和“NOT MATERIALIZED”短语。感谢Marat Sharafutdinov提供的请求。

    .. seealso::
        :meth:`_expression.HasCTE.cte`

.. change::
    :tags: bug, mssql
    :tickets: 5045

    修正了使用 :class:`_mssql.DATETIMEOFFSET`列的时区感知日期时间值被转换为用作参数值的字符串时省略小数秒的问题。

.. change::
    :tags: bug, orm
    :tickets: 5068

    在删除使用“version_id”功能的对象时，修复ORM flush进程中的一个未被测试覆盖的警告。该警告一般不可达，除非使用将“supports_sane_rowcount”标志设置为False的方言，这通常不是情况，但对于某些MySQL配置以及较旧的Firebird驱动程序以及可能是某些第三方方言，这是可能的。

.. change::
    :tags: bug, orm
    :tickets: 5065

    修复了使用连接式急切加载时不会正确包装查询以针对:meth:`_query.Query.group_by`用于查询的情况的回归。当使用了任何形式的结果限制方法，例如 DISTINCT、LIMIT、OFFSET 时，连接急切加载将所限制的行查询嵌入到子查询中，以便集合结果不受影响。由于某种原因，从未包括这种标准的 GROUP BY 表示标准，即使它的效果与使用 DISTINCT 相似。此外，大多数数据库平台要求查询中不包含非聚合、非分组列，因为连接急切加载的其他列将不被数据库接受。

.. changelog::
    :version: 1.3.12
    :released: December 16, 2019

    .. change::
        :tags: bug, sql
        :tickets: 5028

        修复了在向 :func:`_expression.select` 传递“distinct”关键字时，string值不会像:meth:`_expression.select.distinct`一样被视为“label引用”的问题；它的运作将引发异常而不是从始至终。。

    .. change::
        :tags: bug, orm
        :tickets: 4997

        修复“lazy='raise'”策略的问题，该问题意味着 ORM 删除使用配置了 lazy='raise' 的简单的“use-get”风格多对一关系的对象将被引发。这与第一次级别的操作不发出SQL而是绕过“lazy='raise'”检查有所不同，而是对这种情况下的“lazy='raise'”有效地当作是“lazy='raise_on_sql'”。修复了懒惰加载策略，如果懒惰加载策略被指示，它不应在未出现该对象的情况下发出SQL。

    .. change::
        :tags: bug, sql
        将“无法解析标签引用”的异常文本更改为包含其他类型的标签强制转换，即PostgreSQL dialect下的“DISTINCT”。

    .. change::
        :tags: bug, orm
        :tickets: 5000

        修复了与 :ticket:`4351`相关联的关系代理的重构在使用引用它们的协会代理的情况下不起作用的回归。

    .. change::
        :tags: bug, mssql
        :tickets: 4983

        通过增加基于PyODBC的结果处理程序解决了在PyODBC上的 :class:`_mssql.DATETIMEOFFSET` 数据类型的支持问题，因为它不包括本机支持此数据类型。这包括使用Python 3“timezone”tzinfo子类，以便设置时区，在 Python 2 中，它使用了 sqlalchemy.util 的“timezone”的最小后备。

    .. change::
        :tags: bug, orm
        :tickets: 4993

        将 :paramref:`_orm.relationship.omit_join` 标志设置为True将不再被意外设置，并且将发出相应的警告，因为这些标志对于标记为视图的关系不起作用。特别地，“级联”设置有它们自己的警告，这些设置是基于单个值生成的，例如“delete、delete-orphan”，不应该应用于标记为“view-only”的关系。不过，即使在标记为“view-only”的关系中，这些设置仍会在某些平台下生效，这些平台禁止不聚合、未分组的列出现在查询中，因为不能被接受。这个问题已经解决。

    .. change::
        :tags: bug, sqlite
        :tickets: 5014

        修复了围绕SQLite行为的问题，该行为在“pyformat”或“named”格式的绑定参数中干扰，而对于标量数值的JSON值会被作为数字而不是可以JSON反序列化的字符串返回。SQLite特定的JSON反序列化器现在会将这种情况优雅地降级为异常，并将对于单个数值的JSON不进行反序列化。

    .. change::
        :tags: bug, orm, py3k
        :tickets: 4990

        修复了当将集合作为分片分配给本身时，突变操作将在意外清除被分配集合的情况下将其置于失败的问题。由于不会生成事件的内容不应该产生事件，操作现在是无操作。注意，该修复仅适用于Python 3；在Python 2中，如果一个复合为None的事件涉及集合，不会调用__setitem__钩子；而是使用__setslice__，它在所有情况下逐个重新创建列表项。.. version: 1.3.8
.. released: August 27, 2019

- Fixed a bug where **and_** queries would incorrectly eliminate **True** components of the condition in some cases
- Fixed a bug when querying with **or\_** on a class that's a join of two entities which could cause an incorrect **DISTINCT** to be rendered
- Fixed a bug where binary types like :class:`.LargeBinary` would not coerce **bytearray** objects correctly
- Fixed a bug where the `json_serializer` and `json_deserializer` parameters for MySQL and SQLite dialects were named incorrectly causing them to be ignored
- Fixed a bug where column\_prefix would ignore primaryjoin clauses that contained operators
- Added PostgreSQL **q\_substring**, **q\_substring\_regex**, **q\_match**, **q\_notmatch** and **q\_expression** functions which allows for better control over function arguments
- Fixed a bug where rendering of new JOIN version with **SELECT FOR UPDATE** would not produce expected output
- Fixed a bug where SQLITE_INTEGER and SQLITE_FLOAT types were not being declared with **0 precision**
- Fixed a bug where **.first()** and using **LIMIT** with subqueries could cause the subquery to be repeated several times
- Fixed a bug where **q\_delete()** would emit **UPDATE** statements inappropriately
- Fixed regression whereby the `text()` construct with a :class:`_types.TIME` element would fail to produce any SQL
- Fixed issue where **eager\_load()** would not assert correctly if it received only a string format to represent many-to-many relationships
- Fixed a bug where **enum.Enum** values could not be used as values for a scalar subquery in criterion expressions despite their being made to support pickling
- Added more specific error message when attempting to use a stored procedure with PyMySQL which does not have a cursor
- Fixed a bug where processing a batch of values with the Oracle driver used in conjunction with the cx\_Oracle library would raise unexpected errors
- Added the ability to specify PostgreSQL data types with **both zero** as well as **one parameter**
- Added **create\_engine\_pool\_size** and **max\_overflow** options to the connections pool configuration
- Added **use\_custom\_decimal\_type** parameter to allow the use of custom ``Decimal`` types
- Added **UserDefinedTypeComparator.negate()** method support for bitwise complement/negation
- Added support for **symlink** files to dialects that make use of the **os.path.realpath(\_\_file\_\_)** construct
- Added **update\_args** parameter to Sqlite dialect :meth:`.Visitable.replace\_with()` method to support binding parameters correctly in a SQLite query
- Fixed bug where a primaryjoin for a relationship targetting an **aliased()** target with a special **join condition** was not specifying the same join condition in the generated ON clause
- Fixed a bug where using **session.query()** with a **not\_** and scalar subquery combinations would not produce the expected results
- Fixed a bug wherein certain incorrect usages of **".__table__"** within the ORM caused cryptic, hard to diagnose attribute reference issues
- Updated the Error message (raises TypeError) when passing the overall pool object to the **PoolClearer** on the DisconnectedInterface to make it more informative. 

.. version: 1.3.7
.. released: August 14, 2019

- Fixed a bug where nested **mutable default dictionary** objects would create the same default object for all keys in the dictionary
- Fixed memory leak that occurs when using **WeakSet** and **Session** to track session instances
- Fixed another bug with the same root cause as [ticket:3921] where a correlated **IN/EXISTS** subquery in a WHERE clause can cause the bind\_param dictionary to grow without bound
- Fixed **CHECK** constraint reflection to properly include **constraint comments**
- Added **.default\_schema\_name()** method to the :class:`.Engine` object to allow for the default schema to be set based on engine configurations
- When using **sqlite3.reset\_warnings()**, sqlalchemy now includes a call to **sqlite3.get\_warnings()** in order to ensure all warnings are accumulated
- When using **SQLite** with **custom**:class:`.TypeEngine` classes a new parameter called **support\_slicing** is now available to allow for type slicing as normal
- Fixed issue where excluding an element of the mapping in the relationship would fail to disambiguate the **ON clause**
- Addressed issue where infinitely recursive Python dictionary wtih **defaultdict** can cause **AttributeError**
- Enhanced the **ORM** so that when using **with\_for\_update()** with join() on statement, the **FOR UPDATE** clause is rendered always along all other locking clauses
- Extend the **enum.Enum** usage in all applicable places for ORM elements
- Fixed a bug where the type of column created when ALTERing a column from **VARCHAR(MAX)** was not being handled correctly and resulting in an incorrect column type after the ALTER 
- Made **NULL** constants compatible with **Interval** Type.
- Added support for **array comprehension** and **ArraySlice** objects in the postgresql dialect
- Added more fine-grained message system to connection pool parameters \_\_doc\_\_ and :class:`.ConnectionPool`
- Added the ability to using **psycopg2.extras.DictCursor** objects to the **execute** and **stream\_select** methods in the postgresql dialect 
- Added **ScalarSelect** object which allows for an easy way of specifying subqueries requiring only a scalar value where raw SQL is not needed
- Added a new parameter **batch=False** to the **.execute()** method of the connection pool, to enable the output of parameters from **PostgreSQL** similar to those found in **PSYCOPG2** and its use of **execute\_many()**
- Added support for **JSONB** scalar type and JSONB-based **array()** type, including new objects **JSON** and **JSONB** to ease usage
- Added **automap\_batch** option to help automate the use of :func:`.automap`, such that it can be used to map tables that reference one another without requiring a user-defined iteration sequence
- Fixed a bug where using **PostgreSQL {\_\_db\_link\_\_}** feature and dictionaries as the bind parameters would result in them being passed to the database as string
- Added support for arbitrary order-by clauses in subqueries, such as those generated by an applied FROM clause or to reflect the best relationship join between the tables being queried
- Fixed that the MySQL multimaster replication SQL would fail on unhandled VARCHAR type with encoding such as utf8mb4
- Corrected a deep relational dependency issue related to the way the ORM handles the "select" concept, so that queries involving deep inheritance relationships no longer produce unnecessary joins and subqueries
- Added Postgres **tsrange**, **tstzrange**, **json** and **jsonb** types to :class:`.auto\_generate\_type()` method

.. version: 1.3.6
.. released: July 21, 2019

- Fix a bug that causes the MySQL dialect wouldn't interpret None correctly when converting a String type to TEXT
- Fix a bug where the xmltodict package receives Unicode data and does not convert to a string, causing compatibility issues with the ORM when using **json.loads**
- Bug fixed where SQLite types (NUMERIC, INTEGER, REAL, and TEXT) do not have a :func:`.create()` method
- Fix a bug where the ORM lazy load of related rows would cause the query to go into an infinite loop
- Fix a bug where the :meth:`.default()` feature of the :class:`.Column` object is used, the **server\_default** parameter would not be able to access the **metadata** argument due to the **Column** object being at initialization time in **metadata**
- Fix an error in the **sqlalchemy.dialects.mysql.types.JSON** implementation where **BinaryString** keys to JSON were not being interpreted.
- Starting a transaction on postgresql, for example, could result in a warning if the **XLOG** of the PostgreSQL server was too large
- The SQLAlchemy ORM will now silently omit any non-primary-items from insert and update when using the bulk operations if they are not defined to be included in the table
- Fixed a regression in the ORM whereby inheritance-based queries where features from subclass tables were included in the WHERE clause would have inconsistent results across backends
- Added support for **Protocol Version 14** in the :class:`.postgresql.psycopg2` dialect
- Fixed Postgresql dialect support for **ARRAY** when elements are of varying length
- Enhancements to the :func:`.create_engine` function allow for more granular control of **connection pool** limits, such as specifying a maximum number of connections for the pool itself
- Fix an issue with the underlying Unicode codepoint programming model that was causing alphanumeric characters to be mistakenly included when creating SQL queries with explicit bounds
- Fixed an issue where the ``__tablename__`` attribute of ORM classes was not being created properly if any of the characters used in the name were illegal in a Windows filename
- Added **Enum.Enum** support to SQLExpressions.
- Fixed an issue with regexes, such that advanced unicode-compatible flags will raise a compile-time error message if they are being used on earlier versions of Python that don't support them
- Fixed an issue introduced in version 1.3.4 whereby boolean expressions with nested functions like **Case** or **cast()** could raise an inadvertent **CursorClosedException**
- Fixed an incompatibility between **execute\_values()** and **ON DUPLICATE KEY UPDATE** functionality on the MySQL/MariaDB dialects, caused by the keywords being treated as SQL fragments during the parsing step
- Fixed a bug where lazy loaded many-to-one bi-directional relationships would add a SQL statement that is not needed, if subordinate objects were loaded by eager load due to the path between the related attribute of the parent and the subordinate object going through their mutual **mapper** instance
- Fixed a regression introduced in version 1.3.4 such that the "autoincrement" attribute of a :class:`.Column` will only be disabled for columns that don't support autoincrement explicitly
- Fixed a bug where the **.cache\_key()** method of :class:`_orm.Query` objects did not clear the ORM weak-referenced cache when called
-  Fixed regression in the ORM around the ability for the component registry storing **primary** objects related to ancestor/descendant management to serialize/deserialize itself for some cases such that it stores the correct state related to **eager** operations
- Added a **sqlalchemy.future** sub-package which includes various performance-improving and time-saving features for applications. It provides several improvements to the ORM and SQL expression language, and is intended to be a preview of what's in store for the SQLAlchemy 2.0 release due in 2020.  

.. version: 1.3.5
.. released: May 27, 2019

- Fixed an issue with the **MySQL** dialect where a query without results for a certain column would cause all following queries to raise a warning
- Fixed **CHECK** constraint reflection to include **constraint comments**
- Fixed an issue whereby some cases of deeply nested **FILTER** clauses may cause an assertion error.
- Fixed an issue where in the **ORM** setting an unloaded scalar attribute to a non scalar value would raise an error when a cycled backref has been created
- Fixed an issue where arrays with zero or one dimension(s) where being wrapped in parens
-Fixed an issue where the :func:`_model_identity.util.default\_column\_name` function could be passed a tuple as an argument, which would cause a JSON Encoding failure
- Fixed an issue where reflection of **postgres hstore** returns a **NoneType** value as the second element of a **(key,) tuple**
- Fixed an issue where ``postgres.contrib.postgis`` didn't allow **Geography** columns to be passed to its constructor
- Fixed a bug where the timezone-sensitive :class:`.DateTime` would leak a connection on transaction flip when the server-side default timezone was different from the client´s timezone
- Fixed **TypeDecorator** issues where types would not compile when used with **postgresql.array** or **postgresql.Composite**
- Fixed an issue where Python 3's **OrderedDict** returned values list for a dictionary sometimes wouldn't hash correctly
- Fixed an issue where **pooling=True** in :meth:`_engine.Engine.connect` would not always return **Connection**
- Fixed an issue where the **"rowcount does not match number of statements executed"** warning would only occur for delete statements that have multiple subqueries
- Fixed an issue where **BOOLEAN** expressions with nested functions like **CASE** or **cast()** could raise **AttributeError** in Python 3.x
- Fixed an issue where discrepancies in column names between server and client will be not correctly handled by sqlite's **DROP TABLE**
- Fixed a bug where the **SQLAlchemy** ORM would break when queries were run that contained both a UNION with a subquery that defined a column label and a column label that existed in the query itself
- Fixed an issue where **Query.join** with a **NULL** column would raise an exception


.. 版本: 1.3.3
    :发布时间: 2019年4月15日

    .. change::
       :tags: bug, postgresql
       :tickets: 4601

       修复了从1.3.2版本引入的回归错误，该错误由于：ticket:`4562`导致的，在其中仅包含查询字符串而没有主机名的URL上，例如为了指定带有连接信息的服务文件，将不再正确传递给psycopg2。 :ticket:`4562`的更改已经调整为进一步适应psycopg2的确切要求，即如果有任何连接参数，那么不再需要“dsn”参数，因此在这种情况下仅传递查询字符串参数。

    .. change::
       :tags: bug, orm
       :tickets: 4647

       现在发出有关将易失性对象与:class:`.Session`中的:meth:`.Session.merge`合并时的警告，当对象已经是Session中的易失性对象时，例如。 这警告适用于对象通常会被重复插入的情况。

    .. change::
        :tags: bug, orm
        :tickets: 4676

        修复了新关系m2o比较逻辑中出现的错误回归，该逻辑最初是在：ref:`change_4359`中引入的，当将其与映射实例中处于未调用状态的已持久化为NULL的属性进行比较时，由于该属性没有明确的默认值，因此需要在访问持久设置时将其默认为NULL。

    .. change::
        :tags: bug, sql
        :tickets: 4569

        将:class:`.GenericFunction`命名空间迁移，以便以不区分大小写的方式查找函数名称，因为SQL函数不会因区分大小写之间的差异而产生冲突，也不会在用户定义的函数或存储过程中发生这种情况。 :class:`.GenericFunction`声明的函数现在使用不区分大小写的方案进行查找，但是支持废弃的情况，该情况允许两个或更多具有相同名称的：class：`.GenericFunction`对象具有不同大小写，对于该特定名称将导致进行区分大小写的查找，同时在函数注册时发出警告。感谢Adrien Berchet对这个复杂功能的大量工作

    .. change::
       :tags: bug, orm
       :tickets: 4584

       修复了新的“模棱两可的FROM”查询逻辑中的1.3回归，在其中，使用:meth:`_Query.select_from`将实体明确放置在FROM子句中且同时使用:meth:`_Query.join` 还会在将该实体用于其他联接时导致“模棱两可的FROM”错误，因为该实体会在:class:`_query.Query`的“from”列表中出现两次。 通过在 :class:`_query.Query`渲染“SELECT”语句时对独立实体进行折叠来解决此歧义，以与其在相同的联接中的部分相同，最终会发生什么

    .. change::
       :tags: bug, pool
       :tickets: 4585

       修复了行为回归，因为弃用了:class:`_pool.Pool`的"use_threadlocal"标志，从而使:no:class:`.SingletonThreadPool`不再使用此选项，这将导致“回滚”逻辑在上下文中使用同一:class:`_engine.Engine`的情况下多次使用引擎连接或隐式执行，从而取消事务。虽然这不是建议使用引擎和连接的方式，但这仍然是一种令人困惑的行为更改，因为在使用:no:class:`.SingletonThreadPool`时，事务应保持打开状态，而不管在同一线程中使用相同的引擎做了什么。但是，“use_threadlocal”标志仍然已弃用，但:no:class:`.SingletonThreadPool`现在实现了其自己的版本。

    .. change::
        :tags: bug, ext
        :tickets: 4603

        修复了在:class:`.MutableList`上使用``copy.copy()``或``copy.deepcopy()``时会导致列表中的项目被重复创建的错误，因为Python pickle和copy在涉及列表时使用``__getstate__()``和``__setstate__()``的一致性不一致。为了解决，必须向:no:class:`.MutableList`添加一个``__reduce_ex__``方法。为了保持基于``__getstate__()``的现有pickle的向后兼容性，``__setstate__()``方法仍然存在；测试套件断言基于旧版本类制作的pickle仍然可以由pickle模块反序列化。

    .. change::
        :tags: bug, mssql
        :tickets: 4587

        修正了SQL Server方言中的一个问题，其中如果订单BY表达式中存在绑定参数，而该参数最终不会呈现在SQL Server版本的语句中，则参数仍然将成为执行参数，导致DBAPI级别的错误。 。Pull request由Matt Lewellyn提供。

    .. changelog::
    :version: 1.3.2
    :released: April 2, 2019

    .. change::
       :tags: bug, documentation, sql
       :tickets: 4580

       由于:ref:`change_3981`，我们不再需要依赖直接子类化特定dialect类型的配方，:class:`.TypeDecorator`现在可以处理所有情况。此外，上述更改使得直接子类化基本:mod:`SQLAlchemy`类型的类无法按预期工作的可能性略微减小，这可能会使人产生误导。文档已更新为使用:class:`.TypeDecorator`进行这些示例，包括PostgreSQL的"ArrayOfEnum"示例数据类型，并直接支持"直接子类化类型"的子类的支持已被删除。

    .. change::
       :tags: bug, postgresql
       :tickets: 4550

       修改了:paramref:`.Select.with_for_update.of`参数，使得如果传递了连接或其他组合可选择，则会从中过滤出单独的.:class: '_schema.Table'对象，从而允许在该参数中传递join（）对象，就像在ORM中使用联接表继承一样正常进行处理。Pull request由Raymond Lu提供。


    .. change::
        :tags: feature, postgresql
        :tickets: 4562

        为psycopg2方言添加了无参数连接URL的支持，这意味着可以将URL作为``"postgresql+psycopg2://"``传递给:func:`_sa.create_engine`，而不需要其他参数来指示传递给libpq的空DSN ，表示连接到“localhost”，并且未提供用户名，密码或数据库。Pull request由Julian Mehnle提供。

    .. change::
       :tags: bug, orm, ext
       :tickets: 4574, 4573

       在协会代理与Python描述符（例如``@property``）一起使用时，恢复了简单Python描述符的实例级支持，前提是代理对象根本不在ORM范围内，在这种情况下，将其分类为“模糊”的，但直接进行代理。对于类级访问，基本类级" __get __()"现在直接返回:no:class:`.AmbiguousAssociationProxyInstance`，而不是引发其异常，这是最接近以前返回:class:`.AssociationProxy`本身的行为。还改进了这些对象的字符串表示方式，以更详细地描述当前状态。

    .. change::
       :tags: bug, orm
       :tickets: 4537

       修复了使用with_polymorphic或其他别名结构时，不会正确适应别名目标作为:meth:`_expression.Select.correlate_except`子查询的目标时，如果该别名目标在:class:`_expression.ColumnProperty`中内用于在声明中的约束 or other column-oriented scenario时，使用不完全声明的列或延迟属性将会更加友好，尽管在开放式表达式中仍无法正常工作；如果收到``TypeError```，请调用:attr:`.ColumnProperty.expression`属性。

    .. change::
       :tags: bug, orm
       :tickets: 4566

       修复了一个新的错误消息，该错误消息应在未使用:meth:`.PropComparator.of_type`将关系选项与:class:`.AliasedClass `链接时引发，而不会引发``AttributeError``。请注意，在1.3中，已不再有效地使用从映射关系到:class:`.AliasedClass`的选项路径，而不使用:meth:`.PropComparator.of_type`。

.. changelog::
    :version: 1.3.1
    :released: March 9, 2019

    .. change::
       :tags: bug, mssql
       :tickets: 4525

       修复了SQL Server反射中来自1.3.2版本的回归，由于删除了 :class:`.Float` 数据类型中的开放式``**kw``，导致此类型的反射失败，因为会传递“ scale”参数。

    .. change::
       :tags: bug, ext
       :tickets: 2642

       当使用集或字典的关联代理时，实现更全面的分配操作（例如“批量替换”）时，修复了重复的代理对象会导致要删除成员和其父对象之间的关联关系的反向引用。在唯一约束的情况下，这会导致刷新失败。

       .. seealso::

          :ref:`change_2642`

    .. change::
        :tags: bug, postgresql
        :tickets: 4550

        修正了在:paramref:`_expression.select`或:class:`_query.Query`对象的组件中直接传递字符串的行为，其中这些字符串将自动转换为:func:`_expression.text`构造；现在将发出ArgumentError或在:func:`_query.Query.order_by()` /:meth:`_query.Query.group_by()`中的情况下发出CompileError。自版本1.0以来一直发出警告的存在继续引发关于这种行为的潜在误用的担忧。

       注意，公共CVE已针对:func:`_query.Query.order_by()`/ :meth:`_query.Query.group_by()`发布，并由本次提交解决：CVE-2019-7164 CVE-2019-7548

       .. seealso::

        :ref:`change_4481`

    .. change::
       :tags: bug, orm
       :tickets: 3777

       实现了对与指定合成词义相同的同义词属性的属性历史记录的访问,现在，尝试通过同义词访问属性历史记录将引发一个``AttributeError``。

    .. change::
       :tags: feature, engine
       :tickets: 3689

       添加了公共访问器.:meth:`.QueuePool.timeout`，返回配置的超时.:class:`.QueuePool`对象。Pull request由Irina Delamare提供。

    .. change::
       :tags: feature, sql
       :tickets: 4386

       在:class:`.AnsiFunction`类上进行了修改，它是基础常见SQL函数的基础，例如“``CURRENT_TIMESTAMP``”，以接受位置参数，就像常规的临时函数一样。这适用于许多特定后端支持这些函数的函数接受诸如“小数秒”精度之类的参数。如果使用参数创建函数，则渲染括号和参数。如果没有参数，则编译器会生成无括号形式。:class:`.Unicode` 和:class:`.UnicodeText`类型的数据默认情况下会被当作nchar或者nvarchar类型处理，当使用 :func:`_sa.create_engine`方法时可以使用参数``use_nchar_for_unicode=True``来指定，包括CREATE TABLE 以及``setinputsizes()``用于绑定参数。在Python 2下，Char，Varchar和Clob类型结果行的自动Unicode转换已经添加，与Python 3下的cx_Oracle行为相匹配。为了缓解Python 2下的性能影响，当使用C扩展时，SQLAlchemy使用高性能（当C扩展被构建时）的本地Unicode处理程序。 

    .. seealso::

            :ref:`change_4242`

    .. change::
        :tags: bug, orm
        :tickets: 3844

        修复了关于passive_deletes="all"的问题：即使对象从其父集合中删除，其外键属性也会保持其值不变。之前，unit of work会将其设置为NULL，即使passive_deletes指示不应修改它。 

        .. seealso::

            :ref:`change_3844`

    .. change::
        :tags: bug, ext
        :tickets: 4268

        将协会代理集合仅保留对父对象的弱引用的长期行为还原为，代理现在将对父对象保持强引用，只要代理集合本身也在内存中，从而消除了“陈旧的协会代理”错误。此更改正在试验性地进行，以查看是否存在导致副作用的用例。

        .. seealso::

            :ref:`change_4268`


    .. change::
        :tags: bug, sql
        :tickets: 4302

        添加了“like” 类型的比较运算符，包括：``.ColumnOperators.startswith``， ``.ColumnOperators.endswith``，``.ColumnOperators.ilike``，``.ColumnOperators.notilike``等等，使得所有这些运算符都可以成为ORM“primaryjoin”条件的基础。


    .. change::
        :tags: feature, sqlite
        :tickets: 3850

        通过:new:`_sqlite.JSON`向 :class:`_types.JSON`添加了对SQLite 的json功能的支持，使用的类型名称为``JSON``，遵循SQLite自己的文档中提供的示例。感谢Ilja Everilä的贡献。

        .. seealso::

            :ref:`change_3850`

    .. change::
       :tags: feature, engine

       向 :class:`.QueuePool` 添加了“lifo”模式，通常通过将标志:paramref:`_sa.create_engine.pool_use_lifo`设置为True启用。“lifo”模式意味着刚检查过的相同连接将首先再次被检查出来，从而允许在池部分利用期间从服务器端清除过多的连接。感谢 Taem Park的贡献。

       .. seealso::

          :ref:`change_pr467`

    .. change::
       :tags: bug, orm
       :tickets: 4359

       改进了与关系绑定的多对一对象表达式，使其检索与相关对象上的列值时可以在对象从其父 :class:`.Session` 分离时具有弹性，即使该属性已过期。使用 :class:`.InstanceState`内的新功能来存储在过期之前某个列属性的最后已知值的记忆化，以便可以在对象被分离和过期的同时进行表达式计算。使用现代属性状态来生成根据需要的更具体的消息的错误条件也得到改进。 

       .. seealso::

            :ref:`change_4359`

    .. change::
        :tags: feature, mysql
        :tickets: 4219

        支持MySQL“WITH PARSER”句法的CREATE FULLTEXT INDEX,使用 ``mysql_with_parser`` 关键字参数。同时支持反射，这可以适应MySQL报告此选项的特殊注释格式。此外，“FULLTEXT”和“SPATIAL”索引前缀现在也在``mysql_prefix``索引选项中反映出来。


    .. change::
        :tags: bug, orm, mysql, postgresql
        :tickets: 4246

        ORM现在会在与joined eager loading同时渲染的子查询中重复使用“FOR UPDATE”子句，因为观察到MySQL不会锁定子查询中的行。这意味着查询将会渲染两个FOR UPDATE子句;if后端（如Oracle）上的FOR UPDATE子句将被默默忽略，因为它们不必要。此外，在主要与PostgreSQL一起使用的“OF”子句的情况下，仅在内部子查询上渲染FOR UPDATE，以便可将可选择性定位到SELECT语句中的表格。

        .. seealso::

            :ref:`change_4246`

    .. change::
        :tags: feature, mssql
        :tickets: 4158

        在SQL Server pyodbc方言中添加了``fast_executemany=True``参数，这使得使用Microsoft ODBC驱动程序时可以使用pyodbc的新性能功能。 

        .. seealso::

            :ref:`change_4158`

    .. change::
        :tags: bug, ext
        :tickets: 4308

        修复了有关去协会标量对象的多个问题。现在 ``del``的工作正常，另外增加了一个新标志 :paramref:`.AssociationProxy.cascade_scalar_deletes`，如果将其设置为True，则表明将标量属性设置为 ``None`` 或通过 ``del`` 删除也将将源关联设置为 ``None``。

        .. seealso::

            :ref:`change_4308`


    .. change::
        :tags: feature, ext
        :tickets: 4318

        添加了新功能 :meth:`.BakedQuery.to_query`，它允许在不需要明确引用:class:`.Session`的情况下，在另一个:class:`.BakedQuery` 内使用一个 :class:`.BakedQuery` 作为子查询。


    .. change::
       :tags: feature, sqlite
       :tickets: 4360

       将SQLite ``ON CONFLICT``子句作为DDL级别理解实现了。例如，在主键、唯一性和CHECK约束上，以及在指定为 ``_schema.Column`` 的上的情况下，满足内联主键和NOT NULL。感谢Denis Kataev的贡献。

       .. seealso::

          :ref:`change_4360`

    .. change::
       :tags: feature, postgresql
       :tickets: 4237

       增加对PostgreSQL分区表反射的基本支持，例如：返回表信息的反应查询中添加relkind ='p'。

       .. seealso::

            :ref:`change_4237`

    .. change::
       :tags: feature, ext
       :tickets: 4351

       当目标属性为普通列时，:class:`.AssociationProxy`现在具有标准列比较操作，例如 :meth:`.ColumnOperators.like`和 :meth:`.ColumnOperators.startswith`。仍使用JOIN到目标表的EXISTS表达式，但是列表达式现在用于EXISTS的WHERE条件。请注意，在列基础属性上使用``.contains()``方法时，此更改将更改 :meth:`.ColumnOperators.contains`的行为。

       .. seealso::

          :ref:`change_4351`


    .. change::
        :tags: feature, orm

        向 :meth:`.Session.bulk_save_objects` 方法添加了新标志 :paramref:`.Session.bulk_save_objects.preserve_order`，默认为True。当设置为False时，给定的映射将被分组成Per每个对象类型的插入和更新，以允许更大程度地批量地将常见操作放在一起。感谢Alessandro Cucci的贡献。

    .. change::
        :tags: bug, orm
        :tickets: 4365

        重构了 :meth:`_query.Query.join`以进一步澄清组成连接的各个部件。这个重构添加了新的功能 :meth:`_query.Query.join`，在FROM列表中的元素超过一个或查询针对多个实体时将确定最适合的“左”侧连接。如果多个FROM/entity匹配，将引发一个错误，要求指定ON子句以解决歧义问题。特别地，这针对我们在 :ticket:`4363` 中看到的回归，但也是通用的。现在，:meth:`_query.Query.join`中的代码路径更易于跟踪，并且错误案例在操作的早期决定得更加具体。

        .. seealso::

            :ref:`change_4365`

    .. change::
        :tags: bug, sql
        :tickets: 3981

        修复了 :meth:`.TypeEngine.bind_expression`和 :meth:`.TypeEngine.column_expression`方法的问题，其中这些方法如果目标类型是:class:`.Variant`的一部分或者是 :class:`.TypeDecorator`的其他目标类型，则不起作用。此外，SQL编译器现在在呈现这些方法时调用了方言级别实现，以便dialect现在可以为内置类型提供SQL级别处理。

        .. seealso::

            :ref:`change_3981`


    .. change::
        :tags: bug, orm
        :tickets: 4304

        修复了ORM中的一个长期存在的问题，即当如 :meth:`_query.Query.exists` ，:meth:`_query.Query.as_scalar`等函数在 :attr:`_query.Query.statement`属性中生成标量子查询时，在使用需要实体适应的新 :class:`_query.Query`，例如 turn into a union 或者是 from_self()等时，标量子查询将不会正确地适应。该更改从 :attr:`_query.Query.statement`访问器中删除了“no adaptation”注解。


    .. change::
        :tags: bug, orm, declarative
        :tickets: 4133

        当添加或删除其他属性后，declarative无法更新 :class:`_orm.Mapper`，以至于已经调用并缓存了mapper属性集合后，它们未更新。此外，如果正在映射的类从中完全映射的属性（例如列，关系等）中删除，则现在会引发 ``NotImplementedError``，因为如果删除属性，则映射程序将无法正常工作。

    .. change::
       :tags: bug, mssql
       :tickets: 4362

       废弃了在SQL Server中使用 :class:`.Sequence`来影响IDENTITY值的“start”和“increment”的用法，改用新的参数``mssql_identity_start`` 和``mssql_identity_increment`` 直接设置这些参数的值， :class:`.Sequence`将在未来的版本中用于生成SQL Server的真实 ``CREATE SEQUENCE`` DDL。

       .. seealso::

            :ref:`change_4362`

    .. change::
        :tags: feature, mysql

        支持MySQL上的ON DUPLICATE KEY UPDATE语句中的参数被排序的方式，因为MySQL UPDATE语句中的参数排序顺序很重要，与 :ref:`tutorial_parameter_ordered_updates`中描述的方式类似。感谢Maxim Bublis的贡献。

        .. seealso::

            :ref:`change_mysql_ondupordering`

    .. change::
       :tags: feature, sql
       :tickets: 4144

       添加新功能 :class:`.Sequence`，用于在序列包含“sequence nextvalue”表达式的情况下，以生成有意义的字符串表达式（”<next sequence value: my_sequence>”）而不是引发编译错误时，将其字符串化。


    .. change::
        :tags: bug, orm
        :tickets: 4232

        当在Python 3下检索不可排序的主键值，如没有 __lt__ 方法的``Enum``时，现在在ORM flush期间会重新引发有用的异常，例如 ``lazy="raise"`` 或者分离的session引发错误;正常情况下Python 3下会引发TypeError。在ORM flush过程中，使用Python按主键排序持久化对象，因此值必须是可排序的。


    .. change::
       :tags: orm, bug
       :tickets: 3604

       删除了 :class:`.MappedCollection`类中使用的集合转换程序。这个转换程序仅用于断言传入的字典键与相应对象的键匹配，并且仅在批量设置操作期间使用。转换程序可能会干扰自定义验证器或 :meth:`.AttributeEvents.bulk_replace`列表，后者需要进一步转换传入的值。该转换器在传入键不匹配值时引发 TypeError，接受则将使用其值生成的键，而不是显式在字典中存在的键。总体而言，@converter被 :ticket:`3896`的 :meth:`.AttributeEvents.bulk_replace`事件处理器替代。

    .. change::
       :tags: feature, sql
       :tickets: 3989

       添加了新的命名约定tokens ``column_0N_name``， ``column_0_N_name``，等等，它将为序列中特定约束引用的所有列的名称/键/标签提供名称。为此，SQL编译器的自动截断特性现在也适用于约束名称，这将为此约束在不超过后端字符限制的情况下创建一个缩短的确定性生成的名称。还修复了两个问题。其中一个是即使这个标记被记录在案，也不可用于 ``column_0_key``，另一个问题是如果这两个值不同，则 ``referred_column_0_name`` token 会意外渲染 ``.key`` 而不是 ``.name``的列。


    .. change::
        :tags: feature, ext
        :tickets: 4196

        向横向分片扩展中的 :class:`.ShardedQuery` 添加了对 :meth:`_query.Query.update` 和 :meth:`_query.Query.delete` 的批量支持。这还为 :meth:`_query.Query._execute_crud`中的批量更新/删除方法添加了另一个扩展挂钩。

        .. seealso::

            :ref:`change_4196`

    .. change::
        :tags: feature, sql
        :tickets: 4271

        对于扩展的IN绑定参数功能，如果给定的列表为空，则会生成特殊的“空集”表达式，该表达式特定于不同的后端，从而允许IN表达式完全动态，包括空的IN表达式。

        .. seealso::

            :ref:`change_4271`



    .. change::
        :tags: feature, mysql

        连接池的“预调用”功能现在在MySQLClient、PyMySQL和mysql-connector-python的情况下使用DBAPI连接的``ping()``方法。感谢Maxim Bublis的贡献。

        .. seealso::

            :ref:`change_mysql_ping`

    .. change::
       :tags: feature, orm
       :tickets: 4340

       “selectin”加载策略现在在简单的一对多加载的情况下省略JOIN，其中它仅从相关表中加载，依赖于外键列以匹配父表中的主键。这种优化可以通过将 :paramref:`_orm.relationship.omit_join`标志设置为False来禁用。非常感谢Jayson Reis的努力。

        .. seealso::

            :ref:`change_4340`

    .. change::
       :tags: bug, orm
       :tickets: 4353

       现在在获取多对一旧值时，跳过由于 ``lazy="raise"`` 或者分离的session引发非异常情况，而是依照原样返回异常。

       .. seealso::

        :ref:`change_4353`

    .. change::
        :tags: feature, sql

        Python内置的 ``dir()`` 现在支持SQLAlchemy“properties”对象，例如 Core 列集合（例如， ``.c``），``mapper.attrs``等。它允许iPython的自动完成正常工作。

    .. change::
       :tags: feature, orm
       :tickets: 4257

       向 :class:`.InstanceState`类添加了信息字典，通过调用 :func:`_sa.inspect`返回。

       .. seealso::

            :ref:`change_4257`

    .. change::
        :tags: feature, sql
        :tickets: 3831

        添加了新的特性 :meth:`.FunctionElement.as_comparison`，它允许一个SQL函数充当二进制比较操作，可以在ORM内使用。 

        .. seealso::

            :ref:`change_3831`

    .. change::
       :tags: bug, orm
       :tickets: 4354

       ORM中的一个长期问题“__delete__”方法对于多对一关系实际上是无效的，例如，对于操作如 ``del a.b``。现在已经实现了该方法，等同于将属性设置为 ``None`` 。

       .. seealso::

            :ref:`change_4354`