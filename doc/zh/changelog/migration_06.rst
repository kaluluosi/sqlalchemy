=============================
SQLAlchemy 0.6 有哪些新变化?
=============================

.. admonition:: 本文档简介

    本文档记录了自2010年1月16日发布的SQLAlchemy 0.5版本和2012年5月5日发布的SQLAlchemy 0.6版本之间的更改。

    文档日期：2010年6月6日

本指南记录了影响用户从0.5系列的SQLAlchemy迁移应用程序的API更改。请注意，SQLAlchemy 0.6删除了一些在0.5系列期间已弃用的行为，并且还弃用了更多的0.5特定行为。

平台支持
================

* cPython版本从2.4到2.xx系列。

* Jython 2.5.1 - 使用Jython附带的zxJDBC DBAPI。

* cPython 3.x - 请参阅[source: sqlalchemy / trunk / README.py3k]以获取有关如何为python3构建的信息。

新编译器体系结构
==================

现在，方言模块被拆分成作为单个数据库后端范围内独特的子组件，方言实现现在在“ sqlalchemy.dialects”包中。 “ sqlalchemy.databases”包仍然存在，以提供对简单导入的某些级别的向后兼容性。

对于每个受支持的数据库，都存在一个子包存在于“ sqlalchemy.dialects”内，其中包含几个文件。每个包都包含一个名为“ base.py”的模块，它定义了该数据库使用的特定SQL方言。它还包含一个或多个“ driver”模块，每个模块对应于DBAPI中的特定模块，这些文件命名为DBAPI本身，例如“ pysqlite”，“ cx_oracle”或“ pyodbc”。SQLAlchemy方言所使用的类首先在“ base.py”模块中声明，定义了数据库定义的所有行为特征。这些包括能力映射，例如“支持序列”，“支持返回”等，类型定义以及SQL编译规则。每个“ driver”模块依次提供这些类的子类，override默认行为以适应该DBAPI的其他特点，行为和怪癖。对于支持多个后端的DBAPI（pyodbc，zxJDBC，mxODBC），方言模块将使用“ sqlalchemy.connectors”包中的mixin，这些mixin提供对该DBAPI在所有后端之间通用的功能，最常见的是处理连接参数。这意味着使用pyodbc，zxJDBC或mxODBC（实现时）连接非常一致。

“ create_engine（）”使用的URL格式已经改进，以处理特定后端的任意数量的DBAPI，使用受JDBC启发的方案。以前的格式仍然有效，将选择“默认”的DBAPI实现，例如下面使用psycopg2的PostgreSQL URL：

::

    create_engine("postgresql://scott:tiger@localhost/test")

但是，要指定特定的DBAPI后端，例如pg8000，请使用URL的“ protocol”部分将其添加到URL中，使用加号“+”：

::

    create_engine("postgresql+pg8000://scott:tiger@localhost/test")

重要的方言链接：

* 连接参数的文档：https://www.sqlalchemy.org/docs/06/dbengine.html#create-engine-url-arguments.

* 单独方言的参考文档：https://www.sqlalchemy.org/docs/06/reference/dialects/index.html

* 在DatabaseNotes上的贴士和技巧。

其他注意事项：

* 0.6中的类型系统发生了巨大变化。这对于命名惯例，行为和实现等方面的所有方言都有影响。请参见下面有关“类型”的部分。

* 现在，“ ResultProxy”对象在某些情况下提供了2倍速度提高，得益于一些重构。

* RowProxy，即单个结果行对象，现在可以直接进行挑选并腌制。

* 用于定位外部方言的setuptools entrypoint现在称为“ sqlalchemy.dialects”。针对0.4或0.5编写的外部方言将需要在任何情况下进行修改，以适用于0.6，因此此更改不会增加任何其他困难。

* 现在在初始连接时方言会接收initialize（）事件以确定连接属性。

* 由编译器生成的函数和运算符现在使用（几乎）常规分发函数的形式“ visit_ <opname>”和“ visit_ <funcname> _fn”来提供客户化处理。这取代了使用编译器子类中的“ functions”和“ operators”字典的需要，用直接的访问者方法代替，并且还允许编译器子类完全控制渲染，因为传递了完整的_Function或 _BinaryExpression对象。

方言导入
---------

方言的导入结构已经改变。每个方言现在都将其基本的“方言”类导出为以及该方言支持的所有SQL类型集合，通过“ sqlalchemy.dialects。”导入。例如，要导入一组PG类型：

