=============
0.6 更新日志
=============


.. changelog::
    :version: 0.6.9
    :released: Sat May 05 2012

    .. change::
        :tags: general
        :tickets: 2279

      调整了“importlater”机制，该机制用于内部解决导入周期，使在导入sqlalchemy或sqlalchemy.orm时完成__import__的使用，从而避免应用程序在启动新线程后使用__import__。

    .. change::
        :tags: orm
        :tickets: 2197

      修复了一个错误，即如果针对将多个实体组合在一起的列表达式，则query.join()使用的来源子句将是不一致的。

    .. change::
        :tags: orm, bug
        :tickets: 2310

      修复了在query.get()中非恰当地在布尔上下文中计算映射的用户映射对象的错误评估。

    .. change::
        :tags: orm
        :tickets: 2228

      修复了仅在Python 3中出现的错误，即当持久+挂起对象在flush期间进行排序时，如果持久对象的主键不是单个整数，则会产生非法比较。

    .. change::
        :tags: orm
        :tickets: 2234

      修复了bug，其中query.join()+aliased=True从一个基于连接的结构转换为它自己的relationship()上的join条件，在子表上的连接条件会不适当地将主实体转换为加入的实体。

    .. change::
        :tags: orm
        :tickets: 2287

      修复了一个错误，即mapper.order_by属性将在子查询贪婪负载内的“inner”查询中被忽略。

    .. change::
        :tags: orm
        :tickets: 2215

      修复了该错误，即如果映射类重新定义__hash__()或__eq__()为某些非标准内容，则这些方法将在该类是“组合”（即非单个实体）结果集的一部分时被查阅，这是一个受支持的用例，因为SQLA不应查询__hash__()或__eq__()。

    .. change::
        :tags: orm
        :tickets: 2188

      修复了一个微妙的错误，导致在发生以下情况时SQL会崩溃：仅对subquery中的column_property()进行操作+joinedload+LIMIT+ order by column property()。

    .. change::
        :tags: orm
        :tickets: 2207

      with_parent产生的join条件以及使用“dynamic”关系同时针对父项的方式，将生成唯一的bindparams，而不是不正确地重复相同的bindparam。

    .. change::
        :tags: orm
        :tickets: 2199

      修复了Query中“无语句条件”的断言，如果在调用from_statement()之后调用了一个生成方法，则会尝试调用该方法。

    .. change::
        :tags: orm
        :tickets: 1776

      Cls.column.collate（“一些排序”）现在可以工作。

    .. change::
        :tags: orm, bug
        :tickets: 2297

      修正了因意外传递元组到session.query()而引发的错误格式。

    .. change::
        :tags: engine
        :tickets: 2317

      回退了在0.7.4中引入的更改，确保在尝试在savepoint和two-phase事务上调用rollback()/prepare()/release()之前连接处于有效状态。

    .. change::
        :tags: sql
        :tickets: 2188

      修复了涉及一个可选择的列的列对应关系的两个微妙错误。 一个是重复标记的子查询，另一个是当标签被“分组”并且丢失自身时。

    .. change::
        :tags: sql
        :tickets:

      修正了bug，即当与某些方言一起使用时，String类型与“对unicode发出警告”的标志会被设置。

    .. change::
        :tags: sql
        :tickets: 2270

      修复了一个错误，即如果传递了可选择，则Select的with_only_columns()方法会失败。但是，在任何情况下，此用例都需要使用0.7。

    .. change::
        :tags: schema
        :tickets:

      当ForeignKeyConstraint引用在父表中未找到的列名时，会添加一个信息性错误消息。

    .. change::
        :tags: postgresql
        :tickets: 2291, 2141

      修复错误，相关原因是在PG 9中出现的相同修改的索引行为影响了重命名列的主键反射。

    .. change::
        :tags: mysql
        :tickets: 2186

      OurSQL方言使用ansi-neutral引号符“'”（而不是'“'”）来进行XA命令。

    .. change::
        :tags: mysql
        :tickets: 2225

      创建表将把COLLATE选项放置在CHARSET之后，这似乎是MySQL关于它是否实际起作用的任意规则的一部分。

    .. change::
        :tags: mssql, bug
        :tickets: 2269

      检索索引名称列表和这些索引中列的名称时，对传入的值进行解码。

    .. change::
        :tags: oracle
        :tickets: 2200

      将ORA-00028添加到断开代码中，使用cx_oracle _Error.code来访问代码。

    .. change::
        :tags: oracle
        :tickets: 2220

      修复了object.RAW类型，该类型没有生成正确的DDL。

    .. change::
        :tags: oracle
        :tickets: 2212

      添加CURRENT到保留字列表中。

    .. change::
        :tags: examples
        :tickets: 2266

      调整了dictlike-polymorphic.py示例，以便应用CAST，使其在PG和其他数据库上工作。

