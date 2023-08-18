=============
0.2 更新日志
=============


.. 更改日志::
    :version: 0.2.8
    :released: Tue Sep 05 2006

    .. 更改::
        :tags:
        :tickets:

      清理连接方法和文档。自定义的DBAPI参数在查询字符串中指定，'connect_args' 参数传递给'create_engine' 或通过'creator' 创造函数来自定义创建函数，用于'create_engine'。

    .. 更改::
        :tags:
        :tickets: 274

       添加 "recycle" 参数到Pool， 在 'create_engine' 中是"pool_recycle"，默认为3600秒；浪费这段时间没有使用的连接，并重新创建一个新连接，以处理自动关闭过期连接的数据库

    .. 更改::
        :tags:
        :tickets: 121

      当使用的一个pooled connection无效时，会指示底层连接记录在下一次调用它时重新连接。如果任何错误在底层调用connection.cursor()时抛出，那么"invalidate"也会自动被调用。这将希望连接池能够重新连接到一个已经停止并重新启动的数据库而无需重新启动连接应用程序。

    .. 更改::
        :tags:
        :tickets:

      eesh! 程序教程的doctest无法在相当长的时间内使用。

    .. 更改::
        :tags:
        :tickets:

      'mapper' 上添加 'add_property()' 方法，如果给定属性引用一个未编译的mapper（就像在教程中一样)时，可以在"编译所有映射程序"步骤中编译。


    .. 更改::
        :tags:
        :tickets: 277

      检查pg序列是否已经存在， 在create之前。


    .. 更改::
        :tags:
        :tickets:

      如果通过 MapperExtension.get_session 建立上下文会话（如在 sessioncontext 插件中使用等），则懒加载操作将使用默认情况下使用该session(如果父对象没有session)。


    .. 更改::
        :tags:
        :tickets:

      对于没有数据库标识符的对象，不会触发懒惰加载（为什么？请参见https://www.sqlalchemy.org/trac/wiki/WhyDontForeignKeysLoadData)


    .. 更改::
        :tags:
        :tickets:

      对于"delete-orphan"级联中的 "孤儿"对象的更好检查, 对于某些没有可用的父对象级联的条件，这将使它们不再是孤立的。

    .. 更改::
        :tags:
        :tickets:

      mappers可以根据与属性包的交互确定其对象是否为“孤儿”，当对象被附加和分离时，此检查基于每个关系维护的状态标志。

    .. 更改::
        :tags:
        :tickets:

      现在不能声明自引用关系与"delete-orphan"一起使用(因为以上提到的检查将使它们无法保存)。

    .. 更改::
        :tags:
        :tickets:

      对于 UOWTask 寻求刷新它们作为关系的一部分时，它现在更好地检查对象是否已经在会话中作为对象。


    .. 更改::
        :tags:
        :tickets: 280

       语句执行支持在表达式中多次使用相同的BindParam 对象，简化了定位参数的处理。 由Bill Noon可视化出的精彩想法。

    .. 更改::
        :tags:
        :tickets: 60, 71

      将postgres的反射移至使用pg_schema表，可以通过use_information_schema=True参数覆盖。

    .. 更改::
        :tags:
        :tickets: 155

      在 MetaData, Table, Column 中添加了 case_sensitive 参数，根据它是否有父模式项的not-None设置或不为AllLower的标识符名称自动确定,当设置为True时，引用将应用于大小写混合或大写标题的标识符。 对已知是保留字或包含其他非标准字符的标识符，引号会自动应用。各种数据库方言可以覆盖所有这些行为，但当前它们都使用默认行为。通过postgres, mysql, sqlite, oracle测试。 需要在firebird, ms-sql上进行更多测试。这是当前正在进行的工作的一部分

    .. 更改::
        :tags:
        :tickets:

      更新了unit测试，不需要安装任何pysqlite；pool测试使用虚拟DBAPI。

    .. 更改::
        :tags:
        :tickets: 281

      urls支持在密码中使用转义字符

    .. 更改::
        :tags:
        :tickets:

      在 UNION 查询中添加了 limit 和 offset （尽管oracle还没有）

    .. 更改::
        :tags:
        :tickets:

      在DateTime和Time类型上添加了“timezone=True”标志。 截至目前为止，postgress会将其转换为 "TIME [STAMP] (with-without) TIME ZONE"，从而使时区存在的控制变得更加可控（如果可用，psycopg2会返回带有tzinfo的日期时间，这可能会与不带tzinfo的日期时间产生混淆）。

    .. 更改::
        :tags:
        :tickets: 287

      修复了在具有不同查询的distinc时使用 query.count() 、 \**kwargs 的SelectResults 的count() 方法的错误

    .. 更改::
        :tags:
        :tickets: 289

      当 reflection 失败时，Deregister Table 并再次找到。

    .. 更改::
        :tags:
        :tickets: 293

      导入了py2.5s sqlite3

    .. 更改::
        :tags:
        :tickets: 296

      针对startswith()/endswith() 的unicode修复。