::

    from sqlalchemy.dialects.postgresql import (
        INTEGER,
        BIGINT,
        SMALLINT,
        VARCHAR,
        MACADDR,
        DATE,
        BYTEA,
    )

上面，“ INTEGER”实际上是来自“ sqlalchemy.types”的普通“ INTEGER”类型，但PG作为那些特定于PG的类型，例如“ BYTEA”和“ MACADDR”以相同方式提供其类型。

表达式语言更改
=======================================

一个重要的表达式语言陷阱
---------------------------------------

表达式语言中有一个非常重要的行为更改可能会影响某些应用程序。即，Python布尔表达式的布尔值，例如“ ==”，“！”等，现在根据两个要比较的从句对象准确计算。

我们知道，将“ClauseElement”与其他对象进行比较返回另一个“ClauseElement”：

::

    >>> from sqlalchemy.sql import column
    >>> column("foo") == 5
    <sqlalchemy.sql.expression._BinaryExpression object at 0x1252490>

这样，当将Python表达式转换为字符串时，Python表达式会产生SQL表达式：

::

    >>> str(column("foo") == 5)
    'foo = :foo_1'

但是，如果我们说这句话会发生什么？

::

    >>> if column("foo") == 5:
    ...     print("yes")

在SQLAlchemy的以前版本中，返回的“_BinaryExpression”是一个普通的Python对象，它计算为“True”。现在它评估是否应该将实际的“ClauseElement”具有与将进行比较的哈希值相同的哈希值。这意味着：

::

    >>> bool(column("foo") == 5)
    False
    >>> bool(column("foo") == column("foo"))
    False
    >>> c = column("foo")
    >>> bool(c == c)
    True
    >>>

这意味着像以下这样的代码：

::

    if expression:
        print("the expression is:", expression)

将不会评估如果“expression”是二元子句。由于上面的模式永远不应该使用，因此基本的“ClauseElement”现在会在布尔上下文中引发异常：

::

    >>> bool(c)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      ...
        raise TypeError("Boolean value of this clause is not defined")
    TypeError: Boolean value of this clause is not defined

要检查“ClauseElement”表达式的存在，代码应该说：

::

    if expression is not None:
        print("the expression is:", expression)

请记住，**这也适用于Table和Column对象**。

更改表现方式
---------------------------------------- 

"executemany"的严格行为
-------------------------------

在SQLAlchemy中，"executemany"对应于对“execute()”的调用，同时传递一组绑定参数集合：

::

    connection.execute(table.insert(), {"data": "row1"}, {"data": "row2"}, {"data": "row3"})

当``Connection``对象对给定的``insert()``操作进行编译时，它将传递第一个绑定到该语句的VALUES子句中的键名集合，以确定选择器的构建。熟悉此构造的用户将知道，剩余字典中存在的其他键没有任何影响。现在的不同之处在于，所有后续的字典都需要包含第一个字典中存在的所有键。这意味着这个方法不再起作用了：

::

    connection.execute(
        table.insert(),
        {"timestamp": today, "data": "row1"},
        {"timestamp": today, "data": "row2"},
        {"data": "row3"},
    )

因为第三行没有指定“timestamp”列。在SQLAlchemy的以前版本中，这意味着这些缺少的列插入NULL。然而，如果上面的示例中的“timestamp”列包含Python侧默认值或函数，则不会使用。这是因为“executemany”操作针对处理大量参数集的最大性能进行了优化，并且不会尝试对这些缺少的键进行Python侧默认值的评估。因为默认值通常是嵌入到INSERT语句中的SQL表达式，或者是服务器端表达式，这些表达式由定义不可能根据每个参数集进行有条件地触发的插入字符串的结构定义（即使定义了SQL表达式为基于每个参数集的条件，每个参数集触发都是不可能的），所以Python侧的默认值与SQL和服务器侧的默认值的行为不一致。 （自0.5系列以来，SQL表达式默认使用内嵌的，目的是最小化巨大数量的参数集的影响）。

因此，SQLAlchemy 0.6通过禁止后续的参数集留空任何字段来建立可预测的一致性。这样一来，就不再有Python端默认值和函数的隐式失败了，这些默认值和函数此外仍然保持与SQL和服务器端默认值一致。

统一联合和其他“复合”构造输入一致
---------------------------------------------------------------

