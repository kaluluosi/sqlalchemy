=============
1.1 更新日志
=============

.. changelog_imports::

    .. include:: changelog_10.rst
        :start-line: 5


    .. include:: changelog_09.rst
        :start-line: 5


    .. include:: changelog_08.rst
        :start-line: 5


    .. include:: changelog_07.rst
        :start-line: 5


.. changelog::
    :version: 1.1.18
    :released: March 6, 2018

    .. change::
        :tags: bug, mysql
        :tickets: 4205
        :versions: 1.2.5

        MySQL方言现在明确使用“SELECT @@version”向服务器查询服务器
        版本，以确保我们获得正确的版本信息。 代理服务器像MaxScale会
        干扰传递给DBAPI的connection.server_version值，因此这不再是可靠的。

    .. change::
        :tags: bug, postgresql, py3k
        :tickets: 4208
        :versions: 1.2.5

        修复了在PostgreSQL COLLATE / ARRAY调整中首次引入的错误，
        Python 3.7正则表达式的新行为导致修复失败。

.. changelog::
    :version: 1.1.17
    :released: February 22, 2018

    .. change::
        :tags: bug, ext
        :tickets: 4185

        修复1.2.3和1.1.16中的regression，涉及到关联代理对象，修订
        对于“拥有类”的计算方法，当代理对象没有直接关联到映射
        类时，默认选择当前类。例如一个mixin。

.. changelog::
    :version: 1.1.16
    :released: February 16, 2018

    .. change::
        :tags: bug, postgresql
        :versions: 1.2.3

        将“SSL SYSCALL error：Operation timed out”添加到触发psycopg2
        驱动程序“disconnect”场景的消息列表中。从Andre Cruz的pull请求。

    .. change::
        :tags: bug, orm
        :tickets: 4187
        :versions: 1.2.3

        修正了后更新(POST_UPDATE)功能中，当父对象已被删除，但
        从属对象没有删除时，会发送UPDATE语句的问题。此问题已经存在
        很长时间，然而，由于1.2现在断言POST_UPDATE的匹配行数，
        因此这引发了错误。

    .. change::
        :tags: bug, mysql
        :tickets: 4136
        :versions: 1.2.0b4

        修复MySQL“concat”和“match”操作符无法将kwargs传播到左侧
        和右侧表达式的bug，从而导致编译器选项(如“literal_binds”)
        失败的问题。

    .. change::
        :tags: bug, sql
        :versions: 1.2.0b4

        在“sqlalchemy。”和“sqlalchemy.sql。”名称空间中作为顶
        级导入添加 :func:。同样，增加了:func: `.nullsfirst`和
        :func: `.nullslast`。 感谢Lele Gaifax的pull request。

    .. change::
        :tags: bug, orm
        :tickets: 4185
        :versions: 1.2.3

        修复了在修复issue:ticket:`4116`时引起的regression，影响
        版本1.2.2和1.1.15，该修复有可能错误地重新计算“拥有类”
        的 :class:`.AssociationProxy` 作为`nonetype`类，这在一些
        表述式声明混合/继承情况下会出现，或者如果association proxy
        是从未映射的类访问的。新的查找算法替换了“找出所有者”的逻辑，
        它搜索分配给类或子类的完整映射器层次结构，以确定正确(我们
        希望能够)的匹配；如果未找到匹配项，则不会分配所有者。如果
        代理针对未映射的实例使用，则现在会引发异常。

    .. change::
        :tags: bug, sql
        :tickets: 4162
        :versions: 1.2.1

        修复了使用“multi-values”格式的:meth:`_expression.Insert.values`
        时，如果以而不是字符串作为键来使用 :class:`_schema.Column`
        对象，则会失败。感谢Aubrey Stark-Toller的pull请求。

    .. change::
        :tags: bug, postgresql
        :versions: 1.2.3

        将“TRUNCATE”添加到在PostgreSQL方言中接受作为“autocommit”的
        触发关键字的关键字列表。感谢Jacob Hayes的pull请求。

    .. change::
        :tags: bug, pool
        :tickets: 4184
        :versions: 1.2.3

        修复了一种相当严重的连接池错误，其中，当用户定义的决断错误
        (例-1.2中发布的“pre_ping”特性)导致刷新后重新获取连接时，如果
        连接由weakref清除(例如-正面的对象被垃圾收集)，则将无法正确重
        置。 弱引用仍然将引用之前无效的DBAPI连接，这将导致logs中的堆
        栈跟踪以及将连接检入池中而未被重置，这可能会导致锁定问题。

    .. change::
        :tags: bug, orm
        :tickets: 4151
        :versions: 1.2.1

        修复了一个问题，即在带嵌套或子事务的回滚期间，一个在主键经过
        变异的对象在被正确从会话中删除的情况下是错误的，这会导致后续
        在使用会话时出现问题。

