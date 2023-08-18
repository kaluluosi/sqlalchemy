=============================
SQLAlchemy 0.7有什么新特性？
=============================

.. admonition:: 关于本文档

    本文档描述了SQLAlchemy版本0.6和0.7之间的变化，0.7最近一次维护发布是在2012年10月。

    文档日期：2011年7月27日

介绍
====

本指南介绍SQLAlchemy 0.7的新特性，并记录影响用户从0.6转移到0.7系列的应用程序的更改。

尽可能地使更改按照不破坏兼容性的方式进行。必须不兼容的更改非常少，除了一种，即可变属性默认值的更改，应该会影响极少部分应用程序-许多更改涉及非公共API和一些用户可能一直尝试使用的未记录的黑客。

第二个，更小的非向后兼容性变化类别也有所记录。这个类别的更改涉及自0.5版本起就已被弃用并且已经发出警告的功能和行为。这些更改只会影响仍在使用0.4或早期0.5样式API的应用程序。随着项目的成熟，我们的API具有越来越少的这些类型的功能，这是我们API具有的少数不太理想的功能，而这些功能并不适用于它们旨在解决的用例。

SQLAlchemy 0.7中有一系列现有功能被替换。“supereded”和“deprecated”的术语没有太大的区别，只是前者更弱地表明旧特性将会被删除。在0.7中，例如“同义词”和“comparable_property”，以及所有“Extension”和其他事件类，都已被取代。但是，这些“不再建议使用”的功能已经被重新实现，使其实现大部分在ORM核心代码之外，因此它们的持续“挂起”不会影响SQLAlchemy进一步简化和完善其内部的能力，并且我们预计它们将在可预见的未来的API中保持不变。

新特性
======

新事件系统
----------

SQLAlchemy从``MapperExtension``类开始，为映射器的持久化周期提供了钩子。随着SQLAlchemy的不断组件化，越来越多的“扩展”，“监听器”和“代理”类出现，以以各种ad-hoc方式解决各种活动拦截用例。其中的一部分是由活动的分歧带来的； ``ConnectionProxy``对象希望提供一个重写语句和参数的系统； ``AttributeExtension``提供了一种替换传入值的系统，并且``DDL``对象具有可以关闭特定方言调用的事件。

0.7现在使用新的，统一的方法来实现几乎所有的插件点，该方法保留不同系统的所有功能，提供了更多的灵活性和更少的样板文件，执行效果更好，并且消除了为每个事件子系统学习根本不同的API的需要。预先存在的类``MapperExtension``，``SessionExtension``，``AttributeExtension``，``ConnectionProxy``，``PoolListener``以及``DDLElement.execute_at``方法已被弃用并且现在是基于新系统的实现-这些API仍然功能齐全，并且预计会保留在该位置可预见的未来。

新方法使用具名事件和用户定义的可调用项将活动与事件关联起来。 API的外观和感觉由多种来源驱动，例如JQuery，Blinker和Hibernate，并且在多次与Twitter上的许多用户进行会议期间进行了进一步修改，该应用程序似乎具有更高的响应率，更高于问题的邮件列表。

它还具有一种开放式的目标规范系统，该系统允许将事件与API类（例如所有“Session”或“Engine”对象）相关联，以特定API类的特定实例（例如某个“Pool”或“Mapper”），以及与之相关的对象，例如一个用户定义类，它被映射，或者像特定映射父类的特定子类的实例上的某个属性一样特定。单个侦听器子系统可以应用包装器到传入的用户定义侦听器函数，这些包装器修改它们如何被调用-映射器事件可以接收要操作的对象的实例或其底层``InstanceState``对象。属性事件可以选择是否有返回新值的责任。

现在有多个系统基于新事件API构建，包括新的“可变属性”API和组合属性。更大的事件重视还导致引入了一些新事件，包括属性过期和刷新操作，pickle 加载/卸载操作，完成的映射器构造操作。

.. seealso::

  :ref:`event_toplevel`

:ticket:`1902`

混合属性，实现/替换同义词(), comparable_property()
---------------------------------------------------------