已删除旨在帮助SQLite的规则，即另一个复合（例如，“except_（）”内的“union（）”）内的第一个复合元素不会被括号括起来。这是不一致的，并且在PostgreSQL上产生了预优先级规则，通常是一个惊喜。现在，在SQLite中使用复杂的组合需要将第一个元素转换为子查询（这也与PG兼容）。SQL表达式教程中有一个新示例，在https://www.sqlalchemy.org/docs/06/sqlexpression.html#unions-and-other-set-operations的末尾。请参见：ticket：'1665'和r6690以了解更多背景信息。

结果获取的C扩展
================================

结果直接获取的``ResultProxy``以及相关元素，包括大多数常见的“行处理”函数（例如Unicode转换，数字/布尔转换和日期解析），现已重新实现为可选的C扩展，以实现更高的性能 目的。 这代表了SQLAlchemy通往“黑暗面”的开端，我们希望继续通过在C中重新实现关键部分来提高性能。扩展可以通过指定“--with-cextensions”，即“python setup.py --with-cextensions install”来构建。

无论何时，无论创建了引擎，池还是映射器，只要牺牲一些额外的方法调用，就可以设置INFO和DEBUG的日志级别，并启动日志记录。现在每个“Connection”都调用“isEnabledFor（INFO）”方法，“ResultProxy”如果它已经在父级连接上启用，则调用“isEnabledFor（DEBUG）”。池日志记录会发送到“log.info（）”和“log.debug（）”，没有检查 - 请注意，池checkout / checkin通常每个事务执行一次。

