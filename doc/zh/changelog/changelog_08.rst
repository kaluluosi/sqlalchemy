=============
0.8 更新日志
=============

.. changelog_imports::

    .. include:: changelog_07.rst
        :start-line: 5


.. changelog::
    :version: 0.8.7
    :released: 2014年7月22日

    .. change::
        :tags: bug, mssql
        :versions: 1.0.0b1, 0.9.7

        在 "SET IDENTITY_INSERT" 语句中添加语句编码，这些语句在将显式INSERT插入到IDENTITY列中时操作，以支持在不支持Unicode语句的驱动程序上使用非ASCII表标识符，如pyodbc + unix + py2k。

    .. change::
        :tags: bug, mssql
        :versions: 1.0.0b1, 0.9.7
        :tickets: 3091

        在SQL Server的pyodbc方言中，修复了当未明确设置“description_encoding”方言参数时，当结果集包含替代编码中的名称时，cursor.description无法正确解析的实现。此参数不应再使用。

    .. change::
        :tags: bug, sql
        :versions: 1.0.0b1, 0.9.7
        :tickets: 3124

        修复了   :class:`.Enum`  和其他   :class:` .SchemaType`  子类中的一个bug,其中将类型直接关联到   :class:`_schema.MetaData`  会导致在   :class:` _schema.MetaData`  上发出事件(如创建事件)时出错。

    .. change::
        :tags: bug, sql
        :versions: 1.0.0b1, 0.9.7
        :tickets: 3102

        在自定义运算符加  :meth:`.TypeEngine.with_variant`  系统中修复了一个bug，在使用   :class:` .TypeDecorator`  结合 variant 时，当使用比较运算符时将失败，并出现MRO错误。

    .. change::
        :tags: bug, mysql
        :versions: 1.0.0b1, 0.9.7
        :tickets: 3101

        显然，MySQL错误2014“commands out of sync”可能是由ProgrammingError，而不是OperationalError引起的，目前的MySQL-Python版本会引发ProgrammingError，并检查了在所有MySQL错误代码中测试“ is disconnect”的错误，而无论是否在OperationalError和ProgrammingError中都检查了所有MySQL错误代码。

    .. change::
        :tags: bug, mysql
        :versions: 1.0.0b1, 0.9.5
        :tickets: 3085

        修复了一个bug，在index的mysql_length参数上添加了列名时，如果带引号的名称需要具有相同的引号才能识别该问题。修复使引号可选，但也提供了旧行为以向后兼容使用解决方法的人。

    .. change::
        :tags: bug, declarative
        :versions: 1.0.0b1, 0.9.5
        :tickets: 3062

        当访问"__mapper_args__"字典时，从声明性混合或抽象类中复制该字典，因此declarative自身所做的修改将不会与其他映射的修改发生冲突。该字典的修改涉及"version_id_col"和"polymorphic_on"参数，用本地类/表映射的官方映射的列替换其中的列。

    .. change::
        :tags: bug, sql
        :versions: 0.9.5, 1.0.0b1
        :tickets: 3044

        修复了 INSERT..FROM SELECT 结构中的一个bug，在这种情况下，从 UNION 中进行选择会在匿名(例如未标记)子查询中包装该联合。

    .. change::
        :tags: bug, postgresql
        :versions: 0.9.5, 1.0.0b1
        :tickets: 3053

        在 PG  :class:`.HSTORE`  类型中添加了新的 "hashable=False" 标志，这是需要的，以允许ORM在请求混合列/实体列表中的ORM映射HSTORE列时跳过尝试对ORM映射HSTORE列进行"hash"的过程。补丁由Gunnlaugur Þór Briem提供。

    .. change::
        :tags: bug, orm
        :versions: 0.9.5, 1.0.0b1
        :tickets: 3055

        修复ORM中的bug，在子查询贪婪加载中，当跨越多个多态子类和多态加载的长连贪婪加载链时，可能会失败，因此无法定位链中的子类链接，从而在  :class:`.AliasedClass`  上缺少属性名称。

    .. change::
        :tags: bug, ext
        :versions: 0.9.5, 1.0.0b1
        :tickets: 3051, 3093

        修复可变扩展中的一个bug，其中   :class:`.MutableDict`  没有为 "setdefault()" 字典操作报告更改事件。

    .. change::
        :tags: bug, ext
        :versions: 0.9.5, 1.0.0b1
        :tickets: 3093, 3051

        修复了一个bug，其中  :meth:`.MutableDict.setdefault`  不返回现有值或新的值(在0.8版本中未发布该bug)。由Thomas Hervé发起的拉取请求。

    .. change::
        :tags: bug, mysql
        :versions: 0.9.5, 1.0.0b1

        添加对通过使用等号包括KEY_BLOCK_SIZE的索引进行反映的支持。涉及到  :class:`_schema.Table`  的“mysql_partition_by='value'”和“mysql_partitions='value'”。

    .. change::
        :tags: bug, orm
        :tickets: 3047
        :versions: 0.9.5, 1.0.0b1

        修复了ORM bug，其中   :func:`.class_mapper`  函数会遮盖由于用户错误导致的mapper配置期间发生的属性错误或KeyError。已将attribute/keyerror的catch设置得更具体，以不包括配置步骤。

    .. change::
        :tags: bug, sql
        :tickets: 3045
        :versions: 0.9.5, 1.0.0b1

        修复了一个bug，在  :meth:`_schema.Table.update`  和  :meth:` _schema.Table.delete`  中，如果将空   :func:`.and_()`  或   :func:` .or_()`  或其他空表达式应用于 INSERT 中，则将产生一个空WHERE子句。现在与   :func:`_expression.select`  保持一致。

    .. change::
        :tags: bug, postgresql
        :versions: 0.9.5, 1.0.0b1

        添加了一个新的“disconnect”消息“connection has been closed unexpectedly”，这与较新版本的SSL有关。Pull request由Antti Haapala提供。