“派生属性”示例现在已成为官方扩展程序。 `` synonym（）``的典型用例是提供对映射列的描述符访问； `` comparable_property（）``的用例是能够从任何描述符返回``PropComparator`` 。实践中，“衍生”方法更容易使用，更易于扩展，使用几十行纯Python几乎没有导入，而且不需要ORM核心甚至意识到它。该功能现在被称为“混合属性”扩展程序。

`` synonym（）``和`` comparable_property（）``仍然是ORM的一部分，尽管它们的实现已经移出，并建立在类似于混合扩展的方法上，因此除此之外，核心ORM_
mapper/query/property模块并不真正知道它们。

.. seealso::

  :ref:`hybrids_toplevel`

:ticket:`1903`

速度增强
----------

按照所有主要的SQLA版本，对内部进行了广泛的遍历，以减少常见情况下的开销和调用次数。此版本的亮点包括：

* 现在，刷新进程将INSERT语句捆绑到批量中，用于主键已经存在的行，而无需调用``cursor.execute（）``。特别是这通常适用于连接表继承配置的“child”表，这意味着用于加入表对象的大量批量插入的``cursor.execute``调用数量可以减半，允许本地DBAPI优化针对那些传递给``cursor.executemany（）``的语句运作（例如重用预准备语句）。

* 以前已加载的``relationship()``引用的许多对一引用的访问者执行路径现在已经大大简化。直接检查身份映射而不需要先生成新的``Query``对象，这在许多内存中访问许多对一数据时是代价高昂的。此外，也不再使用每次调用构造的“loader”对象大多数的懒惰属性加载。混合体重新编写使在刷新期间mapper内部访问映射属性的代码路径更短。旧的内联属性访问函数替换了以前在“save-update”和其他级联操作需要在全部数据成员中级联的情况下使用的“history”。这减少了为此速度关键操作生成新的``History``对象的开销。

* ExecutionContext的内部，即对应于语句执行的对象，已被内联化和简化。

* 每次语句执行时由类型生成的``bind_processor（）``和``result_processor（）``可调用项现在已经被缓存（小心使用，以避免临时类型和方言的内存泄漏），对于该类型的寿命进一步减少每个语句的调用开销。

* 特定Compiled实例的“bind processor”的传递的收藏也缓存到了“Compiled”对象上，利用刷新过程使用的“编译缓存”，以重新使用相同的编译INSERT，UPDATE，DELETE语句。

呼吁减少调用次数的演示，包括示例基准脚本在
https://techspot.zzzeek.org/2010/12/12/a-tale-of-three-profiles/

组合重写
---------

“composite”特性已被重新编写，就像“synonym（）”和“comparable_property()”一样，使用基于描述符和事件的轻量级实现，而不是构建到ORM内部中。这允许从mapper/unit of work内部删除一些延迟并简化组合的工作方式。复合属性现在不再隐藏它们构建属性上的基础列，而现在将其作为常规属性保留。复合材料还可以充当“relationship()”以及“Column()”属性的代理。

组合体的主要不向后兼容的更改是它们不再使用“mutable = True”系统来检测原地突变。请使用“变异跟踪<https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html>`_”扩展来为现有复合材料使用建立就地变更事件。

.. seealso::

  :ref:`mapper_composite`

  :ref:`mutable_toplevel`

:ticket:`2008` ：ticket:`2024`

更简洁的查询.join(target，onclause)格式
------------------------------------------------

现在，将带有显式onclause的目标作为``query.join（）``的默认方法是：

::

    query.join(SomeClass, SomeClass.id == ParentClass.some_id)

在0.6中，这种用法被认为是错误的，因为``join()``接受与多个JOIN子句相对应的多个参数-两个参数的形式需要以元组的形式出现，以消除单个参数和二元参数之间的歧义目标加入。在0.6中间，由于这种用法非常普遍，所以添加了检测和针对此特定调用风格的错误消息。在0.7中，由于我们正在检测确切的模式，而需要出于没有理由的目的键入元组则非常麻烦，并且现在非元组方法现在成为执行此操作的“正常”方式。复杂的连接次数比单个连接案例要少得多，而且多个连接现在更能明确地表示为多个``join()``调用。

元组形式将保持向后兼容性。

请注意，所有其他形式的“query.join（）”仍然保持不变：

::

    query.join(MyClass.somerelation)
    query.join("somerelation")
    query.join(MyTarget)
    # ... etc

`使用连接进行查询
<https://www.sqlalchemy.org/docs/07/orm/tutorial.html#querying-with-joins>`_

