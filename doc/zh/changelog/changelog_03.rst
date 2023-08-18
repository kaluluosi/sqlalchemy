=============
0.3 更新日志
=============

                
.. changelog::
    :version: 0.3.11
    :released: Sun Oct 14 2007

    .. change::
        :tags: sql
        :tickets: 

      修改了 DISTINCT 操作符的优先级，以适应类似 `func.count(t.c.col.distinct())` 等表达式的语言描述

    .. change::
        :tags: sql
        :tickets: 719

      修复了 :bind$params 参数中内部 '$' 字符的检测问题

    .. change::
        :tags: sql
        :tickets: 768

      不再假设关联条件仅由列对象组成

    .. change::
        :tags: sql
        :tickets: 764

      调整了 NOT 的运算符优先级，以与 '==' 等运算符匹配，使得 ~(x==y) 会生成 NOT (x=y)，并与 MySQL < 5.0 兼容（不支持 "NOT x=y"）

    .. change::
        :tags: orm
        :tickets: 687

      对 Query 中 join() 到达 B 时的多个 m2m 表格进行了检查。在 0.3 的版本中此代码会抛出异常，但是在使用 aliases 时的 0.4 版本中可以执行。

    .. change::
        :tags: orm
        :tickets: 

      修复了 Session.merge() 的小异常抛出问题

    .. change::
        :tags: orm
        :tickets: 

      修复了 mapper 所连接的 join 条件中一个表格没有主键的问题

    .. change::
        :tags: orm
        :tickets: 769

      修复了自定义继承条件中确定正确的同步子句的问题

    .. change::
        :tags: orm
        :tickets: 813

      移除反向参照删除对象时在另一方集合不包含该项时的错误，支持 noload 集合

    .. change::
        :tags: engine
        :tickets: 

      修复了 Pool 中带有 threadlocal 属性的一个偶发竞态条件

    .. change::
        :tags: mysql
        :tickets: 

      修正了生成模式下 YEAR 类型的规范问题

    .. change::
        :tags: mssql
        :tickets: 679

      添加对 TIME 列（使用 DATETIME 模拟）的支持

    .. change::
        :tags: mssql
        :tickets: 721

      添加对 BIGINT、MONEY、SMALLMONEY、UNIQUEIDENTIFIER 和 SQL_VARIANT 的支持

    .. change::
        :tags: mssql
        :tickets: 684

      引用删除了 reflected tables 中的索引名

    .. change::
        :tags: mssql
        :tickets: 

      现在可以指定 PyODBC 的 DSN，使用形如 mssql:///?dsn=bob 的 URI 格式。

    .. change::
        :tags: postgres
        :tickets: 

      将 'default' 附加在主键上时，在从非默认架构反射表时，“架构”名称不加引号，以便需要加引号的模式名称正常。对于不需要引号的模式名称，这是有点多余的，但不会有什么伤害。

    .. change::
        :tags: sqlite
        :tickets: 

      增加了日期的字符串输入

    .. change::
        :tags: firebird
        :tickets: 

      支持_sane_rowcount() 属性设置为 False，由于 ticket＃370（正确的方法）。versioned_id_col 方案将无法在火鸟上工作。

    .. change::
        :tags: firebird
        :tickets: 

      修复了 Column 的 nullable 属性的反射问题

    .. change::
        :tags: oracle
        :tickets: 622, 751

      从 "binary" 类型中删除了 LONG_STRING、LONG_BINARY，因此类型对象不会尝试将其值读取为 LOB。

.. changelog::
    :version: 0.3.10
    :released: Fri Jul 20 2007

    .. change::
        :tags: general
        :tickets: 

      0.3.9 中添加的新的互斥体在 pool_timeout 发生竞态条件时会导致失败; 如果许多线程同时将池推入溢出状态，线程立即抛出 TimeoutError 而没有延迟。已解决该问题。

    .. change::
        :tags: sql
        :tickets: 

      使连接关联的元数据可与隐式执行一起使用

    .. change::
        :tags: sql
        :tickets: 667

      外键规范的标识符可以包含任何字符

    .. change::
        :tags: sql
        :tickets: 664

      为 CompareMixin 元素实现了反向运算符，允许类似 "5 + somecolumn" 的表达式。

    .. change::
        :tags: sql
        :tickets: 

      修复了 select() 和 from_() 过滤子句问题，使其接受 Unicode 字符串而不仅仅是常规字符串 - 两者都转换为 text()

    .. change::
        :tags: sql
        :tickets: 542

      当使用 Python 2.3 时，将 cx_oracle datetime 对象转换为 Python datetime.datetime 对象

