=============
1.0 更新日志
=============

.. changelog_imports::

    .. include:: changelog_09.rst
        :start-line: 5


    .. include:: changelog_08.rst
        :start-line: 5


    .. include:: changelog_07.rst
        :start-line: 5



.. changelog::
    :version: 1.0.19
    :released: 2017 年 8 月 3 日

    .. change::
        :tags: bug, oracle, performance, py2k
        :tickets: 4035
        :versions: 1.0.19, 1.1.13, 1.2.0b3

        修复了“3937”问题的性能回退，其中从版本 5.3 起 cx_Oracle 从其名称空间中删除了“.UNICODE”符号，这被解释为无条件开启 SQLAlchemy 中的 cx_Oracle 的“ WITH_UNICODE”模式，该模式会在 SQLAlchemy 中调用函数，无条件地将所有字符串转换为 Unicode，并导致性能影响。实际上，根据 cx_Oracle 的作者，“WITH_UNICODE”模式已于 5.1 版本中被完全删除，因此不再需要昂贵的 Unicode 转换函数，在检测到 Python 2 中的 cx_Oracle 5.1 或更高版本时会禁用这些函数。 “3937”问题中删除的针对“WITH_UNICODE”模式的警告也已恢复。 

.. changelog::
    :version: 1.0.18
    :released: 2017 年 7 月 24 日

    .. change::
        :tags: bug, tests, py3k
        :tickets: 4034
        :versions: 1.0.18, 1.1.12, 1.2.0b2

        为测试固件中的修复问题与 Python 3.6.2 发生的更改不兼容的问题。

    .. change:: 3937
        :tags: bug, oracle
        :tickets: 3937
        :versions: 1.1.7

        修复了 cx_Oracle 在版本 5.3 中硬编码此标志的解决方法，此标志是小写的“with_unicode”模式，因为一个使用此模式的内部方法未使用正确的签名。

.. changelog::
    :version: 1.0.17
    :released: 2017 年 1 月 17 日

     .. change::
        :tags: bug, py3k
        :tickets: 3886
        :versions: 1.1.5

        解决与 Python 3.6 DeprecationWarnings 相关的转义字符串的问题，而未使用 'r' 修饰符，并为 Python 3.6 添加了测试覆盖范围。

    .. change::
        :tags: bug, orm
        :tickets: 3884
        :versions: 1.1.5

        修复了在使用多个实例与多个实体进行连接的联接贪婪加载时会抛出"'NoneType' object has no attribute 'isa'。问题，此问题是由于修复“3611”的问题引起的。

.. changelog::
    :version: 1.0.16
    :released: 2016 年 11 月 15 日

    .. change::
        :tags: bug, orm
        :tickets: 3849
        :versions: 1.1.4

        在  :meth:`.Session.bulk_update_mappings`  中修复了错误的问题，其中备用命名的主键属性将无法正确地跟踪到 UPDATE 语句中。

    .. change::
        :tags: bug, mssql
        :tickets: 3810
        :versions: 1.1.0

        更改了用于获取“默认架构名称”的查询，从查询数据库原理表的一个查询到使用 "schema_name()" 函数，因为已经有关于 Azure Data Warehouse 版本上的原理不可用的问题。希望这将最终适用于所有 SQL Server 版本和验证样式。

    .. change::
        :tags: bug, mssql
        :tickets: 3814
        :versions: 1.1.0

        更新了 pyodbc 对 SQL Server 的服务器版本信息方案，改用 SQL Server
        SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者继续不可靠，
        特别是在 FreeTDS 中。

    .. change::
        :tags: bug, orm
        :tickets: 3800
        :versions: 1.1.0

        修复了当联接贪婪加载用于多态地加载的映射器时，其中 polymorphic_on 设置为未映射的表达式，例如 CASE 表达式时，joined eager 加载会失败的 bug。

    .. change::
        :tags: bug, orm
        :tickets: 3798
        :versions: 1.1.0

        修复了当通过  :meth:`.Session.bind_mapper` 、  :meth:` .Session.bind_table`   或构造函数发送给 Session 的无效绑定时，会引发的 ArgumentError 失败的问题。

    .. change::
        :tags: bug, mssql
        :tickets: 3791
        :versions: 1.1.0

        将错误代码 20017 “服务器的意外 EOF” 添加到导致连接池重置的断开异常列表中。这个错误代码由 Ken Robbins 贡献。

    .. change::
        :tags: bug, orm.declarative
        :tickets: 3797
        :versions: 1.1.0

        修复了设置为 joined-table 订阅类的单个表的外键可能会破坏映射表的外键集合，从而干扰关系初始化的问题。

    .. change::
        :tags: bug, orm
        :tickets: 3781
        :versions: 1.1.4

        修复了  :meth:`.Session.bulk_save`  中的错误，其中 UPDATE 与实现版本 id 计数器的映射一起使用将无法正常使用。

    .. 3778

    .. change::
        :tags: bug, orm
        :tickets: 3778
        :versions: 1.1.4

        修复了当添加属性、映射器属性或其他 ORM 构造子后，第一次调用这些访问器时，将失败的  :attr:`_orm.Mapper.attrs` 、  :attr:` _orm.Mapper.all_orm_descriptors`   和其他衍生属性的 bug。

    .. change:: 3762
        :tags: bug, mssql
        :tickets: 3762
        :versions: 1.1.4

        修复了 pyodbc 方言中的 bug（也适用于大部分不起作用的 adodbapi 方言），即当在密码或用户名字段中存在分号时，会将这个分号解释为另一个标记的分隔符；当分号存在时，这些值现在将被引用。

.. changelog::
    :version: 1.0.15
    :released: 2016 年 9 月 1 日

    .. change::
        :tags: bug, mysql
        :tickets: 3787
        :versions: 1.1.0

        添加对在 URL 查询字符串中解析 MySQL/Connector 布尔值和整数参数的支持：connection_timeout、connect_timeout、pool_size、get_warnings、raise_on_warnings、raw、consume_results、ssl_verify_cert、force_ipv6、pool_reset_session、compress、allow_local_infile、use_pure。

    .. change::
        :tags: bug, orm
        :tickets: 3773, 3774
        :versions: 1.1.0

        修复了在子查询贪婪加载中，将“of_type()”对象的子查询负载链接到第二个常规映射类或更长的多个“of_type()”属性的链时，子查询载入的问题。多个 “of_type()” 选项现在将被考虑。

    .. change::
        :tags: bug, sql
        :tickets: 3755
        :versions: 1.1.0

        修复了   :class:`_schema.Table`  中的错误，其中内部方法“ _reset_exported()”会破坏对象的状态。此方法旨在从可选择的对象中调用，并在某些情况下由 ORM 调用。错误的映射配置将导致 ORM 在   :class:` _schema.Table`  对象上调用此方法。

    .. change::
        :tags: bug, ext
        :tickets: 3743
        :versions: 1.1.0b3

        修复了“sqlalchemy.ext.baked”中的错误，其中由于多个子查询加载程序涉及变量作用域问题，因此一个字典状贪婪加载器查询无法解除反演。

