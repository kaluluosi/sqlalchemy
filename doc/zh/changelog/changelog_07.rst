0.7版本更新说明

.. 版本::

    :version: 0.7.11
    :released:

    .. 更改::
    
        :tags: bug, engine
        :tickets: 2851
        :versions: 0.8.3, 0.9.0b1

        :func:`~sqlalchemy.engine.url.make_url`函数所使用的正则表达式现在解析ipv6地址，例如被括号括起来的地址。
        
    .. 更改::
    
        :tags: bug, orm
        :tickets: 2807
        :versions: 0.8.3, 0.9.0b1

        修正列表仪表物会没有正确地表示一个切片为“[0:0]”，这在使用“insert(0, item)”和联合代理时特别容易发生。由于Python集合中的某些奇怪特性，该问题在Python 3中比在Python 2中更有可能出现。
        
    .. 更改::
    
        :tags: bug, sql
        :tickets: 2801
        :versions: 0.8.3, 0.9.0b1

        修复自0.7.9的回归，其中CTE的名称如果在多个FROM子句中引用，则不能正确地引用。
        
    .. 更改::
    
        :tags: mysql, bug
        :tickets: 2791
        :versions: 0.8.3, 0.9.0b1

        更新MySQL保留字版本5.5、5.6的信息，感谢Hanno Schlichting。
        
    .. 更改::
    
        :tags: sql, bug, cte
        :tickets: 2783
        :versions: 0.8.3, 0.9.0b1

        修复公共表达式系统中的错误，其中，如果只将CTE用作“alias（）”结构，则不会使用WITH关键字渲染。
        
    .. 更改::
    
        :tags: bug, sql
        :tickets: 2784
        :versions: 0.8.3, 0.9.0b1

        修复:class:`.CheckConstraint` DDL中的错误，在其中，从：class:`_schema.Column`对象继承的“quote”标志不会被传播。
        
    .. 更改::
    
        :tags: bug, orm
        :tickets: 2699
        :versions: 0.8.1

        修复查询形式为:``query(SubClass).options(subqueryload(Baseclass.attrname))``, 其中“SubClass”是“BaseClass”的一个连接inh ，当子查询中没有``JOIN``在属性加载中进行时，会产生卡式积。填写的结果仍然往往是正确的，因为新的一行将被忽略，所以这个问题可能作为应用程序性能下降而存在于其他方面正确工作的应用程序中。
        
    .. 更改::
    
        :tags: orm, bug
        :tickets: 2689
        :versions: 0.8.1

        修复连接继承子类在“子”表中插入行的bug，如果两个表之间没有设置ForeignKey约束，则会先插入“sub”表的行，然后再插入父表的行。
        
    .. 更改::
    
        :tags: feature, postgresql
        :tickets: 2676
        :versions: 0.8.0

        新增支持PostgreSQL的传统SUBSTRING功能语法，使用常规的``func.substring（）``时渲染为“SUBSTRING（x FROM y FOR z）”，感谢Gunnlaugur Þór Briem。
        
    .. 更改::
    
        :tags: bug, tests
        :tickets: 2669
        :pullreq: 41

        修复``test_execute``中的“logging”导入，在某些Linux平台上无法正常工作。
        
    .. 更改::
    
        :tags: bug, orm
        :tickets: 2674

        当检测到“backref loop”时发出的错误消息得到改进，即当属性事件触发两个其他属性之间的双向赋值时，没有结束时。这种情况不仅可能发生在分配错误类型的对象时，而且还可能发生在将属性错误地配置为backref到现有的backref对中时。
        
    .. 更改::
    
        :tags: bug, orm
        :tickets: 2674

        当将MapperProperty分配给替代现有属性的映射器时，警告将被发出，如果涉及的属性不是基于列的属性。更换关系属性很少（几乎没有？）是想要的，通常是指映射器配置错误。如果一个backref配置自己在继承关系上的现有的一个上（这是在0.8中的一个错误），它也会警告。
        