.. changelog::
    :version: 0.6.8
    :released: Sun Jun 05 2011

    .. change::
        :tags: orm
        :tickets: 2144

      如果针对基于列的实体调用query.get()，则无效，此条件现在引发弃用警告。

    .. change::
        :tags: orm
        :tickets: 2151

      非主键映射器将继承主映射器的_identity_class。这样，在继承映射到通常处于继承映射中的类时，会产生与主映射器兼容的标识映射的结果。

    .. change::
        :tags: orm
        :tickets: 2148

      从0.7中导入的identity map实现的修复，它不会在删除时围绕自己使用互斥体。尽管在0.6.7中进行了调整，但某些用户仍会遇到死锁问题。

    .. change::
        :tags: orm
        :tickets: 2163

      修复了在“同步规则无法执行目标列”错误时引发的错误，即“映射器'X'不映射此列”。

    .. change::
        :tags: orm
        :tickets: 2149

      修复了当“自我引用”关系失败且针对其不兼容的超类型时，确定“自我引用”关系失败的bug，其中，超类型具有在联接中没有子子级类别的列。

    .. change::
        :tags: orm
        :tickets: 2153

      mapper()现在将忽略到无关联表的非配置外键，以确定父类和子类之间的遗传条件。这相当于在声明性中已经应用于join()自身的行为。请注意，0.7具有更全面的解决此问题，改变了join()本身确定FK错误的方式。

    .. change::
        :tags: orm
        :tickets: 2171

      修复了当类映射到匿名别名时，如果使用日志记录，则由于别名名称中的未转义的％符号而失败。

    .. change::
        :tags: orm
        :tickets: 2170

      修改了“identity”键在刷新时未检测到时出现的消息文本，以包括常见原因，即未正确设置该列以正确检测自动递增。

    .. change::
        :tags: orm
        :tickets: 2182

      修复了事务级别的“已删除”集合将无法清除已删除状态的expunged对象的错误，如果它们稍后变为瞬态，则会引发一个错误。

    .. change::
        :tags: sql
        :tickets: 2147

      修复了如果column server_onupdate中已传递FetchedValue，则它将不会分配其父“column”的错误。

    .. change::
        :tags: sql
        :tickets: 2167

      修复了嵌套选择的标签，其中它具有另一个标签中的标签，这将产生不正确的导出列。此外，这将破坏针对另一个column_property()的ORM column_property()映射的ORM column_property()映射。

    .. change::
        :tags: engine
        :tickets: 2178

      将RowProxy结果行的__包含__（）方法调整为不会在内部生成任何异常； NoSuchColumnError（）也将生成其消息，无论列构造是否可以强制转换为字符串。

    .. change::
        :tags: postgresql
        :tickets: 2141

      修复了会在列名发生更改的列上反射索引时出错的错误。

    .. change::
        :tags: postgresql
        :tickets: 2175

      圆形数组的某些单元测试修复，MATCH运算符。修复潜在的浮点错误， 对于特定locale，MATCH运算符的某些测试仅在EN位向性语言环境中执行。

    .. change::
        :tags: mssql
        :tickets: 2169

      修复了MSSQL方言中的错误，即表示为模式限定符的别名会泄漏到包含的select语句中。

    .. change::
        :tags: mssql
        :tickets: 2159

      修复了DATETIME2类型在结果集或绑定参数中使用时将在“适应”步骤中失败的错误。此问题在0.7中不存在。

.. changelog::
    :version: 0.6.7
    :released: Wed Apr 13 2011

    .. change::
        :tags: orm
        :tickets: 2087

      收紧了identity映射迭代的互斥体，尝试减少（极少发生的）可重入gc操作导致死锁的机会。在0.7中可能会删除互斥体。

    .. change::
        :tags: orm
        :tickets: 2030

      在"Query.subquery()"中添加了一个"name"参数，以允许为别名对象分配一个固定的名称。

    .. change::
        :tags: orm
        :tickets: 2019

      在joined-table继承映射器上指定本地映射表上没有主键（但在超类表上有pks）会引发警告。

    .. change::
        :tags: orm
        :tickets: 2038

      修复了一个错误，其中如果一个"中间"类在多态层次结构中具有"polymorphic_on"列，但没有指定"polymorphic_identity"，则刷新期间该类将没有"polymorphic_on"列（并且不会对该列进行预加载）将用不合适的错误类加载。

    .. change::
        :tags: orm
        :tickets: 1995

      修复了一个错误，即如果将SQL或服务器端默认值的列用include_properties或exclude_properties从映射中排除，则会导致UnmappedColumnError。

    .. change::
        :tags: orm
        :tickets: 2046

      在关系()和column_property()中添加了active_history标志，可以强制属性事件始终加载“旧”值，以便该值可用于attributes.get_history()。

    .. change::
        :tags: orm
        :tickets: 2044

      在inheritance映射器上指定version_id_col时，如果继承的映射器已经有一个，则发出警告，如果那些列表达式不相同，则发出警告。

    .. change::
        :tags: orm
        :tickets:

      修复了查询中的标签问题，其中在任何列表达式未标记的情况下，命名元组将错误地应用标记。

    .. change::
        :tags: orm
        :tickets: 1914

      添加了一个新的“lazyload”选项“immediateload”。随着对象的填充，自动发出通常的“lazy”加载操作。在将对象加载到离线缓存中或其他情况下，希望使用直接'选择'加载而不是'已加入'或'子查询'时使用。

    .. change::
        :tags: orm
        :tickets: 1920

      新的查询方法：query.label（name），query.as_scalar（），返回带/不带标签的查询语句作为标量子查询；query.with_entities（\*ent），使用新实体替换查询的SELECT列表。大致相当于一种可生成的形式的查询.values（），接受映射实体以及列表达式。

    .. change::
        :tags: orm
        :tickets:

      修复了当一个对象从一个引用移动到另一个引用时引发递归错误，并涉及反向参考，其中发起的父项是前一个父项的子类（具有自己的映射器）。

    .. change::
        :tags: orm
        :tickets: 1918

      修复了0.6.4中发生的回归，如果将空列表传递给mapper（）中的"include_properties"。

    .. change::
        :tags: orm
        :tickets:

      在查询中修复了标记错误，其中，在任何列表达式不带标签的情况下，该命名元组会错误地应用标签。

    .. change::
        :tags: orm
        :tickets: 1925


修复了一个查询语句中join()方法被错误适配左连接右边的问题；

增强了Query.select_from()的方法，以保证查询对象的实体entity字段默认使用select_from()实体而不是Query对象列表中的第一个实体；

当Session在Autocommit=False模式下执行flush失败导致子事务回滚时，Session发出的异常消息已被重新定义；

Mapper在初始化失败后，以及Mapper启动时重复请求其初始化时，异常消息不再假定属性具有“hasattr”情况，因为有其他情况也会产生此消息，并且这个消息也不会重复多次复合在一起；

修复了query.update()中的一个bug，其中如果列表达式键是具有不同键名的类属性，则“评估”或“获取”过期将失败；

在flush期间添加了断言，以确保没有针对“新建永久性”对象生成具有空值的标识键；

lazy load关系属性现在在发出SQL时使用外键和主键属性的当前状态而不是“已提交”状态，前提是没有正在执行flush操作；