:ticket:`1923`

.. _07_migration_mutation_extension:

突变事件扩展，取代“mutable = True”
---------------------------------------------

新扩展程序：ref:`mutable_toplevel`，提供了一种机制，通过该机制，用户定义的数据类型可以将更改事件提供给其所拥有的一个或多个父级。扩展包括标量数据库值的方法，例如由:class:`.PickleType`管理的值，`postgresql.ARRAY`或其他自定义``MutableType``类，以及ORM“组合体”，使用:func:`~.sqlalchemy.orm.composite`进行配置。

.. seealso::

    :ref:`mutable_toplevel`

NULLS FIRST / NULLS LAST操作符
------------------------------------------

这是``asc（）``和``desc（）``操作符的扩展，称为``nullsfirst（）``和``nullslast（）``。

.. seealso::

    :func:`.nullsfirst`

    :func:`.nullslast`

:ticket:`723`

select.distinct()，query.distinct()对于PostgreSQL DISTINCT ON现在支持\*args
----------------------------------------------------------------------------------------

现在，通过传递表达式的列表到``select（）` `， ``select()``的``distinct()``方法以及``Query``的位置参数现在将呈现为PostgreSQL后端使用DISTINCT ON时呈现的DISTINCT ON。

`distinct() <https://www.sqlalchemy.org/docs/07/core/expressi
on_api.html#sqlalchemy.sql.expression.Select.distinct>`_

`Query.distinct() <https://www.sqlalchemy.org/docs/07/orm/que
ry.html#sqlalchemy.orm.query.Query.distinct>`_

:ticket:`1069`

``Index()``可以内联放置在``Table``，``__table_args__``内
-------------------------------------------------------------------

可以使用字符串作为列名称，将Index（）构造内联到表定义中，作为在表之外创建索引的替代方法。也就是说：

::

    Table(
        "mytable",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50), nullable=False),
        Index("idx_name", "name"),
    )

这里的主要原因是为了声明性``__table_args__``的好处，尤其是与mixin一起使用时：

::

    class HasNameMixin(object):
        name = Column("name", String(50), nullable=False)

        @declared_attr
        def __table_args__(cls):
            return (Index("name"), {})


    class User(HasNameMixin, Base):
        __tablename__ = "user"
        id = Column("id", Integer, primary_key=True)

`Indexes <https://www.sqlalchemy.org/docs/07/core/schema.html
#indexes>`_

窗口函数SQL构造
------------------------

“窗口函数”为语句提供了关于生成的结果集的信息。这允许针对诸如“行号”，“秩”等各种条件进行标准化。它们已知至少由PostgreSQL，SQL Server和Oracle支持。

最好的窗口函数介绍在PostgreSQL的网站上，自从8.4版本开始，窗口函数就受到了支持：

https://www.postgresql.org/docs/current/static/tutorial-window.html

SQLAlchemy提供了一个简单的构造，通常通过现有的函数子句调用，使用``over（）``方法，它接受``order_by``和``partition_by``关键字参数。下面我们复制PG教程中的第一个示例：

::

    from sqlalchemy.sql import table, column, select, func

    empsalary = table("empsalary", column("depname"), column("empno"), column("salary"))

    s = select(
        [
            empsalary,
            func.avg(empsalary.c.salary)
            .over(partition_by=empsalary.c.depname)
            .label("avg"),
        ]
    )

    print(s)

SQL:

.. sourcecode:: sql

    SELECT empsalary.depname, empsalary.empno, empsalary.salary,
    avg(empsalary.salary) OVER (PARTITION BY empsalary.depname) AS avg
    FROM empsalary

`sqlalchemy.sql.expression.over <https://www.sqlalchemy.org/d
ocs/07/core/expression_api.html#sqlalchemy.sql.expression.ov
er>`_

:ticket:`1844`

Connection中的execution_options（）接受“isolation_level”参数
-------------------------------------------------- -------

这将为单个“Connection”设置事务隔离级别，直到该“Connection”关闭并其底层DBAPI资源返回到连接池，此后隔离级别将被重置为默认值。使用``isolation_level``参数设置默认隔离级别``create_engine()``。