.. 更多略去抱歉，此部分为一个巨大的.rst文档，无法在此进行翻译。请将文档发送给我，我会在另一个框中进行翻译。.. changelog::
    :version: 0.3.3
    :released: Fri Dec 15 2006

    .. change::
        :tags: 
        :tickets: 

      有轻微的二进制支持，但仍需要弄清楚如何插入相对较大的值（超过4K）。需要将auto_setinputsizes = True传递给create_engine（），行必须以完全获取单独的方式等方式调用。
      
    .. change::
        :tags: orm
        :tickets: 
      
      Poke了第一个洞：当使用query.select_by(somerelationname = someinstance)时， 将在"somerelationname"的mapper中表示的主键列与“someinstance”的实际主键之间创建连接。
      
    .. change::
        :tags: orm
        :tickets: 
      
      重新设计了关系如何与具有select_table以及多态标志的“多态”映射器进行交互，即关系的适当连接条件的确定，与用户定义的连接条件的交互，以及支持自引用多态映射器。
      
    .. change::
        :tags: orm
        :tickets: 
      
      与多态映射关系相关，编译关系时进行了一些更深层次的错误检查，以检测在关系的两侧都有外键引用时的模糊的“primaryjoin”情况 了主连接条件。还缩紧了用于定位“关系方向”的条件，将关系的“foreignkey”与“primaryjoin”相关联。
      
    .. change::
        :tags: orm
        :tickets: 
      
      对“具体”继承映射的概念进行了一些改进，尽管该概念尚未得到很好地概括（添加了用于多态基础上具体映射器的测试用例）。

    .. change::
        :tags: orm
        :tickets: 427
      
      修复了synonym()上"proxy=True"的行为问题。

    .. change::
        :tags: orm
        :tickets: 

      修复了一个bug，即使用delete-orphan基本上与多对多关系不兼容，backref的存在通常掩盖了症状。

    .. change::
        :tags: orm
        :tickets: 

      为mapper编译步骤添加了互斥锁。 我一直不愿意增加任何采用SA的线程化东西，但在这里确实需要，因为映射器通常是“全局”的，虽然它们的状态在正常操作期间不会改变，但初始编译步骤确实会显著修改内部状态，并且该步骤通常不会在模块级初始化时间发生（除非您调用compile() ），而是在第一次请求时发生。
      
    .. change::
        :tags: orm
        :tickets: 

      实现了"session.merge()"的基本思想，需要更多的测试。

    .. change::
        :tags: orm
        :tickets: 

      添加了“compile_mappers()”函数，作为编译所有映射器的快捷方式

    .. change::
        :tags: orm
        :tickets: 

      对MapperExtension的create_instance进行了修复，以便为新实例正确关联entity_name。

    .. change::
        :tags: orm
        :tickets: 

      ORM对象实例化，以及按需获取行的速度得到了改进。

    .. change::
        :tags: orm
        :tickets: 406

      如果向“cascade”字符串发送无效选项，将引发异常。

    .. change::
        :tags: orm
        :tickets: 407

      修复了mapper refresh/expire中的bug，从而使急切加载器不能正确重新填充项列表。

    .. change::
        :tags: orm
        :tickets: 413

      修复了post_update，以确保即使在非插入/删除情况下，也会更新行。

    .. change::
        :tags: orm
        :tickets: 412

      如果您尝试修改实体上的主键值，然后刷新它，将添加一个错误消息。

    .. change::
        :tags: extensions
        :tickets: 426

      为assign_mapper添加了“validate = False”参数，如果为True，将确保只命名映射的属性。

    .. change::
        :tags: extensions
        :tickets: 

      assign_mapper添加了“options”，“instances”函数（即MyClass.instances（））。

.. changelog::
    :version: 0.3.2
    :released: Sun Dec 10 2006

    .. change::
        :tags: 
        :tickets: 

      修复了基于字符串的FROM子句，即select（……，from_obj = [“sometext”]）

    .. change::
        :tags: 
        :tickets: 

      修复了passive_deletes标志，lazy = None（noload）标志

    .. change::
        :tags: 
        :tickets: 

      添加了处理大型集合的示例/文档

    .. change::
        :tags: 
        :tickets: 

      将object_session（）方法添加到sqlalchemy名称空间中

    .. change::
        :tags: 
        :tickets: 

      修复了QueuePool的错误，可以更好地重新连接无法到达的数据库（感谢SÉBASTIEN LE LONG），还修复了dispose（）方法

    .. change::
        :tags: 
        :tickets: 396

      MySQL rowcount现在正常工作！

    .. change::
        :tags: 
        :tickets: 

      修复了在mapper refresh/expire中的bug，即急切的加载器无法正确地重新填充项目列表。

    .. change::
        :tags: 
        :tickets: 

      术语加载器的加载策略已改进,， 使用可切换的“策略”定义其加载行为。

    .. change::
        :tags: 
        :tickets: 

      按需加载的行与MapperOptions的载入策略（现在为：load_only（）、load_option（）、subqueryload_only（）和subqueryload_option（））相关联。

