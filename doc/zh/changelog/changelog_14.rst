=============
1.4 版本更新历史
=============

本文档详细记录了在1.4版中进行的各个单个问题级别的更改。有关1.4版中的新内容的概述，请参见  :ref:`migration_14_toplevel` 。

.. changelog_imports::

    .. include:: changelog_13.rst
        :start-line: 5


.. changelog::
    :version: 1.4.50
    :include_notes_from: unreleased_14

.. changelog::
    :version: 1.4.49
    :released: 2023年7月5日

    .. change::
        :tags: bug, sql
        :tickets: 10042
        :versions: 2.0.18

        修复了当使用"flags"时， :meth:`_sql.ColumnOperators.regexp_match` 方法无法生成"稳定"缓存键的问题，即缓存键每次都会改变，导致缓存污染。 :meth:`_sql.ColumnOperators.regexp_replace` 方法也存在同样的问题，既包括标志，也包括实际替换表达式。现在，将标志表示为固定的修饰符字符串呈现为安全字符串，而不是绑定参数，并且在"二进制"元素的主要部分中建立替换表达式，以生成适当的缓存键。

        注意，作为此更改的一部分， :paramref:`_sql.ColumnOperators.regexp_match.flags` 和 :paramref:`_sql.ColumnOperators.regexp_replace.flags` 已被修改为仅呈现为字面字符串，而以前它们以完整的SQL表达式呈现，通常是绑定参数。应始终将这些参数作为普通的Python字符串传递，而不是作为SQL表达式构造传递；不希望在实践中使用SQL表达式构造这个参数，因此这是一项不兼容的更改。

        更改还修改了表达式的内部结构，对于带或不带标志的  :meth:`_sql.ColumnOperators.regexp_replace` ，以及对于带标志的  :meth:` _sql.ColumnOperators.regexp_match` 。第三方方言可能已经实现了它们自己的正则表达式实现(在搜索中找不到这样的方言，所以影响预计很小)，它们需要调整结构的遍历以适应。


    .. change::
        :tags: bug, sql
        :versions: 2.0.18

        修复了大多数内部问题   :class:`.CacheKey`  构造，在该构造中
        ``__ne __()``运算符未正确实现，导致比较   :class:`.CacheKey`  实例之间时得到非常奇怪的结果。




    .. change::
        :tags: bug, extensions
        :versions: 2.0.17

        修复了使用mypy 1.4 的mypy插件中的错误。

    .. change::
        :tags: platform, usecase

        兼容性改进以完全与Python 3.12配合使用

.. changelog::
    :version: 1.4.48
    :released: 2023年4月30日

    .. change::
        :tags: bug, orm
        :tickets: 9728
        :versions: 2.0.12

        修复了关于缓存的重要问题，组合：func:`_orm.aliased()`和   :func:`_hybrid.hybrid_property`  对象，同时也无法匹配等效结构，填充/占用缓存。

    .. change::
        :tags: bug, orm
        :tickets: 9634
        :versions: 2.0.10

        修复了当SQL语句本身为"复合选择"(如UNION)时，各种特定于ORM的getter(例如  :attr:`.ORMExecuteState.is_column_load` 、  :attr:` .ORMExecuteState.is_relationship_load`  、  :attr:`.ORMExecuteState.loader_strategy_path`  等)会抛出“AttributeError”的错误。

    .. change::
        :tags: bug, orm
        :tickets: 9590
        :versions: 2.0.9

        修复了使用“关系到别名类”的功能并在加载程序上表示递归的持续循环，例如在加载程序中同时使用“lazy = selectinload”和另一个eager loader的相反面。检查循环的检查已被修复，以包含别名类关系。

.. changelog::
    :version: 1.4.47
    :released: 2023年3月18日

    .. change::
        :tags: bug, sql
        :tickets: 9075
        :versions: 2.0.0rc3

        修复了一个错误/回归，即在使用   :func:`.bindparam()`  与相同名称的列在  :meth:` .Update.values`  方法和2.0中的  :meth:`_dml.Insert.values`  方法中的某些情况下，会在默默中失败，不遵循其中的SQL表达式 , 用参数表达式替换表达式，例如SQL函数等等。特定的情况是针对ORM实体构建的语句，而不是纯   :class:` .Table` .Session`或 :class:`.Connection` 调用语句，则会发生。

          :class:`.Update`  部分的问题存在于2.0和1.4中，并已退回到1.4。

    .. change::
        :tags: bug, oracle
        :tickets: 5047

        将   :class:`_oracle.ROWID`  添加到反映类型中，因为该类型可能在“CREATE TABLE”语句中使用。

    .. change::
        :tags: bug, sql
        :tickets: 7664

        为   :class:`.CreateSchema`  和   :class:` .DropSchema`  的字符串化修复，当字符串化它们时，将引发 ``AttributeError，`` 这些字符串包含的字母数字的节点的内容不被识别为元素的其余部分。

    .. change::
        :tags: usecase, mysql
        :tickets: 9047
        :versions: 2.0.0

        添加对SQLAlchemy中的MySQL索引反射的支持，以正确反映先前被忽略的 “mysql_length”字典的长度。

    .. change::
        :tags: bug, postgresql
        :tickets: 9048
        :versions: 2.0.0

        在与asyncpg方言一起使用时，添加对于SELECT语句返回“cursor.rowcount”值的支持。虽然这不是“cursor.rowcount”的典型使用方式，但是其他PostgreSQL方言通常提供此值。来自Michael Gorven的拉取请求。

    .. change::
        :tags: bug, mssql
        :tickets: 9133

        修复了通过使用尖括号给定的架构名称，但名称中没有句点的参数（例如  :paramref:`_schema.Table.schema` ）时的问题，在这种情况下，参考Oracle方言的有关解释显式括号指示符的文档行为（在＃2626中首次添加），在反映操作中引用模式名称时不会解析表达式中的这些符号定界符，并发现最初对于＃2626行为的假设是括号的特殊解释仅在句点存在时才是显着的 在实践中，括号不包括在所有SQL呈现操作的标识符名称中，因为这些在常规标识符或分隔符中不是有效的字符。拉取请求来自Shan。

    .. change::
        :tags: bug, mypy
        :versions: 2.0.0rc3

        修改mypy插件以适应SQLAlchemy 1.4的针对问题＃236 sqlalchemy2-stubs可能正在进行的某些潜在更改。这些更改与SQLAlchemy 2.0保持同步。这些更改还与旧版sqlalchemy2-stubs兼容。

    .. change::
        :tags: bug, mypy
        :tickets: 9102
        :versions: 2.0.0rc3

        修复了mypy插件中的崩溃问题，该问题可能会在1.4和2.0版本中发生，如果引用表达式的装饰器用了超过两个组件（例如 @Backend.mapper_registry.mapped），则不理。该场景现在被忽略；在使用插件时，必须将装饰器表达式用于两个组件（即 @reg.mapped）。

    .. change::
        :tags: bug, sql
        :tickets: 9506

        修正了使用 :meth:`_sql.Operators.op` 自定义操作符函数会不会生成适当缓存键的关键SQL缓存问题，导致降低了SQL缓存的有效性。


.. changelog::
    :version: 1.4.46
    :released: 2023年1月3日

    .. change::
        :tags: bug, engine
        :tickets: 8974
        :versions: 2.0.0rc1

        修复了连接池中的一个长期的竞争条件问题，该问题在与事件修补程序 /闪影装置的monkeypatching方案一起使用时，会与使用eventlet / gevent Timeout条件的情况下中断的连接池检出失败清除失败状态，导致潜在地泄漏底层的连接记录，有时也会泄漏数据库连接本身，使池处于无效状态，其输入不可达。这个问题在SQLAlchemy 1.2中首次被识别和修复，用于：ticket:`4225`，但在该修复中检测到的故障模式未适应 BaseException，而不是 Exception，这阻止了eventlet / gevent Timeout 从被捕获，而在原始连接恢复到连接池过程中发出“自动提交”信号时。此外，在初始池连接中的Block也已经被识别，并使用“BaseException”->“清除连接失败”块加强了这种情况。非常感谢Github用户@niklaus在最终识别此复杂问题方面的耐心长期帮助。

    .. change::
        :tags: bug, postgresql
        :tickets: 9023
        :versions: 2.0.0rc1

        修复了问题，即当使用  :paramref:`_postgresql.Insert.on_conflict_do_update.constraint`  参数时，该参数将接受   :class:` .Index`  对象，但是不会将其分解成其单个索引表达式，而是将其名称呈现为 ON CONSTRAINT子句中不被PostgreSQL接受；唯一或排除约束名字只接受“约束名”形式。该参数继续接受索引，但现在将其扩展为其组件表达式进行渲染。

    .. change::
        :tags: bug, general
        :tickets: 8995
        :versions: 2.0.0rc1

        修复了一个回归问题，即基础 compat模块在调用“platform.architecture（）”以检测某些系统属性时，结果会通过系统级“file”调用产生过度广泛的系统调用，这在某些情况下是不可用的否则 go.shell安全的环境配置。

    .. change::
        :tags: usecase, postgresql
        :tickets: 8393
        :versions: 2.0.0b5

        添加了 PostgreSQL 类型 ``MACADDR8``。拉取请求的好处来自Asim Farooq。

    .. change::
        :tags: bug, sqlite
        :tickets: 8969
        :versions: 2.0.0b5

        修复了引入在1.4.45中添加了对SQLite部分索引的反射支持的回归问题，其中“index_list” pragma命令在非常旧的SQLite版本中(可能在3.8.9之前)不会返回当前期望的列数，导致在反射表和索引时引发异常。

    .. change::
        :tags: bug, tests
        :versions: 2.0.0rc1

        修复了tox.ini文件中的问题，其中对于“passenv”的格式，放在tox 4.0系列中的更改会导致tox无法正常工作，特别是从tox 4.0.6开始引发错误。

    .. change::
        :tags: bug, tests
        :tickets: 9002
        :versions: 2.0.0rc1

        为第三方方言添加了新的排除规则，称为“unusual_column_name_characters”，即使该名称经过正确的引用，也不能支持例如点、斜杠或百分号等不寻常字符的列名字，例如在引用名称时使用  :meth:`~_schema.Table.c`  或类似的构造。即使正确引用这些名称，第三方方言也不支持这个功能。


    .. change::
        :tags: bug, sql
        :tickets: 9009
        :versions: 2.0.0b5

        添加  :paramref:`.FunctionElement.column_valued.joins_implicitly`  参数，该参数有助于防止在使用表值函数或列值函数时出现“笛卡尔积”警告。该参数已经在  :ticket:` 7845`  中为  :meth:`.FunctionElement.table_valued` .FunctionElement.column_valued` 也未添加。


    .. change::
        :tags: change, general
        :tickets: 8983

        现在，在第一次发出任何SQLAlchemy 2.0弃用警告之前，运行时会发出新的弃用“超级警告”，但未设置“SQLALCHEMY_WARN_20”环境变量。最多发出警告一次，然后设置布尔值以防止第二次发出警告。

        这个弃用警告意在通知可能没有在其要求文件的约束中设置适当约束的用户阻止惊喜SQLAlchemy 2.0升级，并且警示SQLAlchemy 2.0升级过程是可用的，因为第一个完整的2.0版本很快就会发布。可以通过将环境变量“SQLALCHEMY_SILENCE_UBER_WARNING”设置为“1”来关闭弃用警告。

        .. seealso::

              :ref:`migration_20_toplevel` 

    .. change::
        :tags: bug, orm
        :tickets: 9033
        :versions: 2.0.0rc1

        修复了   :class:`_dml.Update`  和   :class:` _dml.Delete`  等 DML 语句的内部 SQL 遍历中可能发生的一系列问题，其中将在按照ORM Update/Delete功能使用lambda语句时导致多个潜在问题。
        
    .. change::
        :tags: bug, sql
        :tickets: 8989
        :versions: 2.0.0b5

        修复了使用包括  :meth:`_types.TypeEngine.bind_expression` ` regexp_replace()` `函数，在用于  :ticket:`4123` .CTE` 结构的“嵌套”功能中，以及使用Oracle的  :meth:`.FunctionElement.column_valued`  方法形成的可选择表。

    .. change::
        :tags: bug, oracle
        :tickets: 8945
        :versions: 2.0.0rc1

        修复了Oracle编译器中语法 :meth:`.FunctionElement.column_valued` 的语法不正确，将名称“COLUMN_VALUE”呈现为未正确限定源表的名称。

    .. change::
        :tags: bug, engine
        :tickets: 8963
        :versions: 2.0.0rc1

        修复了当使用   :func:`_sql.text`  或  :meth:` _engine.Connection.exec_driver_sql`  的文本SQL时，  :meth:`_engine.Result.freeze`  方法将不起作用的问题。


.. changelog::
    :version: 1.4.45
    :released: 2022年12月10日

    .. change::
        :tags: bug, orm
        :tickets: 8862
        :versions: 2.0.0rc1

        修复了  :meth:`_orm.Session.merge`  无法保留使用  :paramref:` _orm.relationship.viewonly`  参数指示的事实的当前加载的关系属性的问题，从而破坏了使用  :meth:`_orm.Session.merge`  从缓存和其他类似技术中获取完全加载的对象的策略。相关的修复  :meth:` _orm.Session.merge`  英语和持有具有已加载的关系的对象的问题的陈述，但是  :paramref:`_orm.Session.merge.load`  参数保持其默认值为True。

        总体而言，这是对1.4系列中引入的更改进行行为调整的结果，该更改自票号：4994起将“合并”从默认应用于“仅视图”的关系的级联集中取出。由于“视图”关系在任何情况下都不会被持久化，因此允许它们的内容传输不会影响目标对象的持久化行为。这使  :meth:`_orm.Session.merge`  能够正确适应其其中一个用例，即将从其他地方加载的完全加载的对象添加到   :class:` .Session`  中，通常是为了从缓存中恢复。

    .. change::
        :tags: bug, orm
        :tickets: 8881
        :versions: 2.0.0rc1

        修复了在使用   :func:`_orm.with_expression`  中，由来自封闭SELECT的列组成的表达式，在某些情况下，在使用与  :meth:` .Select.join`  结合使用时  :meth:`.Select.select_from` ，以及当  :meth:` .Select.join_from`  时将导致   :func:`_orm.with_loader_criteria`  功能以及需要IN标准的单表继承查询不起作用，如果查询的列子句未明确包括JOIN的左侧实体，则正确实体现在被转移给生成的内部   :class:` .Join`  对象，以便针对左侧实体的判断被正确添加。


    .. change::
        :tags: bug, sqlite
        :tickets: 8866

        后退一个针对SQLite附加模式中唯一约束的修复，该修复已在2.0中作为：ticket:`4379` 的一小部分发布。之前，唯一约束在附加模式中会被SQLite忽略反射。拉取请求的好处来自Michael Gorven。

    .. change::
        :tags: bug, asyncio
        :tickets: 8952
        :versions: 2.0.0rc1

        从   :class:`_asyncio.AsyncResult` ` merge()``方法。此方法从未起作用，并且错误地包含在中   :class:`_asyncio.AsyncResult` 。

    .. change::
        :tags: bug, oracle
        :tickets: 8708
        :versions: 2.0.0b4

        修复了错误的重复调试，即包含类似命名的数据库列的绑定参数名称，包括从具有类似名称的数据库列自动生成的参数名称，不包含Oracle通常需要用引号引用的字符时，当在Oracle方言的“展开参数”与Oracle一起使用时，导致执行错误。有关Oracle方言的绑定参数的通常“引号”未与“展开参数”体系结构一起使用，因此使用了大量字符的转义，现在使用了一组仅适用于Oracle的字符/转义序列。

    .. change::
        :tags: bug, orm
        :tickets: 8721
        :versions: 2.0.0b3

        修复了   :class:`.Select`  构件中的问题，其中使用  :meth:` .Select.select_from`  与  :meth:`.Select.join`  的组合，以及使用  :meth:` .Select.join_from`  时，如果查询的列子句未显式包括JOIN的左侧实体时，会导致   :func:`_orm.with_loader_criteria`  选项，以及在单表继承查询中所需的IN标准不呈现。传递到查询的正确实体现在转移到生成的内部   :class:` .Join`  对象中，以便正确添加对左侧实体的标准。

    .. change::
        :tags: bug, mssql
        :tickets: 8714
        :versions: 2.0.0b3

        修复了  :meth:`.Inspector.has_table`  在使用SQL Server方言针对临时表时会失败的问题，在某些Azure变体中，在返回其DBAPI连接到连接池时，可能由于不支持这些服务器版本上的一些不必要的信息模式查询而失败。拉取请求的好处来自Mike Barry。

    .. change::
        :tags: bug, orm
        :tickets: 8711
        :versions: 2.0.0b3

        现在，在使用   :func:`_orm.with_loader_criteria`  选项作为添加到特定“加载程序路径”(例如，在  :meth:` .Load.options`  中使用此选项)的加载程序选项时，将引发有信息的异常，该异常仅指定要将   :func:`_orm.with_loader_criteria`  作为顶级加载程序选项使用。以前将生成内部错误。

    .. change::
        :tags: bug, oracle
        :tickets: 8744
        :versions: 2.0.0b3

        修复了在第一次连接时查询“nls_session_parameters”视图以获取默认小数点字符的问题可能会不可用，具体取决于Oracle连接模式，并因此引发错误。检测小数点的方法已经简化为直接测试十进制值，而不是读取系统视图，这在任何后端/驱动程序上都有效。

    .. change::
        :tags: bug, orm
        :tickets: 8753
        :versions: 2.0.0b3

        改进了  :meth:`_orm.Session.get`  中作为命名字典的“字典模式”，因此可以指示引用主键属性名的同义字名称。

    .. change::
        :tags: bug, engine
        :tickets: 8717
        :versions: 2.0.0b3

        修复了  :meth:`.PoolEvents.reset`  事件钩子将无法在将   :class:` _engine.Connection`  关闭并正在将其DBAPI连接返回到连接池的过程中，在所有情况下都被调用的问题。

        场景是当  :class:`_engine.Connection` ` .rollback（）``时，如果其指示连接池放弃执行自己的“重置”以节省附加 方法调用。但这将防止使用自定义重置方案从该挂钩中使用，因为这些挂钩本质上不仅仅是调用 ``.rollback()``，而且需要在所有情况下调用。这是版本1.4中出现的回归。

        对于版本1.4，  :meth:`.PoolEvents.checkin`   仍然是用于自定义“重置”实现的可行事件挂钩的替代方法。版本2.0将具有改进版本的  :meth:` .PoolEvents.reset`  ，该版本适用于多种情况，例如终止asyncio连接，并提供有关重置的上下文信息，以允许用于不同的重置方案在不同的情况下进行反应。


    .. change::
        :tags: bug, orm
        :tickets: 8704
        :versions: 2.0.0b3

        修复了面向表达式的WHERE标准的反射中玩具继承映射程序的“selectin_polymorphic”加载将不起作用的问题，如果  :paramref:`_orm.Mapper.polymorphic_on`  参数所引用的SQL表达式未直接在类上进行映射。

    .. change::
        :tags: bug, engine
        :tickets: 8710
        :versions: 2.0.0b3

        当在字典模式下使用  :meth:`_orm.Session.get`  时，改进了字典模式，因此引用主键属性名称的同义字名称可以在命名字典中指示。.. _changelog-1.4.42:

.. _change-8710:

Version 1.4.42