relationship()新增一个加载标志load_on_pending，可在未执行flush的情况下启动用于挂起的对象以及手动“附加”到会话的瞬时对象的懒惰加载程序；

关系relationship()具有一个新标志cascade_backrefs，它在“双向”关系的“反向”方面发起事件时禁用“save-update”级联；

如果只将passive_updates = False放置在关系的“多对一”侧，则对这个行为进行了轻微的改进；

在关系的“多对一”上放置passive_deletes=True，则会发出警告，因为您可能打算将其放在“一对多”侧；

修复了subqueryload与子类的单表继承的关系上的工作问题，例如从子类到父类的关系，其中“where”形式为type in（x，y，z）仅在内部被放置，而不是重复被放置；

使用single table inheritance使用from_self()时，“where type in（x，y，z）”仅放置在查询的外部，而不是被重复引用；

scoped_session现在在调用configure()时发出警告，如果已经存在Session（只检查当前线程）；

重新整理Mapper.cascade_iterator()的内部实现以减少某些情况下的方法调用约9%；

TypeDecorator现在可以具有完全构造的类型而不仅仅是类型类；

可在Callable中使用type_coerce(expr, type)表达式元素，以在评估表达式和处理结果行时将给定表达式视为给定类型，但不影响SQL的生成，除了匿名标签；

Table.tometadata()现在还复制与Table相关的Index对象；

如果具有还未分配名称的Column，则使用在declarative中，将在导出到包含选择的列集合的上下文中使用该Column时引发信息性错误消息；

@classproperty现在可以用于基类上的__mapper_args__，__table_args__，__tablename__ 。对于使用者来说，目前没有使用@classproperty的好处。目前时间是如同没有使用@classproperty一样。但是我们至少允许它如预期功能般运行；

如果在多个列中存在相同名称，则警告消息现在会显示“无法添加附加列”消息；

修复了load对于任何自定义类型的问题，例如“枚举”从而构建的“domain”被反射的内置类型；

取消了保留字属性names，在_firebird.py中；

在Oracle方言中，少量的实现变化现在可以使得使用ROWID类型更加简单；

更新了文档；

重构示例，以便Session，缓存管理器，declarative_base是环境的一部分。；- 版本发布日期为Sun Mar 28 2010
- ORM功能方面：新增了relationship()的“子查询”加载功能。这是一个急切加载选项，用于在一个查询中生成第二个SELECT，并将其应用于目标集合，以一次性地加载所有这些集合的结果。类似于“join”急切加载，但使用所有内部连接，并且不会反复重新获取完整的父行（大多数DBAPI似乎都这样做，即使跳过了列）。子查询加载在映射器配置级别使用“lazy ='subquery'”并且在关系级别使用“lazy ='subquery'”。
- ORM功能方面：Python的gc使用内置的标记和解除引用技术进行垃圾收集。对于ORM的存储对象（通常在会话结束时），使用标记清除算法进行垃圾回收处理。增加了批量垃圾收集的支持，以缓解此类应用程序的瓶颈，这是通过在批次中处理条目并进行懒惰标记清除来实现的：所有被存储的对象都被相应地标记，然后保留在批次内，直到第一次有足够的数量的垃圾被收集到批号中为止。
- ORM功能方面：将Session.add\_all()的内部行为更改为直接通过bulk \_save\_objects()，而不是先添加到“new”列表中，然后再通过add\_iter()或add()添加。这非常适合于大量创建对象的情况。
- ORM功能方面：使用default\_group和ExplicitGrouping来提高SQL解析的可预测性，这解决了跨不同方言和不同实现的无限/递归表别名的问题。这在with\_polymorphic()和other\_config的设置中尤其有用。
- ORM功能方面：对于可扩展类中的关系或属性，在属性构造函数上调用super()将使它们像默认映射一样“自我注册”，而不是被忽略。
- ORM功能方面：复杂的查询查询（具有数量的联合和深层嵌套的子查询）现在可以通过只使用SELECT语句中的子集作为预准备算法的解析器来解决。
- ORM功能方面：使用mklnk即使二级cascading的配置改变，mapper-configureable的级联行为仍然是正确的。
- ORM功能方面：在查询中添加更多复杂的计算列支持：支持在SELECT或WHERE表达式中使用嵌套子查询的列名，并支持在嵌套子查询的SELECT子句中再次包含嵌套子查询。
- ORM功能方面：新增了one（）方法，它类似于first（），但省略了Order By，通常情况下速度更快，一般用于评论查询，只需要最新的一条记录。
- ORM功能方面：新增了“load\_on\_pending”属性，即是否在有ORM对列的插入（INSERT）操作时加载与该列相关联的对象，这在需要在父对象的新对象上立即执行插入而父对象和对象之间的关系不再是从数据库加载，而是在内存中创建的时候特别有用。
- ORM功能方面：RelationshipProperty类现在也被称为relationship()，从而可以轻松进行扩展，但仍应保持向后兼容性。老方法还被添加到了RelationshipProperty类，使关系配置和配置更加依赖于属性名称。
- ORM功能方面：移除了mapper()中一些旧的字串参数（如uselist），并且cancel.deprecated属性不再接受这些设置。该部分的配置接口应仅由_ORMC即mapper.configure\_params或_ORMC即mapper.configure\_properties进行设置。
- ORM功能方面：在分派“对JSON类型的SQL表达式别名作为字符串列（JsonEncodedDict / JsonEncodedList”）时，两个别名现在都被调用为json。已添加.json\_encoder属性，允许指定按名称引用的模块，以提供自定义的JSON编码器。
- ORM功能方面：整个ORM现在都为“自底向上的”模式。即一个反映DB结构映射的mapper最终是一个抽象类，而其他具体的Python类型都是mapper或抽象类。这是为了更好地支持HEADS使用CASE结构的ORM类型。
- ORM功能方面：新增了Session.flush\_sequences属性，该属性返回所有可用的Sequence对象，用于在生成默认值之前手动插入标识符值。
- ORM功能方面：使用keep\_session()装饰器保持会话不会被关闭，直到嵌套会话被推出，这适用于大多数情况，而不是显式传递会话到使用它的所有地方。
- sqlalchemy.util.plugin.PluginLoader现在包括对包内工具模块的支持：即，它可以扫描模块的顶层__init__.py文件，以确定哪些模块应被视为实际插件。类自身现在也是
- sa.Column进行初始化的所有参数都可选，接受的唯一必需参数是Column编写的属性的名称。