不管是否构建扩展程序，0.6与0.5相比性能已经提高。以下是使用SQLite，使用大部分直接SQLite访问，`ResultProxy``以及一个简单的映射ORM对象连接到和获取50,000行的连接情况：

.. sourcecode:: text

    sqlite select/native: 0.260s

    0.6 / C extension

    sqlalchemy.sql select: 0.360s
    sqlalchemy.orm fetch: 2.500s

    0.6 / Pure Python

    sqlalchemy.sql select: 0.600s
    sqlalchemy.orm fetch: 3.000s

    0.5 / Pure Python

    sqlalchemy.sql select: 0.790s
    sqlalchemy.orm fetch: 4.030s

上面，ORM相比0.5版本以33％更快的速度获取行，原因在于Python性能改进。使用C扩展，我们获得了另外20％。与C扩展相比，“ResultProxy”获取的提升提高了67％。其他测试报告表明，在某些情况下，诸如大量字符串转换正在发生的情况下，速度提高了多达200％。

新模式功能
=======================

“sqlalchemy.schema”包已经得到了长期需要的关注。最明显的变化是新扩展的DDL系统。在SQLAlchemy中，自0.5版本以来，可以创建自定义DDL字符串并将其与表或元数据对象关联：

::

    from sqlalchemy.schema import DDL

    DDL("CREATE TRIGGER users_trigger ...").execute_at("after-create", metadata)

现在，DDL构造的完整套件都可以在相同的系统中使用，包括用于CREATE TABLE，ADD CONSTRAINT等的DDL构造。

::

    from sqlalchemy.schema import Constraint，AddConstraint

    AddContraint(CheckConstraint("value > 5")).execute_at("after-create", mytable)
    
    此外，现在所有DDL对象都是就像任何其他SQLAlchemy表达式对象一样的常规“ClauseElement”对象：
    
    ::
    
    from sqlalchemy.schema import CreateTable

    create = CreateTable(mytable)

    # dumps the CREATE TABLE as a string
    print(create)

    # executes the CREATE TABLE statement
    engine.execute(create)

并且使用“ sqlalchemy.ext.compiler”扩展，您可以自己制作：

::

    from sqlalchemy.schema import DDLElement
    from sqlalchemy.ext.compiler import compiles


    class AlterColumn(DDLElement):
        def __init__(self, column, cmd):
            self.column = column
            self.cmd = cmd


    @compiles(AlterColumn)
    def visit_alter_column(element, compiler, **kw):
        return "ALTER TABLE %s ALTER COLUMN %s %s ..." % (
            element.column.table.name,
            element.column.name,
            element.cmd,
        )


    engine.execute(AlterColumn(table.c.mycolumn, "SET DEFAULT 'test'"))

弃用/删除的模式元素
----------------------------------

模式包也得到了大量简化。在整个0.5期间已弃用的许多选项和方法已被删除。其他鲜为人知的访问器和方法也已被删除。

* 从``Table``中删除了“owner”关键字参数。使用“schema”来表示要预先置于表名前面的任何名称空间。

* 已删除已弃用的``MetaData.connect（）``和``ThreadLocalMetaData.connect（）`` - 发送“bind”属性以绑定元数据。

* 已删除已弃用的metadata.table_iterator（）方法（使用sorted_tables）

* 从“DefaultGenerator”和其子类中删除了“metadata”参数，但仍在“Sequence”上本地存在，后者是DDL中的独立构造。

* 废弃“PassiveDefault” - 使用“DefaultClause”。

* 从“Index”和“Constraint”对象中删除了公共可变性：

  * ``ForeignKeyConstraint.append_element()``

  * ``Index.append_column()``

  * ``UniqueConstraint.append_column()``

  * ``PrimaryKeyConstraint.add()``

  * ``PrimaryKeyConstraint.remove()``

  这些应在声明式地构造（即在一个构造内）。

  其他被删除的内容包括：

  * ``Table.key``（不知道这是用来做什么的）

  * ``Column.bind``（通过column.table.bind获取）

  * ``Column.metadata``（通过column.table.metadata获取）

  * ``Column.sequence``（使用column.default）

其他行为变化
------------------------

* “UniqueConstraint”，“Index”，“PrimaryKeyConstraint”现在都接受列名或列对象的列表作为参数。

* ``ForeignKey``上的``use_alter``标志现在是一种快捷选项，用于手工构造使用``DDL()``事件系统。这次重构的一个副作用是，使用“use_alter = True”的``ForeignKeyConstraint``对象现在不会在SQLite上发出，因为SQLite不支持外键的ALTER。这对SQLite的行为没有影响，因为SQLite实际上不支持FOREIGN KEY约束。

* ``Table.primary_key``不可分配 - 使用``table.append_constraint（PrimaryKeyConstraint（...））``

* 带有``ForeignKey``但没有类型的``Column``定义，例如 ``Column(name, ForeignKey(sometable.c.somecol))`` 以前会获得引用列的类型。现在自动类型推断的支持是部分的，并且可能在所有情况下都无法工作。

开发日志
=================

牺牲了一些额外的方法调用，可以在引擎、池或映射器创建后设置INFO和DEBUG的日志级别，并开始记录日志。现在，“Connection”每个都调用“isEnabledFor（INFO）”方法，“ResultProxy”如果已经在父级连接上启用，则调用“isEnabledFor（DEBUG）”。池日志记录会发送到“log.info（）”和“log.debug（）”，没有检查-请注意，池 checkout / checkin通常每个事务执行一次。

反射/检查器API
========================

反射系统，允许通过``Table（'sometable'，metadata，autoload = True）``反射表列，现在已经开放到自己的细粒度API中，允许直接检查表元素，例如 表格，列，约束，索引等。该API将返回值表达为简单的字符串列表，字典以及“ TypeEngine”对象。``autoload = True``的内部现在建立在此系统之上，因此将原始数据库信息转换为“ sqlalchemy.schema”构造的集中化，以及各种方言的合同，大大减少了跨不同后端的各种错误和不一致性。

要使用检查器：

::

    from sqlalchemy.engine.reflection import Inspector

    insp = Inspector.from_engine(my_engine)

    print(insp.get_schema_names())

在某些情况下，“from_engine（）”方法将提供具有其他功能的特定于后端的检查器，例如PostgreSQL提供“get_table_oid（）”方法：

::


    my_engine = create_engine("postgresql://...")
    pg_insp = Inspector.from_engine(my_engine)

    print(pg_insp.get_table_oid("my_table"))

返回支持
=================

“ insert（）”，“ update（）”和“ delete（）”构造现在支持“ returning（）”方法，它对应于SQL RETURNING子句，如PostgreSQL，Oracle，MS-SQL和Firebird支持。此时没有为任何其他后端提供支持。

以与“select()”构造的列表达式相同的方式，给定标签的这个字符串类型将限制可以给那些标签的可能值。默认情况下，此类型会生成使用最大标签的“ VARCHAR”，并在CREATE TABLE语句中应用CHECK约束。当使用MySQL时，该类型默认情况下使用MySQL的ENUM类型，当使用PostgreSQL时，该类型将使用“ CREATE TYPE <mytype> AS ENUM”生成用户定义的类型（仅需在用于构造函数中指定“ name”参数构造函数时才能在PostgreSQL中创建该类型）。在没有显式指定``returning()``调用的情况下，SQLAlchemy还会自动使用返回，以获得单行INSERT语句的新生成主键值。这意味着对于需要主键值的插入语句，不再需要“SELECT nextval（sequence）”预执行。实话实说，隐式返回的OVERHEAD比旧的“select nextval（）”系统更多的方法调用开销，后者使用了快速且脏的cursor.execute（）来获取序列值，并且在Oracle的情况下需要其他关闭out的绑定参数是重要的，它们被重新路由到“ mock”结果集中，而在MS-SQL的情况下则使用笨拙的SQL语法。如果方法/协议开销比额外的数据库往返次数更昂贵，则该功能生成的默认扩展程序程序可以通过向``create_engine（）``指定``implicit_returning = False``来禁用。

类型系统更改
===================

新建筑
----------------

类型系统在幕后完全重建，以实现两个目标：

*将绑定参数和结果行值（通常是DBAPI要求）的处理与该类型本身的SQL规范（数据库要求）分开。这与分离数据库SQL行为和DBAPI的整体方言重构一致。

*建立清晰且一致的合同，以从``TypeEngine``对象生成DDL并基于列反射构造``TypeEngine``对象。

这些变化的亮点包括：

*方言内部构造类型的方式已完全改进。方言现在仅将公开可用类型定义为大写名称，并使用下划线标识符（即私有）定义内部实现类型。将类型表达为SQL和DDL的系统已经移动到编译器系统中。这样一来，在大多数方言中，类型对象明显减少了。有关为方言作者编写的此体系结构的详细文档，请参阅[source：/ lib / sqlalchemy / dialects / type_migration_guidelines.txt]。

*类型反映现在返回types.py中的确切大写类型，或者是该类型不是标准SQL类型的情况下Dialect本身内的大写类型。这意味着反射现在返回有关反射类型的更准确信息。

*子类化“TypeEngine”的用户定义类型（例如，``UserDefinedType``）现在应提供``get_col_spec（）``。

*所有类型类上的“result_processor（）”方法现在接受一个额外参数“coltype”。这是连接数.description附加的DBAPI类型对象，应在适用的情况下使用它，以更好地决定应该返回什么类型的结果处理可调用函数。理想情况下，结果处理函数不需要使用“isinstance（）”，在这个层次上，这是一个昂贵的调用。

本机Unicode模式
-------------------

随着越来越多的DBAPI支持直接返回Python Unicode对象，基本方言现在在第一次连接时执行检查，以确定DBAPI是否为基本选择器返回Python Unicode对象VARCHAR值。如果是，'String'类型及其所有子类（即'Text'，'Unicode'等）在接收到结果行时将跳过“unicode”检查/转换步骤。这为大型结果集提供了显着的性能提高。目前已知的“本地Unicode”模式可以与以下工作：

* sqlite3 / pysqlite

* psycopg2 - SQLA 0.6现在每个psycopg2连接对象默认使用“ UNICODE”类型扩展名

* pg8000

* cx_oracle（我们使用输出处理器-很好的功能！）

其他类型可以根据需要禁用Unicode处理，例如在与MS-SQL一起使用时使用“ NVARCHAR”类型。

特别是，如果转移基于先前返回非Unicode字符串的DBAPI的应用程序，则“原生Unicode”模式有与默认行为显然不同的默认行为，即已声明为“String”或“VARCHAR”的列现在默认情况下返回Unicode而不是以前返回的字符串。这可能会破坏希望字符串不是Unicode的代码。可以通过传递``use_native_unicode = False``到``create_engine（）``来禁用psycopg2的“本地Unicode”模式。

针对明确不想要Unicode对象的字符串列的更一般的解决方案是使用``TypeDecorator``，将Unicode转换回utf-8或任何所需格式：

::

    class UTF8Encoded(TypeDecorator):
        """Unicode type which coerces to utf-8."""

        impl = sa.VARCHAR

        def process_result_value(self, value, dialect):
            if isinstance(value, unicode):
                value = value.encode("utf-8")
            return value

注意，``assert_unicode``标志现在已弃用。SQLAlchemy允许DBAPI和正在使用的后端数据库在可用时处理Unicode参数，并且不通过检查传入类型添加操作需求；现代系统如sqlite和PostgreSQL将在其端上通过引发编码错误来提高错误，在那些情况下，SQLAlchemy确实需要将Python Unicode转换为编码字符串，或者显式使用Unicode类型时会发出警告如果对象是bytestring，则发出警告。可以使用在https://docs.python.org/library/warnings.html上记录的Python警告过滤器抑制此警告或将其转换为异常

通用枚举类型
-----------------

我们现在在“ types”模块中有一个“ Enum”。这是一个给定“标签”的字符串类型，它将其可能值的范围限制为这些标签。默认情况下，此类型将生成使用最大标签的“ VARCHAR”，并在CREATE TABLE语句中应用CHECK约束。当使用MySQL时，默认情况下，类型使用MySQL的ENUM类型，使用PostgreSQL时，该类型将使用“ CREATE TYPE <mytype> AS ENUM”生成用户定义的类型（仅需在用于构造函数中指定“ name”参数的构造函数中时才能在PostgreSQL中创建该类型）。该类型还接受一个列表， 提供标签给定字符串值。

`````native_enum=False`` 选项将为所有数据库发出VARCHAR/CHECK策略。请注意，PostgreSQL ENUM类型目前无法使用pg8000或zxjdbc。