Released on October 16, 2022, this version of SQLAlchemy includes the following changes:

- A previous version had caused the iterator used during Query.yield_per to close unexpectedly, leading to issues with server-side cursors. The new version includes a catch for GeneratorExit in the iterator method, allowing the result object to be closed where the iterator is interrupted. The new version includes a .close method for all Result implementations.
- The previous version caused Inspector.has_table to return False erroneously for views in SQL Server. The new version fixes this issue and ensures the method continues working per views specifications.
- An issue with literal_column within Select construct and other occurrences where "anonymized labels" may be generated has also been resolved.

本文档介绍了SQLAlchemy的一些变化和修复，版本号为1.4.31和1.4.32。主要内容如下：

版本1.4.31中:

-修复了 Postgresql 的枚举类型在出现空数组时无法正确处理的 bug。

-修复了 Session.bulk_save_objects 方法中的排序问题。

-修复了 asyncmy 方言部分功能的问题。

-增加了 MSSQL 中 VARBINARY(max) 使用 FILESTREAM 的支持。

版本1.4.32中:

-修复了数列 bug。

-修复了 enum 类型 bug。

-对 MySQL 的 SET 类型和 enum 类型进行修复，以在 Alembic 自动生成时能够正确呈现所有可选参数。

-修复了 SQLite 中反射具有下划线名称的检查约束时的错误。

-修复了全限定类名包含错误路径 token 的时候抛出的随机异常情况。

-修复了 Oracle 方言中与绑定参数有关的列名的问题，这些列名需要在 Oracle 中使用引号。 

-对 PostgreSQL 的空数组进行修复。

-修复了 ORM 中使用类名排序而出现的异常问题。

-修复了 asyncmy 方言异步操作的问题。

-增加了新参数以支持 PostgreSQL 中的“不有效”短语的生成。

-增加了日志处理的新功能，包括 Engine 和 Connection，这样可以建立适当的堆栈级别参数，使 Python 日志标记“funcName”和“lineno”在使用自定义日志格式程序时报告正确的信息。该功能支持 Python 3.8 及以上版本。

-修复了 ORM 中 Insert 处理问题，此问题会导致 primary key 值丢失。

-SQLAlchemy 对 Count 的表达式对象支持 SQL 函数，如 Sum，Avg 等。

-修改了 DeclarativeMeta 元类。这样，自定义基类中加入新的属性应该可以在 ORM 中使用。.. 更新日志::
    :version: 1.4.30
    :released: 2022年1月19日

    .. change::
        :tags: usecase, asyncio
        :tickets: 7580

        新增  :meth:`.AdaptedConnection.run_async`  方法到 DBAPI 的连接接口中，允许在不能使用 ` `await`` 关键字的同步风格函数中直接对底层“driver”连接调用方法，例如 SQLAlchemy 事件处理函数。该方法类似于  :meth:`_asyncio.AsyncConnection.run_sync`  方法，它将异步风格调用转换为同步风格调用。此方法对于需要在驱动程序连接首次创建时调用驱动程序连接上的可等待方法的连接池连接处理程序等方面很有用。

        .. seealso::

              :ref:`asyncio_events_run_async` 


    .. change::
        :tags: bug, orm
        :tickets: 7507

        修复了多级继承中的 joined-inheritance 加载附加属性功能中的问题，其中包含不包含任何列的中间表不会被包括在联接的表中，而是将这些表链接到它们的主键标识符。虽然这样可以正常工作，但在 1.4 开始时会产生笛卡儿积编译器警告。该逻辑已更改，以使这些中间表始终被包括。虽然这会在查询中包括不必要的其他表，但这仅在深度为 3 级或更深层次的具有没有非主键列的中间表的高度不寻常的情况下发生，因此预计性能影响将是可以忽略的。

    .. change::
        :tags: bug, orm
        :tickets: 7579

        修复了在同一类别中对  :meth:`_orm.registry.map_imperatively`  方法进行多次调用会产生意外错误而不是信息性错误的问题，而错误的行为与   :func:` _orm.mapper`  函数不同，后者确实会报告信息性消息。

    .. change::
        :tags: bug, sql, postgresql
        :tickets: 7537

        为从 Python 字面量生成``TypeEngine`` 实现的系统添加了第二级调整类型的规则，以便带或不带 tzinfo 的 Python datetime 可以将 ``timezone=True`` 参数设置为返回的   :class:`.DateTime`  对象，以及   :class:` .Time` 。这有助于某些往返于特定类型的 PostgreSQL 方言（例如 asyncpg、psycopg3（仅限 2.0））的情况。

    .. change::
        :tags: bug, postgresql, asyncpg
        :tickets: 7537

        改进了用于异步处理 TIME WITH TIMEZONE 的 asyncpg 处理支持，该特性尚未完全实现。

    .. change::
        :tags: usecase, postgresql
        :tickets: 7561

        为   :class:`.postgresql.UUID`  数据类型添加了字符串渲染功能，因此，使用此类型的“literal_binds”语句的字符串表示将为 PostgreSQL 后端呈现适当的字符串值。拉取请求由 José Duarte 提供。

    .. change::
        :tags: bug, orm, asyncio
        :tickets: 7524

        为   :class:`_asyncio.AsyncSession`  类添加了缺少的  :meth:` _asyncio.AsyncSession.invalidate`  方法。


    .. change::
        :tags: bug, orm, regression
        :tickets: 7557

        修复了在 1.4.23 中出现的 ORM 回归问题，该问题可能会导致在某些情况下处理加载器选项时出现问题，特别是在结合“polymorphic_load =“selectin””选项的连接表继承中使用时，以及加载关系时懒洋洋的加载，导致“TypeError”。


    .. change::
        :tags: bug, mypy
        :tickets: 7321

        修复了在运行 id 守护程序模式时由内部 mypy“Var”实例的缺少属性引起的 Mypy 崩溃。

    .. change::
        :tags: change, mysql
        :tickets: 7518

        在 MySQL 和 MariaDB 方言初始化中，使用等效的 ``SELECT @@variable`` SQL 语句替换 ``SHOW VARIABLES LIKE`` 语句。这应该避免由“SHOW VARIABLES”引起的互斥量争用，从而提高了初始化性能。

    .. change::
        :tags: bug, orm, regression
        :tickets: 7576

        修复了 ORM 回归问题，其中针对现有   :func:`_orm.aliased`  建构的   :func:` _orm.aliased`  方法调用将在允许情况下失败，并且如果将原始   :func:`_orm.aliased`  构造物仅针对现在正在替换的表，则原始   :func:` _orm.aliased`  构造物将被忽略。如果没有可选择的参数构建   :func:`_orm.aliased`  并在   :func:` _orm.aliased`  中使用   :func:`_orm.aliased`  对象的嵌套行为将保留不变。

        如果原始构造就是针对子查询的，而该子查询又引用内部   :func:`_orm.aliased`  对象，则外部   :func:` _orm.aliased`  对象会继续嵌套。这是一个相对较新的 1.4 功能，可以帮助适应之前由弃用的 ``Query.from_self()`` 方法提供的用例。

    .. change::
        :tags: bug, orm
        :tickets: 7514

        修复了  :meth:`_sql.Select.correlate_except`  方法中的问题，当在 ORM 上下文中使用时（即将 ORM 实体作为 FROM 子句传递）且传递给它 None 值或无参数时，不会关联任何元素，而不是像在使用纯核心结构时一样让所有 FROM 元素都被视为“相关”，而是发生这种情况，当仅使用“from_statement”的情况下，通常会导致出现错误。

    .. change::
        :tags: bug, orm, regression
        :tickets: 7505

        修复了 1.3 以来的“子查询加载程序”加载器策略将在使用  :meth:`_orm.Query.from_statement`  或  :meth:` _sql.Select.from_statement`  时针对不可修改的查询失败的回归问题。由于 subqueryload 需要修改原始语句，因此它与“from_statement”用例不兼容，特别是对于使用   :func:`_sql.text`  构造的语句。现在，行为相当于 1.3 和以前，即加载器策略会静默降级为不用于这种语句，通常会回退到使用 lazyload 策略。


    .. change::
        :tags: bug, reflection, postgresql, mssql
        :tickets: 7382

        修复了反映索引时将``include_columns`` 作为反射索引字典中 ``dialect_options`` 条目的一部分报告的问题，从而使得从反射-> 创建的往返完全。包含列继续以 ``include_columns`` 键存在，以向后兼容。

    .. change::
        :tags: bug, mysql
        :tickets: 7567

        从 asyncmy 方言中删除对 PyMySQL 的不必要依赖。拉取请求出自 long2ice。

    .. change::
        :tags: bug, postgresql
        :tickets: 7418

        修复了对需要转义字符的枚举值数组的处理。

    .. change::
        :tags: bug, sql
        :tickets: 7032

        对于传递给 SQL 构造函数的方法对象，现在会在出现信息性错误时启用详细错误消息。以前，当传递这样一个可调用对象时（如处理方法链接的 SQL 结构的常见书写错误），它们将被解释为要在编译时调用的“lambda SQL”目标，这将导致静默的失败。由于此功能不应与方法一起使用，因此现在会拒绝方法对象。

.. changelog::
    :version: 1.4.29
    :released: 2021年12月22日

    .. change::
        :tags: usecase, asyncio
        :tickets: 7301

        添加   :func:`_asyncio.async_engine_config`  函数，以从配置字典创建异步引擎。除此之外，它的行为与   :func:` _sa.engine_from_config`  相同。

    .. change::
        :tags: bug, orm
        :tickets: 7489

        修复了新的“loader criteria”方法  :meth:`_orm.PropComparator.and_`  中的问题，该方法在使用诸如   :func:` _orm.selectinload`  这样的加载器策略并且列位于子查询对象的 ``.c.`` 集合中，其中子查询将在执行时被动态添加到语句的 FROM 子句中，将会受到 SQL 语句缓存中参数值的过时参数问题，因为加载器策略用于在执行时替换参数的过程无法在接收到此格式的子查询时进行适应。

    .. change::
        :tags: bug, orm
        :tickets: 7491

        修复了在使用一个加载器策略中的  :meth:`_orm.with_loader_criteria`  功能或使用该策略修改一个 loader 策略中的  :meth:` _orm.PropComparator.and_`  方法时，在涉及到引用相同的加载器功要引起递归溢出的问题。增加了一个递归检查来容纳这种情况。

    .. change::
        :tags: bug, orm, mypy
        :tickets: 7462, 7368

        修复了由   :func:`_orm.as_declarative`  生成的声明基类的 ` `__class_getitem__()`` 方法不会从给定的超类复制会导致无法访问的类属性（如 ``__table__``）的问题，其中应在类层次结构中使用 ``Generic[T]`` 样式的类型声明。这是继  :ticket:`7368`  中首次添加 ` `__class_getitem__()`` 的基本功能延续而来。拉取请求由 Kai Mueller 提供。

    .. change::
        :tags: bug, mypy
        :tickets: 7496

        修复了由 mypy 0.930 版本引入的额外内部检查来检查“命名类型”的格式时导致的崩溃问题，需要确保命名类型应完全限定并可定位，因为其上使用的符号诸如 ``__builtins__`` 和其他不可定位或未经常规限定的名称将不再引起任何断言。


    .. change::
        :tags: bug, engine
        :tickets: 7432

        修正了在试图在   :class:`_result.Row`  类中的一个属性上写入时发生 ` `AttributeError`` 的错误消息，这可能会产生歧导性，因为之前的消息声称列不存在。

    .. change::
        :tags: bug, mariadb
        :tickets: 7457

        修复了“is_disconnect”检查的错误类，这在 mariadbconnector 方言中会导致断开连接，例如由常见的 MySQL/MariaDB 错误代码（如 2006）引起的断开，DBAPI 目前似乎使用 ``mariadb.InterfaceError`` 异常类标记此类断开连接错误，而这已添加到检查类列表中。


    .. change::
        :tags: bug, orm, mypy
        :tickets: 7368

        修复了   :func:`_orm.as_declarative`  装饰器和类似函数生成声明基类时不会将 ` `__class_getitem__()`` 方法从给定超类复制到给定超类的问题，这导致了通过与 ``Base`` 类结合使用 pep-484 泛型的使用。拉取请求由 Kai Mueller 提供。

    .. change::
        :tags: usecase, engine
        :tickets: 7400

        为   :class:`_url.URL`  类添加了对 ` `copy()`` 和 ``deepcopy()`` 的支持。拉取请求由 Tom Ritchford 提供。

    .. change::
        :tags: bug, orm
        :tickets: 7425

        修复了使用 ORM 多态加载的相关类别的代理属性（例如   :func:`_orm.relationship` ）时，如果与其关联的类同样使用 ORM 多态加载，则无法直接在第二个加载器选项中使用   :func:` _orm.aliased`  构造与目标上的属性进行定位，例如 ``selectinload(A.aliased_bs).joinedload(aliased_b.cs)``，而不需要在路径的前面元素上使用  :meth:`_orm.PropComparator.of_type`  进行显式的验证。此外，直接针对非别名类别的目标进行定位将被接受（不适当），但将静默失败，例如 ` `selectinload(A.aliased_bs).joinedload(B.cs)``；现在，它会引发引用类型不匹配的错误。


    .. change::
        :tags: bug, schema
        :tickets: 7295

        修复了   :class:`.Table`  在将  :paramref:` .Table.implicit_returning`  参数与  :paramref:`.Table.extend_existing`  一起传递以扩展现有   :class:` .Table`  时未能正确处理该参数的问题。

    .. change::
        :tags: bug, postgresql, asyncpg
        :tickets: 7283

        将 asyncpg 方言绑定到“float”PostgreSQL 类型而非“numeric”类型上，以便可以容纳值 ``float(inf)``。为测试套件提供了对“inf”值的持久性支持。


    .. change::
        :tags: bug, orm
        :tickets: 7274
        :versions: 2.0.0b1

        修复了  :meth:`_engine.CursorResult.fetchmany`  方法在结果完全耗尽时无法自动关闭服务器端游标（即当使用 ` `stream_results`` 或 ``yield_per`` 来自于 Core 或 ORM 结果时），这方法在未来的   :class:`_engine.Connection`  对象中无法接受非字典映射对象（例如 SQLAlchemy 自己的   :class:` .RowMapping`  或其他 ``abc.collections.Mapping`` 对象）作为参数字典时，并且会引发异常。

    .. change::
        :tags: bug, orm
        :tickets: 7274
        :versions: 2.0.0b1

        所有   :class:`_result.Result`  对象现在都会在硬关闭后抛出   :class:` _exc.ResourceClosedError` ，包括非常规   :class:`.FilteredResultSet` ，并在这种情况下返回空结果或根本不会“软关闭”并将继续从底层光标中产生值。 ：meth:` _result.Result.first` 和  :meth:`_result.Result.scalar`  方法使用的“硬关闭”后的行为。最常用的基于   :class:` _engine.CursorResult`  的 Core 语句执行。因此，这种行为并不新颖。但是，该变化已扩展为正确适应 2.0 样式 ORM 查询时返回的过滤结果对象。使得它们能在 "yield_per" 执行选项中调用底层的  :meth:`_engine.CursorResult.close`  方法以在接收所有 ORM 结果之前关闭服务器端游标。这已经适用于 Core 结果集的最常见类别，但是此更改使其适用于 2.0 样式 ORM 结果。

        作为此更改的一部分，还向基础   :class:`_result.Result`  类添加了  :meth:` _result.Result.close` ，并为 ORM 使用的过滤结果实现了该方法，以便在使用 ``yield_per`` 执行选项处理 ORM 结果时，可以调用底层的  :meth:`_engine.CursorResult.close`  方法以在获取所有 ORM 结果之前关闭服务器端游标。这一直是对于 Core 结果集而言是可用的，但是更改使其对 2.0 样式的 ORM 结果也可用。


    .. change::
        :tags: bug, mysql
        :tickets: 7281
        :versions: 2.0.0b1

        修复了 MySQL  :meth:`_mysql.Insert.on_duplicate_key_update`  中的问题，该问题在值表达式中使用表达式时将呈现错误的列名称。拉取请求由 Cristian Sabaila 提供。

    .. change::
        :tags: bug, sql, regression
        :tickets: 7319

        扩展了 ORM 查询返回的行对象，这些行对象现在是标准的   :class:`_sql.Row`  对象，因此  :meth:` _sql.ColumnOperators.in_`  运算符将不再将其解释为单个绑定参数，并且而是将元组值分成单个绑定的参数，然后将其传递给驱动程序，从而导致失败。根据现在对“扩展 IN”系统的更改，如果表达式已经是   :class:`.TupleType`  类型，则如果应如此，则相应地处理值以及考虑值。如果在不带有类型信息的语句中使用“tuple-in”，例如当用于无类型语句时，例如文本语句使用无类型，但是包含非 str 或 bytes 的 ` `collections.abc.Sequence`` 的元组值将被检测到，就像在测试 ``Sequence`` 时一样。


    .. change::
        :tags: usecase, sql


版本：1.4.26
发布时间：2021年10月19日

Added   :class:`.TupleType`  to the top level ` `sqlalchemy`` import namespace.