目前，事务隔离支持仅由PostgreSQL和SQLite后端支持。

`execution_options() <https://www.sqlalchemy.org/docs/07/core
/connections.html#sqlalchemy.engine.base.Connection.executio
n_options>`_

:ticket:`2001`

``TypeDecorator``可与整数主键列一起使用
-----------------------------------------------------------

扩展了``Integer``行为的``TypeDecorator``可以与主键列一起使用。 ``Column``的“自动递增”功能现在将识别底层数据库列仍然是整数，使得lastrowid机制继续正常工作。自动生成的主键也将应用其结果值处理器，包括DBAPI ``cursor.lastrowid``访问器所接收的值。

:ticket:`2005` ：ticket:`2006`

``TypeDecorator``在“sqlalchemy”导入空间中存在
-----------------------------------------------------------

不再需要从“sqlalchemy.types”导入此功能，它现在在“sqlalchemy”中进行镜像。

新方言
--------

方言已添加：

* 用于Drizzle数据库的MySQLdb驱动程序：

   `Drizzle <https://www.sqlalchemy.org/docs/07/dialects/drizz
  le.html>`_

* 支持pymysql DBAPI：

   `pymsql Notes
  <https://www.sqlalchemy.org/docs/07/dialects/mysql.html
  #module-sqlalchemy.dialects.mysql.pymysql>`_

* psycopg2现在可以与Python 3一起使用

行为变化（向后兼容）
============================

默认情况下，构建C扩展
-----------------------------

这是在0.7b4中。如果检测到cPython 2.xx，则会构建exts。如果构建失败，例如在Windows安装中，则会捕获该条件并继续执行非-C安装。如果使用Python 3或PyPy，则不会构建C exts。

查询计数（）简化，应该几乎总是起作用
--------------------------------------