版本：0.6beta2

发布：2010年3月20日

全局

- 安装/测试设置上的改进，特别是解决分布式在py3k上的问题。
- 将关系函数relation()更改为relationship()。
- 添加version_id_generator参数给Mapper。
- 在string类型中添加unicode_errors参数。
- 开始支持math negation操作，即-x。
- Removed the keys()方法支持ResultProxy。
- 对于Join的左边返回与右边完全一样的SQL代码。

ORM

- 有"subqueryload(props..)", "subqueryload_all(props...)"和"eagerload_all(props...)"两种新的预读取方式。
- eagerload()和eagerload_all()现已更名为joinedload()和joinedload_all()。
- relationship()取代了relation()。
- ForeignKey允许空值和空字符串。
- relationship()中的__table__.exists()只适用于单一继承。
- "lazy"标志现在接受字符串参数。
- 根据警告看到的内容更改表实例。
- 优化加载过程，提升速度。
- 修改了Query.join()和aliased()。
- 再次修复“one-to-many”广度优先加载问题。
- 用属性名称而不是属性实例调用join()时，并且存在其他别名对象与该属性对象具有相同名称，Query会更准确地调用join()。

SQL

- 添加了with_hint()方法来向Query() construct添加新的select().with_HINT()功能。
- 标签由代替。
- 这个pull request扩大了MySQL的支持范围。
- 删除 join()上的to_outerjoin()。
- 修复了行交换操作时不更新主键列的不必要更新问题。

Engines

- Engine对象的logger名称更改为hex字符串。

declarative

- 现在可以直接接受mixin类。
- 修正一个错误，即如果单表子类指定了已经存在于基类中的列，则会抛出异常。

Postgresql

- 现在直接使用时间戳和时间类型类型。

mysql

- 不再猜测TINYINT(1)应该是布尔型。如果要获得布尔转换行为，请在表定义中使用Boolean/BOOLEAN。
- 在特定条件下，视循环检测为误报并生成警告。改动：

对beaker的缓存示例进行了一些更改，增加了一个专门用于lazyload缓存的RelationCache选项。通过将多个属性分组到一个公共结构中，可以更高效地在任意数量的潜在属性之间进行查找。FromCache和RelationCache都是单独的。

文档中进行了重大清理工作，将类名、函数名和方法名链接到API文档中。

发布版本号为0.6beta1，发布时间为2010年2月3日。

更改：

完整的功能描述集请参见https://docs.sqlalchemy.org/en/latest/changelog/migration_06.html。该文档仍在完善中。

最新的0.5版本及以下的所有bug修复和功能增强都已包含在0.6中。

现在所针对的平台包括Python 2.4/2.5/2.6、Python 3.1和Jython2.5。

对query.update()和query.delete()进行了更改：

- query.update()上的“expire”选项已重命名为“fetch”，与query.delete()相匹配。 "expire"已作废并会发出警告。
- query.update()和query.delete()都默认使用"synchronize"策略中的"evaluate"。- update()和delete()的"synchronize"策略在失败时引发错误。没有隐式地回到“fetch”。评估失败是基于标准的结构，因此基于代码结构，成功/失败是确定性的。

改进了多对一关系：

- 现在，在许多情况下，多对一的关系将会更少地触发lazyload，包括在大多数情况下将不会在替换新值时提取“旧”值。
- 来自连接表子类的多对一关系现在使用get()进行简单的加载（称为“使用get条件”），即Related->Sub(Base)，无需重新定义基表中的primaryjoin条件。
- 使用声明列指定外键，即ForeignKey(MyRelatedClass.id)不会打破发生“使用获取”条件的状态。
- relation()，eagerload()和eagerload_all()现在具有一个名为“innerjoin”的选项。指定True或False以控制eager join是作为INNER还是OUTER join构造的。默认值与往常一样为False。mapper选项将覆盖relation()上指定的任何设置。通常应为多对一而设置，以允许提高连接性能。
- 当LIMIT / OFFSET存在时，饱和加载的行为现在除了所有eager loads都是多对一连接的情况之外，主查询被包装在子查询中，其中这些eager loads是针对父表直接进行的，同时限制/offs，没有子查询的额外开销，因为多对一连接不会向结果添加行。

改进Session.merge()：

- 现在，Session.merge()是性能优化的，对于“load=False”模式，这比0.5版本的调用计数少一半，并且对于使用“load=True”模式的集合而言，在执行较少的SQL查询中会有显着减少。
- 如果给定的实例与已存在的实例相同，则Session.merge()将不会发出属性的不必要合并。

现在，Session.merge()也将合并与给定状态相关联的“options”，例如通过query.options()与实例一起传递的选项，例如选项以eager或lazy的方式加载不同的属性。这对于构建高度集成的缓存方案非常重要。与0.5版本相比，这是一个微妙的行为变化。

修复了一个关于存在于实例状态的“加载程序路径”的序列化的错误，这也是与合并()结合使用并应保存的关联选项的必要性。

新的merge()在一个新的综合示例中进行了展示，展示了如何与SQLAlchemy集成Beaker。请参阅下面的“示例”说明中的注释。

现在，可以更改连接表继承对象上的主键值，并且ON UPDATE CASCADE将在刷新发生时考虑其中的内容。在SQLite或MySQL / MyISAM中使用mapper()时，请在mapper()上设置新的“passive_updates”标志为False。

现在，flush()可以检测到主键列是否由另一个主键的ON UPDATE CASCADE操作更新，并且可以定位新PK值的行以进行后续更新。当relation()存在以确立关系以及使用passive_updates=True时发生这种情况。

现在，“save-update”级联将在添加操作中将标量或集合属性中的挂起* removed *值级联到新会话中，以便刷新（）操作也将删除或修改那些已断开连接的项目的行。