.. 版本::

    :version: 0.7.10
    :released: Thu Feb 7 2013

    .. 更改::
    
        :tags: engine, bug
        :tickets: 2604
        :versions: 0.8.0b2

        修复Meta通的反射，以正确使用给定的Connection，如果有，则不打开从该连接的_engine.Engine中获得的第二个连接。

    .. 更改::   
                
        :tags: mssql, bug
        :tickets:2607
        :versions: 0.8.0b2

        修复了使用"key"与Column一起使用时，在拥有表的"schema"的情况下会由于MSSQL语言环境的"schema rendering"逻辑未考虑.key而无法定位结果行。
                
    .. 更改::  
    
        :tags: sql, mysql, gae
        :tickets: 2649

        为“gaerdbms” dialect添加了一个条件导入，该导入尝试导入rdbms_googleapi和rdbms_apiproxy以在dev和生产平台上工作。 还现在尊重“instance”属性。谢谢Sean Lynch的帮助。 还回传了username/password的增强以及从0.8开始修正错误代码解释。
        
    .. 更改::
    
        :tags: sql, bug
        :tickets: 2594、2584

        更改:class:`.TypeDecorator`的“__repr__”，以便允许:class:`.PickleType`生成干净的“repr()”，以帮助Alembic。
        
    .. 更改::
    
        :tags: sql, bug
        :tickets: 2643

        修复了:meth:`_schema.Table.tometadata`上的错误，即如果:class:`_schema.Column`既具有外键，又具有列的备选“。key”名称，则无法使用该函数，忽略这一点。
        
    .. 更改::
    
        :tags: mssql, bug
        :tickets: 2638

      在与cx_Oracle一起使用时添加了一个Py3K条件，以使在“信息模式”中获取正确的行，修复了在Py3k上执行此函数会出现错误的情况。
    
    .. 更改::
    
        :tags: orm, bug
        :tickets: 2650

      修复潜在的内存泄漏，如果创建任意数量的:class:`.sessionmaker`对象会发生，当与事件调度器一起使用自适应谷类时将无法垃圾回收匿名子类被创建时，当这个子类被引用时，由于事件的类级引用保留，将不被垃圾回收。该问题还适用于与事件分派器一起使用的单独制定的系统，这些系统使用自定义子系统。
    
    .. 更改::
    
        :tags: orm, bug
        :tickets: 2640

      :meth:`_query.Query.merge_result`现在可以加载外键约束条目而该约束可以是 ``None``，没有抛出异常。
    
    .. 更改::
    
        :tags: sqlite, bug
        :tickets: 2568
        
        调整原来的bugfix，以试图解决SQLite问题，该问题已在sqlite 3.6.14之后解决，即在“foreign_key_list”原语中使用表名时将其用引号括起来。该修复已调整为尽可能不干扰具有与列或表名实际相同名称的引号的名称的程度；如果目标表名称实际是在其名称中用引号括起来的名称的一部分（即““mytable””），则sqlite仍将不返回正确的foreign_key_list()结果。
    
    .. 更改::
    
        :tags: sqlite, bug
        :tickets: 2265
        
        内容列的设置类型，例如 Integer，确保将非字符串值转换为字符串，并处理旧的SQLite版本，因为这些版本不返回字符串格式的默认值。
        
    .. 更改::
    
        :tags: sqlite, feature
        :tickets:
    
        添加SQLite执行选项“sqlite_raw_colnames= True”，将绕过尝试从SQLite cursor.description returned的列名中删除“.”。
    
    .. 更改::
    
        :tags: sqlite, bug
        :tickets: 2525
        
        当替换Table的主键列（例如通过extend_existing）时，插入()构造使用的“自动增量”列会被重置。先前，它将继续引用先前的主键列。sessionmaker()实例在与Session类关联后创建。

.. change::
    :tags：orm，bug
    :tickets: 2425

  修复了一个错误，即带有“literal”的primaryjoin条件将
  在一些深度嵌套的表达式上编译时引发错误的问题
  还需要呈现相同的绑定参数名称超过一次。

.. change::
    :tags：orm，功能
    :tickets：

  添加了“no_autoflush”上下文
  管理器到Session，与：一起使用：
  将暂时禁用自动刷新。

.. change::
    :tags：orm，功能
    :tickets: 1859

  在Query中添加了cte（）方法，
  从核心（参见下文）调用公共表达式支持。

.. change::
    :tags：orm，bug
    :tickets: 2403

  删除了做多元映射时检查行数的内容
  针对映射对象。如果ON DELETE
  CASCADE存在于两行之间，我们不能
  从DBAPI中获取精确的行数;
  在任何情况下大多数DBAPI都不支持该计数，
  MySQLdb是值得注意的情况，因为它是。

.. change::
    :tags：orm，bug
    :tickets: 2409

  修复了一个错误，即使用的对象
  attribute_mapped_collection或
  column_mapped_collection无法
  腌制。

.. change::
    :tags：orm，bug
    :tickets: 2406

  修复了一个错误，即MappedCollection
  如果仅在使用了自定义子类中使用，则无法获取适当的集合
  @ collection.internally_instrumented。