.. changelog::
    :version: 1.1.15
    :released: November 3, 2017

    .. change::
        :tags: bug, sqlite
        :tickets: 4099
        :versions: 1.2.0b3

        修复了SQLite CHECK约束反射失败的错误，如果引用的表在远程模式下，
        例如在SQLite中，由ATTACH引用的远程数据库。

    .. change::
        :tags: bug, mysql
        :tickets: 4097
        :versions: 1.2.0b3

        警告 - 修复了在1.2.3和1.1.16中引发回归问题的原因：计算“关联代理”
        的“拥有类”引发错误，因为在某些声明混合/继承情况下，以及如果
        关联代理是从未映射的类访问的，在某些情况下会向映射器属性之外的地
        方引用。此修复现在不再选择NoneType类作为“拥有者”，而是通过深入搜索
        分配给类或子类的完整映射器层次结构来确定正确的匹配方案；如果未找到
        匹配项，则不会分配所有者。如果代理针对未映射的实例使用，则现在会
        引发异常。

    .. change::
        :tags: bug, mssql
        :tickets: 4095
        :versions: 1.2.0b3

        为SQL Server的PyODBC方言添加了一整个“连接关闭”异常代码的范围，
        包括'08S01'、'01002'、'08003'、'08007'、'08S02'、'08001'、'HYT00'、
        'HY010'。以前，只支持'08S01'。

    .. change::
        :tags: bug, sql
        :tickets: 4126
        :versions: 1.2.0

        修复了:class: `.ColumnDefault` __repr__ 当参数是元组的时候会失败的bug。

    .. change::
        :tags: bug, orm, declarative
        :tickets: 4124
        :versions: 1.2.0

        修复了一个从 :class:`.AbstractConcreteBase` 渐进式的混合/继承层次结构中，
        描述符在别处被映射到，对最近的特定记录进行刷新操作将导致错误的问题。此问题
        最好的解决方案是在对应的映射器中包含“concrete=True”，这样就可以防止此场
        景导致问题，但是这里的检查还应该防止该场景导致问题。

    .. change:: 4006
        :tags: bug, postgresql
        :tickets: 4006
        :versions: 1.2.0b3

        随着COLLATE一起使用时修复了 :class:`_types.ARRAY` 类中的错误，在某些封装
        布中可能无法正确生成CREATE TABLE。

    .. change::
        :tags: bug, orm, ext
        :tickets: 4116
        :versions: 1.2.0

        修复了将关联代理用于:class:`.AliasedClass`，如果首先使用
        :class:`.AliasedClass` 作为父对象，它将错误地链接到自己，导致在后续
        用法中发生错误。

    .. change::
        :tags: bug, mysql
        :tickets: 4120
        :versions: 1.2.0

        现在使用 @transaction_isolation 而不是@tx_isolation 变量，避免出
        现警告。

    .. change::
        :tags: bug, postgresql
        :tickets: 4107
        :versions: 1.2.0b3

        修复 :obj:`_functions.array_agg` 函数中的错误，其中传递已经是 :class:`_types.ARRAY` 的
        参数会导致ValueError，因为该函数尝试嵌套数组。

    .. change::
        :tags: bug, orm
        :tickets: 4078
        :versions: 1.2.0b3

        修复了ORM关系的冲突同步不对冲突的兄弟类的情况进行警告问题
        (即，两个关系都将写入同一列)，其中在一个继承层次结构中的兄弟关系不会
        导致冲突。

    .. change::
        :tags: bug, postgresql
        :tickets: 4074
        :versions: 1.2.0b3

        修复了PostgreSQL :meth:`.postgresql.dml.Insert.on_conflict_do_update` 中的错误，
        它将阻止使用该插入语句作为CTE，例如通过 :meth:`_expression.Insert.cte` 在另一个语
        句中。

    .. change::
        :tags: bug, orm
        :tickets: 4103
        :versions: 1.2.0b3

        修复了相对于单表继承实体使用的相关选择在外部查询中不正确呈现的问题，例如
        在处理单个继承鉴别器标准的情况下，将标准重新应用到外部查询。

    .. change::
        :tags: bug, mysql
        :tickets: 4096
        :versions: 1.2.0b3

        修复了由于语法更改，在MariaDB 10.2系列中CURRENT_TIMESTAMP可能不会正确反映的
        问题，现在该函数表示为“current_timestamp()”。 

    .. change::
        :tags: bug, mysql
        :tickets: 4098
        :versions: 1.2.0b3

        现在，当出现类似“PostgreSQL 10beta1”的版本字符串时，支持解析PostgreSQL版本字
        符串。虽然PostgreSQL现在提供更好的方法来获取此信息，但我们至少在1.1.x中仍保持
        正则表达式，以最大程度地降低与旧版或替代PostgreSQL数据库的兼容性风险。

.. changelog::
    :version: 1.1.14
    :released: September 5, 2017

    .. change::
        :tags: bug, orm
        :tickets: 4069
        :versions: 1.2.0b3

        修复了由于检查目标对象未及时获取到目标对象的原因，级联例如
        “delete-orphan”（但还有其他的）将无法找到链接到其本身是
        在继承关系中的现存于子类的关系的对象，从而导致操作不执行。

    .. change::
        :tags: bug, orm
        :tickets: 4048
        :versions: 1.2.0b3

        修复了当一个与子查询载入到一个基类的关系，然后子查询通过基类关系进行连接
        预加载时，如果该查询还包括针对子类的条件，那么加载“次级负载”更深层次的操作将
        工作过度，这会导致额外的工作。该子查询从为作为第一级的继承子类而开始并且在
        查询中包含对基类关系的一个关系。以前的修复不能为从一级进一步加载的任何子查询提供
        处理，因此现在它已进一步概括。

    .. change:: 4033
        :tags: bug, orm
        :tickets: 4033
        :versions: 1.2.0b2

        修复了从1.1.11中添加的检查引起的错误，当多个非实体列添加到包含具有subqueryload经
        关系的实体的查询中时，该检查可能会导致错误，因为其假设将在父亲上获取所有列，
        而这包括虚拟外键。 

    .. change:: 4031
        :tags: bug, orm
        :versions: 1.2.0b2
        :tickets: 4031

        修复了在 :func:`.synonym` used when it was not against a :class:`.MapperProperty`, such as an
        association proxy. Previously, a recursion overflow would occur trying to locate non-existent attributes.
        
