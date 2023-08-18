SQLAlchemy 0.5有哪些新特性？
===========================

.. 注意::
    
    本文档介绍了SQLAlchemy 0.4和最新版本0.5之间的变化。
    SQL文档：2010年1月16日
    本文档的日期：2009年8月4日


此指南记录了影响用户从0.4系列迁移应用程序到0.5的API更改。对于那些使用“Essential SQLAlchemy”的人也建议看一下，因为它只包括了0.4并且似乎甚至有一些旧的0.3模式。请注意，SQLAlchemy 0.5删除了许多在0.4系列的时间段内已弃用的行为，并且还弃用了特定于0.4的更多行为。

主要文档更改
===========================

文档的某些部分已经重新编写，并且可以用作新ORM功能的介绍。``Query``和``Session``对象特别具有某些API和行为上的不同点，从根本上改变了许多基本的操作方式，特别是关于构造高度自定义的ORM查询以及处理过时的会话状态，提交和回滚。

* `ORM教程
  <https://www.sqlalchemy.org/docs/05/ormtutorial.html>`_

* `会话文件
  <https://www.sqlalchemy.org/docs/05/session.html>`_

退化源
===================

另一个信息来源记录在一系列单元测试中，这些单元测试说明了一些常见“查询”模式的最新用法; 可以在
[source:sqlalchemy/trunk/test/orm/test_deprecations.py]
查看此文件。

需求更改
====================

* 需要Python 2.4或更高版本。SQLAlchemy 0.4语句是具有Python 2.3支持的最后一个版本。

ORM
=========================

* **查询中的列级表达式。** - 如细节
  在ORM说明中所示，“Query”具有创建特定SELECT的能力
  语句，而不仅仅是针对完整行的语句：

  ::

      session.query(User.name, func.count(Address.id).label("numaddresses")).join(
          地址
      ).group_by(User.name)

  任何多列/实体查询返回的元组均为*命名'元组：

  ::

      for row in (
          session.query(User.name, func.count(Address.id).label("numaddresses"))
          .join(Address)
          .group_by(User.name)
      ):
          print("name", row.name, "number", row.numaddresses)

  “查询”有一个“声明”访问器，以及
  一个“subquery()”方法允许使用“Query”来
  创建更复杂的组合：

  ::

      subq = (
          session.query(Keyword.id.label("keyword_id"))
          .filter(Keyword.name.in_(["beans", "carrots"]))
          .subquery()
      )
      recipes = session.query(Recipe).filter(
          exists()
          .where(Recipe.id == recipe_keywords.c.recipe_id)
          .where(recipe_keywords.c.keyword_id == subq.c.keyword_id)
      )

* **建议使用显式的ORM别名进行别名加入**
  - “aliased()”函数生成一个“alias”of a
  类，它允许使用ORM查询中的别名进行细粒度的控制。虽然
  仍可使用表级别别名（即table.join（someothertable）），但ORM级别的
  别名保留了ORM映射对象的语义，这对于继承映射，选项和
  其他场景非常重要。例如：

  ::

      Friend = aliased(Person)
      session.query(Person, Friend).join((Friend, Person.friends)).all()

* **查询。大大增强。**-您现在可以指定
  为多种方式中的目标和ON子句之一。
  单个类可以在提供相关外键的情况下提供，方法是
  使用与“table.join（someothertable）”相同的方式来形成join。一个目标和一个明确的
  ON条件也可以提供，其中ON条件可以是“relation ()”名称，
  实际类描述符或SQL表达式。或旧方式的“relation ()”
  名称或类描述符也可以正常工作。请参阅ORM教程
  其中有几个例子。

* **推荐Declarative用于不需要（也不喜欢）表和的应用程序**
  映射器之间的抽象“ - [/ docs / 05 / reference / ext / declarative.html
  Declarative]模块，用于将``Table''，''mapper()`'和用户定义的table类对象结合在一起，由于
  简化了应用程序配置，确保“每个类一个映射器”模式，并允许整个范围的配置
  可用于不同的“mapper()`”呼叫。分开“映射器()”和“表”的使用现在已被称为“古典
  SQLAlchemy用法”，当然可以自由混合使用声明式。

