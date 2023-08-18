=============
0.4 变更日志
=============

                
.. changelog::
    :version: 0.4.8
    :released: Sun Oct 12 2008

    .. change::
        :tags: orm
        :tickets: 1039

        修复了关于 inherit_condition 参数传递问题的 bug，现在传递“ A=B ”与 “ B=A ”时不会导致错误。

    .. change::
        :tags: orm
        :tickets: 

        更改了 SessionExtension.before_flush() 中的 new，dirty 和 deleted 集合，现在这些更改将会对该 flush 所执行的操作生效。

    .. change::
        :tags: orm
        :tickets: 

        在 InstrumentedAttribute 中添加了 label() 方法，以便与 0.5 版本进行兼容。

    .. change::
        :tags: sql
        :tickets: 1074

        现在可以在不将子查询传给 FROM 子句的情况下将 column.in_(someselect) 用作列子句表达式。

    .. change::
        :tags: mysql
        :tickets: 1146

        新增 MSMediumInteger 类型。

    .. change::
        :tags: sqlite
        :tickets: 968

        提供了一个自定义的 strftime() 函数，用于处理 1900 年之前的日期。

    .. change::
        :tags: sqlite
        :tickets: 

        现在 sqlite dialect 会禁用 String、Unicode、UnicodeText 等的 convert_unicode 功能，以调整到 pysqlite 2.5.0 的新需求，即只接受 Python unicode 对象。

    .. change::
        :tags: oracle
        :tickets: 1155

        现在 has_sequence() 方法会考虑模式名。

    .. change::
        :tags: oracle
        :tickets: 1121

        在反射类型列表中添加了 BFILE。

.. changelog::
    :version: 0.4.7p1
    :released: Thu Jul 31 2008

    .. change::
        :tags: orm
        :tickets: 

        在 scoped_session 中新增了“add()”和“add_all()”方法，可解决 0.4.7 版本的问题： ::

            from sqlalchemy.orm.scoping import ScopedSession, instrument

            setattr(ScopedSession, "add", instrument("add"))
            setattr(ScopedSession, "add_all", instrument("add_all"))

    .. change::
        :tags: orm
        :tickets: 

        修复了在 relation() 中使用 set() 和 generator expression 时不能与 2.3 兼容的问题。

.. changelog::
    :version: 0.4.7
    :released: Sat Jul 26 2008

    .. change::
        :tags: orm
        :tickets: 1058

        在 many-to-many 中使用 contains() 运算符时，将会为二次联接（关系）表别名，这样多次 contains() 调用就不会相互冲突。

    .. change::
        :tags: orm
        :tickets: 

        修复了防止 merge() 与一个 comparable_property() 一起使用时无法使用的错误。

    .. change::
        :tags: orm
        :tickets: 

        现在 relation() 中的 enable_typechecks=False 设置仅允许继承映射器的副本。没有关系的类型，或不针对目标映射器进行映射器继承设置的子类型仍然不允许。

    .. change::
        :tags: orm
        :tickets: 976

        在 Sessions 中添加了 is_active 标志，以检测事务是否正在进行中。对于“transactional”（在 0.5 中不是“autocommit”）Session，此标志始终为 True。

    .. change::
        :tags: sql
        :tickets: 

        修复了调用 select([literal（'foo'）]）或 select([bindparam('foo')]) 时的错误。

    .. change::
        :tags: schema
        :tickets: 571

        如果表名或架构名包含超过该方言配置的字符限制，则 create_all()，drop_all()，create() 和 drop() 将引发错误。一些数据库在使用中可以处理过长的表名，SQLA 也可以处理，但由于需要在 DB 的目录表中查找名称，因此各种反射/checkfirst-during-create 方案会失败。

    .. change::
        :tags: schema
        :tickets: 571、820

        将“index=True”用于 Column 时生成的索引名称将截断为适合方言的长度。此外，如果索引名称过长，则不能直接使用 Index.drop() 显式删除索引。

    .. change::
        :tags: postgres
        :tickets: 

        修复了 server_side_cursors 无法正确检测 text() 子句的问题。

    .. change::
        :tags: postgres
        :tickets: 1092

        添加 PGCidr 类型。

    .. change::
        :tags: mysql
        :tickets: 

        在返回结果行的 SQL 关键词中添加“CALL”。

    .. change::
        :tags: oracle
        :tickets: 

        get_default_schema_name() 现在在返回之前对名称进行“标准化”，这意味着对于检测为大小写不敏感标识符的标识符，它返回小写名称。

    .. change::
        :tags: oracle
        :tickets: 709

        创建/删除表时会考虑模式名称，以便在搜索现有表时不与其他所有者名称空间中具有相同名称的表冲突。

    .. change::
        :tags: oracle
        :tickets: 1062

        Cursor 现在默认情况下 arraysizes 设置为 50，可以使用 Oracle dialect 中的 "arraysize" 参数对其进行配置，此参数的值也可以通过 create_engine() 进行配置。这是为了配合 BLOB/CLOB-boundcursors 使用的，默认提供了任意数量的可用的，但时限只有那个行请求的（因此仍需要 BufferedColumnRow，但不那么需要）。

    .. change::
        :tags: oracle
        :tickets: 

        sqlite)>
        - 添加 SLFloat 类型，与 SQLite REAL 类型 affinity 匹配。以前仅提供了 SLNumeric，它符合 NUMERIC 关联，但与 REAL 不同。