解决bug，如下：
Fixed issue where using the feature of using a string label for ordering or grouping described at   :ref:`tutorial_order_by_label`  would fail to function correctly if used on a   :class:` .CTE`  construct, when the CTE were embedded inside of an enclosing   :class:`_sql.Select`  statement that itself was set up as a scalar subquery.
(Fixed 1.4 regression where  :meth:`_orm.Query.filter_by`  would not function correctly on a   :class:` _orm.Query`  that was produced from  :meth:`_orm.Query.union` ,  :meth:` _orm.Query.from_self`  or similar.)
Fixed issue where deferred polymorphic loading of attributes from a joined-table inheritance subclass would fail to populate the attribute correctly if the   :func:`_orm.load_only`  option were used to originally exclude that attribute, in the case where the load_only were descending from a relationship loader option.  The fix allows that other valid options such as ` `defer(..., raiseload=True)`` etc. still function as expected.
(特定于postgresql和asyncpg使用情况的问题)
Added overridable methods ``PGDialect_asyncpg.setup_asyncpg_json_codec`` and ``PGDialect_asyncpg.setup_asyncpg_jsonb_codec`` codec, which handle the required task of registering JSON/JSONB codecs for these datatypes when using asyncpg. The change is that methods are broken out as individual, overridable methods to support third party dialects that need to alter or disable how these particular codecs are set up.
Fixed issue in future   :class:`_engine.Engine`  where calling upon  :meth:` _engine.Engine.begin`  and entering the context manager would not close the connection if the actual BEGIN operation failed for some reason, such as an event handler raising an exception
(Adjusted the compiler's generation of "post compile" symbols including those used for "expanding IN" as well as for the "schema translate map" to not be based directly on plain bracketed strings with underscores, as this conflicts directly with SQL Server's quoting format of also using brackets, which produces false matches when the compiler replaces "post compile" and "schema translate" symbols.
Improve array handling when using PostgreSQL with the pg8000 dialect.
Fixed 1.4 regression where  :meth:`_orm.Query.filter_by`  would not function correctly when  :meth:` _orm.Query.join`  were joined to an entity which made use of  :meth:`_orm.PropComparator.of_type`  to specify an aliased version of the target entity. The issue also applies to future style ORM queries constructed with   :func:` _sql.select` .
(Fixed regression where the   :func:`_sql.text`  construct would no longer be accepted as a target case in the "whens" list within a   :func:` _sql.case`  construct.)

新增功能：（罗列的地方都是新增功能）
Added   :class:`.TupleType`  to the top level ` `sqlalchemy`` import namespace.

版本：1.4.25
发布时间：2021年9月22日

解决bug，如下：
Fixed regression due to  :ticket:`7024`  where the reorganization of the "platform machine" names used by the ` `greenlet`` dependency mis-spelled "aarch64" and additionally omitted uppercase "AMD64" as is needed for Windows machines.

版本：1.4.24
发布时间：2021年9月22日

解决bug，如下：
Fixed a bug in  :meth:`_asyncio.AsyncSession.execute`  and  :meth:` _asyncio.AsyncSession.stream`  that required ``execution_options`` to be an instance of ``immutabledict`` when defined. It now correctly accepts any mapping.
Improve the interface used by adapted drivers, like the asyncio ones, to access the actual connection object returned by the driver.
Account for the  :paramref:`_sql.table.schema`  parameter passed to the   :func:` _sql.table`  construct.

新增功能：（罗列的地方都是新增功能）
None

版本：1.4.23
发布时间：2021年8月6日

解决bug，如下：
Fixed issue in ORM joined eager loading where circular mode would incorrectly get enabled on a single-table inheritance relationship, when using the query option for the relationship to set loader options like   :func:`_orm.subqueryload`  or   :func:` _orm.noload`  upon the target subclass.
Corrected mis-spelling of the word "nonexistent" in error message when a row count is not returned by an UPDATE or DELETE statement.
Fixed issue in PGDialect where a subquery with a UNION would not render correctly if the subquery were without labels.
(Made the naming of base casedict and typeddict for versatility.)
Fixed issue where a CPython 3.10 compatibility issue was introduced regarding the use of code objects in collections.deque() and perhaps other places where the result would have circular references.
Added clarity of documentation surrounding JOIN and ON conditions in ORM queries.

新增功能：（罗列的地方都是新增功能）
None.. change::
    :tags: bug, SQLAlchemy ORM, mypy
    :tickets: N/A

    修复了mypy插件的一个问题，当映射类依赖于一个来自超类的`__tablename__`例程时，
    mixin上的列将不会被正确解释，出现了问题。

.. change::
    :tags: bug, PostgreSQL, SQLAlchemy ORM
    :tickets: 6106

    PostgreSQL原生的 :class:`_postgresql.ENUM` 数据类型，
    因此不应使用``native_enum=False``标志。 
    如果将该标志传递给 :class:`_postgresql.ENUM` 数据类型，
    则会忽略该标志并发出警告；以前，该标志会导致类型对象不能正确运行。

.. change::
    :tags: bug, SQLAlchemy ORM
    :tickets: 7036

    修复了一个问题，当将两个“INSERT ... FROM SELECT”语句同时对应起来时，
    会丢失两个独立的SELECT语句的跟踪，导致错误的SQL。

.. change::
    :tags: asyncio, bug, SQLAlchemy ORM
    :tickets: 6746

    弃用使用Asyncio驱动程序的  :class:`_orm.scoped_session` 。
    在使用Asyncio时，应改用  :class:`_asyncio.async_scoped_session` 。

.. change::
    :tags: bug, platform, SQLAlchemy ORM
    :tickets: 7024

    进一步调整了setup.cfg中“greenlet”包指示符，
    以使用一长串“or”表达式，这样将``platform_machine``
    与特定标识符进行比较仅会匹配完整字符串。

.. change::
    :tags: bug, SQLite, SQLAlchemy ORM
    :tickets: N/A

    修复了一个问题，在pysqlite驱动程序上，
    SQLite无效的隔离级别的错误消息未能指示“AUTOCOMMIT”是有效的隔离级别之一。

.. change::
    :tags: SQLAlchemy ORM, bug, SQL
    :tickets: 7060

    修复了一个问题，当将ORM列表达式用作传递给 
     :meth:`_sql.Insert.values`  的字典列表的键时，会将其正确处理为正确的列表达式。

.. change::
    :tags: asyncio, SQLAlchemy ORM, usecase
    :tickets: 6746

      :class:`_asyncio.AsyncSession` 。
    可以使用  :paramref:`~.AsyncSession.sync_session_class`  参数传递自定义的` `Session``类，
    或通过子类化``AsyncSession``并指定自定义的  :attr:`~.AsyncSession.sync_session_class`  来指定。

.. change::
    :tags: bug, Oracle, SQLAlchemy ORM, performance
    :tickets: 4486

    向反射查询针对Oracle系统视图（如ALL_TABLES，ALL_TAB_CONSTRAINTS等）的DDL-name参数添加CAST（VARCHAR2（128））
    ，以便更好地针对这些列上的索引执行操作，因为在Python中使用Unicode用于字符串时，
    这些列以前将会被隐式处理为NVARCHAR2;这些列在所有Oracle版本中的文档中均被记录为VARCHAR2，长度根据服务器版本而异，从30到128个字符不等。
    另外，针对Oracle数据库启用了支持Unicode命名的DDL结构的测试支持。

.. changelog::
    :version: 1.4.23
    :released: August 18, 2021

    .. change::
        :tags: bug, SQL, SQLAlchemy ORM
        :tickets: 6752

        修复了在版本1.4.21 /  :ticket:`6752`  功能中，如何处理来自超类的` __tablename__`例程的一列的问题。

    .. change::
        :tags: bug, PostgreSQL, SQLAlchemy ORM
        :tickets: 6752

        PostgreSQL的  :class:`_postgresql.ENUM` ` native_enum=False``标志。
        如果将该标志传递给 :class:`_postgresql.ENUM` 数据类型，则忽略该标志并发出警告。
        以前，该标志会导致类型对象不能正确运行。

    .. change::
        :tags: bug, ORM, SQLAlchemy ORM, SQL
        :tickets: 7036

        修复了与新  :meth:`_sql.HasCTE.add_cte`  功能相关的问题，
        在使用此功能时，"INSERT .. FROM SELECT"语句对应失败，丢失了 SELECT 语句的跟踪。

    .. change::
        :tags: asyncio, SQLAlchemy ORM, bug
        :tickets: 6746

        弃用使用Asyncio驱动程序的  :class:`_orm.scoped_session` 。
        在使用Asyncio时，应改用  :class:`_asyncio.async_scoped_session` 。

    .. change::
        :tags: bug, platform, SQLAlchemy ORM
        :tickets: 7024

        进一步调整了setup.cfg中“greenlet”包指示符，
        以使用一长串“or”表达式，这样将``platform_machine``
        与特定标识符进行比较仅会匹配完整字符串。

    .. change::
        :tags: SQLAlchemy ORM, bug, SQLite
        :tickets: N/A

        修复了一个问题，在pysqlite驱动程序上，
        SQLite无效的隔离级别的错误消息未能指示“AUTOCOMMIT”是有效的隔离级别之一。

    .. change::
        :tags: SQLAlchemy ORM, bug, SQL
        :tickets: 7060

        修复了一个ORM列表达式用作“多值插入”的  :meth:`_sql.Insert.values`  的字典列表中的键时无法正确处理为正确列表达式的问题。

    .. change::
        :tags: asyncio, SQLAlchemy ORM, usecase
        :tickets: 6746

          :class:`_asyncio.AsyncSession` 。
        可以使用  :paramref:`~.AsyncSession.sync_session_class`  参数传递自定义的` `Session``类，
        或通过子类化``AsyncSession``并指定自定义的  :attr:`~.AsyncSession.sync_session_class`  来指定。 

    .. change::
        :tags: SQLAlchemy ORM, bug, Oracle, performance
        :tickets: 4486

        向反射查询针对Oracle系统视图（如ALL_TABLES，ALL_TAB_CONSTRAINTS等）的DDL-name参数添加CAST（VARCHAR2（128））
        ，以便更好地针对这些列上的索引执行操作，因为在Python中使用Unicode用于字符串时，
        这些列以前将会被隐式处理为NVARCHAR2;这些列在所有Oracle版本中的文档中均被记录为VARCHAR2，长度根据服务器版本而异，从30到128个字符不等。
        另外，针对Oracle数据库启用了支持Unicode命名的DDL结构的测试支持。

.. changelog::
    :version: 1.4.22
    :released: July 21, 2021

    .. change::
        :tags: SQLAlchemy ORM, bug, SQL
        :tickets: 6786

        修复了使用``whens``参数将字典作为位置参数传递，
        而不是关键字参数时，会发出2.0弃用警告的问题，请参见  :ref:`change_6786` 。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6775

        修复了新的  :meth:`_schema.Table.table_valued`  方法中生成的  :class:` _sql.TableValuedColumn` 
        构造方式的问题，当它们应用于ORM中的贪婪加载，多态加载等时，不会正确响应别名适配。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6769

        解决了ORM加载程序策略中使用  :meth:`_orm.Load.options`  方法时出现的一个问题，
        特别是在多次嵌套调用时，会产生过长且非确定性的缓存密钥，导致非常大的缓存密钥，
        以及在ORM上下文的总内存使用和缓存本身中使用的条目数方面都没有效率。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6869

        修复了加载器策略中使用  :new:`orm.with_only_columns`  
        作为列元素被替换为  :meth:`_sql.Select.with_only_columns`  
        或  :meth:`_orm.Query.with_entities`  时无法应用过滤器的回归问题。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6887

        调整了ORM的  :attr:`_orm.ORMExecuteState.user_defined_options`  访问器接收来自上下文的   :class:` _orm.UserDefinedOption`  
        和相关选项对象的方法，特别是在使用“loader
        strategy”的“selectinload”中以前没有工作;其他策略没有这个问题。
        随着修复的效果，一个自定义选项，例如 dogpile.caching 示例中使用的选项，
        以及用于其他配方的自定义选项，如定义水平共享扩展的“分片ID”，现在将被正确传递到急切和懒惰加载器中，
        无论是否调用了缓存查询。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6886

        调整了“From lint”的警告功能，以适应不止一个级别深的连续联接的链，
        在这里on语句没有显式匹配目标，例如“ON TRUE”
        。此使用模式旨在通过事实上“a to b”的连接来取消笛卡尔积警告，
        仅在其中具有多个元素的连接链具有一定的效果。

    .. change::
        :tags: SQLAlchemy ORM, bug, PostgreSQL
        :tickets: 6886

        向PostgreSQL的'overlaps'，'contained_by'，'contains'运算符中添加“is_comparison”标志，
        以便在相关的ORM上下文以及与“From lint”功能结合使用时起作用。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6812

        修复了未加载2.0标记的SQL表达式形式的内部使用问题，即在使用INSERT语句时会发出弃用警告的问题。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6881

        修复了使用新的  :meth:`_orm.PropComparator.and_` 
        特性时出现的问题，在嵌套多个层次deep中的选项中，选项会失败并导致在nested criteria内的绑定参数值不更新。

    .. change::
        :tags: bug, SQLAlchemy, general
        :tickets: 6136

        修改了安装要求，使``greenlet``仅成为默认要求，而适用于``greenlet``已安装且pypi上已经有预构建二进制文件的众所周知的平台。
        当不支持``greenlet``的平台上安装和测试SQLAlchemy 1.4时，将默认情况下不安装``greenlet``，
        不包括任何asyncio 功能。要包含``greenlet``包依赖项以在上述列表之外的机器体系结构上安装，
        可以通过运行``pip install sqlalchemy [asyncio]`` 来包括 ``[asyncio]`` 扩展，这将尝试安装``greenlet``。此外，测试套件已更新，
        以便在不安装 ``greenlet`` 时可以完全完成测试，对于与asyncio相关的测试，适当的跳过测试。

    .. change::
        :tags: SQLAlchemy, schema
        :tickets: 6146

        在原生和非原生实现中统一了 :class:`_schema.Enum` 在别名元素的枚举中针对接受的值的行为。
        当  :paramref:`_schema.Enum.omit_aliases`  为` `False``时，将接受所有值，包括别名。
        当  :paramref:`_schema.Enum.omit_aliases`  为` `True``时，只有非别名值才被接受为有效值。

    .. change::
        :tags: bug, SQLAlchemy, ext
        :tickets: 6816

        修复了水平分片扩展无法正确适应在 ``execute`` 方法中传递纯文本SQL语句的问题。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6889, 6079

        调整了ORM加载器内部使用的“lambda缓存”系统，对于在一个ORM查询中，需要沿着固定的使用模式移动的查询是降低开销的有效方法；
        在加载器策略的情况下，所使用的查询负责通过大量的任意选项和标准，
        这些标准通常生成并且有时通过最终用户代码进行消耗，
        这使得lambda缓存概念与不使用它没有任何差别，
        代价是更多的复杂性。特别是言及了  :ticket:`6881`  和  :ticket:` 6887`  
        的问题已经得到了解决，从而在内部删除了此特性。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6889

        修复锁定系统中未正确创建缓存密钥引起的效率问题。

    .. change::
        :tags: SQLAlchemy ORM, usecase, mypy
        :tickets: 6804, 6759

        通过实现Python特殊方法`` __class_getitem __（）``，
        允许在用户代码中定义SQLAlchemy类时使用“通用类”语法，例如``Column [String]``，
        而无需在``TYPE_CHECKING``块中限定这些构造。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: N/A

        修复了在  :meth:`_sql.Insert.values`  和Python ` `None``值一起使用时，
        在特定情况下，会使特定类型的绑定参数处理程序不被调用，

.. changelog::
    :version: 1.4.21
    :released: July 14, 2021

    .. change::
        :tags: bug, SQLAlchemy ORM, SQL
        :tickets: 6752

        向每个符合功能的  :meth:`_sql.select`  ，  :meth:` _sql.insert` ，  :meth:`_sql.update`  和  :meth:` _sql.delete` 
        构建函数添加了一个新的方法  :meth:`_sql.HasCTE.add_cte`  。
        此方法将给定的 :class:`_sql.CTE` 添加为语句的“独立”CTE，这意味着它会在WITH子句中无条件地呈现在语句上面，
        即使它在主语句中没有被引用。 这是在PostgreSQL数据库上的流行用例，
        其中CTE用于独立于主语句运行针对数据库行的DML语句。

    .. change::
        :tags: bug, PostgreSQL, SQLAlchemy ORM
        :tickets: 6755

        修复了  :meth:`_postgresql.Insert.on_conflict_do_nothing`  和  :meth:` _postgresql.Insert.on_conflict_do_update` 
        中``constraint``参数给定的唯一约束名不会正确截断为长度时，
        基于创建表语句的实际大小于63个字符的PostgreSQL max标识符长度的情况。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6710

        修复了CTE构造中的一个问题，其中递归CTE引用具有重复列名的SELECT，
        这些列名通常在1.4中使用标记逻辑进行去重，但在选择WITH子句内引用清除了去重标签名称。

    .. change::
        :tags: bug, regression, SQL Server, SQLAlchemy ORM
        :tickets: 6697

        修复了SQL Server dialect中的特殊点模式名称处理不正确，
        如果在 ``schema_translate_map`` 中使用了点模式名称，则不会起作用。

    .. change::
        :tags: bug, SQLAlchemy ORM, loading
        :tickets: 6718

        修复了ORM导出的可执行选择器不能在  :meth:`_orm.Session.execute`  中运行的问题。
        现在，  :meth:`_orm.SessionEvents.do_orm_execute`  处理程序可以正常工作。

    .. change::
        :tags: bug, SQLAlchemy ORM
        :tickets: 6538

        在将作为``@validates``验证器函数或``@reconstructor``重构的可调用对象传递给mapper时，
        调整了可调用对象的检测。

.. changelog::
    :version: 1.4.12
    :released: 2021年4月29日

    .. change::
        :tags: bug, orm, regression, caching
        :tickets: 6391

        修复了重大的回归问题，其中作为 SQL 缓存系统中使用的绑定参数跟踪可能无法跟踪所有参数，特别是在使用了类继承等特性的 ORM 相关查询中使用相同 SQL 表达式包含参数的情况下出现问题。ORM 将在编译时单独复制每个 SELECT 语句，这将导致在嵌套语句中使用相同的表达式多次，例如 UNION 时失败。跟踪此条件的逻辑已经调整为适用于参数的多个副本。

    .. change::
        :tags: bug, sql
        :tickets: 6258, 6397

        修改了"EMPTY IN" 表达式，不再依赖于使用子查询，因为这会导致一些兼容性和性能问题。在已选数据库的新方法中利用返回 NULL 来判断空值。更改后在所有受支持的数据库上兼容，此更改也提高了许多未使用子查询的查询的性能。

    .. change::
        :tags: bug, general
        :tickets: 6395

        修复在使用类似`COPY FROM`等 PostgreSQL 命令时可能发生的问题，其中在连接上多次交替地进行写入和（或）读取，导致造成额外的传输和尝试读取未读取的内容。

    .. change::
        :tags: bug, general
        :tickets: 6407

        使用类似 `pymysql` 等 Python 本机库连接到 MariaDB 时，将允许传递 `unix_socket` 参数给 MariaDB 的连接库；此参数仍在 Unix 套接字连接中使用。

    .. change::
        :tags: bug, dialect
        :tickets: 6405

        对于与 PostgreSQL 数据库的新连接，将在 `unix_socket` 参数穿越到连接库时保留该值，以便于在必要时使用该值。此技巧取代了以前在这种情况下使用 `host` 参数以及将 `host` 设置为空字符串以阐明 <http://stackoverflow.com/questions/34110725> 问题；原因是如果没有正确的值，PostgreSQL 驱动程序会关闭套接字，而套接字依赖于该值来进行有效配置。

    .. change::
        :tags: bug, orm, regression
        :tickets: 6254

        在像调用 Python 函数一样使用 SQL 函数时，修复了上下文表达式中使用的多个函数的问题，这会导致内部结果缓存不正确。

    .. change::
        :tags: bug, orm, regression
        :tickets: 6255

        去掉了使用于联接的显式类型带来的无法在  :paramref:`_orm.relationship.primaryjoin`  中传入字符串的限制（将提示消息更改为导致失败的原因）。这在 1.4.10 中是不必要的限制。

    .. change::
        :tags: bug, typecasting, postgresql
        :tickets: 6374

        修复了在 Postgres 中某些类型的 `element()` 函数根据平台和设置会导致 `ArgumentError` 的问题。

    .. change::
        :tags: bug, sql, mssql
        :tickets: 6381

        确保在以 SQLServer 2012 的身份使用 `order by offset/fetch` 语句时，生成的 `forward_only` 游标 总是带上 `static`，这是对该版本 SQL Server 默认的行为。

    .. change::
        :tags: bug, orm, relationship
        :tickets: 2263

        修复了当在“单一”继承层次结构中使用外键约束定义来引用父类中的某些数据时，  :paramref:`_orm.relationship`   的反向引用失败的问题。

    .. change::
        :tags: bug, mysql
        :tickets: 6249

        如果 MySQL 表具有引号字符，则  :mod:`.mysql`  将带有引号字符的表名正确地订阅为标识符。

    .. change::
        :tags: bug, postgresql
        :tickets: 6399

        在使用 PostgreSQL 的 `introspection` 时，现在更好地获取索引列的数据，如果没有为某些计算表达式获取数据列，则更好地报告这些情况。

    .. change::
        :tags: bug, postgresql
        :tickets: 6384

        修复了在使用 Postgres / psycopg2 时不能正常检测无效游标状态的问题，该游标状态应该作为闲置状态（在 Postgres 中，当游标显式关闭时才是普遍用法）; 最终会发现此状态并重新连接。

    .. change::
        :tags: bug, orm, regression
        :tickets: 6393

        当将独立的对象提交到处于 `expire_on_commit=True` 状态的 ORM 会话时，修复了延迟加载加载器无法处理已提交的对象的问题。

    .. change::
        :tags: bug, engine, regression
        :tickets: 6396

        修复了在使用 `nullsfirst()`、`nullslast()` 等更改 `ORDER BY` 子句的 SQL 表达式（而不是 ORM 查询）时可能会出现的某些连接对象上的断开的关闭问题，当另一个操作不在当前事务中将连接返回到池中时可能会出现此类问题。本文档是SQLAlchemy 1.4.11版本的发行说明。 SQLAlchemy是一个用于Python编程语言的SQL工具包和对象关系映射器（ORM）。

.. change::
    :tags: bug, sql, regression, oracle, mssql
    :tickets: 6202

    修复了一个在SQL Server和Oracle数据库上触发的查询SQL错误的问题，该错误由一个非空但无效的字典键/值对引起，这些地方使用了新的SQLAlchemy 1.4的编译器有关字典推断的特性。

.. change::
    :tags: bug, regression, mssql
    :tickets: 6173, 6184

    修复了在SQL Server上指定 OFFSET 0 操作时的分页问题。这个问题会导致结果不符合预期。该问题的解决也解决了在其它类似场景下导致 SQL 解析器抛出异常的问题。

.. change::
    :tags: bug, sql
    :tickets: 6174

    修改了 "expanding" 特性，在使用 in 操作符的时候，能够从右侧列表的元素中推断出表达式的类型。能够支持字符串化等一些操作。

.. change::
    :tags: bug, orm
    :tickets: 6188

    现在  :meth:`_orm.Session.commit`  和  :meth:` _orm.Session.rollback`  方法不再强制执行 SQL 刷新语句。这在某些情况下有利于在数据库事务期间执行多个查询。但是，在执行其他查询或事务之前，需要解决现有的任何挂起问题。

.. change::
    :tags: feature, orm
    :tickets: 5172

     :meth:`_orm.Session.add` ,  :meth:` _orm.Session.bulk_save_objects` ,  :meth:`_orm.Session.bulk_insert_mappings` 
    和  :meth:`_orm.Session.bulk_update_mappings`  的参数 nowreturn_defaults 现在可以设置为列表。这个参数用于支持返回主键值的情况 ` "returning" 插入 <insert_returning_clause>`，`"insert" or "update from select"<insert_select_clause>`，可支持使用序列的 DBAPI 驱动。

.. change::
    :tags: bug, testing
    :tickets: 6185

    Fixed the failure of the ``test_query_callable`` test for PostgreSQL in Python 3.10.

.. change::
    :tags: bug, orm
    :tickets: 6175

    Fixed query criteria involving the use of a NULL discriminator value with   :class:`_orm.polymorphic_union` ; the query would fail to match in some situations when the non-ORM equivalent would succeed.

.. change::
    :tags: bug, sql
    :tickets: 6193

    Fixed the SQL compiler's ALTER TABLE command to constrain the primary key column to NOT NULL when using PostgreSQL.

.. change::
    :tags: bug, orm
    :tickets: 6195

    Deeply nested aliased joins will better retain the correlation and name of the original path from where they were generated.

.. change::
    :tags: bug, core
    :tickets: 6095

    Fixed the detection of sequences during a table reflection; the old sequence detection code was being invoked even when explicitly requested not to.

.. change::
    :tags: bug, orm
    :tickets: 6203

    Fixed issue in the ORM query translation system whereby the leftmost side of a multi-table join could be misunderstood as not being "aliased" if that table contained columns that were not already present in the ON clause of the join.

.. change::
    :tags: bug, sql, postgresql
    :tickets: 6206

    Fixed an issue with the addition/removal of an IF clause to constraints that explicitly included the "DEFERRABLE" clause in a PostgreSQL environment.


.. change::
    :tags: bug, sql
    :tickets: 6208

    Fixed issue whereby the INFO logging level would not emit logs for certain event hooks on the execution of a SELECT statement.

.. change::
    :tags: bug, sql, postgresql
    :tickets: 6209

    ``array_of_enum`` is now treated as an explicit   :class:`_types.ARRAY`  type and not coerced within   :class:` _postgresql.base.PGCompiler` .

.. change::
    :tags: bug, orm, postgresql
    :tickets: 6201

    Fixed serialization difference in reentrant   :class:`_orm.Session`  uow transactions that involved before_flush() handlers, when targeting PostgreSQL databases.

.. change::
    :tags: bug, mysql
    :tickets: 6180

    Fixed regression caused by the patch for  :ticket:`5292` , where the new behavior required by that ticket caused a failure when setting a "charset" that was not part of the MySQL first party set; this can occur with other proprietary branches of MySQL like MariaDB.

.. change::
    :tags: doc, orm
    :tickets: 6170

    Improved documentation of the  :meth:`_orm.sessionmaker.using_bind`  method to explain more clearly when and why the method is used.

.. change::
    :tags: feature, sql
    :tickets: 5154

    Extended the multi-row VALUES syntax to allow for tuples in the VALUES clause, so that the VALUES clause may contain a list of tuples to insert, rather than just a list of lists. 

.. change::
    :tags: feature, orm
    :tickets: 6187

    New ``autocommit=False`` option added to  :meth:`_orm.Session.begin_nested`  that will provide the same "savepoint" behavior of a nested transaction but without implicitly ending the containing/outer transaction when the innermost/nested transaction is committed.   The ` `autocommit=False`` flag can be used in the event that a business rule requires the things that occur inside of a nested transaction to not be committed if the outermost transaction is rolled back, and to still remain isolated if the outermost transaction rolls back other changes.

.. change::
    :tags: change, orm
    :tickets: 6190

    Significant changes and extensions to the unified   :class:`_orm.sync`  method, which can be used to synchronize all mapped tables to that of the current metadata.  In particular the method can now copy constraints between tables even when the target table already exists.   All constraints are further categorized according to their specifiers such as that they are uniquely named, non-nullable, etc. and can thus be better targeted for copy operations as needed.  All such sync operations are now emitted under a "DDL" category of SQL, instead of that of "INSERT", "UPDATE", or "DELETE".

.. change::
    :tags: bug, orm
    :tickets: 6181

    Fixed regression whereby a   :class:`_orm.subqueryload`  option specified along with   :class:` _orm.joinedload`  would fail to load columns from the joined table for joined-inheritance relationships.

.. change::
    :tags: bug, orm
    :tickets: 6196

    Fixed issue with the 1.4 series commit of  :ticket:`6051` , whereby subquery eager loading would fail to detect "self-referential" conditions if those cycles involved the use of a   :class:` _sql.ColumnElement`  above the "lowest" of the eager load joined tables.

.. change::
    :tags: bug, postgresql
    :tickets: 6214

    Fixed regression where the expression generated for the generation for   :class:`_postgresql.base.UUID`  type no longer included a cast to text in some cases, which fails for certain UUID fields that contain a dash.


.. change::
    :tags: improvement, sql
    :tickets: 4378

    Added support for "compound SELECT", whereby SQL expressions UNION, EXCEPT and INTERSECT can stack multiple SELECT expressions with different column collections.  Django's "Q" object makes extensive use of compound select.


.. change::
    :tags: bug, orm, postgresql
    :tickets: 6226

    Fixed issue with  :meth:`_orm.Session.flush`  failure due to a   :class:` _orm.relationship`  that contains two self-referential pairs of foreign keys both referring to a single target column, in the case where the column from the target table is aliased.

.. change::
    :tags: improvement, orm
    :tickets: 6023

    Added support for reentrant   :class:`_orm.Session`  uow transactions to have pending ACTIVATE_SESSION events and listen for them appropriately, so that a less verbose form of sub-session management can be applied when each reentrant transaction is committed.

.. change::
    :tags: improvement, sql
    :tickets: 5093

    Added support for the shorthand notation of "WHERE <col> = <param_name>" when passing a dictionary to  :meth:`_sql.expression.insert`  or  :meth:` _sql.expression.update`  to specify a SQL WHERE clause within those operations.

.. change::
    :tags: bug, orm
    :tickets: 6224

    Fixed regression of the feature added by  :ticket:`4999` , whereby the behavior added which would rerun autoflush operations in the case of a   :class:` _orm.Session`  using legacy autocommit mode (i.e. an "implicit" transaction) could lead to unintended updates to the database in the event that modified persistent or detached objects are present within the session.

.. change::
    :tags: bug, sqlite
    :tickets: 6225

    Repaired the usage of "schema" options in SQLite reflecting a divided behavior between URI-based (``sqlite:///myfile.db``) and file name based (``sqlite:///myfile.db?cache=shared&mode=memory&...``) filename specification.   The fix includes that the special keyword ":memory:" may be used. Also the usage of a custom absolute file path is now deprecated.

.. change::
    :tags: bug, orm
    :tickets: 6246

    Fixed issue whereby the contextual handling of database exceptions, specifically   :class:`_exc.StatementError`  and its related types used as well as any user-defined exceptions that inherit from these types, were not being correctly handled in terms of the transactional state of the   :class:` _orm.Session`  object.

.. change::
    :tags: bug, sql
    :tickets: 6236

    Fixed issue in the SQL statement compiler whereby placing a  :meth:`_sql.expression.Cast`  expression within the RETURNS portion of a PostgreSQL FUNCTION call would raise an assertion failure.

.. change::
    :tags: bug, orm, mssql
    :tickets: 6245

    Fixed a bug whereby the ORM, when using the legacy mode for querying by database objects (table *(name)*), would raise the wrong exception type when a hint or with_hint method were used in MSSQL.

.. change::
    :tags: bug, orm
    :tickets: 6250

    Fixed issue in the   :class:`_orm.Mapper`  object whereby it would fail to check for a ` `_sqla_adapter`` key within the foreign key argument list when generating synthetic foreign keys.

.. change::
    :tags: bug, postgresql
    :tickets: 6270

    Fixed a bug in the Postgresql dialect whereby the  :meth:`_types.ARRAY.Comparator.overlap`  and similar methods might give incorrect results for arrays whose dimension exceeds the default value of ` dimension`.

.. change::
    :tags: bug, mysql
    :tickets: 6243

    Fixed the dialect so that the creation of a temporary table now uses the custom table arguments specified in the dialect.

.. change::
    :tags: bug, sql, postgresql
    :tickets: 6251

    Fixed a bug where a surplus comma was appearing when using a   :class:`_sql.text`  object within the columns list of a SELECT statement in PostgreSQL.

.. change::
    :tags: bug, orm
    :tickets: 6253

    Fixed regression that caused a harmless assertion failure by an   :class:`_orm.relationship`  object when a subclass mapper in the cycle of a self-referential mapper was deferred.

.. change::
    :tags: bug, mysql
    :tickets: 6263

    Fixed regression in the MySQL/MariaDB dialect regarding the generation of foreign keys with data type. The fix also addresses the inheriting of data types of columns for generated foreign keys.

.. change::
    :tags: bug, orm, sql
    :tickets: 6275, 6090

    Fixed a regression where the ORM, when streaming query results and using autoflush simultaneously, would fail to insert rows that involved child objects with a bi-directional backref. A partial fix for this regression was in 1.4.5, which partially fixed the issue only for the special case of when querying only one object row, not for when a full result set of rows is streamed.

.. change::
    :tags: bug, mysql
    :tickets: 6284

    Fixed a bug in the MySQL/MariaDB dialect whereby the VALUES clause within an INSERT statement would fail to guess the parameter style of the given items and raise an `InvalidRequestError`. This can lead to compatibility issues when using third party libraries that specify parameters in a different style than the default.

.. change::
    :tags: bug, sqlite
    :tickets: 6293

    Fixed a regression in the 1.4 series where the SQLite dialect would attempt to emit an "empty set" expression that is not supported by SQLite version 3.31, rather than an equivalent, supported expression like "SELECT rowid FROM sqlite_master LIMIT 0".

.. change::
    :tags: bug, sqlite
    :tickets: 6294

    Fixed a regression in the 1.4 series where the SQLite dialect would, under certain conditions, emit an extra leading semicolon in the SQL statement.

.. change::
    :tags: bug, mssql
    :tickets: 6056

    Fixed a bug in the MSSQL dialect where floating point "Nan" and "Inf" values would not be properly quoted.

.. change::
    :tags: bug, orm
    :tickets: 6297

    Fixed a regression introduced in 1.4.9 whereby the "immediateload" option wouldn't be fully respected when used via   :func:`_orm.with_loader_criteria`  or inline with a relationship option.

.. change::
    :tags: bug, mysql
    :tickets: 6286

    Fixed a regression whereby the MySQL dialect wouldn't correctly enforce the "sql_mode" provided in the dialect options.

.. change::
    :tags: bug, orm, postgresql
    :tickets: 6240

    Fixed a bug in the ORM with updating single table inheritance tables when partial updates wish to refer to table column labels that are remapped as part of the polymorphic identity rules.

.. change::
    :tags: bug, sqlexpression
    :tickets: 6289

    Fixed a regression in the 1.4.x series where the  :meth:`_sql.visitors.Visitable._compiler_dispatch`  method would fail to return a value when called with a non-expression argument.

.. change::
    :tags: bug, mssql
    :tickets: 6283

    Fixed a regression whereby the MSSQL/sybase dialect would not treat "datetimeoffset" columns as timezoned. This fix builds on the fix in 1.4.9 for the related issue involving "DateTime" columns.

.. change::
    :tags: bug, postgresql
    :tickets: 6292

    Fixed a regression introduced in 1.4.1 whereby a   :class:`_postgresql.RE_NUM`  object would not fulfill the expected contract to have an "escape" method.

.. change::
    :tags: bug, mysql
    :tickets: 6298

    Fixed inconsistency in the MySQL server version detection code whereby the server banner wasn't being processed in all cases.

.. change::
    :tags: bug, orm, postgresql
    :tickets: 6307

    Fixed regression whereby ORM wouldn't properly apply server_default to columns that have a string based default and server_default is also passed as a string.

.. change::
    :tags: bug, postgresql
    :tickets: 6278

    Fixed regression in 1.4 series in PGEnum whereby lower case attribute was also converted to upper-case, if it's within ''.

.. change::
    :tags: bug, postgresql
    :tickets: 6258

    Fixed bug introduced in 1.4 when adding a UNIQUE constraint that included a VARCHAR column as part of an array type.

.. change::
    :tags: bug, mysql
    :tickets: 6127

    Fixed regression in MySQL dialect whereby SQLAlchemy would not convert a string to a specific ENUM type, if it's wrapped with a local class.

.. change::
    :tags: bug, postgresql
    :tickets: 6310

    Fixed regression where the Postgresql dialect would emit GROUP BY COLUMN expressions without a proper type.

.. change::
    :tags: bug, orm, postgresql
    :tickets: 6255

    Fixed regression introduced in 1.4.1 whereby using the   :class:`_orm.Mapped`  annotation in conjunction with a   :func:` _orm.relationship`  that refers to a class by string name would not integrate correctly with the ORMs query generation and lead to typing errors.

.. change::
    :tags: bug, orm, postgresql, mysql
    :tickets: 5569

    Fixed to allow the SELECT statement to reference columns that appear in the ON clauses of outer joins when those columns are selected through a subquery.  This fix requires the use of MySQL version 8.0.13 or later.

.. change::
    :tags: bug, sqlite
    :tickets: 6312

    Fixed a regression whereby the SQLite dialect would use a column's data type from a preexisting source table when creating a temporary table, instead of checking the actual data.

.. change::
    :tags: bug, mysql
    :tickets: 6316

    Fixed issue with MySQL 8.0.23 where the `CURRENT_TIMESTAMP` function is no longer treated as a TIMESTAMP. SQLAlchemy now refers to the server_version_info property for information about the server version which enables a fix for MySQL version < 8.0.23.

.. change::
    :tags: bug, mysql
    :tickets: 6317

    Fixed issue with MySQL 5.7 where MySQL would not accept a connection that specified the password in a non-ASCII format.


.. change::
    :tags: bug, sql
    :tickets: 6329

    Fixed a regression in the 1.4.x series where a   :class:`_types.Numeric`  object would not fulfill the expected contract to have an "escape" method.

.. change::
    :tags: bug, mysql
    :tickets: 6328

    Fixed an issue with `mysqlflex` that caused errors when connecting to a MySQL server that had sha256 authentication enabled.

.. change::
    :tags: bug, orm, postgresql
    :tickets: 6313

    Improve the behavior of the Postgresql dialect with regard to UUID columns in the absence of the "UUID-OSSP" extension, so that a string representation of a UUID column value will always be produced in a query.

.. change::
    :tags: bug, sql
    :tickets: 6332

    Fixed regression whereby a   :func:`_sql.expression.delete`  operation containing a  :meth:` _sql.expression.FromClause.select`  object and an ORDER BY would fail to interpret the ORDER BY statement properly.

.. change::
    :tags: bug, sqlite
    :tickets: 6336

    Fixed an issue with the SQLite dialect whereby a table generated with the 'IF NOT EXISTS' clause would be created in the default schema rather than in the specified schema when using a non-default schema.

.. change::
    :tags: bug, postgresql, sqlalchemy_utils
    :tickets: 6340

    Fixed a regression caused by sqlalchemy-utils upon SQLAlchemy 1.4's usage of a new object name quoting function.

.. change::
    :tags: bug, orm
    :tickets: 6334

    Fixed regression introduced in 1.4.9 whereby failed statement handlers would be raised under the eventual   :class:`_orm.SessionException`  exception which is too high in the exception hierarchy.

.. change::
    :tags: bug, postgresql
    :tickets: 6348

    Fixed an issue in the Postgresql dialect where a zero-length schema on a schema-qualified table name would prevent the subsequent resolution of the stack trace for an exception raised by a SQL statement containing this table name.




.. change::
    :tags: bug, orm
    :tickets: 6358

    Fixed the ORM's assertions of cached expression along with server defaults that were added in 1.4 that could cause false errors in the event a column object such as that of a   :class:`_sql.expression.ColumnClause`  is shared across multiple tables.


.. change::
    :tags: bug, sql
    :tickets: 6635

    Fixed the SQL statement compiler so that it can visit a   :class:`_sql.expression.TextClause`  object that refers to a   :class:` _expression.ColumnElement`  with no table assigned.

.. change::
    :tags: bug, sql, mssql
    :tickets: 6369, 6339

    Fixed an issue where  :meth:`_sql.elements.AnonymousGrouping.element`  it would not reflect the correct labeling of columns for the purposes of ORDER BY clause generation, leading to incorrect queries on backends like MSSQL.


.. change::
    :tags: bug, orm, postgresql
    :tickets: 6356

    Fixed issue relating to the type returned by  :meth:`_postgresql.base.UUID.Comparator.__eq__`  when comparing against None.

.. change::
    :tags: bug, sql, postgresql
    :tickets: 6359

    Fixed the Postgresql compiler to better support the use of %(foo)s parameters within SQL expressions.


.. change::
    :tags: feature, orm
    :tickets: 6375

    Added support for the SQLAlchemy ORM to the new "wait for startup" behavior added to the  :meth:`_engine.Connection.connect`  method in version 1.4.10.   New options allow for a custom error message to be emitted when the PostgreSQL server is not yet started up, as well as an option to persist retry a fixed number of times.


.. change::
    :tags: change, orm
    :tickets: 6406

    Changed the availability of the 2.0 preview feature of using external (non-Generated) elements with the ORM.   This feature is now turned off by default.   The previous behavior was to enable this feature by default but to raise a deprecation warning.

.. change::
    :tags: bug, sql, regressin
    :tickets: 6410

    Fixed an incompatibility with SQLite version 3.36.0 or later where a `:memory:` database could not be opened due to the existence of a small (3-byte) footer at the end of the file.




.. change::
    :tags: bug, diaketomation, json
    :tickets: 6321

    Fixed handling of implicit encoders within the JSON serializer for boolean values, so that single-table inheritance systems with BooleanEnum, as well as JSON serialization of many-to-many join tables would render using consistent syntax.

.. change::
    :tags: bug, schema, mssql
    :tickets: 6204

    Fixed issue on MSSQL whereby the "IDENTITY" property would not be reflected when using the  :meth:`_schema.Table.create`  method on a table having a column with this property.

.. change::
    :tags: bug, mysql
    :tickets: 6365

    Fixed a regression whereby the dialect would not properly quote a reserved keyword when it occurs as a string within a column server default string. 

.. change::
    :tags: bug, sqlite
    :tickets: 6412

    Fixed a regression whereby a string containing only a schema and no table within a quoted name wasn't properly recognized as a schema.


.. change::
    :tags: bug, sql
    :tickets: 6444

    Fixed a regression whereby a column expression containing a label and an alias that also includes quotes is invalid SQL syntax and would lead to an error when compiled... 该版本更新主要修复了一些发现的错误和问题以及实现了一些新特性
.. changelog::
    :version: 1.4.0
    :released: March 15, 2021

    .. change::
        :tags: bug, mssql
        :tickets: 5919

        修复了反射MSSQL 2005时由过滤的索引和=引入的反射错误

    .. change::
        :tags: feature, mypy
        :tickets: 4609

        添加了支持Mypy插件的原始支持。这个插件依赖于新的SQLAlchemy类型存根。插件允许声明式映射以符合Mypy，并为映射的类和实例提供类型支持。

    .. change::
        :tags: bug, sql
        :tickets: 6016

        修复了对于使用格式化的绑定参数风格的方言（如“格式化”或“pyformat”）而言，“op”和“custom_op”构造中的百分比转义功能未被启用的错误，对百分比符号进行必要的自动加倍。

    .. change::
        :tags: bug, regression, sql
        :tickets: 5979

        修复了未知数据类型的“不支持编译错误”未能正确引发的回归

    .. change::
        :tags: ext, usecase
        :tickets: 5942

        添加了新的参数 ``reflection_options``，允许传递``only``或特定于方言的反射选项，如 ``oracle_resolve_synonyms``。

    .. change::
        :tags: change, sql

        改变了``CTE``构造的编译方式，以便在上下文的闭合SELECT外直接字符串化时返回表示内部SELECT语句的字符串，与``FromClause.alias``和``Select.subquery``的行为相同。以前，在生成SELECT之后，CTE通常会放置在SELECT上面，这通常会导致调试时产生的混淆。


    .. change::
        :tags: bug, orm
        :tickets: 5981

        修复了“query_class”参数对于“动态”关系已经停止工作的回归。``AppenderQuery``仍然依赖于传统的“Query”类；鼓励用户从使用“动态”关系迁移。关系到使用   :func:`_orm.with_parent`  的联系。

.. change::
    :tags: bug, orm, regression
    :tickets: 6003

    修复了一个问题，即如果查询本身以及联接目标都针对一个   :class:`_schema.Table`  对象而不是映射类，则  :meth:` _orm.Query.join`  将不起作用的回归。这是一个更全面问题的一部分，即如果语句中没有在其中出现 ORM 实体，则传统 ORM 查询编译器将不会被正确地使用，这种现象从一个   :class:`_orm.Query`  中开始。

.. change::
    :tags: bug, regression, sql
    :tickets: 6008

    修复了一个问题，即以被直接选择为形式使用的独立的   :func:`_sql.distinct()`  无法通过列身份在结果集中找到的回归，这是 ORM 查找列的方式。虽然独立的   :func:` _sql.distinct()`  不适合直接选择（使用  :meth:`_sql.select.distinct`  进行常规 ` `SELECT DISTINCT..``），但它在这方面在早些时候是有限可用的（但在子查询中不起作用，例如）。如此改进一元表达式（例如"DISTINCT <col>"）的列定位器，以使这种情况再次工作，并且已经进行了额外的改进，以使这种方式在子查询中的使用至少能够生成有效的 SQL，但这在之前并不是这种情况。

    此更改还增强了基于 ORM enabled SELECT 语句中的 SQL 表达式对象定位 ``row._mapping`` 中的元素的能力，包括语句是由 ``connection.execute()`` 还是 ``session.execute()`` 调用的。

.. change::
    :tags: bug, orm, asyncio
    :tickets: 5998

     :meth:`_asyncio.AsyncSession.delete`  的 API 现在是可等待的。这种方法会级联到必须以类似的方式加载触发的关系，就像  :meth:` _asyncio .AsyncSession.merge`  方法一样。

.. change::
    :tags: usecase, postgresql, mysql, asyncio
    :tickets: 5967

    在 SQLAlchemy 的模拟 DBAPI 游标中添加了一个 ``asyncio.Lock()``，该锁是连接范围内的，适用于 ``cursor.execute()`` 和 ``cursor.executemany()`` 方法的 aiomysql dialect 和 asyncpg。其理由是为了防止连接同时被多个可等待对象使用而导致故障和损坏。虽然此用例在使用线程代码和非 asyncio dialect 时也可能会出现，但我们预计这种用法在 asyncio 下会更常见，因为 asyncio API 鼓励这样的用法。但是，如果不使用单独的连接，则绝对最好针对每个并发 awaitable 使用一个不同的连接，因为否则不会实现并发性。对于 asyncpg dialect，这是为了防止 ``prepare()`` 调用和 ``fetch()`` 之间的空间允许并发执行，在该连接上启动事务时避免竞争条件。其他 PostgreSQL DBAPI 在连接级别上是线程安全的，因此这旨在提供类似的行为，在服务器端游标的领域之外。

    对于 aiomysql dialect，互斥锁将提供安全性，例如防止语句执行和结果集获取，这是连接级别的两个不同步骤，被同时执行在同一个连接上可能会被并发执行的情况所破坏。

.. change::
    :tags: bug, engine
    :tickets: 6002

    改进了 engine 日志记录，以记录在 DBAPI 驱动程序处于 AUTOCOMMIT 模式时记录的 ROLLBACK 和 COMMIT。这些 ROLLBACK / COMMIT 在库级别上，如果采用 AUTOCOMMIT，则没有任何影响，但是它们仍然值得记录，因为它们指示 SQLAlchemy 看到的“事务”划分。

.. change::
    :tags: bug, regression, engine
    :tickets: 6004

    修复了 reset-agent 连接池的回归，它未真正被   :class:`_engine.Connection`  在关闭时使用，并且还导致了一种双重回滚的情况，这种情况有些浪费。对于引擎的新体系结构，已更新连接池“归还时重置”逻辑，以便在   :class:` _engine.Connection`  在将池返回到连接之前明确关闭事务时将跳过连接池“reset-on-return”逻辑。

.. change::
    :tags: bug, schema
    :tickets: 5953

    不再支持模式级别的所有 ``.copy()`` 方法，并将其重命名为 ``_copy()``。这些不是标准的 Python "copy()" 方法，因为它们通常依赖于在其内部传递给方法的特定上下文中实例化。  :meth:`_schema.Table.tometadata`   方法提供了   :class:` _schema.Table`  对象的复制公共 API。

.. change::
    :tags: bug, ext
    :tickets: 6020

    ``sqlalchemy.ext.mutable`` 扩展现在使用与对象关联的   :class:`.InstanceState`  跟踪“父”集合，而不是对象本身。后一种方法要求对象是可哈希的，因此可以位于 ` `WeakKeyDictionary`` 中，这违反了 ORM 的整体行为契约，即 ORM 映射对象不需要提供特定的 ``__hash__()`` 方法，支持不可哈希的对象。

.. change::
    :tags: bug, orm
    :tickets: 5984

    当刷新进行时，工作单元流程现在会完全关闭所有“lazy ='raise'”行为。尽管有些地方 UOW 有时会加载不需要的东西，但是 lazy="raise" 策略在这里并不有用，因为用户通常对刷新过程没有太多的控制或可见性。

.. changelog::
    :version: 1.4.0b3
    :released: March 15, 2021
    :released: February 15, 2021

    .. change::
        :tags: bug, orm
        :tickets: 5933

        修复了新的 1.4/2.0 样式 ORM 查询中，语句级标签样式不会保留为结果集键使用的问题；这已应用于所有 Core/ORM 列 / 会话与连接等组合，以便语句与结果行之间的链接在所有情况下都是相同的。作为这个更改的一部分，已经改进了行中列表达式的标记，使其保留 ORM 属性的原始名称，即使在子查询中使用。

    .. change::
        :tags: bug, sql
        :tickets: 5924

        修复了“笛卡尔积”断言没有正确适应了表之间的联接情况，其中使用 LATERAL 从嵌套上下文中的一个子查询连接到另一个子查询。

    .. change::
        :tags: bug, sql
        :tickets: 5934

        修复了 1.4 回归，其中  :meth:`_functions.Function.in_`  方法没有被测试覆盖并且无法正常工作。

    .. change::
        :tags: bug, engine, postgresql
        :tickets: 5941

        与  :ticket:`5653`  中进行的改进继续支持针对列名生成的绑定参数名称，包括包含冒号、括号和问号的名称，以及改进了测试支持，使得即使它们是基于列名自动派生的绑定参数名称也不会出现问题。目前，如果它们是从列名自动派生的，绑定参数名称应该无法包括括号。检查redshift的操作。

        作为此更改的一部分，asyncpg DBAPI 适配器（仅适用于 SQLAlchemy 的 asyncio dialect）使用的格式已更改，其中使用“format”格式替换了使用“qmark” paramstyle，因为使用“format”样式有一个标准和内部支持的 SQL 字符串转义样式，以用百分号形式的名称（即将百分比符号加倍）为名，而使用“qmark”样式的名称（其中转义系统未由 pep-249 或 Python 定义）使用问号。

        .. seealso::

            :ref:`change_5941` 

    .. change::
        :tags: sql, usecase, postgresql, sqlite
        :tickets: 5939

        增加了 ``set_`` 关键字到   :class:`.OnConflictDoUpdate` ，其接受包含来自   :class:` Selectable`  的 ``.c`` 集合或上下文对象的 ``.excluded`` 的   :class:`.ColumnCollection` 。

    .. change::
        :tags: feature, orm

        在  :term:`2.0 风格`  方法，从由 UPDATE..RETURNING 或 INSERT..RETURNING 语句返回的行中返回 ORM 对象。

        .. seealso::

            :ref:`orm_dml_returning_objects` 



    .. change::
        :tags: bug, sql
        :tickets: 5935

        修复了使用   :func:`_sql.select`  函数的任意可迭代对象不起作用的回归，除了纯列表。向前/向后的兼容性逻辑现在检查更广阔的传入“可迭代”类型，包括从可选择的 ` `.c`` 集合直接传递的情况。 感谢 Oliver Rice 的推荐。

    1) :meth:`_asyncio.AsyncSession.in_transaction` 访问器。

    .. change::
        :tags: bug, sql
        :tickets: 5785

        修复了新的 :class:`_sql.Values` 构造中的问题，在此构造中，传递对象的元组将返回到每个值类型检测而不是使用 :class:`_sql.Values` 传递的 :class:`_schema.Column` 对象，该对象告诉SQLAlchemy预期的类型是什么。这将导致枚举和numpy字符串等对象出现问题，因为并不需要这些对象，因为已经给出了预期的类型。

    .. change::
        :tags: bug, engine

        将“future”关键字添加到已知于  :func:`_sa.engine_from_config` ` sqlalchemy.future = true``或``sqlalchemy.future = false``这样的关键字时可以将“true”和“false”配置为“boolean”值。

    .. change::
        :tags: usecase, schema
        :tickets: 5712

        ：meth：`_events.DDLEvents.column_reflect`事件现在可以应用于:class：`_schema.MetaData`对象，该对象将为那些归属于该集合的:class：`_schema.Table`对象生效。

        .. seealso::

             :meth:`_events.DDLEvents.column_reflect` 

              :ref:`mapper_automated_reflection_schemes`  - 在ORM映射文档中

              :ref:`automap_intercepting_columns`  - 在  :ref: 'automap_toplevel'文件中

    .. change::
        :tags: feature, engine

        可以根据方言将特定于数据库的构造（例如：meth:`_postgresql.Insert.on_conflict_do_update`）直接字符串化，而无需指定显式方言对象。构造在使用``str()``, ``print()``等调用时，现在具有内部方向来调用它们的适当方言，而不是不能将其字符串化的“默认”方言。此方法也适用于通用架构级别的创建/删除，例如   :class:`_schema.AddConstraint` ，它将使其字符串化方言适应其中的元素所指示的方言，例如 :class:` _postgresql.ExcludeConstraint`对象。

    .. change::
        :tags: feature, engine
        :tickets: 5911

        新增了execu-tion option参数:paramref：'_engine.Connection.execution_options.logging_token'。
        此选项将在   :class:`_engine.Connection`  参数来改变)
        这样对于临时连接使用而言不会产生许多新的记录器副作用。选项可以设置在 :class:`_engine.Connection` 或  :class:`_engine.Engine` 级别。

        .. seealso::

            :ref:`dbengine_logging_tokens` 

    .. change::
        :tags: bug, pool
        :tickets: 5708

        修复了一个回退问题，当在设置事件时使用关键字(最常见的是``insert = True``)时，该事件将丢失。
        这将阻止需要在方言级别之前启动事件的启动事件正常工作。

    .. change::
        :tags: usecase, pool
        :tickets: 5708, 5497

        引擎连接程序的内部机制已更改，使得当使用“insert = True”“建立自定义事件处理程序”时，确保预处理在方言特定的初始化启动之前运行，这对于一些操作很有用，例如检测默认模式名称。先前，这只在大多数情况下发生，但不是无条件地发生。在模式文档中添加了一个新示例，该文档说明了如何在连接事件中建立“默认模式名称”。

    .. change::
        :tags: usecase, postgresql

        为asyncpg方言的DBAPI适配层添加了可读/写``.autocommit``属性。
        这样，当需要直接使用DBAPI连接使用“autocommit”与DBAPI特定方案时，就可以使用同样的 ``.autocommit`` 属性，该属性对于psycopg2和pg8000都适用。

    .. change::
        :tags: bug, oracle
        :tickets: 5716

        Oracle方言现在使用 ``select sys_context('userenv','current_schema') from dual``
        从默认模式名称中获取，而不是SELECT USER FROM DUAL，以适应Oracle下的会话本地模式名称更改。

    .. change::
        :tags: schema, feature
        :tickets: 5659

        添加  :meth:`_types.TypeEngine.as_generic`  以将特定于方言的类型，例如:class：` sqlalchemy.dialects.mysql.INTEGER'与“最佳匹配”通用SQLAlchemy类型，例如:class：`_types.Integer'进行映射。陈述请求courtesy Andrew Hannigan。

        .. seealso::

            :ref:`metadata_reflection_dbagnostic_types`  - 示例用法

    .. change::
        :tags: bug, sql
        :tickets: 5717

        修复  :attr:`.RemovedIn20Warning`  错误，其会错误地在对象内部调用` `bind``属性时触发，特别是在字符串化SQL构造时。

    .. change::
        :tags: bug, orm
        :tickets: 5781

        修复了一个1.4回退问题，即在使用  :meth:`_orm.Query.having`  时，结合使用内部调整过的SQL元素(通常在继承方案中使用)将失败，因为函数调用不正确。由esoh提供了拉取请求。

    .. change::
        :tags: bug, pool, pypy
        :tickets: 5842

        Fixed连接池事件指定一个关键词时可能丢失的回退问题，最突出的是“insert = True”时，当未显式关闭/返回连接到池时，连接池将不返回连接或以其他方式完成垃圾回收。由于pypy的gc行为与调用弱引用终结器与其他正在进行垃圾回收的对象存在关联的应有行为不同，这是一个长期存在的问题。现在维护与相关记录的强引用，这样弱引用就有了一个强引用的“基础”，可触发其弱引用终结器。

    .. change::
        :tags: bug, sqlite
        :tickets: 5699

        在使用sqlite时，当使用  :meth:`Column.regexp_match`  方法时，将` `re.search()``而不是``re.match()``作为操作使用的问题已得到解决。这对于在其他数据库上支持的普通表达式的行为以及着名的SQLite插件也适用。

    .. change::
        :tags: changed, postgresql

        解决了psycopg2方言在Python 3下会静默传递“use_native_unicode = False”标志而没有实际影响的问题，因为psycopg2 DBAPI在Python 3下无条件使用Unicode。在Python 3下使用此用法现在会引发: class：`_exc.ArgumentError`。添加对Python 2的测试支持

    .. change::
        :tags: bug, postgresql
        :tickets: 5722
        :versions: 1.4.0b2

        已为  :class:`_schema.Column`   和 :meth:` _sqlite.Insert.on_conflict_do_update`方法的“set_”字典的键，与该目标中的:class：`_schema.Column`对象相匹配的“c”集合。以前，只期望字符串列名，假定列表达式是不在表格之外的表达式，它将随附一个警告而完全呈现。

    .. change::
        :tags: feature, sql
        :tickets: 3566

        实现了“表值函数”的支持，以及PostgreSQL支持的其他语法。它是最常见的提出的功能之一。表值函数是返回值或行的列表的SQL函数，在JSON函数的领域中在PostgreSQL中非常普遍，其中常常将“表值”常称为“记录”数据类型。表值函数由Oracle和SQL Server支持。

        添加的功能包括：

        *  :meth:`_functions.FunctionElement.table_valued`   修改器，从SQL函数创建类似于表格的可选择对象。
        * :class:`_sql.TableValuedAlias` 结构，它将SQL函数呈现为命名表
        * 支持PostgreSQL特殊的“派生列”语法，其中包括列名和有时数据类型，例如``json_to_recordset``函数，使用  :meth:`_sql.TableValuedAlias.render_derived`  。
        * 使用  :paramref:`_functions.FunctionElement.table_valued.with_ordinality`  参数，支持PostgreSQL的“WITH ORDINALITY”结构。
        * 支持从作为列值标量的SQL函数选择，该语法由PostgreSQL和Oracle支持，通过  :meth:`~_functions.FunctionElement.column_valued`  实现
        * 如果不使用FROM子句，可以从表值表达式中SELECT单个列，通过  :meth:`~_functions.FunctionElement.scalar_table_valued`  实现。

        .. seealso::

            :ref:`tutorial_functions_table_valued`  - 在 :ref:` unified_tutorial`中

    .. change::
        :tags: bug, asyncio
        :tickets: 5827

        修复异步连接池中的错误，其中“asyncio.TimeoutError”将被引发而不是: class:`.exc.TimeoutError`修复了 由:asyncio链接到引发asyncio.TimeoutError的错误。当使用async引擎时，现在可以将  :paramref:`_sa.create_engine.pool_timeout`  参数设置为零，以前这将忽略超时并阻塞而不是立即超时与常规的 :class:` .QueuePool`一样行事。

    .. change::
        :tags: bug, postgresql, asyncio
        :tickets: 5824

        解决asyncpg方言中的错误，即在“提交”或更少的情况下失败时，应取消整个事务；
        现在不再可能发出回滚。可以继续等待连接以回滚，但由于asyncpg将拒绝它，因此现在不再可能执行此操作。

    .. change::
        :tags: bug, orm

        修复了使用扩展``sqlalchemy.ext.compiles``创建自定义可执行SQL构造的API时，按照多年来文档中的说明将不再起作用，如果只使用“Executable，ClauseElement”作为基类，则需要使用其他类。这已经解决了这些额外类不需要存在的问题。

    .. change::
        :tags: bug, regression, orm
        :tickets: 5867

        修复ORM工作单元回退错误，其中一个不当的“assert primary_key”语句干扰了主键生成序列，这些序列实际上不考虑表中的列使用真正的主键约束，而会使用  :paramref:`_orm.Mapper.primary_key`  将某些列确定为“主键”。

    .. change::
        :tags: bug, sql
        :tickets: 5722
        :versions: 1.4.0b2

        已对  :class:`_sql.Sequence`  和   :class:` _sql.Identity` `cycle = False``和``order = False``进行了正确呈现，将其呈现为“NO CYCLE”和“NO ORDER”。

    .. change::
        :tags: schema, usecase
        :tickets: 2843

        为  :class:`_ddl.CreateTable` ,   :class:` _ddl.Drop-t-able` ,   :class:`_ddl.CreateIndex` ,
         :paramref:`_ddl.CreateIndex.if_not_exists` ,  :paramref:` _ddl.DropTable.if_exists`   和 :paramref:`_ddl.DropIndex.if_exists` 参数，这将导致添加到CREATE / DROP中的“IF NOT EXISTS”/“IF EXISTS” DDL。这些短语并不被所有数据库接受，如果数据库不支持这种操作，则操作将失败，因为单个DDL语句的范围内没有类似兼容的回退。Pull request courtesy Ramon Williams。

    .. change::
        :tags: bug, pool, asyncio
        :tickets: 5823

        使用asyncio引擎时，连接池现在将一个未明确关闭/返回到池中的池化连接分离并丢弃，当其跟踪对象处于垃圾回收状态时发出警告，指出连接未正确关闭。由于此操作发生在Python gc终结器期间，因此不安全的运行任何IO操作，包括事务回滚或连接关闭，因为它们通常在事件循环之外.

    .. changelog::
        :version: 1.4.0b1
        :released: March 15, 2021
        :released: November 2, 2020

        .. change::
            :tags: feature, orm
            :tickets: 5159

            ORM现在可以生成此前仅在使用 :class:`_orm.Query` 时可用的查询，使用  :func:`_sql.select` 构造直接进行。在Core  :class:`_sql.Select` 内部建立ORM“插件”的新系统，允许此前完全在m中进行查询构建逻辑 :class:`_orm.Query` 现在在 :class:`_sql.Select` 的编译级扩展中进行。 :class:`_orm.Session.execute` 调用时，ORM将在方法本身中执行这些构造。对于 :class:`_sql.Select` 进行调用时，返回的:clas：`_engine.Result`对象现在包含ORM级别实体和结果。

            .. seealso::

                  :ref:`change_5159` 

        .. change::
            :tags: feature,sql
            :tickets: 4737

            已将“子句强转”系统从一系列特定函数重写为一个完全一致的基于类的系统以从一个参数接受并解析 :class:`_expression.ClauseElement` 结构而创建SQL表达式对象。这个变化是内部的，对最终用户没有影响，除了在向表达式对象传递错误类型的参数时提供更具体的错误消息。但是，变化是一组更改的一部分，涉及到：clas：`_expression.select'对象的角色和行为。

        .. change::
            :tags: deprecated, orm
            :tickets: 5606

            块操作在: class:`_orm.Query`中使用的条件子句现在不再接受负索引。

        .. change::
            :tags: sql, change
            :tickets: 4617

            “子句协作”系统，这是SQLAlchemy核心用于接收参数并将它们解决为:class：`_expression.ClauseElement`结构，以便构建SQL表达式对象的系统，已从一组临时函数重写为完全一致的基于类的系统。这个变化对最终用户没有影响，除了在向表达式对象传递错误类型的参数时提供更具体的错误消息。但是，变化是一组更改的一部分，涉及到：func：`_expression.select'对象的角色和行为。

        .. change::
            :tags: bug, mysql

            现在从information_schema.tables系统视图查询以确定特定表格是否存在于MySQL和Mariadb中。以前使用“DESCRIBE”命令，如果检测不存在，则会使用异常处理来检测它，在连接上导致回滚。不宜在所有情况下使用“SHOW TABLES”来解决此问题，但由于MySQL支持的版本现在为5.0.2或更高版本，因此在所有情况下都可用这些信息_schema表格。

        .. change::
            :tags: bug, orm
            :tickets: 5122

            向刷新时处于标识映射子类上且通过  :meth:`~_query.Query.select_entity_from`  或类似技术来提供现有子查询以选择的查询中添加任意条件的能力已添加，它适用于方 法，例如  :meth:` _query.Query.join`  以及加载器选项，例如  :func:`_orm.joinedload` 。此外，该选项的“全局”版本允许限制可应用于查询中的特定实体的条件。


        .. change::
            :tags: renamed, engine
            :tickets: 5244

             :meth:`_reflection.Inspector.reflecttable`  。

        .. change::
            :tags: change, orm
            :tickets: 4662

            当将未解决的对象刷新到具有已经存在于身份映射中的标识时，现在发出警告而不是引发  :class:`.FlushError` 。其原因是以便刷新将继续进行并引发  :class:` .IntegrityError` ，就像表中不存在现有对象一样。这有助于使用:class：`.IntegrityError`捕获行是否已存在于表格中的主键捕获方案。


    .. change::
        :tags: bug, sql
        :tickets: 5004

        重新设计了  :paramref:`.Connection.execution_options.schema_translate_map`  功能，使其在语句的执行阶段而不是编译阶段处理将特定模式名称注入SQL语句的操作。这是为了支持有效地缓存语句。以前，在对几百个模式进行运行的情况下，渲染到某个特定运行的当前模式将被认为是缓存键自身的一部分，这意味着对于运行数百个模式的机器，将具有数百个缓存键，从而使缓存性能大大降低。现在，呈现方式与在1.4中作为  :ticket:` 4645`  ,  :ticket:`4808`  部分而添加的“后编译”呈现方式相似。

    .. change::
        :tags: usecase, sql
        :tickets: 527

        现在可以将操作为  :func:`.true`  and   :func:` .false`  的“逻辑与或”表达式就是这样。支持的后端包括SQLite，PostgreSQL，MySQL/MariaDB和Oracle。

    .. change::
        :tags: feature, engine
        :tickets: 5087, 4395, 4959

        实现了全新的 :class:`_result.Result` 对象来取代以前的 “ResultProxy” 对象。正如在Core中实现的那样，子类：class:`_result.CursorResult`具有与以前的“ResultProxy” 兼容的调用接口，此外还添加了大量可以应用于Core结果集以及ORM结果集（现在集成到相同的模型中）的新功能...class：`_result.Result`包括的特性包括列选择和重新排列，改进的fetchmany模式，去重以及可以用于从内存结构创建数据库结果的各种实现。

        .. seealso::

              :ref:`change_result_14_core` 

    .. change::
        :tags: renamed, engine
        :tickets: 5526

          :class:`~_engine.URL` ~_engine.URL.set` 方法生成新的URL对象。

        .. seealso::

              :ref:`change_5526`  - 迁移说明


    .. change::
        :tags: change, postgresql

        当使用psycopg2数据库适配器的PostgreSQL方言时，将psycopg2的最低版本设置为2.7。psycopg2方言依赖于过去几年中发布的psycopg2的多个特性，因此为了简化方言，现在需要使用于2017年3月发布的版本2.7。

    .. change::
        :tags: usecase, sql

          :func:`.true` .false` 运算符现在可以应用于JOIN的“onclause”，并呈现为“1 = 1”和“1 = 0”。

    .. change::
        :tags: feature, sql, mssql, oracle
        :tickets: 4808

        添加了新的“后编译参数”功能。此功能允许  :meth:`.bindparam`  构造在编译步骤后，但将其值呈现为SQL字符串之前，使用编译器的“文本呈现”功能，从而在传递给DBAPI驱动程序之前将其呈现为SQL字符串。这个功能的直接原因是支持不支持或性能效果差的以数据库驱动程序处理的限制/偏移量方案，同时仍然允许SQLAlchemy SQL构造以其编译的形式缓存的一种方式。新功能的直接目标是SQL Server使用的“TOP N”子句(和Sybase)，这不支持绑定参数， 以及Oracle方言使用的“ROWNUM”和可选的“FIRST_ROWS()”方案，前者已知如果不使用绑定参数更好。该功能建立在用于支持IN表达式的“扩展”参数的机制之上的机制上。作为该功能的一部分，现在在所有情况下不可更改地打开了Oracle ` `use_binds_for_limits`` 功能，现在此标志已弃用。

        .. seealso::

              :ref:`change_4808` 

    .. change::
        :tags: bug, mysql
        :tickets: 5568

        在使用“skip_locked”关键字与``with_for_update()``时，每个MySQL后端现在将呈现“SKIP LOCKED”，说明此将无法在MySQL小于版本8的地方运行并在当前MariaDB后端上失败。这是因为这些后端不支持“SKIP LOCKED”或任何等价物，因此不应默默忽略该错误。这从1.3系列升级为警告。

    .. change::
        :tags: performance, postgresql
        :tickets: 5401

        现在默认情况下，psycopg2方言使用非常高效的``execute_values()``扩展程序来编译INSERT语句，并且在使用此扩展程序时实现了RETURNING支持。这使得即使包含自动增量的SERIAL或IDENTITY值的INSERT语句也可以运行得非常快，同时仍然能够返回新生成的主键值。ORM将在另一个更改中将此新特性集成在一个Web上。

        .. seealso::

              :ref:`change_5401`  - 关于“executemany_mode”参数的完整更改列表。

    .. change::
        :tags: feature, orm
        :tickets: 4472

        添加了一种ir 的方法，允许将任意条件添加到由关系属性在查询中生成的ON子句中，这适用于方法，例如  :meth:`_query.Query.join`  以及加载器选项，例如  :func:` _orm.joinedload` 。另外，该选项的“全局”版本允许全局应用条件限制。

        .. seealso::

              :ref:`loader_option_criteria` 

              :ref:`do_orm_execute_global_criteria` 

              :func:`_orm.with_loader_criteria` 

    .. change::
        :tags: renamed, sql

          :class:`_schema.Table`  ，并发出警告。

    .. change::
        :tags: removed, sql
        :tickets: 4632

        1.3中已弃用的“threadlocal”执行策略已在1.4中删除，以及“引擎策略”的概念和``Engine.contextual_connect``方法。现在在一些场合下仍可用的“strategy ='mock'”关键字参数被替换为DeprecatedWarning；异于此用例时，请使用 :func:`.create_mock_engine` 来替代。


    .. change::
        :tags: mssql, postgresql, reflection, schema, usecase
        :tickets: 4458

        改进了支持cover索引(具有包含列)的功能。 为postgresql添加了 "由Core渲染“包含”子句的CREATE INDEX语句的支持。 对于mssql和postgresql (11+)，索引反射还单独报告包含列。
.. _change_14_0:

Release 1.4.0b1 / October 19, 2019
====================================

.. seealso::

      :ref:`migration_14` 

      :ref:`change_13_0`  - previous release

      :ref:`change_15_0`  - subsequent release

This release includes many improvements and bug fixes.   Please refer to the
changelog for a detailed listing of changes.


What's New / Change Log
========================


SQL Expression Language
-----------------------

- [engine] Improved connection reconnect logic to handle common connection drop scenarios.   `#5187
  <https://github.com/sqlalchemy/sqlalchemy/issues/5187>`_