使用“secondary”表的“dynamic”加载器现在会生成一个查询，其中“secondary”表未被别名。这允许在关系的order_by属性中使用次要表对象，并允许在根据动态关系的过滤条件中使用它。

当eagerload()调用超过四个参数时，将发出警告。

当在synonym()上使用map_column = True时，现在会明确检查是否存在单独存在于属性字典中的ColumnProperty（延迟或其他类型），在某些映射器上的同名键名。而不是静默地替换现有属性（和可能存在于该属性上的选项），将引发错误。

现在，“dynamic”加载器在构造时设置其查询标准，以便通过像“statement”这样的非克隆访问器返回实际查询操作。

现在，Query中迭代的“命名元组”对象是可pickle化的。

现在必须对select()结构进行显式地alias()映射，以从中映射到。请注意，这将消除关于此类问题的混淆，例如

query.join()已重新设计以提供更一致的行为和更多的灵活性（包括）

query.select_from()接受多个子句以在FROM子句内生成多个逗号分隔的条目。在选择多个联接的情况下很有用。

query.select_from()还接受映射类、别名()结构和映射器作为参数。特别是这有助于当查询来自多个连接表类时，确保完整的连接被呈现。

具有属性值为None的主键列可以与映射到外部连接的查询一起使用。

现在，用于将select()构造映射到select()构造的映射需要将其作为别名()构造独立出来。这是为了消除对这样的问题的混淆，例如：

现在，relation()的primaryjoin和secondaryjoin现在会检查它们是否是列表达式，而不仅仅是子句元素。这禁止直接将FROM表达式放置在此处。

现在，expression.null()在使用query.filter()、filter_by()等过滤器时，完全像None那样被理解用于比较对象/集合引用属性。

添加了“make_transient()”辅助函数，将持久/分离实例转换为瞬态实例（即删除实例键并从所有会话中删除）。

现在，mapper()上的allow_null_pks标志已被弃用，并且该特性默认“打开”。这意味着，任何主键列具有任何非空值的行都将被认为是标识。此情况通常仅在映射到外连接时发生。

“backref”的力学已完全合并到更精细的“back_populates”系统中，并完全在RelationProperty的 _generate_backref()方法内发生。这使得RelationProperty的初始化过程更简单，并允许更容易地将设置（例如从RelationProperty的子类）传播到反向引用中。内部的BackRef()已经不存在，backref()返回一个简单的tuple，RelationProperty已被理解。

mapper()上的version_id_col功能将在使用不支持“rowcount”足够的方言时发出警告。

添加了“execution_options()”到Query中，因此可以将选项传递到生成的语句中。目前，只有Select语句具有这些选项，唯一使用的选项是“stream_results”，以及唯一知道“stream_results”的方言是psycopg2。

Query.yield_per()将自动设置“stream_results”语句选项。

save()，update()和save_or_update()已删除。使用session.add()和session.add_all()。

显式检查检查同义词()是否与map_column = True一起使用时，列名是ClauseElement而不仅仅是称为Column的对象。而不是默默地替换现有的属性（和可能在该属性中的选项），将引发错误。

用于entrypoint驱动方言的导入现在不依赖于愚蠢的tb_info技巧，以确定导入错误状态。

ResultProxy现在具有名为first（）的方法，立即返回第一行并关闭结果集。

RowProxy对象现在是可pickle化的，即result.fetchone()，result.fetchall()等返回的对象。

RowProxy不再具有close()方法，因为该行不再保留对父对象的引用。在父ResultProxy上调用close()，或使用autoclose。

当获取列时，新的ResultProxy内部进行了彻底的改进，以大大减少方法调用计数。对于获取大型结果集时，可以提供大量的速度改进（多达100％以上）。当获取没有应用类型级别处理的列时，并且当将结果作为元组（而不是作为字典）时，改进会更大。感谢Elixir的Gaëtan de Menten为这一巨大改进做出的贡献！

现在，依赖于“last inserted id”的“插入”操作或“update”操作中的“后提取”现已正确处理具有组合主键的情况，其中“自动增量”列不是表中的第一个主键列。

现在，“last_inserted_ids()”方法已重命名为描述符“inserted_primary_key”。

现在，使用所有public keyword参数复制ForeignKey和ForeignKeyConstraint对象。

ForeignKey和ForeignKeyConstraint对象现在正确地将其所有公共关键字参数复制()。

元数据的“__contains __（）”方法现在接受字符串或表对象作为参数。如果给定一个“Table”，则会首先将参数转换为“table.key”，即“[schemaname。]<tablename>”。

已删除了metadata.table_iterator()方法（使用sorted_tables）

已删除了PassiveDefault-使用DefaultClause。

所有公共可变性从Index和Constraint对象中被删除：

- ForeignKeyConstraint.append_element()
- Index.append_column()
- UniqueConstraint.append_column()
- PrimaryKeyConstraint.add()
- PrimaryKeyConstraint.remove()

如果需要，这些应以声明的方式构造（即在一次构造中）。

Sequence上的“start”和“increment”属性现在在Oracle和PostgreSQL上默认生成“START WITH”和“INCREMENT BY”。Firebird目前不支持这些关键字。

UniqueConstraint，Index和PrimaryKeyConstraint都接受列名或列对象列表作为参数。

Table.key已删除（不知道这是为什么）

Table.primary_key不可分配-使用table.append_constraint(PrimaryKeyConstraint(...))

Column.bind（通过column.table.bind获取）

Column.metadata（通过column.table.metadata获取）

Column.sequence（使用column.default）

ForeignKey（约束=某些parent）（现在是私有_constraint）

ForeignKey和ForeignKeyConstraint现在正确复制其所有public关键字参数()。

视图现在可以作为普通的Table对象进行推导。使用相同的Table构造函数，但必须注意，“有效”的主键和外键约束不是反射结果的一部分。如果需要，这些必须显式指定。

现有的autoload=True系统现在使用Inspector在其下面，从而每个方言仅需要返回关于表和其他对象的“原始”数据- Inspector是编译信息的单一位置，因此最大程度地确保了一致性。