.. change::
    :tags：orm，bug
    :tickets: 2419

  修复了一个错误，即SQL调整机制
  在涉及嵌套的情况下会失败
  继承，joinedload（），limit（）和一个
  在列子句中推导函数中有一个衍生函数。

.. change::
    :tags：orm，bug
    :tickets: 2417

  修复了CascadeOptions的repr（）以
  包括refresh-expire。还改写了
  CascadeOptions是<frozenset>。

.. change::
    :tags：orm，功能
    :tickets: 2400

  添加了通过查询表限定列名的功能
  使用query（sometable）。filter_by（colname = value）。

.. change::
    :tags：orm，bug
    :tickets:

  将“声明式反射”示例改进为支持单表继承，
  多次调用prepare（），表
  在替代模式下存在，
  仅建立子集类型的情况。

.. change::
    :tags：orm，bug
    :tickets: 2390

  将外部事务期间应用的测试比例缩小到
  NULL PK内部的UPDATE的行数范围内只是实际的
  如果真的有 UPDATE 进行。
  

.. change::
    :tags：orm，bug
    :tickets: 2352

  修复了错误，其中如果方法名称
  与列名冲突，则在映射器上尝试检查__get __（）方法
  方法对象上会引发TypeError。

.. change::
    :tags：bug，sql
    :tickets: 2427

  修复了使用C扩展时core中的内存泄漏
  当使用特定类型的结果获取时发生，
  特别是当orm查询计数()
  被叫。

.. change::
    :tags：bug，sql
    :tickets: 2398

  修复了在行上基于属性的错误
  会在非C版本中引发AttributeError，
  在C版本中引发NoSuchColumnError。现在
  在这两种情况下都会引发AttributeError。

.. change::
    :tags：feature，sql
    :tickets: 1859

  添加了对SQL标准的支持
  公共表达式（CTE），允许
  选择对象作为CTE源（DML
  尚未受支持）。这是通过
  选择任何select（）构造 cte（）方法。

.. change::
    :tags：bug，sql
    :tickets: 2392

  支持使用Column的.key
  作为结果集行的字符串标识符。 .key目前
  被列标记为“备选”，并被该列的名称取代
  其常规名称具有该键值。为下一个主要版本
  SQLAlchemy，我们可以翻转此优先级
  所以。键拥有优先权，但这个
  尚未决定。

.. change::
    :tags：bug，sql
    :tickets: 2413

  在insert（）或update（）构造的values（）子句中
  列不存在时会发出警告。
  会在0.8中移动到例外情况。

.. change::
    :tags：bug，sql
    :tickets: 2396

  对SELECT语句中的列进行标记的更改
  声明允许“截断”标签，即标签名称
  在Python中生成的名称超过
  最大标识符长度（请注意，这是
  通过使用label_length创建_engine（）配置），
  将在子查询内正确引用，以及使用其原始
  在Python名称中呈现的$result$行中。
  

.. change::
    :tags：bug，sql
    :tickets: 2402

  在reflected表的情况下，修复了新的“autoload_replace”标志中的错误
  如果反映该表的主键约束将无法保留。

.. change::
    :tags：bug，sql
    :tickets: 2380

  当不能将参数解释为列或表达式时，索引将引发
  传递的参数。没有列时警告
  全部创建的索引。

.. change::
    :tags：engine，功能
    :tickets: 2407

  添加了“no_parameters = True”执行
  选项用于连接。如果没有参数
  出现，在cursor.execute（statement）中传递该语句
  从而调用DBAPI的行为，当没有参数时
  在某些情况下，在字符串中不能解释%符号。
  这仅在此选项中发生，而不是
  只有如果参数列表为空，否则
  这将产生SQL表达式的不一致行为
  通常转义百分号（且在编译时无法判断参数是否存在
  在某些情况下）。

.. change::
    :tags：engine，bug
    :tickets:

  向MockConnection（即用于）添加execution_options（）调用
  strategy =“mock”），它充当经过传递
  参数的通道。

.. change::
    :tags：engine，功能
    :tickets: 2378

  在create_engine中添加了pool_reset_on_return参数
  允许控制
  “连接返回”行为。还添加了一个新参数'rollback'，'commit'，None
  到pool.reset_on_return以允许更多的控制
  连接返回活动。