* **已删除类中的“.c。”属性**（即
  ``MyClass.c.somecolumn''）。与0.4一样，类级别的属性可用作查询元素，即
  ``Class.c.propname''现已被``Class.propname''取代，并且``c''属性继续保留在``Table''
  对象上，其中它们指示``Table''对象上存在的``Column''对象的命名空间。

  要获取映射类的表（如果您没有保存它）：

  ::

      table = class_mapper(someclass).mapped_table

  迭代某个表中的列：

  ::

      for col in table.c:
          print(col)

  使用特定列的工作方式：

  ::

      table.c.somecolumn

  类绑定的描述符支持完整的列
  运算符以及文档记录的关系导向
  操作符，例如“has（）”，“any（）”，“contains（）”等。

  删除“一词。”的原因是在0.5中
  类绑定的描述符传递可能不同
  意思，以及有关类映射的信息，
  相比简单的“Column”对象 - 这里有使用案例
  您可能想专门使用其中之一。一般而言，使用类绑定-描述符会调用一组映射/多态感知翻译，使用table-绑定列则不会。在0.4中，这些翻译
  适用于所有表达式，但是在0.5中，它们不再完全区分列和映射描述符，而仅将翻译应用于后者。因此，在许多情况下，特别是在处理加入表时
  遗传配置以及使用“query（<columns>）”时，“Class.propname”和“table.c.colname”不能互换使用。

  例如，“session.query（users.c.id，users.c.name）”与“session.query（User.id, User.name）”是不同的;在后一种情况下，“Query”知道使用的映射器，并且还可以使用进一步的映射器特定操作，例如“query.join（<propname>）”、“query.with_parent（）”等， 但是在前一种情况下不可以。另外，在多态继承方案中，“Class”绑定的描述符是指在用于多态选择的表多态查询中存在的列，而不一定是直接对应于描述符的表列。例如，通过对所有由加入表继承到与每个表的'person_id'列相关联的类集进行关联来创建一组类将使它们的“Class.person_id”属性映射到“person_id”列，而不是子类表。在0.4年，这种行为会自动将此类行为映射到table-附加“Column”对象。在0.5中，已删除了此自动转换，以使您实际上*可以*使用table-绑定列作为重写多态查询时发生的翻译;这使得“Query”能够在联接表或具体表继承设置中创建优化的选择，以及可移植的子查询等。

* **会话现在会自动与事务同步。** 默认情况下，会话
  现在自动与事务同步，包括自动刷新和自动到期。一个交易一直存在，除非使用“autocommit”选项禁用。当所有三个标志设置为默认值时，会话在回滚后会恢复得很好，很难在会话中提取过时的数据，详情请参见新的Session
  文档。

* **隐式排序已删除**，这将影响基于ORM
  用户依赖于SA的“隐式排序”行为，该行为指的是所有没有
  “order_by（）”的查询对象将ORDER BY“id”或“oid”列
  主映射表，并且所有惰性/饱和集合应用类似的排序。在0.5中，必须明确地在“mapper()”和“relation()”对象上配置自动排序，或在使用“Query”时否则添加自动排序。

  将0.4映射转换为0.5，使其排序行为与0.4或以前的版本极为相似，请使用mapper（...，server_default ='val'），弃用Column（...，PassiveDefault（'val'））。
  现在使用“server_default = FetchedValue（）”替换了“PassiveDefault（''）”形式标记作为受外部触发器影响的列的一种方法，并且没有DDL副作用。

* **Session现在是
  autoflush = True / autoexpire = True / autocommit = False。** -为了
  配置它，只需不带参数调用“sessionmaker（）”。名称“transactional = True”现在
  “autocommit = False”。在发出每个查询时进行flush（使用“autoflush = False”禁用），在每个“commit（）”中（如旧方式）和在每个“begin_nested（）”之前进行flush（因此将回滚到SAVEPOINT很有意义）。所有对象均在每个commit（后以及每个rollback（后过期。在回滚后，待定对象会被清除，删除的对象会移回persistent状态。这些默认值可以很好地协同工作，在很难将过时数据带入会话的情况下，容易导致old技术，例如“clear（）”（也重命名为“expunge_all（）”）

  P.S .:会话现在可以在“rollback（）”后重新使用。标量和集合属性更改，添加和删除均被回滚。

* **session.add（）替换session.save（），session.update（），
  session.save_or_update（）。 **- the
  “session.add（someitem）”和“session.add_all（[list of
  items]）”方法替换了保存（），“update（）”和
  `save_or_update（）`。这三种方法将在整个0.5中保持不兼容。