.. 更改日志::
    :version: 0.2.7
    :released: Sat Aug 12 2006

    .. 更改::
        :tags:
        :tickets:

      转义字符集设施的设置，允许在所有查询 /创建 /删除中对个别表、模式和列标识符启用特定于数据库的引用。试用在 Table 或 Column 中使用 "quote=True" ，在Table中的"quote_scheme=True" 以及默认情况下所有情况下应用于已知是保留字或包含其他非标准字符标识符的双引号。如果标识符没有已知方言，可能会覆盖所有这些行为，但目前它们都使用了默认行为。被测试了与postgres，mysql，sqlite，oracle。需要在firebird，ms-sql上进行更多的测试。

    .. 更改::
        :tags:
        :tickets:

      assignmapper设置is_primary=True，导致当设置了冗余映射器时未引发错误，已修正。

    .. 更改::
        :tags:
        :tickets:

      添加了允许将一些主键列为null的行（例如，在映射到外部连接等时）的 "allow_null_pks"选项给 mapper。

    .. 更改::
        :tags:
        :tickets:

      禁止了self-referential关系使用"delete-orphan"级联（因为通过上述检查原因会使它们无法保存）。

    .. 更改::
        :tags:
        :tickets:

      对于关系，强制应用父类标识符的“元”级联条件有效的外部/内部join条件

    .. 更改::
        :tags:
        :tickets:

      修复了使用pgsql时，当操作一个复合主键时，primarykey字段中多次反射原型对象的情况，而不是静态的。这可以通过传递一个名为 "primary_key_is_static = False" 的额外参数来禁用生成作为Tuple的复合主键元素的行为。

    .. 更改::
        :tags:
        :tickets:

      增加了映射到具有不同名称的外部模式的表的功能。

    .. 更改::
        :tags:
        :tickets:

      对于POSTGRES，表，自动反投影空间几何类型的反射。

    .. 更改::
        :tags:
        :tickets:

      关于 CREATE/DROP 在所有调用中都有关键字参数 "connectable"。 "engine" 已经被弃用。

    .. 更改::
        :tags:
        :tickets:

      将其改正，以便与adodbapi一起使用时，ms-sql的connect()功能可正常使用。

    .. 更改::
        :tags:
        :tickets:

      在Select()函数中添加了“nowait”标志

    .. 更改::
        :tags:
        :tickets:

      使用了更多具体的类型，因而更易于使用数据库特定类型；mysql文本类型的修复也可以适用于此方法

    .. 更改::
        :tags:
        :tickets:

      对于sqlite的日期类型组织做了一些修复。

    .. 更改::
        :tags:
        :tickets: 263

      MS-SQL现在在数值类型中添加了反映的'tinyint'，'mediumint'类型等。

    .. 更改::
        :tags:
        :tickets: 267, 265

      修复与惰性装入器结合使用的一些 pickle bug(s)。

    .. 更改::
        :tags:
        :tickets:

      对于mysql反射默认值的修复以成为PassiveDefault。

    .. 更改::
        :tags:
        :tickets: 224

      来自更改为使用溢出计数器导致的对连接无效的补丁。

    .. 更改::
        :tags:
        :tickets: 252

      在查询SelectResults上使用聚合（如：max，min 等）可以使用子查询。

    .. 更改::
        :tags:
        :tickets: 269

      修复了类型，从而更容易使用特定于数据库的类型；对mysql文本类型的修复

    .. 更改::
        :tags:
        :tickets:

      修复了在mysql反射中，对于某些版本，对于SHOW CREATE TABLE的调用返回数组而不是字符串的可能性。