- [engine] The  :meth:`_future.sa.future.Engine.connect`  method and related methods now raise an   :class:` .OperationalError` 
  exception when the connection is invalidated on checkin as a result of a previous error, rather than always putting
  that transaction into a rollback-only state.   This allows the exception to properly propagate from the calling point,
  so that an application can catch the exception in order to restore the transaction, which works particularly well
  in the context of the context-manager based API.  `#4879 <https://github.com/sqlalchemy/sqlalchemy/issues/4879>`_

- [engine] The Disconnection Detection methodology used by   :class:`_engine.Engine`  has now been isolated
  to the new top-level component   :class:`_engine.DisconnectionDetector` .  Connection and engine now subclass
  this component.   The component itself can be subclassed to customize the behavior of disconnection detection.
  Two new events are introduced "handle_disconnects" and "dbapi_error", which allow custom handlers to be applied in
  response to disconnect as well as raw DBAPI level error conditions.  `#5451 <https://github.com/sqlalchemy/sqlalchemy/issues/5451>`_

- [engine] Deprecated  :paramref:`_engine.Options.pool_threadlocal` .  This parameter has been a no-op since
  SQLAlchemy 1.4 as use of threadlocal pooling has always been the default.  The parameter will immediately raise
  a   :class:`_exc.ArgumentError`  to alert users that the parameter is no longer used.  ` #5416 <https://github.com/sqlalchemy/sqlalchemy/issues/5416>`_