.. changelog::
    :version: 0.4.6
    :released: Sat May 10 2008

    .. change::
        :tags: orm
        :tickets: 

        修复了与 relation() 的最近重构有关的 bug，可以解决在本地和远程表之间多次连接使用的奇异视图，共享在连接之间共享的公共列的关系。

    .. change::
        :tags: orm
        :tickets: 

        还恢复了跨多个表连接的视图意图配置。

    .. change::
        :tags: orm
        :tickets: 610

        添加了一个 experiment relation() 标志，用于帮助跨函数等的 primaryjoins，_local_remote_pairs=[tuples]。这与复杂的 primaryjoin 条件相辅相成，允许您提供构成关系的本地和远程侧的个体列对。还改进了可懒惰加载 SQL 生成，以处理将绑定参数放置在函数和其他表达式中。

    .. change::
        :tags: orm
        :tickets: 1036

        重新修复了 single table inheritance，现在您可以将单表继承从具有连接表继承映射器的继承映射器一次性继承而来的内容。

    .. change::
        :tags: orm
        :tickets: 1027

        修复了“连接元素存在于目标但不存在于合并的集合中的元素未被从目标中移除”的错误。

    .. change::
        :tags: orm
        :tickets: 

        允许对具有继承映射器的属性使用 synonym()。

    .. change::
        :tags: orm
        :tickets: 

        修复了“exists”函数（涉及继承的任何() 、 has() 、~contains()）涉及继承时涉及完整的目标联接时存在的错误都将呈现在 EXISTS 子句中。

    .. change::
        :tags: orm
        :tickets: 

        对于处于挂起状态的实例上的属性过期时，当触发“refresh”操作并且未找到任何结果时，将不会引发错误。

    .. change::
        :tags: orm
        :tickets: 

        Session.execute 现在可以从元数据中查找绑定。

    .. change::
        :tags: orm
        :tickets: 

        将“自我关联”定义调整为具有公共父级的任何两个映射器（这会影响到查询是否需要使用 aliased=True 加入查询）。

    .. change::
        :tags: orm
        :tickets: 

        对于 join() 中的 from_joinpoint 参数进行了一些修复，以便在先前的连接具有别名而此连接没有别名的情况下仍能成功加入。

    .. change::
        :tags: orm
        :tickets: 895

        进行了各种“级联删除”修复：
        - 修复了 dynamic relations“级联删除”操作，该操作仅针对 0.4.2 中的 foreign-key nulling 行为进行了实现，而不是实际的级联删除。
        - 在一个 many-to-one 上，如果没有将 delete cascade 和 delete-orphan cascade 混合使用，则不会删除在父对象上调用 session.delete() 之前已从父级分离的孤儿（one-to-many 已经实现了这一点）。
        - 和父对象分离后是否仍然附加到已删除的父对象身上，delete-orphan 级联将删除孤儿。
        - 在使用 inheritance 时，将正确检测出具有父类中存在的关系的关系。

    .. change::
        :tags: orm
        :tickets: 

        修复了 Query.order_by() 在使用 select_from() 时正确别名 mapper-config'ed order_by 的计算。

    .. change::
        :tags: orm
        :tickets: 

        对更换一个集合到另一个集合中的 diffing logic 进行了重构，现在可以使用 collections.bulk_replace 进行 collections 处理。

    .. change::
        :tags: orm
        :tickets: 

        cascade 遍历算法从 recursive 改成了 iterative，以支持具有深层对象图。

    .. change::
        :tags: sql
        :tickets: 999

        现在，使用架构限定表时，在所有列表达式以及生成列标签时，schemaname 将出现在 tablename 前。这可以在所有情况下防止跨模式名称出现冲突。

    .. change::
        :tags: sql
        :tickets: 

        现在可以允许查询选择关联所有 FROM 子句并且没有自己的 FROM 的语句。这些通常用于标量上下文中，即 SELECT x，（SELECT x WHERE y）FROM table。需要使用 correlate() 调用。

    .. change::
        :tags: sql
        :tickets: 1014

        改进了 text() 表达式用作 FROM 子句时的行为，例如 select().select_from(text("sometext"))。

    .. change::
        :tags: sql
        :tickets: 1021

        Column.copy() 现在会遵循“autoincrement”值，解决了其与 Migrate 的使用问题。

    .. change::
        :tags: engine
        :tickets: 

        现在可以将 Pool 监听器提供为可调用字典或可标识为 PoolListener 的部分鸭式类型对象，任你选择。

    .. change::
        :tags: engine
        :tickets: 

        添加“rollback_returned”选项，以在返回连接时禁用回滚（在 MySQL/MyISAM 中不安全）。

    .. change::
        :tags: ext
        :tickets: 

        set-based association proxies 的 \|=, -=, ^= 和 &= 现在更严格，只能操作集合、frozenset 或集合类型的子类。以前它们会接受任何 duck-typed 集合。

    .. change::
        :tags: mssql
        :tickets: 1005

        在 engine/dburi 参数中新增了“odbc_autotranslate”参数。任何给定字符串都将通过 ODBC 连接字符串作为“AutoTranslate=%s”传递。

    .. change::
        :tags: mssql
        :tickets: 

        在 engine/dburi 参数中添加了“odbc_options”参数。给定字符串将简单地添加到 SQLAlchemy 生成的 odbc 连接字符串中。
        
        这可以避免将来添加需要大量 ODBC 选项的情况。

    .. change::
        :tags: firebird
        :tickets: 

        处理“SUBSTRING(:string FROM :start FOR :length)”内置函数。