现在，DDL中的DDL()类已大大扩展。DDL() 概况现在扩展了更通用的DDLElement()，这是许多新构造的基础：

- CreateTable()
- DropTable()
- AddConstraint()
- DropConstraint()
- CreateIndex()
- DropIndex()
- CreateSequence()
- DropSequence()

与纯DDL()一样，这些都支持"on"和"execute-at()"。可以创建用户定义的DDLElement子类，并使用sqlalchemy.ext.compiler扩展将其链接到编译器。

.. change::
    :tags: ddl
    :tickets:

"on"回调传递给DDL()和DDLElement()的签名如下修改：

    ddl
        DDLElement对象本身
    event
        字符串事件名称。
    target
        先前的“schema_item”，触发事件的Table或MetaData对象。
    connection
        执行操作的Connection对象。
    \**kw
        关键字参数。在_MetaData之前/之后_create/drop_中，将要发出CREATE /DROP DDL的表对象列表作为kw参数“tables”传递。对于依赖于特定表存在的元数据级DDL，这是必需的。

DDL的"schema_item"属性已重命名为"target"。

.. change::
    :tags: dialect，refactor
    :tickets:

现在，方言模块已分为数据库方言和DBAPI实现。现在，首选使用方言+驱动程序的连接URL，即“mysql+mysqldb://scott:tiger @localhost/test”。有关示例，请参见0.6文档。

.. change::
    :tags: dialect，refactor
    :tickets:

外部方言的setuptools entrypoint现在称为“sqlalchemy.dialects”。

.. change::
    :tags: dialect，refactor
    :tickets:

已从Table中删除"owner"关键字参数。使用"schema"表示要添加到表名之前的任何名称空间。

.. change::
    :tags: dialect，refactor
    :tickets:

server_version_info成为静态属性。

.. change::
    :tags: dialect，refactor
    :tickets:

方言在初始连接时接收initialize()事件，以确定连接属性。

.. change::
    :tags: dialect，refactor
    :tickets:

方言接收visit_pool事件，有机会建立池侦听器。

.. change::
    :tags: dialect，refactor
    :tickets:

缓存的TypeEngine类按方言类而不是按方言缓存。

.. change::
    :tags: dialect，refactor
    :tickets:

应使用新的UserDefinedType作为新类型的基类，以保留0.5 get_col_spec()的行为。

.. change::
    :tags: dialect，refactor
    :tickets:

所有类型类的result_processor()方法现在接受第二个参数"coltype"，即来自cursor.description的DBAPI类型参数。此参数可帮助某些类型决定结果值的最有效处理方式。

.. change::
    :tags: dialect，refactor
    :tickets:

已删除已弃用的Dialect.get_params()。

.. change::
    :tags: dialect，refactor
    :tickets:

Dialect.get_rowcount()已重命名为描述符"rowcount"，并直接调用cursor.rowcount。需要为某些调用硬编码rowcount的方言覆盖该方法以提供不同的行为。

.. change::
    :tags: dialect，refactor
    :tickets: 1566

DefaultRunner及其子类已被删除。该对象的工作已简化并移至ExecutionContext。 支持序列的方言应向其执行上下文实现添加fire_sequence()方法。

.. change::
    :tags: dialect，refactor
    :tickets:

编译器生成的函数和运算符现在使用(几乎)常规的分派函数，格式为"visit_<opname>"和"visit_<funcname>_fn"，以提供定制处理。这替换了在编译器子类中复制"函数"和"操作符"字典所需的需求，同时还允许编译器子类完全控制呈现，因为完整的_Function或_BinaryExpression对象被传递。

.. change::
    :tags: postgresql
    :tickets:

新方言：pg8000，zxjdbc和pypostgresql在py3k上。

.. change::
    :tags: postgresql
    :tickets:

"postgres"方言现在已更名为"postgresql"！连接字符串的格式如下：

      postgresql：// scott：tiger @ localhost / test
      postgresql + pg8000：// scott：tiger @ localhost / test

为了向后兼容，"postgres"名称仍然以以下方式导入：

     -  "postgres.py"虚拟方言允许旧URL正常工作，即
         postgres：// scott：tiger @ localhost / test

     -  "postgres"名称可以从旧的"databases"模块导入，即
         "from sqlalchemy.databases import postgres"以及
         "dialects"，"from sqlalchemy.dialects.postgres
         import base as pg"，将发送退役
         警告。

     -  特殊表达式参数现在命名为
         "postgresql_returning"和"postgresql_where"，但是
         旧的"postgres_returning"和"postgres_where"名称仍然使用退役警告。

.. change::
    :tags: postgresql
    :tickets:

"postgresql_where"现在接受SQL表达式，也可以包含文字，需要对其进行引号处理。

.. change::
    :tags: postgresql
    :tickets:

psycopg2方言现在在所有新连接上使用psycopg2的"unicode扩展"，它允许所有String / Text /等。类型跳过将bytestrings处理为unicode的必要性（由于obj的体积，这是一个昂贵的步骤）。返回本地unicode(pg8000，zxjdbc)的其他方言也跳过了unicode后处理。

.. change::
    :tags: postgresql
    :tickets: 1511

添加了新的ENUM类型，作为基于架构的构造存在，并扩展了通用的Enum类型。会自动将自己与表及其父元数据相关联，以根据需要发出相应的CREATE TYPE / DROP TYPE命令，支持unicode标签，支持反射。

.. change::
    :tags: postgresql
    :tickets:

INTERVAL支持可选的“precision”参数，对应于PG接受的参数。

.. change::
    :tags: postgresql
    :tickets:

使用新的dialect.initialize()功能来设置特定版本的行为。

.. change::
    :tags: postgresql
    :tickets: 1279

在表/列名称中的％符号的支持稍微好了一些；然而，当使用executemany()时，psycopg2无法处理名为%(foobar)s的绑定参数名称，而SQLA不想添加开销来处理那个不存在的使用情况。

.. change::
    :tags: postgresql
    :tickets: 1516

将NULL插入具有主键+外键列将允许"not null constraint"错误引发，而不是尝试执行不存在的"col_id_seq"序列。