.. changelog::
    :version: 1.0.14
    :released: 2016 年 7 月 6 日

    .. change::
        :tags: bug, postgresql
        :tickets: 3739
        :versions: 1.1.0b3

        调整了用于解析 MySQL 视图的正则表达式，因此不再假设“ALGORITHM”关键字存在于反射视图源中，因为一些用户报告称在某些 Amazon RDS 环境中不存在此关键字。

    .. change::
        :tags: bug, oracle
        :tickets: 3741
        :versions: 1.1.0b3

        修复了  :paramref:`.Select.with_for_update.of`  中参考表存在模式限定符时将失败的问题，如 pg 需要省略模式名称。可以使用特殊符号  :attr:` _schema.BLANK_SCHEMA`  作为  :paramref:`_schema.Table.schema`  和  :paramref:` .Sequence.schema`  的可用值，表示即使指定了  :paramref:`_schema.MetaData.schema`  也应该强制模式名称为 None。

    .. change::
        :tags: bug, sql
        :tickets: 3643

        修复了  :meth:`_schema.Table.tometadata`  中的错误，这种错误从 0.9 系列开始出现，而通过反序列化的   :class:` _schema.Table`  添加列会失败。此时，不会正确建立“ c”集合中的   :class:`_schema.Column` ，导致在 ORM 配置等区域中出现问题。它可能会影响到使用场景，如 extend_existing 等等。

    .. change::
        :tags: bug, py3k
        :tickets: 3660

        解决了将单个字节对象转换为单个字符列表的问题，在“to_list”转换中，这将在主键是字节对象时影响到一些使用  :meth:`_query.Query.get`  方法的情形。

    .. change::
        :tags: bug, orm
        :tickets: 3658

        解决了 1.0 系列中的 ORM 加载中发生的问题，即预期的列缺失抛出的异常将不正确地成为 NoneType 错误，而不是预期的   :class:`.NoSuchColumnError` 。

    .. change::
        :tags: bug, mssql, oracle
        :tickets: 3657

        修复了 1.0 系列中出现的回归，该回归导致 Oracle 和 SQL Server 方言在这些方言将 SELECT 包装在子查询中以提供 LIMIT/OFFSET 行为的情况下不正确地考虑结果集列，而原始 SELECT 语句多次引用相同的列，例如一个列和一个该列的标签。这个 issue 和  :ticket:`3658`  有关，当错误发生时，也会导致 NoneType 错误，而不是报告无法定位列。

.. changelog::
    :version: 1.0.13
    :released: 2016 年 5 月 16 日

    .. change::
        :tags: bug, orm
        :tickets: 3700

        修复了  :meth:`.Session.merge`  中的问题，其中具有复合主键的对象对某些但不是所有 PK 字段具有值的情况下，将发出 SELECT 语句泄露内部 NEVER_SET 符号到查询中，而不是检测到此对象没有可搜索的主键并且不应该发出 SELECT。

    .. change::
        :tags: bug, postgresql
        :tickets: 3644

        修复了在   :func:`_expression.text`  构造中使用双冒号表达式时不会正确转义的问题，例如“some\\:\\:expr”，如 PostgreSQL 样式 CAST 表达式所需的最常见情况。

    .. change::
        :tags: bug, sql
        :tickets: 3642

        """
        __contains__ """
        的意外使用   :class:`_postgresql.ARRAY`  类型将导致这个 issue，在这种情况下，Python 会将其推迟到对“__getitem__”访问的访问中，而此类型永远不会因此引发。现在将引发 NotImplementedError。

    .. change::
        :tags: bug, orm
        :tickets: 3640

        修复了多个多态联接从 Mapper 命名空间中删除属性后未正确清除关系的问题。

.. changelog::
    :version: 1.0.12
    :released: 2016 年 2 月 15 日

    .. change::
        :tags: bug, orm
        :tickets: 3647

        修复了将具有复合主键且具有一些但不是所有 PK 字段的对象与   :func:`.post_update`  一起使用时，会失败并发出一个 UPDATE 的问题。将此属性设置为 None 并且以前未加载。

    .. change::
        :tags: bug, mysql
        :tickets: 3613

        修复了解析 MySQL 视图中的“ALGORITHM”关键字在 Amazon RDS 环境中可能不被使用的问题。

    .. change::
        :tags: bug, orm
        :tickets: 3611

        修复了由于修复“3593”问题而导致的问题，其中添加的用于 polymorphic joinedload 的检查会在 poly_subclass->class->poly_baseclass 连接的情况下失败，用于类->poly_subclass->class 的场景。

.. _changelog_1.0:

SQLAlchemy 1.0 Change Log
=========================

.. contents::

.. changelog::
    :version: 1.0.9
    :released: 20 October 2015

    .. change::
        :tags: bug, ext, orm
        :tickets: 3597, 3593, 3592, 2696, 3571

        Fixed several bugs related to querying in the ORM and SQL functionality.

    .. change::
        :tags: feature, sql
        :tickets: 3591

        Added support for parameter-ordered ``SET`` clauses in an ``UPDATE`` statement.

    .. change::
        :tags: bug, examples
        :tickets: 3510

        Fixed an issue in the "history_meta" example where history tracking
        could encounter empty history.

    .. change::
        :tags: bug, orm
        :tickets: 3525

        Fixed bug in  :meth:`.Session.bulk_save_objects`  where a mapped
        column that had some kind of "fetch on update" value and was not
        locally present in the given object would cause an AttributeError.

.. changelog::
    :version: 1.0.8
    :released: 22 July 2015

    .. change::
        :tags: bug, engine
        :tickets: 3494, 3481, 3483

        Fixed various engine and connection pool bugs.

    .. change::
        :tags: feature, orm
        :tickets: 3510

        Added new method  :meth:`_query.Query.one_or_none` ; same as
         :meth:`_query.Query.one`  but returns None if no row found.

    .. change::
        :tags: feature, schema
        :tickets: 3411, 3455

        Added support for new features of CREATE SEQUENCE and CREATE INDEX statements.

.. changelog::
    :version: 1.0.7
    :released: 20 July 2015

    .. change::
        :tags: feature, sql
        :tickets: 3459

        Added a  :meth:`_expression.ColumnElement.cast`  method which performs the same
        purpose as the standalone   :func:`_expression.cast`  function.

    .. change::
        :tags: bug, orm, postgresql
        :tickets: 3556

        Fixed regression in 1.0 where new feature of using "executemany"
        for UPDATE statements in the ORM would break on PostgreSQL.

.. changelog::
    :version: 1.0.6
    :released: 25 June 2015

    .. change::
        :tags: bug, orm
        :tickets: 3465, 3424, 3430

        Fixed several ORMs bugs, including one for MSSQL dialect.

    .. change::
        :tags: bug, postgresql
        :tickets: 3454

        Repaired the   :class:`.ExcludeConstraint`  construct to support common
        features that other objects like   :class:`.Index`  now do.

.. changelog::
    :version: 1.0.5
    :released: 7 June 2015

    .. change::
        :tags: bug, orm, pypy
        :tickets: 3405

        Fixed regression from 0.9.10 prior to release due to  :ticket:`3349` 
        where the check for query state on  :meth:`_query.Query.update`  or
         :meth:`_query.Query.delete`  compared the empty tuple to itself using ` `is``,
        which fails on PyPy to produce ``True`` in this case.

    .. change::
        :tags: feature, engine
        :tickets: 3379

        Added features to support engine/pool plugins with advanced functionality.

    .. change::
        :tags: bug, orm
        :tickets: 3403, 3320

        Fixed regression that caused issues in the querying functionality.

如 :class:`_schema.Table` 或 :class:`_expression.CTE` 对象。

.. change::
    :tags: feature, sql

    添加了一个占位符方法  :meth:`.TypeEngine.compare_against_backend`  ,
    如今Alembic迁移会使用它从而能被消耗到0.7.6。用户定义的types可以实现这个方法来帮助对一个从数据库中反映出来的type进行比较。