.. changelog::
    :version: 1.1.12
    :released: July 24, 2017

    .. change:: cache_order_sequence
        :tags: feature, oracle, postgresql
        :versions: 1.2.0b1

        向 :class:`.Sequence` 添加了新的关键字 :paramref:`.Sequence.cache` 和
        :paramref:`.Sequence.order`，以允许渲染Oracle和PostgreSQL所理解的
        CACHE参数和ORDER参数。感谢David Moore的Pull request。

    .. change:: 4033
        :tags: bug, orm
        :tickets: 4033
        :versions: 1.2.0b2

        从1.1.11释放的regression错误修复问题，当从连接的继承子类开始并然后从基类关系载
        入到一个关系而查询也包括与子类的条件，则“子查询”包含正确的FROM子句，而该策略
        可能针对基类中的关系与subqueryload一起使用。

    .. change:: 4030
        :tags: bug, orm
        :versions: 1.2.0b2
        :tickets: 4030

        在 :class:`.WeakInstanceDict` 所有方法中，如果在“key in dict”检查之后，对该键进行
        索引访问，则添加了 `KeyError`检查。 加上它可以防止瓶颈的垃圾回收在加载负载
        并在代码假定其存在后从dict中删除键时将其从dict中删除。

.. changelog::
    :version: 1.1.11
    :released: Monday, June 19, 2017

    .. change:: 4012
        :tags: bug, sql
        :tickets: 4012
        :versions: 1.2.0b1

        修复 :class:`.WithinGroup` 结构迭代期间将会出现的 AttributeError。

    .. change:: 4011
        :tags: bug, orm
        :tickets: 4011
        :versions: 1.2.0b1

        修复了在保持正确的“子查询”时，“子查询”包含正确的FROM FROM的问题。对于从
        连接的子类开始，然后从基类关系快速加载到一个关系的情况，这在orm地图中经常出现，
        如果查询也包括基类中的条件以及与子类的关系，则会出现此问题。之前的修复未适
        当地纳入更进一步的加载。 

    .. change:: 4005
        :tags: bug, postgresql
        :tickets: 4005
        :versions: 1.2.0b1

        添加MySQL 8.0保留字，以正确引用MySQL方言。感谢Hanno Schlichting的pull请求。

    .. change:: 4006
        :tags: bug, postgresql
        :tickets: 4006
        :versions: 1.2.0b1

        修复了使用 :class:`_types.ARRAY` 的字符串类型时可能无法正确呈现CREATE TABLE的错误，
        该字符串类型包含了一个COLLATE，曾经参考的2461912824412118269。

    .. change:: 4007
        :tags: bug, mysql
        :tickets: 4007
        :versions: 1.2.0b1

        MySQL 5.7现在限制了“SHOW VARIABLES”命令的权限，MySQL方言现在将处理SHOW返回的
        没有行的情况，特别是SQL_MODE的初始获取，将发出警告，指出用户权限应修改为使该行存在。

    .. change:: 3994
        :tags: bug, mssql
        :tickets: 3994
        :versions: 1.2.0b1

        向SQL Server dialect添加占位符 type :class:`_mssql.XML`，以便可以将反映包含该类型的表
        重新呈现为CREATE TABLE。该类型没有特殊的往返行为，也不支持其他限定参数。

    .. change:: 3997
        :tags: bug, oracle
        :tickets: 3997
        :versions: 1.2.0b1

        如果在使用cx_Oracle DBAPI版本6.0b1或更高版本的情况下，将完全删除了cx_Oracle的两个
        阶段事务的支持。但是，在任何情况下，两阶段特性从未在cx_Oracle 5.x 中可用，cx_Oracle
        6.x已经除去了该特征所依赖的连接级“twophase”标志。

.. changelog::
    :version: 1.1.9
    :released: April 4, 2017

    .. change:: 3956
        :tags: bug, ext
        :tickets: 3956

        修复1.1.8 中发布的bug，由于 :ticket:`3950`，导致新型/模式 :class:`.TypeDecorator`中的列类型信息被丢失

    .. change:: 3952
        :tags: bug, sql
        :versions: 1.2.0b1
        :tickets: 3952

        由于在右边表达式上应用:class:`.Variant`的基础类型的“右边”规则，引起的问题，
        将 :class:`.Variant` 和metadata:, class:`.SchemaType` 相互兼容。也就是说，
        可以在 :class:`.Enum` 类型中创建与变种相关的约束并传播到dialect mapping。

    .. change:: 3955
        :tags: bug, sql, postgresql
        :versions: 1.2.0b1
        :tickets: 3955

        改变了 :class:`_engine.ResultProxy` 的机制，不会在 :class:`_engine.Connection` 完成处理对象之前自动关闭。
        在包含RETURNING的INSERT/UPDATE/DELETE上发生的像PostgreSQL ON CONFLICT返回零行的情况中，自动关闭
        前的autocommit会失败，因为之前不存在这种使用情况。 

一些涉及热心联接路径构建的机制已经进行了优化。最坏的负载情况下，在构建查询并招募了一个聚合加载器的跨端-端查询+单个行抓取测试中，调用次数已经比1.1.5减少了约60%，比0.8.6少了42%。

