=============
0.5 更新日志
=============

.. changelog::
    :version: 0.5.9
    :released: 

    .. change::
        :tags: sql
        :tickets: 1661

      修复了表达式包中self_group()调用错误的问题。

.. changelog::
    :version: 0.5.8
    :released: 2010年1月16日

    .. change::
        :tags: sql
        :tickets: 

      Column上的copy()方法现在支持未初始化的匿名列对象。这使得可以轻松地创建用于多个子类共有的常见列的声明式辅助工具。

    .. change::
        :tags: sql
        :tickets: 

      默认生成器(如Sequence())在复制操作中可以正确转换。

    .. change::
        :tags: sql
        :tickets: 

      现在可以将Sequence()和其他DefaultGenerator对象作为Column的"default"和"onupdate"关键字参数的值，除了位置参数接受外。

    .. change::
        :tags: sql
        :tickets: 1568, 1617

      修复了一个列算术错误，会影响包含独立列表达式的克隆可选的列对应。通常只有在通过ORM行为测试新的0.6功能时才会注意到此错误，但是在SQL表达式级别上更加正确。

    .. change::
        :tags: postgresql
        :tickets: 1647

      extract()函数，在0.5.7中略微改进，需要更多工作才能生成正确的类型转换(在PG的EXTRACT中，类型转换在很多情况下似乎是必要的)。现在会使用基于PG的日期/时间/间隔算术文档的规则字典生成类型转换。它再次接受text()构造，这在0.5.7中已损坏。

    .. change::
        :tags: firebird
        :tickets: 1646

      将更多错误识别为断开连接。

.. changelog::
    :version: 0.5.7
    :released: 2009年12月26日

    .. change::
        :tags: orm
        :tickets: 1543

      contains_eager()现在能够与查询(Parent).join(Parent.somejoinedsubclass)自动生成的子查询配合使用，即当Parent与一个joined-table-inheritance子类进行关联时。以前，contains_eager()会错误地将子类表单独添加到查询中，从而产生笛卡尔乘积。

    .. change::
        :tags: orm
        :tickets: 1553

      现在，query.options()仅对潜在的进一步子加载加载的对象传播，对于这种行为相关的各种不可序列化选项(如contains_eager()生成的选项)保持独立的状态。

    .. change::
        :tags: orm
        :tickets: 1054

      Session.execute()现在根据传递的表示insert()/update()/delete()构造的表达式来定位特定于表和映射的绑定。

    .. change::
        :tags: orm
        :tickets:

      Session.merge()现在会正确地将一个many-to-one或uselist=False属性覆盖为None，如果在待合并的对象中属性也为None。

    .. change::
        :tags: orm
        :tickets: 1618

      修复了在合并包含空主键标识符的临时对象时可能发生的不必要选择错误。

    .. change::
        :tags: orm
        :tickets: 1585

      不可变的关系()扩展现在不会因为在多次检测期间导致重复扩展而被修改或在多个检测调用之间共享，这样就防止了将重复的扩展(如backref populators)插入到列表中。

    .. change::
        :tags: orm
        :tickets: 1504

      修复了在CompositeProperty上调用get_committed_value()的错误。

    .. change::
        :tags: orm
        :tickets: 1602

      修复了当在句柄列表中出现非映射列实体时，调用没有明确"left"边的join()时，查询将崩溃的错误。

    .. change::
        :tags: orm
        :tickets: 1616, 1480

      修复了在joined-table子类上配置代表复合列的情况下，复合列不会正确加载的错误，并在0.5.6中引入了修复。感谢Scott Torborg。

    .. change::
        :tags: orm
        :tickets: 1556

      许多对一关系的"use get"行为，即着急加载将回退到可能缓存的query.get()值，现在可以跨联接条件工作，在这种情况下，两个比较的类型不完全相同，但共享相同的“亲和度”，即整数和小整数。还允许反射类型的组合和非反射类型与0.5样式类型反射一起使用，例如PGText/Text(注意0.6反射类型为它们的通用版本)。

    .. change::
        :tags: orm
        :tickets: 1436

      在query.update()中传递Cls.attribute作为值字典中的键并使用synchronize_session='expire'('fetch'在0.6中)会导致错误。

    .. change::
        :tags: sql
        :tickets: 1603

      修复了两阶段事务中commit()方法未设置完整状态的问题，这使得后续调用close()方法可以成功。

    .. change::
        :tags: sql
        :tickets: 

      修复了 "numeric" paramstyle，这似乎是Informixdb使用的默认paramstyle。

    .. change::
        :tags: sql
        :tickets: 1574

      在选择的columns子句中重复表达式现在基于每个clause元素的标识而不是实际字符串进行了去重处理。这允许位置元素正确呈现，即使它们全部呈现相同，如"qmark"样式绑定参数。

    .. change::
        :tags: sql
        :tickets: 1632

      连接池连接上关联的光标(即_CursorFairy)现在将__iter__()代理到基础光标。

    .. change::
        :tags: sql
        :tickets: 1556

      类型现在支持"相关比较"操作，即Integer/SmallInteger是"兼容的"，或Text/String、PickleType/Binary等。部分 。

    .. change::
        :tags: sql
        :tickets: 1641

      修复了嵌套别名()无法被克隆或适应的错误(在ORM操作中经常出现)。

    .. change::
        :tags: sqlite
        :tickets: 1439

      sqlite dialect 现在可以为位于另一模式中的表生成 "CREATE INDEX" 语句。

    .. change::
        :tags: postgresql
        :tickets: 1085

      添加了对反映DOUBLE PRECISION类型的支持，通过新的postgres.PGDoublePrecision对象来实现。在0.6中postgresql.DOUBLE_PRECISION。

    .. change::
        :tags: postgresql
        :tickets: 460

      添加了对正则间隔 INTERVAL YEAR TO MONTH 和 INTERVAL DAY TO SECOND 语法的支持。

    .. change::
        :tags: postgresql
        :tickets: 1576

      将当前模式或显式序列规定的模式加入到"has_sequence"查询中，以考虑。

    .. change::
        :tags: postgresql
        :tickets: 1611

      修复extract()的行为，以对运算符优先级规则应用 "::" 运算符，当运用 "timestamp" typecast 时，确保适当的适量化。

    .. change::
        :tags: mssql
        :tickets: 1561

      改变了构造pyodbc连接参数时TrustedConnection名称为Trusted_Connection。

    .. change::
        :tags: oracle
        :tickets: 1637

      "table_names" dialect函数，由MetaData.reflect()使用，现在省略"index overflow tables"，并使用Oracle在使用"index only tables"时生成的系统表。这些表无法通过SQL访问，也无法反射。

    .. change::
        :tags: ext
        :tickets: 1570, 1523

      在类被构建后(即通过类级属性赋值)，可以将列添加到joined-table声明超类上，并将该列传播到子类。这与0.5.6中修复的情况相反。

    .. change::
        :tags: ext
        :tickets: 1491

      在分片示例中，修正了一个细节不精确的问题。最好使用列的等价性在ORM中进行比较，即使用col1.shares_lineage(col2)。

    .. change::
        :tags: ext
        :tickets: 1606

      从ShardedQuery中删除了未使用的load()方法。