.. change::
    :tags: bug, orm
    :tickets: 3402

    修复了当一个属性为UPDATE的SQL表达式时，如果和该属性的上一个值进行比较将会产生SQL比较而不是“==”或“!=”时，出现的回归性问题。会出现"Boolean value of this clause is not defined" 的例外。这个修复确保了工作单元不会像此前那样解析SQL表达式。

.. change::
    :tags: bug, ext
    :tickets: 3397

    修复了一个关联代理的错误，其中任意()/has()在特定关系的标量非对象属性上进行比较将会失败，
    例如
    ```filter(Parent.some_collection_to_attribute.any(Child.attr == 'foo'))```

.. change::
    :tags: bug, sql
    :tickets: 3396

    修复了截断SQL中长标签的错误，这可能会产生一个标签，它与未被截断的标签重叠，这是因为截断的长度阈值大于仍然截断后的标签部分。这两个值现在已经变成了同样的长度; 标签长度减6。这里的影响是，较短的列标签将在以前不会被截断的地方“截断”。

.. change::
    :tags: bug, orm
    :tickets: 3392

    修复了由于  :ticket:`2992`  引入的意外使用回归问题，在joined eager加载修改查询时，如果文本元素放置在  :meth:` _query.Query.order_by`  应该在此出现的警告（请注意，问题不会影响显式使用   :func:`_expression.text`  的情况），仍会生成一个与先前不同的查询，其中"name desc"表达式不正确地复制到了列子句中。解决方法是，在"连接的贪婪加在"特征的内部列子句时，对这些所谓的"label reference"表达式进行跳过，就好像它们已经是   :func:` _expression.text`  的构造一样。

.. change::
    :tags: bug, sql
    :tickets: 3391

    修复了  :ticket:`3282` .DDLEvents.before_create` 、  :meth:`.DDLEvents.after_create` 、  :meth:` .DDLEvents.before_drop`  和  :meth:`.DDLEvents.after_drop`  事件中不再是表的列表，而是包含外键要添加或删除的第二个条目的元组列表。由于“表”集合虽然被记录为不一定是稳定的，但已被依赖，因此此更改被视为回归。此外，在某些情况下（例如删除），此集合将是一个迭代器，如果过早迭代操作将导致操作失败。此集合现在在所有情况下都是表对象的列表，并为该集合的格式添加了测试覆盖。

.. change::
    :tags: bug, orm
    :tickets: 3388

    修复了   :class:`.MapperEvents.instrument_class`  事件的回归问题，其中它的调用移到类管理器对类进行仪器化之后，这与事件的文档明确说明相反。切换的理由是由于Declarative在将类映射为new“@ declared_attr”功能的目的之前，就已经设置好了类的完整“仪表管理器”，但对于经典使用  :class:` _orm.Mapper`来说，这个变化也是对于一致性的。但是，SQLSoup在经典映射下依赖于仪器化事件在任何仪器化之前发生。情况在经典和声明性映射的情况下都实现了反转行为，后者使用简单的备忘录而没有使用类管理器。

.. change::
    :tags: bug, orm
    :tickets: 3387

    修复了  :meth:`.QueryEvents.before_compile`  报告的错误，这是由于  :ticket:` 3061`  引入的，其中NEVER_SET在使用joined eager loading修改  :meth:`.Query`  对象时会出现在与多个级别相关联的inner join查询中。None这个对象被返回，在这种情况下，但许多这些查询从来没有被正确支持过，而且已经在没有使用IS运算符的情况下产生对NULL的比较。出于这个原因，一个警告也被添加到这个子集的关系查询中，这些查询没有当前对“IS NULL”的支持。

.. change::
    :tags: bug, orm
    :tickets: 3388

    修复了使用了一个透明的  :class:`_expression.BinaryExpression` 对象后UI有可能会找不到预期列之前的问题，被  :ticket:` 3061`  回归的问题。对于持久化值而言，它总是会使用被保存到数据库的值而不是当前设置的值。

.. change::
    :tags: bug, sql
    :tickets: 3338, 3385

    修复了在1.0.0b4错误地修复的错误回归问题(因此变成了两个错误回归问题)。关于select语句，一个标签名称的后续排序问题被误解为某些后端比如SQL Server根本不应该在简单的标签名称上发出ORDER BY或GROUP BY；事实上，我们忘记了0.9已经为所有后端发出一个简单标签名称的ORDER BY了，如   :ref:`migration_1068` .Label` 结构时再次发出ORDER BY。此外，除非通过  :class:`.Label` 结构传递，否则不再发出对简单标签的GROUP BY 。

.. change::
    :tags: bug, orm, declarative
    :tickets: 3383

    修复了声明性“__declare_first__”和“__declare_last__”访问器的预期使用方式的错误回归，这些访问器不能在声明性基类的超类上正确调用属性和配置的缘故。

.. changelog::
    :version: 1.0.1
    :released: April 23, 2015

.. change::
    :tags: bug, firebird
    :tickets: 3380

    由于 :ticket:`3034` ，解决了对Firebird方言使用limit/offset时没有正确解释问题的回归问题。向公约致意的PR。

.. change::
    :tags: bug, firebird
    :tickets: 3381

    修复了Firebird在使用limit/offset和"literal_binds"模式时的支持，以便在选择此选项时，这些值可以再次内联呈现。与  :ticket:`3034`  相关。

.. change::
    :tags: bug, sqlite
    :tickets: 3378

    由于  :ticket:`3282` ，在创建和删除架构时，我们尝试根据ALTER的可用性来假定约束等对象不会存在，并针对SQLite实际上没有用ALTER的情况来说，我们只是不再担心foreign_keys了。这意味着在SQLite的情况下基本上跳过了表的排序，对于大多数SQLite用例，这是没有问题的。但是，正在使用带有数据且启用引用完整性的DROP SQLite操作的用户将遇到错误，当那些表具有数据时，依赖关系排序仍然有用，即使那些表具有引用的数据(SQLite 仍然可以愉快地让您创建不存在表的外键，并且启用约束的表引用现有表，只要没有引用的数据)。为了在保留 :ticket:` 3282` 新功能的同时，允许SQLite的DROP操作保持顺序，我们现在使用完整的前 FK 来按顺序进行排序，并且如果我们遇到无法解析的循环，那么只有这些对象才会忽略尝试进行表排序;我们输出警告并若使用特殊标志表不能排序则处理为没有排序；只有同时需要有排序的备份和FK循环遍历时，警告会通知他们需要恢复ALTER和他们的  :class:`_schema.ForeignKey` 和  :class:`_schema.ForeignKeyConstraint` 对象的use_alter标志。
    .. seealso::
          :ref:`feature_3282`  - 包含有关SQLite的更新注释。

.. change::
    :tags: bug, sql
    :tickets: 3372

    修复了直接SELECT EXISTS查询会失败地问题，原因是 Result Proxy 的列类型未正确分配为布尔型的结果映射:

.. change::
    :tags: bug, orm
    :tickets: 3374

    修复了使用一种形式的attribute比较将使NEVER_SET 的问题, 对于瞬态对象，它不会使用数据库保留的值，而是总是使用当前设置的值，对于持久值，它将始终使用数据库持久化的值而不是当前设置的值。假设是打开自动刷写，像持久化值一样进行调用通常不会出现这种情况，因为任何待处理的更改都将在任何情况下首先进行刷新。然而，这与 :meth:`_orm.Query.filter` 和 ` _orm.Query.with_parent` 中使用的逻辑不一致，它使用当前值并允许与瞬态对象进行比较。现在比较使用当前值而不是数据库存储的值。

    与此前修复 :ticket:`3061` 相关的所谓“NEVER_SET”问题不会在外部使用修饰的 "@ declared_attr" 对象时阻止对象绑定到新会话，即使在触发错误引发后继续执行。

   .. seealso::
      :ref:`bug_3374` 