* **backref配置简化。** - “backref（）”功能现在
  使用前向“relation（）”的“primaryjoin”和
  “secondaryjoin”参数，当不明确声明时使用。现在是
  不需要在两个方向上分别指定“primaryjoin” /“secondaryjoin”。

* **简化多态选项。** - ORM的
  “多态加载”行为已经简化了。在0.4中，
  Mapper（）具有名为“polymorphic_fetch”的参数
  可配置为“select”或“deferred”。删除了此选项；
  现在，映射器仅推迟没有出现在SELECT语句中的任何列。
  使用的实际SELECT语句由
  “with_polymorphic”mapper参数控制（在0.4中也是这样
  并替换了“select_table”），以及“with_polymorphic（）”方法
  在“查询”中（在0.4中也是如此）。

  继承类的延迟加载的改进是映射现在生成所有情况下的“优化”SELECT语句。即，如果类B从A继承，并且仅供B类拥有的多个属性已过期，则刷新操作将仅包括B的表在SELECT语句中，并且不会连接到A。

* “Session.execute（）”方法将纯字符串转换为“text（）”构造，以便可以指定所有绑定参数为“：bindname”，而无需显式调用“text（）”。如果需要“原始”SQL，请使用“session.connection（）.execute（'raw
  text'）”。

* “session.Query（）.iterate_instances（）”现已更名为仅使用“instances（）”。“旧式”方法返回列表而不是迭代器不再存在。如果你依赖于这种行为，则应使用“list（your_query.instances（））”。

ORM扩展
=================

在0.5中，我们正在推进更多修改和扩展ORM的方法。这里是一个摘要：

* **MapperExtension。** - 这是经典的扩展
  类，仍然存在。很少需要“create_instance（）”方法，而需要
  “populate_instance（）”。要对初始化进行控制
  从数据库加载对象时，请使用“reconstruct_instance（）”方法，或者更容易地使用
  文件中描述的“@reconstructor”装饰符。

* **SessionExtension。** - 这是用于会话事件的易于使用的扩展
  类。特别是，它提供了“before_flush（）”，“after_flush（）”和
  “after_flush_postexec（）”方法。在许多情况下，使用这种用法
  建议使用而不是在“MapperExtension”内部使用，因为在
  在“before_flush（）”中，您可以自由地修改会话的刷新计划，这是不能
  从“MapperExtension”中进行的。

* **AttributeExtension。** - 这个类现在是公共API的一部分，
  允许拦截属性上的用户事件，包括属性集和删除
  操作，以及集合附加和移除。它也允许修改要设置或附加的值。
  文档中描述的“@validates”装饰符提供了一种将任何映射
  属性标记为由特定类方法“验证”的快速方式。

* **属性仪表盘定制。** - 提供了一个API以完全替换SQLAlchemy的属性仪表盘，或者仅在某些情况下增强它。此API是为Trellis工具包而生产的，但可以作为公共API使用。分发中提供了一些示例，在``/examples/custom_attributes``目录中。

模式/类型
============

* **String没有长度不再生成TEXT，它是VARCHAR** - “String”类型不再神奇地转换为“Text”类型，当没有长度时指定它。这仅在发出CREATE TABLE时有影响，因为它将没有长度参数的“VARCHAR”发出DDL，这在许多（但不是全部）数据库上无效。要创建TEXT（或CLOB，即无限制的字符串）列，请使用“Text”类型。

* **具有mutable = True的PickleType（）需要__eq __（）方法** - 当mutable = True时，“PickleType”类型需要比较值。比较“pickle.dumps（）”的方法效率低而不可靠。如果传入的对象没有实现“__eq __（）”并且也不是“None”，则使用“dumps（）”比较，但会发出警告。对于“__eq __（）”中包含所有字典，列表等的类型来说，比较将使用“==”，现在默认情况下是可靠的。

* **TypeEngine / TypeDecorator的convert_bind_param（）和convert_result_value（）方法已删除。**-O'Reilly博客不幸地记录了这些方法，尽管它们在0.3之后被弃用。对于子类化“TypeEngine”的用户定义类型，应使用“bind_processor（）”和“result_processor（）”方法进行绑定/处理。在扩展“TypeEngine”或“TypeDecorator”的用户定义类型中使用旧0.3样式的用户定义类型可以通过使用以下适配器轻松适应新样式：

  ::

      class AdaptOldConvertMethods（object）：
          “”“一个混入，用于调整0.3样式的convert_bind_param和
          convert_result_value方法

          ”“”

          def bind_processor（self，dialect）：
              def convert（value）：
                  return self.convert_bind_param（value，dialect）

              返回转换

          def result_processor（self，dialect）：
              def convert（value）：
                  return self.convert_result_value（value，dialect）

              返回转换

          def convert_result_value（self，value，dialect）：
              返回值

          def convert_bind_param（self，value，dialect）：
              返回值

  要使用上述mixin：

  ::

      class MyType（AdaptOldConvertMethods，TypeEngine）：
          ...