.. changelog::
    :version: 0.8.6
    :released: 2014年3月28日

    .. change::
        :tags: bug, orm
        :tickets: 3006
        :versions: 0.9.4

        修复了ORM bug，其中更改对象的主键，然后将其标记为DELETE，将会无法将DELETE指向正确的行。

    .. change::
        :tags: feature, postgresql
        :versions: 0.9.4

        为psycopg2 DBAPI启用了“sane multi-row count”检查，因为这似乎在psycopg2 2.0.9中得到支持。

    .. change::
        :tags: bug, postgresql
        :tickets: 3000
        :versions: 0.9.4

        修复了由发布0.8.5 / 0.9.3的兼容性增强引起的回归，其中包括了专门适用于仅适用于8.1、8.2系列的PostgreSQL版本的索引反射。再次修复将具有以下非常棘手的查询用于PG版本<8.1的情况下的推荐查询的“int2vector”数据类型，该查询虽然非常hacky，但已经定位。

    .. change::
        :tags: bug, orm
        :tickets: 2995,
        :versions: 0.9.4

        修复了从0.8.3引入的回归，由于  :ticket:`2818` ，因此  :meth:` _query.Query.exists`  对于仅具有  :meth:`_query.Query.select_from`  条目但没有其他实体的查询将无法正常工作。

    .. change::
        :tags: bug, general
        :tickets: 2986
        :versions: 0.9.4

        调整了"setup.py"文件，以支持setuptools中可能未来将 "setuptools.Feature" 扩展删除的可能性。如果不存在此关键字，则在没有 setuptools 的情况下仍将成功设置 django，而不是回退到distutils。现在还可以通过设置DISABLE_SQLALCHEMY_CEXT环境变量来禁用C扩展构建，无论 setuptools 是否可用，该变量都适用。

    .. change::
        :tags: bug, ext
        :versions: 0.9.4
        :tickets: 2997

        修复了一个问题，在   :class:`.MutableDict`  以及   :func:` .attributes.flag_modified`  中，如果将属性重新分配给自己，则更改事件将不会传播。

    .. change::
        :tags: bug, orm
        :versions: 0.9.4

        改进了一个错误消息，当光标对象传递到   :func:`.class_mapper`  相关的函数而不是查询语句时，当尝试使用  :meth:` _query.Query.join`  使“left”侧确定为 ``None`` 并且失败时，错误消息会发生，因此错误已在显式检测到的情况下进行了处理。

    .. change::
        :tags: bug, sql
        :versions: 0.9.4
        :tickets: 2977

        修复了   :func:`.tuple_`  构造中的bug，其中实际上第一个SQL表达式的 "type" 会应用为比较元组值的 "comparison type" 的效果。在某些情况下，这会导致不适当的 "类型强制转换"，例如当一个具有String和Binary值的元组不应该将目标值不适当地强制为Binary时，即使左侧不是Binary。   :func:` .tuple_`  现在期望在其值列表中具有异构类型。

    .. change::
        :tags: orm, bug
        :versions: 0.9.4
        :tickets: 2975

        从  :mod:`sqlalchemy.orm.interfaces`  中删除过时的名称，并使用当前名称进行刷新，以便再次从模块进行 "import *"。

.. changelog::
    :version: 0.8.5
    :released: 2014年2月19日

     .. change::
        :tags: postgresql, bug
        :versions: 0.9.3
        :tickets: 2936

        添加了一个额外的次要 "could not send data to server" 到psycopg2断开连接检测的消息，它补充了现有的"could not receive data from server"，并已被用户观察到。

     .. change::
        :tags: postgresql, bug
        :versions: 0.9.3

        改进了对非常旧版本(8.1 之前) 的 PostgreSQL 反射行为的支持，以及对可能适用于其他PG引擎（假设Redshift将版本报告为<8.1的引擎）的支持。 "indexes"和"primary keys"的查询依赖于检查名为“int2vector”的所谓“int2vector”数据类型，该数据类型在8.1之前拒绝强制为数组，从而导致有关在查询中使用的"ANY()"操作的失败。谷歌的范围广泛，但已经定位到非常hacky的，但是建议使用PG-Core-Developer查询的查询，以用于版本<8.1的情况下的索引和主键约束反射现在可以正常工作。

    .. change::
        :tags: bug, mysql
        :tickets: 2941
        :versions: 0.9.3

        添加了新的MySQL特定的   :class:`.mysql.DATETIME` ，其中包括分数秒支持；   :class:` .mysql.TIMESTAMP`  也添加了分数秒支持。DBAPI支持有限，尽管MySQL Connector/Python已知支持分数秒。 感谢Geert JM Vanderkelen。

    .. change::
        :tags: bug, mysql
        :tickets: 2966
        :versions: 0.9.3

        添加了对“PARTITION BY”和“PARTITIONS”MySQL表关键字的支持，指定为“mysql_partition_by ='value'”和“mysql_partitions ='value'”到   :class:`_schema.Table` 。感谢Marcus McCurdy的拉动请求。

    .. change::
        :tags: bug, sql
        :tickets: 2944
        :versions: 0.9.3

        修正了  :meth:`_expression.Insert.values`  与空列表或元组一起调用时会引发IndexError。现在将产生一个空插入操作，与空字典的情况相同。

    .. change::
        :tags: bug, engine, pool
        :versions: 0.9.3
        :tickets: 2880, 2964

        修复  :ticket:`2880`  引入的严重回归，其中从池返回连接的并发能力意味着“first_connect”事件现在也不再同步，因此根本不需要dialect的" is_disconnect"例程同时包含包装和" is_disconnect"例程的异常。连接过程本身。

    .. change::
        :tags: bug, sqlite

        恢复了唯一约束反射的变化，在SQLite中使用   :class:`.UniqueConstraint`  时，如果保留字包含在列名中，则会失败。现在忽略保留字，并将其删除。感谢Roman Podolyaka。

    .. change::
        :tags: bug, postgresql
        :tickets: 2291
        :versions: 0.9.3

        修订这个非常古老的问题，在PostgreSQL中， "get primary key" 反射查询更新为考虑重命名的主键约束。新查询在旧版本的PostgreSQL(例如版本7)上就会失败，好像只支持CAST到varchar的int2vector类型。现在在检测到 server_version_info <(8，0)的情况下，将使用旧查询。

    .. change::
        :tags: bug, sql
        :tickets: 2957
        :versions: 0.9.3

        修复了  :meth:`.ColumnOperators.in_`  的Bug，如果错误地传递了包含"__getitem__()"方法的列表达式(例如，使用   :class:` _postgresql.ARRAY`  类型的列)，将进入无限循环。

    .. change::
        :tags: bug, orm
        :tickets: 2951
        :versions: 0.9.3

        修复了一个bug，其中  :meth:`_query.Query.get`  将无法一致地提出引发   :class:` .InvalidRequestError`  的查询，该查询在具有现有标准的身份映射中已存在于标识映射中时。

    .. change::
        :tags: bug, mysql
        :tickets: 2933
        :versions: 0.9.3

        修复了 Py3K 错误，一个缺少导入将导致 "literal binary" 模式在渲染绑定参数时无法导入 "util.binary_type"。0.9以不同的方式处理此问题。

    .. change::
        :tags: bug, orm
        :versions: 0.9.2

        添加到cymysql方言中的一些丢失的方法，包括 _get_server_version_info() 和_detect_charset()。Pullreq由Hajime Nakagami提供。

    .. change::
        :tags: bug, orm
        :tickets: 2818
        :versions: 0.9.0

        由于  :ticket:`2818`  引入的更改，这是：如果在只有  :meth:` _query.Query.select_from`   条目但没有其他实体的查询中调用  :func:`._query.Query.exists` , 将引发“替换列”的警告，对于语句具有两个相同命名的列会产生警告，因为内部SELECT没有使用标签。