.. changelog::
    :version: 1.0.2
    :released: April 24, 2015
.. change::
    :tags: bug, sql
    :tickets: 3338, 3385

    关闭 MSSQL、Oracle 方言上的“simple order by”标志；这是指：根据  :ticket:`2992`  的描述，对于在表达式中与列子句中的相同结构重叠的 ordey by 或 group by，不管被解析为整个表达式的名称还是分析为与子句中FROM的SELECT子句中的名称重叠，都将按照标签进行复制。对于MSSQL，这个行为现在是整个表达式默认复制的旧行为，因为MSSQL对这些特别是 GROUP BY 表达式可能非常挑剔。这种标志也被防御性地关闭到Firebird和Sybase方言。 
    .. seealso::
          :ref:`change_3338` 

.. changelog::
    :version: 1.0.3
    :released: May 18, 2015

    .. change::
        :tags: bug, engine
        :tickets: 3394

        修复了   :class:`_engine.Transaction`  的使用，在 __del__ 中强制关闭底层连接，如果初次尝试回滚事务时报错的问题。

    .. change::
        :tags: bug, sql
        :tickets: 3400

        修复了   :func:`_expression.text`  对象中常量参数的带有周期的内存泄漏。在这里，引用   :class:` .TextClause`  实例在不断带有新参数的情况下不被垃圾回收，这与   :class:`.BindParameter`  和   :class:` .TypeDecorator`  实例的情况不同。

    .. change::
        :tags: bug, orm
        :tickets: 3401

        修复了对纯SQL表达式连接的文本修饰符助手 (e.g. User.some_column + "foo") 的缓存问题, 而这只是某些情况下才进行缓存。

    .. change::
        :tags: bug, orm
        :tickets: 3403

        修复了使用了 **distinct()** 和 **scalar subquery** 时, 排序表达式的出现可以干扰子查询中 ORDER BY 子句的生成。

    .. change::
        :tags: feature, mssql
        :tickets: 3357

        将 SQL 执行时间附加到过程级别的“SQL Server Profiler”事件，如果在   :class:`_engine.Engine`  上启用了snowflake_logging，这个事件只有在Microsoft的SQL Server上才可以使用。并且这个功能要求使用 Python 2.

        返回的时间戳可以表示DateTimes，Time，或给定精度下的Decimal。 ::

    .. code-block:: python

            2015-05-01T00:14:10.312575
            2015-05-01T00:14:10.312575098
            2015-05-01 00:14:10.312575

    .. seealso::

          :ref:`dialects_mssql_profiling` 

    .. change::
        :tags: bug, mssql
        :tickets: 3412

        修正了   :class:`_mssql.TZDateTime`  的行为，使之更加与   :class:` _types.DateTime`   关联，并防止与 Python 的 `datetime.time` 类型相互作用。.. _change_notes_09_1:

Release 0.9.1
==============

Release date: 2014-06-20

This release contains a number of bug fixes and minor
enhancements, most prominently improved compatibility with certain
DBAPIs used within greenlets, as well as improvements for compiled
query execution.