反射返回方言特定类型
-------------------------

反射现在从数据库返回最具体的类型。也就是说，如果您使用``String``创建一个表，然后将其反射回来，反射的列很可能是``VARCHAR``。对于支持类型更具体形式的方言，那就是您将获得的内容。因此，如果Oracle上使用``Text``类型，则会返回``oracle.CLOB``，``LargeBinary``可能是``mysql.MEDIUMBLOB``等。这里的明显优点在于，反射尽可能地保留了数据库的所有信息。

一些处理表元数据的应用程序可能希望比较反射表和/或非反射表中的类型。``TypeEngine``上有一个半私有的访问器称为``_type_affinity``和一个相关的比较助手``_compare_type_affinity``。此访问器返回类型对应的“常规”``types``类：

::

    >>> String(50)._compare_type_affinity(postgresql.VARCHAR(50))
    True
    >>> Integer()._compare_type_affinity(mysql.REAL)
    False

杂项API更改
-----------------------------------------

通常使用的“常规”类型仍然是通用系统，即``String``，``Float``，``DateTime``。有些变化：

* 类型不再猜测默认参数。特别是``Numeric``，``Float``以及子类NUMERIC，FLOAT，DECIMAL除外，在未指定的情况下不生成任何长度或比例。这也继续包括有争议的``String``和``VARCHAR``类型（尽管MySQL方言将在请求不带长度的VARCHAR时先发出警告）。不假设默认值，并且如果它们在CREATE TABLE语句中使用，则在底层数据库不允许这些类型的无长度版本时将引发错误。