- [postgresql] The PostgreSQL FASTAPI implementational package has been updated to version 0.24.  Most backend-specific features
  of this dialect have been removed in favor of using the dbapi itself to generate PG-specific commands.   The result is the faster
  version of the driver is now supported consistently across a wider variety of PostgreSQL use cases. Additionally, the 
  standard usage of all SQLAlchemy datatypes is now supported in combination with the psycopg2cffi driver.   Use of psycopg2cffi allows
  this driver to be used on CPython 3.5 and above, not just PyPy.  `#5299 <https://github.com/sqlalchemy/sqlalchemy/issues/5299>`_

- [postgresql] Added support for   :class:`.JSON`  and   :class:` .JSONB`  datatypes on PostgreSQL when using psycopg2cffi exclusively.
  These datatypes continue to use Python's JSON parser and are equivalent to using   :class:`_types.JSON`  on regular Python.
  `#5259 <https://github.com/sqlalchemy/sqlalchemy/issues/5259>`_.

- [postgresql] Added support for SQL/MED standard remote table access.   Provides for a fully compatible
  declarative API, so that classes referred to as "remote tables" can appear in joins, relationships as though they were
  local tables.   Also supports remote schema reflection and creation, creation of foreign servers and user-mapping.
  Refer to  :doc:`postgresql_sql_med`  for general information on how SQL/MED works.