.. changelog::
    :version: 0.5.6
    :released: 2009年9月12日

    .. change::
        :tags: orm
        :tickets: 1300

      修复了在复合主键中继承鉴别器部分更新失败的错误。 

    .. change::
        :tags: orm
        :tickets: 1507

      修复了一个阻止一个双向多对多引用的一侧将自己声明为“只读视图”的错误。

    .. change::
        :tags: orm
        :tickets: 1526

      添加断言，以防止@validates函数或其他AttributeExtension加载未加载的集合，从而可能破坏内部状态。

    .. change::
        :tags: orm
        :tickets: 1519

      修复了一个问题，该问题阻止了两个实体在单个flush()中互相替换各自的主键值，对于某些操作顺序，这是不可能的。

    .. change::
        :tags: orm
        :tickets: 1485

      修复了一个晦涩的问题，当在基类到joined-table子类的自引用急切加载中使用自引用急切加载时，它将使用与父对象的"subclass"表中的数据填充相关对象的"subclass"表。

    .. change::
        :tags: orm
        :tickets: 1477

      现在，relation()拥有更大的 "重写" 能力，这意味着显式指定某个关系与父类关系不同的子类将在flush期间被公认并被识别。 这支持了从 具体继承 安装的多对多关系。 除此之外的情况，效果各异。

    .. change::
        :tags: orm
        :tickets: 1483

      当集合被改变时，除非设置了"single_parent=True"，否则在另一侧的多对一反向引用不会加载"旧"值。现在直接赋值多对一将加载"旧"值，以更新那个值的反向引用集合, 该值可能已经存在于会话中，因此保持了0.5的行为合同。

    .. change::
        :tags: orm
        :tickets: 1480

      修复了在joined-table子类上配置基于column_property()的载入/刷新的情况下，组合列不会被正确评估的错误。

    .. change::
        :tags: orm
        :tickets: 1488

      对于非具体继承设置，MapperProperty对象重写了一个继承的映射器用于覆盖底层映射器，不会随机与其他属性扩展冲突。

    .. change::
        :tags: orm
        :tickets: 1487

      UPDATE和DELETE在标准SQL中不支持ORDER BY、LIMIT、OFFSET等。现在，如果已调用limit()、offset()、order_by()、group_by()或distinct()中的任何一个，则Query.update()和Query.delete()将引发异常。

    .. change::
        :tags: orm
        :tickets:

      在SQL/实体表达式以外的情况下，使用query()时，如果查询中出现非SQL/entities表达式，则改进了错误消息。

    .. change::
        :tags: orm
        :tickets: 1440

      如果False或0用作多态鉴别器的值，现在在子类和基类中都能正常工作。

    .. change::
        :tags: orm
        :tickets: 1424

      在Query中添加enable_assertions(False)方法，以禁用预期状态的通常断言-被Query子类用于设计定制状态。请参阅#https://www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery 的示例。

    .. change::
        :tags: orm
        :tickets: 1501

      修复了一个问题，在该问题中，一个映射对象的__len__()或__nonzero__()方法会导致状态更改。

    .. change::
        :tags: orm
        :tickets: 1506

      修复了在Weak/StrongIdentityMap.add()中抛出的不正确的异常。

    .. change::
        :tags: orm
        :tickets: 1522

      修复了出现"没有找到FROM子句"错误的查询.join()在没有明确"left"边的情况下进行调用时，当列实体出现在columns列表中时。

    .. change::
        :tags: orm
        :tickets: 1486

      修复了一种假设性的问题，即当使用旧的polymorphic_union函数并使用继承映射器时，会计算出错误的主键值。

    .. change::
        :tags: sql
        :tickets: 1373

      修复了Column.copy()无法复制默认值和onupdates的错误。

    .. change::
        :tags: sql
        :tickets: 

      修复了在extract()中引入的一个错误，当将字符串"field"参数视为ClauseElement时，会在更复杂的SQL转换中引发各种错误。

    .. change::
        :tags: sql
        :tickets: 1420

      类似DISTINCT的一元表达式将其类型处理传播到结果集，允许进行unicode等转换。

    .. change::
        :tags: sql
        :tickets: 1482

      在Table和Column中传递空dict作为"info"参数将引发异常的问题已经修复。

    .. change::
        :tags: oracle
        :tickets: 1309

      回退Oracle别名名称未被截断的0.6修复。

    .. change::
        :tags: ext
        :tickets: 1446

      由associationproxy生成的集合代理现在是可pickle的。但是，除非它定义了__getstate__和__setstate__，否则仍无法pickle用户定义的proxy_factory。

    .. change::
        :tags: ext
        :tickets: 1468

      如果__table_args__作为没有字典参数的元组传递，则declarative现在会发出易于理解的异常。改进了文档。

    .. change::
        :tags: ext
        :tickets: 1527

      现在可以将在MetaData中定义的Table对象用于发送到primaryjoin/secondaryjoin/secondary的字符串表达式-名称从声明基数的MetaData中提取。

    .. change::
        :tags: ext
        :tickets: 1523

      在类被构建后，可添加列到joined-table声明后，并将列添加到底层Table，而不是引发关于"没有这样的列，请改用column_property()"的错误。

    .. change::
        :tags: test
        :tickets: 

      将示例添加到测试套件中，以便可以定期执行它们，并清理掉一些过时引用的警告。.. changelog::
    :version: 0.5.3
    :released: 2009年3月24日

    .. change::
        :tags: mssql
        :tickets: 

      修改了保存点逻辑的工作方式，以防止其对非保存点取向的程序产生影响。
      保存点支持仍然是实验性的。

    .. change::
        :tags: mssql
        :tickets: 1310

      添加了 MSSQL 的保留字，涵盖了 2008 年的所有版本及更早版本。

    .. change::
        :tags: mssql
        :tickets: 1343

      修复了信息模式与基于二进制排序的数据库不兼容的问题。清理了信息模式，因为现在只有 MSSQL 使用它。

    .. change::
        :tags: sqlite
        :tickets: 1402

      修复了 SLBoolean 类型，以便它正确地将 1 视为 True。

    .. change::
        :tags: sqlite
        :tickets: 1273

      修复了 float 类型，使其正确地映射到 SLFloat 类型在反映时。

    .. change::
        :tags: extensions
        :tickets: 1379

      修复将延迟或其他列属性添加到声明性类的问题。