* ``Binary``类型已重命名为``LargeBinary``，用于BLOB/BYTEA/类似类型。对于``BINARY``和``VARBINARY``，这些直接作为``types.BINARY``，``types.VARBINARY``以及在MySQL和MS-SQL方言中出现。

* 如果未指定“comparator”参数，则``PickleType``现在使用==来比较值，即mutable=True。如果您正在pickle定制对象，则应实现一个``__eq__()``方法，以便基于值的比较准确无误。

* Numeric和Float的默认“精度”和“比例”参数已被删除，现在默认为None。NUMERIC和FLOAT将默认不带数值参数呈现，除非提供了这些值。

* SQLite上的DATE，TIME和DATETIME类型现在可以接受可选的“storage_format”和“regexp”参数。可使用“storage_format”使用自定义字符串格式存储这些类型。“regexp”允许使用自定义正则表达式来匹配来自数据库的字符串值。

* SQLite上的Time和DateTime类型上的``__legacy_microseconds__``不再受支持。您应该使用“storage_format”参数。

* SQLite中的DateTime类型现在默认使用更严格的正则表达式来匹配来自数据库的字符串。如果正在使用存储在旧格式中的数据，请使用新的“regexp”参数。

ORM更改
===========

从0.5升级至0.6的ORM应用程序应该几乎不需要更改，因为ORM行为保持几乎相同。有一些默认参数和名称更改，并且已经改进了一些加载行为。

新的工作单元
----------------

