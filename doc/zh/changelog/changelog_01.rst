版本记录
=============

                
.. changelog::
    :version: 0.1.7
    :released: Fri May 05 2006

    .. change::
        :tags: 
        :tickets: 

      顶点排序算法的一些修复。

    .. change::
        :tags: 
        :tickets: 

      Postgres添加DISTINCT ON支持（只需提供distinct=[col1，col2 ..]）。

    .. change::
        :tags: 
        :tickets: 

      sql expressions添加__mod__（%运算符）。

    .. change::
        :tags: 
        :tickets: 

      从父映射继承的属性"order_by”。

    .. change::
        :tags: 
        :tickets: 

      修复了mapper UPDATES / DELETEs使用的列类型。

    .. change::
        :tags: 
        :tickets: 

      使用convert_unicode = True时，反射失败，已经修复。

    .. change::
        :tags: 
        :tickets: 

      类型类型类型！仍未解决....不得不再次使用TypeDecorator :(

    .. change::
        :tags: 
        :tickets: 

      Mysql二进制类型将数组输出转换为缓冲区，修复PickleType。

    .. change::
        :tags: 
        :tickets: 

      最后一次修复attributes.py的内存泄漏。

    .. change::
        :tags: 
        :tickets: 

      基于每个数据库支持它们的unittests的合格。

    .. change::
        :tags: 
        :tickets: 

      修复了列默认值将覆盖INSERT对象的VALUES子句的错误。

    .. change::
        :tags: 
        :tickets: 

      修复了带有模式名称的表def会强制引擎连接的错误。

    .. change::
        :tags: 
        :tickets: 

      用于括号的修复使INSERT / UPDATE中的子查询正常工作。

    .. change::
        :tags: 
        :tickets: 

      HistoryArraySet获得extend（）方法。

    .. change::
        :tags: 
        :tickets: 

      修复了除=以外的其他比较运算符的延迟加载支持。

    .. change::
        :tags: 
        :tickets: 

      延迟加载修复，其中联接条件中的两个比较指向同一列。

    .. change::
        :tags: 
        :tickets: 

      在mapper中添加了一个"construct_new"标志，将使用__new__来创建实例而不是__init__（标准为0.2）。

    .. change::
        :tags: 
        :tickets: 

      将selectresults.py添加到SVN，上次错过了它

    .. change::
        :tags: 
        :tickets: 

      允许通过一个关联表将表与其自身建立多对多关系

    .. change::
        :tags: 
        :tickets: 

      修复"polymorphic"示例，以便在populate_instance()方法外使用lazyload支持。

    .. change::
        :tags: 
        :tickets: 

      致力于"invalidate"早期的session.refresh()/session.expire()的改进/修复。

    .. change::
        :tags: 
        :tickets: 

      完全删除当前会话中的对象的session.expunge()。

    .. change::
        :tags: 
        :tickets: 

      在引擎执行compile时，允许\*args，\**kwargs传递，以允许更容易地创建具有事务功能的装饰器函数。

    .. change::
        :tags: 
        :tickets: 

      添加了ResultProxy的迭代器接口："for row in result:..."

    .. change::
        :tags: 
        :tickets: 

      tx = session.begin(); tx.rollback(); tx.begin()添加了断言，即不能在回滚之后使用它。


.. changelog::
    :version: 0.1.6
    :released: Wed Apr 12 2006

    .. change::
        :tags: 
        :tickets: 

      添加了MS-SQL支持.

    .. change::
        :tags: 
        :tickets: 

      具有继承的ActiveMapper的初步支持（Jeff Watkins）

    .. change::
        :tags: 
        :tickets: 

      添加了一个"mod"系统，它允许改进/扩展核心功能，使用函数“install_mods（\*modnames）”

    .. change::
        :tags: 
        :tickets: 

      添加了第一个"mod"，SelectResults，它修改了mapper selects以返回将范围转换为LIMIT / OFFSET查询的生成器
      （Jonas Borgstr？）

    .. change::
        :tags: 
        :tickets: 

      MapperExtension添加了populate_instance()方法，允许一个扩展更改对象属性的填充。该方法可以调用另一个映射器的populate_instance()方法来代理从一个映射器到另一个映射器的属性填充;也内置一些行翻译逻辑，以帮助处理这一点。

    .. change::
        :tags: 
        :tickets: 

      Mapper查询能力彻底分离。这提高了映射器的性能. 

    .. change::
        :tags: 
        :tickets: 

      对象存储/Session重构，现在使用flush()方法正式保存对象, 已经将Session的begin/commit功能转移到LegacySession中，直到0.2系列。

    .. change::
        :tags: 
        :tickets: 

      类型系统订阅为查询编译时的引擎而不是方案构建时间。这简化了类型系统以及ProxyEngine。

    .. change::
        :tags: 
        :tickets: 

      向 Mapper 添加"version_id "关键字，这个关键字应该引用一个Column对象，具有类型Integer、最好是非空的，它将在映射表上用于跟踪版本号。每次保存操作都会将该数字递增，并在UPDATE / DELETE条件中指定，以使其纳入返回的行计数中，如该值不是预期计数，则会产生ConcurrencyError。 

    .. change::
        :tags: 
        :tickets: 

      向 Mapper 添加 'entity_name'关键字，现在，一个Mapper与类相关联，而且可选的entity_name参数，它是一个字符串，默认情况下是None。对于一个类，可以创建任意数量的主映射器，由entity名称限定。这些类的实例将通过其限定的entity名称的映射器发出所有的加载和保存操作，并保持在标识图中具有单独的标识，对于一个其他方面等效的对象。

    .. change::
        :tags: 
        :tickets: 

      属性系统的修复,如果一个对象被提交，则如果没有被加载，则其惰性加载的列表就会消失

    .. change::
        :tags: 
        :tickets: 

      更好的对象属性支持，可以说myclass.attr.property，它将为您提供与该属性对应的PropertyLoader，即myclass.mapper.props['attr']

    .. change::
        :tags: 
        :tickets: 

      急切读取使用别名优化。

    .. change::
        :tags: 
        :tickets: 

      新增标记"use_update"，用于关系。它指示该关系应该由两条UPDATE语句处理，要么在主要INSERT之后或在主要DELETE之前。处理循环行依赖性。 

    .. change::
        :tags: 
        :tickets: 

      添加异常模块，除了一些KeyError/AttributeError之外，所有引发的异常（从这些类继承）。

    .. change::
        :tags: 
        :tickets: 

      连续的create_engine参数（主机/主机名，数据库名，密码，等）。

    .. change::
        :tags: 
        :tickets: 

      添加SELECT语句嵌入到列子句中的支持，使用标志"scalar=True"。

    .. change::
        :tags: 
        :tickets: 

      修复了向mappers建立的关联当中，使用多个键的对象未被选中的错误。

    .. change::
        :tags: 
        :tickets: 

      修复了关于创建自我引用的映射器，使用回引用，比如在继承关系中的错误（但仍不容易- 改变集 1019）

    .. change::
        :tags: 
        :tickets: 

      添加PSQL中的date/time问题 None的修复（在change set 1005）

    .. change::
        :tags: 
        :tickets: 

      MySQL4自定义表引擎的修复,  i.e. TYPE而不是ENGINE

    .. change::
        :tags: 
        :tickets: 

      增强logging，包括时间戳和某种程度的可配置格式化系统

    .. change::
        :tags: 
        :tickets: 

      向ActiveMapper添加一些支持i.e. 许多对多关系。

    .. change::
        :tags: 
        :tickets: 

      Double和TinyInt支持mysql

.. changelog::
    :version: 0.1.3
    :released: Thu Mar 02 2006

    .. change::
        :tags: 
        :tickets: 

      更正了巨大数量的递归调用，使得schema代码在正常完成之前已经运行了994次，幸亏jpellerin发现了这个问题。

.. changelog::
    :version: 0.1.1
    :released: Thu Feb 23 2006

    .. change::
        :tags: 
        :tickets: 

      更正了目标表没有被attach到单一表上的错误。

    .. change::
        :tags: 
        :tickets: 

      修复了基于 join 的mappers的mapper.get（）不选择多个键入对象的错误。

    .. change::
        :tags: 
        :tickets: 

      现在sql包可以独立运行，可以生成不带引擎依赖项的 selects、inserts等等。建立在新的TableClause/ColumnClause词法对象之上。Schema的Table/ Column对象是它们的“physical”子类。这一度简化了schema/sql关系，扩展性（如proxyengine）以及整体性能。

    .. change::
        :tags: 
        :tickets: 

      objectstore.commit现在返回一个事务对象（SessionTrans），以更清楚地表示事务边界。

    .. change::
        :tags: 
        :tickets: 

      计划中添加的Index对象，拥有create/drop支持。