Changelog for 0.9.1
===================

    .. change::
        :tags: mysql

        The MySQL dialect now supports MySQL 5.7's "JSON" datatype via the new
          :class:`.mysql.JSON`  construct.

        Additionally, the MySQL dialect applies ``ENGINE=InnoDB`` when creating
        tables by default. Previously, no engine type was specified, allowing
        the database to use whatever the default engine was, which was often
        MyISAM for MySQL <5.5.

    .. change::
        :tags: mysql

        Building upon the MySQL BOOLEAN datatype alpha implementation added in
        0.9, the MySQL dialect can now render fully syntactical BOOLEAN expressions,
        including ALL, ANY, and CASE constructs.   Updated the boolean comparison
        operator to use <= and >= to achieve the same effect as BEWTWEEN.

        .. seealso::

              :ref:`mysql_boolean` 

    .. change::
        :tags: postgresql

        The PostgreSQL dialect now uses the "simple" query protocol for
        `async` connections by default instead of the `extended` format, which
        was found to be incompatible with certain asynchronous libraries.

    .. change::
        :tags: mysql, security

        Added the ``unix_socket`` parameter to the MySQL connection URL syntax, allowing
        connecting to MySQL over Unix sockets. Also added an optional port
        parameter that can be set to None for connections to sockets.

    .. change::
        :tags: mysql

        The MySQL dialect now quotes table and schema names passed to it; this
        includes table names implicit in derived classes, as well as those
        in mappers and columns.   Patch courtesy yorik.sar.

        .. seealso::

              :ref:`change_4428` 

    .. change::
        :tags: mysql

        When MySQL is used with the   :class:`_mysql_connector.MySQLConnector`  driver,
        the character set charset can be overridden by setting the connect_args
        string to:
        "?charset=<charset_name>". charset_name must be one of MySQL's recognized
        character sets/catalog collations.

        .. seealso::

              :ref:`mysql_oursql_charset` 

    .. change::
        :tags: mysql

        The MySQL dialect now supports MySQL 5.7's "generated" columns, newly added in
        MySQL 5.7.6.
        A generated column is similar to a SQL Server computed column, in that it is
        computed based on the definition of another column or columns.
        Generated columns can be either STORED, meaning the values are calculated at
        insert time and stored on the disk, or VIRTUAL, meaning that the values are
        computed on the fly, per-query.

    .. change::
        :tags: postgresql, performance

        Added support for asynchronous execution to the PostgreSQL dialect.
        When using the async driver,  :meth:`_engine.Engine.execute`  methods
        return immediately after sending the query to the database, and a
        coroutine must be used to receive results.

    .. change::
        :tags: bug, mysql

        Fixed regression in change 069cbb55a410 where MySQL defaults for
        DATETIME, DATE, and TIMESTAMP types were broken, specifically that a
        default of CURRENT_TIMESTAMP with a null on update column setting would
        not function as expected with the DBAPI.

    .. change::
        :tags: bug, mysql

        MySQL zero-datetime support has been fixed for PEP-249 compatible
        databases: Now both null and the zero datetime can be inserted, as well
        as being returned as INSERTed via a rowcount.

    .. change::
        :tags: postgresql

        Adjusted psycopg2 server version detection to support version strings
        with text at the end.

        .. seealso::

              :ref:`postgresql_tips` 

    .. change::
        :tags: mysql

        An attempt is now made to automatically detect the connection character
        set when connecting to MySQL via PyMySQL. PyMySQL does not return the
        connection character set used automatically, since this needs to be manually
        requested. Added an optional charset parameter for use with PyMySQL. When
        it is not provided the dialect will attempt to determine the correct charset
        for the connection.

    .. change::
        :tags: mysql

        Added a new flag  :paramref:`_mysql_no_result_parameters`  to the MySQL dialect.
        When set to true (defaults to false), the MySQLdb DBAPI will not perform a SELECT
        following an INSERT or UPDATE which has column-level RETURNING / OUTPUT parameters.
        This can provide a performance boost when not using these result sets.

    .. change::
        :tags: mysql

        The MySQL dialect now skips the call to the "start transaction" method when
        autocommit is enabled on connections; this is a performance optimization
        when large numbers of small transactions are being emitted to the database,
        and is safe as the transaction is started irregardless of an explicit "begin"
        calling.

    .. change::
        :tags: mysql

        The MySQL dialect can now render the COLLATION keyword for a SELECT, UPDATE,
        DELETE, or INSERT statement, by specifying the collation keyword at the
        bind parameter level, e.g.:
        ``select([mytable], mytable.c.textfield.collate("utf8_general_ci"))``.
        This allows for selecting or ordering by a column and overriding the default
        collation method on a per-query basis.

        Note that the "charset" option has existed on the MySQL dialect for some time.
        In MySQL, the charset specifies how each character is encoded in bytes.
        The collation specifies how character comparison is decided, including case-
        sensitivity; it is "bound to" a specific charset. If none is specified, a default
        collation is used, that relies on the default charset of the table.
        See the MySQL documentation for more information.

        .. seealso::

              :ref:`mysql_collation` 

    .. change::
        :tags: postgresql

        PostgreSQL now supports binding arrays of datetime objects in the style expected
        by psycopg2. datetime objects were being converted to sqlalchemy.util.TimezoneUTC
        causing a TypeError during insertion.

    .. change::
        :tags: mysql

        The MySQL dialect now supports the "innodb_large_prefix" row format. It is used
        to enable index keys larger than 767 bytes when "innodb_file_format = barracuda"
        and "innodb_file_per_table = ON" are set.

        .. seealso::

              :ref:`mysql_large_prefix` 

    .. change::
        :tags: mssql

        The FreeTDS driver now report transactional characteristics such as AUTOCOMMIT
        directly from query execution results.

    .. change::
        :tags: postgresql

        Fixed an issue where arrays containing an identifier larger than 63
        characters would not be reflected correctly. The column name is now truncated
        to the proper length.

    .. change::
        :tags: orm

        Added the method  :meth:`.orm.interfaces.MapperOption.short_circuit` ,
        which applies a given filtering criterion to a relationship at a level
        of specificity greater than that of the simpler ``.contains_eager()``
        method.   The new modifier is intended for the unusual case where a
        joinedload needs to depend on multiple criteria in a series of joins,
        and the overhead of mapping each row to custom objects outweighs
        the benefit.  The new option replaces the precedent
         :paramref:`.orm.contains_eager.optimized`  parameter.

        .. seealso::

              :ref:`contained_eager` 

    .. change::
        :tags: mysql, mariadb

        Added support for the MariaDB dialect which is a fork of MySQL server
        from one of its original authors. MariaDB features and compatibility are
        generally similar to MySQL.

        .. seealso::

              :ref:`dialect_mariadb` 

    .. change::
        :tags: orm

        Improved support for non-contiguous PRIMARY KEYs between the mapping and the target
        table, or between the joins in a relationship(), in that
        "secondary and "primaryjoin" can now be expressed as a textual
        SQL expression.

        .. seealso::

              :ref:`relationship_non_contiguous_key` 

    .. change::
        :tags: orm

        Added a new flag  :paramref:`_generic_autoincrement`  which is used to adapt the
        handling of GENERATED AS IDENTITY for sequences which are of type BIGINT with a
        registered DDL level INTEGER type. Without this flag, SQLA reports an incompatible
        type error when such a type referenced by a table column defaults to GENERATED AS
        IDENTITY. If the flag is set to be True and the version of the targeted backend is
        greater than or equal to 10.2 (for PostgreSQL) or 6.12 (for SQLite), SQLA will cast
        INTEGER type to BIGINT type to prevent returning value of id column being limited
        by INTEGER. As an example, PostgreSQL dialect has this flag True by default for
        versions greater than or equal to 10.2.

    .. change::
        :tags: postgresql

        The postgresql dialect now correctly represents references to sequence-backed columns
        in default expressions when reflecting table/view constructs.  Previously, when
        `serial` or `bigserial` was used, the default expression was an SQL expression, not a
        Sequence object. This issue was initially addressed and then backed out in SQLA 0.8
        series, as it was found that using an early Py2k-era pg8000 version, an IDENT_CURRENT
        expression must be used to achieve the same functionality. The pg8000 issue was resolved
        after backporting a fix from alchemy master branch. The dialect uses a regular Sequence
        object such that the backend can use it in constructing e.g. default expressions via
        pre-execution, server-side defaults.

    .. change::
        :tags: mysql

        MySQL timestamp and datetime types that have null=False and no default,
        and which use the "default" backend translator, now render correctly
        with quotes so that the "DEFAULT" keyword is not used.

    .. change::
        :tags: postgresql

        The PostgreSQL dialect will add a simple "ORDER BY" clause of the primary key
        to queries that are aliased using the `alias()` construct, or which otherwise
        omit a simple ORDER BY clause. This can help to avoid undefined result set order.

    .. change::
        :tags: postgresql

        The PostgreSQL dialect will additionally pass through the `cursor_factory`
        option as passed via the   :func:`_engine.create_engine`  connect_args dictionary
        to the psycoPg2 driver.

        .. seealso::

              :ref:`postgresql_tips` 

    .. change::
        :tags: mysql

        MySQL datetime types that have a precision or fractional component are now
        properly reflected using the same precision. Previously the microseconds
        component of a MySQL 5.5+ timestamp could be lost and converted to a date type
        without microseconds.

    .. change::
        :tags: mssql

        The dialect now supports the "statistics_time" column returned by the server's
        profiling feature as a timed double.

    .. change::
        :tags: mysql

        The MySQL dialect's schema reflection for "FULLTEXT" index definitions now
        includes the entire index definition directly, rather than a limited summary,
        as it is necessary for the rendering of CREATE TABLE statements. In the past,
        DISTINCT and FULLTEXT index syntax were not fully recognized by the dialect
        but support for both has been added.

    .. change::
        :tags: bug, postgresql

        Fixed compile-time issue with PostgreSQL ARRAY types whose "NULL" status was
        uncertain. The configure.in script tests available Postgres installations and
        extends dialects that show a version 9.2 and above that ARRAY types can have
        NULL values.

    .. change::
        :tags: mysql

        Added support for the "ON UPDATE CURRENT_TIMESTAMP" clause AUTO_UPDATE_TIME in the
        MySQL dialect.

    .. change::
        :tags: mysql

        The MySQL/MariaDB dialect will now render the MySQL-specific syntax for
        expressions like `CURRENT_TIMESTAMP(n)` when the dialect flag
         :paramref:`_mysql_use_fractional_seconds`  is True. Currently MySQL
        stores fractional seconds as a value up to six digits in length.

        .. seealso::

              :ref:`mysql_fractional_time` 

    .. change::
        :tags: orm

        The `Query` method `enable_assertions()` may now also act as a
        context manager.

    .. change::
        :tags: orm

        Fixed regression caused by not copying all fields when a MapperOptions object
        is created with kwargs.

    .. change::
        :tags: mysql

        MySQL will now automatically add "ENGINE=InnoDB" to non-temporary CREATE
        TABLE statements when the server's default table type is something
        other than InnoDB.

    .. change::
        :tags: mysql

        MySQL datetime types that have a precision or fractional component are now
        properly reflected and used in schema generation.

    .. change::
        :tags: mysql

        Fixed bug in the MySQL dialect's MySQLdb backend where use_unicode=True and
        CHARSET kwargs do not work together.

    .. change::
        :tags: mysql

        Removed deprecated warning raised when executing queries that might invoke
        the true (non-emulated) decimal type in the mysql dialect.

    .. change::
        :tags: mysql

        Added some conditional behavior to adjust the MySQLdb connection charset
        for certain Python 2 installs that don't include a Unicode codecs for
        some Linux platforms, in order to improve compatibility with a wider range
        of Python/MySQL installations.

    .. change::
        :tags: oracle

        The check for whether the `cx_Oracle` driver is correctly installed
        has been enhanced to cover specific errors users were encountering.

    .. change::
        :tags: postgresql

        Postgresql in streams mode now correctly sets the transaction isolation level.

    .. change::
        :tags: mssql

        The Microsoft ODBC driver for Python `pyodbc` now supports passing
        a string (e.g. nchar or nvarchar) column param to the `.type()` method
        of `Column`, instead of explicitly specifying a Unicode type.

        .. seealso::

              :ref:`mssql_pyodbc_nvarchar` 

    .. change::
        :tags: mssql

        Support for managing GUID types has been added to the Microsoft SQL Server
        dialect when used via the `pyodbc` driver. This fixes a regression where `VARCHAR(36)`
        had been emitted instead of the `UNIQUEIDENTIFIER` type.

    .. change::
        :tags: mssql

        SQL server dialects now wrap "text" and "image" column types with the
        "TEXT" and "IMAGE" data type controls, which are required for streaming BLOBs and
        other large types.

    .. change::
        :tags: mysql

        MySQL fractional-second support has been improved, such that sql.DateTime and
        String types with an embedded fractional component are now handled both at the bind
        parameter level as well as in rendering for CREATE TABLE.

    .. change::
        :tags: mysql

        MySQL added support for more than 64 indexes, MySQL indexes were hardcoded to 64 by default.
        A new dialect level flag has been added that allows the default to 64, which is
        provided merely for backwards-compatibility.

    .. change::
        :tags: mysql

        The MySQL dialect now fully provides DATE_TRUNC support including
        MADTP syntax if otherwise available. Until now, to use features like 'hour' which
        are not explicit in the DATE_TRUNC implementation, expressions like DATE_ADD(DATE_TRUNC('day', x),'1 hour') had to be used.

        .. seealso::

              :ref:`mysql_date_trunc` 

    .. change::
        :tags: mysql

        Added support for MySQL RENAME INDEX statement with the new
         :meth:`_mysql.MysqlDialect.execute_index_rename`  method.

    .. change::
        :tags: mysql

        The MySQL dialect now correctly quotes backticks in CHECK constraint strings
        along with other quoted constructs.

    .. change::
        :tags: mysql

        The MySQL dialect now encodes JSON data to text in utf8, which matches the default
        // encoding MySQL uses for JSON.

    .. change::
        :tags: oracle

        Added support for fetching the Lob data when using the `fetchone` method of
        an Oracle Cursor object.

    .. change::
        :tags: postgresql

        Added support for the `jsonb` type added in PostgreSQL 9.4's json module
        to the `postgresql` dialect, using the new `JSONB` supported type.

    .. change::
        :tags: oracle

        Added support for the `Oracle 12` type `BINARY_FLOAT` and `BINARY_DOUBLE`
        when using `cx_Oracle`.

    .. change::
        :tags: mysql

        Added support for the 5.7.5 `JSON_VALID` MySQL JSON function.

    .. change::
        :tags: mysql

        Fixed issue where MySQL's `CAST` would be used instead of `CONVERT_TZ` when
        converting time values in the MySQL >= 5.5 dialect.

    .. change::
        :tags: mysql

        MySQL boolean "sqlite-style" boolean support was added with the addition of the
          :class:`_mysql.BOOLEAN`  and   :class:` _mysql.TINYINT`  types. These types should
        be used with columns which use MySQL's "TINYINT(1)" type.

        .. versionadded:: 0.9.7

        .. seealso::

              :ref:`mysql_boolean` 

    .. change::
        :tags: mysql

        MySQL character set compatibility upgrades have been made in the dialect.
        Unicode types will now work with the default "latin1" character set of MySQL
        when used with particular drivers, including "oursql" and "pyodbc including
        the Microsoft-provided pyodbc driver for Linux and Mac".

        .. seealso::

              :ref:`mysql_unicode_support` 


    .. change::
        :tags: postgresql

        PostgreSQL arrays must be passed to to_sql() using the namedtuple form of the
        psycopg2.extras.register_dialect() function.

    .. change::
        :tags: mysql

        Added to the MySQL dialect support for the new 5.7.6 Microsecond timestamps.
        The new Microsecond timestamps are Python datetime objects with six digits of precision.

        .. seealso::

             :ref:`mysql_microsecond_precision

    .. change::
        :tags: mysql

        Added to the MySQL dialect support for the new 5.7.9 Instant columns (DATE, DATETIME, TIMESTAMP).
        Instant columns are Python datetime/timestamp objects that can represent a MySQL column
        with precision up to 9 fractions of a second.

        .. seealso::

              :ref:`mysql_instant` 

    .. change::
        :tags: mysql

        Added to the MySQL dialect Query an additional argument for the pyMySQL cue which specifies whether
        metadata reflection should include hidden tables.

    .. change::
        :tags: mysql

        Additional improvements to MySQL's microseconds precision support have been made.
        The new implementation supports microseconds for `LOAD DATA INFILE` in the MySQLdb as well
        as for `cursor.fast_executemany` and the `executemany()` method of MySQLdb's connection object.

    .. change::
        :tags: mysql

        Added to the MySQL dialect support for MariaDB dynamic columns.
        MariaDB dynamic columns are user-defined types which can handle structured
        data in a flexible way.        

    .. change::
        :tags: mysql

        The MySQL dialect now correctly supports the JSON type when used in conjunction with
        LOAD DATA INFILE.

    .. change::
        :tags: postgresql

        Added support for PostgreSQL `SKIP LOCKED` syntax on UPDATE/DELETE querying
        with the dialect's `Query.update()` and `Query.delete()` methods.

        .. seealso::

              :ref:`postgresql_skip_locked` 


    .. change::
        :tags: mysql

        Added support for MySQL 8 TIMESTAMP columns that are set to NONE
        when using the MySQLConnector Python driver.

    .. change::
        :tags: postgresql

        Added support for PostgreSQL 9.6 arrays that use the range operator `..`
        when compared with SQL expressions.

    .. change::
        :tags: postgresql

        The Postgres dialect checks if \d is supported on a version of PG before testing
        indices for comments as \d does not exist on some versions of PG.

    .. change::
        :tags: oracle

        Added support for the Oracle driver's `matchlog`, `updatelog` and `insertlog`
        Cursor functions.

    .. change::
        :tags: mysql

        Support has been added for features which prevent SQL loading of local files
        via MySQL clauses such as `LOAD DATA LOCAL INFILE`. These are toggled by
        setting `local_infile` parameter to `0`

    .. change::
        :tags: mysql

        The MySQL dbapi `oursql` is now supported by the MySQL dialect.

    .. change::
        :tags: mysql

        Added support for 5.7.6 fractional second timestamps in the mysql
        and pymysql dialects.


    .. change::
        :tags: mysql

        Added support for the now second default column format of MySql 5.7.5,
        in which the column precision is included.

    .. change::
        :tags: postgresql

        Added support for PostgreSQL 10's identity columns; the `SERIAL` type should
        now be considered deprecated in PostgreSQL.

    .. change::
        :tags: sqlite

        `adhoc` Autoincrement support was added for the SQLite dialect when the
        SQLite version in use is 3.31 or greater. This dialect flag causes newly
        created integer primary key AUTOINCREMENT columns to use sequences instead
        of the `AUTOINCREMENT` keyword, which allows the type of AUTOINCREMENT to be
        determined automatically by the SQLite backend.

    .. change::
        :tags: mssql

        Changed the behavior for SaString to string conversion for when the value exceeds
        8060 bytes (the maximum allowable size for a record). Previously, an error would be
        raised. Now, the conversion will succeed and the generated value will be truncated
        to the allowable maximum length... seealso::

      :ref:`feature_3150` 

.. change::
    :tags: feature, ext
    :tickets: 3210

     :mod:`sqlalchemy.ext.automap` .automap.generate_relationship` 的关键字中，但仍可被覆盖。此外，如果  :class:`_schema.ForeignKeyConstraint` ”或“ondelete="SET NULL"` ”，则参数“passive_deletes=True”也会被添加到关系中。请注意，并非所有后端都支持ondelete反射，但支持反射的后端包括 PostgreSQL 和 MySQL。

.. change::
    :tags: feature, sql
    :tickets: 3206

    添加新方法  :meth:`_expression.Select.with_statement_hint`  和 ORM 方法
     :meth:`_query.Query.with_statement_hint` ，以支持不特定于表的语句级提示。

.. change::
    :tags: bug, sqlite
    :tickets: 3203

    SQLite 现在支持从临时表中反射唯一限制; 之前，这将导致 TypeError 错误。Pull request 由 Johannes Erdfelt 提供。

    .. seealso::

          :ref:`change_3204`  - 关于 SQLite 临时表和视图反射的更改。

.. change::
    :tags: bug, sqlite
    :tickets: 3204

    添加  :meth:`_reflection.Inspector.get_temp_table_names`  和
     :meth:`_reflection.Inspector.get_temp_view_names` ；当前，仅 SQLite 和 Oracle 方言支持这些方法。删除了临时表和视图名称的返回，SQLite 和 Oracle 的版本：meth:` _reflection.Inspector.get_table_names` 和
     :meth:`_reflection.Inspector.get_view_names` ；其他数据库后端无法支持此信息（例如MySQL），而操作的范围受到限制，因为表可以局部于会话，通常不支持远程模式。

    .. seealso::

          :ref:`change_3204` 

.. change::
    :tags: feature, postgresql
    :tickets: 2891

    添加对材料化视图和外部表反射的支持，以及  :meth:`_reflection.Inspector.get_view_names`  内支持材料化视图的支持，以及在 PostgreSQL 版本的   :class:` _reflection.Inspector`  上可用的新方法  :meth:`.PGInspector.get_foreign_table_names` 。Pull request 由 Rodrigo Menezes 提供。

    .. seealso::

          :ref:`feature_2891` 


.. change::
    :tags: feature, orm

    添加新事件处理程序  :meth:`.AttributeEvents.init_collection`  和  :meth:` .AttributeEvents.dispose_collection` ，用于跟踪当集合首次与实例关联以及被替换的时间。这些处理程序取代了  :meth:`.collection.linker`  注释。旧的挂钩仍然通过事件适配器得到支持。

.. change::
    :tags: bug, orm
    :tickets: 3148, 3188

    对表达式标签的行为进行了重大改进，尤其是在使用自定义 SQL 表达式的 ColumnProperty 构造以及与“按标签排序”逻辑一起使用时的行为。修复包括一个“order_by(Entity.some_col_prop)”将使用“ 按标签排序”的规则，即使通过继承呈现或使用“aliased()”构造进行了别名处理，也将使用 “按标签排序”规则，多个别名的相同列属性（例如“query(Entity.some_prop，entity_alias.some_prop)”）将使每个实体的每个出现具有不同的标签，并且此外，“按标签排序”规则将适用于两者（例如“order_by(Entity.some_prop，entity_alias.some_prop)”）。
    可能会阻止“order by 标签”逻辑在 0.9 中正常工作的其他问题，特别是如果 Label 的状态会因调用方式不同而发生更改，例如取决于如何被调用， 将已经修复。

    .. seealso::

          :ref:`bug_3188` 


.. change::
    :tags: bug, mysql
    :tickets: 3186

    MySQL 布尔符号 "true"、"false" 再次生效。0.9 中的 更改和 :ticket: 2682 禁止 MySQL 方言在“IS”/"IS NOT" 的上下文中使用 "true" 和 "false" 符号，但是 MySQL 支持此语法，即使它没有布尔类型。
    MySQL 仍然是 “非本机布尔”，但   :func:`.true`  和   :func:` .false`  符号再次生成关键字 "true" 和 "false"，因此可以在 MySQL 上正常使用表达式，例如 ``column.is_(true())``。

    .. seealso::

          :ref:`bug_3186` 

.. change::
    :tags: changed, mssql
    :tickets: 3182

    在使用 pyodbc 时，基于主机名的 SQL Server 连接格式将不再指定默认的“driver name”，如果缺少，会发出警告。SQL Server 的最佳驱动程序名称经常更改且基于每个平台，因此基于主机名的连接需要指定此名称。DSN 连接被推荐使用。

    .. seealso::

          :ref:`change_3182` 

.. change::
    :tags: changed, sql

      :func:`_expression.column`  和   :func:` _expression.table`  构造现在可以从“from sqlalchemy”命名空间中导入，就像所有其他 Core 构造一样。

.. change::
    :tags: changed, sql
    :tickets: 2992

    当被传递到大多数   :func:`_expression.select`  构造器方法以及   :class:` _query.Query`  的 builder 方法中时，字符串到   :func:`_expression.text`  构造器的隐式转换现在发出警告，带有纯字符串的警告。然而，文本转换仍正常进行。唯一一个不带警告接受字符串的方法是“label reference”方法，例如 order_by()，group_by()；这些函数现在在编译时将尝试将单个字符串参数解析为可选择的可选择可选项中存在的列或标签表达式；如果未找到其中任何一个，则该表达式仍将呈现，但您会再次获得警告。这里的基本理由是，从字符串到文本的隐式转换现在是更加意外的，因此最好在将原始字符串传递时为 Core/ORM 提供更多的指导方向。 Core/ORM 教程已更新为深入介绍文本处理的方式。

    .. seealso::

          :ref:`migration_2992` 


.. change::
    :tags: feature, engine
    :tickets: 3178

    现在可以发出新样式的警告，这些警告将“过滤”掉最多 N 次参数化字符串中的出现。这允许传递参数的警告引用其参数被传递的固定次数，然后允许 Python 警告过滤器将它们消除，并防止 Python 的警告注册表无限增长。

    .. seealso::

          :ref:`feature_3178` 

.. change::
    :tags: feature, orm

    当  :meth:`_query.Query.yield_per`  与可能进行子查询的延迟加载或集合执行的联接自动加载不兼容时，  :class:` _query.Query`  将抛出异常。这些加载策略目前与 yield_per 不兼容，因此通过引发此错误，该方法更安全可用。可以使用 ``lazyload('*')`` 选项或  :meth:`_query.Query.enable_eagerloads`  禁用急切加载。

    .. seealso::

          :ref:`migration_yield_per_eager_loading` 

.. change::
    :tags: bug, orm
    :tickets: 3177

    在使用  :meth:`_query.Query.from_selfhttps`  或其常用用户  :meth:` _query.Query.count`  时，"单一继承标准"的应用程序方式已更改，现在应用要限制行只有特定类型的标准指示“在内部子查询中”，而不是外部子查询中，因此，即使“ type”列不在列子句中，我们也可以在“内部”查询上过滤它。

    .. seealso::

          :ref:`migration_3177` 

.. change::
    :tags: changed, orm

    通过自定义   :class:`.Bundle`  类的  :attr:` .proc`  可调用对象现在只接受一个“行”参数。

    .. seealso::

          :ref:`bundle_api_change` 

.. change::
    :tags: changed, orm

    已删除弃用的事件钩子：“populate_instance”、“create_instance”、“translate_row”、“append_result”

    .. seealso::

          :ref:`migration_deprecated_orm_events` 

.. change::
    :tags: bug, orm
    :tickets: 3145

    对延迟加载的机制进行了小调整，以便在极少见的情况下不会干扰右外联接操作中的 joinload()；在这种情况下，对象在加载其属性时指向自身；这可能导致加载程序之间的混淆。"对象指向自身"的用例并未完全得到支持，但是该修复还消除了一些开销，因此目前是测试的一部分。

.. change::
    :tags: feature, orm
    :tickets: 3176

      :class:`.KeyedTuple`  的新实现，由   :class:` _query.Query`  对象使用，在获取大量面向列的行时提供了显着的速度提高。

    .. seealso::

          :ref:`feature_3176` 

.. change::
    :tags: feature, orm
    :tickets: 3008

    当内部加入连接加载链接到外部加入加载时，现在会使用 "嵌套"内部连接作为默认行为，即右嵌套，作为内连接 joined eager load。具体请参阅  :paramref:`_orm.joinedload.innerjoin`  和  :paramref:` _orm.relationship.innerjoin` 。

    .. seealso::

          :ref:`migration_3008` 

.. change::
    :tags: bug, orm
    :tickets: 3171

    已删除“复活”ORM事件。此事件钩子没有用处，因为在 0.8 中移除了旧的“可变属性”系统。

.. change::
    :tags: bug, sql
    :tickets: 3169

    使用  :meth:`_expression.Insert.from_select`  现在会隐含地设置   :func:` _expression.insert`  上的 ``inline=True``。这有助于修复一个错误，即 INSERT...FROM SELECT 结构通常会被意外地在支持的后端上编译为“隐式返回”，这会在 INSERT 插入零行的情况下（因为隐式返回期望一行），导致错误，也会在插入多个行的情况下产生任意的返回数据（例如，仅多个中的第一行）。

.. change::
    :tags: bug, orm
    :tickets: 3145

    对延迟加载的机制进行了小调整，以便在极少见的情况下不会干扰右外联接操作中的 joinload()；在这种情况下，对象在加载其属性时指向自身；这可能导致加载程序之间的混淆。"对象指向自身"的用例并未完全得到支持，但是该修复还消除了一些开销，目前是测试的一部分。

.. change::
    :tags: feature, orm
    :tickets: 2963

    添加了   :class:`.SynonymProperty`  和   :class:` .ComparableProperty`  构造函数中的 ``info`` 参数到构造中。

.. change::
    :tags: sql, feature
    :tickets: 2963

    添加了所有模式构造函数，包括   :class:`_schema.MetaData` 、  :class:` .Index` 、  :class:`_schema.ForeignKey` 、  :class:` _schema.ForeignKeyConstraint` 、  :class:`.UniqueConstraint` 、  :class:` .PrimaryKeyConstraint` 、  :class:`.CheckConstraint`  的构造器中的 ` `info`` 参数。

.. change::
    :tags: orm, feature
    :tickets: 2971

     :attr:`.InspectionAttr.info`  集合现在移到了   :class:` .InspectionAttr`  中，在所有   :class:`.MapperProperty`  对象上均可使用，也在混合属性和关联代理时可通过  :attr:` _orm.Mapper.all_orm_descriptors`  访问。

.. change::
    :tags: sql, feature
    :tickets: 3027

    现在  :paramref:`_schema.Table.autoload_with`  标志意味着  :paramref:` _schema.Table.autoload`  应为 ``True``。通常，这用于允许传递绑定参数，稍后可以用值替换它们，从而允许 Python 端缓存 SQL 查询。在其中实现此功能的第三方方言现在完全向后兼容，但是实现特殊的 LIMIT/OFFSET 系统的那些方言将需要修改以利用新的功能。Limit 和 offset 现在也支持 "literal_binds" 模式，其中绑定参数基于编译时选项以字符串形式内联呈现。

    .. seealso::

          :ref:`feature_3034` .

.. change::
    :tags: feature, postgresql

    当使用检查器反映 PostgreSQL 时，添加了新方法  :meth:`.PGInspector.get_enums` ，将为 PostgreSQL 提供 ENUM 类型的列表。Pull request 由 Ilya Pekelny 提供。

.. change::
    :tags: mysql, bug

    现在，MySQL 方言将为其内部使用的语句禁用  :meth:`_events.ConnectionEvents.handle_error`  事件以检测表是否存在。这是使用执行选项 ` `skip_user_error_events`` 实现的，该选项禁用了该执行范围内的句柄错误事件。因此，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的偶然发生的情况。

.. change::
    :tags: mysql, bug
    :tickets: 2515

    将 MySQLconnector 的“raise_on_warnings”默认值更改为 False。某些原因将其设置为 True。不幸的是，“buffered”标志必须继续保持为 True，因为 MySQLconnector 不允许关闭游标，除非完全获取了所有结果。

.. change::
    :tags: orm, feature
    :tickets: 2963

      :class:`.ComparableProperty`  和   :class:` .SynonymProperty`  的构造函数中现在添加了 ``info`` 参数。

.. change::
    :tags: orm, feature

    现在可以对 ORM 刷新中的 UPDATE 语句进行分批处理，以便更高效地进行调用 execute_many()，类似于 INSERT 语句可以进行批处理。这将在刷新中调用到，以后续的 UPDATE 语句涉及到相同的映射和表，其 VALUES 子句中涉及到的列是相同的，其中没有嵌入 SET-level SQL表达式，且映射的版本需求与后端方言的能力兼容，以返回正确的行数。插入了多个行时，上述限制也适用于 INSERT..VALUES，随着插入行数增加，  :attr:`_engine.ResultProxy.inserted_primary_key`  访问器不适用。在这些构造处理不定行数的情况下，如果需要插入的数据可用，则应该使用常规的显式  :meth:` _expression.Insert.returning` 。

.. change::
    :tags: engine, bug
    :tickets: 3163

    移除（或添加）事件侦听器的时候，如果正在运行事件本身，则发出 RuntimeError 异常，不管是在侦听器内部还是在并发线程中发生的。原因是现在使用 `collections.deque()` 的实例进行遍历，而在遍历过程中不支持更改。之前使用普通 Python 列表，当事件本身内置已被删除时，删除将导致静默失败。

.. change::
    :tags: oracle, feature

    添加了对 Oracle 表选项 ON COMMIT 的支持。

.. change::
    :tags: postgresql, feature
    :tickets: 2051

    当通过   :class:`_schema.Table`  渲染 DDL 时，PG 表选项 TABLESPACE、ON COMMIT、WITH(OUT) OIDS 和 INHERITS 的支持已添加。Pull request 由 Malik Diarra 提供。

    .. seealso::

          :ref:`postgresql_table_options` 

.. change::
    :tags: bug, orm, py3k

      :class:`.IdentityMap`  现在在 Py3K 中为 ` `items()`` 和 ``values()`` 返回列表。此前，早期向 Py3K 传递迭代器时，它们实际上是“迭代式视图”…但是，暂时，列表是可行的。

.. change::
    :tags: sql, feature
    :tickets: 3034

     :meth:`_expression.Select.limit`  和  :meth:` _expression.Select.offset`  方法现在接受任何 SQL 表达式（除了整数的值）作为参数。通常，这是用于允许传递绑定参数，可以稍后用值替换它们，从而允许 Python 端缓存 SQL 查询。实现在现有的第三方方言中完全向后兼容，但是那些实现了特殊的 LIMIT/OFFSET 系统的方言将需要修改以利用新功能。Limit 和 offset 还支持 “literal_binds” 模式，其中绑定参数根据编译时选项作为字符串进行内联呈现。

    .. seealso::

          :ref:`feature_3034` .