* Column和Table的“quote”标志以及“quote_schema”标志现在控制引用
  正面和反面。默认值为“空白'，意味着让常规引用规则生效。在
  “True”时，引用被强制打开。在``False``时，强制引用被关闭。

* ColumnDEFAULT值DDL现在可以更方便地使用Column（...，server_default ='val'）指定，弃用了``Column(..., PassiveDefault('val'))``。只有Python启动的默认值可以与“default =”一起使用，可以与server_default共存。新的“server_default = FetchedValue（）”替换了“PassiveDefault（''）”成语来标记列是否受到外部触发器的影响，并且没有DDL副作用。

* SQLite的“DateTime”，“Time”和“Date”类型现在仅接受datetime对象，而不是字符串作为绑定参数输入。如果您想创建自己的“混合”类型，该类型接受字符串并将结果作为日期对象返回（从您想要的任何格式），请创建一个基于“String”的“TypeDecorator”。如果您只想使用基于字符串的日期，请使用“String”。

* 此外，SQLite的“DateTime”和“Time”类型，当使用SQLite时，现在表示Python“datetime.datetime”对象的“microseconds”字段的方式与“str（datetime）”相同 - 作为小数秒，而不是毫秒的计数。也就是说：

  ::

       dt = datetime.datetime（2008,6,27,12,0,0,125）＃125 usec

       #老方法
       “2008-06-27 12:00:00.125”

       #现在的方法
       “2008-06-27 12:00:00.000125”

  因此，如果现有的SQLite基于文件的数据库打算在0.4和0.5之间使用，则必须将datetime列升级为存储新格式（注意：请测试此功能，我相当确定其是正确的）：

  .. sourcecode :: sql

       UPDATE mytable SET somedatecol =
          substr（somedatecol，0,19）||'.'||substr（（substr（somedatecol，21，-1）/ 1000000），3，-1）;

  或者，在这里启用“传统”模式：

  ::

       from sqlalchemy.databases.sqlite import DateTimeMixin

       DateTimeMixin.__传统的微秒__ = True

连接池不再默认为线程本地
================================================

0.4有一个令人遗憾的默认设置“pool_threadlocal = True”，因此在单个线程中使用多个会话时会出现意外的行为。在0.5中，此标志已关闭。要重新启用0.4的行为，请将“pool_threadlocal = True”指定为“create_engine（）”，或者通过“strategy =” threadlocal“”使用“threadlocal”策略。

\ *args已接受，不再接受\ *args
============================

使用“方法（\ *args）”与“方法（[args]）”之间的策略是，如果方法接受表示固定结构的变长项目集，则采用“\ *args”。如果方法接受变长项集的数据驱动，则采用“[args]”。

*各种查询选项函数“eagerload（）”，
   “eagerload_all（）”，“lazyload（）”，“contains_eager（）”，
   “defer（）”，“undefer（）”现在都接受包含可变长度的“\ *keys”参数
   作为其参数，这允许使用描述符制定路径，即：

  ::

          query.options(eagerload_all(User.orders, Order.items, Item.keywords))

  一个单独的数组参数仍会被接受用于向后兼容。

*类似地，“Query.join（）”和“Query.outerjoin（）”
  方法现在接受可变长度的\ *args，其中单个
  数组用于向后兼容：

  ::

          query.join（“orders”，“items”）
          query.join(User.orders，Order.items)

*列等的“in_（）”方法现在仅接受列表参数。它不再接受“\ *args”。

删除
=======

* **entity_name** - 此功能一直存在困难，并且很少使用。 0.5更深入地阐述了“entity_name”的用例，展示了进一步的问题，导致其被删除。如果需要针对单个类请求不同的映射，请将类拆分为单独的子类并单独进行映射。一个这样的例子位于[wiki：UsageRecipes / EntityName]。有关原理的更多信息描述在https://groups.google.c中。