非常老的猜测在``Query.count()``中已经现代化，使用``.from_self（）```。也就是说，``query.count()``现在等同于：

::

    query.from_self(func.count(literal_column("1"))).scalar()

以前，内部逻辑试图重写查询本身的列子句，在检测到“子查询”条件时（例如，列的查询可能具有聚合，或具有DISTINCT），会经过一个错综复杂的过程通过重写列子句。

这种逻辑在复杂情况，特别是涉及联接表继承时失败，因此在更全面的``.from_self（）` `调用之前已经过时。 通过这种方式的“孤儿”行为发生在将具有“delete-orphan”级联的``relationship()``关联对象新添加到INSERT中，而没有建立父关系。多年前，此检查是为了适应一些测试案例，这些测试案例测试孤儿的行为的一致性。在现代SQLA中，不再需要这种检查。该对象的父外键引用与数据库列的数据一致性相同，而SQLA则允许大多数其他操作完成工作。如果对象的父外键是可空的，那么可以插入行。当对象以特定父级持久化，然后与该父级解除联系时，将运行“孤儿”行为，导致删除语句发出。

:ticket:`1912`

查询中含有集合成员，标量引用不属于flush
-----------------------------------------------

在父对象标记为“脏”时通过加装的``relationship()``引用加载的相关对象没有出现在当前``Session``中时，现在会发出警告。

``save-update``级联在将对象添加到``Session``或首次将对象与父级相关联时生效，因此对象及其相关对象通常都存在同一个``Session``中。但是，如果禁用了特定``relationship()``的“save-update”级联，则不会发生此行为，并且刷新过程不会尝试纠正它，而是保持一致与配置的级联行为一致。以前，在刷新过程中检测到这些对象时，它们会被默默地跳过。新行为是发出警告，目的是警示导致意外行为的情况。

:ticket:`1973`

不再装置安装Nose插件
-----------------------------------------

自从我们使用鼻子以来，我们就使用了一个插件，通过setuptools安装该插件，以便``nosetests``脚本会自动运行SQLA的插件代码，从而使我们的测试具有完整的环境。在0.6中途，我们意识到这种导入模式意味着“coverage”将损坏，“coverage”要求在导入要覆盖的任何模块之前启动;因此，在0.6中间期间，我们为此特定情况添加了检测和错误消息，因为这是如此普遍。在0.7中，由于该模式确实检测到了准确的模式，并且由于为无缘无故地为整个引擎创建多个Nose配置选项会产生额外的标识字符串，因此不再尝试使“nosetests”自动工作。 SQLAlchemy模块将在所有``nosetests``的使用中产生大量nose配置选项，而不仅仅是SQLAlchemy单元测试本身，并且额外的``sqlalchemy-nose``安装是更糟的想法，因为Python环境中会产生额外的软件包。0.7中的``sqla_nose.py``脚本现在是使用nose进行测试的唯一方法。

:ticket:`1949`

非由“Table”派生的构造适用映射
------------------------------------------------

可以映射不针对任何“Table”的构造，例如函数。

::from sqlalchemy import select, func
from sqlalchemy.orm import mapper

class Subset(object):
    pass

selectable = select(["x", "y", "z"]).select_from(func.some_db_function()).alias()
mapper(Subset, selectable, primary_key=[selectable.c.x])

#ticket: 1876#

aliased()接受“FromClause”元素
---------------------------

如果向orm.aliased()构造中传递了一个普通的FromClause，例如select、Table或join，则实用程序将通过那个from构造的.继而让该from构造的.alias()方法而不是构造一个ORM级别的AliasedClass。

#ticket: 2018#

Session.connection(), Session.execute()接受'bind'
--------------------------------------------------

这是为了使execute/connection操作明确参与引擎的打开事务。它还允许自定义Session子类实现它们自己的get_bind()方法和参数，以同时使用这些自定义参数和execute()和connection()方法。

Session.connection
Session.execute


#ticket: 1996#

在列子句中使用独立的绑定参数会被自动标记。
-------------------------------------------------------

如果在select的“columns clause”中存在绑定参数，则如其他“匿名”clause一样，现在会自动标记它们，这使得它们的“类型”在提取行时是有意义的，例如在结果行处理器中。

#ticket: 2036#

SQLite - 相对文件路径通过 os.path.abspath() 进行规范化
-------------------------------------------------------------

这样，更改当前目录的脚本将继续针对后续建立的SQLite连接的相同位置。

#ticket: 1833#
 
在 MS-SQL 上，“String” /“Unicode” /“VARCHAR”/“NVARCHAR” /“VARBINARY”在不指定长度的情况下发出“max”
--------------------------------------------------------------------------------------------------------

在MS-SQL后端上，字符串/unicode类型及其对应的VARCHAR / NVARCHAR，以及VARBINARY(:ticket:1833)在未指定长度时发出“max”。这使其与未指定长度时同样无限制的PostgreSQL的VARCHAR类型相比更加兼容。SQL Server在未指定长度时将这些类型的默认长度设置为“1”。

行为更改（向后不兼容）
===========================

注意，除了默认的可变性更改之外，大多数更改都是*非常小*的，不会影响大多数用户。

PickleType和ARRAY可变性默认关闭
-----------------------------------

ORM将映射具有PickleType或postgresql.ARRAY数据类型的列时，将默认情况下将mutable标志设置为False。如果现有应用程序使用这些类型并依赖于就地更改的检测，则必须使用mutable=True构造类型对象以恢复0.6行为。

较早的使用“mutable=True”方法不提供更改事件，相反，ORM必须扫描会话中存在的所有可变值，并在每次调用“flush()”时将它们与其原始值进行比较，这是一个非常耗时的事件。这是SQLAlchemy早期版本的遗留问题，当时“flush()”不是自动的，并且历史记录跟踪系统还不像现在这样复杂。

使用PickleType，postgresql.ARRAY或其他MutableType子类的现有应用程序，并且需要就地更改的检测，应该迁移到新的变异跟踪系统，因为很可能在将来废弃“mutable=True”。

#ticket:1980#

“Composite（）”的可变性检测需要Mutation Tracking Extension
----------------------------------------------------------------------

被称为“composite”映射属性的属性，这些属性使用在“Composite Column Types”中描述的技术进行配置，已重新实现，以使ORM内部不再知道它们（从而在关键节省和更有效的代码路径）。尽管复合型通常被视为不可变的值对象，但从未实施过这一点。对于使用可变性的应用程序使用复合型，Mutation Tracking扩展为用户定义的复合类型提供了一种机制，以向每个对象的所有者或父级发送更改事件消息。

使用组合类型并依赖于这些对象的就地突变检测的应用程序应迁移到“变异跟踪”扩展，或更改组合类型的使用方式，以使就地更改不再需要（即将它们视为不可变的值对象）。

SQLite - 现在对于基于文件的数据库，SQLite方言使用“NullPool”
---------------------------------------------------------

此更改**在99.999％的情况下向后兼容**，除非您在连接池连接跨连接池连接使用临时表。

基于文件的SQLite连接速度非常快，使用“NullPool”意味着每次调用“Engine.connect”都会创建一个新的pysqlite连接。

以前使用的是“SingletonThreadPool”，这意味着在一个线程中对某个引擎的所有连接都将是相同的连接。新方法更直观，尤其是使用了多个连接时。

使用内存数据库时，默认引擎仍为“SingletonThreadPool”。

请注意，**这种更改会破坏Session提交之间使用临时表的情况**，因为SQLite处理临时表的方式。如果需要临时表超出一个池连接的范围，则请参阅https://www.sqlalchemy.org/docs/dialects/sqlite.html#using-temporary-tables-with-sqlite上的注释。

#ticket: 1921#

“Session.merge()”检查具有版本控制的映射器的版本id
------------------------------------------------------------

如果传入状态的版本ID与数据库的版本ID不匹配，则Session.merge()将检查入站状态的版本ID，并引发StaleDataError。这是正确的行为，因为如果传入状态包含陈旧的版本ID，则应该假定该状态已过期。

如果将数据合并到具有版本控制的状态中，则版本ID属性可以保留未定义，将不会进行版本检查。

Hibernate做的事情被证实了这个检查 - “merge（）”和版本化功能最初都是从Hibernate中适应的。

#ticket:2027#

Query改进中的元组标签名称
--------------------------------

这种改进对于依赖旧行为的应用程序可能有些向后不兼容。

给出两个映射类“Foo”和“Bar”，每个类都有一个列“spam”：

::

    qa = session.query(Foo.spam)
    qb = session.query(Bar.spam)

    qu = qa.union(qb)

对于由“qu”产生的单个列命名为“spam”。以前，由于“union”的方式结合了这些内容，这个名称将是“foo_spam”之类的内容，它与非union查询中的名称“spam”不一致。

#ticket: 1942#

映射列属性首先引用最具体的列
--------------------------------------------

这是行为的变化，涉及映射列属性引用多个列的情况，特别是在一个属性的情况下，该属性位于具有与超类相同的名称的属性上。

使用声明式，情况如下：

::

    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)

    class Child(Parent):
        __tablename__ = "child"
        id = Column(Integer, ForeignKey("parent.id"), primary_key=True)

上面，属性“Child.id”同时指代“child.id”列和“parent.id”列--这是由于属性的名称，在这种情况下是如此。如果在类上命名为不同的属性，例如“Child.child_id”，则它将明确地映射到“child.id”，其中“Child.id”与“Parent.id”相同。

当"id"属性用于引用“parent.id”和“child.id”时，它们将被存储在有序列表中。然后，如“Child.id”这样的表达式在呈现时仅引用这些列中的*一列*。在0.6以前，此列将是“parent.id”。在0.7中，它是更少令人惊讶的“child.id”。

这种行为的遗留问题与ORM的行为和限制有关，这些限制现在不再适用；所需的是翻转顺序。

这种方法的主要优点是现在更容易构造"primaryjoin"表达式，这些表达式引用本地列：

::

    class Child(Parent):
        __tablename__ = "child"
        id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
        some_related = relationship(
            "SomeRelated", primaryjoin="Child.id==SomeRelated.child_id"
        )

    class SomeRelated(Base):
        __tablename__ = "some_related"
        id = Column(Integer, primary_key=True)
        child_id = Column(Integer, ForeignKey("child.id"))

在0.7之前，"Child.id"表达式将引用"Parent.id"，并且很可能需要将child.id映射到一个不同的属性中。

这也意味着查询像这样的查询将更改其行为：

::

    session.query(Parent).filter(Child.id > 7)

在0.6中，这将会呈现：

.. sourcecode:: sql

    SELECT parent.id AS parent_id
    FROM parent
    WHERE parent.id > :id_1

在0.7中，您会得到：

.. sourcecode:: sql

    SELECT parent.id AS parent_id
    FROM parent, child
    WHERE child.id > :id_1

您会注意到这是一个笛卡尔积--这种行为现在等效于用于“Child”的任何其他本地属性。with_polymorphic()函数或类似的策略通过显式连接基础“Table”对象来渲染针对“Child”的所有“Parent”对象的查询，以与0.5和0.6的情况相同：

::

    print(s.query(Parent).with_polymorphic([Child]).filter(Child.id > 7))

它们在0.6和0.7上呈现：

.. sourcecode:: sql

    SELECT parent.id AS parent_id, child.id AS child_id
    FROM parent LEFT OUTER JOIN child ON parent.id = child.id
    WHERE child.id > :id_1



这种更改的另一个影响是跨两个以上同名列的连接绑定需要明确声明。

#ticket: 1875#

DDL()构造现在转义百分号
-----------------------

以前，DDL()字符串中的百分号必须被转义，即视DBAPI而定，对于那些接受“pyformat”或“format”绑定的DBAPI（即psycopg2，mysql-python等），这是不一致的，与text()构造不同，后者会自动执行此操作。

现在，DDL()和text()相同，例如：

::

    from sqlalchemy import DDL
    conn.execute(DDL("CREATE TRIGGER foo_ins ..."))

#ticket: 1870#

列出类型名称的types.type_map现在已不公开，types._type_map
-------------------------------------------------- 

我们注意到，一些用户在“sqlalchemy.types”中的字典中拦截，作为将Python类型与SQL类型关联的捷径。我们无法保证该字典的内容或格式，并且另外，将Python类型一对一地关联具有些许灰色地带，应由单个应用程序决定，因此我们对此属性进行了下划线。

#ticket: 1892#:ticket: 1917#

编译器的mappers()重命名为configure_mappers（），简化配置内部
---------------------------------------------------------

这个系统从最初是一些小型，实现本地到单个映射器的某些应用程序并且出于历史原因被命名不佳，演变成一个全球“注册表”-级别函数并且命名不佳。因此，我们将其实现从“Mapper”移出并将其命名为“configure_mappers()”来重新命名。正常情况下，应用程序无需调用“configure_mappers()”，因为此过程会基于属性或查询访问需要的情况在需要的情况下立即发生。

#ticket:1966#

Core侦听器/代理被事件侦听器取代
------------------------------------------

PoolListener，ConnectionProxy，DDLElement.execute_at被PoolEvents，EngineEvents，DDLEvents分发对象使用的“event.listen()”取代。

ORM扩展被事件侦听器取代
-----------------------------------

MapperExtension，AttributeExtension，SessionExtension被Event监听器替代，使用MapperEvents/InstanceEvents，AttributeEvents，SessionEvents分发对象。

在select()中向MySQL发送字符串到'distinct'应该通过前缀完成
-------------------------------------------------------------

这种隐晦的特性允许MySQL后端使用这样的模式：

::

    select([mytable], distinct="ALL", prefixes=["HIGH_PRIORITY"])

应该在需要使用非标准或不寻常前缀的情况下使用关键字参数“prefixes”或“prefix_with（）”方法：

::

    select([mytable]).prefix_with("HIGH_PRIORITY", "ALL")

#ticket: 1896#

映射到具有两个或多个同名列的联接需要显式声明
----------------------------------------------------

这与1792号（#ticket: 1792）的先前变更有关。在映射到联接时，必须显式链接具有相同名称的列到映射的属性中，即如`Mapping a Class Against Multiple Tables<http://www.sqlalchemy.org/docs/07/orm/mapper_config.html#mapping-a-class-against-multiple-tables>`_所述。