单元检查过程的内部，主要是topological.py和unitofwork.py，已完全重写并大大简化。这对于使用应该没有影响，因为所有现有的行为在刷新期间都已完全维护了（或者至少是我们的测试套件和少数测试了它大量的生产环境。）。刷新的性能现在使用20-30%较少的方法调用，并且还应该使用更少的内存。源代码的意图和流程现在应该相当容易理解，并且刷新的架构目前相当开放，为潜在的新领域提供了空间。刷新过程不再依赖于递归，因此可以刷新具有任意大小和复杂度的刷新计划。此外，映射器的“保存”进程（发出INSERT和UPDATE语句）现在缓存了两个语句的“编译”形式，因此随着非常大的刷新，调用计数进一步显着减少。

刷新与0.6或0.5早期版本的行为差异，应立即向我们报告-我们将确保没有功能丢失。

对``query.update()``和``query.delete()``的更改
----------------------------------------------------

* query.update（）上的“expire”选项已更名为“fetch”，因此与query.delete（）的名称相匹配

*对于同步策略的query.update（）和query.delete（），“评估”是默认设置。为同步策略指定值“评估”。


*update()和delete()的“synchronize”策略在失败时引发错误。没有隐含的回退到“获取”。评估的失败基于标准的结构，因此基于代码结构是确定的成功/失败。

``relation()``正式命名为``relationship()``
-----------------------------------------------------

为了解决长期存在的问题，在关系代数术语中，“关系”表示“表或派生表”的问题。然后大家开始称呼关系为relationship。因此，``relation()``名称（输入较少）将在可预见的将来继续存在，因此这种更改应该完全无痛无痛。

子查询贪婪加载
----------------------

添加了一个称为“子查询”加载的新类型的贪婪加载。这是发出第二个SQL查询的加载，该查询立即在第一个加载项中加载所有父项的完整集合，使用INNER JOIN向上连接到父项。类似于当前的加入贪婪加载，“``subqueryload()``和“``subqueryload_all()``”选项以及在“``relationship()``”上设置的“``lazy ='subquery'”设置都是使用“。子查询负载通常比使用INNER JOIN无条件地使用INNER JOIN，而且还不会重新加载父行的更多集合更有效。  

``eagerload()``和``eagerload_all()``现在是``joinedload()``和``joinedload_all()``
------------------------------------------------------------------------------------------------

为了为新的子查询加载特性让出空间，现有的`eagerload() /`eagerload_all()选项现在被``joinedload()``和``joinedload_all()``取代。与旧名称一样，新名称将继续存在，就像``relation()``一样。