- [postgres] The Postgres "python" driver, which is based on the open-source "pg8000" driver, is now officially deprecated.
  This is driven by the maintainer's decision to retire the pg8000 driver.  Moving forward, the "psycopg2" and "asyncpg"
  drivers remain the recommended drivers for PostgreSQL.


ORM
---

- [orm] ORM now automatically tracks in Python all SQL executions that a flush operation uses, in addition to
  those that are ultimately emitted.   When flush() emits SQL, the new method  :meth:`_orm.Session.get_transaction` 
  allows the user to selectively retrieve all SQL emitted within the transaction, in a dictionary keyed to the
    :class:`_engine.Connection`  used to perform the statement.
  Additionally, when a flush operation experiences an error, `Session.rollback()` will now report
  the full SQL emitted leading up to the error in the raised exception message.
  This change is particularly useful in that it allows a new "debugging mode" that can be enabled on the
  connection or per-transaction basis to provide a detailed log of activity in the application, including all SQL statements
  both emitted and attempted/executed`.   `#4777 <https://github.com/sqlalchemy/sqlalchemy/issues/4777>`_

- [orm] Fixed the behavior of the   :class:`_orm.util.AbstractEntity`  base class to correctly apply an explicit primary key
  to an inheritance hierarchy when mapping any child that doesn't itself separate out the primary key columns.  `#5035 <https://github.com/sqlalchemy/sqlalchemy/issues/5035>`_

- [orm] Improved the error message and detection behavior comes when one passes an entity attribute as a parameter into a SELECT.
  An   :class:`_sql.elements.InvalidRequestError`  is emitted when this condition is detected. ` #5119

- [orm] Added  :meth:`_orm.Session.get_bind` , which returns the   :class:` _engine.Engine`  object that represents the target database
  for a given   :class:`_orm.Session` .   This method is intended as a more robust means of acquiring the bind given a session
  that handles more combinations of configurations and features than the existing  :meth:`_orm.Session.connection`  method.
  `#5529 <https://github.com/sqlalchemy/sqlalchemy/issues/5529>`_

- [orm] The  :meth:`_orm.query.Query.get`  method now renders a warning recommending the use of
   :meth:`_orm.query.Query.get_or_404`  and related methods, which provide more detailed information on when a
  NoResultFound is raised.  `#5543 <https://github.com/sqlalchemy/sqlalchemy/issues/5543>`_

- [orm] Added support for raising a custom exception within the ORM that rolls back a transaction as part of the exception handling process,
  using a new method  :meth:`_orm.Session.raise_for_rollback` .   This method allow for the near-total elimination of  :meth:` _orm.Session.rollback` ,
  and leads to easy usage of a pattern where a large chunk of Python code is considered within the context of a single transaction,
  with predictable commit/rollback behavior designed for that specific code block. `#5641 <https://github.com/sqlalchemy/sqlalchemy/issues/5641>`_

- [orm] Added support for "_for_update" style of locking to the ORM   :class:`_orm.query.Query`  object, allowing the query to be executed
  with "FOR UPDATE" or equivalent, producing a locking effect on rows selected by the statement.   The method 
   :meth:`_orm.query.Query.with_for_update`  provides this feature.  ` #5644 <https://github.com/sqlalchemy/sqlalchemy/issues/5644>`_

- [orm] Added support for a new ORM "chained transaction" system which allows for the emission of multiple "subtransactions"
  within the context of a single top level transaction, each of which can be committed independently of each other as well as 
  rolled back as a group.   This feature allows entire sets of work performed within an application to be easily divided up 
  into units that can be committed as discrete steps, with a nested level of transactionality that allows for complex error handling
  scenarios as well. The feature is heavily integrated with  :meth:`_orm.Session.begin_nested`  and  :meth:` _orm.Session.rollback` 
  and so is compatible with much existing transactional code.  `#5658 <https://github.com/sqlalchemy/sqlalchemy/issues/5658>`_

- [orm] Bugfix: Fixed a regression in 1.3 where joined inheritance subclasses which contain another table or polymorphic union above
  them would not be eager loaded properly when used in a join.   `#5494 <https://github.com/sqlalchemy/sqlalchemy/issues/5494>`_

- [orm] Fixed a bug whereby passing more than one column expression to  :meth:`_orm.query.Query.group_by`  would raise UnmappedColumnError.  Ideally,
  a group_by parameter set in this way generates error messages indicating that it is not the preferred way to use group_by, as 
  using unaliased column names causes confusion in the generated SQL and is easily misused by being passed in the wrong order.
  `#5522 <https://github.com/sqlalchemy/sqlalchemy/issues/5522>`_.

- [orm] Fixed a regression relevant to   :class:`_orm.aliased`  whereby a deadlock would occur when the aliased object were a joined
  subclass object.   `#5742 <https://github.com/sqlalchemy/sqlalchemy/issues/5742>`_