.. 更改日志::
    :version: 0.2.6
    :released: Thu Jul 20 2006

    .. 更改::
        :tags:
        :tickets:

      对于polymorphic inheritance 的实现做了很大的改进，使其能够适用于连接列表表结构。这个改变修复了相关的漏洞。

    .. 更改::
        :tags:
        :tickets:

      重新实现的MapperExtension调用方案，以前并不是非常有效。

    .. 更改::
        :tags:
        :tickets:

      尝试修复到达两个引用彼此的映射器的遍历时在select_by()中的可能导致无限循环的错误。

    .. 更改::
        :tags:
        :tickets:

      更新了所有单元测试以将 “./lib/” 插入sys.path，以解决新的setuptools PYTHONPATH-killing行为。

    .. 更改::
        :tags:
        :tickets:

      属性字符串模块已被完全重写；它现在更加简单和清晰，速度略微提高。 属性的“历史”不再在每次更改时被精细地管理，而是作为实例首次加载时创建的“已提交状态”的一部分。HistoryArraySet消失了，列表属性的行为现在更加开放式(也就是它们不再是集合了)。

    .. 更改::
        :tags:
        :tickets:

      在使用type()检查类A是否继承自B时，使用issubclass()代替了直接的__mro__检查，以确保python示出类继承更加灵活。

    .. 更改::
        :tags:
        :tickets: 238

      尝试让unit-of-work中的FlushErrors没有在异常时停止整个过程。对于某些级联操作特别是考虑到backrefs，这会说明性能显着提高了。

    .. 更改::
        :tags:
        :tickets:

      单例线程池(SingletonThreadPool)有一个大小和一个清除，只有给定数量的线程本地连接会留在周围(需要于大量sqlite的程序来处理同时处置的线程)

    .. 更改::
        :tags:
        :tickets: 249

      修复文档。当构造application时，不应该使用ORM objects的__del__（）方法。应该使用SessionTools.close_all_sessions() 应及时关闭所有目前还建立的会话（或将采用好的方法）

    .. 更改::
        :tags:
        :tickets:

      原代码中出现的大量“stmt.compile（）”已被视为无效。

    .. 更改::
        :tags:
        :tickets: 243

      优化屏蔽列特征，对于一个列的关键字，也进行类型转换。

    .. 更改::
        :tags:
        :tickets:

      固定了对象构造失败时不添加到会话中的问题。

.. 更改日志::
    :version: 0.2.5
    :released: Sat Jul 08 2006

    .. 更改::
        :tags:
        :tickets:

      Mapper编译全面的延迟。这允许 mapper 在任何顺序下构建，并且它们之间的关系被编译当mapper首次使用。

    .. 更改::
        :tags:
        :tickets:

      处理级联行为(需要特别考虑backrefs)特别是针对OneToMany关系，之前会出现比较大的速度瓶颈，现在出现了更好的处理方式,得到了大改进。

    .. 更改::
        :tags:
        :tickets:

      现在支持py2.4的“set”构造函数，在需要排序时，使用py2.4中的“set”构造函数,否则就使用 set.Set 。

    .. 更改::
        :tags:
        :tickets:

      对于“给定对象失败构造”的情况，不添加到会话中。

    .. 更改::
        :tags:
        :tickets:

      py2.3上的修改，使得 SETSESSION CHARACTERISTICS AS TRANSACTION x可以正常工作。 可能在其他平台上有疯狂的开销。

    .. 更改::
        :tags:
        :tickets:

      开始实现基础架构的措施，使得ConnectionPool可以自动重连。

    .. 更改::
        :tags:
        :tickets:

      修复SQLAlchemy集群实现上的权重错误，现在随机地选择有效的池。