.. changelog::
    :version: 0.4.5
    :released: Fri Apr 04 2008

    .. change::
        :tags: sql
        :tickets: 975

        现在可以创建与文本 FROM 子句选择的别名。

    .. change::
        :tags: sql
        :tickets: 

        可以使用 bindparam() 的值为回调函数，在语句执行时将其评估为获取值。

    .. change::
        :tags: sql
        :tickets: 978

        在结果集获取时添加了异常包装/重新连接支持。重新连接适用于那些在结果过程中引发可捕获数据错误的数据库（即不适用于 MySQL）。

    .. change::
        :tags: sql
        :tickets: 

        通过 engine.begin_twophase()、engine.prepare() 实现了“threadlocal” engine 的两阶段 API。

    .. change::
        :tags: sql
        :tickets: 986

        修复了 UNIONS 不可克隆的 bug。

    .. change::
        :tags: sql
        :tickets: 

        对于 insert()、update()，现在可以使用关键字参数。

这篇文档记录了SQLAlchemy版本0.4.2的更新内容。更新主要涉及两个方面：SQL和ORM。

在SQL方面，加入了一些通用函数，如current_timestamp、coalesce等，开发人员可以通过这些函数在特定的数据库中执行相应语句。另外还新增了String和create_engine()方法的assert_unicode标志，用于避免unicode操作中传入非unicode字节串。在生成“唯一”的绑定参数方面，SQLAlchemy现在使用了与其他标识符相同的“唯一标识符”机制生成绑定参数，形如“<paramname>_<num>”。

在ORM方面，修复了一些bug，使得ORM更加稳定和完善。新增功能主要集中在Query对象上，如yield_per()方法、query.yield_per(<number of rows>)。这个方法允许我们在迭代上下文中基于特定结果数返回数据。所有已加载的集合都将在结果批次边界处被清除出去。此外，关联代理列表还新增了'+', '*', ' += '和'* ='等方法。同时加入了schema.DDL()、useexisting标志等新功能。其他方面的更新如扩充了已知SQL函数数据库、断开连接检测等。在列标签中支持“tablename.columname”的形式，即点号。

select（）中的from_obj关键字参数可以是标量或列表。

集合反向引用的主要行为变更：它们不再触发延迟加载！“reverse”添加和删除已排队，并在实际从集合读取和加载时与集合合并；但不会在此之前触发加载。对于注意到此行为的用户，在某些情况下，这应该比在某些情况下使用动态关系方便得多。对于没有注意到此行为的用户，您可能会注意到在某些情况下使用的查询要少得多。

可变主键支持已添加。主键列可以自由更改，并且在刷新时实例的标识将更改。此外，通过在关系上设置标志“passive_cascades = False”，还支持外键引用（主键或其他）的更新级联，或者与数据库中的ON UPDATE CASCADE（如Postgres等需要）一起直接由ORM发出UPDATE语句。

继承映射器现在会直接继承其父映射器的MapperExtensions，以便所有特定MapperExtension的方法也为子类调用。如往常一样，任何MapperExtension都可以返回EXT_CONTINUE以继续扩展处理或EXT_STOP以停止处理。映射器解析的顺序是：<在类的映射器上声明的扩展><在类的父映射器上声明的
扩展><全局声明的扩展>。注意，如果你分别实例化相同的扩展类并分别将其应用于继承链中的两个映射器，那么该扩展将应用于继承类两次，并且每个方法将被调用两次。要显式地向每个继承类应用MapperExtension，但每个操作仅调用每个方法一次，请在两个映射器中使用相同的扩展实例。

MapperExtension.before_update（）和after_update（）现在称为对称;以前，具有没有修改的列属性（但具有relation（）修改）的实例可能会使用before_update（）调用，但不使用前者after_update（）

当filter_by（）将关系与None进行比较时，修复了查询中的错误。

查询中缺少的列现在在负载期间会自动推迟处理。

映射的扩展类和不提供__init __（）方法的继承“object”的映射类现在在实例构建时间存在非空*args或**kwargs时（并且未由类scoped_session mapper等扩展名为my_的对象占用时引发TypeError，与正常Python类的行为一致
方法）。