现有的代码应修改使使用元素名称明确返回，如下所示：

::


    foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
    mapper(FooBar, foobar, properties={"id": [foo.c.id, bar.c.id]})

应在“left join”，“outer join”等情况下运行，以将“properties”参数相应更改为所需的列列表。

#ticket: 1893#

映射器要求polymorphic_on列存在于预定选择中
----------------------------------------------------------------

这是在0.6中的警告，现在在0.7中是一个错误了。给定为polymorphic_on的列必须存在于映射的可选择内容中。防止偶尔存在一些用户错误，例如：

::

    mapper(SomeClass, sometable, polymorphic_on=some_lookup_table.c.id)

上面的polymorphic_on需要在“sometable”列上，例如也许是“sometable.c.some_lookup_id”（在这种情况下）。还有一些“多态联合”的情况，类似的错误有时会发生。

这种配置错误一直是“错的”，上面的映射不按照指定的那样工作-列将被忽略。然而，直到此更改，仍然可能会出现潜在的向后不兼容的情况。

#ticket: 1875#

现-func()构造不支持额外的参数
------------------------------------------------ -------------

核心types模块中的简单类型如Integer、Date等不接受参数。默认构造函数接受/忽略一个万能参数'args，\** kwargs'`的默认构造函数被恢复到`0.7b4/0.7.0，但发出了一个“过时”的警告。