添加了IMPORT FOREIGN SCHEMA和REFRESH MATERIALIZED VIEW PostgreSQL语句的正则表达式，以便在没有显式事务的情况下，通过连接或引擎调用时自动提交。感谢Frazer McLean和Paweł Stiasny的Pull requests。

修复了"eager_defaults"功能中的重大低效性问题。之前出现了这样一个不必要的SELECT，对应于ORM已经显式插入了NULL的列值，与对象上未设置但没有任何服务器默认值的属性相对应，以及更新时过期的属性，在没有设置服务器的onupdate的情况下。由于这些列不是RETURNING中eager_defaults尝试使用的一部分，因此也不应该进行后SELECT。

修复了两个与映射器eager_defaults标志以及单表继承相关的明显错误：一个是，在获取eager_defaults时，eager_defaults逻辑会错误地尝试访问映射器的"exclude_properties"列表(由具有单表继承的Declarative使用)中的列；另一个是，在获取默认值时，完整行数据加载操作没有使用正确的继承映射器。

修复了"DDLEvents.column_reflect"事件的bug，行为是允许非文本表达式作为新列的默认值传递，例如一个FetchedValue对象，以表明一个通用的触发默认值或一个_expression.text()构造函数。在相关文件中也对这方面的文档进行了澄清。

修复了新ext.indexable扩展中的bug，其中自身引用的属性设置将失败。

修复了PostgreSQL.ExcludeConstraint的一个错误，其中"whereclause"和"using"参数在操作Table.tometadata()时将不会被复制。

已增加版本检查，用于get_isolation_level功能，在第一次连接时调用，因此它会跳过SQL Server 2000，因为SQL Server在2005年之前不提供必要的系统视图。

已添加baked.Result.scalar和baked.Result.count到"烘焙"查询系统。.. _changelog-1.1.0b3:

Changelog for 1.1.0b3
======================

.. change::
    :tags: bug, orm
    :tickets: 3749

    Removed a warning that dates back to 0.4 which emits when a same-named
    relationship is placed on two mappers that inherits via joined or
    single table inheritance. The warning does not apply to the current
    unit of work implementation.

    .. seealso::
    
        :ref:`change_3749`

.. change::
    :tags: bug, sql
    :tickets: 3745

    Fixed bug in new CTE feature for update/insert/delete stated as a CTE
    inside of an enclosing statement (typically SELECT) whereby oninsert
    and onupdate values weren't called upon for the embedded statement.

.. change::
    :tags: bug, sql
    :tickets: 3744

        Fixed a bug in the new CTE feature for update/insert/delete, whereby
        an anonymous (e.g. no name passed) CTE construct around the statement
        would fail.

.. change::
    :tags: bug, ext

    The sqlalchemy.ext.indexable will still intercept IndexError as well as
    KeyError, and raises an AttributeError.

.. change::
    :tags: feature, ext

    Added a "default" parameter to sqlalchemy.ext.indexable extension.

.. _changelog-1.1.0b2:

Changelog for 1.1.0b2
======================

.. change::
    :tags: bug, ext, postgresql
    :tickets: 3732

    Made a slight behavioral change in the sqlalchemy.ext.compiler extension,
    whereby the existing compilation schemes for an established construct
    would be removed if that construct itself didn't already have its own
    dedicated __visit_name__. This was a rare occurrence in 1.0, however in
    1.1 ARRAY subclasses ARRAY and has this behavior. As a result, setting
    up a compilation handler for another dialect such as SQLite would render
    the main ARRAY object no longer compilable.

.. change::
    :tags: bug, sql
    :tickets: 3730

    The processing performed by the Boolean datatype for backends that only
    feature integer types have been made consistent between the pure Python
    and C-extension versions in that the C-extension version will accept any
    integer value from the database as a boolean and non-boolean integer
    values being sent to the database are coerced to exactly zero or one,
    instead of being passed as the original integer value.

.. change::
    :tags: bug, sql
    :tickets: 3725

    Rolled back the validation rules a bit in Enum to allow unknown string
    values to pass through, unless the flag "validate_string=True" is passed to
    the Enum. Any other object is still rejected. While the immediate use is to
    allow comparisons to enums with LIKE, the fact that this use exists indicates
    there may be more unknown-string-comparison use cases than we expected, which
    hints that perhaps there are some unknown string-INSERT cases, too.

.. change::
    :tags: bug, mysql
    :tickets: 3726

    Dialed back the "order the primary key columns per auto-increment" described
    in change_mysql_3216 a bit, so that if the PrimaryKeyConstraint is
    explicitly defined, the order of columns is maintained exactly, allowing
    control of this behaviour when necessary.

.. _changelog-1.1.0b1:

Changelog for 1.1.0b1
======================

.. change::
    :tags: feature, sql
    :tickets: 3718

    Added TABLESAMPLE support via the new tablesample method and standalone
    function. Pull request courtesy Ilja Everilä.

    .. seealso::

        :ref:`change_3718`

.. change::
    :tags: feature, orm

    A new ORM extension indexable_toplevel added, which allows construction
    of Python attributes which refer to specific elements of "indexed"
    structures such as arrays and JSON fields.

    .. seealso::

        :ref:`feature_indexable`

.. change::
    :tags: bug, sql
    :tickets: 3724

    FromClause.count is deprecated since it makes use of an arbitrary column in
    the table and is not reliable; for Core use, func.count() should be
    preferred.