查询.order_by（）考虑到别名加入，例如查询.join（'orders'，aliased = True）.order_by（Order.id）

eagerload（）lazyload（）eagerload_all（）使用第二个类或映射器参数，其将选择要向其应用选项的映射器。这可以选择使用add_entity（）添加的其他映射器。

eagerloading现在可以与通过动态添加的集合使用具有“级联删除”行为的关系，就像普通关系一样。如果未设置persistent_deletes标志（也刚刚添加），则删除父项目将触发完整加载子项，以便它们可以相应地删除或更新。

此外与dynamica，正确的计数（）行为已实现以及其他辅助方法。

修复了在与多态关系上的级联中出现的错误，这样来自对象的级联将继续沿着集合中每个元素特定的属性集连续级联。

查询.get（）和query.load（）不考虑现有的过滤器或其他条件;这些方法始终在数据库中查找给定id或从标识图中返回当前实例，忽略任何已配置的现有过滤器，join，group_by或其他条件

继承映射器现在会直接继承其父映射器的MapperExtensions，以便所有特定MapperExtension的方法也为子类调用。如往常一样，任何MapperExtension都可以返回EXT_CONTINUE以继续扩展处理或EXT_STOP以停止处理。映射器解析的顺序是：<在类的映射器上声明的扩展><在类的父映射器上声明的
扩展><全局声明的扩展>。注意，如果你分别实例化相同的扩展类并分别将其应用于继承链中的两个映射器，那么该扩展将应用于继承类两次，并且每个方法将被调用两次。要显式地向每个继承类应用MapperExtension，但每个操作仅调用每个方法一次，请在两个映射器中使用相同的扩展实例。

可变主键支持已添加。主键列可以自由更改，并且在刷新时实例的标识将更改。此外，通过在关系上设置标志“passive_cascades = False”，还支持外键引用（主键或其他）的更新级联，或者与数据库中的ON UPDATE CASCADE（如Postgres等需要）一起直接由ORM发出UPDATE语句。

关闭（）方法在使用strategy ='threadlocal'时对事务进行修改

fix to compiled bind parameters to not mistakenly populate None

MapperExtension.before_update（）和after_update（）现在称为对称;以前，具有没有修改的列属性（但具有relation（）修改）的实例可能会使用before_update（）调用，但不使用前者after_update（）

到Query加入一个字段（“info”）以在架构项上存储任意数据

PG反射，在已明确使用默认模式名称作为“架构”参数的表中看到默认模式名称时，将假定这是用户的预期约定，并将明确设置“模式”参数.. changelog::
    :version: 0.4.0beta6
    :released: Thu Sep 27 2007

    .. change::
        :tags: 
        :tickets: 808

      修复了BOOL/BOOLEAN在sqlite上的反射问题。

    .. change::
        :tags: 
        :tickets: 

      支持mysql上的UPDATE with LIMIT。

    .. change::
        :tags: 
        :tickets: 803

      修复了m2o中的空外键不触发lazyload的问题。

    .. change::
        :tags: 
        :tickets: 800

      对于oracle来说，不带typeEngine/String/Unicode类型的结果集（即未使用类型引擎/String/Unicode类型），不再自动进行unicode转换（先前会检测DBAPI类型并进行转换）。

    .. change::
        :tags: 
        :tickets: 806

      修复了匿名标签长表/列名生成问题。

    .. change::
        :tags: 
        :tickets: 

      Firebird dialect现在使用SingletonThreadPool作为池类。

    .. change::
        :tags: 
        :tickets: 

      Firebird现在使用dialect.preparer来格式化序列名称。

    .. change::
        :tags: 
        :tickets: 810

      修复了postgres和多个两阶段事务之间的问题，两阶段提交和回滚不会像dbapi提交/回滚那样自动以新事务结束。

    .. change::
        :tags: 
        :tickets: 

      新添加了一个_SCopedExt mapper extension选项，用于在对象初始化时不自动将新对象保存到session中。

    .. change::
        :tags: 
        :tickets: 

      修复了Oracle非ansi join语法。

    .. change::
        :tags: 
        :tickets: 

      对于不支持的DbType.PickleType和DbType.Interval类型，现在表现得稍微快一些。

    .. change::
        :tags: 
        :tickets: 

      将Float和Time类型添加到Firebird（FBFloat和FBTime），并修复TEXT和Binary类型的BLOB SUB_TYPE。

    .. change::
        :tags: 
        :tickets: 

      更改了in\_操作符的API。现在in\_()只接收一个序列值或一个可选择的值，旧API使用varargs添加值的方式仍然可用。