如果在核心类型如“Integer”中使用参数，则可能是您打算使用特定于方言的类型，例如“sqlalchemy.dialects.mysql.INTEGER”，该类型接受“display_width”参数。

compile_mappers()改名为configure_mappers()，简化配置内部
------------------------------------------------------------

该系统从最初的实施在一个单独的映射器本地，名称不佳，并发展成一个全球“注册表”级别函数，名称不佳，因此我们通过将实现从“Mapper”移到“configure_mappers()”来修复这两个问题。使用属性或查询访问时，这在需要时立即发生，通常情况下无需应用程序调用“configure_mappers()”。

#ticket: 1966#

callables传递给bindparam()不会被求值-影响了 Beaker 示例
-----------------------------------------------------------------

#ticket:1950#

请注意，这影响了Beaker缓存示例，其中“_params_from_query()”函数的工作需要进行轻微调整。如果您使用Beaker示例中的代码，则应应用此更改。 

类型.type_map现在是私有的，types._type_map
-------------------------------------------------- 

我们注意到，一些用户正在利用types内部的诸如“sqlalchemy.types”中的字典，以将Python类型与SQL类型关联。我们不能保证该字典的内容或格式，另外，将Python类型一对一地关联具有一些灰色地带，最好由各个应用程序决定，因此我们已经标记了下划线。