.. changelog::
    :version: 0.5.3
    :released: 2009年3月24日

    .. change::
        :tags: orm
        :tickets: 1315

      "session.flush()" 中的 "objects" 参数已弃用。在父子对象之间表示链接的状态（自上而下或自下而上）不支持在链接的一侧有 "flushed" 状态而在另一侧没有，因此支持此操作会导致误导性结果。

    .. change::
        :tags: orm
        :tickets: 

      Query 现在实现了 "__clause_element__()"，它生成其可选择的值，这意味着 Query 实例可以被用于许多 SQL 表达式中，包括 col.in_(query)、union(query1,query2)、select([foo]).select_from(query) 等等。

    .. change::
        :tags: orm
        :tickets: 1337

      如果需要，Query.join() 现在可以构造多个 FROM 语句。例如，query(A, B).join(A.x).join(B.y) 可能是 SELECT A.，B. FROM A JOIN X, B JOIN Y。Eager 加载还可以将其加入这些多个 FROM 语句。

    .. change::
        :tags: orm
        :tickets: 1347

      修复了在 dynamic_loader() 中存在的 append/remove 事件在构建后未被传播到 UOW 以拾取 flush() 的问题。

    .. change::
        :tags: orm
        :tickets: 

      在未显示设置 column_prefix 的情况下，现在检查属性是否存在类级别名称，也就是在不添加到 mapper() 属性字典中的属性上。

    .. change::
        :tags: orm
        :tickets: 1315

      对特定集合属性上的 session.expire() 还会清除任何待处理的反向引用添加，以便下一个访问正确返回数据库中存在的内容。解决了某种程度上的问题，因为我们正在考虑删除 flush() 特性。

    .. change::
        :tags: orm
        :tickets: 

      现在，Session.scalar() 会将原始 SQL 字符串转换为 text()，并接受相同的可选参数 \**kw。

    .. change::
        :tags: orm
        :tickets: 

      修改了关系中“确定方向”的逻辑，使得可以确定具有困难情况的方向，例如：mapper(A.join(B)) -> relation-> mapper(B)。

    .. change::
        :tags: orm
        :tickets: 1306

      只刷新部分对象集合时使用 session.flush([somelist]）时，用于判断持久状态的挂起对象不能被错误地作为持久对象添加。

    .. change::
        :tags: orm
        :tickets: 1314

      在编写 InstrumentationManager 的“post_configure_attribute”方法之后，让“listen_for_events.py”示例再次正常工作。

    .. change::
        :tags: orm
        :tickets: 

      现在检测具有相同方向的前向和互补的后向引用，例如ONETOMANY或MANYTOONE，然后引发错误消息。避免在以后出现“CicularDependencyErrors”。

    .. change::
        :tags: orm
        :tickets: 

      修复了 Query 中与具有共同基类的多个联接表继承实体同时选择时的问题，例如：

      - 以前在 "A JOIN    B" 上应用的
        关于“B”的适应不准确地部分应用到“A”中
      
      - 关系上的对比（即 A.related==someb）
        没有在应该适应时适应.
      
      - 其他过滤，如
        query(A).join(A.bs).filter(B.foo=='bar')，错误地
        适应"B.foo"，就好像它是"A"一样。

    .. change::
        :tags: orm
        :tickets: 1325

      在与左侧使用别名() 构造联接时、右侧的 of_type() 是 on_type 类型时，通过 EXISTS 子句适应的问题得到了修正。

    .. change::
        :tags: orm
        :tickets: 

      在 sqlalchemy.orm.attributes 中添加了一个属性函数助手 ``set_committed_value``，给定对象、属性名称和值，将该值设置在该对象上作为其 "committed" 状态，即已经从数据库加载的状态。对于创建自己家庭集合加载程序等有所帮助。

    .. change::
        :tags: orm
        :tickets: 

      如果属性为非映射器 / 类覆盖描述符，则 Query 不会因弱引用错误而失败，将报 "Invalid column expression"。

    .. change::
        :tags: orm
        :tickets: 

      Query.group_by() 正确考虑了别名应用于 FROM 子句时的情况，例如，使用 select_from()，使用 with_polymorphic()，或使用 from_self()。

    .. change::
        :tags: sql
        :tickets: 

      当在比较操作中使用时，选择别名() 的情况将转换为“标量子查询”。例如，ORM 在使用 query.subquery() 时也会执行此操作。

    .. change::
        :tags: sql
        :tickets: 1302

      在使用列属于 orm 中的功能性对象时，在 select() 中使用 use_labels (例如在 ORM column_property() 中使用) 时，Function 对象上的 _label 属性等会显示错误缺失.

    .. change::
        :tags: sql
        :tickets:

      现在 __ clause_element__() 接口已完全替换了 __ selectable__()。

    .. change::
        :tags: sql
        :tickets: 1299

      TypeEngine 用于缓存特定于方言的类型的每个方言的缓存现在是 WeakKeyDictionary 。这是为了防止 dialect 对象被引用并适用于应用程序创建任意数量的引擎或方言。这会产生一些小的性能损失，将在 0.6 中解决。

    .. change::
        :tags: sqlite
        :tickets: 

      修复了 SQLite 反射方法，以便在检测到没有非常量 cursor.description 时，不会触发自动光标关闭，因为近期版本的 pysqlite 调用 fetchone() 时，如果没有行存在，则会引发错误。

    .. change::
        :tags: postgresql
        :tickets: 

      当遇到具有多个表达式的索引时，索引反射不会失败。

    .. change::
        :tags: postgresql
        :tickets: 1327

      在 sqlalchemy.databases.postgres 中添加了 PGUuid 和 PGBit 类型。

    .. change::
        :tags: postgresql
        :tickets: 1327

      当在域中指定了未知 PG 类型时，没有将其指定的引用误导他。

    .. change::
        :tags: mssql
        :tickets: 

      对 pymssql 1.0.1 进行了初步的支持。

    .. change::
        :tags: mssql
        :tickets:

      修正了 mssql 中 max_identifier_length 未被使用的问题。

    .. change::
        :tags: extensions
        :tickets: 

      在序列化程序中修正了递归 pickling 问题，该问题由存在 EXISTS 或其他嵌入的 FROM 实体引起。

    .. change::
        :tags: extensions
        :tickets: 

      当使用 __ table__ 进行声明性设置以及该列已经存在于 __ table__ 中时，可以将 table-bound （表绑定）列作为属性添加，如果映射器() 属性字典中已经存在该列，则会将该列重新映射到给定的键，就像将其添加到映射器() 属性字典时一样。

.. changelog::
    :version: 0.5.2
    :released: 2009年1月24日

    .. change::
        :tags: orm
        :tickets: 

      删除了内部联接高速缓存，该缓存在重复使用 query.join() 等到 ad-hoc selectables 时可能会泄漏内存。

    .. change::
        :tags: orm
        :tickets: 

      Session 中的 “clear()”， “save()”， “update()”， “save_or_update()” 方法已弃用，替换为 “expunge_all()” 和 “add()”。ScopedSession 中也添加了 “expunge_all()” 方法。

    .. change::
        :tags: orm
        :tickets: 

      更新了“没有映射表”异常和添加了更明确的 __ table__ /__ tablename__ 异常以适应 Declarative。

    .. change::
        :tags: orm
        :tickets: 1237

      现在，具体继承映射器会用最终的 InstrumentedAttribute 对象在构建时建立类属性的仍然持久。去掉了 '_CompileOnAttr' / '__getattribute__（）'方法。净效应是，与声明性相关的 Column-based 映射类属性现在可以在类级别上完全使用，而不需要触发 mapper() 编译操作，这极大地简化了声明性的典型用法。

    .. change::
        :tags: orm
        :tickets: 

      column_property() 现在不再忽略未知的 \ ** 关键参数。

    .. change::
        :tags: orm
        :tickets: 

      修复了“row switch”机制中引入的错误，即将 INSERT / DELETE 转换为 UPDATE 时，与 joined-table 继承和包含不为子表中的子表（实际上，在 UPDATE 中没有 SET 子句将要呈现的情况下）贴在一起的对象将会导致问题。

    .. change::
        :tags: orm
        :tickets: 1281

      使用 delete-orphan 级联时，始终需要 delete cascade。没有指定删除而指定了 delete-orphan 现在会引发一个弃用警告。

    .. change::
        :tags: orm
        :tickets: 

      Query.from_self() 和 query.subquery() 现在都禁用了在生成的子查询中的急切关联。具有“禁用所有急切关联”功能可以通过新的 query.enable_eagerloads() 动态获得。

    .. change::
        :tags: orm
        :tickets: 

      接受 Query.join()/outerjoin() 中的 none，将从查询中删除任何挂起的 order_by 状态，并取消任何在 Mapper / Relation 中配置的排序。

    .. change::
        :tags: orm
        :tickets: 1237

      在具体映射器的属性中指定的具有双向引用的关系已添加到新的'relationship()'的关键字参数 back_populates 中。当创建具有混合继承层次结构的具体映射器和另一个类之间的双向关系时，需要这种配置。

    .. change::
        :tags: orm
        :tickets: 

      具体继承 mappers 现在会使用混入到子类本身的 __bases__ 中的搜索来查找 "inherits" 类。

    .. change::
        :tags: orm
        :tickets: 

      即使显式给出了 "inherits" map参数，单个继承子类的继承表主加入条件也会正确地解释。 

    .. change::
        :tags: orm
        :tickets: 

      如果 backref() 的“foreign_keys”参数是字符串，则 Declarative 将正确解释该参数。

    .. change::
        :tags: extensions
        :tickets: 

      在序列化器中修复了递归 pickling 问题，该问题由嵌入了 EXISTS 或其他 FROM 构造而产生。

.. changelog::
    :version: 0.5.0rc4
    :released: Fri Nov 14 2008

    .. change::
        :tags: orm
        :tickets: 

      Query.count()被强化，可以在更广泛的情况下工作。现在它能够计算多个实体查询和基于列的查询。请注意，这意味着如果您没有任何关联准则地说query(A, B).count()，它会计算A和B的笛卡尔积。任何对列的查询，都会自动发出"SELECT count(1) FROM (SELECT...)"，以便返回实际的行数，这意味着查询如query(func.count(A.name)).count()将返回1的值，因为该查询将返回一行。

    .. change::
        :tags: orm
        :tickets: 

      优化大量的性能。与0.5.0rc3相比，各种ORM操作的粗略估计增加了10%，与0.4.8相比增加了25-30%。

    .. change::
        :tags: orm
        :tickets: 

      bugfixes和行为变化。

    .. change::
        :tags: general
        :tickets: 

      全局"propigate"->"propagate" change。

    .. change::
        :tags: orm
        :tickets: 

      调整了InstanceState的增强垃圾回收，以更好地防止因丢失状态而产生错误。

    .. change::
        :tags: orm
        :tickets: 1220

      Query.get()在针对多个实体执行时返回更详细的错误消息。

    .. change::
        :tags: orm
        :tickets: 1140, 1221

      恢复了Cls.relation.in_()上的NotImplementedError

    .. change::
        :tags: orm
        :tickets: 1226

      修复PendingDeprecationWarning，该警告涉及relation()的order_by参数。

    .. change::
        :tags: sql
        :tickets: 

      删除了Connection对象的 'properties' 属性，应该使用Connection.info。

    .. change::
        :tags: sql
        :tickets: 

      恢复了一次 "查询前结帐" 获取结果行数的fetch操作，因为ResultProxy现在将使用 total_rowcount。

    .. change::
        :tags: sql
        :tickets: 

      重新排列`TypeDecorator`类中的 `load_dialect_impl()` 方法，使其在实现了其他 `TypeDecorator` 来实现的情况下也会生效。

    .. change::
        :tags: access
        :tickets: 

      增加对 Currency类型的支持。

    .. change::
        :tags: mysql
        :tickets: 

      现在可以反射临时表。

    .. change::
        :tags: sqlite
        :tickets: 968

      重新制定 SQLite `date/time` 的 `bind/result` 处理方案，使用正则表达式和格式字符串，而不是`strptime/strftime`，以便支持通用的前1900的日期、微秒日期。

.. changelog::
    :version: 0.5.0rc3
    :released: Fri Nov 07 2008

    .. change::
        :tags: orm
        :tickets: 

      增加了两个新的 SessionExtension 钩子：after_bulk_delete()和after_bulk_update()。after_bulk_delete()在查询上的批量删除（bulk delete()）操作完成之后调用。after_bulk_update() 在查询上的批量更新（bulk update()）操作完成之后调用。

    .. change::
        :tags: sql
        :tickets: 

      对 SQL 编译器进行了优化和复杂性降低。与0.5.0rc2相比，典型 select() 构建的编译调用次数减少了20％。

    .. change::
        :tags: sql
        :tickets: 1211

      `dialect` 现在可以生成可调整长度的标签名。使用 "label_length=<value>" 参数传递给 create_engine()，以调整动态生成的列标签中将存在多少个字符，例如"somecolumn AS somelabel"。任何小于6的值都会产生最小大小的标签，由一个下划线和一个数字计数器组成。编译器使用dialect.max_identifier_length的值作为默认值。

    .. change::
        :tags: ext
        :tickets: 

      添加一个新的扩展 sqlalchemy.ext.serializer。提供 Serializer/Deserializer “类” ，类似于 Pickle/Unpickle，以及 dumps() 和 loads()方法。这个序列化程序实现了一个“外部对象”Pickler，将重点上下文敏感对象保留在 pickle 流之外，包括 engines、sessions 等metadata，Tables/Columns和mappers，并且可以稍后使用任何engine/metadata/session提供商恢复pickle。这不是为了pickle常规对象实例而建立，这些对象实例是pickleable而无需任何特殊逻辑，而是用于pickle表达式对象和完整查询（Query）对象，这样所有映射引擎/会话/依赖关系都可以在unpickle时恢复。

    .. change::
        :tags: oracle
        :tickets: 

      为了修复OCI错误，元素的值将在传递到底层光标执行语句时转换为Unicode字符串。

    .. change::
        :tags: oracle
        :tickets: 

      修改了 create_xid() 的格式，以恢复两阶段提交。我们现在有来自现场的Oracle两阶段提交的报告工作正常。

    .. change::
        :tags: oracle
        :tickets: 1233

      添加了 OracleNVarchar 类型，生成 NVARCHAR2，也将 Unicode 子类化，以便convert_unicode=True成为默认值。NVARCHAR2会自动反射到此类型中，因此在反射表（reflected table）上没有显式的convert_unicode=True标志时，这些列会在Unicode上传递。

    .. change::
        :tags: mysql
        :tickets: 

      固定了在反射时缺少FK列而引发异常的问题。

    .. change::
        :tags: mysql
        :tickets: 

      MySQLReflection导入该引擎。

    .. change::
        :tags: orm
        :tickets: 

      "not equals" comparisons of simple many-to-one relation to an
      instance不会掉进EXISTS子句中，并且将比较而不是等于外键列。

    .. change::
        :tags: sql
        :tickets: 1212

      简化了ResultProxy“不包含结果的自动关闭”的检查方法，将其仅基于cursor.description的存在属性判断。删除了关于字符串是否在表达式中返回行的正则表达式猜测。

    .. change::
        :tags: mysql
        :tickets: 

      修正了在反射一个具有外键引用该架构(database)的表时的外键问题。

    .. change::
        :tags: mysql
        :tickets: 

      现在没有预期 include_columns 在表反射中需要小写。

    .. change::
        :tags: misc
        :tickets: 1077

      修复 bug，这个 bug 在 pypy 中使用具有 __iter__() 方法的字符串时，将其解释为迭代器。

.. 0.5.0beta3版本

变化:

orm

- “Session”等中的“extension”参数现在可以是列表，支持发送事件给多个SessionExtension实例，“Session”将SessionExtensions放在Session.extensions中。
- 重复调用flush()将会引发错误。这也是一个原始的，但不完美的，防止并发使用Session.flush()的检查。
- 在使用明确的连接标准（即不通过关系连接）时，改进query.join()方法的行为，以连接到继承的子类。
- @orm.attributes.reconstitute和MapperExtension.reconstitute已更名为@orm.reconstructor和MapperExtension.reconstruct_instance。
- 修复了子类继承基类时@reconstructor挂钩的错误。
- composite（）属性类型现在支持具有__set_composite_values__()方法的复合类，该方法是必需的，如果该类使用属性名称而不是列的keynames表示状态，则默认生成的值现在将被填充正确在 flush期间。此外，设置为None的属性的复杂性正确比较。
- attributes.get_history()返回的3个可迭代的元组现在可以是列表和元组的混合物。 （以前成员始终是列表。）
- 处理在Flush()中修改了一个父键的属性的实体，该属性的先前值已过期时会产生错误。
- 修复定制工具中的内在性问题以及针对只由ORM加载的新构造实例工具的get_instance_dict（）未被调用的问题。
- Session.delete()将给定的对象添加到会话中（如果还不存在）。这是0.4中的回归错误。
- 在Session上的“echo_uow”标志已弃用，并且工作单元日志记录现在仅限于应用程序级别，而不是每个会话级别。
- 从InstrumentedAttribute中删除了冲突的“contains()”运算符，该运算符未接受“escape”关键字参数。
- 在具有复合主键引用另一个尚未定义的表的情况下，将无法初始化映射程序的错误修复为bug。
- 修复了使用字符串描述符的主键连接条件与backref结合使用时抛出的异常错误的问题。
- 添加了对MetaData的“sorted_tables”访问器，它按依赖关系顺序返回Table对象列表。这会淘汰MetaData.table_iterator()方法。“reverse = False”关键字参数也已从util.sort_tables（）中删除；使用Python的'reversed'函数来反转结果。
- 所有数字类型的'length'实参现已重命名为'scale'。'length'已弃用，并仍会发出警告。
- 用户定义的类型（convert_result_value、convert_bind_param）已放弃0.3兼容性。
- 暂时回滚了“ORDER BY”增强功能，该功能已暂停等待进一步开发。
- exists()构造不会将其包含的元素“导出”为FROM子句，从而允许它们在SELECT的columns子句中更有效地使用。
- and_()和or_()现在生成ColumnElement，允许将布尔表达式作为结果列，即select([and_(1,0)])。
- Bind params现在是ColumnElement的子类，这使它们可以按orm.query进行选择（它们已经具有大多数ColumnElement的语义）。
- 在exists()结构中添加了select_from()方法，该结构越来越兼容常规select()。
- 添加了作为“通用功能”的func.min（）、func.max（）和func.sum（），它基本上允许自动确定它们的返回类型。有助于SQLite上的日期，十进制类型和其他类型。
- 将decimal.Decimal添加为“自动检测”类型；当使用Decimal时，绑定参数和通用函数将其类型设置为Numeric。
- MSEnum类型的“长度”参数现在已更名为“display_width”。
- 添加了MSMediumInteger类型。
- func.utc_timestamp()功能编译为UTC_TIMESTAMP，不使用括号，这似乎会妨碍与executemany()一起使用时的使用。
- 限制/偏移不再使用ROW NUMBER OVER来限制行，并且改用子查询与特殊优化注释一起使用。允许LIMIT / OFFSET与DISTINCT配合使用。
- 现在，has_sequence()考虑当前的“schema”参数。
- 添加了BFILE到反射类型名称中。

.. 0.5.0beta2版本

变化:

orm

- 在过期属性之外，延迟属性也会在其数据存在于结果集中时加载。
- 当指定的属性列表不包括任何基于列的属性时，session.refresh()将引发信息性错误消息。
- 如果未指定列或映射器，query()将引发信息性错误消息。
- 惰性加载程序现在在继续之前触发自动Flush。这允许在autoflush上下文中正确地进行集合或标量关系的过期（expire()）。
- column_property()属性现在在插入或更新后自动过期，假设它们未被本地修改，以便在访问时使用最新数据进行刷新，表示SQL表达式或映射表中不存在的列（例如来自视图）。。
- class-bound属性现在在Mapper初始化期间检测到并保留在原位。这意味着，如果在路上有同名的table-bound列的@property（并且该列没有映射到不同的名称），则将完全不映射表绑定的列，或者继承类的已简仪属性不会被应用。与包含属性/排除属性集合使用的名称相同的规则适用。
- 添加了新的SessionExtension钩子，称为after_attach()。这是在通过add()，add_all()，delete()和merge()添加对象的时候调用的，point of attachment
- 继承自另一个映射器的映射器，在继承其继承的映射器的列时，将使用该继承的映射器指定的任何重新分配的属性名称。
- 在Session中的一系列潜在竞争况已修正，异步GC可能会从处理列表中删除未修改、不再引用的项，通常在session.expunge_all()和依赖方法中出现。
- 改进了_preprae_instrumentation机制，应减少“Attribute x was not replaced during compile”警告的概率（对于SQLA黑客，例如Elixir的开发人员）。
- 修复了在父上没有任何列的情况下引发的“未保存的待处理实例”FlushError问题，该问题在生成负责引发错误的关系列表时不使用超级类映射程序时不考虑超级类映射程序时产生。
- func.count()没有参数时呈现为COUNT（*），相当于func.count（text（' * '））。
- 对于ORDER BY表达式中的简单标签名称，不会将其自己呈现为其相应表达式的重述，而是自己。目前仅为SQLite、MySQL和PostgreSQL启用此功能。当每个都显示出支持此行为的功能时，它可以在其他方言上启用。
- 用于relation()的remote_side和foreign_keys参数的类绑定访问器现在可接受，从而允许使用声明性语法。此外，修复了在与eager加载结合使用时将order_by指定为类绑定属性时发生的错误。
- 调整了列的初始化，使非重命名列的初始化与非声明性映射器的初始化方式相同。这使得继承映射器可以设置其同名的“id”列，特别是父“id”列优先于子列时，当请求该值时会减少数据库往返。
- MetaData中添加了"sorted_tables"访问器，它将Table对象按依赖关系排序为列表。这淘汰了MetaData.table_iterator()方法。也删除了util.sort_tables()中的"reverse=False"关键字参数；使用Python的"reversed"函数来反转结果。
- Unicode、UnicodeText类型现在默认设置"assert_unicode"和"convert_unicode"，但接受覆盖\**kwargs这些值。
- 添加了一个新的SessionExtension钩子，称为after_attach()。这是在通过add()，add_all()，delete()和merge()附加对象时调用的，point of attachment
- polymorphic_union()函数如果它们与该列的名称不同，则会考虑每个Column的"key"。
- 修复了0.4的问题，该问题防止组合列正常使用继承映射程序
- 也许当ForeignKey对象中使用声明性构造时，重新进入Mapper编译()调用时会产生Deadlock的RLock问题。从0.5移植过来的。 
- 修复了组合类型中防止主键复合类型被改变的错误。
- 添加了ScopedSession.is_active访问器。
- 可以将绑定于类上的访问器用作relation() order_by的参数。
- ShardedSession.execute()中的shard_id参数的错误已修复。
- Connection.invalidate()检查关闭状态以避免属性错误。
- NullPool在出现故障时支持重新连接行为。
- TypeEngine用于缓存特定于方言的类型的每个dialect的缓存现在是WeakKeyDictionary。这是为了防止Dialect对象因创建任意数量的引擎或dialect而被引用。这有很小的性能损失，将在0.6中解决。
- 修复了SQLite反射方法的问题，以便在最近的pysqlite版本中检测不到不在的光标说明，这会触发auto-cursor close，以便当没有结果时不会失败。
- Postgres添加了反映支持，使用我们长期忽视的伟大修补程序Ken Kuhlman提交。
- 外键列在反射期间未出现时，引发异常错误的异常错误修复。
- 现在Per-dialect缓存用于TypeEngine以缓存方言特定的类型，它们是WeakKeyDictionary。这是为了防止Dialect对象永远引用任何引擎或方言的应用程序。这存在一些小型的性能损失，在0.6中将解决此问题。
- 修复了SQLite Date、DateTime和Time类型仅使用Python日期时间对象，而不使用字符串的问题。如果您希望使用SQLite格式化字符串，请使用String类型。如果您想要它们仍然返回日期时间对象，尽管它们在接受字符串作为输入的情况下，请使用TypeDecorator在String周围 - SQLA不鼓励此模式。 

.. 0.5.0beta1版本

变化:

orm

- 由于装饰器“__init__”触发器现在试图精确映射原始__init__的参数签名。传递给'_sa_session'的传递不再是隐式的-您必须在构造函数中允许此关键字参数。
- ClassState现已更名为ClassManager。
- 类可以通过提供__sa_instrumentation_manager__属性来提供自己的InstrumentationManager。
- 自定义的工具可以使用任何机制将ClassManager与类相关联，并将InstanceState与实例相关联。仍然是默认关联机制使用的对象的属性，ClassBound
- 从实例对象中移除了实体名称_entity_name，_sa_session_id和_instance_key，并将这些值移到instance state中。这些值仍然可以以旧方式使用，这已被弃用，使用附加到类的描述符。在访问时发出警告。
- _prepare_instrumentation的别名已删除。
- sqlalchemy.exceptions已重命名为sqlalchemy.exc。模块可能在任何一个名称下导入。
- ORM相关的异常现在在sqlalchemy.orm.exc中定义。ConcurrentModificationError，FlushError和UnmappedColumnError兼容性别名安装在sqlalchemy.exc中，在导入sqlalchemy.orm时触发警告。
- sqlalchemy.logging已重命名为sqlalchemy.log。
- sqlalchemy.log.SADeprecationWarning的过渡 sqlalchemy.log别名已被删除。
- exc.AssertionError已删除，并替换为Python内置的AssertionError。
- 附加到单个类的多个使用entity_name的主要映射器的MapperExtensions的行为已更改。首先为类定义的第一个映射器才有资格获取MapperExtension'instrument_class'，'init_instance'和'init_failed'事件。这是向后不兼容的；之前，最后一个定义映射器的扩展将接收这些事件。
- Firebird现在支持从插入（仅限2.0 +），更新和删除（仅限2.1 +）中返回值。
- 全局"propigate">"propagate"更改。
- 对于polymorphic_union()函数，如果它们与该列的名称不同，则会考虑每个Column的"key"。
- 改进了在使用声明性构造时发送到relation()的远程_side_和foreign_keys参数的类限制访问器。此外，修复了在与eager加载结合使用时在使用类绑定属性指定order_by时指定的错误。
- 更改了列的初始化，使非重命名列的初始化与非声明映射器的初始化方式相同。这使得继承映射器可以设置其同名的"id"列，特别是父“id”列优先于子列时，当请求该值时会减少数据库往返。
- 继承多个，entity_name= primary mappers for a single class已更改MapperExtension's instrument_class'，'init_instance'和'init_failed'事件的行为。为类定义的第一个映射器才有资格。这是向后不兼容的；之前，最后一个定义映射器的扩展将接收这些事件。
- 在函数func.count()不带参数的功能中，呈现为COUNT(*)，相当于func.count(text('*'))。
- 修复了SQLite反射方法的问题，以便在最近的pysqlite版本中检测不到不存在的cursor.description，这会触发auto-cursor close，以便当没有结果时不会失败。 