.. change::
    :tags: feature, postgresql
    :tickets: 3529

    Added support for PostgreSQL's INSERT..ON CONFLICT using a new
    PostgreSQL-specific Insert object.

    .. seealso::

        :ref:`change_3529`

.. change::
    :tags: feature, postgresql

    The DDL for DROP INDEX will emit "CONCURRENTLY" if the postgresql_concurrently
    flag is set upon the Index, and if the database in use is detected
    as PostgreSQL version 9.2 or greater. For CREATE INDEX, database version
    detection is also added which will omit the clause if PG version is less
    than 8.2. Pull request courtesy Iuri de Silvio.

.. change::
    :tags: bug, orm
    :tickets: 3708

    Fixed an issue where a many-to-one change of an object from one parent
    to another could work inconsistently when combined with an unflushed
    modification of the foreign key attribute. The attribute move now
    considers the database-committed value of the foreign key in order to
    locate the "previous" parent of the object being moved. This allows
    events to fire off correctly including backref events. Previously,
    these events would not always fire. Applications which may have relied
    on the previously broken behaviour may be affected.

    .. seealso::

        :ref:`change_3708`

.. change::
    :tags: feature, sql

    Added support for ranges in window functions, using the over.range_ and
    over.rows parameters.

.. change::
    :tags: feature, orm

    Added new flag Query.order_by in order to cancel all order by's. The
    Query.group_by method now resets the group by collection if an argument
    of None is passed, in the same way that Query.order_by has worked for a
    long time. Pull request courtesy Iuri Diniz.

.. change::
    :tags: feature, sql
    :tickets: 3763

    Fixed an error where Index would fail to extract columns from compound SQL
    expressions if those SQL expressions were wrapped inside of an ORM-style
    __clause_element__() construct.

.. change::
    :tags: feature, orm

    Constructing a declarative base class that inherits from another class
    will also inherit its docstring. This means
    ext.declarative.as_declarative acts more like a normal class decorator.

.. change::
    :tags: bug, mysql
    :tickets: 3766

    Fixed bug where the "literal_binds" flag would not be propagated to a
    CAST expression under MySQL.

.. change::
    :tags: bug, sql, postgresql, mysql
    :tickets: 3765

    Fixed regression of JSON datatypes, where the "literal processor" for a
    JSON index value would not be invoked. The String and Integer datatypes are
    now invoked from within the JSONIndexType and JSONPathType. This is applied
    to the generic, PostgreSQL, and MySQL JSON types and also has a dependency
    on 3766.

.. change::
    :tags: bug, orm
    :tickets: 3776

    Fixed bug where although the use of more than one validator at a time is
    supported, there's no mechanism to chain these together, as the order of
    the validators at the level of function decorator can't be made
    deterministic.

    .. seealso::

        :ref:`change_3776`

.. _changelog-1.1.0b4:

Changelog for 1.1.0b4
======================

.. change::
    :tags: breakpoint

    Renamed Release Notes.rst to CHANGES.rst, guaranteed to include
    :ref:`changelog`.

.. change::
    :tags: bug, sql
    :tickets: 3807

    Fixed issue where in certain situations where a SELECT statement contains
    a UNION the output of column_property() would use only the first SELECT.
    Pull request courtesy Jeff Widman.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, sql
    :tickets: 3823

    Added support to work with the special-case case-sensitive table and column
    identifiers in Oracle dialect. The new built-in class CaseSensitiveOracleIdentifierPreparer
    will now be called upon when the parameter case_sensitive=True, which
    which then escapes those characters which would normally not be case sensitive.
    SQL fragments are not supported, and it is the variety of quoting used
    by Oracle itself, using the "double quotes" quote type.

.. change::
    :tags: bug, sql
    :tickets: 3827

    Fixed issue where an INSERT from a SELECT could generate a column-ordinal based
    INSERT rather than a table-ordinal based INSERT when len(select_columns) != len(target_columns).

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: bug, postgresql
    :tickets: 3828

    Changed DDLCompiler.get_column_type to not change prefixes like "CHARACTER VARYING" on
    JSON-like types. Fixes a regression of sorts on an existing column containing
    ARRAY columns.

.. change::
    :tags: bug, mysql
    :tickets: 3648

    Made the dialect more forgiving when attempting to reconstruct types from
    string datatypes where additional parameters are present. For example,
    MySQL's string-based "VARCHAR(15) DEFAULT 'foo'" is now accepted in situations
    when the existing MySQL dialect would raise an assertion.

.. change::
    :tags: feature, sql
    :tickets: 3769

    Added a new sql.expression.raiseerr() construct, a new Core SQL expression that
    can emit a SQL statement that indicates an error in the SQL.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: bug, postgresql
    :tickets: 3832

    Fixed bug whereby if a SQL SELECT contained a given column only because of an OUTER
    JOIN and the given column was an aggregate function, and the column in question
    was the NULL result of that function, Postgresql would emit a more confusing
    result for it compared to other database backends.

.. change::
    :tags: feature, core
    :tickets: 3830, 3687

    - Added type_ argument to Column.copy()
    - Added type_ argument to Table.c.copy()
    - Added type_ and existing_nullable arguments to Column.copy()