.. change::
    :tags: postgresql
    :tickets:

SELECT自增语句，即那些从修改行的过程中选择的语句，现在可在服务器端游标模式下使用（命名游标不用于这些语句）。

.. change::
    :tags: postgresql
    :tickets: 1636

postgresql方言现在可以正确检测到pg“devel”版本字符串，即“8.5devel”。

.. change::
    :tags: postgresql
    :tickets: 1619

psycopg2现在尊重语句选项"stream_results"。该选项覆盖了连接设置"server_side_cursors"。 如果为true，则语句将使用服务器端游标。如果为false，则不会使用，即使在连接上"server_side_cursors"为true也是如此。

.. change::
    :tags: mysql
    :tickets:

新方言：oursql，一个新本机方言，MySQL Connector / Python，MySQLdb的本机Python端口，当然，还有Jython上的zxjdbc。

.. change::
    :tags: mysql
    :tickets:

VARCHAR / NVARCHAR现在将不带长度呈现，将在传递到MySQL之前引发错误。由于CAST中不允许使用VARCHAR，因此在这些情况下，方言呈现CHAR / NCHAR。

.. change::
    :tags: mysql
    :tickets:

所有_detect_XXX()函数现在仅运行一次dialect.initialize()之下

.. change::
    :tags: mysql
    :tickets: 1279

在表/列名称中的％符号的支持稍微好了一些；当使用executemany()时，MySQLdb无法处理SQL中的％符号，SQLA不想添加开销来处理那个不存在的使用情况。

.. change::
    :tags: mysql
    :tickets:

二进制和MSBinary类型现在在所有情况下均生成"BINARY"。省略"length"参数将生成没有长度的"BINARY"，使用BLOB可以生成未经长度处理的二进制列。

.. change::
    :tags: mysql
    :tickets:

"MSEnum / ENUM"的引用关键字参数quoting='quoted'已弃用。最好依赖于自动引用。

.. change::
    :tags: mysql
    :tickets:

Enum现在是新的通用Enum类型的子类，并且如果给定的标签名称是unicode对象，则隐式处理unicode值。

.. change::
    :tags: mysql
    :tickets: 1539

如果未将"nullable=False"传递给Column()，则类型为TIMESTAMP的列现在默认为NULL，如果没有默认值。现在与所有其他类型一致，并且在TIMESTAMP中显式呈现"NULL"，因为MySQL对TIMESTAMP列的默认空值进行了"switching"。

.. change::
    :tags: oracle
    :tickets:

cx_oracle现在可以让单元测试100％通过！

.. change::
    :tags: oracle
    :tickets:

支持cx_Oracle的“本机unicode”模式，该模式不需要设置NLS_LANG。使用最新的5.0.2或更高版本的cx_oracle。

.. change::
    :tags: oracle
    :tickets:

已添加NCLOB类型到数据库类型中。

.. change::
    :tags: oracle
    :tickets:

使用_ansi=False不会泄漏到从使用JOIN / OUTERJOIN的子查询选择的FROM / WHERE子句中。

.. change::
    :tags: oracle
    :tickets: 1467

添加本机INTERVAL类型到方言。由于缺乏cx_oracle对YEAR TO MONTH的支持，因此目前仅支持DAY TO SECOND interval类型。

.. change::
    :tags: oracle
    :tickets:

使用CHAR类型将绑定cx_oracle的FIXED_CHAR dbapi类型到语句中。

.. change::
    :tags: oracle
    :tickets: 885

Oracle方言现在具有作为Oracle类型数字类型的NUMBER。它是由表反射返回的主要数字类型，并根据精度/比例参数尝试返回Decimal()/float/int。

.. change::
    :tags: oracle
    :tickets:

func.char_length是用于长度的通用函数。

.. change::
    :tags: oracle
    :tickets:

包括onupdate=<value>的ForeignKey()将发出警告，不发出不受Oracle支持的ON UPDATE CASCADE。

.. change::
    :tags: oracle
    :tickets:

RowProxy()的keys()方法现在返回被标准SQLalchemy不区分大小写的名称规范化的结果列名。这意味着对于大小写不敏感的名称，它们将是小写的，而DBAPI通常会以大写名称返回它们。这允许行键()与进一步的SQLAlchemy操作兼容。

.. change::
    :tags: oracle
    :tickets:

使用新的dialect.initialize()功能来设置特定版本的行为。

.. change::
    :tags: oracle
    :tickets: 1125

使用Oracle将types.BigInteger与NUMBER(19)生成。

.. change::
    :tags: oracle
    :tickets:

"大小写敏感"功能将检测到在reflect时所有小写大小写敏感的列名，并在生成的Column中添加"quote=True"，以便维护正确的引用。

.. change::
    :tags: firebird
    :tickets:

RowProxy()的keys()方法现在返回被标准SQLalchemy不区分大小写的名称规范化的结果列名。这意味着对于大小写不敏感的名称，它们将是小写的，而DBAPI通常会以大写名称返回它们。这允许行键()与进一步的SQLAlchemy操作兼容。

.. change::
    :tags: firebird
    :tickets:

使用新的dialect.initialize()功能来设置特定版本的行为。

.. change::
    :tags: firebird
    :tickets:

"大小写敏感"功能将检测到在reflect时所有小写大小写敏感的列名，并在生成的Column中添加"quote=True"，以便维护正确的引用。

.. change::
    :tags: mssql
    :tickets:

MSSQL + Pyodbc + FreeTDS现在基本上可以正常工作，可能会涉及二进制数据以及unicode架构标识符的异常情况。

.. change::
    :tags: mssql
    :tickets:

"has_window_funcs"标志已删除。 LIMIT / OFFSET使用将像总是一样使用ROW NUMBER，并且如果在较旧版本的SQL Server上，操作失败。除了SQL server将错误引发而不是方言之外，行为完全相同，不需要设置标志以启用它。

.. change::
    :tags: mssql
    :tickets:

已删除"auto_identity_insert"标志。此功能始终在覆盖已知具有序列的列的INSERT语句时生效。与"has_window_funcs"一样，如果底层驱动程序不支持此操作，则无论如何都不能执行此操作，因此没有标志。