.. change::
    :tags：engine，功能
    :tickets:

  添加了一些不错的上下文管理器
  Engine，Connection ::

      with engine.begin() as conn:
         # <用conn在事务中工作>
         ...

  并且::

      with engine.connect() as conn:
         # <用conn工作>
         ...

  在完成后关闭连接，
  在engine.begin()上错误提交或回滚
  时。

.. change::
    :tags：sqlite，bug
    :tickets: 2432

  修复了C扩展错误，在返回整数作为Numeric值时不会应用参数格式;
  这主要影响SQLite，它不会
  维护数字比例设置。

.. change::
    :tags：mssql，功能
    :tickets: 2430

  增加了对MSSQL INSERT的支持，
  UPDATE和DELETE表提示，使用
  新的with_hint（）方法在UpdateBase上。

.. change::
    :tags：feature，mysql
    :tickets: 2386

  通过将mysql_using参数添加到Index和PrimaryKeyConstraint中，为MySQL索引和
  支持主键约束类型（即USING），由Diana Clarke提供。

.. change::
    :tags：feature，mysql
    :tickets: 2394

  为所有MySQL方言添加了“isolation_level”
  参数的支持。谢谢
  Mu_mind提供的补丁。

.. change::
    :tags：oracle，功能
    :tickets: 2399

  增加了一个新的create_engine（）标志
  coerce_to_decimal = False，停用精度
  通过将所有数字值都转换为
  十进制的处理，会产生很多开销。

.. change::
    :tags：oracle，bug
    :tickets: 2401

  为LONG添加了缺少的编译支持

.. change::
    :tags：oracle，bug
    :tickets: 2435

  将'LEVEL'添加到保留列表中
  用于Oracle。

.. change::
    :tags：examples，bug
    :tickets:

  在Beaker示例中的_params_from_query（）函数进行了更改
  ，从完全编译的语句中提取bindparams，
  作为获取包括子查询在内的所有内容的快速手段
  列子句等。版本：0.7.3
发布日期：2011年10月16日

变化：
- joined和subquery加载现在会遍历已经存在的相关对象和集合，以查找在整个定义的急切加载的范围内没有填充的所有属性。 
- 修复了当在 'columns-only' 的情况下使用mapper.polymorphic_on 时出现的错误。 
- 修复了将column_property（）应用于subquery时+joinedload + LIMIT +按column_property()排序会导致SQL出现问题的问题。 
- ORM现在具有一个经过改进的可替换遍历，可将selectables更改为针对某些内容的别名（即子句适应），其中包括对与joined table结构的多重嵌套any（）/has（）结构的修复。很抱歉，这段文字没有提及任何修改。.. deprecated:: 0.7.0
    The following changes have been deprecated as of version 0.7.0:

    * ScopedSession.mapper is removed.
    * Query.join(), Query.outerjoin(), eagerload(), eagerload_all(), and others no longer allow lists of attributes as arguments (i.e. option([x, y, z]) form).

The following changes have been made in version 0.7.0b2:

.. change::
    :tickets: 2083
    :tags:

    The "implicit_returning" flag on create_engine() is honored if set to False.

.. change::
    :tickets: 2092
    :tags: informix

    Added the "RESERVED_WORDS" informix dialect.

.. change::
    :tickets: 2090
    :tags: ext

    The "horizontal_shard" ShardedSession class accepts the common Session argument "query_cls" as a constructor argument to enable further subclassing of ShardedQuery.

.. change::
    :tickets:
    :tags: examples

    Updated the association and association proxy examples to use declarative, added a new example dict_of_sets_with_default.py, a "pushing the envelope" example of association proxy.

.. change::
    :tickets: 2090
    :tags: examples

    The Beaker caching example allows a "query_cls" argument to the query_callable() function.

The following changes have been made in version 0.7.0b1:

.. change::
    :tickets: 1902
    :tags: general

    New event system supersedes all extensions, listeners, etc.

.. change::
    :tickets: 1926
    :tags: general

    Logging enhancements.

.. change::
    :tickets: 1949
    :tags: general

    Setup no longer installs a Nose plugin.

.. change::
    :tickets:
    :tags: general

    The "sqlalchemy.exceptions" alias in sys.modules has been removed. Base SQLA exceptions are available via "from sqlalchemy import exc".

.. change::
    :tickets: 1923
    :tags: orm

    More succinct form of query.join(target, onclause).

.. change::
    :tickets: 1903
    :tags: orm

    Added Hybrid Attributes and superseded synonym().

.. change::
    :tickets: 2008
    :tags: orm

    Rewrite of composites.

.. change::
    :tickets:
    :tags: orm

    Mutation Event Extension supersedes "mutable=True".