-   :class:`_schema.Column` .foreign` 可能会产生问题，因为会进行注释的固有复制操作，这导致父表未在联接中呈现。
- 非工作的 :class:`_schema.ForeignKey` 上的"schema"参数已被废弃;引发了一个警告。在0.9中将被删除。
- 修复了PostgreSQL版本字符串无法解析的错误，该字符串具有在单词"PostgreSQL"或"EnterpriseDB"之前的前缀。感谢Scott Schaefer。
- 现在， :class:`.URL` 的 :class:`_engine.Engine` 表述将使用星号隐藏密码。感谢Gunnlaugur Þór Briem。
- 修复  :meth:`_query.Query.exists`  无WHERE条件时无法正常工作的bug。感谢Vladimir Magamedov。
- 修复了使用`column_reflect`事件更改传入的  :class:`_schema.Column` .key` 时会防止正确反映主键约束，索引和外键约束的bug。
- 增加了一个新标志：`system=True`，用于  :class:`_schema.Column` ，将列标记为"系统"列，数据库会自动使其存在（例如PostgreSQL ` oid`或`xmin`）。列将从`CREATE TABLE`语句中省略，但其他情况下可用于查询。此外， :class:`.CreateColumn` 结构可以应用于自定义编译规则，该规则允许跳过列，即生成一个返回''的规则。
- 从0.9中回退了一项更改，此更改也适用于0.8.子级映射器层次结构的迭代现在进行了排序，这有助于具有确定性渲染的多态查询生成的SELECT语句，而这又有助于缓存方案，在SQL字符串本身缓存时缓存该字符串本身。
- 在ORM中使用的有序序列实现中的潜在问题已过去，该问题用于迭代器继承加载器层次结构；在Jython解释器下，此实现未排序，即使cPython和PyPy都保持排序。
- 在版本控制示例中创建历史记录表，添加了“autoincrement=False”，因为在任何情况下，此表都不应具有autoinc，感谢Patrick Schmid。
- :meth:`.ColumnOperators.notin_` 运算符现在正确地生成`IN`返回的否定表达式，即使针对空集合使用该运算符时。
- 改进了``examples/generic_associations``中的一些示例，包括``discriminator_on_association.py``使用单表继承执行"discriminator"的工作。还增加了一个真正的"通用外键"示例，其工作方式类似于其他流行的框架。它使用开放式整数来指向任何其他表，这样就可以省略传统的参照完整性。虽然我们不推荐此模式，但是信息想要变得自由。
- 增加了一个方便的类装饰器：func:`.as_declarative`，它是 :func:`.declarative_base` 的一个包装器，允许使用嵌套的类进行传递。
- 修复了ORM中事件注册的bug，其中"原始"或"propagate"标志在某些"未映射的基类"配置中可能会配置错误。
- 改进了  :func:`.defer` ` defer（）``的目的是减少DB/网络开销，而不一定是函数调用计数）；在所有情况下，函数调用开销现在小于从列加载数据的开销。此外，从N（结果中的总延迟值）到1（延迟列的总数）每次加载的“懒惰可调用”对象数量也有所减少。
- 为PostgreSQL 9.2范围类型添加了支持。目前，不提供类型转换，因此当前可以直接使用字符串或psycopg2 2.5范围扩展类型。修补程序由Chris Withers提供。
- 出现了一个HSTORE类型的错误，其中包含了反斜杠引号的键/值将无法在使用HSTORE数据的“非本机”（即非psycopg2）方法转换时正确转义。感谢Ryan Kelly。
- 修复了多种错误，否则会导致在：meth:`_query.Query.order_by`中发送复合属性产生括号表达式，这是某些数据库不接受的。
- 修复  :meth:`_schema.MetaData.reflect`  在远程模式和本地模式均具有相同名称的表时可能会产生错误结果的bug。
- 修复了  :meth:`_schema.Index`  中使用的` mysql_length`参数，使其成为列名称/长度的字典，供复合索引使用。感谢Russell Stuart。
- 在  :meth:`_orm.relationship`  中添加了一个新方法  :meth:` _query.Query.select_entity_from`  ，该方法将在0.9中替换  :meth:`_query.Query.select_from`  的部分功能。在0.8中，这两种方法执行相同的功能，因此可以根据需要将代码迁移到使用  :meth:` _query.Query.select_entity_from`  方法。详见0.9迁移指南。
- 修复Bug：多对多关系使用`uselist = False`将无法删除关联行并引发错误，如果将标量属性设置为None。这是由为：ticket:`2229`更改引起的回归。
- ORM映射对象中对于强Session引用的创建已做出改进;当该对象处于瞬态状态或进入分离状态时，将不再创建内部引用循环——只有当对象附加到Session并且在分离时才创建强引用。即使不建议使用这种方法，一个对象仍然可以具有 `¨del（）` 方法，因为关系与backrefs一起产生循环。映射了一个带有 `¨del（）方法的类`，增加了一个警告。
- SQLAlchemy延迟规则的错误已修复，原先用于防止在自定义运算符上实现`__getitem __（）`方法的对象调用`list（）`时出现无限且扩大内存的循环，但这实际上会使列元素报告它们是可迭代类型，然后尝试循环时抛出错误。这里无法有太多选择，所以我们坚持最佳实践。在自定义运算符上实现`__getitem __（）`方面要特别小心！
- 定义`id`属性的便利方法，该方法将查询转换为一个形如 `EXISTS（SELECT 1 FROM ... where ...)`的子查询。
- 修复bug：由于在多表更新时，辅助表是具有其自己的绑定参数的SELECT，因此绑定参数的位置与语句本身相比会颠倒，而使用多表更新时出现`mysql+gaerdbms`方言的一些问题。
- 提供额外的条件用于将"disconnect"与psycopg2/libpq检测到的字符串消息进行检查，以检测已知异常类型层次结构中的所有不同"断开连接"消息。特别是，在至少三种不同的异常类型中，已经发现了“意外关闭连接”的消息。感谢Eli Collins。
- 由于MySQL配置“不锁定外键”，将不再在MySQL方言上渲染“deferrable”关键字，原因是很长一段时间以来，可撤销的外键与不可撤销的外键之间的差异非常大，但是一些环境仅在MySQL上禁用FK，因此我们将在此处不再具有意见。
- 修复了 ORM 中集合属性和`orm.join()`函数的序列化。使得方便在SQLAlchemy中对序列化的对象和项目进行反序列化并生成JSON。
- 更新了用于从Google App Engine v1.7.5及更高版本开始的mysqlconnector 方言的正则表达式，以正确提取错误代码。由Dan Ring赞助的。
- 修复了ORM映射对象的反序列化的条件分支，以便在对象被拾取器序列化时丢失引用时，我们不会错误地尝试设置`_sa_instance_state`，以修复NoneType错误。
- 修复了多个与 CPython 2.x 兼容BUG，可能会导致在CPython 2.x 上出现未经处理的异常。修复来自Ben Trofatter。
- 对于从多个外键路径扩展到表"B"的SELECT()，其与多个外键路径扩展到表"A"直接连接到"B"的情况不同，现在会正常出现很多冲突的连接条件。此前在一些情况下，在SQLAlchemy 0.7会在另一个“duped”列上静默地发出第二次SELECT， 而在SQLAlchemy 0.8则会发出模糊的列错误。现在的 'keys' 应用到 .C. 集合，现在每个 SELECT() 中的“被替换的列”警告将不再发出线程阻断 切换，因为多个访问它们 的线程排他地获取一个链接，这个未释放的连接一直到访问的所有线程都已经完成后才会被释放。0.8.0版本变更日志：

问题修复：

修复了“metadata.create_all”和“metadata.drop_all”方法的参数存在且所包含的内容为空列表时无法创建/删除任何项目的问题。

修复了DBAPI可能会返回“0”来标识最后一行ID的情况下，不能在“ResultProxy.inserted_primary_key”中正确运行的问题。

修复了在“mapper”实例配置时，ORM在刷新映射类时会运行错误类型的查询的问题。

修复了描述符“MapperProperty”上的“__repr __()”方法无法在对象初始化之前运行的问题。

修复了删除对象后“object_session”函数仍返回相关“Session”的情况。

改进：

添加了“QueryableAttribute.info”属性，该属性默认直接从“Column”对象中的“.info”属性继承特性，它也可以从符合条件的“MapperProperty”对象中继承。

现在“KeyedTuple”类拥有“_asdict”属性和“_fields”属性。

现在“_expression.Insert”结构支持插入多个对象。

现在“postgresql.ARRAY.Comparator.all”和“postgresql.ARRAY.Comparator”支持“any”方法。

现在默认情况下允许在“Table()”方法中传递没有特定数据库引擎的选项。只需要检查参数格式即可，而无需在SQLAlchemy方言中查找该引擎。

添加了更多的“disconnect”消息来支持“pymssql”扩展。

现在对“Oracle”数据类型支持表反射和导入。

现在支持表的基本键和次要键的ID属性改变的情况。

现在允许使用同义词来定义关系的主要/次要连接。

现在可以使用“query.select_from”方法和“aliased”构造，因此当前方案返回合法的“SELECT” SQL语句。

添加了新参数“inherit_schema”到“Enum”和其基本类“SchemaType”，如果设置为“True”，则类型将继承其所属的“Table”的" schema"属性。

现在支持通过用于“column”注释的任意SQL表达式或函数，而不是仅仅是直接使用列名来创建“Index”。

各类更新：

现在，SQL表达式如“select()”和“alias()”将与映射实体如“aliased”构造分开来处理。

增加了一些关于ORM/扩展的新内部事件函数，用来检索ORM或ORM扩展中关联的所有Python描述符。

现在“ColumnOperators.in_”运算符把“None”值转换为“null”。

添加了一些新的列修饰符“mssql_include”和“mssql_clustered”，来指定用于呈现“INCLUDE”和“CLUSTERED”关键字的SQL语句。

现在只有当运算符为“==”或“! =”时，“__nonzero__”方法返回一个值- 其他情况抛出“TypeError”。

现在表别名和内部路径可以精准地处理子类名称相同的关系。

现在查找移动信息架构表中表名和架构名称的表的所有信息模式查询都将增加一个新参数，目的是避免出现比较“NVARCHAR”和“NTEXT”的问题。

修复了在连接对象已关闭的情况下检测到“disconnect detect on error”的bug。

改进了包含ORM继承映射类的查询时ORM将运行错误查询的bug。现在，当父类被映射到一个非“Table”对象（例如自定义“join()”或“select()”）时，ORM只会运行适用于每个类的单独“Table-per-class”查询。比如类似这样的查询之前会运行错误“SELECT * FROM foo”，现在会为“foo”类和继承它的“bar”类分别运行查询语句。

在ORM中加入了“__repr __()”方法，使其在对象初始化之前工作。这是为了使使用最新Sphinx版本生成文档时可以正确读取它们。

PS：其中包含的英文标点符号不能转化为中文标点符号！.. 相关记录::
    :version: 0.8.0b1
    :released: 2012年10月30日

    .. 更改::
        :tags: mssql, feature
        :tickets: 2600

        支持反射主键约束的“名称”，由 Dave Moore 提供。

    .. 更改::
        :tags: informix

        已删除有关 informix 事务处理的无用代码，包括一个跳过调用 commit()/rollback() 和在 begin() 中一些硬编码隔离级别假设的特性。由于我们没有任何使用它的用户，也没有访问 Informix 数据库的权利，所以此方言的状态不为人所知。如果有访问 Informix 的人想要帮助测试此方言，请告诉我们。

    .. 更改::
        :tags: pool, feature

          :class:`_pool.Pool`  现在将记录所有连接的 close() 操作，包括用于无效连接、分离连接、超出池容量的连接。

    .. 更改::
        :tags: pool, feature
        :tickets: 2611

          :class:`_pool.Pool`  现在在与事务相关的功能方面咨询   :class:` .Dialect` ，包括连接应该如何“自动回滚”以及关闭。这授予方言更多的事务范围控制，因此我们将更好地能够实现事务性处理的解决方案，如可能需要为 pysqlite 和 cx_oracle 做的解决方案。

    .. 更改::
        :tags: pool, feature

        添加新的  :meth:`_events.PoolEvents.reset`  钩子，以捕获连接在返回池之前自动回滚的事件。以及  :meth:` _events.ConnectionEvents.rollback` ，这允许拦截所有回滚事件。

    .. 更改::
        :tags: feature, sql
        :tickets: 2547

        重大修改，用于重新定义已有的操作符以及添加新的操作符。新类型可以从现有类型创建，这些新类型在列表达式中释放成为绑定和列表达式。这是适合于透明加密/解密、使用 PostGIS 函数等方案的使用案例。

    .. 更改::
        :tags: bug, sql
        :tickets: 2564

        修复了当参数作为 Python bytearray 实例时，PostgreSQL 环境中某些情况下的 "set" 操作会导致字节换行符被关闭的问题。



    .. 更改::
        :tags: informix

        已删除有关 informix 事务处理的无用代码，包括一个跳过调用 commit()/rollback() 和在 begin() 中一些硬编码隔离级别假设的特性。由于我们没有任何使用它的用户，也没有访问 Informix 数据库的权利，所以此方言的状态不为人所知。如果有访问 Informix 的人想要帮助测试此方言，请告诉我们。

    .. 更改::
        :tags: pool, feature

          :class:`_pool.Pool`  现在将记录所有连接的 close() 操作，包括用于无效连接、分离连接、超出池容量的连接。

    .. 更改::
        :tags: pool, feature
        :tickets: 2611

          :class:`_pool.Pool`  现在在与事务相关的功能方面咨询   :class:` .Dialect` ，包括连接应该如何“自动回滚”以及关闭。这授予方言更多的事务范围控制，因此我们将更好地能够实现事务性处理的解决方案，如可能需要为 pysqlite 和 cx_oracle 做的解决方案。

    .. 更改::
        :tags: pool, feature

        添加新的  :meth:`_events.PoolEvents.reset`  钩子，以捕获连接在返回池之前自动回滚的事件。以及  :meth:` _events.ConnectionEvents.rollback` ，这允许拦截所有回滚事件。

    .. 更改::
        :tags: engine, bug
        :tickets:

        此项更改删除了 ResultProxy.last_inserted_ids，用 inserted_primary_key 替换。

    .. 更改::
        :tags: general
        :tickets:

        SQLAlchemy 0.8 现在支持 Python 2.5 及更高版本。Python 2.4 不再受支持。

    .. 更改::
        :tags: removed, general
        :tickets: 2433

        完全删除了“sqlalchemy.exceptions”的同义词“sqlalchemy.exc”。

    .. 更改::
        :tags: removed, orm
        :tickets: 2442

        已删除 ORM 的传统“可变”系统，包括 MutableType 类以及 PickleType 和 postgresql.ARRAY 上的 mutable=True 标志。通过使用 sqlalchemy.ext.mutable 扩展在 0.7 中介绍，ORM 可以检测不落地的变化。删除 MutableType 和相关构造从 SQLAlchemy 的内部删除了大量复杂性。在使用时，该方法的效率低下，因为在使用时它将扫描 Session 的全部内容。

    .. 更改::
        :tags: orm, moved
        :tickets:

        InstrumentationManager 的接口和所有相关的修改了类实现的系统现已移动到 sqlalchemy.ext.instrumentation 中。这是几乎不常用的系统，它增加了对类仪器装置机制的力度和开销。新的体系结构允许它保持未使用状态，直到 InstrumentationManager 实际导入，此时它会引导进核心。

    .. 更改::
        :tags: orm, feature
        :tickets: 1401

        对 relationship 内部的重大改写，现在允许包含指向具有复合外键的自身的列的连接条件。添加了一个新的 API 供专门的 primaryjoin 条件使用，允许使用 SQL 函数、CAST 等来处理需要时将注释函数 remote() 和 foreign() 内联放置在表达式中。以前使用半私有的 _local_remote_pairs 方法的方法可以升级到此新方法。

    .. 更改::
        :tags: orm, bug
        :tickets: 2527

        如果通过公共继承进行关联，并且 FK 依赖关系没有成为继承条件的一部分，则在提交期间，ORM 将尽额外的努力来确定此类依赖关系是否具有意义，从而节省使用 use_alter 指令的用户的代价。

    .. 更改::
        :tags: orm, feature
        :tickets: 2333

        新的单独函数 with_polymorphic()，提供了在查询中使用 query.with_polymorphic() 可以应用到任何查询实体的功能，包括作为 join 中的目标，用于替代 "of_type()" 修饰语。

    .. 更改::
        :tags: orm, feature
        :tickets: 1106, 2438

        现在，of_type() 属性上的构造函数接受了别名化类构造函数以及 with_polymorphic 构造函数，并且使用 query.join()、any()、has() 以及一些特定的饥饿式类型加载器 subqueryload()、joinedload()，contains_eager()。

    .. 更改::
        :tags: orm, feature
        :tickets: 2585

        改进了对映射类的事件监听器，允许指定未映射的类用于实例和映射器事件。当传递 propagate=True 选项时，当该类实际映射时，这些已建立的事件将自动设置为子类。有关更多信息，请参阅事件文档。

    .. 更改::
        :tags: orm, bug
        :tickets: 2590

        仪器化中的事件 class_instrument()、class_uninstrument() 和 attribute_instrument() 现在仅针对监听器指定的子类启动，为对象分配的监听器，而不考虑传递的 "target" 实参的情况。

    .. 更改::
        :tags: orm, bug
        :tickets: 1900

        如果在提交期间（如 before_flush()、before_update() 等中）内部发出懒加载，例如调用 SomeClass.（somename），则现在将与在非事件代码内部一样运作，即从惰性发射查询中考虑到PK/FK 值。先前，会建立特殊标志，导致惰性目录在基于其前一个父 PK/FK 值而不是基于当前对象状态查询其相关项。在提交事件内部使用惰性加载时，缺省参数"passive_updates" 的使用或不使用可能会影响集合在提交事件内部访问时还是代表“旧”的或“新”的数据，具体取决于发出惰性加载的时间。这个改变在极少的情况下是向后不兼容的，那就是用户事件代码依赖旧行为的情况。

    .. 更改::
        :tags: orm, feature
        :tickets: 2517

        declared_attr 现在可以用于不是 Column 或 MapperProperty 的属性，包括任何用户定义的值以及协会代理对象。

    .. 更改::
        :tags: orm, feature
        :tickets: 2229

        现在向映射的集合添加/删除 None 将产生属性事件。先前，在某些情况下，None 追加会被忽略。相关的。

    .. 更改::
        :tags: orm, feature
        :tickets: 2229

        映射的集合中出现 None 现在会在提交时引发错误。先前，在集合中出现 None 值时，不会发出任何警告。

    .. 更改::
        :tags: orm, feature
        :tickets: 2592

        查询现在可以加载实体/标量混合的"元组"行，其中包含不可哈希类型，通过设置类型引擎对象中"hashable=False" 标志。通常来说，客户类型返回不可哈希类型（通常是列表），可以将标志设置为 False。

    .. 更改::
        :tags: orm, bug
        :tickets: 2481

        改进了已加入继承的子类实体之间存在关系的连接/子查询饥饿加载逻辑，这些子类实体具有公共的基类，且未提供特定的"join depth"。将在检测到“cycle”之前依次到每个子类 mapper 上链接，而不是将基类视为“cycle”的源。

    .. 更改::
        :tags: orm, bug
        :tickets: 2320

        Session.is_modified() 中的 "被动" 标志现在不再有任何效果。在所有情况下，is_modified() 仅查看本地的 in-memory 修改标志，并且不会发出任何 SQL 或调用加载器可调用/初始化器。

    .. 更改::
        :tags: orm, bug
        :tickets: 2405

        现在在一对多或多对多的 delete-orphan 级联中，如果未单独引用一个父项，则当 single-parent=True 未启用时发出警告消息。不管在任何情况下 ORM 都不能在此警告之后工作。

    .. 更改::
        :tags: orm, bug
        :tickets: 2350

        现在在 flush 事件内发出的惰性加载（如 before_flush()、before_update() 等中）将按照非事件代码内执行时的方式运行程序，考虑每个 PK/FK 值的使用情况。之前，特殊标志将被确立，导致惰性负载基于先前的父 PK/FK 值从相关项目中载入，具体取决于何时在提交事件内发射了惰性加载。现在，仅在单元必须以这种方式加载时， lazy 负载才会这样加载。请注意，UOW 有时会在 before_update() 事件被调用之前加载这些集合，因此在 flush 事件内访问这些集合时， "passive_updates" 的使用或不使用将影响集合代表的“旧”还是“新”数据，具体取决于何时发射了惰性加载。这个改变在极少的情况下是向后不兼容的，那就是用户事件代码依赖旧行为的情况。

    .. 更改::
        :tags: orm, feature
        :tickets: 2179

        Query 现在“自动关联”与 select() 相同的方式，默认情况下。现在，用作子查询的查询需要显式调用 correlate() 方法以将内部的表关联到外部。与往常一样，correlate(None) 取消关联。

    .. 更改::
        :tags: orm, feature
        :tickets: 2464

        在 Session.new 或 Session.identity_map 上建立对象时，会在该对象代理的状态下发出 after_attach 事件，从而在事件调用时表示该对象。已添加 before_attach 事件来适应需要带有属性附加的自动闪存的对象的用例。

    .. 更改::
        :tags: orm, feature
        :tickets:

        现在在提交计划的执行中使用不受支持的方法，会发出警告，包括 add()、delete() 等、集合和关联对象操纵，如在 mapper-level flush 事件中调用的那些方法，如 after_insert()、after_update() 等。长期以来，它已经被重点记录，即当 Session 在执行闪存计划过程中发生了操作时，SQLAlchemy 无法保证结果，但用户仍在做这件事，所以现在有了一个警告。也许有一天，Session 将被增强以支持这些操作，在 flush 中，但现在无法保证结果。

    .. 更改::
        :tags: orm, feature
        :tickets: 2517

        现在可以通过字符串名称或完全限定名称来基于关系() 函数中实体的模块查找多个具有相同名称的类。

    .. 更改::
        :tags: orm, feature
        :tickets: 2595

        服务器端使用的 auto-correlation 特性，现在在查询直接呈现在 FROM 列表中的 SELECT 语句时不会生效了。SQL 中的 correlation 仅适用于列
        表达式，例如 WHERE、ORDER BY、列子句。

    .. 更改::
        :tags: orm, bug
        :tickets: 2593

        修复了传递给  :meth:`.Compiler.process`  的关键字参数不会传播到 SELECT 语句的列子句中存在的列表达式中的 bug。特别是，当使用依赖于特殊标志的自定义编译方案时，这将出现。

    .. 更改::
        :tags: orm, feature
        :tickets: 2179

        Query 现在在以下语句块中能够使用由 processors 处理的剪贴板 (<tablename>_<colkey>)：SELECT、UPDATE、DELETE、INSERT。

    .. 更改::
        :tags: orm, feature
        :tickets: 2245

        接受了从 ORM 实体派生的列举用于 select() 构造并传递给 select_from()、correlate() 以及 correlate_except() 方法时，它们将被直接解压缩为可选的可选择。### 修订

- 调整列优先级，将"concat"和"match"运算符的优先级与"is"，"like"等相同；这有助于在与"IS"结合使用时进行括号呈现。

### 增强

- `GenericFunction`和`func.*`可以允许用户定义的`GenericFunction`子类按照类名自动通过`func.*`命名空间提供，可选择使用包名，以及具有与在`func.* `中标识的名称不同的呈现名称的功能。

- `cast（）`和`extract（）`构造现在也将通过`func.*`访问器生成，由于用户通常尝试从`func.*`访问这些名称，因此它们可能会做出预期，即使返回对象不是`FunctionElement`。

- `expression.sql` 中的大多数类现在不再在前面加下划线，即`Label`，`SelectBase`，`Generative`，`CompareMixin`。`_BindParamClause`也重新命名为 `BindParameter`。对于这些类的旧下划线名称将在将来可预见的情况下保留为同义词。

- Inspector对象现在可以使用新的检查服务`inspect()`获得，作为 ...

- `column_reflect`事件现在将`Inspector`对象作为第一个参数接受，其前面是“table”。使用0.7版本这种非常新的事件的代码将需要修改以添加“验证者”对象作为第一个参数。

- 现在，默认情况下结果集中的列定位行为区分大小写。多年来，SQLAlchemy一般会在这些值上运行不区分大小写的转换，可能是为了缓解Dialect（例如Oracle和Firebird）的早期大小写敏感性问题。这些问题已经在更现代的版本中得到更有效的解决，因此调用标识符上的`lower（）`的性能开销被删除。不区分大小写的比较可以通过在`create_engine()`上设置“`case_insensitive = False`”来重新启用。

- 现在，将应用添加到`select`语句的列表达式，使用具有或不带其他修改构造的标签，不再将其"target"到底层`Column`；这影响依赖于 Column 定位以便检索结果的ORM操作。也就是说，查询`query（User.id，User.id.label（'foo'）`现在将单独跟踪每个`User.id`表达式的值，而不是将它们混合在一起。不希望受到影响任何用户，但是使用`select()`与`query.from_statement()`相结合的用法，并尝试以任意`.label()`名称的select()列对象加载完整组合ORM实体可能不能如预期地运行，因为这些对象不再定位到那个实体映射的`Column`对象。

- `insert.values()`或`update.values()`中存在键，但不在目标表中的“未使用的列名”警告现在是异常。

- `ForeignKey`和`ForeignKeyConstraint`添加了“MATCH”子句，由Ryan Kelly提供。

- 添加了从表的别名到DELETE和UPDATE的支持，假定在查询中的其他位置与其相关，由Ryan Kelly提供。

- 现在在`select()`上提供`correlate_except()`方法，自动排除除传递的所有可选择之外的所有可选择。

- `prefix_with()`方法现在在每个`select（）`，`insert（）`，`update（）`，`delete（）`上都可用，使用相同的API接受多个前缀调用，以及“方言名称”，以便将前缀限制为一种语言方言。

- 将`reduce_columns（）`方法添加到`select（）`构造中，内联替换列使用`util.reduce_columns`实用程序函数以删除等效列。`reduce_columns()`还添加了“with_only_synonyms”以将减少仅限于拥有相同名称的那些列。已删除已弃用的`fold_equivalents()`功能。

- 重新设计了`startswith()`，`endswith()`和`contains()`操作符，以更好地支持否定（NOT LIKE）并在编译时装配它们，以便可以更改它们的呈现SQL，例如Firebird的情况下开头叫STARTING WITH。

- 添加了一个系统的渲染CREATE TABLE的钩子，通过针对新的 schema.CreateColumn构造一个@compiles函数，可以为每个列提供单独的渲染。

- “标量”选择现在具有WHERE方法，以帮助进行创造性的构建。此外，关于如何“关联”列的SS的略微调整;新方法不再适用于正在选择的基础表列。这改善了一些相当偏门的情况，并且似乎存在的逻辑没有任何目的。

- 修复了将列“default”参数解释为可调用参数的问题，以便不要将ExecutionContext传递给关键字参数。

- 当它们直接引用表绑定的列对象（即不仅仅是字符串列名称），并且仅引用一个表时，所有UniqueConstraint、ForeignKeyConstraint、CheckConstraint和PrimaryKeyConstraint将自动附加到它们的父表上。在0.8之前，此行为适用于UniqueConstraint和PrimaryKeyConstraint，但不适用于ForeignKeyConstraint和CheckConstraint。

- TypeDecorator现在包含by默认在“impl”类型中工作的通用repr()。对于指定自定义`__init__()`方法的TypeDecorator类，这是一种行为上的变化;如果这些类型需要`__repr__()`提供忠实的构造函数表示形式，则这些类型将需要重新定义`__repr__()`

- `column.label(None)`现在产生匿名标签，而不是返回列对象本身，与`label(column, None)`的行为一致。

- 显式错误会在第一次使用被构造用于引用多个远程表的`ForeignKeyConstraint()`时引发。

- MS Access方言已移至Bitbucket上的自己的项目中，利用新的SQLAlchemy方言符合性套件。然而，该方言仍处于非常粗略的状态，可能尚未准备好进行一般使用。 https://bitbucket.org/zzzeek/sqlalchemy-access

- 多年未运行的MaxDB方言移至挂起的Bitbucket项目 https://bitbucket.org/zzzeek/sqlalchemy-maxdb中。

- SQLite日期和时间类型已进行了大幅改进，以支持更开放的格式以进行输入和输出，使用基于名称的格式字符串和正则表达式。新的“`microseconds`”参数还提供了省略时间戳“毫秒”部分的选项。感谢Nathan Wright提供工作和测试。

- SQL Server方言现在可以给予数据库合格的模式名称，即“schema ='mydatabase.dbo'”；反射操作将检测此，将模式分成“。”来单独获取所有者，并在反映目标之前发出“USE mydatabase”语句，“dbo”所有者;然后恢复现有的从DB_NAME（）返回的数据库。

- 已删除将`==`用于将列比较为标量`SELECT`的旧的不可用行为-of用SQL Server方言。这是隐含的行为，在其他情况下会失败，因此被删除。依赖于此的代码需要修改为明确使用`column.in_（select）`。

- 更新了mxodbc驱动程序的支持; mxodbc 3.2.1建议进行全面兼容性。

- postgresql.ARRAY带有可选的“尺寸”参数，将代表特定数量的数组，将呈现为DDL中的ARRAY[][]...，还提高了绑定/结果处理的性能。

- `postgresql.ARRAY`现在支持索引和切片。所有类型为ARRAY的SQL表达式都可以使用Python []运算符；可以传递整数或简单片段。还可以在UPDATE语句的SET子句中使用这些片段，在Update.values()中将它们传入。请参阅文档以获取示例。

- 添加了新的“‘array literal’”构造postgresql.array()。基本上，它是一个将呈现为ARRAY [1,2,3]的“元组”。

- 添加了支持PostgreSQL ONLY关键字的新功能，该关键字可以对应于SELECT，UPDATE或DELETE语句中的表。该短语使用with_hint（）建立。由Ryan Kelly提供

- PostgreSQL方言的"ischema_names"字典是“非官方”自定义的。也就是说，可以将新类型添加到此字典中，例如PostGIS类型，该PG类型反射代码应开始处理具有可变数目参数的简单类型。这里的功能是“非官方”的，因为：

1.这不是“官方”的API。理想情况下，“官方”的API将允许在方言或全局级别以一种通用的方式使用自定义类型处理可调用的API。

2.这仅针对PG方言实现，特别是因为PG具有广泛支持自定义类型，而其他数据库后端没有。将在默认方言级别实现真正的API。

3.这里的反射代码仅针对简单类型进行了测试，并且可能存在更多组合类型的问题。

- "startswith()"运算符现在将呈现为“STARTING WITH”，“~startswith()”后者渲染为“NOT STARTING WITH”，使用FB的更高效运算符。

- 制造`VARCHAR`，并省略长度时将引发`CompileError`，与MySQL相同。

- 现在，Firebird使用严格的“ansi绑定规则”，因此在语句的列子句中不会渲染绑定参数-它们的字面意思仍然会被显示出来

- 当使用DateTime类型与Firebird时，还支持将datetime作为date传递。其他方言支持这一点。

- 添加了fbd驱动程序的实验方言，但是未经测试，因为我无法使fbd包编译。

- 现在，dialect不再在第一次连接时发出昂贵的服务器排列查询，以及服务器大小写。这些功能仍然是半私有的。

- MySQL方言现在将TIME类型添加到其支持范围内，接受“fst”参数，该参数是最近MySQL版本的新的“分数秒”说明符。数据类型将解释从驱动程序接收到的微秒部分，但请注意，此时，大多数/所有MySQL DBAPI都不支持返回此值。

- 现在从具有quote = True的Column传递引用信息，当生成一个与绑定参数同名的构建参数时，就像在生成的INSERT和UPDATE语句中一样，这样可以完全支持未知的保留名称。

- 可以通过将字符串DBAPI类型名称的列表发送到`exclude_setinputsizes`方言参数来自定义将排除setinputsizes（）集的类型的列。先前，此列表是固定的。列表现在也默认为STRING，UNICODE，从中删除CLOB，NCLOB。

- Oracle构造中的CreateIndex将现在使索引的名称模式限定为父表的名称。先前，该名称被省略，这显然会在默认模式中创建索引，而不是表中的模式。

- 将`ColumnOperators.notin_`，`ColumnOperators.notlike`，`ColumnOperators.notilike`添加到`ColumnOperators`中。