- [orm] Added the ability to raise an explicit exception when a flush operation encounters a failure, but only for certain
  exceptions such as "unique constraint".  In order to support this feature, a new event "flush_error" has been added which
  will be emitted whenever a top level flush operation encounters a database error.   The event is passed a dictionary of 
  information about the flush operation including the transaction, the Session object, the states involved, and the exception itself.
  Event handlers can check for specific types of errors, as well as set a bool flag indicating that the flush should be stopped.
  Exceptions that are considered unrecoverable will continue to propagate as before.  `#5418 <https://github.com/sqlalchemy/sqlalchemy/issues/5418>`_

- [orm] All ORM query objects now have a "_with_invoke_all_eagers" flag, when set to False overrides the usual behavior of the ORM
  to always perform eager loading of all related collections and scalar relationships, in a depth-first, breadth-last pattern.
  This new flag, applied in an end-user context via the new   :method:`_orm.Query.with_invoke_all_eagers`  method,
  will turn off eager loading behavior temporarily across all related entities.  This change is aimed towards use with
  more complex ORM-based applications which require more fine-grained control over how eager loading behavior occurs
  for particular queries.  `#5754 <https://github.com/sqlalchemy/sqlalchemy/issues/5754>`_

- [orm] Added the flag "single_parent" to  :meth:`_orm.relationship`  in order to allow a unidirectional relationship from a child object
  to exactly one parent object, without the use of a many-to-one relationship in the parent.  The flag has a few other quirks,
  but supports the usual features such as delete and load cascades as well as querying. `#5755 <https://github.com/sqlalchemy/sqlalchemy/issues/5755>`_

- [orm] Reworked the behavior of unit-of-work deletion so that it uses the same set of state transitions as that used for
  persistence and update.  The net effect of this is the addition of a "deleted" flag as opposed to a "weak-deleted" flag,
  which is no longer necessary, as well as the reworking of the algorithm for handling self-referential trees of objects so 
  that they may be excluded before the DELETE statement is emitted for higher efficiency.  The flag "cascades" is now the
  authoritative flag regarding which relationships to follow for a particular delete operation.  `#5814 <https://github.com/sqlalchemy/sqlalchemy/issues/5814>`_

- [orm] The  :meth:`_orm.Mapper.compile`  method now allows for a "self-referential recursive walker" as used in  :meth:` _orm.query.Query._compile_context` , 
  allowing a Mapper to be compiled with recursion detected in a related-walker style.   This feature becomes relevant
  when compiler plugins are in use that may require for a joined-relationship's child object to be available to the parent
  object's mapper as it is being compiled.  An example usage is when using SQL expressions as some child object's attributes.
  `#5837 <https://github.com/sqlalchemy/sqlalchemy/issues/5837>`_

- [orm] Improved the error message when the "secondary" mapper passed to  :meth:`_orm.relationship`  is incorrect.  The error now
  indicates the correct class that should be passed.  `#5864 <https://github.com/sqlalchemy/sqlalchemy/issues/5864>`_


Improved Tests
---------------

- [test] Changed the "asyncpg" driver to be used for the CI testing for PG 11 and up, rather than psycopg2cffi.  `#5690 <https://github.com/sqlalchemy/sqlalchemy/issues/5690>`_

 Deprecated / Changed Behavior
 ------------------------------

- [orm] The  :meth:`_orm.Session.rollback`  method is now deprecated in favor of using  :meth:` _orm.Session.raise_for_rollback` 
  to raise a rollback exception as part of the exception handling behavior of SQLAlchemy.  The existing pattern of checking a
    :class:`.StatementError`  for the string "ROLLBACK" will continue to be fully supported, as will any other framework or application patterns
  that work with greater dynamics via exception handling rather than using the explicit rollback method.  `#5641 <https://github.com/sqlalchemy/sqlalchemy/issues/5641>`_

- [orm] The   :class:`_orm.InstanceState`  class has been moved from private to public, and is no longer considered
  an implementation detail.  Code that may have used its __dict__ directly is now encouraged to use its public
  accessor methods.  `#5786 <https://github.com/sqlalchemy/sqlalchemy/issues/5786>`_

- [orm] `Query.get` along with several variant methods has been deprecated.   The new method  :meth:`_orm.query.Query.get_or`  provides a
  more fine-grained approach to handling the possibility of multiple results which allows for an exception to be raised or 
  a default value to be returned based on a specified condition.  `.get_or_404` and other specialized methods for throwing exceptions
  have been added as well.  References to the get method that do not specify arguments are raised as a deprecation warning.
  `#5543 <https://github.com/sqlalchemy/sqlalchemy/issues/5543>`_

- [orm] The  :meth:`_orm.query.Query.group_by`  method now raises a   :class:` _exc.ArgumentError`  exception when passed more than one column in an 
  iterable, as this style of usage produces hard-to-use SQL constructs.  The recommended method is to use composition in a list as per
  the usual usage, or to pass all columns as positional arguments.  `#5522 <https://github.com/sqlalchemy/sqlalchemy/issues/5522>`_

- [orm] The  :meth:`_orm.load_only`  method is now considered private; the official public expression is
   :meth:`_orm.defer`  with the "passive=True" flag.  ` #5463 <https://github.com/sqlalchemy/sqlalchemy/issues/5463>`_

- [orm] The  :meth:`_orm.isinstance`  utility method is now considered obsolete.  Use Python's built-in "isinstance", which also accounts
  for custom metaclasses that might be in use.  `.isinstance` now emits a deprecation warning before calling `id`/`__class__` directly.
  `#4903 <https://github.com/sqlalchemy/sqlalchemy/issues/4903>`_

- [orm] Passing non-metadata keyword arguments to   :class:`_orm.Mapper`  is deprecated.  These keyword arguments 
  historically were given support for purposes of backwards compatibility with prior versions of SQLAlchemy.  `#5713 <https://github.com/sqlalchemy/sqlalchemy/issues/5713>`_

- [orm] The  :paramref:`_orm.Mapper.version_id_col`  argument of   :class:` _orm.Mapper`  is now considered deprecated, meaning it is not intended
  to be used towards forthcoming features, although the feature remains fully functional.   The newer approach is to use the "version"
  extension with  :meth:`_orm.mapper.configure` .  ` #3781 <https://github.com/sqlalchemy/sqlalchemy/issues/3781>`_

- [engine] The  :meth:`_future.sa.future.Connectable.connect`  method has been deprecated in favor of using the constructor argument 'connect'
  in order to specify that the connection should not be performed immediately.  `#5678 <https://github.com/sqlalchemy/sqlalchemy/issues/5678>`_

- [engine] The deprecated event aliases ``after_fork``, ``after_invalidate`` and ``pool_reset`` are removed, while the
  original event names remain available.

- [postgresql] Removed the Postgresql "pg8000" dialect along with its specificity to a particular driver.  SQLAlchemy's own deprecated
  pg8000 implentational component was removed in 1.3; as a pure-Python third-party driver with common DBAPI compatibility for whole-driver 
  replacement it can still be used with the psycopg2 dialect if so desired.  `#5299 <https://github.com/sqlalchemy/sqlalchemy/issues/5299>`_

- [postgresql] The functionality of   :class:`_psycopg2.PGBinary`  is now available in unadorned form with psycopg2 2.8.   SQLAlchemy's own
    :class:`_psycopg2.PGBinary`  was deprecated in 1.3.  ` #5532 <https://github.com/sqlalchemy/sqlalchemy/issues/5532>`_

Bugs Fixed
----------

- [bug] Fixed migration issue where the 1.0 sequence batch restructure code would not work for a table that had a long
  PRIMARY KEY or UNIQUE constraint, as well as if the schema contained a large number of columns.
  Regression occurred in SQLAlchemy 1.3. `#5283 <https://github.com/sqlalchemy/sqlalchemy/issues/5283>`_

- [bug] Fixed a regression in 1.3 whereby an   :class:`_orm.exc.InvalidRequestError`  would be raised internally when the flag
  "with_polymorphic" combines an abstract and a concrete entity, instead of being silently ignored.  `#5737 <https://github.com/sqlalchemy/sqlalchemy/issues/5737>`_

- [bug] Fixed a regression since SQLAlchemy 1.3.18 whereby SQLite throws a generic `DatabaseError: database disk image is malformed`
  exception instead of a `DatabaseError: near "C": syntax error` if a hash symbol followed by a number was used as a column name.
  A mapping dictionary is now used to determine if a column name needs to be escaped, removing the usage of Python's `str.isdigit()` method.
  `#5788 <https://github.com/sqlalchemy/sqlalchemy/issues/5788>`_

- [bug] Fixed bug whereby a wrongly named alias generated by a select statement could make its way into a subsequent query
  and cause an assertion error. `#5736 <https://github.com/sqlalchemy/sqlalchemy/issues/5736>`_

- [bug] The   :class:`_types.JSON`  column type now correctly emits a warning, rather than an error, when the dictionary passed contains non-keywords.
  Due to backwards compatibility, these unrecognized objects are still included on output.  `#5839 <https://github.com/sqlalchemy/sqlalchemy/issues/5839>`_

- [bug] Fixed bug whereby a parameter within a  :meth:`_orm.Query.group_by`  construct that is correlated to an alias within
  a subsquery might fail to render properly. `#5394 <https://github.com/sqlalchemy/sqlalchemy/issues/5394>`_

- [bug] Fixed issue with Union where sqlalchemy would attempt to render the DISTINCT keyword twice, given that
  a UNION statement implicitly provides a UNDISTINCT keyword.  The fix makes Union's logic consistent with that of CompoundSelect.
  This issue was introduced in SQLAlchemy 1.3.20.  `#5832 <https://github.com/sqlalchemy/sqlalchemy/issues/5832>`_

- [bug] An issue with handling decimal literals as well as digits column values throughout a result set has been fixed.  The issue
  could result in incorrect digit values returned in certain cases.  `#5474 <https://github.com/sqlalchemy/sqlalchemy/issues/5474>`_.

- [bug] Fixed a race condition that could occur with cleanup of connection pools due to the way that
  connections were checked in and out of the connection pool to ensure that inactive connections
  would correctly get closed.  This would only manifest when the connection pool was emptied at the
  same time as connections were being checked out, for example, during application shutdown. `#4589 <https://github.com/sqlalchemy/sqlalchemy/issues/4589>`_

- [bug] Fixed error when using __len__ with eager load subquery join.
  The subquery expression was not properly being mutated per invocation
  of len().  `#5856 <https://github.com/sqlalchemy/sqlalchemy/issues/5856>`_

- [bug] Fixed bug whereby an assertion would fail when using WITH RECURSIVE and batching enabled at the same time
  and a limit established in the "outer" SELECT, typically producing a "Missing "recursive" parameter" error message. `#5833 <https://github.com/sqlalchemy/sqlalchemy/issues/5833>`_

- [bug] Fixed issue where a join with a non-mapped table, when used with a query that had a string-based FROM entity,
  could fail to render correctly.  The behavior now matches when a mapped entity is used; a join with a non-mapped entity
  is rendered using the legacy joined-table method.  `#5849 <https://github.com/sqlalchemy/sqlalchemy/issues/5849>`_

- [bug] Fixed behavior in 1.3 whereby an object added to an identity map and subsequently deleted from a
  session would not emit a flush event for deletion as it was assumed flushed when added.  A check has been added
  so that the deleted event will now emit according to actual database state.  `#5478 <https://github.com/sqlalchemy/sqlalchemy/issues/5478>`_

- [bug] Fixed regression since 1.3.18 which caused Sqlite to raise an error when using `TableOverlay` with a reflection.  `#5802 <https://github.com/sqlalchemy/sqlalchemy/issues/5802>`_.


Improved Documentation
----------------------

- [doc] Added a new tutorial "Organizing Query-Related Code" which offers a variety of techniques for organizing query code
  that uses select expressions, including standard queries, parameterized queries, object queries and plain SQL.
  The tutorial aims to demonstrate ways to make query code clearer, easier to test, and less likely to duplicate.
  `#5871 <https://github.com/sqlalchemy/sqlalchemy/issues/5871>`_

- [doc] Improved the documentation for sample input and output for various methods of execution, including more 
  explicit text calling out execution status codes. `#5656 <https://github.com/sqlalchemy/sqlalchemy/issues/5656>`_

- [doc] Multiple clarifications and improvements to the documentation related to how secondary features such as connection pooling,
  connection management, transaction management, and other backend-specific features operate within SQLAlchemy. `#5638 <https://github.com/sqlalchemy/sqlalchemy/issues/5638>`_

- [doc] Clarifications to the relation between threading and connection pools in   :ref:`core_engines_using_connections` . ` #5648 <https://github.com/sqlalchemy/sqlalchemy/issues/5648>`_

- [doc] Added a tutorial on SQL expression fundamentals.  Starting with   :func:`_expression.select` , the tutorial shows in detail all 
  the core SQL expression constructs, accompanied with full Python code examples.  `#5027 <https://github.com/sqlalchemy/sqlalchemy/issues/5027>`_

- [doc] The recipes section has been moved off of the main documentation home page and merged into a new documentation section.  
  The new recipe section provides more detailed documentation of specific techniques along with full runnable code examples.
  `#5639 <https://github.com/sqlalchemy/sqlalchemy/issues/5639>`
.. _change_1_4_0:

Version 1.4.0

.. note::

  SQLAlchemy 1.4 is now fully compatible with Python 3.6 through Python 3.10.


Deprecations, Behavior Changes, and Feature Removals
---------------------------------------------------

- The  :meth:`_expression.SelectBase.as_scalar`  and
   :meth:`_query.Query.as_scalar`  methods have been renamed to
   :meth:`_expression.SelectBase.scalar_subquery`  and
   :meth:`_query.Query.scalar_subquery` , respectively. The old names
  continue to exist within 1.4 series with a deprecation warning. In
  addition, the implicit coercion of   :class:`_expression.SelectBase` ,
    :class:`_expression.Alias` , and other SELECT oriented objects into
  scalar subqueries when evaluated in a column context is also
  deprecated, and emits a warning that the
   :meth:`_expression.SelectBase.scalar_subquery`  method should be called
  explicitly. This warning will in a later major release become an
  error, however the message will always be clear when
   :meth:`_expression.SelectBase.scalar_subquery`  needs to be
  invoked.

- The   :class:`_mssql.DATETIMEOFFSET`  datatype has been corrected to
  be based on the   :class:`.DateTime`  class hierarchy.

- The   :class:`_engine.Connection`  object will now not clear a rolled-back
  transaction until the outermost transaction is explicitly rolled back.
  This is intended to help with testing scenarios.

- Calling the  :meth:`_query.Query.instances`  method without passing a
    :class:`.QueryContext`  is deprecated. The  :meth:` _query.Query.from_statement` 
  method should be used instead.

- The   :class:`_schema.Table`  class raises a deprecation warning when
  columns with the same name are defined. A new parameter
   :paramref:`_schema.Table.append_column.replace_existing`  was added to
  the  :meth:`_schema.Table.append_column`  method.

- The  :meth:`_expression.ColumnCollection.contains_column`  method should
  be called with the ``in`` operator instead of passing the column name
  as a string.

- The ``case_sensitive`` flag on   :func:`_sa.create_engine`  is deprecated.
  All string access for a row should be assumed to be case sensitive just
  like any other Python mapping.

- The   :func:`_sql.tuple_`  construct now behaves predictably when used in a
  columns-clause context. The SQL tuple is not supported as a "SELECT" columns
  clause element on most backends; on those that do, the Python DBAPI does not
  have a "nested type" concept. Use of   :func:`_sql.tuple_`  in a
    :func:`_sql.select`  or   :class:` _orm.Query`  will now raise a
    :class:`_exc.CompileError`  at the point at which the
    :func:`_sql.tuple_`  object is seen as presenting itself for fetching rows.

- SQL statements against PostgreSQL, Oracle and MSSQL support
  ``FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}``.

- Added  :meth:`.Query.count_by` , a more flexible alternative to
   :meth:`.Query.count` . Old usages of the positional argument for
   :meth:`.Query.count`  will emit a deprecation warning.

- Added support to ``.get()`` for pessimistic loading given a query option of
  ``load_only()`` or ``defer()``.

- Use of strings for relationship names are deprecated for methods such as
   :meth:`_orm.Query.join`  and should be replaced with the class-bound attribute.
  Additionally, the ``aliased`` and ``from_joinpoint`` parameters to
   :meth:`_orm.Query.join`  are deprecated. The   :func:` _orm.aliased` 
  construct now provides for a great deal of flexibility and capability,

- The following features have been removed:

  - Jython support;
  - deprecated loader options ``joinedload_all``, ``subqueryload_all``,
    ``lazyload_all``, ``selectinload_all``;
  - ``comparable_property``;
  - ``compile_mappers``;
  - deprecated method ``collection.linker``;
  - deprecated method ``Session.prune`` and parameter
    ``Session.weak_identity_map``;
  - deprecated parameter ``mapper.order_by``;
  - deprecated parameter ``Session._enable_transaction_accounting``;
  - deprecated parameter ``Session.is_modified.passive``;
  - deprecated class ``Binary``;
  - deprecated methods ``Compiled.compile``,
    ``ClauseElement.__and__``, ``ClauseElement.__or__`` and
    attribute ``Over.func``;
  - deprecated parameter ``text.bindparams`` and ``text.typemap``;
  - deprecated parameter ``Table.useexisting``.


Improved Behavior
-----------------

- The ORM now sets up eager loaders to be invoked during the refresh on an
  expired object.

- The   :class:`_schema.Column`  can't be used as the key in a result set row
  lookup when that   :class:`_schema.Column`  is not part of the SQL selectable
  that is being selected.

- A deprecation warning now emits when the ORM loads a row for a polymorphic
  instance that has a primary key but the discriminator column is NULL, as
  discriminator columns should not be null.

- Custom functions that are created as subclasses of
    :class:`.FunctionElement`  will now generate an "anonymous label" based on
  the "name" of the function.

- Custom compiler constructs created using the  :mod:`sqlalchemy.ext.compiled` 
  extension will automatically add contextual information to the compiler
  when a custom construct is interpreted as an element in the columns clause
  of a SELECT statement.

- Support for expanding IN expressions has been improved.

- The maximum buffer size for the   :class:`.BufferedRowResultProxy`  can now be
  set to a number greater than 1000.

- The ``readonly`` and ``deferrable`` flags are now supported on the
  following dialects: PostgreSQL, Oracle and MSSQL.

- SQLAlchemy 1.4 now has a separate version of the
    :class:`~sqlalchemy.orm.declarative`  system under ` `sqlalchemy.orm`` to
  enable full integration of Declarative with 3rd party class attribute systems
  like ``dataclasses`` and ``attrs``.

Backwards Incompatibilities, notes for upgrading
------------------------------------------------

- Call the  :meth:`_expression.SelectBase.scalar_subquery`  for the method
  formerly known as  :meth:`_expression.SelectBase.as_scalar` .

- Call the  :meth:`_query.Query.scalar_subquery`  for the method formerly known as
   :meth:`_query.Query.as_scalar` .

- Use a subquery for cases where a select() construct is used in the FROM
  clause of another select() construct.

- Pass classes instead of string names to ORM operations such as
   :meth:`_orm.Query.join`  and   :func:` _orm.selectinload` .