.. change::
    :tickets: 1980
    :tags: orm

    PickleType and ARRAY mutability turned off by default.

.. change::
    :tickets: 1895
    :tags: orm

    Simplified polymorphic_on assignment.

.. change::
    :tickets: 1912
    :tags: orm

    Flushing of Orphans that have no parent is allowed.

.. change::
    :tickets: 2041
    :tags: orm

    Adjusted flush accounting step to occur before the commit in the case of autocommit=True. This allows autocommit=True to work appropriately with expire_on_commit=True, and also allows post-flush session hooks to operate in the same transactional context as when autocommit=False.

.. change::
    :tickets: 1973
    :tags: orm

    Warnings generated when collection members, scalar referents not part of the flush.

.. change::
    :tickets: 1876
    :tags: orm

    Non-Table-derived constructs can be mapped.

.. change::
    :tickets: 1942
    :tags: orm

    Tuple label names in Query Improved.

.. change::
    :tickets: 1892
    :tags: orm

    Mapped column attributes reference the most specific column first.

.. change::
    :tickets: 1896
    :tags: orm

    Mapping to joins with two or more same-named columns requires explicit declaration.

.. change::
    :tickets: 1875
    :tags: orm

    Mapper requires that polymorphic_on column be present in the mapped selectable.

.. change::
    :tickets: 1966
    :tags: orm

    compile_mappers() renamed configure_mappers(), simplified configuration internals.

.. change::
    :tickets: 2018
    :tags: orm

    The "aliased()" function, if passed a SQL FromClause element (i.e. not a mapped class), returns an element alias() instead of raising an error on AliasedClass.

.. change::
    :tickets: 2027
    :tags: orm

    Session.merge() will check the version id of the incoming state against that of the database, assuming the mapping uses version ids and incoming state has a version_id assigned, and raise StaleDataError if they don't match.

.. change::
    :tickets: 1996
    :tags: orm

    Session.connection(), Session.execute() accept 'bind', to allow execute/connection operations to participate in the open transaction of an engine explicitly.

.. change::
    :tickets:
    :tags: orm

    Query.join(), Query.outerjoin(), eagerload(), eagerload_all(), and others no longer allow lists of attributes as arguments (i.e. option([x, y, z]) form).

.. change::
    :tickets:
    :tags: orm

    ScopedSession.mapper is removed and deprecated since 0.5.

.. change::
    :tickets: 2031
    :tags: orm

    Horizontal shard query places 'shard_id' in context.attributes where it's accessible by the "load()" event.

.. change::
    :tickets: 2032
    :tags: orm

    A single contains_eager() call across multiple entities indicates all collections along that path should load, instead of requiring distinct contains_eager() calls for each endpoint (which was never correctly documented).

.. change::
    :tickets:
    :tags: orm

    The "name" field used in orm.aliased() now renders in the resulting SQL statement.

.. change::
    :tickets: 1473
    :tags: orm

    Session weak_instance_dict=False is deprecated.

.. change::
    :tickets: 2038
    :tags: orm

    Fixed bug where "middle" class in a polymorphic hierarchy would have no 'polymorphic_on' column if it didn't also specify a 'polymorphic_identity', leading to strange errors upon refresh, wrong class loaded when querying from that target. Also emits the correct WHERE criterion when using single table inheritance.

.. change::
    :tickets: 1995
    :tags: orm

    Fixed bug where a column with a SQL or server side default that was excluded from a mapping with include_properties or exclude_properties would result in UnmappedColumnError.

.. change::
    :tickets: 2046
    :tags: orm

    An exception is raised in the unusual case that an append or similar event on a collection occurs after the parent object has been dereferenced, which prevents the parent from being marked as "dirty" in the session. Was a warning in 0.6.6.

.. change::
    :tickets: 1069
    :tags: orm

    Query.distinct() now accepts column expressions as *args, interpreted by the PostgreSQL dialectas DISTINCT ON (<expr>).

.. change::
    :tickets: 2049
    :tags: orm

    Additional tuning to "many-to-one" relationship loads during a flush(). A change in version 0.6.6 ([ticket:2002]) required that more "unnecessary" m2o loads during a flush could occur. Extra loading modes have been added so that the SQL emitted in this specific use case is trimmed back, while still retrieving the information the flush needs in order to not miss anything.

.. change::
    :tickets:
    :tags: orm

    The value of "passive" as passed to attributes.get_history() should be one of the constants defined in the attributes package. Sending True or False is deprecated.