* **get（）/load（）清理**

  已移除“load（）”方法。它的
  功能有些随意，基本上复制了
  来自Hibernate，其也并不是特别
  有意义的方法。

  要获得等效功能：

  ::

       x = session.query(SomeClass).populate_existing().get(7)

  “Session.get（cls，id）”和“Session.load（cls，id）”已删除。“Session.get（）”与“Session.query（cls）.get（id）”重复。

  “MapperExtension.get（）”也已删除（同样是“MapperExtension.load（）”）。要覆盖“Query.get（）”的功能，请使用子类：

  ::

       class MyQuery（Query）：
           def get（self，ident）：
               ...

       session = sessionmaker（query_cls = MyQuery）（）

       ad1 = session.query(Address).get(1)

*“sqlalchemy.orm.relation（）”


  已删除以下弃用的关键字参数：

  foreignkey，association，private，attributeext，is_backref

  特别是，“attributeext”替换为“extension”  - “AttributeExtension”类现在在公共API中。

*“session.Query（）”


  已删除以下弃用功能：

  list，scalar，count_by，select_whereclause，get_by，
  select_by，join_by，selectfirst，selectone，select，
  执行，select_statement，select_text，join_to，join_via，
  selectfirst_by，selectone_by，apply_max，apply_min，
  apply_avg，apply_sum

  此外对“join（）”，“outerjoin（）”，“add_entity（）”和“add_column（）”使用“id”关键字参数已删除。使用“aliased”构造向“Query”中的别名目标，以将结果列定向到目标。

  ::

          from sqlalchemy.orm import aliased

          address_alias = aliased(Address)
          print(session.query(User, address_alias).join((address_alias, User.addresses)).all())

*“sqlalchemy.orm.Mapper”


  * instances（）


  * get_session（）-此方法并不引人注目，但
    如果使用扩展名，例如“scoped_session（）”或旧的“SessionContextExt”
    使用时，即使父对象完全分离，它的效果是将懒加载与特定会话相关联。应用程序将不能按预期工作。

*“mapper（MyClass，mytable）”


  现在不再使用“c”类属性进行映射，即“MyClass.c”

*“sqlalchemy.orm.collections”


  _prepare_instrumentation别名为空
  调用已删除。

*“sqlalchemy.orm”


  已删除“EXT_PASS”别名“EXT_CONTINUE”。

*“sqlalchemy.engine”


  从“DefaultDialect.preexecute_sequences”到的别名已删除
  ``.preexecute_pk_sequences``。

  已删除已弃用的engine_descriptors（）函数。

*“sqlalchemy.ext.activemapper”


  模块已删除。

*“sqlalchemy.ext.assignmapper”


  模块已删除。

*“sqlalchemy.ext.associationproxy”


  代理的下传关键字参数“.append（item，\ ** kw）”已被删除，现在是``.append（item）``

*“sqlalchemy.ext.selectresults”，
  “sqlalchemy.mods.selectresults”

  已删除模块。

*“sqlalchemy.ext.declarative”


  已删除“declared_synonym（）”。

*“sqlalchemy.ext.sessioncontext”


  模块已删除。

*“sqlalchemy.log”


  别名“SADeprecationWarning”已删除
  “sqlalchemy.exc.SADeprecationWarning”。

*“sqlalchemy.exc”


  “exc.AssertionError”已删除，并使用相同名称的内置代替。

*“sqlalchemy.databases.mysql”


  已删除弃用的“get_version_info”dialect方法。

已重命名或已移动
=====================

*“sqlalchemy.exceptions”现在是“sqlalchemy.exc ”


  该模块可以在0.6之前仍然以旧名称导入。

*“FlushError”，“ConcurrentModificationError”，
  ``UnmappedColumnError`` - > sqlalchemy.orm.exc

  这些异常移动到orm包中。导入'
  'sqlalchemy.orm'会在sqlalchemy.exc中安装别名，以便向后兼容直到0.6。

*“sqlalchemy.logging”->“sqlalchemy.log”


  此内部模块已被重命名。不再需要进行特殊处理
  当使用py2app和类似的扫描导入的工具打包SA时。

*“session.Query（）.iterate_instances（）”->
  “session.Query（）.instances（）”。

已弃用
=========================

*“Session.save（）”，“Session.update（）”，
  “Session.save_or_update（）”

  所有三个替换为“Session.add（）”

*“sqlalchemy.PassiveDefault”


  使用“Column（server_default = ...）”进行翻译到sqlalchemy.DefaultClause()。
  在幕后。

*“session.Query（）.iterate_instances（）”。已更名为“instances（）”。