.. change::
    :tags: mssql
    :tickets:

使用新的dialect.initialize()功能来设置特定版本的行为。

.. change::
    :tags: mssql
    :tickets:

已删除对不再使用的序列的引用。在mssql上，隐式标识与在任何其他方言上隐式序列的行为相同。使用 "default = Sequence()" 可以通过构造显式序列来启用它。有关更多信息，请参见MSSQL方言文档。

.. change::
    :tags: sqlite
    :tickets:

DATE，TIME和DATETIME类型现在可以采用可选的storage_format和regexp参数。storage_format可用于使用自定义字符串格式存储这些类型。regexp允许使用自定义正则表达式匹配来自数据库的字符串值。

.. change::
    :tags: sqlite
    :tickets:

Time和DateTime类型现在使用更严格的正则表达式来匹配来自数据库的字符串值（默认情况下）。如果您使用存储在旧格式中的数据，请使用regexp参数。

.. change::
    :tags: sqlite
    :tickets:

__legacy_microseconds__在SQLite Time和DateTime类型上不再受支持。您应使用storage_format参数。  

.. change::
    :tags: sqlite
    :tickets:

Date, Time和DateTime类型现在严格限制它们接受的绑定参数：Date类型仅接受日期对象（和日期时间对象，因为它们从日期继承），Time仅接受time对象，DateTime仅接受日期和日期时间对象。

.. change::
    :tags: sqlite
    :tickets: 1016

Table()支持关键字参数“sqlite_autoincrement”，在生成DDL时将SQLite关键字"AUTOINCREMENT"应用于单个整数主键列。将防止生成独立的PRIMARY KEY约束。

.. change::
    :tags: types
    :tickets:

dialect中类型的构造已完全重构。 方言现在将公共可用类型定义为仅大写名称，并使用下划线标识符（即是私有的）来定义内部实现类型。 表示SQL和DDL中类型的系统已移至编译器系统。 这意味着在大多数方言中都有更少的类型对象，用于方言建议的文档的详细说明在lib / sqlalchemy / dialects / type_migration_guidelines.txt中。

.. change::
    :tags: types
    :tickets:

类型不再猜测默认参数。具体而言，Numeric，Float，NUMERIC，FLOAT，DECIMAL现在除非指定了这些值，否则不会生成任何长度或比例。

.. change::
    :tags: types
    :tickets: 1664

types.Binary已重命名为types.LargeBinary，它仅生成BLOB，BYTEA或类似的“长二进制”类型。添加了基本的BINARY和VARBINARY类型，以便以通用方式访问这些MySQL / MS-SQL特定的类型。

.. change::
    :tags: types
    :tickets:

如果dialect检测到DBAPI本地返回Python unicode对象，则String / Text / Unicode类型现在对每个结果列值跳过unicode()检查。首次使用"SELECT CAST 'some text' AS VARCHAR(10)" 或等效语句时，将发出此检查，然后检查返回的对象是否为Python unicode。这允许原生unicode DBAPI（包括pysqlite / sqlite3，psycopg2和pg8000）的巨大性能提升。

.. change::
    :tags: types
    :tickets:

已检查大多数类型结果处理程序以获取可能的速度提升。 具体而言，已优化以下通用类型，从而导致不同的速度提升：Unicode，PickleType，Interval，TypeDecorator，Binary。 此外，以下dbapi特定实现已得到改善：Sqlite上的Time，Date和DateTime，PostgreSQL上的ARRAY，MySQL上的Time，MySQL / oursql和pypostgresql上的Numeric（as_decimal = False），cx_oracle上的DateTime和基于LOB的类型上的cx_oracle。

.. change::
    :tags: types
    :tickets:

类型的反射现在返回types.py中的确切大写类型或如果该类型不是标准SQL类型则返回dialect本身的大写类型。这意味着反射现在返回更准确的反射类型信息。

.. change::
    :tags: types
    :tickets: 1511, 1109

添加了一个新的通用Enum类型。 Enum是一个支持需要特定DDL才能使用enum或等效内容的数据库的模式感知对象； 在PG的情况下，它处理'CREATE TYPE'的细节，在其他没有本地枚举支持的数据库上将生成VARCHAR+内联CHECK约束以强制执行枚举。

.. change::
    :tags: types
    :tickets: 1467

间隔类型包括一个"native"标志，用于控制是否选择本机间隔类型（postgresql + oracle）如果可用，或者不选择。还添加了“day_precision”和“second_precision”参数，如果适当地传递到这些本机类型，将相应地进行传递。相关的。

.. change::
    :tags: types
    :tickets: 1589

当布尔类型在没有本机布尔支持的后端上使用时，将生成一个CHECK约束"col IN（0, 1）"以及基于int / smallint的列类型。 如果不需要此功能，则可以将create_constraint = False关闭。 请注意，MySQL既没有本机布尔支持也没有CHECK约束支持，因此在该平台上不可用。

.. change::
    :tags: types
    :tickets:

当mutable=True时，现在将为PickleType使用==比较值，除非指定了具有比较功能的"comparator"参数来提供类型。进行pickle的对象将基于身份进行比较（这会破坏mutable=True），如果__eq__()未被覆盖或没有提供比较函数，则如此。

.. change::
    :tags: types
    :tickets:

Numeric和Float的默认"precision"和"scale"参数已删除，现在默认为None。 NUMERIC和FLOAT不默认提供任何数字参数，除非提供这些值。

.. change::
    :tags: types
    :tickets:

已删除AbstractType.get_search_list() - 对于它使用的游戏不再需要。

.. change::
    :tags: types
    :tickets: 1259

传递给association_proxy的proxy_factory可调用的签名现在为（lazy_collection，creator，value_attr，association_proxy），添加了一个第四个参数，即父AssociationProxy参数。 允许序列化和内置集合的子类。

.. change::
    :tags: types
    :tickets: 1372

association_proxy现在具有基本比较方法.any()、.has()、.contains()、 ==、！=，感谢Scott Torborg。