.. change::
    :tickets: 2030
    :tags: orm

    Added a `name` argument to `Query.subquery()`, to allow a fixed name to be assigned to the alias object.

.. change::
    :tickets: 2019
    :tags: orm

    A warning is emitted when a joined-table inheriting mapper has no primary keys on the locally mapped table (but has pks on the superclass table).

.. change::
    :tickets: 2046
    :tags: orm

    A warning is emitted in the unusual case that an append or similar event on a collection occurs after the parent object has been dereferenced, which prevents the parent from being marked as "dirty" in the session. This will be an exception in 0.7.

.. change::
    :tickets: 2050
    :tags: declarative

    Added an explicit check for the case that the name 'metadata' is used for a column attribute on a declarative class.

.. change::
    :tickets: 1844
    :tags: sql

    Added `over()` function and method to FunctionElement classes, produces the _Over() construct which in turn generates "window functions", i.e. "<window function> OVER (PARTITION BY <partition by>, ORDER BY <order by>)".

.. change::
    :tickets: 805
    :tags: sql

    LIMIT/OFFSET clauses now use bind parameters.

.. change::
    :tickets: 723
    :tags: sql

    Added NULLS FIRST and NULLS LAST support. It's implemented as an extension to the asc() and desc() operators, called nullsfirst() and nullslast().

.. change::
    :tickets:

    The Index() construct can be created inline with a Table definition, using strings as column names, as an alternative to the creation of the index outside of the Table.

.. change::
    :tickets: 2001
    :tags: sql

    Execution_options() on Connection accepts "isolation_level" argument, sets transaction isolation level for that connection only until returned to the connection pool, for those backends which support it (SQLite, PostgreSQL).

.. change::
    :tickets: 2020
    :tags: sql

    Result-row processors are applied to pre-executed SQL defaults, as well as cursor.lastrowid when determining the contents of result.inserted_primary_key.

.. change::
    :tickets:

    Bind parameters present in the "columns clause" of a select are now auto-labeled like other "anonymous" clauses, which among other things allows their "type" to be meaningful when the row is fetched, as in result row processors.

.. change::
    :tickets: 2015
    :tags: sql

    Non-DBAPI errors which occur in the scope of an execute() call are now wrapped in sqlalchemy.exc.StatementError, and the text of the SQL statement and repr() of params is included. This makes it easier to identify statement executions which fail before the DBAPI becomes involved.

.. change::
    :tickets: 2048
    :tags: sql

    The concept of associating a ".bind" directly with a ClauseElement has been explicitly moved to Executable, i.e. the mixin that describes ClauseElements which represent engine-executable constructs. This change is an improvement to internal organization and is unlikely to affect any real-world usage.

.. change::
    :tickets: 2028
    :tags: sql

    Column.copy(), as used in table.tometadata(), copies the 'doc' attribute.

.. change::
    :tickets: 2023
    :tags: sql

    Added some defs to the resultproxy.c extension so that the extension compiles and runs on Python 2.4.

.. change::
    :tickets: 2042
    :tags: sql

    The compiler extension now supports overriding the default compilation of expression._BindParamClause, including that the auto-generated binds within the VALUES/SET clause of an insert()/update() statement will also use the new compilation rules.

.. change::
    :tickets: 1921
    :tags: sql

    SQLite dialect now uses `NullPool` for file-based databases.

.. change::
    :tickets: 2036
    :tags: sql

    The path given as the location of a sqlite database is now normalized via os.path.abspath(), so that directory changes within the process don't affect the ultimate location of a relative file path.

.. change::
    :tickets: 1083
    :tags: postgresql

    When explicit sequence execution derives the name of the auto-generated sequence of a SERIAL column, which currently only occurs if implicit_returning=False, now accommodates if the table + column name is greater than 63 characters using the same logic PostgreSQL uses.

.. change::
    :tickets: 2044
    :tags: postgresql

    Added an additional libpq message to the list of "disconnect" exceptions, "could not receive data from server".

.. change::
    :tickets: 1885
    :tags: firebird

    Some adjustments so that Interbase is supported as well. FB/Interbase version idents are parsed into a structure such as (8, 1, 1, 'interbase') or (2, 1, 588, 'firebird') so they can be distinguished.

.. change::
    :tickets: 1991
    :tags: mysql

    New DBAPI support for pymysql, a pure Python port of MySQL-python. 
    
.. change::
    :tickets: 2047
    :tags: mysql

    Oursql dialect accepts the same "ssl" arguments in create_engine() as that of MySQLdb.