.. changelog::
    :version: 0.4.0beta6
    :released: Thu Sep 27 2007

    .. change::
        :tags: 
        :tickets: 

      默认情况下，Session标识映射现在使用弱引用而不是常规字典。使用weak_identity_map=False来使用常规字典。弱引用字典已经定制好，可以检测到“脏”实例，并在更改被刷新之前维护对这些实例的临时强引用。

    .. change::
        :tags: 
        :tickets: 758

      Mapper编译已经重构，大多数编译都是在mapper构造时完成的。这使得我们可以减少调用mapper.compile()，也可以让基于类的属性强制进行编译（即User.addresses==7将编译所有映射器，这是有效的）。唯一的注意事项是，一个继承映射器在构造时现在会寻找其继承映射器；因此，引用关系中的映射器需要按继承顺序构造（这应该是正常情况）。

    .. change::
        :tags: 
        :tickets: 

      添加了“FETCH”关键字以及postgresql检测结果行持有语句（即除“SELECT”之外的语句）的全部列表。

    .. change::
        :tags: 
        :tickets: 

      完全列出SQLite保留关键字，以便正确转义它们。

    .. change::
        :tags: 
        :tickets: 

      查询的“eager load”别名生成和Query.instances()之间的关系得到了加强。如果别名没有由EagerLoader特定地为该语句生成，则EagerLoader在获取急切加载行时将不起作用。它可以防止意外捕获列，作为一个很好的例子，尤其是因为“匿名别名”的列使用简单的整数计数来生成标签。

    .. change::
        :tags: 
        :tickets: 

      从clauseelement.compile()中删除“parameters”参数，用“column_keys”代替。传递给execute()的参数只与当前存在但不在该参数中的列名称相关。更一致的execute/executemany行为，但在内部有点简化。

    .. change::
        :tags: 
        :tickets: 560

      为PickleType添加了“comparator”关键字参数。默认情况下，“可变”PickleType使用其dumps（）表示形式对对象进行“深度比较”。但是对于字典，这种方法并不起作用。提供充分__eq__()实现的Pickled对象可以使用“PickleType（comparator=operator.eq）”设置。

    .. change::
        :tags: 
        :tickets: 

      session.is_modified（obj）方法已添加；执行与刷新操作中发生的“历史记录”比较操作相同；将include_collections=False设置为使用在flush确定是否对实例的行发出更新的相同结果。

    .. change::
        :tags: 
        :tickets: 584, 761

      为Sequence添加“schema”参数，用于在Postgres / Oracle上使用该序列位于备用模式中的情况。这部分实现了，应该会修复。

    .. change::
        :tags: 
        :tickets: 

      修复了mysql枚举类型的空字符串反射。

    .. change::
        :tags: 
        :tickets: 794

      将MySQL dialect从LIMIT <limit> OFFSET <offset>语法更改为较旧的LIMIT <offset>，<limit>语法，以适用于使用3.23的用户。

    .. change::
        :tags: 
        :tickets: 

      在relation()中添加了“passive_deletes =“ all ””标志，禁用在父对象被删除时所有空外键属性的清零。

    .. change::
        :tags: 
        :tickets: 

      为实现inline的列默认值和onupdate添加了括号，用于子查询和其他需要括号的表达式。

    .. change::
        :tags: 
        :tickets: 793

      关于String/Unicode类型，当没有长度时将它们自动转换为TEXT/CLOB的行为现在仅适用于没有参数的String或Unicode的确切类型。如果使用没有长度的VARCHAR或NCHAR（String/Unicode的子类），则会被方言视为VARCHAR / NCHAR，不会发生任何“魔法”转换。这是更少想要的行为，特别是这有助于Oracle将基于字符串的绑定参数保持为VARCHAR，而不是CLOB。

    .. change::
        :tags: 
        :tickets: 771

      修复ShardedSession与延期列一起使用的问题。

    .. change::
        :tags: 
        :tickets: 

      user-defined shard_chooser（）函数必须接受“clause=None”参数；这是传递给session.execute（statement）函数的ClauseElement并可用于确定正确的分片ID（因为execute（）不采用实例）。

    .. change::
        :tags: 
        :tickets: 764

      将NOT的运算符优先级调整为与'=='等运算符相同，以便“~（x <operator> y）”产生“NOT（x <op> y）”。这更好地与旧版本的MySQL兼容..不适用于“~（x==y）”，因为（x==y）将编译为“x！=y”，但仍适用于BETWEEN之类的运算符。

    .. change::
        :tags: 
        :tickets: 757, 768, 779, 728

      其他票证：..。