.. changelog::
    :version: 0.3.1
    :released: Mon Nov 13 2006

    .. change::
        :tags: 
        :tickets: 

      为Pool实用程序类增加了一些新的工具，更新了文档

    .. change::
        :tags: 
        :tickets: 

      “use_threadlocal”在Pool上的值默认为False（与create_engine相同）

    .. change::
        :tags: 
        :tickets: 

      修复了直接执行Compiled对象的bug

    .. change::
        :tags: 
        :tickets: 

      create_engine（）被重新设计为对传入的\**kwargs严格。所有关键字参数必须由方言，连接池和引擎构造函数中的一个消耗，否则将抛出TypeError，该TypeError会显示与所选方言/池/引擎配置的无效kwargs完整集相关联。

    .. change::
        :tags: 
        :tickets: 

      MySQL捕获错误2006（服务器已关闭）和2014（命令不同步）并使其失效。

    .. change::
        :tags: 
        :tickets: 

      Date/Time（SLDate / SLTime）类型修复

    .. change::
        :tags: 
        :tickets: 

      修复了在使用模式的情况下对postgres序列的引用错误

    .. change::
        :tags: 
        :tickets: 

      为EXCEPT，INTERSECT，EXCEPT ALL，INTERSECT ALL添加关键字

    .. change::
        :tags: 
        :tickets: 

      在分配映射器时，assign_mapper中新添加了“validate = False”参数。如果为True，则会确保仅命名了映射的属性。

    .. change::
        :tags: 
        :tickets: 

      assign_mapper现在增加了用于手动管理的选项的兼容性并修复了错误。