#ticket: 1870#

独立的alias（）函数的别名关键字参数改为name
------------------------------------------------------

因此，显式关键字名称“alias”名称现在与所有“FromClause”对象上的alias()方法以及Query.subquery()上的“name”参数匹配。

应该只修改在传递alias名称时使用显式关键字名称“alias”（而不是位置）且没有使用方法绑定函数的代码。非公共的Pool方法现在有下划线。这样的方法有：

'Pool.create_connection（）'->'Pool._create_connection（）'

'Pool.do_get（）'->'Pool._do_get（）'

'Pool.do_return_conn（）'->'Pool._do_return_conn（）'

'Pool.do_return_invalid（）'->删除，未使用

“Pool.return_conn（）” ->“Pool._return_conn（）”

'Pool.get（）'-> 'Pool._get（）'，公共API是'Pool.connect（）'以前被弃用，现在被移除
============================

Query.join（），Query.outerjoin（），eagerload（），eagerload_all（）和其他方法不再允许将属性列表作为参数
----------------------------------------------------------------------------------------------------------------------

自从0.5版本以来，将属性列表或属性名称列表传递给“Query.join（）”，“eagerload（）”等方法已被弃用：

::

    # 自从0.5以来弃用
    session.query(Houses).join([Houses.rooms, Room.closets])
    session.query(Houses).options(eagerload_all([Houses.rooms, Room.closets]))

从0.5版本以后，这些方法都接受\ * args：

::

    # 当前的方式，自0.5以来已经使用
    session.query(Houses).join(Houses.rooms, Room.closets)
    session.query(Houses).options(eagerload_all(Houses.rooms, Room.closets))

``ScopedSession.mapper``被移除
--------------------------------------

这个功能提供了一种映射器扩展，将基于类的功能与特定的“ScopedSession”关联起来，特别是提供了这样的行为，使得新的对象实例自动与该会话关联。这个功能被教程和框架过度使用，导致用户由于其隐式行为而产生了极大的困惑，并在0.5.5版中被弃用。可以在[wiki:UsageRecipes/SessionAwareMapper]上找到复制其功能的技术。