.. changelog::
    :version: 0.4.0beta5
    :released: 

    .. change::
        :tags: 
        :tickets: 

      连接池修复；beta4的更好性能仍然存在，但修复了存在的“连接溢出”和其他错误，比如。

    .. change::
        :tags: 
        :tickets: 769

      修复确定自定义继承条件的正确同步子句的错误。

    .. change::
        :tags: 
        :tickets: 763

      扩展'engine_from_config' coercion以支持QueuePool大小/溢出。

    .. change::
        :tags: 
        :tickets: 748

      现在可以重新反射mysql视图。

    .. change::
        :tags: 
        :tickets: 

      AssociationProxy现在可以采用自定义的getter和setter。

    .. change::
        :tags: 
        :tickets: 

      修复orm查询中的BETWEEN故障。

    .. change::
        :tags: 
        :tickets: 762

      修复OrderedProperties pickling。

    .. change::
        :tags: 
        :tickets: 

      现在SQL表达式默认值和序列将在INSERT或UPDATE期间“内联”执行，以及在executemany（）样式调用期间执行所有列。 inline=True标志也会通过单个execute（）强制相同的行为。result.postfetch_cols（）是一个列的集合，这些列包含上一个单个插入或更新语句包含SQL端默认表达式的其余列。

    .. change::
        :tags: 
        :tickets: 759

      修复PG executemany（）行为。

    .. change::
        :tags: 
        :tickets: 

      postgres为没有默认值的主键列反射autoincrement=False。

    .. change::
        :tags: 
        :tickets: 

      postgres不再将executemany（）包装在单个execute（）调用中，而是更加注重性能。与Deleted项（使用executemany）的“rowcount” /“concurrency”检查被禁用，因为psycopg2不能为executemany（）报告正确的rowcount。

    .. change::
        :tags: tickets, fixed
        :tickets: 742

      

    .. change::
        :tags: tickets, fixed
        :tickets: 748

      

    .. change::
        :tags: tickets, fixed
        :tickets: 760

      

    .. change::
        :tags: tickets, fixed
        :tickets: 762

      

    .. change::
        :tags: tickets, fixed
        :tickets: 763

      