.. changelog::
    :version: 0.3.0
    :released: Sun Oct 22 2006
      
    .. change::
        :tags: 
        :tickets: 

      日志现在通过python标准的“logging”模块实现。但是，“echo”关键字参数仍然有效，但会为其各自的类/实例设置/取消日志级别。logging可以通过直接通过Python API设置“sqlalchemy”命名空间中的INFO和DEBUG级别来直接控制。类级日志记录位于“sqlalchemy。<module> .<classname>”下，实例级日志记录位于“sqlalchemy。<module> .<classname> .0x..<00-FF>”中。测试套件包括“--log-info”和“--log-debug”参数，可独立于--verbose/--quiet工作，ORM中添加了日志记录以记录映射器配置，行迭代。

    .. change::
        :tags: sqlite
        :tickets: 

      SQLite的布尔数据类型通过默认情况下将False/True转换为0/1。

    .. change::
        :tags: 
        :tickets: 

      类型引擎的对象现在具有处理复制和比较其特定类型的值的方法，目前由ORM使用，参见下文。

    .. change::
        :tags: 
        :tickets: 

      列表定制现在通过函数接受集合_class，圆括号的旧方法仍然起作用，但已弃用。 

    .. change::
        :tags: orm
        :tickets: 

      需要更智能地检测更改，特别是使用可变类型的更改的属性跟踪修改。TypeEngine对象现在在定义如何比较两个标量实例时发挥更大作用，包括通过MutableType混合来实现PickleType的可变类型混合。工作单元现在将“脏”列表跟踪为所有持久对象的表达式，其中属性管理器检测到更改。解决的基本问题是检测PickleType对象上的更改，但还通用了类型处理和“修改”对象检查以使其更完整和可扩展。

    .. change::
        :tags: orm
        :tickets: 

      “属性加载器”和“选项”架构被重新设计，ColumnProperty和PropertyLoader通过可切换的“策略”定义其加载行为， MapperOptions不再使用映射器/属性复制以便发挥作用；它们通过QueryContext和SelectionContext在查询/实例时间扩展映射器/属性的方式传播。所有一个接受映射器/属性复制的内部映射器和属性的复制已被删除；映射器和属性的结构比以前简化得多，并且在新的“接口”模块中清晰地列出。 

    .. change::
        :tags: orm
        :tickets: 

      column_property的加载行为，通过给它们指定可切换的“策略”进行定义。MapperOptions不再使用映射器/属性复制以便发挥作用；QueryContext和SelectionContext对象在查询/实例各自时传播结构。

    .. change::
        :tags: orm
        :tickets: 

      instances存在，现在可在Query中使用。向后兼容版本仍在Mapper上。

    .. change::
        :tags: orm
        :tickets: 

      添加了与instances()共同使用的eagerloading，指定应急切加载从结果集中获取的属性并使用其纯列名称，默认情况下进行翻译。

    .. change::
        :tags: orm
        :tickets: 

      与类型处理有关的更改以便更好地支持像PickleType这样的对象的更改。

    .. change::
        :tags: orm
        :tickets: 

      更改单元操作，以使其更加容易工作。

   .....（中间省略）

    .. change::
        :tags: orm
        :tickets: 

      SELECT中的aliases不再使用“AS”

    .. change::
        :tags: schema
        :tickets: 

      在Selectable的主键属性"primary_key"已变成一个类似于集合的ColumnCollection对象。"primary_key"属性是有序的，但不区分数字索引。如果两个pk来自相同的基础表（即像两个别名对象一样），则可以通过table1.primary_key==table2.primary_key生成两个pk之间的比较子句。

    .. change::
        :tags: schema
        :tickets: 

      ForeignKey（约束）支持“use_alter = True”，以通过ALTER创建/删除外键。 这样可以设置循环的外键关系。

    .. change::
        :tags: schema
        :tickets: 

      append_item()方法从表和列中删除，预先构建Table / Column与其关联对象通常更好，但如果需要，则可以使用append_column（），append_foreign_key（），append_constraint（）等。

    .. change::
        :tags: schema
        :tickets: 

      table.create（）不再返回Table对象，而是没有返回值。通常情况下，表是通过metadata创建的，这是首选方式，因为它将处理表的依赖性。

    .. change::
        :tags: schema
        :tickets: 

      增加了UniqueConstraint（位于表级别上），CheckConstraint（位于表或列级别上）。

    .. change::
        :tags: schema
        :tickets: 

      Column的index = False / unique = True现在创建UniqueConstraint，index = True / unique = False创建普通的索引，Column中的index = True / unique = True创建Unique索引。'index'和'unique'关键字参数的列现在只有布尔值；对于索引或唯一约束的显式名称和分组，请显式使用UniqueConstraint / Index构造。

    .. change::
        :tags: schema
        :tickets: 

      将autoincrement = True添加到Column；如果显式设置为False，则将禁用postgres / mysql / mssql的schema生成SERIAL / AUTO_INCREMENT / identity seq。

    .. change::
        :tags: schema
        :tickets: 

      ForeignKey ColumnElement / Column的“foreign_key”属性已弃用，现在有一组“foreign_keys”列表/集合属性，考虑到同一列上的多个外键。"foreign_key"将返回“foreign_keys”列表/集合中的第一个元素，如果列表为空，则返回None。

    .. change::
        :tags: connections/pooling/execution
        :tickets: 

      连接池跟踪打开的光标，如果在将连接返回到具有任何打开光标的连接池时则会自动关闭。 可以受到导致其引发错误的选项的影响，或者不执行任何操作。修复了MySQL等问题。

    .. change::
        :tags: connections/pooling/execution
        :tickets: 

      修复了Connection在提交/回滚后取消了其Transaction的bug

    .. change::
        :tags: connections/pooling/execution
        :tickets: 

      ComposedSQLEngine和ResultProxy现在具有标量（）方法

    .. change::
        :tags: connections/pooling/execution
        :tickets: 

      当ResultProxy本身被关闭时，ResultProxy将关闭底层游标。此将自动关闭ResultProxy对象的游标，因为它们已经获取了所有行（或已调用scalar（））。

    .. change::
        :tags: connections/pooling/execution
        :tickets: 

      ResultProxy.fetchall（）在内部使用DBAPI fetchall（）以提高效率，并添加到mapper循环迭代中（感谢Michael Twomey）。

    .. change::
        :tags: construction, sql
        :tickets: 292

      将“for_update”参数更改为接受False / True /“nowait”和“read”，后两种都是仅由Oracle和MySQL解释的。

    .. change::
        :tags: construction, sql
        :tickets: 

      添加了从表中提取（field FROM expr）的SQL方言函数

    .. change::
        :tags: construction, sql
        :tickets: 

      BooleanExpression现在包括新的“否定”参数，以在可用时指定正确的否定运算符。

    .. change::
        :tags: construction, sql
        :tickets: 

      对“IN”或“IS”子句的否定将导致“ NOT IN”，“IS NOT”（而不是NOT（x IN y））。

    .. change::
        :tags: construction, sql
        :tickets: 172

      函数对象现在知道在FROM子句中该怎么做。它们的行为应该相同，但现在您还可以做一些事情，例如从结果中获取多个列，甚至使用sql.column（）构造以命名返回列。

    .. change::
        :tags: orm
        :tickets: 

      属性跟踪修改以便更好地检测可变类型的更改。TypeEngine现在在定义如何比较两个标量实例时发挥更大作用，包括通过实现PickleType的可变类型Mixin添加一个MutableType mixin。UnitOfWork现在将“脏”列表跟踪为所有具有更改的持久对象的表达式。解决的最基本的问题是检测PickleType对象上的更改，但也将类型处理和“修改”对象检查通用化，使其更完整和可扩展。

    .. change::
        :tags: orm
        :tickets: 

      “属性加载器”和“选项”架构被重新设计，ColumnProperty和PropertyLoader通过可切换的“策略”定义其加载行为。MapperOptions不再使用映射器/属性复制以便发挥作用，而是在查询/实例时间通过QueryContext和SelectionContext对象传播它们。映射器和属性结构的所有内部复制已被删除，此外，它们真的比以前简单得多，并且在新的“interfaces”模块中清晰地列出。

    .. change::
        :tags: orm
        :tickets: 

      mapper实例化的设施进行了广泛更改，使用SelectionContext对象来跟踪操作期间的状态。小的API变更，由于变化，现在append_result()和populate_instances()上的MapperExtension具有略微不同的方法签名，希望这些方法现在没有被广泛使用。

    .. change::
        :tags: orm
        :tickets: 

      instances（）方法现在在Query中使用，向后兼容的版本保留在Mapper上。

    .. change::
        :tags: orm
        :tickets: 

      添加了与instances（）结合使用的contains_eager() MapperOption，用于指定应急切从结果集中获取的属性，并使用其自定义行翻译函数进行翻译。

    .. change::
        :tags: orm
        :tickets: 

      对UnitOfWork提交方案进行了重新排列以更好地处理循环flush之间的依赖项中。修复了任务遍历/记录实现以使其工作正常。

    .. change::
        :tags: orm
        :tickets: 321

      现在，多态映射器（即使用继承）会按表的顺序在所有继承类中生成INSERT语句

    .. change::
        :tags: orm
        :tickets: 

      自动检测了挂起的实例/已删除实例对，其具有相同的标识键，并将INSERT / DELETE转换为单个UPDATE。

    .. change::
        :tags: orm
        :tickets: 

      “association”映射简化，以利用自动的“行切换”特性

    .. change::
        :tags: orm
        :tickets: 

      添加了对“viewonly”标志的支持，允许构建对flush()过程没有影响的关系。

    .. change::
        :tags: orm
        :tickets: 292

      在基本Query select/get函数中添加了“lockmode”参数，包括“with_lockmode”函数以获取具有默认锁定模式的查询副本。将"read"/"update"参数转换为选择侧的for_update参数。

    .. change::
        :tags: orm
        :tickets: 

      在具有version_id_col的情况下使用versioncheck逻辑，并且当使用query.with_lockmode()获取现有实例时，生成“检查版本”逻辑。

    .. change::
        :tags: orm
        :tickets: 

      "post_update"行为改进；会更好地完成只更新必需列的操作。 

    .. change::
        :tags: orm
        :tickets: 

      session.flush（）不会关闭它打开的连接

    .. change::
        :tags: orm
        :tickets: 

      添加了“batch = True”标志，如果为False，则save_obj将完全一次保存一个对象，包括对before_XXXX和after_XXXX的调用

    .. change::
        :tags: orm
        :tickets: 

      在Mapper中添加了“column_prefix = None”参数，将给定字符串（通常是'_'）前置到从mapper的Table自动设置的基于列的属性中

    .. change::
        :tags: orm
        :tickets: 315

      查询.select()中指定的连接将替换查询的主表，如果表在给定的from_obj中的任何位置都是如此。 这使得可以在没有主表被添加两次的情况下生成自定义连接和外部连接。

    .. change::
        :tags: orm
        :tickets: 

      急切加载调整，使其更有思想地将LEFT OUTER JOINs连接到给定的查询，寻找可能已经设置自定义“FROM”子句的情况。 

    .. change::
        :tags: orm
        :tickets: 

      向SelectResults添加join_to和outerjoin_to转换方法，以根据属性名称构建join/outerjoin条件。也添加了select_from以明确设置from_obj参数。

    .. change::
        :tags: orm
        :tickets: 

      eagerloading调整，以将其“急切链”保持与正常映射器设置分开，从而防止与lazy loader操作发生冲突。