``lazy = False | None | True | 'dynamic'````现在接受````lazy ='noload'|'joined' |'subquery'|'select'|'dynamic'```
-------------------------------------------------------------------------------------------------------------

继续沿着打开的加载器策略主题，``relationship()``上的标准关键字现在是``select``用于懒惰加载（通过在访问属性时发出SELECT发出），``joined``用于连接的贪婪加载，“subquery”用于子查询贪婪式加载，“noload”表示不应加载任何加载，而“dynamic”表示动态关系。以前的``True''， "``False``和'None'参数仍然被接受，行为与以前完全相同。

关系内联根据关系，joinedload
--------------------------------------

可以指示许多对一个NOT NULLable外键的关系使用INNER JOIN而不是OUTER JOIN。在PostgreSQL上，这被观察到在某些查询上提供了300-600％的加速。对于任何保证相关项存在的多对一，都设置此标志。关系上的``innerjoin = True``标志还将对一个不覆盖该值的任何``joinedload()``选项生效``relationship()``级别。 

多对一增强
------------------------

*许多对一关系现在在较少的情况下发出懒加载，包括在大多数情况下不会在替换新值时获取“旧”值。

*简单加载（称为“use_get”条件）现在使用get（）进行子类多对一关系查询表的many-to-one relation(即' Related' - >“Sub（Base）”），而无需在基地址表中重新定义primaryjoin条件即可确保父表之间的外键关系。 [ 票务：1186]

*为使用具有声明性列（即``ForeignKey（MyRelatedClass.id）``）指定外键不会破坏“使用get”的条件（即无法进行多对一关系）[ ticket:1492]

* relationship()，joinedload()和joinedloa_all()现在具有名为'innerjoin'的选项。指定True或False以控制是否将贪婪连接构建为INNER或OUTER连接。默认值仍然是False。映射器选项将覆盖relationship()上指定的任何设置。通常应设置为多对一，非空外键关系以允许改进的连接性能。 [ ticket:1544]

*与0.5不同，加载多个父级时使用子查询贪婪加载，现在主查询在存在LIMIT/OFFSET时被包装在子查询中的行为现在不再适用于父级全部为多对一联接的情况。在这些情况下，即使存在贪婪加载，父表的贪婪连接也直接连接到而行号/偏移量，而不带有对额外开销的子查询的附加开销，因为多对一连接不会向结果添加行。

例如，在0.5中，此查询： 

  session.query(Address).options(eagerload(Address.user)).limit(10)

  会产生SQL，例如： 

  SELECT * FROM
    (SELECT * FROM addresses LIMIT 10）AS anon_1
    LEFT OUTER JOIN users AS users_1 ON users_1.id = anon_1.addresses_user_id
  
  这是因为任何贪婪加载的存在都表明其中一些或全部可能与多行集相关，这将需要在子查询中包装任何类型的行数敏感修饰符，例如LIMIT。 

  在0.6中，该逻辑更加敏感，可以检测到所有贪婪加载是否代表多对一，如果是，则贪婪连接不会影响行数：

  SELECT * FROM addresses LEFT OUTER JOIN users AS users_1 ON users_1.id = addresses.user_id LIMIT 10

具有联合表继承的可变主键
--------------------------------------------------

在子表具有外键指向父表主键的联合表继承配置现在可以在支持联级的数据库（例如PostgreSQL）上进行更新。``mapper()``现在具有一个``passive_updates = True``选项，表示此外键将自动更新。如果在不支持级联的数据库（例如SQLite或MySQL / MyISAM）上，则将此标志设置为``False``。未来的功能增强将尝试根据使用的方言/表样式自动配置此标志。

Beaker缓存
--------------

Beaker集成的一个有前途的新示例是在“examples / beaker_caching”中。这是一个直接应用Beaker高速缓存到``Query``的结果生成引擎的简单配方。缓存参数通过“query.options()”提供，并允许完全控制缓存的内容。 SQLAlchemy 0.6通过改进``Session.merge（）``方法来支持此类配方以及类似的配方，并提供了更高效的性能，以大多数情况下提高性能。

其他变更
-------------

* 当选择多个列/实体时，``Query``返回的“行元组”对象现在可以进行插值以及更高的性能。
  
* ``query.join()``已重新设计以提供更一致的行为和更高的灵活性（包括[ticket:1537]）
  
* ``query.select_from()``接受多个子句以生成多个逗号分隔的条目。在从多区间join()子句中进行选择时非常有用。
  
* Session上的'transactional'标志以及其他标志已删除。使用'autocommit = True'表示'transactional = False'。
  
* mapper()上的“polymorphic_fetch”参数已删除。可以使用“with_polymorphic”选项来控制加载。
  
* mapper()上的“select_table”参数已删除。使用“with_polymorphic =（“*”，<some selectable>）”以获取此功能。
  
* synonym()上的'proxy'参数已删除。整个0.5期间此标志都是没有意义的，因为“代理生成”行为现在是自动的。
  
* 在joinedload（），joinedload_all（），lazyload（），defer（），undefer（）中仅传递一个元素列表代替多个位置\ * args现在已弃用。
  
* 在query.order_by（），query.group_by（），query.join（）或query.outerjoin()上构建单个元素列表，而不是多个位置的\ * args现已弃用. 
  
* ``query.iterator_instances()``已删除。使用``query.instances()``.

* "dont_load=True"标志对``Session.merge（）``不再支持。

* Session上的'engine'参数已删除。使用'bind'关键字参数。

扩展程序
==========

SQLSoup
-------

SQLSoup已现代化并更新，以反映常见的0.5/0.6功能，包括明确定义的会话集成。请阅读新文档，网址为[https://www.sqlalchemy.org/docs/06/reference/ext/sqlsoup.html]。

Declarative
-----------

``DeclarativeMeta''（``declarative_base``的默认元类）以前允许子类修改``dict_''以添加类属性（例如，列）。这不再使用，``DeclarativeMeta``构造函数现在忽略``dict_''。相反，类属性应直接赋值，例如``cls.id = Column（...）''，或者应使用`MixIn类<https://www.sqlalchemy.org/docs/reference/ext/declarative.html#mix-in-classes>''方法而不是元类方法。