.. 更改日志::
    :version: 0.2.4
    :released: Tue Jun 27 2006

    .. 更改::
        :tags:
        :tickets:

      升级 Engine 系统，前任SQLEngine现在是ComposedSQLEngine，并且由许多组件组成，包括方言，连接提供者等。这影响所有db模块，以及Session和Mapper。

    .. 更改::
        :tags:
        :tickets:

     create_engine现在只接受RFC-1738样式的字符串：
     ``driver：//user：password@host：port / database``
     **更新**大体上应满足RFC-1738, 不过它不完全如此，其中"方案"部分接受下划线而不是破折号或点。


    .. 更改::
        :tags:
        :tickets:

      去除线程局部作用域(ConnectionLocal)的默认方式。正确的使用 'scoped_session' 和 'sessionmaker'已经被证实可以提供更安全，更精细的线程本地作用域控制。

    .. 更改::
        :tags:
        :tickets:

      修复了一个大大小小错误， Connection 对象现在可以直接执行条款元素，添加了显式“关闭”以及在整个引擎和ORM中处理关闭的支持， 不再依赖于内部的__del__函数来返回连接到池。

    .. 更改::
        :tags:
        :tickets:

      overhaul to connection-scoping methodology, Connection objects
      can now execute clause elements directly, added explicit "close" as
      well as support throughout Engine/ORM to handle closing properly,
      no longer relying upon __del__ internally to return connections
      to the pool.

    .. 更改::
        :tags:
        :tickets:

      Session界面和作用域的改变。使用类似hibernate的方法，包括 query(class),save(), save_or_update()等。默认情况下没有设置线程本地作用域。通过提供专用于特定引擎和/或连接的绑定接口，可以使底层模式对象不必绑定到引擎。添加了一个基本的会话事务(SessionTransaction)对象，可以简单地聚合多个引擎的事务。

    .. 更改::
        :tags:
        :tickets:

      Mapper的依赖和“级联”行为大幅修改; 依赖逻辑从properties.py移动到名为“dependency.py”的单独的模块中。“级联” 行为现在可以显式控制，合理实现“delete”、“delete-orphan”等。依赖系统现在可以在刷新时确定子对象是否具有父对象，因此它会更好地决定如何更新该子对象与删除相关性数据库。

    .. 更改::
        :tags:
        :tickets:

      Schema在 MetaData对象之上进行了重构，而不是程序之前需要Engine。整个SQL/Schema系统甚至可以在没有引擎的情况下使用，仅通过显式的Connection对象执行。 “bound”方法存在于BoundMetaData以用于schema对象。 ProxyEngine现在基本上不再需要，用DynamicMetaData替换。

    .. 更改::
        :tags:
        :tickets: 167

      真正的多态行为采用有效的方式进行实现。

    .. 更改::
        :tags:
        :tickets: 147“oid”系统已完全移入编译时行为；如果它们在order_by中被使用，但无法使用，则order_by不会被编译，修复了该问题。

包装已进行全面修改；“mapping”现在是“orm”，“objectstore”现在是“session”，如果使用旧的“objectstore”命名空间，则通过“threadlocal” mod载入旧的“objectstore”命名空间。

mods现在通过“import <modname>”进行调用。扩展优先于mods，因为mods会对全局进行monkeypatching。

修复add_property以使其将属性传播到继承映射器。

backrefs自动生成其原始属性的主映射器，可以指定主/辅助联接参数以覆盖。帮助它们与多态映射器一起使用。

实现了“表存在”函数。

在MetaData对象中添加了“create_all / drop_all”。

拓扑排序算法进行了改进和修复，以及更多的单元测试。

文档添加了教程页面，也可以使用自定义doctest运行器运行该页面以确保其正常工作。文档通常进行了改进以处理新的代码模式。

许多其他的修复，重构等。

迁移指南可在Wiki上找到，网址为https://www.sqlalchemy.org/trac/wiki/02Migration。