.. changelog::
    :version: 0.4.0beta4
    :released: Wed Aug 22 2007

    .. change::
        :tags: 
        :tickets: 

      整理了“from sqlalchemy import *”命名空间中出现的内容：：

    .. change::
        :tags: 
        :tickets: 

      不再导入“table”和“column”。它们通过直接引用（如“sql.table”和“sql.column”）或通过sql包的全局导入（即星号）保留。这样做的原因是刚开始使用SQLAlchemy时容易意外使用sql.expressions.table，以及column。

    .. change::
        :tags: 
        :tickets: 

      其他类似于ClauseElement、FromClause、NullTypeEngine等的类也不再导入到命名空间中。

    .. change::
        :tags: 
        :tickets: 

      对于映射器中的列名称，默认情况下不导入以小写字母i命名的“Smallinteger”，但现在仍保留在schema.py中。SmallInteger（大写i！）仍然被导入。

    .. change::
        :tags: 
        :tickets: 

      连接池在内部使用“threadlocal”策略，以便为“上下文”连接返回已绑定到线程的连接；这些是在没有连接的情况下使用的连接，例如insert()。execute()。这就像“部分”版本的“threadlocal”引擎策略，但没有线程本地的交易部分。我们希望它可以减少连接池开销以及数据库使用情况。但是，如果证明对稳定性产生负面影响，我们将立即回退。

    .. change::
        :tags: 
        :tickets: 

      修复绑定参数处理，以使“False”值（如空字符串）仍然得到处理/编码。

    .. change::
        :tags: 
        :tickets: 752

      修复了select()“generative”行为，使使用列，select_from()，correlate()和with_prefix()不会修改原始select对象。

    .. change::
        :tags: 
        :tickets: 

      添加了“legacy”适配程序到类型中，使得定义了convert_bind_param()和/或convert_result_value()的用户定义的TypeEngine和TypeDecorator类仍然可以正常工作。还支持调用这些方法的超级（）版本。

    .. change::
        :tags: 
        :tickets: 

      添加了session.prune()函数，修剪session中不再被其他地方引用的实例。其作用类似于强引用标识映射。

    .. change::
        :tags: 
        :tickets: 

      Transaction现在具有close()方法。如果是外部事务，则使用回滚结束事务，否则直接结束。

    .. change::
        :tags: 
        :tickets: 

      支持从mapper扩展到路径load（即user.addresses.notes等）的动态关系内容已经添加；这是一个可写的集合（支持append()和remove（）），同时当它被读取时，加载了完全活跃的查询对象。在处理非常大的集合时很理想，其中仅需要部分加载。

    .. change::
        :tags: 
        :tickets: 

      在flush（）中嵌入的inline INSERT / UPDATE表达式。将任何SQL表达式（如“sometable.c.column +1”）分配给实例的属性。在flush（）期间，映射器检测到表达式并将其直接嵌入INSERT或UPDATE语句中；该属性被延迟在实例上，因此在下一次访问时加载新值。

    .. change::
        :tags: 
        :tickets: 618

      引入了一个基本的分片（水平扩展）系统。该系统使用一个修改过的Session，该Session可以根据用户定义的函数定义的“分片策略”在多个数据库中分发读写操作。基于属性值、轮询方法或任何其他用户定义的系统，可以将实例及其从属项分别分布并查询到多个数据库中。

    .. change::
        :tags: 
        :tickets: 659

      已增强eager loading，以允许更多的连接在更多的位置。现在可以在任意自我参照和循环结构深度上运作。在加载循环结构时，请在relation()上指定“join_depth”指示您想要表加入自身的次数；每个级别都有一个不同的表别名。现在，别名名称使用简单的计数方案在编译时生成，并且对于肉眼来说更容易看到，而且当然是完全确定性的。

    .. change::
        :tags: 
        :tickets: 211

      添加了复合列属性。这允许在使用ORM时创建由多个列表示的类型。具有新类型的对象在查询表达式、比较、查询.get（）子句等方面是完全可用的，并且表现得好像它们是常规的单列标量...但它们不是！在mapper的“properties”字典中使用函数composite(cls，*columns），并将cls的实例创建/映射到单个属性，由*columns对应的值组成。

    .. change::
        :tags: 
        :tickets: 

      为具有相关子查询的自定义column_property（）属性增强支持，现在与eager loading表现更好。

    .. change::
        :tags: 
        :tickets: 611

      主键“折叠”行为；映射器会分析给定可选择的所有列，以查找通过外键关系或通过显式的inherit_condition等效的主键。主要用于加入表继承场景，其中不同命名的PK列在继承表中应折叠为单值（或更少值）主键。修复这样的问题。``joined-table``继承现在将只会在连接的根表中生成所有继承类的主键列。这意味着根表中的每一行只与单个实例对应。如果出于某种少见原因不希望这样，单个映射器上的显式``primary_key``设置将覆盖它。

.. change::
    :tags: orm
    :tickets: 

    当将“polymorphic”标志与连接表或单表继承一起使用时，所有身份标识键都是针对继承层次结构的根类生成的；这使得使用与非多态获取相同的缓存语义的``query.get()``函数可以实现多态获取。请注意，这目前无法与具体的继承一起使用。

.. change::
    :tags: orm
    :tickets: 

    *在不使用select_table参数的情况下*可以构建多态映射器，继承映射器的表未包含在初始加载中，将会针对每个实例（即针对大型列表不太高效）立即发出第二个SQL查询，以加载其余的列。

.. change::
    :tags: orm
    :tickets: 

    辅助继承加载还可以通过“polymorphic_fetch”参数将其第二次查询移入列级别的“延迟”加载，并且可以将其设置为“select”或“deferred”。

.. change::
    :tags: orm
    :tickets: 696

    现在可以通过使用``include_columns/exclude_columns``仅将可用可选择列的子集映射到映射器属性。

.. change::
    :tags: orm
    :tickets: 

    添加``undefer_group()``MapperOption，将一组由“group”连接的“deferred”列设置为“undeferred”以加载。

.. change::
    :tags: orm
    :tickets: 

    重写了“确定性别名”逻辑，使其成为SQL层的一部分，产生更简单的别名和标签名称，更符合Hibernate的风格。

.. change::
    :tags: sql
    :tickets: 

    速度！语句构建和编译的机制已经大幅简化和优化，比0.3版的语句构建和编译开销提高了20-30%。

.. change::
    :tags: sql
    :tickets: 

    所有“type”关键字参数（例如传递给``bindparam()``, ``column()``, ``Column()``和``func.<something>()``的关键字参数）都已重命名为“type\_”。这些对象仍将其“type”属性命名为“type”。

.. change::
    :tags: sql
    :tickets: 

    从模式元素中删除了case_sensitive=(True|False)设置，因为检查此状态会添加大量方法调用开销，而且几乎没有合理的原因将其设置为False。 所有全小写的表和列名称都将被视为不区分大小写（是的，我们也会调整Oracle的大写样式）。

.. change::
    :tags: transactions
    :tickets: 

    添加了对事务的上下文管理器（即使用``with``语句的支持）。

.. change::
    :tags: transactions
    :tickets: 

    添加了对两阶段提交的支持，目前可以在mysql和postgres中使用。

.. change::
    :tags: transactions
    :tickets: 

    添加了使用保存点的子事务实现。

.. change::
    :tags: transactions
    :tickets: 

    添加了对保存点的支持。

.. change::
    :tags: metadata
    :tickets: 

    可以在不提前声明表的情况下从数据库信息中一次性反射表。MetaData(engine, reflect=True)将加载数据库中存在的所有表或使用metadata.reflect()进行更好的控制。

.. change::
    :tags: metadata
    :tickets: 

    DynamicMetaData已被重命名为ThreadLocalMetaData。

.. change::
    :tags: metadata
    :tickets: 

    ThreadLocalMetaData构造函数现在不需要参数。

.. change::
    :tags: metadata
    :tickets: 

    移除了BoundMetaData - 普通的MetaData等效。

.. change::
    :tags: metadata
    :tickets: 646

    Numeric和Float类型现在具有“asdecimal”标志；Numeric默认为True，Float默认为False。当为True时，值将作为decimal.Decimal对象返回；当为False时，值将作为float()返回。True / False的默认值已是PG和MySQL的DBAPI模块的行为。

.. change::
    :tags: metadata
    :tickets: 475

    删除了所有硬编码运算符的实现，并将它们移入编译；允许更大的运算符编译灵活性；例如，在字符串环境中使用时，“+”编译为“||”，或者在MySQL上编译为“concat(a,b)”，而在数字环境中则编译为“+”。修复了一些问题。

.. change::
    :tags: metadata
    :tickets: 

    生成在SQL编译时的“匿名”别名和标签名称的完全确定性方式...不再使用随机的十六进制ID。

.. change::
    :tags: metadata
    :tickets: 

    SQL元素（ClauseElement）的重大架构改进。所有元素都共享一个通用的“可变性”框架，它允许统一的方法修改元素的值和生成行为。提高了ORM的稳定性，ORM大量使用SQL表达式的突变。

.. change::
    :tags: metadata
    :tickets: 

    ``select()``, ``union()``现在具有“生成”行为。 ``order_by()``和``group_by()``等方法将返回一个*新的实例* - 原始实例保持不变。非生成方法也依然存在。

.. change::
    :tags: metadata
    :tickets: 569, 52

    在选择子查询或联合查询的内部，所有关于“是子查询”和“相关性”的决策推到了SQL生成阶段。 ``select()``元素现在永远不会被其封装容器或任何方言的编译过程修改。

.. change::
    :tags: metadata
    :tickets: 

    ``select(scalar=True)``参数已弃用；请使用 ``select(..).as_scalar()``。生成的对象服从完整的*column*接口，并能在表达式内更好地使用。

.. change::
    :tags: metadata
    :tickets: 504

    添加了``select().with_prefix('foo')``，允许将任何一组关键字放置在该SELECT的列子句之前。

.. change::
    :tags: metadata
    :tickets: 686

    向row[<index>]添加了数组切片支持。

.. change::
    :tags: metadata
    :tickets: 

    结果集现在会更好地尝试将出现在cursor.description中的DBAPI类型与方言定义的TypeEngine对象匹配，然后用于结果处理。请注意，这仅对文本SQL生效；构造的SQL语句始终具有显式的类型映射。

.. change::
    :tags: metadata
    :tickets: 

    CRUD操作的结果集立即关闭其底层游标，并且如果定义了操作，还会自动关闭连接；这允许在使用历史CRUD操作的连接时更有效地使用连接，同时减少了“挂起连接”的可能性。

.. change::
    :tags: metadata
    :tickets: 559

    列默认值和onupdate Python函数（即传递给ColumnDefault的函数）可以接受零个或一个参数；一个参数是ExecutionContext，您可以从中调用“context.parameters[someparam]”以访问附加到语句的其他绑定参数值。执行所使用的连接也可用，以便您可以预执行语句。

.. change::
    :tags: metadata
    :tickets: 

    为序列添加了“显式”的创建/删除/执行支持（即可以在Sequence的每个方法上传递“可连接”）。

.. change::
    :tags: metadata
    :tickets: 

    当操作模式时，标识符更好地进行了引用。

.. change::
    :tags: metadata
    :tickets: 

    在反射表时标准化了类型无法定位的行为；使用NullType进行替换，并发出警告。

.. change::
    :tags: metadata
    :tickets: 606

    表上的列集合（即表的'c'属性）遵循字典的“__contains__”语义。

.. change::
    :tags: engines
    :tickets: 

    速度！结果处理和绑定参数处理的机制已经进行了全面敲门、简化和优化，以尽可能少地发出方法调用。批量INSERT和批量行集迭代的性能测试都表明0.4版本比0.3版本快两倍以上，使用的函数调用比例减少了68%。

.. change::
    :tags: engines
    :tickets: 

    您现在可以连接到池生命周期并在新的每个DBAPI连接，池检出和检入时运行SQL语句或其他逻辑。

.. change::
    :tags: engines
    :tickets: 

    连接现在具有.properties集合，其中的内容限定为与基础DBAPI连接的生存期相同。

.. change::
    :tags: engines
    :tickets: 

    从池中删除了auto_close_cursors和disallow_open_cursors参数；这减少了开销，因为游标通常由ResultProxy和Connection关闭。

.. change::
    :tags: extensions
    :tickets:

    proxyengine已被临时移除，暂时不提供实际可用的替代品。

.. change::
    :tags: extensions
    :tickets: 

    SelectResults已被Query替换。SelectResults/SelectResultsExt仍然存在，但只是为了向后兼容返回稍微修改的查询对象。SelectResults的join_to()方法不再存在，需要使用join()。

.. change::
    :tags: mysql
    :tickets: 

    通过反射加载的表和列名称现在为Unicode。

.. change::
    :tags: mysql
    :tickets: 

    现在支持所有标准列类型，包括SET。

.. change::
    :tags: mysql
    :tickets: 

    现在可以在仅进行一次往返的情况下执行表反射。

.. change::
    :tags: mysql
    :tickets: 

    现在支持ANSI和ANSI_QUOTES sql模式。

.. change::
    :tags: mysql
    :tickets: 

    现在反射索引。

.. change::
    :tags: postgres
    :tickets: 

    添加了PGArray数据类型，用于使用Postgres数组数据类型。

.. change::
    :tags: oracle
    :tickets: 507

    添加了对OUT参数的原始支持；使用``sql.outparam(name, type)``设置OUT参数，就像``bindparam()``一样；执行后，值可以通过``result.out_parameters``字典获得。