.. change::
    :tags: feature, core
    :tickets: 3850

    Added DBAPI-level support for a "multirow" extension to INSERT statements,
    via the 'executemany_values_page_size' option passed to create_engine().

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, sql
    :tickets: 3802

    Added support for "Nulls First" and "Nulls Last" on most ORDER BY clauses
    through the new method ``.nulls_first()/.nulls_last()``.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, postgresql
    :tickets: 3865

    Added support for PostgreSQL `GROUPING SETS` syntax.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: bug, orm
    :tickets: 3871

    Fixed an issue whereby an "unhashable type object" error could be raised in
    the ORM session when an object was detached that contained a relationship()
    with an "order_by" specified referencing a Python descriptor not mounted by the ORM.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, orm
    :tickets: 3886

    The methods Session.bulk_insert_mappings(), Session.bulk_update_mappings(),
    and Session.bulk_save_objects() now use an INSERT..ON DUPLICATE KEY UPDATE
    style statement to emit UPDATEs instead of UPDATE/DELETE by default; the
    parameter `preserve_duplicates` is added to selectively restore the
    original behavior.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, core
    :tickets: 3907

    Added the new type Interval()/Interval.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, event
    :tickets: 3912

    Allow an application to postprocess information returned by ResultMetaData._get_colparams() for
    its own usage. To that end an event "after_get_colparams" is added which is called when
    _get_colparams() returns its values.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, core
    :tickets: 3887

    A column that is passed to a primaryjoin or foreign_keys parameter in a relationship()
    configuration will now have a default name generated based on its local name if it
    does not already have a name.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, mysql
    :tickets: 3931

    The MySQL dialect now matches names based on lowercase comparison for
    tables and columns

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, sql
    :tickets: 3947

    Simplified the basic methods of the MutableDict and MutableSet
    constructs within sqlalchemy.ext.mutable, as well as their
    subclassing method utilizations.


    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, sql
    :tickets: 3951

    Added basic support for mapping of composite types; these types
    are returned in postgresql as well as db2 using a literal string-based
    format, and will be stored and processed just like other string-based
    types until more advanced parsing is added.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`


.. _changelog-1.1.0b5:

Changelog for 1.1.0b5
========================

.. change::
    :tags: feature, oracle
    :tickets: 3469

    Added date returning support for the out-parameter form of functions in Oracle. A new type
    oracle.OUT_RETURNING was added, the callable _oracle_wrapped_cursor_execute() was changed to
    handle this type and the test suite was updated.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, sql
    :tickets: 3960

    Added new functions: within_group(), grouping(), and group_id();

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: bug, sql
    :tickets: 3948

    Made the pickle tool "pickle-safe" on C extension types.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, sql
    :tickets: 3465

    Added support for INTERVAL type on SQLite;

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, core
    :tickets: 3968

    Added support for expiring and refreshing per-Bucket sessions within the
    Connection pooling regime, when poolclass is set explicitly.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, core
    :tickets: 3961

    Added support for changing the "transaction isolation level" on a per-connection
    basis. This is currently only tested for a few dialects, including MySQL dialects
    as well as PostgreSQL.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: bug, mysql
    :tickets: 3723

    The dialect emits USE statement before SET SESSION to fix issues with some MySQL connectors
    when used in a read replication scenario.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, postgresql
    :tickets: 3977

    Another adjustment to the previous change related to column types and "domain-based"
    constraints added to tables where the domain refers to a non-built-in type, added
    a more sophisticated parsing process on constraints that refers to these types.
    This includes the added ability to parse incoming type declarations as well.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: feature, mysql
    :tickets: 3979

    Added the new dialect for google.cloud.sql, matching to
    SQLAlchemy's mysql dialect.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`

.. change::
    :tags: bug, sql
    :tickets: 3834

    Fixed bug when using self-joins with the sql literal construct.

.. change::
    :tags: bug, sql
    :tickets: 3981

    Fixed a regression when subquery() with a SELECT subquery that contains
    labels that use substring.