- Use the   :class:`.Subquery`  construct instead of the
    :class:`_expression.Alias`  construct to create named subqueries.

- The usage of the string distinct in non-MySQL dialects, as well as the
  ``DISTINCT ON`` feature in dialects other than PostgreSQL are deprecated.

- The ``name`` parameter is required when creating a   :class:`.Sequence`  for
  PostgreSQL, Oracle and MariaDB >= 10.3.

- A warning is emitted when a   :class:`_schema.Table`  is defined with columns
  that have the same name. Use the new parameter
   :paramref:`_schema.Table.append_column.replace_existing`  to replace such a
  column.

For more information about upgrading, see   :ref:`migration_1_4` .


New Features and Enhancements
-----------------------------

Expressions and SQL
^^^^^^^^^^^^^^^^^^^

-   :func:`_expression.select`  now supports ` `FETCH {FIRST | NEXT} [ count ]
  {ROW | ROWS} {ONLY | WITH TIES}``.

-   :class:`_functions.now`  and   :class:` _functions.current_timestamp`  now
  support the ``timezone`` parameter on Oracle.

- The   :func:`_sql.or_`  and   :func:` _sql.and_`  functions now accept any
  number of SQL expressions.

ORM
^^^

-  :meth:`_orm.Query.count_by` , a more flexible alternative to
   :meth:`_orm.Query.count` .

-  :meth:`.Query.all_in`  method, a less strictly-typed version of
  `Query.filter_by`.

-   :func:`_orm.with_loader_criteria` , a context manager to define
   :meth:`.Query.filter`  rules temporarily.

- A new parameter  :paramref:`_orm.relationship.overlaps`  has been added to
  indicate that an overlapping persistence arrangement may be unavoidable.

- The ORM now allows calling a method on an unloaded attribute, setting its
  value, then committing the session without triggering a SELECT.

- ORM objects attribute values with get/set/delete methods in a subclass are
  now considered *modified* when they are set to a new value, and will be
  marked by the UnitOfWork for persistence on the next flush.

- The  :meth:`_orm.Session.get_bind`  method is now instrumented with the new
  parameter ``attr``, allowing the hook function to inspect the attribute of
  the ORM object as a context for determining what bind to return.

- The  :meth:`.Query.update`  and  :meth:` .Query.delete`  methods now
  accommodate for the discriminator column needed for limiting the statement
  to rows referring to the specific subtype requested.

-   :class:`_orm.Mapper`  now has the ` `use_get`` parameter, to determine if the
  flush should use the overloaded get method to load instances.

-   :class:`_orm.attributes.CollectionAttributeImpl`  now allows modification
  of the state of an unloaded collection.

- The deferred load of a large scalar, e.g. a BLOB, to support a
  `joined_table` subclass flush when also using the Subtypes feature will now
  not issue subsequent SELECTs after the load has completed.

- The  :meth:`_orm.Session.execute`  method now supports the ` `bind_arguments``
  parameter instead of passing keyword arguments.

-  :meth:`_orm.attributes.QueryableAttributeImpl.expression`  now accepts a
    :class:`_expression.FromClause`  in addition to   :class:` _orm.Mapper` .

- The   :func:`_orm.selectinload`  and   :func:` _orm.subqueryload`  constructs
  now support the   :class:`_expression.FromClause`  object instrumented with a
  mapper as an argument.

- Enhancements are made to   :func:`_orm.relationship`  to improve the
  performance of related table probing operations.

General Changes and Improvements
--------------------------------

- SQLAlchemy now logs the query plan when ``EXPLAIN`` is used.

- When an incompatible engine is detected, SQLAlchemy raises a
    :class:`_exceptions.NotImplementedError`  exception.

-   :func:`_expression.case`  and   :func:` _expression.cast`  have been added to
  the  :attr:`.func` .

- A new   :class:`.QueryContext`  mechanism replaces the old
  `Query.values(params, inline=True)` API.

- SQLAlchemy now emits a warning when invoking a session from a thread
  different than the one it was created in.

- The   :class:`_engine.URL`  ` `query`` parameter now accepts
  space-separated query values to construct multi-value parameter
  strings; the existing syntax of providing values as a list or
  tuples is still supported.

- Added support to   :func:`_schema.Index`  for MySQL.

- Added support for MariaDB Connector/Python to the MySQL dialect.

Bugfixes
--------

- Fixed a bug in the ``update()`` method where SQL expressions
  would not be generated correctly when referencing the primaryjoin
  criterion of relationship.

- Fixed issue in the ORM where multi-level "joined" lazy-loading with
  composite foreign keys would emit incorrect queries.

- Fixed several issues where UNION expressions within an ORDER BY with
  facets would emit incorrect SQL.

- Fixed an issue where reusing a Connection context with a different
  DBAPI connection raises a AttributeError: '_ConnectionFairy' object
  has no attribute 'execute'.

- Resolved a race condition in the Connection event listeners, which would
  allow for an especially timed cancellation of a statement to deregister a new
  GUID and issue a statement with the same GUID.

- Fixed an issue where ORDER BY expressions containing correlated subqueries
  could be discarded under certain conditions.

- Fixed an  issue where floated values passed to parameters were treated as
  NULL.

- Fixed an issue where the MySQL dialect would encounter error 2014 when
  using non-blocking cursors with this database service.

- The reflection of ENUM values on MySQL is now case-insensitive.

- Fixed an issue where the ``compare()`` method on a   :class:`_schema.Column` 
  would not compare collation.

- Fixed an issue where slicing a   :class:`_expression.Query`  twice
  subsequently produced different results.

- The memory usage of the postgresql.advisory_lock context manager of the
  PostgreSQL dialect has been optimized.

- Fixed an issue where the PostgreSQL dialect had errors using text() with
  an UPDATE statement.

- Fixed an issue where loading query options from an ORM query directly when
  applying them on another query would produce an error.

- Corrected a string formatting issue in the MySQL dialect.

- Corrected a bug in the light-weight migration feature introduced in 1.3.21
  where running a migration script created a database schema.

- Fixed an issue in the ORM where a circular dependency between two entities
  referencing the same table would fail to create database tables in the
  correct order.

- Improved the deprecation warning for unsupported platform sqlite3 versions.

- The column name in the form of string should no longer be passed to the
   :meth:`_connection.Connection.execution_options` 

- Fixed an issue where the reflection of the "default" value of a BIGINT
  column on MySQL was returning a 32 bit integer instead of a 64-bit integer.

- Resolved a bug where the Oracle dialect didn't treat the result of
  ``DBMS_LOB.SUBSTR()`` as `LOB` object.

- Fixed an issue where the SQLite dialect would not raise an error for
  conflicting CHECK constraints.

- Fixed an issue where NULL and other scalar values with tuple-typed or
  array-typed columns in a result set with SQLite would be coerced with an
  inappropriate length.

- A workaround is added for an issue in MySQLdb when using a DISTINCT and
  other SELECT constructs together.

- Fixed an issue where the   :class:`_orm.ScopedSession`  instance would not always
  propagate the configured session to the next generation.

- The SQLite dialect now supports Python's standard "memory" temporary database,
  allowing for its usage when explicitly passed in a URL string.

- Fixed issue where filter-by function fails when calling in a query together
  with the eagerload option.

- Fixed an issue in  :meth:`_engine.create_engine`  where passing in
  ``echo=True, echo_pool=True`` would not enable pool-level echoing.

- Added support for JSON values in SQLite, including the SQLite JSON1
  extension.

- Resolved a regression where  :meth:`_engine.Engine.execute` 
  didn't apply the statement-level `execution_options`.

- Corrected an issue where calling a stored procedure that accepted an array
  (PostgreSQL array) and passed an empty list parameter would generate
  invalid SQL.

- Corrected a behavior issue where a ``CHECK CONSTRAINT`` on PostgreSQL
  relying on the same column twice would fail to resemble in a   :class:`.Table` 
  object.

- Fixed an issue affecting MySQL which was preventing reuse of SQL statements
  with prepared statements with a single value passed.

- Fixed issue in the ORM affecting joined inheritances missing the entity type
  in many-to-one relationships.

- Fixed a regression in  :meth:`_engine.Engine.connect`  where the call would
  trigger a connection on an   :class:`_engine.Engine`  that hasn't been
  previously connected.

- Fixed a bug where using the SQLite dialect to run a ``SELECT`` query would
 only work the first time.

- Fixed an issue where the   :class:`_engine.ConnectionFairy`  could be wrongly
  attached to a connection pool after it represents an erroneous state.

- Updated unique constraint creation on MySQL to only compare the columns
  that are actually present on each side of the comparison.

- Fixed an issue where reflection of legacy indices on MySQL could produce
  warnings.

- Fixed the   :class:`_orm.DeferredReflection`  extension to return an empty set
  when reflecting against an empty database schema.

- Corrected a bug whereby an INSERT..ON DUPLICATE KEY UPDATE statement in MySQL
  would not work correctly if a sequence was used as the linked sequence.

- Fixed an issue where the sqlite3 dialect was not properly cleaning up the dbapi
  connection.

- The Oracle dialect fits LOB values larger than 4 MB Oracle's LOB limits... change::
        :tags: change, orm
        :tickets: 4617
        
        当ORM在隐式地将   :func:`_expression.select`  构造管理为一个子查询时，现在将会发出警告。这发生在一些地方，例如   :meth:` _query.Query.select_entity_from`   和  :meth:`_query.Query.select_from`  方法，以及在   :func:` .with_polymorphic`  函数内。 当   :param:`_expression.SelectBase` (这是   :func:` _expression.select`  产生的内容)或类似   :class:`_query.Query`  的对象被直接传递给这些函数和其他函数时，ORM 通常通过自动调用  :meth:` _expression.SelectBase.alias`  方法 (现在被  :meth:`_expression.SelectBase.subquery`  方法替代) 将其强制转换为一个子查询。请参阅下面链接的迁移说明以获取更多详细信息。

        .. seealso::

              :ref:`change_4617` 

    .. change::
        :tags: bug, sql
        :tickets: 4617

          :class:`_selectable.CompoundSelect`  的 ORDER BY 子句，例如 UNION, EXCEPT等，在对绑定到   :class:` _schema.Table`  的列的  :meth:`_selectable.CompoundSelect.order_by`  进行应用时，将不再呈现与给定列关联的表名。大多数数据库要求ORDER BY子句中的名称仅表示标签名称，这些标签名称与第一个 SELECT 语句中的名称匹配。 更改与  :ticket:` 4617`  相关，因为以前的解决方法是在引用 ``.c`` 属性的时候指定   :class:`_selectable.CompoundSelect`  的子查询 ，以便获得没有表名的列。由于现在子查询已命名，此更改允许继续使用这种解决方法，并允许在  :meth:` _selectable.CompoundSelect.order_by`  方法中使用绑定到表的列以及  :attr:`_selectable.CompoundSelect.selected_columns`  集合。

    .. change::
        :tags: bug, orm
        :tickets: 5226

        现在，如果已过期的对象的过期属性列表中包含一个或多个使用  :meth:`.Session.expire`  或  :meth:` .Session.refresh`  方法明确过期或刷新的属性，刷新将触发自动刷新。这是试图找到正常未过期的属性在许多情况下不需要自动提交的情况和那些属性正在明确过期或刷新的情况之间的中间状态，而这些属性可能依赖于会话中需要刷新的其他挂起状态。这两种方法现在还增加了一个新标志  :paramref:`.Session.expire.autoflush`  和  :paramref:` .Session.refresh.autoflush` ，默认值为 True； 当设置为 False 时，这将禁用在未过期时自动提交会话的属性的自动提交行为。

    .. change::
        :tags: feature, sql
        :tickets: 5380

        随着作为  :ticket:`4369`  的一部分引入的新的透明语句缓存功能的推出，还添加了一个新功能，旨在减少创建语句的Python开销，允许在指示传递给类似 select()、Query()、update() 等语句对象的参数时使用lambda，并允许在lambda中以与“烘焙查询”系统类似的方式构建完整的语句。使用使用Lambda的基础原理来自于“烘焙查询”方法，该方法使用Lambda将任意数量的Python代码封装为一个可调用对象，该对象只需在第一次构建语句为字符串时调用即可。 但是，新功能更加复杂，因为自动提取会传递作为参数的Python文字值，因此不再需要使用带有这些查询的绑定参数(bindparam())对象。使用该功能是可选的，并且可以以任何程度使用它，同时仍然允许语句完全可缓存。

        .. seealso::

              :ref:`engine_lambda_caching` 
            

    .. change::
        :tags: feature, orm
        :tickets: 5027

        添加对直接映射使用 Python ``dataclasses`` 装饰器定义的 Python 类的支持。 由Václav Klusák提供的拉取请求。 新功能在声明式级别上与诸如“dataclasses”和“attrs”等系统的新支持集成。

        .. seealso::

              :ref:`change_5027` 

              :ref:`change_5508` 

    .. change::
        :tags: change, engine
        :tickets: 4710

        ``RowProxy`` 类不再是代理对象，而是直接在构建时填充处理后的 DBAPI 行元组的内容。 现在名为  :class:`.Row` ，Python级别的值处理器的机制已经简化，特别是影响 C 代码格式的机制，以便在最前面将 DBAPI 行处理为结果元组。   :class:` _engine.ResultProxy`  返回的对象现在是 ``LegacyRow`` 子类，该子类保留映射/元组混合行为，但现在   :class:`.Row`  类的基本行为更像命名元组。

        .. seealso::

              :ref:`change_4710_core` 

    .. change::
        :tags: change, orm
        :tickets: 4710

          :class:`_query.Query`  返回的“KeyedTuple”类现在被替换为 Core 的   :class:` .Row`  类，该类的行为与 KeyedTuple 相同。在 SQLAlchemy 2.0 中，Core 和ORM 将使用相同的   :class:`.Row`  对象返回结果行。在此期间，Core 使用一个向后兼容性类 ` `LegacyRow``，该类维护先前由“RowProxy”使用的映射/元组混合行为。

        .. seealso::

              :ref:`change_4710_orm` 

    .. change::
        :tags: feature, orm
        :tickets: 4826

        通过  :paramref:`.orm.defer.raiseload`  参数添加 ORM 映射列的“raiseload”功能。这为只能使用列表达式映射的属性提供了与   :func:` .raiseload`  选项相似的行为，以便在关系映射属性中使用。 此更改还包括有关延迟列的行为更改方面的某些行为方面（有关详细信息，请参见迁移说明）。

        .. seealso::

              :ref:`change_4826` 

    .. change::
        :tags: bug, orm
        :tickets: 5150

        在 SQLAlchemy 2.0 中，会将  :paramref:`_orm.relationship.cascade_backrefs`  参数的行为翻转为始终将其设置为 ` `False``，因此后代属性不会从一个正向赋值向后向分配的操作中被级联保存-更新操作。当操作发生时，默认情况下，如果参数保持在其默认值 ``True``，则会发出2.0弃用警告。可以通过在特定的   :func:`_orm.relationship`  上将标志设置为 ` `False``，也可以通过将  :paramref:`_orm.Session.future`  标志设置为True，一般情况下对其进行设置来设定新行为。

        .. seealso::

              :ref:`change_5150` 

    .. change::
        :tags: deprecated, engine
        :tickets: 4755

        废弃了剩余的引擎级别的内省和实用程序方法，包括  :meth:`_engine.Engine.run_callable` ，  :meth:` _engine.Engine.transaction`  ，  :meth:`_engine.Engine.table_names`  ，  :meth:` _engine.Engine.has_table`  。实用程序方法被现代上下文管理器模式取代，而表内省任务则适用   :class:`_reflection.Inspector`  对象。

    .. change::
        :tags: removed, engine
        :tickets: 4755

        内部方言方法 ``Dialect.reflecttable`` 已被删除。对第三方方言的审核未发现任何使用此方法，因为它已被记录为不应由外部方言使用。此外，还删除了私有 ``Engine._run_visitor`` 方法。

    .. change::
        :tags: removed, engine
        :tickets: 4755

        长期以来已弃用的 ``Inspector.get_table_names.order_by`` 参数已删除。

    .. change::
        :tags: feature, engine
        :tickets: 4755

        现在，  :paramref:`_schema.Table.autoload_with`   参数直接接受   :class:` _reflection.Inspector`  对象，以及之前的任何   :class:`_engine.Engine`  或   :class:` _engine.Connection` 。

    .. change::
        :tags: change, performance, engine, py3k
        :tickets: 5315

        将在 Python 3 下运行时在方言启动时运行的“unicode returns”检查禁用，多年来这是为了测试当前 DBAPI 对于VARCHAR和NVARCHAR数据类型是否返回Python Unicode或Py2K字符串的行为。在 Python 2 下仍默认情况下进行检查，然而，在删除 Python 2支持的 SQLAlchemy 2.0中，测试行为的机制也将被删除。

        当需要时，这个逻辑很有效，但是现在 Python 3 是标准的，所有 DBAPI 都应该返回字符数据类型的 Python 3字符串。如果第三方 DBAPI 不支持此操作，则仍可在第三方方言中使用   :class:`.String`  中的转换逻辑，并且第三方方言可以在其前端方言标志中通过将方言级别标志 ` `returns_unicode_strings`` 设置为  :attr:`.String.RETURNS_CONDITIONAL`  或  :attr:` .String.RETURNS_BYTES`  中的一个来指定这一点，两者都允许在 Python 3 下启用 Unicode 转换。

    .. change::
        :tags: renamed, sql
        :tickets: 5435，5429

        几个运算符已重新命名，以实现更一致的命名方式。

        运算符的更改如下:

        * ``isfalse`` 现在是 ``is_false``
        * ``isnot_distinct_from`` 现在是 ``is_not_distinct_from``
        * ``istrue`` 现在是 ``is_true``。
        * ``notbetween`` 现在是 ``not_between``
        * ``notcontains`` 现在是 ``not_contains``。
        * ``notendswith`` 现在是 ``not_endswith``。
        * ``notilike`` 现在是 ``not_ilike``。
        * ``notlike`` 现在是 ``not_like``。
        * ``notmatch`` 现在是 ``not_match``。
        * ``notstartswith`` 现在是 ``not_startswith``。
        * ``nullsfirst`` 现在是 ``nulls_first``。
        * ``nullslast`` 现在是 ``nulls_last``。
        * ``isnot`` 现在是 ``is_not``。
        * ``notin_`` 现在是 ``not_in``。

        由于这些是核心运算符，因此对于此更改的内部迁移策略是要支持旧术语的扩展使用时间-如果不是无限期地-但更新所有文档、教程和内部使用以使用新术语。使用新术语定义函数，并且已将传统术语弃用为新术语的别名。


    .. change::
           :tags: orm, deprecated
           :tickets: 5192

             :func:`.eagerload`  和   :func:` .relation`  是旧别名，现在已弃用。使用   :func:`_orm.joinedload`  和   :func:` _orm.relationship`  分别替换。

    .. change::
        :tags: bug, sql
        :tickets: 4621

          :class:`_expression.Join`  构造不再将“onclause”作为一个源来自省略包含在   :class:` _expression.Select`  对象的 FROM 列表中的其他 FROM 对象的一个单独的 FROM 对象。这适用于包括对 JOIN 之外的另一个 FROM 对象的引用的 ON 子句；虽然这通常从 SQL 的角度来看是不正确的，但省略它也是不正确的。 更改使   :class:`_expression.Select`  /   :class:` _expression.Join`  的行为更加直观。