.. change::
    :tags: feature, mssql
    :tickets: 3982

    Allow EXECUTE AS to be added to the "SET" portion of T-SQL statements.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`
        
.. change::
    :tags: feature, sql
    :tickets: 3950

    Added new feature to enable the eager loading of the "base table" of an
    inherited table via .with_polymorphic() when the entities in the sub-table
    contain no additional information.

    .. seealso::

        :ref:`changelog_1_1_whatsnew`.. _change_3582:

.. 改进::
    :tags: enhancement, schema

    现在:class:`_schema.Column`对象默认使用的生成函数将经过``"update_wrapper"`` 或等效函数运行，
    所以introspection工具可以保留包装函数的名称和docstring。pull request由hsum提供。

.. 改进::
    :tags: change, sql, mysql
    :tickets: 3216

    更改了:class:`_schema.Column`对象认为其自己是“自动增量”列的系统，以便“autoincrement”不再隐含启用具有复合主键的:class:`_schema.Table`的“autoincrement”。
    为了适应以下情况：能够同时为复合PK成员列启用autoincrement，并保持SQLAlchemy长期以来启用单个整数主键隐式autoincrement的行为，已添加了:paramref:`_schema.Column.autoincrement`参数中的第三个状态“auto” ，现在是默认值。

    .. 参见::

        :ref:`change_3216`

        :ref:`change_mysql_3216`

.. 改进::
    :tags: change, mysql
    :tickets: 3216

    现在，使用InnoDB并且有一个自增的复合主键的表在生成CREATE TABLE DDL时不会再生成额外的“KEY”指令；
    为了克服InnoDB在此处的限制，现在将使用AUTO_INCREMENT的列放在列列表中的第一位来生成主键约束。

    .. 参见::

        :ref:`change_mysql_3216`

        :ref:`change_3216`

.. 改进::
    :tags: change, sqlite

    现在支持SQLite方言中:meth:`_reflection.Inspector.get_schema_names` 方法的使用; 由于Brian Van Klaveren提供的拉取请求，还修复了创建具有模式的索引以及反射具有模式绑定表的外键约束的支持。

    .. 参见::

        :ref:`change_sqlite_schemas`

.. 改进::
    :tags: change, mssql
    :tickets: 3434

    引入了``legacy_schema_aliasing``标志，以允许MSSQL方言禁用试图为模式限定表创建别名的尝试的版本1.0.5的一部分，现在默认值为False;现在除非显式打开，否则已禁用旧行为。

    .. 参见::

        :ref:`change_3434`

.. 改进::
    :tags: bug, orm
    :tickets: 3250

    添加了一个新的类型级别修饰符:meth:`.TypeEngine.evaluates_none`,
    它指示ORM应该将正面的None集保持为值NULL，而不是从INSERT语句中省略列。
    此功能既用于:ticket:`3514`的实现，也作为任何类型可用的独立功能。

    .. 参见::

        :ref:`change_3250`

.. 改进::
    :tags: bug, postgresql
    :tickets: 2729

    现在使用:class:`_postgresql.ARRAY`对象来引用:class:`_types.Enum`或:class:`_postgresql.ENUM`子类型时，
    将在使用类型时通过“CREATE TYPE”和“DROP TYPE”DDL进行处理。

    .. 参见::

        :ref:`change_2729`

.. 改进::
    :tags: bug, sql
    :tickets: 3531

    现在，:func:`.type_coerce`构造是一个完全成熟的Core表达式元素，
    它在编译时进行迟评估。以前，该函数只是通过返回基于列定向表达式的:class:`.Label`或给定的:class:`.BindParameter`对象的副本来处理不同的表达式输入的转换函数，
    这特别是在ORM级别的表达式转换将列转换为绑定参数（例如用于延迟加载）时，防止了操作被逻辑地维护。

    .. 参见::

        :ref:`change_3531`

.. 改进::
    :tags: bug, orm
    :tickets: 3526

    将:meth:`.Session.bulk_save_objects`和相关的批量方法中的“会计”函数的内部调用缩小到没有使用到该功能的程度，例如获取修改后的INSERT或UPDATE语句的默认值检查。

.. 改进::
    :tags: feature, orm
    :tickets: 2677

    :class:`.SessionEvents`现在包括事件，以允许使用:class:`.Session`本身对所有对象生命周期状态转换进行无歧义的跟踪，
    例如待处理的(transient)，瞬态((transient) )，持久性(persistent)，已脱机(detached)。每个事件中对象的状态也已定义。

    .. 参见::

        :ref:`change_2677`

.. 改进::
    :tags: feature, orm
    :tickets: 2677

    添加了一个新的会话生命周期状态：:term:`删除（deleted）`。
    这种新状态表示已从持久状态中删除的对象，并且在提交事务后会转移到脱机状态中。
    这解决了长期存在的问题，即被删除对象存在于持久状态与脱机状态之间的灰色区域。
    :attr:`.InstanceState.persistent`访问器**不再**将删除的对象报告为persistent; 相反，这些对象的:attr:`.InstanceState.deleted` 访问器将为True，直到它们变成detached。

    .. 参见::

        :ref:`change_2677`

.. 改进::
    :tags: change, orm
    :tickets: 2677

    现在:meth:`.Session.weak_identity_map`参数被弃用。
    请参阅:ref:`session_referencing_behavior`中的新方法以通过事件维护强标识映射。
    
    .. 参见::

        :ref:`change_2677`

.. 改进::
    :tags: bug, sql
    :tickets: 2919

    :class:`.TypeDecorator`类型扩展器现在与:class:`.SchemaType`实现一起工作，通常为:class:`.Enum`或:class:`.Boolean`,
    关于确保来自实现类型传播到外部类型的每个表的事件。
    这些事件用于确保正确创建（可能是删除）约束或PostgreSQL类型（例如ENUM）与父表一起。

    .. 参见::

        :ref:`change_2919`

.. 改进::
    :tags: feature, sql
    :tickets: 1370

    添加了一种形式为“<function> WITHIN GROUP (ORDER BY <criteria>)”的“set-aggregate”函数支持，
    使用方法:meth:`.FunctionElement.within_group`。已添加一系列带返回类型的常见集合聚合函数。
    这包括:class:`.percentile_cont`，:class:`.dense_rank`等函数。

    .. 参见::

        :ref:`change_3132`

.. 改进::
    :tags: feature, sql, postgresql
    :tickets: 3132

    现在支持SQL标准函数:class:`_functions.array_agg`，
    它自动返回正确类型的:class:`_postgresql.ARRAY`并支持索引/分片操作，
    以及:func:`_postgresql.array_agg`，它将返回带有额外比较功能的:class:`_postgresql.ARRAY`。
    由于数组目前仅在PostgreSQL上受支持，因此仅在PostgreSQL上有效。
    还添加了新构造:class:`_postgresql.aggregate_order_by`，以支持PG的“ORDER BY”扩展。

    .. 参见::

        :ref:`change_3132`

.. 改进::
    :tags: feature, sql
    :tickets: 3516

    向Core添加了一个新类型:class:`_types.ARRAY`。这是PostgreSQL :class:`_postgresql.ARRAY`类型的基础，现在是Core的一部分，以开始支持各种支持数组的SQL标准特性，
    包括一些函数以及最终对其他具有“数组”概念的数据库（如DB2或Oracle）的本机数组的支持。
    此外，添加了新运算符:func:`_expression.any_`和:func:`_expression.all_`，这些运算符不仅支持PostgreSQL上的阵列结构，还支持可用于MySQL的子查询(但遗憾的是，在PostgreSQL上不起作用）。

    .. 参见::

        :ref:`change_3516`

.. 改进::
    :tags: feature, orm
    :tickets: 3321

    添加了对常见错误情况的新检查，其中将映射类或映射实例传递到将它们解释为SQL绑定参数的上下文中;
    出现这种错误将引发一个新的异常。

    .. 参见::

        :ref:`change_3321`

.. 改进::
    :tags: bug, postgresql
    :tickets: 3499

    对于诸如:class:`_postgresql.ARRAY`，:class:`_postgresql.JSON`和:class:`_postgresql.HSTORE`之类的特殊数据类型，
    现在将“可哈希”的标志设置为False，以允许在包括具有的行中的ORM查询中获取这些类型。

    .. 参见::

        :ref:`change_3499`

        :ref:`change_3499_postgresql`

.. 改进::
    :tags: bug, postgresql
    :tickets: 3487

    现在，当:paramref:`.postgresql.ARRAY.dimensions`参数设置为所需的维数时，
    PostgreSQL :class:`_postgresql.ARRAY`类型支持多维索引访问，
    例如表达式:somecol[5][6]。其中没有需要显式转换或类型转换的需求。

    .. 参见::

        :ref:`change_3503`

.. 改进::
    :tags: bug, postgresql
    :tickets: 3503

    当使用索引访问时，:class:`_postgresql.JSON`和:class:`_postgresql.JSONB` 
    的返回类型已被修复，其工作方式类似于PostgreSQL本身，
    并返回作为:class:`_postgresql.JSON`或:class:`_postgresql.JSONB`类型的表达式。 
    在这之前，访问器将返回:class:`.NullType`，不允许使用后续类似JSON的操作符。

    .. 参见::

        :ref:`change_3503`

.. 改进::
    :tags: bug, postgresql
    :tickets: 3503

    :class:`_postgresql.JSON`，:class:`_postgresql.JSONB`和:class:`_postgresql.HSTORE`数据类型现在允许完全控制从索引文本访问操作的返回类型，
    即``column[someindex].astext``用于JSON类型或``column[someindex]``用于HSTORE类型,
    通过:paramref:`.postgresql.JSON.astext_type`和:paramref:`.postgresql.HSTORE.text_type`参数。

    .. 参见::

        :ref:`change_3503`

.. 改进::
    :tags: bug, postgresql
    :tickets: 3503

    :attr:`.postgresql.JSON.Comparator.astext` 修改器现在不再隐式调用:meth:`_expression.ColumnElement.cast`，
    因为PG的JSON / JSONB类型也允许彼此之间的交叉转换。
    在JSON索引访问上使用:meth:`_expression.ColumnElement.cast`的代码，例如 ``col[someindex].cast(Integer)``,
    现在需要显式调用:attr:`.postgresql.JSON.Comparator.astext`进行更改。

    .. 参见::

        :ref:`change_3503_cast`

.. 改进::
    :tags: bug, orm, postgresql
    :tickets: 3514

    关于``None``值与PostgreSQL的:class:`_postgresql.JSON`类型配合使用的问题，已增加了额外的修复。
    当:paramref:`_types.JSON.none_as_null`标志保留其默认值为``False``时，
    ORM现在将正确地将JSON 'null'字符串插入到列中，无论ORM对象上的值是否设置为值``None``
    还是是否使用值``None``与:meth:`.Session.bulk_insert_mappings`，
    包括在列上设置了默认值或服务器默认值的情况下。

    .. 参见::

        :ref:`change_3514`

        :ref:`change_3250`

.. 改进::
    :tags: feature, postgresql
    :tickets: 3514

    增加了一个新的常数:attr:`.postgresql.JSON.NULL`，表示应该使用JSON空值（NULL）的值，
    而与其他设置无关。

    .. 参见::

        :ref:`change_3514_jsonnull`

.. 改进::
    :tags: bug, sql
    :tickets: 2528

    :func:`_expression.union`构造和相关构造（例如:meth:`_query.Query.union`）的行为现在处理嵌入式SELECT语句需要由于包括LIMIT，OFFSET和/或ORDER BY而被括在括号中。查询在SQLite上不起作用，并像以前一样在该后端上失败，
    但现在应该在所有其他后端上工作。

    .. 参见::

        :ref:`change_2528`

.. 改进::
    :tags: feature, orm
    :tickets: 3512

    添加了新的关系加载策略：:func:`_orm.raiseload`（也可以通过``lazy ='raise'``访问）。
    此策略的行为几乎类似于:func:`_orm.noload`，但不会返回``None``，而是引发一个InvalidRequestError异常。
    由Adrian Moennich提供的拉请求。

    .. 参见::

        :ref:`change_3512`

.. 改进::
    :tags: bug, mssql
    :tickets: 3504

    修复了SQL Server方言在将字符串或其他可变长度列类型反射为无界长度时将令牌``'max'``赋值给字符串的长度属性的问题。
    虽然显式使用“max”令牌受到SQL Server方言的支持，但它不是基本字符串类型的正常约定的一部分，
    而是应使用长度保持为None。现在在反射类型时将长度分配为空，以便类型在其他情况下正常工作。

    .. 参见::

        :ref:`change_3504`